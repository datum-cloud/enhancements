---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Telemetry Ingest Pipeline — NATS JetStream

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Design Details](#design-details)
  - [Topology](#topology)
  - [Edge NATS — leaf nodes](#edge-nats--leaf-nodes)
  - [Hub NATS — centralized fan-out](#hub-nats--centralized-fan-out)
  - [Subject structure](#subject-structure)
  - [JetStream stream design](#jetstream-stream-design)
  - [The OTLP-NATS bridge](#the-otlp-nats-bridge)
  - [ClickHouse consumer](#clickhouse-consumer)
  - [Customer export consumer](#customer-export-consumer)
  - [mTLS](#mtls)
  - [POC topology](#poc-topology)
- [Alternatives](#alternatives)

## Summary

NATS JetStream is inserted between the OTel Collector and ClickHouse as the
durable ingest hub. Telemetry from edge clusters flows to a local NATS leaf
node, which replicates to a centralized hub in GCP. The hub fans out to
multiple consumers: a ClickHouse writer, customer export pipelines, and future
real-time alerting — all reading from the same stream without coupling to each
other or to the storage write path.

## Motivation

The current write path is a direct OTel Collector → ClickHouse INSERT. This is
simple and works at low scale, but has two structural weaknesses:

**Durability.** The OTel Collector has an in-memory queue. If ClickHouse is
slow (compaction, schema migration, maintenance window) or the WAN link between
an edge cluster and GCP is interrupted, the queue fills and logs are dropped.
There is no replay.

**Fan-out.** Adding a second destination (customer export, alerting) requires
the Collector to write to multiple targets simultaneously. Each new consumer
adds direct coupling to the ingest path. A processing failure in one consumer
can stall the pipeline for others.

NATS JetStream addresses both: the stream is the durable record of what arrived,
and consumers are independently positioned within it. ClickHouse and customer
export each advance their own cursor; neither can block the other.

This also directly enables the customer export story. Datum's position is that
it is not an observability company — customers who want long-term retention,
custom dashboards, or integration with existing tooling should be able to export
their telemetry. NATS makes that a separate consumer, not a separate write path.

### Goals

- Durable ingest with replay: logs survive OTel Collector restarts and
  transient ClickHouse or WAN outages
- Fan-out to multiple consumers (ClickHouse, customer export, alerting) without
  coupling between them
- Per-project subject isolation: consumer ACLs enforce that a customer export
  consumer can only read that project's data
- No data loss during edge cluster WAN outages (store-and-forward at the edge)
- Leaf-to-hub replication over mTLS

### Non-Goals

- Real-time alerting implementation — the pipeline enables it; the alerting
  consumer is a separate concern
- Replacing the OTel Collector — it remains the collection and batching layer
- Long-term retention in NATS — JetStream is a buffer (hours to days), not the
  system of record; ClickHouse remains that
- Custom OTel Collector NATS exporter — a first-class `natsexporter` for
  otelcol-contrib does not yet exist and is tracked in
  [open-telemetry/opentelemetry-collector-contrib#39540](https://github.com/open-telemetry/opentelemetry-collector-contrib/issues/39540).
  Until it lands, a thin bridge service (see [below](#the-otlp-nats-bridge))
  fills the gap and is the intended replacement point.

## Design Details

### Topology

```
[Edge cluster — e.g. us-east-1]

  Envoy ──→ OTel Collector ──OTLP/HTTP──→ bridge ──publish──→ NATS leaf node
  Compute ↗          (in-cluster)                  (in-cluster)      │
                                                              local JetStream
                                                              (store-and-forward)
                                                                      │
                                                                  mTLS / WAN
                                                                      │
[Edge cluster — e.g. eu-west-1]                                       ↓
  Envoy ──→ OTel Collector ──────────────→ bridge ──────────→ NATS leaf node
  Compute ↗                                                            │
                                                                  mTLS / WAN
                                                                      │
[GCP]                                                                 ↓
                                                             NATS hub cluster
                                                                      │
                                              ┌───────────────────────┤
                                              ↓                       ↓
                                     ClickHouse consumer    Customer export consumer
                                              │                  (per tenant, scoped)
                                              ↓
                                         ClickHouse
```

### Edge NATS — leaf nodes

Running a NATS leaf node at each edge cluster provides local durability that
survives WAN outages. Without it, the OTel Collector's in-memory queue is the
only buffer between an edge cluster and GCP. When the WAN link drops — or when
ClickHouse on the hub side is unavailable — the queue fills within minutes and
logs are lost permanently.

NATS leaf nodes store messages on disk locally. When the WAN link is restored,
the leaf's JetStream stream forwards accumulated messages to the hub. No
operator intervention needed; no replay scripts; no manual reconciliation.

Leaf nodes are operationally simple in two important ways:

**They dial out.** Leaf nodes initiate an outbound TCP connection to the hub on
port 7422. Edge clusters need no inbound firewall rules and work behind NAT and
on cellular or variable-quality networks.

**They are independently operational.** During a hub disconnection, the edge
leaf continues to accept messages from local clients (OTel Collector, bridge).
Local JetStream streams work normally. The bridge and Collector are unaffected.
Only hub consumers (ClickHouse writer, customer export) stop advancing during
the outage — they resume from where they left off when the leaf reconnects.

```
                     ┌─ hub unreachable ─────────────────────────┐
edge OTel Collector  │  bridge ──→ leaf JetStream (accumulating) │
continues writing ──→│                                           │
                     │            leaf reconnects ───────────────┴──→ hub drains backlog
```

### Hub NATS — centralized fan-out

The hub cluster (GCP) is where all consumer logic lives. Its responsibilities:

- **Aggregate** streams from all regional leaf nodes into a single logical
  stream via JetStream source configuration
- **Fan out** to independently positioned consumers (ClickHouse writer,
  customer export, future alerting)
- **Enforce per-project ACLs** — customer export consumers are authorized to
  subscribe only to `telemetry.logs.<their_project_id>`

Separating the hub from the edge keeps consumer complexity out of edge clusters,
which are intentionally lightweight. Adding a new consumer (e.g., a streaming
alerts service) requires no changes at the edge.

### Subject structure

All log records are published to a per-project subject:

```
telemetry.logs.<project_id>
```

The bridge extracts `datum.project.id` from the OTLP resource attributes of
each `ResourceLogs` entry and routes to the corresponding subject. A single
OTLP batch may contain records from multiple projects; the bridge splits by
project before publishing. Org ID is not used for routing — it is materialized
at query time from Milo.

Future signal types follow the same pattern:

```
telemetry.metrics.<project_id>
telemetry.traces.<project_id>
```

### JetStream stream design

**Edge leaf — local buffer stream**

Each edge cluster has a local JetStream stream:

| Property | Value |
|---|---|
| Name | `TELEMETRY-EDGE` |
| Subjects | `telemetry.>` |
| Storage | File (persists across restarts) |
| Retention | Limits: 24h max age, 10 GiB max size |
| Purpose | Local durability during WAN outage |

**Hub — aggregate stream**

The hub stream sources from all edge leaf streams:

| Property | Value |
|---|---|
| Name | `TELEMETRY` |
| Subjects | `telemetry.>` |
| Storage | File |
| Retention | Limits: 48h max age, 100 GiB max size |
| Sources | One per edge cluster `TELEMETRY-EDGE` stream |
| Purpose | Durable buffer and fan-out point for consumers |

Both streams use `telemetry.>` to capture all signal types under the
`telemetry.*` hierarchy. Phase 1 publishes only `telemetry.logs.*`; Phase 2
adds `telemetry.metrics.*`; Phase 3 adds `telemetry.network.*`. The stream
config does not need to change as new signal types are introduced.

48 hours of retention at the hub gives time to recover from a ClickHouse outage
without data loss. It is not the system of record; ClickHouse is.

**Consumers (Phase 1 — logs)**

All consumers on the hub stream are durable, with explicit ack policy. A
consumer that fails to ack causes redelivery, not a pipeline stall for others.

| Consumer | Subject filter | Ack | Notes |
|---|---|---|---|
| `clickhouse-logs-writer` | `telemetry.logs.>` | After successful INSERT | Batches records for throughput |
| `export-<project_id>` | `telemetry.logs.<project_id>` | After confirmed delivery to sink | One per customer export destination |

**Consumers (Phase 2 — metrics)**

| Consumer | Subject filter | Ack | Notes |
|---|---|---|---|
| `clickhouse-metrics-writer` | `telemetry.metrics.>` | After successful INSERT | Batches data points for throughput |
| metrics export consumer | `telemetry.metrics.<project_id>` | TBD | Design TBD — ExportPolicy currently exports metrics via MetricsQL pull, not NATS push; Phase 2 may not need a per-tenant metrics export consumer |

**Consumers (Phase 3 — network)**

| Consumer | Subject filter | Ack | Notes |
|---|---|---|---|
| `clickhouse-network-writer` | `telemetry.network.>` | After successful INSERT | gNMIc event format; one consumer for all devices |

Each signal type gets its own ClickHouse writer consumer with its own cursor,
allowing independent replay and backpressure.

### The OTLP-NATS bridge

The bridge is a small service (single binary) that lives on each edge cluster
alongside the OTel Collector and NATS leaf node.

**What it does:**

1. Listens on OTLP/HTTP (`/v1/logs` and `/v1/metrics`)
2. Parses the OTLP protobuf payload
3. For each `ResourceLogs` entry, extracts `datum.project.id` from resource
   attributes
4. If `datum.project.id` is missing: increments
   `bridge_log_records_dropped_total{reason="missing_project_id"}` (logs) or
   `bridge_metric_datapoints_dropped_total{reason="missing_project_id"}` (metrics),
   excludes those records from the NATS publish, and reports them in the OTLP
   partial success response. The Collector receives HTTP 200 and does not retry
   — dropped records are gone.
5. Publishes one JSON message per `LogRecord` (as `JSONEachRow`) to
   `telemetry.logs.<project_id>` on the local NATS leaf, with `ProjectId` as a
   top-level field for ClickHouse NATS engine ingestion
6. Returns HTTP 200 with a partial success body if any records were dropped;
   HTTP 200 with an empty success body if all records were routed

The OTel Collector's `otlphttp` exporter points at the bridge endpoint. No
custom Collector build or feature gate is required.

The bridge is the intended replacement point for a future `natsexporter` in
otelcol-contrib. When that lands, the Collector config changes an endpoint and
the bridge is removed.

**What it does not do:**

- It does not buffer, aggregate, or transform records
- It does not validate project_id against a registry
- It does not authenticate the Collector (in-cluster — Collector identity is
  enforced by Kubernetes network policy)

### ClickHouse consumer

ClickHouse has a native NATS table engine (`ENGINE = NATS`) that subscribes to
NATS subjects and ingests messages directly — no separate consumer service
required. `telemetry.logs_ingest` uses this engine to subscribe to
`telemetry.logs.>` and read messages as `JSONEachRow`. A Materialized View
extracts `ProjectId` and routes rows into `telemetry.logs`.

Because the bridge publishes `ProjectId` as a top-level field in the JSON payload
(not inside `ResourceAttributes`), the MV reads it as a plain string column.
The three attribute columns (`ResourceAttributes`, `ScopeAttributes`,
`LogAttributes`) are stored as `String` in the ingest table — the NATS engine
does not support the `JSON` column type — and cast to `JSON` in the MV.

**JetStream durability caveat.** Binding the NATS engine to a JetStream durable
consumer requires the `nats_stream` and `nats_consumer_name` settings. Without
them the engine uses a core NATS subscription — messages published while
ClickHouse is down are not replayed. The POC runs ClickHouse 26.5.1 but these
settings were never enabled or tested.

> [!WARNING]
>
> **Must verify before staging.** The `nats_stream` and `nats_consumer_name`
> setting names and the minimum ClickHouse version that supports them must be
> confirmed against the ClickHouse NATS engine documentation before the staging
> deployment. See the [ClickHouse NATS engine reference](https://clickhouse.com/docs/en/engines/table-engines/integrations/nats).
>
> If these settings do not exist as described, the fallback is a separate
> consumer service (a small Go binary) that reads from the JetStream durable
> consumer and bulk-inserts into ClickHouse via the HTTP interface. This adds
> one component but restores the durability guarantee without changing the
> bridge, NATS topology, or ClickHouse schema. It should be treated as the
> backup plan, not a surprise.

### Customer export consumer

Each export destination is a separate durable consumer scoped to
`telemetry.logs.<project_id>`. The consumer reads batches and forwards to the
customer's sink (S3 bucket, Datadog ingest endpoint, etc.). Credentials for
external sinks are the consumer's concern, not the hub's.

NATS authorization ensures that a consumer authorized for
`telemetry.logs.acme-prod` cannot subscribe to `telemetry.logs.other-project`.
This is enforced at the NATS account layer, not in application code.

Customer export is a follow-on. The pipeline design here accommodates it without
requiring it at launch.

### mTLS

All inter-cluster NATS connections use mTLS. In-cluster connections (Collector
→ bridge, bridge → leaf, consumers → hub) use mTLS as defense-in-depth.
Certificates are managed by cert-manager.

| Hop | Auth | Notes |
|---|---|---|
| OTel Collector → bridge | None (in-cluster network policy) | Bridge is not externally reachable |
| Bridge → NATS leaf | mTLS | cert-manager issues leaf client cert |
| NATS leaf → NATS hub | **mTLS required** | Leaf `tls` block: `cert_file`, `key_file`, `ca_file`. Hub verifies leaf identity. |
| ClickHouse consumer → Hub | mTLS | Consumer dials hub; presents client cert |
| Export consumer → Hub | mTLS | Consumer dials hub; per-consumer cert scoped to tenant subject |

Leaf-to-hub mTLS is configured in the NATS leaf server config:

```conf
leafnodes {
  remotes [
    {
      url: "tls://nats-hub.telemetry.datum.net:7422"
      tls {
        cert_file: "/certs/leaf-client.crt"
        key_file:  "/certs/leaf-client.key"
        ca_file:   "/certs/ca.crt"
      }
    }
  ]
}
```

### POC topology

The POC ran in two stages. The initial setup ran all components co-located in a
single kind cluster (`telemetry-dev`); the diagram below reflects that layout. A
second iteration extended to a two-cluster setup (hub + edge kind clusters) to
exercise leaf-to-hub replication. NATS runs as a single node in JetStream hub
mode. ClickHouse runs in docker-compose on the host and is reached from the
cluster at `host.docker.internal:9000`.

```
[kind: telemetry-dev]

Envoy ──────────────────→ OTel Collector ──OTLP/HTTP (gzip)──→ bridge
telemetrygen (compute) ↗       (batch 10s)                        │
                                                    publish one JSON row per LogRecord
                                                    to telemetry.logs.<project_id>
                                                                   │
                                                                   ↓
                                                         NATS JetStream
                                                        (TELEMETRY stream)
                                                                   │
                                                                   ↓
                                                    logs_ingest (NATS engine, JSONEachRow)
                                                                   │
                                                                   ↓ Materialized View
                                                    ClickHouse (docker-compose, host)
                                                      telemetry.logs (MergeTree)
```

**Validated:**
- Per-project subject routing (both `gateway-tenant-001` and `compute-tenant-001` as project_id values in the POC)
- gzip body decoding in the bridge
- One JSON message per LogRecord with `ProjectId` as a top-level field
- ClickHouse NATS engine (`ENGINE = NATS`) consuming from `telemetry.logs.>` with `JSONEachRow`
- Materialized View casting `String`→`JSON` for attribute columns
- Row policy enforcement on a per-project reader user
- No separate consumer service — ClickHouse reads from NATS directly

**Known limitations in the POC (ClickHouse 26.5.1):**
- JetStream durable consumer binding (`nats_stream` / `nats_consumer_name`) was
  not enabled or tested. The engine uses a core NATS subscription; messages
  published while ClickHouse is unavailable are not replayed. These settings
  must be verified against the ClickHouse NATS engine docs before staging —
  see the warning in the [ClickHouse consumer](#clickhouse-consumer) section.
- `JSON` column type is not supported in NATS engine tables. Attribute columns
  are stored as `String` and cast to `JSON` in the MV.

**Not yet validated:** mTLS, leaf-to-hub WAN replication, file-backed JetStream
storage, customer export consumer, JetStream durable consumer binding. These are
deferred to an integration environment with a real leaf→hub topology.

## Production Readiness

The POC validates the core design. The following items are required before
the pipeline can carry production traffic.

### What remains

**Hub and edge staging/production overlays (#2 — NATS ingest pipeline staging and production deployment)**

The POC uses dev overlays with plaintext passwords and `NodePort` services.
Production overlays must:
- Reference ExternalSecrets for ClickHouse and NATS credentials
- Patch services to `LoadBalancer` type with stable external IPs
- Use the production GCS cold-storage bucket name in the CHI storage policy

**mTLS for leaf-to-hub connections (#3 — NATS mTLS cert-manager integration)**

All leaf-to-hub NATS connections must use mTLS. The leaf node config needs
cert-manager `Certificate` resources that issue:
- A CA cert for the hub cluster, trusted by all leaves
- A per-leaf client certificate for the hub to verify leaf identity

Hub NATS config must require TLS on the leafnode listener port (7422):

```conf
leafnodes {
  port: 7422
  tls {
    cert_file: "/certs/hub-server.crt"
    key_file:  "/certs/hub-server.key"
    ca_file:   "/certs/ca.crt"
    verify: true  # require and verify leaf client cert
  }
}
```

**JetStream durable consumers for ClickHouse**

The POC runs ClickHouse 26.5.1 but `nats_stream` and `nats_consumer_name` were
never enabled or tested. Verify these settings work against the ClickHouse NATS
engine docs before staging — see the WARNING in the
[ClickHouse consumer](#clickhouse-consumer) section. If they are confirmed,
enable them in the staging and production NATS engine table definition to bind
to a JetStream durable consumer and ensure messages published during a
ClickHouse outage are replayed on restart.

**Edge staging/production overlays (#2 — NATS ingest pipeline staging and production deployment)**

Edge overlays need the real hub NATS endpoint (not `host.docker.internal`) and
the production bridge image from the container registry.

**OTel Collector config changes (#6 — OTLP-NATS bridge implementation and deployment)**

The OTel Collector gateway config is part of the `milo-os/telemetry` codebase
— the edge-logs-system Collector in `datum-cloud/infra` is deprecated in favour
of what we're building. Required changes for the new Collector config:

- **Switch exporter**: replace the existing ClickHouse exporter with
  `otlphttp` pointing at the bridge (`http://otlp-nats-bridge.telemetry.svc:4318`)
- **Add resource processor**: configure the `k8sattributes` processor to
  derive `datum.project.id` from the namespace label
  `meta.datumapis.com/upstream-cluster-name` (strip `cluster-` prefix), and
  inject `datum.project.id = 'datum-internal'` for platform namespaces

Without these changes the bridge receives records with no `datum.project.id`
and drops everything.

**Query layer API user (#4 — query layer service initial implementation)**

Create the `api_reader` ClickHouse user with the tenant row policy. The current
schema has the user and policy defined as a placeholder in `schema.sql`; it
must be wired to the query layer service's connection credentials.

---

## Open Questions

### JetStream consumer fan-out at high project cardinality

The `TELEMETRY` hub stream captures `telemetry.logs.>` across all projects.
The ClickHouse writer is one consumer on `telemetry.logs.>`. ExportPolicy
export destinations are durable consumers filtered to
`telemetry.logs.<project_id>` — one per customer export destination.

JetStream filtered consumers are routed server-side by subject index: the
server does not scan the full stream for each consumer on every message. A
filtered consumer on `telemetry.logs.acme-prod` receives only messages
published to that subject; the routing is O(1) at the server. This is
materially different from a naive sequential scan and means per-message
overhead does not grow with consumer count.

The real cost is per-consumer state: each durable consumer holds a cursor,
an ack-pending set, and associated goroutines on the NATS server. At low
ExportPolicy counts (tens to hundreds) this is negligible. At thousands of
active ExportPolicy consumers on a single stream, the aggregate memory and
CPU overhead is unknown and must be benchmarked before assuming the current
design scales to that point.

Note that consumer count is bounded by ExportPolicy usage, not project count.
A project with no ExportPolicy configured creates no consumer. Early
deployments (Compute private alpha) are unlikely to stress this limit; it
becomes relevant as ExportPolicy adoption grows.

**Mitigations to evaluate if benchmarking shows a limit:**

- **Per-project streams**: create a dedicated JetStream stream per project
  (`TELEMETRY-<project_id>`) sourced from the hub. ExportPolicy consumers
  attach to the project stream directly with no filtering overhead. Trades
  consumer-count overhead for stream-count overhead; NATS supports large
  numbers of streams but adds management complexity.
- **Stream sharding**: shard the hub stream into N streams by
  `project_id` hash (e.g., 16 shards). Each shard has a proportionally
  smaller consumer set, capping per-shard consumer count without per-project
  streams.

**What needs to be validated before ExportPolicy scales:**

- Maximum durable consumer count on the hub stream at production message
  rates without degrading ClickHouse writer throughput or delivery latency
- Memory and CPU overhead per consumer at idle vs. active delivery
- Whether the NATS server's per-account consumer limit needs tuning

## Alternatives

**No NATS — OTel Collector writes directly to ClickHouse.** The current state.
Sufficient for early operation but provides no replay on ClickHouse outage, no
fan-out without Collector-level dual-write, and in-memory-only buffering against
WAN outages at the edge. Retained as the starting point; this enhancement is the
planned follow-on.

**Hub-side bridge only — no edge NATS.** Edge OTel Collectors export OTLP/HTTP
directly to a bridge in GCP. Simpler (one bridge, no per-edge NATS). Rejected
because the OTel Collector's in-memory queue is the only buffer during WAN
outage; a multi-minute connectivity interruption between an edge cluster and GCP
will drop logs. Viable if log loss during WAN interruptions is acceptable, and
revisitable if operating per-edge NATS proves too costly.

**NATS Leaf with no local JetStream — core NATS only.** Leaf nodes forward
messages to the hub in real time without local persistence. Simpler to configure
than leaf JetStream, but provides no durability during hub disconnection.
Messages published during a WAN outage are lost. Rejected for the same reason as
hub-side bridge only.

**Kafka.** Stronger ordering guarantees, larger ecosystem, more operational
tooling. Rejected at this stage: Kafka clusters are significantly heavier to
operate than NATS, particularly at the edge where a full Kafka broker is
impractical. NATS leaf nodes are designed for exactly this edge-to-hub topology.
Revisit if NATS proves insufficient at scale.

**NATS JetStream mirroring instead of sourcing.** A hub stream could mirror each
edge stream rather than source from it. Mirror is a 1:1 exact copy from a single
origin; source aggregates from multiple origins. Since the hub stream must
aggregate from N edge clusters, sourcing is the correct primitive.
