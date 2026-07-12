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
| SRv6 SID locators        | `[LOCATOR-BLOCK]` (RIR PI)  | SRv6 segment identifiers                      | Advertised in the underlay per node (`/64`s, not the `/48` aggregate); reachability is required for SRv6 forwarding                        |
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
> As the platform scales, additional RIR PI blocks must be obtained for both the SRv6 locator and infrastructure loopback pools. Initiate RIR procurement well before existing allocations are exhausted. Lead time varies: straightforward requests from organizations with an existing RIR/LIR relationship and complete justification are often processed in days to about two weeks; first-time applicants or requests needing to establish LIR/RIR membership first should budget for a month or more.

The ULA pool (`fd00::/8`) is used exclusively for private tenant networking. These prefixes are private by definition (RFC 4193 §4.1 mandates default filtering of `fc00::/7` at exterior BGP sessions) and are treated as bogon space by essentially all transit providers and IXPs, giving ULA a strong filtering backstop GUA lacks — though this is an operational convention enforced by other operators' configurations, not a structural internet guarantee, so no policy is required on this platform's side to prevent external advertisement, though outbound filtering at the PoP border remains good defense-in-depth practice.

> [!NOTE]
> **What is the DFZ?** The Default-Free Zone (DFZ) is the set of Internet core routers — operated by Tier-1 carriers such as Lumen, NTT, and Cogent — that carry a full global BGP routing table with no default route. If a prefix is not in the table, the packet is dropped. Advertising individual `/128` loopbacks into the DFZ is operationally hostile: it bloats the global routing table, exposes internal infrastructure topology, and will be filtered by most peers regardless — IPv6 prefix length limits at the DFZ boundary are typically `/48` maximum. Aggregates only.

The SRv6 SID locator block is a globally registered PI prefix. It is a platform-registered RIR PI block, subdivided per PoP. Within a PoP, each node advertises its own `/64` locator into the underlay via IGP and BGP IPv6 Unicast SAFI — the `/48` itself is never advertised as an aggregate route inside the PoP, since that would create an anycast route ambiguous as to which node owns a given SID (see [SRv6 Locator /48](#srv6-locator-48)). The `/48` remains the unit of RIR registration and of any inter-PoP or external advertisement. It must not overlap with the ULA pool or the infrastructure loopback block.

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
     ├── /49  Underlay BGP router loopbacks
     └── /49  Overlay BGP router loopbacks
```

Tenant VPC addressing is independent of this per-PoP allocation. Tenants define their own CIDR blocks (ULA or GUA) and subnet them across PoPs; see [VPC Address Space Options](#vpc-address-space-options).

Per-PoP address assignments (actual prefix values) are tracked in the platform's IPAM service. This document defines the allocation scheme; the IPAM service is the source of truth for specific assignments.

> [!NOTE]
> All address assignments are tracked by the platform's IPAM service. Allocation is triggered when a project enables networking services at a PoP.

> [!NOTE]
> Per-PoP address register (actual prefix assignments by PoP): tracked in the IPAM service — [datum-cloud/enhancements#715](https://github.com/datum-cloud/enhancements/issues/715).

### VPC Address Space Options

Tenant VPC addressing is fully abstracted from the provider's infrastructure locator space. Tenants define a contiguous CIDR block — either from the ULA range (`fd00::/8`) or from their own globally unique allocation — and subnet it across PoPs and availability zones as needed. The tenant's IP space travels strictly as the inner payload inside the SRv6 encapsulation; it is agnostic to the physical PoP's locator block and never advertised into the underlay. Two addressing models are supported; ULA is the default.

**Option A — ULA (`fd00::/8`)**, default

Tenants select a ULA prefix from `fd00::/8` as their VPC address space. These prefixes are private by definition and are treated as bogon space by essentially all transit providers and IXPs, so no policy is required on this platform's side to prevent external advertisement. Per RFC 4193 §3.2.1 (Global IDs MUST be generated with a pseudo-random algorithm and MUST NOT be assigned sequentially or from well-known values) and the uniqueness analysis in §3.2.3 (collision probability rises as the effective Global ID field length shrinks), the platform should not constrain tenants to a narrow shared sub-range of the 40-bit Global ID space, as doing so reduces effective entropy and increases collision probability on interconnects. If operational visibility into "which prefixes are ours" is required, the platform tracks assigned prefixes in IPAM without constraining the generation range.

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
| Scale                             | Effectively unbounded                                                                                                        | Bounded by allocated block size                                                    |

ULA is the simpler default for a closed overlay. GUA is preferable if tenants require direct IPv6 interconnects with external networks, or if the platform intends to offer bring-your-own-prefix (BYOP) services.

### Internal Infrastructure Addressing

The per-PoP ULA `/48` is reserved exclusively for the platform's internal infrastructure. Internal infrastructure subnets are allocated from a dedicated `/52` block carved out of the PoP's `/48`. This space is entirely separate from tenant VPC addressing — tenant prefixes are defined by the tenant and carried as inner payload inside the SRv6 encapsulation (see [VPC Address Space Options](#vpc-address-space-options)).

```
PoP ULA /48
└── /52  Internal infrastructure (platform-owned)
     ├── /64   per compute node network
     ├── /64   per management subnet
     └── /64   reserved for future use
```

| Subnet type  | Allocation       | Purpose                                                                                                                                                                             |
|--------------|------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Compute node | `/64` per node   | Data plane traffic for compute instances (VMs, containers, bare metal). Each node's `/64` carries instance `/128` addresses.                                                        |
| Management   | `/64` per subnet | Control plane traffic — orchestration, monitoring, logging, and cluster interconnect. Management addresses are typically loopback-style `/128`s, stable across instance lifecycles. |
| Reserved     | `/64` blocks     | Reserved for future internal services.                                                                                                                                              |

Internal infrastructure subnets are **not** customer VPCs. They are not assigned to tenants, not tracked as VPCAttachment resources, and not subject to tenant-facing policies. They are allocated by IPAM when infrastructure services (compute, management, storage) are provisioned at a PoP and are managed by the platform operations team.

Internal infrastructure subnets must not be reachable from tenant networks. This is enforced by the same import deny policy that blocks the infrastructure loopback block — the entire internal infrastructure `/52` range must appear in the tenant VRF prefix deny list.

#### GUA Tenant Addressing Policy

These rules are mandatory whenever GUA VPC addressing (Option B) is in use. They apply to both platform-allocated GUA space and BYOP tenant space. Failure to enforce them will result in tenant prefixes entering the global DFZ.

**Default deny export.** All prefixes that fall within the GUA VPC block are denied export at every underlay-facing BGP session by default. This is a BGP outbound prefix list applied at the session level, not a route-map condition that depends on community or attribute matching. For this control to be effective, the deny term must be evaluated after all allowlist permit terms in the session's export policy chain, and no other export policy applied to the same session may contain a terminating accept for a GUA VPC prefix based on other attributes — operators must verify per-platform that no earlier-evaluated policy term can produce a terminating permit before the deny is reached. There is no opt-out for individual sessions.

**Explicit export intent required.** A GUA VPC prefix may only be exported from the overlay if it appears on a per-session explicit allowlist maintained by the platform operations team. Adding a prefix to the allowlist requires: (1) a documented tenant request, (2) successful prefix ownership validation (see below), (3) a valid RPKI ROA, and (4) change-control approval. Operator convenience is not a valid justification for export. The default is deny; any export is an exception that must be justified, reviewed, and recorded.

**Per-tenant prefix lists.** The export allowlist is maintained at the individual prefix level, not at the covering GUA block level. A single platform-wide community-based policy is insufficient — it creates a single control point whose misconfiguration or exception exposes all tenants. Each tenant's permitted export prefixes must be enumerated explicitly in a per-tenant prefix list. Allowlist membership does not transfer between tenants.

**RPKI and ROA requirements.** The operator must publish ROAs for the platform's GUA VPC block with `maxLength = 48`. Per RFC 6483's ROA validation algorithm, this ensures any advertisement of a more-specific prefix (e.g., an individual `/64`) is classified RPKI Invalid; whether an Invalid route is actually dropped or merely deprioritized is a matter of each receiving peer's local policy (RFC 6811 §5 permits either), though in practice many transit and peering networks do filter Invalid routes outright, so this ROA still functions as a meaningful out-of-band backstop against more-specific leaks — treat it as defense-in-depth alongside the export prefix-list and route-leak-detection controls, not a substitute for them. For BYOP prefixes, the tenant must hold a valid ROA that either (a) origin-authorizes the platform's overlay ASN or (b) origin-authorizes the tenant's own ASN if the tenant manages route origination. Prefixes with RPKI Invalid status must not enter the overlay and must not be exported under any circumstances.

**BYOP validation.** A tenant may use their own Provider Independent (PI) IPv6 space as their VPC address block. Before a BYOP prefix is accepted into the overlay, the platform must verify: (1) the tenant's RIR allocation record (ARIN, RIPE, APNIC, or equivalent) confirms the prefix is theirs, (2) the tenant demonstrates current control of the prefix via a platform-issued challenge — publishing a platform-provided token or certificate in the prefix's RDAP record (analogous to AWS's X.509-certificate-in-RDAP plus signed authorization message), creating a PTR record under the prefix resolving to a platform-issued token (analogous to GCP's reverse-DNS validation), or a signed Letter of Authorization on company letterhead (as commonly required by transit providers and Cloudflare's BYOIP program), (3) the prefix is not currently being announced in the global DFZ by another origin AS (a migration/conflict-safety check, not itself an ownership proof), and (4) a valid ROA exists. The maximum prefix length for a BYOP allocation is the allocation's RIR-registered length — sub-allocations that are more specific than the registered block are not accepted without additional RIR evidence. BYOP validation is re-run on a scheduled basis; if a tenant's RIR record lapses, the BYOP prefix is quarantined pending re-validation.

**Maximum prefix length for export.** Only the per-PoP `/48` aggregate may be exported at the underlay boundary — this is the platform's own conservative export ceiling, chosen to align with widely-adopted (though not RFC-mandated) DFZ filtering practice among major transit providers, which commonly cap accepted IPv6 prefix length at `/48`. No prefix longer than `/48` may appear in an underlay BGP RIB or be sent to any external peer, regardless of whether a given peer's own filters would accept something more specific. This applies to all export paths: transit, peering, and customer interconnects. Individual subnet `/64`s and instance `/128`s must never leave the overlay.

**Route leak detection.** Every underlay BGP session must be monitored for the appearance of GUA VPC prefixes in the peer's received-routes table. A GUA VPC prefix appearing in an underlay RIB is an operational emergency. Route leak detection must integrate with the platform's observability and alerting infrastructure and must page on-call immediately, without waiting for a human to notice.

**External global monitoring.** In addition to session-level Adj-RIB-In monitoring, the platform must continuously check the GUA VPC block and any registered BYOP prefixes against independent, third-party route-collector data (e.g., RIPE RIS, RouteViews/BGPStream) for unexpected origin ASes or more-specific announcements anywhere in the global DFZ — a leak that has already propagated past a directly-monitored session will not be visible from local Adj-RIB-In monitoring alone. Any such appearance triggers the same emergency quarantine procedure below.

**Emergency quarantine.** If a GUA VPC prefix is detected outside the overlay:

1. Withdraw all GUA VPC prefix exports from every underlay-facing session immediately.
2. Isolate the affected PoP from underlay BGP peering until the export policy is confirmed clean.
3. Do not re-enable any export until the root cause is identified and the policy defect is corrected.
4. Notify affected tenants.

Emergency quarantine is disruptive to any tenant that legitimately exports GUA prefixes. This is acceptable. A leak into the DFZ is more disruptive and harder to recover from than a controlled withdrawal.

#### Tenant Subnet Allocation

Tenants subdivide their allocated CIDR block across PoPs and availability zones. The platform does not impose a fixed per-VPC prefix length; tenants choose their own subnet granularity based on host density requirements. A `/64` per VPC is the recommended default. This aligns with the prevailing IPv6 operational convention: RFC 4291 §2.5.1 fixes the interface-identifier field at 64 bits for standard unicast address formats, and RFC 7421 documents the operational risks and tooling assumptions that make deviating from /64 costly in practice — even though neither RFC mandates /64 for overlay/VRF subnet allocation, and this platform assigns addresses via IPAM rather than SLAAC.

```
Tenant CIDR (e.g., /56 ULA or /48 GUA)
├── /64  per VPC at PoP A
│    └── /128  per instance / endpoint address
├── /64  per VPC at PoP B
│    └── /128  per instance / endpoint address
└── ...  additional PoP allocations
```

The platform tracks tenant prefix and MAC assignments by association (VPC ↔ location) in the IPAM service, without encoding topology in address bits. Tenant prefixes and MACs are programmed into VRF and Bridge Domain forwarding tables by the control plane and distributed via BGP EVPN Route Type 5 and Route Type 2 routes (see [SRv6 uSID Plan — Control Plane Mechanics](../srv6/README.md#control-plane-mechanics-bgp-evpn-l2vpn)).

> [!NOTE]
> Tenants requiring larger per-VPC blocks (e.g., `/56` or `/48` per VPC) or a greater number of subnets than their allocated CIDR supports should request a larger CIDR allocation at provisioning time.

### SRv6 Locator /48

Each PoP receives a `/48` from the platform's registered SRv6 locator block. This `/48` functions as the **uSID Block** — the global routing domain prefix — under this platform's `uFMT 48+16` (F4816) carrier format, built on the compressed SID structure defined in RFC 9800.

> [!NOTE]
> `uFMT 48+16` is this platform's shorthand for the F4816 NEXT-C-SID format with a shared Next uSID slot. The format defines the SRv6 SID structure as a 48-bit uSID Block (the PoP-scoped IPv6 prefix) followed by a 16-bit Node-ID (Locator-Node / LNL = 16) and a shared 16-bit slot containing a 4-bit Function (FL = 4) and a 12-bit Instance ID (AL = 12) as Argument. The remaining 48 bits of the 128-bit SID container are zero-filled padding, automatically populated as uSIDs are shifted left during processing. The 16-bit C-SID space is partitioned into a Global ID Block (GIB) for Node-IDs (`0x0001–0xDFFF`) and a Local ID Block (LIB) for local endpoint functions (`0xE000–0xFFFF`). See [SRv6 uSID Plan — SID Structure](../srv6/README.md#sid-structure-usid-carrier) for the full field layout.

The PoP's `/48` uSID Block is subdivided into per-node `/64` locators. Each node within the PoP is assigned a unique 16-bit Node-ID (Locator-Node), which forms the node's `/64` locator prefix (`[uSID Block][Node-ID]::/64`). Underlay routing resolves packets based on this `/64` prefix to deliver them to the exact node.

**Underlay advertisement is per-node, not aggregate.** Each node *must* advertise its unique `/64` locator into the underlay IGP and BGP IPv6 Unicast. Advertising only the aggregate `/48` would create an anycast route: the underlay ECMP could steer encapsulated packets to any node within the PoP, causing uSID stepping failures when the packet lands on a node that does not own the target VRF or behavior context. Per-node `/64` advertisement ensures unicast delivery to the exact egress node identified by the Node-ID. This requirement is scoped to the PoP's internal underlay domain — see [PoP Boundary Aggregation Policy](#pop-boundary-aggregation-policy) for how the PoP summarizes these locators when advertising outward.

#### PoP Boundary Aggregation Policy

Per-node `/64` advertisement (above) applies within the PoP's internal underlay IGP/BGP domain, where per-node granularity is what lets the underlay deliver a packet to the specific node identified by the Node-ID. This does not extend to the wider network: PoP boundary/edge nodes must summarize all of a PoP's per-node `/64` locators into the single covering `/48` uSID Block aggregate before advertising toward the broader underlay core or other PoPs. Individual `/64` node locators must be suppressed at the PoP perimeter and must never leak into the global underlay backbone routing table.

This bounds the blast radius of per-node route churn (link flaps, node reboots) to the originating PoP's internal IGP area rather than propagating it network-wide, while still letting external routers reach the PoP as a whole via the `/48`. This is a deliberate departure from typical Segment Routing operational practice, which normally disables route summarization for Node-SID-carrying prefixes precisely because SR/SRv6 forwarding depends on end-to-end unicast reachability to the specific node (RFC 8402 §3.2 requires Node-SID uniqueness only *within a routing domain*, but says nothing about whether the prefix carrying it may be aggregated at a domain boundary — summarizing it typically breaks reachability, which is exactly why vendor SR implementation guides document disabling summarization for loopback/Node-SID prefixes at area/level boundaries). This design is safe here only because of this platform's specific topology: the boundary node performing the aggregation is itself a full participant in the destination PoP's internal IGP domain (analogous to an OSPF Area Border Router or an IS-IS Level-1/Level-2 router) — it holds the complete set of per-node `/64` routes in its local RIB from that internal adjacency, even though it originates only the `/48` summary toward the backbone or other PoPs. Once a packet ingresses the destination PoP via that boundary node, the PoP's own IGP takes over and resolves the exact egress node — the boundary node need not itself resolve the specific node, only route the packet into the PoP that owns it.

The Node-ID embedded in the `/64` locator corresponds to the 16-bit Active uSID (bits 49–64, using this document's 1-based, MSB-first bit numbering — see [SRv6 uSID Plan — SID Structure](../srv6/README.md#sid-structure-usid-carrier)). The next 16-bit slot (bits 65–80) is the shared Next uSID block carrying the 4-bit Function code (FL = 4, bits 65–68) and the 12-bit Instance ID (AL = 12, bits 69–80) as Argument. The control plane programs each node's FIB/local-SID-table entry to match on its own locator `[uSID Block][Node-ID]::/64` via longest-prefix match. When a packet matches this entry, the node executes the NEXT-C-SID behavior, shifting the IPv6 destination address left by 16 bits to expose the shared `[Func (4)][Instance ID (12)]` block in the active position, which selects the correct endpoint behavior based on the Function prefix (matching `LBL+FL = 52` bits: `[uSID Block][Function]::/52`). The Instance ID is read out of the Argument bits by the endpoint behavior to select the tenant VRF or Bridge Domain (see [SRv6 uSID Plan — VRF / EVI ID](../srv6/README.md#vrf-evi-id-argument)).

SID structure, function code registry, and VRF ID semantics are defined in [SRv6 uSID Plan](../srv6/README.md).

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

Point-to-point links between underlay routers are numbered from the underlay BGP router loopback `/49`. Each backbone link receives a `/127` subnet. RFC 6164 documents `/127` as the useful practice for point-to-point inter-router links, since it eliminates the Neighbor Discovery/neighbor-cache exhaustion and ping-pong forwarding-loop attack surface inherent in full `/64` point-to-point subnets; per RFC 6164 §6, routers MUST also disable Subnet-Router anycast for the prefix when `/127`s are used.

```
Underlay /49
├── /128  per router loopback
├── /127  per backbone link
└── ...   additional /127 subnets as needed
```

A `/49` provides 2^78 possible `/127` subnets — an effectively unlimited supply for the lifetime of any PoP.

Backbone link subnets are not advertised externally — they are only needed for underlay routing between routers. External peers do not need to reach individual backbone link addresses; only loopbacks (for BGP peering) and SIDs (for SRv6 forwarding) require external reachability. As with loopbacks, backbone link addresses must not be reachable from tenant networks. This is enforced by the same import deny policy that blocks the infrastructure loopback block.

---

## Capacity and Scaling

| Resource                       | Notes                                                                        |
| ------------------------------ | ---------------------------------------------------------------------------- |
| SRv6 locator `/48`s            | One per PoP; additional RIR PI blocks required as the platform scales        |
| Infrastructure loopback `/48`s | One per PoP; additional RIR PI blocks required as the platform scales        |
| ULA VPC space                  | Effectively unbounded (`fd00::/8`)                                           |

Additional SRv6 locator and infrastructure loopback blocks must be registered with a RIR before existing allocations are exhausted. Procurement lead time for PI space varies — straightforward requests from organizations with an existing RIR/LIR relationship are often processed in days to about two weeks, while first-time applicants should budget for a month or more; initiate the process well in advance of exhaustion. Block sizing and procurement thresholds are tracked in the IPAM service.

> [!NOTE]
> **uSID Instance scaling limit.** Under the selected uSID architecture with `uFMT 48+16` (F4816) and a shared Next uSID slot, the per-tenant VRF or EVI ID is encoded in bits 69–80 of the SID (the 12-bit Argument field). This establishes a namespace of **4,095 usable Instance IDs per uSID Block per universe (4,095 VRF IDs and 4,095 EVI IDs)**. When a PoP approaches this threshold, the platform supports two scaling paths:
>
> 1. **Secondary uSID Block allocation** — A second `/48` locator block is assigned to the same physical PoP, creating a distinct, independent Instance ID namespace of 4,095 IDs. Each block allocates its range independently — the primary and secondary blocks each serve Instance IDs 1–4,095 — for a combined capacity of 8,190 Instance IDs per universe on the same physical PoP. Tenant SIDs referencing the secondary block use the same Node-ID, Function, and Instance ID values but with the secondary block's `/48` prefix. This requires a supplementary RIR PI allocation but avoids tenant-visible renumbering.
> 2. **Logical PoP partitioning** — The physical PoP is split into two logical domains (e.g., PoP-1A, PoP-1B), each with its own `/48` locator block. This creates independent Instance ID namespaces while preserving the tenant experience through transparent cross-domain steering.
>
> See [SRv6 uSID Plan — Instance ID Space and Scale](../srv6/README.md#instance-id-space-and-scale) for the operational thresholds and mitigation playbook.

---

## Allocation Hierarchy Summary

| Resource                                   | Block      | Source                                          |
|--------------------------------------------|------------|-------------------------------------------------|
| Platform ULA pool                          | `fd00::/8` | RFC 4193 ULA                                    |
| Per-PoP ULA internal infrastructure        | `/48`      | From `fd00::/8`                                 |
| Per-PoP internal infra subnets             | `/52`      | From PoP ULA `/48`                              |
| Per-PoP SRv6 locator (uSID Block)          | `/48`      | RIR PI block                                    |
| Per-Node SRv6 locator (underlay reachable) | `/64`      | uSID Block + 16-bit Node-ID                     |
| Per-Node ID (Active uSID)                  | N/A        | 16-bit Node-ID (GIB: 0x0001–0xDFFF), bits 49–64 |
| Per-Node Instruction (Function)            | N/A        | 4-bit Function (FL = 4), bits 65–68             |
| Per-Tenant VRF / EVI ID (Argument)         | N/A        | 12-bit Instance ID (AL = 12), bits 69–80        |
| Padding                                    | N/A        | Zero-filled, bits 81–128                        |
| Per-PoP loopback block                     | `/48`      | RIR PI (infrastructure loopback block)          |
| Per-PoP underlay loopbacks                 | `/49`      | From PoP loopback `/48`                         |
| Per-PoP overlay loopbacks                  | `/49`      | From PoP loopback `/48`                         |
| Per-VPC at a PoP                           | `/64`      | From tenant-allocated CIDR                      |
| Per-instance address                       | `/128`     | From subnet `/64`                               |
| Per-router loopback                        | `/128`     | From PoP `/49`                                  |

---

## Isolation and Security Properties

**VPC space never leaves the overlay.** The VPC pool — whether ULA or GUA — must not be advertised to the underlay, leaked to transit providers, or made reachable from outside the overlay. For ULA (`fd00::/8`), this has a strong backstop: upstreams conventionally filter `fd00::/8` as bogon space (RFC 4193 §4.1 mandates this default at exterior BGP sessions), so a misconfiguration is unlikely to result in global reachability — though, as with any convention enforced by other operators' configurations rather than the protocol itself, the platform should still apply its own outbound filtering as defense-in-depth. For GUA, there is no automatic filtering backstop — a missing or misconfigured export policy will result in the prefix entering the global DFZ. GUA VPC addressing therefore requires the full set of controls defined in [GUA Tenant Addressing Policy](#gua-tenant-addressing-policy): default deny export, explicit allowlist per tenant, RPKI ROA enforcement, route leak detection, and a defined emergency quarantine procedure. Any advertisement of VPC prefixes outside the overlay is a misconfiguration regardless of which option is in use.

**Locator and infrastructure blocks do not overlap.** The SRv6 locator block (`[LOCATOR-BLOCK]`) and the infrastructure loopback block (`[INFRA-BLOCK]`) must be drawn from distinct, non-overlapping allocations confirmed at RIR registration time. Routing policy must be able to distinguish them unambiguously. In a uSID deployment, the `/48` uSID Block is the shared registration and filtering unit for all nodes within a PoP; individually, each node also advertises its own `/64` locator carved from that block (see [SRv6 Locator /48](#srv6-locator-48)) so the underlay can route to the correct node. The Active uSID slot (bits 49–64) and VRF ID slot (bits 65–80) are opaque to underlay routing and do not affect prefix reachability or filtering — routing policy still only needs to key off the covering `/48` for isolation purposes.

**Infrastructure loopbacks are not reachable from tenant networks.** Tenant VRFs must not have routes to the infrastructure loopback block. This is enforced by import policy at the VRF handoff point — the `[INFRA-BLOCK]` aggregate must appear in a prefix deny list applied to all tenant VRF imports.

**External advertisement uses aggregates only.** For PoPs with Internet transit that require loopback reachability from external paths, only the PoP-level aggregate (the `/48` covering that PoP's loopbacks) may be advertised. Individual `/128`s must never appear in the global DFZ.
