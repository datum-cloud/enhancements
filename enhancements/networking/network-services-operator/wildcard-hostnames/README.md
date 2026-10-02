---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Wildcard Hostnames on ALBs

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [User Experience](#user-experience)
  - [Ownership Rule](#ownership-rule)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Architecture](#architecture)
  - [Certificate Service](#certificate-service)
  - [Network Services Operator](#network-services-operator)
  - [Infrastructure](#infrastructure)
  - [Why a Separate Service](#why-a-separate-service)
  - [Phases](#phases)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Dependencies](#dependencies)
- [Security Considerations](#security-considerations)
- [Open Questions](#open-questions)
- [Implementation History](#implementation-history)
- [Alternatives](#alternatives)

## Summary

Users can attach a wildcard hostname such as `*.s3.example.com` to an ALB, so one listener serves every name under a domain they own, with a Datum-issued certificate. Ownership is proven at the DNS level, and a wildcard reserves its whole subtree for the owning project. Certificates for wildcards issue through DNS, so the name never has to point at Datum before it is ready. The same DNS-based issuance is offered for exact hostnames, so a certificate can be in place before traffic moves from another provider.

## Motivation

Tenant-based services, such as an object store that gives every bucket its own subdomain, need to serve many subdomains through one ALB. Today each hostname needs a separate entry, which makes these workloads impractical.

Certificates for custom hostnames also require traffic to point at Datum before they can issue. Moving a live site to Datum therefore risks a window of broken HTTPS.

### Goals

- Accept a wildcard as a hostname on an ALB and route every matching name to its backends.
- Issue and renew certificates for wildcard hostnames automatically, using a DNS challenge the user enables with one CNAME shown in the resource status.
- Offer DNS-based issuance for exact hostnames as well, so a certificate can be in place before traffic moves to Datum.
- Reserve the subtree under a claimed wildcard for the owning project, and refuse a wildcard when another project already holds a name beneath it.
- Require DNS-level proof of ownership for wildcard hostnames.
- Show users what they are waiting on at every step: ownership, DNS delegation, routing record, certificate.

### Non-Goals

- Bring-your-own certificates.
- Multi-label wildcards such as `*.*.example.com`.
- Changing how the platform's own `*.datumproxy.net` names work.

## Proposal

### User Stories

#### Story 1

As an operator of a bucket-per-subdomain object store, I attach `*.s3.example.com` to one ALB, so every new bucket is reachable over HTTPS without touching the ALB.

#### Story 2

As a user moving `www.example.com` from another provider, I get a Datum certificate issued through DNS first, then cut traffic over with no window of broken HTTPS.

### User Experience

The user adds the wildcard to the ALB's hostnames:

```yaml
apiVersion: networking.datumapis.com/v1alpha
kind: HTTPProxy
metadata:
  name: s3
spec:
  hostnames:
    - "*.s3.example.com"
  rules:
    - backends:
        - endpoint: https://storage.internal.example.com
```

The steps that follow:

1. **Prove ownership.** The domain for the base, or a parent of it, must be Verified by DNS TXT record or by a Datum-hosted DNS zone. HTTP-token verification does not qualify a wildcard: control of one host under the wildcard, such as a bucket owner serving a token, is exactly what an attacker has.
2. **Publish two records** in external DNS. For a Datum-hosted zone the platform writes both.
   - Routing: `*.s3.example.com CNAME <canonical>.datumproxy.net`
   - Certificate: `_acme-challenge.s3.example.com CNAME <random>.<delegation zone>`. The target appears in the certificate's status. It is random per certificate, not derived from the hostname.
3. **Wait on status.** The ALB reports each step the user is waiting on: ownership verified, delegation in place, hostname claimed, routing record, certificate ready.

Exact hostnames can opt into the same DNS-based issuance, publishing only the certificate record ahead of the cutover.

### Ownership Rule

Platform-wide hostname claims become subtree-aware:

- A wildcard claim reserves every name beneath it, at any depth.
- A later claim from another project for a specific name under it is refused.
- A wildcard is refused while other projects hold names beneath it, and the refusal names them.

Exclusivity is required because the data plane prefers an exact match over a wildcard. If another project could claim `login.s3.example.com`, it would silently take that traffic from the wildcard owner.

Domain verification is not cross-project exclusive today. Subtree-exclusive claims make that acceptable for this feature: two projects can verify the same domain, but only one can hold a given subtree.

### Notes/Constraints/Caveats

- **Single-label wildcards only.** `*.s3.example.com` is accepted; `*.*.example.com` is not.
- **A wildcard certificate covers one label.** `a.b.s3.example.com` routes through the wildcard but is not covered by its certificate. AWS behaves the same way.
- **At most 8 names per certificate.**

### Risks and Mitigations

- **Hostname hijack under a wildcard.** Subtree-exclusive claims refuse any other project's name beneath a claimed wildcard.
- **Wildcard granted on weak proof.** Only DNS-level verification qualifies a wildcard.
- **Another project obtains the certificate.** Delegation targets are random per certificate and bound to the requesting project.
- **Tenant forges certificate status to steer issuance.** The service rebuilds status from spec and its own state every reconcile, and only platform identities may write status.
- **Rollout disturbs existing certificates.** Enabling the feature does not reissue certificates already in place.

## Design Details

### Architecture

The ALB controller requests certificates from a new Milo certificate service, which issues through a dedicated DNS-01 issuer and a platform delegation zone.

![Wildcard hostnames: containers for the ALB, the certificate service, and DNS-01 issuance through a delegation zone](architecture.png)

### Certificate Service

A new Milo service, `milo-os/certificates`, serves `TLSCertificate` in `certificates.miloapis.com`.

```yaml
apiVersion: certificates.miloapis.com/v1alpha1
kind: TLSCertificate
metadata:
  name: s3
spec:
  dnsNames:
    - "*.s3.example.com"
  issuance: DNS01
  secretName: s3-tls
status:
  issuance: DNS01
  delegationTarget: k3f9q2x7.acme-dns.staging.env.datum.net
  requiredDNSRecords:
    - name: _acme-challenge.s3.example.com
      type: CNAME
      value: k3f9q2x7.acme-dns.staging.env.datum.net
  secretRef:
    name: s3-tls
  notAfter: "2027-01-01T00:00:00Z"
  renewalTime: "2026-12-02T00:00:00Z"
  conditions:
    - type: Accepted
      status: "True"
    - type: DNSDelegationReady
      status: "True"
    - type: Issuing
      status: "False"
    - type: Ready
      status: "True"
```

- **Issuance modes.** `Auto`, `HTTP01` or `DNS01`. In HTTP-01 mode the service publishes the challenge in status and the caller serves it.
- **Multicluster.** Reconciles across every project control plane through Milo's multicluster runtime.
- **Issuance.** Wraps cert-manager on the infra control plane. No ACME order is placed until the delegation CNAME resolves.
- **Delivery.** Writes the issued Secret into the project. A service-side copy is what platform components distribute.
- **Status integrity.** Status is rebuilt from spec and service state on every reconcile, so tenants cannot drive issuance through status. A webhook restricts writers to platform identities.
- **No ownership check.** The service does not verify ownership. Callers request only names they have verified.

### Network Services Operator

The operator consumes the certificate service behind a feature flag:

- Creates a `TLSCertificate` per claimed listener instead of a hub-side cert-manager `Certificate`.
- Serves HTTP-01 challenges only for hostnames it has claimed.
- Mirrors the issued certificate to edges from the service-side copy.
- Maps certificate conditions onto HTTPProxy status.
- Admits wildcards and enforces subtree-exclusive claims (phase 2).

Enabling the flag does not reissue existing certificates. Platform `*.datumproxy.net` names are unchanged.

### Infrastructure

- A dedicated DNS-01 issuer scoped to a platform delegation zone. Staging uses `acme-dns.staging.env.datum.net` and the Let's Encrypt staging endpoint.
- A dedicated DNS writer identity for that zone.
- A denylist of platform domains the service refuses to issue for.

### Why a Separate Service

- **Foundation, not feature.** Certificates are provider-agnostic, like billing, DNS and IPAM.
- **Narrow interface.** For HTTP-01 the contract with the ALB controller is "publish the challenge in status; the controller serves it".
- **Room to grow.** A certificate resource is the natural home for bring-your-own certificates later.

Domains may move into milo-os later on the same reasoning.

### Phases

1. Certificate service, staging deployment, and operator consumption behind the flag. In flight: milo-os/certificates#1 and #2, datum-cloud/infra#6622 and #6624, datum-cloud/network-services-operator#526.
2. Wildcard admission in the operator, subtree-exclusive claims, DNS-only verification for wildcards, zero-touch records for Datum DNS zones, and portal display of required records.
3. Production enablement.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

- **Enablement.** A feature flag on the Network Services Operator switches listeners to the certificate service. Wildcard admission ships behind the same rollout.
- **Default behavior.** Unchanged for existing listeners; their certificates are not reissued.
- **Rollback.** Turning the flag off returns listeners to hub-side cert-manager certificates on the next reconcile. Wildcard listeners have no fallback, because HTTP-01 cannot issue a wildcard.

### Dependencies

- cert-manager on the infra control plane, with a DNS-01 issuer for the delegation zone.
- Datum DNS hosting the delegation zone.
- Milo's multicluster runtime and subresource authorization (see below).

## Security Considerations

- **Status forgery.** Tenants must not write certificate status. This depends on a separate fix to subresource authorization in the Milo authorizer, a prerequisite for production.
- **Project-bound delegation targets.** A target is random and tied to one project's certificate, so it cannot be reused to issue for another project.
- **Dedicated issuer credentials.** The DNS-01 issuer can write only the delegation zone, never customer zones.
- **Private keys at edges.** Wildcard keys are distributed to every edge that serves the name, widening their exposure compared with a single host.
- **Shared rate limits.** Let's Encrypt limits are per registered domain, so tenants sharing a parent domain share a budget.
- **Stale ownership.** Verification is never rechecked and claims never expire. A domain that changes hands keeps its old claims. Flagged for follow-up.

## Open Questions

<<[UNRESOLVED wildcard entitlement]>>
Should wildcards be a per-project entitlement rather than available to every project?
<<[/UNRESOLVED]>>

<<[UNRESOLVED cross-project domain verification]>>
Should Domain verification become cross-project exclusive, rather than relying on subtree-exclusive hostname claims?
<<[/UNRESOLVED]>>

<<[UNRESOLVED default issuance]>>
Once DNS-based issuance is stable, should it replace HTTP-01 as the default for all custom hostnames?
<<[/UNRESOLVED]>>

## Implementation History

- 2026-10-01: Provisional proposal.

## Alternatives

- **Delegation record on the Domain.** Rejected: the record is keyed on the wildcard base, which the Domain cannot know before a hostname is requested.
- **Hash-derived delegation target.** Rejected: the target would be identical across projects, letting a second project obtain the certificate.
- **Coexistence with most-specific-wins.** Rejected: another project's exact name would hijack the wildcard owner's traffic.
- **Customer-supplied certificates.** Out of scope; a future addition on the same certificate resource.
- **Solving DNS-01 directly in customer Datum DNS zones.** Rejected for now: it needs platform-wide write credentials across customer zones.
