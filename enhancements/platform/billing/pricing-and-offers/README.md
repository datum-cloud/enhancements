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
  - [Real-world example: Data Transfer](#real-world-example-data-transfer)
  - [Real-world example: Default Pay As You Go Offer](#real-world-example-default-pay-as-you-go-offer)
  - [How Offers relate to charge types](#how-offers-relate-to-charge-types)
  - [Composition with quota](#composition-with-quota)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Resource Topology](#resource-topology)
  - [Charge type schema](#charge-type-schema)
  - [Charges on ServiceConfiguration](#charges-on-serviceconfiguration)
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
- [Competitive Research](#competitive-research)

## Summary

We can measure what customers use, but we can't yet charge them for it.
[Metering][681] gave us usage data and the `BillingAccount`, but no prices are
attached to anything, so when we invoice we'll have no amount to charge.

This enhancement doc proposes the missing pieces: service owners declare
Usage, OneTime, and Recurring charges on `ServiceConfiguration`, the platform
bundles those into named **Offers** (pay-as-you-go, Pro, and so on), and every
billing account gets one by default. Once that is in place, an Amberflo invoice
run finally returns a real dollar amount, which is the input the credit ledger
and eventual Stripe charge step depend on.

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

### Goals

- **Unified `spec.charges[]` on `ServiceConfiguration`.** One list for
  `Usage`, `OneTime`, and `Recurring`. Metrics stay telemetry and quota only.
  Usage charges reference a meter by `metricRef` and carry currency,
  `pricingUnit`, and rates (flat or tiered, including graduated volume bands,
  with per-dimension match so rates can vary by meter dimension value).
- **Fan-out to `ServicePricing` resources**, one per charge, emitted by a
  single `ChargeFanOut` sibling to the existing `QuotaFanOut`.
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
- **Re-rating historical usage**: prevented by construction via immutable
  published Offers.

## Proposal

### Concepts

| Concept                 | Owner             | Lives in        | What it does                                              |
| ----------------------- | ----------------- | --------------- | --------------------------------------------------------- |
| **Service Pricing**     | Service owner     | service-catalog | A charge (Usage, OneTime, or Recurring) fanned out to a CRD. |
| **Offer**               | Platform owner    | service-catalog | A named, versioned bundle of service pricings (a "tier"). |
| **Billing Entitlement** | Platform / policy | service-catalog | Binds a billing account to exactly one active Offer.      |

Service owners author meters and charges on the same `ServiceConfiguration`.
Meters describe what is measured. Charges describe what is billed. A Usage
charge points at a meter by name; OneTime and Recurring charges do not need
a meter. The platform composes those charges into Offers. A billing account
becomes billable by pointing a Billing Entitlement at an Offer. Everything
downstream (Amberflo Product Plan, Customer-Plan assignment, invoice) is
derived from those three objects.

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

**How the proposal addresses it.** The owner adds entries to
`spec.charges[]` on `ServiceConfiguration`. Usage charges set `metricRef` to
a declared metric name and attach rates (with optional per-dimension match on
labels such as `region`, `tier`, or `model`). OneTime and Recurring charges
set a fixed `amount` plus a `trigger` or `interval`. One fan-out controller
(`ChargeFanOut`) emits a `ServicePricing` per charge, distinguished by
`chargeType`. Providers watch that `ServicePricing` shape rather than parsing
`ServiceConfiguration`.

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
zero-rated Offer (`staff-zero-v1`) for testing, or onto a negotiated tier: and
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

#### datum-cloud/compute: ServiceConfiguration

The compute `ServiceConfiguration` already declares `region` and `tier` as
labels on the `compute.datumapis.com/Instance` monitored resource type, and
publishes five billing metrics. Billable meters get matching Usage entries on
`spec.charges[]` that set `metricRef` to the metric name. The `pricingUnit` is
a human-readable label for the line item; it does not need to be the literal
UCUM unit string.

`cpu-allocated` and `memory-allocated` are Gauges (instantaneous snapshots of
reserved capacity). Pricing those means an account pays for what it has
reserved. `cpu-seconds` and `memory-seconds` are Cumulatives (actual
consumption). Both sets can carry charges; which ones apply to a given account
is determined by the Offer, not the `ServiceConfiguration`. That lets the
platform offer an allocated-capacity model, a consumption model, or a future
combination from the same service definition. Metrics with no matching charge
stay unpriced (telemetry or quota only). Compute opts in to billing-gated
quota so accounts without an active Offer are blocked from creating resources.

```yaml
spec:
  billing:
    quotaGating: BillingEntitlement

  metrics:
    - name: compute.datumapis.com/instance/cpu-allocated
      # ... kind, unit, dimensions ...
    - name: compute.datumapis.com/instance/memory-allocated
    - name: compute.datumapis.com/instance/cpu-seconds
    - name: compute.datumapis.com/instance/memory-seconds
    - name: compute.datumapis.com/instance/uptime-seconds

  charges:
    - name: compute.datumapis.com/instance/cpu-allocated
      chargeType: Usage
      currency: USD
      usage:
        metricRef: compute.datumapis.com/instance/cpu-allocated
        pricingUnit: vcpu
        rates:
          - match: { dimension: tier, value: standard }
            flat: "0.025"
          - flat: "0.030"

    - name: compute.datumapis.com/instance/memory-allocated
      chargeType: Usage
      currency: USD
      usage:
        metricRef: compute.datumapis.com/instance/memory-allocated
        pricingUnit: gib
        rates:
          - match: { dimension: tier, value: standard }
            flat: "0.003"
          - flat: "0.0035"

    - name: compute.datumapis.com/instance/cpu-seconds
      chargeType: Usage
      currency: USD
      usage:
        metricRef: compute.datumapis.com/instance/cpu-seconds
        pricingUnit: cpu-second
        rates:
          - match: { dimension: region, value: us-central1 }
            flat: "0.0000125"
          - flat: "0.0000130"

    - name: compute.datumapis.com/instance/memory-seconds
      chargeType: Usage
      currency: USD
      usage:
        metricRef: compute.datumapis.com/instance/memory-seconds
        pricingUnit: byte-second
        rates:
          - match: { dimension: region, value: us-central1 }
            flat: "0.000000000008"
          - flat: "0.0000000000085"

    - name: compute.datumapis.com/instance/uptime-seconds
      chargeType: Usage
      currency: USD
      usage:
        metricRef: compute.datumapis.com/instance/uptime-seconds
        pricingUnit: instance-second
        rates:
          - flat: "0.000001"
```

#### datum-cloud/service-catalog: ServicePricing fan-out

The `ChargeFanOut` controller (sibling to `QuotaFanOut`) watches
`ServiceConfiguration` and emits one `ServicePricing` per charge into the
`milo-system` namespace. These are owned by service-catalog and must not be
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

One `ServicePricing` is emitted per charge. The five compute Usage charges
above produce five `ServicePricing` resources.

#### datum-cloud/service-catalog: Offers

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
resource type. Usage charges on `spec.charges[]` price those meters with rates
that vary by `model`. A Recurring platform fee is also declared on
`spec.charges[]`.

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

  charges:
    - name: assistant.miloapis.com/conversation/input-tokens
      chargeType: Usage
      currency: USD
      usage:
        metricRef: assistant.miloapis.com/conversation/input-tokens
        pricingUnit: token
        rates:
          - match: { dimension: model, value: claude-sonnet-4-6 }
            flat: "0.000003"
          - match: { dimension: model, value: claude-opus-4-8 }
            flat: "0.000015"
          - flat: "0.000003"

    - name: assistant.miloapis.com/conversation/output-tokens
      chargeType: Usage
      currency: USD
      usage:
        metricRef: assistant.miloapis.com/conversation/output-tokens
        pricingUnit: token
        rates:
          - match: { dimension: model, value: claude-sonnet-4-6 }
            flat: "0.000015"
          - match: { dimension: model, value: claude-opus-4-8 }
            flat: "0.000075"
          - flat: "0.000015"

    - name: assistant.miloapis.com/conversation/cache-read-tokens
      chargeType: Usage
      currency: USD
      usage:
        metricRef: assistant.miloapis.com/conversation/cache-read-tokens
        pricingUnit: token
        rates:
          - match: { dimension: model, value: claude-sonnet-4-6 }
            flat: "0.0000003"
          - flat: "0.0000003"

    - name: assistant.miloapis.com/conversation/cache-write-tokens
      chargeType: Usage
      currency: USD
      usage:
        metricRef: assistant.miloapis.com/conversation/cache-write-tokens
        pricingUnit: token
        rates:
          - match: { dimension: model, value: claude-sonnet-4-6 }
            flat: "0.00000375"
          - flat: "0.00000375"

    - name: assistant.miloapis.com/conversation/messages
      chargeType: Usage
      currency: USD
      usage:
        metricRef: assistant.miloapis.com/conversation/messages
        pricingUnit: message
        rates:
          - flat: "0.000001"

    - name: assistant.miloapis.com/access-fee
      chargeType: Recurring
      displayName: AI Assistant Access Fee
      currency: USD
      recurring:
        amount: "10.00"
        interval: monthly
```

#### datum-cloud/service-catalog: ServicePricing fan-out

`ChargeFanOut` emits six `ServicePricing` resources: five Usage charges and
one Recurring access fee.

#### datum-cloud/service-catalog: Offer

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

#### datum-cloud/service-catalog: ServiceConfiguration (Data Transfer)

```yaml
spec:
  # Data Transfer example uses OrganizationDefault quota gating unless the
  # owning service opts into BillingEntitlement separately.
  metrics:
    - name: networking.datumapis.com/transfer/egress-internet
    - name: networking.datumapis.com/transfer/internal
    - name: networking.datumapis.com/transfer/ingress

  charges:
    - name: networking.datumapis.com/transfer/egress-internet
      chargeType: Usage
      currency: USD
      usage:
        metricRef: networking.datumapis.com/transfer/egress-internet
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

    - name: networking.datumapis.com/transfer/internal
      chargeType: Usage
      currency: USD
      usage:
        metricRef: networking.datumapis.com/transfer/internal
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

    - name: networking.datumapis.com/transfer/ingress
      chargeType: Usage
      currency: USD
      usage:
        metricRef: networking.datumapis.com/transfer/ingress
        pricingUnit: gib
        rates:
          - flat: "0"
```

#### datum-cloud/service-catalog: ServicePricing fan-out

`ChargeFanOut` emits three `ServicePricing` resources:

- `networking-datumapis-com--transfer-egress-internet`
- `networking-datumapis-com--transfer-internal`
- `networking-datumapis-com--transfer-ingress`

#### datum-cloud/service-catalog: Offer

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
    - name: networking-datumapis-com--transfer-egress-internet
    - name: networking-datumapis-com--transfer-internal
    - name: networking-datumapis-com--transfer-ingress
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

#### datum-cloud/service-catalog: Offer

The `default-pay-as-you-go-v1` Offer references `ServicePricing` resources
from Compute, AI Assistant, and Data Transfer. Compute uses the consumption
model (`cpu-seconds`, `memory-seconds`, `uptime-seconds`); AI Assistant
includes all token meters plus the monthly access fee; Data Transfer includes
internet egress, internal transfer, and the explicit zero-rate ingress meter
so “ingress is free” appears on invoices. Offers may later omit transfer meters
(for example a Pro tier that includes egress differently) without changing the
`ServiceConfiguration` rate schema.

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
    - name: networking-datumapis-com--transfer-egress-internet
    - name: networking-datumapis-com--transfer-internal
    - name: networking-datumapis-com--transfer-ingress
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
| `BillingEntitlement` | Quota granted only when the service's `ServicePricing` appears in the account's active Offer. The service is inaccessible if absent from the Offer. |

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
- **Billed but not gated.** A service owner adds Usage charges to their
  `ServiceConfiguration` but omits `quotaGating: BillingEntitlement`, so
  accounts are billed for usage but never blocked even without an active Offer.
  _Mitigation:_ `ChargeFanOut` logs a warning when Usage `ServicePricing`
  objects are created for a service whose `quotaGating` is
  `OrganizationDefault`, so the owner can confirm that intent.

## Design Details

### Resource Topology

```text
ServiceConfiguration            ServicePricing            Offer
  .spec.charges[]         ─►  (one per charge)  ◄──refs── .spec.servicePricings[]
   (Usage / OneTime /          (fan-out emitted by         (snapshot at Publish)
    Recurring; Usage            service-catalog)                  ▲
    sets metricRef)                                               │
                                    │                             │
                                    ▼                             ▼
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

All three charge types are declared on `spec.charges[]`. They share `name`,
`chargeType`, `displayName`, and `currency`. Type-specific fields live under
nested option objects so supported configuration is explicit per chargeType:

- **`Usage`** requires `usage` with `metricRef` (must match a
  `spec.metrics[].name`), `pricingUnit`, and `rates`. The rate is multiplied
  by the meter reading, optionally filtered by a dimension. `oneTime` and
  `recurring` must not be set.
- **`OneTime`** requires `oneTime` with `amount` and `trigger`. It fires once
  at that trigger. `usage` and `recurring` must not be set.
- **`Recurring`** requires `recurring` with `amount` and `interval`. It fires
  every billing interval. `usage` and `oneTime` must not be set.
- The last band in a `tiered` rate list must omit `upTo` (open-ended). Earlier
  bands require `upTo`.

`ChargeFanOut` emits one `ServicePricing` per charge, distinguished by
`chargeType`. Offers snapshot every kind; amberflo-provider maps each to the
matching Amberflo concept (Product Plan Item for Usage, setup fee for OneTime,
fixed recurring item for Recurring).

```yaml
spec:
  billing:
    quotaGating: BillingEntitlement

  metrics:
    - name: compute.datumapis.com/instance/cpu-allocated
      # ... kind, unit, dimensions ...

  charges:
    - name: compute.datumapis.com/instance/cpu-allocated
      chargeType: Usage
      currency: USD
      usage:
        metricRef: compute.datumapis.com/instance/cpu-allocated
        pricingUnit: vcpu
        rates:
          - match: { dimension: tier, value: standard }
            flat: "0.025"
          - flat: "0.030"

    - name: compute.datumapis.com/instance/setup-fee
      chargeType: OneTime
      displayName: Compute Setup Fee
      currency: USD
      oneTime:
        amount: "10.00"
        trigger: BillingAccountActivation

    - name: compute.datumapis.com/platform-fee
      chargeType: Recurring
      displayName: Compute Platform Fee
      currency: USD
      recurring:
        amount: "5.00"
        interval: monthly
```

### Charges on ServiceConfiguration

`spec.charges[]` is the only place commercial terms are authored. Usage
charges set those fields under `usage` (`metricRef`, `pricingUnit`, and rates).
Per-dimension `match` entries let a rate vary by a label declared on the
service's `monitoredResourceType` (one dimension per match entry;
multi-dimension matches deferred). Services that need a compound key (for
example source and destination region) emit a single derived dimension at
metering time; see
[Real-world example: Data Transfer](#real-world-example-data-transfer).
`pricingUnit` is a human-readable label for the billing line item; it does not
need to be the literal UCUM unit string of the meter.

Each `rates[]` entry carries either `flat` or `tiered`, never both:

- **`flat`** is a single decimal USD string multiplied by metered usage.
- **`tiered`** is ordered graduated volume bands. Each band has a `rate` and
  an exclusive upper bound `upTo` in `pricingUnit` units. The last band omits
  `upTo` (open-ended). Zero (`"0"`) is a valid rate for free allowances.
  Aggregation for tier breaks is monthly, at billing-account scope.
  Volume/retroactive (reprice-all) modes are not expressed in v1.

```yaml
spec:
  metrics:
    - name: compute.datumapis.com/instance/cpu-allocated
    - name: compute.datumapis.com/instance/cpu-seconds
    - name: networking.datumapis.com/transfer/egress-internet

  charges:
    - name: compute.datumapis.com/instance/cpu-allocated
      chargeType: Usage
      currency: USD
      usage:
        metricRef: compute.datumapis.com/instance/cpu-allocated
        pricingUnit: vcpu
        rates:
          - match: { dimension: tier, value: standard }
            flat: "0.025"
          - flat: "0.030"
    - name: compute.datumapis.com/instance/cpu-seconds
      chargeType: Usage
      currency: USD
      usage:
        metricRef: compute.datumapis.com/instance/cpu-seconds
        pricingUnit: cpu-second
        rates:
          - match: { dimension: region, value: us-central1 }
            flat: "0.0000125"
          - flat: "0.0000130"
    - name: networking.datumapis.com/transfer/egress-internet
      chargeType: Usage
      currency: USD
      usage:
        metricRef: networking.datumapis.com/transfer/egress-internet
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
          - tiered:
              - upTo: "200"
                rate: "0"
              - upTo: "10240"
                rate: "0.15"
              - upTo: "153600"
                rate: "0.12"
              - rate: "0.09"
```

### ServicePricing fan-out

service-catalog gains a `ChargeFanOut` controller alongside the existing
`QuotaFanOut`. It watches `spec.charges[]` and emits one `ServicePricing` per
charge (`Usage`, `OneTime`, or `Recurring`) into `milo-system`. The fan-out
follows the `MeterDefinition` pattern and must not be authored by hand.
Providers watch the single `ServicePricing` shape, distinguished by
`chargeType`, rather than parsing `ServiceConfiguration`.

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

An Offer need not reference all `ServicePricing`s for a service: the subset
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
- `flat` rates map to a single LeafNode price as today. `tiered` rates map to
  Amberflo graduated LeafNode / TieredPriceGroup tiers: band `i` becomes
  `{ startAfterUnit, batchSize: 1, pricePerBatch: rate, allowPartialBatch: true }`
  where `startAfterUnit` is `0` for the first band and the previous band's
  `upTo` thereafter; the open-ended last band has no following tier.

### Display names

`Offer` carries a `kubernetes.io/display-name` annotation (matching the
convention used on milo `Role`s) so the human-readable name can change without
bumping the Offer version. Usage `ServicePricing` can fall back to the
referenced metric's display name when the charge omits `displayName`. Fixed
charges use the charge `displayName`. `BillingEntitlement` defers to its
Offer's display name.

### Boundary with credits

Invoice-run sequence: Amberflo computes the subtotal using Offer rates → the
credit ledger ([#747][747]) draws down → the remainder hits Stripe. The
invoice-run controller that orchestrates this step is out of scope here and
will be designed in a follow-on enhancement once phases 1-4 are green in
staging. #747 designs against the Amberflo subtotal as its input.

## Acceptance Criteria

- The `ServiceConfiguration` for `compute.datumapis.com` carries Usage
  charges (via `spec.charges[]`) for all five billing metrics:
  `instance/cpu-allocated`, `instance/memory-allocated`, `instance/cpu-seconds`,
  `instance/memory-seconds`, and `instance/uptime-seconds`.
- `ServicePricing` resources are emitted by `ChargeFanOut` into `milo-system`,
  one per charge.
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
- The `ServiceConfiguration` charge schema accepts graduated `tiered` rate
  entries (`upTo` / `rate`, mutually exclusive with `flat` on the same entry).
- The Data Transfer rates in
  [Real-world example: Data Transfer](#real-world-example-data-transfer)
  (internet egress by geo with volume bands, internal path flats, free ingress)
  are authorable as `ServicePricing` snapshots on a published Offer.

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

## Drawbacks

Introducing a second "Entitlement" kind alongside `ServiceEntitlement` adds
conceptual surface area. The alternative (renaming or overloading the existing
kind) was judged more invasive and riskier than adding a distinct, clearly
scoped resource.

## Alternatives

- **Mutable Offers.** Allowing published Offers to change rates in place would
  avoid version proliferation but would silently re-price live customers and
  make historical re-rating possible. Rejected in favor of snapshot + immutable
  publish.
- **Pricing nested on `spec.metrics[]`.** An earlier draft put Usage rates on
  each metric. That forces every priced meter to look "commercial" and leaves
  no shared place for OneTime/Recurring. Rejected in favor of `spec.charges[]`
  with `metricRef` for Usage, so unbilled meters and non-usage charges share
  one authoring surface.
- **Pricing as a standalone catalog separate from ServiceConfiguration.**
  Authoring prices far from the meters they rate would let pricing and metering
  drift without a clear ownership path. Rejected: keep charges on the same SC
  as the meters, with Usage charges pointing at meters by name.
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

**"Service config owns the meter; charges own the commercial terms"** matches
how GCP separates meter declaration from SKU pricing. GCP's Service
Infrastructure declares `MonitoredResource` types and DELTA metrics on a
service config; Cloud Billing attaches SKU pricing to those meters. Services
call the Service Control API to report usage; the rating engine prices it.
This enhancement keeps meters on `ServiceConfiguration` and attaches prices
as charges that fan out to `ServicePricing`, then into Offers.
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
