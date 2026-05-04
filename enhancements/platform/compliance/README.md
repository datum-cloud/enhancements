---
status: provisional
stage: alpha
latest-milestone: "v0.1"
---

# Compliance Service

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Launch Scope](#launch-scope)
  - [The Compliance Profile Lifecycle](#the-compliance-profile-lifecycle)
  - [Public Disclosure](#public-disclosure)
  - [User Stories](#user-stories)
  - [Constraints](#constraints)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Resource Model](#resource-model)
  - [Public Feed Projection](#public-feed-projection)
- [Alternatives](#alternatives)
- [Implementation History](#implementation-history)

## Summary

The compliance service is the regulatory lifecycle engine inside Milo —
the control plane that service providers use to offer their services to
consumers. At launch it captures every third-party vendor the Milo
operator uses as a `Vendor` record; today that operator is Datum Cloud.
Vendors that process personal data carry an optional **compliance
profile** on that record, and the compliance controller derives a public
`Subprocessor` resource from each qualifying vendor. The marketing site's
legal section renders from the derived `Subprocessor` list. Both resources
live in the compliance service for now; vendor identity is a candidate for
extraction into its own service later, once non-compliance consumers
(procurement, security review, incident response) arrive to justify the
seam. The roadmap extends the service to service-provider consent and
objection workflows, data subject request orchestration, retention
enforcement, data residency policy, breach notification, and
machine-readable processing activity records.

## Motivation

Service providers building on Milo carry real compliance obligations to
their own consumers — GDPR disclosures, data subject requests, retention
controls, breach notifications, audit evidence. Milo does not offer a way
to meet those obligations today. The compliance service introduces that
capability: a set of structured, API-driven primitives service providers
use to manage their compliance goals on Milo, with the transparency needed
to pass downstream to their consumers.

The launch is scoped to the most fundamental piece: an accurate, current,
publicly-disclosed list of the subprocessors that handle personal data on
behalf of the Milo operator. Every downstream capability — service-provider
consent flows, DSR orchestration, residency enforcement — presumes that
this list exists and is maintained. Getting the source of truth in place
first, with a structured path to the marketing site's legal section,
unblocks everything that follows.

For Datum Cloud specifically, as the operator, the same machinery produces
the audit evidence (SOC 2, ISO 27001) and incident-ready structure
(Article 33 breach timelines, blast-radius queries) the operational role
requires. That is a useful side effect, not the primary framing.

### Goals

The launch is successful when:

- Every third-party vendor in Datum Cloud's platform supply chain — the
  systems and services used to operate Milo — has a `Vendor` record.
  Vendors that process personal data carry a compliance profile on that
  record.
- The compliance controller derives and maintains a `Subprocessor` for
  every vendor whose compliance profile is in the `Active` or `Deprecated`
  phase, with vendor identity surfaced on the subprocessor's status.
- The marketing site's subprocessor page renders from the set of derived
  `Subprocessor` resources — no separate spreadsheet, no hand-maintained
  HTML, no drift between what staff believes is true and what consumers
  see on the public disclosure.
- Mutations on `Vendor` and compliance-profile phase transitions produce
  entries in the platform's raw audit log (the kube-apiserver audit
  output). Semantic audit events beyond that default — a
  `profile-activated` event distinct from a generic resource UPDATE — are
  a separate investment not covered by launch.

### Non-Goals

- **Service-provider consent and objection workflows at launch.**
  Per-service-provider acknowledgement, notification fan-out, and
  objection handling are explicitly deferred to a roadmap item. The
  launch produces a public disclosure; the consent flow follows once the
  registry is proven.
- **A standalone vendor management service.** Vendor identity lives in
  the compliance service for now and is expected to extract once it has
  consumers beyond compliance — see the Vendor Service Extraction entry
  in [vision.md](vision.md).
- **Legal interpretation.** The service captures regulatory facts. It does
  not generate legal advice, draft DPAs, or make determinations about
  lawful basis or legitimate interests.
- **Consumer consent for service-provider applications.** Service
  providers' consumers have their own consent relationships with the
  service provider. The downstream DSR orchestration (roadmap) supports
  those flows; launch does not.
- **Contract lifecycle.** DPAs are contracts; contract lifecycle lives in
  the CLM layer when that integration arrives. The compliance profile
  references a DPA and validates that one is linked.

## Proposal

This enhancement covers the launch scope only — public subprocessor
disclosure. The broader compliance vision, including the two-level data
controller model and the multi-year capability arc this launch anchors,
lives in [vision.md](vision.md).

### Launch Scope

The launch introduces one staff-managed resource and one derived public
resource.

**`Vendor`** is a staff-managed resource on the Platform Control Plane.
It carries vendor identity — display name, legal entity, country of
incorporation (ISO 3166-1 alpha-2), website — and an optional
**compliance profile**. A vendor without a compliance profile is a vendor
that does not process personal data (OSS dependencies, tooling that never
touches consumer data) and has no corresponding subprocessor. A vendor
*with* a compliance profile captures the fields that exist for GDPR
purposes: purpose of processing, data categories (identity,
authentication, telemetry, billing, user-content, access-logs,
audit-trail), data subject types (organization-admin, consumer,
platform-staff), processing regions, legal transfer mechanism (SCCs,
adequacy decision, BCRs), risk tier, DPA reference, effective date, and
a lifecycle phase.

**`Subprocessor`** is a controller-derived resource on the Platform
Control Plane. Every `Vendor` whose compliance profile is `Active` has
exactly one corresponding `Subprocessor`. Its status carries the full
disclosure surface: vendor identity projected from the `Vendor` record,
plus the compliance-profile fields flattened for consumption.
`Subprocessor` is the compliance service's public disclosure surface;
`Vendor` remains staff-only. How that surface reaches consumers without
Milo credentials is resolved by the design document — see
[Public Feed Projection](#public-feed-projection) for the shape of the
choice.

### The Compliance Profile Lifecycle

A compliance profile on a `Vendor` has two phases at launch.

| Phase | `Subprocessor` present? | What's happening | When does it advance |
|---|---|---|---|
| **Draft** | No | Internal preparation. Staff fills in DPA linkage, risk classification, data categories, and review. No public disclosure. | When staff promotes the profile to `Active`. |
| **Active** | Yes | The subprocessor is live and appears in the public feed. | — |

Two extensions of this lifecycle are on the roadmap. A `Proposed` phase
(described in the Service-Provider Consent entry in
[vision.md](vision.md)) will sit between `Draft` and `Active` and gate
promotion on a notification window plus per-service-provider
acknowledgement. Retirement phases (`Deprecated`, `Removed`) will be
added when vendor retirement becomes a real operation.

### Public Disclosure

The marketing site's subprocessor page renders from the current set of
`Subprocessor` resources. Each entry carries vendor identity (display
name, legal entity, country of incorporation), purpose, data categories,
processing regions, transfer mechanism, and effective date.

Updating the page is a control-plane action: a staff operator promotes a
vendor's compliance profile from `Draft` to `Active` and the controller
reconciles the corresponding `Subprocessor` into existence. The next feed
read reflects the change. The `Vendor` record stays staff-only; the
derived `Subprocessor` is what consumers read — see
[Public Feed Projection](#public-feed-projection). Service providers who
want to reference the list from their own legal pages can link to the
page or consume the `Subprocessor` feed directly.

### User Stories

**As a Datum Cloud compliance operator introducing a new subprocessor**, I
create a `Vendor` record with identity fields and a compliance profile
carrying purpose, data categories, transfer mechanism, and DPA linkage. I
promote the profile to `Active` when ready. The controller reconciles a
`Subprocessor` and the marketing site's legal section reflects the change
on its next build.

**As a Datum Cloud compliance operator adding a vendor that doesn't
process personal data**, I create a `Vendor` record without a compliance
profile. No `Subprocessor` is generated and nothing appears on the
public feed. The vendor record still exists for future use as the
registry evolves.

**As a visitor to the Datum Cloud marketing site reviewing our compliance
posture**, I can see the current list of subprocessors — who they are,
what they process, where they process it, under what transfer mechanism —
without emailing the company and waiting for a PDF.

**As a service-provider admin preparing my own disclosure obligations**,
I either link to the Datum Cloud subprocessor page or consume the
`Subprocessor` feed directly. My own compliance documentation stays
current when Milo's does, without manual synchronization on my side.

### Constraints

**Vendors and subprocessors live on the Platform Control Plane.** The
operator's vendors apply uniformly to every service provider on the
platform.

**Draft is hard-hidden.** A compliance profile in `Draft` has no derived
`Subprocessor`. There is nothing for an external caller to see, and the
protection is structural rather than policy-based.

**The `Subprocessor` is the source of truth for disclosure.** The
marketing site does not maintain a parallel copy. Styling and surrounding
context live in the site's own rendering layer on top of the feed data.

**Audit evidence is the raw kube-apiserver audit log.** It is distinct
from the Activity service's user-facing timeline, which explicitly does
not cover compliance reporting. Downstream compliance capabilities read
from the raw audit log directly.

**Disclosure alone is not consent.** Publishing a subprocessor list
satisfies the disclosure obligation. It does not satisfy the stronger
consent and objection obligations that some contracts and jurisdictions
impose. Those obligations are addressed by the Service-Provider Consent
capability on the roadmap.

**Extraction seam is preserved.** `Subprocessor` references `Vendor`
rather than inlining identity, so `Vendor` can later move to its own
service when a non-compliance consumer arrives. See the Vendor Service
Extraction entry in [vision.md](vision.md).

### Risks and Mitigations

**Risk: the public feed goes stale.** If staff promotes compliance
profiles inconsistently, the published list drifts from reality.
**Mitigation:** the `Subprocessor` is derived from `Vendor` state, not
from a separate export step — the feed cannot drift from its source.

**Risk: subprocessor sprawl.** A too-broad list dilutes the signal.
**Mitigation:** inclusion is gated on whether the `Vendor` record
carries a compliance profile. OSS dependencies that never touch personal
data do not, and therefore do not generate a `Subprocessor`.

**Risk: vendor schema accumulates fields without consumers.**
**Mitigation:** the launch keeps `Vendor` to identity plus the compliance
profile. Additional fields (contacts, contract references, technical
integrations, security profile) land when a consuming capability needs
them, and at that point the resource is a candidate for extraction.

**Risk: the data-category taxonomy is load-bearing and under-validated.**
The launch enumerates seven data categories. Retention, residency, and
ROPA on the roadmap all key off this taxonomy, so getting it wrong means
schema churn across multiple future capabilities. **Mitigation:** treat
the taxonomy as a versioned vocabulary — additions should be easier than
renames, each category is documented with its rationale, and the set is
reviewed with legal before locking.

## Design Details

The compliance service is implemented as a set of Platform Control Plane
custom resources with an associated controller, plus a public disclosure
surface that feeds the marketing site. The service integrates with IAM,
the agreements system, and the platform's raw audit log. This document
is product-focused and does not specify the full API surface; a separate
design document will cover resource schemas, controller reconciliation
logic, admission webhooks, the RBAC model, and the mechanism by which
`Subprocessor` reaches consumers without Milo credentials.

### Resource Model

| Resource | Control Plane | Visibility | Owner |
|---|---|---|---|
| `Vendor` | Platform | Staff-only | Compliance service (today); candidate for extraction |
| `Subprocessor` | Platform | Publicly disclosed (mechanism per design doc) | Compliance controller (derived from `Vendor`) |

### Public Feed Projection

Milo apiserver access requires authentication — every integrator runs an
authentication webhook that validates incoming bearer tokens. `Vendor`
inherits that default and must not be publicly readable regardless,
because it carries the compliance profile and will accumulate further
staff-only fields. A public feed therefore cannot be a direct view onto
`Vendor`; it needs a dedicated projection whose schema is, by
construction, safe to disclose.

The pattern is a derived resource. The compliance controller watches
`Vendor` with its elevated service-account credentials and reconciles
`Subprocessor` — a resource containing only disclosure-safe fields:
display name, legal entity, country of incorporation, purpose, data
categories, processing regions, transfer mechanism, effective date, and
lifecycle state. RBAC on `Vendor` remains staff-only.

Exposing `Subprocessor` to consumers without Milo credentials is a
design-document concern. Plausible mechanisms include a dedicated
aggregated-apiserver endpoint with permissive authorization for this
subresource, a lightweight projection service that reads `Subprocessor`
and serves it unauthenticated, or a scheduled render to the Datum Cloud
marketing site's content system. Each has different operational
tradeoffs; the product surface is the same in every case.

Two properties fall out of this design. First, the security boundary is
structural: the public type does not have the sensitive fields, so they
cannot leak through a misconfigured policy. Second, updates propagate
live — a vendor rename or compliance-profile lifecycle transition
triggers a reconciliation, the projection rewrites, and the next feed
read reflects the change without a manual publication step.

## Alternatives

**A standalone `PlatformSubprocessor` resource in addition to `Vendor`.**
Splits identity from regulatory overlay cleanly, but at launch every
staff-managed regulatory record is effectively a projection of a
vendor — the split just doubles the resources staff manage without
adding expressiveness. Rejected. Can be reintroduced alongside the
vendor-service extraction if the overlay needs its own home.

**Two services from day one (compliance + vendor management).** Cleaner
conceptual seam, but at launch scope the vendor service would be a
thin identity table whose only consumer is the compliance controller —
two CRDs, two controllers, two RBAC surfaces, and a cross-service
projection to express what one service can express simply. Rejected for
now; reconsidered when a non-compliance consumer arrives.

**Rely on a hand-maintained subprocessor page.** No structured source of
truth, no audit trail, drifts from reality. Rejected.

**Adopt an off-the-shelf GRC tool.** Not Kubernetes-native, doesn't
integrate with Milo's resource model, and doesn't give service providers
structured API access to their own compliance chain as later capabilities
land. Possibly useful for staff workflows (vendor security reviews) once
the vendor service is extracted, but not as the primary system.

**Defer compliance automation until a specific scale or regulatory
trigger.** Tempting, but puts us in the position of building compliance
tooling under time pressure. Starting with a minimal, deployable launch
scope lets us make steady progress in product time.

## Implementation History

- 2026-04-14: Initial enhancement draft.
