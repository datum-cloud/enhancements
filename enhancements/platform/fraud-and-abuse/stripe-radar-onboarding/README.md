---
status: provisional
stage: alpha
latest-milestone: "v0.0"
---

# Stripe Radar fraud signals in onboarding

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User stories](#user-stories)
  - [Notes and constraints](#notes-and-constraints)
  - [Risks and mitigations](#risks-and-mitigations)
- [Design details](#design-details)
  - [Onboarding flow change](#onboarding-flow-change)
  - [Data we pass into fraud scoring](#data-we-pass-into-fraud-scoring)
  - [Option A: Stripe Radar only](#option-a-stripe-radar-only)
  - [Option B: MaxMind plus Stripe Radar](#option-b-maxmind-plus-stripe-radar)
  - [Cloud portal: collecting the card](#cloud-portal-collecting-the-card)
  - [Billing service: persisting payment signals](#billing-service-persisting-payment-signals)
  - [Fraud controller: provider adapters](#fraud-controller-provider-adapters)
  - [Triggering evaluation after payment setup](#triggering-evaluation-after-payment-setup)
  - [Policy examples](#policy-examples)
  - [Platform access gating](#platform-access-gating)
- [Production readiness review questionnaire](#production-readiness-review-questionnaire)
  - [Feature enablement and rollback](#feature-enablement-and-rollback)
  - [Monitoring requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
- [Implementation history](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure needed](#infrastructure-needed)

## Summary

Datum's current fraud pipeline runs at user creation with MaxMind and sanctions
screening. That works for email and IP signals, but it misses the strongest signal we
will have during onboarding: whether the card Stripe accepts is suspicious.

This enhancement moves the primary fraud evaluation to after the user adds a payment
method in the cloud portal. Stripe Radar supplies payment-side risk data. We can use
Radar on its own, or feed Stripe fields into MaxMind's minFraud API alongside the
existing identity signals.

The onboarding UI is still in progress ([#762](https://github.com/datum-cloud/enhancements/issues/762)),
and billing is not finished ([#763](https://github.com/datum-cloud/enhancements/issues/763)).
This doc describes the backend and portal wiring so fraud scoring and payment collection
land together instead of as two separate follow-ups.

Parent design: [Fraud and Abuse Prevention API](../README.md).

## Motivation

Issue [#763](https://github.com/datum-cloud/enhancements/issues/763) already states the
intent: payment methods at signup should improve fraud scoring. Issue [#737](https://github.com/datum-cloud/enhancements/issues/737)
lists Stripe Radar research as an explicit deliverable. The v1 fraud API
([#505](https://github.com/datum-cloud/enhancements/issues/505)) deliberately skipped
payment data; `PaymentSubmitted` and `BillingProfile` sources were marked future work.

Collecting a card during onboarding gives us billing country, card fingerprint, CVC
check results, and Radar risk scores. Running fraud checks only at `UserCreated` means
we score users before we have that data, then never re-score them when we do.

### Goals

- Run the main fraud pipeline after a user successfully attaches a payment method during
  onboarding, not at account creation.
- Add a built-in `stripe-radar` FraudProvider adapter (or equivalent webhook provider)
  that reads Radar outcomes from Stripe objects we already create for billing.
- Pass Stripe payment fields into MaxMind when both providers are enabled.
- Keep sanctions screening (watchman) on the pipeline; it does not depend on payment data.
- Start in OBSERVE mode, same as the existing fraud rollout.
- Block or flag platform access based on `FraudEvaluation` before the user leaves the
  onboarding loading step.

### Non-Goals

- Building the full onboarding UI (tracked separately in #762).
- Replacing Stripe with another payment processor.
- Charging users during onboarding. Card collection uses SetupIntent, not a real charge.
- Auto-deactivating users in production on day one. OBSERVE first, AUTO later.
- Storing full PAN or CVC in Milo. Stripe Elements keeps card data off our servers.

## Proposal

Shift the default `FraudPolicy` trigger from `UserCreated` to `PaymentMethodAttached`
(a new event name; see [Triggering evaluation](#triggering-evaluation-after-payment-setup)).

At a high level:

1. User authenticates and gets a Milo `User` (unchanged).
2. Portal onboarding creates a billing account and collects a card through Stripe
   Elements.
3. Billing service confirms the SetupIntent, stores the Stripe customer and payment
   method references, and copies Radar outcome fields onto the billing account status.
4. Billing emits `PaymentMethodAttached` (via annotation, event, or explicit API call).
5. Fraud controller runs the configured pipeline and updates `FraudEvaluation`.
6. Portal polls or watches onboarding state and either continues or shows a review or
   decline path.

<<[UNRESOLVED radar-only vs combined pipeline]>>
Two viable defaults:

- **Radar only** for the payment stage: cheaper if we already pay for Radar through
  Stripe billing, fewer external API calls, weaker on email/IP-only abuse before card entry.
- **MaxMind then Radar** (or parallel): better coverage, two vendors to operate, ~$0.02
  extra per evaluation on top of Stripe.

Recommendation for first ship: MaxMind + watchman + Stripe Radar in one pipeline,
Radar in a dedicated stage after payment fields exist. Revisit if Radar-only OBSERVE
data is strong enough on its own.
<<[/UNRESOLVED]>>

### User stories

#### New user completes onboarding with a clean card

A user signs up, fills in billing account details, and adds a Visa through Stripe
Elements. The billing service confirms the SetupIntent, sees Radar `risk_level: normal`,
and triggers fraud evaluation. MaxMind returns a low score, watchman finds no sanctions
match, and the evaluation decision is `NONE`. The portal loading screen finishes and the
user enters the org.

#### Stripe flags elevated risk

Same flow, but Radar returns `risk_level: elevated` with a high risk score. In OBSERVE
mode the user still proceeds, but staff get a `FraudThresholdExceeded` event and the
`FraudEvaluation` shows `REVIEW`. In AUTO mode (later), platform access stays blocked
until an operator approves via `PlatformAccessApproval`.

#### User abandons onboarding before adding a card

The user exists in Milo but has no payment method. No `PaymentMethodAttached` event
fires, so no fraud evaluation runs and platform access stays limited to the onboarding
screens. This is intentional friction.

### Notes and constraints

- Radar evaluates the SetupIntent confirmation attempt. We do not need a separate charge
  for onboarding fraud scoring.
- MaxMind's minFraud API accepts billing address and payment instrument metadata. We
  map Stripe fields we already have; we never send raw card numbers to MaxMind.
- The existing v1 fraud doc waits for audit log IP data. That still helps for MaxMind,
  but payment-triggered evaluation can run immediately because the portal has client IP
  at submit time and can pass it through the billing API request.
- Users who cannot use a card (enterprise overflow path from #730) need a whitelist or
  manual approval path that skips `PaymentMethodAttached` and uses a staff-triggered
  evaluation instead.

### Risks and mitigations

| Risk | Mitigation |
|------|------------|
| False positives block real customers | OBSERVE mode first; tune thresholds on production data; manual REVIEW before AUTO |
| Stripe outage during onboarding | FailOpen on Radar provider; allow retry in portal; do not hard-lock user without fallback messaging |
| MaxMind outage | Existing FailOpen policy; Radar stage still runs |
| PCI scope creep | Stripe Elements + SetupIntent only; store Stripe IDs and outcome metadata, not card numbers |
| Users stuck if evaluation is slow | Portal timeout with "still checking" state; async completion via polling `FraudEvaluation` or onboarding status |

## Design details

### Onboarding flow change

```
User signs in (IdP)
       |
       v
Create Milo User                    <-- no fraud evaluation here anymore
       |
       v
Portal: billing account form
       |
       v
Stripe Elements: SetupIntent        <-- card collected client-side
       |
       v
Billing service confirms SetupIntent
  - save stripeCustomerId
  - save stripePaymentMethodId
  - copy radar outcome to status
       |
       v
Emit PaymentMethodAttached
       |
       v
Fraud controller: pipeline
  (watchman + maxmind + stripe-radar)
       |
       v
Portal: loading step completes or shows blocked/review state
```

Sanctions screening can stay in the pipeline even though it only needs name fields from
`User`. It does not need to wait for payment data.

### Data we pass into fraud scoring

Fields assembled at `PaymentMethodAttached` time:

| Canonical field | Source | Used by |
|-----------------|--------|---------|
| `emailAddress`, `firstName`, `lastName` | `User` | MaxMind, watchman |
| `ipAddress`, `userAgent` | Portal request headers (and audit log when available) | MaxMind |
| `billingCountry`, `billingPostalCode`, `billingCity` | Billing account address | MaxMind |
| `cardCountry`, `cardFunding`, `cardBrand`, `cardLast4` | Stripe PaymentMethod | MaxMind |
| `stripeRiskScore`, `stripeRiskLevel` | Stripe SetupIntent / latest attempt outcome | stripe-radar adapter |
| `stripePaymentMethodId`, `stripeCustomerId` | Billing account status | audit metadata only |

### Option A: Stripe Radar only

Use Radar as the only payment-risk provider. Keep watchman for sanctions. Drop MaxMind
from the post-payment pipeline (or run a cheap pre-payment MaxMind pass only if we still
want email/IP signal before card entry).

FraudPolicy sketch:

```yaml
apiVersion: fraud.miloapis.com/v1alpha1
kind: FraudPolicy
metadata:
  name: onboarding-payment
spec:
  stages:
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

    - name: stripe-radar
      providers:
        - providerRef:
            name: stripe-radar
      thresholds:
        - minScore: 65
          action: REVIEW
        - minScore: 85
          action: DEACTIVATE
      required: false

  enforcement:
    mode: OBSERVE

  triggers:
    - type: Event
      event: PaymentMethodAttached
```

Stripe maps `risk_level` to our 0-100 scale in the adapter:

| Stripe `risk_level` | Normalized score |
|---------------------|------------------|
| normal | 20 |
| elevated | 70 |
| highest | 95 |
| unknown / missing | 50 (neutral, flagged in reasons) |

When Stripe exposes a numeric `risk_score` on the attempt outcome, prefer that value
and map 0-100 directly.

### Option B: MaxMind plus Stripe Radar

Run both in one evaluation after payment setup. MaxMind gets richer input because billing
address and card metadata exist. Radar catches payment patterns MaxMind will not see.

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
          inputMapping:
            fields:
              ipAddress: "device.ip_address"
              emailAddress: "email.address"
              userAgent: "device.user_agent"
              billingCountry: "billing.address.country"
              billingPostalCode: "billing.address.postal_code"
              cardCountry: "payment.credit_card.issuer_id_country"  # mapped from Stripe country
        - providerRef:
            name: stripe-radar
      thresholds:
        - minScore: 70
          action: REVIEW
        - minScore: 90
          action: DEACTIVATE
      required: false
      shortCircuit:
        skipWhenBelow: 15

    - name: sanctions-screening
      providers:
        - providerRef:
            name: watchman
      required: true

  enforcement:
    mode: OBSERVE

  triggers:
    - type: Event
      event: PaymentMethodAttached
```

Stage decision logic stays the same as the parent fraud API: highest severity action
wins across executed providers.

### Cloud portal: collecting the card

Illustrative React flow using Stripe.js. Assumes billing API exposes
`POST /orgs/{org}/billingAccounts/{id}/setup-intent` and
`POST /orgs/{org}/billingAccounts/{id}/payment-method`.

```typescript
// app/onboarding/payment-step.tsx (illustrative)
import { loadStripe } from "@stripe/stripe-js";
import { Elements, PaymentElement, useStripe, useElements } from "@stripe/react-stripe-js";

async function createSetupIntent(billingAccountId: string): Promise<{ clientSecret: string }> {
  const res = await fetch(`/api/billing/accounts/${billingAccountId}/setup-intent`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      // Forward client IP for fraud input; prefer edge-injected header in production.
      "X-Forwarded-For": await fetch("/api/client-ip").then((r) => r.text()),
    },
  });
  if (!res.ok) throw new Error("setup intent failed");
  return res.json();
}

function PaymentForm({ billingAccountId, onComplete }: { billingAccountId: string; onComplete: () => void }) {
  const stripe = useStripe();
  const elements = useElements();

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    if (!stripe || !elements) return;

    const { error, setupIntent } = await stripe.confirmSetup({
      elements,
      confirmParams: {
        return_url: `${window.location.origin}/onboarding/complete`,
      },
      redirect: "if_required",
    });

    if (error) {
      // Show card decline or validation error in UI.
      return;
    }

    if (setupIntent?.status === "succeeded" && setupIntent.payment_method) {
      // Tell billing to attach PM and kick off fraud evaluation.
      await fetch(`/api/billing/accounts/${billingAccountId}/payment-method`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          stripeSetupIntentId: setupIntent.id,
          stripePaymentMethodId:
            typeof setupIntent.payment_method === "string"
              ? setupIntent.payment_method
              : setupIntent.payment_method.id,
        }),
      });
      onComplete();
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <PaymentElement />
      <button type="submit">Continue</button>
    </form>
  );
}
```

The portal loading step after submit polls onboarding status until fraud evaluation
completes or times out.

### Billing service: persisting payment signals

After the portal posts the confirmed SetupIntent, billing retrieves the Stripe objects,
updates the billing account, and requests fraud evaluation.

```go
// internal/billing/payment_method.go (illustrative)
func (s *Service) AttachPaymentMethod(ctx context.Context, req AttachPaymentMethodRequest) error {
    si, err := s.stripe.SetupIntents.Get(req.StripeSetupIntentID, nil)
    if err != nil {
        return fmt.Errorf("get setup intent: %w", err)
    }
    if si.Status != stripe.SetupIntentStatusSucceeded {
        return ErrSetupIntentNotSucceeded
    }

    pm, err := s.stripe.PaymentMethods.Get(req.StripePaymentMethodID, nil)
    if err != nil {
        return fmt.Errorf("get payment method: %w", err)
    }

    riskScore, riskLevel := extractRadarOutcome(si)

    account, err := s.store.GetBillingAccount(ctx, req.BillingAccountID)
    if err != nil {
        return err
    }

    account.Spec.DefaultPaymentMethodRef = req.StripePaymentMethodID
    account.Status.Stripe = billingv1alpha1.StripeStatus{
        CustomerID:        account.Status.Stripe.CustomerID,
        PaymentMethodID:   pm.ID,
        RiskScore:         riskScore,
        RiskLevel:         riskLevel,
        CardCountry:       pm.Card.Country,
        CardFunding:       string(pm.Card.Funding),
        CardBrand:         string(pm.Card.Brand),
        CardLast4:         pm.Card.Last4,
        SetupIntentID:     si.ID,
        AttachedAt:        metav1.Now(),
    }

    if err := s.store.UpdateBillingAccount(ctx, account); err != nil {
        return err
    }

    return s.fraud.RequestEvaluation(ctx, fraudeval.Request{
        UserRef:      req.UserRef,
        Trigger:      fraudeval.TriggerPaymentMethodAttached,
        BillingRef:   account.Name,
        ClientIP:     req.ClientIP,
        UserAgent:    req.UserAgent,
    })
}

func extractRadarOutcome(si *stripe.SetupIntent) (score *int64, level string) {
    if si.LatestAttempt == nil || si.LatestAttempt.RiskScore == 0 {
        return nil, "unknown"
    }
    scoreVal := si.LatestAttempt.RiskScore
    return &scoreVal, string(si.LatestAttempt.RiskLevel)
}
```

Extend `BillingAccount.status` (exact shape TBD with billing API owners):

```yaml
apiVersion: billing.miloapis.com/v1alpha1
kind: BillingAccount
metadata:
  name: org-acme-billing
  namespace: org-acme
spec:
  displayName: ACME Engineering
  address:
    country: US
    postalCode: "94107"
    city: San Francisco
status:
  phase: Active
  stripe:
    customerId: cus_abc123
    paymentMethodId: pm_xyz789
    setupIntentId: seti_456
    riskScore: 12
    riskLevel: normal
    cardCountry: US
    cardFunding: credit
    cardBrand: visa
    cardLast4: "4242"
    attachedAt: "2026-06-23T10:15:00Z"
  fraudEvaluationRef:
    name: user-12345
```

### Fraud controller: provider adapters

#### stripe-radar adapter

Reads canonical input populated from `BillingAccount.status.stripe`. Does not call
Stripe again if billing already copied the outcome (avoids duplicate API traffic and
race conditions). Optionally re-fetches from Stripe when `forceRefresh: true` on manual
re-evaluation.

```go
// internal/controllers/fraud/providers/stripe_radar.go (illustrative)
type StripeRadarAdapter struct {
    stripe *stripe.Client // optional; used for refresh only
}

func (a *StripeRadarAdapter) Evaluate(ctx context.Context, input fraud.Input) (fraud.ProviderResult, error) {
    score, ok := input.Float("stripeRiskScore")
    level := input.String("stripeRiskLevel")

    if !ok {
        score = normalizeRiskLevel(level)
    }

    reasons := []string{}
    if level != "" && level != "normal" {
        reasons = append(reasons, "stripe_risk_level_"+level)
    }
    if input.String("cardCountry") != "" && input.String("billingCountry") != "" &&
        input.String("cardCountry") != input.String("billingCountry") {
        reasons = append(reasons, "card_country_mismatch")
    }

    return fraud.ProviderResult{
        Score:    score,
        Reasons:  reasons,
        Metadata: map[string]any{
            "stripePaymentMethodId": input.String("stripePaymentMethodId"),
            "stripeRiskLevel":       level,
        },
    }, nil
}

func normalizeRiskLevel(level string) float64 {
    switch level {
    case "normal":
        return 20
    case "elevated":
        return 70
    case "highest":
        return 95
    default:
        return 50
    }
}
```

FraudProvider resource:

```yaml
apiVersion: fraud.miloapis.com/v1alpha1
kind: FraudProvider
metadata:
  name: stripe-radar
spec:
  type: stripe-radar
  onFailure: FailOpen
  timeout: 2s
  inputMapping:
    fields:
      stripeRiskScore: "stripeRiskScore"
      stripeRiskLevel: "stripeRiskLevel"
      stripePaymentMethodId: "stripePaymentMethodId"
      cardCountry: "cardCountry"
      billingCountry: "billingCountry"
```

#### MaxMind adapter with payment fields

When billing address and card metadata exist, extend the existing MaxMind request builder:

```go
// internal/controllers/fraud/providers/maxmind.go (illustrative extension)
func buildMinFraudRequest(input fraud.Input) minfraud.Request {
    req := minfraud.Request{
        Device: minfraud.Device{
            IPAddress:      input.String("ipAddress"),
            UserAgent:      input.String("userAgent"),
            AcceptLanguage: input.String("acceptLanguage"),
        },
        Email: minfraud.Email{
            Address: input.String("emailAddress"),
            Domain:  input.String("emailDomain"),
        },
    }

    if input.Has("billingCountry") {
        req.Billing = minfraud.Billing{
            Country: input.String("billingCountry"),
            Postal:  input.String("billingPostalCode"),
            City:    input.String("billingCity"),
        }
    }

    if input.Has("cardCountry") {
        req.Payment = minfraud.Payment{
            Processor: "stripe",
            WasAuthorized: false, // SetupIntent only; no charge yet
            CreditCard: minfraud.CreditCard{
                Country: input.String("cardCountry"),
                Brand:   input.String("cardBrand"),
                Last4:   input.String("cardLast4"),
            },
        }
    }

    return req
}
```

### Triggering evaluation after payment setup

Introduce `PaymentMethodAttached` as a first-class trigger event (name bikeshedding OK;
`PaymentSubmitted` from the parent doc is broader and may include actual charges later).

Implementation options, in preference order:

1. **Billing calls fraud service directly** after persisting Stripe status (simplest,
   synchronous request, controller creates or updates `FraudEvaluation`).
2. **Annotation on BillingAccount** `fraud.miloapis.com/evaluate=true` watched by fraud
   controller (more Kubernetes-native, slightly more latency).
3. **Explicit FraudEvaluation create** from billing with `spec.trigger.event:
   PaymentMethodAttached` (matches how registration could work).

Example of option 3:

```yaml
apiVersion: fraud.miloapis.com/v1alpha1
kind: FraudEvaluation
metadata:
  name: user-12345
spec:
  userRef:
    name: user-12345
  policyRef:
    name: onboarding-payment
  trigger:
    type: Event
    event: PaymentMethodAttached
    billingAccountRef:
      name: org-acme-billing
      namespace: org-acme
```

Controller input assembly adds a billing account fetch:

```go
func (r *FraudEvaluationReconciler) assembleInput(ctx context.Context, eval *fraudv1alpha1.FraudEvaluation) (fraud.Input, error) {
    input := fraud.Input{}

    user, err := r.getUser(ctx, eval.Spec.UserRef.Name)
    if err != nil {
        return nil, err
    }
    input.Merge(userFields(user))

    if ref := eval.Spec.Trigger.BillingAccountRef; ref != nil {
        acct, err := r.getBillingAccount(ctx, ref.Namespace, ref.Name)
        if err != nil {
            return nil, err
        }
        input.Merge(billingStripeFields(acct))
        input.Merge(billingAddressFields(acct))
    }

    if eval.Spec.Trigger.ClientIP != "" {
        input.Set("ipAddress", eval.Spec.Trigger.ClientIP)
    }

    return input, nil
}
```

Deprecate `UserCreated` as the primary trigger for new signups once this ships. Keep it
configurable for environments without billing enabled.

### Policy examples

See [Option A](#option-a-stripe-radar-only) and [Option B](#option-b-maxmind-plus-stripe-radar).

Default recommendation: Option B in OBSERVE mode for at least one release cycle.

### Platform access gating

Onboarding completion checks fraud state before granting full portal access:

```typescript
// app/onboarding/wait-for-fraud.ts (illustrative)
type OnboardingStatus = {
  paymentMethodAttached: boolean;
  fraudEvaluation?: {
    phase: "Pending" | "Completed" | "Error";
    decision: "NONE" | "REVIEW" | "DEACTIVATE";
  };
};

export async function waitForOnboardingReady(userId: string, timeoutMs = 30000): Promise<"ready" | "review" | "blocked" | "timeout"> {
  const deadline = Date.now() + timeoutMs;

  while (Date.now() < deadline) {
    const status: OnboardingStatus = await fetch(`/api/onboarding/status/${userId}`).then((r) => r.json());

    if (!status.paymentMethodAttached) {
      return "blocked";
    }

    const eval_ = status.fraudEvaluation;
    if (!eval_ || eval_.phase === "Pending") {
      await new Promise((r) => setTimeout(r, 1000));
      continue;
    }

    if (eval_.decision === "DEACTIVATE") return "blocked";
    if (eval_.decision === "REVIEW") return "review";
    return "ready";
  }

  return "timeout";
}
```

In OBSERVE mode, `REVIEW` still returns `ready` for the user but surfaces the case to
staff. AUTO mode maps `REVIEW` or `DEACTIVATE` to blocked or manual approval flows.

## Production readiness review questionnaire

### Feature enablement and rollback

#### How can this feature be enabled / disabled in a live cluster?

- Configure `FraudPolicy` with `PaymentMethodAttached` trigger and desired providers.
- Set `enforcement.mode: OBSERVE` for initial rollout.
- Disable by switching trigger back to `UserCreated` only, or deleting the payment-stage
  providers from the policy.
- Billing feature flag can skip fraud request on attach without blocking card save.

#### Does enabling the feature change any default behavior?

Yes, for new signups: fraud evaluation moves from registration time to after payment
method attach. Users who never add a card will not have a completed fraud evaluation.

#### Can the feature be disabled once it has been enabled?

Yes. Existing `FraudEvaluation` resources remain. Onboarding can bypass fraud polling
via portal feature flag.

### Monitoring requirements

Metrics (extend existing fraud metrics from parent doc):

- `fraud_evaluations_total{trigger="PaymentMethodAttached",decision=...}`
- `fraud_onboarding_blocked_total` (portal-reported or derived from decisions)
- `billing_payment_method_attach_total{outcome=...}`
- `billing_stripe_radar_level{level=...}` histogram or counter

Events:

- `PaymentMethodAttached`
- `FraudThresholdExceeded` with `trigger=PaymentMethodAttached`
- `FraudEvaluationCompleted`

### Dependencies

| Dependency | Usage | Outage impact |
|------------|-------|---------------|
| Stripe API | SetupIntent, PaymentMethod, Radar outcome | Cannot complete onboarding payment step |
| MaxMind minFraud | Optional identity/payment enrichment | FailOpen; Radar and watchman still run |
| moov-io/watchman | Sanctions | FailClosed per existing policy |
| Billing API | Persist PM, emit evaluation trigger | Blocks onboarding completion |
| Cloud portal onboarding UI | Collect card, poll fraud status | No user-facing flow |

Related issues: [#505](https://github.com/datum-cloud/enhancements/issues/505),
[#737](https://github.com/datum-cloud/enhancements/issues/737),
[#762](https://github.com/datum-cloud/enhancements/issues/762),
[#763](https://github.com/datum-cloud/enhancements/issues/763).

## Implementation history

- 2026-06-23: Initial proposal (provisional).

## Drawbacks

- Onboarding gets longer: users must add a card before full platform access.
- Couples fraud rollout to billing and Stripe integration readiness.
- Two-provider pipelines cost more and need separate threshold tuning.
- Radar on SetupIntent may behave differently than Radar on real charges; chargeback
  signal arrives later at first actual payment.

## Alternatives

### Keep fraud at UserCreated, re-run at payment attach

Run a lightweight MaxMind pass at signup, then a full pass at payment. More evaluations,
more cost, but catches obvious abuse before card entry. Worth it if card-testing abuse
becomes a problem.

### Stripe Radar only, no Milo fraud API

Call Radar in billing and branch in application code without `FraudEvaluation` resources.
Faster to hack together, worse audit trail, does not reuse staff portal fraud tooling
from #505.

### $0 auth instead of SetupIntent

Run a zero-amount PaymentIntent to force a charge-shaped Radar evaluation. Stronger signal,
worse UX, possible issuer declines on $0 auths. Not recommended for v1.

### Webhook-only Stripe integration

Use `FraudProvider` type `webhook` pointing at a small service that wraps Stripe Radar
rules. More moving parts than a built-in adapter.

## Infrastructure needed

- Stripe account with Radar enabled (included on standard Stripe pricing tiers; confirm
  Radar for Fraud Teams if we need custom rules later).
- Secrets: `stripe-secret-key` in billing service, existing `maxmind-credentials` in
  fraud-system namespace.
- Billing API endpoints for SetupIntent creation and payment method attach (part of #763).
- Portal onboarding routes from #762.
