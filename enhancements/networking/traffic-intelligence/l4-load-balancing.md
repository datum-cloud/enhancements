# Dani Rojas: Layer 4 Load Balancing

**Parent:** [Total Load Balancing](total-load-balancing.md)  
**Status:** Early definition  
**Codename:** Internal project name — not a go-to-market product name.

---

Named after Dani Rojas — the striker who doesn't overthink. No complex reads, no hesitation. He just gets the ball where it needs to go.

> "Traffic is life!"

Layer 4 load balancing works the same way. No application-layer inspection, no header parsing, no route matching. It sees an IP and a port and gets the packet to the right backend — fast.

---

Datum uses **Cilium** as its Layer 4 (transport-layer) load balancer. Cilium operates at the TCP/UDP level and provides the first load-balancing hop for inbound traffic, sitting in front of Envoy (Datum's Layer 7 load balancer) in the platform stack. Today Cilium is platform-managed and not customer-configurable. The goal is to expose it as the customer-configurable L4 load balancer for compute workloads — UFO Compute (Unikraft) and any other compute target a customer places behind Datum's edge.

---

## Platform Architecture

```
Internet
    │
    ▼
Cilium (L4 LB)   ← customer-configurable for compute targets
    │
    ├─── Envoy (L7 LB)    ← platform-managed; customers configure Envoy via delivery policies
    │        │
    │        └─── Origin upstreams / delivery edge
    │
    └─── UFO Compute (Unikraft)   ← customer-configurable L4 target
    └─── Other customer compute   ← customer-configurable L4 target
```

The L4→Envoy path is platform-managed infrastructure — Cilium's routing to Envoy is not customer-configurable. Customers who need Layer 7 features (HTTP routing, TLS termination, header manipulation) configure those through Envoy's delivery policy surface, not through the L4 layer.

The L4→compute path is customer-configurable. When a customer deploys compute workloads (UFO, VMs, containers, or other targets) behind Datum's edge, they configure Cilium directly to balance traffic across those backends.

---

## Current State

Cilium is deployed as platform infrastructure and is **not currently customer-configurable**. Traffic steering to Envoy is managed by Datum operations. There is no self-service surface for L4 configuration.

---

## Goal

Expose L4 load balancing configuration as a customer-facing product for compute use cases:

- **Basic configuration** — available in the Datum portal UI; covers common patterns without requiring CLI knowledge
- **Advanced configuration** — available via `datumctl` and the [Datum MCP](https://datum.net) for programmatic and infrastructure-as-code workflows

---

## Scope

### Customer-Configurable

Every Datum PoP that runs compute exposes an L4 load balancer option. Customers placing workloads at a PoP can configure the L4 LB as part of that deployment — it is a standard capability at any compute-enabled PoP, not a separately provisioned add-on.

| Target | Description |
|---|---|
| **UFO Compute (Unikraft)** | Unikernel workloads running on Datum's compute infrastructure |
| **Customer compute** | VMs, containers, bare metal, or any other compute the customer connects via Datum |

Configuration controls which backends receive traffic, how load is distributed across them, and how health is assessed at the transport layer.

### Platform-Managed (Not Customer-Configurable)

The routing path between Cilium and Envoy is platform infrastructure. Customers cannot modify how Cilium routes to the Layer 7 load balancer. L7 policy (routing rules, TLS termination, header manipulation, origin selection) is configured through the delivery edge product surface, not the L4 layer.

---

## Configuration Model

### Basic Configuration (UI)

The portal exposes a simplified L4 load balancer configuration surface covering the most common patterns:

| Setting | Description |
|---|---|
| **Backend pool** | Add and remove compute targets (IP:port or resource reference) |
| **Load balancing algorithm** | Round-robin, least connections |
| **Active health check** | TCP connect probe with configurable interval and timeout |
| **Passive health check** | Outlier detection — eject a backend after consecutive connection failures on real traffic |
| **Session persistence** | Source-IP affinity on/off |
| **Protocol** | TCP, UDP |
| **Listener port** | Port(s) that the L4 LB listens on |

### Advanced Configuration (datumctl / MCP)

`datumctl` and the Datum MCP expose the full configuration surface for infrastructure-as-code and programmatic workflows:

| Setting | Description |
|---|---|
| All basic settings | Full parity with UI |
| **Connection limits** | Max connections per backend, connection rate limits |
| **Advanced algorithms** | IP hash, weighted round-robin, random |
| **Active health check protocol** | TCP, HTTP, HTTPS, or custom probe type |
| **Passive health check tuning** | Consecutive failure threshold, ejection percentage, base ejection duration |
| **PROXY protocol** | Emit PROXY protocol v1/v2 headers to backends |
| **Timeout tuning** | Connection timeout, idle timeout, backend connect timeout |
| **Backend weights** | Per-backend traffic weight for weighted distribution |

Active and passive health checks are complementary. Active checks probe backends on a schedule and remove them before real traffic hits a failed backend. Passive checks observe live connection outcomes and eject backends that are failing under real load. Both can be enabled simultaneously. Active health checks at the L4 layer are distinct from [Nate](nate.md) signals — Nate publishes platform-wide health state to [Higgins Bus](higgins-bus.md) for routing decisions across the platform; L4 active checks are local to the load balancer and drive immediate backend pool membership.

---

## Relationship to Layer 7 Load Balancing

L4 and L7 load balancing serve different purposes in the stack and are configured independently:

| | L4 (Cilium) | L7 (Envoy) |
|---|---|---|
| **Layer** | TCP/UDP | HTTP/HTTPS/gRPC |
| **Routing basis** | IP:port | Path, header, host, method |
| **TLS** | Passthrough | Termination and re-encryption |
| **Customer surface** | For compute targets | For delivery/origin routing |
| **Health checks** | Transport-layer (TCP/UDP) | Application-layer (HTTP status codes, body match) |

Customers who need HTTP routing, TLS termination, or origin health checking configure those through Envoy's policy surface. L4 configuration is appropriate when the customer controls the compute and wants direct transport-level load balancing without HTTP inspection.

---

## Open Questions

- **Resource model.** What Kubernetes-native resource type represents a customer L4 load balancer? Does it align with the Gateway API (GatewayClass, Gateway, TCPRoute) or use a Datum-specific type?
- **IP assignment.** Does each L4 load balancer get a dedicated IP, or does it share a VIP with other listeners? How are IPs allocated and advertised?
- **UFO integration.** How does the L4 LB discover UFO Compute backend IPs as unikernel instances start and stop? Does it read from the UFO control plane, or does the customer configure static pools?
- **Quota limits.** How many L4 load balancers can a customer create? What limits apply to backend pool size, listener count, and connection rate?
- **Health check signals to Higgins Bus.** Should L4 health check results be published as [Nate](nate.md) signals on [Higgins Bus](higgins-bus.md), or are they local to the LB?
