---
status: implementable
stage: alpha
latest-milestone: "v0.x"
---

# Shared TLS Certificate for Default HTTPS Listeners

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Operator Configuration](#operator-configuration)
  - [Gateway Controller Behavior](#gateway-controller-behavior)
  - [Certificate Provisioning (Host Cluster)](#certificate-provisioning-host-cluster)
  - [Secret Distribution: Host → Downstream Clusters (Karmada)](#secret-distribution-host--downstream-clusters-karmada)
  - [Secret Distribution: Downstream Namespace Fan-out (Kyverno)](#secret-distribution-downstream-namespace-fan-out-kyverno)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Dependencies](#dependencies)
- [Alternatives](#alternatives)

## Summary

Every HTTPProxy gets a platform-managed default HTTPS listener with a
UID-based hostname under the configured target domain (e.g.
`5e5783b223b34143b05af91df95c94e5.datumproxy.net`). Today, each of these
listeners triggers an individual certificate issuance via cert-manager. At
scale this creates unnecessary load on the ACME provider, slows down initial
HTTPS availability for new proxies, and produces a large volume of Certificate
resources that must be individually tracked and renewed.

This enhancement introduces a `defaultListenerTLSSecretName` configuration
option that allows default HTTPS listeners to reference a shared, pre-provisioned
TLS certificate (typically a wildcard like `*.datumproxy.net`) instead of
issuing individual certificates. Custom hostname listeners are unaffected and
continue to receive their own per-listener certificates.

## Motivation

### Goals

- Eliminate per-gateway certificate issuance for default HTTPS listeners.
- Reduce time-to-HTTPS for new HTTPProxy resources from certificate issuance
  latency (seconds to minutes) to near-instant (secret already exists).
- Reduce load on ACME providers and cert-manager.
- Provide a simple operator configuration knob with no changes to the
  HTTPProxy API or user-facing behavior.
- Ensure custom hostname listeners remain fully independent and unaffected.

### Non-Goals

- Changing how custom hostname certificates are issued or managed.
- Managing the lifecycle of the shared certificate itself (that is the
  responsibility of cert-manager and cluster infrastructure).
- Distributing the shared certificate secret to downstream namespaces (that
  is the responsibility of infrastructure tooling such as Kyverno).

## Proposal

### User Stories (Optional)

#### Story 1

As an infrastructure operator, I want new HTTPProxy resources to have working
HTTPS on their default hostname immediately, without waiting for individual
certificate issuance.

#### Story 2

As an infrastructure operator managing dozens of downstream clusters, I want
to reduce the number of Certificate resources and ACME requests from
O(gateways) to O(1) per cluster for platform-managed hostnames.

### Notes/Constraints/Caveats (Optional)

**Wildcard certificates require DNS-01 challenges.** Let's Encrypt (and most
ACME providers) cannot issue wildcard certificates via HTTP-01 challenges. The
issuing cluster must have a cert-manager `ClusterIssuer` configured with a
DNS-01 solver (e.g. Cloud DNS).

**Edge clusters do not have DNS-01 capability.** The `letsencrypt-dns`
ClusterIssuer and GCP CloudDNS credentials are only deployed to the host
cluster (staging/production). Edge clusters have cert-manager installed but
no DNS-01 issuers, so they cannot issue wildcard certificates independently.

**The certificate is issued centrally on the host cluster.** A single
cert-manager `Certificate` resource on the host cluster provisions the
wildcard. The resulting TLS secret is then distributed to downstream clusters
via Karmada and into per-gateway namespaces via Kyverno.

**Secret distribution spans two hops.** The operator itself only references
the secret by name on downstream gateway listeners. Getting the secret into
the right place is handled by infrastructure tooling outside the operator:
1. **Host → downstream clusters:** Karmada `ClusterPropagationPolicy`
2. **Downstream cluster → `ns-*` namespaces:** Kyverno `ClusterPolicy`

### Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Shared cert secret missing from a downstream namespace | Kyverno policy with `generateExisting: true` and `synchronize: true` covers both new and existing namespaces. Gateway will report TLS errors, prompting investigation. |
| Secret not propagated to a downstream cluster | Karmada `ClusterPropagationPolicy` handles distribution. Standard Karmada status reporting surfaces propagation failures. |
| Wildcard cert renewal failure on host cluster | Same risk as any cert-manager certificate. Standard cert-manager monitoring and alerting applies. Karmada + Kyverno propagate the renewed secret automatically. |
| Rollback to per-gateway certs | Remove `defaultListenerTLSSecretName` from config; the operator reverts to the existing per-listener cert-manager flow on next reconcile. |

## Design Details

### Operator Configuration

A single string field on `GatewayConfig`:

```yaml
# config.yaml
gateway:
  defaultListenerTLSSecretName: wildcard-datumproxy-tls
```

When empty (the default), the operator behaves exactly as before — each
default HTTPS listener gets an individual certificate via cert-manager.

### Gateway Controller Behavior

In `getDesiredDownstreamGateway`, when constructing the downstream gateway for
each upstream listener:

1. **Default HTTPS listener with `defaultListenerTLSSecretName` set:**
   The listener's `CertificateRefs` point to the shared secret. cert-manager
   annotations are skipped for this listener since no individual certificate
   needs to be issued.

2. **Custom hostname HTTPS listeners:** Unchanged. They are never default
   listeners (`IsDefaultListener` returns false), so they always receive
   per-listener certificates via cert-manager.

3. **TrafficProtectionPolicy certificate readiness checks:** Default listeners
   using the shared secret are skipped when checking for cert-manager
   `Certificate` resource readiness, since no such resource exists for them.

### Certificate Provisioning (Host Cluster)

The wildcard certificate is issued once on the host cluster (staging or
production) where the `letsencrypt-dns` ClusterIssuer and GCP CloudDNS
credentials are available. This is a one-time setup per environment:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-datumproxy
  namespace: datum-downstream-gateway
spec:
  secretName: wildcard-datumproxy-tls
  issuerRef:
    kind: ClusterIssuer
    name: letsencrypt-dns
  dnsNames:
    - "*.datumproxy.net"
  duration: 2160h
  renewBefore: 720h
```

cert-manager on the host cluster handles issuance and automatic renewal. The
resulting secret (`wildcard-datumproxy-tls`) is the single source of truth.

### Secret Distribution: Host → Downstream Clusters (Karmada)

The certificate secret must be pushed from the host cluster into the Karmada
API so that Karmada can distribute it to downstream/edge clusters. The
existing federated flow is one-directional (Flux → Karmada → downstream), so
bridging a host-cluster-managed secret into Karmada requires one of:

- A Flux `Kustomization` on the host cluster that reads the secret and applies
  it to the Karmada API (using `kubeConfig` pointing to the Karmada control
  plane).
- Making the host cluster a Karmada member cluster so secrets can be pulled up.

Once the secret exists in the Karmada API, a `ClusterPropagationPolicy`
distributes it to downstream clusters. The existing propagation policy
selects secrets by label (`meta.datumapis.com/upstream-namespace` or
`cert-manager.io/issuer-name: nso-gateway`), so a new selector must be added
to match the wildcard certificate secret — for example by labeling the secret
and adding a matching rule:

```yaml
- apiVersion: v1
  kind: Secret
  labelSelector:
    matchLabels:
      meta.datumapis.com/default-listener-tls: "true"
```

Karmada places the secret into the `datum-downstream-gateway` namespace on
each downstream cluster.

### Secret Distribution: Downstream Namespace Fan-out (Kyverno)

On each downstream cluster, a Kyverno `ClusterPolicy` clones the secret from
`datum-downstream-gateway` into every gateway namespace (`ns-*`):

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: sync-default-listener-tls-secret
spec:
  rules:
    - name: sync-default-listener-tls-secret
      match:
        all:
          - resources:
              kinds:
                - Namespace
      preconditions:
        all:
          - key: "{{ request.object.metadata.name }}"
            operator: Equals
            value: "ns-*"
      generate:
        synchronize: true
        generateExisting: true
        apiVersion: v1
        kind: Secret
        name: wildcard-datumproxy-tls
        namespace: "{{ request.object.metadata.name }}"
        clone:
          namespace: datum-downstream-gateway
          name: wildcard-datumproxy-tls
      skipBackgroundRequests: true
  useServerSideApply: true
```

- `synchronize: true` — when the source secret is updated (via Karmada after
  a cert-manager renewal on the host cluster), Kyverno propagates the update
  to all copies.
- `generateExisting: true` — backfills namespaces that existed before the
  policy was created.

### End-to-End Flow

```
┌─────────────────────────────────────────────────────┐
│  Host Cluster (staging / production)                │
│                                                     │
│  cert-manager ──► Certificate ──► Secret            │
│  (letsencrypt-dns)   (wildcard-datumproxy-tls)      │
│                          │                          │
│                    Flux Kustomization                │
│                    (kubeConfig → Karmada)            │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│  Karmada Control Plane                              │
│                                                     │
│  Secret ──► ClusterPropagationPolicy                │
│                  │                                  │
└──────────────────┼──────────────────────────────────┘
                   │
          ┌────────┼────────┐
          ▼        ▼        ▼
┌──────────┐ ┌──────────┐ ┌──────────┐
│ Edge 1   │ │ Edge 2   │ │ Edge N   │
│          │ │          │ │          │
│ Secret   │ │ Secret   │ │ Secret   │
│ (datum-  │ │ (datum-  │ │ (datum-  │
│ downstrm)│ │ downstrm)│ │ downstrm)│
│    │     │ │    │     │ │    │     │
│ Kyverno  │ │ Kyverno  │ │ Kyverno  │
│    │     │ │    │     │ │    │     │
│ ┌──┴──┐  │ │ ┌──┴──┐  │ │ ┌──┴──┐  │
│ │ns-a │  │ │ │ns-a │  │ │ │ns-a │  │
│ │ns-b │  │ │ │ns-b │  │ │ │ns-b │  │
│ │ns-..│  │ │ │ns-..│  │ │ │ns-..│  │
│ └─────┘  │ │ └─────┘  │ │ └─────┘  │
└──────────┘ └──────────┘ └──────────┘
```

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

#### How can this feature be enabled / disabled in a live cluster?

- [x] Other
  - Describe the mechanism: Set or remove `gateway.defaultListenerTLSSecretName`
    in the operator configuration. The operator picks up the change on restart
    and reconciles all gateways.
  - Will enabling / disabling the feature require downtime of the control plane? No.
  - Will enabling / disabling the feature require downtime or reprovisioning of a node? No.

#### Does enabling the feature change any default behavior?

No. The feature is opt-in. When `defaultListenerTLSSecretName` is empty (the
default), behavior is identical to the existing per-listener certificate flow.

#### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Removing `defaultListenerTLSSecretName` from the operator config causes
the gateway controller to revert to per-listener cert-manager issuance on the
next reconcile. The existing Kyverno policy for per-gateway issuers
(`per-datum-gateway-issuer`) handles issuer creation and cert-manager takes
over certificate management.

### Dependencies

#### Does this feature depend on any specific services running in the cluster?

- **cert-manager (host cluster)** — must be installed on the host cluster with
  a `ClusterIssuer` that supports DNS-01 challenges (e.g. `letsencrypt-dns`
  with GCP CloudDNS credentials). Edge clusters do not need DNS-01 capability.
- **Karmada** — must be configured with a `ClusterPropagationPolicy` to
  distribute the wildcard certificate secret from the Karmada API to
  downstream clusters.
- **Kyverno (downstream clusters)** — must be installed on each downstream
  cluster to fan out the secret from `datum-downstream-gateway` into `ns-*`
  namespaces.

All three are existing dependencies in the current deployment model.

## Alternatives

### Per-gateway certificate issuance (current behavior)

The status quo. Each gateway gets its own certificate via cert-manager. Works
but creates O(gateways) certificates per cluster and adds latency to initial
HTTPS availability.

### Per-cluster independent wildcard issuance

Deploy the `letsencrypt-dns` ClusterIssuer and GCP CloudDNS credentials to
every edge cluster so each can issue its own wildcard certificate. This keeps
private keys cluster-local and removes the Karmada propagation hop. Rejected
because edge clusters are not necessarily in GCP and may not have access to
CloudDNS credentials. Deploying cloud provider service account keys to edge
nodes introduces significant operational and security overhead.

### Operator-managed secret copying

The operator reads the source secret and copies it into each downstream
namespace during gateway reconciliation. Rejected because it couples the
operator to secret lifecycle management, creates staleness windows between
cert renewal and copy propagation, and duplicates functionality already
provided by Kyverno.

### Cross-namespace CertificateRef with ReferenceGrant

Place the wildcard cert in a single namespace and use Gateway API
cross-namespace `CertificateRefs`. Rejected because `ReferenceGrant` is capped
at 16 entries, which is insufficient for the number of downstream namespaces
at scale.
