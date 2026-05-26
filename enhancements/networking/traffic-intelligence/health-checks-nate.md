# The Nate Project: Active Health Checks

**Parent:** [Total Load Balancing](total-load-balancing.md)  
**Status:** Early definition  
**Codename:** Internal project name — not a go-to-market product name.

---

Named after Nathan Shelley — the kit man who obsessively catalogued every weakness, noticed what others overlooked, and produced meticulous analysis nobody asked for but everyone eventually relied on. Active health checks work the same way: unglamorous, thorough, and quietly essential to everything that routes.

---

## What Nate Does

Nate is Datum Cloud's active health checking system. It probes endpoints and infrastructure continuously from geographically distributed vantage points, measures availability, latency, and throughput across a wide range of protocols, and publishes the results as health signals on [Higgins Bus](signal-distribution-higgins-bus.md) so every routing and policy component in the platform has a current view of what is up, what is degraded, and what is unreachable.

Nate is the active side only — it initiates probes and reports measurements. Applications and infrastructure manage their own passive health state through their internal mechanisms. Nate does not observe, aggregate, or configure that process.

---

## Design Principles

**Active only.** Nate sends probes. It does not observe application-layer signals, sidecar telemetry, or infrastructure events. Passive health state is the responsibility of the systems that generate it.

**Distributed.** Probers run at Datum PoPs. No single probe location is authoritative — health is a consensus across multiple vantage points, with geography informing which vantage points are meaningful for a given target.

**Protocol-agnostic.** Nate can check endpoints over any protocol with a defined probe type. Adding a new protocol requires only a new probe type implementation — no changes to scheduling, distribution, or aggregation.

**Signal-first.** Nate publishes raw measurements and aggregated health state as signals on Higgins Bus. What to do with a degraded endpoint — reroute, alert, circuit-break — is the consumer's responsibility. Nate does not make failover decisions.

**Dual-use.** Nate serves both Datum-internal infrastructure monitoring (PoP health, upstreams, connectors) and customer-defined health checks (origin servers, third-party APIs, internal services behind connectors). The same probe infrastructure, object model, and distribution path serves both.

**Boundary-agnostic.** Probes can reach targets inside or outside the Datum network. An endpoint behind a Connector is reachable from PoP-local probers via that Connector. A third-party SaaS API is reachable from any PoP with external egress. The HealthCheck definition controls where probes originate; the target address controls where they go.

---

## What Nate Measures

### Availability

Is the target responding? Availability is a binary signal at the probe level — a probe either succeeds or fails — but aggregate availability over a time window and across multiple probe locations produces a more useful signal than any single result.

A target is considered available when a configured majority of probes from a configured set of locations succeed within the timeout window.

### Latency

How long does a successful probe take? Nate measures latency at multiple layers depending on protocol:

| Measurement | Description | Protocols |
|---|---|---|
| **Connection time** | Time to establish transport connection | All stream protocols |
| **TLS handshake time** | Time to complete TLS negotiation | HTTPS, gRPC, TLS |
| **Time to first byte (TTFB)** | Time from request sent to first byte received | HTTP, HTTPS, gRPC |
| **Total response time** | Time from probe initiation to response complete | HTTP, HTTPS, gRPC |
| **Round-trip time** | ICMP round-trip time | ICMP |
| **Query time** | DNS resolution time | DNS |

All latency measurements are tagged with the probe origin region so consumers can distinguish latency differences between geographic vantage points.

### Throughput

How much data can the target transfer? Throughput probes measure bulk transfer rate between the prober and the target. HTTP throughput probes fetch a large object (download) or post a payload (upload). Custom throughput probes can be defined for protocols that support them.

Throughput probes are more resource-intensive than availability and latency probes. They run on a separate, configurable schedule — typically less frequent than availability checks — and must be explicitly opted into per HealthCheck.

---

## Supported Protocols

| Protocol | Probe | Measures |
|---|---|---|
| **HTTP** | GET or POST to a configured path | Status code, body match, connection time, TTFB, response time |
| **HTTPS** | Same as HTTP over TLS | Same as HTTP plus TLS handshake time, certificate validity, certificate expiry |
| **TCP** | SYN + ACK (connect only) | Connection success, connect time |
| **TLS** | TCP connect + TLS handshake | Certificate validity, certificate expiry, handshake time, negotiated cipher suite |
| **ICMP** | Echo request (ping) | Reachability, round-trip time |
| **DNS** | Resolve a configured name at a configured resolver | Resolution success, answer match, query time |
| **gRPC** | gRPC Health Check protocol (`grpc.health.v1`) | RPC success, response code, latency |
| **UDP** | Echo or custom payload | Reachability (where the target supports UDP echo), round-trip time |
| **SMTP** | EHLO exchange | Server reachability, banner match |
| **Custom** | Extensible probe type | Defined per probe type implementation |

Protocol coverage is additive. New probe types do not require changes to the scheduling, distribution, or aggregation infrastructure.

---

## Core Objects

Nate defines a small set of Kubernetes-native API resources.

### HealthCheck

A HealthCheck defines what to probe, how to probe it, and from where. It is the primary configuration surface for both Datum operators and customers.

```yaml
apiVersion: networking.datum.net/v1alpha
kind: HealthCheck
metadata:
  name: origin-api
  namespace: acme-corp
spec:
  target:
    address: api.acme.com
    port: 443
  protocol: HTTPS
  http:
    path: /healthz
    method: GET
    expectedStatusCodes: [200]
    expectedBody: "ok"          # optional substring match
    followRedirects: false
  interval: 30s
  timeout: 10s
  probe:
    regions:
      - us-east
      - eu-west
      - ap-southeast
    minimumProbeCount: 2        # minimum PoPs per region
  checks:
    - availability
    - latency
    - throughput                # opt-in; lower default frequency
```

### HealthCheckPolicy

A HealthCheckPolicy defines thresholds and consensus rules for a HealthCheck. It is separate from the probe configuration so thresholds can be updated without modifying probe parameters.

```yaml
apiVersion: networking.datum.net/v1alpha
kind: HealthCheckPolicy
metadata:
  name: origin-api-policy
  namespace: acme-corp
spec:
  targetRef:
    kind: HealthCheck
    name: origin-api
  availability:
    healthyThreshold: 0.90      # 90% of probes must succeed to be HEALTHY
    unhealthyThreshold: 0.50    # below 50% = UNHEALTHY
    window: 60s
  latency:
    warning: 300ms              # p95 warning threshold
    critical: 1000ms            # p95 critical threshold
  evaluation:
    consensus: MAJORITY         # MAJORITY | ALL | ANY
    minimumRegions: 2           # requires signal from at least N regions
```

### HealthStatus

HealthStatus is the aggregated view published to Higgins Bus. Individual raw probe results are internal to the system; HealthStatus is what consumers see. It is emitted on every status transition and on a heartbeat interval.

```yaml
# Published to: platform/health/pop/{pop-id}
#            or platform/health/endpoint/{endpoint-id}
#            or org/{org-id}/health/{check-id}
status:
  checkRef: origin-api
  overall: DEGRADED             # HEALTHY | DEGRADED | UNHEALTHY | UNKNOWN
  availability: 0.91
  latency:
    p50: 48ms
    p95: 340ms
    p99: 890ms
  byRegion:
    us-east:
      status: HEALTHY
      availability: 0.99
      latency: { p50: 30ms, p95: 85ms }
    eu-west:
      status: DEGRADED
      availability: 0.88
      latency: { p50: 120ms, p95: 340ms }
    ap-southeast:
      status: HEALTHY
      availability: 0.98
      latency: { p50: 62ms, p95: 155ms }
  lastProbeTime: "2025-11-15T14:23:01Z"
  message: "eu-west latency above critical threshold"
```

---

## Geographic Probe Distribution

Nate uses geography in two ways: selecting where probes originate, and interpreting results relative to where users and targets are located. Geographic data for PoP and target coordinates comes from the [Roy Kent Project](ip-geo-roy-kent.md) GeoDB.

### Probe Location Selection

The HealthCheck spec accepts a `regions` list that controls probe origin. When a region is specified, Nate selects PoPs within that region to run probes using these criteria:

- **Coverage:** use multiple PoPs per region where possible, to distinguish PoP-local issues from regional network issues
- **Diversity:** prefer PoPs on different upstream providers or ASNs within the same region, so a single-ISP outage does not misrepresent regional health
- **Egress awareness:** for targets outside the Datum network, probes originate from PoPs with external egress capability

`regions: all` instructs Nate to use a representative set of probe locations spanning all geographic regions where Datum operates. This is the default for Datum-internal infrastructure checks.

For Datum-internal targets (PoP infrastructure, upstreams, backbone), the probe scheduler is platform-managed and not customer-configurable. The platform defines the probe topology for its own infrastructure.

### Regional Signal Interpretation

Latency and availability signals are always tagged with the probe origin region. A target may be healthy from US East and degraded from EU West — these are distinct signals, not averaged together unless the HealthCheckPolicy explicitly defines cross-region consensus rules. Consumers receive per-region breakdown and the aggregated overall status.

### Probing Targets Behind Connectors

For targets that are not publicly reachable, probes can be routed through a [Connector](../connectors/initial-proposal/README.md). The HealthCheck spec can reference a Connector by name; the probe agent at the relevant PoP routes the probe through that Connector's tunnel to reach the private target. This allows health checking of services inside customer networks, private cloud environments, or other non-public endpoints.

---

## Distribution via Higgins Bus

Health status objects are published to [Higgins Bus](signal-distribution-higgins-bus.md) using the MOQT track model. The Nate control plane is the publisher; routing and policy systems across the platform are the consumers.

### Track Namespaces

| Track | Contents | Publisher | Consumers |
|---|---|---|---|
| `platform/health/pop/{pop-id}` | Aggregated HealthStatus for a Datum PoP | Nate control plane | Total Load Balancing, GSLB, Galactic VPC |
| `platform/health/endpoint/{endpoint-id}` | HealthStatus for a Datum-managed endpoint or upstream | Nate control plane | ALB, Connectors, Total Load Balancing |
| `org/{org-id}/health/{check-id}` | HealthStatus for a customer-defined HealthCheck | Nate control plane | Customer ALB policy, customer DNS config, customer consumers |

### Publish Behavior

Objects are published to the appropriate track:

- **Immediately on status transition** — HEALTHY → DEGRADED, DEGRADED → UNHEALTHY, and the reverse
- **On heartbeat interval** — even when status has not changed, so consumers can detect a stale or failed publisher

### Consumer Bootstrap

A new consumer subscribing to a health track receives the most recent HealthStatus object immediately — no separate bootstrap step required. The most recent object is always the authoritative current state.

### Metrics Integration

All probe measurements (raw latency samples, availability counts, throughput observations) are written to the Datum metrics pipeline in parallel with Higgins Bus publication. The metrics path provides time-series data for dashboards, historical analysis, and alerting thresholds. Higgins Bus provides the real-time signal path for routing decisions. The two paths are complementary and serve different consumers.

---

## Consumers

### Datum Platform

| Consumer | Signal Used | How |
|---|---|---|
| [Total Load Balancing](total-load-balancing.md) | PoP health, endpoint health | Avoids routing to degraded or unreachable targets |
| GSLB (DNS) | PoP health | Excludes unhealthy PoPs from DNS answer sets |
| Application Load Balancer | Endpoint health | Removes unhealthy upstreams from rotation |
| [Galactic VPC](../research/galactic-vpc/README.md) | PoP health | Excludes degraded paths from path computation |
| [Connectors](../connectors/initial-proposal/README.md) | Endpoint health | Reflects target health state for connector-backed upstreams |
| Metrics / Observability | All probe measurements | Time-series retention for dashboards, alerting, capacity analysis |

### Customer Access

Customers can define HealthChecks targeting their own origins, third-party APIs, or any endpoint reachable from Datum PoPs (including endpoints accessible through a Connector). Customer-defined checks are subject to quota limits — the number of checks, probe frequency, and probe locations are bounded resources that will be defined during capacity planning.

Customers can read HealthStatus objects for their own checks via API and portal. Customer health tracks on Higgins Bus are accessible to customer systems that subscribe via the Higgins Bus customer access path.

What to do when a check reports UNHEALTHY — remove an upstream from the ALB pool, stop answering DNS for a PoP, send a notification — is configured in the consuming system. Failover decisions are not part of Nate's scope.

---

## Out of Scope

**Failover decisions.** Nate reports health signals. What to do about an unhealthy endpoint is the consumer's responsibility. Failover logic lives in ALB upstream policy, GSLB configuration, and Galactic VPC path selection.

**Passive health checks.** Applications and infrastructure components monitor their own internal health through their own mechanisms — Kubernetes readiness probes, application health endpoints, sidecar telemetry. Nate does not configure, observe, or aggregate passive health state. If a passive signal needs to reach routing components, the application or infrastructure is responsible for publishing it to the appropriate system.

**Alerting and on-call notification.** Nate publishes measurements to the metrics pipeline. Alert thresholds, on-call escalation, and notification routing are configured in the metrics and alerting system, not in Nate.

**Failover logging.** Records of failover events, topology changes triggered by health state, and circuit state history live in the systems that make those decisions. Nate's own log output covers probe results, status transitions, and distribution events only.

**Synthetic user journeys.** Nate checks individual endpoints and protocols. Multi-step synthetic transaction monitoring — simulate a login flow, complete a checkout sequence, verify an end-to-end API workflow — is outside the scope of this project.

---

## Capabilities

Work areas are listed without implied order. Sequencing will be determined during planning.

| Capability | Scope | Status |
|---|---|---|
| **Core probe infrastructure** | HTTP/HTTPS and TCP availability checks; Datum-internal infrastructure targets (PoPs, upstreams); PoP and endpoint health tracks on Higgins Bus | Not started |
| **Latency measurement** | Latency instrumentation across probe types; p50/p95/p99 reporting; per-region latency breakdown in HealthStatus | Not started |
| **Routing integration** | GSLB and ALB consume health signals; HealthCheckPolicy evaluation and consensus logic | Not started |
| **Customer-defined checks** | Customer HealthCheck and HealthCheckPolicy resources; `org/{org-id}/health/` track namespace; quota enforcement | Not started |
| **Connector-based probing** | Probing private targets through Connectors; routing probe traffic through customer network contexts | Not started |
| **Throughput checks** | Download and upload throughput probes; separate scheduling from availability/latency checks | Not started |
| **Extended protocol support** | TLS, DNS, gRPC, ICMP, UDP, SMTP probe types | Not started |
| **Advanced probe selection** | ASN diversity within regions; egress-aware prober assignment | Not started |
| **External vantage points** | Integration with third-party probe networks for probing from outside Datum IP space; provider evaluation required | Not started |
| **Customer distribution access** | Customer system access to Higgins Bus health tracks | Not started |

---

## Open Questions

- **Probe agent architecture.** Where does the probe agent run within a PoP? As a standalone process, a sidecar, or integrated into existing edge infrastructure? What process manages its scheduling and lifecycle?
- **Customer quota limits.** How many HealthChecks can a customer define? What is the maximum probe frequency? What does a check cost in terms of platform resources? To be defined during capacity planning.
- **Connector-based probing routing path.** How exactly does a probe agent at a PoP route a probe through a Connector tunnel? What authentication and authorization controls apply?
- **Consensus defaults.** Should the default consensus model weight regions equally or weight by probe count? The HealthCheckPolicy model supports both; the default needs to be specified.
- **External vantage points.** Should the external vantage points capability integrate a third-party probe network (Catchpoint, ThousandEyes) to probe from outside Datum IP space? No decision made — placeholder for future evaluation.
- **Probe result retention.** How long are raw probe results retained before aggregation? What is the retention policy for aggregated HealthStatus history in the metrics pipeline?
