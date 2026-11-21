---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

<!-- omit from toc -->
# Pricing by Location

- [Summary](#summary)
- [Motivation](#motivation)
  - [What the Catalog Needs](#what-the-catalog-needs)
  - [What We Have Today](#what-we-have-today)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Facilities](#facilities)
  - [Catalog Prices by Facility](#catalog-prices-by-facility)
  - [Contract Lines](#contract-lines)
  - [Invoices](#invoices)
  - [Tax by Facility](#tax-by-facility)
  - [Usage at a Facility](#usage-at-a-facility)
  - [What Customers See](#what-customers-see)
  - [Notes and Constraints](#notes-and-constraints)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Overview](#design-overview)
- [Rollout](#rollout)
- [Open Questions](#open-questions)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [References](#references)

## Summary

Colocation, power, cross connects and bare metal are sold at a facility. A
customer can have racks in two data centers, each with its own power and
cross connects, and the Q4 2026 service catalog taxes most of these products
where the facility is, not where the customer is.

Our catalog has no idea of location. A fixed fee has one amount, a contract
can include each price once, and every line is taxed at the customer's
address. So today, staff would have to make a catalog price for every product
in every facility, or fall back to one-off contract fees. Both work against
the goal that every price we charge lives in the catalog.

This enhancement adds **facilities** to billing:

- A catalog price can be offered at one or more facilities, with its own
  amount at each where prices differ.
- A contract line names the facility it's for, so the same catalog price can
  appear once per facility, each with its own quantity and amount.
- Invoice lines say which facility they're for, and products the catalog
  taxes by facility are taxed at the facility's address.
- Customers see their contract and invoices by facility.

One-off fees stay for what the catalog really doesn't cover.

## Motivation

### What the Catalog Needs

The Q4 2026 service catalog lists 25 SKUs. Its "Tax Sourcing" column puts
11 of them at the **facility location**:

| Category | SKUs | How they're billed |
|---|---|---|
| Colocation | COLO-RACK, COLO-CAGE-BUILD, COLO-STRUCTURED-CABLING, COLO-RACK-SETUP | Rack space monthly; build-out, cabling and rack setup once per project |
| Power | PWR-PORT, PWR-COMMIT, PWR-OVERAGE, PWR-SETUP | Ports and committed kW monthly; overage metered; setup once, passed through at cost |
| Interconnect | XCONN, XCONN-INSTALL | Cross connects monthly; install once |
| Bare metal | BM-SETUP | Once per server |

Two more, BM-B300-USAGE and BM-B300-COMMIT, are taxed at the customer's
location, but the catalog says "Order specifies server type and location".

The rest are taxed at the customer's location and aren't tied to a place we
sell from: cloud usage (data transfer, DNS, load balancers, tunnels, compute,
storage), IPv4 space, and the platform tools, which are included. They need
nothing from this enhancement.

The catalog also says:

- **Some prices are set per project.** Cage build, structured cabling, rack
  setup and power setup are "usually quoted per project; price on order
  form". They're still catalog products: a $0 list price that staff must
  price on the contract, which billing already supports.
- **Lines need a little detail.** For COLO-RACK-SETUP, "Rack type (GPU, CPU,
  Network) goes in the invoice line description". Bare metal orders name a
  server type.
- **Rack space, cross connects and committed bare metal prorate their first
  month**, which contracts billed on the account's invoice day already do.

### What We Have Today

| | Today |
|---|---|
| Usage prices | A rate can match one value of one of the meter's dimensions, such as `region`. Usage could be priced per location if its meter carried one. |
| Fixed fees (one-time and recurring) | One amount, with no location. |
| Offers and contracts | An Offer includes each price once, keyed by its name, with a quantity, discount, timing and number of periods. A customer price based on a list price can carry its own display name. |
| Staff portal contract editor | The price list has one row per list price. A contract fee outside the catalog can be added any number of times. |
| Tax | A price carries a Stripe tax code. Stripe Tax taxes every line at the customer's address. |
| Locations | Milo has a `Location` resource (a location class, topology and coordinates), used by compute and the edge. It has no street address. |

So a customer with racks in Dallas and Chicago can't be put on one catalog
price for each. Staff could add "Colocation Rack" once from the price list,
then add Chicago as a one-off fee, losing the catalog's SKU, tax code and
reporting. Or someone could add a catalog price per product per facility,
which multiplies the catalog by the number of facilities and still taxes
everything at the customer's address.

### Goals

- Every product in the Q4 catalog can be sold from the catalog at any
  facility, with no one-off fees.
- One catalog product per SKU, whatever the number of facilities. Adding a
  facility doesn't add products.
- A price can differ by facility where it needs to, and stay the same where
  it doesn't.
- A contract can include the same product at several facilities, each with
  its own quantity, amount and note.
- Invoice lines say which facility they're for.
- Products the catalog taxes by facility are taxed at the facility's
  address.
- Customers can see their contract and invoices by facility.

### Non-Goals

- **Where resources run.** This is about billing. Placing workloads, servers
  or racks at a facility stays with the services that do it.
- **Metering by facility.** Usage priced per facility needs its meter to
  carry the facility. Adding that to a meter belongs to the service that
  emits it (power metering, for PWR-OVERAGE).
- **Currencies per facility.** Everything stays in the account's currency.
- **Inventory and capacity.** Whether a facility has a free rack isn't
  billing's concern.
- **Moving a customer between facilities mid-term.** It's an amendment like
  any other: end one line and start another.

## Proposal

### User Stories

- **Staff selling colocation.** A customer takes two racks in Dallas and one
  in Chicago, with power and cross connects at both. Staff add Colocation
  Rack from the price list twice, pick Dallas and Chicago, and set the
  quantity and the price from the order form for each. Every line keeps the
  catalog's SKU and tax code.
- **Staff quoting a project.** A cage build in Dallas is priced on the order
  form. Staff add Colocation Cage Build at Dallas, enter the quoted amount and
  a note for the invoice line ("Cage C-12 build-out").
- **Finance.** Finance sees revenue per product and per facility, and every
  facility product taxed in the facility's jurisdiction.
- **Product.** A new facility opens. Product adds it once, and the catalog
  products sold there get a price for it, or use their default price.
- **Customer.** The billing page lists "Colocation Rack · Dallas · 2 racks"
  and "Colocation Rack · Chicago · 1 rack", and invoices show the same.

### Facilities

A **facility** is a place we sell physical products from: a data center, or a
cage within one if they're priced or taxed differently. Billing keeps a list
of them, each with:

- a name, such as `dfw-1`, and a display name, such as "Dallas";
- its street address, for tax;
- the Milo `Location` it corresponds to, when there is one.

Staff choose from this list. Facilities are platform-wide, like the catalog.

<<[UNRESOLVED where facilities live]>>
Billing could own a small `Facility` resource, or Milo's `Location` could
gain an address and billing would use Locations directly. Locations are about
where workloads run, and not every facility we sell from will be one, so the
proposal leans towards billing's own list that points at a Location when it
can.
<<[/UNRESOLVED]>>

### Catalog Prices by Facility

A fixed-fee catalog price (one-time or recurring) can say which facilities
it's sold at, and its amount at each:

- **One amount everywhere.** The price has an amount and lists the
  facilities it's sold at, or none for everywhere.
- **Amounts by facility.** The price has a default amount and an amount for
  each facility that differs.
- **Priced on the order form.** A $0 amount, as today, means staff must enter
  the customer's amount on the contract, at whatever facility.

A price keeps one SKU, one tax code and one product in Stripe, whatever the
number of facilities. Prices can't change once published, so a new facility
amount means a new version of the price, as any price change does today. The
new version keeps its SKU.

Usage prices already have rates by dimension; [Usage at a
Facility](#usage-at-a-facility) covers them.

### Contract Lines

Each fee on a contract can name a facility. A contract can then include the
same catalog price more than once, at different facilities, each with its
own:

- facility;
- quantity, discount, billing timing and number of periods, as today;
- amount, when the customer's price differs from the list price, as today;
- **note** for the invoice line, such as the rack type or server type.

A price sold only at certain facilities can only be added at those. A price
that's priced on the order form needs an amount at each facility it's added
at.

In the staff portal's contract editor, each price list row gets an "Add
another location" action. Each row shows its facility, and a row can't be
saved without one when its price is sold by facility.

A fee without a facility stays valid, so existing contracts and prices don't
change.

### Invoices

Every invoice line for a fee at a facility says where it is: "Colocation Rack
· Dallas". The note, when there is one, goes in the line's description. SKUs
stay internal: they're in the line's metadata, never in its name or
description.

The facility is also in the line's metadata, so finance can report revenue by
product and facility.

### Tax by Facility

A catalog price says whether it's taxed at the customer's address, as today,
or at the facility's. The catalog's "Tax Sourcing" column says which, for
each SKU.

Stripe Tax can tax a single invoice line at a given address: a **performance
location**, a saved address that a line item points to so Stripe taxes it
there instead of at the customer's address. It's meant for goods and services
delivered at a physical place, it's per line, and Stripe lists invoices among
the places it can be used. So one invoice can carry lines taxed in several
jurisdictions, and we don't need an invoice per facility.

Each facility gets a performance location in Stripe, made by stripe-provider
from the facility's address, as it makes Stripe Products from catalog prices.
stripe-provider writes every Stripe invoice line ([Writing Our Own Stripe
Invoices][stripe-invoicing]), so a line for a price taxed by facility points
to its facility's performance location. Previews do the same, so staff see
each line's tax at its facility before a contract is saved.

The tax code must allow it. Stripe rejects a performance location on a tax
code that doesn't take one, such as digital services. Finance's tax codes for
colocation, power, cross connects and setup work have to be ones that do.

### Usage at a Facility

PWR-OVERAGE is metered and taxed by facility. A usage price can already set a
rate per value of one of its meter's dimensions, so once the power meter
carries the facility, the price can have a rate per facility. The invoice
then needs one usage line per facility, so each can be taxed where it is.

Bare metal usage (BM-B300-USAGE) is taxed at the customer's address, but
showing its facility on the line would match the order form.

### What Customers See

- **Billing page.** The Pricing list groups fees by facility, with quantity and
  amount, and the contract card shows each line's facility.
- **Invoices.** The hosted Stripe invoice shows each line's facility and
  note, and its tax at that facility.
- **Usage.** Usage by facility follows once a meter carries it.

### Notes and Constraints

- **Existing prices and contracts don't change.** A fee without a facility is
  billed as today.
- **OpenMeter.** A contract's fees become OpenMeter rate card items. Each line
  at a facility needs its own item, keyed by price and facility, so OpenMeter
  rates them separately and stripe-provider can tell them apart when it
  writes the invoice.
- **Customer prices.** A customer's price based on a list price has its list
  price's facilities, SKU and tax, as today.
- **Facility names are permanent.** Contracts and invoices refer to them, so
  a closed facility is retired, not deleted.

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| A line is taxed in the wrong jurisdiction | Finance signs off each facility's address and each tax code. The contract preview shows tax per line, at its facility, before a contract is saved. |
| Stripe's performance locations don't cover rack space or power | Confirm with Stripe and finance before building the tax part. Facilities, catalog prices by facility and contract lines are useful without it. |
| Staff pick the wrong facility | The editor shows the facility on every line, and the preview and the customer's contract card repeat it. |
| The catalog grows unreadable | One price per product, with facility amounts inside it, instead of one price per product per facility. |
| Old contracts break | A facility is optional on every line, and nothing existing gets one. |

## Design Overview

The shape below is a sketch for discussion. The design lands in billing's own
docs once this is agreed.

| Area | Change |
|---|---|
| billing | A facility list. `ServicePricing` fees get the facilities they're sold at, amounts by facility, and a tax location (customer or facility). An Offer's fees are keyed by price and facility, with a facility and a note per line. `status.contract` and `status.prices` show facilities. Validation: a facility must exist and the price must be sold there. |
| openmeter-provider | One rate card item per price and facility, with the facility in its metadata. |
| stripe-provider | A Stripe performance location per facility. Invoice lines carry the facility in their name, description and metadata, and point to the facility's performance location when their price is taxed by facility. Previews do the same. |
| staff-portal | Facilities in the contract editor and the price list, more than one row per price, a note per line. A page listing facilities. |
| cloud-portal | Facilities on the Pricing list, the contract card and the upcoming invoices. |
| datumctl billing, Patch | Usage and prices by facility. |

## Rollout

1. **Facilities and catalog prices by facility.** Add the facility list, and
   let fees name facilities and amounts. Put the Q4 facility products on
   staging at the facilities we sell from.
2. **Contract lines by facility.** Offers, contracts, the contract editor and
   OpenMeter rate card items. Invoice lines show the facility in their name.
3. **Customer views.** The cloud portal's Pricing list, contract card and
   upcoming invoices by facility.
4. **Tax by facility.** Once Stripe and finance confirm the tax codes and
   addresses: a performance location per facility, used on every line
   stripe-provider writes for a price taxed by facility.
5. **Usage by facility.** When the power meter carries the facility,
   PWR-OVERAGE gets a rate per facility.

Steps 1 to 3 need nothing from Stripe and help on their own: the catalog
stops multiplying, and contracts stop needing one-off fees for a second
facility.

## Open Questions

- **Where facilities live.** Billing's own list, or Milo `Location`s with an
  address? See [Facilities](#facilities).
- **Which facilities, and their addresses.** The list we sell from today, and
  which are separate tax jurisdictions.
- **Do prices actually differ by facility?** The catalog doesn't say. If they
  never do, facilities could be a contract line setting only, with the catalog
  saying where a product is sold.
- **Stripe tax codes and performance locations.** Which tax codes finance
  picks for rack space, power, cross connects and setup work, and whether
  Stripe confirms each can take a performance location.
- **Bare metal.** Is BM-B300 taxed at the customer's location even though the
  order names a facility, as the catalog says?
- **Notes on lines.** Free text, or a short list per product, such as GPU,
  CPU and Network for rack setup?
- **Power overage metering.** Peak kW or kWh (the catalog says "Confirm
  unit"), and how the meter will report its facility.

## Implementation History

- 2026-11-21: Proposed, from the Q4 2026 service catalog.

## Drawbacks

- It's work across billing, both providers and both portals, for products
  only some customers buy.
- Tax by facility depends on Stripe confirming performance locations for our
  tax codes.

## Alternatives

- **A catalog price per product per facility.** Works today with no code. The
  catalog grows by every facility, prices drift apart, a contract still can't
  hold the same product twice, and tax stays at the customer's address.
- **One-off contract fees for every facility after the first.** Works today.
  Loses the SKU, tax code and reporting, and is exactly what the catalog is
  meant to replace.
- **Facility only on the contract line, never in the catalog.** Simpler, and
  enough if prices never differ by facility (see [Open
  Questions](#open-questions)). It can't say where a product is sold, and the
  catalog can't hold a facility's price.
- **An invoice per facility.** Would tax each at its facility through the
  invoice's address, without performance locations. Customers would get
  several invoices a month, and OpenMeter bills one per account.

## References

- [Writing Our Own Stripe Invoices][stripe-invoicing]
- [Service Pricing, Offers, and Billing Entitlements](../pricing-and-offers/)
- [Stripe: Tax in-person sales at a specific location][stripe-performance-location]
- [Stripe: Tax location-based sales][stripe-location-sales]

[stripe-invoicing]: ../stripe-invoicing/
[stripe-performance-location]: https://docs.stripe.com/tax/optional-tax-location/integration-guide
[stripe-location-sales]: https://docs.stripe.com/tax/location-sales
