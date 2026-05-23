# Datum Cloud — IP Addressing Plan

**Status:** Draft
**Source:** Derived from [network-services-operator#163](https://github.com/datum-cloud/network-services-operator/issues/163) and [underlay network requirements](../underlay-network/README.md)

---

## Overview

This document defines the IPv6 address allocation strategy for the Datum Cloud platform. It covers three distinct address domains: private tenant VPC addressing, SRv6 SID locator space, and infrastructure loopback addressing. Each domain is allocated independently and must not overlap.

All infrastructure is IPv6-only. IPv4 addressing within the overlay is out of scope for this document.

**Open pre-condition:** The actual prefix values for the SRv6 locator block and infrastructure loopback block are placeholders throughout this document. Both are ARIN PI allocations; specific prefixes must be confirmed non-overlapping and registered in Milo IPAM before this plan is finalized.

---

## Platform-Wide Address Pools

The platform draws from three top-level IPv6 address pools. These pools are managed centrally and subdivided per PoP.

| Pool                     | Block                          | Purpose                                       | Visibility                                                                                  |
| ------------------------ | ------------------------------ | --------------------------------------------- | ------------------------------------------------------------------------------------------- |
| ULA private VPC          | `fc00::/8`                     | Tenant overlay addressing                     | Overlay only — never leaves the overlay                                                     |
| SRv6 SID locators        | `[LOCATOR-BLOCK]/33` (ARIN PI) | SRv6 segment identifiers                      | Advertised in the underlay (BGP IPv6 Unicast); reachability is required for SRv6 forwarding |
| Infrastructure loopbacks | `[INFRA-BLOCK]/33` (ARIN PI)   | Router loopbacks for underlay and overlay BGP | Underlay-reachable; aggregate-only to external peers                                        |

> **Note on registered blocks:** Both the SRv6 locator block and the infrastructure loopback block must be Provider Independent (PI) space registered with ARIN. Using Provider Aggregatable (PA) space assigned by an upstream carrier is not acceptable — it creates a hard dependency on that carrier and prevents portability. The actual prefix values are placeholders in this document; specific assignments are tracked in Milo IPAM.

The ULA pool (`fc00::/8`) is used exclusively for private tenant networking. These prefixes are never exposed to the underlay, never advertised to transit providers, and never appear in the global DFZ.

The SRv6 SID locator block is a globally registered PI prefix. It is a Datum-registered ARIN PI block, subdivided per PoP. Each PoP's `/48` locator is advertised into the underlay via BGP IPv6 Unicast SAFI — underlay reachability of the locator prefix is required for SRv6 forwarding. It must not overlap with the ULA pool or the infrastructure loopback block.

---

## Per-PoP Allocation

Each PoP receives exactly three `/48` allocations, one from each platform pool.

```
Per PoP
├── /48  ULA private VPC space       (from fc00::/8)
├── /48  SRv6 SID locator space      (from [LOCATOR-BLOCK]/33)
└── /48  Infrastructure loopbacks    (from [INFRA-BLOCK]/33)
         ├── /49  Underlay BGP router loopbacks
         └── /49  Overlay BGP router loopbacks
```

Per-PoP address assignments (actual prefix values) are tracked in Milo's IPAM service. This document defines the allocation scheme; Milo is the source of truth for specific assignments. See: [datum-cloud/enhancements#715](https://github.com/datum-cloud/enhancements/issues/715).

### ULA /48 — Private VPC Addressing

Each PoP's ULA `/48` is the pool from which tenant VPC prefixes at that PoP are allocated. This space is entirely internal to the overlay and is never reachable from outside it.

From the PoP's `/48`, each consumer VPC at that PoP is allocated a `/64`. A `/48` provides 65,536 possible `/64` allocations — this is the practical capacity ceiling for tenant VPCs at a single PoP, not a hard platform limit. The platform tracks allocations by association (VPC ↔ location) in the IPAM service, not by encoding topology in address bits.

```
PoP ULA /48
└── /64  per consumer VPC at this PoP
     └── /128  per instance / endpoint address
```

### SRv6 Locator /48

Each PoP receives a `/48` from the platform's registered SRv6 locator block. This `/48` is the SID locator for all SRv6 SIDs originated at that PoP (e.g., `End.DT4`, `End.DT6`, `End.DT46` per-VRF SIDs). SIDs are drawn from this block.

The locator block is PoP-scoped. All SIDs for a given PoP share the same `/48` locator prefix, enabling aggregate advertisement and clean per-PoP filtering.

The PoP's `/48` locator prefix is advertised into the underlay via BGP IPv6 Unicast. This is not optional — underlay reachability of the locator is a prerequisite for SRv6 forwarding. Without it, remote PoPs cannot route encapsulated packets to their destination. The aggregate `/48` is advertised, not individual SIDs.

### SRv6 SID Structure

SIDs within a PoP's `/48` locator follow a fixed-length layout: 48-bit locator, 32-bit function, 16-bit argument.

```
|<------- 48 bits ------->|<------- 32 bits ------->|<-- 16 bits -->|
        Locator                    Function               Arguments
   [LOCATOR-BLOCK]:X:Y::/48       e.g. End.DT4/VRF         VRF ID
```

| Field     | Bits | Description                                           |
| --------- | ---- | ----------------------------------------------------- |
| Locator   | 48   | Identifies the PoP; drawn from `[LOCATOR-BLOCK]/33`   |
| Function  | 32   | Identifies the SRv6 behavior (e.g., End.DT4 per VRF)  |
| Arguments | 16   | Behavior-specific; used for VRF/tenant disambiguation |
VRF-specific SIDs (one per tenant VRF per PoP) are allocated from the function space.

### Infrastructure Loopback /48

Each PoP receives a `/48` from the infrastructure loopback block. This `/48` is split into two `/49`s:

| Sub-block    | Assignment                    |
| ------------ | ----------------------------- |
| First `/49`  | Underlay BGP router loopbacks |
| Second `/49` | Overlay BGP router loopbacks  |
Individual loopbacks are `/128`s allocated from the appropriate `/49`. Every underlay router (route reflectors at both tiers, and worker nodes with underlay BGP sessions) takes one loopback from the first `/49`. Every overlay BGP node takes one loopback from the second `/49`.

Loopback addresses are stable across reboots and control plane restarts. The underlay advertises them and ensures they remain reachable for the lifetime of the node.

Per the security requirements in the underlay document, these loopbacks must not be reachable from tenant networks. This is enforced by import policy on each tenant VRF — the infrastructure loopback block must be present in a prefix deny list applied at VRF handoff.

External advertisement — required for Internet transit PoPs — uses the covering PoP-level aggregate (`/48`), not individual `/128`s.

> **Open item:** The `[INFRA-BLOCK]/33` actual prefix value must be confirmed as non-overlapping with `[LOCATOR-BLOCK]/33` before this plan is finalized. This is a prerequisite for the plan to be actionable.

---

## Capacity and Scaling

| Resource                 | Capacity per Registered Block      | Notes                                    |
| ------------------------ | ---------------------------------- | ---------------------------------------- |
| SRv6 locators            | 32 PoPs per `/33`                  | Additional `/33` required beyond 32 PoPs |
| Infrastructure loopbacks | 32 PoPs per `/33`                  | Additional `/33` required beyond 32 PoPs |
| ULA VPC space            | Effectively unbounded (`fc00::/8`) | —                                        |
**Action required:** Initiate RIR registration for a second `/33` pair (SRv6 locators and infrastructure loopbacks) before the platform reaches 28 PoPs, leaving a 4-PoP buffer for RIR lead time.

---

## Allocation Hierarchy Summary

| Resource                              | Block                | Source                    |
| ------------------------------------- | -------------------- | ------------------------- |
| Platform ULA pool                     | `fc00::/8`           | RFC 4193 ULA              |
| Platform SRv6 locator pool            | `[LOCATOR-BLOCK]/33` | ARIN PI                   |
| Platform infrastructure loopback pool | `[INFRA-BLOCK]/33`   | ARIN PI                   |
| Per-PoP ULA VPC space                 | `/48`                | From `fc00::/8`           |
| Per-PoP SRv6 locator                  | `/48`                | From `[LOCATOR-BLOCK]/33` |
| Per-PoP loopback block                | `/48`                | From `[INFRA-BLOCK]/33`   |
| Per-PoP underlay loopbacks            | `/49`                | From PoP loopback `/48`   |
| Per-PoP overlay loopbacks             | `/49`                | From PoP loopback `/48`   |
| Per-VPC at a PoP                      | `/64`                | From PoP ULA `/48`        |
| Per-instance address                  | `/128`               | From subnet `/64`         |
| Per-router loopback                   | `/128`               | From PoP `/49`            |
---

## Isolation and Security Properties

**ULA space never leaves the overlay.** The `fc00::/8` pool is not advertised to the underlay, not leaked to transit providers, and not reachable from outside the overlay. Any router or policy that would advertise these prefixes externally is a misconfiguration.

**Locator and infrastructure blocks do not overlap.** The SRv6 locator block (`[LOCATOR-BLOCK]/33`) and the infrastructure loopback block (`[INFRA-BLOCK]/33`) must be drawn from distinct, non-overlapping allocations confirmed at RIR registration time. Routing policy must be able to distinguish them unambiguously.

**Infrastructure loopbacks are not reachable from tenant networks.** Tenant VRFs must not have routes to the infrastructure loopback block. This is enforced by import policy at the VRF handoff point — the `[INFRA-BLOCK]/33` aggregate must appear in a prefix deny list applied to all tenant VRF imports.

**External advertisement uses aggregates only.** For PoPs with Internet transit that require loopback reachability from external paths, only the PoP-level aggregate (the `/48` covering that PoP's loopbacks) may be advertised. Individual `/128`s must never appear in the global DFZ.

> **What is the DFZ?** The Default-Free Zone (DFZ) is the set of Internet core routers — operated by Tier-1 carriers such as Lumen, NTT, and Cogent — that carry a full global BGP routing table with no default route. If a prefix is not in the table, the packet is dropped. Advertising individual `/128` loopbacks into the DFZ is operationally hostile: it bloats the global routing table, exposes internal infrastructure topology, and will be filtered by most peers regardless — IPv6 prefix length limits at the DFZ boundary are typically `/48` maximum. Aggregates only.

---

## Operational Notes

- The per-PoP `/48` ULA allocation should be drawn from a contiguous per-region range within `fc00::/8`, enabling eventual region-level aggregation if route scale management becomes necessary.
- As the platform scales, additional registered blocks must be obtained for both the SRv6 locator and infrastructure loopback pools. Initiate RIR procurement at 28 PoPs to maintain a buffer for lead time.
- Default VPC allocation is `/64` per VPC. Projects requiring more than 65,536 subnets per PoP, or that need larger per-VPC blocks, require a non-default allocation request. The platform reserves the right to make larger allocations on a per-request basis.
- All address assignments are tracked by Milo's IPAM service (`milo-os/ipam`). Allocation is triggered when a project enables networking services at a PoP.
- Per-PoP address register (actual prefix assignments by PoP): tracked in Milo IPAM — [datum-cloud/enhancements#715](https://github.com/datum-cloud/enhancements/issues/715).
