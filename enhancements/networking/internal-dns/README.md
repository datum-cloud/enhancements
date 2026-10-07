---
status: provisional
stage: alpha
latest-milestone: 'October 2026 - "Hedy"'
---

# Internal DNS for Galactic VPC

Tracking issue: [Internal DNS for Galactic VPC (#921)](https://github.com/datum-cloud/enhancements/issues/921)

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

## Proposed user experience

### Automatic private names

When you create a supported resource in a VPC, Datum assigns it a private DNS
name. Workloads receive DNS configuration automatically. You can use the assigned
name without creating a zone or choosing a zone for each resource. Public names
continue to resolve through the same DNS service.

From a workload in the VPC, query an instance's IPv6 address with `dig`:

```console
$ dig +short AAAA web-01.instances.vpc-123.internal.example.net
2001:db8:123::10
```

The examples use the workload's configured resolver. Hostnames and addresses are
illustrative. The final automatic DNS suffix remains undecided.

Datum updates managed records when addresses change and removes them when you
delete the resource. You can view assigned names on the resource and use custom
aliases for names that your applications depend on.

### Custom private zones

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

### VPC isolation

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

### Health-aware service discovery

For supported services, DNS returns usable endpoints and removes endpoints that
the service reports as unhealthy or unavailable. Recovered endpoints become
eligible again. If no usable endpoints remain, the service name returns no
endpoint addresses.

Query a service with two healthy endpoints:

```console
$ dig +short AAAA api.services.vpc-123.internal.example.net
2001:db8:123::21
2001:db8:123::22
```

After the service reports the second endpoint as unhealthy and DNS updates
propagate, the same query returns only the healthy endpoint:

```console
$ dig +short AAAA api.services.vpc-123.internal.example.net
2001:db8:123::21
```

Cached answers can persist until their DNS cache lifetime expires. Applications
still need connection retries. Custom records do not receive automatic health
checks.

### Status and management

You can manage zones and records through the portal, datumctl, and Datum APIs.
You can inspect a zone's associated VPCs, distinguish managed records from custom
records, and see whether a name is ready to resolve. When a name is pending or
unavailable, status explains why.

## Success criteria

- A workload reaches another supported resource by its automatic private name.
- A VPC resolves names from multiple private zones without exposing those names
  to an unassociated VPC.
- Resource changes and service health changes update DNS results within
  documented time limits.

## Out of scope

- Public authoritative DNS hosting.
- Automatic discovery of arbitrary services on connected networks.
- Application load balancing and connection retry behavior.
