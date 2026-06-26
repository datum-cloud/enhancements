---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Telemetry System

The telemetry system collects, processes, stores, and exposes telemetry data
(metrics, logs, traces, and flows) from all services and infrastructure on the
Datum Cloud platform. It serves both internal operational visibility and
tenant-scoped visibility for customers running workloads on Datum infrastructure.

Tracking issue: [datum-cloud/enhancements#765](https://github.com/datum-cloud/enhancements/issues/765)

## Guiding Principles

- OpenTelemetry by default
- Minimize database engines
- Avoid storing the same data more than once
- Avoid running in public clouds; avoid managed services when in public clouds
- Multi-tenancy first class support
- Data residency first class support

## Status at a Glance (as of June 2026)

| Component | Enhancement | Status | Phase |
|---|---|---|---|
| Telemetry export (ExportPolicy) | [export-policies](./export-policies/) | **Shipped — v0.3.0** | — |
| NATS ingest pipeline | [nats-ingest-pipeline](./nats-ingest-pipeline/) | **POC complete** | 1, 2 |
| Log pipeline | [logs](./logs/) | Designing | 1 |
| Query layer | [query-layer](./query-layer/) | Designed | 1 |
| OTel resource attribute standards | — | Not started | 1 |
| Compute operations dashboards | — | Not started | 1 |
| Network telemetry (gNMIc / SNMP / flows) | [network-telemetry](./network-telemetry/) | Not started | 2 |
| Log and metric definition/policy CRDs | [definition-policy](./definition-policy/) | Designed | 1, 3 |
| Metrics pipeline | [metrics](./metrics/) | Designing | 3 |
| Retention and rollup | [retention](./retention/) | Designed | 3 |

## Roadmap

### Phase 1: Customer-Facing Logs (target: July 2026)

Expose structured, tenant-scoped log access to customers via `datumctl logs`.
Builds on the Phase 1 query layer and ingest pipeline. Implements the log CRD
layer for declaration, collection policy, and redaction.

**Deliverables and sub-issues:**

1. **Log CRDs — LogDefinition, LogCollectionPolicy, LogRedactionPolicy**
   Implement the API types for log declaration (what log types a resource
   publishes), collection opt-in, and attribute-level redaction rules. See
   [definition-policy](./definition-policy/).

2. **Platform logs ClickHouse — deployment and schema**
   Deploy the `platform_logs` ClickHouse database and schema. We may repurpose
   the existing `edge-logs-system` ClickHouse storage resources rather than
   provisioning a new instance. Schema sorted by
   `(project_id, resource_type, resource_name, log_id, timestamp)`. NATS engine
   ingest via the existing hub pipeline. See [logs](./logs/).

3. **Log collection controller — LogCollectionPolicy reconciler**
   Implement the controller that reconciles `LogCollectionPolicy` resources
   against `LogDefinition` types and configures OTel Collector routing for
   opted-in log categories. See [definition-policy](./definition-policy/).

4. **LogSource support in ExportPolicy**
   Extend `ExportPolicy` to accept log sources alongside metric sources,
   enabling customers to export their logs to third-party platforms (Datadog,
   Grafana Cloud, etc.) over OTLP or a future log-specific sink. See
   [export-policies](./export-policies/).

---

### Phase 1: Compute Observability (target: July 2026)

Required for the Compute private alpha deliverable (#682). Builds on the validated POC in
`datum-cloud/infra/feat/telemetry-system`. The NATS → ClickHouse write path and
per-tenant subject routing are validated; what remains is promoting the POC to
production and building the read path for customers and operators.

**Deliverables and sub-issues:**

5. **NATS ingest pipeline — staging and production deployment**
   Promote the hub and edge Kustomize overlays from `dev` to `staging` and
   `production`. Wire ExternalSecrets for credentials, configure cert-manager
   for mTLS on the leaf-to-hub connection, and set up GCS cold storage. See
   [nats-ingest-pipeline](./nats-ingest-pipeline/) for what remains.

6. **Query layer service — initial implementation (single region, logs)**
   Deploy the query layer as a Go HTTP service against the production ClickHouse.
   MVP scope: single-region, tenant-scoped log queries resolved via OIDC bearer
   token, streaming chunked responses for `--tail` mode. See [query-layer](./query-layer/).

7. **`datumctl logs` — initial implementation**
   Implement `datumctl logs [--tail] [--since] [filters]` against the query
   layer. kubectl-style streaming output with timestamps, service names, and
   severity. See [logs](./logs/).

8. **Standardized OTel resource attributes — definition and enforcement**
   Define the baseline resource attribute contract for all Datum Cloud workloads
   (`service.name`, `service.namespace`, `k8s.cluster.name`, `datum.project.id`) and update the OTel Collector resource processor to
   inject and validate them. See [logs](./logs/#resource-attribute-contract).

9. **Compute operations Grafana dashboards**
   Dashboards for the operations team: per-tenant log volume, error rates,
   ingest pipeline throughput (NATS lag, ClickHouse insert latency), storage
   utilization.

---

### Phase 2: Network Telemetry (target: August–September 2026)

Required for the physical infrastructure deliverable. Datum will operate
physical network hardware that needs telemetry beyond what OTel and Kubernetes
collectors provide. All network telemetry feeds into the NATS ingest pipeline
for durability.

**Deliverables and sub-issues:**

10. **gNMIc streaming telemetry — deployment and NATS integration**
    Deploy [gNMIc](https://github.com/openconfig/gnmic) to collect streaming
    telemetry from OpenConfig-capable network devices. Publish to the NATS ingest
    pipeline under `telemetry.network.<device_id>` subjects. See
    [network-telemetry](./network-telemetry/).

11. **SNMP collection — deployment and VictoriaMetrics integration**
    For devices without gNMI support, deploy an SNMP Exporter scraped by
    VMAgent into the existing VictoriaMetrics cluster. See
    [network-telemetry](./network-telemetry/).

12. **Akvorado flow data — deployment and ClickHouse schema**
    Deploy [Akvorado](https://github.com/akvorado/akvorado) to collect
    NetFlow/IPFIX/sFlow from network hardware. Define the ClickHouse schema for
    flow records. See [network-telemetry](./network-telemetry/).

13. **NATS mTLS — cert-manager integration for leaf-to-hub connections**
    Complete mTLS on all leaf-to-hub NATS connections using cert-manager-issued
    certificates. Required before accepting data from untrusted edge environments.

14. **Edge staging/production overlays**
    Staging and production Kustomize overlays for edge clusters (real hub NATS
    endpoint, registry bridge image, ExternalSecrets for credentials).

---

### Phase 3: Policy-Driven Metrics (target: Q4 2026)

Introduce a stable, declarative contract for metric definition and production
that separates *what* metrics are from *how* they are produced. Enables
consistent metrics across all platform resources and a forward path for
alerting and metering.

**Deliverables and sub-issues:**

15. **MetricDefinition/MetricPolicy CRDs — implementation**
    Implement the API types designed in [definition-policy](./definition-policy/).
    `MetricDefinition` declares metric identity (name, instrument, unit, label
    schema). `MetricPolicy` binds to a definition and declares production rules
    from control-plane resource fields or data-plane telemetry.

16. **Policy translation controller — VM recording rules + OTel pipelines**
    A controller that watches MetricDefinition/MetricPolicy resources and
    produces backend-specific configs: VictoriaMetrics recording rules and OTel
    Collector processor configurations. See [definition-policy](./definition-policy/).

17. **Metrics ingest pipeline — NATS + ClickHouse schema**
    Extend the NATS hub stream to include `telemetry.metrics.<project_id>`
    subjects alongside existing log subjects. Add a metrics ingest table and
    materialized view in ClickHouse, parallel to the logs schema. See
    [metrics](./metrics/).

18. **Retention rollup — three-tier AggregatingMergeTree**
    Implement the three-tier rollup (raw → hourly → daily with TTL) and log TTL
    configuration designed in [retention](./retention/).

---

### Phase 4: North Star (target: 2027)

The long-term platform that the short and medium term work converges toward:
ClickHouse as the unified backing store, data residency as a first-class
property, and full customer self-service over their own telemetry.

**Deliverables and sub-issues:**

19. **Multi-region query fanout**
    Extend the query layer to fan out to per-region ClickHouse instances without
    replicating data across regions. Implement the region registry and k-way
    result merge. See [query-layer](./query-layer/#multi-region-fanout).

20. **Customer-facing traces — tenancy model and query API**
    IAM-gated trace access via Tempo (already deployed). The OTel tail-sampling
    pipeline and Tempo backend are live; what's missing is a tenancy model,
    query API, and customer-facing surface.

21. **Compute observability** (issue #99)
    Design and implement customer-facing observability for compute workloads:
    CPU, memory, and network metrics scoped per tenant workload.

22. **Migration of ClickHouse and NATS hub to Datum physical infrastructure**
    When Datum-operated physical infrastructure is available, migrate off GCP.

---

## Component Enhancements

| Enhancement | Description |
|---|---|
| [logs](./logs/) | Log pipeline — ingest, ClickHouse storage, and `datumctl logs` |
| [metrics](./metrics/) | Metrics pipeline — VictoriaMetrics (current) and NATS + ClickHouse (future) |
| [query-layer](./query-layer/) | Tenant-enforcing HTTP query service over ClickHouse for logs and metrics |
| [definition-policy](./definition-policy/) | LogDefinition, LogCollectionPolicy, MetricDefinition, MetricPolicy CRDs |
| [export-policies](./export-policies/) | ExportPolicy — customer-configured OTLP/Prometheus export |
| [retention](./retention/) | Log TTL and metrics three-tier rollup — raw → hourly → daily |
| [nats-ingest-pipeline](./nats-ingest-pipeline/) | NATS JetStream ingest with per-edge store-and-forward durability |
| [network-telemetry](./network-telemetry/) | gNMIc, SNMP, and flow data for network hardware |

## System Context

![System context diagram](./system-context.png)
