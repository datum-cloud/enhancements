---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Customer-Facing Logs

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Design Details](#design-details)
  - [Resource attribute contract](#resource-attribute-contract)
  - [Ingest path](#ingest-path)
  - [Bridge processing](#bridge-processing)
  - [ClickHouse schema](#clickhouse-schema)
  - [Query access](#query-access)
  - [datumctl logs](#datumctl-logs)
- [Alternatives](#alternatives)

## Summary

Customers running workloads on Datum Cloud need visibility into the logs their
workloads produce — access logs from gateways, compute workload stderr, WAF
events. This enhancement covers the ingest pipeline and `datumctl logs` query
interface. For the declarative log API (LogDefinition, LogCollectionPolicy,
LogRedactionPolicy), see [definition-policy](../definition-policy/).

Logs flow through an OTel Collector → NATS → ClickHouse pipeline and are
queryable by operations staff via Grafana and by customers via `datumctl logs`.
No new CRDs are required.

## Motivation

A standard observability stack provides internal visibility for staff but not
for tenants. Customers who want to see logs from their gateway or compute
workloads cannot today — they must file a support ticket or deploy their own
collectors with `ExportPolicy`.

Providing tenant-scoped log access is necessary for the Compute private alpha deliverable
and is a baseline expectation for any infrastructure platform.

### Goals

- Customers can access structured logs from their workloads without setting up
  their own collectors
- Tenant isolation: a customer can only see their own logs, enforced server-side
- `datumctl logs` provides kubectl-style streaming log access
- Operations staff can query logs across all tenants via Grafana
- At-least-once delivery from edge cluster to ClickHouse (via NATS durability)
- The log pipeline is extensible to new signal types (metrics, traces)

### Non-Goals

- Full log query support — a useful subset is sufficient for the initial
  implementation; see [query-layer](../query-layer/)
- Long-term retention beyond platform-defined defaults — customers who need
  more should use ExportPolicy to forward to their own platform; see
  [retention](../retention/) for configured TTLs
- Attribute-level redaction — that is [definition-policy](../definition-policy/#redaction)
- Real-time alerting — the pipeline enables it; alerting is a separate concern

## Design Details

### Resource attribute contract

The OTel Collector resource processor on each cluster is responsible for
ensuring `datum.project.id` is present on every log record before it reaches
the bridge. There are two cases:

**Tenant workloads** do not need to set `datum.project.id` themselves. The
Collector derives it from the namespace label
`meta.datumapis.com/upstream-cluster-name` via the `k8sattributes` processor.
The label value has the form `cluster-<project-id>` (e.g.
`cluster-personal-project-2650fdb4`); the Collector strips the `cluster-`
prefix and sets `datum.project.id = 'personal-project-2650fdb4'`. Records
where this label is absent and `datum.project.id` is not otherwise set are
silently dropped by the bridge with a partial success response.

| Attribute | Description | Example |
|---|---|---|
| `service.name` | Service name from the OTel semantic conventions | `compute-workload` |
| `service.namespace` | Kubernetes namespace | `tenant-acme-corp` |
| `k8s.cluster.name` | Cluster where the workload runs | `us-east-1-edge-01` |
| `datum.project.id` | Derived from namespace label `meta.datumapis.com/upstream-cluster-name` by stripping `cluster-` prefix | `personal-project-2650fdb4` |

**Internal platform components** (Envoy, control plane services, NATS,
ClickHouse operator, etc.) do not set `datum.project.id` themselves. The
Collector resource processor detects these by Kubernetes namespace — any
source in a platform namespace has `datum.project.id = 'datum-internal'`
injected automatically. This routes their logs to
`telemetry.logs.datum-internal` on NATS and applies the 90-day retention tier
(vs. 7 days for tenant data). See [retention](../retention/#log-ttl-tiers).

The complete namespace list is an implementation detail maintained in the OTel
Collector resource processor config. Known platform namespaces include
`kube-system`, `telemetry`, `milo-system`, and `gateway-system`; the full list
must be audited against the actual cluster namespaces before the Collector
config is finalized.

The bridge service extracts `datum.project.id` from the OTLP resource attributes
of each `ResourceLogs` entry and routes to the corresponding NATS subject
(`telemetry.logs.<project_id>`). This attribute is the root tenancy signal for
the entire pipeline.

In Phases 1–4, the bridge collects all records with a valid `datum.project.id`
unconditionally. The `LogCollectionPolicy` opt-in gate is a Phase 5 feature;
no policy is required for Phase 1 log collection to work. See
[definition-policy](../definition-policy/) for the Phase 5 design.

### Ingest path

```
Compute workload / Envoy
    │
    ▼ OTLP/gRPC
OTel Collector gateway (validates & enriches resource attributes)
    │
    ▼ OTLP/HTTP (gzip)
OTLP-NATS bridge (extracts datum.project.id → publishes to telemetry.logs.<project_id>)
    │
    ▼ NATS JetStream (edge leaf — local durability during WAN outage)
    │
    ▼ NATS JetStream (hub — 48h buffer, fan-out to consumers)
    │
    ├──► logs_ingest (NATS engine, ClickHouse) → logs_mv → logs (MergeTree)
    └──► customer export consumer (export-policies)
```

See [ingest-pipeline](../ingest-pipeline/) for full topology and
component-level design.

### Bridge processing

The OTLP-NATS bridge splits incoming OTLP payloads into per-project NATS
messages. It routes; it does not enrich.

For each `ResourceLogs` entry:

1. Extract `datum.project.id` from resource attributes
2. If missing: increment `bridge_log_records_dropped_total{reason="missing_project_id"}`,
   exclude from the NATS publish, and include in the OTLP partial success
   response. The Collector receives HTTP 200 and does not retry.
3. Publish one JSON message per `LogRecord` (as `JSONEachRow`) to
   `telemetry.logs.<project_id>` on the local NATS leaf, with `ProjectId` as a
   top-level field for ClickHouse NATS engine ingestion

Org ID is not stamped here. This is a deliberate design choice: org and folder
hierarchy is mutable. Baking it into stored records means historical data
misattributes the moment a project is reorganized. Storing only `project_id`
(the immutable origin) means historical data stays correct automatically when
ownership changes — no backfill, no stale attribution. This matches how Google
Cloud Logging and Cloud Monitoring work: telemetry is stored against its project
of origin; org/folder relationships are resolved from the control plane at read
time ([observability scopes][obs-scopes]).

> [!NOTE]
>
> **Org resolved at read time from Milo.** The query layer looks up which
> projects a requesting user can access from the Milo control plane and issues
> scoped ClickHouse queries (`ProjectId IN (...)`). The specific Milo API and
> caching strategy are TBD; see [query-layer](../query-layer/) for the scope
> resolution design.
>
> **Exception — ExportPolicy:** Hierarchy is used on the outbound export path
> to determine routing — an org-scoped ExportPolicy can route logs from all
> descendant projects to a single sink. The exported records themselves are not
> annotated with org metadata. See [export-policies](../export-policies/).

[obs-scopes]: https://docs.cloud.google.com/stackdriver/docs/observability/scopes

### ClickHouse schema

Logs land in `telemetry.logs` via a NATS engine ingest table and materialized
view. The schema stores the full OTLP log record with tenant isolation enforced
at the row policy level.

Key columns:

| Column | Type | Notes |
|---|---|---|
| `ProjectId` | `LowCardinality(String)` | Extracted from `datum.project.id` by bridge |
| `Timestamp` | `DateTime64(9)` | OTLP `time_unix_nano`; may be 0 if the source did not set one |
| `ObservedTimestamp` | `DateTime64(9)` | OTLP `observed_time_unix_nano`; set by the OTel Collector on receipt; always non-zero |
| `ServiceName` | `LowCardinality(String)` | `service.name` |
| `SeverityText` | `LowCardinality(String)` | `INFO`, `WARN`, `ERROR`, etc. |
| `Body` | `String` | Log message body |
| `ResourceAttributes` | `JSON` | Full resource attributes |
| `ScopeAttributes` | `JSON` | Instrumentation scope attributes |
| `LogAttributes` | `JSON` | Per-record attributes |

Partition key: `(ProjectId, toDate(ObservedTimestamp))` — provisional; see
[retention open questions](../retention/#partition-key-design-at-high-tenant-count)
for the partition explosion concern at high tenant count.
Order key: `(ProjectId, ServiceName, toDateTime(ObservedTimestamp), ObservedTimestamp)`

Row policy: `ProjectId = getSetting('project_id')` on the `api_reader` user.
The query layer sets this before every tenant query. For TTL configuration and
cold storage tiers, see [retention](../retention/#log-retention).

### Query access

The query layer service exposes a single HTTP endpoint that resolves tenant
identity from the OIDC bearer token, sets the ClickHouse session variable, and
executes the query. See [query-layer](../query-layer/) for the full design.

**Operator access (Grafana):** The query layer connects as `grafana_ops` with no
row policy. Only platform operators can reach this path.

**Tenant access (`datumctl`):** The query layer connects as `api_reader`, sets
`SET project_id = '<resolved_project>'`, and the row policy enforces isolation.

### datumctl logs

`datumctl logs` is a kubectl-style log access command implemented against the
query layer.

**Usage:**

```
datumctl logs [flags]

Flags:
  --tail         Stream logs in real time (live tail)
  --since        Show logs since a relative time (e.g. 1h, 30m) or absolute timestamp; filters on ObservedTimestamp (ingest time)
  --service      Filter by service.name
  --severity     Filter by severity (INFO, WARN, ERROR)
  --project      Target project (defaults to current context)
  --limit        Maximum number of log lines to return (default 100, max 1000)
```

> [!NOTE]
>
> **`--since` filters on ingest time, not event time.** `ObservedTimestamp` is
> set by the OTel Collector when it receives the record, not when the event
> occurred. This is intentional — `Timestamp` (event time) may be 0 or
> unreliable for sources that don't set it. Under normal conditions ingest and
> event time are within seconds of each other and the distinction is invisible.
>
> The gap becomes noticeable during a NATS replay after a WAN outage. If a
> site is offline for 2 hours and the NATS leaf replays buffered records to
> the hub on reconnect, `--since 1h` will surface those records — ClickHouse
> received them in the last hour even though the events happened 2+ hours ago.
> Users investigating what happened *during* the outage should use an absolute
> `--since` timestamp rather than a relative window.

**Output format:**

```
2026-07-01T12:00:00.123Z  INFO  compute-workload  request completed in 45ms
2026-07-01T12:00:00.456Z  WARN  compute-workload  connection retry 1/3
2026-07-01T12:00:01.789Z  ERROR envoy-gateway     upstream timeout: cluster backend-svc
```

Streaming (`--tail`) uses chunked HTTP with cursor-based polling. The query
layer holds the connection open and issues a new ClickHouse query on a short
interval (e.g. 500ms), filtering for rows with
`ObservedTimestamp >= <last_seen_cursor>`. New rows are flushed to the client
as they arrive. The cursor advances to the maximum `ObservedTimestamp` in each
batch.

The delivery guarantee is **at-least-once**. `ObservedTimestamp` is set by the
OTel Collector on receipt and is not unique — multiple records in the same
ingest batch can share an identical nanosecond timestamp. A strict `>` cursor
would silently drop those records (gap); `>=` re-delivers the records at the
cursor boundary on the next poll or reconnect (duplicate). At-least-once with
occasional duplicates at boundaries is the correct tradeoff for a log tail —
the same contract `kubectl logs` provides.

If exact-once delivery is required in a future iteration, the bridge can
generate a UUID per log record (`LogId`) stored as a top-level ClickHouse
column, and the cursor becomes `(ObservedTimestamp, LogId)` — a unique,
monotonically-advanceable position.

ClickHouse `LIVE VIEW` is not used — it is experimental and deprecated, and
does not compose with the row policy session variable the query layer sets per
request.

## Alternatives

**Direct OTel Collector → ClickHouse (no NATS)** — the current deployed state
for edge logs (`edge-logs-system`). Simple but provides no replay on ClickHouse
outage and no fan-out to customer export without dual-write. The edge-logs-system
pipeline is deprecated; it will be decommissioned as part of Phase 4 once the
NATS pipeline has fully replaced it.

**Loki as the log store** — Loki is already deployed. Rejected for
customer-facing logs: Loki's multi-tenancy model uses `X-Scope-OrgID` which
puts tenancy in the caller's control. ClickHouse with row policies enforces
isolation server-side at the session level. ClickHouse also gives us the NATS
engine ingest path (no separate writer service) and SQL query flexibility.
