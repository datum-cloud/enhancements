# Milo's Billing Platform for Service Providers

## Overview

This document outlines the vision and enhancements for the Milo platform
billing system. The billing platform provides billing and financial management
for B2B service providers. Using declarative configuration principles, it
enables flexible monetization models, cost tracking, and enterprise-grade
financial controls for organizations consuming services through the platform.

> [!IMPORTANT]
>
> **Status (November 2026)**: Datum Cloud bills its customers through this
> platform in production. Billing accounts, card payments, payment terms,
> usage metering, pricing, discounts and invoicing are built. Contracts and
> commitments are in progress, and so is restricting and suspending accounts
> that don't pay. Credits are built but switched off. Budgets, hierarchical
> billing and multi-currency are not built yet.
>
> The detailed designs live next to the code in
> [milo-os/billing `docs/enhancements`](https://github.com/milo-os/billing/tree/main/docs/enhancements).

## Core Capabilities

### Billing Accounts (Built)

Billing accounts are the primary billing entity. Each one lives in an
organization and holds its contact details, invoice email, address, tax IDs
and default payment method.

- **Payment Profiles**: Cards collected through Stripe, or payment by invoice
  on terms staff grant or customers apply for (Net 30 by default, up to Net
  90).
- **Project Linking**: Each project is bound to the account that pays for it.
  Bindings can't be edited; moving a project creates a new binding, so the
  history of who paid for what is kept.
- **Multi-Currency Support** (not built): every account is billed in USD.
- **Hierarchical Billing** (not built): no sub-accounts for resellers or
  channel partners yet.

See [Billing Accounts](./accounts/), [payment methods][payment-methods] and
[billing arrangements][arrangements].

### Commitment Management (In progress)

Contracts and commitments are still in progress. The first pieces are built
from Offers rather than a dedicated commitment resource:

- **SKU-Level Discounts** (built): an Offer can discount everything in it, or
  each service separately. A customer's negotiated discount is their own
  Offer. See [offer discounts][offer-discounts].
- **Contracts** (first version built): one-time setup fees, fixed monthly fees and a
  contract start date that sets the billing day. Contracts can be amended from
  the next billing date. See [contract billing][contract-billing].
- **Minimum Spend Commitments** (not built): no minimum spend, true-ups or
  commitment periods yet.
- **Progress Tracking** (not built): no view of commitment utilization.
- **End of Term** (designed, not built): what happens when a contract's term
  ends.

The [platform user stories](../README.md#user-stories) show how each kind of
customer is set up today.

### Service Integration (Built)

Billing is integrated with the platform's service catalog:

- **Entitlement Controls**: Each billing account is entitled to one published
  Offer, which sets the prices it pays. New accounts get the platform's default
  Offer. See [Pricing and Offers](./pricing-and-offers/).
- **Usage Aggregation**: Services emit usage events through the billing SDK.
  Billing validates them, attributes them to the paying account and delivers
  them durably for rating. See the [usage pipeline][usage-pipeline].
- **Cost Allocation**: Customers and staff query usage and spend by account,
  project, meter and dimension. See [usage query][usage-query].
- **Invoice Generation**: Accounts are invoiced monthly, or on their contract
  billing day, and invoices and payment status are shown in Milo. Staff can
  preview an account's next invoice. See [invoicing][invoicing].
- **Credits** (built, switched off): credit grants that usage draws down
  before anything is charged, and an optional signup credit. See
  [credits][credits].

### Budget Management & Alerts (Not started)

Budget controls and alerting spanning billing and telemetry:
- **Budget Definition**: Set spending limits at billing account and project
  levels
- **Threshold Monitoring**: Real-time tracking against defined budget limits
- **Alert Configuration**: Customizable alerts at various threshold levels (50%,
  75%, 90%, 100%)
- **Integration Points**:
  - Billing system owns budget definitions and threshold calculations
  - Telemetry system provides real-time usage data for budget tracking
  - Communication platform handles alert delivery via email, webhooks, and
    notifications
- **Forecasting**: Projected spend based on current usage patterns
- **Cost Anomaly Detection**: Automatic alerts for unusual spending patterns

Billing changes, such as invoices issued or paid, cards expiring and payment
terms ending, already appear on the organization's
[activity timeline][billing-activity].

## Architecture Principles

### Resource Model

Billing resources follow declarative patterns:

- Organization-scoped billing accounts ensure proper isolation
- Immutable project links for comprehensive audit trails
- Declarative configuration for all billing entities, as Kubernetes resources
  in the `billing.miloapis.com` API group
- Status tracking for operational visibility

### Providers

The billing service owns the APIs, validation and business rules. Providers
connect them to the external systems that rate usage, issue invoices and take
payments, so a provider can be replaced without changing the API or the
portals:

- **[openmeter-provider](https://github.com/milo-os/openmeter-provider)**
  rates usage and runs invoicing in OpenMeter, whose Stripe app sends the
  invoices and charges cards. It went live in production on 24 October 2026,
  replacing Amberflo.
- **[stripe-provider](https://github.com/milo-os/stripe-provider)** owns each
  account's Stripe customer and collects cards.

### Security & Compliance

- RBAC-based access controls via IAM system
- Audit logging for all billing operations
- Encrypted storage of sensitive financial information
- Compliance-ready financial reporting

### Scalability Goals

- Design for high-volume transaction processing
- Efficient aggregation of usage across hundreds of thousands of projects
- Optimization for global, multi-region deployments
- Support for millions of billing events per month

## Target Use Cases

The billing platform aims to support the following scenarios:

### Enterprise Customers

- Department-level cost centers with isolated billing
- Regional billing accounts for global operations
- Contractual commitments with volume discounts
- Flexible payment terms aligned with procurement policies

### Service Providers

- Client-specific billing isolation
- White-label billing for managed service providers
- Usage-based pricing with commitment options
- Integrated invoicing and revenue recognition

### Channel Partners

- Reseller billing with sub-account management
- Marketplace scenarios with revenue sharing
- Partner-specific pricing and discounts
- Consolidated billing for partner networks

## Portals

- **[Cloud portal](https://github.com/datum-cloud/cloud-portal)**: where an
  organization's owners manage billing: onboarding with a card, billing
  accounts and payment methods, applying for payment terms, discounts,
  invoices, which account pays for each project, and usage and spend
  dashboards.
- **[Staff portal](https://github.com/datum-cloud/staff-portal)**: where
  Datum staff run billing: account dashboards, payment terms and the
  applications queue, Offer authoring, contract set-up with invoice
  previews, and usage views.

## Enhancement Proposals

### Implemented

- **[Billing Accounts](./accounts/)**: billing accounts and project bindings
- **[Service Pricing, Offers, and Billing Entitlements](./pricing-and-offers/)**
- In milo-os/billing:
  - [Usage pipeline][usage-pipeline] and [emitting usage][emitting-usage]
  - [Payment methods][payment-methods]
  - [Billing arrangements][arrangements] (payment terms)
  - [Invoicing][invoicing]
  - [Usage query][usage-query]
  - [Offer discounts][offer-discounts]
  - [Billing activity][billing-activity]

### In Progress

- **[Credits][credits]**: built in billing behind a feature gate, waiting on
  provider support
- **[Non-payment][nonpayment]**: restricting and suspending accounts that
  don't pay, tracked in the epic [milo-os/billing#219][nonpayment-epic].
  Customers get reminders as soon as an invoice is overdue. Card accounts have
  growth paused at day 7 (no new projects or resources) and service paused at
  day 21, through Milo's [project suspension](../project-suspension/).
  Invoice accounts are only paused when staff decide to. Paying restores
  everything, nothing is ever deleted, and it ships switched off, rolling out
  in phases: observe, reminders, pause growth, pause service
- **Contracts and commitments**: the first version of
  [contract billing][contract-billing] is built (start dates, fixed fees,
  amendments). Still to come: end of term (renewal, month-to-month or
  stopping), minimum spend, true-ups and reserved capacity

### Planned

- **Budget Management & Alerts**
- **Hierarchical Billing**: sub-accounts for resellers and channel partners
- **Multi-Currency**
- **Cost Analytics**: advanced reporting and cost optimization insights

See also the [initial scope](./initial-scope.md) for Datum Cloud consumers.

## Integration Points

The billing system integrates with several platform components:

- **IAM System**: Identity and access management to resources
- **Service Catalog**: Meters, service pricing and the default Offer
- **Telemetry Platform**: Usage data collection and aggregation
- **Resource Management**: Organizations and projects for billing associations
- **Activity**: Billing changes on the organization's activity timeline
- **Communication Platform**: Emails about payment terms applications; alert
  delivery for budget thresholds is planned

## Future Directions

The billing platform roadmap includes:

- Advanced cost allocation and chargeback mechanisms
- Real-time billing with immediate charge processing
- Sophisticated revenue recognition capabilities
- AI-powered cost optimization recommendations
- Enhanced marketplace and revenue sharing models
- Cross-cloud billing integration

[emitting-usage]: https://github.com/milo-os/billing/blob/main/docs/emitting-usage.md
[usage-pipeline]: https://github.com/milo-os/billing/blob/main/docs/enhancements/usage-pipeline.md
[payment-methods]: https://github.com/milo-os/billing/blob/main/docs/enhancements/payment-methods.md
[invoicing]: https://github.com/milo-os/billing/blob/main/docs/enhancements/invoicing.md
[arrangements]: https://github.com/milo-os/billing/blob/main/docs/enhancements/billing-arrangements.md
[usage-query]: https://github.com/milo-os/billing/blob/main/docs/enhancements/usage-query.md
[offer-discounts]: https://github.com/milo-os/billing/blob/main/docs/enhancements/offer-discounts.md
[credits]: https://github.com/milo-os/billing/blob/main/docs/enhancements/credits/README.md
[contract-billing]: https://github.com/milo-os/billing/blob/main/docs/enhancements/contract-billing.md
[billing-activity]: https://github.com/milo-os/billing/blob/main/docs/enhancements/billing-activity.md
[nonpayment]: https://github.com/milo-os/billing/blob/main/docs/enhancements/nonpayment-suspension.md
[nonpayment-epic]: https://github.com/milo-os/billing/issues/219
