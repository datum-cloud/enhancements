---
status: provisional
stage: alpha
latest-milestone: "v0.0"
---

# Fraud and Abuse Prevention API

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [API Resources](#api-resources)
    - [FraudProvider](#fraudprovider)
    - [FraudPolicy](#fraudpolicy)
    - [FraudEvaluation](#fraudevaluation)
  - [Pipeline Execution Model](#pipeline-execution-model)
  - [Evaluation Triggers and Re-evaluation](#evaluation-triggers-and-re-evaluation)
  - [Provider Failure Handling](#provider-failure-handling)
  - [Input Data Model](#input-data-model)
  - [v1 Implementation Scope](#v1-implementation-scope)
  - [Milo Integration](#milo-integration)
  - [Zitadel Integration](#zitadel-integration)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)

## Summary

This enhancement introduces a Kubernetes-native fraud and abuse prevention API for the
Datum platform. The system provides a pluggable, layered provider architecture that
evaluates users for fraud risk throughout their lifecycle -- at registration, on
data changes, on a recurring schedule, or on demand -- and produces composite
fraud scores used to drive enforcement actions (manual review or automatic
deactivation).

The API is designed around three core resources: **FraudProvider** (configures individual
fraud detection backends), **FraudPolicy** (orchestrates providers into an ordered
evaluation pipeline with short-circuit logic), and **FraudEvaluation** (captures
per-user scoring results and enforcement decisions).

v1 targets 1-2 providers: a device fingerprinting / IP / email risk analysis service
(e.g. MaxMind minFraud) and a sanctions screening service (e.g. moov-io/watchman for
OFAC/CSL).

## Motivation

Cloud platforms that allow self-service sign-up are inherently exposed to fraud: stolen
credit cards, disputed charges, abuse of freemium tiers, and sanctioned-entity access.
Datum currently has no automated mechanism to evaluate user risk at registration time or
take enforcement action based on risk signals.

The problem has two parts:

1. **Determining whether a user is malicious** -- aggregating signals from multiple
   sources (IP reputation, email age, device fingerprint, sanctions lists) into an
   actionable risk assessment.
2. **Preventing the user from consuming resources** -- integrating with the existing
   user deactivation flow to isolate or terminate bad actors.

This system must be cost-conscious (cheap checks should gate expensive ones), resilient
to provider failures, and extensible so that new fraud signals can be added without
redesigning the core pipeline.

### Goals

- Define a Kubernetes-idiomatic API for configuring fraud detection providers and
  evaluation policies.
- Support a **pluggable provider model** where providers can be added, removed, or
  swapped without changes to the core fraud controller.
- Support **layered pipeline execution** with configurable short-circuit logic so that
  high-confidence cheap checks can prevent unnecessary expensive checks.
- Support **required stages** that always execute regardless of prior stage results
  (e.g. sanctions screening must always run).
- Accept **flexible input data** from multiple sources (registration form data, HTTP
  request metadata, device fingerprint payloads) through a canonical data model with
  per-provider field mapping.
- Handle **provider failures gracefully** with configurable failure policies per
  provider (fail-open, fail-closed, use-cached).
- Produce per-user **FraudEvaluation** resources that capture individual provider
  scores, an overall decision, and an audit trail.
- Integrate with the existing **UserDeactivation** resource for enforcement actions.
- Support an **observe mode** where the system scores users and emits events but takes
  no enforcement action (analogous to the WAF observe mode).
- Deliver a working v1 with MaxMind minFraud (or equivalent) and OFAC/sanctions
  screening.

### Non-Goals

- **Abuse detection** (outbound attack detection, IP reputation degradation monitoring,
  abuse@ email handling) -- this is a related but distinct problem that warrants its own
  enhancement. See [issue #505 discussion](https://github.com/datum-cloud/enhancements/issues/505).
- **Behavioral anomaly detection** (account takeover detection, activity pattern
  analysis) -- the system supports re-evaluation on data changes and schedules, but
  inferring fraud from behavioral patterns (e.g. unusual API call volumes, impossible
  travel) is a future phase that will build on the activity API
  ([#536](https://github.com/datum-cloud/enhancements/issues/536)).
- **Machine account fraud** -- machine accounts have different threat models and
  verification patterns. Will be addressed separately.
- **CAPTCHA or UI-layer bot prevention** -- while CAPTCHA is part of a defense-in-depth
  strategy, it is a frontend concern orthogonal to the backend fraud scoring API.
- **Building a full enterprise fraud dashboard** -- v1 focuses on API primitives and
  controller automation. Operator visibility comes through standard K8s resource
  inspection and events.
- **Replacing enterprise fraud platforms** (Sift, Castle.io) -- the system should
  complement or be replaced by these when needed, not compete with them.

## Proposal

The fraud prevention system is modeled as a **pipeline of providers** orchestrated by a
**policy resource**. When a user event triggers evaluation (initially: account
creation), the fraud controller reads the active FraudPolicy, executes providers in
stage order according to the policy's pipeline configuration, and writes results to a
FraudEvaluation resource for the user.

The pipeline supports two execution modes that can be mixed within a single policy:

1. **Layered (short-circuit)**: Stages execute in order. If a stage's result crosses a
   configured score threshold, subsequent stages in the same mode can be skipped. This
   optimizes cost by running cheap checks first.
2. **Required**: Certain stages are marked as always-execute regardless of prior results.
   These are used for compliance-mandatory checks like sanctions screening.

Enforcement is decoupled from scoring. The FraudPolicy specifies threshold-to-action
mappings and an enforcement mode (AUTO or OBSERVE). In OBSERVE mode, the system records
scores and emits events but creates no UserDeactivation resources. In AUTO mode, it can
create UserDeactivation resources when thresholds are met.

### User Stories

#### Platform operator enables fraud detection with MaxMind

A platform operator creates a FraudProvider resource pointing to MaxMind minFraud with
their API credentials stored in a Secret. They create a FraudPolicy with a single stage
that runs MaxMind on every new user registration. They set the enforcement mode to
OBSERVE to monitor scoring before enabling automatic enforcement.

#### Platform operator adds sanctions screening

The operator deploys watchman (or uses a hosted instance) and creates a second
FraudProvider. They update their FraudPolicy to add a second stage for OFAC screening
marked as `required: true`. Even if MaxMind returns a low risk score, the OFAC check
always runs.

#### Cost-optimized multi-provider pipeline

The operator configures a 3-stage pipeline: Stage 1 runs a cheap email-age check. If
the email is older than 1 year and has no risk signals, stage 2 (MaxMind, $0.02/query)
is skipped. Stage 3 (OFAC) always runs because it is marked required. This reduces
per-user cost for obviously legitimate sign-ups while maintaining compliance.

#### Scheduled re-evaluation catches sanctions list updates

OFAC publishes a new SDN list entry. The operator has configured a weekly scheduled
trigger on the FraudPolicy. On the next scheduled run, the fraud controller re-evaluates
all users against watchman with the updated list. A previously-clean user now matches
the new entry, their FraudEvaluation is updated from NONE to REVIEW, and the operator
is alerted to investigate.

#### User changes email and triggers re-evaluation

A user who passed initial fraud checks later changes their email address to a newly
created domain. The FraudPolicy has a data-change trigger on `emailAddress`. The fraud
controller detects the change, re-runs the pipeline with the new email, and MaxMind
returns a higher risk score. The FraudEvaluation is updated, the score delta is
recorded in history, and a `FraudThresholdExceeded` event is emitted.

#### Provider outage does not block sign-ups

MaxMind experiences an outage. The FraudProvider for MaxMind is configured with
`onFailure: FailOpen`. The fraud controller marks the MaxMind result as `UNAVAILABLE`
in the FraudEvaluation, emits a warning event, and continues pipeline execution. The
user is not blocked, but the evaluation is flagged for manual review.

### Notes/Constraints/Caveats

- Fraud scoring is inherently probabilistic. The system produces risk signals, not
  definitive fraud determinations. Human review should remain part of the workflow,
  especially in early deployments.
- Score normalization across providers is non-trivial. Each provider uses different
  scoring scales. The FraudPolicy defines thresholds per-provider rather than
  attempting to normalize to a universal scale.
- While the initial high-priority trigger is user registration, the system is
  designed to support re-evaluation over time. See the
  [Evaluation Triggers and Re-evaluation](#evaluation-triggers-and-re-evaluation)
  section for details.
- Provider API rate limits and costs should be considered when configuring pipelines.
  The layered execution model is specifically designed to address this.

### Risks and Mitigations

**False positives blocking legitimate users**
- Mitigation: OBSERVE mode for initial deployment. Deactivation is never deletion --
  accounts can be reactivated after review. FraudEvaluation resources provide full
  audit trail for appeals.

**Provider API credentials leaking**
- Mitigation: Credentials are stored in Kubernetes Secrets and referenced by name in
  FraudProvider resources. The fraud controller reads secrets at runtime; they are
  never stored in FraudProvider status or FraudEvaluation results.

**Single provider failure cascading to all sign-ups**
- Mitigation: Per-provider `onFailure` policy. FailOpen is the recommended default for
  non-compliance providers. Circuit breaker pattern prevents repeated calls to
  unhealthy providers.

**Sanctions screening data staleness**
- Mitigation: For self-hosted watchman, the controller should reconcile list freshness
  and emit warnings if lists are older than a configurable threshold.

## Design Details

### API Resources

#### FraudProvider

A FraudProvider configures a single fraud detection backend. It is a cluster-scoped
resource because providers are infrastructure-level configuration shared across the
platform.

```yaml
apiVersion: fraud.miloapis.com/v1alpha1
kind: FraudProvider
metadata:
  name: maxmind
spec:
  # The type determines which built-in adapter the controller uses.
  # "webhook" enables arbitrary external providers via HTTP.
  type: maxmind  # maxmind | watchman | webhook

  # Connection configuration
  endpoint: "https://minfraud.maxmind.com/minfraud/v2.0/score"
  authSecretRef:
    name: maxmind-credentials
    namespace: fraud-system

  # How long to wait before considering a provider call failed
  timeout: 5s

  # What to do when this provider fails (timeout, 5xx, network error)
  #   FailOpen   - skip this provider, mark result as UNAVAILABLE, continue pipeline
  #   FailClosed - halt pipeline, mark evaluation as ERROR, flag for review
  #   UseCached  - use the most recent successful result for this user if available
  onFailure: FailOpen

  # Circuit breaker: after this many consecutive failures, stop calling the
  # provider and treat all requests as the onFailure policy dictates.
  # The controller will periodically probe the provider to detect recovery.
  circuitBreaker:
    failureThreshold: 5
    probeInterval: 60s

  # Input mapping: defines which fields from the canonical FraudInput data model
  # this provider requires. The controller extracts these fields and formats them
  # for the provider's API.
  inputMapping:
    # Each key is a canonical field name, value is the provider-specific field path
    fields:
      ipAddress: "device.ip_address"
      emailAddress: "email.address"
      emailDomain: "email.domain"
      userAgent: "device.user_agent"
      acceptLanguage: "device.accept_language"

status:
  # Controller-managed status
  conditions:
    - type: Ready
      status: "True"
      lastTransitionTime: "2026-03-10T12:00:00Z"
    - type: CircuitOpen
      status: "False"
      lastTransitionTime: "2026-03-10T12:00:00Z"
  lastSuccessfulCheck: "2026-03-10T15:30:00Z"
  consecutiveFailures: 0
```

For **webhook** type providers, the endpoint receives a POST with the mapped input
fields and must return a JSON response conforming to a standard schema:

```json
{
  "score": 0.0,
  "reasons": [],
  "metadata": {}
}
```

Score must be a float between 0.0 (no risk) and 100.0 (maximum risk). The fraud
controller normalizes provider-native scores to this range using the built-in adapter
for known provider types. `reasons` is an array of provider-specific strings explaining
what contributed to the score (e.g. `"ip_proxy_detected"`, `"email_domain_new"`). The
`metadata` object is freeform and is what downstream providers reference via `inputFrom`
mappings -- this is how enrichment providers pass structured data (e.g. email age, geo
coordinates, risk factors) to scoring providers later in the DAG.

Risk level labels (LOW, MEDIUM, HIGH) are intentionally **not** part of the provider
response contract. Each provider has different scoring semantics and ranges; expecting
them to agree on categorical labels would be either lossy or require brittle
normalization. Instead, the FraudPolicy thresholds define what score values mean
operationally (REVIEW at 70, DEACTIVATE at 90, etc.).

#### FraudPolicy

A FraudPolicy defines the evaluation pipeline: which providers to run, in what order,
with what thresholds, and what enforcement actions to take. It is a cluster-scoped
singleton (one active policy at a time, selected by a well-known name or label).

```yaml
apiVersion: fraud.miloapis.com/v1alpha1
kind: FraudPolicy
metadata:
  name: default
spec:
  # Namespace where fraud input ConfigMaps are created and read from.
  inputNamespace: fraud-system

  # Pipeline stages execute in order. Each stage references one or more providers.
  # Within a stage, providers form a DAG: providers with no dependencies run
  # concurrently, providers with dependsOn wait for their dependencies to complete
  # and can consume their output via inputFrom.
  stages:
    # Stage 1: Cheap IP/email/device risk check
    - name: risk-analysis
      providers:
        - name: maxmind-check
          providerRef:
            name: maxmind
          # Weight determines this provider's influence when a stage has multiple
          # providers. For single-provider stages, weight is ignored.
          weight: 100

      # Thresholds define actions triggered by provider scores.
      # Evaluated per-provider, not on aggregate stage scores.
      thresholds:
        - minScore: 70
          action: REVIEW
        - minScore: 90
          action: DEACTIVATE

      # Short-circuit: if the maximum provider score in this stage is below this
      # value, skip all subsequent non-required stages.
      # Omit to never short-circuit from this stage.
      shortCircuit:
        skipWhenBelow: 30

      # Required stages always execute regardless of prior short-circuit decisions.
      required: false

    # Stage 2: Sanctions/watchlist screening (always runs)
    - name: sanctions-screening
      providers:
        - name: watchman-check
          providerRef:
            name: watchman
          weight: 100
      thresholds:
        - minScore: 80
          action: REVIEW
        - minScore: 95
          action: DEACTIVATE
      required: true

  # Enforcement configuration
  enforcement:
    # OBSERVE - score and emit events only, no automatic enforcement
    # AUTO    - create UserDeactivation when decision is DEACTIVATE
    mode: OBSERVE

    # Template for UserDeactivation resources created in AUTO mode
    onDeactivate:
      reason: "Fraud policy threshold exceeded"
      description: >-
        Deactivated by fraud controller. Provider {{ .Provider }} reported risk
        score {{ .Score }} (threshold: {{ .Threshold }}).

  # Trigger configuration: when to create or re-run fraud evaluations.
  # Multiple triggers can be active simultaneously.
  triggers:
    # Event-based triggers: fire when a specific event occurs
    - type: Event
      event: UserCreated

    # Scheduled re-evaluation: periodically re-score existing users
    - type: Scheduled
      schedule:
        interval: 168h  # weekly
        # Optional: only re-evaluate users whose last evaluation is older than this
        staleAfter: 168h

    # Data-change triggers: re-evaluate when upstream data changes
    - type: DataChange
      dataChange:
        # Which data sources to watch for changes
        sources:
          - emailAddress   # e.g. user changes their email
          - ipAddress      # e.g. login from new IP (future)

    # Future trigger types:
    # - type: Event
    #   event: UserLogin
    # - type: Event
    #   event: PaymentSubmitted
    # - type: Event
    #   event: ResourceCreated

  # How many historical evaluation summaries to retain per FraudEvaluation
  historyRetention:
    maxEntries: 50
    maxAge: 8760h  # 1 year

status:
  conditions:
    - type: Ready
      status: "True"
      lastTransitionTime: "2026-03-10T12:00:00Z"
  activeProviders: 2
  lastEvaluationTime: "2026-03-11T09:15:00Z"
```

**Pipeline execution semantics:**

1. Stages execute in definition order.
2. Within a stage, providers form a **DAG** based on `dependsOn` declarations.
   Providers with no dependencies start concurrently. Providers with dependencies
   wait for all named dependencies to complete before executing.
3. When a provider declares `inputFrom`, the referenced dependency's output fields
   are merged into that provider's input before execution. This allows enrichment
   chains (e.g. an email-age lookup feeds its result into a risk scoring provider).
4. After all providers in a stage complete, the controller evaluates short-circuit
   conditions.
5. If `shortCircuit.skipWhenBelow` is set and all provider scores in the stage are
   below that value, all subsequent stages where `required: false` are skipped.
6. Stages with `required: true` always execute regardless of short-circuit state.
7. The overall decision is the **highest severity action** triggered across all
   providers in all executed stages (NONE < REVIEW < DEACTIVATE).

**DAG example -- enrichment chain within a stage:**

```yaml
- name: enriched-risk-analysis
  providers:
    # Step 1: Email age lookup (cheap, fast)
    - name: email-age
      providerRef:
        name: email-verifier
      weight: 30

    # Step 2: IP geolocation enrichment (runs concurrently with email-age)
    - name: ip-geo
      providerRef:
        name: ip-enricher
      weight: 0  # enrichment-only, does not contribute to stage score

    # Step 3: Full risk scoring -- depends on both, uses their output
    - name: risk-score
      providerRef:
        name: maxmind
      weight: 70
      dependsOn:
        - email-age
        - ip-geo
      inputFrom:
        email-age:
          # Map output fields from the email-age provider into maxmind's input
          emailAge: "metadata.account_age_days"
          emailValid: "metadata.deliverable"
        ip-geo:
          geoCountry: "metadata.country_code"
          geoRegion: "metadata.region"
```

In this example, `email-age` and `ip-geo` run concurrently (no dependencies).
`risk-score` waits for both, then receives their output fields merged into its
input alongside the canonical FraudInput fields. If `email-age` or `ip-geo`
fail, the failure policy on their respective FraudProviders applies, and
`risk-score` proceeds with whatever data is available.

Providers with no `dependsOn` and no `inputFrom` behave exactly as if all
providers in the stage run concurrently -- the DAG model is a superset of
the simple concurrent model, not a replacement.

#### Fraud Input ConfigMap

Input data for fraud evaluation is stored in a ConfigMap rather than directly on the
FraudEvaluation spec. This keeps the data schema flexible -- any key-value pair can
be added without CRD changes -- and separates collected data from evaluation results.

The namespace for input ConfigMaps is configured on the FraudPolicy:

```yaml
spec:
  inputNamespace: fraud-system
```

The actions-server (or fraud controller on re-evaluation) creates the ConfigMap in
this namespace. The FraudEvaluation references it via `spec.inputRef`.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: user-12345-fraud-input
  namespace: fraud-system
  ownerReferences:
    - apiVersion: fraud.miloapis.com/v1alpha1
      kind: FraudEvaluation
      name: user-12345
data:
  ipAddress: "203.0.113.42"
  emailAddress: "jane@example.com"
  emailDomain: "example.com"
  firstName: "Jane"
  lastName: "Doe"
  userAgent: "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)"
  acceptLanguage: "en-US,en;q=0.9"
  # Arbitrary additional fields can be added without schema changes
  # deviceFingerprint: "mxm_abc123"
```

Built-in provider adapters know which ConfigMap keys they need (e.g. MaxMind reads
`ipAddress`, `emailAddress`, `emailDomain`; watchman reads `firstName`, `lastName`).
Unknown keys are ignored. This means new data sources can be added to the ConfigMap
and consumed by future providers without changing the fraud controller or CRDs.

#### FraudEvaluation

A FraudEvaluation is a **living resource** created per-user. Rather than being a
one-shot record, it is updated in place each time the pipeline runs for that user --
whether triggered by initial registration, a scheduled re-evaluation, or a data
change. The resource always reflects the **most recent** evaluation results while
retaining a history of prior evaluations for audit purposes.

This design means a single FraudEvaluation per user tracks their fraud posture over
time. A user who was clean at registration but later changes their email to a
high-risk domain will have their evaluation updated to reflect the new score.

FraudEvaluation is **cluster-scoped** to match User, which is also cluster-scoped
in Milo. At registration time the user is not yet associated with an organization
namespace, so namespace-scoping would create a "where do we put this?" problem.
Cluster-scoping keeps a clean 1:1 relationship between User and FraudEvaluation.

```yaml
apiVersion: fraud.miloapis.com/v1alpha1
kind: FraudEvaluation
metadata:
  name: user-12345
  labels:
    fraud.miloapis.com/policy: default
    fraud.miloapis.com/decision: REVIEW
spec:
  # Immutable reference to the user being evaluated
  userRef:
    name: user-12345

  # Snapshot of the policy version used for the most recent evaluation
  policyRef:
    name: default
    resourceVersion: "12345"

  # Reference to a ConfigMap containing the fraud input data.
  # The ConfigMap holds arbitrary key-value pairs that providers consume.
  # This keeps input data flexible (no fixed schema) and separates it
  # from the evaluation resource itself.
  inputRef:
    name: user-12345-fraud-input
    namespace: fraud-system

status:
  # Evaluation phase: Pending, Running, Completed, Error
  phase: Completed

  # What triggered this evaluation run
  trigger:
    type: Event        # Event | Scheduled | DataChange | Manual
    event: UserCreated # populated for Event triggers
    # dataChange:      # populated for DataChange triggers
    #   field: emailAddress
    #   oldValue: "user@legit.com"
    #   newValue: "user@suspicious.xyz"

  # How many times this user has been evaluated
  evaluationCount: 1

  # Per-provider results in stage execution order (most recent run)
  stageResults:
    - stage: risk-analysis
      providers:
        - provider: maxmind
          score: 42.0
          reasons:
            - "email_domain_new"
            - "ip_proxy_detected"
          observedAt: "2026-03-11T09:15:01Z"
          duration: 230ms
          status: SUCCESS  # SUCCESS | UNAVAILABLE | ERROR

    - stage: sanctions-screening
      providers:
        - provider: watchman
          score: 0.0
          reasons: []
          observedAt: "2026-03-11T09:15:01Z"
          duration: 45ms
          status: SUCCESS

  # Stages that were skipped due to short-circuit
  skippedStages: []

  # Overall decision computed from all provider results
  decision:
    action: NONE
    # If action is REVIEW or DEACTIVATE, these fields identify the trigger
    triggerProvider: ""
    triggerScore: 0
    triggerThreshold: 0
    decidedAt: "2026-03-11T09:15:02Z"

  enforcement:
    # NONE     - no enforcement action taken
    # OBSERVED - thresholds were met but policy is in OBSERVE mode
    # REVIEWED - flagged for review (operator has not yet acted)
    # DEACTIVATED - UserDeactivation resource was created
    state: NONE
    userDeactivationRef: null

  # Total pipeline execution time for the most recent run
  totalDuration: 280ms

  # Score trend: tracks how the user's risk posture changes over time.
  # Each entry is appended after a completed evaluation run.
  # Older entries are pruned based on a configurable retention policy.
  history:
    - evaluatedAt: "2026-03-11T09:15:02Z"
      trigger:
        type: Event
        event: UserCreated
      decision: NONE
      highestScore: 42.0
      highestScoreProvider: maxmind
```

**FraudEvaluation with a REVIEW decision (MaxMind flags the user):**

```yaml
status:
  phase: Completed
  stageResults:
    - stage: risk-analysis
      providers:
        - provider: maxmind
          score: 78.0
          reasons:
            - "ip_risk_high"
            - "email_first_seen_recent"
            - "device_new"
          observedAt: "2026-03-11T10:00:01Z"
          duration: 310ms
          status: SUCCESS
    - stage: sanctions-screening
      providers:
        - provider: watchman
          score: 0.0
          reasons: []
          observedAt: "2026-03-11T10:00:01Z"
          duration: 50ms
          status: SUCCESS
  skippedStages: []
  decision:
    action: REVIEW
    triggerProvider: maxmind
    triggerScore: 78
    triggerThreshold: 70
    decidedAt: "2026-03-11T10:00:02Z"
  enforcement:
    state: OBSERVED  # Policy is in OBSERVE mode
```

**FraudEvaluation with a provider failure (MaxMind down, FailOpen):**

```yaml
status:
  phase: Completed
  stageResults:
    - stage: risk-analysis
      providers:
        - provider: maxmind
          score: 0
          reasons:
            - "provider_unavailable"
          observedAt: "2026-03-11T11:00:01Z"
          duration: 5000ms
          status: UNAVAILABLE
          error: "connection timeout after 5s"
    - stage: sanctions-screening
      providers:
        - provider: watchman
          score: 0.0
          reasons: []
          observedAt: "2026-03-11T11:00:06Z"
          duration: 40ms
          status: SUCCESS
  skippedStages: []
  decision:
    action: REVIEW  # Escalated to REVIEW because a provider was unavailable
    triggerProvider: maxmind
    triggerScore: 0
    triggerThreshold: 0
    decidedAt: "2026-03-11T11:00:07Z"
  enforcement:
    state: NONE
  conditions:
    - type: ProviderDegraded
      status: "True"
      reason: "MaxMindUnavailable"
      message: "Provider maxmind failed with FailOpen policy; evaluation is incomplete"
```

### Pipeline Execution Model

The fraud controller implements the pipeline as a reconciliation loop. The same
pipeline runs regardless of what triggered the evaluation -- initial registration,
scheduled re-evaluation, or data change. The only difference is the input data
snapshot and the trigger metadata recorded on the FraudEvaluation.

```
Trigger (Event / Schedule / DataChange / Manual)
    │
    ▼
Load active FraudPolicy
    │
    ▼
Create or update FraudEvaluation (phase: Pending)
    │
    ▼
Read input ConfigMap (via spec.inputRef)
    │
    ▼
┌─── For each stage in order ───────────────────────────┐
│                                                        │
│   Is this stage skipped by short-circuit?              │
│   ├─ YES and stage.required == false → skip            │
│   └─ NO or stage.required == true → execute            │
│                                                        │
│   Resolve provider DAG within stage                    │
│   ├─ Providers with no dependsOn → start concurrently  │
│   ├─ Providers with dependsOn → wait for dependencies  │
│   │   └─ Merge inputFrom outputs into provider input   │
│   ├─ Provider succeeds → record score + output         │
│   └─ Provider fails → apply onFailure policy           │
│       ├─ FailOpen → mark UNAVAILABLE, continue DAG     │
│       ├─ FailClosed → mark ERROR, halt pipeline        │
│       └─ UseCached → use last good result if available │
│                                                        │
│   Evaluate short-circuit conditions                    │
│   └─ If all scores < skipWhenBelow → set skip flag     │
│                                                        │
└────────────────────────────────────────────────────────┘
    │
    ▼
Compute overall decision (highest severity across all providers)
    │
    ▼
Apply enforcement policy
    ├─ OBSERVE → record decision, emit event
    └─ AUTO    → record decision, create UserDeactivation if DEACTIVATE
    │
    ▼
Append to FraudEvaluation history, update status (phase: Completed)
```

### Evaluation Triggers and Re-evaluation

Fraud is not a point-in-time determination. A user who appears clean at registration
may later exhibit risk signals (new IP ranges, email changes, billing country changes,
sanctions list updates). The system supports multiple trigger types so that evaluations
can run at the right moments throughout a user's lifecycle.

#### Trigger Types

**Event triggers** fire in response to discrete lifecycle events:

| Event | When it fires | Available in |
|---|---|---|
| `UserCreated` | New user registration | v1 |
| `UserLogin` | User authenticates | Future |
| `PaymentSubmitted` | Billing event occurs | Future |
| `ResourceCreated` | User provisions infrastructure | Future |

Event triggers create or re-evaluate the FraudEvaluation for the user associated with
the event. The FraudInput is populated with the latest data available at the time of
the event (e.g. the IP from the login, not the original registration IP).

**Scheduled triggers** run the pipeline on a recurring interval:

```yaml
- type: Scheduled
  schedule:
    interval: 168h    # how often to run
    staleAfter: 168h  # only re-evaluate if last evaluation is older than this
```

The controller maintains a work queue of users whose FraudEvaluation is stale. This is
processed at a configurable concurrency to avoid overwhelming providers. Scheduled
triggers are especially useful for sanctions screening: if OFAC publishes a new SDN
list, the next scheduled run will re-screen all users against the updated data without
requiring any user-initiated event.

**Data-change triggers** fire when specific fields on a user or related resources change:

```yaml
- type: DataChange
  dataChange:
    sources:
      - emailAddress
      - billingCountry
```

The fraud controller watches for updates to User resources (and future resources like
billing profiles). When a watched field changes, the controller re-evaluates the user.
The FraudEvaluation's `trigger.dataChange` field records what changed, providing an
audit trail of why the re-evaluation happened.

**Manual triggers** allow operators to force a re-evaluation by annotating the
FraudEvaluation:

```bash
kubectl annotate fraudevaluation user-12345 \
  fraud.miloapis.com/reevaluate=true
```

The controller detects this annotation, runs the pipeline, and removes the annotation
when complete. This is useful for re-checking a user after manual review, or after a
provider outage is resolved and operators want to fill in the gaps.

#### Re-evaluation Semantics

When a FraudEvaluation already exists for a user:

1. The controller updates the existing resource rather than creating a new one.
2. The current `stageResults` and `decision` are replaced with the new run's results.
3. A summary of the previous evaluation is appended to the `history` array.
4. The `evaluationCount` is incremented.
5. Labels (`fraud.miloapis.com/decision`) are updated to reflect the new decision.
6. If the new decision is **more severe** than the previous one (e.g. NONE → REVIEW,
   or REVIEW → DEACTIVATE), the enforcement policy is re-applied.
7. If the new decision is **less severe** (e.g. a previously flagged user now scores
   clean), the controller emits a `FraudScoreImproved` event but does **not**
   automatically reverse prior enforcement actions. Reversing a deactivation requires
   explicit operator action to avoid prematurely reinstating a bad actor.

#### History Retention

The `history` array on FraudEvaluation is bounded by a configurable retention policy
on the FraudPolicy:

```yaml
spec:
  historyRetention:
    maxEntries: 50       # keep at most this many history entries
    maxAge: 8760h        # prune entries older than 1 year
```

When either limit is exceeded, the oldest entries are pruned. The full evaluation
detail for each historical run is not retained in the history (only the summary);
detailed per-provider results are available in the current `stageResults` and in
the activity/audit log if the activity API integration is enabled.

### Provider Failure Handling

Provider resilience is a first-class concern. Each FraudProvider configures:

- **Timeout**: Maximum time to wait for a provider response. Default: 5s.
- **onFailure policy**:
  - `FailOpen` -- treat as if the provider returned score 0, mark UNAVAILABLE, continue
    the pipeline. The FraudEvaluation is flagged with a `ProviderDegraded` condition.
    Recommended default for non-compliance checks.
  - `FailClosed` -- halt the pipeline, mark the evaluation as ERROR, flag for manual
    review. Use for compliance-critical providers where a missing check is worse than
    blocking the user.
  - `UseCached` -- if the user has a prior successful evaluation, reuse the last known
    score from that provider. Otherwise, fall back to FailOpen behavior.
- **Circuit breaker**: After N consecutive failures, the controller stops calling the
  provider entirely and treats all requests according to the onFailure policy. A
  background probe runs at `probeInterval` to detect recovery. When the probe succeeds,
  the circuit closes and normal operation resumes. The `CircuitOpen` condition on the
  FraudProvider status provides visibility into provider health.

### Input Data Model

Input data is stored as key-value pairs in a ConfigMap referenced by the
FraudEvaluation. The keys below are conventions used by the built-in provider
adapters. Since the ConfigMap is freeform, additional keys can be added for
custom providers without schema changes.

**Well-known ConfigMap keys:**

| Field | Source | Description |
|---|---|---|
| `ipAddress` | HTTP request | User's IP address at registration |
| `emailAddress` | Registration form | User's email address |
| `emailDomain` | Derived | Domain portion of email address |
| `firstName` | Registration form | User's first name (for sanctions screening) |
| `lastName` | Registration form | User's last name (for sanctions screening) |
| `userAgent` | HTTP request | Browser/client user agent string |
| `acceptLanguage` | HTTP request | Accept-Language header |
| `deviceFingerprint` | Client SDK | Opaque device fingerprint payload (future) |

For **built-in provider types** (maxmind, watchman), the controller's adapter reads
the keys it needs from the ConfigMap and constructs the provider-specific API request.
Unknown keys are ignored. For future **webhook** providers, the controller would send
the ConfigMap data (or a configured subset) as a JSON body.

### v1 Implementation Scope

v1 delivers the minimum viable fraud prevention pipeline with a narrow, concrete scope.

**Resources:**
- FraudProvider (built-in adapters for `maxmind` and `watchman` only)
- FraudPolicy (single active policy, stages with short-circuit, OBSERVE/AUTO modes)
- FraudEvaluation (per-user, living resource with history)

**Providers and what they consume:**

MaxMind minFraud Score plan ($0.02/query):
- **Input**: `ipAddress`, `emailAddress`, `emailDomain`, `userAgent` (optional)
- **Output**: `risk_score` float (0.01-99), mapped to 0-100 scale by the adapter
- **What it tells us**: IP reputation, email newness/disposability, proxy/VPN detection
- **Failure policy**: FailOpen (don't block sign-ups if MaxMind is down)

moov-io/watchman (self-hosted, free):
- **Input**: `firstName`, `lastName` (combined into a name search query)
- **Output**: list of potential matches with match score (0.0-1.0); adapter takes
  the highest match score and maps it to 0-100 scale
- **What it tells us**: whether the user's name matches OFAC SDN, CSL, or other
  sanctions lists
- **Failure policy**: FailClosed (sanctions screening is a compliance requirement;
  if watchman is down, flag for manual review rather than skipping)

**Input fields populated at registration (v1):**

| Field | Populated | Source |
|---|---|---|
| `ipAddress` | Yes | HTTP X-Forwarded-For / RemoteAddr |
| `emailAddress` | Yes | Zitadel event payload |
| `emailDomain` | Yes | Derived from emailAddress |
| `firstName` | Yes | Zitadel event payload |
| `lastName` | Yes | Zitadel event payload |
| `userAgent` | Yes | HTTP User-Agent header |
| `acceptLanguage` | Yes | HTTP Accept-Language header |
| `deviceFingerprint` | No | Requires frontend JS tag (future) |

**Default v1 policy:**

```yaml
apiVersion: fraud.miloapis.com/v1alpha1
kind: FraudPolicy
metadata:
  name: default
spec:
  inputNamespace: fraud-system

  stages:
    - name: risk-analysis
      providers:
        - providerRef:
            name: maxmind
      thresholds:
        - minScore: 70
          action: REVIEW
        - minScore: 90
          action: DEACTIVATE
      required: false
      shortCircuit:
        skipWhenBelow: 20

    - name: sanctions-screening
      providers:
        - providerRef:
            name: watchman
      thresholds:
        - minScore: 80
          action: REVIEW
        - minScore: 95
          action: DEACTIVATE
      required: true

  enforcement:
    mode: OBSERVE

  triggers:
    - type: Event
      event: UserCreated

  historyRetention:
    maxEntries: 50
```

**Triggers in v1:**
- `UserCreated` event trigger (primary)
- Manual re-evaluation via annotation on FraudEvaluation

**Not in v1 (future enhancements):**
- Webhook provider type and `inputMapping` customization
- Intra-stage DAG execution (`dependsOn`, `inputFrom`, `weight`)
- Data-change triggers (watching User field diffs)
- Scheduled re-evaluation (requires work queue, rate limiting, scale design)
- Event triggers beyond UserCreated (UserLogin, PaymentSubmitted, etc.)
- `UseCached` failure policy
- Circuit breaker with background health probing
- Multi-policy support
- `maxAge`-based history pruning (v1 uses `maxEntries` only)

The threshold values above (70/90 for MaxMind, 80/95 for watchman) are starting
points. These should be tuned during OBSERVE mode based on real scoring data before
enabling AUTO enforcement.

### Milo Integration

The fraud prevention system is implemented as a new `fraud.miloapis.com` API group
within Milo, following the same conventions as existing API groups (`iam.miloapis.com`,
`notification.miloapis.com`, etc.).

**Package layout:**

```
pkg/apis/fraud/v1alpha1/
├── doc.go                        # +groupName=fraud.miloapis.com
├── register.go                   # SchemeGroupVersion, addKnownTypes()
├── types.go                      # Shared types (references, enums)
├── fraudprovider_types.go        # FraudProvider resource
├── fraudpolicy_types.go          # FraudPolicy resource
└── fraudevaluation_types.go      # FraudEvaluation resource

internal/controllers/fraud/
├── fraudevaluation_controller.go # Main pipeline execution controller
└── fraudprovider_controller.go   # Provider health monitoring

internal/webhooks/fraud/v1alpha1/
├── fraudprovider_webhook.go      # Validate provider config, secret refs
├── fraudpolicy_webhook.go        # Validate provider refs, threshold logic
└── fraudevaluation_webhook.go    # Immutable spec validation
```

**Controller integration with User lifecycle:**

The FraudEvaluation controller reconciles FraudEvaluation resources. For
registration, the actions-server creates the FraudEvaluation. For re-evaluations,
the controller watches User resources for data changes and manages the scheduled
re-evaluation queue:

```go
ctrl.NewControllerManagedBy(mgr).
    For(&fraudv1alpha1.FraudEvaluation{}).
    Watches(&iamv1alpha1.User{},
        handler.EnqueueRequestsFromMapFunc(r.mapUserToEvaluation)).
    Complete(r)
```

When a FraudEvaluation is created or queued for re-evaluation, the controller:

1. Loads the active FraudPolicy.
2. Reads the input ConfigMap referenced by `spec.inputRef`.
3. For re-evaluations, updates the ConfigMap with the latest available data.
4. Executes the provider pipeline using the ConfigMap data.
4. Based on the result:
   - **NONE**: No action. The user proceeds through the normal approval flow.
   - **REVIEW**: Emits a `FraudThresholdExceeded` event. Operator reviews the
     FraudEvaluation before approving the user via PlatformAccessApproval.
   - **DEACTIVATE** (AUTO mode): Creates a UserDeactivation resource using the
     existing deactivation mechanism. The UserDeactivation's `reason` and
     `description` fields are populated from the FraudPolicy enforcement template.

**Integration with existing resources:**

| Milo Resource | Integration |
|---|---|
| `User` | FraudEvaluation controller watches User creation |
| `UserDeactivation` | Created by fraud controller in AUTO enforcement mode |
| `PlatformAccessApproval` | Operators review FraudEvaluation before approving |
| `Email` (notification API) | Fraud controller can create Email resources for alerts |
| `Note` (notes API) | Fraud decisions can be recorded as Notes on the user |

**Field indexing** (for efficient lookups):

```go
mgr.GetFieldIndexer().IndexField(ctx,
    &fraudv1alpha1.FraudEvaluation{},
    "spec.userRef.name",
    func(obj client.Object) []string {
        return []string{obj.(*fraudv1alpha1.FraudEvaluation).Spec.UserRef.Name}
    },
)
```

### Zitadel Integration

The fraud evaluation pipeline is triggered from data that originates in the Zitadel
authentication flow. The `auth-provider-zitadel` actions server is the first point of
contact when a user registers and is where HTTP-level metadata (IP, user-agent, etc.)
is available. The actions server creates the input ConfigMap and FraudEvaluation
together during user registration.

**Registration event flow with fraud evaluation:**

```
┌──────────────────────────────────────────────────────────────────┐
│  User registers via IdP (Google, GitHub, etc.)                   │
│  Zitadel fires: user.human.selfregistered / user.human.added     │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│  auth-provider-zitadel actions-server                            │
│  POST /v1/actions/create-user-account                            │
│                                                                  │
│  1. Validate HMAC signature                                      │
│  2. Extract user data (email, name) from event payload           │
│  3. [NEW] Extract HTTP metadata from request context:            │
│     - X-Forwarded-For / RemoteAddr → IP address                  │
│     - User-Agent header                                          │
│     - Accept-Language header                                     │
│  4. Create Milo User resource (registrationApproval: Pending)    │
│  5. [NEW] Create input ConfigMap in fraud-system namespace:      │
│     - ipAddress, userAgent, acceptLanguage from HTTP context      │
│     - emailAddress, firstName, lastName from event payload        │
│  6. [NEW] Create FraudEvaluation with inputRef → ConfigMap       │
└─────────────────────────┬────────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────────┐
│  Milo core control plane                                         │
│                                                                  │
│  FraudEvaluation controller detects new FraudEvaluation          │
│  1. Reads input ConfigMap via spec.inputRef                      │
│  2. Executes FraudPolicy pipeline using ConfigMap data           │
│  3. Writes FraudEvaluation status                                │
│  4. Takes enforcement action if configured                       │
└──────────────────────────────────────────────────────────────────┘
```

**Why a ConfigMap for input data**: The input data is fraud-specific context, not core
user data. A ConfigMap keeps the User resource clean, allows arbitrary key-value pairs
without CRD schema changes, and makes the input inspectable and auditable. The
ConfigMap is owned by the FraudEvaluation (via ownerReference) so it is garbage
collected when the evaluation is deleted.

**Who populates the ConfigMap for each trigger type:**

| Trigger | ConfigMap populated by |
|---|---|
| `UserCreated` event | actions-server (has HTTP context) |
| `Scheduled` | fraud controller (reads latest User spec fields) |
| `DataChange` | fraud controller (reads updated User fields) |
| `Manual` | fraud controller (reads current state) |

For re-evaluations, the fraud controller updates the existing ConfigMap with the
latest available data before executing the pipeline. Fields like `ipAddress` and
`userAgent` are only meaningful at registration time and are left as-is; the
controller only refreshes fields that have a current source (e.g. `emailAddress`
from the User spec).

**Device fingerprinting**: For v1, device fingerprint data is not captured at the
Zitadel level. MaxMind's device tracking requires a JavaScript tag on the registration
page that communicates directly with MaxMind. The resulting device ID would be added
as a `deviceFingerprint` key in the input ConfigMap by the actions-server when device
fingerprinting is enabled. This requires frontend changes to the registration flow
that are outside the scope of the backend API design but should be planned for.

**Authentication webhook enhancement**: The existing `authn-webhook` in
auth-provider-zitadel returns `registrationApproval` status in TokenReview responses.
This can be extended to also check whether a FraudEvaluation exists with a DEACTIVATE
decision, providing defense-in-depth even if the UserDeactivation has not yet been
processed.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

#### How can this feature be enabled / disabled in a live cluster?

The FraudPolicy `enforcement.mode` field supports OBSERVE mode, which scores users
and emits events but takes no enforcement action. Initial deployment will use OBSERVE
mode so the system can be evaluated in production before enabling AUTO enforcement.

#### Does enabling the feature change any default behavior?

No. The feature is opt-in: no fraud evaluation occurs until a FraudPolicy and at
least one FraudProvider are created. User registration proceeds unchanged without
a policy.

#### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Deleting the FraudPolicy stops new evaluations. Setting enforcement mode to
OBSERVE stops enforcement but continues scoring. The feature gate provides a
hard-disable at the controller level. Existing FraudEvaluation resources are
retained; any UserDeactivation resources created by fraud enforcement are
independent and must be reviewed/reverted manually.

### Monitoring Requirements

#### How can an operator determine if the feature is in use by workloads?

- Presence of FraudPolicy and FraudProvider resources in the cluster
- `fraud_evaluations_total` metric (counter of evaluations by decision)
- `fraud_provider_requests_total` metric (counter of provider calls by provider
  and status)

#### How can someone using this feature know that it is working for their instance?

- [x] API .status
  - FraudProvider: `Ready` condition indicates the provider is reachable
  - FraudPolicy: `Ready` condition indicates all referenced providers are available
  - FraudEvaluation: `phase` field shows evaluation lifecycle state
- [x] Events
  - `FraudEvaluationCompleted` -- emitted when an evaluation finishes
  - `FraudThresholdExceeded` -- emitted when a provider score crosses a threshold
  - `FraudProviderUnavailable` -- emitted when a provider call fails
  - `FraudUserDeactivated` -- emitted when AUTO mode triggers a deactivation

#### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [x] Metrics
  - `fraud_evaluation_duration_seconds` (histogram) -- end-to-end pipeline duration
  - `fraud_provider_request_duration_seconds` (histogram, by provider) -- per-provider
    latency
  - `fraud_provider_errors_total` (counter, by provider and error type)
  - `fraud_provider_circuit_open` (gauge, by provider) -- 1 if circuit breaker is open
  - `fraud_evaluations_total` (counter, by decision: NONE/REVIEW/DEACTIVATE/ERROR)

### Dependencies

#### Does this feature depend on any specific services running in the cluster?

- **MaxMind minFraud API** (external SaaS)
  - Usage: IP/email/device risk scoring
  - Impact of outage: Provider marked UNAVAILABLE, FailOpen policy applies, evaluation
    continues without risk scoring (flagged for review)
  - Impact of degraded performance: Increased evaluation latency, potential timeouts

- **moov-io/watchman** (self-hosted or external)
  - Usage: OFAC/CSL sanctions screening
  - Impact of outage: If configured as FailClosed, pipeline halts and evaluations
    enter ERROR state. If FailOpen, sanctions check is skipped (compliance risk).
  - Impact of degraded performance: Increased evaluation latency

## Implementation History

- 2025-12-15: Issue [#505](https://github.com/datum-cloud/enhancements/issues/505)
  opened, initial problem statement and requirements defined
- 2025-12-16: Provider research completed (MaxMind, Sift, Castle, Socure, Persona,
  watchman, Altcha, Stripe Radar)
- 2025-12-16: Initial API design brainstorm (FraudPolicy + FraudScore) by @scotwells
- 2025-12-17: 3-layer architecture proposal (CAPTCHA, risk analysis, sanctions) by
  @JoseSzycho
- 2026-01-11: Feedback on enterprise provider path, abuse handling, machine accounts
  by @jacobsmith928
- 2026-03-11: This enhancement proposal (provisional)

## Drawbacks

- **Operational complexity**: Adds a new controller, 3 new CRDs, and external service
  dependencies. Operators must manage provider credentials and monitor provider health.
- **Cost**: MaxMind charges per-query ($0.02). High-volume sign-ups will incur ongoing
  costs. The pipeline's short-circuit model mitigates this but doesn't eliminate it.
- **False positive risk**: Automated enforcement (even in AUTO mode) can block
  legitimate users. The OBSERVE-first deployment model and manual REVIEW action
  mitigate this, but operator diligence is required.
- **Sanctions list staleness**: Self-hosted watchman requires periodic data refresh.
  Stale lists create compliance risk.

## Alternatives

### Single-provider approach (MaxMind only)

Simpler to implement but does not satisfy the sanctions screening requirement and
provides no path to multi-provider evaluation. The pipeline abstraction has modest
additional complexity but enables significant future extensibility.

### Adopt an enterprise fraud platform (Sift, Castle.io)

These provide comprehensive dashboards and workflow automation but are expensive
($2000+/month), create vendor lock-in, and are overkill for v1 needs. The pluggable
provider model allows migrating to an enterprise platform later by implementing it as
a FraudProvider.

### Inline fraud checks (no CRD abstraction)

Embedding fraud logic directly in the registration flow is simpler but creates tight
coupling, makes it impossible to change providers without code changes, and provides
no audit trail. The CRD-based approach follows Kubernetes conventions, enables
declarative configuration, and produces inspectable resources.

### Unified score normalization

Instead of per-provider thresholds, normalize all scores to a single scale and apply
thresholds on the aggregate. This was considered but rejected because different
providers measure fundamentally different things (IP risk vs. sanctions match
probability) and a single score obscures the source of risk. Per-provider thresholds
are more transparent and actionable.
