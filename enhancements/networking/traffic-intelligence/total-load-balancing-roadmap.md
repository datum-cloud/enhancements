# Total Load Balancing — Roadmap

**Product area:** Deliver — Edge Delivery / Load Balancing / Fraud & Traffic Mgmt  
**Status:** Early definition  
**Parent:** [Total Load Balancing](total-load-balancing.md)

---

The roadmap is organized around four phases, each building on the foundation established by the last. Phase 1 puts the core infrastructure in place — the signal distribution fabric, health monitoring, DDoS protection, and Compute location awareness in Envoy. Every subsequent phase adds signals and consumers: geo in Phase 2, path quality in Phase 3, and sovereignty and AI routing in Phase 4.

---

## Phase 1 — Signal Fabric & Compute Awareness

Phase 1 establishes the three foundational systems that every later phase depends on: Higgins Bus as the signal distribution fabric, Nate as the active health probe infrastructure, and Beard as the DDoS detection and scrubbing layer. On top of that foundation, Zava gains its first active use of the platform's new real-time awareness — Compute location discovery via EDS.

### Higgins Bus

| # | Task | Description |
|---|---|---|
| 1.1 | **MOQT relay cluster** | Deploy the moqstream relay cluster as the signal distribution backbone. Regional relays fan out published objects from the control plane to all PoP subscribers without a hub-and-spoke bottleneck — a relay failure in one region does not affect subscribers in others. |
| 1.2 | **Track namespace model** | Define canonical track namespace conventions for all current and future signal types. Each signal type occupies a named track; publishers and consumers are fully decoupled. Track namespaces are additive — a new signal type requires no changes to existing relay infrastructure or subscribers. |
| 1.3 | **Pub/sub and bootstrap semantics** | Implement subscribe, bootstrap (a new subscriber receives the most recent object immediately on connect), heartbeat, and TTL expiry behaviors. A subscriber that loses connectivity enforces the last-received object until the TTL expires, then fails to a configured safe default. |
| 1.4 | **PoP subscriber agents** | Deploy subscriber agents at each PoP. Agents receive published objects from the regional relay and serve signal data to local consumers (Envoy, DNS, ALB) in-process — no round-trip to a central service on the request path. |
| 1.5 | **Named IP list distribution** | First production use of Higgins Bus: customer-managed and platform-managed named IP lists distributed to every PoP. Each list is a track; every update is a new object carrying the full list payload, a monotonic sequence number, and a TTL. Updates propagate within QUIC's latency budget — typically sub-second from commit to enforcement. |

### Nate

| # | Task | Description |
|---|---|---|
| 1.6 | **Core probe infrastructure** | Deploy probe agents at Datum PoPs. Implement HTTP/HTTPS and TCP availability checks against Datum infrastructure targets — PoPs and upstreams. Establish probe scheduling, timeout, and retry logic. Protocol coverage is additive; new probe types require no changes to scheduling or aggregation infrastructure. |
| 1.7 | **PoP health tracks on Higgins Bus** | Publish aggregated HealthStatus objects to `platform/health/pop/{pop-id}` on every status transition (HEALTHY → DEGRADED → UNHEALTHY and back) and on a heartbeat interval, so consumers can detect a stalled publisher. New subscribers receive the most recent object immediately on connect. |
| 1.8 | **Latency measurement** | Instrument connection time, TLS handshake time, time-to-first-byte, and total response time across probe types. Publish p50/p95/p99 latency per probe origin region in the HealthStatus object. |
| 1.9 | **Per-region health status** | Aggregate probe results by region. HealthStatus includes a per-region breakdown alongside the overall status, allowing consumers to distinguish PoP-local issues from regional network issues. |

### Beard

| # | Task | Description |
|---|---|---|
| 1.10 | **FastNetMon detection** | Deploy FastNetMon at each PoP to monitor flow telemetry (sFlow, NetFlow, IPFIX) and XDP-derived counters. Configure per-prefix detection thresholds (pps/bps) at warning and attack levels. Enable adaptive baseline learning so thresholds derive from each prefix's observed traffic profile rather than manual fixed values. |
| 1.11 | **XDP/eBPF inline scrubbing** | Implement kernel fast-path packet processing: SYN cookie proxying, UDP source validation, bogon filtering, IP reputation lookup, and PPS/BPS rate limiting. Rules load into eBPF maps and update dynamically as attack signatures evolve. Cleaned traffic passes directly to Envoy within the same PoP — no redirection, no tunneling, no added latency. |
| 1.12 | **BGP FlowSpec (surgical upstream)** | Integrate GoBGP to announce BGP FlowSpec rules (RFC 8955) to upstream transit peers. Rules encode flow-matching conditions and drop or rate-limit actions. Rule lifecycle is managed automatically — rules are withdrawn when the attack signature clears. |
| 1.13 | **RTBH (last resort)** | Implement Remote Triggered Black Hole via GoBGP. RTBH requires explicit customer opt-in or a platform-defined volume threshold. All announcements are time-bounded; the control plane monitors for attack cessation and withdraws as soon as conditions allow. |
| 1.14 | **Attack-state signals to Higgins Bus** | Publish attack state (clean / elevated / mitigating / blackholed), attack class, intensity (bps/pps), active mitigation mode, and affected prefixes to `platform/ddos/pop/{pop-id}` on every state transition and on a heartbeat during active attacks. Enables downstream consumers to distinguish attack-induced degradation from infrastructure failure. |

### Zava

| # | Task | Description |
|---|---|---|
| 1.15 | **Compute location data model** | Define a canonical representation of a Compute Instance location: ID, region, zone, PoP affinity, address, and capacity tier. Stored in the platform control plane and exposed via a gRPC xDS-compatible API as EDS-ready endpoint metadata. |
| 1.16 | **Envoy EDS integration** | Configure Envoy's Endpoint Discovery Service (EDS) to subscribe to Compute Instance location updates from the control plane. Envoy receives endpoint adds, removes, and weight changes without restart. |
| 1.17 | **Compute upstream cluster** | Define a named Envoy cluster (`compute`) backed by EDS. Initial policy: round-robin across all healthy Compute endpoints in the local PoP, failing over to any reachable Compute endpoint. |
| 1.18 | **Passive outlier detection** | Configure Envoy's built-in outlier detection on the Compute cluster — eject endpoints on consecutive 5xx errors, connection failures, or latency outliers. At high volume this reacts faster than active probe-based checks and requires no external dependency. Nate active-check results will be layered in as an additional signal when the Nate Project ships. |
| 1.19 | **Locality-aware LB** | Tag EDS endpoints with Envoy locality (region + zone). Enable Envoy's locality-weighted load balancing so same-PoP Compute endpoints are preferred before spilling to remote PoPs. |
| 1.20 | **Observability** | Emit per-endpoint request count, error rate, and latency from Envoy to the metrics pipeline. This becomes the baseline for all future signal-driven steering decisions. |

**Exit criteria:** Higgins Bus is carrying named IP lists to every PoP. Nate is publishing PoP health state to Higgins Bus. Beard is detecting and mitigating attacks inline at every PoP with attack-state signals live on Higgins Bus. Envoy at any PoP can discover all Compute endpoints via EDS, load balance with locality preference, and automatically eject unhealthy endpoints — with no manual cluster config changes.

---

## Phase 2 — Geo Signal Integration

Geo signals from Roy Kent are introduced across the full stack in dependency order: distribution first, then the consumers. Higgins Bus delivers GeoDB snapshots to every PoP. Zava enforces geo-based policy and enriches requests. Jamie Tartt steers DNS to the nearest healthy PoP. Beard incorporates geo data into XDP scrubbing rules. Nate integrates with routing consumers.

### Jamie Tartt

| # | Task | Description |
|---|---|---|
| 2.1 | **GeoIP backend (Roy Kent MMDB)** | Feed the Roy Kent GeoDB into the PowerDNS GeoIP backend in MMDB format. DNS resolves to the nearest healthy PoP based on client country and region. EDNS Client Subnet is used where available so the geo lookup reflects the client's actual location rather than the resolver's. |
| 2.2 | **Health-aware failover (Nate)** | Subscribe to `platform/health/pop/{pop-id}` from Nate via Higgins Bus. When a PoP transitions to UNHEALTHY or DEGRADED, remove or deprioritize it from the PowerDNS answer set. Restore it on recovery. |
| 2.3 | **DDoS-aware steering (Beard)** | Subscribe to `platform/ddos/pop/{pop-id}` from Beard. When a PoP is mitigating or blackholed, deprioritize it in DNS answers to steer new sessions to less-affected PoPs. This signal is distinct from Nate health — it prevents incorrectly retiring a healthy PoP that is absorbing attack traffic. |
| 2.4 | **GeoSteeringPolicy resource** | Implement the customer-facing `GeoSteeringPolicy` CRD. Customers configure geo steering on/off, DNS TTL, fallback behavior, and sovereignty rules per zone. The control plane translates policies into PowerDNS zone configuration. |
| 2.5 | **EDNS Client Subnet** | Configure PowerDNS ECS handling. When a resolver includes an ECS option, use the client subnet for the geo lookup rather than the resolver IP. Document accuracy implications for clients behind ECS-absent resolvers. |

### Zava

| # | Task | Description |
|---|---|---|
| 2.6 | **Geoblocking** | Evaluate country and region block-list policies against the client's geo lookup on every inbound request using the PoP-local Roy Kent GeoDB replica. Return HTTP 403 for denied origins. Envoy is the correct enforcement point — DNS cannot block after TLS termination with full request context. |
| 2.7 | **Request metadata enrichment** | Set `x-datum-geo-country`, `x-datum-geo-region`, `x-datum-asn`, and `x-datum-ip-type` on every request. Downstream services and WAF rules consume these headers without performing their own geo lookups. |
| 2.8 | **IP type policy** | Use IP type signal (residential / datacenter / proxy / VPN / satellite) to apply differential rate limits or route requests to a scrubbing path before they reach the Compute upstream. |

### Roy Kent

| # | Task | Description |
|---|---|---|
| 2.9 | **GeoDB distribution via Higgins Bus** | Publish GeoDB version notifications to `platform/geodb/version`. PoP subscriber agents pull the current snapshot from object storage on notification and install it locally. Where the vendor supports incremental feeds, publish delta patches to `platform/geodb/delta/{version}` to avoid full re-downloads. |
| 2.10 | **ASN and IP type signals** | Extend the PoP-local GeoDB with ASN and IP type data. Make Autonomous System Number (carrier, cloud provider, peering relationship) and IP type (residential, datacenter, proxy, VPN, satellite) available to all local consumers alongside geo data. |

### Nate

| # | Task | Description |
|---|---|---|
| 2.11 | **GSLB integration (Jamie Tartt)** | Confirm PoP health signal consumption by Jamie Tartt's PowerDNS control plane. Validate that PoP health transitions trigger timely PowerDNS record updates and that the update mechanism operates without service interruption. |
| 2.12 | **Endpoint health to EDS** | Publish endpoint-level health to `platform/health/endpoint/{endpoint-id}`. Envoy EDS consumes endpoint health state to complement passive outlier detection — covering zero-traffic scenarios such as cold Compute nodes and pre-launch regions that passive detection cannot observe. |
| 2.13 | **Customer-defined HealthChecks** | Implement the `HealthCheck` and `HealthCheckPolicy` CRDs. Customers configure probes targeting their own origins, third-party APIs, or private endpoints accessible via Connectors. Customer health results published to `org/{org-id}/health/{check-id}`. |
| 2.14 | **Roy Kent geo for probe selection** | Use Roy Kent GeoDB to inform probe location selection — choose origin PoPs with ASN diversity within each region to distinguish PoP-local issues from single-ISP outages. |

### Beard

| # | Task | Description |
|---|---|---|
| 2.15 | **Geo filtering via Roy Kent in XDP** | Incorporate the PoP-local Roy Kent GeoDB into XDP/eBPF scrubbing rules. Customers whose service has no legitimate users in a particular country can configure geo-based drop rules enforced at the XDP layer, tightened automatically during an active attack. |
| 2.16 | **Customer IP block lists** | Consume customer-managed named IP lists from Higgins Bus (`org/{org-id}/lists/{list-id}`) and load them into XDP eBPF maps. A customer can add a newly identified attack source range and have it enforced at every PoP within seconds. |
| 2.17 | **DNS attack-state integration (Jamie Tartt)** | Confirm Beard's `platform/ddos/pop/{pop-id}` signal is consumed by Jamie Tartt's PowerDNS control plane for PoP deprioritization during active attacks. Validate that GSLB and Nate health signals are interpreted independently so attack-induced degradation is not confused with infrastructure failure. |
| 2.18 | **Coraza L7 escalation** | Establish the signal flow from Coraza (Envoy WASM WAF) to the Beard mitigation control plane. When Coraza identifies HTTP flood or slowloris source IPs, Beard escalates to the XDP layer — blocking at L3 before packets reach Envoy. Coordinate per-source rate limit tightening between layers. |

### Higgins Bus

| # | Task | Description |
|---|---|---|
| 2.19 | **GeoDB tracks** | Establish `platform/geodb/version` and `platform/geodb/delta/{version}` track namespaces. Validate the hybrid distribution model: version notification via MOQT, bulk snapshot pull from object storage, delta patch where supported. Confirm PoP-local install and reload behavior for all GeoDB consumers (Envoy, DNS, ALB). |
| 2.20 | **Health tracks (Nate)** | Validate `platform/health/pop/{pop-id}` and `platform/health/endpoint/{endpoint-id}` track delivery end-to-end. Confirm bootstrap behavior, heartbeat delivery, and consumer behavior on status transition. |
| 2.21 | **DDoS attack-state tracks (Beard)** | Validate `platform/ddos/pop/{pop-id}` and `org/{org-id}/ddos/{prefix}` track delivery. Confirm attack-state transition objects propagate to GSLB and Total Load Balancing consumers within the latency budget. |

---

## Phase 3 — Latency and Path Quality

Add RTT, packet loss, and congestion as first-class routing inputs across the stack. Jamie Tartt begins scoring PoPs by measured path quality rather than geographic distance alone. Zava scores and weights upstreams dynamically. Nate latency signals feed the scoring model. Beard's attack-state integrates directly into Envoy's routing weight model.

### Jamie Tartt

| # | Task | Description |
|---|---|---|
| 3.1 | **RTT-proxied PoP scoring** | Incorporate RTT signal from Higgins Bus into PowerDNS PoP selection. Prefer PoPs with lower measured path latency to the client over pure geographic proximity. Geographic distance remains the fallback where RTT signal is unavailable. |
| 3.2 | **Weighted multi-PoP answers** | When multiple PoPs serve the same region, return weighted answers based on path quality scores. Distribute load across healthy PoPs in proportion to their current score rather than returning a single best PoP. |

### Zava

| # | Task | Description |
|---|---|---|
| 3.3 | **Scored upstream routing** | Extend the EDS endpoint model with a latency score derived from RTT and loss signals on Higgins Bus. Replace round-robin with a scored selection policy that weights endpoints by path quality. |
| 3.4 | **Congestion-aware spillover** | When a PoP's congestion signal exceeds a threshold, reduce its EDS endpoint weight rather than removing it entirely — graceful degradation keeps endpoints in rotation at reduced capacity rather than creating binary in/out state. |
| 3.5 | **Policy-over-performance** | Establish evaluation order: sovereignty constraints eliminate candidates first; latency scoring applies within compliant candidates only. Performance optimization never overrides policy. |

### Nate

| # | Task | Description |
|---|---|---|
| 3.6 | **Latency signals to scored routing** | Confirm Nate's per-region latency measurements (p50/p95/p99) are consumed by Zava's scored routing model. Validate that Nate latency data and RTT signals from Higgins Bus compose correctly in the endpoint scoring function. |
| 3.7 | **Throughput checks** | Implement download and upload throughput probes on a separate, lower-frequency schedule. Throughput signals inform congestion-aware spillover — a PoP with degraded throughput under load should receive reduced weight before it becomes unavailable. |
| 3.8 | **Extended protocol support** | Add TLS, DNS, gRPC, and ICMP probe types. Extended protocol coverage improves health signal accuracy for infrastructure components not reachable over plain HTTP. |

### Beard

| # | Task | Description |
|---|---|---|
| 3.9 | **Attack-state into Envoy routing** | Wire Beard's `platform/ddos/pop/{pop-id}` attack-state signal into Envoy's EDS endpoint weight model. When a PoP is mitigating an attack, reduce the EDS weight of its endpoints rather than waiting for passive outlier detection or Nate health transitions to catch the degradation. |
| 3.10 | **Endpoint weight reduction under attack** | When Beard signals an active attack against a specific endpoint or prefix, reduce that endpoint's EDS weight directly. This is the L7 counterpart to the L3/L4 inline scrubbing — both layers respond to the same attack state simultaneously. |

### Higgins Bus

| # | Task | Description |
|---|---|---|
| 3.11 | **RTT / loss / congestion tracks** | Establish `platform/rtt/{pop-id}` track namespace. Define object schema for RTT measurements, packet loss rate, and link utilization. Validate delivery latency — path quality signals must reach consumers fast enough to be actionable for routing decisions. |

---

## Phase 4 — Sovereignty, AI Routing, and Geo for Compute

Add hard-constraint sovereignty enforcement at both DNS and L7. Extend the routing decision layer to inference-bound traffic — model locality and GPU availability join geography and health as routing inputs. Make geo and health signals available to Compute workloads. Deliver L4 customer-configurable load balancing for Compute targets. Complete Higgins Bus with durable replay for historical analysis and AI workload history requirements.

### Jamie Tartt

| # | Task | Description |
|---|---|---|
| 4.1 | **Sovereignty enforcement** | Implement hard jurisdiction constraints in GSLB. A non-compliant PoP is excluded from the DNS answer set regardless of proximity or health. Sovereignty rules are evaluated before proximity or quality scoring — no performance signal overrides a compliance constraint. |
| 4.2 | **Jurisdiction-aware PoP selection** | Maintain a PoP jurisdiction registry in the control plane mapping each PoP to its legal jurisdiction. Translate customer `GeoSteeringPolicy` sovereignty rules into PowerDNS routing logic that excludes non-compliant PoPs per client region. |

### Dani Rojas

| # | Task | Description |
|---|---|---|
| 4.3 | **Compute-target L4 routing** | Implement customer-configurable L4 load balancing for Compute targets via `L4LoadBalancerPolicy` and Gateway API. Cilium routes TCP/UDP traffic to Compute instances based on policy — a transport-layer complement to Zava's L7 routing for workloads that do not require HTTP-level decisions. |

### Zava

| # | Task | Description |
|---|---|---|
| 4.4 | **Sovereignty constraints** | Model jurisdiction constraints as a first-class routing signal. Evaluate against compliant PoP and endpoint sets before any latency scoring in Envoy route selection. A non-compliant endpoint is eliminated regardless of performance. |
| 4.5 | **Model locality routing** | Publish which Compute nodes have a given inference model loaded as an EDS endpoint attribute. When a request carries an inference context, filter the EDS candidate set to nodes with the model warm before applying latency scoring. |
| 4.6 | **Inference routing** | Route inference-bound requests to the Compute endpoint best positioned to serve — model warm, low current load, within sovereignty constraints — applying the same scoring and policy model as standard routing extended with inference-specific attributes. |
| 4.7 | **GPU weight modifier** | Extend the Compute location data model with GPU utilization. Apply as a weight modifier in Envoy EDS so saturated GPU nodes receive progressively less traffic before they become unavailable. |

### Roy Kent

| # | Task | Description |
|---|---|---|
| 4.8 | **Geo for Compute workloads** | Make the GeoDB available to Compute workloads via enriched request headers forwarded from Envoy (established in Phase 2). Provide a local GeoDB SDK for Compute functions that need direct lookups for data residency enforcement, personalisation, or model selection without an external round-trip. |

### Nate

| # | Task | Description |
|---|---|---|
| 4.9 | **Connector-based probing** | Enable probes to reach targets inside customer networks, private cloud environments, or non-public endpoints via Datum Connectors. The probe agent routes through the Connector tunnel; the HealthCheck spec references the Connector by name. |
| 4.10 | **Compute endpoint health checks** | Extend probe targets to include Compute instances directly. Health state for individual Compute endpoints feeds EDS and informs GPU weight modifiers — a Compute node that is reachable but failing health checks is removed from routing before inference requests are sent to it. |

### Beard

| # | Task | Description |
|---|---|---|
| 4.11 | **Compute endpoint protection** | Extend Beard detection and scrubbing to traffic targeting Compute instances and inference endpoints. GPU nodes under volumetric attack trigger the same XDP/eBPF and FlowSpec response as any other PoP target. Per-prefix attack state published to `org/{org-id}/ddos/{prefix}`. |

### Higgins Bus

| # | Task | Description |
|---|---|---|
| 4.12 | **Inference and agent coordination tracks** | Establish track namespaces for inference workload coordination: `platform/inference/capacity/{model-id}/{region}/{worker-id}` for worker heartbeat and load state, `platform/model-locality/{model-id}/{worker-id}` for model warm/cold state, and `{org}/inference/{request-id}/tokens` for token streaming. TTL acts as a heartbeat — a dead worker disappears from the namespace automatically. |
| 4.13 | **Durable replay (TSDB)** | Deploy the time-series adapter that subscribes to Higgins Bus tracks and writes each received object to a TSDB (Hydrolix or equivalent). Enables historical signal replay for incident review, post-mortem analysis, and AI workloads that require signal history rather than current state only. Retention windows vary by signal type — health events at 30–90 days, IP list updates at 7 days. |

**Exit criteria:** Sovereignty constraints eliminate non-compliant PoPs at both DNS and L7 before any performance scoring is applied. Inference requests route to warm-model, low-utilization Compute endpoints within sovereignty boundaries. Compute workloads have direct access to geo, ASN, and IP type context. Higgins Bus carries inference coordination signals alongside routing signals on a single shared fabric, with full durable replay capability.
