---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Global Server Load Balancing

Tracking issue:
[#833](https://github.com/datum-cloud/enhancements/issues/833)

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [Prior Art: Two Proven Architectures](#prior-art-two-proven-architectures)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed](#infrastructure-needed)

## Summary

This enhancement proposes how Datum routes a client to the right origin: a
control plane that continuously assembles origin eligibility from health,
capacity, and policy, and pushes it into the Envoy fleet already running at
every PoP.

Most of the data plane exists. Envoy runs at each edge, configured over xDS by
the network services operator, behind an anycast VIP, on an FRR-based backbone.
What is missing is an abstraction describing what may be routed to, and a
control plane that turns live signal into routing state. That is the scope of
this work.

This document supersedes the earlier GSLB design in this directory, which
proposed DNS-layer PoP steering via PowerDNS. The
[Alternatives](#alternatives) section explains why that approach is not
adopted.

## Motivation

The user-facing requirements are established in
[#833](https://github.com/datum-cloud/enhancements/issues/833) and are not
restated here. This document addresses how to satisfy them.

### Goals

- Establish an origin abstraction that decouples routing from the systems that
  produce routable endpoints.
- Define how health, capacity, and policy combine into a routing decision, and
  where each signal comes from.
- Fit the control plane to Datum's cell-based architecture so that its blast
  radius is one cell.
- Reuse the xDS control and data plane already deployed rather than
  introducing a parallel mechanism.
- Define failure behavior explicitly, including partition, staleness, and
  exhaustion.

### Non-Goals

- Ingress into Datum's network. Anycast VIP announcement and the underlay are
  foundational concerns this work builds on.
- Expressing capacity by withdrawing network reachability. See
  [Notes/Constraints/Caveats](#notesconstraintscaveats).
- DDoS detection, scrubbing, or attack-state steering. GSLB consumes an
  attack-state signal if one exists; producing it is separate work.
- IP-to-geography data sourcing and distribution. This design consumes a geo
  database; selecting and distributing one is separate work.
- Defining how individual workloads produce a capacity signal. This design
  establishes how such signals are consumed.

### Prior art: two proven architectures

Routing a client to the right origin is a solved problem with two divergent
industry answers. They disagree on one thing: *where the decision lives.*

**AWS puts the decision in DNS.** Route 53 latency-based routing infers the
client's location from the resolver's IP, consults a measured-latency database,
and returns the region expected to be fastest. Health checks are a bolt-on that
removes a failed region; they are binary, and weighted-routing weights are
static. The decision is made once, at resolution time, and encoded in an answer
the client caches.

That model's ceiling is visible in AWS's own catalog. Because Route 53 controls
only the *initial* resolution — packets then cross the public internet on their
own — AWS sells a second product, Global Accelerator, that advertises static
anycast addresses from every edge and steers in-path. DNS alone stopped being
enough once the requirement grew past picking a region.

**GCP puts the decision in the path, on an anycast substrate, and rejects DNS
load balancing explicitly** — because it depends on clients correctly expiring
cached records, which they do not reliably do. A single anycast VIP lets DNS
TTLs be *raised* rather than lowered, and a control plane behind that VIP
decides capacity twice: a frontend with headroom, then a backend with headroom.
Where Google does not operate an endpoint and cannot observe its load, that
endpoint degrades to health-only routing rather than being excluded.[^managing-load]

The divergence is the instructive part. AWS began at DNS and added an in-path
anycast product; GCP began on the far side of the same boundary. Both converged
on the same line: DNS is enough to pick a region, and not enough to make a
fresh, load-aware decision that stays correct after the client has cached an
answer.

**Datum's infrastructure already is the GCP-shaped substrate.** The anycast VIP
is live, Envoy runs at every PoP, and xDS is the deployed channel into it.
Choosing the in-path answer is therefore not the more ambitious of two bets; it
adopts the architecture the platform is already most of the way toward, and
declines to build the DNS-steering layer that AWS itself treats as only half the
answer. [Alternatives](#alternatives) returns to the DNS-steered design in
detail.

## Proposal

### Four inputs, but only two kinds

Routing considers proximity, health, capacity, and policy. It is tempting to
treat these as four terms in one scoring function. They are not the same kind
of thing.

Proximity, health, and capacity are **preferences**. They trade against each
other. A slightly more distant origin with headroom beats a nearer one that is
full, and the routing layer's job is to make that trade well.

Policy is a **filter**. "This client's traffic may not leave the EU" is not a
factor weighed against latency; it restricts which origins are candidates at
all. A system that models it as a heavily weighted preference will violate it
precisely when it matters most — under saturation, when the weights holding
traffic in bounds are the same weights being overwhelmed.

The decision therefore has two stages: filter candidates by policy, then
optimize over what remains. Everything downstream follows from that ordering.

### The decision is made early and must be revised often

A routing decision has to be made before the client opens a connection. That
is what makes this a *global* load balancing problem rather than a local one.
But the facts it depends on change after the decision is made. An origin with
headroom at connection time may have none a second later.

This produces the central architectural requirement: something must
**continuously** compute which origins are eligible and how loaded they are,
and **continuously** deliver that to whatever makes per-request decisions. Not
a lookup at connection time. A standing subscription.

### The data plane already consumes exactly that

xDS is a streaming protocol. EDS carries endpoints with locality, weight, and
health status. Priority levels with an overprovisioning factor already
implement "spill to the next tier when this one degrades." Locality-weighted
load balancing already implements proximity preference within a tier.

Envoy is not the gap. Nothing currently computes what to put in those
messages.

### Therefore

Three things are needed:

1. An **origin** resource describing what may be routed to.
2. A **health and capacity pipeline** that observes origins.
3. An **eligibility assembler** that combines those with policy and emits EDS.

The rest of this document is a consequence of getting those three right under
Datum's specific constraints.

### User Stories

#### A service running in three regions

A customer enables routing on a service and receives a stable name. Clients
reach the nearest region that is healthy and has headroom. When one region's
origins fill, traffic shifts to the next-nearest without intervention, and the
cloud portal shows it happening.

#### A data residency obligation

A customer declares that EU-originating traffic may only be served from EU
origins. When EU capacity is exhausted, requests fail rather than being served
from Virginia. The portal distinguishes "at capacity" from "excluded by
policy" so the customer can tell which constraint bound them.

#### An operator during a PoP incident

A PoP degrades. The staff portal shows which cells and customers are affected
and whether routing has already shifted traffic away. The operator drains the
PoP and observes where load moved.

#### An automated rollout

A platform team derates one origin to ten percent from CI, observes error
rates, and either proceeds or restores — without a browser, and without
needing to know Kubernetes is involved.

### Notes/Constraints/Caveats

**Anycast already selects the PoP.** Datum announces an anycast VIP, so BGP
delivers a client to a nearby PoP before any application-layer decision is
made. A DNS layer that also tries to select a PoP is either redundant or
requires abandoning anycast for per-PoP unicast, giving up anycast's DDoS
absorption and fast failover. This design does not re-decide PoP selection. It
decides which **origin** serves the request once traffic is on-net, and uses
the backbone to reach origins in other PoPs when that is the better answer.

**Controllers must be blind to federation.** Per the
[federation enhancement](../../../datum/federation/README.md), controllers are
built as though federation is not present. The eligibility assembler
reconciles against whatever API server it can see. It does not know it runs in
a cell, does not know Karmada exists, and does not reason about other cells.
Federation places it; it does not participate in federation.

**No Kubernetes vocabulary reaches users.** `datumctl` is explicitly not a
Kubernetes CLI, and user-facing text must not reference Kubernetes-native
resource types. If the origin abstraction cannot be named and explained
without saying "deployment," "endpoint slice," or "pod," the abstraction is
wrong. This constrains API design, not only help text.

**The overlay is not the transport foundation.** Inter-PoP transport is the
FRR-based underlay. Galactic is an origin *attachment mode* — an origin may be
reachable as a private VPC endpoint rather than a public address — not the
path this design depends on.

**Never steer by withdrawing routes.** Galactic is virtual wiring, and you do
not unplug wires to do traffic engineering. Route presence means *can be
reached*, never *should receive traffic*. Conflating them puts an application
concern into a routing protocol, where it converges slowly, flaps, and cannot
express degrees. Eligibility lives in EDS.

### Risks and Mitigations

#### A capacity signal can invert under failure

Google's published account describes a load balancer reading CPU utilization
from containers to estimate fullness. One region reached its load-shedding
threshold first and began returning errors instead of processing requests. CPU
stayed flat while requests were rejected, so per-request CPU cost *fell* — and
the balancer concluded that region was roughly ten times more efficient and
sent it more traffic. It settled serving mostly errors.[^managing-load]

This is the most important failure mode to design against, because the system
fails toward the broken component rather than away from it.

Mitigations: prefer signals the proxy observes over signals the origin
reports; count errors explicitly against an origin rather than assuming they
surface in utilization; treat a shedding origin as saturated by definition.
Any capacity metric added later must be evaluated against this failure before
adoption.

#### Routing and autoscaling form a positive feedback loop

Routing to the nearest origin grows it, growth attracts more traffic, and the
cycle repeats until capacity concentrates in one location whose failure
overloads everything else.[^managing-load]

Mitigations: minimum capacity floors per location so failover headroom
survives; autoscaling driven by the routing layer's observed capacity rather
than raw instance metrics, so unhealthy instances are discounted rather than
dragging the average down and suppressing scaling entirely.

#### Stale routing state is silent by default

A PoP that loses contact with its assembler keeps serving on its last known
routing table. That is the correct behavior, but it means traffic flows on
outdated information with no local symptom.

Mitigation: staleness is a first-class metric with an explicit budget,
surfaced in the staff portal and alertable. Fail-open is a decision, not an
accident, and must be observable.

#### Policy plus saturation produces user-visible failure

Once routing can fail closed at a policy boundary, capacity exhaustion stops
being "somewhat higher latency" and becomes errors.

Mitigation: exhaustion notification is a requirement rather than a
nice-to-have, and the staff portal surfaces where policy constraints are
materially compressing available headroom — before a traffic spike finds it.

[^managing-load]: Google SRE Workbook, chapter 11, "Managing Load."
    <https://sre.google/workbook/managing-load/>

## Design Details

### The request path, concern by concern

The scope of this work is easiest to see by following a request from client to
origin. It passes through nine distinct concerns, each of which must be solved
somewhere. Walking them in order shows that six are already solved or nearly
free with what is deployed, and that the new work is exactly three — all of
them control plane.

Only free and open-source options are named. The network services operator is
AGPL-3.0, so component licensing is a real selection constraint, noted where it
bites.

**1. Getting packets to the right PoP.** *Solved.* Anycast delivers a client to
a topologically near PoP, and the FRR-based underlay carries traffic between
them. The BGP announcer (Cilium today) is expected to change without affecting
the layers above. The known cost is that anycast selects on topology rather
than latency and can reset in-flight TCP on route flap; Google's "stabilized
anycast" mitigates both but has no off-the-shelf FOSS implementation. *FOSS:
FRRouting, BIRD, GoBGP for BGP; Cilium, Katran, MetalLB for packet-level
balancing.*

**2. Terminating the connection and selecting an origin.** *Solved.* Envoy runs
at each edge over xDS. It already provides priority levels with an
overprovisioning factor (spill to the next tier), locality-weighted balancing
(proximity within a tier), outlier detection, circuit breaking, and retry
budgets. The gap is not Envoy's capability — nothing computes the priorities
and weights it consumes, which is concerns 3 through 6.

**3. Knowing what is healthy.** *Nearly free.* Envoy active health checking runs
per PoP, which is correct by construction: an origin reachable from Frankfurt
but not Singapore is a condition no single global prober can see. Outlier
detection ejects on observed errors without a probe. The remaining work is
aggregating per-PoP health into a cell-wide view; see [Health](#health). *FOSS:
Envoy native checks, Prometheus Blackbox Exporter for vantage points outside the
proxy path.*

**4. Knowing what is busy.** ***New.*** Health is binary; capacity is not, and a
healthy origin with no headroom is still the wrong answer. This is the signal
that separates capacity-aware routing from geo-DNS, and the one nothing provides
today. ORCA carries load from origins and LRS streams it from Envoy back to the
control plane; see [Capacity](#capacity). *FOSS: ORCA, LRS, Envoy client-side
weighted round robin, or Prometheus with prometheus-adapter.*

**5. Assembling and distributing the global view.** ***New.*** Health and
capacity are observed per PoP, but spillover needs a view across PoPs. The
distribution channel exists — the xDS server — but the assembly does not.
Concretely: drive EDS from live aggregated state rather than static config,
scoped per cell; see [Eligibility assembly](#eligibility-assembly) and
[Cell scoping](#cell-scoping). *FOSS: `envoyproxy/go-control-plane`, likely
already a dependency.*

**6. Describing an origin.** ***New.*** Everything above needs a stable,
producer-independent answer to "what may be routed to?" This is the
load-bearing API decision — the equivalent of GCP's Network Endpoint Group; see
[The origin abstraction](#the-origin-abstraction). *FOSS precedent worth
studying rather than reinventing: Gateway API's `BackendRef` and `Backend`, and
`gateway-api-inference-extension`, whose endpoint selection on model-server load
is close to the accelerator scenario.*

**7. Reacting when nothing has headroom.** *Available, with coupling
requirements.* Routing can only choose among what exists; when nothing has
capacity, autoscaling or shedding has to give. Both must be driven by
routing-layer observation rather than raw instance metrics — otherwise zombie
instances drag the average down and shedding hides the inversion in concern 4 —
per [Risks and Mitigations](#risks-and-mitigations). *FOSS: KEDA, HPA with
prometheus-adapter, Envoy overload manager and adaptive concurrency.*

**8. Seeing what happened.** *Available, needs one addition.* Generic metrics
are native to Envoy; the missing piece is per-origin eligibility *with a
reason* — unhealthy, at capacity, drained, excluded by policy, or load not
observable. It is the item most likely to be deferred and most needed during an
incident; see [Interfaces](#interfaces). *FOSS: Prometheus, OpenTelemetry,
Grafana.*

**9. Name resolution.** *Solved, with a smaller role than expected.* DNS maps a
name to a stable anycast VIP with a long TTL, precisely because the VIP does not
move; see [What DNS is still for](#what-dns-is-still-for). Steering at this layer
would reintroduce a geo-database licensing constraint — MaxMind GeoLite2 and its
peers are not permissive FOSS — that anycast avoids entirely.

| # | Concern | Status |
|---|---|---|
| 1 | Getting packets to the right PoP | Solved — anycast, FRR underlay |
| 2 | Terminating and selecting an origin | Solved — Envoy, xDS |
| 3 | Knowing what is healthy | Nearly free — Envoy native |
| 4 | Knowing what is busy | **New** — ORCA, LRS |
| 5 | Assembling and distributing the view | **New** — per-cell assembler to EDS |
| 6 | Describing an origin | **New** — the load-bearing API decision |
| 7 | Reacting when nothing has headroom | Available — KEDA, Envoy overload manager |
| 8 | Seeing what happened | Available — needs eligibility reasons |
| 9 | Name resolution | Solved — name to stable VIP, long TTL |

Six of nine are solved or nearly free with what is already deployed. The new
work is concerns 4, 5, and 6 — and all three are control plane, not data plane.

### The origin abstraction

An origin describes something that may be routed to: an endpoint set, a
protocol, a health check definition, locality, an optional capacity signal
source, and a balancing target with a manual derate control.

Producers populate it. Compute controllers derive origins from workload
placement. Other origin types populate it their own way. **The routing layer
reads only this resource and never reaches into producer-specific types.**
This is what makes the capability compute-motivated without being
compute-limited, and it is the load-bearing API decision in this document.

Two properties matter beyond the field list.

An origin is inherently **plural**. It names a set of endpoints, not one
address. Distributing across those endpoints is therefore part of this
problem, not adjacent to it — particularly under spillover, where an origin
receiving overflow must spread it sensibly.

The capacity fields are **optional, and their absence is meaningful**. An
origin whose load cannot be observed is routed on health alone rather than
excluded. Stated as a principle:

> Health is universal across origin types. Capacity is a property of origins
> the platform can observe.

GCP's load balancer works the same way: balancing mode, target capacity, and
capacity scaler are unsupported for endpoint types Google does not operate.
Degrading to health-only is designed behavior, not a failure.

<<[UNRESOLVED naming ]>>
The resource name must pass the `datumctl` test: explicable without Kubernetes
vocabulary. "Origin" is the house term and reads well in customer-facing
prose, but it obscures plurality — an origin is a set of endpoints, and the
name suggests one place. Alternatives worth weighing include origin pool,
origin group, and backend. The user-facing model needs settling before the API
is written.
<<[/UNRESOLVED]>>

### Health

Envoy's active health checking runs from each PoP, which produces per-PoP
health for free and correctly. An origin reachable from Frankfurt but not
Singapore is a real and common condition that a single global prober cannot
observe. Outlier detection complements active probing by ejecting origins on
observed error rates without a probe.

The work is aggregating per-PoP health into a cell-wide view, not producing
it.

<<[UNRESOLVED health-checks ]>>
Customer-defined health checks — a customer specifying the path, expected
status, and thresholds for their own origins — are a distinct product need
that this design does not yet cover. It may be an extension of the origin's
health check definition, or a separate capability that GSLB consumes. Worth
deciding before the API is written, since it affects the origin schema.
<<[/UNRESOLVED]>>

### Capacity

**ORCA** is the transport. It is the xDS-ecosystem standard for load
reporting, carrying CPU utilization, memory utilization, application
utilization, QPS, EPS, request cost, and an open `named_metrics` map, reported
either in response trailers or over a periodic out-of-band stream.

That open map resolves the tension between a generic vocabulary and
workload-specific signals. Well-known fields stay comparable across all
origins; a workload with an unusual definition of "full" — accelerator queue
depth, KV-cache occupancy — reports it in `named_metrics`, and the routing
layer applies it by policy. One system, not one per workload type.

**LRS** carries load statistics from Envoy back to the control plane, per
cluster and per locality. It is the aggregation channel, and it already
exists.

Per the inversion risk above, proxy-observed signals are preferred where both
are available, and error rate is incorporated explicitly rather than assumed
to appear in utilization.

<<[UNRESOLVED capacity-vocabulary ]>>
Which well-known ORCA fields are supported at GA, and what policy governs
interpretation of `named_metrics` — in particular whether a customer-defined
metric can be trusted to drive routing without a proxy-side check.
<<[/UNRESOLVED]>>

### Eligibility assembly

The assembler holds the cell's view: which origins exist, their health per
PoP, their load, and the policy constraints that apply. It emits EDS —
endpoints with locality, weight, priority, and health status — to the Envoy
fleet.

Policy is applied first, as a filter on candidates. Proximity and capacity
then determine priority tiers and weights within the filtered set. Envoy's
overprovisioning factor handles spillover among the tiers the assembler
defines, which means **the assembler decides what may spill where, and Envoy
decides when.** That split keeps per-request decisions local and fast while
keeping policy decisions central and auditable.

**Spillover fails closed at policy boundaries.** Origins excluded by policy
are not placed in a lower priority tier; they are absent from the candidate
set entirely. When no eligible origin can serve, the result is a defined error
with a distinct reason code, surfaced as policy-bound exhaustion rather than a
generic failure.

### Cell scoping

Datum targets a
[cell-based architecture](../../../datum/federation/README.md), where a cell
is a vertical slice through the infrastructure rather than a location: Cell A
exists in every PoP where its infrastructure is deployed. Projects are
assigned to cells.

This determines the assembler's scope. Because a project lives in one cell,
all of that project's origins live in that cell's slices across PoPs. **The
assembler needs a cell-global view, not a fleet-global one** — and because a
cell already spans PoPs, cross-PoP spillover works inside a cell with no
cross-cell machinery.

One assembler per cell. Its blast radius is one cell, which is what cells are
for.

**Invariant: routing never crosses cell boundaries.** If a customer's origins
are saturated within their cell, that is capacity exhaustion. It is not
spillover into another cell, which serves different customers. Crossing cells
would recreate the coupling cells exist to prevent. The consequence is that
cell sizing determines whether customers can spill at all, making per-cell
headroom a staff-portal concern rather than an implementation detail.

**Failure behavior is fail-open.** A PoP that loses contact with its cell's
assembler retains its last known routing table and continues serving. Stale
state beats no state; the mitigation is that staleness is measured and
alertable rather than silent.

<<[UNRESOLVED assembler-placement ]>>
Where the assembler runs. A cell spans PoPs, so it is not naturally a per-PoP
workload. Candidates: a single instance per cell in a control-plane location,
serving long-lived xDS streams to every PoP in that cell; or a per-PoP
instance with a cell-wide state channel between them. The first is simpler and
has an obvious consistency story but adds WAN dependency to config delivery.
The second survives partition better at the cost of a distributed state
problem.
<<[/UNRESOLVED]>>

### Fast path and slow path

Two paths, different latency budgets, no shared machinery.

The **slow path** carries intent: origin definitions, policy constraints,
balancing configuration. It flows through the API server hierarchy and the
federation stack. Seconds to minutes is acceptable.

The **fast path** carries observation: ORCA from origins into Envoy, LRS from
Envoy to the assembler, EDS from the assembler to Envoy. Sub-second to
seconds, over long-lived gRPC streams, **bypassing the federation stack
entirely.**

Capacity data cannot traverse federation workspaces and propagation policies
and remain useful. Stating this as a rule now prevents an appealing but fatal
simplification later.

### Signal distribution

Signals split by scope, and the split determines the mechanism.

**Per-customer routing state** — origin eligibility, weights, health,
capacity — is cell-scoped and belongs on xDS. It is already the deployed
mechanism, it has versioning and incremental update semantics, and it
terminates at the component that consumes it. Broadcasting it platform-wide
would both leak tenant state across cells and create a fleet-wide dependency
of exactly the shape cells exist to avoid.

**Platform-wide reference data** — geo databases, attack state per PoP — is
genuinely broadcast-shaped: one publisher, every consumer, low frequency,
identical payload. A pub/sub fabric is a reasonable fit, and this design
consumes such signals if they exist rather than specifying their transport.

The failure mode to avoid is choosing one mechanism for both. A fabric sized
for reference data will not carry per-customer state safely, and a
config-delivery protocol is a poor broadcast bus.

### VIP progression

VIPs start global and become cell-scoped when the complexity is warranted.
What makes that evolution cheap is a rule adopted from day one:

> The user-facing contract is the name, never the VIP.

No portal, document, API response, or CLI output presents a VIP as the thing
to depend on. If customers never bind to an address, relocating a service to a
different VIP is a DNS change invisible to clients.

**Phase 0 — one cell, one VIP.** At Launch there is a single cell per PoP. A
global anycast VIP requires no demultiplexing. Trivially correct.

**Phase 1 — multiple cells, global VIP.** Traffic arrives on the shared VIP
and is demultiplexed to the correct cell's Envoy fleet by SNI or Host header.
This is honest technical debt: the demux layer is a shared failure domain that
partially defeats cell isolation at ingress. Acceptable as an interim state,
and recorded as debt with an exit condition rather than allowed to become
permanent.

**Phase 2 — cell-scoped VIPs.** Each cell announces its own VIP. A service's
name resolves to its cell's VIP. The demux layer is removed and ingress
isolation matches the rest of the architecture.

Phase 2 is warranted when cell count exceeds one *and* either the demux
layer's blast radius becomes a material fraction of incidents, or a customer
requires isolation guarantees a shared ingress cannot provide.

### Proximity, and the latency-measurement progression

The goal is the origin closest in *latency*, which is not the origin closest
on a map or in network topology. Anycast delivers packets to a topologically
near PoP, which is a good first approximation and sometimes wrong.

**Phase 1 uses geographic distance as a latency proxy.** Available
immediately, right most of the time. The cost is real and should be stated
plainly: paths exist that are geographically short and slow, and this approach
will choose them.

**Phase 2 uses measured RTT.** Envoy already observes connection establishment
times per locality. That data becomes the input to proximity ranking,
replacing distance.

The trigger for Phase 2 is evidence: proximity decisions that measurement
would have made differently, visible once per-locality latency is
instrumented. Instrumentation therefore comes first, in Phase 1, so the
decision to upgrade is driven by data rather than intuition.

### What DNS is still for

DNS maps a service name to a stable VIP with a long TTL. Because the VIP does
not move, the TTL can be long — the opposite of the tradeoff a DNS-steered
design is forced into. This is also what makes the VIP progression possible.

One case genuinely favors DNS-layer steering and is deliberately out of scope
here: moving clients away from a PoP under active attack *before* their
packets arrive. An in-path design must absorb that traffic to redirect it. If
attack-state steering is built, DNS is the right layer for it, and it should
be designed with the DDoS work rather than folded into GSLB, whose
proximity-and-capacity decision is a different problem on a different
timescale.

### Interfaces

The assembler holds the state all three surfaces need and exposes it as API
status, rather than each surface deriving it independently.

**Cloud portal** reads per-origin state: receiving traffic or not, and if not,
why — unhealthy, at capacity, drained, excluded by policy, or load not
observable. It writes policy constraints and derate or drain settings. Live
status is already available through the Watch API, so real-time display is a
matter of surfacing existing state.

**Staff portal** reads the operator view: cell and PoP health, staleness,
blast radius of a degraded location, and where policy constraints are
compressing available headroom.

**datumctl** reads and writes the same resources with machine-readable output,
and supports previewing a change before applying it. Traffic-shifting changes
warrant the care already given to destructive operations.

### Test plan

**Unit.** Eligibility assembly is a pure function from state to EDS and should
be tested as one: given origins, health, load, and policy, assert the emitted
endpoints, tiers, and weights. Policy filtering and fail-closed behavior are
the highest-value cases.

**Integration.** Assembler through xDS to a real Envoy in a Kind cluster.
Assert that state changes produce the intended data-plane behavior, and that
priority and overprovisioning configuration spills when and only when
intended.

**End-to-end.** A multi-PoP topology using the ContainerLab pattern already
established for the SRv6 work, with clusters standing in for PoPs.

**Failure injection**, where the real risk lives:

- Partition a PoP from its assembler; assert fail-open and reported staleness.
- Saturate an origin; assert spillover and no traffic to the saturated origin.
- Saturate all policy-eligible origins; assert failure rather than boundary
  crossing.
- Induce the inversion scenario — an origin shedding load while holding
  utilization flat — and assert it is treated as saturated rather than
  efficient. This is a permanent regression test against a documented failure
  mode.

### Graduation criteria

**Alpha.** Health-based routing for a single origin type within one cell.
Capacity plumbing present but not driving decisions. Internal workloads only.

**Beta.** Capacity-aware routing and spillover. Policy constraints enforced
with fail-closed behavior. Portal visibility with reason codes. `datumctl`
support. Multi-PoP within a cell. Failure injection suite passing.

**Stable.** Multiple origin types. Staleness budget defined and alerting.
Staff portal operator views. Documented behavior under partition and
exhaustion. VIP migration path validated whether or not it has been exercised.

## Production Readiness Review Questionnaire

This section is completed as the enhancement approaches `implementable`. What
is knowable now is recorded; the rest is marked unresolved.

### Feature Enablement and Rollback

Routing is opt-in per service. A service with no routing configuration behaves
as it does today. Disabling routing returns a service to its prior behavior
and does not require control plane or node downtime.

### Rollout, Upgrade and Rollback Planning

The dominant rollout risk is that the assembler emits incorrect eligibility —
too few origins, causing artificial exhaustion, or too many, sending traffic
to origins that cannot serve. Metrics that should inform a rollback: the
proportion of origins marked ineligible, spillover rate, and origin-level
error rate.

Because the data plane fails open, a failed assembler rollout degrades toward
stale-but-working rather than toward outage. A rollback restores the previous
assembler; Envoy converges on the next EDS push.

### Monitoring Requirements

An operator can determine the feature is in use by the presence of origins
with routing enabled and by active EDS subscriptions.

A customer can tell it is working from origin status — which origins are in
the answer set and why the others are not — surfaced in the cloud portal and
through `datumctl`.

Signals that matter: eligibility churn, routing state staleness per PoP,
spillover rate, policy-bound rejection rate, and per-origin fullness.

<<[UNRESOLVED slos ]>>
Concrete SLOs, particularly the staleness budget: how old a capacity reading
may be before it is worse than none, and what the data plane does at that
threshold.
<<[/UNRESOLVED]>>

### Dependencies

- **Envoy fleet and xDS control plane.** Outage means no routing state
  updates; the data plane continues on last known state.
- **Origin producers** (initially Compute controllers). Outage means origin
  definitions go stale; existing routing continues.
- **Geo database**, for the Phase 1 proximity proxy. Outage or staleness
  degrades proximity accuracy without affecting health or capacity behavior.

### Scalability

New API calls: long-lived LRS and EDS streams per Envoy instance, plus ORCA
reports from participating origins. Volume scales with origins and PoPs per
cell rather than with request rate, which is the property that makes this
affordable.

New API types: the origin resource and its policy constraints.

<<[UNRESOLVED scale-targets ]>>
Supported origins per cell, endpoints per origin, and the EDS push rate under
churn. These bound the assembler's design and need numbers before
`implementable`.
<<[/UNRESOLVED]>>

### Troubleshooting

If the API server is unavailable, the assembler cannot learn about origin or
policy changes; already-computed routing state continues to be served.

Known failure modes, with detection and mitigation:

- **Assembler unreachable from a PoP.** Detected by staleness metric.
  Mitigated by fail-open. Diagnosed from stream connection state.
- **Capacity signal inversion.** Detected by rising error rate at an origin
  whose reported fullness is falling. Mitigated by administratively draining
  the origin. Covered by the failure injection suite.
- **Policy-bound exhaustion.** Detected by policy-bound rejection rate.
  Mitigated by adding eligible capacity; there is deliberately no routing
  remedy.
- **Eligibility flapping.** Detected by eligibility churn rate. Mitigated by
  damping in the assembler.

## Implementation History

- Enhancement issue
  [#833](https://github.com/datum-cloud/enhancements/issues/833) filed,
  establishing user-facing requirements.
- This design opened for review, superseding the prior GSLB document.

## Drawbacks

**It adds a control plane to the request path's dependencies.** Envoy's
configuration is comparatively static today. After this, routing correctness
depends on a component that must keep working. Fail-open bounds the damage
without removing the dependency.

**Capacity awareness is a commitment to a hard problem.** The inversion
failure is not hypothetical, and every future capacity signal must be
evaluated against it. A health-only system would be meaningfully simpler and
would satisfy a real fraction of the requirement.

**Cell scoping caps a customer's spillover at their cell's capacity.** This is
correct — it is what isolation means — but it will produce incidents where
headroom existed elsewhere in the fleet and was deliberately not used.

**The best behavior requires origins to cooperate.** Origins that do not
report via ORCA fall back to health-only routing, so capacity awareness is
available only to workloads that opt in.

## Alternatives

### DNS-steered GSLB

Return different addresses per client based on resolver location and origin
health, implemented with PowerDNS's GeoIP backend. This was the previous design
in this directory, and it is the AWS-shaped answer from
[Prior art](#prior-art-two-proven-architectures) — Route 53 is its production
instance: identify the user from the resolver IP, consult a measured-latency
database, remove failed endpoints via health checks.

Not adopted, for four reasons in descending weight.

**The health signal available at the DNS layer is binary.** It can remove a
failed origin; it cannot express "healthy but ninety percent full." Weighted
answers exist, but the weights are static. Capacity awareness — the input that
distinguishes this work from geo-DNS — is what DNS is worst at.

**The freshness required is incompatible with the mechanism.** Capacity moves
in seconds, so answers need TTLs in seconds, which sacrifices cache efficiency
and still leaves correctness hostage to resolvers that over-cache. Google
rejected DNS load balancing for precisely this reason — it depends on clients
correctly expiring records — and noted that a single anycast VIP lets TTLs be
*raised* instead.[^managing-load]

**Its accuracy is bounded by things outside Datum's control.** Geo lookup uses
the resolver's address unless the resolver sends EDNS Client Subnet, and ECS
is not universally supported. The prior design acknowledged this: users behind
public resolvers get degraded accuracy. Anycast has no equivalent problem
because proximity is determined by where packets actually arrive.

**It duplicates infrastructure that already exists.** Envoy is deployed at
every PoP behind an anycast VIP with a working xDS channel. A DNS-steered
design leaves that in place and builds a second decision path beside it. It
also required solving several problems the prior document left open —
PowerDNS runtime update latency, GeoDB reload semantics, TTL strategy — none
of which arise here.

DNS retains the role described in
[What DNS is still for](#what-dns-is-still-for).

### Anycast alone, without capacity awareness

Let BGP deliver traffic to the nearest PoP and serve locally. Simple, fast to
fail over, no control plane.

Not adopted. BGP selects on topology rather than latency or load, cannot see
that an origin is full, and expressing saturation by withdrawing routes
conflates reachability with eligibility. This is the status quo and the reason
[#833](https://github.com/datum-cloud/enhancements/issues/833) exists.

### Policy as a weighted input

Fold jurisdiction into the scoring function by heavily penalizing
non-compliant origins rather than excluding them.

Not adopted. A weight large enough to be safe is a filter with extra steps,
and a weight small enough to be traded off will be traded off — under
saturation, exactly when the constraint matters most.

### A dedicated pub/sub fabric for all routing signals

Distribute every signal — geo data, health, capacity, attack state,
eligibility — over one platform-wide pub/sub transport, with each signal type
occupying a named track fanned out to every PoP. This was the direction of the
prior work in this directory.

Not adopted for per-customer routing state, for two reasons.

**It contradicts cell isolation.** Fanning tenant-specific eligibility to
every PoP creates a fleet-wide shared dependency and distributes state across
cell boundaries. The blast radius of the distribution layer becomes the whole
fleet, which is the shape cells exist to prevent.

**It duplicates purpose-built protocols already deployed.** xDS, LRS, and ORCA
exist for exactly this problem, are already carrying Envoy configuration at
Datum, and provide versioning, incremental update, and consistency semantics
that a general-purpose bus would have to reimplement.

The narrower claim survives: platform-wide reference data is broadcast-shaped
and a fabric suits it. See [Signal distribution](#signal-distribution).

### Cell-local routing with a global overlay

Route within a cell normally, with a fleet-wide system that shifts traffic
between cells under pressure.

Not adopted. It reintroduces the coupling cells exist to eliminate, and the
cross-cell component becomes a fleet-wide failure domain. If a cell is
chronically short of headroom, the answer is cell sizing, not a routing
mechanism that borrows from neighbors.

## Infrastructure Needed

None beyond what exists. The assembler runs as a workload in the control plane
alongside other operators. Test infrastructure reuses the existing ContainerLab
multi-cluster topology.
