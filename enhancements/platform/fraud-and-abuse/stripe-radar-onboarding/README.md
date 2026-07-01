---
status: provisional
stage: alpha
latest-milestone: "v0.0"
---

# Stripe Radar fraud signals in onboarding

- [Summary](#summary)
- [Motivation](#motivation)
  - [Why MaxMind alone falls short](#why-maxmind-alone-falls-short)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Current state in the codebase](#current-state-in-the-codebase)
- [Proposal](#proposal)
  - [User stories](#user-stories)
  - [Notes and constraints](#notes-and-constraints)
  - [Risks and mitigations](#risks-and-mitigations)
- [Design details](#design-details)
  - [Onboarding flow change](#onboarding-flow-change)
  - [Data we pass into fraud scoring](#data-we-pass-into-fraud-scoring)
  - [Option A: Stripe Radar only](#option-a-stripe-radar-only)
  - [Option B: MaxMind plus Stripe Radar](#option-b-maxmind-plus-stripe-radar)
  - [stripe-provider: Radar capture and projection](#stripe-provider-radar-capture-and-projection)
  - [Billing service: payment-method side effects](#billing-service-payment-method-side-effects)
  - [Fraud service: trigger and provider adapters](#fraud-service-trigger-and-provider-adapters)
  - [Policy examples](#policy-examples)
- [Cross-repo work breakdown](#cross-repo-work-breakdown)
- [Production readiness review questionnaire](#production-readiness-review-questionnaire)
  - [Feature enablement and rollback](#feature-enablement-and-rollback)
  - [Monitoring requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
- [Implementation history](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure needed](#infrastructure-needed)

## Summary

Datum's signup fraud check runs at user creation and only uses MaxMind. It scores email,
IP, and device signals, which is useful, but it runs before onboarding collects a payment
method. The card never reaches the evaluation, and neither does Stripe Radar.

That gap is the point of this work. MaxMind on identity signals alone has not been a
strong enough gate for platform access (see
[Why MaxMind alone falls short](#why-maxmind-alone-falls-short)). The payment instrument
is the single best fraud signal we have at signup, and right now we score users before we
ever see it.

This enhancement moves the main signup evaluation to run after a user confirms a payment
instrument during onboarding. Stripe Radar provides payment-network risk. MaxMind keeps
scoring identity and can also use the normalized card metadata the billing API already
models. The work spans three repos: `billing`, the existing `stripe-provider` service,
and `fraud`. It follows the CRD-based payment architecture in billing, not a REST billing
API.

Parent design: [Fraud and Abuse Prevention API](../README.md).

Payment architecture (authoritative): [Payment Methods](https://github.com/milo-os/billing/blob/main/docs/enhancements/payment-methods.md) in the billing repository.

## Motivation

Issue [#763](https://github.com/datum-cloud/enhancements/issues/763) states that payment
methods at signup should improve fraud scoring. Issue [#737](https://github.com/datum-cloud/enhancements/issues/737)
includes Stripe Radar research. The parent fraud enhancement ([#505](https://github.com/datum-cloud/enhancements/issues/505))
deferred payment data to a later phase.

The fraud and compliance research doc ([#739](https://github.com/datum-cloud/enhancements/pull/739),
`research.md`) lists Stripe Radar as a short-term quick win:
block payment fraud at signup, alongside self-hosted OFAC name screening and a trusted-client
whitelist, with specialized user and account platforms (Castle.io, SEON, Sift) as a longer-term
horizon. This doc is the implementation of that Radar recommendation. It is deliberately scoped
to payment fraud; abuse and compliance screening are tracked separately under the parent
enhancement.

Billing defines `PaymentMethod`, `PaymentMethodClass`, and a provider-controller
pattern, and a separate `stripe-provider` service (in `milo-os`) owns SetupIntents and
`StripePaymentMethod` CRDs. Between them they already collect and normalize the card. This
doc adds the missing pieces on top: capturing Radar outcomes in `stripe-provider`,
summarizing them on the billing account, and scoring them in fraud.

### Why MaxMind alone falls short

The first fraud pass shipped with MaxMind minFraud on the `UserCreated` event. It scores
what you can see at registration: IP reputation, email age and disposability, proxy and
VPN detection, and a device signal when the page carries the MaxMind tag. That is a fine
first filter. It is not a strong enough gate to decide who gets onto the platform.

Where it came up short:

- It scores identity and network, not payment. Someone using a stolen card from a clean
  residential IP with an aged email looks fine to MaxMind. The abuse we care most about at
  signup is card fraud, and MaxMind cannot see the card if we never collect it before
  scoring.
- minFraud is at its strongest when you feed it card data: BIN, AVS, CVC, issuer country.
  We were calling it without any of that, so we paid for a score built from half the
  inputs the API supports.
- The scores clustered too tightly to set an auto-enforce threshold we trusted. Real users
  landed in the same band as suspicious ones often enough that the policy had to stay in
  OBSERVE, with a person reviewing most of the flagged cases by hand.

Stripe Radar sees a different layer. It runs on Stripe's own network, so it knows about
card testing, cards that have failed across many other merchants, and velocity patterns
that never touch our MaxMind call. Running both after the card is attached gives us a
signal we can act on, instead of a partial identity score we can only watch.

### Goals

- Run the main fraud pipeline after payment method attach, not at `UserCreated`.
- Keep Stripe integration in **`stripe-provider`**, not in the billing operator.
- Capture Radar outcomes during SetupIntent confirmation and surface them to fraud
  scoring without storing PAN or CVC in Milo.
- Add a `BillingPaymentMethodAttached` trigger and a card-metadata input path to fraud so
  MaxMind can score the card alongside identity.
- Add a `stripe-radar` fraud provider type that scores from projected billing status
  (no live Stripe call during evaluation unless manually refreshed).
- Start in OBSERVE mode before any AUTO enforcement during onboarding.

### Non-Goals

- Building the full onboarding UI ([#762](https://github.com/datum-cloud/enhancements/issues/762)).
- Adding Stripe SDK code to the billing operator (explicitly out of scope per payment
  methods design).
- REST endpoints such as `POST /billingAccounts/{id}/setup-intent` (not part of the
  billing architecture).
- Charging users during onboarding (SetupIntent only).
- Auto-deactivating users on day one.

## Current state in the codebase

This is what exists in the repos today, verified against the code.

| Component                                             | Today          | Notes                                                                                                                                                                                                                   |
| ----------------------------------------------------- | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `PaymentMethod` / `PaymentMethodClass` CRDs           | Yes            | Card details on `PaymentMethod.status.details.card`: brand, last4, BIN, country, expiry, AVS, CVC                                                                                                                       |
| `stripe-provider` service                             | Yes            | In `milo-os/stripe-provider`: `StripePaymentMethod` + `StripeProviderConfig` CRDs, SetupIntent lifecycle, signature-verified webhook, `stripe-go/v81`, pins billing `v0.2.0`; deployed via `infra/apps/stripe-provider` |
| stripe-provider projects card details to billing      | Yes            | The webhook handler, on `setup_intent.succeeded`, sets `PaymentMethod.status.details.card` (brand, last4, country, AVS, CVC) and flips the method to Active                                                             |
| Stripe SDK in billing operator                        | No, by design  | The provider owns Stripe                                                                                                                                                                                                |
| Radar capture in `stripe-provider`                    | **No**         | `StripePaymentMethod.status` has no risk fields, and the webhook never reads the SetupIntent `latest_attempt` risk. This is the core new work                                                                           |
| `BillingAccount.status.paymentMethod` summary         | **No**         | Account status has phase, conditions, linked-projects count, and observed generation only                                                                                                                               |
| `PaymentMethodAttached` condition on `BillingAccount` | **No**         | Billing has `DefaultPaymentMethodReady` only                                                                                                                                                                            |
| Stripe risk fields anywhere                           | **No**         | Not on stripe-provider, billing, or fraud                                                                                                                                                                               |
| Fraud payment / card integration                      | **No**         | Fraud runs `UserCreated` + MaxMind identity only. `provider.Input` has no card or Stripe fields, and fraud has no billing dependency at all                                                                             |
| `stripe-radar` FraudProvider                          | **No**         | Only `maxmind` is registered; the FraudProvider `type` enum allows `maxmind` only                                                                                                                                       |
| Portal onboarding UI                                  | Partial (#762) | Watches CRDs per payment-methods design                                                                                                                                                                                 |

So the picture today is: billing and stripe-provider already collect and normalize the
card (BIN, country, AVS, CVC land on `PaymentMethod.status.details.card`), but nothing reads
Radar, nothing summarizes the payment method at the account level, and the shipped fraud
pipeline never sees any of it. That gap is what this enhancement closes.

## Proposal

Shift the default signup `FraudPolicy` trigger from `UserCreated` to
`BillingPaymentMethodAttached`.

At a high level:

1. User authenticates; Milo creates `User` (no fraud evaluation yet).
2. Portal creates `PaymentMethod`; `stripe-provider` creates `StripePaymentMethod` and
   SetupIntent.
3. Portal confirms card via Stripe.js using `StripePaymentMethod.status.setupIntent.clientSecret`.
4. `stripe-provider` handles `setup_intent.succeeded`, copies Radar outcome to
   `StripePaymentMethod.status`, projects normalized fields to `PaymentMethod.status`.
5. Billing operator sets `BillingAccountConditionPaymentMethodAttached=True` and
   projects a fraud-friendly summary to `BillingAccount.status.paymentMethod`.
6. Fraud `BillingPaymentMethodAttachedTriggerReconciler` creates `FraudEvaluation`.
7. Fraud pipeline runs MaxMind (with card fields) and optionally `stripe-radar`.
8. Portal loading step waits for evaluation completion before granting access.

<<[UNRESOLVED radar-only vs combined pipeline]>>
Two viable defaults:

- **Radar only** after attach: no extra MaxMind cost per signup; weaker on email/IP-only
  signals if we drop the `UserCreated` pass entirely.
- **MaxMind + Radar** after attach: uses card metadata MaxMind already accepts; ~$0.02
  per evaluation; Radar catches payment patterns MaxMind misses.

Recommendation: MaxMind + `stripe-radar` in one post-attach pipeline, OBSERVE mode
first. Keep a lightweight `UserCreated` MaxMind pass only if card-testing abuse shows
up before attach.
<<[/UNRESOLVED]>>

### User stories

#### New user completes onboarding with a clean card

A user signs up, creates a billing account, and adds a card. `stripe-provider` confirms
the SetupIntent with Radar `risk_level: normal`. Billing sets
`PaymentMethodAttached=True`. Fraud runs MaxMind and Radar; decision is `NONE`. The
portal loading screen finishes.

#### Stripe flags elevated risk

Same flow, but Radar returns `elevated`. In OBSERVE mode the user still proceeds; staff
see `FraudThresholdExceeded` and `FraudEvaluation` decision `REVIEW`. AUTO mode (later)
blocks until `PlatformAccessApproval`.

#### User abandons onboarding before adding a card

No `PaymentMethod` reaches `Active`, billing never sets `PaymentMethodAttached`, fraud
does not run, and platform access stays on onboarding screens.

### Notes and constraints

- The billing operator must not import Stripe. All Radar API interaction lives in
  `stripe-provider`.
- `PaymentMethod.status.details` already documents fields "useful for downstream fraud
  scoring" (BIN, country, AVS, CVC). Radar-specific fields stay on
  `StripePaymentMethod` and are copied into `BillingAccount.status.paymentMethod` for
  fraud consumption.
- Fraud resolves the user's billing account via label
  `iam.miloapis.com/owner-user=<uid>` (written by Milo's auto-billing-account
  controller).
- Session IP and user agent come from the identity `Session` API today, not portal HTTP
  headers passed through a billing REST layer.
- Enterprise overflow paths (#730) need staff whitelist or manual evaluation; they skip
  the payment attach trigger.

### Risks and mitigations

| Risk                                           | Mitigation                                                                                                                            |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| False positives block real customers           | OBSERVE first; tune thresholds; manual REVIEW                                                                                         |
| Stripe outage during onboarding                | Portal retry; `PaymentMethod` stays `AwaitingConfirmation` or `Failed` with clear UX                                                  |
| Fraud runs before billing projection completes | Trigger on `PaymentMethodAttached` condition, set only after status summary is written                                                |
| PCI scope creep                                | Stripe Elements + SetupIntent; store IDs and outcomes only                                                                            |
| Radar capture not built in stripe-provider yet | Land the trigger and provider behind a flag; until stripe-provider projects risk fields, `stripe-radar` scores neutral and fails open |

## Design details

### Onboarding flow change

```
User signs in (IdP)
       |
       v
Milo User created                         (no fraud eval)
       |
       v
Portal creates PaymentMethod CRD
       |
       v
stripe-provider creates StripePaymentMethod + SetupIntent
       |
       v
Portal reads clientSecret, confirms via Stripe.js
       |
       v
stripe-provider: setup_intent.succeeded webhook / reconcile
  - write Radar outcome on StripePaymentMethod.status
  - patch PaymentMethod.status (phase Active, details)
       |
       v
billing operator watches PaymentMethod
  - set BillingAccount PaymentMethodAttached=True
  - project status.paymentMethod summary for fraud
       |
       v
fraud BillingPaymentMethodAttached trigger
  - create FraudEvaluation
  - pipeline: maxmind (+ stripe-radar)
       |
       v
Portal waits on FraudEvaluation / onboarding status
```

### Data we pass into fraud scoring

Fraud's `provider.Input` is assembled by the existing resolver (`User`, `Session`,
`BillingAccount.status.paymentMethod`). Proposed additions are marked **new**.

| Field                                                          | Source                                                               | Used by                                     |
| -------------------------------------------------------------- | -------------------------------------------------------------------- | ------------------------------------------- |
| `EmailAddress`, names                                          | `User` CRD                                                           | MaxMind                                     |
| `IPAddress`, `UserAgent`, `TrackingToken`                      | Identity `Session`                                                   | MaxMind                                     |
| `CreditCard.IssuerIDNumber`, `LastDigits`, `Country`, AVS, CVC | `BillingAccount.status.paymentMethod` (from `PaymentMethod.details`) | MaxMind (already wired)                     |
| `StripeRiskScore`, `StripeRiskLevel` **new**                   | Projected from `StripePaymentMethod.status.radar`                    | `stripe-radar` provider                     |
| Billing address on account                                     | `BillingAccount.spec.contactInfo.address`                            | MaxMind billing (future resolver extension) |

MaxMind mapping today (already in `fraud/internal/provider/maxmind/maxmind.go`):

```go
if input.CreditCard.HasAny() {
    cc := &creditCardField{
        LastDigits: input.CreditCard.LastDigits,
        Country:    input.CreditCard.Country,
        AVSResult:  input.CreditCard.AVSResult,
        CVVResult:  input.CreditCard.CVVResult,
    }
    if input.CreditCard.IssuerIDNumber != "" {
        cc.Issuer = &creditCardIssuer{IIN: input.CreditCard.IssuerIDNumber}
    }
    req.CreditCard = cc
}
```

### Option A: Stripe Radar only

Use Radar for payment risk after attach. Optionally keep a cheap `UserCreated` MaxMind
pass before card entry; drop MaxMind from the post-attach policy if Radar-only OBSERVE
data is strong enough.

```yaml
apiVersion: fraud.miloapis.com/v1alpha1
kind: FraudPolicy
metadata:
  name: onboarding-payment
spec:
  stages:
    - name: stripe-radar
      providers:
        - providerRef:
            name: stripe-radar
      thresholds:
        - minScore: 65
          action: REVIEW
        - minScore: 85
          action: DEACTIVATE

  enforcement:
    mode: OBSERVE

  triggers:
    - type: Event
      event: BillingPaymentMethodAttached
```

| Stripe `risk_level` | Normalized score |
| ------------------- | ---------------- |
| normal              | 20               |
| elevated            | 70               |
| highest             | 95               |
| unknown / missing   | 50               |

Prefer numeric `risk_score` from the SetupIntent attempt when Stripe returns it.

### Option B: MaxMind plus Stripe Radar

Default recommendation. Both providers run in the post-attach evaluation. MaxMind uses
card metadata from billing status; Radar uses projected Stripe fields.

```yaml
apiVersion: fraud.miloapis.com/v1alpha1
kind: FraudPolicy
metadata:
  name: onboarding-payment
spec:
  stages:
    - name: identity-and-payment-risk
      providers:
        - providerRef:
            name: maxmind
        - providerRef:
            name: stripe-radar
      thresholds:
        - minScore: 70
          action: REVIEW
        - minScore: 90
          action: DEACTIVATE

  enforcement:
    mode: OBSERVE

  triggers:
    - type: Event
      event: BillingPaymentMethodAttached
```

### stripe-provider: Radar capture and projection

`stripe-provider` is the only component that calls the Stripe API. Its
`StripePaymentMethod.status` already carries `instrument` card details, and the webhook
already projects a normalized subset onto `PaymentMethod.status.details.card`. The new work
is Radar: read the SetupIntent `latest_attempt` risk on confirmation and add a `radar` block
to status. Extend `StripePaymentMethod.status` as follows:

```yaml
apiVersion: stripe.billing.miloapis.com/v1alpha1
kind: StripePaymentMethod
metadata:
  name: corp-visa-stripe
  namespace: org-acme
status:
  phase: Active
  stripeCustomerId: cus_abc123
  stripePaymentMethodId: pm_xyz789
  setupIntent:
    id: seti_456
    clientSecret: seti_456_secret_xxx
  confirmedAt: "2026-06-23T10:15:00Z"
  instrument:
    type: card
    card:
      brand: visa
      last4: "4242"
      country: US
  radar:
    riskScore: 12
    riskLevel: normal
    outcomeType: authorized
```

Illustrative Go in `stripe-provider` after `setup_intent.succeeded`:

```go
func (r *StripePaymentMethodReconciler) applySetupIntentOutcome(
    ctx context.Context,
    spm *stripev1alpha1.StripePaymentMethod,
    si *stripe.SetupIntent,
) error {
    attempt := si.LatestAttempt
    if attempt != nil {
        spm.Status.Radar = &stripev1alpha1.RadarOutcome{
            RiskScore:   attempt.RiskScore,
            RiskLevel:   string(attempt.RiskLevel),
            OutcomeType: string(attempt.Outcome.Type),
        }
    }

    spm.Status.Phase = stripev1alpha1.PhaseActive
    spm.Status.StripePaymentMethodID = si.PaymentMethod.ID

    if err := r.patchPaymentMethodDetails(ctx, spm, si); err != nil {
        return err
    }
    return r.Status().Update(ctx, spm)
}
```

Project normalized card fields onto the parent `PaymentMethod` (billing-owned status
subresource), matching fields already defined on `PaymentMethodCardDetails`:

```go
func (r *StripePaymentMethodReconciler) patchPaymentMethodDetails(
    ctx context.Context,
    spm *stripev1alpha1.StripePaymentMethod,
    si *stripe.SetupIntent,
) error {
    pm := &billingv1alpha1.PaymentMethod{}
    key := client.ObjectKey{
        Namespace: spm.Namespace,
        Name:      spm.Spec.PaymentMethodRef.Name,
    }
    if err := r.Get(ctx, key, pm); err != nil {
        return err
    }

    card := si.PaymentMethod.Card
    pm.Status.Phase = billingv1alpha1.PaymentMethodPhaseActive
    pm.Status.Details = &billingv1alpha1.PaymentMethodDetails{
        Type: billingv1alpha1.PaymentMethodInstrumentTypeCard,
        Card: &billingv1alpha1.PaymentMethodCardDetails{
            Brand:                      string(card.Brand),
            Last4:                      card.Last4,
            Country:                    card.Country,
            ExpiryMonth:                int32(card.ExpMonth),
            ExpiryYear:                 int32(card.ExpYear),
            IssuerIdentificationNumber: card.IIN,
            AVSResult:                  mapAVS(si.LatestAttempt),
            CVCResult:                  mapCVC(si.LatestAttempt),
        },
    }
    return r.Status().Update(ctx, pm)
}
```

RBAC boundary (from payment-methods design): `stripe-provider` patches
`PaymentMethod.status` only; billing operator owns the CRD spec.

### Billing service: payment-method side effects

Billing does not call Stripe or fraud. When a `PaymentMethod` linked to a
`BillingAccount` becomes `Active`, the billing operator:

1. Projects a fraud summary onto `BillingAccount.status.paymentMethod`.
2. Sets `BillingAccountConditionPaymentMethodAttached=True` (triggers fraud).
3. Optionally sets `DefaultPaymentMethodReady` when `spec.defaultPaymentMethodRef`
   points at the active method (condition exists today).

Proposed billing API additions:

```go
const BillingAccountConditionPaymentMethodAttached = "PaymentMethodAttached"

type BillingAccountPaymentMethodSummary struct {
    PaymentMethodRef PaymentMethodRef `json:"paymentMethodRef"`
    BIN              string           `json:"bin,omitempty"`
    Last4            string           `json:"last4,omitempty"`
    Country          string           `json:"country,omitempty"`
    AVSResult        string           `json:"avsResult,omitempty"`
    CVCResult        string           `json:"cvcResult,omitempty"`
    StripeRiskScore  *int64           `json:"stripeRiskScore,omitempty"`
    StripeRiskLevel  string           `json:"stripeRiskLevel,omitempty"`
    AttachedAt       metav1.Time      `json:"attachedAt,omitempty"`
}

type BillingAccountStatus struct {
    // ... existing fields ...
    PaymentMethod *BillingAccountPaymentMethodSummary `json:"paymentMethod,omitempty"`
}
```

Illustrative controller logic:

```go
func (r *BillingAccountReconciler) reconcilePaymentMethodAttached(
    ctx context.Context,
    account *billingv1alpha1.BillingAccount,
    pm *billingv1alpha1.PaymentMethod,
    radar *billingv1alpha1.RadarSummary, // read from PaymentMethod annotation or status extension
) {
    summary := &billingv1alpha1.BillingAccountPaymentMethodSummary{
        PaymentMethodRef: billingv1alpha1.PaymentMethodRef{Name: pm.Name},
        AttachedAt:       metav1.Now(),
    }
    if pm.Status.Details != nil && pm.Status.Details.Card != nil {
        c := pm.Status.Details.Card
        summary.BIN = c.IssuerIdentificationNumber
        summary.Last4 = c.Last4
        summary.Country = c.Country
        summary.AVSResult = c.AVSResult
        summary.CVCResult = c.CVCResult
    }
    if radar != nil {
        summary.StripeRiskScore = radar.RiskScore
        summary.StripeRiskLevel = radar.RiskLevel
    }
    account.Status.PaymentMethod = summary

    apimeta.SetStatusCondition(&account.Status.Conditions, metav1.Condition{
        Type:   billingv1alpha1.BillingAccountConditionPaymentMethodAttached,
        Status: metav1.ConditionTrue,
        Reason: "PaymentMethodActive",
    })
}
```

How Radar reaches billing: `stripe-provider` copies `StripePaymentMethod.status.radar`
onto the `PaymentMethod` via a status annotation (short term) or a small shared status
extension agreed with billing owners. Billing reads that when building the account
summary. Long term, only fraud-relevant normalized fields belong on
`BillingAccount.status`; raw Stripe IDs stay on `StripePaymentMethod`.

Fraud trigger check on the `BillingAccount`:

```go
func isPaymentMethodAttached(a *billingv1alpha1.BillingAccount) bool {
    c := apimeta.FindStatusCondition(a.Status.Conditions,
        billingv1alpha1.BillingAccountConditionPaymentMethodAttached)
    return c != nil && c.Status == metav1.ConditionTrue
}
```

When the condition flips False to True, fraud creates `FraudEvaluation` if a policy
lists `BillingPaymentMethodAttached`. No direct billing-to-fraud RPC.

Portal integration for card collection and onboarding gating is tracked in
[#762](https://github.com/datum-cloud/enhancements/issues/762) and follows the
CRD watch flow in
[payment-methods.md](https://github.com/milo-os/billing/blob/main/docs/enhancements/payment-methods.md).
This doc does not specify portal implementation.

### Fraud service: trigger and provider adapters

#### Resolver extension

Extend `resolvePaymentMethod` (already reads `status.paymentMethod`) to copy Radar
fields into `provider.Input`:

```go
func (r *Resolver) resolvePaymentMethod(ctx context.Context, userUID string, input *provider.Input) error {
    // ... existing BillingAccount lookup ...
    pm := pick.Status.PaymentMethod
    input.CreditCard = provider.CreditCard{
        IssuerIDNumber: pm.BIN,
        LastDigits:     pm.Last4,
        Country:        pm.Country,
        AVSResult:      pm.AVSResult,
        CVVResult:      pm.CVCResult,
    }
    if pm.StripeRiskScore != nil {
        input.StripeRiskScore = *pm.StripeRiskScore
    }
    input.StripeRiskLevel = pm.StripeRiskLevel
    return nil
}
```

Add optional Radar fields to `provider.Input` (new):

```go
type Input struct {
    // ... existing fields ...
    StripeRiskScore int64
    StripeRiskLevel string
}
```

#### stripe-radar provider (new)

Register alongside MaxMind in `supportedProviderTypes`. Reads projected billing status;
does not call Stripe during normal evaluation.

```go
type StripeRadar struct{}

func (StripeRadar) Name() string { return "stripe-radar" }

func (StripeRadar) Evaluate(ctx context.Context, input provider.Input) provider.Result {
    if input.StripeRiskScore > 0 {
        return provider.Result{Score: float64(input.StripeRiskScore)}
    }
    switch input.StripeRiskLevel {
    case "normal":
        return provider.Result{Score: 20}
    case "elevated":
        return provider.Result{Score: 70}
    case "highest":
        return provider.Result{Score: 95}
    default:
        return provider.Result{Score: 50}
    }
}
```

```yaml
apiVersion: fraud.miloapis.com/v1alpha1
kind: FraudProvider
metadata:
  name: stripe-radar
spec:
  type: stripe-radar
  config: {}
  failurePolicy: FailOpen
```

Update `FraudProvider` CRD validation to allow `stripe-radar` in addition to `maxmind`.

#### Trigger wiring

Add a `BillingPaymentMethodAttachedTriggerReconciler` to fraud that:

- Watches `BillingAccount` updates where `PaymentMethodAttached` flips to True
- Finds a `FraudPolicy` with `event: BillingPaymentMethodAttached`
- Creates `FraudEvaluation` keyed by the `iam.miloapis.com/owner-user` label

Deprecate `UserCreated` as the primary signup trigger once billing projection ships.
Keep it for environments without payment collection.

### Policy examples

See [Option A](#option-a-stripe-radar-only) and [Option B](#option-b-maxmind-plus-stripe-radar).

## Cross-repo work breakdown

| Repo                            | Work                                                                                                                                                                                                                                                                                          |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **stripe-provider** (`milo-os`) | The webhook already projects card details to `PaymentMethod.status`. Add Radar: read the SetupIntent `latest_attempt` risk (`risk_score`, `risk_level`, outcome) in the webhook, add a `RadarOutcome` to `StripePaymentMethod.status`, and project the risk fields alongside the card details |
| **billing**                     | Add an account-level payment summary on `BillingAccount.status`, a `PaymentMethodAttached` condition, and Stripe risk fields, all projected when a `PaymentMethod` goes Active. `DefaultPaymentMethodReady` and the card details already exist                                                |
| **fraud**                       | Add a billing dependency at `v0.2.0`+, card and Stripe fields on `provider.Input`, a resolver that reads the new billing summary, the `BillingPaymentMethodAttached` trigger, the `stripe-radar` provider, the `stripe-radar` enum value, and policy tuning                                   |
| **cloud-portal**                | Onboarding screens that watch CRDs (#762)                                                                                                                                                                                                                                                     |
| **milo**                        | Ensure the auto-created `BillingAccount` carries `iam.miloapis.com/owner-user` (fraud resolves by this label; it is not set in billing today)                                                                                                                                                 |

## Production readiness review questionnaire

### Feature enablement and rollback

- Enable: ship Radar capture in `stripe-provider`, billing projection, fraud trigger +
  `stripe-radar` provider, `FraudPolicy` with `BillingPaymentMethodAttached` in OBSERVE
  mode.
- Disable: revert policy trigger to `UserCreated` only; leave payment collection
  working without Radar scoring.
- Billing and fraud can ship independently behind feature flags until projection is
  consistent.

### Monitoring requirements

- `fraud_evaluations_total{trigger="BillingPaymentMethodAttached",decision=...}`
- `billing_payment_method_attached_total`
- `stripe_setup_intent_outcome{level=...}` (stripe-provider metrics)
- Activity timeline events from payment method policy (planned in billing config)

### Dependencies

| Dependency           | Role                                                          |
| -------------------- | ------------------------------------------------------------- |
| **stripe-provider**  | SetupIntent, Radar capture, `PaymentMethod` status projection |
| **billing operator** | `PaymentMethodAttached` condition and fraud summary           |
| **fraud operator**   | Trigger + MaxMind + stripe-radar providers                    |
| **Stripe Radar**     | Evaluates SetupIntent confirmation                            |
| **MaxMind minFraud** | Identity + card metadata scoring                              |
| **cloud-portal**     | CRD-driven onboarding UI                                      |

Related issues: [#505](https://github.com/datum-cloud/enhancements/issues/505),
[#737](https://github.com/datum-cloud/enhancements/issues/737),
[#762](https://github.com/datum-cloud/enhancements/issues/762),
[#763](https://github.com/datum-cloud/enhancements/issues/763).

## Implementation history

- 2026-06-23: Initial proposal (provisional).
- 2026-06-23: Revised to align with the billing payment-methods CRD architecture and
  stripe-provider ownership of Stripe.
- 2026-06-24: Scoped against the current state of each repo. Today: `stripe-provider` runs
  the SetupIntent lifecycle and projects card details onto
  `PaymentMethod.status.details.card`, but captures no Radar data; billing has the card
  details and `DefaultPaymentMethodReady` but no account-level payment summary, no
  `PaymentMethodAttached` condition, and no risk fields; fraud has no billing dependency or
  card path. Added the motivation for moving past MaxMind-only scoring and the work needed
  in each repo.

## Drawbacks

- Depends on `stripe-provider` gaining Radar capture; the service exists but does not read
  or project risk data yet, so fraud-onboarding cannot score Radar until that lands.
- More moving parts than a monolithic billing REST service, but matches the platform
  extensibility model.
- Radar on SetupIntent may differ from Radar on first real charge.
- Cross-repo coordination on `BillingAccount.status.paymentMethod` shape.

## Alternatives

### Billing service owns Stripe (rejected)

Single service calling Stripe and fraud directly. Rejected in billing payment-methods
design: couples billing to one provider and breaks the Gateway-style extensibility model.

### Radar-only scoring in stripe-provider with no fraud API

Branch in stripe-provider on Radar level and set a `BillingAccount` annotation to block
access. Fast to prototype, no `FraudEvaluation` audit trail, bypasses staff portal
fraud tooling.

### $0 PaymentIntent instead of SetupIntent

Stronger Radar signal, worse UX, unnecessary for signup card-on-file.

## Infrastructure needed

- Stripe account with Radar enabled.
- `stripe-provider` deployment with webhook endpoint and secret key.
- Existing MaxMind credentials for fraud operator.
- `PaymentMethodClass` + `StripeProviderConfig` installed by platform operators.
