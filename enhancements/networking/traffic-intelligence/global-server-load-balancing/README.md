---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Global Server Load Balancing

Tracking issue:
[#833](https://github.com/datum-cloud/enhancements/issues/833)

- [Summary](#summary)
- [Terminology](#terminology)
- [Problem](#problem)
- [Proposal](#proposal)
  - [Three components, and how they connect](#three-components-and-how-they-connect)
  - [Key constraints](#key-constraints)
  - [Risks](#risks)
- [Design decisions](#design-decisions)
  - [VIP progression](#vip-progression)
  - [Proximity phases](#proximity-phases)
  - [Signal distribution](#signal-distribution)
  - [Interfaces](#interfaces)
- [Phasing and open items](#phasing-and-open-items)
- [Alternatives](#alternatives)

## Summary

Routing a client to the right origin requires knowing which origin is nearby,
healthy, has headroom, and is permitted to serve the request. Datum has none of
these inputs today. Most of the data plane needed to act on them already exists
— an Envoy fleet at every PoP behind an anycast VIP, configured over xDS —
but nothing computes the priorities and weights those messages carry. This
design adds a control plane that assembles origin eligibility from health,
capacity, and policy, and pushes it into the existing xDS channel. The
architecture is the GCP-shaped answer (in-path decision on an anycast
substrate) and is the direction both major clouds converged on after DNS-layer
steering proved insufficient for load-aware routing.

This document supersedes an earlier GSLB design that proposed DNS-layer PoP
steering via PowerDNS. See [Alternatives](#alternatives) for why that approach
was not adopted.

## Terminology

| Term | Expansion | What it is here |
|---|---|---|
| **BGP** | Border Gateway Protocol | Underlies anycast PoP selection |
| **ECS** | EDNS Client Subnet | Lets a resolver forward client subnet for geo lookup |
| **EDS** | Endpoint Discovery Service | xDS service carrying endpoints with locality, weight, priority |
| **EPS** | Errors per Second | ORCA field for explicit error-rate incorporation |
| **FRR** | FRRouting | FOSS BGP suite; Datum's inter-PoP underlay |
| **LRS** | Load Reporting Service | xDS service streaming load from Envoy to control plane |
| **ORCA** | Open Request Cost Agent | xDS standard for origin load reporting |
| **PoP** | Point of Presence | Datum edge site terminating client connections |
| **RTT** | Round-Trip Time | Measured latency; Phase 2 proximity signal |
| **SNI** | Server Name Indication | TLS hostname; Phase 1 cell demux key |
| **VIP** | Virtual IP | Stable anycast address a service name resolves to |
| **xDS** | Discovery Services | Streaming gRPC protocol for Envoy configuration |

## Problem

Routing considers four inputs. Proximity, health, and capacity are
**preferences** that trade against each other — a slightly more distant origin
with headroom beats a nearer one that is full. Policy is a **filter**: "this
traffic may not leave the EU" is not weighed against latency; it restricts
which origins are candidates at all. A system that models policy as a weighted
preference will violate it under saturation, exactly when the constraint
matters most. The decision therefore has two stages: filter candidates by
policy, then optimize over what remains.

The routing decision must be made before the client opens a connection, but
the facts it depends on change continuously — an origin with headroom at
connection time may have none a second later. This demands something that
**continuously** computes which origins are eligible and how loaded they are,
and **continuously** delivers that to whatever makes per-request decisions. A
standing subscription, not a lookup.

The data plane already consumes exactly this. xDS is a streaming protocol.
EDS carries endpoints with locality, weight, and health status. Priority
levels with an overprovisioning factor implement "spill to the next tier when
this one degrades." Locality-weighted load balancing implements proximity
preference within a tier. Envoy is not the gap. Nothing currently computes
what to put in those messages.

## Proposal

### Three components, and how they connect

```
                          SLOW PATH (seconds-minutes)
  +-------------+         +-------------+
  | Origin      |  watch  | Cell API    |
  | producers   |-------->| server      |
  | (Compute    |         |             |
  |  etc.)      |         |             |
  +-------------+         +------+------+
                                 | watch (federation-blind)
                                 v
                          +---------------------+
                          | Eligibility         |   <-- HA: 2-3 replicas
                          | assembler           |       per cell, leader
                          | (per cell)          |       election
                          +----+----------+----+
                               |          |
                EDS push       |          |  LRS stream
                (fast path)    |          |  (fast path)
                               v          |
                          +--------+      |
                          | Envoy  |------+
                          | (PoP A)|
                          +---+----+
                              |  ORCA (origin -> Envoy, in-band)
                              v
                          +--------+
                          | Origins|
                          | (PoP A)|
                          +--------+
```

**1. The origin resource.** A new resource describing what may be routed to:
an endpoint set, a protocol, a health check definition, locality, an optional
capacity signal source, and a balancing target with a manual derate control.
Producers populate it; the routing layer reads only it and never reaches into
producer-specific types. This is the load-bearing API decision — the
equivalent of GCP's Network Endpoint Group.

Two properties matter beyond the field list. An origin is inherently
**plural**: it names an endpoint set, not one address. And capacity fields are
**optional, and their absence is meaningful**: an origin whose load cannot be
observed is routed on health alone, not excluded. Degrading to health-only is
designed behavior, not a failure — GCP's load balancer works the same way for
endpoint types it does not operate.

<<[UNRESOLVED naming ]>> The resource name must pass the `datumctl` test:
explicable without Kubernetes vocabulary. "Origin" obscures plurality. Origin
pool, origin group, and backend are alternatives. <<[/UNRESOLVED]>>

**2. The health and capacity pipeline.** Mostly already exists; no new
transport is built. *Health* comes from Envoy active health checks (per-PoP,
correct by construction — an origin reachable from Frankfurt but not Singapore
is a real condition no global prober can see) and outlier detection (local
ejection on observed error rates). The work is aggregating per-PoP health into
a cell-wide view, not producing it.

*Capacity* flows through the xDS ecosystem's existing reporting channel.
Origins report load via ORCA — well-known fields (RPS, CPU, memory, EPS) stay
comparable across origin types; workload-specific signals (accelerator queue
depth, KV-cache occupancy) ride in the open `named_metrics` map. Envoy
aggregates ORCA per cluster and streams it to the assembler via LRS. The
assembler subscribes to ORCA directly from origins alongside consuming LRS: ORCA
carries self-reported utilization, LRS carries proxy-observed rate and error
rate. Proxy-observed signal wins where they diverge — the inversion risk
below depends on this.

<<[UNRESOLVED capacity-vocabulary ]>> Which ORCA fields at GA, and whether a
customer-defined metric can be trusted to drive routing without a proxy-side
check. <<[/UNRESOLVED]>>

<<[UNRESOLVED health-checks ]>> Customer-defined health checks (path, expected
status, thresholds) affect the origin schema and are not yet scoped.
<<[/UNRESOLVED]>>

**3. The eligibility assembler.** A new controller, one per cell, running in
the control plane. It holds the cell-wide view of origins, health, load, and
policy, and emits EDS to every PoP's Envoy in that cell.

*Inputs (two paths, no shared machinery):* the **slow path** carries intent —
origin definitions and policy constraints, watched from the cell API server,
seconds to minutes, over the federation stack. The **fast path** carries
observation — ORCA from origins, LRS from Envoy, EDS to Envoy — sub-second to
seconds over long-lived gRPC, **bypassing the federation stack entirely.**
Capacity data cannot traverse federation workspaces and stay useful; stating
this as a rule now prevents an appealing but fatal simplification later.

*What it computes:* (1) a **policy filter** removes excluded origins — this is
first, so spillover fails closed at a policy boundary; (2) **priority tiers**
ordered by proximity — same-PoP origins in priority 0, cross-PoP origins in
the same cell in priority 1, so backbone cost is only paid when local headroom
is exhausted; (3) **locality weights within tiers** by capacity headroom;
(4) Envoy's **overprovisioning factor** handles spillover among the tiers.

*The split that keeps the assembler off the hot path:* the assembler decides
**what may spill where** (tier membership, coarse weights). Envoy decides
**when** (per-request, using live local ORCA and the overprovisioning factor).
The assembler pushes EDS when tier membership or coarse weights change —
seconds. Within those constraints, Envoy balances per-request with sub-second
reactivity. "Reacts in seconds" is a property of Envoy, not the assembler.

*Per-PoP outlier detection is additive, not overridden.* Each PoP ejects
locally. The assembler consumes ejection state via LRS. If an origin is ejected
in a majority of PoPs, the assembler marks it unhealthy cell-wide. If in only
one PoP, that PoP's local ejection stands; other PoPs still route to it.

*Placement:* 2–3 replicas per cell across control-plane PoPs, with leader
election. Only the leader emits EDS. This is the standard HA pattern for xDS
control planes (Istio's istiod). It keeps single-writer consistency while
bounding SPOF to a leader-election window. On leader loss, a follower
re-establishes xDS streams; Envoy fails open on stale state during the switch,
bounded by the staleness SLO. A PoP cut off from all control-plane PoPs serves
on its last known EDS and reports staleness.

*Cell scoping:* one assembler per cell. A project lives in one cell, so all
its origins live in that cell's slices across PoPs. **Routing never crosses
cell boundaries** — that would recreate the coupling cells exist to prevent.
Cell sizing determines whether customers can spill at all, making per-cell
headroom a staff-portal concern. Failure is **fail-open**: a PoP that loses
contact with its assembler serves on its last known EDS; staleness is measured
and alertable.

### Key constraints

- **Anycast already selects the PoP.** This design decides which **origin**
  serves the request once traffic is on-net, and uses the backbone to reach
  origins in other PoPs when that is the better answer. It does not re-decide
  PoP selection.
- **Controllers are federation-blind.** The assembler reconciles against
  whatever API server it can see. Federation places it; it does not
  participate in federation.
- **No Kubernetes vocabulary reaches users.** If the origin abstraction cannot
  be named without saying "deployment" or "pod," the abstraction is wrong.
- **Never steer by withdrawing routes.** Route presence means *can be reached*,
  never *should receive traffic*. Eligibility lives in EDS.

### Risks

| Risk | Symptom | Mitigation |
|---|---|---|
| Capacity signal inversion | Origin sheds load → CPU flat → balancer thinks it's efficient and sends more traffic | Prefer proxy-observed signal (LRS) over self-reported (ORCA); count EPS explicitly; treat shedding origins as saturated |
| Routing↔autoscaling feedback | Nearest origin grows → attracts more traffic → concentration → single failure overloads everything | Minimum capacity floors per location; scale on observed capacity, not raw instance metrics |
| Stale routing state | PoP loses contact with assembler, serves on outdated table | Staleness is a first-class metric with explicit budget; alertable |
| Policy-bound exhaustion | Saturated cell returns errors while another cell has headroom | Notification before exhaustion; staff portal surfaces compressed headroom |

## Design decisions

### VIP progression

The user-facing contract is the **name**, never the VIP. If customers never
bind to an address, relocating a service is a DNS change invisible to clients.

- **Phase 0:** One cell per PoP, one global anycast VIP. Trivially correct.
- **Phase 1:** Multiple cells, one global VIP. Traffic is demuxed by SNI/Host
  header. Honest technical debt — recorded with an exit condition.
- **Phase 2:** Cell-scoped VIPs. Each cell announces its own VIP. Warranted
  when the demux layer's blast radius becomes material.

### Proximity phases

- **Phase 1:** Geographic distance as a latency proxy. Available immediately.
- **Phase 2:** Measured RTT from Envoy's connection-establishment times.
  Triggered by evidence — instrument in Phase 1 so the upgrade decision is
  driven by data.

### Signal distribution

Per-customer routing state (eligibility, weights, health) is cell-scoped and
belongs on xDS — it already has versioning, incremental updates, and
terminates at the consumer. Platform-wide reference data (geo databases,
attack state) is broadcast-shaped and suits a pub/sub fabric if one exists.
Choosing one mechanism for both is the failure mode to avoid.

### Interfaces

The assembler exposes state as API status for all three surfaces. **Cloud
portal:** per-origin eligibility with reason codes (unhealthy, at capacity,
drained, excluded by policy, load not observable). **Staff portal:** cell/PoP
health, staleness, blast radius, policy-compressed headroom. **datumctl:**
read/write same resources; machine-readable output; preview before applying.

### Test plan

Unit: eligibility assembly as a pure function from state to EDS. Integration:
assembler through xDS to real Envoy in Kind. E2E: multi-PoP topology with
ContainerLab. Failure injection: partition (fail-open + staleness), saturation
(spillover), policy exhaustion (fail-closed), inversion (permanent regression
test).

## Phasing and open items

**Alpha.** Health-based routing, single origin type, one cell. Capacity
plumbing present but not driving decisions. Internal workloads only.

**Beta.** Capacity-aware routing and spillover. Policy constraints with
fail-closed. Portal visibility with reason codes. `datumctl` support. Multi-PoP
within a cell. Failure injection passing.

**Stable.** Multiple origin types. Staleness SLO defined and alerting. Staff
portal operator views. Documented partition and exhaustion behavior. VIP
migration path validated.

Open items that must close before the API is written: origin naming (P2),
customer-defined health checks scope (P2), ORCA field set for GA (P2),
staleness SLO budget (P2). Scale targets (origins/cell, endpoints/origin, EDS
push rate) bound the assembler design and need numbers before `implementable`.

Routing is opt-in per service. A service with no routing configuration behaves
as today. The data plane fails open, so a failed assembler rollout degrades
toward stale-but-working rather than outage.

## Alternatives

| Approach | Why not adopted |
|---|---|
| **DNS-steered GSLB** (PowerDNS GeoIP) | Binary health signal (cannot express "healthy but full"); TTLs incompatible with second-scale capacity changes; accuracy bounded by resolver ECS support; duplicates the existing Envoy+xDS layer |
| **Anycast alone** | Selects on topology, not load; cannot distinguish healthy from full; withdrawing routes conflates reachability with eligibility |
| **Policy as weighted input** | A weight safe enough is a filter with extra steps; a weight that can be traded off will be — under saturation, exactly when the constraint matters |
| **Pub/sub fabric for all signals** | Contradicts cell isolation; duplicates xDS/LRS/ORCA which already provide versioning and incremental updates |
| **Cell-local routing with global overlay** | Reintroduces cross-cell coupling; cross-cell component becomes fleet-wide failure domain |
