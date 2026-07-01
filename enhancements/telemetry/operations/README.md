---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Telemetry Platform Operations

Alerts and runbook stubs for the telemetry pipeline. Each entry describes what
to alert on, what it means, and the initial investigation steps.

- [Alerts](#alerts)
  - [Ingest pipeline](#ingest-pipeline)
  - [ClickHouse storage](#clickhouse-storage)
- [Staging validation checklist](#staging-validation-checklist)
- [Out of scope](#out-of-scope)

## Alerts

### Ingest pipeline

---

#### `TelemetryBridgeDropRateHigh`

**Metrics:** `bridge_log_records_dropped_total{reason="missing_project_id"}`,
`bridge_metric_datapoints_dropped_total{reason="missing_project_id"}`

**Alert condition:** Drop rate > N records/min sustained for 5 minutes on either
counter (threshold TBD at implementation; baseline from staging).

**What it means:** The OTLP-NATS bridge is receiving log records or metric
data points without a `milo.project.id` resource attribute and silently
dropping them. The OTel Collector is not retrying (the bridge returns HTTP 200
with a partial success body), so these records are gone. A sustained elevated
rate indicates a misconfigured or newly onboarded service that is not emitting
the required resource attribute.

**Impact:** Tenant or platform telemetry is silently lost. The affected
workload's logs will not appear in `datumctl logs` or Grafana; dropped metric
data points will create gaps in dashboards and rollup tables.

**Runbook:**

1. Identify the source service by querying the bridge's drop metric broken down
   by any available label (e.g. Collector scrape target, edge cluster). If the
   bridge exposes a `service_name` label on the counter, filter on it.
2. Check the OTel Collector config for the affected edge cluster — confirm the
   resource processor is configured to inject `milo.project.id`. If it is
   missing or the attribute key is misspelled, the bridge will drop every record
   from that Collector.
3. If the Collector config looks correct, check what the workload is emitting —
   it may be overriding the resource attribute with an empty value, which the
   bridge treats as missing.
4. After fixing the Collector config or workload, confirm the drop counter stops
   increasing. Lost records are not recoverable.

**See also:** [ingest-pipeline — OTLP-NATS bridge](../ingest-pipeline/#the-otlp-nats-bridge),
[logs — resource attribute contract](../logs/#resource-attribute-contract)

---

### ClickHouse storage

---

#### `ClickHouseHotTierLow`

**Metric:** `clickhouse_disks_free_space_bytes{disk="hot"}` (exposed by the
ClickHouse Prometheus exporter from `system.disks`)

**Alert condition:** Free space on the hot (PD-SSD) volume < 20% of total
capacity sustained for 15 minutes. Page at < 10%.

**What it means:** The ClickHouse hot tier is filling up. At 0% free space
ClickHouse stops accepting inserts, which stalls the NATS ingest consumer and
causes the hub stream to back up. The 48-hour hub buffer buys time but is not
infinite.

**Impact:** If unaddressed, ClickHouse insert failures back up into NATS
JetStream. The hub has a 100 GiB size limit and a 48-hour age limit; once
either is exceeded, the oldest messages are dropped and log data is permanently
lost.

**Runbook:**

1. **Check what's consuming space.** In ClickHouse:
   ```sql
   SELECT database, table,
          formatReadableSize(sum(bytes_on_disk)) AS size
   FROM system.parts
   WHERE disk_name = 'hot' AND active
   GROUP BY database, table
   ORDER BY sum(bytes_on_disk) DESC
   LIMIT 20;
   ```
   Identify whether growth is uniform across projects or concentrated in one
   table or project. Unexpected concentration may indicate a misconfigured
   workload emitting at very high volume.

2. **Force a TTL merge** to evict any data past its retention window that
   hasn't been merged yet. ClickHouse applies TTL lazily; expired data can
   sit on disk until the next background merge.
   ```sql
   OPTIMIZE TABLE telemetry.logs FINAL;
   ```
   Check free space again after the merge completes. If this recovers
   significant space, consider tuning `merge_with_ttl_timeout` to run TTL
   merges more aggressively.

3. **Resize the PVC** if the hot tier is genuinely undersized. GCP PD-SSD
   supports online volume expansion — resize the PVC in the
   `clickhouse-operator` manifest and Kubernetes will expand the underlying
   disk without downtime. Do not set the new size below current usage.
   ```yaml
   # In the ClickHouseInstallation manifest:
   volumeClaimTemplates:
     - name: data
       spec:
         resources:
           requests:
             storage: <new-size>Gi
   ```

4. **Add a ClickHouse node** if a single node's disk ceiling has been reached
   and PVC resize is not sufficient. This involves updating the
   `clickhouse-operator` replica count and rebalancing shards — coordinate
   with the infra team before proceeding.

5. **Do not lower the cold storage threshold for logs** as a short-term
   fix. The current design keeps tenant logs on PD-SSD for their full 7-day
   life specifically to avoid cold-query latency degrading `datumctl logs`.
   If disk pressure is sustained, scale the disk rather than compromise query
   latency.

**See also:** [retention — log tiered storage](../retention/#log-tiered-storage),
[retention — storage cost profile](../retention/#storage-cost-profile)

---

## Staging validation checklist

These are one-time checks that must pass before promoting to production. They
verify properties the design depends on but that are not inherently enforced by
the platform.

---

#### Row policy throws on unset `project_id`

**What to verify:** Connect to ClickHouse as `api_reader` without issuing
`SET project_id = '...'`. Run any query against `telemetry.logs` or
`telemetry.otel_metrics_gauge`. Confirm the query throws rather than returning
rows or an empty result set.

**Why it matters:** `getSetting('project_id')` only throws if the custom
setting has no configured default. If a default exists, the row policy silently
matches against it — potentially leaking all rows (if default is `''` and
`ProjectId = ''` matches nothing, it's a silent empty result; if the default
is a real project_id, it leaks that project's data to every session). An empty
result is indistinguishable from "no logs" and masks the misconfiguration.

**How to fix if it fails:** Remove the default value from the `project_id`
custom setting in the ClickHouse server config, or add a `CONSTRAINT` that
rejects empty values. Re-run the test.

**See also:** [query-layer — tenant enforcement](../query-layer/#tenant-enforcement)

---

## Out of scope

- Tenant-facing alerting on their own log/metric pipelines — that is a future
  ExportPolicy concern
- Infrastructure-level alerts (NATS cluster health, ClickHouse replication lag,
  GCS cold storage availability) — those belong in the infra repo runbooks
