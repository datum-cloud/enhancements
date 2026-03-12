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
  - [Authentication Flow](#authentication-flow)
  - [Authorization Integration](#authorization-integration)
  - [Attribute-Based Access Control](#attribute-based-access-control)
- [Future Work](#future-work)
- [Dependencies](#dependencies)
- [Alternatives](#alternatives)

## Summary

Workload Identity Federation enables external platforms (GitHub Actions, GCP,
AWS, Azure) to authenticate to Milo using their native OIDC tokens instead of
long-lived credentials. Project owners configure trust relationships through
WorkloadIdentityPool and WorkloadIdentityPoolProvider resources, and federated
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

Workload Identity Federation introduces two resources that project owners use to
configure trust relationships with external identity providers:

- **WorkloadIdentityPool**: Groups related external identity providers
- **WorkloadIdentityPoolProvider**: Configures a specific OIDC issuer with
  attribute conditions

When an external workload presents an OIDC token, Milo validates it against
matching providers and constructs a principal URI. This principal URI becomes a
subject in PolicyBindings, granting the federated identity access to project
resources.

### How It Works

**Setup (one-time):**

1. Project owner creates a WorkloadIdentityPool to group related providers
2. Project owner creates a WorkloadIdentityPoolProvider specifying the OIDC
   issuer and attribute conditions
3. Project owner creates a PolicyBinding granting the principal access to
   resources

**Runtime (every request):**

1. External workload requests OIDC token from its platform (automatic in GitHub
   Actions)
2. Workload calls Milo API with token in Authorization header
3. Milo validates token against matching provider
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
          provider: github-actions-main
      - run: datumctl apply -f manifests/
```

The workflow authenticates using its native GitHub OIDC token. No secrets
configuration required.

#### Restrict Access by Branch

As a team lead, I want to grant deploy access only to workflows running on the
main branch, while allowing any branch to run read-only operations.

**Experience:** Create two providers with different attribute conditions:

- `github-actions-main`: Condition requires `attribute.ref ==
  "refs/heads/main"` → bound to deployer role
- `github-actions-all`: No branch restriction → bound to viewer role

Workflows on feature branches can view resources but cannot deploy.

#### Authenticate from GCP Cloud Functions

As a developer, I want my GCP Cloud Function to call Milo APIs using its service
identity without embedding credentials in code.

**Experience:** Create a provider for the GCP issuer with a condition matching
the service account email. The Cloud Function authenticates using its native
identity token.

#### Revoke Access Immediately

As a security engineer, I want to immediately revoke access for a compromised
workflow without waiting for credential expiration.

**Experience:** Delete the PolicyBinding or disable the
WorkloadIdentityPoolProvider. Access is denied immediately on the next request,
even though the OIDC token may still be valid.

#### Audit CI/CD Activity

As a compliance officer, I want to see which CI/CD workflows accessed my project
and what actions they performed.

**Experience:** View the Activity timeline filtered by principal. Each action
shows the principal URI, which encodes the pool and provider. Cross-reference
with CI/CD logs to identify specific workflow runs.

### Supported Platforms

Initial support targets platforms with mature OIDC token support:

| Platform | Issuer | Key Claims |
|----------|--------|------------|
| **GitHub Actions** | `https://token.actions.githubusercontent.com` | `repository`, `repository_owner`, `ref`, `workflow`, `actor` |
| **GCP** | `https://accounts.google.com` | `email`, `sub` (service account) |
| **AWS** | `https://sts.amazonaws.com` | Role ARN in `sub` |
| **Azure AD** | `https://login.microsoftonline.com/{tenant}/v2.0` | `sub`, `oid`, `appid` |

These platforms automatically provide OIDC tokens to workloads without
additional configuration.

### Security

#### Trust Model

- **Token validation**: Signature verified using JWKS from issuer
- **Issuer validation**: Token's `iss` claim must match provider's configured
  issuer
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
| Disable provider | Authentication rejected | Immediate |
| Delete provider | Authentication rejected | Immediate |
| Disable pool | All providers in pool rejected | Immediate |

Unlike long-lived credentials, there's no need to rotate secrets or wait for
expiration.

### Notes/Constraints/Caveats

#### Project Control Plane Scope

WorkloadIdentityPool and WorkloadIdentityPoolProvider are cluster-scoped within
a project's control plane. Since projects are dedicated control planes in Milo,
these resources are naturally project-scoped without requiring namespaces.

#### Attribute Conditions at Authentication Time

Attribute conditions (CEL expressions) are evaluated during authentication, not
authorization. This means:

- All tokens from a provider share the same principal URI
- To grant different permissions based on attributes (e.g., branch), create
  separate providers

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
| **Issuer spoofing** | Fake tokens accepted | JWKS fetched only from configured issuer over HTTPS |
| **Misconfigured provider** | Authentication failures | Status conditions surface configuration errors |

## Design Details

### Resource Model

#### WorkloadIdentityPool

Groups related identity providers and provides a common point of control (e.g.,
emergency disable).

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
  providerCount: 2
  conditions:
    - type: Ready
      status: "True"
```

#### WorkloadIdentityPoolProvider

Configures trust for a specific OIDC issuer with attribute mapping and
conditions.

```yaml
apiVersion: iam.miloapis.com/v1alpha1
kind: WorkloadIdentityPoolProvider
metadata:
  name: github-actions-main
spec:
  poolRef:
    name: ci-cd-pool

  displayName: "GitHub Actions - Main Branch"

  oidc:
    issuerUri: "https://token.actions.githubusercontent.com"
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
  principalIdentifier: "principal://iam.miloapis.com/projects/my-project/pools/ci-cd-pool/providers/github-actions-main"
  resolvedJwksUri: "https://token.actions.githubusercontent.com/.well-known/jwks"
  conditions:
    - type: Ready
      status: "True"
```

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
       │                                  │  4. Fetch JWKS (cached)             │
       │                                  │────────────────────────────────────>│
       │                                  │                                     │
       │                                  │  5. Validate signature, claims      │
       │                                  │  6. Extract attributes              │
       │                                  │  7. Evaluate CEL condition          │
       │                                  │  8. Construct principal URI         │
       │                                  │  9. Check PolicyBinding (OpenFGA)   │
       │                                  │                                     │
       │  10. API response                │                                     │
       │<─────────────────────────────────│                                     │
```

### Authorization Integration

Federated identities become subjects in PolicyBindings using principal URI
strings:

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
      name: "principal://iam.miloapis.com/projects/my-project/pools/ci-cd-pool/providers/github-actions-main"
  resourceSelector:
    resourceKind:
      apiGroup: resourcemanager.miloapis.com
      kind: Project
```

The principal URI format follows GCP's proven pattern:

```
principal://iam.miloapis.com/projects/{project}/pools/{pool}/providers/{provider}
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

Conditions are required—providers without conditions are rejected during
validation.

## Future Work

The MVP focuses on OIDC federation for CI/CD platforms. Future phases will
expand based on customer feedback:

**Binding-level conditions:**

- CEL conditions on PolicyBinding subjects for finer-grained authorization
- Requires OpenFGA integration design
- Enables different permissions based on attributes without multiple providers

**Organization-level pools:**

- Shared pools across projects within an organization
- Centralized trust management for platform teams

**Additional platforms:**

- Custom OIDC providers (user-defined issuers with security guardrails)
- Grafana and other SaaS provider integrations
- SAML and X.509 provider types

**Enhanced matching:**

- `principalSet://` URIs for attribute-based principal matching (GCP pattern)
- Wildcard pool matching

## Dependencies

Workload Identity Federation builds on existing platform services:

- **IAM**: PolicyBindings grant permissions to federated identities
- **Activity**: Authentication events appear in audit timeline
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

### Service Mesh / SPIFFE

Use SPIFFE IDs and service mesh for workload identity.

**Rejected because:**

- Requires external platforms to participate in service mesh
- Additional infrastructure complexity
- OIDC is already the industry standard for CI/CD federation

<!-- References -->
[platform-workload-identity]: ../workload-identity/README.md
