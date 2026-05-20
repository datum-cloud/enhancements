# Traffic Intelligence

**Product area:** Deliver — Edge Delivery / Load Balancing / Fraud & Traffic Mgmt  
**Status:** Early definition

---

## Total Load Balancing

Traffic Intelligence should operate like "Total Football," the revolutionary strategy popularized by the Dutch national teams of the 1970s led by Johan Cruyff and AFC Ajax. Instead of locking players into rigid positions, Total Football allowed every player to fluidly move across the field, adapting dynamically to where they could create the most value. The concept later became a defining and winning philosophy in Ted Lasso, where flexibility, coordination, and shared awareness transformed the team.

We believe modern networking should evolve the same way. Traffic intelligence signals such as geography, latency, ASN, congestion, sovereignty, health, and risk should not be trapped inside a single appliance or isolated service. These capabilities should move fluidly throughout the platform, becoming accessible anywhere decisions are made — across DNS, Anycast ingress, Layer 3 and 4 load balancing, Envoy Proxy routing, tunnels, AI gateways, and UFO workloads.

We refer to this model as **Total Load Balancing** — a network architecture where routing and intelligence operate as a coordinated system rather than a collection of isolated features.

---

## Problem

Datum currently uses anycast to direct users to the edge. Anycast is topology-driven: BGP selects the nearest PoP by AS-path, and the user lands wherever the network delivers them. This works well for basic connectivity but has no awareness of what's on the other side of that PoP.

There is no mechanism today to make deliberate routing decisions based on where a user actually is, what regulations apply, how congested a path is, what compute resources are available, or whether an AI model is warm and ready. Every system — DNS, load balancer, proxy, WAF, AI gateway — makes decisions independently and in isolation, with no shared intelligence.

Traffic Intelligence closes that gap: a signal-driven decision layer that makes routing information available to every part of the platform, so each component can act on shared awareness rather than local guesswork.

---

## What Traffic Intelligence Is

Traffic Intelligence is the decision layer that sits between inbound traffic and Datum's edge infrastructure. It collects, maintains, and distributes signals about users, paths, and infrastructure — then makes those signals available to every system that needs to make a routing or policy decision.

The output at any given component is a **decision**: which PoP, which upstream, allow or block, which inference endpoint — and why.

---

## Signals

Traffic intelligence is built from a growing set of signals, introduced in phases:

| Signal | Phase | Description |
|---|---|---|
| **Geography** | 1 | Country, region, city, lat/lon derived from client IP |
| **ASN** | 1 | Autonomous System Number — identifies carrier, cloud provider, peering relationship |
| **IP Type** | 1 | Residential, datacenter, proxy, VPN, satellite |
| **RTT** | 2 | Round-trip time to candidate edge locations — first real latency signal |
| **Packet Loss** | 2 | Loss rate on candidate paths — distinguishes congestion from distance |
| **Congestion** | 2 | Link utilization at candidate PoPs and upstream |
| **Sovereignty** | 3 | Data residency and legal jurisdiction rules — hard constraints on path selection |
| **Risk** | 3 | Reputation score of source IP/ASN — feeds fraud and bot decisions |
| **Model Locality** | 4 | Where the required inference model is currently loaded |
| **GPU Availability** | 4 | Utilization and capacity at candidate compute nodes |

---

## Routing Decision Vision

The end state is a layered decision hierarchy applied to every traffic flow:

1. **Policy-driven** — sovereignty and compliance rules evaluated first; a non-compliant path is eliminated regardless of performance
2. **Latency-aware** — within compliant candidates, paths are scored on RTT, packet loss, and congestion
3. **AI-aware** — for inference-bound traffic, model locality and GPU availability determine the final selection

---

## Phase Documents

| Phase | Name | Status |
|---|---|---|
| 1 | [Geographic Intelligence](geo-phase1.md) | In progress |
| 2 | Network Performance Signals | Not started |
| 3 | Sovereignty and Risk | Not started |
| 4 | AI and Compute Awareness | Not started |

---

## Related Areas

- **Connectivity / gVPC** — private path selection uses the same signal set
- **Interconnect / Tunnels** — sovereign path enforcement extends into the Connect layer
- **Observability / Metrics** — RTT and loss signals are sourced from and feed back to the Manage layer
