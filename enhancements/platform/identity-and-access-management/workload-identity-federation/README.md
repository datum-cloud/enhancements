---
status: provisional
stage: alpha
latest-milestone: "v0.1"
---

<!-- omit from toc -->
# Workload Identity Federation

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [How It Works](#how-it-works)
  - [User Stories](#user-stories)
  - [Supported Platforms](#supported-platforms)
  - [Security](#security)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Resource Model](#resource-model)
  - [Subject Resolution](#subject-resolution)
  - [Verification and Observability](#verification-and-observability)
  - [Authentication Flow](#authentication-flow)
  - [Authorization Integration](#authorization-integration)
  - [Attribute-Based Access Control](#attribute-based-access-control)
- [Future Work](#future-work)
- [Dependencies](#dependencies)
- [Alternatives](#alternatives)

## Summary

Workload Identity Federation enables external platforms (GitHub Actions, GCP,
AWS, Azure) to authenticate to Milo using their native OIDC tokens instead of
long-lived credentials. Platform administrators register trusted OIDC issuers,
and project owners configure trust relationships through WorkloadIdentityPool,
WorkloadIdentityPoolIssuer, and WorkloadIdentityPoolRule resources. Federated
identities become subjects in PolicyBindings using principal URI strings.

This approach eliminates credential management burden, removes the need to store
secrets in CI/CD systems, and provides automatic token expiration with clear
audit attribution.

## Motivation

Many teams need external platforms to access their Milo projects for CI/CD
pipelines, automation, and third-party integrations. Common scenarios include:

- **GitHub Actions workflows** deploying resources to a project
- **Cloud platform workloads** (GCP Cloud Functions, AWS Lambda) calling Milo
  APIs
- **Third-party services** like Grafana reading project metrics

Today, these use cases require creating MachineAccounts with long-lived
MachineAccountKeys. This approach has significant problems:

- **Credential management burden**: Teams must generate, store, rotate, and
  revoke keys manually
- **Secret exposure risk**: Keys stored in CI/CD systems can leak through logs,
  breaches, or misconfiguration
- **Audit difficulty**: Attributing actions to specific CI/CD runs is harder
  with shared credentials
- **No automatic expiration**: Keys remain valid until manually revoked,
  creating persistent risk

Workload Identity Federation solves these problems by letting external platforms
authenticate using their native short-lived OIDC tokens. A GitHub Actions
workflow uses its job token; a GCP Cloud Function uses its service identity
token. No long-lived credentials are shared or stored.

### Goals

- Enable external workloads to authenticate using native OIDC tokens
- Provide project owners with control over which external identities can access
  their projects
- Support attribute-based access control using OIDC token claims (repository,
  branch, actor)
- Integrate with existing PolicyBinding authorization model
- Support GitHub Actions, GCP, AWS, and Azure as initial platforms

### Non-Goals

- Replacing user authentication (human users continue using existing auth)
- Replacing [Platform Workload Identity][platform-workload-identity] for
  internal service-to-service authentication
- Organization-level identity pools (project-scoped only for v1)
- Non-OIDC federation protocols (SAML, X.509)
- Custom OIDC providers with arbitrary issuers (security risk)

## Proposal

Workload Identity Federation introduces one platform-level resource and three
project-level resources. Trust in an external OIDC issuer and the conditions
under which a token from that issuer is accepted are configured separately,
so a single issuer registration can back many different access levels without
duplicating issuer configuration:

- **TrustedIssuer** *(platform-level)*: Registers an OIDC issuer's identity
  (issuer URL and JWKS source) as safe to federate against. Managed by
  platform administrators. v1 ships TrustedIssuers for the platforms in
  [Supported Platforms](#supported-platforms); project-defined custom issuers
  are [Future Work](#future-work).
- **WorkloadIdentityPool** *(project-level, optional)*: Groups related issuers
  and rules for bulk enable/disable. Projects that don't need grouping can
  omit it; ungrouped resources are placed in an implicit `default` pool so the
  principal URI shape never changes.
- **WorkloadIdentityPoolIssuer** *(project-level)*: Enables a TrustedIssuer
  for use within a project's pool.
- **WorkloadIdentityPoolRule** *(project-level)*: Matches tokens from a
  specific issuer against audience, claim, and CEL attribute conditions, and
  maps matching tokens to a principal URI.

When an external workload presents an OIDC token, Milo validates it against
matching rules and constructs a principal URI. This principal URI becomes a
subject in PolicyBindings, granting the federated identity access to project
resources.

### How It Works

**Setup (one-time):**

1. Platform administrator registers a TrustedIssuer (already done for
   supported platforms in v1)
2. Project owner creates a WorkloadIdentityPoolIssuer enabling that
   TrustedIssuer within the project (optionally within a named
   WorkloadIdentityPool)
3. Project owner creates one or more WorkloadIdentityPoolRules specifying
   audience, claim, and attribute conditions
4. Project owner creates a PolicyBinding granting the resulting principal
   access to resources
5. Project owner optionally verifies the setup end-to-end (see
   [Verification and Observability](#verification-and-observability))

**Runtime (every request):**

1. External workload requests OIDC token from its platform (automatic in GitHub
   Actions)
2. Workload calls Milo API with token in Authorization header
3. Milo validates token against matching rules for the token's issuer
4. Milo constructs principal URI and checks PolicyBinding authorization
5. Request proceeds with federated identity context

### User Stories

#### Deploy from GitHub Actions

As a developer, I want my GitHub Actions workflow to deploy resources to my Milo
project without storing credentials as secrets.

**Experience:**

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    permissions:
      id-token: write  # Request OIDC token
    steps:
      - uses: datum-cloud/auth-action@v1
        with:
          project: my-project
          pool: ci-cd-pool
          rule: github-actions-main
      - run: datumctl apply -f manifests/
```

The workflow authenticates using its native GitHub OIDC token. No secrets
configuration required.

#### Restrict Access by Branch

As a team lead, I want to grant deploy access only to workflows running on the
main branch, while allowing any branch to run read-only operations.

**Experience:** Create one WorkloadIdentityPoolIssuer for GitHub Actions, then
two WorkloadIdentityPoolRules against it with different attribute conditions:

- `github-actions-main`: Condition requires `attribute.ref ==
  "refs/heads/main"` → bound to deployer role
- `github-actions-all`: No branch restriction → bound to viewer role

Both rules reuse the same issuer registration — no duplicate issuer or JWKS
configuration. Workflows on feature branches can view resources but cannot
deploy.

#### Verify a New Rule Before Relying On It

As a developer, I want to confirm my WorkloadIdentityPoolRule is configured
correctly before I point a real CI/CD pipeline at it.

**Experience:** After creating the issuer and rule, select **Test rule** in
the console. Milo listens for a live token exchange matching that rule for 15
minutes. Trigger a workflow run (or a manual `datumctl` federated login) within
that window; the console shows the exchange succeeding or failing in real
time, including the specific reason for a failure (signature invalid, audience
mismatch, condition not satisfied). If the window elapses without an attempt,
the rule persists and the test can be re-run from the rule's detail page.

#### Authenticate from GCP Cloud Functions

As a developer, I want my GCP Cloud Function to call Milo APIs using its service
identity without embedding credentials in code.

**Experience:** Create a WorkloadIdentityPoolIssuer referencing the GCP
TrustedIssuer, then a rule with a condition matching the service account
email. The Cloud Function authenticates using its native identity token.

#### Revoke Access Immediately

As a security engineer, I want to immediately revoke access for a compromised
workflow without waiting for credential expiration.

**Experience:** Delete the PolicyBinding or disable the
WorkloadIdentityPoolRule. Access is denied immediately on the next request,
even though the OIDC token may still be valid.

#### Audit CI/CD Activity

As a compliance officer, I want to see which CI/CD workflows accessed my project
and what actions they performed.

**Experience:** View the Activity timeline filtered by principal. Each action
shows the principal URI, which encodes the pool and rule. Cross-reference with
CI/CD logs to identify specific workflow runs. For authentication attempts
that never resulted in an action — including failed ones — see the rule's
Authentication History (see
[Verification and Observability](#verification-and-observability)).

### Supported Platforms

Initial support targets platforms with mature OIDC token support. Each is
pre-registered as a platform-level TrustedIssuer:

| Platform | Issuer | Key Claims |
|----------|--------|------------|
| **GitHub Actions** | `https://token.actions.githubusercontent.com` | `repository`, `repository_owner`, `ref`, `workflow`, `actor` |
| **GCP** | `https://accounts.google.com` | `email`, `sub` (service account) |
| **AWS** | `https://sts.amazonaws.com` | Role ARN in `sub` |
| **Azure AD** | `https://login.microsoftonline.com/{tenant}/v2.0` | `sub`, `oid`, `appid` |

These platforms automatically provide OIDC tokens to workloads without
additional configuration. Project owners reference the corresponding
TrustedIssuer from a WorkloadIdentityPoolIssuer; they do not configure issuer
URLs or JWKS sources directly in v1.

### Security

#### Trust Model

- **Token validation**: Signature verified using JWKS from issuer
- **Issuer validation**: Token's `iss` claim must match the TrustedIssuer's
  configured issuer
- **Audience validation**: Token's `aud` claim must match configured audiences
- **Expiration**: Tokens older than 1 hour are rejected
- **Attribute conditions**: CEL expressions evaluated before authentication
  succeeds

#### No Credential Storage

Unlike MachineAccountKeys, no credentials are stored in Milo or external
systems. OIDC tokens are:

- **Short-lived**: Typically 10-60 minutes
- **Automatically provisioned**: Platforms issue tokens without user action
- **Cryptographically signed**: Cannot be forged without the issuer's private
  key

#### Revocation

Access can be revoked through multiple mechanisms:

| Mechanism | Effect | Latency |
|-----------|--------|---------|
| Delete PolicyBinding | Authorization denied | Immediate |
| Disable rule | Authentication rejected | Immediate |
| Delete rule | Authentication rejected | Immediate |
| Disable issuer | Authentication rejected for all rules on that issuer | Immediate |
| Disable pool | All issuers and rules in pool rejected | Immediate |

Unlike long-lived credentials, there's no need to rotate secrets or wait for
expiration.

### Notes/Constraints/Caveats

#### Project Control Plane Scope

WorkloadIdentityPool, WorkloadIdentityPoolIssuer, and WorkloadIdentityPoolRule
are cluster-scoped within a project's control plane. Since projects are
dedicated control planes in Milo, these resources are naturally project-scoped
without requiring namespaces. TrustedIssuer is the one exception: it is a
platform-level resource managed outside any project's control plane.

#### Pools Are Optional

WorkloadIdentityPool exists purely for grouping and bulk enable/disable.
WorkloadIdentityPoolIssuer and WorkloadIdentityPoolRule may omit `poolRef`
entirely, in which case they're placed in an implicit `default` pool scoped to
the project. This keeps the principal URI shape
(`principal://.../pools/{pool}/rules/{rule}`) stable whether or not a project
ever creates an explicit pool, and avoids requiring pool creation for the
common single-issuer, single-rule case.

#### Audiences Are a Rule-Level Match Condition

Allowed audiences are configured per WorkloadIdentityPoolRule, not on the
issuer. This lets two rules against the same issuer require different
audiences (for example, a stricter audience for a deployer rule than for a
read-only rule) without duplicating issuer configuration, and keeps audience
validation grouped with the rest of the rule's match logic (`claims`,
`attributeCondition`).

#### Attribute Conditions at Authentication Time

Attribute conditions (CEL expressions) are evaluated during authentication, not
authorization. This means:

- All tokens matching a rule share the same principal URI
- To grant different permissions based on attributes (e.g., branch), create
  separate rules against the same issuer

Future work may add binding-level conditions for finer-grained control.

#### JWKS Caching

JWKS (JSON Web Key Sets) are cached for 1 hour to reduce latency and external
dependencies. The cache is invalidated on signature validation failures,
ensuring key rotation is handled gracefully.

### Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| **Overly permissive conditions** | Unintended access granted | Conditions are required; validation rejects empty conditions |
| **JWKS endpoint unavailable** | New tokens cannot validate | Cache last-known-good JWKS; tokens cached during outage |
| **Token replay** | Same token reused maliciously | Short token lifetime (1 hour max); `exp` claim enforced |
| **Issuer spoofing** | Fake tokens accepted | JWKS fetched only from configured issuer over HTTPS; v1 issuers are limited to platform-vetted TrustedIssuers |
| **Misconfigured rule** | Authentication failures | Status conditions surface configuration errors; `Test rule` verifies end to end before relying on it |
| **Silent misconfiguration reaches production unnoticed** | Access unexpectedly denied (or, for overly broad conditions, granted) at runtime | Verify issuer dry-run and live rule test catch mistakes before a real workflow depends on them |

## Design Details

### Resource Model

#### TrustedIssuer *(platform-level)*

Registers an OIDC issuer's identity as safe to federate against, cluster-wide.
Managed by platform administrators, not project owners. v1 ships a
TrustedIssuer for each platform in [Supported Platforms](#supported-platforms).

```yaml
apiVersion: iam.miloapis.com/v1alpha1
kind: TrustedIssuer
metadata:
  name: github-actions
spec:
  displayName: "GitHub Actions"
  issuerUri: "https://token.actions.githubusercontent.com"
  jwks:
    source: discovery  # discovery | explicitUrl | inline
status:
  resolvedJwksUri: "https://token.actions.githubusercontent.com/.well-known/jwks"
  conditions:
    - type: Verified
      status: "True"
      lastVerifiedTime: "2026-07-10T12:00:00Z"
```

Separating issuer trust from match conditions means one TrustedIssuer backs
every project's WorkloadIdentityPoolRules for that platform — JWKS
configuration is fetched, verified, and rotated once, centrally, instead of
once per project per attribute condition.

#### WorkloadIdentityPool *(optional)*

Groups related issuers and rules within a project and provides a common point
of control (e.g., emergency disable). Omit `poolRef` on
WorkloadIdentityPoolIssuer or WorkloadIdentityPoolRule to use the project's
implicit `default` pool instead of creating one explicitly.

```yaml
apiVersion: iam.miloapis.com/v1alpha1
kind: WorkloadIdentityPool
metadata:
  name: ci-cd-pool
spec:
  displayName: "CI/CD Workload Pool"
  description: "Pool for CI/CD pipeline authentications"
  disabled: false  # Emergency disable switch
status:
  issuerCount: 1
  ruleCount: 2
  conditions:
    - type: Ready
      status: "True"
```

#### WorkloadIdentityPoolIssuer

Enables a TrustedIssuer for use within a project's pool. Holds no trust
configuration of its own in v1 — issuer identity and JWKS come entirely from
the referenced TrustedIssuer. This indirection is what lets a project
participate in a platform-level issuer without redeclaring its configuration,
and leaves room for project-scoped custom issuers (see [Future
Work](#future-work)) to slot into the same shape later.

```yaml
apiVersion: iam.miloapis.com/v1alpha1
kind: WorkloadIdentityPoolIssuer
metadata:
  name: github-actions
spec:
  poolRef:
    name: ci-cd-pool  # optional; omitted uses the project's default pool
  trustedIssuerRef:
    name: github-actions
status:
  conditions:
    - type: Ready
      status: "True"
```

#### WorkloadIdentityPoolRule

Matches tokens from a WorkloadIdentityPoolIssuer against audience, claim, and
attribute conditions, and defines the resulting principal's attribute mapping.
Multiple rules can reference the same issuer, each with independent match
conditions and audiences — this is how the [Restrict Access by
Branch](#restrict-access-by-branch) story avoids duplicating issuer
configuration per branch policy.

```yaml
apiVersion: iam.miloapis.com/v1alpha1
kind: WorkloadIdentityPoolRule
metadata:
  name: github-actions-main
spec:
  issuerRef:
    name: github-actions

  displayName: "GitHub Actions - Main Branch"

  allowedAudiences:
    - "https://api.miloapis.com"

  # Map token claims to attributes
  attributeMapping:
    attribute.repository: "assertion.repository"
    attribute.repository_owner: "assertion.repository_owner"
    attribute.ref: "assertion.ref"
    attribute.actor: "assertion.actor"

  # CEL expression: tokens must satisfy this condition
  attributeCondition: |
    attribute.repository == "acme-corp/infrastructure" &&
    attribute.ref == "refs/heads/main"

status:
  principalIdentifier: "principal://iam.miloapis.com/projects/my-project/pools/ci-cd-pool/rules/github-actions-main"
  conditions:
    - type: Ready
      status: "True"
    - type: Verified          # flips to "True" once a live test exchange succeeds
      status: "True"
      reason: TestExchangeSucceeded
      lastAuthenticationTime: "2026-07-15T09:04:22Z"
```

### Subject Resolution

Hand-typing a principal URI into a PolicyBinding's `subjects` list is
error-prone — a typo silently produces a binding that never matches. Instead,
`PolicyBinding` accepts a typed reference to a `WorkloadIdentityPoolRule` as
sugar for the URI:

```yaml
apiVersion: iam.miloapis.com/v1alpha1
kind: PolicyBinding
metadata:
  name: github-deployer
spec:
  roleRef:
    name: project-editor
  subjects:
    - kind: WorkloadIdentityPoolRule
      name: github-actions-main
  resourceSelector:
    resourceKind:
      apiGroup: resourcemanager.miloapis.com
      kind: Project
```

This is not new coupling: `PolicyBinding` already resolves typed subject
references for `User` and `ServiceAccount` into the identifier stored as the
OpenFGA subject tuple. Adding `WorkloadIdentityPoolRule` is one more entry in
that existing resolution path, not a new mechanism.

Resolution is a compiled registry (`GroupKind` → subject template), not a
runtime-editable policy: a misresolved subject fails *open* — it grants the
binding to the wrong identity — where a misconfigured `attributeCondition`
only fails *closed*, so this mapping stays code-reviewed rather than
cluster-admin-configurable. The resolved string is also written to the
referenced rule's `status.principalIdentifier`, so it's computed once and
discoverable from the rule itself, whether a PolicyBinding references the rule
directly or a user copies the URI by hand.

A resolved reference validates only that the rule *exists*, not that any
workload will ever present a token matching it — that gap is inherent to
federated identity and is why [Verification and
Observability](#verification-and-observability) exists as a separate check.

### Verification and Observability

A WorkloadIdentityPoolIssuer or WorkloadIdentityPoolRule can be syntactically
valid yet never match a real token (wrong audience, a typo in an attribute
condition, an unreachable JWKS endpoint). The console gives project owners two
pre-flight checks and one ongoing view:

| Feature | What it does | Where it surfaces |
|---|---|---|
| **Verify issuer** | Dry-run: fetches and parses the JWKS from the configured source, without creating or modifying anything. Available on WorkloadIdentityPoolIssuer and, for platform admins, on TrustedIssuer. | `Verified` condition in `status.conditions` |
| **Test rule** | Opens a 15-minute live window on a WorkloadIdentityPoolRule. Trigger a real exchange (a CI run, a manual `datumctl` federated login) within the window and the console reports success or the specific failure reason (signature invalid, audience mismatch, condition not satisfied) in real time. Re-runnable anytime from the rule's detail page if the window elapses unused. | `status.testWindow.expiresAt`, `Verified` condition |
| **Authentication history** | Per-rule (and per-issuer, per-pool) log of every authentication *attempt*, success or failure — distinct from the Activity timeline ([Audit CI/CD Activity](#audit-cicd-activity)), which only records actions by an *already-authenticated* principal and so never sees a failed exchange. Lets a project owner tell "never tried" from "tried and was rejected," and spot a rule under sustained failed attempts (key rotation, claim drift, or misuse). | Rule detail page, `status.lastAuthenticationTime` |

### Authentication Flow

```
┌─────────────┐                  ┌──────────────────┐                  ┌─────────────┐
│   GitHub    │                  │   Milo API       │                  │   GitHub    │
│   Actions   │                  │   Server         │                  │   OIDC      │
└─────────────┘                  └──────────────────┘                  └─────────────┘
       │                                  │                                     │
       │  1. Request OIDC token           │                                     │
       │────────────────────────────────────────────────────────────────────────>│
       │                                  │                                     │
       │  2. OIDC token (signed JWT)      │                                     │
       │<────────────────────────────────────────────────────────────────────────│
       │                                  │                                     │
       │  3. API request                  │                                     │
       │     Authorization: Bearer <jwt>  │                                     │
       │─────────────────────────────────>│                                     │
       │                                  │                                     │
       │                                  │  4. Fetch JWKS (cached from TrustedIssuer) │
       │                                  │────────────────────────────────────>│
       │                                  │                                     │
       │                                  │  5. Find rules for matching issuer  │
       │                                  │  6. Validate signature, claims,     │
       │                                  │     audience against each rule      │
       │                                  │  7. Extract attributes              │
       │                                  │  8. Evaluate CEL condition          │
       │                                  │  9. Construct principal URI         │
       │                                  │ 10. Check PolicyBinding (OpenFGA)   │
       │                                  │ 11. Record authentication attempt   │
       │                                  │     (Authentication History)        │
       │                                  │                                     │
       │  12. API response                │                                     │
       │<─────────────────────────────────│                                     │
```

### Authorization Integration

Federated identities become subjects in PolicyBindings, either as a typed
`WorkloadIdentityPoolRule` reference (see [Subject
Resolution](#subject-resolution), the recommended form) or as the raw
principal URI string it resolves to:

```yaml
apiVersion: iam.miloapis.com/v1alpha1
kind: PolicyBinding
metadata:
  name: github-deployer
spec:
  roleRef:
    name: project-editor
  subjects:
    - kind: Principal
      name: "principal://iam.miloapis.com/projects/my-project/pools/ci-cd-pool/rules/github-actions-main"
  resourceSelector:
    resourceKind:
      apiGroup: resourcemanager.miloapis.com
      kind: Project
```

The principal URI format follows GCP's proven pattern, with `rules` in place
of GCP's `providers` since issuer trust and match conditions are separate
resources here:

```
principal://iam.miloapis.com/projects/{project}/pools/{pool}/rules/{rule}
```

Including the project in the URI ensures principals are globally unique across
the platform. This format is extensible for future enhancements like
`principalSet://` for attribute-based principal matching.

### Attribute-Based Access Control

CEL (Common Expression Language) provides attribute-based access control at
authentication time:

- **Single repository**: `attribute.repository == "acme-corp/app"`
- **Organization-wide**: `attribute.repository.startsWith("acme-corp/")`
- **Main branch only**: `attribute.ref == "refs/heads/main"`
- **Release tags**: `attribute.ref.startsWith("refs/tags/v")`
- **Specific actor**: `attribute.actor == "deploy-bot"`
- **Combined**: `attribute.repository == "acme-corp/app" && attribute.ref == "refs/heads/main"`

Conditions are required—rules without conditions are rejected during
validation.

## Future Work

The MVP focuses on OIDC federation for CI/CD platforms. Future phases will
expand based on customer feedback:

**Binding-level conditions:**

- CEL conditions on PolicyBinding subjects for finer-grained authorization
- Requires OpenFGA integration design
- Enables different permissions based on attributes without multiple rules

**Project-defined custom issuers:**

- Let project owners register a WorkloadIdentityPoolIssuer with its own
  `issuerUri`/JWKS instead of only referencing a platform TrustedIssuer
- Lifts the v1 restriction to platform-vetted issuers; requires the security
  guardrails called out in the original Non-Goals (audience/subject
  constraints, reachability requirements) before it's safe to open up

**Organization-level pools:**

- Shared pools across projects within an organization
- Centralized trust management for platform teams
- Partially addressed by TrustedIssuer, which already shares issuer trust
  platform-wide; this item is about sharing pools/rules, not just issuers

**Additional platforms:**

- Grafana and other SaaS provider integrations
- SAML and X.509 provider types

**Enhanced matching:**

- `principalSet://` URIs for attribute-based principal matching (GCP pattern)
- Wildcard pool matching

## Dependencies

Workload Identity Federation builds on existing platform services:

- **IAM**: PolicyBindings grant permissions to federated identities, and
  resolve `WorkloadIdentityPoolRule` subject references via the [subject
  resolver registry](#subject-resolution)
- **Activity**: Successful authentication events appear in the audit timeline,
  attributed to the resulting principal
- **Authentication history**: Records every authentication *attempt* — including
  failures, which never produce a principal and so never reach Activity — for
  the console's [Verification and Observability](#verification-and-observability)
  views
- **OpenFGA**: Authorization checks use principal URIs as subjects

External dependencies:

- **Identity provider OIDC endpoints**: JWKS and token issuance
- **CEL library**: Expression evaluation (`github.com/google/cel-go`)
- **JWT library**: Token parsing and validation (`github.com/golang-jwt/jwt/v5`)

## Alternatives

### Per-Consumer Credentials (Current Approach)

Create MachineAccounts with MachineAccountKeys for each external platform.

**Rejected because:**

- Credential management burden (rotation, storage, revocation)
- Long-lived secrets create persistent security risk
- Audit attribution is difficult with shared credentials

### OAuth Client Credentials

External platforms obtain tokens via OAuth client credentials flow.

**Rejected because:**

- Still requires managing client secrets
- More complex than native OIDC token support
- Platforms already provide OIDC tokens automatically

### Coupled Issuer and Match Conditions (Single Provider Resource)

Fold issuer trust (issuer URL, JWKS) and match conditions (audience, claims,
attribute condition) into one resource per rule, as GCP's
`WorkloadIdentityPoolProvider` does.

**Rejected because:**

- Two access levels against one issuer (e.g., [Restrict Access by
  Branch](#restrict-access-by-branch)) require two full resources, each
  redeclaring the same issuer URL and JWKS source
- Rotating an issuer's JWKS means updating every resource that uses it,
  instead of once
- Anthropic's WIF ships the decoupled alternative in production: one
  `Federation Issuer` backs many independent `Federation Rules`

### Federation Rule Targets a Service Account (Anthropic Pattern)

Anthropic's federation rules target a pre-provisioned service account; the
minted token carries that account's workspace role, the same authorization
surface as a static API key.

**Rejected because:**

- Requires an account object per access level, reintroducing the
  per-consumer credential proliferation this enhancement exists to avoid
- Coarser-grained than `PolicyBinding` + principal URI, which already scopes
  access to a single Project
- Gains nothing authorization-wise: both a service account and a principal
  URI are just subjects a `PolicyBinding` can reference

### Service Mesh / SPIFFE

Use SPIFFE IDs and service mesh for workload identity.

**Rejected because:**

- Requires external platforms to participate in service mesh
- Additional infrastructure complexity
- OIDC is already the industry standard for CI/CD federation

<!-- References -->
[platform-workload-identity]: ../workload-identity/README.md
