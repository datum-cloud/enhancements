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
    - [Use Case 2: A service owner configures pricing across all charge types](#use-case-2-a-service-owner-configures-pricing-across-all-charge-types)
    - [Use Case 3: The platform raises prices without re-pricing existing customers](#use-case-3-the-platform-raises-prices-without-re-pricing-existing-customers)
    - [Use Case 4: Staff put a specific account on a custom or zero-rated Offer](#use-case-4-staff-put-a-specific-account-on-a-custom-or-zero-rated-offer)
    - [Use Case 5: An invoice run produces a non-zero subtotal across all charge types](#use-case-5-an-invoice-run-produces-a-non-zero-subtotal-across-all-charge-types)
  - [Real-world example: Compute](#real-world-example-compute)
  - [Real-world example: AI Assistant](#real-world-example-ai-assistant)
  - [Real-world example: Default Pay As You Go Offer](#real-world-example-default-pay-as-you-go-offer)
  - [How Offers relate to charge types](#how-offers-relate-to-charge-types)
  - [Composition with quota](#composition-with-quota)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Resource Topology](#resource-topology)
  - [Charge type schema](#charge-type-schema)
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
an Amberflo invoice run finally returns a real dollar amount — the input the
credit ledger and eventual Stripe charge step depend on.

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

We will know this has succeeded when a new billing account in staging lands on
a default Offer automatically and an invoice run for an organization with
real usage returns a non-zero dollar subtotal.

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

#### Use Case 2: A service owner configures pricing across all charge types

**Situation.** The owner of `compute.datumapis.com` wants to charge for
allocated vCPUs and memory with rates that differ by service tier (`Usage`),
collect a one-time setup fee when an account first activates the service
(`OneTime`), and collect a monthly platform fee (`Recurring`). For an AI
service, the owner wants `claude-sonnet-4` priced differently from
`claude-opus-4` on the same meter.

**How the proposal addresses it.** For `Usage` charges, the owner adds a
`pricing` block to the relevant `spec.metrics[]` entry in
`ServiceConfiguration`, with per-dimension match rules that key off a label
declared on the service's `monitoredResourceType` (`region`, `tier`, `model`,
…). For `OneTime` and `Recurring` charges, the owner adds entries to
`spec.charges[]` with a fixed `amount` and either a `trigger` or `interval`.
Two fan-out controllers emit one `ServicePricing` per entry — `PricingFanOut`
for meter-based Usage, `ChargeFanOut` for fixed charges — each with a
`chargeType` discriminator. Providers watch the single `ServicePricing` shape
rather than parsing `ServiceConfiguration`.

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

### Real-world example: Compute

This section walks through what each repository contributes to make compute
billing work end-to-end.

#### datum-cloud/compute — ServiceConfiguration

The compute `ServiceConfiguration` already declares `region` and `tier` as
labels on the `compute.datumapis.com/Instance` monitored resource type, and
publishes five billing metrics. A `pricing` block is added to each billable
metric. The `pricingUnit` is a human-readable label that reflects the meter's
unit dimension; it does not need to be the literal UCUM unit string.

`cpu-allocated` and `memory-allocated` are Gauges (instantaneous snapshots of
reserved capacity). Pricing these means an account pays for what it has
reserved. `cpu-seconds` and `memory-seconds` are Cumulatives (actual
consumption). Both sets carry pricing blocks; which ones apply to a given
account is determined by the Offer — not the `ServiceConfiguration`. This lets
the platform offer an allocated-capacity model, a consumption model, or a future
combination, all from the same service definition. Compute opts in to
billing-gated quota so accounts without an active Offer are blocked from
creating resources.

```yaml
spec:
  billing:
    quotaGating: BillingEntitlement

  metrics:
    - name: compute.datumapis.com/instance/cpu-allocated
      pricing:
        currency: USD
        pricingUnit: vcpu
        rates:
          - match: { dimension: tier, value: standard }
            flat: "0.025"
          - flat: "0.030"

    - name: compute.datumapis.com/instance/memory-allocated
      pricing:
        currency: USD
        pricingUnit: gib
        rates:
          - match: { dimension: tier, value: standard }
            flat: "0.003"
          - flat: "0.0035"

    - name: compute.datumapis.com/instance/cpu-seconds
      pricing:
        currency: USD
        pricingUnit: cpu-second
        rates:
          - match: { dimension: region, value: us-central1 }
            flat: "0.0000125"
          - flat: "0.0000130"

    - name: compute.datumapis.com/instance/memory-seconds
      pricing:
        currency: USD
        pricingUnit: byte-second
        rates:
          - match: { dimension: region, value: us-central1 }
            flat: "0.000000000008"
          - flat: "0.0000000000085"

    - name: compute.datumapis.com/instance/uptime-seconds
      pricing:
        currency: USD
        pricingUnit: instance-second
        rates:
          - flat: "0.000001"
```

#### datum-cloud/service-catalog — ServicePricing fan-out

The `PricingFanOut` controller (new, sibling to `QuotaFanOut`) watches
`ServiceConfiguration` and emits one `ServicePricing` per priced metric into
the `milo-system` namespace. These are owned by service-catalog and must not be
authored by hand.

```yaml
apiVersion: billing.miloapis.com/v1alpha1
kind: ServicePricing
metadata:
  name: compute-datumapis-com--instance-cpu-allocated
  namespace: milo-system
spec:
  serviceRef: compute.datumapis.com
  metric: compute.datumapis.com/instance/cpu-allocated
  currency: USD
  pricingUnit: vcpu
  rates:
    - match: { dimension: tier, value: standard }
      flat: "0.025"
    - flat: "0.030"
```

One `ServicePricing` is emitted per priced metric. The five compute billing
metrics above produce five `ServicePricing` resources.

#### datum-cloud/service-catalog — Offers

Two Offers are authored for compute in draft form, each referencing a different
subset of the `ServicePricing` resources by name. The rate values are not
duplicated here — the controller snapshots them automatically at publish.

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
    - name: compute-datumapis-com--instance-cpu-allocated
    - name: compute-datumapis-com--instance-memory-allocated
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
    - name: compute-datumapis-com--instance-cpu-seconds
    - name: compute-datumapis-com--instance-memory-seconds
    - name: compute-datumapis-com--instance-uptime-seconds
```

#### datum-cloud/service-catalog — BillingEntitlement (applied by controller)

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

#### datum-cloud/amberflo-provider — new reconcilers

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

#### datum-cloud/cloud-portal — ServiceConfiguration

The `assistant-miloapis-com` `ServiceConfiguration` already declares `model`
and `region` as labels on the `assistant.miloapis.com/Conversation` monitored
resource type. A `pricing` block is added to each billing metric, with rates
that vary by `model`. A `Recurring` platform fee is added to `spec.charges[]`.

```yaml
spec:
  billing:
    quotaGating: BillingEntitlement

  metrics:
    - name: assistant.miloapis.com/conversation/input-tokens
      pricing:
        chargeType: Usage
        currency: USD
        pricingUnit: token
        rates:
          - match: { dimension: model, value: claude-sonnet-4-6 }
            flat: "0.000003"
          - match: { dimension: model, value: claude-opus-4-8 }
            flat: "0.000015"
          - flat: "0.000003"

    - name: assistant.miloapis.com/conversation/output-tokens
      pricing:
        chargeType: Usage
        currency: USD
        pricingUnit: token
        rates:
          - match: { dimension: model, value: claude-sonnet-4-6 }
            flat: "0.000015"
          - match: { dimension: model, value: claude-opus-4-8 }
            flat: "0.000075"
          - flat: "0.000015"

    - name: assistant.miloapis.com/conversation/cache-read-tokens
      pricing:
        chargeType: Usage
        currency: USD
        pricingUnit: token
        rates:
          - match: { dimension: model, value: claude-sonnet-4-6 }
            flat: "0.0000003"
          - flat: "0.0000003"

    - name: assistant.miloapis.com/conversation/cache-write-tokens
      pricing:
        chargeType: Usage
        currency: USD
        pricingUnit: token
        rates:
          - match: { dimension: model, value: claude-sonnet-4-6 }
            flat: "0.00000375"
          - flat: "0.00000375"

    - name: assistant.miloapis.com/conversation/messages
      pricing:
        chargeType: Usage
        currency: USD
        pricingUnit: message
        rates:
          - flat: "0.000001"

  charges:
    - name: assistant.miloapis.com/access-fee
      chargeType: Recurring
      displayName: AI Assistant Access Fee
      currency: USD
      amount: "10.00"
      interval: monthly
```

#### datum-cloud/service-catalog — ServicePricing fan-out

`PricingFanOut` emits five `ServicePricing` resources (one per priced metric,
`chargeType: Usage`). `ChargeFanOut` emits one additional `ServicePricing` for
the monthly access fee (`chargeType: Recurring`).

#### datum-cloud/service-catalog — Offer

The Offer is authored in draft form, referencing `ServicePricing` resources by
name. Rates are not duplicated — the controller snapshots them at publish.

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
    - name: assistant-miloapis-com--conversation-input-tokens
    - name: assistant-miloapis-com--conversation-output-tokens
    - name: assistant-miloapis-com--conversation-cache-read-tokens
    - name: assistant-miloapis-com--conversation-cache-write-tokens
    - name: assistant-miloapis-com--conversation-messages
    - name: assistant-miloapis-com--access-fee
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

### Real-world example: Default Pay As You Go Offer

This shows how a single default Offer bundles both Compute and AI Assistant
into the Offer every new billing account lands on automatically.

#### datum-cloud/service-catalog — Offer

The `default-pay-as-you-go-v1` Offer references `ServicePricing` resources
from both services. Compute uses the consumption model (`cpu-seconds`,
`memory-seconds`, `uptime-seconds`); AI Assistant includes all token meters
plus the monthly access fee.

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
    - name: compute-datumapis-com--instance-cpu-seconds
    - name: compute-datumapis-com--instance-memory-seconds
    - name: compute-datumapis-com--instance-uptime-seconds
    - name: assistant-miloapis-com--conversation-input-tokens
    - name: assistant-miloapis-com--conversation-output-tokens
    - name: assistant-miloapis-com--conversation-cache-read-tokens
    - name: assistant-miloapis-com--conversation-cache-write-tokens
    - name: assistant-miloapis-com--conversation-messages
    - name: assistant-miloapis-com--access-fee
```

#### datum-cloud/service-catalog — ServiceConfiguration.spec.defaultOffer

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
`claude-sonnet-4-6`:

| Line item | Volume | Rate | Subtotal |
|---|---|---|---|
| `instance/cpu-seconds` | 7,200 cpu-sec | $0.0000125 | $0.09 |
| `instance/memory-seconds` | 3.87 × 10¹⁰ byte-sec | $0.000000000008 | $0.31 |
| `instance/uptime-seconds` | 3,600 instance-sec | $0.000001 | $0.0036 |
| `conversation/input-tokens` (sonnet) | 500,000 tokens | $0.000003 | $1.50 |
| `conversation/output-tokens` (sonnet) | 1,000,000 tokens | $0.000015 | $15.00 |
| `conversation/messages` | 1,000 messages | $0.000001 | $0.001 |
| AI Assistant Access Fee (monthly) | 1 | $10.00 | $10.00 |
| **Total** | | | **~$26.90/mo** |

### How Offers relate to charge types

Every `Offer` carries a required `spec.chargeTypes` field — a set of the charge
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
| `BillingEntitlement` | Quota granted only when the service's `ServicePricing` appears in the account's active Offer. The service is inaccessible if absent from the Offer. |

Today, quota grants are issued unconditionally by `GrantCreationPolicy` +
`GrantCreationController` — this is the `OrganizationDefault` path and
requires no changes. The `BillingEntitlement` path introduces one new
controller:

- **`ServiceEntitlementReconciler.ensureQuotaGrants`** — watches
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
Compute from the Offer and Compute quota is revoked — the account can no longer
create Compute resources. DNS quota is unaffected because it is not
billing-gated.

Swapping an account to a different Offer atomically replaces opted-in quota
grants and the Amberflo Customer-Plan assignment in a single reconcile.

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
- **Billed but not gated.** A service owner adds pricing to their
  `ServiceConfiguration` but omits `quotaGating: BillingEntitlement`, so
  accounts are billed for usage but never blocked even without an active Offer.
  _Mitigation:_ the `PricingFanOut` controller emits a warning event when a
  `ServicePricing` is created for a service whose `quotaGating` is
  `OrganizationDefault`, prompting the owner to confirm the intent is correct.

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

### Charge type schema

The three charge types use different fields in `ServiceConfiguration` because
they represent fundamentally different things:

- **`Usage`** is declared on `spec.metrics[]` via a `pricing` block. The rate
  is multiplied by the meter reading, optionally filtered by a dimension.
- **`OneTime`** and **`Recurring`** are declared on a new `spec.charges[]`
  list. They carry a fixed `amount` rather than a rate, and no meter is
  required. `OneTime` charges fire once at a defined `trigger`; `Recurring`
  charges fire every billing `interval`.

Both fans — `PricingFanOut` for meter-based Usage and `ChargeFanOut` for
fixed charges — emit `ServicePricing` resources distinguished by a `chargeType`
field. Offers snapshot both kinds; the amberflo-provider maps each to the
corresponding Amberflo concept (Product Plan Item for Usage, setup fee for
OneTime, fixed recurring item for Recurring).

```yaml
spec:
  billing:
    quotaGating: BillingEntitlement

  metrics:
    - name: compute.datumapis.com/instance/cpu-allocated
      pricing:
        chargeType: Usage
        currency: USD
        pricingUnit: vcpu
        rates:
          - match: { dimension: tier, value: standard }
            flat: "0.025"
          - flat: "0.030"

  charges:
    - name: compute.datumapis.com/instance/setup-fee
      chargeType: OneTime
      displayName: Compute Setup Fee
      currency: USD
      amount: "10.00"
      trigger: BillingAccountActivation

    - name: compute.datumapis.com/platform-fee
      chargeType: Recurring
      displayName: Compute Platform Fee
      currency: USD
      amount: "5.00"
      interval: monthly
```

### Per-meter pricing on ServiceConfiguration

`spec.metrics[]` entries carry a `pricing` block for Usage charges: currency,
`pricingUnit`, and rates (flat or tiered). Per-dimension `match` entries let a
rate vary by a label declared on the service's `monitoredResourceType` (one
dimension per match entry; multi-dimension matches deferred).
`pricingUnit` is a human-readable label for the billing line item; it does not
need to be the literal UCUM unit string of the meter.

```yaml
spec:
  metrics:
    - name: compute.datumapis.com/instance/cpu-allocated
      pricing:
        currency: USD
        pricingUnit: vcpu
        rates:
          - match: { dimension: tier, value: standard }
            flat: "0.025"
          - flat: "0.030"
    - name: compute.datumapis.com/instance/cpu-seconds
      pricing:
        currency: USD
        pricingUnit: cpu-second
        rates:
          - match: { dimension: region, value: us-central1 }
            flat: "0.0000125"
          - flat: "0.0000130"
```

### ServicePricing fan-out

service-catalog gains two new fan-out controllers alongside the existing
`QuotaFanOut`:

- **`PricingFanOut`** — watches `spec.metrics[].pricing` and emits one
  `ServicePricing` per priced metric with `chargeType: Usage`.
- **`ChargeFanOut`** — watches `spec.charges[]` and emits one `ServicePricing`
  per fixed charge with `chargeType: OneTime` or `chargeType: Recurring`.

Both emit into the `milo-system` namespace, follow the `MeterDefinition`
fan-out pattern, and must not be authored by hand. Providers watch the single
`ServicePricing` shape — distinguished by `chargeType` — rather than parsing
`ServiceConfiguration`.

### Offer

`Offer` bundles service pricings into a named tier with `launchStage`
semantics. An Offer has two distinct states:

**Draft** — the platform owner references `ServicePricing` resources by name
via `spec.servicePricingRefs[]`. No rates are stored inline; the Offer is
mutable and not yet usable by accounts.

**Published (GA)** — the platform owner advances `launchStage` to `GA`. The
controller reads each referenced `ServicePricing`, copies the current rates
into `spec.servicePricings[]` as an immutable snapshot, and the Offer becomes
live. Rates are **not** authored twice — the snapshot is generated
automatically at publish time.

An Offer need not reference all `ServicePricing`s for a service — the subset
chosen determines which services are accessible and billed for accounts on
that Offer. `spec.chargeTypes` must enumerate every charge type present in the
referenced pricings.

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
    - name: compute-datumapis-com--instance-cpu-allocated
    - name: compute-datumapis-com--instance-memory-allocated
    - name: compute-datumapis-com--instance-setup-fee
    - name: compute-datumapis-com--platform-fee
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

### Display names

`Offer` carries a `kubernetes.io/display-name` annotation (matching the
convention used on milo `Role`s) so the human-readable name can change without
bumping the Offer version. `ServicePricing` inherits its `MeterDefinition`'s
display name; `BillingEntitlement` defers to its Offer's.

### Boundary with credits

Invoice-run sequence: Amberflo computes the subtotal using Offer rates → the
credit ledger ([#747][747]) draws down → the remainder hits Stripe. The
invoice-run controller that orchestrates this step is out of scope here and
will be designed in a follow-on enhancement once phases 1–4 are green in
staging. #747 designs against the Amberflo subtotal as its input.

## Acceptance Criteria

- The `ServiceConfiguration` for `compute.datumapis.com` carries `pricing`
  blocks on all five billing metrics: `instance/cpu-allocated`,
  `instance/memory-allocated`, `instance/cpu-seconds`, `instance/memory-seconds`,
  and `instance/uptime-seconds`.
- `ServicePricing` resources are emitted by the fan-out controller into
  `milo-system`, one per priced metric.
- Two published Offers exist: `compute-allocated-v1` (bundles
  `cpu-allocated` + `memory-allocated`) and `compute-consumed-v1` (bundles
  `cpu-seconds` + `memory-seconds` + `uptime-seconds`). Dimension filters are
  honored in the Amberflo Product Plan Items for each.
- Every new billing account in staging lands with a `BillingEntitlement`
  referencing the default Offer automatically, within a single reconcile.
- Quota grants are issued only for services present in the active Offer;
  switching an account to an Offer that excludes Compute revokes the Compute
  quota grant and blocks Compute resource creation.
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

## Suggested Implementation Phases

1. **Resources + service-catalog fan-out.** Land `ServicePricing`, `Offer`, and
   `BillingEntitlement` types, `PricingFanOut`, `ChargeFanOut`, and the
   default-entitlement controller. No Amberflo writes yet.
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
   is filed once phases 1–4 are green in staging.

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
