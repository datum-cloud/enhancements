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

In a staging load test, private DNS changes took 3 to 5 seconds at p50 to become
answerable after the write, and longer when the platform API was slow. The name travels as
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

- A provider can update and withdraw the records of names it holds without an API
  write and central recompile per change.
- The platform enforces that a provider only publishes where it is allowed:
  which projects, zones, names and record types.
- A change is visible on authoritative serving within a target to be set by a
  per-hop latency budget, excluding client caches, which keep answers up to
  their TTL.
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
- Providers changing zone configuration: apex, SOA, NS or associations.
- Third-party providers before a separate trust review. In alpha, compute is the
  only provider.

## Proposal

Split a zone into slices. The zone's owner, through the project API, defines the
zone and grants providers a slice. Each provider publishes and versions its own
slice directly into the serving path. The serving fleet merges a zone's slices.

- **Publishing grants (API, in the tenant's project).** A grant names a
  provider, a zone, the record types it may publish and their limits, such as the
  maximum TTL. Creating a grant is a tenant or platform decision, recorded and
  visible like any other access binding. In zones the platform owns, such as a
  network's managed `datum.internal` namespace, the platform holds the grant;
  see the open decisions.
- **Edge enforcement (platform primitive).** Enforcing a tenant's grants where
  data enters the edge is not specific to DNS, so it will be defined as its own
  platform primitive in a follow-up enhancement. The broker can only check a
  provider's identity and the subjects it publishes to; it cannot see names or
  record types inside a slice. Serving is therefore the authoritative check of
  zone, name, type and TTL against the grant, and drops anything outside it.
- **Slices.** A provider publishes whole versions of its slice of a zone, numbered
  and fenced like today's zone versions. A serving member loads a slice version
  atomically and catches up by taking the latest one.
- **Merging.** The serving fleet answers a zone from the zone owner's slice (user
  records, zone defaults and ownership markers) plus each granted provider's
  slice. Each slice is atomic on its own; the zone as a whole is not atomic across
  slices, and nothing may depend on cross-slice atomicity.
- **Name ownership.** Which slice may answer a name is decided before any slice
  is published; see below.
- **Status.** The platform aggregates per-slice acknowledgements from serving
  members into the grant's status, and into the status of the names it carries,
  through the API. Visibility stays bounded by API latency, which is acceptable
  because publishing no longer waits on it.

### Name ownership

Names are not naturally disjoint between providers. Compute names today are flat
(`<resource>.datum.internal`) and share a namespace with other providers and with
tenants' own records. The control plane decides conflicts: tenants' records claim
first, then registrations in creation order, with CNAME exclusivity and naming
policies applied across all of them. Two options keep that decided:

- **(a) Names stay central, records move.** A provider keeps claiming a name
  through the API once per workload lifetime, and that claim is still compiled
  and conflict-checked centrally. Only the record content and eligibility for
  names it holds, which is the part that changes often, move into its slice.
  A brand-new name still goes through the API path; address and health changes
  take the fast path.
- **(b) Providers own subtrees.** Grants assign each provider a subtree, and
  admission prevents tenants' records and other grants from overlapping it. This
  changes the flat naming users see today.

The recommendation is (a): it keeps today's naming, naming policies and conflict
status, and it still removes the per-change API write and recompile.

### User Stories

#### An instance's address change is answered quickly

A tenant's instance moves to a new address. Compute, which holds the instance's
name, publishes the new address in its slice, and authoritative serving answers
with it within the latency target. Clients that cached the old answer keep it
until its TTL expires.

#### A tenant revokes a provider

A tenant removes a provider's grant. Serving drops the provider's slice from the
zone within a stated bound, and the provider can no longer publish into it.

#### A provider stops publishing

A provider has an outage. Serving keeps its last slice version, as it keeps all
applied state through outages today, until the grant is revoked or the slice's
lease, if one is adopted, expires.

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
- **Two sources for one zone.** Each slice is versioned and loaded atomically, but
  the zone is not atomic across slices. Ownership decided centrally (option a)
  keeps a name in one slice at a time.
- **One provider affecting others.** Slices must not share the control plane's
  stream limits, de-duplication window or bootstrap keys. Each provider gets its
  own stream or account, its own quotas and its own bootstrap keys.
- **Wedged or resurrected slices.** A provider that publishes an extreme version
  could block its slice forever, and one that loses state could be ignored as
  stale. Slice versions are bound to the grant's identity and generation, the
  epoch comes from a durable compare-and-set, and serving drops a slice whose
  grant is gone. Revocation and grant changes still reach serving through the
  control plane's serving snapshot, at control-plane latency.

## Design Details

<<[UNRESOLVED]>>
Decisions needed before this is implementable:

1. Name ownership: option (a) or (b).
2. How grants reach serving (the serving snapshot) and how the edge enforcement
   primitive maps provider identity to publish permissions.
3. Placement and fan-out: whether a provider follows a zone's region and shard
   placement, or the broker maps a provider-level subject onto placement.
4. Slice fencing: epoch source, binding to the grant, and recovery from a bad
   high-water mark.
5. Zone readiness with several slices: per-slice required versions in the
   serving snapshot, or serving with whatever slices have arrived. Also a merged
   SOA and per-slice proof for verification, carrying today's ownership markers
   and reserved names.
6. Rules that span slices today, such as SRV targets that must resolve in the
   zone: drop them, or check them at serving time.
7. Who aggregates acknowledgements into status, and where.
8. Revocation semantics and its time bound.
9. Migration from compute's contributions to a compute slice without a window
   where a name is served twice or not at all, rollback to the central path,
   and mixed serving versions during upgrade.
10. A per-hop latency budget that shows the gain over the faster central compile.
<<[/UNRESOLVED]>>

## Production Readiness Review Questionnaire

To be completed when the design is implementable.

## Alternatives

- **Faster central compile.** Making the control plane recompile only the
  changed zone from cached reads is in progress, with an expected 0.8 to 1 s p50
  from write to authoritative answer. It keeps every change passing through the
  platform API and a central writer, and doesn't open publishing to other
  providers. The latency budget (decision 10) should show what this proposal saves
  beyond it, to justify a new trust boundary.
- **Record-level diffs instead of slices.** Diffs need ordered delivery, gap
  detection and atomic application on every serving member. Whole slice versions
  keep the current, simpler model.
