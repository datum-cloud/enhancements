---
status: implementable
stage: alpha
latest-milestone: "v0.x"
---

# Feature-flagging via Milo Entitlements

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Notes and Constraints](#notes-and-constraints)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Conceptual Model](#conceptual-model)
  - [Resource Hierarchy](#resource-hierarchy)
  - [Operator Workflow](#operator-workflow)
  - [Consumer API](#consumer-api)
  - [Bootstrap Configuration](#bootstrap-configuration)
  - [IAM and Audit Trail](#iam-and-audit-trail)
  - [Open Questions](#open-questions)
- [Incremental Path](#incremental-path)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)

## Summary

Datum Cloud has no mechanism to selectively enable features for a subset of
organizations before a full rollout. This enhancement defines an org-level
(and eventually project-level) boolean feature-flagging system built entirely
on the existing `quota.miloapis.com/v1alpha1` entitlement primitives — no new
CRD kinds, no binary changes, no third-party dependencies.

A feature flag is a `ResourceRegistration` with `type=Feature` (requires a
one-line Milo CRD prerequisite; see [Incremental Path](#incremental-path)).
Granting a flag to an organization is a `ResourceGrant` with `amount=1`.
Services check whether the flag is enabled through an
[OpenFeature](https://openfeature.dev)-compatible provider that queries the
auto-maintained `AllowanceBucket` for that (org, resourceType) pair, giving
all consumers a standard SDK interface regardless of the underlying quota API.

## Motivation

Product-led growth requires the ability to run private betas with a specific
set of customers before broad release. Today, enabling a feature for one
organization means enabling it for all. The team needs a way to gate features
per organization that:

- Requires no code change per new flag (data-driven)
- Is operator-managed initially, with self-serve as a future concern
- Produces a full audit trail through existing IAM logging
- Integrates with the Milo API server without bespoke machinery

### Goals

- Operators can define a feature flag by creating a `ResourceRegistration`
- Operators can grant or revoke a flag for a specific organization by
  creating or deleting a `ResourceGrant`
- Services (portal, API server) can determine whether an organization holds
  a feature flag with a single API read
- All grant/revoke operations are captured in the existing IAM audit log
- v1 handles the boolean on/off case without requiring the full
  quota/limit machinery (claims, admission enforcement)

### Non-Goals

- Self-serve flag requests by organization admins (v2)
- Project-scoped flags (v2; v1 targets org-level only)
- Tiered or quantity-based feature access, e.g., "5 GPU hours" (v3)
- Third-party feature flag services (LaunchDarkly, PostHog, etc.)
- Per-user feature flags (explicitly out of scope)

## Proposal

### User Stories

#### Story 1: Operator defines a new feature flag

As a Datum operator, I want to define a new boolean feature flag so that I
can later grant it to specific organizations for a private beta. I apply a
single `ResourceRegistration` YAML — the flag is immediately available to
grant, with no binary rollout required.

#### Story 2: Operator grants a feature to an organization

As a Datum operator, I want to enable a feature for a specific organization
during a private beta. I apply a `ResourceGrant` targeting that organization.
The organization's `AllowanceBucket` reflects the grant within one
reconciliation cycle, and the portal begins rendering the gated feature for
that org.

#### Story 3: Operator revokes access

As a Datum operator, I want to end a private beta for an organization. I
delete the `ResourceGrant`. The `AllowanceBucket` `status.available` drops to
zero, and the portal stops rendering the gated feature. The deletion is
recorded in the IAM audit log.

#### Story 4: Service checks whether a feature is enabled

As the Datum portal or API server, I want to check whether an organization
has a specific feature enabled before rendering UI or serving an API call. I
call `client.BooleanValue("flag-name", false, ctx)` on the OpenFeature client;
the provider resolves the check against `AllowanceBucket` with no per-request
authentication hop required.

#### Story 5: Organization admin observes their features

As an organization admin, I want to see which features are enabled on my
organization. I list `AllowanceBuckets` scoped to my organization using my
existing `organization-quota-manager` role permissions.

### Notes and Constraints

The `quota.miloapis.com/v1alpha1` API group is already deployed and
operational. All six resource kinds (`ResourceRegistration`, `ResourceGrant`,
`AllowanceBucket`, `ResourceClaim`, `ClaimCreationPolicy`,
`GrantCreationPolicy`) and their CRDs are present in
`config/crd/overlays/core-control-plane/kustomization.yaml`. No CRD
deployment work is required for v1.

Feature checks are read-path only. The quota admission plugin is not
exercised for feature flags — services consult the quota API before rendering
UI or serving calls, but no admission webhook enforces the check.

The `organization-quota-manager` IAM role already grants organization admins
read access to `ResourceGrants` and `AllowanceBuckets` in their organization's
scope, so org admins can observe their entitlements without any IAM changes.

### Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| `AllowanceBucket` eventual consistency causes stale feature checks | Low (sub-second reconciliation lag) | Low (brief incorrect state during grant/revoke) | Document; use `ResourceGrant` query for authoritative checks when needed |
| `claimingResources=[]` fails `ResourceRegistration` validation | Confirmed | Resolved | `minItems: 1` is enforced; use sentinel `features.miloapis.com/FeatureGrant` to satisfy the constraint without enabling admission enforcement |
| Feature flag namespace pollution (many `ResourceRegistrations`) | Medium (grows with feature count) | Low | Labeling convention (`features.miloapis.com/feature-name`) enables easy filtering and management |
| Operator accidentally grants wrong organization | Low | Medium | Naming convention `feature-<flag>-<org>` for `ResourceGrant` names makes grants easy to audit and identify |

## Design Details

### Conceptual Model

Feature-flagging is a degenerate case of the existing quota system. The
mapping is exact:

| Quota concept | Feature-flag meaning |
|---|---|
| `ResourceRegistration` | Feature flag definition |
| `ResourceGrant` with `amount=1` | "Organization X has feature Y enabled" |
| `AllowanceBucket.status.available > 0` | "Feature Y is on for organization X" |
| Deleting the `ResourceGrant` | "Revoke feature Y from organization X" |

`limit=1` means enabled; no grant (or `limit=0`) means disabled. The
`ResourceClaim` and `ClaimCreationPolicy` machinery is not used in v1 —
no admission enforcement is applied.

### Resource Hierarchy

```
ResourceRegistration (cluster-scoped)
  name: feature-<feature-name>
  spec.consumerType: Organization (resourcemanager.miloapis.com)
  spec.type: Feature   # requires Milo prerequisite; see Incremental Path
  spec.resourceType: features.miloapis.com/<feature-name>
  spec.baseUnit: feature
  spec.displayUnit: features
  spec.unitConversionFactor: 1
  spec.claimingResources:      # sentinel satisfies minItems=1; no admission enforcement
    - apiGroup: features.miloapis.com
      kind: FeatureGrant

ResourceGrant (cluster-scoped)
  spec.consumerRef:
    apiGroup: resourcemanager.miloapis.com
    kind: Organization
    name: <org-name>
  spec.allowances:
    - resourceType: features.miloapis.com/<feature-name>
      buckets:
        - amount: 1

# Auto-created by quota system when grant becomes Active:
AllowanceBucket (cluster-scoped)
  spec.consumerRef: (mirrors grant)
  spec.resourceType: features.miloapis.com/<feature-name>
  status.limit: 1
  status.available: 1   # → feature is enabled
```

### Operator Workflow

#### Step 1: Define a feature flag (once per feature)

```yaml
apiVersion: quota.miloapis.com/v1alpha1
kind: ResourceRegistration
metadata:
  name: feature-private-beta-gpu-inference
  labels:
    app.kubernetes.io/name: milo
    app.kubernetes.io/component: feature-flags
    features.miloapis.com/feature-name: private-beta-gpu-inference
spec:
  consumerType:
    apiGroup: resourcemanager.miloapis.com
    kind: Organization
  type: Feature
  resourceType: features.miloapis.com/private-beta-gpu-inference
  description: "Grants access to the GPU inference private beta"
  baseUnit: feature
  displayUnit: features
  unitConversionFactor: 1
  claimingResources:
    - apiGroup: features.miloapis.com
      kind: FeatureGrant   # sentinel; satisfies minItems=1, no admission enforcement
```

#### Step 2: Grant the feature to an organization

```yaml
apiVersion: quota.miloapis.com/v1alpha1
kind: ResourceGrant
metadata:
  name: feature-private-beta-gpu-inference-acme-corp
  labels:
    app.kubernetes.io/name: milo
    app.kubernetes.io/component: feature-flags
    features.miloapis.com/feature-name: private-beta-gpu-inference
    features.miloapis.com/org: acme-corp
spec:
  consumerRef:
    apiGroup: resourcemanager.miloapis.com
    kind: Organization
    name: acme-corp
  allowances:
    - resourceType: features.miloapis.com/private-beta-gpu-inference
      buckets:
        - amount: 1
```

#### Step 3: Revoke

```bash
kubectl delete resourcegrant feature-private-beta-gpu-inference-acme-corp
```

### Consumer API

Services check feature flags through an
[OpenFeature](https://openfeature.dev)-compatible provider backed by the
`AllowanceBucket` API. OpenFeature is a CNCF-incubating vendor-neutral
standard for feature flag evaluation; by shipping a compliant provider,
every service gets a typed, testable, and swappable feature-check interface
without hand-rolling quota API queries.

**Provider contract**

The provider implements `resolveBooleanValue(flagKey, default, EvaluationContext)`:

- `flagKey` maps to `features.miloapis.com/<flagKey>`
- `EvaluationContext.targetingKey` carries the org name
- Resolution queries `AllowanceBuckets` by field selector on
  `spec.consumerRef.name` + `spec.resourceType`
- Returns `true` (reason `TARGETING_MATCH`) if `status.available > 0`;
  `false` (reason `DEFAULT`) on miss or error

**Underlying query**

```
GET /apis/quota.miloapis.com/v1alpha1/allowancebuckets
  ?fieldSelector=spec.consumerRef.name=acme-corp,spec.resourceType=features.miloapis.com/private-beta-gpu-inference
```

Field selector indexes on `spec.consumerRef.kind`, `spec.consumerRef.name`,
and `spec.resourceType` already exist in the quota system.

**Usage example (Go)**

```go
client := openfeature.NewClient("my-service")
enabled, _ := client.BooleanValue(ctx, "private-beta-gpu-inference", false,
    openfeature.NewEvaluationContext("acme-corp", nil))
```

**Usage example (TypeScript)**

```ts
const enabled = await client.getBooleanValue('private-beta-gpu-inference', false, {
  targetingKey: 'acme-corp',
});
```

**Deliverables**

- Go provider package (e.g. `pkg/featureflags`) — wraps `AllowanceBucket`
  list query; unit tests with fake lister; integration test via `envtest`
- TypeScript provider (e.g. `app/lib/feature-flags/`) — wraps the quota API
  field selector endpoint; short TTL cache (5 s); unit tests with mocked fetch

Both providers expose only `resolveBooleanValue` in v1. String, number, and
structure resolution are non-goals until v2 introduces tiered access.

### Bootstrap Configuration

The quota CRDs are already deployed. The Milo bundle needs one new component:

```
config/services/features/
  kustomization.yaml
  registrations/
    kustomization.yaml
    # One ResourceRegistration per feature flag
  iam/
    kustomization.yaml
    roles/
      feature-flag-operator.yaml
```

This follows the pattern of `config/services/quota/`. No changes to
`config/crd/` are needed.

The `datum-cloud/infra` repository does not need changes for v1 beyond
consuming the updated Milo bundle.

### IAM and Audit Trail

Feature flag operations are standard Milo API operations covered by existing
IAM audit logging. `ResourceGrant` create/delete events are logged with full
actor identity and timestamp.

A new `feature-flag-operator` Role (or an extension of `quota-operator`)
should grant:

- `quota.miloapis.com/resourceregistrations`: `get`, `list`, `watch`
- `quota.miloapis.com/resourcegrants`: `get`, `list`, `watch`, `create`,
  `update`, `delete`
- `quota.miloapis.com/allowancebuckets`: `get`, `list`, `watch`

### Open Questions

**Resolved**

- **`claimingResources` validation**: `minItems: 1` is enforced in the live
  CRD (`quota.miloapis.com_resourceregistrations.yaml`) and the field is
  required — an empty list is rejected. Feature flag registrations must
  include at least one sentinel claiming resource. The proposed sentinel is
  `features.miloapis.com/FeatureGrant` (a non-existent kind), which satisfies
  the schema without enabling any admission enforcement. The Resource Hierarchy
  and operator workflow examples use this sentinel.
- **Consumer check strategy**: `AllowanceBucket` field selector query is the
  standard read path, codified behind an OpenFeature provider (Go + TypeScript)
  so callers never query the quota API directly. See [Consumer API](#consumer-api).
- **Bundle vs. infra split**: `ResourceRegistration` instances live in the
  Milo bundle — flags are global product decisions. Per-cluster variation is
  handled by creating or deleting `ResourceGrant` objects in `datum-cloud/infra`,
  not by varying registrations.
- **IAM role placement**: Dedicated `feature-flag-operator` role. Keeps
  least-privilege separation; `quota-operator` is not extended.
- **Naming convention**: `features.miloapis.com/<name>` confirmed as the
  `resourceType` pattern — consistent with existing API group conventions.
- **Read permission for org admins**: `organization-quota-manager` already
  grants read access to `AllowanceBuckets` in org scope. No IAM changes needed.
- **`spec.type` value**: The live `ResourceRegistration` CRD only accepts
  `Entity` and `Allocation` today. `Feature` requires a one-line Milo change
  (`+kubebuilder:validation:Enum=Entity;Allocation;Feature` in
  `pkg/apis/quota/v1alpha1/resourceregistration_types.go` + code-gen re-run).
  This is tracked as a prerequisite in the Incremental Path below. Because
  `spec.type` is immutable after creation, this Milo change must land before
  any `ResourceRegistration` instances are created; using `type=Entity` and
  migrating later would require deleting and recreating every flag definition.

## Incremental Path

### Prerequisite — Add `type=Feature` to Milo CRD

Add `Feature` to the `+kubebuilder:validation:Enum` marker in
`pkg/apis/quota/v1alpha1/resourceregistration_types.go` and re-run
`controller-gen` to regenerate the CRD YAML. One-line Go change plus
code-gen; no controller logic affected. Must land in Milo before any
`ResourceRegistration` instances are created (field is immutable).

### v1 — Boolean on/off, operator-managed

- Add `ResourceRegistration` instances (`type=Feature`) to
  `config/services/features/registrations/`
- Operators create/delete `ResourceGrant` objects via kubectl or the Milo API
- Services check feature flags through the OpenFeature provider (Go +
  TypeScript); provider queries `AllowanceBucket.status.available > 0`
- IAM: new `feature-flag-operator` role or extension of `quota-operator`
- No new controllers or CRDs beyond the Milo prerequisite above

Estimated complexity: **low** — Milo one-liner, configuration, a helper IAM
role, and two thin OpenFeature provider packages.

### v2 — Self-serve requests and project-scoped flags

- Add `GrantCreationPolicy` for self-serve grant creation workflows
- Add `ClaimCreationPolicy` if admission-time enforcement is desired
- Support `consumerType=Project` for project-scoped flags

### v3 — Tiered and quantity-based feature access

- Use `spec.type=Allocation` with non-binary amounts (e.g., 5 GPU hours)
- Full quota/limit machinery engaged

## Implementation History

- 2026-04-17: Discovery complete; confirmed zero-code v1 path using existing
  quota primitives. Enhancement document created.
- 2026-04-24: Consumer check strategy resolved — OpenFeature provider
  (Go + TypeScript) wrapping `AllowanceBucket` field selector query adopted
  as standard pattern. `type=Feature` CRD prerequisite identified and added
  to incremental path; `spec.type` immutability makes this a hard pre-req
  before any flag instances are created.
- 2026-04-27: `claimingResources` blocking question resolved — `minItems: 1`
  is enforced by the live CRD; sentinel `features.miloapis.com/FeatureGrant`
  adopted to satisfy the constraint without enabling admission enforcement.
  All remaining open questions resolved: registrations in Milo bundle,
  dedicated `feature-flag-operator` role, `features.miloapis.com/<name>`
  naming confirmed, org admin read permissions already covered by
  `organization-quota-manager`. Enhancement ready for implementation.

## Drawbacks

Using the quota system for feature flags couples two concerns: resource
consumption tracking and feature entitlement. If the quota system evolves in
ways that affect `ResourceRegistration` semantics, feature flags may be
incidentally affected. A dedicated `FeatureFlag` CRD kind would be more
explicit but would require new controllers and binary changes — the quota
reuse is the explicit tradeoff for keeping v1 zero-code.

## Alternatives

### Dedicated FeatureFlag CRD

A new `FeatureFlag` and `FeatureGrant` kind would be more semantically clear
and would not depend on quota system internals. Rejected for v1 because it
requires new CRD installation, new controllers, and binary changes — all of
which the quota-based approach avoids.

### Labels on Organization objects

Operators could enable features by setting labels on `Organization` resources
(e.g., `features.miloapis.com/gpu-inference: enabled`). Rejected because
labels on business objects are not access-controlled at the field level,
produce no structured audit trail, and cannot be discovered without
listing all organizations.

### Custom client SDK (roll-your-own)

Rather than implementing an OpenFeature provider, services could use a
bespoke Datum feature-flag client that wraps the quota API directly. Rejected
because it produces a Datum-specific interface that every service must adopt
independently, has no built-in mock/no-op for tests, and would need to be
replaced if the backend ever changes. OpenFeature provides all of this for
free at the cost of implementing one provider interface.

### Third-party feature flag service

LaunchDarkly, PostHog, GrowthBook, and similar services provide feature
flagging as a managed product. Rejected because they introduce an external
runtime dependency, require data to leave the platform, and create a
parallel identity/authorization system outside Milo's IAM.
