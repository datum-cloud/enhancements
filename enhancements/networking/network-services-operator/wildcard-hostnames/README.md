---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

<!-- omit from toc -->
# Wildcard Hostnames on ALBs

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Experience](#user-experience)
  - [User Stories](#user-stories)
  - [Security](#security)
  - [Certificate Issuance](#certificate-issuance)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Certificate Service](#certificate-service)
  - [ALB Integration](#alb-integration)
  - [Delegation Zone](#delegation-zone)
- [Future Work](#future-work)
- [Dependencies](#dependencies)
- [Implementation History](#implementation-history)
- [Alternatives](#alternatives)
  - [Delegation Record on the Domain](#delegation-record-on-the-domain)
  - [Hash-Derived Delegation Target](#hash-derived-delegation-target)
  - [Coexistence with Most-Specific-Wins](#coexistence-with-most-specific-wins)
  - [Issue Inside the ALB Controller](#issue-inside-the-alb-controller)
  - [Solve DNS-01 in Customer Zones](#solve-dns-01-in-customer-zones)

## Summary

Wildcard hostnames let users attach a name such as `*.s3.example.com` to an ALB,
so one listener serves every name under a domain they own, with a Datum-issued
certificate. Ownership is proven at the DNS level, and a wildcard reserves its
whole subtree for the owning project. Certificates for wildcards issue through
DNS, so the name never has to point at Datum before it is ready. The same
DNS-based issuance is offered for exact hostnames, so a certificate can be in
place before traffic moves from another provider.

## Motivation

Tenant-based services need to serve many subdomains through one ALB, but today
each hostname needs its own entry:

- **Bucket-per-subdomain storage**: Every bucket in an object store gets its own
  name, and buckets come and go faster than anyone should edit an ALB.
- **Multi-tenant SaaS**: Onboarding a customer at `acme.app.example.com` should
  be a database row, not an infrastructure change.
- **Zero-downtime migration**: A site moving from another provider cannot get a
  Datum certificate until it points at Datum, which risks a window of broken
  HTTPS.

### Goals

- Accept a wildcard as a hostname on an ALB and route every matching name to its
  backends
- Issue and renew certificates for wildcard hostnames automatically, using a DNS
  challenge the user enables with one CNAME shown in the resource status
- Offer DNS-based issuance for exact hostnames as well, so a certificate can be
  in place before traffic moves to Datum
- Reserve the subtree under a claimed wildcard for the owning project, and
  refuse a wildcard when another project already holds a name beneath it
- Require DNS-level proof of ownership for wildcard hostnames
- Show users what they are waiting on at every step: ownership, DNS delegation,
  routing record, certificate

### Non-Goals

- Bring-your-own certificates
- Multi-label wildcards such as `*.*.example.com`
- Changing how the platform's own `*.datumproxy.net` names work

## Proposal

Users add a wildcard to an ALB like any custom hostname. The platform checks
DNS-level ownership, reserves the subtree for the project, and issues the
certificate through a DNS challenge.

Certificates move into a new platform certificate service. The ALB asks for a
certificate for the names it has claimed; the service obtains it and tells the
user which DNS records it needs.

<p align="center">
  <img src="./architecture-context.png" alt="System Context" />
</p>

### User Experience

**Portal workflow:**

- **Verify the domain** by DNS TXT record or by hosting the zone on Datum DNS.
- **Add the hostname** `*.s3.example.com` to the ALB.
- **Publish the records** the portal lists; on Datum DNS the platform publishes
  them itself.
- **Watch progress** on the ALB's checklist of ownership, delegation, claim,
  routing and certificate.

**CLI experience:**

```bash
# Create the load balancer and attach the wildcard
datumctl alb create s3 --endpoint https://storage.internal.example.com
datumctl alb hostname add s3 '*.s3.example.com'

# Each custom hostname with its ownership, DNS and certificate state,
# and the records left to publish
datumctl alb describe s3

# Domain verification state
datumctl get domains
```

`hostname add` does not wait; `describe` shows each custom hostname's available,
DNS and certificate status and the records still to publish. With this proposal
the plugin accepts wildcards, lists required records including the delegation
CNAME, and points its "DNS not delegated" hint at the certificate record.

**Records to publish** for a zone hosted outside Datum:

- **Routing**: Name `*.s3.example.com`, Type `CNAME`, Value
  `<canonical>.datumproxy.net`
- **Certificate**: Name `_acme-challenge.s3.example.com`, Type `CNAME`, Value
  `<random>.<delegation zone>`
- **Ownership** (once per domain): Name `datum-custom-hostname.example.com`,
  Type `TXT`, Value the token shown on the Domain

### User Stories

#### Attach a Wildcard

As the operator of a bucket-per-subdomain object store, I verify `example.com`
once, attach `*.s3.example.com` to one ALB, and publish two CNAMEs. Every new
bucket then works over HTTPS without touching the ALB.

#### Migrate with the Certificate Ahead of Cutover

As a team moving `www.example.com` from another provider, I add the hostname
with DNS-based issuance and publish only the certificate CNAME. Once the
certificate is ready, I switch the routing record, and visitors never see a
certificate error.

### Security

**Ownership proof.** A wildcard requires its base, or a parent, to be verified
by DNS TXT record or a Datum DNS zone. HTTP-token verification does not qualify:
serving a token from one host under the wildcard, such as a bucket, is exactly
what an attacker can do.

**Subtree reservation.** Hostname claims become subtree-aware, because the edge
prefers an exact match over a wildcard:

- A wildcard claim reserves every name beneath it, at any depth
- Another project's later claim under the wildcard is refused
- A wildcard is refused while other projects hold names beneath it, and the
  refusal names them

<<[UNRESOLVED]>>
Should Domain verification become cross-project exclusive?
<<[/UNRESOLVED]>>

**Who can write status.** Only platform identities can write certificate status,
and the service rebuilds it every reconcile.

<<[UNRESOLVED]>>
Should wildcards be a per-project entitlement?
<<[/UNRESOLVED]>>

### Certificate Issuance

- **HTTP-01 (today's default)**: No extra record, but the hostname must already
  point at Datum and cannot be a wildcard.
- **DNS-01**: Covers wildcards and issues before traffic moves, at the cost of
  one CNAME.
- **Auto**: DNS-01 for wildcards, HTTP-01 otherwise.

Pick DNS-01 for any wildcard and for any hostname that must not see a gap during
migration.

The certificate CNAME delegates `_acme-challenge.<base>` to a Datum-run zone, so
users never grant write access to their DNS. Its target is random per
certificate, because a target derived from the hostname would let a second
project asking for the same name obtain the first project's certificate.

Renewal needs no user action while the CNAME stays in place. HTTP-01 stays
the default for exact hostnames; DNS-01 is an explicit choice.

### Notes/Constraints/Caveats

- **Single-label wildcards only.**
- **One label of coverage**: `a.b.s3.example.com` routes through
  `*.s3.example.com` but is not covered by its certificate, as on AWS.
- **At most 8 names per certificate.**

### Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Hijack of a name under a wildcard | Subtree-exclusive claims |
| Wildcard granted on weak proof | DNS-level verification only |
| Another project obtains the certificate | Random, project-bound delegation targets |
| Forged status steers issuance | Platform-only writers; status rebuilt each reconcile |
| Shared Let's Encrypt rate limits | No order until DNS is ready; alert on issuance failures |
| Stale ownership after a domain changes hands | Re-verification and claim expiry (future work) |

## Design Details

### Certificate Service

Certificates move out of the ALB controller into a new Milo foundation service,
`milo-os/certificates`, which serves `TLSCertificate` in
`certificates.miloapis.com`. The ALB decides which names a project may serve;
the certificate service proves them to the certificate authority and reports
what the user still has to publish. Any future service that terminates TLS
requests certificates the same way, and bring-your-own certificates later land
on the same resource.

```yaml
apiVersion: certificates.miloapis.com/v1alpha1
kind: TLSCertificate
metadata:
  name: s3
spec:
  dnsNames:
    - "*.s3.example.com"
  issuance: DNS01
status:
  delegationTarget: k3f9q2x7.acme-dns.example.net
  requiredDNSRecords:
    - name: _acme-challenge.s3.example.com
      type: CNAME
      content: k3f9q2x7.acme-dns.example.net
      purpose: Certificate
  secretRef:
    name: s3-tls
  conditions:
    - type: DNSDelegationReady
      status: "True"
    - type: Ready
      status: "True"
```

The service reconciles every project control plane, places no order until the
delegation CNAME resolves, and delivers the certificate as a Secret in the
project. Only platform identities can write its status, and it does not verify
ownership: callers request only names they have claimed.

### ALB Integration

The ALB requests a certificate for each hostname it has claimed, serves HTTP-01
challenges for those names, and distributes the issued certificate to the edge.
Hostname claims become subtree-aware, and wildcards are admitted only on
DNS-level verification. Existing certificates are not reissued when the
integration is enabled, and `*.datumproxy.net` names are unchanged.

### Delegation Zone

A platform-owned DNS zone receives every DNS-01 challenge. Its issuer
credential can write only that zone, and the service refuses to issue for the
platform's own domains.

## Future Work

**Verification:**

- **Cross-project exclusive domains**: Only one project may verify a domain
- **Re-verification and claim expiry**: Release claims when ownership lapses

**Platform:**

- **Domains in Milo**, alongside the certificate service
- **Portal and CLI records**: Copy buttons in the portal and in `datumctl alb
  describe`

## Dependencies

- **Network Services Operator**: Admits hostnames, enforces claims, serves
  certificates at the edge.
- **Domains**: Records which domains a project has verified, and how.
- **Datum DNS**: Hosts the delegation zone and publishes records for zones on
  Datum.
- **cert-manager**: Places and renews certificate orders.
- **datumctl alb plugin**: Shows hostname and certificate state and the records
  to publish.
- **Milo multicluster runtime**: Lets the certificate service reconcile every
  project.
- **Milo authorizer subresource fix**: Stops tenants writing certificate status;
  required before production.

## Implementation History

- 2026-10-01: Provisional proposal.

## Alternatives

### Delegation Record on the Domain

Publish the challenge delegation once per Domain instead of per certificate.

**Rejected because:** The record is keyed on the wildcard base, which the Domain
cannot know before a hostname is requested.

### Hash-Derived Delegation Target

Derive the delegation target from a hash of the hostname.

**Rejected because:** The target would be identical across projects, letting a
second project obtain the certificate.

### Coexistence with Most-Specific-Wins

Let other projects claim exact names under a wildcard, with the exact name
winning.

**Rejected because:** Any project could hijack the wildcard owner's traffic by
claiming one name.

### Issue Inside the ALB Controller

Add DNS-01 to the ALB controller's existing issuance.

**Rejected because:** It leaves no home for bring-your-own certificates and
makes every future TLS service rebuild the same flow.

### Solve DNS-01 in Customer Zones

Write challenge records directly into customer zones on Datum DNS.

**Rejected because:** It needs platform-wide write credentials across customer
zones.

