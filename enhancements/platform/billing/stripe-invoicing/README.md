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
- [Work This Builds On](#work-this-builds-on)
- [Proposal](#proposal)
  - [How It Works](#how-it-works)
  - [User Stories](#user-stories)
  - [What Every Invoice Line Carries](#what-every-invoice-line-carries)
  - [Price Overrides](#price-overrides)
  - [Service Periods](#service-periods)
  - [Charges Staff Add](#charges-staff-add)
  - [Previewing a Contract's Invoices](#previewing-a-contracts-invoices)
  - [Tax on Line Items](#tax-on-line-items)
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
- an optional note for staff, which the customer never sees;
- our metadata, linking it back to the price and contract it came from.

The same path lets staff add one-off charges to an account's next invoice,
each with its own service period, without editing invoices by hand in Stripe.
And while staff write a contract, they can see the invoices it will produce
as Stripe itself works them out, tax included.

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
  a service period and a note for staff.
- Staff writing a contract can see its first invoices as Stripe will
  produce them, with real tax, before the contract is saved.
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

## Work This Builds On

Several billing changes made this week are the first steps of this
enhancement. This work keeps them and builds on them.

| Change | Status | What it becomes here |
|---|---|---|
| A SKU on `ServicePricing` and on a contract's fees ([milo-os/billing#256](https://github.com/milo-os/billing/pull/256)) | Released in billing v0.10.0 | The SKU of each Stripe Product. A customer's price has its list price's SKU, so it shares the list price's Product. |
| "[SKU] name" on invoice lines ([milo-os/openmeter-provider#118](https://github.com/milo-os/openmeter-provider/pull/118)) | Merged, not released | The stopgap until the writer. Retired in the last [Rollout](#rollout) step. |
| SKUs on contract fees in the staff portal ([datum-cloud/staff-portal#853](https://github.com/datum-cloud/staff-portal/pull/853)) | Draft | The catalog picker for [charges staff add](#charges-staff-add) reuses it. |
| Contracts that bill on the account's invoice day, with a prorated first period ([milo-os/billing#257](https://github.com/milo-os/billing/pull/257), [milo-os/openmeter-provider#119](https://github.com/milo-os/openmeter-provider/pull/119), [datum-cloud/staff-portal#854](https://github.com/datum-cloud/staff-portal/pull/854)) | Billing merged, not released; provider and portal drafts | The prorated first period is a line with its own part-period [service period](#service-periods). The contract summary's first-invoice estimate is replaced by [Stripe's preview](#previewing-a-contracts-invoices). |
| Colocation and bare metal fees on staging's price list ([datum-cloud/infra#9445](https://github.com/datum-cloud/infra/pull/9445), [datum-cloud/fleet#73](https://github.com/datum-cloud/fleet/pull/73)) | Open | The first catalog items to get Stripe Products and tax codes. |
| A searchable country picker for billing addresses ([datum-cloud/cloud-portal#1827](https://github.com/datum-cloud/cloud-portal/pull/1827)) | Merged | Complete customer addresses, which [tax on line items](#tax-on-line-items) depends on. |

None of these change when the writer lands. Each one's data moves from a
line's name or our own arithmetic onto the Stripe line itself.

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
- **Staff.** As a staff member adding a charge, I want to set its tax code, or
  mark it not taxable, so that it's taxed like the product it is.
- **Staff.** As a staff member, I want to bill a customer for an installation
  or a period of service that isn't part of their contract, with the dates it
  covers, so that it appears on their next invoice with everything else.
- **Staff.** As a staff member, I want to add a note to a line, so that I can
  explain an unusual charge to finance and the rest of staff.
- **Staff.** As a staff member writing a contract, I want to see the actual
  invoices it will produce, tax included, so that I can check them with the
  customer before I save it.
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
| Tax code | The product's tax code, or the one staff picked. See [Tax on Line Items](#tax-on-line-items). | Per-product tax |
| Tax per line | Stripe Tax | The customer and staff see what each line is taxed |
| Note | Staff, for charges they add; the contract, for its fees | Context for staff and finance. Kept in the line's metadata, so it's never printed on the invoice. |
| Metadata | Our IDs: the price, the contract, the account, OpenMeter's line and invoice | Tracing a line back to where it came from |

Discounts stay as they are today in amount. They're written as Stripe
discounts on the line they reduce, not as negative lines, because Stripe Tax
won't calculate an invoice with negative items.

### Price Overrides

The Product says what was sold, and the line says what it cost. So a
customer's own price never needs a Product of its own.

| Override | How it's written |
|---|---|
| A customer's price (`basedOn`) | It has its list price's SKU, so it uses the list price's Product. The line's unit price is the customer's. The list price goes in the line's metadata, so finance can report each customer's price against list. |
| A customer's usage rates | The same: the list rate's Product, the customer's rate as the unit price. Tiered usage is one line per tier, each with its tier's unit price. |
| An Offer's percentage discount | A Stripe coupon on the line, not a lower unit price. Reports show the full price and the discount apart, and Stripe Tax handles it. |
| A contract's quantity | The line's quantity. An amendment that changes the quantity changes the line, not the Product. |
| A custom fee with no list price | Staff pick a SKU from the catalog, so it joins that Product. A new SKU is a deliberate choice, which creates a new Product. A custom fee without a SKU follows the [no-SKU rule](#stripe-products). |
| A prorated period | The line can't show quantity × unit price and still match the prorated amount. See [Open Questions](#open-questions). |

None of these changes how a product is taxed. A customer's price has its
list price's tax code, as it has its SKU.

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
- **A note**, either way, for staff only. The customer never sees it.
- **Invoice now**, as an option, for a charge that shouldn't wait for the
  period to close.

The charge is a platform resource, with the same audit trail and approvals as
the rest of billing. It shows in the account's invoice preview before it's
billed. OpenMeter takes it as a pending line on the gathering invoice, so it
lands on the same invoice as everything else and keeps its period.

Creating invoices by hand in Stripe keeps working, and they keep showing in
the portals. It becomes the exception, not the routine.

### Previewing a Contract's Invoices

While staff write a contract in the staff portal, they can preview its
invoices as Stripe will produce them. Today's preview is our own arithmetic.
This one is Stripe's.

- **What it shows.** The contract's first invoice, and the first full one
  after it, line by line: products and SKUs, quantities, unit prices, service
  periods, discounts, tax per line and totals. For a contract billed on the
  account's invoice day, the first invoice shows the prorated first period.
- **How it works.** The preview builds the contract's invoice lines with the
  same code that writes real invoices. It sends them to Stripe's invoice
  preview with the customer and Stripe Tax on. Stripe returns the invoice it
  would create, but creates nothing and sends nothing.
- **Tax is real.** Stripe Tax calculates it from the customer's address, tax
  IDs and exemptions, and from each line's tax code. A customer with no
  address yet can be previewed with one staff enter for the preview.
- **Usage is an estimate.** Before a contract starts there's no usage. The
  preview shows the contract's fees, and usage only if staff enter an
  estimate.

Stripe's preview has no PDF or hosted page, so the staff portal shows it in
the layout of a Datum invoice. If staff want Stripe's own rendering, they can
**open it as a Stripe draft**:

- The integration creates a real draft invoice on the customer in Stripe and
  links to it in the Stripe dashboard.
- Drafts aren't emailed and don't show in the customer's Stripe portal.
- It's deleted when staff close the preview, save the contract, or after a
  day. A deleted draft can't be recovered, and nothing was charged.

A preview draft is a real invoice on a real customer, so it has safeguards:

- **Lines are created on the draft.** Every line is created with the draft's
  invoice ID. A line created loose on the customer would be a pending item,
  and Stripe could put it on the customer's next real invoice.
- **It's tagged as a preview.** Its metadata marks it, and the `Invoice`
  projection and our Stripe webhook handling skip it. It never shows in the
  portals.
- **It can't send itself.** `auto_advance` is off, so Stripe never finalizes
  it. Its memo says it's a preview and not to send it, since anyone in the
  Stripe dashboard could still send it by hand.
- **It's cleaned up.** The integration deletes it when the preview closes or
  the contract is saved, and a sweep deletes any preview draft older than a
  day.
- **Discounts use standing coupons.** Discounts are Stripe coupons, so we
  keep one coupon per discount level, not a new one per preview.

It has two limits:

- **Only people with Stripe dashboard access can open it.** Everyone else
  uses the preview in the staff portal.
- **It's dated today.** Lines have the right service periods, but the
  invoice's own date and due date are the day the draft was made, unless we
  can set them to the first invoice's.

The preview also works for an amendment: it shows the next invoice with the
amended contract.

### Tax on Line Items

Stripe Tax keeps calculating tax from the customer's address, tax IDs and
exemptions, as today. What changes is that each line says how it's taxed.

Every line carries:

| Tax setting | Where it comes from | Notes |
|---|---|---|
| Tax code | The catalog item's `ServicePricing`, or what staff pick for a charge they add | One of Stripe's tax codes, chosen by finance. A customer's price has its list price's tax code, as it has its SKU. |
| Taxable or not | The tax code | A line that isn't taxable uses Stripe's nontaxable tax code. Staff can mark a charge they add as not taxable, with a note saying why. |
| Tax behavior | Always exclusive | Tax is added on top of the price, as today. Every line states it, because Stripe Tax won't calculate a line that leaves it unspecified. |
| Tax location | The catalog item: the customer's address, or the facility's | Customer is how Stripe taxes today. Facility is an [open question](#open-questions). |

- **Catalog items.** `ServicePricing` gets an optional tax code, next to its
  SKU, and an optional tax location. Both are part of the immutable spec, so a
  new tax code for a product means a new price.
- **Charges staff add.** Staff get the catalog item's tax settings, and can
  change the tax code or mark the charge not taxable.
- **Contract fees.** They use their pricing's tax code. Staff can't change
  tax per contract, so the same product is always taxed the same way.
- **Usage.** It uses its usage pricing's tax code.
- **Lines without a tax code** use the account's default, as every line does
  today.
- **Customers.** Exemptions, reverse charge and tax IDs stay per customer, as
  today. stripe-provider keeps them in step with the billing account.
- **Seeing it.** The contract preview and the Stripe invoice show tax per
  line, so staff can check how each product is taxed before a contract is
  saved.

Tax codes don't have to wait for the writer. OpenMeter's rate cards take a
Stripe tax code, and its Stripe app already puts it on the Stripe line. So
tax codes on catalog prices can ship first, through OpenMeter. The writer
then puts the same code on each line itself, and adds tax codes for charges
staff add.

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
| A preview draft is finalized and sent by mistake | Preview drafts never auto-advance, are tagged in metadata, and are deleted by the integration. Only the integration creates them. |

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

stripe-provider keeps a Stripe Product for each SKU in the catalog, with:

- its SKU and name;
- its tax code;
- our metadata linking it to its `ServicePricing`s.

The rules:

- **One Product per SKU, not per price.** Prices are immutable in our
  catalog, so a new price for the same product is a new `ServicePricing` with
  the same SKU. Every price with that SKU shares one Product, and each line
  sets its own unit price. Stripe's reports group a product's old and new
  prices together.
- **Created when the price is applied,** not when it's first billed, so
  previews and preview drafts can use it at once.
- **No Stripe Prices.** Each line passes its unit price against the Product,
  so our immutable prices don't pile up as Stripe Prices.
- **Customer prices don't create Products.** See
  [Price Overrides](#price-overrides).
- **Prices without a SKU** get a Product of their own, named after the
  `ServicePricing`, so nothing is blocked from billing. Finance can add a SKU
  with the next price.
- **Names come from the list price.** When prices with one SKU have
  different display names, the Product takes the name of the newest list
  price. Each line keeps its own price's display name as its description.
- **Tax codes come from the list price**, the same way. Prices with one SKU
  must agree on their tax code.
- **Retired, not deleted.** Stripe won't delete a Product that has been
  invoiced. A Product is archived once no live price uses its SKU, and comes
  back if one does again.
- **One set per environment.** Staging's prices make Products in staging's
  Stripe account, and production's in production's, as the prices
  themselves are kept apart.

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

1. **Tax codes, through OpenMeter.** Add a tax code and tax location to
   `ServicePricing`, and set the tax code on OpenMeter's rate cards. Lines
   are taxed per product from then on, before anything else changes.
2. **Stripe Products.** Create a Product for each catalog item, with its SKU
   and tax code. Nothing on invoices changes yet.
3. **The writer, on staging.** Move staging accounts to it. Compare each
   invoice with what OpenMeter's Stripe app would have written: same totals,
   same tax, better lines.
4. **Charges staff add, and contract previews.** Ship both staff portal
   flows on top of the writer. They share its line building.
5. **Production, account by account.** Internal accounts first, then contract
   customers, then everyone else. Each account moves at its own billing
   boundary.
6. **Retire the stopgap.** Once every account is on the writer, stop putting
   the SKU in line names.

Rolling back an account is the same switch the other way, at its next
billing boundary.

Rough size: a few days for tax codes through OpenMeter, 5–7 engineer-weeks
for Products and the writer, 2–3 more for charges staff add, and 1–2 more for
contract previews.

## Open Questions

- **Tax codes.** Finance needs to choose a Stripe tax code for each catalog
  item, and say which items aren't taxable.
- **Tax by facility.** Some catalog items are taxed where the facility is,
  not where the customer is. Stripe Tax works out tax from the customer's
  location for the whole invoice. We need to ask Stripe whether that means a
  separate invoice per facility.
- **Who can add charges, and up to what amount** without a second approval.
- **Prorated lines.** Quantity × unit price can't equal a prorated amount.
  Stripe accepts decimal quantities, so a line could read "0.43 × $1,000.00",
  though rounding must still match OpenMeter's amount to the cent. Or it
  could read "1 × $430.00", with the period showing it's part of a month.
- **SKUs on everything billable.** Whether billing should require a SKU on
  every price, rather than fall back to a Product per price.
- **Who opens Stripe drafts.** Whether the staff who write contracts have
  Stripe dashboard access. If they don't, the preview in the staff portal is
  what they'll use, and Stripe drafts matter less.
- **One-time fee periods.** Should a setup fee's period be the day it's
  charged, or the contract's first period?
- **Custom Invoicing on staging.** We need to confirm how OpenMeter behaves
  when an account moves app mid-period, and how long it will hold a draft.

## Implementation History

- 2026-11-18: First draft.
- 2026-11-18: Notes are for staff only. Added previews of a contract's
  invoices from Stripe, with safeguards for preview drafts.
- 2026-11-18: Added the work this builds on, and tax settings on every line.
- 2026-11-18: Added price overrides, and the rules for Stripe Products.

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
