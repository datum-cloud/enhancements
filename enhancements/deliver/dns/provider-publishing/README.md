---
status: provisional
stage: alpha
latest-milestone: "TBD"
---

# DNS publishing by trusted providers

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
- [Alternatives](#alternatives)

## Summary

Let services that run workloads, like compute and later third-party providers,
publish DNS names for those workloads straight into Datum's DNS serving path.
The platform enforces which tenants, zones and names each provider may publish,
at the point where names enter the serving path. Today every published name goes
through a project API object and a central control plane, and that path takes
seconds per change.

## Motivation

Private DNS names for compute instances take 3 to 5 seconds to appear after an
instance changes, and longer when the platform API is slow. The name travels as
a contribution object in the project API. A central control plane reads it,
recompiles the whole zone, writes the result back into the API, and only then
sends it to the serving fleet.

The trust decision in that path, "may this provider publish this name in this
tenant's zone?", doesn't change per record. It is made once, when a provider is
allowed to serve a project. Everything after it is data movement that doesn't
need the platform API.

Making that decision a platform primitive has a second benefit: compute stops
being special. Any provider that runs workloads for a tenant can publish names
for them under the same rules.

### Goals

- A provider can publish, update and withdraw names for the workloads it runs
  without writing an API object per name.
- The platform enforces, at the serving edge, that a provider only publishes
  where it is allowed: which projects, which zones, which names and which record
  types.
- A changed workload's name is answerable in well under a second at p50.
- Tenants and operators can see which providers may publish into a zone, and
  revoke that permission.
- Zone shape stays with the zone's owner: apex, which networks resolve it, and
  which providers may contribute.

### Non-Goals

- Designing how the edge enforces tenancy RBAC. That is a platform primitive
  with its own follow-up enhancement; this proposal consumes it.
- Changing how tenants write their own records. User-written record sets keep
  their current API.
- Public DNS. This covers private zones served to networks.
- Cross-tenant sharing of zones.

## Proposal

Split a zone into slices. The zone's owner, through the project API, defines the
zone and grants providers a slice. Each provider publishes and versions its own
slice directly into the serving path. The serving fleet merges a zone's slices.

- **Publishing grants (API, in the tenant's project).** A grant names a
  provider, a zone, and the names and record types it may publish: for example,
  compute may publish `A` and `AAAA` records for `*.instances.<zone>`. Creating a
  grant is a tenant or platform decision, recorded and visible like any other
  access binding.
- **Edge enforcement (platform primitive).** Enforcing a tenant's grants where
  data enters the edge is not specific to DNS, so it will be defined as its own
  platform primitive in a follow-up enhancement. This proposal relies on it to
  ensure a provider can only publish the slices its grants allow, so a
  misbehaving or compromised provider cannot publish into a zone it has no grant
  for.
- **Slices.** A provider publishes whole versions of its slice of a zone, numbered
  and fenced like today's zone versions. A serving member loads a slice version
  atomically and catches up by taking the latest one.
- **Merging.** The serving fleet answers a zone from the zone owner's slice (user
  records and zone defaults) plus each granted provider's slice. Names in
  different slices cannot conflict, because grants assign disjoint name spaces.
- **Status.** Providers report their publication state through the platform's
  existing status surfaces, so a tenant can still see whether a name is being
  served.

### User Stories

#### A tenant's instance gets a private name immediately

A tenant creates an instance in a network that resolves `internal.example`.
Compute, which holds a grant for `*.instances.internal.example`, publishes the
instance's address in its slice. The name answers on the network's resolver
within a second, and disappears within a second of the instance stopping.

#### A third-party provider publishes names for its workloads

A database provider runs managed databases inside a tenant's network. The tenant
grants it `*.db.internal.example`. The provider publishes and withdraws names as
databases come and go, without access to any other part of the zone.

### Risks and Mitigations

- **A provider publishes outside its grant.** The platform's edge enforcement
  rejects it, and serving rejects slice content outside the grant's names and
  types as a second check.
- **A grant is revoked.** Revocation withdraws the provider's permission through
  the edge enforcement primitive and tells serving to drop the slice, both driven
  from the API object.
- **Visibility.** Tenants could lose sight of names that never become API
  objects. Slice status per grant, and a read API over served names, keep them
  visible.
- **Two sources for one zone.** Each slice is versioned and loaded atomically, so
  a zone is always the latest complete version of each slice.

## Design Details

<<[UNRESOLVED]>>
- Grant API shape for DNS slices.
- How DNS grants are expressed in the platform's edge enforcement primitive
  (provider identity, permissions, rotation), once that primitive is proposed.
- Slice format, fencing and ordering across providers; whether the zone owner's
  slice stays compiled by the current control plane.
- How serving merges slices and enforces grant boundaries a second time.
- Migration of compute's contributions to a compute slice.
<<[/UNRESOLVED]>>

## Production Readiness Review Questionnaire

To be completed when the design is implementable.

## Alternatives

- **Faster central compile.** Making the control plane recompile only the
  changed zone, from cached reads, is in progress and brings a change to about a
  second. It keeps every name passing through the platform API and a central
  writer, and doesn't open publishing to other providers.
- **Record-level diffs instead of slices.** Diffs need ordered delivery, gap
  detection and atomic application on every serving member. Whole slice versions
  keep the current, simpler model.
