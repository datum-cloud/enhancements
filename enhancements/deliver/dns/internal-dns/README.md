---
status: provisional
stage: alpha
latest-milestone: 'October 2026 - "Hedy"'
---

# Internal DNS for Galactic VPC

Tracking issue: [Internal DNS for Galactic VPC (#921)](https://github.com/datum-cloud/enhancements/issues/921)

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-goals](#non-goals)
- [Proposal](#proposal)
  - [User stories](#user-stories)
  - [Notes, constraints, and caveats](#notes-constraints-and-caveats)
  - [Risks and mitigations](#risks-and-mitigations)
- [Design details](#design-details)
- [Production readiness review questionnaire](#production-readiness-review-questionnaire)
- [Implementation history](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure needed](#infrastructure-needed)

## Summary

Internal DNS lets you reach resources and services by name within a Galactic
virtual private cloud (VPC). Datum provides automatic private names, custom
private zones, and service discovery without requiring you to operate DNS
infrastructure.

## Motivation

Tracking IP addresses adds configuration work and breaks when resources move or
change. Private DNS gives applications a consistent way to find Compute
instances, Connector-backed services, and network services across a VPC's
locations.

### Goals

- A workload reaches another supported resource by its automatic private name.
- A VPC resolves names from multiple private zones without exposing those names
  to an unassociated VPC.
- Resource changes and service health changes update DNS results within
  documented time limits.

### Non-goals

- Public authoritative DNS hosting.
- Automatic discovery of arbitrary services on connected networks.
- Application load balancing and connection retry behavior.

## Proposal

Datum provides automatic resource names and optional custom private zones. The
following user stories describe the consumer experience.

### User stories

#### Automatic private names

When you create a supported resource in a VPC, Datum assigns it a private DNS
name. Workloads receive DNS configuration automatically. You can use the assigned
name without creating a zone or choosing a zone for each resource. Public names
continue to resolve through the same DNS service.

Every VPC uses `datum.internal` as its default domain. Automatic names follow
`<assigned-resource-name>.datum.internal`. Namespace, VPC, and project identifiers
do not appear in the domain. Assigned names are unique within the VPC's DNS context.
Each VPC resolves its own records, so the same name can have different answers
in different VPCs.

From a workload in the VPC, query an instance's IPv6 address with `dig`:

```console
$ dig +short AAAA web-01-k7m2.datum.internal.
2001:db8:123::10

$ dig +search +short AAAA web-01-k7m2
2001:db8:123::10
```

Workloads receive `datum.internal` as the default search domain. Short names
resolve within the attached VPC. A fully qualified name does not select another
VPC's DNS context.
Additional private zones join the search list only through explicit configuration.

Datum updates managed records when addresses change and removes them when you
delete the resource. You can view assigned names on the resource and use custom
aliases for names that your applications depend on.

#### Custom private zones

You can create private zones and records for your applications. A zone groups
names under a domain, such as `prod.internal`. You can make multiple zones
available in one VPC and explicitly share a zone with other VPCs you control.

For example, an application can use `api.prod.internal` for a Compute service
and `database.corp.internal` for a database reached through a Connector. You
manage custom records; Datum maintains automatically assigned resource records.

From the same workload, query names in both associated zones:

```console
$ dig +short AAAA api.prod.internal
2001:db8:123::20

$ dig +short AAAA database.corp.internal
2001:db8:123::30
```

#### VPC isolation

Private names resolve only in VPCs associated with their zone. Separate VPCs can
use the same domain and record names with independent answers. For example,
production and development VPCs can each resolve `database.corp.internal` to
their own database.

From a workload in the production VPC:

```console
$ dig +short AAAA database.corp.internal
2001:db8:123::30
```

From a workload in the development VPC:

```console
$ dig +short AAAA database.corp.internal
2001:db8:456::30
```

Connecting two VPCs does not automatically share their private zones. Publishing
a private record does not publish it to public DNS or grant access to its
destination.

#### Health-aware service discovery

For supported services, DNS returns usable endpoints and removes endpoints that
the service reports as unhealthy or unavailable. Recovered endpoints become
eligible again. If no usable endpoints remain, the service name returns no
endpoint addresses.

Query a service with two healthy endpoints:

```console
$ dig +short AAAA api.datum.internal.
2001:db8:123::21
2001:db8:123::22
```

After the service reports the second endpoint as unhealthy and DNS updates
propagate, the same query returns only the healthy endpoint:

```console
$ dig +short AAAA api.datum.internal.
2001:db8:123::21
```

#### Status and management

You can manage zones and records through the portal, datumctl, and Datum APIs.
You can inspect a zone's associated VPCs, distinguish managed records from custom
records, and see whether a name is ready to resolve. When a name is pending or
unavailable, status explains why.

### Notes, constraints, and caveats

The examples use the workload's configured resolver. Resource labels and
addresses are illustrative; `datum.internal` is the default managed domain.

Cached answers can persist until their DNS cache lifetime expires. Applications
still need connection retries. Custom records do not receive automatic health
checks.

### Risks and mitigations

- Accidental name exposure: require explicit zone associations and preserve VPC
  isolation, including when VPCs are connected.
- Stale endpoint answers: expire outdated health reports and document update and
  cache lifetimes.
- Unclear availability: show assigned names, DNS readiness, and reasons for
  pending or unavailable names.

## Design details

The DNS service runs in its own VPC. Consumer VPCs reach it through Galactic
private endpoints. Shared DNS capacity serves multiple VPCs; creating a VPC does
not require a separate DNS deployment. Workloads inherit resolver configuration
from their network.

```mermaid
flowchart LR
  W[Workloads in a consumer VPC] -->|DNS queries| G[Galactic private access]
  G -->|Authorized DNS context| D[Internal DNS service]
  P["Product services such as Compute and Connect"] -->|Publish DNS records| D
  U[Zone owners] -->|Private zones, records, and VPC associations| D
  D -->|Public name resolution| R[Public DNS]
```

The networking integration connects each VPC to an isolated DNS context and
configures workload access. DNS manages zones and resolution. Product services,
such as Compute and Connect, publish and maintain their records. Zone owners
manage custom records and decide which VPCs can resolve a zone.

DNS readiness requires both published records and a working resolver path.
Resource creation can finish before DNS is ready; an assigned name alone does
not confirm that workloads can resolve it.

The [internal DNS architecture](../../../../architecture/deliver/dns/internal-dns/README.md)
defines DNS contexts, Galactic integration, resource placement across control
planes, query and record flows, and failure behavior. The
[Private Service Connect design](https://github.com/datum-cloud/galactic/blob/docs/private-service-connect-architecture/docs/enhancements/networking/private-service-connect/README.md)
defines the private endpoint capability used by DNS. API and implementation
details belong in the component repositories.

### Adoption and release scope

The proposed default enables automatic DNS for new VPCs. Existing VPC adoption
requires a rollout plan that preserves workload DNS configuration and public
resolution. Disabling internal DNS can interrupt applications that use private
names; consumers need a clear description of that effect.

The initial release must identify which product resources support automatic
names and which support health-aware discovery. Those capabilities appear in
product status and documentation.

## Production readiness review questionnaire

Pending implementation design. Complete the questionnaire before changing the
enhancement status to implementable.

### Feature enablement and rollback

Define adoption for existing VPCs and the effect of disabling internal DNS on
workloads that depend on private names.

### Rollout, upgrade, and rollback planning

Define rollout and rollback validation for private names and public resolution.

### Monitoring requirements

Define availability, query latency, and record update targets. Consumers need
DNS readiness and failure reasons on their resources.

### Dependencies

Confirm DNS, Galactic, and product service dependencies and how their outages
affect consumers.

### Scalability

Define supported zone, record, and query limits before release.

### Troubleshooting

Define how consumers diagnose unavailable names, missing endpoints, and stale
answers.

## Implementation history

Initial product proposal: [PR #922](https://github.com/datum-cloud/enhancements/pull/922).

## Drawbacks

Custom zones introduce additional names and VPC associations to manage.
Automatic names cover supported resources without requiring custom zones.

## Alternatives

Manual IP configuration requires updates when addresses change. Operating your
own DNS infrastructure adds setup and maintenance work.

## Infrastructure needed

To be determined during implementation design.
