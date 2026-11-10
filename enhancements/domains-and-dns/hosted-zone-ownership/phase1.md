# Phase 1: Claims on shared nameservers

Part of [Hosted zone claims and ownership by delegation](README.md), which has
the background, goals, and user stories.

## Delivery

Phase 1 can ship in two parts. Phase 1a carries the critical fixes, the ones
customers notice. Phase 1b completes the design this document describes.

Phase 1a builds on behavior the operators already have:

- dns-operator already rejects a second project's zone for a claimed name,
  before the verification gate runs. That covers only the exact name. Today the
  gate is what stops a project from creating a zone under another project's
  domain, so claims must cover subtrees before the gate comes out.
- network-services-operator's delegation check already counts only zones that
  are accepted and programmed in the same project, so a rejected zone never
  counts.
- Removing the gate restores how zones were served before
  [dns-operator#100](https://github.com/datum-cloud/dns-operator/pull/100).

### Phase 1a

- Replace the claim ConfigMap with a claim resource, keyed by project, that
  outlives the zone and covers the name's subtree. A zone under or over another
  project's zone is Contested. This lands before the gate is removed.
- A grant: the incumbent project can give a name under its claim, or the whole
  claim, to another project.
- Remove the verification gate behind a configuration flag, and remove the
  apex-only exception that lets a zone pending verification count for delegation
  in the same change.
- Accept verification by delegation as proof for wildcard hostnames.
- Publish the serving DNSZoneClass's nameservers on each zone, so the delegation
  check can match.
- Reserve a deleted zone's name for its project for a fixed period, such as 7
  days.
- Lower the registration refresh interval from 24 hours to about 1 hour.
- Read delegations from the parent zone's authoritative servers for every name,
  with RDAP as a secondary source. Without this, non-RDAP domains keep using
  TXT, and names below a registered domain have no definition of delegation.
- Flag, in status and in an alert, a zone created for a name that is already
  delegated to Datum with no prior claim.
- End-to-end tests: delegating first verifies with no TXT record, a second
  project's zone is never served, whether it's for the same name or a name under
  it, and a granted child zone is.
- A support runbook for releasing a squatted name by hand.

Customers get a hosted domain that serves immediately, delegation in either
order, no TXT record for hosted domains or their wildcards, a safe delete and
recreate, and child zones across their projects.

Phase 1a accepts these risks, which are the ones Datum carried before
dns-operator#100, plus a support path:

- A dangling delegation can be claimed once the fixed reservation expires.
- Squatting is resolved by support, not by the rightful owner.
- Verification stays permanent.

### Phase 1b

- Contests the rightful owner resolves with a TXT record, the Displaced state,
  and the cooldown between contests
- A reservation that lasts as long as the domain is delegated to Datum
- Re-checking delegation-based verification, with the 5-day grace period as a
  re-prove window
- A delegation re-check measured in minutes, with per-registry rate limits
- The rest of the test plan

## Claim on a zone name

One claim exists per name across Datum, held by a project, and it covers the
name's whole subtree. A zone for a name under or over another project's zone is
Contested, and the refusal names the incumbent. Datum's shared nameservers
answer from the most specific zone they host, so without this rule a zone for
`shop.example.com` would take over that part of another project's `example.com`.
[Wildcard Hostnames on ALBs](https://github.com/datum-cloud/enhancements/pull/914)
applies the same rule to hostname claims.

The claim is its own resource, not a side effect of the zone. Today it's a
ConfigMap in the serving cluster, owned by the individual zone and deleted with
it. The states below need a claim that is keyed by project, outlives its zone,
records when and why it changed, and is one claim across Datum rather than one
per serving cluster.

**Grants.** The incumbent can grant a name under its claim to another project,
which then has its own claim for that subtree, or hand the whole claim over.
This is how an organization hosts `example.com` in one project and
`dev.example.com` in another, how a customer moves a zone between projects, and
how a domain owner gives a subdomain to a third party that hosts on Datum. A
grant is recorded on both claims. There is no organization-wide scope: claims,
like everything else, belong to a project.

![Claim states: Active is served; Reserved, Contested, and Displaced are
not](claim-states.png)

The diagram's source is [claim-states.puml](claim-states.puml).

| State     | Meaning                                                                            | Served           |
| --------- | ---------------------------------------------------------------------------------- | ---------------- |
| Active    | The first project to create a zone for the name, or the winner of a contest        | Yes, immediately |
| Reserved  | The incumbent deleted the zone while the domain is still delegated to Datum        | No               |
| Contested | Another project created a zone for a claimed name, or for a name under or over one | No               |
| Displaced | Lost a contest. Records are kept.                                                  | No               |

- **Contests resolve immediately, by TXT only.** A fresh TXT challenge, issued
  for the contest, shows the challenger controls the domain's DNS where it's
  actually delegated. The incumbent's zone becomes Displaced and stops serving.
  Its records stay so its project can export them. HTTP proof never resolves a
  contest: it shows control of one web host, not of DNS, and a dangling CNAME or
  a shared host would be enough to take a zone. A `Domain`'s existing
  verification, which is permanent, is not a contest proof either.
- **When both projects answer the challenge, the most recent one wins.** The
  domain has changed hands. After a contest, the name can't be contested again
  for a cooldown period, so two projects can't flip it back and forth.
- **A reservation lasts as long as the delegation points at Datum.** It ends
  when the re-check confirms the delegation left Datum for 5 days, or when
  support releases it.
- **Existing zones become Active for their names**, with no customer action.

> [!IMPORTANT]
>
> Unresolved: what a contest settles between a parent and a child. A project
> that proves ownership of `shop.example.com` may have been given that subdomain
> by the owner of `example.com`, so displacing the parent zone would be wrong.
> The cooperative case is a grant. For the contested case, the proposed answer
> is that the proven child is carved out of the parent's subtree, exactly as a
> grant would, and the parent keeps the rest.

## Ownership verdict

The claim and the verdict are independent:

|                                   | Not delegated to Datum                                                   | Delegated to Datum                           |
| --------------------------------- | ------------------------------------------------------------------------ | -------------------------------------------- |
| Active                            | Served, but nobody asks Datum. Unverified, unless proven by TXT or HTTP. | Served and live. **Verified by delegation.** |
| Reserved, Contested, or Displaced | Not served. Verified only by TXT or HTTP.                                | Not served. Verified only by TXT or HTTP.    |

- **Verified by delegation** requires both: the project's zone is Active, and
  the parent delegates the name to any of Datum's nameservers. Delegation is
  re-checked periodically. If it leaves Datum, the verdict lapses after a 5-day
  grace period, long enough for a long weekend or a slow registrar.
- **The grace period is a re-prove window.** A customer who moves DNS away but
  keeps hostnames on an ALB would otherwise lose them, because ALB hostnames
  require a verified `Domain`. From the first confirmed change, the `Domain`'s
  status says the proof lapses on a date and names the TXT record and, for
  wildcards, the certificate challenge record to publish at the new provider.
- **Delegation is read from the parent zone's authoritative servers**, for every
  name. For a registered domain that's the TLD's servers. For a name below one,
  such as `shop.example.com`, it's the servers for `example.com`, wherever it's
  hosted. The parent is what resolvers see, and it has no lookup rate limit.
  RDAP stays the source for registration data and a secondary source for
  nameservers.
- **Verified by TXT or HTTP** stays permanent. Customers usually remove those
  records after verifying, so re-checking them would revoke almost everyone.
- **An Active zone that the domain delegates to counts as owning the domain for
  ALBs and certificates, wildcards included.** A project in that position
  already answers the domain's DNS. It could get a certificate from any public
  CA, so a second proof would add friction without adding security. The wildcard
  rules in network-services-operator today accept only a TXT record and refuse a
  Datum zone until Datum serves it and the registry delegates to it. That
  condition is this verdict. The DNS-01 challenge record for a hosted zone
  ([network-services-operator#567](https://github.com/datum-cloud/network-services-operator/issues/567))
  resolves only when the project's zone is the served one, so the proof and the
  challenge path agree.

## Failure handling

| Failure                                                                                     | Behavior                                                                                                                                             |
| ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| A registry lookup fails (RDAP timeout, WHOIS rate limit, SERVFAIL)                          | The last known delegation is kept, and the lookup is retried with backoff. The error is reported separately and never counts as a delegation change. |
| The TLD has no RDAP service, such as `.io`                                                  | No effect on delegation, which comes from the parent's authoritative servers. Registration data is missing until WHOIS answers.                      |
| A partial delegation lists only some of Datum's nameservers, or mixes in another provider's | Counts as delegated to Datum if any Datum nameserver is listed                                                                                       |
| The delegation leaves and returns within 5 days                                             | The grace timer resets. Nothing lapses or is released.                                                                                               |
| Two projects claim a name, or a parent and child, at the same time                          | The first write wins atomically, and the other becomes Contested.                                                                                    |
| Support releases a reserved or contested name                                               | An explicit, audited action, recorded on the claim                                                                                                   |
| The `Domain` for a Reserved name is deleted                                                 | The claim keeps re-checking the delegation on its own. A reservation never depends on a `Domain` existing.                                           |
| A zone is created for a name already delegated to Datum with no claim                       | It serves, and the claim is flagged `DelegatedBeforeClaim` so operators can review it                                                                |
| Datum's own nameservers can't be reached during a check                                     | Treated as a lookup failure                                                                                                                          |

The proposal exposes these primitives for the portal and datumctl: the claim
state, the verification method, and the delegation status, plus the reason for
each.

## What Kubernetes operator decides what

Phase 1 can't be strictly one-way. The reservation and the contest need
ownership facts inside hosting. The rule is instead that **neither operator
reads a decision that waits on its own output**.

| Component                             | Owns                                                                                                                                             | Reads from the other                                                                                                               |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------- |
| dns-operator (hosting)                | Zones and serving. The **claim** on each name and its subtree: first claim, reservation, contest resolution. Serves the Active zone immediately. | From `Domain`: the delegation (for the reservation) and the contest TXT result. Never delegation-based verification or HTTP proof. |
| network-services-operator (ownership) | `Domain`: registry and delegation lookups, TXT/HTTP checks, and the verdict ALBs and certificates use                                            | From `DNSZone`: whether the zone is Active, and the nameserver set it publishes (the shared set in Phase 1)                        |

This can't deadlock:

- The reservation and the contest depend only on the delegation lookup and on
  the contest TXT, which never consult zones.
- Delegation-based verification depends on the zone being Active, which never
  waits on verification.
- The verification gate, which made
  [network-services-operator#421](https://github.com/datum-cloud/network-services-operator/issues/421)
  a cycle, is removed.

## Changes by component

- **dns-operator:**
  - Remove the wait for `Domain` Verified before serving, behind a configuration
    flag.
  - Replace the claim ConfigMap with a claim resource keyed by project that
    outlives its zone and covers the subtree, so a parent and child zone in
    different projects conflict.
  - Add the Reserved, Contested, and Displaced claim states, grants, and the
    `DelegatedBeforeClaim` flag.
  - Publish the serving DNSZoneClass's nameservers on each zone's status,
    replacing the project-side class that lists none.
  - Keep creating the `Domain` for each zone.
- **network-services-operator:**
  - The delegation check counts only the Active zone, and the apex-only
    exception for zones pending verification is removed.
  - Read delegations from the parent's authoritative servers for every name,
    with RDAP as a secondary source.
  - Re-check delegation-based verification, with the 5-day grace period as a
    re-prove window.
  - Accept verification by delegation for wildcard hostnames.
  - Contests use a fresh TXT challenge. HTTP proof is never a contest proof.
- **Portal and datumctl:** build on the new primitives. Design is out of scope
  here.

## Signals

- Time from zone creation to served-and-verified. This feeds the convergence
  SLOs under
  [enhancements#905](https://github.com/datum-cloud/enhancements/issues/905).
- The number of Reserved, Contested, and Displaced claims.
- The registry lookup error rate per TLD. This rate would have exposed the `.io`
  failure.

## Test plan

The current suites verify their own domains before asserting anything
([dns-operator#130](https://github.com/datum-cloud/dns-operator/issues/130),
[network-services-operator#427](https://github.com/datum-cloud/network-services-operator/issues/427)),
so none of these paths is exercised today. Each test asserts what Datum's
nameservers actually answer, not only what a condition says.

| Test                           | Must observe                                                                                                                                                  |
| ------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Delegate first                 | The zone serves immediately. After delegation it's Verified with no TXT record.                                                                               |
| Only the Active zone is served | A second project's zone for the same name: Datum answers from the Active zone only                                                                            |
| Subtree                        | A second project's zone for a name under the Active zone is Contested, and Datum answers for that name from the Active zone                                   |
| Contest                        | The challenger takes the name over with a TXT record, and the displaced zone stops answering.                                                                 |
| HTTP proof refused             | A challenger with HTTP proof only, and a `Domain` already verified by TXT long ago, is refused as a contest proof.                                            |
| Grant                          | A child zone in a second project is Contested until the incumbent grants it, then serves. A handover moves the claim between projects.                        |
| Subdomain, parent elsewhere    | A zone for `shop.example.com` whose parent is hosted outside Datum verifies when that parent delegates it to Datum.                                           |
| Wildcard                       | A wildcard hostname on a delegated, Active zone is accepted with no TXT record.                                                                               |
| Reservation                    | A zone deleted while delegated stops answering. Another project's zone is Contested. The same project recreates it, under any zone name, and it serves again. |
| Lapse                          | When the delegation leaves, status names the records to publish, and verification lapses after the grace period (shortened in tests).                         |
| Lookup failure                 | A simulated registry failure doesn't lapse verification.                                                                                                      |

Fixtures:

- A parent zone the suite can write to, for example in Cloud DNS. Delegating a
  subzone from it flips in seconds, and it exercises the same parent-server
  lookup the `.io` fix uses.
- One real registered domain, to test the RDAP path.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

#### How can this feature be enabled / disabled in a live cluster?

A dns-operator configuration flag controls whether the verification gate
applies. Phase 1's network-services-operator changes are fixes and need no flag.

#### Does enabling the feature change any default behavior?

Yes. Zones waiting for verification start serving, and hosted, delegated domains
verify without a TXT record.

#### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Restoring the gate stops unverified zones from serving. Claim states
written while it was enabled stay valid.

#### What happens if we reenable the feature if it was previously rolled back?

Zones waiting for verification serve again. Claims are unaffected.

#### Are there any tests for feature enablement/disablement?

The Phase 1 test plan runs with the flag on. A refusal test runs with the flag
off and asserts an unverified zone isn't answered.

### Rollout, Upgrade and Rollback Planning

#### How can a rollout or rollback fail? Can it impact already running workloads?

Enabling the flag serves every zone that was waiting for verification and makes
each an Active claim. Review every waiting zone first, not only the delegated
ones. Rolling back stops those zones from serving.

#### What specific metrics should inform a rollback?

A rise in Contested or Displaced claims after enabling, or registry lookup
errors that cause verification to lapse

#### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

To be done in staging before production.

#### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

Phase 1: no. Phase 2 retires the legacy `ns1` to `ns4` and the Phase 1 claim
states.

### Monitoring Requirements

#### How can an operator determine if the feature is in use by workloads?

From the claim state counts and the share of domains verified by delegation

#### How can someone using this feature know that it is working for their instance?

Through API status: the zone's claim state, and the `Domain`'s verification
method and delegation status.

#### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

This proposal publishes one SLO: the time from zone creation to
served-and-verified, for a domain already delegated to Datum. It feeds the DNS
product SLOs under
[enhancements#905](https://github.com/datum-cloud/enhancements/issues/905).

#### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

Time from creation to served-and-verified, the registry lookup error rate per
TLD, and verification lapses

#### Are there any missing metrics that would be useful to have to improve observability of this feature?

There's no verification metric today for projects that
custom-resources-state-metrics doesn't scrape. Newer projects are missing from
it.

### Dependencies

#### Does this feature depend on any specific services running in the cluster?

dns-operator, network-services-operator, and RDAP/WHOIS reachability from the
control plane. Phase 2 adds dnsdist or PowerDNS views, and four registrars.

### Scalability

#### Will enabling / using this feature result in any new API calls?

Periodic delegation re-checks per verified domain, rate-limited per registry, as
registration lookups are today

#### Will enabling / using this feature result in introducing new API types?

Yes. The claim becomes a resource of its own in dns-operator.
