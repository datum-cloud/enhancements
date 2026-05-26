# Total Load Balancing

**Product area:** Deliver — Edge Delivery / Load Balancing / Fraud & Traffic Mgmt  
**Status:** Early definition

---

## The Concept

Traffic Intelligence should operate like "Total Football," the revolutionary strategy popularized by the Dutch national teams of the 1970s led by Johan Cruyff and AFC Ajax. Instead of locking players into rigid positions, Total Football allowed every player to fluidly move across the field, adapting dynamically to where they could create the most value. The concept later became a defining and winning philosophy in Ted Lasso, where flexibility, coordination, and shared awareness transformed the team.

We believe modern networking should evolve the same way. Traffic intelligence signals such as geography, latency, ASN, congestion, sovereignty, health, and risk should not be trapped inside a single appliance or isolated service. These capabilities should move fluidly throughout the platform, becoming accessible anywhere decisions are made — across DNS, Anycast ingress, Layer 3 and 4 load balancing, Envoy Proxy routing, tunnels, AI gateways, and UFO workloads.

We refer to this model as **Total Load Balancing** — a network architecture where routing and intelligence operate as a coordinated system rather than a collection of isolated features.

---

## Problem

Datum currently uses anycast to direct users to the edge. Anycast is topology-driven: BGP selects the nearest PoP by AS-path, and the user lands wherever the network delivers them. This works well for basic connectivity but has no awareness of what's on the other side of that PoP.

There is no mechanism today to make deliberate routing decisions based on where a user actually is, what regulations apply, how congested a path is, what compute resources are available, or whether an AI model is warm and ready. Every system — DNS, load balancer, proxy, WAF, AI gateway — makes decisions independently and in isolation, with no shared intelligence.

Total Load Balancing closes that gap: a signal-driven decision layer that makes routing information available to every part of the platform, so each component can act on shared awareness rather than local guesswork.

---

## What Total Load Balancing Is

Total Load Balancing is the decision layer that sits between inbound traffic and Datum's edge infrastructure. It collects, maintains, and distributes signals about users, paths, and infrastructure — then makes those signals available to every system that needs to make a routing or policy decision.

The output at any given component is a **decision**: which PoP, which upstream, allow or block, which inference endpoint — and why.

Signals are distributed to consumers using [Higgins Bus](signal-distribution-higgins-bus.md) — a QUIC-native pub/sub transport (MOQT) that carries signals as named tracks to every edge PoP. Each signal type occupies its own track namespace, so relay infrastructure established by the Roy Kent Project scales to carry all future signal traffic without architectural changes.

---

## Load Balancing Stack

Datum uses a two-layer load balancing architecture. Both layers consume Total Load Balancing signals to make informed routing decisions.

| Layer | Technology | Customer-Configurable | Scope |
|---|---|---|---|
| **L4 (transport)** | [Dani Rojas / Cilium](l4-load-balancing-dani-rojas.md) | For compute targets | Routes TCP/UDP traffic to UFO Compute or other customer compute; platform-managed in front of Envoy |
| **L7 (application)** | [Zava / Envoy](envoy-routing-zava.md) | Via delivery policies | HTTP/HTTPS routing, TLS termination, origin selection, header manipulation |

See [Dani Rojas](l4-load-balancing-dani-rojas.md) for the Cilium design, customer configuration model, and relationship to the L7 layer. See [Zava](envoy-routing-zava.md) for the full Envoy routing feature map and integration details.

---

## Signals

Total Load Balancing is built from a growing set of signals, introduced across projects:

| Signal | Project | Description |
|---|---|---|
| **Geography** | Roy Kent | Country, region, city, lat/lon derived from client IP |
| **ASN** | Roy Kent | Autonomous System Number — identifies carrier, cloud provider, peering relationship |
| **IP Type** | Roy Kent | Residential, datacenter, proxy, VPN, satellite |
| **Health** | [Nate](health-checks-nate.md) | Availability, latency, and throughput of candidate PoPs and endpoints — prevents routing to degraded or unreachable targets |
| **RTT** | TBD | Round-trip time to candidate edge locations — first real latency signal |
| **Packet Loss** | TBD | Loss rate on candidate paths — distinguishes congestion from distance |
| **Congestion** | TBD | Link utilization at candidate PoPs and upstream |
| **Sovereignty** | TBD | Data residency and legal jurisdiction rules — hard constraints on path selection |
| **Risk** | TBD | Reputation score of source IP/ASN — feeds fraud and bot decisions |
| **Model Locality** | TBD | Where the required inference model is currently loaded |
| **Compute Availability** | TBD | Utilization and capacity of GPU, CPU, and DPU resources at candidate compute nodes |

---

## Routing Decision Vision

The end state is a layered decision hierarchy applied to every traffic flow:

1. **Policy-driven** — sovereignty and compliance rules evaluated first; a non-compliant path is eliminated regardless of performance
2. **Latency-aware** — within compliant candidates, paths are scored on RTT, packet loss, and congestion
3. **AI-aware** — for inference-bound traffic, model locality and GPU availability determine the final selection

---

## Projects

| Project | Signals / Scope | Status |
|---|---|---|
| [The Roy Kent Project](ip-geo-roy-kent.md) | Geography, ASN, IP Type | In progress |
| [The Nate Project](health-checks-nate.md) | Health (Availability, Latency, Throughput) | Early definition |
| [Jamie Tartt](gslb-jamie-tartt.md) | GSLB / DNS — geo + health-aware PoP steering | Early definition |
| [Zava](envoy-routing-zava.md) | L7 routing — geo, health, policy, protocol | Early definition |
| TBD | RTT, Packet Loss, Congestion | Not started |
| TBD | Sovereignty, Risk | Not started |
| TBD | Model Locality, GPU Availability | Not started |

---

## Distribution Transport

Signals are distributed platform-wide using **Higgins Bus** — a QUIC-native publish/subscribe transport (MOQT) where each signal type is a named track. Consumers subscribe to the tracks they need; relay infrastructure fans out updates to all edge PoPs.

Each Total Load Balancing project extends the track namespace:

| Signal Group | Track Namespace | Project |
|---|---|---|
| Geography, ASN, IP Type | `platform/geodb/version`, `org/{id}/lists/{id}` | Roy Kent |
| Health | `platform/health/pop/{pop-id}`, `platform/health/endpoint/{endpoint-id}`, `org/{id}/health/{check-id}` | [Nate](health-checks-nate.md) |
| RTT, Packet Loss, Congestion | TBD | TBD |
| Sovereignty, Risk | TBD | TBD |
| Model Locality, Compute Availability | TBD | TBD |

Track namespaces are additive — a new signal type requires no changes to existing relay infrastructure or existing subscribers. See [Higgins Bus](signal-distribution-higgins-bus.md) for the full transport design.

---

## Related Areas

- **[Dani Rojas](l4-load-balancing-dani-rojas.md)** — Cilium as the transport-layer LB; customer-configurable for compute targets
- **Connectivity / gVPC** — private path selection uses the same signal set
- **Interconnect / Tunnels** — sovereign path enforcement extends into the Connect layer
- **Observability / Metrics** — RTT and loss signals are sourced from and feed back to the Manage layer
