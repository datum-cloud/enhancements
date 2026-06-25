# Total Load Balancing — Roadmap

**Product area:** Deliver — Edge Delivery / Load Balancing / Fraud & Traffic Mgmt  
**Status:** Early definition  
**Parent:** [Total Load Balancing](total-load-balancing.md)

---

The roadmap is organized around four phases. The first phase is deliberately narrow: give Envoy a real-time view of UFO Compute locations and enable load balancing across them. Every subsequent phase layers signals on top of that foundation.

---

## Phase 1 — Envoy UFO Location Awareness

Envoy (Zava) currently treats origins as static upstream clusters defined at deploy time. It has no runtime awareness of UFO Compute locations, their availability, or their capacity. This phase wires Envoy into a live UFO location registry so it can make load-balancing decisions across UFO endpoints.

| # | Task | Description |
|---|---|---|
| 1.1 | **UFO location data model** | Define a canonical representation of a UFO location: ID, region, PoP affinity, address, and capacity tier. Stored in the platform control plane and exposed via a gRPC xDS-compatible API. |
| 1.2 | **Envoy EDS integration** | Configure Envoy's Endpoint Discovery Service (EDS) to subscribe to UFO location updates from the control plane. Envoy receives endpoint adds, removes, and weight changes without restart. |
| 1.3 | **UFO upstream cluster definition** | Define a named Envoy cluster (`ufo_compute`) backed by EDS. Initial policy: round-robin across all healthy UFO endpoints in the local PoP, failing over to any reachable UFO endpoint. |
| 1.4 | **Passive outlier detection** | Configure Envoy's built-in outlier detection on the `ufo_compute` cluster — eject endpoints on consecutive 5xx, connection failures, or latency outliers. At high volume this reacts faster than probe-based checks and requires no external dependency. Nate active-check results will be layered in during Phase 2 to cover zero-traffic scenarios. |
| 1.5 | **Locality-aware LB** | Tag EDS endpoints with Envoy `locality` (region + zone). Enable Envoy's locality-weighted load balancing so same-PoP UFO endpoints are preferred before spilling to remote PoPs. |
| 1.6 | **Observability** | Emit per-endpoint request count, error rate, and latency from Envoy to the metrics pipeline. This becomes the baseline for all future signal-driven steering decisions. |

**Exit criteria:** Envoy at any PoP can discover all UFO Compute endpoints via EDS, load balance across them with locality preference, and automatically drop unhealthy endpoints from rotation — with no manual cluster config changes.

**Nate dependency note:** When the [Nate Project](health-checks-nate.md) ships, its synthetic health-check results will be fed into EDS endpoint health state to cover zero-traffic scenarios (cold UFO nodes, pre-launch regions) that passive outlier detection cannot see. This is not gated on a phase — it lands whenever Nate is ready.

---

## Phase 2 — Geo Signal Integration

Geo signals from Roy Kent are introduced across three components in dependency order: distribution first, then the two consumers.

### 2a — GeoDB Distribution

Get the Roy Kent geo database reliably to every edge PoP. This is the prerequisite for all geo consumers — neither Envoy nor DNS can act on geo until the data is local.

| # | Task | Description |
|---|---|---|
| 2.1 | **Higgins Bus geo tracks** | Publish Roy Kent geography, ASN, and IP type data as named tracks on Higgins Bus (`platform/geodb/version`, `platform/geodb/asn`, `platform/geodb/iptype`). Full database on connect, delta updates on change. |
| 2.2 | **Local PoP cache** | Each PoP maintains a local copy of the geo database refreshed from Higgins Bus. Lookups are in-process — no round-trip to a central service on the request path. |
| 2.3 | **Version coordination** | Track the active GeoDB version at each PoP and emit it to the metrics pipeline. Detect and alert on version skew across PoPs so stale geo data is visible before it causes routing inconsistencies. |

### 2b — Geo in Envoy

Use the local GeoDB for request-time enforcement and metadata enrichment at L7.

| # | Task | Description |
|---|---|---|
| 2.4 | **Geoblocking** | Evaluate country/region block-list policies against the client's geo lookup on every inbound request. Return HTTP 403 for denied origins. DNS cannot block — Envoy is the correct enforcement point because it operates after TLS termination with full request context. |
| 2.5 | **Request metadata enrichment** | Set `x-datum-geo-country`, `x-datum-geo-region`, `x-datum-asn`, and `x-datum-ip-type` on every request. Downstream services and WAF rules consume these headers without performing their own lookups. |
| 2.6 | **IP type policy** | Use IP type signal (residential / datacenter / proxy / VPN / satellite) to apply differential rate limits or route to a scrubbing path before the request reaches the UFO upstream. |

### 2c — Geo in DNS

Use geo for PoP steering at the DNS layer — a performance decision made before any connection is established.

| # | Task | Description |
|---|---|---|
| 2.7 | **PowerDNS GeoIP backend** | Feed Roy Kent geography into the Jamie Tartt PowerDNS GeoIP backend. DNS resolves to the nearest healthy PoP based on client country and region. |
| 2.8 | **Health-aware GSLB hand-off** | Align DNS PoP health state with Envoy endpoint health so Jamie Tartt does not steer users to a PoP that Envoy has already marked unhealthy. |
| 2.9 | **DDoS state in DNS** | Consume Beard attack-state signals in Jamie Tartt. Remove PoPs under active volumetric attack from DNS resolution before the attack traffic saturates the PoP's Envoy layer. |

---

## Phase 3 — Latency and Path Quality

Add RTT, packet loss, and congestion as first-class inputs to the Envoy upstream selection decision.

| # | Task | Description |
|---|---|---|
| 3.1 | **RTT signal collection** | Define the RTT measurement architecture (passive QUIC/TCP observation or active probing). Publish per-PoP RTT as a Higgins Bus track. |
| 3.2 | **Envoy upstream scoring** | Extend the EDS endpoint model with a latency score derived from RTT and loss signals. Replace round-robin with a scored selection policy. |
| 3.3 | **Congestion-aware spillover** | When a PoP's congestion signal exceeds a threshold, reduce its EDS endpoint weight rather than removing it entirely — graceful degradation. |
| 3.4 | **Policy-over-performance** | Establish evaluation order: sovereignty constraints eliminate candidates first; latency scoring applies within compliant candidates only. |

---

## Phase 4 — Sovereignty, AI Routing, and Geo for Compute

Add hard-constraint sovereignty enforcement, extend the decision layer to inference-bound traffic, and make geo data available to UFO Compute workloads.

| # | Task | Description |
|---|---|---|
| 4.1 | **Sovereignty signal** | Model jurisdiction constraints as a signal (country → allowed-PoP-set). Evaluate before any latency scoring in Envoy route selection. |
| 4.2 | **Model locality signal** | Publish which UFO compute nodes have a given inference model loaded. Add as an EDS endpoint attribute. |
| 4.3 | **Inference routing in Envoy** | When a request carries an inference context header, filter the EDS endpoint set to nodes with the model warm before applying latency scoring. |
| 4.4 | **GPU availability signal** | Extend the UFO location data model with GPU utilization. Apply as a weight modifier in Envoy EDS so saturated nodes receive less traffic before becoming unavailable. |
| 4.5 | **Geo for Compute** | Make the GeoDB available to UFO Compute workloads — expose client geography, ASN, and IP type as request context so compute functions can make locality-aware decisions (e.g. data residency enforcement, personalisation, model selection) without an external lookup. Delivered via enriched request headers forwarded from Envoy (established in Phase 2b) and a local GeoDB SDK for workloads that need direct lookups. |
