---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Telemetry Retention

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Design Details](#design-details)
  - [Log retention](#log-retention)
    - [Log TTL tiers](#log-ttl-tiers)
    - [Log tiered storage](#log-tiered-storage)
  - [Metrics retention](#metrics-retention)
    - [Metrics retention tiers](#metrics-retention-tiers)
    - [Rollup tables](#rollup-tables)
    - [Aggregation semantics by metric type](#aggregation-semantics-by-metric-type)
    - [Metrics TTL configuration](#metrics-ttl-configuration)
    - [Storage cost profile](#storage-cost-profile)
- [Dependencies](#dependencies)
- [Alternatives](#alternatives)

## Summary

Logs and metrics have different retention requirements. Logs are operationally
useful at full resolution for a short window (7 days for tenant data) and need
cold storage for a longer audit trail. Metrics need full resolution for 90 days,
hourly rollups for a year, and indefinite daily rollups for business analysis.

This enhancement defines the retention policy for both signal types, the
ClickHouse TTL expressions that implement it, and the tiered storage
(hot PD-SSD → cold GCS) that makes indefinite retention economically viable.

## Motivation

Datum is not an observability company. Tenant-facing logs and metrics have short
hot retention; tenants who need more export to their own platform via
`ExportPolicy`. Internal operational data needs enough history for trend analysis
and incident post-mortems. Business-correlated metrics — compute hours billed,
active project counts, API call volumes — must survive indefinitely for financial
reporting and long-term analysis.

Keeping everything at raw resolution forever is not viable. Downsampling metrics
to daily granularity makes indefinite retention economically trivial. For logs,
cold GCS storage at $0.02/GB makes a long audit window cheap after the hot
window expires.

### Goals

- 7-day hot retention for tenant log data; 90 days for `datum-internal` logs
- 90-day retention for raw metrics at full resolution
- 1-year retention for hourly metric rollups
- Indefinite retention for daily metric rollups, for all metrics
- No manual tagging of "business critical" metrics — keep everything at daily
  rollup to avoid regretting it later
- Cold storage to GCS for both logs and daily metric rollups after their hot
  window expires

### Non-Goals

- Sub-minute metric rollup granularities (that is what raw metrics are for)
- Replacing a data warehouse for financial reporting — daily rollups in
  ClickHouse are the source; export to BigQuery is a separate concern
- User-configurable per-project retention — customers who need longer retention
  use `ExportPolicy` to forward to their own platform

## Design Details

### Log retention

#### Log TTL tiers

| Category | Hot retention | Purpose |
|---|---|---|
| Tenant data (`allLogs`) | 7 days | Operational debugging, `datumctl logs` |
| `datum-internal` | 90 days | Incident post-mortems, audit trail |

After the hot window expires, log records are deleted. Tenants who need a longer
audit trail should configure an `ExportPolicy` to forward logs to their own
storage.

#### Log tiered storage

Tenant logs live on PD-SSD for their full 7-day retention window. Since tenant
data is deleted at day 7, there is no cost benefit to an intermediate cold tier
— the storage is reclaimed at the same point it would have been moved, so the
only effect of early cold migration would be degraded `datumctl logs` query
latency for lookbacks beyond 1 day.

`datum-internal` logs move to GCS cold storage at day 7 (the end of the hot
window) and are deleted at day 90. Days 1–7 are always on PD-SSD; the long
audit tail lives cheaply on GCS.

```sql
TTL
    -- datum-internal: cold at day 7; tenant: never moved to cold (~100 years)
    toDateTime(ObservedTimestamp) + toIntervalDay(if(ProjectId = 'datum-internal', 7, 36500)) TO VOLUME 'cold',
    -- datum-internal: deleted at day 90; tenant: deleted at day 7
    toDateTime(ObservedTimestamp) + toIntervalDay(if(ProjectId = 'datum-internal', 90, 7)) DELETE
```

For tenant data, the `TO VOLUME` deadline is set to ~100 years — effectively
never — so only the `DELETE` at day 7 fires. For `datum-internal`, the cold
move fires at day 7 and the `DELETE` fires at day 90. The two deadlines never
coincide for either class of data, so there is no ambiguity about TTL rule
ordering.

The row policy on `api_reader` (`ProjectId = getSetting('project_id')`) means
the TTL expression branching on `ProjectId` only needs to distinguish internal
from tenant data — individual project retention is not configurable at the TTL
layer.

`datum-internal` is injected by the OTel Collector resource processor for
platform components running in platform Kubernetes namespaces. See
[logs — resource attribute contract](../logs/#resource-attribute-contract)
for how internal logs acquire this value.

---

### Metrics retention

#### Metrics retention tiers

| Tier | Granularity | TTL | Purpose |
|---|---|---|---|
| Raw | Full resolution (seconds) | 90 days | Operational debugging, alerting |
| Hourly rollup | 1 row per metric per label set per hour | 1 year | Trend analysis, capacity planning |
| Daily rollup | 1 row per metric per label set per day | None | Business metrics, long-term analysis |

All three tiers currently use `(ProjectId, Date)` as the partition key,
matching the logs schema. This is provisional — see
[Partition key design at high tenant count](#partition-key-design-at-high-tenant-count)
for why this needs to change before production.

#### Rollup tables

Each raw metrics table (one per OTel metric type) has two corresponding rollup
tables populated by materialized views:

```
metrics_ingest  (NATS engine — receives all metric types)
    │
    ├──▶ MV (MetricType='gauge') → otel_metrics_gauge       (raw, 90-day DELETE TTL)
    ├──▶ MV (MetricType='gauge') → otel_metrics_gauge_hourly (AggregatingMergeTree, 1-year DELETE TTL)
    ├──▶ MV (MetricType='gauge') → otel_metrics_gauge_daily  (AggregatingMergeTree, cold-move at day 7, no delete)
    └──▶ ... (same pattern for sum, histogram)
```

All MVs read directly from `metrics_ingest`. ClickHouse only fires MVs on
direct INSERTs — rows inserted by one MV do not trigger downstream MVs. Rollup
MVs cannot read from `otel_metrics_gauge` because that table receives rows via
a MV, not a direct INSERT; the rollup MVs would never fire.

Materialized views truncate the timestamp to the rollup bucket boundary
(`toStartOfHour`, `toStartOfDay`) and use `AggregateFunction` state columns so
that ClickHouse's background merge process correctly combines partial aggregates
written across multiple insert batches.

Example for the gauge type:

Some attributes are consistent across all signals — the OTel resource attributes
emitted by the `k8sattributes` processor for every workload. These are promoted
to typed columns in the ORDER BY so AggregatingMergeTree uses them as merge
dimensions. Because `metrics_ingest` stores `ResourceAttributes` as a `String`
(the NATS engine cannot hold `JSON`), the MV extracts each promoted attribute
with `JSONExtractString`.

Freeform metric-specific attributes (e.g. `http.status_code`, `cpu.state`) vary
per metric and cannot be enumerated upfront. They are **not carried into the
rollup**: rows sharing the promoted dimensions collapse into one rollup row,
which is correct for a daily aggregate. (A freeform attribute cannot be both
absent from the GROUP BY and preserved per-row — and `JSON` is not a groupable
type, so it cannot be a merge dimension either.) For per-label breakdowns on
freeform attributes, query the raw `otel_metrics_*` tables directly, within
their 90-day window.

```sql
-- Daily rollup
CREATE TABLE otel_metrics_gauge_daily (
    ProjectId       LowCardinality(String),
    Date            Date,
    MetricName      LowCardinality(String),
    -- Promoted from resource attributes; consistent across all signals
    ServiceName     LowCardinality(String),   -- service.name
    ClusterName     LowCardinality(String),   -- k8s.cluster.name
    NodeName        LowCardinality(String),   -- k8s.node.name
    PodName         LowCardinality(String),   -- k8s.pod.name
    ValueAvg        AggregateFunction(avg, Float64),
    ValueMin        AggregateFunction(min, Float64),
    ValueMax        AggregateFunction(max, Float64),
    SampleCount     AggregateFunction(count)
) ENGINE = AggregatingMergeTree
PARTITION BY toYear(Date)
ORDER BY (ProjectId, MetricName, Date, ServiceName, ClusterName, NodeName, PodName)
TTL Date + INTERVAL 7 DAY TO VOLUME 'cold'   -- move to cold at day 7; retained indefinitely
SETTINGS storage_policy = 'hot_cold';

CREATE MATERIALIZED VIEW otel_metrics_gauge_daily_mv
TO otel_metrics_gauge_daily AS
SELECT
    ProjectId,
    toDate(TimeUnix)                                          AS Date,
    MetricName,
    JSONExtractString(ResourceAttributes, 'service.name')     AS ServiceName,
    JSONExtractString(ResourceAttributes, 'k8s.cluster.name') AS ClusterName,
    JSONExtractString(ResourceAttributes, 'k8s.node.name')    AS NodeName,
    JSONExtractString(ResourceAttributes, 'k8s.pod.name')     AS PodName,
    avgState(Value)                            AS ValueAvg,
    minState(Value)                            AS ValueMin,
    maxState(Value)                            AS ValueMax,
    countState()                               AS SampleCount
FROM metrics_ingest
WHERE MetricType = 'gauge'
GROUP BY ProjectId, Date, MetricName,
    ServiceName, ClusterName, NodeName, PodName;

-- Hourly rollup — same shape, hour bucket, 1-year DELETE TTL (no cold move)
CREATE TABLE otel_metrics_gauge_hourly (
    ProjectId       LowCardinality(String),
    Hour            DateTime,
    MetricName      LowCardinality(String),
    ServiceName     LowCardinality(String),
    ClusterName     LowCardinality(String),
    NodeName        LowCardinality(String),
    PodName         LowCardinality(String),
    ValueAvg        AggregateFunction(avg, Float64),
    ValueMin        AggregateFunction(min, Float64),
    ValueMax        AggregateFunction(max, Float64),
    SampleCount     AggregateFunction(count)
) ENGINE = AggregatingMergeTree
PARTITION BY toYYYYMM(Hour)
ORDER BY (ProjectId, MetricName, Hour, ServiceName, ClusterName, NodeName, PodName)
TTL Hour + INTERVAL 365 DAY DELETE;

CREATE MATERIALIZED VIEW otel_metrics_gauge_hourly_mv
TO otel_metrics_gauge_hourly AS
SELECT
    ProjectId,
    toStartOfHour(TimeUnix)                                   AS Hour,
    MetricName,
    JSONExtractString(ResourceAttributes, 'service.name')     AS ServiceName,
    JSONExtractString(ResourceAttributes, 'k8s.cluster.name') AS ClusterName,
    JSONExtractString(ResourceAttributes, 'k8s.node.name')    AS NodeName,
    JSONExtractString(ResourceAttributes, 'k8s.pod.name')     AS PodName,
    avgState(Value)                            AS ValueAvg,
    minState(Value)                            AS ValueMin,
    maxState(Value)                            AS ValueMax,
    countState()                               AS SampleCount
FROM metrics_ingest
WHERE MetricType = 'gauge'
GROUP BY ProjectId, Hour, MetricName,
    ServiceName, ClusterName, NodeName, PodName;
```

Both rollup MVs read `metrics_ingest` directly with `WHERE MetricType = 'gauge'`
— the hourly MV does **not** read the daily table, nor vice versa, since
ClickHouse would not fire a MV off another MV's inserts. `SampleCount` is
`AggregateFunction(count)` (zero-argument `countState()`), not
`AggregateFunction(count, UInt64)`, which would be a type mismatch.

The exact set of promoted columns is provisional — the final list depends on
which resource attributes the `k8sattributes` processor guarantees on every
record. Metrics that require per-label rollup granularity on a freeform
attribute (e.g. a dedicated HTTP status code rollup) should promote that
attribute to a typed column in a separate rollup table.

Query rollup tables using the `Merge` combinator:

```sql
SELECT
    MetricName,
    avgMerge(ValueAvg) AS avg_value,
    minMerge(ValueMin) AS min_value,
    maxMerge(ValueMax) AS max_value
FROM otel_metrics_gauge_daily
WHERE ProjectId = 'datum-internal'
  AND Date >= '2026-01-01'
GROUP BY MetricName
```

#### Aggregation semantics by metric type

Different OTel metric types require different rollup aggregations. Using the
wrong function produces incorrect results (e.g. averaging a counter sum). The
sum and summary rollups follow the same table+MV structure as the gauge example
above (read `metrics_ingest` directly, filter `MetricType`, promote the same
resource attributes), differing only in the aggregate state columns per the
table below. Histogram and exponential-histogram rollup schemas are the hard
cases and are deferred to implementation (see below).

| Metric type | Correct rollup aggregation |
|---|---|
| Gauge | avg, min, max, count of samples |
| Sum (monotonic / counter) | max of the cumulative value per bucket; or sum of delta values if reset-aware |
| Sum (non-monotonic) | avg, min, max |
| Histogram | sum of counts per bucket boundary; bucket boundaries must be identical across rolled-up records |
| Exponential histogram | no native ClickHouse merge for the OTel sketch; exact re-aggregation is not supported (see below) |
| Summary | sum of count and sum fields; quantile fields cannot be re-aggregated and are dropped |

Histograms and exponential histograms at rollup granularity are the hardest
cases and require careful schema design before implementation:

- **Standard histograms** can be rolled up by summing per-bucket counts, but
  only when bucket boundaries are identical across all records being merged. OTel
  recommends fixed boundaries per metric, which makes this feasible. Percentile
  estimation from rolled-up histograms uses linear interpolation *within a
  bucket*, which is a valid approximation — but this requires the bucket
  boundaries and per-bucket counts, not just a total count and sum.
- **Exponential histograms** use a logarithmic scale that supports exact
  merge in theory, but ClickHouse has no native aggregate that merges the OTel
  exponential-histogram sketch. Any approximation (e.g. feeding raw values
  through a t-digest aggregate) operates on raw values rather than the sketch
  and loses the exponential-histogram structure. The approach is deferred to
  implementation — no concrete mechanism is committed here.

The rollup schema for these types (ClickHouse column layout, MV definition) is
not yet designed and is deferred to implementation. If histogram rollup proves
impractical at scale, retaining raw histograms for the full 90-day window and
omitting them from the hourly/daily rollup tables is a valid fallback — raw
data covers the operational window where percentile queries are most needed.

#### Metrics TTL configuration

The hourly and daily rollup TTLs are defined on their table DDLs above (hourly:
`Hour + INTERVAL 365 DAY DELETE`; daily: `Date + INTERVAL 7 DAY TO VOLUME 'cold'`
with `storage_policy = 'hot_cold'`). The raw per-type tables use a single
delete-only TTL:

```sql
-- Raw tables (otel_metrics_<type>): delete at 90 days, all projects, no cold move
TTL toDateTime(TimeUnix) + INTERVAL 90 DAY DELETE
```

The raw table TTL is 90 days for all projects — unlike logs, there is no reason
to keep raw operational metrics longer, since the rollup tables preserve the
long-term signal. Raw metrics stay on hot storage for the full 90 days; cold-
tiering raw data within that window is a possible cost optimization but is not
adopted here (it would require `storage_policy = 'hot_cold'` on the raw tables).

#### Storage cost profile

At daily rollup granularity, even a large deployment produces a small number of
rows per day. A metric with 1,000 unique label combinations generates 1,000 rows
per day in the daily rollup table. At 1,000 metrics and 1,000 label sets each,
that is 1M rows per day — roughly 50–100 MB uncompressed, an order of magnitude
less with ClickHouse's ZSTD compression. At GCS cold storage pricing (~$0.02/GB),
a decade of daily rollups costs tens of dollars per year. "Forever" is
economically trivial at this granularity.

## Open Questions

### Cross-tenant query performance and projections

The current sort key (`ProjectId, ...`) is optimal for per-tenant queries —
`datumctl logs`, the query layer's row policy enforcement, and per-project
dashboards all benefit from partition and index pruning. However, operator and
platform queries that scan across tenants — platform-wide error rate dashboards,
billing aggregations, incident response across an organization — will hit full
table scans because they cannot prune on `ProjectId`.

ClickHouse [projections](https://clickhouse.com/docs/en/sql-reference/statements/alter/projection)
address this by storing a copy of the data sorted by an alternate key (e.g.
`(MetricName, Date, ProjectId)`) within the same table. Queries that match the
projection's key are automatically rewritten to use it; per-tenant queries
continue to use the primary sort key. The tradeoff is storage: each projection
roughly doubles the on-disk footprint of the table it covers.

This is validated in the audit log pipeline and is the likely solution here, but
the right projection keys and the actual storage multiplier need to be benchmarked
against realistic query patterns and data volumes before the schema is finalized
for production.

**What needs to happen before schema is finalized:**
- Identify the cross-tenant query patterns that matter (platform dashboards,
  org-level aggregations, billing)
- Benchmark without projections against a realistic data volume to quantify the
  performance gap
- Design projections for the query patterns that require them
- Measure the storage overhead and validate it is acceptable under the tiered
  storage cost model above

### Partition key design at high tenant count

The current partition key for all tables is `(ProjectId, Date)`. At low project
counts this is fine, but at high tenant cardinality it creates a partition
explosion: N projects × D days = N×D partitions per table. With 1,000 projects
and 90 days of raw log retention that is 90,000 partitions on the logs table
alone. ClickHouse merge performance degrades well before that point — the common
failure mode is `DB::Exception: Too many parts`.

The sort key `(ProjectId, ...)` already provides per-project scan efficiency
without any help from the partition key. The partition key's job is data
lifecycle management: TTL expression evaluation, cold storage moves, and
controlling part merge granularity. A date-only partition key handles all of
these without the cardinality explosion.

**Recommended direction:**

| Table | Current | Recommended |
|---|---|---|
| Raw logs / metrics (90-day TTL) | `(ProjectId, toDate(...))` | `toYYYYMM(...)` — monthly; max ~3 partitions in the hot window |
| Hourly rollup (1-year TTL) | `(ProjectId, Date)` | `toYYYYMM(...)` — monthly; max ~12 partitions |
| Daily rollup (no TTL, indefinite) | `(ProjectId, Date)` | `toYear(...)` — annual; accumulates slowly |

Monthly partitioning on raw tables gives at most 3 active partitions within the
90-day window, which is well within ClickHouse's comfortable range. The sort key
handles per-project pruning; partitions handle TTL and tiered storage moves.

The existing `(ProjectId, Date)` partition key in the schema examples and POC is
provisional. It must be revisited during staging benchmarking before any
production schema is committed — altering the partition key on a table with data
requires a full table rebuild.

**What needs to happen before schema is finalized:**
- Validate that monthly partitioning correctly scopes TTL operations (the TTL
  expression operates on rows, not partitions — date-only partitioning is safe)
- Confirm cold storage `TO VOLUME` moves still work as expected with monthly
  partitions
- Benchmark merge performance at target data volume with both partition schemes

### ClickHouse vertical and horizontal scaling

The current design assumes a single ClickHouse instance per region. What
constitutes an appropriately-sized instance — and how to grow beyond it — has
not been investigated. Before the telemetry pipeline carries production load,
there are two open scaling questions:

**Vertical scaling (single-node capacity ceiling).** How large can a single
ClickHouse node get before query latency or merge throughput degrades? What are
the early warning signs? The operations runbook currently handles disk pressure
but does not address CPU or memory saturation, or the point at which a
single-node deployment should move to a sharded cluster.

**Horizontal scaling (sharding).** The ClickHouse operator supports sharded
deployments, but the schema (NATS engine ingest tables, materialized views,
MergeTree tables) has not been designed for sharding. Specifically:
- The NATS engine table has no native sharding — a sharded ingest path would
  require distributing NATS subjects across shards or switching to a Go consumer
  service that can route by project.
- The row policy and `getSetting('project_id')` session variable work on the
  querying node; with a `Distributed` table engine the policy must be enforced
  on every shard, which adds schema complexity.
- Cross-shard query merging in the query layer is additive (see
  [query-layer — result merging](../query-layer/#result-merging)); this extends
  naturally to shards within a region.

**What needs to happen before production:**
- Establish target data volume and ingest rate (messages/sec, GB/day) from
  staging
- Define the vertical ceiling (disk, CPU, memory) and document the
  expansion procedure (PVC resize, node resize, replica count)
- Decide whether the initial production deployment is single-node or a
  small replica set, and document the migration path to sharding if needed

### Single-region deployment

The current design deploys one regional ClickHouse instance per region where
workloads run, with no replication across regions. This is acceptable for the
initial deployment but is not a long-term architecture:

- **No regional redundancy.** A single regional ClickHouse going down takes
  the entire log and metric pipeline for that region offline. The NATS hub
  buffers up to 48 hours, but beyond that window data is lost.
- **No cross-region failover.** If a region is unavailable, tenant queries
  for projects in that region return nothing — there is no replica to fall
  back to.
- **Data residency limits options.** Replicating across regions is not
  straightforward: the data residency requirement (data does not leave its
  region of origin) means cross-region replication must be opt-in and
  carefully scoped, if it is permitted at all.

For the Compute private alpha, a single regional ClickHouse is acceptable.
Longer term, the deployment needs at least a replica set within each region
for availability, and a clear policy on what cross-region durability means
given data residency constraints.

**What needs to happen before GA:**
- Define the within-region availability target and the ClickHouse replica
  count needed to meet it
- Understand what GCP storage options (regional PD, balanced PD vs. SSD) are
  acceptable under data residency requirements
- Decide whether cross-region failover is in scope and, if so, how it interacts
  with the data residency model

## Dependencies

- **Metrics pipeline** — the raw `otel_metrics_*` tables must use the NATS
  engine → MergeTree pattern with `ProjectId` as a first-class column before
  rollup tables can be built on top. See [metrics](../metrics/).
- **Tiered storage** — the GCS cold volume must be configured before daily
  rollup tables are created, as they require
  `SETTINGS storage_policy = 'hot_cold'`.

## Alternatives

**Keep raw metrics forever** — viable in principle but storage costs scale with
resolution and label cardinality. A busy deployment generating millions of
samples per minute would accumulate terabytes per year at raw resolution.
Rejected in favor of rollups.

**Tag specific metrics as "business critical"** — only retain tagged metrics
indefinitely, discard everything else at daily granularity. Rejected because the
cost of keeping everything at daily rollup is negligible, and the cost of
incorrectly tagging something as non-critical and losing years of data is high.
Keep everything; decide what to query later.

**Export to a data warehouse (BigQuery) for long-term storage** — a valid
complement for analyst-facing queries. ClickHouse daily rollups are the source
of truth; periodic export to BigQuery makes the data available to SQL-fluent
analysts without burdening the operational ClickHouse cluster. Not a
replacement — the rollup tables must exist either way.

**Longer log hot retention for tenants** — 30 or 90 days on PD-SSD. Rejected
for cost reasons; the per-GB cost of SSD-backed ClickHouse storage is orders of
magnitude higher than GCS. Tenants who need longer hot access should use
`ExportPolicy` to maintain their own store.
