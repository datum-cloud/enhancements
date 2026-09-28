---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Observability Backend Migration

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Design Details](#design-details)
  - [What is being migrated](#what-is-being-migrated)
  - [Migration strategy](#migration-strategy)
  - [Historical data](#historical-data)
  - [ExportPolicy impact](#exportpolicy-impact)
  - [Grafana dashboards](#grafana-dashboards)
- [Dependencies](#dependencies)
- [Alternatives](#alternatives)

## Summary

Migrate operational logs and metrics from their current specialized backends
(Loki, VictoriaMetrics) into the unified NATS + ClickHouse pipeline built in
Phases 1–2. After migration, ClickHouse is the single query backend for both
signal types and the two specialized backends are decommissioned.

## Motivation

The current observability stack runs two signal-specific backends: Loki for
logs and VictoriaMetrics for metrics. Each was adopted independently and
optimized for its signal type. Running them together creates several problems:

**Operational burden.** Two databases to operate, monitor, and scale. Each has
its own retention config, backup strategy, access model, and failure modes.

**Weak tenancy.** Loki's tenancy model relies on `X-Scope-OrgID`, which is
caller-controlled and not enforced server-side. ClickHouse row policies enforce
`ProjectId = getSetting('project_id')` at the server — tenants cannot query
each other's data regardless of what the client sends.

**Query fragmentation.** Correlating logs and metrics during an incident
requires two tools and two query languages (LogQL, MetricsQL). A unified
ClickHouse backend makes correlated queries possible in SQL.

**ExportPolicy coupling.** ExportPolicy metric sources use MetricsQL queries
against VictoriaMetrics, tying the API to a VictoriaMetrics-specific query
language. Moving metrics to ClickHouse removes this dependency.

### Scenario: troubleshooting a spike in tenant error logs

An operator sees elevated error rates for `project-abc` in the Grafana
dashboard. Today, diagnosing the cause requires:

1. Open Grafana → VictoriaMetrics panel → run a MetricsQL query for error rate
   by service over the past hour.
2. Switch to Loki Explore → construct a LogQL query for `project_id=project-abc`
   with a `level=error` filter, matching the time window from step 1.
3. Notice the spike started at 14:23 and correlate it to a specific pod by
   copying timestamps by hand between two UIs.

After migration, the same investigation is a single query:

```sql
WITH errors AS (
    SELECT
        toStartOfMinute(ObservedTimestamp) AS minute,
        ResourceAttributes['service.name'] AS service,
        count() AS error_count
    FROM telemetry.logs
    WHERE ProjectId = 'project-abc'
      AND SeverityText IN ('ERROR', 'FATAL')
      AND ObservedTimestamp >= now() - INTERVAL 1 HOUR
    GROUP BY minute, service
),
cpu AS (
    SELECT
        toStartOfMinute(TimeUnix) AS minute,
        ResourceAttributes['service.name'] AS service,
        avg(Value) AS avg_cpu
    FROM telemetry.otel_metrics_gauge
    WHERE ProjectId = 'project-abc'
      AND MetricName = 'process.cpu.utilization'
      AND TimeUnix >= now() - INTERVAL 1 HOUR
    GROUP BY minute, service
)
SELECT e.minute, e.service, e.error_count, c.avg_cpu
FROM errors e
LEFT JOIN cpu c USING (minute, service)
ORDER BY e.minute;
```

One query, one tool, one place to look.

### Goals

- All operational logs flow through the NATS + ClickHouse pipeline; Loki
  decommissioned
- All metrics flow through the NATS + ClickHouse pipeline; VictoriaMetrics and
  VMAgent decommissioned
- ExportPolicy metric export migrated from MetricsQL pull to NATS consumer push
- Internal Grafana dashboards updated to query ClickHouse
- No gap in signal coverage during the transition

### Non-Goals

- Migrating historical data from Loki or VictoriaMetrics into ClickHouse —
  start fresh from cutover; existing backends remain in read-only mode for a
  short window
- Traces — Tempo remains in place; tracing migration is out of scope
- Migrating events — out of scope
- Migrating Pyroscope (profiles) or Sentry (errors) — separate concerns
- Changing signal schemas or metric definitions during migration — shape changes
  belong in Phase 5 (definition-policy CRDs)

## Design Details

### What is being migrated

| Signal | Current backend | After migration |
|---|---|---|
| Logs | Loki (Vector / Promtail ingest) | ClickHouse (NATS → ClickHouse pipeline) |
| Metrics | VictoriaMetrics (VMAgent scrape) | ClickHouse (NATS → ClickHouse pipeline) |

### Migration strategy

Each signal migrates independently through the same three stages:

**Stage 1 — Dual emit.** Each signal is sent to both the existing backend and
the new ClickHouse pipeline simultaneously. This validates that values agree and
that queries against ClickHouse produce equivalent results before committing to
cutover.

**Stage 2 — ClickHouse primary.** Grafana dashboards and ExportPolicy are
updated to read from ClickHouse. Existing backends remain running as fallback.

**Stage 3 — Decommission.** Existing backends are shut down. Ingestion paths
(Promtail, VMAgent, Tempo receiver) are removed or redirected.

Signals do not need to migrate simultaneously. Logs are likely first since the
NATS pipeline is built in Phase 1. Metrics follow after Phase 2.

### Historical data

Historical data in Loki and VictoriaMetrics is not backfilled into ClickHouse.
Each backend is retained in read-only mode for 30 days after cutover to allow
lookback queries before decommission.

### ExportPolicy impact

The current ExportPolicy metric source uses a `metricsql` field evaluated
against VictoriaMetrics. After migration, metric export uses a NATS durable
consumer on `telemetry.metrics.<project_id>` — the same push model used for
logs. The `metricsql` field becomes unsupported; existing ExportPolicy resources
must be migrated to the `resourceSelectors` source configuration.

A migration guide and a dual-active validation period will be provided before
the `metricsql` field is removed. See [export-policies](../export-policies/).

### Grafana dashboards

Internal operator dashboards currently query Loki (LogQL), VictoriaMetrics
(MetricsQL), and Tempo (TraceQL). Each dashboard must be updated to use the
ClickHouse data source with equivalent SQL queries before Stage 3 for its
respective signal type. Metric names, label schemas, and log field names are
unchanged; only the query language and data source change.

## Dependencies

- **Phase 1 logs pipeline** — NATS + ClickHouse logs ingest must be
  production-ready before log migration can begin. See [logs](../logs/) and
  [ingest-pipeline](../ingest-pipeline/).
- **Phase 2 metrics pipeline** — NATS + ClickHouse metrics ingest and retention
  rollup tables must be production-ready before metric migration can begin. See
  [metrics](../metrics/) and [retention](../retention/).
- **ExportPolicy NATS consumer** — the NATS-consumer export path must be
  implemented before ExportPolicy can be migrated off MetricsQL. See
  [export-policies](../export-policies/).

## Alternatives

**Keep specialized backends alongside ClickHouse indefinitely** — avoid
migration cost by running Loki and VictoriaMetrics permanently alongside
ClickHouse. Rejected: violates the "minimize database engines" principle,
perpetuates the weak tenancy model in Loki, and leaves the MetricsQL coupling
in ExportPolicy unresolved.

**Migrate metrics only; keep Loki** — limit scope to VictoriaMetrics. Rejected:
the tenancy argument applies equally to Loki, and leaving it in place keeps the
two-tool investigation workflow that this migration is meant to eliminate.

**Include traces (Tempo)** — extend scope to include tracing in this migration.
Not pursued now: the ClickHouse trace schema and ingest path do not exist yet,
and traces are not needed for the Compute private alpha deliverable. Deferred to
Phase 6 alongside the customer-facing traces work.

**Backfill historical data** — replay existing backends into ClickHouse before
decommission. Rejected for the initial migration: rollup tables only contain
meaningful long-term data after months of operation; a 30-day read-only window
covers short-term lookback needs. Revisit if business reporting requires it.
