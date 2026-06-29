---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Metrics Pipeline

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Design Details](#design-details)
  - [Current pipeline: VictoriaMetrics](#current-pipeline-victoriametrics)
    - [Collection](#collection)
    - [ExportPolicy coupling](#exportpolicy-coupling)
    - [Open question: MetricPolicy output path](#open-question-metricpolicy-output-path)
  - [Future pipeline: NATS + ClickHouse](#future-pipeline-nats--clickhouse)
    - [Subject structure](#subject-structure)
    - [ClickHouse schema](#clickhouse-schema)
    - [Ingest path](#ingest-path)
- [Alternatives](#alternatives)

## Summary

The metrics pipeline collects, stores, and exposes metrics from Datum Cloud
platform resources. The current pipeline is VictoriaMetrics-based: VMAgent
scrapes Prometheus-compatible endpoints and metrics are queryable via MetricsQL.
The future pipeline extends the NATS ingest infrastructure to handle
`telemetry.metrics.<project_id>` subjects alongside logs, with ClickHouse as
the unified store.

This document covers the pipeline mechanics. For the declarative API that
defines what metrics resources publish and how they are produced, see
[definition-policy](../definition-policy/). For retention and rollup, see
[retention](../retention/).

## Motivation

The current scrape-based pipeline requires each resource to expose a
Prometheus-compatible `/metrics` endpoint and requires VMAgent to be configured
for each scrape target. This is operationally manageable for a small number of
platform components but does not scale to a large number of tenant resources.

Extending the NATS ingest pipeline to metrics unifies the write path: resources
emit OTel metrics over gRPC to the OTel Collector gateway, which bridges into
NATS. The same hub topology that provides log durability provides metric
durability. ClickHouse becomes the unified query backend for both signals.

### Goals

- Operational metrics available for ExportPolicy and platform dashboards
- Metrics ingest resilient to regional ClickHouse outages via NATS durability
- Per-project subject isolation for metrics, matching the logs subject structure
- Retention and rollup handled in ClickHouse; see [retention](../retention/)
- No cross-region metric replication — data residency first class

### Non-Goals

- Replacing VictoriaMetrics before ClickHouse metrics ingest is production-ready
- Real-time alerting — the pipeline enables it; alerting is a separate concern
- Defining what metrics resources publish — that is [definition-policy](../definition-policy/)

## Design Details

### Current pipeline: VictoriaMetrics

#### Collection

VMAgent is deployed per cluster and scrapes Prometheus-compatible `/metrics`
endpoints exposed by platform components. Metrics land in the VictoriaMetrics
cluster. ExportPolicy queries VictoriaMetrics via MetricsQL to pull metrics for
configured sinks.

```
Platform component (/metrics endpoint)
    │
    ▼ scrape (VMAgent)
VictoriaMetrics
    │
    ├──▶ Grafana (operator dashboards)
    └──▶ ExportPolicy export agent (MetricsQL query → tenant sink)
```

This pipeline is operational and handles all current platform metric needs. It
is not extended to tenant-scoped metric access — tenants access metrics via
ExportPolicy or not at all until the ClickHouse pipeline is available.

#### ExportPolicy coupling

> [!WARNING]
>
> **MetricsQL coupling:** ExportPolicy metric sources use `metricsql`
> queries evaluated against VictoriaMetrics. This ties ExportPolicy to
> VictoriaMetrics as the metrics backend. The `resourceSelectors` approach in
> the target ExportPolicy configuration is the intended long-term direction but
> is not yet implemented. See [export-policies](../export-policies/).

#### Open question: MetricPolicy output path

> [!WARNING]
>
> **Open question — MetricPolicy output path:** The MetricPolicy
> translation controller (see [definition-policy](../definition-policy/))
> produces metrics that need to reach VictoriaMetrics. The options are:
>
> - (a) The controller exposes a Prometheus-compatible `/metrics` scrape
>   endpoint and VMAgent picks it up
> - (b) The controller remote-writes directly to VMInsert
> - (c) VictoriaMetrics is configured with an OTLP receiver
>
> None of these is specified and the choice must be made before the translation
> controller is implemented.

---

### Future pipeline: NATS + ClickHouse

#### Subject structure

Metrics follow the same NATS subject convention as logs:

```
telemetry.metrics.<project_id>
```

The OTel Collector gateway validates and enriches resource attributes (requiring
`datum.project.id`). The OTLP-NATS bridge extracts `datum.project.id` and
publishes per-project NATS messages, with `ProjectId` as a top-level field for
ClickHouse NATS engine ingestion. Org ID is not stamped — the same rationale
applies as for logs: hierarchy is mutable and is resolved at read time from
Milo. See [logs](../logs/#bridge-processing) for the detailed rationale.

#### ClickHouse schema

Metrics land in per-type tables via the NATS engine, mirroring the OpenTelemetry
ClickHouse plugin schema. Key columns common to all metric tables:

| Column | Type | Notes |
|---|---|---|
| `ProjectId` | `LowCardinality(String)` | Extracted from `datum.project.id` by bridge |
| `TimeUnix` | `DateTime64(9)` | OTel `time_unix_nano` |
| `MetricName` | `LowCardinality(String)` | OTel metric name |
| `Attributes` | `JSON` | OTel metric attributes (labels) |
| `ResourceAttributes` | `JSON` | Resource-level attributes |

Partition key: `(ProjectId, toDate(TimeUnix))`
Order key: `(ProjectId, MetricName, toDate(TimeUnix), TimeUnix)`

Row policy: `ProjectId = getSetting('project_id')` on `api_reader`, matching
the logs schema. See [retention](../retention/) for TTL and rollup tables.

#### Ingest path

```
Platform component / workload
    │
    ▼ OTLP/gRPC
OTel Collector gateway (validates & enriches resource attributes)
    │
    ▼ OTLP/HTTP (gzip)
OTLP-NATS bridge (extracts datum.project.id → telemetry.metrics.<project_id>)
    │
    ▼ NATS JetStream (edge leaf)
    │
    ▼ NATS JetStream (hub — 48h buffer)
    │
    ├──► metrics_ingest (NATS engine, ClickHouse) → metrics_mv → otel_metrics_*
    └──► ExportPolicy consumer (future)
```

See [ingest-pipeline](../ingest-pipeline/) for the hub/edge topology
and durability design.

## Alternatives

**Keep VictoriaMetrics as the permanent metrics store** — VictoriaMetrics is
already deployed and handles the current load well. The primary reason to extend
to ClickHouse is unification: a single query backend for both logs and metrics,
project-scoped access via the same row policy mechanism, and the same NATS
ingest durability. If unification is not a priority, VictoriaMetrics remains
a valid long-term choice.

**ClickHouse `Distributed` table engine for cross-region fanout** — rejected for
the same reason as in [query-layer](../query-layer/): intermediate query state
can cross node boundaries, violating data residency. The query layer handles
fanout explicitly.
