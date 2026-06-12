---
status: provisional
stage: alpha
latest-milestone: "v0.0"
---

# Internal AI Gateway

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [v1 Implementation Scope](#v1-implementation-scope)
  - [Architecture](#architecture)
  - [Resource Model](#resource-model)
  - [Authentication and Authorization](#authentication-and-authorization)
  - [Token Handling](#token-handling)
  - [Secret Management](#secret-management)
  - [Observability](#observability)
- [Roadmap](#roadmap)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)

## Summary

This enhancement formalizes an internal Envoy AI Gateway deployment in our staging environment, accessible to Datum staff. It routes traffic to two backends — Anthropic (hosted) and a Qwen model running on Datum's H100 compute — using the existing Envoy AI Gateway infra.

## Motivation

Staff members are hitting per-account token limits on Anthropic while using Claude Code for daily work. Each engineer holding a personal provider key is operationally noisy (rotation, billing aggregation) and produces no shared visibility into usage.

Beyond the immediate cost problem, Datum's product direction involves running and operating AI gateway infrastructure. Before making architectural commitments in that direction, we need first-hand experience: which upstream Envoy AI Gateway features are ready to use as-is, which gaps need work on top, and what operating the gateway against real LLM payloads actually looks like. Running it internally against staff workloads, with bounded blast radius, is the simplest way to get that data.

### Goals

- **Single SSO-gated endpoint** — one hostname, one identity provider (staging Zitadel), one org-owned provider quota shared across staff.
- **Two backends** — Anthropic via `AIServiceBackend` with `schema: Anthropic` and `BackendSecurityPolicy` of type `AnthropicAPIKey`; Qwen running on internal compute via an `AIServiceBackend` with an OpenAI-compatible schema.
- **JWT validation at the edge** — `SecurityPolicy` attached to the `HTTPRoute` so every path is gated; no unauthenticated bypass route.
- **Two interchangeable client flows** — Flow A (`datumctl auth get-token` via `apiKeyHelper`, suitable for any client that can shell out) and Flow B (native OIDC app with PKCE, for clients that speak OIDC directly).
- **Provider credentials never on client machines** — keys are held in GCP Secret Manager, synced into cluster via External Secrets + Workload Identity, injected upstream-only by `BackendSecurityPolicy`.
- **Upstream-only configuration** — every resource is a stock Envoy AI Gateway, Envoy Gateway, or Gateway API CRD. Not building on top in the first iteration.

### Non-Goals

- **Additional hosted providers** (OpenAI, Bedrock, Gemini) — the CRD model is additive so each is mechanical to add, but we want operating experience with the current two backends first. Tracked under [#391](https://github.com/datum-cloud/enhancements/issues/391).
- **Per-user rate limiting and quotas** — the shared org Anthropic account is the only quota in v1. Rate limits are expressible via `BackendTrafficPolicy` once per-user attribution is in place; deferring both together keeps the v1 surface minimal.
- **Prompt caching and model routing** — Envoy AI Gateway supports both; neither is configured in v1, as evaluating effectiveness requires usage data we don't yet have.
- **Production deployment** — this is a single staging cluster. Any production deployment will be its own enhancement with its own scope.
- **External users** — the gateway is for Datum staff only in v1.

## Proposal

When a request arrives, the gateway:

1. Terminates TLS using a cert-manager-issued certificate.
2. Validates the `Authorization: Bearer <jwt>` against the configured Zitadel `SecurityPolicy` (issuer, audience, remote JWKS).
3. Strips the client `Authorization` header.
4. Injects the org-owned provider credential from the matching `BackendSecurityPolicy`.
5. Establishes a hostname-verified TLS connection to the upstream provider (`BackendTLSPolicy`) and proxies the request.

### User Stories

#### Story 1: Staff engineer connects Claude Code via datumctl

An engineer with `datumctl` installed and an active staging login creates a Claude Code profile pointing at the internal gateway:

```bash
mkdir -p ~/datum-claude/
cat > ~/datum-claude/settings.json <<'EOF'
{
  "apiKeyHelper": "datumctl auth get-token",
  "env": {
    "ANTHROPIC_BASE_URL": "https://staging-internal-ai-gateway.datum.net/anthropic",
    "CLAUDE_CODE_API_KEY_HELPER_TTL_MS": "1800000"
  }
}
EOF
CLAUDE_CONFIG_DIR=~/datum-claude claude
```

Claude Code invokes `datumctl auth get-token` every 30 minutes, attaches the resulting JWT as a bearer token, and the gateway substitutes the org Anthropic key upstream. The engineer's personal key is no longer used; usage appears under the org account.

#### Story 2: Staff engineer connects a native desktop client

A desktop AI tool that speaks OIDC natively initiates the authorization-code flow against the Flow B OIDC application, opens the system browser, the engineer authenticates with Zitadel, and PKCE completes on a loopback redirect. The tool stores the resulting JWT and refresh token locally. Subsequent requests refresh the token as needed. From the gateway's perspective this is indistinguishable from Flow A — both produce JWTs against the same audience.

#### Story 3: Provider key rotation

A platform operator updates the Anthropic key in GCP Secret Manager. Within the External Secrets refresh interval (1h), the in-cluster `Secret` is updated. The `BackendSecurityPolicy` references the `Secret` by name, so the gateway picks up the new key on its next upstream connection — no Git change, no client coordination.

#### Story 4: Upstream provider outage

Anthropic returns 5xx responses. The gateway proxies these back to clients unmodified. The existing `envoy_cluster_*` and `envoy_listener_*` metrics surface the elevated error rate on the gateway fleet for the on-call alert path. There is no fallback in v1; clients retry per their own policy.

The same applies to the Qwen backend: if the tunnel endpoint is unreachable or returns errors, the gateway surfaces 5xx to the client. There is no Anthropic fallback for Qwen requests in v1.

#### Story 5: JWT expiry mid-session

An engineer's JWT expires during a long Claude Code session. `datumctl auth get-token` detects the expiry on the next invocation and performs a silent refresh. The client retries with the new token; the user sees a brief delay and no interactive prompt.

### Risks and Mitigations

| Risk                                                                                    | Mitigation                                                                                                                                                                                                                                    |
| --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Shared Anthropic key leakage.** The key is a single high-value credential in cluster. | Stored in GCP Secret Manager. Synced via External Secrets + Workload Identity. Never written to Git. Injected upstream-only by `BackendSecurityPolicy`; never echoed in responses. Client `Authorization` header is stripped before upstream. |
| **Unauthenticated access.** A misconfigured route could expose the provider key.        | `SecurityPolicy` is attached to the `HTTPRoute` itself — every path is gated. No bypass route exists. The hostname is not publicly advertised.                                                                                                |
| **Token replay across environments.**                                                   | JWT audience is bound to the staging Datum Cloud project ID; the JWKS URI points at the staging issuer. Tokens from any other project or environment fail validation.                                                                         |
| **Large-payload abuse.** The 50 MiB buffer could be used to exhaust proxy memory.       | Accepted for the current trust radius; will be tightened once usage data is available.                                                                                                                                                        |
| **Desktop refresh token theft.** Flow B issues refresh tokens to desktop clients.       | The native OIDC app is a public client with PKCE and loopback-only redirect URIs (OAuth 2.1 / RFC 8252). Refresh tokens can be revoked at Zitadel.                                                                                            |
| **Cost runaway on the shared org account.**                                             | Bounded by Anthropic's per-account limits; Anthropic billing alerts are the leading indicator.                                                                                                                                                |
| **Pre-1.0 upstream CRD shapes.**                                                        | We track upstream and contribute back rather than forking. The blast radius of a breaking change is one staging cluster.                                                                                                                      |

## Design Details

### v1 Implementation Scope

**Resources applied:**

- `Gateway` — HTTPS listener on `staging-internal-ai-gateway.datum.net`
- `AIGatewayRoute` — model-name routing to Anthropic and Qwen backends
- `AIServiceBackend` — one for Anthropic (`schema: Anthropic`), one for Qwen (`schema: OpenAI`)
- `Backend` — FQDN endpoints for both upstreams
- `BackendTLSPolicy` — hostname-verified TLS to `api.anthropic.com`; not needed for Qwen (tunnel handles TLS)
- `BackendSecurityPolicy` — injects the org Anthropic key upstream; no equivalent for Qwen
- `SecurityPolicy` — JWT validation + staff access enforcement (see [Authentication and Authorization](#authentication-and-authorization))
- `ClientTrafficPolicy` — 50 MiB buffer limit for LLM payloads
- `ExternalSecret` — syncs Anthropic key from GCP Secret Manager hourly

**Done looks like:** staff engineers can point Claude Code, Claude Desktop, OpenCode, or Pi at the gateway using either `datumctl auth get-token` or a native OIDC flow; requests reach both Anthropic and Qwen; provider credentials never leave the cluster.

**Intentionally not wired up in v1:** per-user rate limiting, prompt caching, model deflection, additional providers, and any production deployment.

### Architecture

```
Desktop / CLI client
        │
        │  HTTPS, Bearer <Zitadel JWT>
        ▼
   staging-internal-ai-gateway.datum.net  ──►  Envoy AI Gateway (staging, mcp-gateway fleet)
                                        │
                                        │  1. TLS termination (cert-manager)
                                        │  2. JWT validation (SecurityPolicy)
                                        │  3. Strip client Authorization header
                                        │  4. Inject provider credential
                                        │     (BackendSecurityPolicy)
                                        │  5. TLS to upstream (BackendTLSPolicy)
                                        ▼
                              api.anthropic.com  /  internal Qwen endpoint
```

### Resource Model

Every resource is a stock upstream CRD.

#### `Gateway`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: ai-gateway
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-dns
spec:
  gatewayClassName: mcp-gateway
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: staging-internal-ai-gateway.datum.net
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: ai-gateway-tls
      allowedRoutes:
        namespaces:
          from: Same
```

#### `AIGatewayRoute` + `AIServiceBackend`

The AI Gateway extracts the model name from the request body and sets it as the `x-ai-eg-model` header internally; rules match on that header. Qwen model names are matched explicitly; all other requests fall through to the Anthropic backend:

```yaml
apiVersion: aigateway.envoyproxy.io/v1beta1
kind: AIGatewayRoute
metadata:
  name: ai-gateway-route
spec:
  parentRefs:
    - name: ai-gateway
      kind: Gateway
      group: gateway.networking.k8s.io
  rules:
    - matches:
        - headers:
            - name: x-ai-eg-model
              value: qwen3.6
        - headers:
            - name: x-ai-eg-model
              value: qwen3.6-35b-a3b
      backendRefs:
        - name: qwen-local
    - backendRefs:
        - name: anthropic
---
apiVersion: aigateway.envoyproxy.io/v1beta1
kind: AIServiceBackend
metadata:
  name: anthropic
spec:
  schema:
    name: Anthropic
  backendRef:
    name: anthropic-upstream
    kind: Backend
    group: gateway.envoyproxy.io
---
apiVersion: aigateway.envoyproxy.io/v1beta1
kind: AIServiceBackend
metadata:
  name: qwen-local
spec:
  schema:
    name: OpenAI # Qwen exposes an OpenAI-compatible API
  backendRef:
    name: qwen-local-upstream
    kind: Backend
    group: gateway.envoyproxy.io
```

#### `Backend` (Qwen)

The Qwen endpoint is reached via a Datum tunnel; TLS is handled by the tunnel so no `BackendTLSPolicy` is needed for this path:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: Backend
metadata:
  name: qwen-local-upstream
spec:
  endpoints:
    - fqdn:
        hostname: <tunnel>.datumproxy.net
        port: 443
```

#### `Backend` + `BackendTLSPolicy` (Anthropic)

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: Backend
metadata:
  name: anthropic-upstream
spec:
  endpoints:
    - fqdn:
        hostname: api.anthropic.com
        port: 443
---
apiVersion: gateway.networking.k8s.io/v1
kind: BackendTLSPolicy
metadata:
  name: anthropic-upstream-tls
spec:
  targetRefs:
    - group: gateway.envoyproxy.io
      kind: Backend
      name: anthropic-upstream
  validation:
    wellKnownCACertificates: System
    hostname: api.anthropic.com
```

#### `BackendSecurityPolicy`

Binds the Anthropic key `Secret` to the `AIServiceBackend`; no equivalent is needed for the local Qwen backend:

```yaml
apiVersion: aigateway.envoyproxy.io/v1beta1
kind: BackendSecurityPolicy
metadata:
  name: ai-gw-anthropic-api-key
spec:
  targetRefs:
    - group: aigateway.envoyproxy.io
      kind: AIServiceBackend
      name: anthropic
  type: AnthropicAPIKey
  anthropicAPIKey:
    secretRef:
      name: ai-gw-anthropic-api-key
```

#### `SecurityPolicy` (JWT)

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: SecurityPolicy
metadata:
  name: ai-gateway-jwt-auth
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: ai-gateway-route
  jwt:
    providers:
      - name: datum-cloud-auth
        issuer: https://auth.staging.env.datum.net
        audiences:
          - "<staging-datum-cloud-project-id>"
        remoteJWKS:
          uri: https://auth.staging.env.datum.net/oauth/v2/keys
```

#### `ClientTrafficPolicy`

Raises the client buffer to accommodate LLM payloads. 50 MiB is the value recommended by the official Envoy AI Gateway examples; Envoy's default 32 KiB is too small for typical Anthropic Messages requests:

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: ClientTrafficPolicy
metadata:
  name: ai-gateway-buffer-limit
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: ai-gateway
  connection:
    bufferLimit: 50Mi
```

#### `ExternalSecret`

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: ai-gw-anthropic-api-key
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-store
    kind: ClusterSecretStore
  target:
    name: ai-gw-anthropic-api-key
    creationPolicy: Owner
  data:
    - secretKey: apiKey
      remoteRef:
        key: ai-gw-anthropic-api-key
```

### Authentication and Authorization

All access is gated by staging Zitadel. JWTs are validated against the issuer's JWKS endpoint and pinned to the staging Datum Cloud project audience.

The authorization boundary in v1 is "is Datum staff." JWT validation alone is not sufficient — the staging project contains accounts beyond Datum staff — so an additional enforcement mechanism is required. The options are unresolved:

<<[UNRESOLVED how to restrict access to Datum staff]>>

The staging Zitadel project contains more than Datum staff, so JWT validation alone is not a sufficient boundary. Options:

- **Email allowlist** — check the `email` claim already present in every Zitadel JWT. No new infrastructure; requires manual maintenance as staff changes.
- **JWT `groups` claim** — check a claim encoding Milo `GroupMembership`, injected via a Zitadel `preaccesstoken` action. Explored in [milo-os/zitadel-provider#115](https://github.com/milo-os/zitadel-provider/pull/115) but token bloat concerns were raised.
- **JWT `is_staff` boolean claim** — same mechanism as groups but a single boolean rather than a membership list, avoiding the bloat concern. Less flexible if finer grained access is needed later.
- **`ext_authz`** — Envoy calls an external service per request to make the access decision, enabling real-time membership checks without baking anything into the token. Requires a new service to build and operate.
- **Google as IdP** — replace Zitadel with Google OAuth; `@datum.net` domain restriction is enforced at authentication time. Breaks `datumctl`-based auth flows.

<<[/UNRESOLVED]>>

**Client flows.** Two interchangeable mechanisms produce JWTs the gateway accepts:

- **Flow A — `datumctl` OIDC application.** Clients invoke `datumctl auth get-token` to fetch a fresh JWT per request or TTL. Suitable for any client that can shell out for credentials.
- **Flow B — native OIDC application with PKCE.** A public OIDC client provisioned in staging Zitadel: authorization-code flow with PKCE, loopback redirect URIs only, refresh tokens enabled. Follows the OAuth 2.1 native-app pattern ([RFC 8252](https://datatracker.ietf.org/doc/html/rfc8252)). The client ID is published via a ConfigMap so in-cluster components can reference it without hard-coding.

Both flows produce JWTs against the same audience; the choice is determined by client capability.

### Token Handling

Claude Code with `apiKeyHelper` sends the JWT in both `Authorization: Bearer` and `x-api-key`. The `SecurityPolicy` is configured to accept the JWT from either header. The gateway then:

1. Validates the token from whichever header it arrives on.
2. Strips both client headers before the request reaches the upstream.
3. Substitutes the provider-specific credential from `BackendSecurityPolicy` (for Anthropic: the org `x-api-key` from the referenced `Secret`).

Provider APIs never see the user's token. Clients never see the org provider key.

### Secret Management

The Anthropic key is synced from GCP Secret Manager via External Secrets + Workload Identity. The `ExternalSecret` refreshes hourly; key rotation is eventual-consistent within that window with no Git change and no client coordination.

### Observability

The deployment inherits standard L7 proxy metrics from the Envoy gateway fleet. No new alert rules are needed in v1 — this gateway is covered by the existing mcp-gateway fleet alert rules: request rate, latency histograms, response-code distributions, and upstream cluster health (`envoy_cluster_upstream_rq_*`, `envoy_cluster_health_check_*`).

On top of that, the Envoy AI Gateway emits GenAI-specific Prometheus metrics following OpenTelemetry semantic conventions:

- `gen_ai_client_token_usage` — input/output token counts per request, labelled by `gen_ai_token_type`, `gen_ai_request_model`, and gateway name.
- `gen_ai_server_request_duration` — end-to-end request latency from first request header received to last response byte.
- `gen_ai_server_time_to_first_token` — latency to first token; the key signal for streaming responsiveness.
- `gen_ai_server_time_per_output_token` — inter-token latency; useful for detecting upstream throttling.

All four are labelled with `gen_ai_operation_name`, `gen_ai_request_model`, `gen_ai_response_model`, and `gen_ai_provider_name`, giving per-model and per-provider breakdowns without any additional instrumentation.

## Roadmap

Each item below will get its own enhancement when scoped. This list makes the [Non-Goals](#non-goals) concrete — each deferral is to a specific follow-up, not "future work".

1. **Multi-provider support.** Add OpenAI, Bedrock, Gemini as additional `AIServiceBackend` + `BackendSecurityPolicy` pairs. Tracked under [#391](https://github.com/datum-cloud/enhancements/issues/391).
2. **Per-user attribution and rate limiting.** Log the JWT `sub` claim on the proxy access log and drive per-user token budgets via `BackendTrafficPolicy`.
3. **Prompt caching and model routing.** Wire up the Envoy AI Gateway prompt cache and the model-routing layer that can deflect short prompts to cheaper models.
4. **Additional local models.** Expand the local model backend beyond Qwen as the H100 capacity and model lifecycle story matures.
5. **External users.** Open the gateway to non-staff users. This involves per-customer billing, quota, and authorization — a significant scoping exercise on its own.

## Implementation History

- **2026-06-03**: Initial AI Gateway infrastructure setup done in [datum-cloud/infra#2589](https://github.com/datum-cloud/infra/pull/2589).

## Drawbacks

- **Shared Anthropic key is a single point of cost runaway.** One misbehaving client can exhaust the org quota. Anthropic's own rate limits and billing alerts are the only current backstop.
- **Upstream Envoy AI Gateway is pre-1.0.** CRD shapes can shift. We are tracking upstream rather than forking; the blast radius of any incompatible change is one staging cluster.

## Alternatives

**Anthropic managed org (status quo).** The existing setup — a shared Anthropic org account with per-engineer keys — solves billing aggregation but locks the team into a single commercial provider, hits per-account rate limits as individual usage grows, and provides no path to routing traffic to OSS models on our own compute. It builds nothing toward the product.

**Skip straight to a production deployment.** We could have targeted a production-grade deployment from the start. We chose not to because this deployment exists to answer open questions — which AI Gateway features work as-is, which need work on top, what the right authorization model looks like in practice. Committing to a production architecture before answering those is premature.

**A different gateway technology.** We already operate Envoy Gateway for MCP and other internal traffic. The AI Gateway is the upstream extension of that ecosystem; no evaluation of alternatives was needed.
