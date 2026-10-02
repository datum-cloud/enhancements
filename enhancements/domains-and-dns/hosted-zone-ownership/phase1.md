# Phase 1: Claims on shared nameservers

Part of [Hosted zone claims and ownership by delegation](README.md), which has
the background, goals, and user stories.

## Delivery

Phase 1 can ship in two parts. Phase 1a carries the critical fixes, the ones
customers notice. Phase 1b completes the design this document describes.

Phase 1a builds on behavior the operators already have:

- dns-operator already rejects a second project's zone for a claimed name,
  before the verification gate runs. The Holder and Contested states effectively
  exist today.
- network-services-operator's delegation check already counts only zones that
  are accepted and programmed in the same project, so a rejected zone never
  counts.
- Removing the gate restores how zones were served before
  [dns-operator#100](https://github.com/datum-cloud/dns-operator/pull/100).

### Phase 1a

- Remove the verification gate behind a configuration flag.
- Publish the serving DNSZoneClass's nameservers on each zone, so the delegation
  check can match.
- Hold a deleted zone's claim for its project for a fixed period, such as 7
  days.
- Lower the registration refresh interval from 24 hours to about 1 hour.
- Read delegations from the parent's authoritative servers when RDAP isn't
  available. Without this, non-RDAP domains keep using TXT.
- End-to-end tests: delegating first verifies with no TXT record, and a second
  project's zone is never served.
- A support runbook for releasing a squatted name by hand.

Customers get a hosted domain that serves immediately, delegation in either
order, no TXT record for hosted domains, and a safe delete and recreate.

Phase 1a accepts these risks, which are the ones Datum carried before
dns-operator#100, plus a support path:

- A dangling delegation can be claimed once the fixed hold expires.
- Squatting is resolved by support, not by the rightful owner.
- Verification stays permanent.

### Phase 1b

- Contests the rightful owner resolves with a TXT or HTTP record, and the
  Displaced state
- A hold that lasts as long as the domain is delegated to Datum
- Re-checking delegation-based verification, with the 5-day grace period
- A delegation re-check measured in minutes, with per-registry rate limits
- The rest of the test plan

## Claim on a zone name

One claim exists per name across Datum.

![Claim states: Holder is served; Held, Contested, and Displaced are
not](claim-states.png)

The diagram's source is [claim-states.puml](claim-states.puml).

| State     | Meaning                                                                     | Served           |
| --------- | --------------------------------------------------------------------------- | ---------------- |
| Holder    | The first project to create a zone for the name, or the winner of a contest | Yes, immediately |
| Held      | The holder deleted the zone while the domain is still delegated to Datum    | No               |
| Contested | Another project created a zone for a name that's already claimed            | No               |
| Displaced | Lost a contest. Records are kept.                                           | No               |

- **Contests resolve immediately.** A valid TXT or HTTP proof shows the
  challenger controls the domain's DNS where it's actually delegated. The
  holder's zone becomes Displaced and stops serving. Its records stay so its
  project can export them.
- **When both projects hold a valid proof, the most recent one wins.** The
  domain has changed hands.
- **A hold lasts as long as the delegation points at Datum.** It ends when the
  re-check confirms the delegation left Datum for 5 days, or when support
  releases it.
- **Existing zones become holders of their names**, with no customer action.

## Ownership verdict

The claim and the verdict are independent:

|                               | Not delegated to Datum                                                   | Delegated to Datum                           |
| ----------------------------- | ------------------------------------------------------------------------ | -------------------------------------------- |
| Holder                        | Served, but nobody asks Datum. Unverified, unless proven by TXT or HTTP. | Served and live. **Verified by delegation.** |
| Held, Contested, or Displaced | Not served. Verified only by TXT or HTTP.                                | Not served. Verified only by TXT or HTTP.    |

- **Verified by delegation** requires both: the project's zone is the holder,
  and the registry delegates to any of Datum's nameservers. Delegation is
  re-checked periodically. If it leaves Datum, the verdict lapses after a 5-day
  grace period, long enough for a long weekend or a slow registrar.
- **Verified by TXT or HTTP** stays permanent. Customers usually remove those
  records after verifying, so re-checking them would revoke almost everyone.
- **Holding a delegated zone counts as owning the domain for ALBs and
  certificates.** A project in that position already answers the domain's DNS.
  It could get a certificate from any public CA, so a second proof would add
  friction without adding security.

## Failure handling

| Failure                                                                                     | Behavior                                                                                                                                             |
| ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| A registry lookup fails (RDAP timeout, WHOIS rate limit, SERVFAIL)                          | The last known delegation is kept, and the lookup is retried with backoff. The error is reported separately and never counts as a delegation change. |
| The TLD has no RDAP service, such as `.io`                                                  | The delegation is read from the parent zone's authoritative servers without recursion.                                                               |
| A partial delegation lists only some of Datum's nameservers, or mixes in another provider's | Counts as delegated to Datum if any Datum nameserver is listed                                                                                       |
| The delegation leaves and returns within 5 days                                             | The grace timer resets. Nothing lapses or is released.                                                                                               |
| Two projects claim a name at the same time                                                  | The first write wins atomically, and the other becomes Contested.                                                                                    |
| Support releases a held or contested name                                                   | An explicit, audited action, recorded on the claim                                                                                                   |
| Datum's own nameservers can't be reached during a check                                     | Treated as a lookup failure                                                                                                                          |

The proposal exposes these primitives for the portal and datumctl: the claim
state, the verification method, and the delegation status, plus the reason for
each.

## What Kubernetes operator decides what

Phase 1 can't be strictly one-way. The hold and the contest need ownership facts
inside hosting. The rule is instead that **neither operator reads a decision
that waits on its own output**.

| Component                             | Owns                                                                                                                        | Reads from the other                                                                                                               |
| ------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| dns-operator (hosting)                | Zones and serving. The **claim** on each name: first claim, hold, contest resolution. Serves the holder's zone immediately. | From `Domain`: the registry delegation (for the hold) and the TXT/HTTP result (for contests). Never delegation-based verification. |
| network-services-operator (ownership) | `Domain`: registry and delegation lookups, TXT/HTTP checks, and the verdict ALBs and certificates use                       | From `DNSZone`: whether the zone is the holder, and the nameserver set it publishes (the shared set in Phase 1)                    |

This can't deadlock:

- The hold and the contest depend only on the registry lookup and on TXT/HTTP,
  which never consult zones.
- Delegation-based verification depends on the zone being the holder, which
  never waits on verification.
- The verification gate, which made
  [network-services-operator#421](https://github.com/datum-cloud/network-services-operator/issues/421)
  a cycle, is removed.

## Changes by component

- **dns-operator:**
  - Remove the wait for `Domain` Verified before serving, behind a configuration
    flag.
  - Add the Held, Contested, and Displaced claim states.
  - Publish the serving DNSZoneClass's nameservers on each zone's status,
    replacing the project-side class that lists none.
  - Keep creating the `Domain` for each zone.
- **network-services-operator:**
  - The delegation check counts only the zone that holds the claim.
  - Read delegations from the parent's authoritative servers when RDAP isn't
    available.
  - Re-check delegation-based verification, with the 5-day grace period.
- **Portal and datumctl:** build on the new primitives. Design is out of scope
  here.

## Signals

- Time from zone creation to served-and-verified. This feeds the convergence
  SLOs under
  [enhancements#905](https://github.com/datum-cloud/enhancements/issues/905).
- The number of Held, Contested, and Displaced claims.
- The registry lookup error rate per TLD. This rate would have exposed the `.io`
  failure.

## Test plan

The current suites verify their own domains before asserting anything
([dns-operator#130](https://github.com/datum-cloud/dns-operator/issues/130),
[network-services-operator#427](https://github.com/datum-cloud/network-services-operator/issues/427)),
so none of these paths is exercised today. Each test asserts what Datum's
nameservers actually answer, not only what a condition says.

| Test                    | Must observe                                                                                                                            |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Delegate first          | The zone serves immediately. After delegation it's Verified with no TXT record.                                                         |
| Non-holder isn't served | A second project's zone for the same name: Datum answers from the holder only                                                           |
| Contest                 | The challenger takes the name over with a TXT record, and the displaced zone stops answering.                                           |
| Hold                    | A zone deleted while delegated stops answering. Another project's zone is Contested. The same project recreates it and it serves again. |
| Lapse                   | When the delegation leaves, verification lapses after the grace period (shortened in tests).                                            |
| Lookup failure          | A simulated registry failure doesn't lapse verification.                                                                                |

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

Enabling the flag serves every zone that was waiting for verification. Run the
pre-flight review first. Rolling back stops those zones from serving.

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

No. Claim state is added to existing status.
