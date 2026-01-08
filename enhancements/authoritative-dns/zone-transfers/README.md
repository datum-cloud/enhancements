---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Datum DNS: Primary / Secondary Zones + TSIG + Transfers

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)

## Summary

This enhancement defines the Datum DNS API model for authoritative zones,
supporting Primary and Secondary DNS roles and secure zone transfers (AXFR/IXFR)
using TSIG. The initial provider backend is PowerDNS.

The design is DNS‑native and Kubernetes‑idiomatic by separating:
- user intent (CustomResourceDefinitions),
- secret material (Kubernetes `Secret` objects), and
- provider plumbing (opaque provider IDs in `.status`).

Transfers occur on a dedicated transfer plane. Anycast edge nameservers continue
to serve DNS from replicated canonical state and do not perform transfer
operations.

## Motivation

Authoritative DNS commonly requires both first‑party hosted primaries and
integration with external primaries via secondaries. Secure, auditable zone
transfers are essential for multi‑tenant safety. This proposal standardizes a
Kubernetes‑centric API that models DNS roles explicitly, requires TSIG for
secondary imports, and keeps cryptographic material out of CRDs while mapping
cleanly to PowerDNS.

### Goals

- Support Primary zones (Datum is source of truth).
- Support Secondary zones (mirror external primary via AXFR/IXFR).
- Support outbound transfers from Datum primaries to external secondaries.
- Require TSIG for Secondary zones.
- Keep secret material out of CRDs (use `Secret`).

### Non-Goals

- IP‑based authorization for secondary imports (intentionally unsupported).
- Changes to the existing anycast serving plane behavior.

## Proposal

### Architecture Overview

#### Serving Plane (existing)
- Global anycast authoritative nameservers: `ns1/ns2.datumdomains.net`.
- Edge PowerDNS instances serve zones from replicated canonical state (LMDB pipeline).
- Edges do not initiate AXFR/IXFR and do not accept public transfers.

#### Transfer Plane (new)
- Dedicated PowerDNS instances with transfer capability.
- Exposed via stable unicast endpoint: `xfr1.datumdomains.net` (static IP).
- Optional Cloud NAT so outbound transfers originate from a stable IP.
- Responsibilities:
  - Pull Secondary zones from upstream primaries (AXFR/IXFR).
  - Optionally serve outbound AXFR/IXFR for Datum primaries (impl choice).
  - Persist resulting zone state into canonical store → replicated to edges.

Invariant: Transfers occur only on the transfer plane. Anycast edges only serve.

### User Stories (Optional)

N/A

### Notes/Constraints/Caveats (Optional)

- Presence‑implies‑intent: if `transfer` is specified, transfers are configured.
- Secondary zones require `masters` and TSIG; IP‑trust is not supported.
- Outbound transfers from Primary zones are disabled by default.
- Edge plane remains unchanged; transfer plane owns AXFR/IXFR traffic.

### Risks and Mitigations

- Misconfiguration of TSIG keys → mitigate via `TSIGKey` schema validation and
  provider reconciliation; surface readiness and conditions.
- Transfer plane exposure → restrict endpoints, require TSIG, optionally use
  Cloud NAT for stable egress IP.
- Multi‑tenant safety → default deny; explicit `transfer` and Ready `TSIGKey`
  required for any transfer activity.

## Design Details

### APIs

#### 1) `TSIGKey` (new CRD)

Purpose: Represents a DNS TSIG key used to authenticate zone transfers.
The controller either:
- Generates secret material and creates a `Secret` (default), or
- Uses an existing `Secret` (BYO) without mutating it.

The controller provisions the PowerDNS TSIG key and exposes its opaque ID in `status.tsigKeyID`.

Minimal Spec:

```yaml
apiVersion: dns.datum.net/v1alpha1
kind: TSIGKey
metadata:
  name: example-com-xfr
spec:
  zoneRef:
    name: example-com
    # namespace: default  # optional; defaults to same namespace as TSIGKey
  algorithm: hmac-sha256
  # optional (BYO secret):
  # secretRef:
  #   name: existing-upstream-tsig
```

Semantics:
- `spec.zoneRef` required:
  - Must reference an existing `DNSZone` (same namespace unless explicitly set).
  - Controller derives provider class/configuration from the parent `DNSZone`
    (e.g., `dnsZoneClassName`) and reconciles TSIG state accordingly.
  - Controller sets `OwnerReference` on the `TSIGKey` pointing to the parent
    `DNSZone` to enable garbage collection when the zone is deleted.
- `spec.secretRef` omitted:
  - Controller generates secret material and creates/manages a `Secret` named deterministically.
- `spec.secretRef` provided:
  - Controller reads existing `Secret` (must exist), validates schema, and never overwrites secret material.

Deterministic Secret naming (generated mode):
- `<tsigKey.metadata.name>-tsig` (e.g., `example-com-xfr-tsig`)

TSIG Secret schema (canonical):
- Keys: `name` (wire key name), `algorithm` (e.g., `hmac-sha256`), `secret` (shared secret value)

Example:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-com-xfr-tsig
type: Opaque
stringData:
  name: datum-example-com-xfr
  algorithm: hmac-sha256
  secret: <shared-secret>
```

Status:

```yaml
status:
  secretName: example-com-xfr-tsig
  tsigKeyID: "b7b4c8a2-..."
  conditions:
  - type: Ready
    status: "True"
```

Controller responsibilities:
- Resolve or generate `Secret`.
- Validate secret schema.
- Create/ensure PowerDNS TSIG key exists and matches secret material.
- Write `status.tsigKeyID` and conditions.
- Ownership and GC:
  - Set `OwnerReference` on `TSIGKey` to parent `DNSZone`.
  - Set `OwnerReference` on generated `Secret` to the `TSIGKey` (not on BYO `Secret`).

#### 2) `ZoneTransfer` (new CRD)

Purpose: Represents a zone transfer configuration associated with a `DNSZone`.
Supports both importing from an upstream primary (Secondary behavior) and
exporting transfers from a Datum primary to external secondaries.

Spec overview:

```yaml
apiVersion: dns.datum.net/v1alpha1
kind: ZoneTransfer
metadata:
  name: example-com-import
spec:
  zoneRef:
    name: example-com
    # namespace: default
  # Discriminator indicating the DNS role this transfer config represents
  role: Secondary # or Primary
  # Exactly one of 'secondary' or 'primary' must be specified, matching 'role'
  secondary:
    masters:
      - 198.51.100.10
      - 198.51.100.11
    tsigKeyRef:
      name: example-com-upstream
  # primary:
  #   tsigKeyRef:
  #     name: example-com-xfr
```

Rules:
- `spec.role` is required and must be `Primary` or `Secondary`.
- Each `ZoneTransfer` must specify exactly one of `secondary` or `primary`,
  and it must match `spec.role`.
- At most one `ZoneTransfer` with `role: Secondary` per `DNSZone` (v1).
- At most one `ZoneTransfer` with `role: Primary` per `DNSZone` (v1).
- `secondary` requires non-empty `masters` and a `tsigKeyRef`.
- `primary` requires a `tsigKeyRef`.
- `spec.zoneRef` must reference a `DNSZone` in the same namespace.

Status (illustrative):

```yaml
status:
  conditions:
  - type: Ready
    status: "True"
  lastSyncSerial: 2025012301
  lastSyncTime: "2026-01-08T12:00:00Z"
  lastError: ""
```

Examples:

Primary zone (no transfers):

```yaml
apiVersion: dns.datum.net/v1alpha1
kind: DNSZone
metadata:
  name: example-com
spec:
  domainName: example.com
  dnsZoneClassName: datum-edge
```

Enable outbound transfers for a primary:

```yaml
apiVersion: dns.datum.net/v1alpha1
kind: ZoneTransfer
metadata:
  name: example-com-export
spec:
  zoneRef:
    name: example-com
  role: Primary
  primary:
    tsigKeyRef:
      name: example-com-xfr
```

Configure a secondary (pull from upstream):

```yaml
apiVersion: dns.datum.net/v1alpha1
kind: DNSZone
metadata:
  name: example-com
spec:
  domainName: example.com
  dnsZoneClassName: datum-edge
---
apiVersion: dns.datum.net/v1alpha1
kind: ZoneTransfer
metadata:
  name: example-com-import
spec:
  zoneRef:
    name: example-com
  role: Secondary
  secondary:
    masters:
      - 198.51.100.10
      - 198.51.100.11
    tsigKeyRef:
      name: example-com-upstream
```

#### 3) `DNSZone` (updated CRD)

Purpose: Represents an authoritative DNS zone served by Datum.
The desired role is not configured directly on the spec; instead, the effective
role is derived from the presence (or absence) of an associated `ZoneTransfer` with `role: Secondary`.

Controller responsibilities:

DNSZone Controller
- Ensure provider zone exists (Master/Native for effective Primary; Slave for effective Secondary).
- Records:
  - When effective role is Primary, program provider state from `DNSRecordSet`.
  - When effective role is Secondary, ingest imported zone data from the transfer plane and
    materialize a read-only record inventory (e.g., controller-owned `DNSRecordSet` objects or
    internal canonical records). User-initiated record mutations are rejected.
- Derive and set `status.role`:
  - `Secondary` if an associated `ZoneTransfer` with `role: Secondary` exists and is Ready.
  - `Primary` otherwise.
- Track zone status/conditions.

ZoneTransfer Controller
- `role: Secondary` (Secondary behavior):
  - Ensure provider zone exists as Slave.
  - Configure `masters[]` and TSIG pull (maps to PDNS `slave_tsig_key_ids`).
  - Track transfer health in `ZoneTransfer.status` (serial, last sync, last error).
- `role: Primary` (Primary outbound transfers):
  - Ensure provider zone exists as Master/Native.
  - Resolve Ready `TSIGKey`, read `status.tsigKeyID`, and configure PDNS to allow AXFR/IXFR using that key
    (maps to PDNS primary-side TSIG wiring: `master_tsig_key_ids` / TSIG-ALLOW-AXFR).
  - Track readiness in `ZoneTransfer.status`.

Record visibility and role switching:
- Effective Secondary (`ZoneTransfer` with `role: Secondary` present and Ready) is read-only from a records perspective:
  - Mutations via `DNSRecordSet` are rejected.
  - A read-only record inventory reflecting the imported zone is exposed (API/UI design TBD).
- Switching roles is lifecycle-driven:
  - Secondary → Primary: delete the `ZoneTransfer` with `role: Secondary`. Adopt the currently imported data
    as the initial source-of-truth; subsequent mutations are allowed.
  - Primary → Secondary: create a `ZoneTransfer` with `role: Secondary`, `secondary.masters`, and `secondary.tsigKeyRef`.
    Mutations are disabled; imported data becomes authoritative.

Transfer Plane Runtime:
- `xfr1.datumdomains.net` maps to a static IP (LB / reserved address).
- Cloud NAT (optional) makes outbound transfer traffic originate from a stable IP.
- Secondary imports: transfer plane pulls via AXFR/IXFR, persists to canonical store,
  LMDB replication pushes to edge servers; anycast NS serves from replicated state.

Security Model:
- TSIG is the DNS-layer authorization mechanism.
- Secondary zones always require TSIG to avoid IP‑trust; simplifies multi‑tenant safety.
- Default deny:
  - No outbound transfers unless a `ZoneTransfer` with `role: Primary` exists and is Ready.
  - No secondary imports unless a `ZoneTransfer` with `role: Secondary` exists and is Ready.

Invariants / Validation Rules:
- `spec.role` must be set to `Primary` or `Secondary`.
- Each `ZoneTransfer` must specify exactly one of `secondary` or `primary`, matching `spec.role`.
- At most one `ZoneTransfer` with `role: Secondary` per `DNSZone` (v1).
- At most one `ZoneTransfer` with `role: Primary` per `DNSZone` (v1).
- `secondary` requires non-empty `masters` and a `tsigKeyRef`.
- `primary` requires a `tsigKeyRef`.
- When a `ZoneTransfer` with `role: Secondary` is Ready for a `DNSZone`, `DNSRecordSet` mutations for that zone are not allowed.
- All `TSIGKeyRef`s must point to a Ready `TSIGKey`.
- TSIG `Secret`s must contain `name`, `algorithm`, `secret`.
- `TSIGKey.spec.zoneRef` must reference a `DNSZone` in the same namespace (cross-namespace refs disallowed).

## Production Readiness Review Questionnaire

TBD (alpha)

### Feature Enablement and Rollback

TBD (alpha)

#### How can this feature be enabled / disabled in a live cluster?

TBD

#### Does enabling the feature change any default behavior?

TBD

#### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

TBD

#### What happens if we reenable the feature if it was previously rolled back?

TBD

#### Are there any tests for feature enablement/disablement?

TBD

### Rollout, Upgrade and Rollback Planning

TBD (beta)

#### How can a rollout or rollback fail? Can it impact already running workloads?

TBD

#### What specific metrics should inform a rollback?

TBD

#### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

TBD

#### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

TBD

### Monitoring Requirements

TBD (beta/GA)

#### How can an operator determine if the feature is in use by workloads?

TBD

#### How can someone using this feature know that it is working for their instance?

TBD

#### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

TBD

#### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

TBD

#### Are there any missing metrics that would be useful to have to improve observability of this feature?

TBD

### Dependencies

TBD (beta)

#### Does this feature depend on any specific services running in the cluster?

TBD

### Scalability

TBD

#### Will enabling / using this feature result in any new API calls?

TBD

#### Will enabling / using this feature result in introducing new API types?

TBD

#### Will enabling / using this feature result in any new calls to the cloud provider?

TBD

#### Will enabling / using this feature result in increasing size or count of the existing API objects?

TBD

#### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

TBD

#### Will enabling / using this feature result in non-negligible increase of resource usage in any components?

TBD

#### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

TBD

### Troubleshooting

TBD (beta/GA)

#### How does this feature react if the API server is unavailable?

TBD

#### What are other known failure modes?

TBD

#### What steps should be taken if SLOs are not being met to determine the problem?

TBD

## Implementation History

TBD

## Drawbacks

TBD

## Alternatives

- Embed transfer configuration directly on `DNSZone` with role-specific blocks.
  - Shape:
    - `spec.role: Primary|Secondary`
    - `spec.primary.transfer.tsigKeyRef` (optional, enables outbound transfers)
    - `spec.secondary.masters[]` + `spec.secondary.transfer.tsigKeyRef` (required)
  - Pros:
    - Single resource expresses zone intent; easy discovery (“open the zone”).
    - Fewer objects to manage for simple cases.
    - Role is explicit in spec without derivation.
  - Cons:
    - Couples transfer lifecycle and status to the zone; harder to promote/demote via independent lifecycles.
    - Validation becomes more complex (mutual exclusivity of primary/secondary blocks, mutation rules).
    - Difficult to prevent accidental “multiple transfers” semantics or to support future multiple profiles.
    - Mixes concerns (serving vs transfer orchestration) in one CRD.
  - Decision:
    - Use a separate `ZoneTransfer` CRD per `DNSZone` with `role: Primary|Secondary`.
      This cleanly separates transfer lifecycle, enables default‑deny by absence,
      supports clearer ownership and status, and keeps `DNSZone` focused on serving/records. Revisit embedding
      only if future requirements strongly favor inline configuration over lifecycle separation.

## Infrastructure Needed (Optional)

TBD
