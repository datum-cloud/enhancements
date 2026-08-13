# Zava: Envoy Routing

**Parent:** [Total Load Balancing](total-load-balancing.md)  
**Status:** Early definition  
**Codename:** Internal project name. Not a go-to-market product name.

---

Named after Zava — powerful, capable of things nobody else on the field can do, and largely an enigma to everyone trying to work with him.

> "I am an empty vessel filled with gold. I am your rock. Mold me."

Envoy gets underestimated for the same reason. It's easy to call it "a proxy" and stop there, when in practice it handles geo authorization, circuit breaking, outlier detection, weighted routing, protocol translation, and policy evaluation hooks. This document explains what Envoy actually does for the Datum platform.

---

Envoy is Datum's Layer 7 ALB (Application Load Balancer). It sits behind Cilium ([Dani Rojas](l4-load-balancing-dani-rojas.md)) in the platform stack and in front of origin upstreams, handling HTTP/HTTPS routing, TLS termination, header manipulation, and origin selection for all delivery traffic. Where Cilium routes by IP and port, Envoy routes by request: path, host, method, header, and increasingly signals from the broader Total Load Balancing layer.

---

## Use Cases

**Datum Compute origins.** Customers running workloads on Compute (Unikraft microVMs) scale instances in and out, move between PoPs, and warm up new capacity as demand shifts. Datum's ALB (Envoy) sits in front of these instances as the entry point for all HTTP/HTTPS traffic, tracking which endpoints are healthy, which are still warming, and which PoP is closest, then shifting traffic between them without customer involvement. Endpoints scale through EDS without a route change. New instances ramp in gradually instead of taking full traffic on arrival. Unhealthy instances are ejected automatically.

**Customer Origins.** Not every workload behind the Datum edge runs on Datum Compute. A customer's CI/CD runners, developer sandboxes, staging or production environments on infrastructure Datum doesn't manage. Datum's ALB routes to these origins the same way it routes to Datum Compute: as upstream clusters reachable over the network, with the same TLS termination, path-based routing, health checking, and failover. No requirement to run on Datum Compute first. Origins can be swapped or moved without changing how clients connect.

**IP and geo filtering.** Some routing decisions happen before an origin is ever selected. A customer may need to block or allow traffic by country, region, or named IP list — sanctions compliance, content licensing, abuse mitigation — regardless of whether the request is headed to a Compute instance or a customer origin. Datum's ALB evaluates these rules at the edge, against GeoIP data kept current by Roy Kent's GeoDB, before the routing step that picks an origin ever runs. Because filtering happens ahead of origin selection, it applies the same way whether the backend is Datum Compute or a customer-managed origin.

---

## Platform Architecture

```
Internet
    │
    ▼
NLB (Cilium L4 LB)   ← Dani Rojas: TCP/UDP routing to Envoy and compute targets
    │
    ▼
ALB (Envoy L7 LB)    ← this document: HTTP routing, TLS, origin selection
    │
    ├─── Origin upstreams / delivery edge
    └─── Compute (Unikraft) via Connectors
```

Envoy is platform-managed — customers configure its behavior through Datum delivery policies, not directly.

---

## Feature Matrix

Feature numbers are stable across this document — they're referenced by the same number in Integration Details and Open Questions below, regardless of which section they're grouped into here.

*Verified 2026-07-15 via `datumctl` against live customer/demo projects (`demos-md21mk`, `tunneldemo-jh9cgl`), not just inferred from this doc's prose. Platform-wide/cluster-scope queries were forbidden for this account, so this can't rule out internal-only config outside customer-visible projects.*

### Features We Have Now

| # | Feature | Envoy Mechanism | Evidence |
|---|---|---|---|
| 14 | Path and Host Based Routing | Core Envoy routing model: `virtual_hosts`, `routes`, `match` on path/prefix/regex, `:authority` (host), headers, methods | Live `HTTPRoute` resources with active path matching, `PROGRAMMED: True` |
| 2 | Static Endpoint Mapping | EDS static resources or static `load_assignment` in cluster config | Live `Backend` resources using static FQDN endpoints (e.g. a single `hostname:port` target) — none use EDS |
| — | TLS termination (edge) | Platform-added HTTPS listener | Confirmed as a platform default: a `Gateway` declared with only an HTTP listener still gets an auto-added `default-https` listener from Datum's controller |
| — | TLS to origin (backend TLS) | `BackendTLSPolicy` | A live `BackendTLSPolicy` resource found attached to a `Backend` |
| 19 | Request Timeout Management | `timeout` and `idle_timeout` per route and cluster; `max_stream_duration` for streaming workloads; `per_try_timeout` on retry policies | Not independently confirmed as customer-configured anywhere — kept here as an Envoy built-in default, not a verified customization |

**Confirmed not in use anywhere accessible:** zero `BackendTrafficPolicy` resources exist in any of the 6 projects checked — so zone-aware routing, health check tuning, circuit breaking, outlier detection, and canary/weighted routing are not actually configured anywhere yet, consistent with #768. Zero `Backend` resources use EDS/dynamic Compute-backed endpoints — independently confirms #768 item 7 (EDS dynamic registration) hasn't shipped. Also independently verified via `datumctl explain backendtrafficpolicy.spec.loadBalancer.zoneAware` that the field exists in the live CRD schema, matching #768's claim.

### Features to Add — Datum Compute (Phase 1)

Scoped by [issue #768](https://github.com/datum-cloud/enhancements/issues/768). Note this covers **zone-aware routing** — Envoy preferring endpoints in its own zone, using Kubernetes topology labels (`topology.kubernetes.io/zone`/`region`) — which is a different mechanism from client-geography-based routing via Roy Kent's GeoDB (still tracked separately below, not covered by this ticket).

*Item numbers below are #768's own numbering, prefixed `768-` to avoid colliding with this document's stable feature numbers (1–19) used elsewhere. 768-1, 768-2, and 768-4 are isolated translator changes and can be built independently of the rest.*

| # | Feature | Description | Envoy xDS Field | Effort | Status |
|---|---|---|---|---|---|
| 768-6 | EDS Locality Enrichment | Tags each endpoint with zone/region derived from node topology labels so Envoy can make locality-aware routing decisions | `topology.kubernetes.io/zone`/`region` → `LocalityLbEndpoints.locality` | S–M | Not started — blocks 768-3 from taking effect |
| 768-7 | EDS Dynamic Endpoint Registration | Automatically adds/removes endpoints as Compute Instances scale up or down across PoPs | New controller watches `Instance` resources, publishes `ClusterLoadAssignment` via xDS EDS | L | Not started |
| 768-5 | Per-Endpoint Health Status | Marks individual endpoints healthy/unhealthy from Kubernetes readiness probe state | `LbEndpoint.health_status` from EndpointSlice conditions | S | Not started |
| 768-3 | Zone-Aware Routing | Prefers endpoints in the proxy's own zone before spilling to other zones | `common_lb_config.zone_aware_lb_config` | S | **Configured, but inert** — live in the CRD; no effect until 768-6 lands |
| 768-1 | Overprovisioning Factor | Tunes how aggressively Envoy spills to other zones before treating local capacity as exhausted | `ClusterLoadAssignment.Policy.overprovisioning_factor` | XS | Not started |
| 768-2 | Locality-Weighted LB | Distributes traffic across zones in proportion to assigned endpoint weight | `common_lb_config.locality_weighted_lb_config` + `LocalityLbEndpoints.load_balancing_weight` | S | Not started |
| 768-4 | Priority-Based Failover | Routes to lower-priority zones only when higher-priority zone health drops below the overprovisioning threshold | `LocalityLbEndpoints.priority` | S | Not started |

**Dependency resolved:** compute nodes already carry standard Kubernetes topology labels (`topology.kubernetes.io/zone`, `topology.kubernetes.io/region`), confirmed with ops — no schema decision blocks 768-6.

**Also relevant to Datum Compute, not covered by #768:**

Confirmed via `datumctl` schema inspection (`BackendTrafficPolicy.spec.loadBalancer` / `.healthCheck`, `Backend.spec.fallback`) — all available in the CRD today, none configured by any customer in the 6 projects checked (zero `BackendTrafficPolicy` resources exist anywhere accessible).

| # | Feature | Integration Point |
|---|---|---|
| 1a | Geo-Aware Upstream Selection (GeoDB) | Routing to the closest healthy Compute instance by client geography requires GeoDB integration with `weighted_clusters` or control plane — a separate mechanism from zone-aware routing above; not covered by Envoy's native GeoIP feature, which is authorization-only |
| 7 (LB type) | Load Balancer Type | `spec.loadBalancer.type`: `LeastRequest` (default), `RoundRobin`, `Random`, `ConsistentHash` (sticky routing on `SourceIP`, `Header`, or `Cookie`) |
| 6 (slow start) | Slow Start | Supported for `RoundRobin` and `LeastRequest`; ramps traffic to new endpoints over a configurable warm-up window |
| — | Endpoint Override | Header-based endpoint pinning — a request header can specify an exact `IP:Port` target; falls back to the configured LB policy if the override endpoint is unavailable. **Not previously in this matrix** — no client geography or zone concept, just direct per-request targeting |
| — | Fallback Backends | `Backend.spec.fallback` — a `Backend` marked as fallback only receives traffic when active backend health drops below 72% (overprovisioning factor 1.4x). **Possibly related to 768-4 (Priority-Based Failover)** but that's a zone-priority mechanism (`LocalityLbEndpoints.priority`) while this is backend-level — need to confirm whether these are the same underlying mechanism exposed two ways, or genuinely separate |
| 4 (active) | Active Health Checks | `spec.healthCheck`: HTTP (poll a path), TCP (send/receive payload), gRPC health check protocol |
| 11 (passive) | Outlier Detection | `spec.healthCheck` passive mode: ejects endpoints on consecutive 5xx, gateway errors, or local origin failures. **Distinct from 768-5** (per-endpoint health status from Kubernetes readiness probes) — this is Envoy's own active/passive checking, not the EndpointSlice-readiness signal |
| 9 | Circuit Breaking | Per-backend *and* per-endpoint (`perEndpoint.maxConnections`): max connections, parallel requests, pending requests, retries |
| 10 | Retry and Failover Logic | `retry_policy` per route or virtual host; configurable conditions, backoff, and retry host predicates |
| 13 | Canary and Weighted Routing | `weighted_clusters` in route config; weights sum to 100 and can be updated via xDS without restart |
| 16 | Connection Aware Routing | Consistent hashing via `ConsistentHash` LB type (see Load Balancer Type above) plus `stateful_session` filter for cookie-based session affinity |
| 3 | Dynamic Latency Based Routing | **Lowest priority.** No native real-time latency routing; control plane must update cluster priorities or weights using Nate health signals from Higgins Bus. Sequenced last because it depends on RTT measurement infrastructure that doesn't exist yet (see Integration Details below) |

### Features to Add — Future Phases (Customer Origins, IP & Geo Filtering)

#### Available in Envoy, not yet enabled

*(none identified yet — most remaining items require integration work; revisit as Customer Origins and IP/geo filtering are scoped)*

#### Needs to be built

| # | Feature | Integration Point |
|---|---|---|
| 1b | Geo Authorization (allow/block by country) | Native in Envoy Gateway 1.8.0 via `SecurityPolicy.spec.authorization.rules[*].principal.clientIPGeoLocations` + `EnvoyProxy.spec.geoIP` — listed here because turning it on requires the GeoDB distribution pipeline (Roy Kent), not because Envoy itself is missing the feature |
| 12 | Traffic Shaping and Rate Limiting | Local rate limiting native via `local_ratelimit` filter; platform-wide limits require external rate limit service via `ratelimit` filter |
| 17 | Policy Driven Routing | `ext_proc` (external processing) filter is the integration point for Datum policy engine; sovereignty, cost, and compliance constraints evaluated externally |
| 18 | Observability Hooks | Native stats, access logs, and distributed tracing built in; requires wiring to Datum's OpenTelemetry pipeline and metrics system |

---

## Integration Details

### 1. Geo Aware Routing

Geo routing covers two distinct use cases with different integration paths.

**Geo authorization (allow/block by country or region).** Envoy Gateway 1.8.0 adds native GeoIP support via `SecurityPolicy.spec.authorization.rules[*].principal.clientIPGeoLocations`, backed by a shared GeoIP provider configured in `EnvoyProxy.spec.geoIP`. This covers the geo blocking use case (Roy Kent consumer #2, Delivery Edge Proxy) without custom filter work. The GeoIP database used by Envoy Gateway needs to be sourced from Roy Kent's GeoDB and kept current. The distribution mechanism, meaning how the GeoDB replica on each PoP is made available to Envoy Gateway's GeoIP provider, is an open question.

**Geo-aware upstream selection (route to closest compute).** Routing traffic to the geographically closest compute instance is not covered by the native GeoIP feature, which is scoped to authorization rules only. This requires geo context injected into routing decisions. Two integration approaches remain possible:

- **Header injection (filter-side).** An HTTP filter performs a per-request GeoDB lookup against the PoP-local Roy Kent replica and injects headers (`X-Client-Country`, `X-Client-Region`) before the routing step. Route config matches on these headers to select the correct upstream cluster.
- **Control plane (xDS-driven).** The control plane encodes geographic topology into cluster priorities or weighted cluster configuration pushed via xDS. This gives coarser granularity, since geo is modeled at the cluster level rather than per request, but avoids per-request lookup cost.

**Datum systems involved:** Roy Kent GeoDB (geo data source), Higgins Bus (GeoDB distribution to PoPs), Datum control plane (xDS publisher), Envoy Gateway 1.8.0 GeoIP provider (for authorization use case).

**Dependency:** Roy Kent Stage 2 (Geo Blocking) covers the authorization integration. Roy Kent Stage 3 (Application Load Balancing) delivers the upstream-selection integration. See [Roy Kent Project](ip-geo-roy-kent.md).

---

### 3. Dynamic Latency Based Routing

Dynamic latency-based routing is a significant project in its own right, not a configuration exercise. It requires a measurement infrastructure, a signal distribution layer, a control plane that translates measurements into routing decisions, and Envoy integration to act on those decisions. It is sequenced after the Roy Kent Project deliberately, because Roy Kent delivers many of the foundational blocks it depends on.

**What Roy Kent provides as a foundation:**

- The Higgins Bus distribution infrastructure: the MOQT pub/sub transport that carries signals to every PoP. Latency signals will use the same relay topology and track model that Roy Kent establishes for geo data.
- PoP-local data stores and the update pipeline for pushing data from the control plane to edge nodes.
- Geographic coordinates (lat/lon) for each PoP and for client IPs, used as a distance proxy in the interim before real RTT measurements are available. Galactic VPC already uses this for initial PoP ranking (see [Roy Kent Project](ip-geo-roy-kent.md), consumer #6).
- Production validation of the end-to-end distribution path under real traffic load, before latency-based routing depends on it for higher-stakes decisions.

**What latency-based routing adds on top:**

Real latency measurement is not the same as geographic distance. A user in London routed to a Frankfurt PoP may see worse latency than a user in Frankfurt routed to Amsterdam, depending on peering, IX routing, and undersea cable paths. The latency project requires:

- **Measurement infrastructure**: probes that measure actual RTT between PoPs and between PoPs and origin upstreams. The [Nate Project](health-checks-nate.md) covers active health check latency for endpoint availability; RTT mapping at the network path level is a separate workstream not yet scoped.
- **Signal aggregation**: raw RTT samples aggregated into stable routing signals, filtered to remove noise from transient congestion or probe outliers.
- **Track namespace on Higgins Bus**: a `platform/rtt/{pop-id}` track (reserved in the Higgins Bus namespace) carrying aggregated RTT signals to consuming components.
- **Control plane translation**: the Datum control plane subscribes to RTT signals and translates them into Envoy xDS updates: cluster priorities, locality weights, or weighted cluster splits. Routing decisions operate at control plane update frequency, not per-request frequency.
- **Envoy integration**: cluster configuration updates via xDS that Envoy acts on. Envoy's built-in algorithms (least-request, ring hash) do not consume external latency measurements; the control plane must encode latency intelligence into cluster topology and weights that Envoy then applies.

The freshness gap between a change in real-world latency and Envoy acting on it (probe → Nate/RTT aggregation → Higgins Bus → control plane → xDS → Envoy) needs to be measured and evaluated against the use cases the feature is meant to serve.

**Datum systems involved:** RTT measurement infrastructure (TBD), Nate (endpoint latency signals), Higgins Bus (signal transport, `platform/rtt/{pop-id}` track), Roy Kent GeoDB (distance proxy in interim), Datum control plane (signal subscriber and xDS publisher).

---

### 12. Traffic Shaping and Rate Limiting

Envoy's `local_ratelimit` filter enforces per-Envoy-instance rate limits without external coordination. This is sufficient for per-connection or per-listener limits on a single Envoy instance but does not enforce a limit across the full fleet. Each Envoy instance has its own independent counter.

Platform-wide rate limiting (a rate limit that applies across all Envoy instances serving a tenant or route) requires the `ratelimit` filter pointing at an external rate limit service. Envoy sends a check request to the rate limit service on each request; the service maintains the shared counter and returns allow or deny. Datum needs to operate this external service or integrate with an existing one.

The scope of what requires platform-wide versus local limits needs to be determined. Protecting individual Envoy instances from connection floods is local. Enforcing a customer's API quota across the fleet is platform-wide.

**Datum systems involved:** External rate limit service (to be determined: standalone deployment or integrated into the policy engine).

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
- **Access logs:** Envoy access logs must be configured with the correct sink, either a gRPC access log service (`grpc_access_log_config`) shipping to the ingest pipeline, or a file sink consumed by a local collector agent.
- **Distributed traces:** Envoy's `http_connection_manager` supports OpenTelemetry trace export. Trace context must be propagated correctly to upstream services so traces are continuous end-to-end.
- **Routing decision context:** Where routing decisions are influenced by geo context, Nate health signals, or policy evaluation, that context should be reflected in access log fields or trace attributes so operators can understand why a request was routed to a particular upstream.

**Datum systems involved:** Datum metrics pipeline (scraping or push target), Datum log ingest pipeline, Datum tracing infrastructure (OpenTelemetry collector).

---

## Open Questions

**Geo authorization: how does Roy Kent GeoDB feed Envoy Gateway's GeoIP provider?**
Envoy Gateway 1.8.0 provides native GeoIP-based authorization via `EnvoyProxy.spec.geoIP`, but the GeoIP database it consumes must be sourced and kept current. Roy Kent distributes GeoDB snapshots to PoP-local storage via Higgins Bus. The integration question is how Envoy Gateway's GeoIP provider is pointed at the Roy Kent-managed local replica and how it picks up updates when a new snapshot is installed. Is this a file path reference that Envoy Gateway hot-reloads, or does it require a restart or xDS push?

**Geo upstream selection: filter or control plane?**
For routing traffic to the geo-closest compute instance (distinct from geo authorization), the filter-side header injection and xDS-driven control plane approaches remain open. Which approach does Datum commit to, and what is the filter implementation path if header injection is chosen (Lua, WASM, ext_proc)?

**Who owns the ext_proc service for policy evaluation?**
Policy Driven Routing depends on an external processing service that evaluates sovereignty, compliance, and cost constraints. This service needs to be low-latency (it is in the hot path for every request), highly available (a failed ext_proc service can either fail open or fail closed, and both have risks), and aware of current policy state. Which team owns this service? Does it integrate with an existing policy store, or is it a new component? What is the acceptable latency budget for a policy evaluation call?

**How does Nate signal propagation latency affect latency-based routing freshness?**
Latency signals flow from Nate probes → Nate control plane → Higgins Bus → Datum control plane → xDS push → Envoy. Each hop adds latency. For a degrading upstream, how quickly does a change in Nate's published HealthStatus translate into updated cluster weights at Envoy? Is the end-to-end propagation time acceptable for the use cases latency-based routing is intended to serve? What is the measurement target?

**Platform-wide rate limiting service: standalone or embedded in policy engine?**
Traffic shaping at fleet scale requires an external rate limit service. Datum can deploy Envoy's reference implementation (`envoy/ratelimit`) as a standalone service or embed rate limit evaluation into the same ext_proc service used for policy decisions. The former is simpler operationally but adds another service to the fleet. The latter consolidates the hot-path external calls but couples concerns. Which direction is preferred?

**How is Envoy's xDS control plane structured?**
Several features in this document depend on the Datum control plane acting as an xDS management server: it pushes EDS updates for endpoint scaling, cluster weight updates for latency-based routing, and route config updates for geo routing. Is there a single xDS control plane, or are multiple components acting as xDS sources for different resource types? Which component subscribes to Higgins Bus signals and translates them into xDS updates?

**Routing decision attribution in observability.**
When Envoy routes a request to an upstream based on geo context, health signals, or policy, that reasoning should be visible to operators in access logs and traces. What fields and attributes are required in the access log schema to reconstruct a routing decision post-hoc? Who defines this schema and ensures it is populated by all relevant filters in the chain?
