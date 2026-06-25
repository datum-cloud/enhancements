# Zava: Envoy Routing

**Parent:** [Total Load Balancing](total-load-balancing.md)  
**Status:** Early definition  
**Codename:** Internal project name — not a go-to-market product name.

---

Named after Zava — the misunderstood striker. Powerful, capable of things nobody else on the field can do, and largely an enigma to everyone trying to work with him.

> "Avocados are misunderstood."

Envoy is the same. It handles more than most people realize — geo authorization, circuit breaking, outlier detection, weighted routing, protocol translation, policy evaluation hooks — and is often reduced in conversation to "the proxy." This document is an attempt to actually understand it.

---

Envoy is Datum's Layer 7 edge proxy. It sits behind Cilium ([Dani Rojas](l4-load-balancing-dani-rojas.md)) in the platform stack and in front of origin upstreams, handling HTTP/HTTPS routing, TLS termination, header manipulation, and origin selection for all delivery traffic. Where Cilium routes by IP and port, Envoy routes by request — path, host, method, header, and increasingly by signals from the broader Total Load Balancing layer.

This document maps the full set of routing capabilities Envoy needs to deliver for the Datum platform, assesses what Envoy handles natively versus what requires integration with Datum systems, and identifies the open questions that need resolution before integration work begins.

---

## Platform Architecture

```
Internet
    │
    ▼
Cilium (L4 LB)   ← Dani Rojas: TCP/UDP routing to Envoy and compute targets
    │
    ▼
Envoy (L7 LB)    ← this document: HTTP routing, TLS, origin selection
    │
    ├─── Origin upstreams / delivery edge
    └─── UFO Compute (Unikraft) via Connectors
```

Envoy is platform-managed. Customers configure Envoy behavior through Datum delivery policies — they do not interact with Envoy configuration directly.

---

## Feature Matrix

| # | Feature | Status | Envoy Mechanism / Integration Point |
|---|---|---|---|
| 1 | Geo Aware Routing | Integration Required | Geo **authorization** (allow/block by country) is native in Envoy Gateway 1.8.0 via `SecurityPolicy.spec.authorization.rules[*].principal.clientIPGeoLocations` + `EnvoyProxy.spec.geoIP`; geo-aware **upstream selection** (route to closest compute) still requires GeoDB integration with `weighted_clusters` or control plane |
| 2 | Static Endpoint Mapping | Native | EDS static resources or static `load_assignment` in cluster config |
| 3 | Dynamic Latency Based Routing | Integration Required | No native real-time latency routing; control plane must update cluster priorities or weights using Nate health signals from Higgins Bus |
| 4 | Active Health Checks | Native | Cluster-level `health_checks` config; HTTP, TCP, gRPC probe types built in |
| 5 | Fast Endpoint Scaling | Native | Endpoint Discovery Service (EDS) via xDS; control plane pushes endpoint adds/removes in real time |
| 6 | Endpoint Warmup Awareness | Native | `slow_start_config` on cluster; new endpoints ramp up weight over a configurable window |
| 7 | Per Request Load Balancing | Native | Default behavior; round-robin, least-request, random, ring hash — selected per cluster |
| 8 | Service Discovery Integration | Native | EDS/xDS; control plane is the authoritative source; Envoy subscribes and updates dynamically |
| 9 | Circuit Breaking | Native | `circuit_breakers` thresholds per cluster — max connections, pending requests, retries, requests |
| 10 | Retry and Failover Logic | Native | `retry_policy` per route or virtual host; configurable conditions, backoff, and retry host predicates |
| 11 | Outlier Detection | Native | `outlier_detection` per cluster; ejects hosts on consecutive errors, success rate deviation, or failure percentage |
| 12 | Traffic Shaping and Rate Limiting | Integration Required | Local rate limiting native via `local_ratelimit` filter; platform-wide limits require external rate limit service via `ratelimit` filter |
| 13 | Canary and Weighted Routing | Native | `weighted_clusters` in route config; weights sum to 100 and can be updated via xDS without restart |
| 14 | Path and Host Based Routing | Native | Core Envoy routing model — `virtual_hosts`, `routes`, `match` on path/prefix/regex, `:authority` (host), headers, methods |
| 15 | Protocol Agnostic Routing | Integration Required | HTTP/1.1, HTTP/2, HTTP/3, TCP, UDP all supported; each requires explicit listener and filter chain configuration — no zero-config protocol detection |
| 16 | Connection Aware Routing | Native | Ring hash and Maglev LB policies for consistent hashing; `stateful_session` filter for cookie-based session affinity |
| 17 | Policy Driven Routing | Integration Required | `ext_proc` (external processing) filter is the integration point for Datum policy engine; sovereignty, cost, and compliance constraints evaluated externally |
| 18 | Observability Hooks | Integration Required | Native stats, access logs, and distributed tracing built in; requires wiring to Datum's OpenTelemetry pipeline and metrics system |
| 19 | Request Timeout Management | Native | `timeout` and `idle_timeout` per route and cluster; `max_stream_duration` for streaming workloads; `per_try_timeout` on retry policies |
| 20 | WAF — OWASP Rule Enforcement | Integration Required | Coraza WAF runs as a WebAssembly filter inside Envoy; implements OWASP ModSecurity Core Rule Set; inspects HTTP/HTTPS inline for injection, XSS, path traversal, and application-layer DDoS patterns |

---

## Integration Details

### 1. Geo Aware Routing

Geo routing covers two distinct use cases with different integration paths.

**Geo authorization (allow/block by country or region).** Envoy Gateway 1.8.0 adds native GeoIP support via `SecurityPolicy.spec.authorization.rules[*].principal.clientIPGeoLocations`, backed by a shared GeoIP provider configured in `EnvoyProxy.spec.geoIP`. This covers the geo blocking use case (Roy Kent consumer #2 — Delivery Edge Proxy) without custom filter work. The GeoIP database used by Envoy Gateway needs to be sourced from Roy Kent's GeoDB and kept current — the distribution mechanism (how the GeoDB replica on each PoP is made available to Envoy Gateway's GeoIP provider) is an open question.

**Geo-aware upstream selection (route to closest compute).** Routing traffic to the geographically closest compute instance is not covered by the native GeoIP feature, which is scoped to authorization rules only. This requires geo context injected into routing decisions. Two integration approaches remain possible:

- **Header injection (filter-side).** An HTTP filter performs a per-request GeoDB lookup against the PoP-local Roy Kent replica and injects headers (`X-Client-Country`, `X-Client-Region`) before the routing step. Route config matches on these headers to select the correct upstream cluster.
- **Control plane (xDS-driven).** The control plane encodes geographic topology into cluster priorities or weighted cluster configuration pushed via xDS. Coarser granularity — geo modeled at the cluster level rather than per request — but no per-request lookup cost.

**Datum systems involved:** Roy Kent GeoDB (geo data source), Higgins Bus (GeoDB distribution to PoPs), Datum control plane (xDS publisher), Envoy Gateway 1.8.0 GeoIP provider (for authorization use case).

**Dependency:** Roy Kent Stage 2 (Geo Blocking) covers the authorization integration. Roy Kent Stage 3 (Application Load Balancing) delivers the upstream-selection integration. See [Roy Kent Project](ip-geo-roy-kent.md).

---

### 3. Dynamic Latency Based Routing

Dynamic latency-based routing is a significant project in its own right — not a configuration exercise. It requires a measurement infrastructure, a signal distribution layer, a control plane that translates measurements into routing decisions, and Envoy integration to act on those decisions. It is sequenced after the Roy Kent Project deliberately, because Roy Kent delivers many of the foundational blocks it depends on.

**What Roy Kent provides as a foundation:**

- The Higgins Bus distribution infrastructure — the MOQT pub/sub transport that carries signals to every PoP. Latency signals will use the same relay topology and track model that Roy Kent establishes for geo data.
- PoP-local data stores and the update pipeline for pushing data from the control plane to edge nodes.
- Geographic coordinates (lat/lon) for each PoP and for client IPs — used as a distance proxy in the interim before real RTT measurements are available. Galactic VPC already uses this for initial PoP ranking (see [Roy Kent Project](ip-geo-roy-kent.md), consumer #6).
- Production validation of the end-to-end distribution path under real traffic load, before latency-based routing depends on it for higher-stakes decisions.

**What latency-based routing adds on top:**

Real latency measurement is not the same as geographic distance. A user in London routed to a Frankfurt PoP may see worse latency than a user in Frankfurt routed to Amsterdam, depending on peering, IX routing, and undersea cable paths. The latency project requires:

- **Measurement infrastructure** — probes that measure actual RTT between PoPs and between PoPs and origin upstreams. The [Nate Project](health-checks-nate.md) covers active health check latency for endpoint availability; RTT mapping at the network path level is a separate workstream not yet scoped.
- **Signal aggregation** — raw RTT samples aggregated into stable routing signals, filtered to remove noise from transient congestion or probe outliers.
- **Track namespace on Higgins Bus** — a `platform/rtt/{pop-id}` track (reserved in the Higgins Bus namespace) carrying aggregated RTT signals to consuming components.
- **Control plane translation** — the Datum control plane subscribes to RTT signals and translates them into Envoy xDS updates: cluster priorities, locality weights, or weighted cluster splits. Routing decisions operate at control plane update frequency, not per-request frequency.
- **Envoy integration** — cluster configuration updates via xDS that Envoy acts on. Envoy's built-in algorithms (least-request, ring hash) do not consume external latency measurements; the control plane must encode latency intelligence into cluster topology and weights that Envoy then applies.

The freshness gap between a change in real-world latency and Envoy acting on it — probe → Nate/RTT aggregation → Higgins Bus → control plane → xDS → Envoy — needs to be measured and evaluated against the use cases the feature is meant to serve.

**Datum systems involved:** RTT measurement infrastructure (TBD), Nate (endpoint latency signals), Higgins Bus (signal transport, `platform/rtt/{pop-id}` track), Roy Kent GeoDB (distance proxy in interim), Datum control plane (signal subscriber and xDS publisher).

---

### 12. Traffic Shaping and Rate Limiting

Envoy's `local_ratelimit` filter enforces per-Envoy-instance rate limits without external coordination. This is sufficient for per-connection or per-listener limits on a single Envoy instance but does not enforce a limit across the full fleet — each Envoy instance has its own independent counter.

Platform-wide rate limiting — a rate limit that applies across all Envoy instances serving a tenant or route — requires the `ratelimit` filter pointing at an external rate limit service. Envoy sends a check request to the rate limit service on each request; the service maintains the shared counter and returns allow or deny. Datum needs to operate this external service or integrate with an existing one.

The scope of what requires platform-wide versus local limits needs to be determined. Protecting individual Envoy instances from connection floods is local. Enforcing a customer's API quota across the fleet is platform-wide.

**Datum systems involved:** External rate limit service (to be determined — standalone deployment or integrated into the policy engine).

---

### 15. Protocol Agnostic Routing

Envoy supports HTTP/1.1, HTTP/2, HTTP/3 (QUIC), TCP proxy, and UDP proxy. Support for each protocol requires explicit listener configuration with the appropriate filter chain: `http_connection_manager` for HTTP variants, `tcp_proxy` for raw TCP, and `udp_proxy` for UDP. There is no automatic protocol detection that selects the right filter chain based on incoming traffic.

For each protocol Datum needs to support at the edge, a corresponding listener and filter chain configuration must be defined and maintained. HTTP/1.1 and HTTP/2 are covered by the standard delivery edge configuration. HTTP/3 requires a QUIC listener with `quic_transport` socket. TCP and UDP proxy require separate listener configurations and do not participate in HTTP-level features (routing, header manipulation, retries at the HTTP layer).

Protocol requirements should be enumerated per product area — delivery edge, AI gateway, compute connectivity — so the listener configuration scope is clear.

**Datum systems involved:** Datum control plane (xDS configuration generation per protocol); no external signal dependencies.

---

### 17. Policy Driven Routing

Datum will enforce routing constraints based on sovereignty (data residency rules), compliance (jurisdiction-specific restrictions), cost (preferring cheaper paths within acceptable latency bounds), and other policy dimensions that are not expressible as static route config.

Envoy's `ext_proc` (external processing) filter is the designed integration point for this class of decision. On each request, `ext_proc` calls an external gRPC service with request headers (and optionally body). The external service can modify headers, inject metadata, or return a direct response (to block a request). A routing filter downstream sees the modified headers or metadata and applies the route.

The external service that evaluates policy has not been identified. Open questions cover ownership, latency budget, and failure mode.

**Datum systems involved:** Datum policy engine (ext_proc service, to be defined); Roy Kent GeoDB (for jurisdiction lookup, likely already injected upstream in the filter chain).

---

### 18. Observability Hooks

Envoy generates a rich set of signals natively: per-cluster and per-route request counts, error rates, latency histograms, upstream connection metrics, access logs (structured JSON or custom format), and distributed traces (Zipkin, OpenTelemetry, Jaeger format). These are available out of the box once Envoy is running.

The integration work is connecting these signals to Datum's observability pipeline:

- **Metrics:** Envoy exposes a Prometheus-compatible stats endpoint (`/stats/prometheus`). Datum's metrics collection agent scrapes this endpoint and routes metrics into the platform metrics store.
- **Access logs:** Envoy access logs must be configured with the correct sink — gRPC access log service (`grpc_access_log_config`) to ship to the ingest pipeline, or file sink consumed by a local collector agent.
- **Distributed traces:** Envoy's `http_connection_manager` supports OpenTelemetry trace export. Trace context must be propagated correctly to upstream services so traces are continuous end-to-end.
- **Routing decision context:** Where routing decisions are influenced by geo context, Nate health signals, or policy evaluation, that context should be reflected in access log fields or trace attributes so operators can understand why a request was routed to a particular upstream.

**Datum systems involved:** Datum metrics pipeline (scraping or push target), Datum log ingest pipeline, Datum tracing infrastructure (OpenTelemetry collector).

---

### 20. WAF — Coraza

[Coraza](https://coraza.io) is an open-source WAF engine that runs as a WebAssembly filter inside Envoy, inspecting HTTP/HTTPS requests and responses inline without an external sidecar or additional network hop.

**What Coraza provides:**

| Capability | Detail |
|---|---|
| OWASP CRS | ModSecurity Core Rule Set — SQL injection, XSS, path traversal, RCE, protocol anomalies |
| Custom rules | Datum and customer-defined rules in SecLang syntax alongside the CRS |
| Request/response inspection | Headers, body, URI — full HTTP/HTTPS payload visibility |
| Application-layer DDoS signals | HTTP flood patterns, scanner fingerprints, malformed request sequences |
| Audit logging | Per-request rule match events for security analysis and incident response |

**Integration path:** Coraza ships as a WASM binary (`coraza-proxy-wasm`) loaded into Envoy's HTTP filter chain via the `wasm` filter. The Datum control plane distributes Coraza's rule set and configuration to each PoP via the standard xDS mechanism or as a versioned artifact via Higgins Bus (same pattern as GeoDB snapshots).

**Relationship to Beard:** Coraza's rule match events and request rate counters are the L7 signal source for [Beard](ddos-scrubbing-beard.md). When Coraza identifies HTTP flood or attack tool patterns, those signals flow to Beard's mitigation control plane, which can escalate to L3/L4 enforcement at the XDP layer — blocking attacking IPs before their packets reach Envoy.

**Datum systems involved:** Datum control plane (rule set distribution), Beard mitigation control plane (L7 signal consumer), Datum log ingest pipeline (Coraza audit log events).

---

## Open Questions

**Geo authorization: how does Roy Kent GeoDB feed Envoy Gateway's GeoIP provider?**
Envoy Gateway 1.8.0 provides native GeoIP-based authorization via `EnvoyProxy.spec.geoIP`, but the GeoIP database it consumes must be sourced and kept current. Roy Kent distributes GeoDB snapshots to PoP-local storage via Higgins Bus. The integration question is how Envoy Gateway's GeoIP provider is pointed at the Roy Kent-managed local replica and how it picks up updates when a new snapshot is installed. Is this a file path reference that Envoy Gateway hot-reloads, or does it require a restart or xDS push?

**Geo upstream selection: filter or control plane?**
For routing traffic to the geo-closest compute instance (distinct from geo authorization), the filter-side header injection and xDS-driven control plane approaches remain open. Which approach does Datum commit to, and what is the filter implementation path if header injection is chosen (Lua, WASM, ext_proc)?

**Who owns the ext_proc service for policy evaluation?**
Policy Driven Routing depends on an external processing service that evaluates sovereignty, compliance, and cost constraints. This service needs to be low-latency (it is in the hot path for every request), highly available (a failed ext_proc service can either fail open or fail closed — both have risks), and aware of current policy state. Which team owns this service? Does it integrate with an existing policy store, or is it a new component? What is the acceptable latency budget for a policy evaluation call?

**How does Nate signal propagation latency affect latency-based routing freshness?**
Latency signals flow from Nate probes → Nate control plane → Higgins Bus → Datum control plane → xDS push → Envoy. Each hop adds latency. For a degrading upstream, how quickly does a change in Nate's published HealthStatus translate into updated cluster weights at Envoy? Is the end-to-end propagation time acceptable for the use cases latency-based routing is intended to serve? What is the measurement target?

**Platform-wide rate limiting service: standalone or embedded in policy engine?**
Traffic shaping at fleet scale requires an external rate limit service. Datum can deploy Envoy's reference implementation (`envoy/ratelimit`) as a standalone service or embed rate limit evaluation into the same ext_proc service used for policy decisions. The former is simpler operationally but adds another service to the fleet. The latter consolidates the hot-path external calls but couples concerns. Which direction is preferred?

**How is Envoy's xDS control plane structured?**
Several features in this document depend on the Datum control plane acting as an xDS management server: it pushes EDS updates for endpoint scaling, cluster weight updates for latency-based routing, and route config updates for geo routing. Is there a single xDS control plane, or are multiple components acting as xDS sources for different resource types? Which component subscribes to Higgins Bus signals and translates them into xDS updates?

**Routing decision attribution in observability.**
When Envoy routes a request to an upstream based on geo context, health signals, or policy, that reasoning should be visible to operators in access logs and traces. What fields and attributes are required in the access log schema to reconstruct a routing decision post-hoc? Who defines this schema and ensures it is populated by all relevant filters in the chain?
