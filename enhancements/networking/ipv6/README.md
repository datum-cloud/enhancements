# Platform — IPv6 Addressing Plan

## Overview

This document defines the IPv6 address allocation strategy for the platform. It covers four distinct address domains: private tenant VPC addressing (ULA default, GUA optional), platform-owned internal infrastructure addressing, SRv6 micro-SID (uSID) locator space, and infrastructure loopback addressing. Each domain is allocated independently and must not overlap.

All infrastructure is IPv6-only. IPv4 addressing within the overlay is out of scope for this document.

> [!NOTE]
> The actual prefix values for the SRv6 locator block and infrastructure loopback block are placeholders throughout this document. Both are RIR PI allocations; specific prefixes must be confirmed non-overlapping and registered in the IPAM service before this plan is finalized.

---

## Platform-Wide Address Pools

The platform draws from three top-level IPv6 address pools. These pools are managed centrally and subdivided per PoP.

| Pool                     | Block                       | Purpose                                       | Visibility                                                                                                                                 |
|--------------------------|-----------------------------|-----------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| VPC (ULA, default)       | `fd00::/8`                  | Tenant overlay addressing                     | Overlay only — never leaves the overlay; see [VPC Address Space Options](#vpc-address-space-options)                                       |
| VPC (GUA, alternative)   | `[VPC-BLOCK]` (RIR PI)      | Tenant overlay addressing                     | Overlay only — must not be advertised externally; policy enforcement required; see [VPC Address Space Options](#vpc-address-space-options) |
| SRv6 SID locators        | `[LOCATOR-BLOCK]` (RIR PI)  | SRv6 segment identifiers                      | Advertised in the underlay (BGP IPv6 Unicast); reachability is required for SRv6 forwarding                                                |
| Infrastructure loopbacks | `[INFRA-BLOCK]` (RIR PI)    | Router loopbacks for underlay and overlay BGP | Underlay-reachable; aggregate-only to external peers                                                                                       |

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
>
> The specific RIR used depends on the platform's operational region and upstream carrier relationships.

> [!NOTE]
> As the platform scales, additional RIR PI blocks must be obtained for both the SRv6 locator and infrastructure loopback pools. Initiate RIR procurement well before existing allocations are exhausted; lead time for PI space is measured in weeks to months.

The ULA pool (`fd00::/8`) is used exclusively for private tenant networking. These prefixes are private by definition and universally filtered by upstreams; there is no policy required to prevent external advertisement.

> [!NOTE]
> **What is the DFZ?** The Default-Free Zone (DFZ) is the set of Internet core routers — operated by Tier-1 carriers such as Lumen, NTT, and Cogent — that carry a full global BGP routing table with no default route. If a prefix is not in the table, the packet is dropped. Advertising individual `/128` loopbacks into the DFZ is operationally hostile: it bloats the global routing table, exposes internal infrastructure topology, and will be filtered by most peers regardless — IPv6 prefix length limits at the DFZ boundary are typically `/48` maximum. Aggregates only.

The SRv6 SID locator block is a globally registered PI prefix. It is a platform-registered RIR PI block, subdivided per PoP. Each PoP's `/48` locator is advertised into the underlay via BGP IPv6 Unicast SAFI — underlay reachability of the locator prefix is required for SRv6 forwarding. It must not overlap with the ULA pool or the infrastructure loopback block.

---

## Per-PoP Allocation

Each PoP receives exactly three `/48` allocations, one from each platform pool.

> [!NOTE]
> **Point of presence (PoP).** A geographic location where the platform deploys networking and compute infrastructure. Each PoP is an independent address allocation unit — each receives its own set of `/48` blocks from the platform pools.


```
Per PoP
├── /48  ULA private VPC space       (from fd00::/8)
├── /48  SRv6 SID locator space      (from [LOCATOR-BLOCK])
└── /48  Infrastructure loopbacks    (from [INFRA-BLOCK])
         ├── /49  Underlay BGP router loopbacks
         └── /49  Overlay BGP router loopbacks
```

Per-PoP address assignments (actual prefix values) are tracked in the platform's IPAM service. This document defines the allocation scheme; the IPAM service is the source of truth for specific assignments.

> [!NOTE]
> All address assignments are tracked by the platform's IPAM service. Allocation is triggered when a project enables networking services at a PoP.

> [!NOTE]
> Per-PoP address register (actual prefix assignments by PoP): tracked in the IPAM service — [datum-cloud/enhancements#715](https://github.com/datum-cloud/enhancements/issues/715).

### VPC Address Space Options

The per-PoP VPC address pool is an implementation decision. Two options are defined below; ULA is the default.

**Option A — ULA (`fd00::/8`)**, default

Each PoP's ULA `/48` is drawn from `fd00::/8` and assigned at PoP provisioning time. These prefixes are private by definition and universally filtered by upstreams; there is no policy required to prevent external advertisement.

> [!NOTE]
> The per-PoP `/48` ULA allocation should be drawn from a contiguous per-region range within `fd00::/8`, enabling eventual region-level aggregation if route scale management becomes necessary.

**Option B — GUA (platform RIR PI block)**

Each PoP's GUA `/48` is drawn from a platform-registered RIR PI block reserved exclusively for tenant overlay use. The block must not overlap with `[LOCATOR-BLOCK]` or `[INFRA-BLOCK]`. Because GUA space is globally routable, leakage does not fail safe the way ULA does — a misconfigured export policy will result in the prefix appearing in the global DFZ. The policy requirements in [GUA Tenant Addressing Policy](#gua-tenant-addressing-policy) are mandatory, not advisory, whenever Option B is in use.

#### Comparison

| Criterion                         | ULA (`fd00::/8`)                                      | GUA (RIR PI)                                               |
|-----------------------------------|-------------------------------------------------------|------------------------------------------------------------|
| RIR registration required         | No                                                    | Yes                                                        |
| Global address uniqueness         | Not guaranteed; overlap possible across organizations | Guaranteed                                                 |
| Risk of accidental DFZ leakage    | None — upstreams universally filter `fd00::/8`        | Certain if export policy is absent or misconfigured        |
| Multi-cloud / direct interconnect | Overlap risk requires coordination between parties    | No overlap risk; prefixes are unambiguous across providers |
| Bring-your-own-prefix (BYOP)      | Not applicable                                        | Tenants may bring own PI space                             |
| IPAM governance                   | Self-managed; no RIR involvement                      | Requires RIR allocation and lifecycle management           |
| Scale                             | Effectively unbounded                                 | Bounded by allocated block size                            |

ULA is the simpler default for a closed overlay. GUA is preferable if tenants require direct IPv6 interconnects with external networks, or if the platform intends to offer bring-your-own-prefix (BYOP) services.

### Internal Infrastructure Addressing

The per-PoP ULA `/48` serves two purposes: customer VPC subnets and the platform's internal infrastructure. Internal infrastructure subnets are allocated from a dedicated `/52` block carved out of the PoP's `/48`, separate from the customer VPC pool. This separation prevents customer VPC capacity from consuming internal infrastructure address space and allows independent scaling.

```
PoP ULA /48
├── /52  Internal infrastructure (platform-owned)
│    ├── /64   per compute node network
│    ├── /64   per management subnet
│    └── /64   reserved for future use
└── /64   per customer VPC (remaining ~61,440 subnets)
```

| Subnet type  | Allocation       | Purpose                                                                                                                                                                             |
|--------------|------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Compute node | `/64` per node   | Data plane traffic for compute instances (VMs, containers, bare metal). Each node's `/64` carries instance `/128` addresses.                                                        |
| Management   | `/64` per subnet | Control plane traffic — orchestration, monitoring, logging, and cluster interconnect. Management addresses are typically loopback-style `/128`s, stable across instance lifecycles. |
| Reserved     | `/64` blocks     | Reserved for future internal services.                                                                                                                                              |

Internal infrastructure subnets are **not** customer VPCs. They are not assigned to tenants, not tracked as VPCAttachment resources, and not subject to tenant-facing policies. They are allocated by IPAM when infrastructure services (compute, management, storage) are provisioned at a PoP and are managed by the platform operations team.

Internal infrastructure subnets must not be reachable from tenant networks. This is enforced by the same import deny policy that blocks the infrastructure loopback block — the entire internal infrastructure `/52` range must appear in the tenant VRF prefix deny list.

> [!NOTE]
> If Option B (GUA) is chosen for customer VPCs, the internal infrastructure `/52` is drawn from the GUA block instead of ULA. The same separation principles apply: internal infrastructure has its own contiguous allocation within the PoP's `/48`, separate from customer VPC subnets.

#### GUA Tenant Addressing Policy

These rules are mandatory whenever GUA VPC addressing (Option B) is in use. They apply to both platform-allocated GUA space and BYOP tenant space. Failure to enforce them will result in tenant prefixes entering the global DFZ.

**Default deny export.** All prefixes that fall within the GUA VPC block are denied export at every underlay-facing BGP session by default. This is a BGP outbound prefix list applied at the session level, not a route-map condition that depends on community or attribute matching. The deny is unconditional — no match condition can override it without an explicit allowlist entry. There is no opt-out for individual sessions.

**Explicit export intent required.** A GUA VPC prefix may only be exported from the overlay if it appears on a per-session explicit allowlist maintained by the platform operations team. Adding a prefix to the allowlist requires: (1) a documented tenant request, (2) successful prefix ownership validation (see below), (3) a valid RPKI ROA, and (4) change-control approval. Operator convenience is not a valid justification for export. The default is deny; any export is an exception that must be justified, reviewed, and recorded.

**Per-tenant prefix lists.** The export allowlist is maintained at the individual prefix level, not at the covering GUA block level. A single platform-wide community-based policy is insufficient — it creates a single control point whose misconfiguration or exception exposes all tenants. Each tenant's permitted export prefixes must be enumerated explicitly in a per-tenant prefix list. Allowlist membership does not transfer between tenants.

**RPKI and ROA requirements.** The operator must publish ROAs for the platform's GUA VPC block with `maxLength = 48`. This ensures that any advertisement of a more-specific prefix (e.g., an individual `/64`) is RPKI Invalid and will be rejected by ROV-enforcing peers, providing an out-of-band backstop against more-specific leaks. For BYOP prefixes, the tenant must hold a valid ROA that either (a) origin-authorizes the platform's overlay ASN or (b) origin-authorizes the tenant's own ASN if the tenant manages route origination. Prefixes with RPKI Invalid status must not enter the overlay and must not be exported under any circumstances.

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

> [!NOTE]
> Default VPC allocation is `/64` per VPC. Projects requiring more than 65,536 subnets per PoP, or that need larger per-VPC blocks, require a non-default allocation request. The platform reserves the right to make larger allocations on a per-request basis.

```
PoP VPC /48
└── /64  per consumer VPC at this PoP
     └── /128  per instance / endpoint address
```

### SRv6 Locator /48

Each PoP receives a `/48` from the platform's registered SRv6 locator block. This `/48` functions as the **uSID Block** — the global routing domain prefix — under a standard `uFMT 48+16` carrier format per RFC 9374.

> [!NOTE]
> The `uFMT 48+16` format defines the SRv6 SID structure as a 48-bit uSID Block (the PoP-scoped IPv6 prefix) followed by a 16-bit Active uSID slot. The remaining bits encode the per-tenant VRF argument and function-specific fields. This format is the default carrier format for micro-SID deployments and is used throughout the platform.

In a uSID architecture, individual routers or nodes within the PoP do not receive separate locator subnets carved out of this `/48`. Instead, all nodes share the same `/48` prefix, and traffic is steered to specific nodes and functions via the first 16-bit slot immediately following the prefix — the **Active uSID** slot (bits 49–64). This slot selects the egress node (or "thread") within the PoP's SRv6 fabric, eliminating the need for per-router locator allocations that characterize classic SRv6 designs.

The locator block is PoP-scoped. All SIDs for a given PoP share the same `/48` uSID Block prefix, enabling aggregate advertisement and clean per-PoP filtering.

The PoP's `/48` locator prefix is advertised into the underlay via BGP IPv6 Unicast. This is not optional — underlay reachability of the locator is a prerequisite for SRv6 forwarding. Without it, remote PoPs cannot route encapsulated packets to their destination. The aggregate `/48` is advertised, not individual SIDs.

SID structure, function code registry, and VRF ID semantics are defined in [SRv6 SID Plan](../srv6/README.md).

### Infrastructure Loopback /48

Each PoP receives a `/48` from the infrastructure loopback block. This `/48` is split into two `/49`s:

| Sub-block    | Assignment                    |
| ------------ | ----------------------------- |
| First `/49`  | Underlay BGP router loopbacks |
| Second `/49` | Overlay BGP router loopbacks  |

Individual loopbacks are `/128`s allocated from the appropriate `/49`. Every underlay router takes one loopback from the first `/49`. Every overlay BGP node takes one loopback from the second `/49`.

Loopback addresses are stable across reboots and control plane restarts. The underlay advertises them and ensures they remain reachable for the lifetime of the node.

Per the security requirements in the underlay document, these loopbacks must not be reachable from tenant networks. This is enforced by import policy on each tenant VRF — the infrastructure loopback block must be present in a prefix deny list applied at VRF handoff.

External advertisement — required for Internet transit PoPs — uses the covering PoP-level aggregate (`/48`), not individual `/128`s.

### Backbone Links

Point-to-point links between underlay routers are numbered from the underlay BGP router loopback `/49`. Each backbone link receives a `/64` subnet.

```
Underlay /49
├── /128  per router loopback
├── /64   per backbone link
└── ...   additional /64 subnets as needed
```

The underlay `/49` provides over one million possible `/64` subnets — more than sufficient for the lifetime of any PoP.

Backbone link subnets are not advertised externally — they are only needed for underlay routing between routers. External peers do not need to reach individual backbone link addresses; only loopbacks (for BGP peering) and SIDs (for SRv6 forwarding) require external reachability. As with loopbacks, backbone link addresses must not be reachable from tenant networks. This is enforced by the same import deny policy that blocks the infrastructure loopback block.

---

## Capacity and Scaling

| Resource                       | Notes                                                                        |
| ------------------------------ | ---------------------------------------------------------------------------- |
| SRv6 locator `/48`s            | One per PoP; additional RIR PI blocks required as the platform scales        |
| Infrastructure loopback `/48`s | One per PoP; additional RIR PI blocks required as the platform scales        |
| ULA VPC space                  | Effectively unbounded (`fd00::/8`)                                           |

Additional SRv6 locator and infrastructure loopback blocks must be registered with a RIR before existing allocations are exhausted. Procurement lead time for PI space is measured in weeks to months; initiate the process well in advance of exhaustion. Block sizing and procurement thresholds are tracked in the IPAM service.

> [!NOTE]
> **uSID VRF scaling limit.** Under a uSID architecture with `uFMT 48+16`, the per-tenant VRF context is encoded in bits 65–80 of the SID (a 16-bit argument field). This establishes a hard architectural boundary of **65,535 concurrent VRF IDs per PoP domain**. This limit is per-PoP and independent of the tenant VPC `/64` subnet capacity (65,536 subnets from the ULA `/48`). When a PoP approaches this VRF ID exhaustion threshold, additional capacity requires either (a) introducing a secondary uSID Block from a new RIR PI allocation and renumbering, or (b) redistributing VRF load across multiple PoP domains.

---

## Allocation Hierarchy Summary

| Resource                                      | Block      | Source                                 |
|-----------------------------------------------|------------|----------------------------------------|
| Platform ULA pool                             | `fd00::/8` | RFC 4193 ULA                           |
| Per-PoP ULA VPC space                         | `/48`      | From `fd00::/8`                        |
| Per-PoP internal infra                        | `/52`      | From PoP ULA `/48`                     |
| Per-PoP SRv6 locator (uSID Block)             | `/48`      | RIR PI block                           |
| Per-Node Instruction + Behavior (Active uSID) | N/A        | Embedded in bits 49–64                 |
| Per-Tenant VRF Context (Argument)             | N/A        | Embedded in bits 65–80                 |
| Per-PoP loopback block                        | `/48`      | RIR PI (infrastructure loopback block) |
| Per-PoP underlay loopbacks                    | `/49`      | From PoP loopback `/48`                |
| Per-PoP overlay loopbacks                     | `/49`      | From PoP loopback `/48`                |
| Per-VPC at a PoP                              | `/64`      | From PoP ULA `/48`                     |
| Per-instance address                          | `/128`     | From subnet `/64`                      |
| Per-router loopback                           | `/128`     | From PoP `/49`                         |

---

## Isolation and Security Properties

**VPC space never leaves the overlay.** The VPC pool — whether ULA or GUA — must not be advertised to the underlay, leaked to transit providers, or made reachable from outside the overlay. For ULA (`fd00::/8`), this is fail-safe: upstreams universally filter `fd00::/8`, so a misconfiguration does not result in global reachability. For GUA, there is no automatic filtering backstop — a missing or misconfigured export policy will result in the prefix entering the global DFZ. GUA VPC addressing therefore requires the full set of controls defined in [GUA Tenant Addressing Policy](#gua-tenant-addressing-policy): default deny export, explicit allowlist per tenant, RPKI ROA enforcement, route leak detection, and a defined emergency quarantine procedure. Any advertisement of VPC prefixes outside the overlay is a misconfiguration regardless of which option is in use.

**Locator and infrastructure blocks do not overlap.** The SRv6 locator block (`[LOCATOR-BLOCK]`) and the infrastructure loopback block (`[INFRA-BLOCK]`) must be drawn from distinct, non-overlapping allocations confirmed at RIR registration time. Routing policy must be able to distinguish them unambiguously. In a uSID deployment, the `/48` uSID Block is shared across all nodes within a PoP — there are no per-router locator subnets to manage or filter. The Active uSID slot (bits 49–64) and VRF argument (bits 65–80) are opaque to underlay routing and do not affect prefix reachability or filtering.

**Infrastructure loopbacks are not reachable from tenant networks.** Tenant VRFs must not have routes to the infrastructure loopback block. This is enforced by import policy at the VRF handoff point — the `[INFRA-BLOCK]` aggregate must appear in a prefix deny list applied to all tenant VRF imports.

**External advertisement uses aggregates only.** For PoPs with Internet transit that require loopback reachability from external paths, only the PoP-level aggregate (the `/48` covering that PoP's loopbacks) may be advertised. Individual `/128`s must never appear in the global DFZ.
