# Galactic VPC — Underlay Network Requirements

**Reference design:** [BGP Control Plane Architecture — PR #705](https://github.com/datum-cloud/enhancements/pull/705)

---

## Purpose

This document defines the underlay network requirements that the BGP control plane design (PR #705) implicitly depends on but does not specify. That design makes a set of hard assumptions about the physical and logical substrate beneath it. Those assumptions are load-bearing: if the underlay fails to satisfy them, the overlay breaks in ways that are difficult to debug and harder to fix post-deployment.

This is a requirements document, not a deployment guide. It defines *what* the underlay must provide, not *how* to provide it.

---

## Foundational Constraints

These are non-negotiable. Every requirement below flows from them.

1. **The underlay provides full reachability between all overlay locations.** The overlay is built on top of the underlay — it does not build its own transport fabric. The underlay must provide routed IP reachability between every PoP pair, every RR node, and every worker node that participates in the overlay. How the underlay achieves this is its concern; that it achieves this is the overlay's requirement.

2. **IPv6 reachability is required.** The data plane is SRv6 (RFC 8986). SRv6 SIDs are IPv6 addresses. Every node in the overlay must be reachable via IPv6. The underlay may run dual-stack; that is an underlay implementation choice. IPv4-only underlay is not compatible with SRv6.

3. **SRv6 encapsulation adds header overhead.** Every forwarding path must be engineered for SRv6 SRH overhead. This is the primary driver of MTU requirements throughout.

4. **The overlay must never peer with the underlay for reachability.** The overlay operates its own control plane and derives reachability from it. The underlay's responsibility is to provide IP connectivity as a transport substrate — its routing protocol, topology, and internal state are opaque to the overlay. If the overlay must participate in the underlay's routing protocol to reach another overlay node, the architectural boundary between the two layers has broken down. This must be treated as a design failure, not a workaround.

---

## IP Addressing

### Underlay Address Space

The underlay must provide IPv6 addressing for all overlay infrastructure. All PoP infrastructure (RRs, worker nodes, fabric interfaces) must have IPv6 addresses routable within the underlay.

**Requirements:**

- A dedicated IPv6 prefix allocated exclusively for Datum infrastructure (not tenant use)
- Sufficient space to accommodate: all PoP loopbacks, all RR loopbacks at each tier, all inter-PoP point-to-point links, and meaningful growth headroom — the overlay is expected to expand and the address plan must not be a constraint on that
- The block must be sub-divided per PoP in a way that allows prefix-level filtering — a PoP-per-prefix structure (e.g., one /48 or /56 per PoP) is the minimum requirement
- The underlay may also carry IPv4 addressing for infrastructure nodes; this is an implementation choice and does not relieve the IPv6 requirement

### Loopback Addressing

Every overlay node requires a stable, routable IPv6 loopback address. All BGP sessions in the RR hierarchy peer to loopbacks, not interface addresses — peering to interface addresses means a link failure tears down unrelated sessions.

**Requirements:**

- Every RR node (global tier and regional tier) must have a /128 IPv6 loopback from the infrastructure block, reachable from all nodes it peers with
- Every worker node running GoBGP must have a /128 IPv6 loopback reachable from all RR nodes in its regional pair
- Loopback addresses must be stable across reboots and control plane restarts
- The underlay must advertise these loopbacks and ensure they remain reachable as long as the node is operational

### SRv6 Locator Space

Locators are not underlay addresses, but they must be reachable via the underlay — the overlay control plane (BGP IPv6 Unicast SAFI) advertises them, and the data plane encapsulates packets with SIDs drawn from locator prefixes. The underlay address block and the locator block must not overlap.

**Requirement:** Locator prefixes and infrastructure prefixes must be drawn from non-overlapping address space with no ambiguity in routing policy.

---

## Reachability Requirements

The overlay requires full IPv6 reachability between all participating nodes. The underlay is responsible for providing it. This section defines the specific reachability pairs and their properties; the mechanisms the underlay uses to achieve them are not constrained here.

### Full Mesh Reachability — All Overlay Locations

The overlay is organized into a hierarchy: worker nodes at the edge, regional RR clusters in the middle, and a global RR tier at the top. This hierarchy exists to manage control plane scale — it does not limit reachability. Every overlay location must ultimately reach every other overlay location, regardless of which tier or region it belongs to. There are no exceptions — partial reachability produces partial tenant connectivity, which is an outage.

**Requirements:**

- Full IPv6 reachability between every PoP in the overlay, regardless of region or tier
- Reachability must be available before any overlay service on a PoP is considered operational
- The underlay must not impose artificial constraints on which PoPs can communicate — any-to-any reachability is the baseline
- As the overlay grows (new PoPs, new regions, new tiers), the underlay must extend reachability to new locations without restructuring existing paths

### Intra-PoP: Worker → Regional RR

Each worker node must reach both RR nodes in its regional pair. This is the minimum control plane connectivity for a worker to be operational.

**Requirements:**

- Full IPv6 reachability between worker node loopbacks and regional RR loopbacks within the same region
- No NAT, no stateful firewall, no middlebox on the path between worker and RR
- TCP port 179 must not be filtered in either direction on any path between worker and RR
- Reachability must be established before galactic-operator attempts BGP session bring-up
- Intra-region paths should not unnecessarily transit inter-PoP backbone infrastructure

### Inter-Region: Regional RR → Global RR

Regional RR pairs must reach all global RR nodes. The global tier must be simultaneously reachable from all regional pairs — this is the path that carries inter-region routes, and its availability determines whether tenants in one region can reach tenants in another.

**Requirements:**

- Full IPv6 reachability between every regional RR loopback and every global RR loopback
- TCP port 179 must not be filtered on any inter-PoP path
- Loss of a single inter-PoP path must not take down all sessions from a regional cluster to the global tier — path diversity is required (see Transport section)
- Each regional cluster must have independent paths to the global tier; a failure in one region's connectivity to the global tier must not affect other regions

### Global Tier Mesh

Global RR nodes form an iBGP full mesh with each other. Every global RR node must have direct reachability to every other global RR node.

**Requirements:**

- Full IPv6 reachability between all global RR loopbacks
- These paths cross the inter-PoP backbone and must be independently diverse from regional-to-global paths where possible
- Loss of reachability between any two global RR nodes degrades inter-region route propagation; the underlay must treat these paths as high-priority

---

## MTU Requirements

This is where SRv6 designs consistently bite operators. Get this wrong and you get silent black-holing of large tenant flows with no obvious failure signal.

### SRv6 Overhead Budget

A single SRv6 encapsulation without a Segment Routing Header (SRH) — i.e., SRv6 with a single SID in reduced encapsulation — adds a 40-byte IPv6 outer header. With an SRH carrying additional SIDs, overhead increases by 8 bytes per additional SID plus 8 bytes of SRH base header.

For the purposes of this design (single-segment End.DT4/DT6 per-VRF forwarding), the minimum overhead is 40 bytes. Policy-based steering with multiple SIDs in the SRH can exceed this.

**Conservative overhead budget: 80 bytes** (accommodates reduced encapsulation + one additional SID in SRH).

### End-to-End MTU Requirement

**Requirement: All inter-PoP transit paths must support a minimum MTU of 9080 bytes.**

Derivation:
- Standard tenant Ethernet MTU: 9000 bytes (jumbo frames) or 1500 bytes (standard)
- Target inner MTU: 9000 bytes
- SRv6 overhead budget: 80 bytes
- **Required wire MTU: 9080 bytes minimum**

If the fabric cannot support jumbo frames end-to-end, the fallback minimum is:
- Inner MTU: 1500 bytes
- SRv6 overhead: 80 bytes
- **Required wire MTU: 1580 bytes**

Standard 1500-byte Ethernet MTU is insufficient for SRv6-encapsulated tenant traffic that itself carries 1500-byte payloads. This is not negotiable — either the fabric supports jumbo frames, or tenants are silently forced to a lower effective MTU.

### Internet Transit PoPs — Fragmentation and MSS Clamping

Some PoPs may have only Internet transit connectivity, limiting the practical wire MTU to 1500 bytes. In these cases, the inner tenant MTU cannot be 1500 bytes without exceeding the wire MTU after SRv6 encapsulation. **Inline IPv6 fragmentation must not be relied upon as the solution.**

IPv6 requires the originating node — not intermediate routers — to handle fragmentation (RFC 8200). "Inline fragmentation" at the SRv6 encapsulating node is stateful, bypasses hardware forwarding on most silicon, and introduces latency and CPU load that is unacceptable in the forwarding path. Any vendor claim of line-rate hardware fragmentation must be backed by qualification data under realistic SRH depth and traffic mix before being trusted in production.

The correct approach for Internet-transit-only PoPs:

1. **MSS clamp TCP to 1360 bytes** at the tenant attachment point. Using 1360 (rather than 1400) provides headroom for worst-case SRv6 encapsulation depth — a conservative 80-byte SRv6 overhead budget plus additional SIDs that may be inserted by policy — without assuming that clamp is applied after all encapsulation decisions are finalized.

2. **ICMPv6 PTB generation and forwarding is mandatory.** For non-TCP flows (UDP, QUIC), PMTUD is the only mechanism available. ICMPv6 Packet Too Big messages must be generated at the point of encapsulation and must traverse the underlay unfiltered to reach the originating host. This is already a hard requirement (see Path MTU Discovery below); it is especially load-bearing on Internet transit paths.

3. **Internet transit PoPs are a degraded MTU operating mode.** They must be tracked as a distinct class in the underlay inventory. MSS clamping and reduced inner MTU must be applied automatically based on per-PoP path MTU capability — not configured manually on a case-by-case basis. The operational target is to migrate all PoPs to jumbo-capable connectivity; running indefinitely on Internet transit MTU must require explicit acknowledgment.

### Path MTU Discovery

**Requirement: ICMPv6 "Packet Too Big" (PTB) messages must not be filtered anywhere in the underlay.**

PMTUD depends on ICMPv6 PTB traversal. Filtering ICMPv6 in the underlay — a common misconfiguration — causes PMTUD black-holes that are exceptionally difficult to diagnose in production. This is a hard operational requirement, not a suggestion.

### Intra-PoP MTU

**Requirement:** All links between worker nodes and their local RR pair must match or exceed the inter-PoP MTU. There must be no MTU cliff between intra-PoP and inter-PoP paths.

---

## Redundancy and Failure Domain Requirements

### Per-PoP Underlay Redundancy

The BGP control plane design requires two independent RR sessions per worker node. The underlay must not introduce a common failure mode that kills both sessions simultaneously.

**Requirements:**

- Each worker node must have at least two independent uplink paths to the PoP fabric — one for each RR session
- The two RR nodes in each regional pair must be on independent physical infrastructure (different failure domains: separate power domains, separate switching planes, or separate physical devices)
- Loss of a single underlay link or device must not simultaneously sever both RR sessions from a worker

### Inter-PoP Path Diversity

The inter-PoP backbone carries regional-to-global RR sessions. These sessions carry inter-region routes for all tenants. A single inter-PoP failure must not black-hole inter-region route propagation.

**Requirements:**

- At least two diverse inter-PoP paths between any PoP pair carrying RR traffic
- "Diverse" means no shared physical layer — different fiber routes, different carrier where possible, or different SRLG
- This requirement applies to all paths between regional PoPs and the PoPs hosting global RR nodes, as well as between global RR nodes themselves
- The underlay must be able to reroute around a single inter-PoP path failure without operator intervention

### Global RR Tier Availability

The PoPs hosting global RR nodes carry elevated operational importance: their loss degrades — though does not immediately break — inter-region route propagation for the entire overlay. The underlay must reflect this.

**Requirements:**

- Every PoP hosting a global RR node must have redundant inter-PoP connectivity to all regional RR anchor PoPs
- Loss of inter-PoP connectivity from one global RR PoP must not isolate the global tier — the remaining global RR nodes must still reach all regional clusters
- This elevated redundancy requirement applies to the role (global RR host), not to specific current deployments — it follows the function wherever it is placed

---

## Convergence Timing Requirements

The BGP timer recommendations in the RR design (10s keepalive / 30s hold) set the floor for what the underlay must support. Underlay convergence that outlasts the BGP hold timer triggers session failure, full RIB rebuild, and significantly extends recovery time.

### Failure Detection

**Requirement:** The underlay must detect and reroute around a link or node failure within **30 seconds** without application-layer intervention.

This is derived from the BGP hold timer (30s). If the underlay takes longer to reroute, BGP sessions fail on top of the physical failure, compounding recovery time.

**Preferred:** Underlay failure detection and rerouting within 1 second, enabling BFD-assisted BGP failure detection (300ms × 3 = 900ms). BFD is a backlog item in the RR design; the underlay must not preclude it.

### BFD Timer Calculation

BFD timers must be set per link class, not globally. A BFD detection interval shorter than the link's round-trip time produces false positives on healthy links — this is worse than slow detection. The minimum interval must satisfy:

```
min_interval ≥ ceil(RTT_ms / 1000) + jitter_margin_s
detection_time = min_interval × multiplier (multiplier = 3)
```

The following per-class values are the recommended baseline. These must be tuned if measured RTTs differ materially from the ranges shown:

| Link class                | Typical RTT | min-interval | Multiplier | Detection time |
|---------------------------|-------------|--------------|------------|----------------|
| Intra-PoP                 | < 1 ms      | 50 ms        | 3          | 150 ms         |
| Regional (same continent) | 5–30 ms     | 100 ms       | 3          | 300 ms         |
| Intercontinental backbone | 80–200 ms   | 300 ms       | 3          | 900 ms         |
| Internet path overlay     | 50–500+ ms  | 1000 ms      | 3          | 3 s            |

**Internet path overlay:** BFD on Internet paths cannot reliably achieve sub-second detection. Jitter, asymmetric routing, and path variation produce false positives at aggressive timer values. The Internet overlay failure detection strategy must be multi-mechanism: BFD at the relaxed timers above for signaling, BGP hold timer as the hard backstop, and BGP Graceful Restart awareness to distinguish planned from unplanned loss. Do not attempt to tune Internet-path BFD to match backbone behavior — the false positive rate will exceed the benefit.

Backbone underlay BFD timers are the primary fast-failure signal and should achieve detection well below the 30-second BGP hold timer floor on all backbone paths. Reroute completion — not just detection — must satisfy the 30-second requirement.

### Reachability Establishment

**Requirement:** Full IPv6 reachability between a newly-connected PoP and all other PoPs must be established within **60 seconds** of the PoP's underlay connectivity being physically present.

This bounds the time from physical connectivity restore to full tenant service restore. The overlay control plane cannot begin converging until the underlay provides reachability.

---

## Transport Requirements — Inter-PoP

These requirements apply to the network that connects PoPs to each other. The specific technology (DWDM, IP transit, MPLS, IXP) is out of scope; the properties the transport must provide are not.

**Requirements:**

1. **IPv6 forwarding.** The inter-PoP fabric must forward IPv6 packets natively. IPv6-over-IPv4 tunneling is permitted only as a transitional measure and must not be the long-term state — it introduces asymmetric MTU behavior and complicates PMTUD across the SRv6 data plane.

2. **No deep packet inspection or proxy behavior.** Overlay control plane sessions must traverse transport transparently. Middleboxes that inspect, intercept, or modify TCP streams are not compatible with this design.

3. **Sufficient MTU.** Inter-PoP links must carry the required 9080-byte MTU (or 1580-byte minimum). This requirement must be confirmed with all transport providers before production deployment.

4. **Consistent DSCP preservation.** Control plane and data plane traffic may be DSCP-marked differently. The transport must not strip or re-mark DSCP values.

5. **Path diversity as described above.** Transport providers must commit to physical diversity between redundant paths, not just logical diversity.

---

## DNS and Service Reachability

RR nodes and worker nodes will require DNS resolution for operational tooling — telemetry pipelines, health checks, operator automation. None of this is part of the overlay control plane data path, and the overlay will function without it. However, operating the overlay without it is significantly harder: tooling breaks, observability gaps appear, and debugging becomes manual.

This section is a strong recommendation, not a hard requirement.

- Infrastructure nodes should have access to internal DNS resolvers reachable from all PoPs over IPv6 (IPv4 additionally if needed)
- DNS must not be a dependency for overlay control plane session establishment — BGP peer configuration uses IP addresses, not hostnames, and must continue to work if DNS is unavailable
- Where DNS is not available at initial PoP deployment, a plan to provide it should exist — running indefinitely without it is an operational liability

---

## Security Requirements

### Infrastructure Isolation

**Requirement:** RR node addresses and worker node loopbacks must not be reachable from tenant networks. The underlay infrastructure address space must be filtered at all tenant handoff points.

This is a hard requirement. An RR that is reachable from tenant networks is an attack surface against the entire control plane.

### Filtering at PoP Edge

**Requirement:** Underlay infrastructure prefixes must not be advertised to external peers (transit providers, IXP route servers, peering partners). Prefix filtering at all external BGP sessions must drop the infrastructure block in both inbound and outbound directions.

**Exception — loopback reachability for transit-only PoPs:** Router loopbacks must be reachable from the public Internet to support interoperability with transit-only sites and failover to Internet-based paths. This reachability must be provided by advertising a **covering aggregate prefix** — not individual /128 loopbacks.

The correct approach:
- Each PoP's loopback space must be contained within a PoP-level aggregate (e.g., a /48 or /56 per PoP, drawn from the infrastructure block)
- Only the aggregate is advertised externally — individual /128s must not appear in the global DFZ
- The aggregate must be originated from the PoP where those loopbacks reside; if the PoP loses connectivity, the aggregate withdraws naturally
- The specific prefixes permitted for external advertisement must be an explicit, enumerated list maintained as an operational invariant — "infrastructure prefixes filtered except loopbacks" expressed only in prose is insufficient and will be misapplied

### Control Plane Protection

**Requirement:** All routers with publicly reachable loopbacks must be protected by an aggressive Control Plane Policing (CoPP) policy.

CoPP requirements:
- BGP (TCP port 179) must be rate-limited to explicitly configured peer addresses only — not just rate-limited globally. Sessions from unknown sources must be dropped at the control plane, not just throttled.
- BFD, ICMPv6, and management traffic must be in separate CoPP queues with independent rate limits. A flood of one must not starve the others.
- All other traffic to the control plane must be dropped or severely rate-limited by default.
- CoPP policy must be validated as part of PoP commissioning. A publicly reachable loopback without a validated CoPP policy is an open attack surface against the control plane.

---

## Observability Requirements

The underlay must expose enough telemetry for the overlay to be operated effectively. Blind infrastructure is not an option at global scale.

**Requirements:**

- **Streaming telemetry via gNMI/gRPC** for all underlay devices carrying RR traffic. SNMP polling is insufficient at this scale and prohibited for new deployments.
- **Interface-level counters** (in/out packets, drops, errors) at minimum 30-second polling intervals, specifically on the interfaces at which overlay nodes attach to the underlay
- **MTU verification tooling** must be runnable on-demand from any PoP to confirm end-to-end MTU across inter-PoP paths

---

## Summary of Hard Requirements

The table below consolidates the non-negotiable requirements for quick reference during design review.

| Area                       | Requirement                                                                  |
|----------------------------|------------------------------------------------------------------------------|
| IP version                 | IPv6 required; dual-stack permitted                                          |
| Reachability               | Full any-to-any IPv6 reachability across all overlay PoPs                    |
| Addressing                 | Dedicated IPv6 infrastructure prefix, /128 loopbacks for all overlay nodes   |
| MTU (preferred)            | 9080 bytes end-to-end                                                        |
| MTU (minimum)              | 1580 bytes end-to-end                                                        |
| MTU (Internet transit PoP) | MSS clamp TCP to 1360; no inline fragmentation; ICMPv6 PTB required          |
| ICMPv6 PTB                 | Must not be filtered anywhere                                                |
| Path diversity             | ≥2 diverse inter-PoP paths for RR-carrying routes                            |
| RR failure domain          | Two RR nodes in each pair on independent hardware                            |
| BFD timers                 | Per link class; min-interval ≥ RTT + jitter margin; multiplier = 3           |
| Underlay convergence       | ≤30s reroute on failure (BFD path: ≤1s on backbone)                          |
| Reachability establishment | ≤60s from physical connectivity to full overlay reachability                 |
| Inter-PoP transport        | IPv6 forwarding, no DPI, no middleboxes, DSCP preserved                      |
| Infrastructure isolation   | Infrastructure prefixes unreachable from tenant networks                     |
| Loopback advertisement     | Aggregate only (per-PoP /48 or /56); no /128s in DFZ; enumerated permit list |
| CoPP                       | BGP peer ACL + per-protocol queues; validated at PoP commissioning           |
| Telemetry                  | gNMI/gRPC at overlay attachment points; no SNMP                              |


