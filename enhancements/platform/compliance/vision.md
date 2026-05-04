# Compliance at Milo: Product Vision

- [Summary](#summary)
- [The Two-Level Data Controller Model](#the-two-level-data-controller-model)
- [Capability Roadmap](#capability-roadmap)
- [How the Pieces Fit Together](#how-the-pieces-fit-together)

## Summary

Milo is a control plane that service providers use to offer their
services to consumers. Those service providers operate in regulated
markets, enterprise procurement processes, and sectors where compliance
is a gating function rather than a checkbox. The compliance surface in
Milo gives service providers the structured, API-driven primitives they
need to meet their obligations to their consumers — GDPR disclosures,
data subject requests, retention and residency controls, breach
notification, and audit evidence. Any organization operating Milo also
inherits compliance obligations to the service providers it hosts. Datum
Cloud operates Milo today; the launch, documented in
[README.md](README.md), begins with that operator-side obligation:
publishing an authoritative list of the subprocessors used to deliver
the platform.

## The Two-Level Data Controller Model

Every decision in the compliance surface traces back to the layered
structure a Milo deployment sits within.

| Level | Who is the controller | Who is the processor | Who are the subprocessors |
|---|---|---|---|
| Platform | The Milo operator (for platform data: accounts, billing, platform telemetry) | The operator's infrastructure vendors | Vendors' own subprocessors (fourth parties) |
| Service provider | The service provider (for their consumers' data) | The Milo operator (processes service-provider data on their behalf) | The operator's infrastructure vendors, disclosed to service providers |

Today Datum Cloud fills the operator role; the model generalizes to any
organization running Milo. Every vendor the operator uses is a
subprocessor from a service provider's perspective. Most of the
platform's regulatory obligations are symmetric: the operator owes
service providers a structured view of its chain; service providers owe
consumers the same for theirs. When the platform builds machinery for
the operator-side obligation, the same machinery (with a
service-provider-scoped variant) helps service providers meet theirs.

## Capability Roadmap

Public disclosure is the launch scope. The next three capabilities are
adjacent — each has direct design implications for what comes after.
Later obligations are noted briefly at the end; their designs depend on
platform conditions that don't yet exist and will land as their own
enhancements.

| Capability | What it does | Rationale |
|---|---|---|
| **Public Disclosure (launch)** | Captures every vendor that processes personal data and derives a public `Subprocessor` feed for the marketing site's legal section. See [README.md](README.md). | Disclosure is the minimum obligation under Article 28(4) and the foundation for every subsequent capability. |
| **Service-Provider Consent and Objection Workflow** | Introduces `SubprocessorConsent` (scoped to the service provider) and a `Proposed` phase in the compliance profile lifecycle. Proposed changes project into each service provider's consent record, notifications fan out through configured channels, service providers acknowledge or object within a defined window, and objections block the transition to `Active` until resolved. | Enterprise and regulated-industry service providers expect structured change notification and objection rights under Article 28(4). |
| **Vendor Service Extraction** | Promotes `Vendor` to its own API group and adds the fields a richer vendor registry needs — contacts, contract references, technical integrations, security profile, richer lifecycle. Lands alongside the first non-compliance consumer (procurement, security review, or incident response). | Keeps vendor concerns out of compliance once a vendor record has multiple consumers, without paying that cost speculatively at launch. |
| **Data Subject Request Orchestration** | Structured fulfillment of Article 15, 17, and 20 requests across the platform's identity, relational, event-streaming, and object-storage subsystems. A service-provider-scoped variant lets service providers handle their own consumers' requests. | The single biggest day-to-day operational burden of GDPR. |

**Further out.** Retention-policy enforcement (data-category-keyed
deletion), data residency policy (admission-time enforcement against the
subprocessor registry), breach notification workflow (Article 33/34
incident tracking), and Article 30 ROPA (machine-readable processing
activity records) are real obligations that will get their own
enhancements when platform conditions make their designs concrete.

## How the Pieces Fit Together

The arc is additive. The subprocessor registry established at launch is
the substrate for service-provider consent and, later, residency policy
and ROPA. Raw kube-apiserver audit-log entries become the evidence
stream every capability contributes to — distinct from the Activity
service's user-facing timeline, which explicitly does not cover
compliance reporting. The derived-resource pattern (`Subprocessor`
projected from `Vendor`) is the template for every
service-provider-facing artifact the service generates: consent views,
DSR progress dashboards, breach notifications. And the lifecycle state
machine on the compliance profile generalizes to every regulated
artifact Milo introduces.
