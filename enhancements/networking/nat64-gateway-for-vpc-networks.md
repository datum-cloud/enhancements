---
title: NAT64 Gateway for VPC Networks
description: RFC 6146 stateful NAT64 translation for IPv6-only Galactic VPC instances reaching the IPv4 internet, delivered by generalizing the existing, proven NAT66 tier into a single combined galactic-nat binary and CRD rather than adding a second, near-duplicate one. Paired with DNS64, tracked separately.
updated: 2026-09-11 14:50
tags: [plan, srv6, nat64, nat66, dns64, evpn, ebpf, egress]
status: provisional
stage: alpha
latest-milestone: "TBD"
---

# NAT64 Gateway for VPC Networks

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Where translation lives in the data plane](#where-translation-lives-in-the-data-plane)
  - [Node-local NAT64, generalized into galactic-nat](#node-local-nat64-generalized-into-galactic-nat)
  - [Per-tenant isolation via the SRv6 Argument field](#per-tenant-isolation-via-the-srv6-argument-field)
  - [Per-tenant session limits: enforcement and collection](#per-tenant-session-limits-enforcement-and-collection)
  - [The DNS64 boundary](#the-dns64-boundary)
  - [End-to-end validation](#end-to-end-validation)

## Summary

Galactic VPC's IPv6-only instances get a private ULA address and, where
configured, an existing stateful NAT66 tier (`galactic-nat66`) gets them
to the IPv6 internet. Nothing today gets them to the IPv4 internet — a
meaningful share of real-world destinations. This document designs a
stateful, RFC 6146-compliant NAT64 translation capability, delivered by
**generalizing the existing `galactic-nat66` binary and its CRD into a
single combined `galactic-nat`**, rather than building a second,
near-duplicate binary alongside it. NAT64 and NAT66 perform the same
function — stateful egress PAT, VRF-scoped session table, port
allocation, decap/re-encap on return — for different address families;
one implementation serving both is a stronger fit than two copies of
nearly the same code, and it is isolated per tenant using the same SRv6
Argument mechanism the fabric already uses to disambiguate VRFs.

## Motivation

Datum Cloud is built on IPv6 as a foundational, forward-looking design
choice — compute workloads on the platform are IPv6-only by default.
That choice doesn't change what the rest of the internet looks like: a
substantial share of real-world destinations — legacy APIs, vendor endpoints,
CDNs and services that have never shipped an AAAA record — remain IPv4-only,
and will for the foreseeable future. Without a translation path, an
IPv6-only workload simply cannot reach them. This feature exists to
close that gap: it lets a compute workload connect outbound to an
IPv4-only service without requiring the workload itself to carry a
second address family, which would undercut the platform's own
IPv6-forward design rather than accommodate it. RFC 6146 already defines
the mechanism; nothing here is novel at the protocol level. What has to
be designed is how it integrates into Galactic's existing SRv6/EVPN data
plane and control plane, at a level of operational complexity that
matches actual, current need rather than anticipated need.

### Goals

- An IPv6-only instance reaches an IPv4-only destination with zero
  tenant-side configuration, no NAT64 prefix handling on the instance,
  and no client-side awareness that translation happened at all.
- **For the MVP, this is enabled by default for all compute
  instances** — not a per-tenant or per-region opt-in a tenant has to
  request. Availability is bounded only by where shard capacity exists,
  not by tenant choice.
- **Deliver NAT64 by generalizing the existing NAT66 tier, not by
  adding a second binary next to it.** `galactic-nat66` becomes
  `galactic-nat`: one session table, one port allocator, one CRD, one
  set of counters, serving both address families instead of two
  near-duplicate implementations. See
  [Node-local NAT64, generalized into galactic-nat](#node-local-nat64-generalized-into-galactic-nat)
  for the shape and
  [Risks and Mitigations](#risks-and-mitigations) for what changing a
  proven production binary actually costs.
- No tenant's translation table or session load can degrade another
  tenant's — enforced structurally, not by convention, via the SRv6
  Argument field already used elsewhere in the fabric to carry per-VRF
  identity. This applies identically to the IPv4 and IPv6 halves of the
  combined table.
- Per-tenant NAT64 session count and configured limit are tracked as
  local, in-datapath state. This design's concern is collection only —
  emitting or exposing that data to any higher-layer system (telemetry,
  quota, insights, or otherwise) is out of scope. See
  [Notes/Constraints/Caveats](#notesconstraintscaveats) for what that
  boundary costs.
- **Anti-spoofing / trust validation on shard ingress ships with this
  design.** Called out as its own goal, separate from everything else,
  because it's a security gap, not a resilience nice-to-have — see
  [Risks and Mitigations](#risks-and-mitigations).
- **A minimum set of operational counters is collected from day one**,
  not retrofitted later: at least per-tenant admission-failure cause
  (limit vs. shard unavailability) and per-shard port-allocation
  exhaustion, tracked as local datapath state. This is collection only —
  emitting this data anywhere is out of scope, so nothing here makes the
  gateway observable to anyone by itself. The reason to collect it now
  regardless: instrumenting the datapath after it has
  already shipped is real rework, and this is the one chance to get the
  counter shape right before `galactic-nat` is generalized and in use.
- **Existing NAT66 behavior is unaffected for tenants already using
  it.** Generalizing the binary and CRD must be provably additive — see
  [Non-Goals](#non-goals) and
  [Risks and Mitigations](#risks-and-mitigations) for how that's kept
  true rather than merely asserted.

### Non-Goals

- **Changing NAT66's existing IPv6-to-IPv6 translation behavior or
  semantics for tenants already using it.** This design extends the
  binary and CRD that implement NAT66 to also perform NAT64 — it does
  not change what NAT66 does for an existing tenant's traffic. See
  [Risks and Mitigations](#risks-and-mitigations) for how that boundary
  is kept, not just asserted.
- **Any form of session failover** — regional or cross-region.
  Translation state is strictly node-local; replicating it anywhere is
  out of scope entirely, not merely out of scope for now.
- **Inbound NAT64** (an IPv4-only client initiating a connection to an
  IPv6-only instance). Outbound-only, matching actual tenant demand.
- **Per-tenant translation prefix or customization.** One Datum-managed
  NAT64 prefix (an NSP — see [Notes/Constraints/Caveats](#notesconstraintscaveats)),
  shared fabric-wide. Per-tenant *isolation* is a hard goal; per-tenant
  prefix *choice* is not offered.
- **Any metrics emission, export, or tenant-facing surface for session
  data.** This design only collects: per-tenant session count, limit,
  admission-failure cause, and port-allocation exhaustion are tracked as
  local datapath state and nothing more. Publishing any of it —
  to a higher-layer platform system (telemetry, quota, insights), a
  tenant-facing dashboard, or anywhere else — is out of scope for this
  document.
- **DNS64 itself.** Synthesis behavior, per-zone opt-out, exclusion
  lists, and marking synthesized responses are entirely the scope of
  [#792](https://github.com/datum-cloud/enhancements/issues/792). This document defines only the boundary DNS64 reads from — see
  [The DNS64 boundary](#the-dns64-boundary).
- **Dual-stack (IPv4) instance addressing.** Galactic's CNI IPAM already
  supports allocating a private IPv4 address to an attachment, but that
  address is VPC-internal only — it has no public path and would need an
  entirely separate NAT44 tier to reach the internet, solving nothing
  this design doesn't already solve more directly. Instances stay
  single-stack; only their traffic's outer envelope changes at the
  gateway.

## Proposal

NAT64 is delivered by generalizing the existing egress translation
tier — `galactic-nat66` and its `NAT66Shard` CRD — into a combined
`galactic-nat` that performs stateful RFC 6146 translation alongside its
existing IPv6-to-IPv6 PAT, rather than adding a fifth independently-
operated datapath program to gateway-role nodes (today's four: the CNI
datapath, the tenant/EVPN BGP control plane, the ingress-only DSR
gateway, and NAT66). It is deliberately **not** folded into the
ingress-only DSR gateway: that question — fold a new translation
behavior into the existing edge XDP program, or ship it as its own
component — was already raised and answered when NAT66 was built, and
the answer is on record (see
[Where translation lives in the data plane](#where-translation-lives-in-the-data-plane)).
Combining NAT64 with NAT66 specifically is a different question from
that one, addressed on its own terms in the same section.

This design is still unambitious relative to #791's original ask in one
respect: it does not build anycast advertisement or a regional session
store, the genuinely novel, unproven piece #791 also asked for, with no
precedent anywhere in this codebase — rejected outright rather than
merely deferred, since nothing today demonstrates a need for it (see
[Risks and Mitigations](#risks-and-mitigations)). But it is not
unambitious about how NAT64 is delivered relative to NAT66: rather than
duplicate NAT66's session-table, port-allocation, and counter-collection
implementation in a second binary, this design generalizes the one that
already exists.

### User Stories

#### Story 1

As a tenant running IPv6-only compute, I want my workload to reach an
IPv4-only vendor API the same way it reaches anything else — a normal
outbound TCP connection to a hostname — without configuring anything or
even knowing translation happened.

#### Story 2

As an operator of this gateway, I want per-tenant NAT64 session count
and configured limit tracked accurately as local datapath state, so the
collected data is correct and trustworthy regardless of how or whether
it's ever read.

### Notes/Constraints/Caveats

Three facts have to stay distinct, because the whole design hinges on
none of them being confused for another:

| Fact | Example | Scope |
|---|---|---|
| **The NAT64 prefix** | a Datum-operated `/96` Network-Specific Prefix — which specific block is an implementation-time allocation decision, not a design question | One value, fabric-wide, never per-tenant |
| **The shard SID** | one SID per shard, chosen at CNI ADD, same as NAT66 today | Per-shard, assigned per tenant VRF |
| **The tenant/VRF identifier** | the existing per-attachment VRFID already carried in ordinary uSIDs today | Per-tenant, carried in the SID's Argument field, never in the prefix |

The NAT64 prefix is a Datum-operated NSP, not the RFC 6052 well-known
prefix (`64:ff9b::/96`). RFC 6052 itself recommends an NSP whenever the
operator controls both the client network and the translator, which
Datum does end-to-end here; an NSP is also the only choice consistent
with how every other special-purpose block in this fabric is addressed
— operator-owned, filterable at the fabric boundary, and divisible per
region if that's ever needed, none of which the WKP allows. DNS64
(#792) was already designed to read whatever prefix is published rather
than assume the WKP, so this costs nothing extra to integrate.

The single shared prefix and per-tenant isolation are not in tension with
each other, even though #791 and #792's language ("the tenant's NAT64
prefix") can read that way at first glance: every tenant's synthesized
addresses are drawn from the *same* prefix; what's tenant-specific is
never the prefix, only the session-table entry and the SRv6 Argument that
scopes a given flow against the right tenant's session count. DNS64
needs to know only the one shared prefix value plus a liveness signal —
it does not need, and must never be given, a per-tenant prefix to
configure.

A second constraint worth stating plainly: every compute instance is
configured for both NAT66 and NAT64 egress by default — this is not a
per-tenant or per-VPC opt-in, it's the default state every instance
gets. Since both are now the same binary, that default is a single
operational action per VRF — pointing its egress route at a
`galactic-nat` shard — rather than two separate ones; that's itself a
small simplification this design produces, though not one it was
primarily justified by.

A third point: this design only collects. It tracks per-tenant session
count and configured limit as local, in-datapath state, and nothing
more — emitting that data anywhere, to a higher-layer platform system or
otherwise, is out of scope for this document. Session-limit
*enforcement* happens locally regardless, so the limit is real even
though nothing outside the datapath can see it. That is a real, current
operational gap: a limited tenant hits a silent, unexplained connection
failure with no way for anyone, tenant or operator, to tell why. This
document leaves that gap open rather than half-solve it — collecting the
data without also solving how to see it would just be a different way
of hiding the same problem.

### Risks and Mitigations

- **This design modifies NAT66's existing binary and CRD rather than
  adding an isolated new one.** NAT66 isn't running in production
  anywhere today, so this isn't a risk to live tenant traffic — but it
  is a risk to code quality: brand-new IPv4 translation logic gets
  entangled with NAT66's own IPv6 logic inside one binary and one
  session table before either has been proven under real traffic,
  instead of NAT66 shipping and stabilizing on its own first.
  Mitigations: (1) the IPv4 code path must be additive and
  feature-gated — a shard running with no `EgressShard` IPv4 fields
  configured must be provably byte-for-byte unchanged in its IPv6
  handling, verified by running NAT66's existing test/regression suite
  unmodified against the generalized binary before any IPv4 field is
  ever set; (2) validate NAT66-only behavior end-to-end in whatever
  environment first deploys this binary before ever enabling IPv4
  fields there, rather than treating the existing test suite alone as
  sufficient proof; (3) keep the IPv4 and IPv6 code paths structurally
  separate inside the one binary (distinct table entries, distinct
  translation functions) rather than a unified code path with
  family-conditional branches throughout, so a defect in one family's
  logic has a narrower blast radius within the shared process even
  though it shares fate at the process level.
- **This design inherits NAT66's single-point-of-failure-per-shard
  limitation, by design.** A shard restart breaks every session it was
  servicing, for both address families now instead of one. An anycast
  SID plus a regional session store would fix this, but building that —
  a new, unproven, read-many/write-many store with no precedent
  anywhere in Galactic today — is a worse trade than shipping the
  proven node-local pattern first, for a resilience property nothing
  today has demonstrated a need for. This is an accepted, documented
  tradeoff, not planned as follow-on work; revisit if this limitation
  proves to matter in practice.
- **Per-flow limit enforcement on the hot path.** Checking a per-tenant
  session count against a limit on every new flow risks adding latency
  to connection setup. Mitigation: keep the live counter in a per-VRF,
  node-local eBPF map, checked synchronously — there's no shared or
  regional state to stay consistent with, so this stays simple.
- **DNS64 bypass is inherent, not fixable here.** An application that
  hardcodes an IPv4 literal, or resolves through a DNS server outside
  the platform's control, never touches DNS64 and has no path through
  this gateway at all. This is a property of NAT64/DNS64 as a mechanism
  (RFC 6146/6147), not a gap in this design, but it needs to be a
  documented, user-facing constraint rather than a silent surprise.
- **Translation is inherently opaque to standard debugging.** A packet
  captured at the destination shows the gateway's address, not the
  tenant's. This design only partially addresses that: it collects the
  session-count/limit data (below), and DNS64's synthesized-response
  marking ([#792](https://github.com/datum-cloud/enhancements/issues/792))
  helps on that side — but since this design's scope stops at
  collection, none of the collected data is actually visible to anyone
  debugging a live issue. This is a known, open gap, not a solved one.
- **A publicly-reachable translation gateway is a new trust boundary.**
  The existing NAT66 tier has a documented gap here already — no
  anti-spoofing or trust validation on shard ingress. Generalizing the
  binary inherits the same exposure for both address families, which is
  why closing it is listed as an explicit [goal](#goals) above rather
  than left as an argument in this Risks section alone — a security gap
  that only lives in prose here tends to quietly slip to "later" the
  moment scope gets squeezed.

## Design Details

### Where translation lives in the data plane

Galactic already has an ingress-only XDP gateway (DSR load balancing,
public client → tenant backend) and a separate binary that does stateful
egress translation. Folding the egress mechanism into the ingress
gateway's own XDP program was explicitly proposed and explicitly
rejected when NAT66 was built — the decision is on record in that
gateway's own architecture documentation, citing the isolation benefit:
a crash or upgrade in one tier must not affect the other, and a DSR
load-balancing program and a stateful-translation program have different
risk profiles and different rollback needs. This design does not revisit
that question — it directly answers the open question in
[#866](https://github.com/datum-cloud/enhancements/issues/866) about
integration point by continuing to build egress translation alongside
the ingress gateway, not inside it.

That precedent is about ingress versus egress — a load-balancing program
and a stateful-translation program are genuinely different functions,
and the isolation argument is strong there. It is a materially different
question whether NAT64 and NAT66 — which perform the *same* function,
stateful egress PAT, VRF-scoped session table, port allocation,
decap/re-encap on return, just for different address families — should
be one binary or two. This document's answer is one: `galactic-nat66`
generalizes into `galactic-nat`, covering both. The case for combining
is the mirror image of the ingress/egress isolation argument above, not
an extension of it — NAT64 and NAT66 don't have different risk profiles
or different rollback needs the way ingress and egress do; they have the
*same* one, twice. One session-table/port-allocation implementation
instead of two near-duplicates means one eBPF program, one DaemonSet,
one CRD, and one set of counters to instrument instead of two —
directly halving the cost of the counter-collection work this design
already commits to doing from day one, and leaving one fewer
independently-operated tier on every gateway-role node going forward. The real cost of combining —
touching a proven, already-in-production binary and CRD instead of
adding an isolated new one — is real and is addressed head-on in
[Risks and Mitigations](#risks-and-mitigations), not minimized.

### Node-local NAT64, generalized into galactic-nat

Same node-local, sharded architecture NAT66 already runs in production,
with the session table and CRD generalized to cover both address
families:

- `galactic-nat66` generalizes into `galactic-nat`, running on
  designated shard nodes exactly where `galactic-nat66` runs today —
  no new node role, no new placement decision.
- The `NAT66Shard` CRD is renamed `EgressShard`, carrying a shard SID
  and the shard's public address(es) for whichever families it serves —
  IPv6 only (today's NAT66 shape), IPv4 only, or both. Because NAT66
  isn't running in production anywhere yet, this is a clean rename, not
  a migration: no conversion path, no dual-shape reconciliation window,
  no existing objects to convert.
- A tenant VRF's egress route for the shared NAT64 prefix (Datum's
  operator-assigned `/96` NSP) is installed the same way
  `EgressDefaultRouteAdd` installs NAT66's `::/0` route today — at CNI
  ADD, SRv6-encapsulating matching traffic toward the assigned shard's
  SID. A shard serving both
  families handles both kinds of route identically at this layer; the
  address-family split happens inside the shard's session table, not in
  how a VRF's route is installed.
- The shard decapsulates, performs RFC 6146 stateful translation for
  NAT64 flows and its existing IPv6-to-IPv6 PAT for NAT66 flows —
  structurally separate table entries and translation functions inside
  one binary, not a unified code path (see
  [Risks and Mitigations](#risks-and-mitigations)) — forwards over
  ordinary internet routing, and reverses the mapping on the return
  path.
- Shard assignment is static, chosen at CNI ADD time, same as NAT66
  today: "first resolvable SID wins" is an accepted limitation of this
  design, not a regression relative to what NAT66 already ships with.
- **Critically, NAT64 traffic must never be routed as IPv4-in-IPv6
  (IPIP-encapsulated).** The existing dual-stack IPv4 IPAM path tags
  IPv4-inner packets with an IPIP outer header, and NAT66's existing
  datapath only matches an IPv6-in-IPv6 outer header — an IPv4-inner
  packet routed at a shard today is silently dropped. NAT64 avoids this
  entirely by construction: because the synthesized destination address
  is itself within Datum's NAT64 NSP, an ordinary IPv6 address, this
  traffic always takes the same IPv6-in-IPv6 SRv6 outer encapsulation every
  other pod-to-pod flow uses. This has to be enforced structurally —
  NAT64 traffic must never be reachable via the dual-stack IPv4
  instance-address path — not left as an implicit assumption, and it
  holds regardless of whether NAT64 ships as its own binary or
  generalized into `galactic-nat`.

### Per-tenant isolation via the SRv6 Argument field

Galactic already reserves an Argument portion of the SID for per-VRF
identity in ordinary instance-attachment uSIDs — a node's locator, a
fixed function code, and the attachment's VRFID are combined into every
attachment's own /128 SID today. This design reuses that pattern for the
shard SID: `<locator>:<function code>:<VRFID>` — the locator component
identifies a specific shard node, the same as NAT66's shard SIDs do
today, with the function code distinguishing a NAT64 destination from a
NAT66 destination on the same shard where both are configured.

A gateway node decapsulating a packet destined to this SID reads the
tenant's VRFID directly out of the packet's own destination address —
no separate lookup is needed to learn which tenant a flow belongs to,
the same property the existing decapsulation path already relies on for
ordinary VRF traffic. Both the session-table key and the per-tenant
session counter (next section) are scoped by that VRFID — and, within
the combined table, by address family — so two tenants can never
collide even if their instances happen to share an identical inner ULA
source address, and a tenant's NAT64 usage can never be confused with
their own NAT66 usage.

### Per-tenant session limits: enforcement and collection

Enforcement happens at flow-creation time, in the datapath, before a new
session is admitted: a per-VRF counter (a node-local eBPF map) is
checked against a configured default limit. A tenant at their limit has
the triggering SYN or first UDP packet dropped — RFC 6146's own guidance
for exhaustion behavior — rather than any existing session being evicted
to make room. One tenant hitting its own ceiling can never affect
another tenant's sessions, and can never even evict its own established
sessions to admit a new one. NAT64 and NAT66 usage count against one
shared per-tenant limit, not two separate ones: a tenant's total
translated-session count across both address families is what's checked
against their configured limit — both are just table entries scoped by
the same VRFID, and there's no reason to give them separate budgets.

This design's responsibility stops at collection: per-tenant current
session count and configured limit are tracked in the same node-local
eBPF map as the limit counter itself, and nothing further. It does not
build any emission mechanism, control-plane sweep, or tenant-facing
API/CRD — getting that data out of the datapath and into anything that
can act on it is out of scope for this document. This data is tracked
but not observable outside the shard it lives on.

### The DNS64 boundary

This design publishes one fact for DNS64
([#792](https://github.com/datum-cloud/enhancements/issues/792)) to
use, and stops there:

1. **The shared NAT64 prefix** — one value, fabric-wide, never
   per-tenant (see [Notes/Constraints/Caveats](#notesconstraintscaveats)).
   It's a static, documented value, not a piece of runtime state, so
   publishing it needs no emission mechanism.

This design does not publish a liveness or availability signal for
NAT64. Its scope stops at collection; publishing anything, a liveness
signal included, is out of scope. That means #792's own goal of
degrading safely when NAT64 is unavailable has no signal from this
gateway to check — a real dependency worth naming plainly rather than
glossing over.

No DNS64 synthesis logic, per-zone toggles, response-marking behavior, or
degradation strategy is designed in this document — all of that is
entirely #792's scope, and #792's own non-goals confirm this data path is
not where any of that logic lives.

### End-to-end validation

1. An IPv6-only instance resolves a known IPv4-only test name; confirm
   DNS64 synthesizes an address inside the shared NAT64 prefix.
2. The instance connects using that synthesized address; confirm the
   connection completes through a `galactic-nat` shard.
3. A second tenant's identical inner source address, on a different VRF,
   is confirmed to never appear in the first tenant's session count or
   collide in the session table.
4. A tenant at their configured session limit has a new connection
   attempt fail closed, without affecting any other tenant's sessions or
   their own already-established ones.
5. Run NAT66's existing test/regression suite, unmodified, against the
   generalized `galactic-nat` binary with no IPv4 fields configured;
   confirm it passes identically to how it passes against
   `galactic-nat66` today — the regression check that actually proves
   the "existing NAT66 behavior is unaffected" goal, not just the
   new-capability happy path.
