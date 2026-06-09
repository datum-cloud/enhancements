---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

<!-- omit from toc -->

# Service Pricing, Offers, and Billing Entitlements

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Concepts](#concepts)
  - [Use Cases](#use-cases)
    - [Use Case 1: A brand-new billing account starts billable on day one](#use-case-1-a-brand-new-billing-account-starts-billable-on-day-one)
    - [Use Case 2: A service owner prices a meter, with rates that vary by dimension](#use-case-2-a-service-owner-prices-a-meter-with-rates-that-vary-by-dimension)
    - [Use Case 3: The platform raises prices without re-pricing existing customers](#use-case-3-the-platform-raises-prices-without-re-pricing-existing-customers)
    - [Use Case 4: Staff put a specific account on a custom or zero-rated Offer](#use-case-4-staff-put-a-specific-account-on-a-custom-or-zero-rated-offer)
    - [Use Case 5: An invoice run produces a non-zero subtotal](#use-case-5-an-invoice-run-produces-a-non-zero-subtotal)
  - [How Offers relate to billing models](#how-offers-relate-to-billing-models)
  - [Composition with quota](#composition-with-quota)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Resource Topology](#resource-topology)
  - [Per-meter pricing on ServiceConfiguration](#per-meter-pricing-on-serviceconfiguration)
  - [ServicePricing fan-out](#servicepricing-fan-out)
  - [Offer](#offer)
  - [BillingEntitlement](#billingentitlement)
  - [Default Offer policy](#default-offer-policy)
  - [amberflo-provider reconcilers](#amberflo-provider-reconcilers)
  - [Display names](#display-names)
  - [Boundary with credits](#boundary-with-credits)
- [Acceptance Criteria](#acceptance-criteria)
- [Suggested Implementation Phases](#suggested-implementation-phases)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)

## Summary

We can measure what customers use, but we can't yet charge them for it.
[Metering][681] gave us usage data and the `BillingAccount`, but no prices are
attached to anything, so when we invoice we'll have no amount to charge.

This enhancement doc propeses missing pieces: service owners set prices on their
meters, the platform bundles those into named **Offers** (pay-as-you-go, Pro,
and so on), and every billing account gets one by default. Once that's in place,
an Amberflo invoice run finally returns a real dollar amount — the number the
invoicing controller will eventually hand to Stripe.

## Motivation

The umbrella billing enhancement names three primitives — Service Pricing,
Offers, and Entitlements — and the current implementation has none of them.
Concrete blockers today:

- **Amberflo has meters but no rates**, so any invoice run returns zero.
- **New organizations have no pricing context.** Even after we build the
  invoice-run controller, there is nothing for it to bill.
- **There is no platform-defined way to roll out a tier change** (for example,
  "Pro now includes egress") without hand-editing every billing account.
- **Quota gating and pricing are unconnected.** `ServiceEntitlement` (per
  project, in service-catalog) drives quota grants but does not connect to
  pricing, even though the umbrella explicitly says the two should be linked.

### Goals

- **Per-meter pricing on `ServiceConfiguration`.** A `pricing` block on each
  `spec.metrics[]` entry: currency, pricing unit, and rates (flat or tiered),
  with per-dimension match support so rates can vary by `MeterDefinition`
  dimension value.
- **Fan-out to `ServicePricing` resources**, one per priced metric, emitted by
  a new `PricingFanOut` sibling to the existing `QuotaFanOut`.
- **`Offer` resource** that bundles service pricings into a named, versioned
  tier with `launchStage` semantics and a snapshot of the referenced pricings
  taken at publish time.
- **`Offer.spec.billingModel` required from day one**, with `Usage` the only
  valid v1 value so that later billing models do not need a breaking schema
  migration.
- **`BillingEntitlement` resource**, billing-account-scoped, with exactly one
  active per account, distinct from the existing per-project
  `ServiceEntitlement`.
- **Default Offer per billing account**, applied automatically by a new
  service-catalog controller on `BillingAccount` create.
- **`amberflo-provider` reconcilers** that sync `Offer` to a Product Plan and
  `BillingEntitlement` to a Customer-Plan assignment.
- **Offer authoring and override in staff-portal**, including the full
  draft → publish lifecycle and the ability to switch a billing account's
  active Offer with an audit-log entry.

We will know this has succeeded when a new billing account in staging lands on
a default Offer automatically and an invoice run for an organization with
real usage returns a non-zero dollar subtotal.

### Non-Goals

- **One-time and recurring charges.** Usage-based only for v1.
- **Real-time rating, spend caps, and spend alerts.** Monetary limits and
  alerting (for example, "don't let an account spend more than $100") depend on
  rated-usage data that this enhancement makes possible but does not itself
  deliver; that work is tracked separately in [#759][759].
- **Commitments and subscriptions.** The `billingModel` field reserves room for
  these, but only `Usage` ships in v1.
- **Consumer-facing Offer browse and self-serve switch.** Cloud-portal shows the
  active Offer name read-only; there is no compare or accept-offer flow.
- **Suspension, dunning, and refunds**, which live with the invoice-run
  controller and Stripe.
- **Multi-currency (USD only), discounting, marketplace billing, and cross-meter
  bundle discounts.**
- **Re-rating historical usage** — prevented by construction via immutable
  published Offers.

## Proposal

### Concepts

| Concept                 | Owner             | Lives in        | What it does                                              |
| ----------------------- | ----------------- | --------------- | --------------------------------------------------------- |
| **Service Pricing**     | Service owner     | service-catalog | Rates attached to a meter, declared next to that meter.   |
| **Offer**               | Platform owner    | service-catalog | A named, versioned bundle of service pricings (a "tier"). |
| **Billing Entitlement** | Platform / policy | service-catalog | Binds a billing account to exactly one active Offer.      |

Service owners author pricing where they already author meters. The platform
composes priced services into "Offers". A billing account becomes billable by
pointing a Billing Entitlement at an Offer. Everything downstream — the Amberflo
Product Plan, the Customer-Plan assignment, the eventual invoice is derived
from those three objects.

### Use Cases

The following walk through the situations this enhancement is meant to address
and how the proposal handles each. They double as the acceptance scenarios.

#### Use Case 1: A brand-new billing account starts billable on day one

**Situation.** A customer signs up and a `BillingAccount` is created (by the
[onboarding flow][748], or by backfill for existing accounts). Today that
account has no pricing context, so even a fully metered organization invoices to
zero.

**How the proposal addresses it.** A new service-catalog controller
(`billing-entitlement-defaults`, a sibling to the `OrganizationDefaultsReconciler`)
watches `BillingAccount` create, reads `ServiceConfiguration.spec.defaultOffer`
on the `billing-miloapis-com` configuration, and SSA-applies a
`BillingEntitlement` referencing that Offer. Every new account lands on a known,
priced default (for example, `default-pay-as-you-go-v1`) within a single
reconcile, with no manual step. The default is a single GitOps-managed field per
environment, so staging and production can default differently.

#### Use Case 2: A service owner prices a meter, with rates that vary by dimension

**Situation.** The owner of `compute.datumapis.com` wants to charge for
`instance/cpu-seconds` and `instance/memory-seconds`, and the CPU rate should
differ by region. For an AI service, the owner wants `claude-sonnet-4` priced
differently from `claude-opus-4` on the same meter.

**How the proposal addresses it.** The owner adds a `pricing` block to the
relevant `spec.metrics[]` entry in `ServiceConfiguration`, including
per-dimension match rules that key off a single `MeterDefinition` dimension
(`region`, `model`, …) for v1. The existing fan-out machinery emits one
`ServicePricing` per priced metric, mirroring how `MeterDefinition`s are already
produced — so providers watch a single, predictable shape and the service
owner's surface area stays minimal.

#### Use Case 3: The platform raises prices without re-pricing existing customers

**Situation.** Pro should now include more resources, and CPU rates are going up.
Existing Pro customers must keep their current rates until they are explicitly
moved; only new or migrated accounts should get the new pricing.

**How the proposal addresses it.** `Offer.spec.servicePricings[]` is a
**snapshot** copied from the referenced `ServicePricing`s at publish time, and a
published Offer is immutable except for its display-name annotation. A rate
change to a `ServicePricing` therefore never silently re-prices a live Offer.
Rolling out new pricing means publishing a new Offer version (`pro-v2`) and
moving accounts to it deliberately. Historical re-rating is impossible by
construction.

#### Use Case 4: Staff put a specific account on a custom or zero-rated Offer

**Situation.** Support needs to move one billing account onto an internal,
zero-rated Offer (`staff-zero-v1`) for testing, or onto a negotiated tier — and
the change must be auditable.

**How the proposal addresses it.** Staff-portal can switch a billing account's
`BillingEntitlement.spec.offerRef` to any published Offer. The mutation writes
through milo with an audit-log entry, and the `amberflo-provider` updates the
account's Customer-Plan assignment within one reconcile. Because exactly one
`BillingEntitlement` is active per account, there is never ambiguity about which
Offer is in force.

#### Use Case 5: An invoice run produces a non-zero subtotal

**Situation.** An organization has consumed metered usage over a billing period.
We need Amberflo to return what they owe so a downstream controller can charge
Stripe.

**How the proposal addresses it.** The `amberflo-provider` reconciles each
`Offer` into a Product Plan (with Product Plan Items per pricing entry, honoring
dimension filters) and each `BillingEntitlement` into a Customer-Plan
assignment. With rates attached and a plan assigned, an Amberflo invoice run for
the billed period returns a non-zero dollar subtotal. That subtotal is the input
the credit ledger ([#747][747]) draws against and the future invoice-run
controller hands to Stripe.

### How Offers relate to billing models

Every `Offer` carries a required `spec.billingModel`. In v1 the only valid value
is `Usage`. Reserving the field now — rather than introducing it later — means
`Commit`, `Subscription`, and `NonBilled` can become valid values in subsequent
versions without a breaking schema migration. This gives an Offer an explicit,
machine-readable answer to "how does this get billed?" from the start, while
keeping v1 scoped strictly to usage-based billing.

### Composition with quota

Pricing and quota stay linked. `BillingEntitlement` activation continues to
trigger the existing quota fan-out via the `OrganizationDefaultsReconciler` and
`ServiceEntitlementReconciler.ensureQuotaGrants`. Pricing rides on the same
activation event, so an Offer that costs a given amount also unlocks the right
quota — the umbrella's requirement that quota and pricing be connected is
satisfied without merging the two resources.

### Risks and Mitigations

- **Snapshot drift confusion.** Operators may expect editing a `ServicePricing`
  to change live Offers. _Mitigation:_ immutability is enforced on publish and
  documented; staff-portal surfaces the snapshot explicitly and requires a new
  Offer version for rate changes.
- **Two "Entitlement" kinds.** `BillingEntitlement` (new) and `ServiceEntitlement`
  (existing per-project quota driver) could be conflated. _Mitigation:_ they are
  deliberately distinct kinds with distinct scopes; the new kind avoids invasive
  renames and is named for the umbrella's terminology.
- **Default Offer misconfiguration.** A wrong `defaultOffer` field would
  mis-price every new account. _Mitigation:_ it is a single GitOps-managed field
  per environment, reviewed like any other config change, and idempotent on
  re-apply.

## Design Details

### Resource Topology

```text
ServiceConfiguration            ServicePricing            Offer
  .spec.metrics[].pricing  ─►  (one per priced  ◄──refs── .spec.servicePricings[]
   (authored by service           metric)                  (snapshot at Publish)
    owner; per-dimension      (fan-out emitted by                ▲
    rates supported)           service-catalog,                  │
                              analogous to                       │
                              MeterDefinition)                   │
                                    │                            │
                                    ▼                            ▼
                            amberflo-provider:           amberflo-provider:
                            Product Plan Items           Product Plan
                            (with dimension              (groups items by Offer)
                             filters)
                                                                ▲
                                                                │ refs
                                                                │
            BillingAccount ──policy──►  BillingEntitlement (new)
                                          .spec.billingAccountRef
                                          .spec.offerRef
                                                │
                                                ▼
                                          amberflo-provider:
                                          Customer-Plan assignment
```

### Per-meter pricing on ServiceConfiguration

A `pricing` block is added to each `spec.metrics[]` entry: currency,
`pricingUnit`, and rates (flat or tiered). Per-dimension `match` entries let a
rate vary by a single `MeterDefinition` dimension value (one dimension per match
entry for v1; multi-dimension matches deferred).

```yaml
spec:
  metrics:
    - name: instance/cpu-seconds
      pricing:
        currency: USD
        pricingUnit: cpu-second
        rates:
          - match: { dimension: region, value: us-east }
            flat: "0.0000125"
          - match: { dimension: region, value: eu-west }
            flat: "0.0000140"
          - flat: "0.0000130" # default when no match
```

### ServicePricing fan-out

service-catalog's existing `QuotaFanOut` gains a sibling `PricingFanOut` that
emits one `ServicePricing` per priced metric, matching the `MeterDefinition`
fan-out pattern already in place. Providers watch the single `ServicePricing`
shape rather than parsing `ServiceConfiguration`.

### Offer

`Offer` bundles service pricings into a named tier (`pay-as-you-go-v1`,
`pro-v1`, …) with `launchStage` semantics. `spec.servicePricings[]` is a
snapshot copied from the referenced `ServicePricing`s at publish time.
`spec.billingModel` is required (`Usage` only in v1). Immutable post-Published
except for the `kubernetes.io/display-name` annotation.

```yaml
apiVersion: billing.miloapis.com/v1alpha1
kind: Offer
metadata:
  name: default-pay-as-you-go-v1
  annotations:
    kubernetes.io/display-name: "Pay As You Go"
spec:
  billingModel: Usage
  launchStage: GA
  servicePricings: # snapshot taken at Publish
    - serviceRef: compute.datumapis.com
      metric: instance/cpu-seconds
      currency: USD
      pricingUnit: cpu-second
      rates: [...]
```

### BillingEntitlement

Billing-account-scoped, with exactly one active per account. Distinct from the
existing per-project `ServiceEntitlement`.

```yaml
apiVersion: billing.miloapis.com/v1alpha1
kind: BillingEntitlement
metadata:
  name: be-<bauid>-default
  namespace: <ba-namespace>
spec:
  billingAccountRef: { name: <billing-account> }
  offerRef: { name: default-pay-as-you-go-v1 }
```

### Default Offer policy

On `BillingAccount` create, the `billing-entitlement-defaults` controller reads
`ServiceConfiguration.spec.defaultOffer` and SSA-applies a `BillingEntitlement`
named deterministically (for example `be-<bauid>-default`) in the account's
namespace. Re-apply on policy reconcile is idempotent.

### amberflo-provider reconcilers

- `Offer` → Amberflo Product Plan (plus Product Plan Items per pricing entry,
  with dimension filters).
- `BillingEntitlement` → Amberflo Customer-Plan assignment.
- Finalizer plus SSA pattern identical to the existing meter and customer sync.

### Display names

`Offer` carries a `kubernetes.io/display-name` annotation (matching the
convention used on milo `Role`s) so the human-readable name can change without
bumping the Offer version. `ServicePricing` inherits its `MeterDefinition`'s
display name; `BillingEntitlement` defers to its Offer's.

### Boundary with credits

Invoice-run sequence: Amberflo computes the subtotal using Offer rates → the
credit ledger ([#747][747]) draws down → the remainder hits Stripe via the
invoice-run controller. #747 designs against the Amberflo subtotal as its input.

## Acceptance Criteria

- A `ServiceConfiguration` for `compute.datumapis.com` carries pricing for
  `instance/cpu-seconds` and `instance/memory-seconds`, with per-`region` rate
  variation on at least one meter.
- `ServicePricing` resources are emitted by the fan-out controller, one per
  priced metric.
- A published `Offer` (`default-pay-as-you-go-v1`) references those pricings as
  a snapshot.
- Every new billing account in staging lands with a `BillingEntitlement`
  referencing that Offer automatically, within a single reconcile.
- Amberflo console shows the Product Plan and Items (dimension filters honored)
  plus a Customer-Plan assignment per account.
- An Amberflo invoice run for an organization with non-zero usage returns a
  non-zero dollar subtotal.
- Staff-portal supports the full Offer authoring lifecycle: list, create-draft,
  edit-draft, publish, view-published, and edit-display-name-only. All mutations
  write through milo with audit-log entries.
- Staff-portal can switch a billing account's active `BillingEntitlement` to a
  different Offer with an audit-log entry; the Amberflo Customer-Plan assignment
  updates within one reconcile.

## Suggested Implementation Phases

1. **Resources + service-catalog fan-out.** Land `ServicePricing`, `Offer`, and
   `BillingEntitlement` types, the `PricingFanOut`, and the default-entitlement
   controller. No Amberflo writes yet.
2. **amberflo-provider reconcilers.** Wire the Product Plan and Customer-Plan
   sync. End-to-end verification: an invoice run returns a non-zero subtotal.
3. **Staff-portal Offer authoring + switcher.** Full CRUD on Offers (draft /
   publish / display-name edit) plus the billing-account Offer-switcher with
   audit. Read-only "active Offer" panel in cloud-portal.
4. **Hand-off to invoice-run controller.** Out of scope here; a separate issue
   is filed once phases 1–3 are green in staging.

## Implementation History

- 2026-06-09: Initial draft extracted from
  [enhancements#758][758] per review feedback to iterate in an enhancement
  document and walk through concrete use cases.

## Drawbacks

Introducing a second "Entitlement" kind alongside `ServiceEntitlement` adds
conceptual surface area. The alternative — renaming or overloading the existing
kind — was judged more invasive and riskier than adding a distinct, clearly
scoped resource.

## Alternatives

- **Mutable Offers.** Allowing published Offers to change rates in place would
  avoid version proliferation but would silently re-price live customers and
  make historical re-rating possible. Rejected in favor of snapshot + immutable
  publish.
- **Pricing as a standalone catalog separate from meters.** Authoring prices far
  from the meters they rate would let pricing and metering drift. Co-locating
  `pricing` on `spec.metrics[]` and fanning out keeps them in lockstep.
- **Overloading `ServiceEntitlement` for billing.** Reusing the per-project
  quota driver for account-level pricing would couple two different scopes and
  require invasive renames. Rejected (see Drawbacks).

<!-- Link references -->

[scope]: ../initial-scope.md
[681]: https://github.com/datum-cloud/enhancements/issues/681
[747]: https://github.com/datum-cloud/enhancements/issues/747
[748]: https://github.com/datum-cloud/enhancements/issues/748
[758]: https://github.com/datum-cloud/enhancements/issues/758
[759]: https://github.com/datum-cloud/enhancements/issues/759
