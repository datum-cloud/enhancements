---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Telemetry Query Layer

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Design Details](#design-details)
  - [Consumers](#consumers)
  - [Request flow](#request-flow)
  - [API registration](#api-registration)
  - [Tenant enforcement](#tenant-enforcement)
  - [Multi-region fanout](#multi-region-fanout)
  - [Result merging](#result-merging)
  - [Streaming](#streaming)
  - [Region registry](#region-registry)
- [Alternatives](#alternatives)

## Summary

The query layer is a thin HTTP service that sits between clients and one or more
regional ClickHouse instances. It resolves tenant identity from bearer tokens,
enforces tenant scoping on ClickHouse sessions, fans out queries across regions,
and merges results before returning them to callers.

It serves two distinct consumers with different access patterns:

- **Tenant users** via `datumctl` — scoped to their own data, streamed log tail
- **Platform operators** via Grafana — cross-tenant, aggregated, no row policy

## Motivation

ClickHouse's `Distributed` table engine fans out queries within a single logical
cluster. It is not suitable for cross-region fanout where data residency
restrictions prohibit data from leaving its region of origin. The query layer
owns fanout explicitly, so only final result rows cross region boundaries — not
intermediate query state.

A single service also provides a consistent enforcement point for tenant
isolation. Without it, tenant scoping depends on each client correctly setting
`project_id` in their ClickHouse session — a fragile contract. The query layer
makes this non-negotiable.

### Goals

- Single HTTP endpoint for both `datumctl` and Grafana
- Tenant isolation enforced server-side, not client-side
- Multi-region query fanout with correct result merging
- Streaming support for live log tail queries
- No cross-region data replication or residency violations

### Non-Goals

- Being a general-purpose SQL proxy
- Replacing Grafana for visualization
- Storing or caching telemetry data
- Long-term log retention — that is ClickHouse's concern

## Design Details

### Consumers

| Consumer | Auth | Scoping | Access pattern |
|---|---|---|---|
| `datumctl` | Bearer token (OIDC) | Single tenant | Streaming log tail, recent history |
| Grafana | Service account | Cross-tenant (operator only) | Aggregated metrics and dashboards |

Grafana is pointed at the query layer, not directly at ClickHouse. This
removes the direct `grafana_ops` → ClickHouse connection from the architecture
and makes the query layer the single enforcement point for all reads.

### Request flow

```
──── Tenant path ─────────────────────────────────────────────────────────────
datumctl logs [--tail]   (bearer token)
    │
    ▼
milo-api
    │  ProjectRouterWithRequestInfo: rewrites URL, injects project_id into
    │  user extras (iam.miloapis.com/parent-name)
    ▼
project control plane (kube-aggregator)
    │  APIService proxy → query layer pod
    ▼
Query Layer
  1. Extract project_id from user extras (injected by milo-api)
  2. Resolve region set: ScopeResolver maps project to regional ClickHouse instances
  3. Fan out: issue query to each resolved regional endpoint
  4. Merge: combine partial results into a single response
  5. Stream or return: chunked HTTP for --tail, complete response otherwise
    │
    ├──▶ ClickHouse (us-central1)   SET project_id = '...'
    ├──▶ ClickHouse (eu-west1)      SELECT ...
    └──▶ ClickHouse (...)

──── Operator path ───────────────────────────────────────────────────────────
Grafana   (service account, operator identity)
    │
    ▼
Query Layer   (cluster-scoped endpoint, no per-project routing)
  1. Verify platform operator identity
  2. Fan out to all configured regional instances
  3. Merge results (no row policy; connects as grafana_ops)
    │
    ├──▶ ClickHouse (us-central1)
    └──▶ ClickHouse (eu-west1)
```

### API registration

The query layer registers as a Kubernetes `APIService` on each project's control
plane under the `telemetry.datumapis.com` API group. This places it inside the
kube-aggregator proxy chain that milo-api uses for all per-project API calls —
`datumctl` does not need to know the query layer's address directly.

The `datumctl` call chain:

```
datumctl
  → milo-api /projects/<id>/control-plane/apis/telemetry.datumapis.com/v1/logs
    → ProjectRouterWithRequestInfo   (URL rewrite + user extras injection)
      → project control plane        (per-project kube-aggregator)
        → APIService proxy           (routes to query layer pod)
          → query layer
```

`ProjectRouterWithRequestInfo` injects `iam.miloapis.com/parent-type: Project`
and `parent-name: <project_id>` into the request's user extras. The query layer
reads `project_id` from these extras rather than re-calling Milo or parsing the
URL path.

**Proxy behavior:**

- **No retries** — kube-aggregator is a transparent reverse proxy. If the query
  layer returns an error the error goes directly back to `datumctl`; there is no
  retry at the proxy. Client-side retries belong in `datumctl`.

- **Rate limiting** — Kubernetes API Priority and Fairness (APF) at the milo-api
  front door handles request queueing. The proxy itself does not rate-limit. The
  query layer can enforce per-project application-layer limits if APF granularity
  is insufficient.

- **Streaming** — kube-aggregator passes chunked HTTP through without buffering.
  See [Streaming](#streaming) for requirements.

### Tenant enforcement

For tenant queries, the query layer connects to ClickHouse as `api_reader` and
issues `SET project_id = '<resolved_project>'` before every query. The
ClickHouse row policy enforces this as a defense-in-depth layer:

```sql
ProjectId = getSetting('project_id')
```

A session without `project_id` set throws — no silent data leak.

The query layer reads `project_id` from the user extras injected into the
request by `ProjectRouterWithRequestInfo` in milo-api. No additional Milo API
call is needed in the query layer — the project identity is already resolved
before the request reaches the APIService.

For operator queries (Grafana), the query layer connects as `grafana_ops`, which
has no row policy. The query layer enforces that only authenticated platform
operators can reach this path.

### Multi-region fanout

Each region runs an independent ClickHouse instance with an identical schema.
Data never replicates across regions. When a query arrives, the query layer:

1. Determines the target region set (see [Region registry](#region-registry))
2. Issues the query in parallel to each target
3. Merges partial results (see [Result merging](#result-merging))

For tenant queries, only the regions that hold that project's data are queried.
For operator queries (e.g. cross-project CPU usage), all regions are queried.

ClickHouse's `Distributed` table engine is explicitly not used for cross-region
fanout. It can transfer intermediate aggregation state across nodes during query
execution, which would violate data residency requirements.

### Result merging

The merge strategy depends on query type:

**Log queries (ordered rows)** — merge by `ObservedTimestamp` descending,
matching the ClickHouse order key. Each region returns an ordered slice; the
query layer performs a k-way merge.

**Simple aggregations** — `SUM` and `COUNT` are additive across regions.
`AVG` requires each region to return `sum` and `count` separately; the query
layer computes the final average. Returning `AVG` directly from each region and
averaging the averages produces incorrect results when regions have different
row counts.

**GROUP BY queries** — group keys must be included in partial results. The query
layer re-aggregates by group key across the regional partial results.

The query language exposed to clients should be constrained to patterns the
query layer knows how to merge correctly. Arbitrary SQL passthrough is not a
goal.

### Streaming

`datumctl logs --tail` requires streaming: rows are returned to the client as
they are read from ClickHouse, not buffered and returned at the end.

The query layer uses chunked HTTP (`Transfer-Encoding: chunked`) and implements
`http.Flusher` to flush rows to the client as they arrive from ClickHouse.
kube-aggregator's reverse proxy passes chunked encoding through without
buffering — this is the same mechanism as `kubectl logs --follow` through the
Kubernetes API aggregation layer.

For multi-region tail queries, the query layer merges streams from each region
by timestamp, emitting the globally oldest row first. Regions that are behind
apply backpressure.

> [!IMPORTANT]
>
> The query layer's `APIService` registration must set `TimeoutSeconds: 0` (no
> timeout). A non-zero value causes the kube-aggregator proxy to terminate
> long-lived tail connections before ClickHouse has finished streaming.

### Region registry

The query layer needs to know which projects a requesting user can access, and
which regions hold data for those projects, before it fans out.

This follows the same model as Google Cloud's [observability scopes][obs-scopes]:
a user designates a scoping project, and queries fan out to all projects in that
scope with results merged at read time. Telemetry is stored against its origin
project only; the scope is resolved from the control plane at request time, not
baked into stored data. Cloud Monitoring calls this a [metrics scope][metrics-scope];
Cloud Logging has the equivalent [log scope][log-scope].

For Datum, `datumctl logs --project <scope>` designates the scoping project; the
query layer resolves the set of accessible projects from Milo and fans out to the
relevant regional ClickHouse instances.

**Project resolution from Milo.** Milo's `Project` resource carries
`spec.ownerRef.name` (the parent org), so the query layer can list all projects
for an org using the existing Milo API — no new scoping concept is needed.
For MVP single-project queries the project comes directly from the user's current
context and no Milo lookup is required. The open question is caching strategy:
whether the query layer calls Milo per-request (with short-TTL cache) or receives
project membership in OIDC token claims.

**Region resolution** follows from project resolution: once the scope is known,
the query layer asks Milo where those projects' workloads are deployed. The
region where workloads run is the region with their logs.

The interface inside the query layer should be a `ScopeResolver` abstraction so
the backing implementation (config file to start, Milo API long-term) can be
swapped without changing call sites.

For operator queries that span all projects, all configured regions are queried
regardless of scope.

[obs-scopes]: https://docs.cloud.google.com/stackdriver/docs/observability/scopes
[metrics-scope]: https://docs.cloud.google.com/monitoring/settings
[log-scope]: https://docs.cloud.google.com/logging/docs/log-scope/create-and-manage

## Alternatives

**ClickHouse `Distributed` table engine** — fans out within a ClickHouse
cluster. Rejected because intermediate query state can cross node boundaries,
violating data residency requirements. Also requires all ClickHouse nodes to
know about each other, which is operationally complex across geographic regions.

**Direct ClickHouse access from Grafana** — Grafana pointed directly at a
regional `grafana_ops` user. Rejected because it breaks multi-region operator
queries (Grafana would need one datasource per region and manual dashboard
federation), and it bypasses the single enforcement point for auth and tenant
scoping.

**NATS JetStream as query bus** — considered for durable fanout. Rejected at
this stage; adds operational complexity before the core pipeline is validated.
Revisit if the query layer needs to handle queuing or backpressure at scale.
