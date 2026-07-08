---
status: provisional
stage: alpha
latest-milestone: "v0.1"
---

# Alerting Communication Standards

A shared standard for how Datum teams raise alerts and how we communicate
about alerts that fire — so that every page is actionable, owned, and
trusted.

This is a process enhancement, not a runtime feature. The template is
followed where it fits; the Production Readiness Questionnaire is replaced
by an "Adoption and Measurement" section, since the thing being rolled out
is a standard, not code.

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
- [Adoption and Measurement](#adoption-and-measurement)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)

## Summary

Alerting is a product with two kinds of users: the engineer who gets paged,
and the teammate who reads the record afterwards. In July 2026 the
production alert beat (datum-cloud/infra#2979) surfaced a consistent set of
defects in how we serve both users: every firing production rule lacked a
`team` label, several lacked runbooks, one rule paged `critical` for
self-recovering events, two rules paged on single transient errors, and one
rule family aggregated away the `cluster` label so lab noise was
indistinguishable from a production failure.

This enhancement establishes (1) a merge-time checklist for raising an
alert — required labels, impact-graded severity, flap-resistant thresholds,
routing-safe expressions — and (2) communication norms for alerts that
fire — a single monthly record, one root-cause ticket per failure class,
evidence-first reporting, watch-then-ticket for transients, and visible
corrections. The standards document lives in the infra repo
(`docs/telemetry/alerting-standards.md`); this enhancement covers its
rationale, scope, and how adoption is measured.

## Motivation

The cost of a bad alert is paid by people, not machines:

- **Unowned alerts stall triage.** With no `team` label, the first question
  in every incident is "whose is this?" — asked again by every responder.
- **Over-graded alerts erode trust.** `ContainerOOMKilledRecently` paged
  `critical` for a single self-recovering OOM (infra#3112). Responders who
  learn that `critical` sometimes means "nothing" respond slower to the
  `critical` that means everything.
- **Flappy rules train people to ignore the channel.** A `> 0` threshold
  paged twice in one afternoon for two isolated 5xx responses on an API
  serving ~200 requests per 5 minutes.
- **Missing runbooks strand the responder** at the exact moment they have
  the least context.
- **Poor communication multiplies the cost.** Without a shared record and
  ticketing discipline, the same alert gets re-investigated from scratch
  (the DLQ leak was re-diagnosed multiple times before its cause ticket
  became the durable reference), and premature "recovered" calls send
  responders away while the incident is still live.

July also showed the upside: when the sentry incident chain (infra#3115)
was tracked as one cause ticket with evidence-first updates and a visibly
flagged correction, six alert names and four fixes resolved in two days
without duplicated work.

### Goals

- Every production alert carries an owner (`team`), a `service`, an
  impact-graded `severity`, and a resolving `runbook_url` at merge time.
- Severity means user impact, uniformly: `critical` is page-worthy by
  definition, and responders can trust that.
- Alert thresholds are flap-resistant: no `> 0` error counters, `for:`
  durations that outlive known burst patterns, volume gates on low-traffic
  signals.
- Rule expressions preserve every label used for notification routing
  (notably `cluster`), so environment separation (prod vs lab) holds by
  construction.
- Firing alerts are communicated through one durable record per month, one
  root-cause ticket per failure class, with evidence, explicit
  severity-disagreement flags, and visible corrections.
- Rule defects are treated as fixable work (a hygiene punch list), not
  tribal knowledge.

### Non-Goals

- Choosing or changing alerting infrastructure (VictoriaMetrics, vmalert,
  Alertmanager stay as they are).
- Defining SLOs for individual services — this standardizes how alerts are
  *expressed and communicated*, not what each team's thresholds should be.
- On-call scheduling, paging escalation policies, or incident-command
  process for customer-facing outages.
- Retrofitting every existing rule at once (adoption is incremental; see
  [Adoption and Measurement](#adoption-and-measurement)).

## Proposal

Two artifacts:

1. **A rule-authoring standard** enforced as a review checklist (and, later,
   CI lint) on every new or changed alert rule: required
   labels/annotations, severity grading, threshold hygiene, routing-label
   preservation, cascade/inhibition expectations, environment separation.
2. **A communication standard** for firing alerts: the monthly "Alerts
   Seen" record, root-cause ticket discipline, evidence-first reporting,
   watch-then-ticket for transients, no recovery calls mid-rollout,
   corrections posted rather than hidden, and silences that always carry a
   ticket link.

Both are specified concretely in
[`docs/telemetry/alerting-standards.md`](https://github.com/datum-cloud/infra/blob/main/docs/telemetry/alerting-standards.md)
in the infra repo, with each standard traced to the July 2026 incident that
motivated it.

### User Stories

#### Story 1 — The paged engineer

I get paged at 02:00 for an alert on a service I don't own. The alert
carries `team`, `service`, a severity I can trust, and a `runbook_url` that
resolves. I follow the runbook; if it's not mine to fix, the `team` label
tells me who to hand it to. I am not the person who has to figure out
ownership at 02:00.

#### Story 2 — The service team raising a new alert

My team ships a new controller and adds alert rules. The review checklist
tells me exactly what a mergeable rule looks like — labels, severity
grading, threshold shape. My rule doesn't become someone else's hygiene
backlog six months later.

#### Story 3 — The teammate catching up

An alert fired last week and I'm picking up the follow-up. The monthly
Alerts Seen issue gives me the timeline, the evidence, and a link to the
one root-cause ticket. I don't re-investigate from scratch, and I can see
whether a "cleared" call was verified or later corrected.

#### Story 4 — The platform operator during a rollout

A release rolls through production. Expected churn (PDB dips, restart
blips) is inhibited or absorbed by `for:` headroom; the only pages I get
during the window are ones that would be real at any other time.

### Risks and Mitigations

- **Checklist fatigue / cargo-culting.** Teams may fill fields to pass
  review without grading honestly. Mitigation: severity definitions come
  with a concrete test ("would being woken at 03:00 for this annoy you?"),
  and the monthly beat flags over-grades as hygiene items with named
  examples.
- **Standards drift once the July pain fades.** Mitigation: each rule in
  the standards doc cites the incident that motivated it; the hygiene punch
  list keeps violations visible monthly; a CI lint (see below) makes the
  label requirements self-enforcing.
- **Process overhead discourages alerting at all.** Under-alerting is worse
  than noisy alerting. Mitigation: the checklist is deliberately small (one
  table plus three threshold rules), stub runbooks are acceptable at merge,
  and the standard explicitly says hygiene items are punch-list work, not
  merge blockers for existing rules.
- **The communication norms depend on someone running the beat.** The
  monthly record is currently maintained by an automated patrol plus one
  operator. Mitigation: the norms are written so any teammate can post a
  conforming entry; ownership of the beat itself is a staffing decision
  outside this enhancement's scope.

## Design Details

The full standard is in
[`docs/telemetry/alerting-standards.md`](https://github.com/datum-cloud/infra/blob/main/docs/telemetry/alerting-standards.md).
The normative core:

**Raising an alert — merge checklist.** Every rule carries `team`,
`service`, `severity`, `runbook_url`, and a `description` that includes
`{{ $labels.cluster }}`. Severity is graded on user impact: `critical` =
page now (core prod service at crash risk or user traffic broken),
`warning` = same-working-day attention, `info` = never routes to a pager.
Thresholds: no `> 0` on error counters (use ratio/SLO forms); `for:` must
outlive known burst patterns; low-volume histograms get volume gates. Rule
expressions must preserve every label the Alertmanager routing tree matches
on. Symptom cascades prefer inhibition rules over per-symptom paging. Lab
and edge-experiment alerts must be routable away from the prod channel.

**Communicating a firing alert.** One monthly tracking issue catalogues
prod fires (Uncleared / Cleared / Hygiene). One root-cause ticket per
failure class; recurring alerts reuse their ticket. Reports carry exact
evidence (log lines, queries, UTC timestamps). When rule severity and real
severity disagree, both are stated and a hygiene item is filed. Transients
get watch-then-ticket. Recovery is not declared mid-rollout; wrong calls
get visibly flagged corrections, not silent edits. Entries close only after
a re-fire-free period. Silences carry a comment, ticket link, and minimal
duration.

**Enforcement mechanics (phased).**

1. *Now*: standards doc merged; review checklist applied to alert-rule PRs
   in infra; hygiene punch list continues in the monthly issue.
2. *Next*: CI lint on `VMRule` resources in infra CI — fail a changed rule
   missing `team`/`severity`/`runbook_url`, warn on `> 0` counter
   thresholds and on expressions that aggregate away `cluster`.
3. *Later*: recording rules exposing hygiene metrics (counts of firing
   alerts missing labels/runbooks) so drift is observable, not audited by
   hand.

## Adoption and Measurement

Replaces the Production Readiness Questionnaire — the observable behavior
of a standard is whether people follow it and whether the pain it targets
recedes.

**How do we know it's in use?**

- Alert-rule PRs in infra reference the checklist (reviewer practice, then
  CI lint results).
- New monthly Alerts Seen issues follow the entry format (cause-ticket
  link, evidence, severity flags).

**Success indicators, measured monthly from the Alerts Seen issue:**

- Count of firing prod rules missing `team` / `runbook_url` — July 2026
  baseline: effectively **all** firing rules missing `team`; target: zero
  among rules touched since the standard merged, trending to zero overall.
- Count of severity over-grades flagged (`real severity` ≠ rule severity) —
  baseline: 2 in early July; target: none recurring on the same rule.
- Count of flap-class hygiene items (page-then-resolve singletons) —
  baseline: 3 rule families in July.
- Re-investigation events (same failure class diagnosed from scratch
  because the prior record wasn't found or linked) — baseline: the DLQ leak
  class; target: zero.

**Rollback.** Reverting is deleting the checklist requirement from review
practice and disabling the lint; already-labeled rules keep working. No
runtime risk.

## Implementation History

- 2026-06 → 2026-07 — Monthly alert beat (infra#2902, infra#2979) surfaces
  the defect classes and communication patterns this standardizes.
- 2026-07-06 — First hygiene fixes land ahead of the standard: rule
  re-grade + runbook (infra#3113), scrape-config fix (infra#3110).
- 2026-07-07 — Standards doc drafted
  (`docs/telemetry/alerting-standards.md` in infra); this enhancement
  opened as `provisional`.

## Drawbacks

- Adds review friction to alert-rule changes; small teams may feel the
  checklist is ceremony for a two-line rule.
- The communication norms encode one team's beat practice; other teams may
  have workflows (e.g. their own incident tooling) that the monthly-issue
  model doesn't fit.
- Standards documents rot unless the enforcement phases actually ship;
  phase 1 alone relies on reviewer memory.

## Alternatives

- **Tooling-only (lint everything, write nothing).** A lint can require
  labels but cannot grade severity honestly, judge threshold sanity for a
  novel signal, or teach anyone *why*. The July over-grades were
  semantically valid YAML.
- **Norms-only (write the doc, enforce nothing).** This is the status quo
  ante: severity guidelines already existed in `docs/telemetry/alerts.md`
  and were not followed. Without review-time and CI enforcement plus a
  monthly measurement loop, the doc joins the rot pile.
- **Adopt an external framework wholesale (e.g. Rob Ewaschuk's / Google
  SRE alerting philosophy, Alertmanager label conventions from
  kubernetes-mixin).** These informed the content, but the standard's
  authority with our teams comes from tracing every rule to a Datum
  incident with a ticket number. A generic framework invites "that doesn't
  apply here."
