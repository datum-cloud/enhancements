---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Network Telemetry

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Design Details](#design-details)
  - [Streaming telemetry — gNMIc](#streaming-telemetry--gnmic)
  - [SNMP collection](#snmp-collection)
  - [Flow data — Akvorado](#flow-data--akvorado)
  - [Integration with the NATS ingest pipeline](#integration-with-the-nats-ingest-pipeline)
  - [ClickHouse schema for flows](#clickhouse-schema-for-flows)
- [Deployment topology](#deployment-topology)
- [Alternatives](#alternatives)

## Summary

Datum operates physical network hardware — routers, switches, and servers —
that produce telemetry beyond what OTel and Kubernetes collectors support.
This enhancement covers three collection paths for network telemetry:

- **gNMIc**: streaming telemetry from OpenConfig-capable devices (gRPC-based)
- **SNMP**: polling-based metrics from legacy devices without gNMI support
- **Akvorado**: NetFlow/IPFIX/sFlow flow data for traffic analysis and capacity planning

All three paths integrate with the existing observability infrastructure:
gNMIc data enters the NATS ingest pipeline alongside OTel data; SNMP data goes
to VictoriaMetrics; flow data lands in ClickHouse.

## Motivation

The August 2026 physical infrastructure deliverable requires the operations
team to have visibility into the health and traffic of Datum's physical network.
Without it, diagnosing failures, planning capacity, and detecting anomalous
traffic requires manual SSH sessions and ad-hoc tooling.

Streaming telemetry and flow data provide the same operational visibility for
network hardware that OTel provides for software services.

### Goals

- Operational metrics from network hardware visible in Grafana
- Flow data (NetFlow/IPFIX/sFlow) queryable for traffic analysis and anomaly
  detection
- Data from network devices enters the NATS ingest pipeline for at-least-once
  durability
- SNMP fallback for devices that do not support gNMI
- Extensible to new device types without redeploying collectors

### Non-Goals

- Real-time anomaly detection or alerting — the pipeline enables it; alerting
  is a separate concern
- Customer-facing network metrics — network telemetry is internal-operations
  only in this phase
- Configuration management or device provisioning

---

## Design Details

### Streaming telemetry — gNMIc

[gNMIc](https://github.com/openconfig/gnmic) is an open-source gNMI client that
supports streaming telemetry subscriptions. It connects to network devices using
the gNMI protocol (gRPC over TLS) and receives push-based telemetry at
configurable intervals — without polling.

**How it works:**

gNMIc subscribes to OpenConfig paths on each device (e.g., interface counters,
BGP session state, CPU utilization). Devices stream updates at the configured
interval. gNMIc receives these and publishes to one or more outputs.

**Outputs:**

- **NATS**: gNMIc has a native NATS output. Events are published to
  `telemetry.network.<device_id>` subjects on the local NATS leaf node, which
  replicates to the hub. This gives network telemetry the same at-least-once
  durability as OTel logs.
- **Prometheus (fallback)**: For simpler metric types (counters, gauges), gNMIc
  can expose a `/metrics` endpoint scraped by VMAgent.

**Subscription paths (initial):**

| Path | Purpose |
|---|---|
| `/interfaces/interface/state/counters` | Per-interface byte/packet/error counters |
| `/interfaces/interface/state/oper-status` | Interface operational status |
| `/bgp/global/state` | BGP session state and prefix counts |
| `/system/cpus/cpu/state/usage` | CPU utilization |
| `/system/memory/state` | Memory utilization |
| `/components/component/state` | Hardware component health |

**Configuration:**

```yaml
# gnmic-config.yaml
username: gnmi-reader
password: ${GNMI_PASSWORD}
skip-verify: false
encoding: json_ietf
subscriptions:
  interfaces:
    paths:
      - /interfaces/interface/state/counters
      - /interfaces/interface/state/oper-status
    mode: stream
    stream-mode: sample
    sample-interval: 30s
  bgp:
    paths:
      - /bgp/global/state
    mode: stream
    stream-mode: on-change

outputs:
  nats:
    type: nats
    address: nats-leaf.telemetry.svc:4222
    subject-prefix: telemetry.network
    format: event
```

**ClickHouse consumer (hub):**

Network telemetry events arriving at the hub are consumed by a ClickHouse NATS
engine table subscribing to `telemetry.network.>`. Schema mirrors the logs
pattern: a NATS ingest table, a materialized view, and a MergeTree table
partitioned by `(DeviceId, toDate(Timestamp))`.

### SNMP collection

For network devices that do not support gNMI (older switches, out-of-band
management hardware, UPS units), SNMP polling via
[SNMP Exporter](https://github.com/prometheus/snmp_exporter) provides metric
collection compatible with the VictoriaMetrics stack.

**Deployment:**

SNMP Exporter runs as a Deployment in the management cluster. VMAgent
configures a scrape job against the SNMP Exporter, which walks OIDs on each
target device at the scrape interval.

**Module selection:**

SNMP Exporter uses modules to define which OIDs to walk. Initial modules:

| Module | Purpose |
|---|---|
| `if_mib` | Interface counters (ifInOctets, ifOutOctets, ifErrors) |
| `cisco_wlc` | Wireless controller metrics (if applicable) |
| `apc_ups` | APC UPS battery status and load |
| `entity_mib` | Hardware component state |

**VMAgent scrape config:**

```yaml
- job_name: snmp
  static_configs:
    - targets:
        - 192.168.1.1  # switch-01
        - 192.168.1.2  # switch-02
  metrics_path: /snmp
  params:
    module: [if_mib]
  relabel_configs:
    - source_labels: [__address__]
      target_label: __param_target
    - target_label: __address__
      replacement: snmp-exporter.telemetry.svc:9116
```

SNMP metrics land in VictoriaMetrics alongside existing infrastructure metrics.
No separate ClickHouse table is needed for SNMP data in Phase 2.

### Flow data — Akvorado

[Akvorado](https://github.com/akvorado/akvorado) is an open-source flow
collector and analyzer. It receives NetFlow v5/v9, IPFIX, and sFlow from network
hardware and stores flow records in ClickHouse.

Flow data is the primary tool for traffic analysis, capacity planning, and
anomaly detection at the network layer. It answers questions like: which
prefixes are generating the most traffic, which ASes are peers vs. transit,
where is bandwidth being consumed, and what traffic patterns look anomalous.

**Architecture:**

```
Network device (router/switch)
    │ NetFlow/IPFIX/sFlow
    ▼
Akvorado inlet (UDP listener)
    │
    ▼
Akvorado orchestrator (enrichment: GeoIP, ASN, interface name)
    │
    ▼
Kafka (durable buffer between inlet and ClickHouse)
    │
    ▼
Akvorado ClickHouse writer
    │
    ▼
ClickHouse (flows database)
    │
    ▼
Akvorado API + UI (built-in analytics and Grafana datasource)
```

Akvorado includes a built-in Kafka broker for durability between inlet and
ClickHouse. This is a separate Kafka instance from any potential future general
message bus — it is internal to the Akvorado deployment and managed by it.

**ClickHouse schema for flows:**

Akvorado manages its own ClickHouse database and schema (`flows`). The schema
is defined by Akvorado and updated automatically on deployment. Key tables:

- `flows` — raw flow records (MergeTree, partitioned by day, 90-day TTL)
- `flows_1m0s` — 1-minute aggregates (for dashboards)
- `flows_5m0s` — 5-minute aggregates (for long-range analysis)

The `flows` ClickHouse instance is a **fourth** instance alongside sentry,
activity, and edge-logs. All share the `clickhouse-operator`.

**Grafana integration:**

Akvorado exposes a Grafana datasource plugin that uses its API for
traffic queries. Dashboards include: top talkers (by source/dest IP, prefix,
AS, interface), traffic by protocol, and sankey diagrams for traffic flow.

### Integration with the NATS ingest pipeline

gNMIc network telemetry integrates with the NATS ingest pipeline by publishing
to `telemetry.network.<device_id>` subjects on the local NATS leaf node:

```
gNMIc
    │ publish to telemetry.network.<device_id>
    ▼
NATS leaf node (local durability during management network disruption)
    │ mTLS / WAN
    ▼
NATS hub (aggregates from all sites)
    │
    ├──► ClickHouse network_ingest (NATS engine) → network_mv → network (MergeTree)
    └──► Future: alerting consumer for interface flap, BGP session down
```

This gives network telemetry the same store-and-forward durability as OTel
logs. If the management network between a site and GCP is disrupted, gNMIc
continues publishing to the local leaf JetStream, and events are forwarded to
the hub when connectivity is restored.

SNMP and Akvorado do **not** use the NATS pipeline: SNMP goes directly to
VictoriaMetrics (pull-based, loss-tolerant) and Akvorado has its own internal
Kafka buffer.

### ClickHouse schema for flows

See the Akvorado section above — Akvorado manages its own schema. For gNMIc
network telemetry, the schema follows the same pattern as logs:

```sql
-- NATS engine ingest table
CREATE TABLE telemetry.network_ingest (
    DeviceId   LowCardinality(String),
    Timestamp  DateTime64(9),
    Path       String,
    Value      String   -- JSON event payload from gNMIc
) ENGINE = NATS
SETTINGS
    nats_url = 'nats://nats-hub.telemetry.svc:4222',
    nats_subjects = 'telemetry.network.>',
    nats_format = 'JSONEachRow';

-- Materialized view + MergeTree (to be defined per OpenConfig path type)
```

The exact MergeTree schema for network metrics will be defined per subscription
path type (interface counters, BGP state, etc.) since each has a different set
of fields. This is deferred to implementation.

---

## Deployment topology

```
[Physical site — e.g. data center 1]

  Router / switch ──gNMI──► gNMIc ──publish──► NATS leaf ──mTLS──► NATS hub (GCP)
  Legacy device ───SNMP──► SNMP Exporter ◄──scrape── VMAgent ──remote-write──► VMCluster
  Router / switch ──NetFlow──► Akvorado inlet ──► Akvorado ClickHouse

[GCP]

  NATS hub ──► network_ingest (NATS engine) ──► network (MergeTree) ──► Grafana
  VMCluster ──► Grafana
  Akvorado ──► Grafana (Akvorado datasource plugin)
```

Management plane access (gNMI credentials, SNMP community strings) is stored in
GCP Secret Manager and injected via ExternalSecrets. gNMIc connects to devices
over the management network using mTLS where the device supports it.

---

## Alternatives

**OpenTelemetry network device receiver** — the OTel Collector community has
experimental receivers for network device telemetry (gNMI, SNMP). Using the
OTel Collector instead of gNMIc and SNMP Exporter would reduce the number of
components. Rejected for Phase 2: gNMIc is purpose-built for streaming
telemetry and has a more complete implementation of gNMI subscriptions and
output formats than the experimental OTel receivers. SNMP Exporter is the
standard tool in the VictoriaMetrics ecosystem and is well-tested. Revisit
when OTel network receivers mature.

**NATS for Akvorado** — replace Akvorado's internal Kafka with the NATS hub.
Akvorado uses Kafka as an internal buffer between inlet and ClickHouse; this is
not exposed as a configurable output. Rejected: Akvorado manages its own buffer;
replacing it requires forking Akvorado or running a Kafka-to-NATS bridge.
The operational benefit is marginal — Akvorado's ClickHouse is already separate
from the main telemetry ClickHouse.

**Single ClickHouse for flows and logs** — reuse the Phase 1 telemetry
ClickHouse for Akvorado flow data. Rejected: Akvorado manages its own schema
migrations and expects to own its database. Mixing schemas in one instance adds
operational coupling. The ClickHouse operator handles multiple independent
instances cheaply.
