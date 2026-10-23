---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Hosted zone claims and ownership by delegation

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Nomenclature](#nomenclature)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Phase 1: Claims on shared nameservers](phase1.md)
  - [How other providers do it](#how-other-providers-do-it)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed](#infrastructure-needed)

## Summary

Datum DNS needs to balance two customer needs: using the product with as little
friction as possible and keeping domain ownership secure. It is that balance
that drives the entirety of this design proposal.

We must enable a customer to host their domains with Datum and begin serving DNS
records without verification steps. The mere act of adding the domain
example.com and an AAAA record for foo.example.com to their project should make
Datum immediately begin resolving it.

We also need to guard against squatters and hijackers, and operate resiliently
amid the general tumult of the Internet.

This proposal makes hosted DNS work the way customers expect from other
providers, in two phases:

- **[Phase 1](phase1.md) (this proposal's ask)** enables a zone to be served as
  soon as it's created. A domain delegated to Datum and hosted by the customer's
  zone counts as verified, with no TXT step. If another project already claims
  the name, the customer proves ownership with a TXT record and takes the name
  over. A deleted zone's name stays reserved while the domain is still delegated
  to Datum.
- **Phase 2 (designed, deferred)** gives every zone its own unique set of four
  nameservers across four TLDs. Delegation then proves _which_ customer owns a
  domain, so no name ever needs to be exclusive, and nameserver attacks can be
  attributed to the zones they target.

Phase 1 is built so Phase 2 changes what the nameserver allocator returns rather
than how the system fits together.

This proposal amends the
[Domain Ownership Verification](../README.md#domain-ownership-verification)
design in its parent, [Domains and DNS](../README.md), for domains whose DNS
Datum hosts. That design requires a TXT record for every domain. Under this
proposal, hosted domains are verified by delegation, and the TXT record flow
remains for domains hosted elsewhere and for contested names.

It also completes
[Wildcard Hostnames on ALBs](https://github.com/datum-cloud/enhancements/pull/914),
which accepts a Datum DNS zone as proof for a wildcard once Datum serves it and
the registry delegates to it. Verification by delegation is that proof, and the
5-day lapse here is the re-verification that proposal lists as future work.

## Motivation

### The customer problem

A customer who delegates their domain to Datum before they verify it gets stuck:
the domain never verifies, and the zone never serves. The only way out is to
delegate the domain somewhere else, add a TXT record there, wait, and then
delegate it back. Separately, any project can claim any domain name first and
block its rightful owner from using it on Datum.

Two domains reproduced the dead end in production. Both were already delegated
to Datum when their zones were created. The zones published no nameservers, so
verification reported that the zone was ready but had no nameservers to match.
One of the two was under a TLD with no RDAP service, and the recursive lookup
that replaces it returned a stale answer, then SERVFAIL. Both domains verified
only after they were delegated back to their registrar's nameservers, the TXT
records were added there, and the delegation was moved to Datum again.

Delegating first is the natural order for anyone moving DNS to a new provider,
and it's how Cloudflare and Route 53 work. On Datum it's a dead end with no
message that says so.

### Why it happens

Every zone on Datum is served from the same four nameservers (`ns1` to
`ns4.datumdomains.net`). Because of that, "this domain is delegated to Datum"
doesn't say which customer delegated it. Two rules follow:

- dns-operator won't serve a zone until its domain is verified
  ([dns-operator#100](https://github.com/datum-cloud/dns-operator/pull/100)), so
  a stranger's zone can't answer for someone else's domain.
- Verification can't trust the delegation alone
  ([network-services-operator#421](https://github.com/datum-cloud/network-services-operator/issues/421)
  rejected delegation-as-proof for this reason).

Recent changes tried to let delegation count anyway
([network-services-operator#519](https://github.com/datum-cloud/network-services-operator/pull/519),
[dns-operator#200](https://github.com/datum-cloud/dns-operator/pull/200)). They
ship in production but have no effect: the waiting zone reads a project-side
DNSZoneClass that lists no nameservers, so there's nothing for the delegation to
match. If they did take effect, any project with a pending zone would be
verified the moment the rightful owner delegated to Datum.

A second, independent problem: the first project to create a zone for a name
claims it across all of Datum, verified or not. The
[Domains and DNS](../README.md#proof-of-domain-ownership) proposal describes
this hijack ("Hacker Corp" claiming `cybercorp.com`), and nothing today lets the
rightful owner take the name back.

### Goals

- A customer can create a zone and point their registrar at Datum in either
  order, and the zone serves.
- A customer who hosts their DNS on Datum never needs a TXT record to use their
  domain with ALBs and certificates, including wildcard hostnames.
- A project can grant a name under its zone to another project, so a parent and
  child zone in different projects of the same organization don't conflict.
- The rightful owner of a domain can always take it back from a project that
  claimed it first.
- Deleting and recreating a zone never opens a window for another project to
  take the domain.
- Verification by delegation is re-checked, not permanent, and a customer who
  moves DNS away is told what to do before it lapses.
- dns-operator and network-services-operator can't deadlock on each other's
  status.
- End-to-end tests observe each control refusing what it should refuse.
- Datum publishes an SLO for the time from zone creation to served-and-verified.

### Non-Goals

- **Customer-facing presentation.** This proposal defines the states and facts a
  good experience needs. How the portal and datumctl show them is UX work under
  [enhancements#905](https://github.com/datum-cloud/enhancements/issues/905).
- **Ownership proof for ALB hostnames whose DNS is hosted elsewhere.** These
  keep using TXT or HTTP, unchanged.
- **Hardening the HTTP verification check.** It follows redirects, has no egress
  policy, and reflects the response body into status. That's tracked separately
  as a security item.
- **DNSSEC, zone transfers, and new record types.**

## Nomenclature

It's easy to confuse the many nouns under the DNS umbrella. So for clarity:

- **domain**: a name like `datum.net` registered with a registrar.
- **registrar**: a company that registers domains for customers. For example,
  `datum.net` is registered through a registrar.
- **registry**: operates a top-level domain, runs its nameservers, and accredits
  registrars. For example, Verisign runs `.net` and permits registrars to sell
  `.net` names to customers.
- **authoritative nameserver**: a nameserver that answers for a zone from its
  own data, rather than by asking other nameservers.
- **delegation**: the parent zone naming the nameservers authoritative for a
  domain. The `.net` registry delegates `datum.net` to `ns1`, `ns2`, `ns3`, and
  `ns4.datumdomains.net`, set through the registrar.
- **zone**: a collection of DNS records on an authoritative nameserver. The
  `datum.net` zone contains various DNS records.
- **record**: a name, a type, and a value (plus a TTL which may or may not be
  explicit and a class which is almost always `IN`). For example,
  `www.datum.net` has a `CNAME` record whose value is the ALB's hostname, which
  has two `A` records: `67.14.164.1` and `67.14.165.1`.
- **`Domain` and `DNSZone`**: Datum's API resources. A `Domain` tracks who owns
  a domain and how that was proven. A `DNSZone` is a zone Datum hosts.

## Proposal

### User Stories

#### Customer hosts a domain with Datum

I add `example.com` to my project, along with an AAAA record for
`foo.example.com`. Datum's authoritative nameservers answer for it immediately,
with no verification step. Datum tells me that to actually use my hosted zone, I
have to delegate my domain to the authoritative nameservers Datum gives me,
through my registrar.

#### Customer who delegates afterwards

I add `example.io` to Datum along with the record `www.example.io`, and add it
as a custom hostname to an ALB. Datum shows me four nameservers, which I then
set at my registrar. Within minutes the Internet starts resolving
`www.example.io`, and my ALB starts receiving traffic.

#### Customer who delegates first

My registrar already delegates `example.org` to Datum's nameservers when I add
the zone. It answers immediately, and within minutes Datum sees the delegation,
so I can use `example.org` with an ALB without adding any verification record.

#### Customer whose name was squatted

I try to legitimately add `cybercorp.com` and learn another project already
claims it. Datum gives me a TXT record, and I add it where my DNS is currently
hosted. The name becomes mine. The other project's zone stops answering, but
they can still export its records.

#### Customer who tries to squat on a name

I try to illegitimately add `cybercorp.com` and learn another project already
claims it. Datum gives me a TXT record, but I can't add it where the DNS is
currently hosted because it's not mine. I scream into the void.

#### Customer who deletes and recreates a zone

I delete my `example.org` zone by mistake while my domain is still delegated to
Datum. The zone stops answering, but nobody else can claim the name. I recreate
the zone and it answers again.

#### Customer with a child zone in another project

My organization hosts `example.com` in a production project. I create a
development project and add a zone for `dev.example.com`. Datum refuses it and
names the production project. From the production project, I grant
`dev.example.com` to the development project, and its zone serves.

#### Customer who moves a zone to another project

I want `example.com` hosted from a new project. From the project that has it, I
hand the name over to the new project. The new project's zone serves as soon as
it exists, and the old zone stops answering.

#### Customer who uses a wildcard hostname

I host `example.com` on Datum and delegate it. I add `*.example.com` to an ALB.
Datum accepts the wildcard on the strength of the delegation, creates the
certificate's challenge record in my zone, and issues the certificate. I never
add a TXT record.

#### Customer who moves DNS away from Datum

I move `example.com` to another DNS provider but keep `www.example.com` on a
Datum ALB. Datum tells me that my ownership proof lapses in 5 days and shows me
the TXT record and the certificate challenge record to publish at my new
provider. I publish them, and nothing goes down.

#### Customer who keeps DNS elsewhere

I keep DNS at my registrar and point `www.example.com` at a Datum ALB with a
CNAME. I prove ownership with a TXT or HTTP record, as I do today.

### Notes/Constraints/Caveats

- **Ownership and hosting are separate concerns.** Datum must know who may use
  `www.example.com` on an ALB or certificate, even when Datum doesn't host
  `example.com`'s DNS. Ownership therefore lives on the `Domain`
  (network-services-operator), not on the `DNSZone` (dns-operator).
- **The delegate-first story is the one Phase 1 can't fully protect.** This
  proposal encourages delegating first, and accepts that a domain delegated to
  Datum before any zone exists goes to whoever creates the zone first. Phase 2
  closes it. A cheaper middle step, a per-project nameserver set drawn from a
  pool as Cloudflare does per account, is listed under
  [Alternatives](#alternatives).
- **Shared nameservers allow one served zone per name.** In Phase 1, two
  projects can't both be served for `example.com`, because Datum's nameservers
  can't tell which one a resolver means. Phase 1 keeps names exclusive and adds
  ways to resolve conflicts. Phase 2 removes the exclusivity. Several projects
  can then have zones for the same domain or subdomain, and only the one the
  registry delegates to is active.

### Risks and Mitigations

| Risk                                                                                                                                                                                             | Phase 1 mitigation                                                                                                                                                                                                                                                                                                                                                                   | Phase 2                                                                          |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------- |
| **Dangling delegation:** a domain still delegated to Datum with no zone can be claimed by anyone, who then receives its traffic and is verified by delegation. Vercel carries the same exposure. | A deleted zone's name stays reserved while the domain is delegated to Datum, which closes the case where a zone existed. A domain that was delegated to Datum but never had a zone stays exposed, and this proposal accepts that risk. A zone created for a name that is already delegated to Datum with no prior claim is flagged in status and alerted on, so the case is visible. | Closed. A new zone gets new nameservers, so a stale delegation never matches it. |
| **Contest trap:** the rightful owner of a squatted domain who has already delegated to Datum can't publish a TXT record, because Datum serves the squatter's zone.                               | A support path: support verifies ownership out of band and releases the name. This is expected to be rare.                                                                                                                                                                                                                                                                           | Closed. Delegation itself identifies the owner.                                  |
| **Subdomain takeover:** another project creates a zone under your zone, and Datum's nameservers answer for that part of your domain from its zone.                                               | Claims cover subtrees, so the other project's zone is Contested and never served.                                                                                                                                                                                                                                                                                                    | Closed. Parent and child zones never share an address.                           |
| **Removing the verification gate** makes every zone currently waiting for verification start serving at once, and each one becomes an Active claim on its name.                                  | Before enabling in production, review every waiting zone. Those whose domain is already delegated to Datum are the takeover case above. The rest become claims that block the rightful owner until a contest, which Phase 1a resolves only through support.                                                                                                                          | Not applicable                                                                   |
| **A registry lookup failure** looks like a delegation change and revokes verification.                                                                                                           | Failed lookups never count as a change. The 5-day grace period starts only from a confirmed change.                                                                                                                                                                                                                                                                                  | Same                                                                             |
| **A lapse breaks a customer who moved DNS away but kept an ALB.** Their hostnames and certificates stop.                                                                                         | The grace period is a re-prove window. Status says the proof lapses, and names the TXT record and the certificate challenge record to publish elsewhere.                                                                                                                                                                                                                             | Same                                                                             |

## Design Details

The design is in two documents:

- [Phase 1: Claims on shared nameservers](phase1.md) is this proposal's ask. It
  can ship in two parts, with the critical fixes in the first. It covers
  delivery, claims, the ownership verdict, failure handling, which operator
  decides what, the changes by component, the test plan, and production
  readiness.
- Phase 2: Per-zone nameservers is designed, deferred, and published separately.
  It covers the per-zone model, its safeguards, nameserver allocation and
  naming, and migration.

### How other providers do it

|                               | Vercel                              | Cloudflare                             | Route 53                        | Phase 1                                   | Phase 2                       |
| ----------------------------- | ----------------------------------- | -------------------------------------- | ------------------------------- | ----------------------------------------- | ----------------------------- |
| Nameservers                   | Shared (`ns1`/`ns2.vercel-dns.com`) | Per zone, consistent across an account | Per zone                        | Shared                                    | Per zone                      |
| Proof before serving          | None (first-come)                   | Activates on delegation                | None                            | None (first-come)                         | None                          |
| Same name in several accounts | No. A TXT proof transfers the name. | Yes, with different nameservers        | Yes, with no shared nameservers | No. A TXT proof transfers the name.       | Yes, with no shared addresses |
| Ownership proof               | TXT when contested                  | Delegation                             | Not needed                      | Delegation to an Active zone, or TXT/HTTP | Delegation, or TXT/HTTP       |

Sources:

- [Vercel nameservers](https://vercel.com/docs/domains/working-with-nameservers)
- [Vercel domain ownership errors](https://vercel.com/docs/domains/troubleshooting)
- [Cloudflare nameserver options](https://developers.cloudflare.com/dns/nameservers/nameserver-options/)
- [Cloudflare full setup](https://developers.cloudflare.com/dns/zone-setups/full-setup/setup/)
- [Route 53 CreateHostedZone](https://docs.aws.amazon.com/Route53/latest/APIReference/API_CreateHostedZone.html)
- [Route 53 concepts](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/route-53-concepts.html)

## Implementation History

- 2026-09-29: dns-operator v0.9.0 and network-services-operator v0.31.0 ship
  delegation-as-proof. It has no effect in production.
- 2026-10-02: Proposal drafted, after the delegate-first dead end was reproduced
  in production.

## Drawbacks

- Phase 1 accepts the dangling-delegation risk for domains that were delegated
  to Datum but never had a zone.
- Phase 1 needs a manual support path for contested domains already delegated to
  Datum.
- Phase 1 needs an explicit grant for a child zone in another project, where
  Phase 2 needs none.
- Phase 2 costs multiple domains, many glue records, per-address serving, and a
  migration for every customer.

## Alternatives

- **Status quo: TXT for everyone.** It's secure, but there's too much friction
  and delegate-first stays a dead end, and squatting still blocks rightful
  owners.
- **Delegation-as-proof on shared nameservers, without claims.** This is what
  network-services-operator#519 ships. If it worked, any pending zone would be
  verified when the rightful owner delegated to Datum.
- **A per-claim challenge nameserver for contests.** The owner adds a nameserver
  unique to their claim at the registrar. This avoids the support path but needs
  a host record at Verisign for each challenge. It's a candidate if contests
  turn out to be common.
- **Per-project nameserver sets.** Each project gets its own set of nameserver
  names drawn from a pool, as Cloudflare assigns per account. Delegation then
  identifies the project, which closes the dangling delegation, the contest
  trap, and the parent-and-child conflict with far fewer addresses than Phase 2.
  Two projects that draw the same set fall back to Phase 1 rules against each
  other. It still needs per-address serving. It's a candidate if Phase 2 is
  deferred for long.
- **Phase 2 only.** It's the stronger design, but a much larger investment
  before customers see any improvement.

## Infrastructure Needed

Phase 1 needs none. Phase 2 needs:

- Additional domains, which Phase 2 names.
- A secret for the allocation key.
