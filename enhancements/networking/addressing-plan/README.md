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

| Pool                          | Block                          | Purpose                                       | Visibility                                                                                  |
| ----------------------------- | ------------------------------ | --------------------------------------------- | ------------------------------------------------------------------------------------------- |
| VPC (ULA, default)            | `fd00::/8`                     | Tenant overlay addressing                     | Overlay only — never leaves the overlay; see [VPC Address Space Options](#vpc-address-space-options) |
| VPC (GUA, alternative)        | `[VPC-BLOCK]` (ARIN PI)     | Tenant overlay addressing                     | Overlay only — must not be advertised externally; policy enforcement required; see [VPC Address Space Options](#vpc-address-space-options) |
| SRv6 SID locators             | `[LOCATOR-BLOCK]` (ARIN PI) | SRv6 segment identifiers                      | Advertised in the underlay (BGP IPv6 Unicast); reachability is required for SRv6 forwarding |
| Infrastructure loopbacks      | `[INFRA-BLOCK]` (ARIN PI)   | Router loopbacks for underlay and overlay BGP | Underlay-reachable; aggregate-only to external peers                                        |

> **Note on registered blocks:** Both the SRv6 locator block and the infrastructure loopback block must be Provider Independent (PI) space registered with ARIN. Using Provider Aggregatable (PA) space assigned by an upstream carrier is not acceptable — it creates a hard dependency on that carrier and prevents portability. The actual prefix values are placeholders in this document; specific assignments are tracked in Milo IPAM.

The ULA pool (`fd00::/8`) is used exclusively for private tenant networking. These prefixes are never exposed to the underlay, never advertised to transit providers, and never appear in the global DFZ.

The SRv6 SID locator block is a globally registered PI prefix. It is a Datum-registered ARIN PI block, subdivided per PoP. Each PoP's `/48` locator is advertised into the underlay via BGP IPv6 Unicast SAFI — underlay reachability of the locator prefix is required for SRv6 forwarding. It must not overlap with the ULA pool or the infrastructure loopback block.

---

## Per-PoP Allocation

Each PoP receives exactly three `/48` allocations, one from each platform pool.

```
Per PoP
├── /48  ULA private VPC space       (from fd00::/8)
├── /48  SRv6 SID locator space      (from [LOCATOR-BLOCK])
└── /48  Infrastructure loopbacks    (from [INFRA-BLOCK])
         ├── /49  Underlay BGP router loopbacks
         └── /49  Overlay BGP router loopbacks
```

Per-PoP address assignments (actual prefix values) are tracked in Milo's IPAM service. This document defines the allocation scheme; Milo is the source of truth for specific assignments. See: [datum-cloud/enhancements#715](https://github.com/datum-cloud/enhancements/issues/715).

### VPC Address Space Options

The per-PoP VPC address pool is an implementation decision. Two options are defined below; ULA is the default.

**Option A — ULA (`fd00::/8`)**, default

Each PoP's ULA `/48` is drawn from `fd00::/8` and assigned at PoP provisioning time. These prefixes are private by definition and universally filtered by upstreams; there is no policy required to prevent external advertisement.

**Option B — GUA (Datum ARIN PI block)**

Each PoP's GUA `/48` is drawn from a Datum-registered ARIN PI block reserved exclusively for tenant overlay use. The block must not overlap with `[LOCATOR-BLOCK]` or `[INFRA-BLOCK]`. Because GUA space is globally routable, leakage does not fail safe the way ULA does — a misconfigured export policy will result in the prefix appearing in the global DFZ. The policy requirements in [GUA Tenant Addressing Policy](#gua-tenant-addressing-policy) are mandatory, not advisory, whenever Option B is in use.

#### Comparison

| Criterion                        | ULA (`fd00::/8`)                                            | GUA (ARIN PI)                                              |
| -------------------------------- | ----------------------------------------------------------- | ---------------------------------------------------------- |
| RIR registration required        | No                                                          | Yes                                                        |
| Global address uniqueness        | Not guaranteed; overlap possible across organizations       | Guaranteed                                                 |
| Risk of accidental DFZ leakage   | None — upstreams universally filter `fd00::/8`              | Certain if export policy is absent or misconfigured        |
| Multi-cloud / direct interconnect | Overlap risk requires coordination between parties         | No overlap risk; prefixes are unambiguous across providers |
| Bring-your-own-prefix (BYOP)     | Not applicable                                              | Tenants may bring own PI space                             |
| IPAM governance                  | Self-managed; no RIR involvement                            | Requires RIR allocation and lifecycle management           |
| Scale                            | Effectively unbounded                                       | Bounded by allocated block size                            |

ULA is the simpler default for a closed overlay. GUA is preferable if tenants require direct IPv6 interconnects with external networks, or if the platform intends to offer bring-your-own-prefix (BYOP) services.

#### GUA Tenant Addressing Policy

These rules are mandatory whenever GUA VPC addressing (Option B) is in use. They apply to both Datum-allocated GUA space and BYOP tenant space. Failure to enforce them will result in tenant prefixes entering the global DFZ.

**Default deny export.** All prefixes that fall within the GUA VPC block are denied export at every underlay-facing BGP session by default. This is a BGP outbound prefix list applied at the session level, not a route-map condition that depends on community or attribute matching. The deny is unconditional — no match condition can override it without an explicit allowlist entry. There is no opt-out for individual sessions.

**Explicit export intent required.** A GUA VPC prefix may only be exported from the overlay if it appears on a per-session explicit allowlist maintained by the platform operations team. Adding a prefix to the allowlist requires: (1) a documented tenant request, (2) successful prefix ownership validation (see below), (3) a valid RPKI ROA, and (4) change-control approval. Operator convenience is not a valid justification for export. The default is deny; any export is an exception that must be justified, reviewed, and recorded.

**Per-tenant prefix lists.** The export allowlist is maintained at the individual prefix level, not at the covering GUA block level. A single platform-wide community-based policy is insufficient — it creates a single control point whose misconfiguration or exception exposes all tenants. Each tenant's permitted export prefixes must be enumerated explicitly in a per-tenant prefix list. Allowlist membership does not transfer between tenants.

**RPKI and ROA requirements.** Datum must publish ROAs for the platform's GUA VPC block with `maxLength = 48`. This ensures that any advertisement of a more-specific prefix (e.g., an individual `/64`) is RPKI Invalid and will be rejected by ROV-enforcing peers, providing an out-of-band backstop against more-specific leaks. For BYOP prefixes, the tenant must hold a valid ROA that either (a) origin-authorizes Datum's overlay ASN or (b) origin-authorizes the tenant's own ASN if the tenant manages route origination. Prefixes with RPKI Invalid status must not enter the overlay and must not be exported under any circumstances.

**BYOP validation.** A tenant may use their own Provider Independent (PI) IPv6 space as their VPC address block. Before a BYOP prefix is accepted into the overlay, the platform must verify: (1) the tenant's RIR allocation record (ARIN, RIPE, APNIC, or equivalent) confirms the prefix is theirs, (2) the prefix is not currently being announced in the global DFZ by another origin AS, and (3) a valid ROA exists. The maximum prefix length for a BYOP allocation is the allocation's RIR-registered length — sub-allocations that are more specific than the registered block are not accepted without additional RIR evidence. BYOP validation is re-run on a scheduled basis; if a tenant's RIR record lapses, the BYOP prefix is quarantined pending re-validation.

**Maximum prefix length for export.** Only the per-PoP `/48` aggregate may be exported at the underlay boundary. No prefix longer than `/48` may appear in an underlay BGP RIB or be sent to any external peer. This applies to all export paths: transit, peering, and customer interconnects. Individual subnet `/64`s and instance `/128`s must never leave the overlay.

**Route leak detection.** Every underlay BGP session must be monitored for the appearance of GUA VPC prefixes in the peer's received-routes table. A GUA VPC prefix appearing in an underlay RIB is an operational emergency. Route leak detection must integrate with the platform's observability and alerting infrastructure and must page on-call immediately, without waiting for a human to notice.

**Emergency quarantine.** If a GUA VPC prefix is detected outside the overlay:

1. Withdraw all GUA VPC prefix exports from every underlay-facing session immediately.
2. Isolate the affected PoP from underlay BGP peering until the export policy is confirmed clean.
3. Do not re-enable any export until the root cause is identified and the policy defect is corrected.
4. Notify affected tenants.

Emergency quarantine is disruptive to any tenant that legitimately exports GUA prefixes. This is acceptable. A leak into the DFZ is more disruptive and harder to recover from than a controlled withdrawal.

From the PoP's `/48` (regardless of which option is chosen), each consumer VPC at that PoP is allocated a `/64`. A `/48` provides 65,536 possible `/64` allocations — this is the practical capacity ceiling for tenant VPCs at a single PoP, not a hard platform limit. The platform tracks allocations by association (VPC ↔ location) in the IPAM service, not by encoding topology in address bits.

```
PoP VPC /48
└── /64  per consumer VPC at this PoP
     └── /128  per instance / endpoint address
```

### SRv6 Locator /48

Each PoP receives a `/48` from the platform's registered SRv6 locator block. This `/48` is the SID locator for all SRv6 SIDs originated at that PoP (e.g., `End.DT4`, `End.DT6`, `End.DT46` per-VRF SIDs). SIDs are drawn from this block.

The locator block is PoP-scoped. All SIDs for a given PoP share the same `/48` locator prefix, enabling aggregate advertisement and clean per-PoP filtering.

The PoP's `/48` locator prefix is advertised into the underlay via BGP IPv6 Unicast. This is not optional — underlay reachability of the locator is a prerequisite for SRv6 forwarding. Without it, remote PoPs cannot route encapsulated packets to their destination. The aggregate `/48` is advertised, not individual SIDs.

### SRv6 SID Structure

SIDs within a PoP's `/48` locator follow a fixed-length layout: 48-bit locator, 16-bit function, 16-bit argument, 48-bit zero padding.

```
|<------- 48 bits ------->|<-- 16 bits -->|<-- 16 bits -->|<------- 48 bits ------->|
        Locator                Function         Arguments           Padding
   [LOCATOR-BLOCK]:X:Y::/48   e.g. 0x0003       VRF ID              ::0
```

| Field     | Bits | Description                                           |
| --------- | ---- | ----------------------------------------------------- |
| Locator   | 48   | Identifies the PoP; drawn from `[LOCATOR-BLOCK]`   |
| Function  | 16   | Platform-assigned behavior code (see registry below)  |
| Arguments | 16   | PoP-local VRF ID; identifies the tenant VRF           |
| Padding   | 48   | Zero; not used                                        |

#### Function Code Registry

The function field encodes the SRv6 behavior applied at the endpoint. Function codes are platform-wide constants — they do not vary per PoP. The locator already identifies the PoP; the function identifies what to do there.

| Code     | Behavior   | Description                                                      |
| -------- | ---------- | ---------------------------------------------------------------- |
| `0x0000` | Reserved   | Invalid / null; must not be allocated                            |
| `0x0001` | `End.DT4`  | Decapsulate and lookup in the tenant IPv4 table                  |
| `0x0002` | `End.DT6`  | Decapsulate and lookup in the tenant IPv6 table                  |
| `0x0003` | `End.DT46` | Decapsulate and lookup in the tenant IP table (dual-stack)       |
| `0x0004`–`0x00FF` | Reserved | Reserved for future platform behaviors              |
| `0x0100`–`0xFFFE` | Unallocated | Available for future assignment                    |
| `0xFFFF` | Reserved   | Must not be allocated                                            |

`End.DT4`, `End.DT6`, and `End.DT46` are distinct behaviors (RFC 8986 §4.1). Each requires its own SID. They differ in which IP table lookup is performed after decapsulation — they are not interchangeable, and a dual-stack VRF cannot substitute `End.DT46` for both `End.DT4` and `End.DT6` unless the implementation's VRF holds both address families in a single table.

In FRR and the Linux kernel `seg6local` dataplane, `End.DT4` and `End.DT6` look up separate per-AF VRF tables, while `End.DT46` performs a single lookup against a combined table. The platform uses `End.DT46` for dual-stack VRFs where FRR is configured with a single VRF per tenant. `End.DT4` and `End.DT6` are retained in the registry for deployments that require per-AF VRF separation.

#### Argument Semantics: VRF ID

The argument field carries a 16-bit VRF ID. VRF IDs are **PoP-local**: the same value may be reused at different PoPs without conflict because the locator prefix already disambiguates per PoP. The SID as a whole — locator + function + argument — is globally unique.

VRF ID `0x0000` is reserved and must not be allocated.

The IPAM service allocates VRF IDs sequentially within each PoP's namespace when a tenant VPC is first instantiated at that PoP. The VRF ID assigned at one PoP has no relationship to the VRF ID assigned for the same tenant VPC at a different PoP — IPAM tracks the mapping of tenant VPC to VRF ID independently per PoP.

#### SIDs per Tenant VRF

A tenant VRF requires one SID per behavior instantiated at the PoP. All SIDs for a given VRF share the same argument (VRF ID) and differ only in the function code.

| VRF type            | SIDs required                       |
| ------------------- | ----------------------------------- |
| IPv4-only           | 1 — `End.DT4` (`0x0001`)            |
| IPv6-only           | 1 — `End.DT6` (`0x0002`)            |
| Dual-stack (single VRF table) | 1 — `End.DT46` (`0x0003`) |
| Dual-stack (per-AF VRF tables) | 2 — `End.DT4` + `End.DT6` |

The dual-stack single-table configuration (`End.DT46` only) is the platform default for FRR deployments. The per-AF configuration is supported but requires explicit operator intent.

#### VRF ID Space and Exhaustion

The 16-bit argument field provides 65,535 usable VRF IDs per PoP (excluding reserved `0x0000`). At expected tenant densities this is unlikely to be exhausted, but the procedure is defined here to avoid an unplanned operational event.

When a PoP reaches 80% utilization (52,429 VRF IDs allocated), IPAM must alert so that the following expansion can be completed before exhaustion:

1. Allocate a second `/48` locator to the PoP from the SRv6 locator block.
2. Advertise the second locator as a separate BGP aggregate into the underlay — do not summarize the two locators into a covering prefix unless the bits are contiguous and the aggregate does not cover unallocated space.
3. Configure the dataplane (FRR SRv6 locator block) to recognize both locator prefixes as local.
4. New VRF ID allocations at this PoP draw from the second locator's namespace. Existing SIDs under the original locator are not disturbed.

This expansion is non-disruptive to existing SIDs.

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

> **Open item:** The `[INFRA-BLOCK]` actual prefix value must be confirmed as non-overlapping with `[LOCATOR-BLOCK]` before this plan is finalized. This is a prerequisite for the plan to be actionable.

---

## Capacity and Scaling

| Resource                       | Notes                                                                        |
| ------------------------------ | ---------------------------------------------------------------------------- |
| SRv6 locator `/48`s            | One per PoP; additional ARIN PI blocks required as the platform scales       |
| Infrastructure loopback `/48`s | One per PoP; additional ARIN PI blocks required as the platform scales       |
| ULA VPC space                  | Effectively unbounded (`fd00::/8`)                                           |

Additional SRv6 locator and infrastructure loopback blocks must be registered with ARIN before existing allocations are exhausted. Procurement lead time for PI space is measured in weeks to months; initiate the process well in advance of exhaustion. Block sizing and procurement thresholds are tracked in Milo IPAM.

---

## Allocation Hierarchy Summary

| Resource                   | Block      | Source                                  |
| -------------------------- | ---------- | --------------------------------------- |
| Platform ULA pool          | `fd00::/8` | RFC 4193 ULA                            |
| Per-PoP ULA VPC space      | `/48`      | From `fd00::/8`                         |
| Per-PoP SRv6 locator       | `/48`      | ARIN PI (SRv6 locator block)            |
| Per-PoP loopback block     | `/48`      | ARIN PI (infrastructure loopback block) |
| Per-PoP underlay loopbacks | `/49`      | From PoP loopback `/48`                 |
| Per-PoP overlay loopbacks  | `/49`      | From PoP loopback `/48`                 |
| Per-VPC at a PoP           | `/64`      | From PoP ULA `/48`                      |
| Per-instance address       | `/128`     | From subnet `/64`                       |
| Per-router loopback        | `/128`     | From PoP `/49`                          |
---

## Isolation and Security Properties

**VPC space never leaves the overlay.** The VPC pool — whether ULA or GUA — must not be advertised to the underlay, leaked to transit providers, or made reachable from outside the overlay. For ULA (`fd00::/8`), this is fail-safe: upstreams universally filter `fd00::/8`, so a misconfiguration does not result in global reachability. For GUA, there is no automatic filtering backstop — a missing or misconfigured export policy will result in the prefix entering the global DFZ. GUA VPC addressing therefore requires the full set of controls defined in [GUA Tenant Addressing Policy](#gua-tenant-addressing-policy): default deny export, explicit allowlist per tenant, RPKI ROA enforcement, route leak detection, and a defined emergency quarantine procedure. Any advertisement of VPC prefixes outside the overlay is a misconfiguration regardless of which option is in use.

**Locator and infrastructure blocks do not overlap.** The SRv6 locator block (`[LOCATOR-BLOCK]`) and the infrastructure loopback block (`[INFRA-BLOCK]`) must be drawn from distinct, non-overlapping allocations confirmed at RIR registration time. Routing policy must be able to distinguish them unambiguously.

**Infrastructure loopbacks are not reachable from tenant networks.** Tenant VRFs must not have routes to the infrastructure loopback block. This is enforced by import policy at the VRF handoff point — the `[INFRA-BLOCK]` aggregate must appear in a prefix deny list applied to all tenant VRF imports.

**External advertisement uses aggregates only.** For PoPs with Internet transit that require loopback reachability from external paths, only the PoP-level aggregate (the `/48` covering that PoP's loopbacks) may be advertised. Individual `/128`s must never appear in the global DFZ.

> **What is the DFZ?** The Default-Free Zone (DFZ) is the set of Internet core routers — operated by Tier-1 carriers such as Lumen, NTT, and Cogent — that carry a full global BGP routing table with no default route. If a prefix is not in the table, the packet is dropped. Advertising individual `/128` loopbacks into the DFZ is operationally hostile: it bloats the global routing table, exposes internal infrastructure topology, and will be filtered by most peers regardless — IPv6 prefix length limits at the DFZ boundary are typically `/48` maximum. Aggregates only.

---

## Operational Notes

- The per-PoP `/48` ULA allocation should be drawn from a contiguous per-region range within `fd00::/8`, enabling eventual region-level aggregation if route scale management becomes necessary.
- As the platform scales, additional ARIN PI blocks must be obtained for both the SRv6 locator and infrastructure loopback pools. Initiate RIR procurement well before existing allocations are exhausted; lead time for PI space is measured in weeks to months.
- Default VPC allocation is `/64` per VPC. Projects requiring more than 65,536 subnets per PoP, or that need larger per-VPC blocks, require a non-default allocation request. The platform reserves the right to make larger allocations on a per-request basis.
- All address assignments are tracked by Milo's IPAM service (`milo-os/ipam`). Allocation is triggered when a project enables networking services at a PoP.
- Per-PoP address register (actual prefix assignments by PoP): tracked in Milo IPAM — [datum-cloud/enhancements#715](https://github.com/datum-cloud/enhancements/issues/715).
