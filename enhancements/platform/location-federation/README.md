---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Locations Where the Work Actually Runs

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Four statements this design rests on](#four-statements-this-design-rests-on)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [What travels, and what stays](#what-travels-and-what-stays)
  - [Who owns publishing it](#who-owns-publishing-it)
  - [A location's lifecycle today](#a-locations-lifecycle-today)
  - [Removal: stop placing versus tear down](#removal-stop-placing-versus-tear-down)
  - [Staleness: what a consumer sees while a copy is behind](#staleness-what-a-consumer-sees-while-a-copy-is-behind)
  - [Telling "nothing to do" from "not doing anything"](#telling-nothing-to-do-from-not-doing-anything)
- [Consistency with compute](#consistency-with-compute)
- [Deferred to the technical design](#deferred-to-the-technical-design)
- [Open Questions](#open-questions)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)

## Summary

A customer picks a location and deploys into it. The platform knows which locations exist
and which services work in each — but only in one place, the control plane where the
catalog lives. The systems that actually place and run the work don't know any of it.

Two pieces of work now need them to. A network has to follow a workload into every
location it runs in, which means the layer handing out addresses at an edge location has to
know which location it is. And compute already tries to answer that question where the work
runs, against information that was never delivered there — inert today only because the
feature is switched off, and wrong the moment it's switched on.

This document describes how location and service-availability information reaches the
systems that place and run work, what each of them is entitled to know, and how the whole
thing behaves when it stops working. It stops short of naming mechanisms; those belong to a
technical design.

## Motivation

Today a location exists as a catalog entry in the platform control plane and as a piece of
hand-typed configuration on each site, with nothing reconciling the two. A service records
where it's available in a third place. None of this reaches the systems doing the placing
and the running, so:

- **A network can't follow a workload.** An address can be asked for, but not handed out,
  because the layer that would hand it out can't tell which location it's serving except by
  being told in configuration.
- **A workload doesn't know where it ran.** An instance is created without a record of its
  location, and nothing fills that in afterwards. A customer asking "where is this running"
  gets a blank.
- **Two hand-maintained copies of one fact can disagree** — the catalog's idea of a
  location and the site's own idea of itself — and nothing notices when they do.
- **The same question gets two different answers.** "Can this workload run here" is answered
  correctly in one place and re-answered, differently and against absent information, in
  another.

### Goals

- Make a location legible to the systems that place and run work, so that "which location
  is this" is something they can read rather than something they have to be told.
- Keep one answer to "can this service run here for this customer," produced once, in the
  place that can see all the inputs.
- Ensure a location going unhealthy stops *new* work from being placed there without
  disturbing the work already running there.
- Make the delivery of this information observable, so a stoppage is detected in hours
  rather than weeks.

### Non-Goals

- Handing out addresses, or a network's presence in a location. Those are covered by NSO
  #360 and #369.
- The data plane that carries the traffic.
- Changing what a customer sees or writes. This document adds no consumer-facing API.
- Settling where the Location primitive itself should ultimately live — that question is
  open in the locations product proposal and this design deliberately survives either
  answer.
- The mechanism. Which components, which resources, and how delivery is implemented are
  technical-design questions.

## Proposal

**Nothing changes for a customer.** They list locations, pick one, and deploy. What changes
is that the systems below them stop guessing.

**Nothing changes in how a service declares availability.** A service operator still records
"this service is deployed and working at this location," once, in the platform control
plane. That record is not shipped anywhere. Its *consequence* is.

### Four statements this design rests on

**1. The decision travels, not the inputs that produced it.**

Whether a service is usable at a location is a join across several facts: the service is
published, the location supports that class of service, the location is healthy, and the
customer is entitled to it. Shipping those inputs outward would mean shipping four more
kinds of record to every site and having each site work out the answer for itself. That is
one evaluator turned into dozens, each able to reach a different conclusion, with no way to
tell that they had. What leaves the platform control plane is a conclusion, not a
calculation — and a conclusion is inert wherever it lands.

**2. A site needs its own identity, not the catalog.**

The systems running work at a location need to know what that location is called and where
it is. They do not need the full list of locations, because they are not choosing between
them — by the time work arrives at a site, the choice has already been made. Delivering the
whole catalog to every site invites exactly the re-deciding that statement 1 rules out, and
still leaves each site needing to be told which entry is itself. Delivering one location per
site makes "which location am I" a question a site can answer by looking, which is what
removes the hand-typed configuration.

**3. Entitlement never leaves the customer's control plane.**

"Can *this customer's* workload run here" is answered where the customer's entitlement
lives, and nowhere else. What reaches the placement layer and the sites is not a permission,
and nothing there may treat it as one. Anything that wants to make an access decision has to
go back to the record that already makes it.

**4. One question per record.**

"Can this workload run here" and "which location is this" are different questions with
different answers and different audiences. They are currently conflated, which is how one of
them ended up being asked against information that isn't there. They stay separate.

### User Stories

**A new location comes online.** A location is registered and a service confirms it's
working there. Entitled customers see it in their location list, as they do today. The site
itself learns its own identity without anyone editing its configuration, and workloads placed
there come up with addresses. Nobody hand-edits two places and hopes they agree.

**A workload is deployed into a location.** The customer picks a city. Every instance that
comes up records where it's running, from the first one onward — not from the second
generation, and not only after something else happens to trigger a rewrite.

**A location goes unhealthy.** New work stops being placed there: the location drops out of
customers' available lists and no new deployments are created for it. Work already running
there is untouched — it keeps its identity, keeps its addresses, and keeps running. Taking
that work down is a separate, deliberate operation, never a side effect of a catalog edit.

**A service isn't available somewhere yet.** A customer sees the location, but not for that
service — the same distinction the locations proposal already draws between "never launched
here" and "was working, now broken." Nothing about delivering this outward flattens that
distinction.

**Delivery stops.** Somebody is told, in hours. Today they wouldn't be — see
[detection](#telling-nothing-to-do-from-not-doing-anything).

### Notes/Constraints/Caveats

**A site can only rely on what survives the trip.** The delivery layer between the platform
and the sites does not carry everything about a record — notably, anything a controller
computed and wrote as observed state is dropped. Anything a site must read has to be part of
what was declared, not part of what was observed. This is a real constraint on the shape of
what gets published, and it is why the published form is a statement rather than a status.

**A site's copy going stale is safe; a site never receiving one is not.** A location's
identity doesn't change, so a site running behind is a site running correctly. The failure
that matters is a *new* location whose site never receives anything — which has to present
as visible waiting that names what's missing and heals when it arrives, not as a workload
stuck for unexplained reasons.

**Removing hand-typed configuration makes a site depend on delivery.** Today a site can
answer "which location am I" even when nothing is reaching it, because someone typed the
answer in. That should stop being the primary path, but keeping it as an explicit fallback
is cheaper than the failure it prevents.

**Locations are hand-created and hand-marked healthy today.** There is no controller behind
either. Fanning a hand-authored catalog out to a fleet raises the cost of a typo
considerably. That's an argument for grounding location records in something real — as the
locations proposal already argues — not an argument against delivering them.

### Risks and Mitigations

- **Risk:** A site re-decides something that was already decided upstream, and reaches a
  different answer.
  **Mitigation:** What reaches a site is a conclusion with no inputs attached, so there is
  nothing there to re-evaluate. Access decisions are structurally absent from what's
  delivered.

- **Risk:** A catalog edit takes down running work — a location marked unhealthy, or deleted
  by mistake, removing the identity that live workloads depend on.
  **Mitigation:** "Stop placing" and "tear down" are separated deliberately, with only the
  first driven by catalog state. Removal is guarded against acting on a location that still
  has a site behind it.

- **Risk:** Delivery stops silently and nobody notices for weeks. This has already happened
  once, for 34 days.
  **Mitigation:** Detection is a requirement of this design, not a follow-up, and it has to
  be the specific kind of detection that works here — see below.

- **Risk:** Making a site's identity depend on delivery introduces a new way for a site to be
  unable to function.
  **Mitigation:** The existing configuration path is retained as a fallback rather than
  removed.

## Design Details

### What travels, and what stays

| | Stays in the platform control plane | Travels outward |
|---|---|---|
| A service's availability record | ✓ | its consequence only |
| A customer's entitlement | ✓ | never |
| The full location catalog | ✓ | never, in full |
| A location's identity and locality | | to the site that is that location |
| The decision "is this service usable here" | | to the placement layer, if and when placement moves there — not to sites |

The last row is deliberately conditional. Placement is decided today in the customer's
control plane, where entitlement already lives and where the decision is already correct.
Publishing that decision further out has no reader yet. Building the path before there's
something to read it adds a thing to keep right for no gain, so it's staged behind the
consumer that would need it.

### Who owns publishing it

The component that already projects location availability into customers' control planes
should own publishing it outward too.

That component already computes this exact decision, from all of its inputs, in one place.
A second owner would mean a second implementation of the same policy against the same
inputs — and two implementations of one policy drift, silently, surfacing only as a location
that's usable in one place and not another.

The alternative — having the networking operator own it, since it already delivers
configuration outward — inverts the dependency: the networking operator would have to
understand service catalogs and entitlements to know what to publish, which is not its
concern. That's a plumbing argument, not an ownership one. The publishing responsibility
follows the decision; the delivery plumbing can be shared.

### A location's lifecycle today

Worth stating plainly, because it's the substrate everything here builds on: **a location is
created by a person when a site is stood up, marked healthy by a person, and never removed.**
There is no controller behind any of it. Separately, the same locality fact is typed into
each site's own configuration. Nothing compares the two, and most sites carry no locality
information at all.

This design does not fix that, and shouldn't — that's the locations proposal's job. It does
mean two things:

- Publishing outward should assert the link between a location record and the site it
  claims to describe, and make a mismatch visible rather than a silent no-op.
- Filling in the missing locality information across the fleet is a prerequisite of this
  work, not a cleanup that can follow it. Compute's own placement is already partly broken
  for the same reason.

### Removal: stop placing versus tear down

These are different decisions with different triggers, and collapsing them turns a catalog
edit into an outage.

**Stop placing new work here** is driven by catalog state: the location becomes unhealthy,
the service's availability record is withdrawn, or the service stops supporting that class
of location. The location leaves entitled customers' lists and no new deployments are created
for it. This already works today and needs nothing from this design.

**Tear down what's here** is an operation, driven by deliberately draining a site. Never by a
catalog edit.

The consequence: what's delivered to a site is gated on the location *existing*, not on it
being healthy. An unhealthy location is still a real place with real workloads in it, and
those workloads still need to know where they are — most acutely at the moment the location
is already degraded. Actual deletion of a location does propagate, but is guarded against
acting while a site still stands behind it.

### Staleness: what a consumer sees while a copy is behind

- **Work already running: unaffected.** Identity and locality don't change, so a stale copy
  is a correct copy. This is the payoff of delivering identity rather than a decision.
- **A location that never arrived: visibly waiting.** Work placed at a site that hasn't
  received its location must wait, say what it's missing, and recover on its own when it
  arrives — not fail, and not stall without explanation. Compute already has close to the
  right shape here; the address-claim path should report the same way.
- **A site that can't identify itself at all** is the one genuinely new failure this
  introduces, and is the reason the configuration fallback is retained.

### Telling "nothing to do" from "not doing anything"

There's prior art on this failing, and it should be read as a specification rather than as
background. Location projection into customers' control planes stopped in staging for 34
days. It survived that long because of four properties this design would otherwise inherit
unchanged:

1. **The publisher reported nothing about its own output.** Nobody could ask how many
   locations *should* have been projected versus how many were.
2. **Absence is indistinguishable from correctness.** Zero locations is a perfectly valid
   state for a customer who isn't entitled to any. From outside, a correct empty answer and
   a dead publisher look identical.
3. **It was intermittent-looking.** Only some requests failed, and a retry usually worked, so
   it read as a blip.
4. **The one end-to-end check couldn't have caught it** — it was disabled, and it asserted
   against a record customers don't actually read.

And the symptom that did surface was actively misleading: the failure presented to users as
*"no locations are registered with the system"* — an infrastructure gap stated as though it
were a fact about the catalog, from a component that had nothing to do with the publisher.

So this design carries, as requirements:

- **The publisher reports its own output against its own input**, and a sustained gap between
  them alerts. Because of property 2, this is the only check that can distinguish the two
  states — no external probe can.
- **Freshness is reported per location**, so a stalled publisher is visible before anyone
  notices a missing location.
- **A stalled site is attributable to the site**, using the fleet's existing per-site
  readiness signal, rather than presenting as a missing location.
- **An end-to-end check exists and runs**, asserting against what consumers actually read.
  The existing one needs fixing before this ships, not after.
- **No infrastructure gap is ever reported as a statement about the catalog.** A component
  that can't find a location says what it expected and who was supposed to provide it.

## Consistency with compute

Compute currently answers one question in two places from two different records — once
correctly in the customer's control plane, and once where the work runs, against information
that was never delivered there. Those are two questions, not one:

| Question | Answered by | Where | Who may answer it |
|---|---|---|---|
| Can this workload run here? | the availability projection | the customer's control plane | admission and placement, only |
| Which location is this? | the location record | the site | site-local components, as an identity read |

What compute should change:

1. **Keep the first answer where it is.** It's the only one that has seen the entitlement,
   the service's configuration, the availability record, and the location's health.
2. **Treat the second as identity, not selection.** With a site holding only its own location,
   "no match" stops being a normal waiting state and becomes a reportable gap.
3. **Record an instance's location from the first instance onward.** Today the location is
   resolved after the instances for that generation have already been built, so the first
   generation is created without one and nothing fills it in. A customer asking where their
   workload runs gets a blank, permanently.
4. **Stop carrying a namespace on location references.** Locations aren't namespaced; the
   field is vestigial, and is already compared in a way that is harmless today and wrong the
   moment anyone populates it.
5. **Fill in the fleet's missing locality information.** Compute's own placement can't reach at
   least one compute-enabled site today for exactly this reason. That's a pre-existing bug,
   but this work is where it gets fixed.

**On location names versus city codes.** City code stays the placement key — it's what the
customer-facing placement API is expressed in, end to end, and changing that is a much larger
piece of work with no benefit here. A location's name is its identity. The two are not
interchangeable: a city code identifies a *set* of sites and cannot be unique, which is
exactly right for choosing where to place work and exactly wrong for identifying a location.
Keeping them distinct is deliberate; what has to change is that their relationship becomes
asserted rather than assumed.

## Deferred to the technical design

Named here so it's clear they were considered and consciously left out:

- The published form of a location outward — the same record as the catalog's, or a smaller
  purpose-built one carrying only identity and locality.
- How a site is matched to its location, and what has to be true of the fleet's own labelling
  for that to work.
- Where the relevant schemas are installed, and in what order relative to delivery.
- How delivery is configured, and whether it rides alongside existing configuration delivery
  or stands apart from it.
- The specific signals and thresholds behind the detection requirements above.
- Whether the availability decision ever gets its own published form, once there's a reader.

## Open Questions

- **Does a site ever serve more than one location?** This design permits it but nothing needs
  it today. If it stays hypothetical, assuming one makes a site's self-identification
  unambiguous by construction and simplifies everything downstream.
- **Is the configuration fallback permanent?** Retained here for the cases where delivery
  hasn't happened or is ambiguous. Whether it's temporary depends on how much a site is
  allowed to depend on delivery.
- **Who creates locations, once anyone does?** Taken as hand-authored here. Fanning a
  hand-authored catalog out to a fleet strengthens the case for grounding it in real records,
  which is the locations proposal's own direction.
- **What happens to a location when its site is decommissioned?** Guarding removal on a site
  still standing behind it implies an ordering in the decommissioning process, which needs to
  be written down or the guard becomes a nuisance.
- **Does a customer ever see any of this?** Nothing here is customer-facing. If "this location
  is temporarily not accepting new work" should be visible rather than a location simply
  disappearing from a list, that's a separate product decision.

## Drawbacks

- Adds a delivery path that must keep working, to a fleet whose delivery layer has frozen for
  days at a time.
- Makes a site's ability to identify itself depend on that path, where today it depends on
  configuration that's already present.
- Doesn't fix the underlying problem that locations are hand-authored and hand-marked healthy;
  it raises the stakes on both while leaving them as they are.
- Requires filling in fleet-wide locality information before it can work, which is unglamorous
  work with no visible outcome of its own.

## Alternatives

- **Deliver the availability records themselves and decide at each site.** Rejected: it turns
  one evaluator into dozens, requires shipping four more kinds of record outward, and makes
  disagreement between sites both possible and invisible.
- **Deliver the whole catalog to every site.** Rejected: sites don't choose between locations,
  so the catalog answers a question they don't ask — and each site would still need telling
  which entry is itself, which is the configuration this design set out to remove.
- **Leave locations out entirely and keep configuring each site by hand.** Rejected: it's the
  status quo, it already produces two copies of one fact that can disagree undetected, and it
  doesn't scale as the fleet grows.
- **Have the networking operator publish it, since it already delivers configuration
  outward.** Rejected: it would have to understand service catalogs and entitlements to know
  what to publish. Sharing the delivery path is fine; owning the decision is not.
