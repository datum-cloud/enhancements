---
status: provisional
stage: alpha
latest-milestone: "v0.1"
---

# Datum Workflows: Event-Driven Automation for Milo

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)

## Summary

**Datum Workflows** introduces a Kubernetes-aligned, event-driven automation layer for **Milo**, enabling teams to describe internal workflows declaratively (definitions to be specified later) so that **“event comes → do something → repeat”** is the norm rather than the exception. The system covers light marketing automation, metric-driven policies (e.g., “when you are at 80% of XYZ”), testability, and **impact analysis**. A key requirement is a **first-class Testing & Simulation UX** in the Milo UI so users can **dry-run** a workflow (or proposed change) and **clearly understand its implications and blast radius before enabling**.

> This document intentionally **omits code and CRDs**; those will arrive in research stories.

## Motivation

Milo is a “business operating system” for product-led B2B companies: a control plane built on a reliable system of record (orgs, projects, contacts, accounts, usage, pricing, agreements, audit logs, etc.). Teams need consistent, auditable automation tightly bound to this data, without writing bespoke reconcilers for every use case. Existing general automation tools often compete for “system of record” and lack deep, internal integration. Datum Workflows focuses on internal, event-driven policies, with safety, testing, and a great operator UX.

### Goals

- Declarative, versioned workflow definitions (spec later).
- Event ingestion from Milo domain events and external sources.
- Action library (Milo API mutations, webhooks, notifications, tickets, lightweight jobs).
- **Testing & Simulation UX**:
  - **Dry run / Impact Preview** before enabling or modifying a workflow.
  - **Change impact analysis** (what changes if I tweak a filter/threshold/step?).
  - **Blast-radius estimates** (entities affected, actions triggered, rate limits).
  - **Replay/backtest** against historical events or sampled live traffic.
  - **Two-phase rollout** (shadow mode → canary → full enable).
- Safety: idempotency, rate limits, dedupe, timeouts, backoff.
- Multi-tenant isolation aligned with Milo tenants/RBAC.
- Optional drag-and-drop builder in the UI that compiles to declarative configs.
- Evaluate 1–2 Kubernetes workflow engines; pick 1 for alpha.

### Non-Goals

- Building a workflow engine from scratch.

## Proposal

Use an **engine-agnostic** design so workflow definitions can be rendered to one of a small set of Kubernetes workflow engines. Initial evaluation: **Argo-family** and **Tekton-family**. The Milo UI will provide:
- **Authoring & review** (form or YAML editor later).
- **Testing & Simulation** (see details below).
- **Observability** (per-workflow runs, success/failure, latency).

### User Stories (Optional)

#### Story 1 — Usage at 80%
As an operator, when an account reaches **80%** of storage quota, the system should:
- Record an “Upgrade Nudge” on the account timeline,
- Notify the account owner and assigned CSM,
- Open an operator ticket with recommended SKUs.

**In the UI**, I can dry-run this workflow over the last 7 days of events, see **how many accounts** would be affected, **which actions** would fire, and confirm **rate-limit** usage before enabling.

#### Story 2 — Agreement Renewal (T–30)
As Legal/Ops, when an MSA/NDA is **30 days** from expiry:
- Notify legal contacts,
- Attach the active agreement,
- Generate a Deal Room link.

**In the UI**, I can tweak the window (T–45 vs T–30) and preview which accounts would newly qualify and which would drop out.

### Notes/Constraints/Caveats (Optional)

- Keep definitions approachable; encourage composition over complex conditionals.
- PII & compliance: strict scoping, encryption, audit trails, allowlists for outbound calls.
- “Personal organization” templates ship as **optional policy packs** only.

### Risks and Mitigations

- Engine lock-in → intermediate model + adapters; conformance tests.
- Runaway automations → circuit breakers, concurrency caps, max fan-out, global kill switch.
- Observability gaps → standardized metrics/logs/traces; correlation IDs.
- Cost/regression risk → dry-run by default for new/changed workflows; canary rollouts.

## Design Details

**Concepts (no code):**
- **Events**: Milo domain events (orgs, projects, contacts, accounts, usage, agreements, entitlements, audit logs); external webhooks/bus.
- **Selectors & Filters**: event-type + attribute filters (thresholds, lists).
- **Actions**: Milo mutations, notifications, tickets, webhooks, lightweight jobs.
- **Safety**: idempotency keys; per-tenant limits; retries/backoff; timeouts.

**Testing & Simulation UX (highlight):**
- **Dry Run / Impact Preview Panel**
  - **Scope controls**: time window (e.g., last 24h/7d/30d), tenant(s), segment lists.
  - **Impact summary**: entities matched, actions that would fire, estimated request volume, expected run time, rate-limit utilization.
  - **Blast-radius map**: top accounts/contacts/vendors affected with drill-down links.
  - **Effect diff**: “Before vs After change” showing added/removed matches when editing a workflow.
  - **Constraint checks**: surfaces policy violations (e.g., outbound domain not on allowlist).
  - **Cost & quota hints**: rough call counts by destination provider and any potential throttling.
- **Replay & Backtest**
  - Run against historical events to validate outcomes; compare to ground truth where available.
  - **Shadow mode**: execute logic and record would-be actions without side effects.
- **Progressive Enablement**
  - **Canary enable** (e.g., 5% of matched entities or one segment),
  - **Auto-pause on error budget** breaches or anomaly spikes,
  - One-click **rollback to last working version**.
- **Auditability**
  - Every dry run and enablement captured with who/when/what-changed, linked to resulting runs.

**Engine Evaluation:**
- Eventing semantics, templating/parameters, retries/idempotency, artifacts/results, multi-tenancy, observability, ecosystem maturity, ops footprint, DX (local dev/validation).

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

- Mechanism: Feature gate in Milo plus values to deploy chosen engine components.
- Default behavior changes: None until a workflow is enabled.
- Disablement: Safe; existing runs drain or are GC’d; definitions remain versioned for later re-enable.
- Tests: E2E enable/disable; soak tests with bursty events; shadow-mode validations.

### Rollout, Upgrade and Rollback Planning

- Failure modes: engine/version skew, stuck runs, queue depth growth.
- Rollback metrics: spike in failure rate, latency regression (P95/P99), backlog size, error budget burn.
- Upgrade tests: dev → staging → prod; canary on representative workflows.
- Deprecations: None for alpha.

### Monitoring Requirements

- “In use” signal: per-workflow trigger count and active-run gauges.
- Health: controller liveness/readiness; event bus lag; action error rates.
- User verification: UI shows last trigger, last success/failure, recent actions, and simulation results.

### Dependencies

- Kubernetes cluster; one workflow engine (to be selected); event bus (initially NATS or equivalent); observability stack; secret management.

### Scalability

- Alpha targets: ~1k events/min/cluster; P95 < 5s for simple flows.
- Scale via horizontal workers, per-tenant concurrency & rate limits.

### Troubleshooting

- UI drill-down from failed run → event payload → action logs.
- Replay a specific event in shadow mode.
- Common issues: missing secrets, RBAC denies, provider throttles, misconfigured filters.

## Implementation History

- Phase 0: Research stories scoped; engine evaluation rubric; pick 1 engine for alpha.
- Phase 1 (Alpha): Intermediate model; adapters for chosen engine; **Testing & Simulation UX** MVP.
- Phase 2 (Beta): Policy packs; backtesting at scale; progressive delivery controls.
- Phase 3 (GA): Hardened defaults; SLOs/runbooks; migration & rollback tooling.

## Drawbacks

- Additional platform complexity and operational overhead.
- Learning curve for declarative workflows and simulation results.
- Engine upgrades and CR changes require careful rollout (mitigated by canaries).

## Alternatives

- Code-first orchestrators (Temporal/Cadence): strong DX but heavier runtime, less declarative.
- Managed state machines (AWS Step Functions): cloud-specific portability tradeoffs.
- SaaS automation (Zapier/Workato/Make): fast GTM but shallow internal/K8s integration.
- DIY reconcilers: high control; repeated effort per use case; limited reuse.

## Infrastructure Needed (Optional)

- Cluster capacity for the chosen engine and event bus.
- Observability (Prometheus/Grafana/OTel).
- Secret management and outbound allowlists.

