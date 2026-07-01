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
  - [ClickHouse schema — gNMIc network telemetry](#clickhouse-schema--gnmic-network-telemetry)
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
gNMIc and SNMP data enter the NATS ingest pipeline and land in ClickHouse;
flow data lands in a separate Akvorado-managed ClickHouse instance.

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
  `telemetry.network.<device_id>` subjects on the local core-NATS leaf node,
  which forwards to the hub. Like OTel logs, durability begins at the hub; the
  edge leaf has no persistent storage (see
  [Integration with the NATS ingest pipeline](#integration-with-the-nats-ingest-pipeline)).

**Subscription paths (initial):**

| Path | Purpose |
|---|---|
| `/interfaces/interface/state/counters` | Per-interface byte/packet/error counters |
| `/interfaces/interface/state/oper-status` | Interface operational status |
| `/bgp/global/state` | BGP session state and prefix counts |
| `/system/cpus/cpu/state/usage` | CPU utilization |
| `/system/memory/state` | Memory utilization |
| `/components/component/state` | Hardware component health |

These OpenConfig paths are illustrative and must be verified against each
device's actual YANG models at implementation — vendors vary in which paths they
support and where (for example, BGP state may live under
`/network-instances/network-instance/protocols/protocol/bgp/...` rather than a
top-level path). Do not treat the paths above as canonical.

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
    address: nats-leaf.telemetry.svc:4222  # leaf client port; leaf↔hub uses 7422
    subject-prefix: telemetry.network
    # format: event emits name/timestamp/tags/values — field names do not map
    # directly to ClickHouse columns via JSONEachRow. An output template or
    # alternative format is required; exact approach is TBD at implementation.
    format: event
```

**ClickHouse consumer (hub):**

Network telemetry events arriving at the hub are consumed by a ClickHouse NATS
engine table subscribing to `telemetry.network.>`. The schema follows the same
pattern as logs: a NATS ingest table, a materialized view, and a MergeTree
table. Column names and partitioning are deferred to implementation — see
[ClickHouse schema — gNMIc network telemetry](#clickhouse-schema--gnmic-network-telemetry).

### SNMP collection

For network devices that do not support gNMI (older switches, out-of-band
management hardware, UPS units), the OTel Collector
[`snmpreceiver`](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/snmpreceiver)
(contrib component) polls SNMP devices directly and emits native OTLP metrics.
No separate SNMP Exporter deployment is required.

**Deployment:**

The OTel Collector (already deployed for OTel log/metric ingest) adds a
`snmpreceiver` configuration targeting each device. The resource processor
injects `milo.device.id` and `milo.project.id = 'internal'`; the
bridge routes to `telemetry.metrics.internal` on NATS. SNMP metrics land
in the `otel_metrics_*` ClickHouse tables and are queryable from Grafana by
filtering on `ResourceAttributes['milo.device.id']`. Device data lives under the
reserved `internal` project and is read by operators via the
no-row-policy operator path (e.g. `grafana_ops`), not the tenant `api_reader`
path — network telemetry is internal-operations only in this phase.

SNMP metrics are structurally OTLP metric data points and cannot share the
`telemetry.network` ingest table with gNMIc — gNMIc publishes OpenConfig
`Path`/`Value` events, a format incompatible with the OTLP metric schema. The
two paths land in separate ClickHouse tables by design.

**OID collections (initial):**

| Collection | Purpose |
|---|---|
| IF-MIB interfaces | Per-interface byte/packet/error counters |
| ENTITY-MIB components | Hardware component state |
| APC UPS MIB | Battery status and load |

The initial set is scoped to the device classes in the Phase 3 deployment.
Vendor-specific MIBs (e.g. a Cisco WLC MIB for wireless controllers) are
candidates for later phases if that hardware is in scope — not part of the
initial collection set.

**OTel Collector config (additions):**

```yaml
receivers:
  snmp:  # component type name is "snmpreceiver"; the config key is "snmp"
    collection_interval: 60s
    agents:
      - endpoint: udp://device-ip:161
        version: v2c
        community: ${SNMP_COMMUNITY}
    attributes:
      device.id:
        value: device-ip
        type: resource-attribute
    metrics:
      # IF-MIB
      if.in.octets:
        unit: By
        description: ifInOctets
        type: gauge
        scalar_oids:
          - oid: "1.3.6.1.2.1.2.2.1.10"
            attributes:
              - name: if.name
                oid: "1.3.6.1.2.1.2.2.1.2"

processors:
  resource/snmp:
    attributes:
      - key: milo.device.id
        from_attribute: device.id
        action: insert
      - key: milo.project.id
        value: internal
        action: insert

exporters:
  otlphttp/bridge:
    endpoint: http://otlp-nats-bridge.telemetry.svc:4318
```

The exact OID-to-metric mapping is deferred to implementation; the config above
is illustrative. The `snmpreceiver` supports column OIDs (for per-interface
metrics) and scalar OIDs (for device-level metrics); the full configuration
reference is in the contrib documentation.

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

The `flows` ClickHouse instance is a **fifth** instance: by Phase 3, sentry,
activity, `edge-logs-system` (not yet decommissioned), and the new telemetry
instance (Phase 1) are already running. `edge-logs-system` is decommissioned at
the end of Phase 4, leaving four instances in the final state. All share the
`clickhouse-operator`.

> [!NOTE]
>
> **Deliberate exception to "Minimize database engines."** Akvorado introduces
> two things that conflict with the guiding principles: an additional ClickHouse
> instance, and an internal Kafka broker alongside NATS. Kafka is also notable
> because the [ingest-pipeline](../ingest-pipeline/) design explicitly rejected
> it as a general-purpose message bus.
>
> Both exceptions are accepted because Akvorado is a self-contained appliance,
> not a component we compose:
>
> - **Kafka** — internal to Akvorado's deployment and managed by it. Replacing
>   it requires forking Akvorado or running a Kafka-to-NATS bridge with no
>   operational benefit. The ingest-pipeline Kafka rejection applies to the
>   telemetry write path; Akvorado's Kafka is not on that path.
> - **Separate ClickHouse** — Akvorado manages its own schema and runs
>   migrations automatically on deployment. Sharing the telemetry ClickHouse
>   would couple Akvorado schema upgrades to the telemetry table lifecycle.
>   The ClickHouse operator handles multiple independent instances cheaply.
>
> Flow data is architecturally distinct — it arrives from network hardware over
> NetFlow/IPFIX/sFlow, not through OTel Collector pipelines. Treating Akvorado
> as a separate appliance is the correct model. See
> [Alternatives](#alternatives) for the explicitly rejected options.

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
NATS leaf node (core NATS, no JetStream — bounded in-memory buffer only)
    │ mTLS / WAN
    ▼
NATS hub (JetStream — aggregates from all sites; durability begins here)
    │
    ├──► ClickHouse network_ingest (NATS engine) → MV(s) → network (MergeTree, schema TBD)
    └──► Future: alerting consumer for interface flap, BGP session down
```

Durability begins at the hub, the same as OTel logs — edge clusters have no
persistent storage, so the leaf is core NATS with a bounded in-memory buffer.
If the management network between a site and GCP is disrupted, gNMIc keeps
publishing to the local leaf, which forwards on reconnect; but a prolonged
outage is bounded by the leaf's in-memory buffer, and beyond it events are
dropped. Network telemetry is operational (not billing-relevant), so under the
[buffer-priority principle](../#guiding-principles) it is shed before
billing-relevant data when capacity is constrained.

Akvorado does **not** use the NATS pipeline — it has its own internal Kafka
buffer between inlet and ClickHouse. SNMP data enters the pipeline as OTLP
metrics via the OTel Collector `snmpreceiver`, published to
`telemetry.metrics.internal` and landing in the `otel_metrics_*`
ClickHouse tables — not in `network_ingest`. See [SNMP collection](#snmp-collection).

### ClickHouse schema — gNMIc network telemetry

Akvorado manages its own flow schema (see the Akvorado section above); this
section covers only the gNMIc `network_ingest` path.

The column schema for `network_ingest` is deferred to implementation. gNMIc's
`event` format emits `name`, `timestamp`, `tags`, and `values` fields — these
do not match the column names `DeviceId/Timestamp/Path/Value` shown in earlier
drafts, so `JSONEachRow` ingestion would require either a gNMIc output template
that renames the fields or a different gNMIc format. The intent is to capture
device identity, timestamp, path, and value payload; the exact column names must
be verified against gNMIc's output at implementation.

The MergeTree schema will be defined per subscription path type (interface
counters, BGP state, etc.) since each has a different set of fields. This is
also deferred to implementation.

---

## Deployment topology

```
[Physical site — e.g. data center 1]

  Router / switch ──gNMI──► gNMIc ───────────────publish──► NATS leaf ──mTLS──► NATS hub (GCP)
  Legacy device ───SNMP──► OTel Collector (snmpreceiver) ──► NATS leaf      (telemetry.metrics.internal)
  Router / switch ──NetFlow──► Akvorado inlet ──► Akvorado ClickHouse

  NATS leaf: core NATS, no JetStream (bounded in-memory buffer)

[GCP]

  NATS hub ──► network_ingest (NATS engine) ──► MV(s) ──► network (MergeTree) ──► Grafana   (gNMIc)
  NATS hub ──► otel_metrics_* (metrics pipeline) ──► Grafana                       (SNMP)
  Akvorado ──► Grafana (Akvorado datasource plugin)
```

Management plane access (gNMI credentials, SNMP community strings) is stored in
GCP Secret Manager and injected via ExternalSecrets. gNMIc connects to devices
over the management network using mTLS where the device supports it.

---

## Alternatives

**SNMP Exporter + prometheus receiver** — run a separate Prometheus SNMP
Exporter and scrape it with the OTel Collector's `prometheusreceiver`. The OTel
Collector contrib `snmpreceiver` is preferred: it polls SNMP devices natively
in a single component, emits OTLP directly without a Prometheus translation
step, and preserves structured OID metadata (interface names, OID hierarchies)
that the Prometheus text format flattens.

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
