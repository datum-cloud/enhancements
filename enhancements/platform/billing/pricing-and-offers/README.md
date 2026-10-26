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
    - [Use Case 2: A service ships billable meters and the platform prices them](#use-case-2-a-service-ships-billable-meters-and-the-platform-prices-them)
    - [Use Case 3: The platform raises prices without re-pricing existing customers](#use-case-3-the-platform-raises-prices-without-re-pricing-existing-customers)
    - [Use Case 4: Staff put a specific account on a custom or zero-rated Offer](#use-case-4-staff-put-a-specific-account-on-a-custom-or-zero-rated-offer)
    - [Use Case 5: An invoice run produces a non-zero subtotal across all charge types](#use-case-5-an-invoice-run-produces-a-non-zero-subtotal-across-all-charge-types)
    - [Use Case 6: Two Offers price the same usage differently](#use-case-6-two-offers-price-the-same-usage-differently)
  - [Real-world example: Compute](#real-world-example-compute)
  - [Real-world example: AI Assistant](#real-world-example-ai-assistant)
  - [Real-world example: Data Transfer](#real-world-example-data-transfer)
  - [Real-world example: Default Pay As You Go Offer](#real-world-example-default-pay-as-you-go-offer)
  - [How Offers relate to charge types](#how-offers-relate-to-charge-types)
  - [Composition with quota](#composition-with-quota)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Resource Topology](#resource-topology)
  - [ServicePricing schema](#servicepricing-schema)
  - [Authoring pricing in GitOps](#authoring-pricing-in-gitops)
  - [Validating pricing against meters](#validating-pricing-against-meters)
  - [Offer](#offer)
  - [BillingEntitlement](#billingentitlement)
  - [Default Offer policy](#default-offer-policy)
  - [amberflo-provider reconcilers](#amberflo-provider-reconcilers)
  - [Display names](#display-names)
  - [Boundary with credits](#boundary-with-credits)
  - [Migration from spec.charges\[\]](#migration-from-speccharges)
- [Acceptance Criteria](#acceptance-criteria)
- [Suggested Implementation Phases](#suggested-implementation-phases)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Competitive Research](#competitive-research)

## Summary

We can measure what customers use, but we can't yet charge them for it.
[Metering][681] gave us usage data and the `BillingAccount`, but no prices are
attached to anything, so when we invoice we'll have no amount to charge.

This enhancement doc proposes the missing pieces. The platform prices meters
with **ServicePricing** resources kept in GitOps, bundles those into named
**Offers** (pay-as-you-go, Pro, and so on), and every billing account gets one
by default. Pricing selects the meters it bills. A meter carries no price, so
the same usage can have a different price in each Offer. Once that is in
place, an Amberflo invoice run finally returns a real dollar amount, which is
the input the credit ledger and eventual Stripe charge step depend on.

> **Revision (2026-10-21).** The previous revision had service owners write
> prices as `spec.charges[]` on their `ServiceConfiguration`, and service-catalog
> turned each charge into one `ServicePricing`. That made every price part of
> the service's own meter document. A second price for the same usage needed a
> PR to the service's repo, and so did every rate change, because published
> charges can't be edited. This revision makes `ServicePricing` a standalone
> resource that selects the meter it prices, so one meter can have many
> prices. Prices stay in GitOps in each service's repo, next to its meters. See
> [Migration from spec.charges[]](#migration-from-speccharges) and
> [Alternatives](#alternatives).

## Motivation

The umbrella billing enhancement names three primitives: Service Pricing,
Offers, and Entitlements: and the current implementation has none of them.
Concrete blockers today:

- **Amberflo has meters but no rates**, so any invoice run returns zero.
- **New organizations have no pricing context.** Even after we build the
  invoice-run controller, there is nothing for it to bill.
- **There is no platform-defined way to roll out a tier change** (for example,
  "Pro now includes egress": see
  [Real-world example: Data Transfer](#real-world-example-data-transfer))
  without hand-editing every billing account.
- **Quota gating and pricing are unconnected.** `ServiceEntitlement` (per
  project, in service-catalog) drives quota grants but does not connect to
  pricing, even though the umbrella explicitly says the two should be linked.
- **A price is a commercial decision, not a property of a meter.** The same
  vCPU-second should be billable at a pay-as-you-go rate, a Pro rate, and a
  negotiated rate without the compute team shipping anything.

### Goals

- **Standalone `ServicePricing`, kept in GitOps in each service's repo.** A Usage
  `ServicePricing` selects one meter by name and carries currency,
  `pricingUnit`, an optional `unitConversion`, and rates (flat or tiered,
  including graduated volume bands, with per-dimension match so rates can vary
  by meter dimension value). Any number of `ServicePricing`s may select the
  same meter. OneTime and Recurring `ServicePricing`s name a service and carry
  a fixed amount.
- **Meters carry no price.** `ServiceConfiguration` declares metrics; nothing
  on it is commercial.
- **Admission validation against `MeterDefinition`**, so a price written as a
  separate object from its meter is still checked against it: the meter exists
  and is not retired, rates only match dimensions the meter declares, and the
  pricing names the service that owns the meter.
- **`Offer` resource** that bundles service pricings into a named, versioned
  tier with `launchStage` semantics and a snapshot of the referenced pricings
  taken at publish time.
- **`Offer.spec.chargeTypes` required**, a set of charge types the Offer covers:
  `Usage` (metered consumption), `OneTime` (fixed charge at a trigger), and
  `Recurring` (fixed charge each billing cycle).
- **`BillingEntitlement` resource**, billing-account-scoped, with exactly one
  active per account, distinct from the existing per-project
  `ServiceEntitlement`.
- **Default Offer per billing account**, applied automatically by a new
  service-catalog controller on `BillingAccount` create.
- **Opt-in quota linkage via `spec.billing.quotaGating`** on
  `ServiceConfiguration`. By default quota is granted independently of billing
  via the existing `OrganizationDefaultsReconciler`. A service owner sets
  `quotaGating: BillingEntitlement` to opt in, after which quota for that
  service is granted only when it appears in the account's active Offer.
  The `ServiceEntitlementReconciler.ensureQuotaGrants` controller (new) enforces
  this for opted-in services.
- **`amberflo-provider` reconcilers** that sync `Offer` to a Product Plan and
  `BillingEntitlement` to a Customer-Plan assignment.
- **Offer authoring and override in staff-portal**, including the full
  draft → publish lifecycle and the ability to switch a billing account's
  active Offer with an audit-log entry.
- **Retire `ServiceConfiguration.spec.charges[]`** and the `ChargeFanOut`
  controller that turns charges into `ServicePricing`s.

We will know this has succeeded when a new billing account in staging lands on
a default Offer automatically, an invoice run for an organization with real
usage returns a non-zero dollar subtotal, and two Offers bill the same usage at
different rates without any change to the service that emits it.

### Non-Goals

- **Commitments and subscriptions.** The `chargeTypes` field reserves room for
  these, but they are not part of the initial scope.
- **Real-time rating, spend caps, and spend alerts.** Monetary limits and
  alerting (for example, "don't let an account spend more than $100") depend on
  rated-usage data that this enhancement makes possible but does not itself
  deliver; that work is tracked separately in [#759][759].
- **Consumer-facing Offer browse and self-serve switch.** Cloud-portal shows the
  active Offer name read-only; there is no compare or accept-offer flow.
- **Suspension, dunning, and refunds**, which live with the invoice-run
  controller and Stripe.
- **Multi-currency (USD only), discounting, marketplace billing, and cross-meter
  bundle discounts.**
- **Re-rating historical usage**: prevented by construction via immutable
  published Offers.
- **Writing `ServicePricing` in staff-portal.** Prices go through code review
  in git like meters do. Staff-portal shows them read-only.

## Proposal

### Concepts

| Concept                 | Owner             | Authored in                     | What it does                                                        |
| ----------------------- | ----------------- | ------------------------------- | ------------------------------------------------------------------- |
| **Meter**               | Service owner     | Service's `ServiceConfiguration` | What is measured. Fanned out to a `MeterDefinition`. No price.      |
| **Service Pricing**     | Platform owner    | Service repo (private: infra)   | A price for one meter (Usage), or a fixed fee (OneTime, Recurring). |
| **Offer**               | Platform owner    | staff-portal                    | A named, versioned bundle of service pricings (a "tier").           |
| **Billing Entitlement** | Platform / policy | Controller, staff-portal        | Binds a billing account to exactly one active Offer.                |

Service owners declare meters on their `ServiceConfiguration`. Meters describe
what is measured and nothing else. `ServicePricing` resources sit beside the
meters in the same service repo, as their own manifests; billing owns the
prices and approves changes to them. A Usage pricing selects a meter by name and says what that
usage costs; OneTime and Recurring pricings name a service and need no meter.
Several pricings can select the same meter, which is how one kind of usage gets
more than one price. The platform composes pricings into Offers. A billing
account becomes billable by pointing a Billing Entitlement at an Offer.
Everything downstream (Amberflo Product Plan, Customer-Plan assignment,
invoice) is derived from those objects.

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
reconcile, with no manual step.

#### Use Case 2: A service ships billable meters and the platform prices them

**Situation.** The owner of `compute.datumapis.com` meters allocated vCPUs and
memory, labelled by service tier. The platform wants to charge for them at rates
that differ by tier (`Usage`), collect a one-time setup fee when an account first
activates compute (`OneTime`), and collect a monthly platform fee (`Recurring`).
For an AI service, the platform wants `claude-sonnet-4` priced differently from
`claude-opus-4` on the same meter.

**How the proposal addresses it.** The compute owner declares the metrics and
their dimensions on compute's `ServiceConfiguration`, as they do today, and sets
no prices on it. The prices go in `ServicePricing` manifests in compute's repo,
in the same per-environment component that carries the configuration, with
billing approving them as CODEOWNER. Usage pricings set `metric` to the meter's name and attach rates (with
optional per-dimension match on labels such as `region`, `tier`, or `model`).
OneTime and Recurring pricings set a fixed `amount` plus a `trigger` or
`interval`. Flux applies them to `milo-system`, and the billing webhook checks
each Usage pricing against its `MeterDefinition` before it is stored. Providers
watch the single `ServicePricing` shape, distinguished by `chargeType`.

#### Use Case 3: The platform raises prices without re-pricing existing customers

**Situation.** Pro should now include more resources, and CPU rates are going up.
Existing Pro customers must keep their current rates until they are explicitly
moved; only new or migrated accounts should get the new pricing.

**How the proposal addresses it.** A `ServicePricing`'s spec can't be changed
once it exists, so the new CPU rate is a new pricing (`compute-cpu-allocated-pro-v2`)
added in git next to the old one. `Offer.spec.servicePricings[]` is a
**snapshot** copied from the referenced `ServicePricing`s at publish time, and a
published Offer is immutable except for its display-name annotation. Rolling out
new pricing means publishing a new Offer version (`pro-v2`) that references the
new pricing and moving accounts to it deliberately. Historical re-rating is
impossible by construction.

#### Use Case 4: Staff put a specific account on a custom or zero-rated Offer

**Situation.** Support needs to move one billing account onto an internal,
zero-rated Offer (`staff-zero-v1`) for testing, or onto a negotiated tier: and
the change must be auditable.

**How the proposal addresses it.** Staff-portal can switch a billing account's
`BillingEntitlement.spec.offerRef` to any published Offer. The mutation writes
through milo with an audit-log entry, and the `amberflo-provider` updates the
account's Customer-Plan assignment within one reconcile. Because exactly one
`BillingEntitlement` is active per account, there is never ambiguity about which
Offer is in force. A negotiated rate is a `ServicePricing` for that deal,
selecting the same meter as the list price, and an Offer that references it.
Negotiated rates go in the private pricing directory in infra rather than the
public service repo (see [Authoring pricing in GitOps](#authoring-pricing-in-gitops)).

#### Use Case 5: An invoice run produces a non-zero subtotal across all charge types

**Situation.** An organization has been active for a billing period. They have
metered usage, a setup fee from initial activation, and a monthly platform fee.
We need Amberflo to return the full amount they owe.

**How the proposal addresses it.** The `amberflo-provider` reconciles each
`Offer` into an Amberflo Product Plan with items covering all charge types:
Product Plan Items with dimension filters for `Usage`, a setup fee item for
`OneTime`, and a recurring fixed item for `Recurring`. Each `BillingEntitlement`
maps to a Customer-Plan assignment. With all charge types attached and a plan
assigned, an Amberflo invoice run for the billed period returns a non-zero
subtotal that covers every line item in the Offer. That subtotal is the input
the credit ledger ([#747][747]) draws against.

#### Use Case 6: Two Offers price the same usage differently

**Situation.** Pro accounts should pay $0.020 per vCPU-hour for allocated CPU
while pay-as-you-go accounts pay $0.025, both measured by the same
`compute.datumapis.com/instance/cpu-allocated` meter.

**How the proposal addresses it.** Two `ServicePricing`s select that meter:
`compute-cpu-allocated-v1` and `compute-cpu-allocated-pro-v1`. The
pay-as-you-go Offer references the first and the Pro Offer the second. Compute
ships nothing and its meter doesn't change. Providers already support this:
each Offer becomes its own plan, and the meter (with its dimension filter) is
shared between plans. Within one Offer a meter can still be priced only once,
because two pricings on the same meter would bill the same usage twice; the
Offer controller refuses to publish an Offer that does that.

### Real-world example: Compute

This section walks through what each repository contributes to make compute
billing work end-to-end.

#### datum-cloud/compute: ServiceConfiguration

The compute `ServiceConfiguration` already declares `region` and `tier` as
labels on the `compute.datumapis.com/Instance` monitored resource type, and
publishes five billing metrics. It carries no prices.

```yaml
spec:
  billing:
    quotaGating: BillingEntitlement

  metrics:
    - name: compute.datumapis.com/instance/cpu-allocated
      # ... kind, unit, dimensions: [tier, region] ...
    - name: compute.datumapis.com/instance/memory-allocated
    - name: compute.datumapis.com/instance/cpu-seconds
    - name: compute.datumapis.com/instance/memory-seconds
    - name: compute.datumapis.com/instance/uptime-seconds
```

`cpu-allocated` and `memory-allocated` are Gauges (instantaneous snapshots of
reserved capacity). Pricing those means an account pays for what it has
reserved. `cpu-seconds` and `memory-seconds` are Cumulatives (actual
consumption). Any of them can be priced; which ones apply to a given account
is determined by the pricings its Offer selects, not by compute. That lets the
platform offer an allocated-capacity model, a consumption model, or a future
combination from the same meters. A meter no pricing selects stays unbilled
(telemetry or quota only). Compute opts in to billing-gated quota so accounts
without an active Offer are blocked from creating resources.

#### datum-cloud/compute: ServicePricing

Compute's meters are priced in
`config/components/service-catalog-staging/service-pricing.yaml`, a resource
in the same staging component that infra already pulls in. Production gets its
own component, so the two environments can price differently. `metadata.name` is
the platform's name for the price, not something derived from the meter, and it
carries a version because a pricing's spec never changes. The `pricingUnit` is a
human-readable label for the line item; it does not need to be the literal UCUM
unit string.

```yaml
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: compute-cpu-allocated-v1
  namespace: milo-system
spec:
  chargeType: Usage
  serviceRef: compute.datumapis.com
  metric: compute.datumapis.com/instance/cpu-allocated
  currency: USD
  pricingUnit: vcpu
  rates:
    - match: { dimension: tier, value: standard }
      flat: "0.025"
    - flat: "0.030"
---
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: compute-memory-allocated-v1
  namespace: milo-system
spec:
  chargeType: Usage
  serviceRef: compute.datumapis.com
  metric: compute.datumapis.com/instance/memory-allocated
  currency: USD
  pricingUnit: gib
  rates:
    - match: { dimension: tier, value: standard }
      flat: "0.003"
    - flat: "0.0035"
---
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: compute-cpu-seconds-v1
  namespace: milo-system
spec:
  chargeType: Usage
  serviceRef: compute.datumapis.com
  metric: compute.datumapis.com/instance/cpu-seconds
  currency: USD
  pricingUnit: cpu-second
  rates:
    - match: { dimension: region, value: us-central1 }
      flat: "0.0000125"
    - flat: "0.0000130"
---
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: compute-memory-seconds-v1
  namespace: milo-system
spec:
  chargeType: Usage
  serviceRef: compute.datumapis.com
  metric: compute.datumapis.com/instance/memory-seconds
  currency: USD
  pricingUnit: byte-second
  rates:
    - match: { dimension: region, value: us-central1 }
      flat: "0.000000000008"
    - flat: "0.0000000000085"
---
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: compute-uptime-seconds-v1
  namespace: milo-system
spec:
  chargeType: Usage
  serviceRef: compute.datumapis.com
  metric: compute.datumapis.com/instance/uptime-seconds
  currency: USD
  pricingUnit: instance-second
  rates:
    - flat: "0.000001"
---
# Same meter as compute-cpu-allocated-v1, at the Pro price point.
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: compute-cpu-allocated-pro-v1
  namespace: milo-system
spec:
  chargeType: Usage
  serviceRef: compute.datumapis.com
  metric: compute.datumapis.com/instance/cpu-allocated
  currency: USD
  pricingUnit: vcpu
  rates:
    - match: { dimension: tier, value: standard }
      flat: "0.020"
    - flat: "0.024"
```

Six `ServicePricing`s price five meters. `cpu-allocated` has two prices; see
[Offer](#offer) for the Pro Offer that uses the second one.

#### staff-portal: Offers

Two Offers are authored for compute in draft form, each referencing a different
subset of the `ServicePricing` resources by name. The rate values are not
duplicated here: the controller snapshots them automatically at publish.

```yaml
apiVersion: billing.miloapis.com/v1alpha1
kind: Offer
metadata:
  name: compute-allocated-v1
  annotations:
    kubernetes.io/display-name: "Compute (Reserved Capacity)"
spec:
  chargeTypes: [Usage]
  launchStage: Draft
  servicePricingRefs:
    - name: compute-cpu-allocated-v1
    - name: compute-memory-allocated-v1
---
apiVersion: billing.miloapis.com/v1alpha1
kind: Offer
metadata:
  name: compute-consumed-v1
  annotations:
    kubernetes.io/display-name: "Compute (Pay As You Go)"
spec:
  chargeTypes: [Usage]
  launchStage: Draft
  servicePricingRefs:
    - name: compute-cpu-seconds-v1
    - name: compute-memory-seconds-v1
    - name: compute-uptime-seconds-v1
```

#### datum-cloud/service-catalog: BillingEntitlement (applied by controller)

When a `BillingAccount` is created the `billing-entitlement-defaults` controller
reads `ServiceConfiguration.spec.defaultOffer` and SSA-applies a
`BillingEntitlement`. Staff can later switch the `offerRef` to any published
Offer via staff-portal.

```yaml
apiVersion: billing.miloapis.com/v1alpha1
kind: BillingEntitlement
metadata:
  name: be-<bauid>-default
  namespace: <billing-account-namespace>
spec:
  billingAccountRef:
    name: <billing-account>
  offerRef:
    name: compute-allocated-v1
```

#### datum-cloud/amberflo-provider: new reconcilers

Two new reconcilers are added:

**Offer → Amberflo Product Plan.** Watches published `Offer` resources. For
each entry in `spec.servicePricings[]`, creates an Amberflo Product Plan Item
with the rate and, where present, a dimension filter (e.g.
`tier=standard`). The Product Plan is named after the Offer. Finalizer blocks
deletion until the Product Plan is removed from Amberflo.

**BillingEntitlement → Amberflo Customer-Plan assignment.** Watches
`BillingEntitlement` resources. Resolves `spec.billingAccountRef` to an
Amberflo customer ID and `spec.offerRef` to an Amberflo Product Plan ID, then
creates or updates the Customer-Plan assignment. On `offerRef` change, swaps
the assignment atomically. Finalizer blocks deletion until the assignment is
removed from Amberflo.

Both reconcilers follow the finalizer + SSA pattern used by the existing meter
and customer sync.

#### What this produces in Amberflo

For an account on `compute-allocated-v1` running a standard-tier instance with
2 vCPUs and 4 GiB for one hour:

| Meter | Reserved | Rate | Subtotal |
|---|---|---|---|
| `instance/cpu-allocated` | 2 vCPU | $0.025/vCPU/hr | $0.05 |
| `instance/memory-allocated` | 4 GiB | $0.003/GiB/hr | $0.012 |
| **Total** | | | **$0.062/hr** |

For an account on `compute-consumed-v1` in `us-central1` with the same instance
running for one hour:

| Meter | Usage | Rate | Subtotal |
|---|---|---|---|
| `instance/cpu-seconds` | 7,200 cpu-sec | $0.0000125 | $0.09 |
| `instance/memory-seconds` | 3.87 × 10¹⁰ byte-sec | $0.000000000008 | $0.31 |
| `instance/uptime-seconds` | 3,600 instance-sec | $0.000001 | $0.0036 |
| **Total** | | | **~$0.40/hr** |

### Real-world example: AI Assistant

This section shows the same end-to-end pattern for the AI Assistant service,
which demonstrates per-`model` dimension pricing and a `Recurring` platform fee
alongside `Usage` charges.

#### datum-cloud/cloud-portal: ServiceConfiguration

The `assistant-miloapis-com` `ServiceConfiguration` already declares `model`
and `region` as labels on the `assistant.miloapis.com/Conversation` monitored
resource type, and declares its token and message meters. It carries no
prices.

```yaml
spec:
  billing:
    quotaGating: BillingEntitlement

  metrics:
    - name: assistant.miloapis.com/conversation/input-tokens
    - name: assistant.miloapis.com/conversation/output-tokens
    - name: assistant.miloapis.com/conversation/cache-read-tokens
    - name: assistant.miloapis.com/conversation/cache-write-tokens
    - name: assistant.miloapis.com/conversation/messages
```

#### datum-cloud/cloud-portal: ServicePricing

`config/components/service-catalog-staging/service-pricing.yaml` prices the token meters
with rates that vary by `model`, and adds a Recurring access fee. The access
fee has no meter; it names the service with `serviceRef`.

```yaml
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: assistant-input-tokens-v1
  namespace: milo-system
spec:
  chargeType: Usage
  serviceRef: assistant.miloapis.com
  metric: assistant.miloapis.com/conversation/input-tokens
  currency: USD
  pricingUnit: token
  rates:
    - match: { dimension: model, value: claude-sonnet-4-6 }
      flat: "0.000003"
    - match: { dimension: model, value: claude-opus-4-8 }
      flat: "0.000015"
    - flat: "0.000003"
---
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: assistant-output-tokens-v1
  namespace: milo-system
spec:
  chargeType: Usage
  serviceRef: assistant.miloapis.com
  metric: assistant.miloapis.com/conversation/output-tokens
  currency: USD
  pricingUnit: token
  rates:
    - match: { dimension: model, value: claude-sonnet-4-6 }
      flat: "0.000015"
    - match: { dimension: model, value: claude-opus-4-8 }
      flat: "0.000075"
    - flat: "0.000015"
---
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: assistant-cache-read-tokens-v1
  namespace: milo-system
spec:
  chargeType: Usage
  serviceRef: assistant.miloapis.com
  metric: assistant.miloapis.com/conversation/cache-read-tokens
  currency: USD
  pricingUnit: token
  rates:
    - match: { dimension: model, value: claude-sonnet-4-6 }
      flat: "0.0000003"
    - flat: "0.0000003"
---
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: assistant-cache-write-tokens-v1
  namespace: milo-system
spec:
  chargeType: Usage
  serviceRef: assistant.miloapis.com
  metric: assistant.miloapis.com/conversation/cache-write-tokens
  currency: USD
  pricingUnit: token
  rates:
    - match: { dimension: model, value: claude-sonnet-4-6 }
      flat: "0.00000375"
    - flat: "0.00000375"
---
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: assistant-messages-v1
  namespace: milo-system
spec:
  chargeType: Usage
  serviceRef: assistant.miloapis.com
  metric: assistant.miloapis.com/conversation/messages
  currency: USD
  pricingUnit: message
  rates:
    - flat: "0.000001"
---
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: assistant-access-fee-v1
  namespace: milo-system
spec:
  chargeType: Recurring
  serviceRef: assistant.miloapis.com
  displayName: AI Assistant Access Fee
  currency: USD
  amount: "10.00"
  interval: monthly
```

#### staff-portal: Offer

The Offer is authored in draft form, referencing `ServicePricing` resources by
name. Rates are not duplicated: the controller snapshots them at publish.

```yaml
apiVersion: billing.miloapis.com/v1alpha1
kind: Offer
metadata:
  name: assistant-pay-as-you-go-v1
  annotations:
    kubernetes.io/display-name: "AI Assistant (Pay As You Go)"
spec:
  chargeTypes: [Usage, Recurring]
  launchStage: Draft
  servicePricingRefs:
    - name: assistant-input-tokens-v1
    - name: assistant-output-tokens-v1
    - name: assistant-cache-read-tokens-v1
    - name: assistant-cache-write-tokens-v1
    - name: assistant-messages-v1
    - name: assistant-access-fee-v1
```

#### datum-cloud/amberflo-provider

No new reconcilers needed beyond those introduced in the compute example. The
existing `Offer → Product Plan` reconciler maps the `Recurring` entry to an
Amberflo fixed recurring item alongside the Usage Product Plan Items. The
`BillingEntitlement → Customer-Plan assignment` reconciler is unchanged.

#### What this produces in Amberflo

For an account sending 1,000 messages in a month using `claude-sonnet-4-6`
with a typical input/output ratio of 500 input tokens and 1,000 output tokens
per message:

| Line item | Volume | Rate | Subtotal |
|---|---|---|---|
| `conversation/input-tokens` (sonnet) | 500,000 tokens | $0.000003 | $1.50 |
| `conversation/output-tokens` (sonnet) | 1,000,000 tokens | $0.000015 | $15.00 |
| `conversation/messages` | 1,000 messages | $0.000001 | $0.001 |
| AI Assistant Access Fee (monthly) | 1 | $10.00 | $10.00 |
| **Total** | | | **~$26.50/mo** |

### Real-world example: Data Transfer

Product framing: *Ingress is free. First 200 GB/month of egress is free.
Internal traffic stays cheap.*

Data Transfer uses **three separate meters** so each rate shape stays readable
and maps 1:1 to Amberflo line items:

| Meter | Rate shape | Dimension (derived at metering time) |
|---|---|---|
| `networking.datumapis.com/transfer/egress-internet` | Graduated volume tiers by destination geo | `destination_region_group` (`us-eu`, `rest-of-world`) |
| `networking.datumapis.com/transfer/internal` | Flat by path class | `path` (`same-region`, `cross-region-na-eu`, `us-to-row`) |
| `networking.datumapis.com/transfer/ingress` | Flat zero | _(none)_ |

Metric names are illustrative (`networking.datumapis.com/...`); a service owner
may place equivalent meters under another API group. Classification into
`destination_region_group` and `path` is a **metering / label-enrichment**
contract: pricing never matches on raw source+destination region pairs.
Multi-dimension `match` stays deferred; compound keys are emitted as a single
derived dimension. Snapshot storage data-transfer is not metered for transfer
charges.

`pricingUnit` is `gib`. Product copy uses GB/TB; the example maps the published
bands to `200` / `10240` (10 × 1024) / `153600` (150 × 1024) GiB. Confirm the
exact unit boundary with product before production rates ship.

#### datum-cloud/network-services-operator: ServiceConfiguration

```yaml
spec:
  # Data Transfer example uses OrganizationDefault quota gating unless the
  # owning service opts into BillingEntitlement separately.
  metrics:
    - name: networking.datumapis.com/transfer/egress-internet
      # ... dimensions: [destination_region_group] ...
    - name: networking.datumapis.com/transfer/internal
      # ... dimensions: [path] ...
    - name: networking.datumapis.com/transfer/ingress
```

#### datum-cloud/network-services-operator: ServicePricing

`config/components/service-catalog-staging/service-pricing.yaml`:

```yaml
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: transfer-egress-internet-v1
  namespace: milo-system
spec:
  chargeType: Usage
  serviceRef: networking.datumapis.com
  metric: networking.datumapis.com/transfer/egress-internet
  currency: USD
  pricingUnit: gib
  rates:
    - match: { dimension: destination_region_group, value: us-eu }
      tiered:
        - upTo: "200"
          rate: "0"
        - upTo: "10240"
          rate: "0.05"
        - upTo: "153600"
          rate: "0.03"
        - rate: "0.01"
    - match: { dimension: destination_region_group, value: rest-of-world }
      tiered:
        - upTo: "200"
          rate: "0"
        - upTo: "10240"
          rate: "0.15"
        - upTo: "153600"
          rate: "0.12"
        - rate: "0.09"
    # Default: treat unknown geo as Rest of World
    - tiered:
        - upTo: "200"
          rate: "0"
        - upTo: "10240"
          rate: "0.15"
        - upTo: "153600"
          rate: "0.12"
        - rate: "0.09"
---
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: transfer-internal-v1
  namespace: milo-system
spec:
  chargeType: Usage
  serviceRef: networking.datumapis.com
  metric: networking.datumapis.com/transfer/internal
  currency: USD
  pricingUnit: gib
  rates:
    - match: { dimension: path, value: same-region }
      flat: "0"
    - match: { dimension: path, value: cross-region-na-eu }
      flat: "0.01"
    - match: { dimension: path, value: us-to-row }
      flat: "0.05"
    # Default: most expensive internal path
    - flat: "0.05"
---
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: transfer-ingress-v1
  namespace: milo-system
spec:
  chargeType: Usage
  serviceRef: networking.datumapis.com
  metric: networking.datumapis.com/transfer/ingress
  currency: USD
  pricingUnit: gib
  rates:
    - flat: "0"
```

#### staff-portal: Offer

A standalone Offer bundles the three transfer pricings (also referenced from
Default PAYG below):

```yaml
apiVersion: billing.miloapis.com/v1alpha1
kind: Offer
metadata:
  name: data-transfer-pay-as-you-go-v1
  annotations:
    kubernetes.io/display-name: "Data Transfer (Pay As You Go)"
spec:
  chargeTypes: [Usage]
  launchStage: Draft
  servicePricingRefs:
    - name: transfer-egress-internet-v1
    - name: transfer-internal-v1
    - name: transfer-ingress-v1
```

#### datum-cloud/amberflo-provider

No new reconcilers. The existing `Offer → Product Plan` reconciler maps each
`tiered` rate entry to Amberflo graduated LeafNode / TieredPriceGroup tiers
(`startAfterUnit` from prior `upTo`, `pricePerBatch` = `rate`, `batchSize: 1`,
`allowPartialBatch: true`) and attaches dimension filters for
`destination_region_group` / `path` as it does for flat match entries today.
Volume/retroactive tier modes are not expressed in v1.

#### What this produces in Amberflo

Assumptions for one billing-account month (usage aggregated monthly at
billing-account scope for tier breaks):

- 500 GiB internet egress to US/EU → 200 free + 300 × $0.05
- 100 GiB internal same-region → $0
- 50 GiB internal cross-region within NA/EU → 50 × $0.01
- 20 GiB internal US → Rest of World → 20 × $0.05
- 1 TiB ingress → $0

| Line item | Volume | Rate band(s) | Subtotal |
|---|---|---|---|
| `transfer/egress-internet` (`us-eu`) | 500 GiB | 200 @ $0 + 300 @ $0.05/GiB | $15.00 |
| `transfer/internal` (`same-region`) | 100 GiB | $0/GiB | $0.00 |
| `transfer/internal` (`cross-region-na-eu`) | 50 GiB | $0.01/GiB | $0.50 |
| `transfer/internal` (`us-to-row`) | 20 GiB | $0.05/GiB | $1.00 |
| `transfer/ingress` | 1024 GiB | $0/GiB | $0.00 |
| **Total** | | | **$16.50** |

### Real-world example: Default Pay As You Go Offer

This shows how a single default Offer bundles Compute, AI Assistant, and Data
Transfer into the Offer every new billing account lands on automatically.

#### staff-portal: Offer

The `default-pay-as-you-go-v1` Offer references `ServicePricing` resources
for Compute, AI Assistant, and Data Transfer. Compute uses the consumption
model (`cpu-seconds`, `memory-seconds`, `uptime-seconds`); AI Assistant
includes all token meters plus the monthly access fee; Data Transfer includes
internet egress, internal transfer, and the explicit zero-rate ingress meter
so “ingress is free” appears on invoices. A later tier can leave the transfer
pricings out, or select a different egress pricing on the same meter, without
any change to the networking service.

```yaml
apiVersion: billing.miloapis.com/v1alpha1
kind: Offer
metadata:
  name: default-pay-as-you-go-v1
  annotations:
    kubernetes.io/display-name: "Pay As You Go"
spec:
  chargeTypes: [Usage, Recurring]
  launchStage: Draft
  servicePricingRefs:
    - name: compute-cpu-seconds-v1
    - name: compute-memory-seconds-v1
    - name: compute-uptime-seconds-v1
    - name: assistant-input-tokens-v1
    - name: assistant-output-tokens-v1
    - name: assistant-cache-read-tokens-v1
    - name: assistant-cache-write-tokens-v1
    - name: assistant-messages-v1
    - name: assistant-access-fee-v1
    - name: transfer-egress-internet-v1
    - name: transfer-internal-v1
    - name: transfer-ingress-v1
```

#### datum-cloud/service-catalog: ServiceConfiguration.spec.defaultOffer

The `billing-miloapis-com` `ServiceConfiguration` is updated to point to this
Offer so the `billing-entitlement-defaults` controller knows which Offer to
apply to new billing accounts:

```yaml
spec:
  defaultOffer: default-pay-as-you-go-v1
```

#### What this produces in Amberflo

For a new account in its first month running a standard-tier Compute instance
in `us-central1` for one hour and sending 1,000 AI Assistant messages using
`claude-sonnet-4-6`, plus the Data Transfer usage from
[Real-world example: Data Transfer](#real-world-example-data-transfer):

| Line item | Volume | Rate | Subtotal |
|---|---|---|---|
| `instance/cpu-seconds` | 7,200 cpu-sec | $0.0000125 | $0.09 |
| `instance/memory-seconds` | 3.87 × 10¹⁰ byte-sec | $0.000000000008 | $0.31 |
| `instance/uptime-seconds` | 3,600 instance-sec | $0.000001 | $0.0036 |
| `conversation/input-tokens` (sonnet) | 500,000 tokens | $0.000003 | $1.50 |
| `conversation/output-tokens` (sonnet) | 1,000,000 tokens | $0.000015 | $15.00 |
| `conversation/messages` | 1,000 messages | $0.000001 | $0.001 |
| AI Assistant Access Fee (monthly) | 1 | $10.00 | $10.00 |
| Data Transfer (see worked example) |-|-| $16.50 |
| **Total** | | | **~$43.40/mo** |

### How Offers relate to charge types

Every `Offer` carries a required `spec.chargeTypes` field: a set of the charge
types the Offer covers:

| Charge type | What it means | Example |
|---|---|---|
| `Usage` | Metered consumption multiplied by a rate. | $0.025 per vCPU per hour. |
| `OneTime` | A fixed amount charged once, at a defined trigger. | $10 setup fee on first activation. |
| `Recurring` | A fixed amount charged on every billing cycle. | $5/month platform fee. |

A single Offer can bundle all three. The field is required so that controllers
can gate on charge type explicitly. A webhook rejects an Offer that declares a
charge type with no corresponding pricing entries.

### Composition with quota

Quota gating on billing is **opt-in** per service via
`spec.billing.quotaGating` on `ServiceConfiguration`:

| Value | Behaviour |
|---|---|
| `OrganizationDefault` (default) | Quota granted via the existing `OrganizationDefaultsReconciler` independent of billing. The service is always accessible regardless of what Offer is active. |
| `BillingEntitlement` | Quota granted only when a `ServicePricing` whose `serviceRef` names the service appears in the account's active Offer. The service is inaccessible if absent from the Offer. |

Quota stays the service owner's switch even though prices are not theirs: the
service decides whether access depends on an Offer, and the platform decides
which Offers include the service. The billing webhook checks a Usage pricing's
`serviceRef` against the service that owns its meter, so a pricing can't grant
quota for a service other than the one it bills.

Today, quota grants are issued unconditionally by `GrantCreationPolicy` +
`GrantCreationController`: this is the `OrganizationDefault` path and
requires no changes. The `BillingEntitlement` path introduces one new
controller:

- **`ServiceEntitlementReconciler.ensureQuotaGrants`**: watches
  `BillingEntitlement` create and `offerRef` changes; for services that have
  opted in (`quotaGating: BillingEntitlement`), issues quota grants for those
  present in the active Offer and revokes grants for those no longer present.

The activation sequence for an opted-in service is:

1. A `BillingEntitlement` is created (or its `offerRef` changes).
2. `ServiceEntitlementReconciler.ensureQuotaGrants` issues quota grants for
   opted-in services present in the Offer and revokes any no longer present.
3. The `amberflo-provider` syncs the Offer's Product Plan and assigns it to
   the account.

For example, if Compute has opted in (`quotaGating: BillingEntitlement`) and
DNS has not (`OrganizationDefault`): a billing account on an Offer that
includes DNS and Compute `ServicePricing`s has normal quota for both. Remove
Compute from the Offer and Compute quota is revoked: the account can no longer
create Compute resources. DNS quota is unaffected because it is not
billing-gated.

Swapping an account to a different Offer atomically replaces opted-in quota
grants and the Amberflo Customer-Plan assignment in a single reconcile.

### Risks and Mitigations

- **Snapshot drift confusion.** Operators may expect editing a `ServicePricing`
  to change live Offers. _Mitigation:_ a `ServicePricing`'s spec can't be
  edited, so there is nothing to drift; a new rate is a new pricing and a new
  Offer version. Staff-portal surfaces the snapshot explicitly.
- **A price no longer matches its meter.** A pricing is a separate object from
  its meter and nothing in the service's `ServiceConfiguration` points back at
  it, so a pricing could name a meter that doesn't exist or match a dimension
  the meter doesn't emit. _Mitigation:_ the billing webhook
  validates every pricing against its `MeterDefinition` when it is created (see
  [Validating pricing against meters](#validating-pricing-against-meters)), so
  a broken price fails the Flux apply instead of billing nothing.
- **A priced meter is retired.** A retired meter stops accepting usage, so a
  pricing on it never bills. _Mitigation:_ the service-catalog webhook keeps
  today's check but reads `ServicePricing`s in `milo-system` instead of
  `spec.charges[]`: it rejects retiring a metric that a pricing still selects.
  Remove the pricing first.
- **A service team changes a price without billing.** Prices sit in the
  service's repo, where its owners can merge. _Mitigation:_ CODEOWNERS makes
  billing a required reviewer on each repo's pricing files.
- **Negotiated prices in public repos.** compute, network-services-operator, and
  cloud-portal are public. List prices are public anyway; a price negotiated
  for one customer is not. _Mitigation:_ those go in a private pricing
  directory in infra instead.
- **Two "Entitlement" kinds.** `BillingEntitlement` (new) and `ServiceEntitlement`
  (existing per-project quota driver) could be conflated. _Mitigation:_ they are
  deliberately distinct kinds with distinct scopes; the new kind avoids invasive
  renames and is named for the umbrella's terminology.
- **Default Offer misconfiguration.** A wrong `defaultOffer` field would
  mis-price every new account. _Mitigation:_ changes go through staff-portal
  with an audit-log entry, and re-apply is idempotent.
- **Billed but not gated.** A service is priced in an Offer but its owner
  hasn't set `quotaGating: BillingEntitlement`, so accounts are billed for usage
  but never blocked even without an active Offer. _Mitigation:_ service-catalog
  logs a warning when a published Offer prices a service whose `quotaGating`
  is `OrganizationDefault`, so the owner can confirm that intent. (This used to
  be logged by `ChargeFanOut`, which this revision removes.)

## Design Details

### Resource Topology

```text
service repo (GitOps)

ServiceConfiguration                      ServicePricing (milo-system)
  .spec.metrics[]                           .spec.metric  (Usage)
        │                                   any number per meter
        │ fan-out                                 │
        ▼                                         │
MeterDefinition  ◄──── selects, and is ───────────┤
  (no price)           validated against          │
                                                  │ refs
                                                  │
                         Offer (staff-portal) ────┘
                           .spec.servicePricingRefs[]
                           .spec.servicePricings[]  (snapshot at publish)
                                  ▲          │
                                  │ refs     ▼
BillingAccount ──policy──► BillingEntitlement      provider: Product Plan
                             .spec.offerRef         (one per Offer)
                                  │
                                  ▼
                        provider: Customer-Plan assignment
```

### ServicePricing schema

`ServicePricing` keeps the shape it has today. Every pricing has `name`,
`chargeType`, `serviceRef`, `currency`, and an optional `displayName`.
Type-specific fields:

- **`Usage`** requires `metric` (a `MeterDefinition`'s `spec.meterName`),
  `pricingUnit`, and `rates`, and may set `unitConversion`. The rate is
  multiplied by the meter reading (converted into `pricingUnit` first when
  `unitConversion` is set), optionally filtered by a dimension.
- **`OneTime`** requires `amount` and `trigger`. It fires once at that trigger.
- **`Recurring`** requires `amount` and `interval`. It fires every billing
  interval.
- The last band in a `tiered` rate list must omit `upTo` (open-ended). Earlier
  bands require `upTo`.

Each `rates[]` entry carries either `flat` or `tiered`, never both:

- **`flat`** is a single decimal USD string multiplied by metered usage.
- **`tiered`** is ordered graduated volume bands. Each band has a `rate` and
  an exclusive upper bound `upTo` in `pricingUnit` units. The last band omits
  `upTo` (open-ended). Zero (`"0"`) is a valid rate for free allowances.
  Aggregation for tier breaks is monthly, at billing-account scope.
  Volume/retroactive (reprice-all) modes are not expressed in v1.

Per-dimension `match` entries let a rate vary by a dimension the meter declares
(one dimension per match entry; multi-dimension matches deferred). Services
that need a compound key (for example source and destination region) emit a
single derived dimension at metering time; see
[Real-world example: Data Transfer](#real-world-example-data-transfer).

What changes from the previous revision:

- The type's doc comments stop saying `ServicePricing` is fan-out only and must
  not be written by hand.
- The spec is immutable after create. A new price is a new `ServicePricing`,
  which keeps a pricing's name meaning one price for audits, Offer refs, and
  invoices. Stripe makes the same choice for `Price` amounts.
- The webhook validates against `MeterDefinition`, below.

```yaml
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: compute-setup-fee-v1
  namespace: milo-system
spec:
  chargeType: OneTime
  serviceRef: compute.datumapis.com
  displayName: Compute Setup Fee
  currency: USD
  amount: "10.00"
  trigger: BillingAccountActivation
---
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: compute-platform-fee-v1
  namespace: milo-system
spec:
  chargeType: Recurring
  serviceRef: compute.datumapis.com
  displayName: Compute Platform Fee
  currency: USD
  amount: "5.00"
  interval: monthly
```

### Authoring pricing in GitOps

List prices live in each service's repo, beside its meters, in the
per-environment component infra already pulls in. The component adds
`ServicePricing` resources where it used to patch `spec.charges` onto the
`ServiceConfiguration`:

```text
config/components/service-catalog-staging/
  kustomization.yaml          # resources: [service-pricing.yaml]
  service-pricing.yaml        # one document per ServicePricing
```

- The Flux identities that apply each service's component get
  `service-pricing-admin` on `milo-system`. Staff keep `service-pricing-viewer`,
  so staff-portal shows prices but doesn't change them.
- Each repo's CODEOWNERS makes billing a required reviewer on its pricing
  files, so a service team can't change a price on its own.
- Prices that can't be public, such as a rate negotiated for one customer, go
  in `apps/billing-system/<env>/pricing/` in infra, which is private. The
  webhook and naming rules are the same.
- Names are the platform's: `<service>-<what>[-<variant>]-v<n>`, for example
  `compute-cpu-allocated-pro-v1`. The service-name prefix and the
  `host--path` encoding the fan-out used no longer apply.
- Removing a pricing from git deletes it (Flux prunes). GA Offers that
  snapshotted it keep billing. A Draft Offer that still references it can't be
  published and reports `ServicePricingResolveFailed`.
- A new meter and its first price can ship in the same PR. The meter's
  `MeterDefinition` appears only after service-catalog fans out the
  configuration, so the pricing's first apply may be rejected; Flux retries
  until the meter exists.

### Validating pricing against meters

The billing `ServicePricing` webhook is stateless today. It gains a client and
checks, on create:

- **Usage:** `metric` names an existing `MeterDefinition` (by
  `spec.meterName`) that isn't `Retired`.
- **Usage:** every `match.dimension` is in the meter's
  `spec.measurement.dimensions`, or is the system dimension `project_name`.
- **Usage:** matched rates share one dimension, match distinct values, and
  there is at most one unmatched catch-all. openmeter-provider enforces this
  today when it maps an Offer, after the Offer is already published; checking
  at admission moves the failure to the PR.
- **Usage:** `serviceRef` equals the meter's `services.miloapis.com/service`
  label, which the meter fan-out sets.
- **OneTime / Recurring:** `serviceRef` names an existing `Service`.
- **Update:** `spec` is unchanged.

### Offer

`Offer` bundles service pricings into a named tier with `launchStage`
semantics. An Offer has two distinct states:

**Draft**: the platform owner references `ServicePricing` resources by name
via `spec.servicePricingRefs[]`. No rates are stored inline; the Offer is
mutable and not yet usable by accounts.

**Published (GA)**: the platform owner advances `launchStage` to `GA`. The
controller reads each referenced `ServicePricing`, copies the current rates
into `spec.servicePricings[]` as an immutable snapshot, and the Offer becomes
live. Rates are **not** authored twice: the snapshot is generated
automatically at publish time.

An Offer need not reference all the pricings for a service: the subset chosen
determines which services are accessible and billed for accounts on that Offer.
`spec.chargeTypes` must enumerate every charge type present in the referenced
pricings. An Offer may reference at most one Usage pricing per meter; the
controller refuses to publish one that references two, because both would bill
the same usage.

The Pro Offer from [Use Case 6](#use-case-6-two-offers-price-the-same-usage-differently)
selects the Pro price for allocated CPU and the list price for memory:

```yaml
apiVersion: billing.miloapis.com/v1alpha1
kind: Offer
metadata:
  name: compute-pro-v1
  annotations:
    kubernetes.io/display-name: "Compute Pro"
spec:
  chargeTypes: [Usage, OneTime, Recurring]
  launchStage: Draft
  servicePricingRefs:
    - name: compute-cpu-allocated-pro-v1
    - name: compute-memory-allocated-v1
    - name: compute-setup-fee-v1
    - name: compute-platform-fee-v1
```

On publish the controller expands `servicePricingRefs` into the full
`servicePricings` snapshot. The draft refs are retained for audit; only
`servicePricings` is used at runtime.

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
- `flat` rates map to a single LeafNode price as today. `tiered` rates map to
  Amberflo graduated LeafNode / TieredPriceGroup tiers: band `i` becomes
  `{ startAfterUnit, batchSize: 1, pricePerBatch: rate, allowPartialBatch: true }`
  where `startAfterUnit` is `0` for the first band and the previous band's
  `upTo` thereafter; the open-ended last band has no following tier.

Providers read the Offer snapshot, not `ServicePricing` or
`ServiceConfiguration`, so moving who writes prices needs no provider change.
Pricing the same meter in two Offers needs none either: each Offer is its own
plan. openmeter-provider already shares the meter-and-filter feature between
plans and puts each Offer's price on its own rate card.

### Display names

`Offer` carries a `kubernetes.io/display-name` annotation (matching the
convention used on milo `Role`s) so the human-readable name can change without
bumping the Offer version. Each `ServicePricing` should set `displayName`. When
a Usage pricing omits it, providers fall back to the meter's
`spec.displayName`. (The fan-out used to fill this in from the metric; with no
fan-out, the fallback moves to the providers.) Fixed charges use the pricing's
`displayName`. `BillingEntitlement` defers to its Offer's display name.

### Boundary with credits

Invoice-run sequence: Amberflo computes the subtotal using Offer rates → the
credit ledger ([#747][747]) draws down → the remainder hits Stripe. The
invoice-run controller that orchestrates this step is out of scope here and
will be designed in a follow-on enhancement once phases 1-4 are green in
staging. #747 designs against the Amberflo subtotal as its input.

### Migration from spec.charges[]

Charges exist only in staging today, in three service repos (compute,
network-services-operator, cloud-portal), and no one is being charged. GA
Offers hold their own copy of every price, so the old pricings can go away
without changing what a live Offer bills.

1. **billing.** Add the `MeterDefinition` checks and spec immutability to the
   `ServicePricing` webhook, and drop "must not be authored by hand" from the
   type docs. The Offer controller refuses to publish two Usage pricings on one
   meter.
2. **infra.** Grant the Flux identities that apply the service components
   `service-pricing-admin`.
3. **Service repos.** In each of the three staging components, replace the
   `spec.charges` patch with `ServicePricing` resources, one per existing
   charge, under new names, and add billing to CODEOWNERS for them. The old
   pricings are owned by their `ServiceConfiguration` (owner reference and the
   `services-operator-charges` field manager), so reusing their names would
   fight the fan-out. Dropping the charges makes `ChargeFanOut` prune the old
   pricings; the webhook already allows removing a published charge.
4. **staff-portal.** Publish new Offer versions that reference the new
   pricings, point `defaultOffer` at the new default, and move existing
   entitlements with `migrateFromOffer`. Accounts on the old Offers keep
   billing from their snapshots until they move.
5. **service-catalog.** Warn on `spec.charges`, then reject it, and remove
   `ChargeFanOut` and the charge types in the next API version. Point the
   retired-metric check at `ServicePricing`s. Staff-portal's service page lists
   the pricings that select the service's meters instead of its charges.

## Acceptance Criteria

- The `ServiceConfiguration` for `compute.datumapis.com` declares its five
  billing metrics (`instance/cpu-allocated`, `instance/memory-allocated`,
  `instance/cpu-seconds`, `instance/memory-seconds`, and
  `instance/uptime-seconds`) and no charges.
- Compute's staging component holds a `ServicePricing` for each of those
  metrics, Flux applies them to `milo-system`, and billing is a required
  reviewer on that file.
- Two `ServicePricing`s select `instance/cpu-allocated` at different rates. Two
  published Offers each reference one of them, and accounts on each are billed
  at their own Offer's rate for the same usage.
- The billing webhook rejects a `ServicePricing` whose meter doesn't exist or is
  retired, that matches a dimension the meter doesn't declare, or whose
  `serviceRef` isn't the meter's service. It rejects any change to a pricing's
  spec.
- The Offer controller refuses to publish an Offer with two Usage pricings on
  the same meter.
- Two published Offers exist: `compute-allocated-v1` (bundles
  `cpu-allocated` + `memory-allocated`) and `compute-consumed-v1` (bundles
  `cpu-seconds` + `memory-seconds` + `uptime-seconds`). Dimension filters are
  honored in the Amberflo Product Plan Items for each.
- Every new billing account in staging lands with a `BillingEntitlement`
  referencing the default Offer automatically, within a single reconcile.
- Quota grants are issued only for services present in the active Offer when
  the service opts into `quotaGating: BillingEntitlement`; switching an account
  to an Offer that excludes Compute revokes the Compute quota grant and blocks
  Compute resource creation.
- Amberflo console shows the Product Plan and Items for both Offers, plus a
  Customer-Plan assignment per account reflecting the active Offer.
- An Amberflo invoice run for an organization with non-zero compute usage
  returns a non-zero dollar subtotal.
- Staff-portal supports the full Offer authoring lifecycle: list, create-draft,
  edit-draft, publish, view-published, and edit-display-name-only. All mutations
  write through milo with audit-log entries.
- Staff-portal can switch a billing account's active `BillingEntitlement` to a
  different Offer with an audit-log entry; the Amberflo Customer-Plan assignment
  updates within one reconcile.
- The `ServicePricing` schema accepts graduated `tiered` rate entries (`upTo` /
  `rate`, mutually exclusive with `flat` on the same entry).
- The Data Transfer rates in
  [Real-world example: Data Transfer](#real-world-example-data-transfer)
  (internet egress by geo with volume bands, internal path flats, free ingress)
  are authorable as `ServicePricing` snapshots on a published Offer.
- No `ServiceConfiguration` sets `spec.charges`, and `ChargeFanOut` is removed.

## Suggested Implementation Phases

1. **Resources + service-catalog fan-out.** Land `ServicePricing`, `Offer`, and
   `BillingEntitlement` types, `ChargeFanOut`, and the default-entitlement
   controller. No Amberflo writes yet.
2. **Quota linkage.** Build `OrganizationDefaultsReconciler` and
   `ServiceEntitlementReconciler.ensureQuotaGrants` to gate quota grants on
   the services present in the active Offer. Verify that an account whose Offer
   excludes Compute cannot create Compute resources.
3. **amberflo-provider reconcilers.** Wire the Product Plan and Customer-Plan
   sync. End-to-end verification: an invoice run returns a non-zero subtotal.
4. **Staff-portal Offer authoring + switcher.** Full CRUD on Offers (draft /
   publish / display-name edit) plus the billing-account Offer-switcher with
   audit. Read-only "active Offer" panel in cloud-portal.
5. **Hand-off to invoice-run controller.** Out of scope here; a separate issue
   is filed once phases 1-4 are green in staging.
6. **Decouple pricing from meters.** The five steps in
   [Migration from spec.charges[]](#migration-from-speccharges), in order.
   This should land before production prices are set, so the old model never
   reaches production.

## Implementation History

- 2026-06-09: Initial draft extracted from
  [enhancements#758][758] per review feedback to iterate in an enhancement
  document and walk through concrete use cases.
- 2026-07-22: Document graduated `tiered` rates and a Data Transfer real-world
  example (internet egress by geo, internal path rates, free ingress); wire
  Data Transfer into the Default PAYG Offer.
- 2026-08-04: Unify Usage onto `spec.charges[]` with `metricRef`. Drop
  `metrics[].pricing` and the separate `PricingFanOut`. Metrics stay
  telemetry/quota-only; one `ChargeFanOut` covers Usage, OneTime, and
  Recurring (service-catalog PR #61). Nest type-specific fields under
  `usage` / `oneTime` / `recurring`, and require the last tiered band to
  omit `upTo`.
- 2026-10-21: Decouple pricing from meters. `ServicePricing` is a standalone
  manifest in each service's repo (private prices in infra) and selects the
  meter it prices; any number of pricings may select the same meter. Validate pricings against
  `MeterDefinition` at admission and make their spec immutable. Retire
  `ServiceConfiguration.spec.charges[]` and `ChargeFanOut`.

## Drawbacks

Introducing a second "Entitlement" kind alongside `ServiceEntitlement` adds
conceptual surface area. The alternative (renaming or overloading the existing
kind) was judged more invasive and riskier than adding a distinct, clearly
scoped resource.

An Offer's prices are spread across several service repos, so a change that
touches every service (a new Pro tier, say) is one PR per service. Offers
themselves are still assembled in one place, in staff-portal.

Immutable pricings mean versioned names accumulate in the pricing directory.
A pricing can be deleted once no Draft Offer references it, since GA Offers
keep their own copy.

## Alternatives

- **Mutable Offers.** Allowing published Offers to change rates in place would
  avoid version proliferation but would silently re-price live customers and
  make historical re-rating possible. Rejected in favor of snapshot + immutable
  publish.
- **Pricing nested on `spec.metrics[]`.** An earlier draft put Usage rates on
  each metric. That forces every priced meter to look "commercial" and leaves
  no shared place for OneTime/Recurring. Rejected.
- **Charges on `ServiceConfiguration` (the previous revision).** Service owners
  declared `spec.charges[]` next to their meters, and service-catalog turned
  each charge into one `ServicePricing`. It kept prices next to the meters they
  rate, which was the argument for it: prices written elsewhere could drift
  from meters without a clear owner. In practice it made the service owner the
  author of every price. A second price for the same meter, or any rate change,
  needed a PR to the service's repo and a new immutable charge there, and
  charge names took the service's prefix. Prices become standalone
  `ServicePricing`s that select meters. They stay in the service's repo, which
  keeps them next to the meters, and the webhook validates each one against
  its `MeterDefinition`.
- **All prices in infra.** One private place for the platform's prices, apart
  from the services. Rejected: it adds a second deploy path for something the
  service components already ship, and it separates prices from the meters
  they rate. Only prices that can't be public go there.
- **Rates inline on the Offer.** The Offer carries prices directly, each
  selecting a meter, with no `ServicePricing`. That is closest to OpenMeter's
  plan → rate card → meter. Rejected for now: one rate couldn't be shared
  across Offers without copying it, and it changes the Offer schema and every
  consumer of the snapshot (both providers, quota gating, staff-portal) for
  little gain over changing who writes `ServicePricing`.
- **Writing `ServicePricing` in staff-portal.** Rejected: prices should be
  code-reviewed and reproducible per environment, like meters.
- **Overloading `ServiceEntitlement` for billing.** Reusing the per-project
  quota driver for account-level pricing would couple two different scopes and
  require invasive renames. Rejected (see Drawbacks).
- **Multi-dimension `match` for Data Transfer.** Matching on raw
  `source_region` × `destination_region` (or a single mega-meter with
  `direction` + geo + path) would un-defer multi-dimension matches for every
  service. Rejected: keep one dimension per match entry and require metering to
  emit derived labels (`destination_region_group`, `path`) on separate meters
  for internet egress, internal transfer, and ingress.

## Competitive Research

The table below summarises how major billing systems model the same concepts,
verified against primary sources (API references, proto definitions, and open
source implementations).

### Primitive mapping

| Platform | Meter definition | Per-meter pricing | Plan / offer | Account assignment | Quota gate? |
|---|---|---|---|---|---|
| **GCP** | `ServiceConfiguration` → `MonitoredResource` + DELTA metrics | `services.skus.list` → `pricingExpression` with `tieredRates` | SKU category (resourceFamily/resourceGroup) + account-level custom pricing API | Committed Use Discounts, negotiated contracts | No: access is separate from billing |
| **AWS** | `serviceCode` + `attributes` block in Price List JSON | `terms.OnDemand` / `terms.Reserved` blocks per SKU | "Offer" in AWS parlance = the `terms` section; one product → many pricing terms | Savings Plans; Marketplace contract | Marketplace only: `GetEntitlements` call required before serving |
| **Azure Marketplace** | Dimension declared per Plan with `includedQuantity` | Flat-rate or per-dimension overage via Metering Service API | `Offer` (top-level) → `Plan` (variant with independent pricing) | `Subscription` object via SaaS Fulfillment API (`Subscribed` state) | Yes: `RegisterUsage` must succeed at container start |
| **Amberflo** | `Meter` (CloudEvents aggregation) | `ProductPlan` → price phases per meter | `ProductPlan` bundles meters + cadence + feature limits | `CustomerPlan` = ProductPlan instance with effective date + overrides | Optional per-plan feature entitlements |
| **Stripe** | `Meter` (event aggregation) | `Price` object (flat, per-seat, metered, tiered) attached to `Product` | `Product` + set of `Price`s; `PricingTable` groups products | `Subscription` → activates `ActiveEntitlement` per mapped feature | No: Stripe documents entitlements, does not enforce them |
| **OpenMeter** | `Meter` (CloudEvents) | `RateCard` (tiered, volume, flat, usage) per `Feature` | `Plan` bundles RateCards with a billing cadence | `Subscription` + `Entitlement` (real-time quota balance) | Yes: `Entitlement` tracks balance; grace period configurable |
| **Zuora** | (external metering) | `ProductRatePlanCharge` (one-time / recurring / usage) | `ProductRatePlan` bundles charges; `Product` is the catalog entry | `Subscription` → `Invoice` | No |

### Key findings

**Meters belong to the service; prices belong to whoever sells it.** GCP's
Service Infrastructure declares `MonitoredResource` types and DELTA metrics on
a service config, and Cloud Billing, not the service, attaches SKU pricing to
them. Stripe, OpenMeter, and Amberflo all let many prices select one meter: a
Stripe `Price` references a `Meter`, an OpenMeter `RateCard` in each `Plan`
points at a feature over a meter, and each Amberflo `ProductPlan` prices
meters independently. This enhancement keeps meters on `ServiceConfiguration`
and adds `ServicePricing`s, owned by billing, that select them.
([source](https://cloud.google.com/service-infrastructure/docs/reporting-billing-metrics))

**Offer → BillingEntitlement is a well-established pattern.**
Amberflo's `ProductPlan → CustomerPlan`, Azure's `Offer → Plan → Subscription`,
AWS Marketplace's entitlement contract, and Stripe's `Product → ActiveEntitlement`
all follow the same shape: a reusable pricing template (the Offer) plus a
per-account assignment record (the BillingEntitlement). The assignment record is
what activates metered invoicing and optional access gating.

**`chargeTypes` (Usage / OneTime / Recurring) maps directly to Zuora's
`ProductRatePlanCharge` and Azure's dimension model.**
Every major billing system distinguishes these three charge types. Zuora's
`ProductRatePlanCharge.type` is the canonical reference implementation.

**Quota gating tied to billing is opt-in, not universal.**  
GCP, Stripe, Zuora, and Lago do not enforce access at the billing layer: they
document entitlements and leave enforcement to the application. Only AWS
Marketplace (container products) and Azure Marketplace mandate an entitlement
check at startup. OpenMeter and Schematic explicitly provide enforcement as a
separate layer. This validates the `spec.billing.quotaGating: BillingEntitlement`
opt-in approach: making it the default would be out of step with how the wider
ecosystem works.

**K8s-native BillingEntitlement has direct prior art.**  
The [ControlPlane flux-operator][flux-operator] ships an `EntitlementReconciler`
that watches Namespaces, calls `RegisterUsage()` to obtain an entitlement token,
stores it in a Secret, and requeues every 30 minutes to re-verify. On failure it
purges the Secret, triggering downstream feature gating. The same pattern , 
entitlement as a Kubernetes-reconciled resource with an external billing authority
,  is what this design proposes for `BillingEntitlement`.

**GCP Marketplace for K8s chose Secrets + ConfigMaps over CRDs.**  
The [GCP Marketplace K8s billing integration][gcp-marketplace-k8s] injects a
Reporting Secret (consumer-id, entitlement-id, reporting-key) at deploy time and
uses a sidecar metering agent instead of CRDs. That trade-off (simplicity over
formal resource hierarchy) is documented so future reviewers can revisit whether
the CRD approach in this enhancement remains the right call as complexity grows.

<!-- Link references -->

[scope]: ../initial-scope.md
[681]: https://github.com/datum-cloud/enhancements/issues/681
[747]: https://github.com/datum-cloud/enhancements/issues/747
[748]: https://github.com/datum-cloud/enhancements/issues/748
[758]: https://github.com/datum-cloud/enhancements/issues/758
[759]: https://github.com/datum-cloud/enhancements/issues/759
[flux-operator]: https://glama.ai/mcp/servers/@controlplaneio-fluxcd/flux-operator/blob/06e1897fcf773428155c5f6b9aadb5538169bb5c/internal/controller/entitlement_controller.go
[gcp-marketplace-k8s]: https://github.com/GoogleCloudPlatform/marketplace-k8s-app-tools/blob/master/docs/billing-integration.md
