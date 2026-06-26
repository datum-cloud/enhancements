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

Hot data lives on PD-SSD for low-latency `datumctl logs` queries. Cold storage
moves to GCS after 1 day — the vast majority of log queries are for recent data,
so moving older records to cold storage has minimal impact on query latency.

```sql
TTL
    toDateTime(ObservedTimestamp) + INTERVAL 1 DAY TO VOLUME 'cold',
    toDateTime(ObservedTimestamp) + INTERVAL if(ProjectId = 'datum-internal', 90, 7) DAY DELETE
```

The row policy on `api_reader` (`ProjectId = getSetting('project_id')`) means
the TTL expression branching on `ProjectId` only needs to distinguish internal
from tenant data — individual project retention is not configurable at the TTL
layer.

---

### Metrics retention

#### Metrics retention tiers

| Tier | Granularity | TTL | Purpose |
|---|---|---|---|
| Raw | Full resolution (seconds) | 90 days | Operational debugging, alerting |
| Hourly rollup | 1 row per metric per label set per hour | 1 year | Trend analysis, capacity planning |
| Daily rollup | 1 row per metric per label set per day | None | Business metrics, long-term analysis |

All three tiers are partitioned by `(ProjectId, Date)` so project isolation and
partition pruning work identically to the logs schema.

#### Rollup tables

Each raw metrics table (one per OTel metric type) has two corresponding rollup
tables populated by materialized views:

```
otel_metrics_<type>          (raw, MergeTree, 90-day TTL)
    │
    ├──▶ otel_metrics_<type>_hourly   (AggregatingMergeTree, 1-year TTL)
    └──▶ otel_metrics_<type>_daily    (AggregatingMergeTree, no TTL)
```

Materialized views truncate the timestamp to the rollup bucket boundary
(`toStartOfHour`, `toStartOfDay`) and use `AggregateFunction` state columns so
that ClickHouse's background merge process correctly combines partial aggregates
written across multiple insert batches.

Example for the gauge type:

```sql
CREATE TABLE otel_metrics_gauge_daily (
    ProjectId       LowCardinality(String),
    Date            Date,
    MetricName      LowCardinality(String),
    Attributes      JSON,
    ValueAvg        AggregateFunction(avg, Float64),
    ValueMin        AggregateFunction(min, Float64),
    ValueMax        AggregateFunction(max, Float64),
    SampleCount     AggregateFunction(count, UInt64)
) ENGINE = AggregatingMergeTree
PARTITION BY (ProjectId, Date)
ORDER BY (ProjectId, MetricName, Date, Attributes);

CREATE MATERIALIZED VIEW otel_metrics_gauge_daily_mv
TO otel_metrics_gauge_daily AS
SELECT
    ProjectId,
    toDate(TimeUnix)         AS Date,
    MetricName,
    Attributes,
    avgState(Value)          AS ValueAvg,
    minState(Value)          AS ValueMin,
    maxState(Value)          AS ValueMax,
    countState()             AS SampleCount
FROM otel_metrics_gauge
GROUP BY ProjectId, Date, MetricName, Attributes;
```

Query rollup tables using the `Merge` combinator:

```sql
SELECT
    MetricName,
    avgMerge(ValueAvg) AS avg_value,
    minMerge(ValueMin) AS min_value,
    maxMerge(ValueMax) AS max_value
FROM otel_metrics_gauge_daily
WHERE ProjectId = 'datum-internal'
  AND Date >= '2024-01-01'
GROUP BY MetricName
```

#### Aggregation semantics by metric type

Different OTel metric types require different rollup aggregations. Using the
wrong function produces incorrect results (e.g. averaging a counter sum).

| Metric type | Correct rollup aggregation |
|---|---|
| Gauge | avg, min, max, count of samples |
| Sum (monotonic / counter) | max of the cumulative value per bucket; or sum of delta values if reset-aware |
| Sum (non-monotonic) | avg, min, max |
| Histogram | sum of counts per bucket; preserve bucket boundaries |
| Exponential histogram | merge HDR sketches if possible; otherwise sum counts |
| Summary | sum of count and sum fields; quantiles cannot be re-aggregated and are dropped |

Histograms and exponential histograms are the most complex. If exact quantile
re-aggregation is not required at rollup granularity, store only count and sum —
sufficient for p50/p95 approximation via linear interpolation and for SLA
reporting.

#### Metrics TTL configuration

```sql
-- Raw table: 90 days regardless of project
TTL toDateTime(TimeUnix) + INTERVAL 90 DAY

-- Tiered storage: move to cold after 1 day, delete at 90
TTL toDateTime(TimeUnix) + INTERVAL 1 DAY TO VOLUME 'cold',
    toDateTime(TimeUnix) + INTERVAL 90 DAY DELETE

-- Hourly rollup: 1 year (Hour = toStartOfHour(TimeUnix), type DateTime64)
TTL toDateTime(Hour) + INTERVAL 365 DAY DELETE

-- Daily rollup: no deletion TTL; move to cold after 7 days and retain indefinitely on GCS
-- Requires: SETTINGS storage_policy = 'hot_cold' on the daily rollup table
TTL toDateTime(Date) + INTERVAL 7 DAY TO VOLUME 'cold'
```

The raw table TTL is 90 days for all projects — unlike logs, there is no reason
to keep raw operational metrics longer since the rollup tables preserve the
long-term signal.

#### Storage cost profile

At daily rollup granularity, even a large deployment produces a small number of
rows per day. A metric with 1,000 unique label combinations generates 1,000 rows
per day in the daily rollup table. At 1,000 metrics and 1,000 label sets each,
that is 1M rows per day — roughly 50–100 MB uncompressed, an order of magnitude
less with ClickHouse's ZSTD compression. At GCS cold storage pricing (~$0.02/GB),
a decade of daily rollups costs tens of dollars per year. "Forever" is
economically trivial at this granularity.

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
