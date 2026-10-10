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
- [Alternatives](#alternatives)

## Summary

NATS JetStream is inserted between the OTel Collector and ClickHouse as the
durable ingest hub. Telemetry from edge clusters flows to a local NATS leaf
node, which forwards to a centralized JetStream hub. Edge clusters without
persistent storage run a core NATS leaf with a bounded in-memory buffer; edge
clusters with persistent storage can run JetStream locally for durable
store-and-forward. In both cases, the hub fans out to multiple consumers: a
ClickHouse writer, customer export pipelines, and future real-time alerting —
all reading from the same stream without coupling to each other or to the
storage write path.

## Motivation

A simple write path is a direct OTel Collector → ClickHouse INSERT. This works
at low scale, but has two structural weaknesses:

**Durability.** The OTel Collector has an in-memory queue. If ClickHouse is
slow (compaction, schema migration, maintenance window) or the WAN link between
an edge cluster and the hub is interrupted, the queue fills and logs are dropped.
There is no replay.

**Fan-out.** Adding a second destination (customer export, alerting) requires
the Collector to write to multiple targets simultaneously. Each new consumer
adds direct coupling to the ingest path. A processing failure in one consumer
can stall the pipeline for others.

NATS JetStream addresses both **at the hub**: the stream is the durable record
of what arrived, and consumers are independently positioned within it. ClickHouse
and customer export each advance their own cursor; neither can block the other.
The hub fully covers the ClickHouse-outage and consumer-stall cases. The WAN-outage
case is only partially covered for storage-constrained edges: a prolonged WAN
outage is bounded by the in-memory buffer (see [Edge NATS](#edge-nats--leaf-nodes)).
Edges with persistent storage running JetStream locally are fully covered.

This also directly enables the customer export story. A service provider is
not an observability company — customers who want long-term retention, custom
dashboards, or integration with existing tooling should be able to export their
telemetry. NATS makes that a separate consumer, not a separate write path.

### Goals

- Durable ingest with replay **from the hub onward**: once data reaches the
  hub, JetStream provides a configurable durable buffer that survives transient
  ClickHouse outages and consumer restarts
- Fan-out to multiple consumers (ClickHouse, customer export, alerting) without
  coupling between them
- Per-project subject isolation: consumer ACLs enforce that a customer export
  consumer can only read that project's data
- Tolerate transient WAN outages: edge clusters with persistent storage use a
  local JetStream stream for durable store-and-forward; edge clusters without
  storage use a bounded in-memory buffer — see [Edge NATS](#edge-nats--leaf-nodes)
- Leaf-to-hub forwarding over mTLS

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

![Production topology](./diagrams/topology.png)

### Edge NATS — leaf nodes

Each edge cluster runs a NATS leaf node that dials the hub and forwards locally
published telemetry to it. The leaf earns its place regardless of whether
local storage exists:

**Retry and buffering live in a battle-tested component.** This is the decisive
reason to keep the leaf rather than have the bridge publish to the hub directly.
Reconnection, the in-flight buffer, and forwarding are handled by NATS — a mature,
widely-operated component — instead of being reimplemented in the thin,
purpose-built bridge. The bridge stays simple: publish to localhost and move on.

**It dials out.** Leaf nodes initiate an outbound TCP connection to the hub on
port 443. Edge clusters need no inbound firewall rules and work behind NAT and
on cellular or variable-quality networks.

**It decouples the bridge from the WAN.** The bridge publishes to the local leaf
and is never blocked on hub round-trips; the leaf owns reconnection and
forwarding to the hub.

**Storage determines durability mode:**

- **With persistent storage — JetStream at the edge.** The leaf runs a
  file-backed JetStream stream. Messages published during a WAN outage are
  durably stored on local disk and replayed on reconnect, providing
  store-and-forward across both hub outages and leaf restarts. This is the
  preferred mode when disk is available.

- **Without persistent storage — core NATS, bounded in-memory buffer.** The
  leaf is routing-only. When the leaf-to-hub link drops, messages accumulate in
  a bounded in-memory buffer (RAM on the edge node). Once the buffer is full,
  messages must be dropped — nothing survives a leaf restart. The buffer sizing
  (memory budget, NATS pending limits) is TBD and must be set against measured
  edge throughput. This is a deliberate trade against the no-storage constraint,
  not a durable store-and-forward guarantee.

> [!WARNING]
>
> **Open question — buffer priority (no-storage case).** When the in-memory
> buffer fills during a hub outage, not all telemetry is equally valuable:
> billing-relevant data matters more than operational logs and metrics. A
> prioritization scheme is needed so that lower-value operational telemetry is
> dropped (or sampled) before billing-relevant data, rather than dropping
> indiscriminately. Whether core NATS can express this (e.g. via separate
> subjects/connections with different buffer budgets) or whether it needs to be
> enforced upstream in the bridge is TBD.

![Edge buffering during hub outage](./diagrams/edge-accumulation.png)

### Hub NATS — centralized fan-out

The hub cluster is where all consumer logic lives. Its responsibilities:

- **Aggregate** telemetry forwarded from all regional leaf nodes into a single
  hub JetStream stream (the leaves forward over core NATS; the hub stream
  captures by subject)
- **Fan out** to independently positioned consumers (ClickHouse writer,
  customer export, future alerting)
- **Enforce per-project ACLs** — customer export consumers are authorized to
  subscribe only to the project-scoped subject for each signal type,
  `telemetry.<signal>.<their_project_id>` (e.g. `telemetry.logs.acme-prod`,
  `telemetry.metrics.acme-prod`)

Separating the hub from the edge keeps consumer complexity out of edge clusters,
which are intentionally lightweight. Adding a new consumer (e.g., a streaming
alerts service) requires no changes at the edge.

### Subject structure

All log records are published to a per-project subject:

```
telemetry.logs.<project_id>
```

The bridge extracts `milo.project.id` from the OTLP resource attributes of
each `ResourceLogs` entry and routes to the corresponding subject. A single
OTLP batch may contain records from multiple projects; the bridge splits by
project before publishing. Org ID is not used for routing — it is materialized
at query time from Milo.

Future signal types follow the same pattern:

```
telemetry.metrics.<project_id>     (Phase 2)
telemetry.network.<device_id>      (Phase 3)
```

### JetStream stream design

JetStream runs at the hub and, where persistent storage is available, optionally
at the edge (see [Edge NATS](#edge-nats--leaf-nodes)).

**Hub — aggregate stream**

| Property | Value |
|---|---|
| Name | `TELEMETRY` |
| Subjects | `telemetry.>` |
| Storage | File |
| Retention | Limits: max age and max size — tune to measured throughput and recovery time requirements (e.g. 48h / 100 GiB) |
| Sources | Telemetry forwarded from each edge leaf |
| Purpose | Durable buffer and fan-out point for consumers |

The stream uses `telemetry.>` to capture all signal types under the
`telemetry.*` hierarchy. Phase 1 publishes only `telemetry.logs.*`; Phase 2
adds `telemetry.metrics.*`; Phase 3 adds `telemetry.network.*`. The stream
config does not need to change as new signal types are introduced.

The hub retention window should be sized to exceed the expected ClickHouse
recovery time for the deployment. It is not the system of record; ClickHouse is.

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

1. Listens on OTLP/HTTP. Phase 1 handles `/v1/logs`; `/v1/metrics` is added in
   Phase 2 when the metrics pipeline lands.
2. Parses the OTLP protobuf payload
3. For each `ResourceLogs` (Phase 1) or `ResourceMetrics` (Phase 2) entry,
   extracts `milo.project.id` from the resource attributes. The project applies
   to every child record under that resource.
4. If `milo.project.id` is missing: increments
   `bridge_log_records_dropped_total{reason="missing_project_id"}` (logs) or
   `bridge_metric_datapoints_dropped_total{reason="missing_project_id"}` (metrics, Phase 2),
   counting each child record under the resource, excludes those records from the
   NATS publish, and reports them in the OTLP partial success response. The
   Collector receives HTTP 200 and does not retry — dropped records are gone.
5. Publishes (as `JSONEachRow`) to the local NATS leaf, with `ProjectId` as a
   top-level field for ClickHouse NATS engine ingestion:
   - Phase 1: one JSON message per `LogRecord` to `telemetry.logs.<project_id>`
   - Phase 2: one JSON message per metric data point to
     `telemetry.metrics.<project_id>`
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
ClickHouse is down are not replayed.

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
      url: "tls://nats-hub.telemetry.example.com:443"
      tls {
        cert_file: "/certs/leaf-client.crt"
        key_file:  "/certs/leaf-client.key"
        ca_file:   "/certs/ca.crt"
      }
    }
  ]
}
```

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

**Edge JetStream — file-backed local stream at the edge.** Run a file-backed
JetStream stream on each edge leaf so that a prolonged WAN outage is buffered
durably on local disk and replayed on reconnect. This is the preferred edge
mode **when persistent storage is available**: it provides full store-and-forward
durability across hub outages and leaf restarts. Where storage is not available,
JetStream's file store is not an option and its memory store offers no durability
advantage over plain core NATS — see below.

**Core NATS leaf, no edge JetStream — the default for storage-constrained edges.**
The edge leaf forwards to the hub over core NATS with a bounded in-memory buffer.
It provides no durability across a buffer overflow or a leaf restart. Chosen for
storage-constrained deployments because the leaf still keeps reconnection and
in-flight buffering inside a mature, widely-operated component (NATS) rather than
in the thin, from-scratch bridge, and gives a single point at which to enforce
[buffer prioritization](#edge-nats--leaf-nodes). Durability begins at the hub.

**Bridge publishes to the hub directly — no edge NATS.** The bridge becomes a
JetStream client and publishes straight to the hub stream over the WAN (with
PubAck), dropping the edge leaf entirely. Simpler topology (one fewer edge
component) and preserves the end-to-end durability signal. Rejected for now
because it pushes reconnection, in-flight buffering, and prioritized-drop logic
into the bridge — a thin, purpose-built service — rather than relying on NATS for
it. Revisit if per-edge NATS proves too costly relative to that benefit, or if
the bridge needs acknowledged publishing for other reasons.

**Kafka.** Stronger ordering guarantees, larger ecosystem, more operational
tooling. Rejected at this stage: Kafka clusters are significantly heavier to
operate than NATS, particularly at the edge where a full Kafka broker is
impractical. NATS leaf nodes are designed for exactly this edge-to-hub topology.
Revisit if NATS proves insufficient at scale.
