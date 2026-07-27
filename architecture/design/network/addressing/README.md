# Platform — Addressing Plan

## Overview

This document defines the platform's address allocation design. It covers the top-level address pools, per-PoP allocation structure, and the isolation properties that govern all address space on the platform.

The platform's underlay and internal infrastructure are IPv6-only. Fabric-level addressing — internal infrastructure subnets, backbone links, SRv6 locator advertisement, and infrastructure loopbacks — is defined in the [Fabric Addressing Plan](fabric.md). Tenant-facing VPC addressing supports both IPv6 (ULA by default, GUA optional) and IPv4 as a separate, additional address family — see the [Tenant Addressing Plan](tenant.md), which is authoritative on VPC, region, and endpoint sizing for both protocols. This document describes the platform-level pools and isolation properties that tenant space must respect. Whether IPv4 is provisioned by default for a given VPC is a platform provisioning decision outside the scope of either document.

> [!NOTE]
> The actual prefix values for the SRv6 locator block and infrastructure loopback block are placeholders throughout this document. Both are RIR PI allocations; specific prefixes must be confirmed non-overlapping and registered in the IPAM service before this plan is finalized.

---

## Platform-Wide Address Pools

The platform draws from three top-level IPv6 address pools. These pools are managed centrally and subdivided per PoP.

| Pool                     | Block                         | Purpose                                    | Visibility                                                                                                                                 |
|--------------------------|-------------------------------|--------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| VPC (ULA, default)       | `fd20::/20` (from `fd00::/8`) | Tenant overlay addressing                  | Overlay only — never leaves the overlay; see [VPC Address Space Options](#vpc-address-space-options)                                       |
| VPC (GUA, alternative)   | `[VPC-BLOCK]` (RIR PI)        | Tenant overlay addressing                  | Overlay only — must not be advertised externally; policy enforcement required; see [VPC Address Space Options](#vpc-address-space-options) |
| SRv6 SID locators        | `[LOCATOR-BLOCK]` (RIR PI)    | SRv6 segment identifiers                   | Per-node `/64` advertised via BGP within PoP only; `/48` aggregate for external reachability                                            |
| Infrastructure loopbacks | `[INFRA-BLOCK]` (RIR PI)      | Router loopbacks (single address per node) | Underlay-reachable; aggregate-only to external peers                                                                                       |

> [!NOTE]
> The VPC pool rows (ULA and GUA) are mutually exclusive. Each PoP uses either ULA (`fd00::/8`) or a GUA block (RIR PI) for tenant overlay addressing — never both. See [VPC Address Space Options](#vpc-address-space-options).

> [!NOTE]
> Both the SRv6 locator block and the infrastructure loopback block must be Provider Independent (PI) space registered with a regional internet registry (RIR). Using Provider Aggregatable (PA) space assigned by an upstream carrier is not acceptable — it creates a hard dependency on that carrier and prevents portability. The actual prefix values are placeholders in this document; specific assignments are tracked in the IPAM service.

> [!NOTE]
> **Regional internet registries.** A regional internet registry (RIR) is a non-profit organization that manages the allocation and registration of Internet number resources (IPv4 and IPv6 address space and AS numbers) within a defined geographic region. Provider Independent (PI) space is obtained from an RIR. The five RIRs are:
>
> | RIR      | Region                            |
> |----------|-----------------------------------|
> | ARIN     | North America                     |
> | RIPE NCC | Europe, Middle East, Central Asia |
> | APNIC    | Asia-Pacific                      |
> | LACNIC   | Latin America and the Caribbean   |
> | AFRINIC  | Africa                            |

The ULA space (`fd00::/8`) serves two separate purposes on this platform: the tenant VPC pool (`fd20::/20`, see the [Tenant Addressing Plan](tenant.md)) and each PoP's internal infrastructure `/48` (see [Fabric Addressing Plan — Internal Infrastructure Addressing](fabric.md#internal-infrastructure-addressing-ula-48)). Both are non-overlapping sub-ranges of the same RFC 4193 ULA space, tracked separately by IPAM. These prefixes are private by definition (RFC 4193 §4.1 mandates default filtering of `fc00::/7` at exterior BGP sessions) and are treated as bogon space by essentially all transit providers and IXPs, giving ULA a strong filtering backstop GUA lacks — though this is an operational convention enforced by other operators' configurations, not a structural internet guarantee, so no policy is required on this platform's side to prevent external advertisement, though outbound filtering at the PoP border remains good defense-in-depth practice.

> [!NOTE]
> The Default-Free Zone (DFZ) is the set of Internet core routers that carry a full global BGP routing table with no default route. Advertising individual `/128` loopbacks into the DFZ is operationally hostile: it bloats the global routing table, exposes internal infrastructure topology, and will be filtered by most peers regardless — IPv6 prefix length limits at the DFZ boundary are typically `/48` maximum. Aggregates only.

The SRv6 SID locator block is a globally registered PI prefix. It is a platform-registered RIR PI block, subdivided per PoP. Within a PoP, each node advertises its own `/64` locator via BGP IPv6 Unicast — the `/48` itself is never advertised as an aggregate route inside the PoP, since that would create an anycast route ambiguous as to which node owns a given SID (see [Fabric Addressing Plan — SRv6 Locator /48](fabric.md#srv6-locator-48)). Because upstream peers accept only `/48` or larger, the `/64` locator is PoP-internal and must never be advertised to external peers. The `/48` remains the unit of RIR registration and of any inter-PoP or external advertisement. It must not overlap with the ULA pool or the infrastructure loopback block.

---

## Per-PoP Allocation

Each PoP receives exactly three `/48` allocations, one from each platform pool.

> [!NOTE]
> **Point of presence (PoP).** A geographic location where the platform deploys networking and compute infrastructure. Each PoP is an independent address allocation unit — each receives its own set of `/48` blocks from the platform pools.


```
Per PoP
├── /48  ULA internal infrastructure (from fd00::/8)
├── /48  SRv6 SID locator space      (from [LOCATOR-BLOCK])
└── /48  Infrastructure loopbacks    (from [INFRA-BLOCK])
     └── /128  per node
```

Tenant VPC addressing is independent of this per-PoP allocation. Each VPC receives a CIDR block (ULA, IPAM-assigned by default, or GUA) subnetted across PoPs; see [VPC Address Space Options](#vpc-address-space-options).

Per-PoP address assignments (actual prefix values) are tracked in the platform's IPAM service. This document defines the allocation scheme; the IPAM service is the source of truth for specific assignments.

### VPC Address Space Options

Tenant VPC addressing is fully abstracted from the provider's infrastructure locator space. Each VPC's contiguous CIDR block — either an IPAM-assigned ULA block or the tenant's own globally unique allocation — is subnetted across PoPs and availability zones as needed. The tenant's IP space travels strictly as the inner payload inside the SRv6 encapsulation; it is agnostic to the physical PoP's locator block and never advertised into the underlay. Two addressing models are supported; ULA is the default.

**Option A — ULA (`fd00::/8`)**, default

Tenants are assigned a ULA `/48` VPC allocation from the platform's tenant ULA pool, `fd20::/20` — a fixed, platform-managed sub-range of `fd00::/8`, issued centrally by IPAM at VPC creation. Full sizing (region and endpoint subnetting below the VPC `/48`) is defined in the [Tenant Addressing Plan — VPC Allocation](tenant.md#vpc-allocation-48). These prefixes are private by definition and are treated as bogon space by essentially all transit providers and IXPs, so no policy is required on this platform's side to prevent external advertisement.

This is a deliberate departure from RFC 4193 §3.2.1's general guidance that Global IDs be self-generated with a pseudo-random algorithm rather than drawn from a narrow shared sub-range — narrowing the range would otherwise reduce effective entropy and raise collision probability, per the analysis in §3.2.3. That collision risk is specific to *tenant-generated* Global IDs. Here, uniqueness is instead guaranteed structurally: IPAM is the sole issuer of `/48`s from `fd20::/20`, so no two VPCs can ever receive the same block regardless of the sub-range's size.

**Option B — GUA (tenant-owned or platform-allocated)**

Tenants use their own globally unique IPv6 allocation — either a Provider Independent (PI) block obtained from a RIR, a platform-allocated block, or a BYOP prefix. Because GUA space is globally routable, leakage does not fail safe the way ULA does — a misconfigured export policy will result in the prefix appearing in the global DFZ. The policy requirements in [GUA Tenant Addressing Policy](#gua-tenant-addressing-policy) are mandatory, not advisory, whenever Option B is in use.

#### Comparison

| Criterion                         | ULA (`fd00::/8`)                                                                                                             | GUA (RIR PI)                                                                       |
|-----------------------------------|------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| RIR registration required         | No                                                                                                                           | Yes                                                                                |
| Global address uniqueness         | Not guaranteed; overlap possible across organizations                                                                        | Guaranteed                                                                         |
| Risk of accidental DFZ leakage    | Very low — no known reachability; upstreams conventionally filter `fd00::/8` as bogon space (not a protocol-level guarantee) | High if export policy is absent or misconfigured — no automatic filtering backstop |
| Multi-cloud / direct interconnect | Overlap risk requires coordination between parties                                                                           | No overlap risk; prefixes are unambiguous across providers                         |
| Bring-your-own-prefix (BYOP)      | Not applicable                                                                                                               | Tenants may bring own PI space                                                     |
| IPAM governance                   | Self-managed; no RIR involvement                                                                                             | Requires RIR allocation and lifecycle management                                   |
| Scale                             | Bounded — ~268.4 million `/48` VPCs (`fd20::/20`); see [Tenant Addressing Plan](tenant.md#ipv6-capacity-summary)             | Bounded by allocated block size                                                    |

ULA is the simpler default for a closed overlay. GUA is preferable if tenants require direct IPv6 interconnects with external networks, or if the platform intends to offer bring-your-own-prefix (BYOP) services.

#### GUA Tenant Addressing Policy

These rules are mandatory whenever GUA VPC addressing (Option B) is in use. They apply to both platform-allocated GUA space and BYOP tenant space. Failure to enforce them will result in tenant prefixes entering the global DFZ.

**Default deny export.** All prefixes that fall within the GUA VPC block are denied export at every underlay-facing BGP session by default. There is no opt-out for individual sessions.

**Explicit export intent required.** A GUA VPC prefix may only be exported from the overlay if it appears on a per-session explicit allowlist. Adding a prefix to the allowlist requires: (1) a documented tenant request, (2) successful prefix ownership validation (see below), and (3) a valid RPKI ROA. The default is deny; any export is an exception that must be justified and recorded.

**Per-tenant prefix lists.** The export allowlist is maintained at the individual prefix level, not at the covering GUA block level. A single platform-wide community-based policy is insufficient — it creates a single control point whose misconfiguration or exception exposes all tenants. Each tenant's permitted export prefixes must be enumerated explicitly in a per-tenant prefix list. Allowlist membership does not transfer between tenants.

**RPKI and ROA requirements.** The operator must publish ROAs for the platform's GUA VPC block with `maxLength = 48`. Per RFC 6483's ROA validation algorithm, this ensures any advertisement of a more-specific prefix (e.g., an individual `/64`) is classified RPKI Invalid; whether an Invalid route is actually dropped or merely deprioritized is a matter of each receiving peer's local policy (RFC 6811 §5 permits either), though in practice many transit and peering networks do filter Invalid routes outright, so this ROA still functions as a meaningful out-of-band backstop against more-specific leaks. For BYOP prefixes, the tenant must hold a valid ROA that either (a) origin-authorizes the platform's overlay ASN or (b) origin-authorizes the tenant's own ASN if the tenant manages route origination.

**Route Origin Validation (ROV) state policy.** Routes with RPKI Invalid status must not enter the overlay or be exported. Routes in the Unknown state (no matching ROA) are accepted by default to accommodate operators that have not yet enrolled in RPKI. The platform must support tightening this policy to drop Unknown routes as RPKI coverage matures.

**BYOP validation.** A tenant may use their own Provider Independent (PI) IPv6 space as their VPC address block. Before a BYOP prefix is accepted into the overlay, the platform must verify: (1) the tenant's RIR allocation record (ARIN, RIPE, APNIC, or equivalent) confirms the prefix is theirs, (2) the tenant demonstrates current control of the prefix via a platform-issued challenge — publishing a platform-provided token or certificate in the prefix's RDAP record (analogous to AWS's X.509-certificate-in-RDAP plus signed authorization message), or creating a PTR record under the prefix resolving to a platform-issued token (analogous to GCP's reverse-DNS validation), (3) the prefix is not currently being announced in the global DFZ by another origin AS (a migration/conflict-safety check, not itself an ownership proof), (4) a valid ROA exists, and (5) properly registered IRR objects are in place authorizing the platform's AS-SET to originate the prefix. The platform's route filtering arrangement with upstream providers is based purely on AS-SET membership and IRR-derived filters; without correct IRR objects, the upstream will not accept the route. The maximum prefix length for a BYOP allocation is the allocation's RIR-registered length — sub-allocations that are more specific than the registered block are not accepted without additional RIR evidence. The platform must periodically re-validate BYOP prefixes; if a tenant's RIR record lapses or IRR objects become stale, the BYOP prefix is quarantined pending re-validation.

**Maximum prefix length for export.** Only the per-PoP `/48` aggregate may be exported at the underlay boundary — this is the platform's own conservative export ceiling, chosen to align with widely-adopted (though not RFC-mandated) DFZ filtering practice among major transit providers, which commonly cap accepted IPv6 prefix length at `/48`. No prefix longer than `/48` may appear in an underlay BGP RIB or be sent to any external peer, regardless of whether a given peer's own filters would accept something more specific. This applies to all export paths: transit, peering, and customer interconnects. Individual subnet `/64`s and instance `/128`s must never leave the overlay.

**Route leak detection.** The platform must detect the appearance of GUA VPC prefixes in underlay BGP sessions and trigger an emergency quarantine procedure. The platform must also monitor third-party route-collector data for unexpected origin ASes or more-specific announcements in the global DFZ.

**Emergency quarantine.** If a GUA VPC prefix is detected outside the overlay, the platform must withdraw all GUA VPC exports and isolate the affected PoP from underlay BGP peering until the root cause is identified and corrected.

#### Tenant Subnet Allocation

Tenant subnet and endpoint sizing is defined in full in the [Tenant Addressing Plan](tenant.md), which is canonical for both protocols:

- **IPv6 (ULA default):** every VPC receives a `/48` from the tenant ULA pool (`fd20::/20`), subdivided into one `/64` per region and one `/96` per endpoint within that region — see [Tenant Addressing Plan — IPv6 Tenant Addressing](tenant.md#ipv6-tenant-addressing-ula).
- **IPv4 (additional, per-VPC address family):** a VPC using IPv4 gets the platform's fixed default scheme — the shared `10.128.0.0/9` pool, carved into `/12` macro-regions, `/20` region sites, and `/32` per endpoint — see [Tenant Addressing Plan — IPv4 Tenant Addressing](tenant.md#ipv4-tenant-addressing).

The platform tracks tenant prefix and MAC assignments by association (VPC ↔ location) in the IPAM service, without encoding topology in address bits. Tenant prefixes and MACs are programmed into VRF and Bridge Domain forwarding tables by the control plane and distributed via BGP EVPN Route Type 5 and Route Type 2 routes (see [SRv6 uSID Plan — Control Plane Mechanics](srv6.md#control-plane-mechanics-bgp-evpn-l2vpn)).

### Fabric Addressing

Internal infrastructure subnets, backbone links, SRv6 locator advertisement, and infrastructure loopbacks are defined in the [Fabric Addressing Plan](fabric.md). Key properties:

- **Internal infrastructure** — a per-PoP ULA `/48` carved into a `/52` for platform-owned subnets (compute, management, reserved) plus `/127` backbone links
- **SRv6 locators** — a per-PoP RIR PI `/48` subdivided into per-node `/64` locators, advertised within the PoP only and summarized to `/48` at the boundary
- **Infrastructure loopbacks** — a per-PoP RIR PI `/48` providing one `/128` per node, advertised externally as `/48` aggregate only
- **Isolation** — fabric addresses are never reachable from tenant VRFs; enforced by prefix deny lists at VRF handoff

---

## Allocation Hierarchy & Capacity Summary

| Resource                       | Block      | Source                                                     | Capacity                                                                                                  |
|--------------------------------|------------|------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------|
| Platform ULA pool              | `fd00::/8` | RFC 4193 ULA                                               | —                                                                                                         |
| Per-PoP fabric addressing      | 3× `/48`   | ULA + 2× RIR PI (see [Fabric Addressing Plan](fabric.md))  | One ULA infra `/48`, one SRv6 locator `/48`, one loopback `/48` per PoP                                   |
| Per-Tenant VPC (IPv6)          | `/48`      | From tenant ULA pool (`fd20::/20`)                         | ~268.4 million; see [Tenant Addressing Plan — IPv6 Capacity Summary](tenant.md#ipv6-capacity-summary)     |
| Per-Region subnet (IPv6)       | `/64`      | From VPC `/48`                                             | 65,536 per VPC                                                                                            |
| Per-Endpoint (IPv6)            | `/96`      | From region `/64`                                          | ~4.29 billion per region subnet                                                                           |
| Tenant IPv4 pool               | `/9`       | `10.128.0.0/9` (default; alternate bases available)        | Shared, fixed pool; see [Tenant Addressing Plan — IPv4 Capacity Summary](tenant.md#ipv4-capacity-summary) |
| Per-Macro-region (IPv4)        | `/12`      | From tenant IPv4 pool                                      | 8 possible                                                                                                |
| Per-Region site (IPv4)         | `/20`      | From macro-region `/12`                                    | 2,048 total                                                                                               |
| Per-Endpoint (IPv4)            | `/32`      | From region site `/20`                                     | 4,092 usable                                                                                              |

Full detail on tenant VPC, region, and endpoint allocation: [Tenant Addressing Plan](tenant.md).
Full detail on fabric addressing (internal infrastructure, SRv6 locators, loopbacks, backbone links): [Fabric Addressing Plan](fabric.md).

Additional SRv6 locator and infrastructure loopback blocks must be registered with a RIR before existing allocations are exhausted.

> [!NOTE]
> **uSID Instance scaling limit.** Under the selected uSID architecture with `uFMT 48+16` (F4816) and a shared Next uSID slot, the per-tenant VRF or EVI ID is encoded in bits 69–80 of the SID (the 12-bit Argument field). This establishes a namespace of **4,095 usable Instance IDs per uSID Block per universe (4,095 VRF IDs and 4,095 EVI IDs)** — distinct from the 57,343 usable Node-IDs available per uSID Block, which bounds the number of physical nodes rather than tenant instances. When a PoP approaches the Instance ID threshold, the platform supports two scaling paths:
>
> 1. **Secondary uSID Block allocation** — A second `/48` locator block is assigned to the same physical PoP, creating a distinct, independent Instance ID namespace of 4,095 IDs. Each block allocates its range independently — the primary and secondary blocks each serve Instance IDs 1–4,095 — for a combined capacity of 8,190 Instance IDs per universe on the same physical PoP. Tenant SIDs referencing the secondary block use the same Node-ID, Function, and Instance ID values but with the secondary block's `/48` prefix. This requires a supplementary RIR PI allocation but avoids tenant-visible renumbering.
> 2. **Logical PoP partitioning** — The physical PoP is split into two logical domains (e.g., PoP-1A, PoP-1B), each with its own `/48` locator block. This creates independent Instance ID namespaces while preserving the tenant experience through transparent cross-domain steering.
>
> See [SRv6 uSID Plan — Instance ID Space and Scale](srv6.md#instance-id-space-and-scale) for scaling details.

---

## Isolation and Security Properties

**VPC space never leaves the overlay.** The VPC pool — ULA, GUA, or the IPv4 pool (`10.128.0.0/9` default) — must not be advertised to the underlay, leaked to transit providers, or made reachable from outside the overlay. ULA and IPv4 (RFC 1918) share the same filtering backstop described in [Platform-Wide Address Pools](#platform-wide-address-pools): both are conventionally treated as bogon space by upstreams, so a misconfiguration is unlikely to result in global reachability, though the platform still applies its own outbound filtering as defense-in-depth. For GUA, there is no automatic filtering backstop — a missing or misconfigured export policy will result in the prefix entering the global DFZ. GUA VPC addressing therefore requires the full set of controls defined in [GUA Tenant Addressing Policy](#gua-tenant-addressing-policy): default deny export, explicit allowlist per tenant, RPKI ROA enforcement, route leak detection, and a defined emergency quarantine procedure. Any advertisement of VPC prefixes outside the overlay is a misconfiguration regardless of which option is in use.

**Locator and infrastructure blocks do not overlap.** The SRv6 locator block (`[LOCATOR-BLOCK]`) and the infrastructure loopback block (`[INFRA-BLOCK]`) must be drawn from distinct, non-overlapping allocations confirmed at RIR registration time. See [Fabric Addressing Plan — Locator Block Non-Overlap](fabric.md#locator-block-non-overlap) for how routing policy distinguishes the two.

**Fabric addresses are not reachable from tenant networks.** Tenant VRFs must not have routes to the infrastructure loopback block, the internal infrastructure `/52`, or backbone link `/127` subnets. This is enforced by import policy at the VRF handoff point — the `[INFRA-BLOCK]` aggregate and the internal infrastructure `/52` range must appear in a prefix deny list applied to all tenant VRF imports. See [Fabric Addressing Plan — Isolation Properties](fabric.md#isolation-properties) for full detail.

**External advertisement uses aggregates only.** For PoPs with Internet transit that require loopback reachability from external paths, only the PoP-level aggregate (the `/48` covering that PoP's loopbacks) may be advertised. Individual `/128`s must never appear in the global DFZ.
