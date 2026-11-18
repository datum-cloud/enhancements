---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

<!-- omit from toc -->
# Writing Our Own Stripe Invoices

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [How It Works](#how-it-works)
  - [User Stories](#user-stories)
  - [What Every Invoice Line Carries](#what-every-invoice-line-carries)
  - [Service Periods](#service-periods)
  - [Charges Staff Add](#charges-staff-add)
  - [Tax](#tax)
  - [Notes and Constraints](#notes-and-constraints)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Overview](#design-overview)
  - [Who Does What](#who-does-what)
  - [Stripe Products](#stripe-products)
  - [The Invoice Writer](#the-invoice-writer)
- [Rollout](#rollout)
- [Open Questions](#open-questions)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [References](#references)

## Summary

Our invoices are built by OpenMeter and sent to Stripe by OpenMeter's own
Stripe app. That app writes every line as a bare amount. There's no quantity,
no unit price, no product, no SKU and no tax code, and only OpenMeter's line
ID as metadata. Finance can't report by product, customers can't see what
they're paying per unit, and every per-product tax rule waits on OpenMeter.

This enhancement keeps OpenMeter doing what it does well: metering usage,
rating it, scheduling fees and assembling one invoice per billing period. It
replaces only the last step. OpenMeter hands each finished invoice to **our
own Stripe integration**, which writes the Stripe invoice line by line, the
way finance needs it.

Every line on a Datum invoice then carries:

- its product, as a Stripe Product with the catalog's SKU;
- a real quantity and unit price;
- a **service period**: the dates the charge is for;
- its own tax code;
- an optional note;
- our metadata, linking it back to the price and contract it came from.

The same path lets staff add one-off charges to an account's next invoice,
each with its own service period, without editing invoices by hand in Stripe.

## Motivation

Contract customers get invoices that finance has to explain and rework by
hand. The gaps come from how OpenMeter's Stripe app writes invoices, not from
how OpenMeter bills.

- **Quantities exist only as text.** A line for four cross-connects is one
  amount, with "4 × $300.00" in its name. Stripe can't total units, and
  reports can't group by product.
- **SKUs exist only in the line's name.** We put "[SKU] name" on lines as a
  stopgap. Stripe has no product to report on, and Revenue Recognition can't
  split revenue by product.
- **Tax codes are one per invoice.** Stripe Tax treats every line the same
  way. Colocation space, power, setup work and cloud usage can be taxed
  differently, and the product catalog already gives each SKU its own tax
  code.
- **Staff-added charges have no service period.** A one-off charge made in
  the Stripe dashboard has no dates. Revenue Recognition then books it all on
  the day the invoice is finalized, even when it's for a month, a quarter or
  an installation date. A charge for "15 November to 15 December" can't say so.
- **Staff-added charges live outside the platform.** Invoices staff create in
  Stripe show in the portals, but the platform didn't price them. They don't
  appear in the invoice preview, they don't use the catalog's prices or SKUs,
  and they're easy to get wrong.
- **We can't change any of this ourselves.** Each fix means a change to
  OpenMeter's Stripe app upstream, on OpenMeter's timeline.

### Goals

- Every line on every invoice has a product (with SKU), a quantity, a unit
  price, a service period and a tax code, whether it's usage, a contract fee
  or a charge staff added.
- Finance can use Stripe's own reports and Revenue Recognition by product and
  by service period, with no spreadsheet work.
- Staff can add one-off charges to an account's next invoice from the staff
  portal. They pick a catalog item or enter a custom charge, with a quantity,
  a service period and a note.
- Customers see the same invoice they see today, but clearer: units, unit
  prices and dates on each line.
- The change is invisible to everything that reads invoices today. The
  `Invoice` resource, both portals, previews and spend estimates keep working
  unchanged.
- We own the Stripe side, so the next thing finance needs on a line is a
  change in our code, not upstream.

### Non-Goals

- **Replacing OpenMeter.** Metering, usage rating, tiers, discounts, fee
  schedules and invoice assembly stay in OpenMeter.
- **Moving to Stripe Billing.** No Stripe subscriptions or meters. Invoices
  stay on Stripe Invoicing, which also avoids Stripe Billing's fee.
- **Backdated contracts and amendments with credits.** These are limits of
  OpenMeter's subscriptions, not of how invoices reach Stripe. They're
  covered by separate contract billing work. This enhancement gives them a
  way to put a catch-up or credit line on an invoice, but doesn't design them.
- **Line items in the customer portal.** The portals keep showing invoice
  totals and the Stripe-hosted invoice. Showing lines is future work.
- **Tax by facility location.** Taxing a line by where the hardware is,
  rather than where the customer is, is an open question for finance and
  Stripe. See [Open Questions](#open-questions).

## Proposal

### How It Works

Nothing changes until an invoice is ready to send.

1. OpenMeter meters usage and charges fees through each billing period, as it
   does today. Charges staff add are put on the same gathering invoice.
2. When the period closes, OpenMeter has a draft invoice. Instead of sending
   it to Stripe itself, it waits for us.
3. Our integration reads the draft and writes a Stripe invoice: one line per
   OpenMeter line, each with its product, quantity, unit price, service
   period, tax code, note and metadata. Stripe Tax calculates tax on it as
   today.
4. We tell OpenMeter the draft is synced, then finalize the Stripe invoice
   when OpenMeter issues it.
5. Stripe collects payment, as today: by card, or by the account's payment
   terms. We report payment status back to OpenMeter.
6. The `Invoice` resource and both portals pick the invoice up as they do now.

OpenMeter supports this hand-off already. Its **Custom Invoicing** app, in
the version we run, exists for an outside system that writes the invoices.
It holds each draft until told it's synced and holds issuing until told it's
finalized.

### User Stories

- **Finance.** As finance, I want revenue split by product and recognised
  over each line's service period, so that I can close the month from
  Stripe's reports.
- **Finance.** As finance, I want each product taxed by its own tax code, so
  that space, power, services and cloud usage are taxed correctly.
- **Staff.** As a staff member, I want to bill a customer for an installation
  or a period of service that isn't part of their contract, with the dates it
  covers, so that it appears on their next invoice with everything else.
- **Staff.** As a staff member, I want to add a note to a line, so that I can
  explain an unusual charge to finance or the customer.
- **Customer.** As a customer, I want to see how many units I'm paying for,
  at what price and for which dates, so that I can check my invoice against
  my contract.

### What Every Invoice Line Carries

| Field | Where it comes from | Why |
|---|---|---|
| Product and SKU | The `ServicePricing`'s SKU, or the catalog item staff picked | Reports, revenue by product, matching to the catalog |
| Description | The price's display name, as on invoices today | What the customer reads |
| Quantity | The contract's quantity for a fee, the rated quantity for usage, or what staff entered | Units the customer can check |
| Unit price | The price, after any customer pricing | Price the customer can check |
| Service period | See [Service Periods](#service-periods) | Revenue Recognition; the dates the customer is paying for |
| Tax code | The product's tax code, or the one staff picked | Per-product tax |
| Note | Staff, for charges they add; the contract, for its fees | See [Open Questions](#open-questions) |
| Metadata | Our IDs: the price, the contract, the account, OpenMeter's line and invoice | Tracing a line back to where it came from |

Discounts stay as they are today in amount, and show as their own lines or as
Stripe discounts against the line they reduce.

### Service Periods

Every line has a service period: a start and an end date. Stripe's Revenue
Recognition spreads a line's revenue over it, and the customer sees what each
charge is for.

| Line | Service period |
|---|---|
| Usage | The period the usage was measured in |
| Recurring fee billed in advance | The period it pays for, which is usually the one starting |
| Recurring fee billed in arrears | The period that just ended |
| Quarterly or annual fee | The whole quarter or year it pays for, even though it's on a monthly invoice |
| First, prorated period of a contract | The part period, from the contract's start to its first billing day |
| One-time fee, such as a setup fee | The day it's charged, unless staff give a period, such as the installation date |
| Charge staff add | Whatever staff enter, which defaults to the account's current billing period |
| Catch-up for a backdated contract | The time that has already passed |
| Discount or credit | The period of the line it reduces |

The rules are simple:

- The end is on or after the start. A single day is allowed.
- A period can be in the past (a catch-up) or the future (paid in advance).
- A period can be longer than the invoice's period, such as an annual fee.
  Revenue Recognition spreads it over the whole period.

OpenMeter already gives a period to every line it creates, so most of this is
carrying it through. The new part is letting staff set it on charges they
add.

### Charges Staff Add

Staff add a charge to a billing account in the staff portal. It goes on the
account's next invoice, with the usage and fees for that period.

- **From the catalog.** Staff pick a catalog item, such as a cross-connect
  install. The SKU, price, unit and tax code come from it. Staff enter a
  quantity and a service period.
- **Custom.** Staff enter a description, an amount, a quantity, a service
  period and a tax code. It gets a general "custom charge" product unless
  staff pick one.
- **A note**, either way.
- **Invoice now**, as an option, for a charge that shouldn't wait for the
  period to close.

The charge is a platform resource, with the same audit trail and approvals as
the rest of billing. It shows in the account's invoice preview before it's
billed. OpenMeter takes it as a pending line on the gathering invoice, so it
lands on the same invoice as everything else and keeps its period.

Creating invoices by hand in Stripe keeps working, and they keep showing in
the portals. It becomes the exception, not the routine.

### Tax

Stripe Tax keeps calculating tax on the whole invoice from the customer's
address and tax IDs, as today. The change is that each line names its own
tax code, through its Stripe Product.

- Each catalog item gets a tax code. Finance chooses them, from Stripe's list.
- A line without one uses the account's default, as every line does today.
- Tax-exempt and reverse-charge customers work as today. Stripe applies
  exemptions per customer.

### Notes and Constraints

- **Amounts match OpenMeter to the cent.** Quantity × unit price must equal
  the amount OpenMeter rated. Where it can't, such as tiered usage or a
  prorated period, the line shows the true quantity and splits by tier or
  falls back to the rated amount. The total never differs from OpenMeter's.
- **We take on what OpenMeter's Stripe app did.** Creating and finalizing the
  Stripe invoice, payment terms, the default payment method and payment
  status all become ours.
- **The switch is per account, at a billing boundary.** No period is synced
  by both the old and the new path.
- **No new Stripe fees.** Invoices stay on Stripe Invoicing.

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| An invoice is charged twice, or a period is missed, during the switch | Switch each account at a billing boundary, after its last invoice through OpenMeter's Stripe app has been issued. Compare invoices side by side on staging first. |
| Our invoices differ from what OpenMeter rated | The writer checks every invoice's total against OpenMeter's before finalizing, and holds the invoice if they differ. |
| An invoice is stuck because our integration is down | OpenMeter holds the draft until we sync it, so nothing is lost. We alert on drafts held longer than expected. |
| Tax changes once lines have their own codes | Finance signs off each tax code. Compare tax on staging invoices before and after. |
| A staff-added charge is wrong | Charges show in the preview before they're billed, and can be removed until the invoice is issued. |

## Design Overview

This section is a sketch. Each provider documents its own design.

### Who Does What

| | Today | With this enhancement |
|---|---|---|
| Meter and rate usage | OpenMeter | OpenMeter |
| Schedule fees, assemble one invoice per period | OpenMeter | OpenMeter |
| Charges staff add | Stripe dashboard, by hand | A platform resource, put on OpenMeter's invoice |
| Stripe Products | None | stripe-provider, from the catalog |
| Write the Stripe invoice | OpenMeter's Stripe app | openmeter-provider |
| Tax calculation | Stripe Tax | Stripe Tax, per line |
| Payment collection | Stripe | Stripe |
| `Invoice` resource and portals | openmeter-provider | openmeter-provider, unchanged |

### Stripe Products

stripe-provider keeps a Stripe Product for each catalog item, with:

- its SKU and name;
- its tax code;
- our metadata linking it to the `ServicePricing`.

Prices are immutable in our catalog, and a SKU names a product, not a price.
So one Product stands for every price of the same SKU, and each line sets its
own unit price. Customer pricing doesn't create new Products.

### The Invoice Writer

openmeter-provider already reads OpenMeter's invoices for every account and
projects them to `Invoice` resources. The writer adds:

1. reading each draft OpenMeter is holding for us;
2. mapping each line back to its price and contract, for its product,
   quantity, unit price and note;
3. writing the Stripe invoice and its lines, idempotently, with the service
   period on every line;
4. telling OpenMeter the draft is synced, then finalizing when it issues, then
   reporting payment status.

It marks every invoice it writes as OpenMeter's, so it isn't also picked up
as a staff-created invoice.

## Rollout

1. **Products and tax codes.** Add tax codes to the catalog and create
   Stripe Products. Nothing on invoices changes yet.
2. **The writer, on staging.** Move staging accounts to it. Compare each
   invoice with what OpenMeter's Stripe app would have written: same totals,
   same tax, better lines.
3. **Charges staff add.** Ship the staff portal flow on top of the writer.
4. **Production, account by account.** Internal accounts first, then contract
   customers, then everyone else. Each account moves at its own billing
   boundary.
5. **Retire the stopgap.** Once every account is on the writer, stop putting
   the SKU in line names.

Rolling back an account is the same switch the other way, at its next
billing boundary.

Rough size: 5–7 engineer-weeks for Products, tax codes and the writer, and
2–3 more for charges staff add.

## Open Questions

- **Notes: staff-only or on the invoice?** A note could be internal, for
  finance only, or printed on the customer's invoice. It could also be both,
  as two fields.
- **Tax codes.** Finance needs to choose a Stripe tax code for each catalog
  item.
- **Tax by facility.** Some catalog items are taxed where the facility is,
  not where the customer is. Stripe Tax works out tax from the customer's
  location for the whole invoice. We need to ask Stripe whether that means a
  separate invoice per facility.
- **Who can add charges, and up to what amount** without a second approval.
- **One-time fee periods.** Should a setup fee's period be the day it's
  charged, or the contract's first period?
- **Custom Invoicing on staging.** We need to confirm how OpenMeter behaves
  when an account moves app mid-period, and how long it will hold a draft.

## Implementation History

- 2026-11-18: First draft.

## Drawbacks

- We own more of the invoicing path. A bug in our writer is a wrong invoice,
  where today the risk sits with OpenMeter's Stripe app.
- The writer depends on OpenMeter's Custom Invoicing app, which is less used
  than its Stripe app.
- OpenMeter's own copy of each invoice won't hold Stripe's tax amounts. Our
  `Invoice` resource already takes amounts from Stripe, so nothing customers
  see depends on them.

## Alternatives

- **Ask OpenMeter to fix its Stripe app.** Quantities, products and metadata
  upstream. We'd wait on their roadmap for each change, and the next thing
  finance needs would wait again.
- **Keep the SKU in the line name.** It's what we do today as a stopgap. It
  gives no products, quantities or per-line tax.
- **Move fees to Stripe Billing subscriptions, with usage from OpenMeter.**
  It solves contract amendments and backdating natively, but costs two to
  three times as much. It also adds Stripe Billing's fee, needs a migration of
  live contracts, and still needs this writer for usage lines. It stays an
  option for later, and this work is its first half.
- **Send usage to Stripe meters as well.** Stripe can't price one meter
  differently per dimension, as our usage pricing does. Spend estimates and
  invoices would then come from different systems.
- **Build our own invoicing engine.** Without OpenMeter assembling invoices,
  we'd own every period, proration and late-usage rule ourselves, with nothing
  to check them against.

## References

- [Service Pricing, Offers, and Billing Entitlements](../pricing-and-offers/README.md)
- billing: `docs/enhancements/invoicing.md` (the `Invoice` resource) and
  `docs/enhancements/contract-billing.md`
- [OpenMeter Custom Invoicing](https://openmeter.io/docs/integrations/external-invoicing/overview)
- [OpenMeter Stripe invoicing](https://openmeter.io/docs/integrations/stripe/invoicing)
- [Stripe invoice items](https://docs.stripe.com/api/invoiceitems/create)
- [Stripe tax codes and tax behavior](https://docs.stripe.com/tax/products-prices-tax-codes-tax-behavior)
- [Stripe Revenue Recognition for invoicing](https://docs.stripe.com/revenue-recognition/methodology/subscriptions-and-invoicing)
