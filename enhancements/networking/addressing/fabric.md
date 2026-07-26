# Platform — Fabric Addressing

## Overview

This document defines the platform's fabric-level addressing: internal infrastructure subnets, backbone links, SRv6 locator advertisement, and infrastructure loopbacks. These are the platform-owned address spaces that form the underlay and control plane backbone within each PoP, entirely separate from tenant VPC addressing.

Each PoP receives two RIR PI `/48` allocations and one ULA `/48` allocation for fabric use — see [Per-PoP Fabric Allocation](#per-pop-fabric-allocation). All three are platform-internal; none are assigned to tenants or tracked as VPCAttachment resources.

The SRv6 locator `/48` and infrastructure loopback `/48` are both Provider Independent (PI) allocations registered with a regional internet registry (RIR). Using Provider Aggregatable (PA) space assigned by an upstream carrier is not acceptable — it creates a hard dependency on that carrier and prevents portability. The actual prefix values are placeholders in the [Addressing Plan](README.md); specific assignments are tracked in the IPAM service.

---

## Per-PoP Fabric Allocation

Each PoP receives three `/48` blocks for fabric use:

```
Per PoP (fabric)
├── /48  ULA internal infrastructure  (from fd00::/8)
│    ├── /52   Internal infrastructure subnets
│    └── /127  Backbone links
├── /48  SRv6 SID locator space       (from [LOCATOR-BLOCK], RIR PI)
│    └── /64   per node
└── /48  Infrastructure loopbacks     (from [INFRA-BLOCK], RIR PI)
     └── /128  per node
```

Per-PoP prefix assignments are operational data tracked in the IPAM service. This document defines the allocation scheme; the IPAM service is the source of truth for specific assignments.

---

## Internal Infrastructure Addressing (ULA /48)

The per-PoP ULA `/48` is reserved exclusively for the platform's internal infrastructure and backbone link addressing. Internal infrastructure subnets are allocated from a dedicated `/52` block carved out of the PoP's `/48`; backbone link `/127`s are carved from the remaining space outside that `/52`.

This space is entirely separate from tenant VPC addressing. Tenant prefixes are defined by the tenant and carried as inner payload inside the SRv6 encapsulation — see [VPC Address Space Options](README.md#vpc-address-space-options).

### Address Hierarchy

```
PoP ULA /48
├── /52  Internal infrastructure (platform-owned)
│    ├── /64   per compute node network
│    ├── /64   per management subnet
│    └── /64   reserved for future use
└── /127 (many)  Backbone link point-to-point subnets
```

| Subnet type   | Allocation       | Purpose                                                                                                                                                                             |
|---------------|------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Compute node  | `/64` per node   | Data plane traffic for compute instances (VMs, containers, bare metal). Each node's `/64` carries instance `/128` addresses.                                                        |
| Management    | `/64` per subnet | Control plane traffic — orchestration, monitoring, logging, and cluster interconnect. Management addresses are typically loopback-style `/128`s, stable across instance lifecycles. |
| Reserved      | `/64` blocks     | Reserved for future internal services.                                                                                                                                              |
| Backbone link | `/127` per link  | Point-to-point underlay router links; see [Backbone Links](#backbone-links).                                                                                                        |

### Subnet Allocation

Internal infrastructure subnets are allocated by IPAM when infrastructure services (compute, management, storage) are provisioned at a PoP. They are not assigned to tenants, not tracked as VPCAttachment resources, and not subject to tenant-facing policies.

### Backbone Links

Point-to-point links between underlay routers are numbered from the PoP's ULA `/48`. Each backbone link receives a `/127` subnet. RFC 6164 documents `/127` as the useful practice for point-to-point inter-router links, since it eliminates the Neighbor Discovery/neighbor-cache exhaustion and ping-pong forwarding-loop attack surface inherent in full `/64` point-to-point subnets; per RFC 6164 §6, routers MUST also disable Subnet-Router anycast for the prefix when `/127`s are used.

A `/48` provides an effectively unlimited supply of `/127` subnets for the lifetime of any PoP.

Backbone link subnets are not advertised externally — they are only needed for underlay routing between routers. External peers do not need to reach individual backbone link addresses; only loopbacks (for BGP peering) and SIDs (for SRv6 forwarding) require external reachability.

---

## SRv6 Locator /48

Each PoP receives a `/48` from the platform's registered SRv6 locator block. This `/48` functions as the **uSID Block** — the global routing domain prefix — under this platform's `uFMT 48+16` (F4816) carrier format, built on the compressed SID structure defined in RFC 9800.

> [!NOTE]
> `uFMT 48+16` is this platform's shorthand for the F4816 REPLACE-CSID format with a shared Next uSID slot (RFC 9800 §4.2.7 — the flavor defined for terminal `End.DT`/`End.DX`-family behaviors; no NEXT-CSID counterpart exists for these). The format defines the SRv6 SID structure as a 48-bit uSID Block (the PoP-scoped IPv6 prefix) followed by a 16-bit Node-ID (Locator-Node / LNL = 16) and a shared 16-bit slot containing a 4-bit Function (FL = 4) and a 12-bit Instance ID (AL = 12) as Argument. The remaining 48 bits of the 128-bit SID container are always zero-filled padding — this carrier has only one Next uSID slot, so there is never a further C-SID to shift in. The 16-bit C-SID space is partitioned into a Global ID Block (GIB) for Node-IDs (`0x0001–0xDFFF`) and a Local ID Block (LIB) for local endpoint functions (`0xE000–0xFFFF`). See [SRv6 uSID Plan — SID Structure](srv6.md#sid-structure-usid-carrier) for the full field layout.

### Locator Hierarchy

```
SRv6 locator /48  (uSID Block, per PoP)
└── /64   per node  ([uSID Block][Node-ID]::/64)
```

The PoP's `/48` uSID Block is subdivided into per-node `/64` locators. Each node within the PoP is assigned a unique 16-bit Node-ID (Locator-Node), which forms the node's `/64` locator prefix. Underlay routing resolves packets based on this `/64` prefix to deliver them to the exact node.

The Node-ID embedded in the `/64` locator corresponds to the 16-bit Active uSID (bits 49–64, using 1-based, MSB-first bit numbering — see [SRv6 uSID Plan — SID Structure](srv6.md#sid-structure-usid-carrier)). The next 16-bit slot (bits 65–80) is the shared Next uSID block carrying the 4-bit Function code (FL = 4) and the 12-bit Instance ID (AL = 12) as Argument. SID structure, function code registry, and VRF / EVI ID semantics are defined in the [SRv6 uSID Plan](srv6.md).

### Underlay Advertisement

Each node *must* advertise its unique `/64` locator via BGP IPv6 Unicast. Advertising only the aggregate `/48` would create an anycast route: the underlay ECMP could steer encapsulated packets to any node within the PoP, causing uSID stepping failures when the packet lands on a node that does not own the target VRF or behavior context. Per-node `/64` advertisement ensures unicast delivery to the exact egress node identified by the Node-ID.

The `/48` itself is never advertised as an aggregate route inside the PoP. Because upstream peers accept only `/48` or larger, the `/64` locator is PoP-internal and must never be advertised to external peers — see [PoP Boundary Aggregation Policy](#pop-boundary-aggregation-policy).

### PoP Boundary Aggregation Policy

Per-node `/64` advertisement applies within the PoP's internal underlay BGP domain, where per-node granularity is what lets the underlay deliver a packet to the specific node identified by the Node-ID. This does not extend to the wider network: PoP boundary/edge nodes must summarize all of a PoP's per-node `/64` locators into the single covering `/48` uSID Block aggregate before advertising toward the broader underlay core or other PoPs. Individual `/64` node locators must be suppressed at the PoP perimeter and must never leak into the global underlay backbone routing table.

This bounds the blast radius of per-node route churn (link flaps, node reboots) to the originating PoP's internal BGP domain rather than propagating it network-wide, while still letting external routers reach the PoP as a whole via the `/48`.

This is a deliberate departure from typical Segment Routing operational practice, which normally disables route summarization for Node-SID-carrying prefixes precisely because SR/SRv6 forwarding depends on end-to-end unicast reachability to the specific node. This design is safe here only because of this platform's specific topology: the boundary node performing the aggregation is itself a full participant in the destination PoP's internal BGP domain — it holds the complete set of per-node `/64` routes and can forward encapsulated packets to the correct internal node. Summarization occurs at the domain boundary, not within the domain.

### Locator Block Non-Overlap

The SRv6 locator block must not overlap with the ULA pool or the infrastructure loopback block. Both are confirmed non-overlapping at RIR registration time. Routing policy must be able to distinguish them unambiguously.

In a uSID deployment, the `/48` uSID Block is the shared registration and filtering unit for all nodes within a PoP. The Active uSID slot (bits 49–64) and Instance ID slot (bits 65–80) are opaque to underlay routing and do not affect prefix reachability or filtering — routing policy still only needs to key off the covering `/48` for isolation purposes.

---

## Infrastructure Loopback /48

Each PoP receives a `/48` from the infrastructure loopback block. Each node takes a single `/128` loopback from this block, used as the BGP peer address for both underlay and overlay sessions.

Loopback addresses are stable across reboots and control plane restarts. The underlay advertises them and ensures they remain reachable for the lifetime of the node.

### External Advertisement

External advertisement — required for Internet transit PoPs — uses the covering PoP-level aggregate (`/48`), not individual `/128`s. Advertising individual `/128` loopbacks into the Default-Free Zone (DFZ) is operationally hostile: it bloats the global routing table, exposes internal infrastructure topology, and will be filtered by most peers regardless — IPv6 prefix length limits at the DFZ boundary are typically `/48` maximum. Aggregates only.

> [!NOTE]
> The Default-Free Zone (DFZ) is the set of Internet core routers that carry a full global BGP routing table with no default route.

---

## Isolation Properties

**Internal infrastructure is not reachable from tenant networks.** Tenant VRFs must not have routes to the infrastructure loopback block or the internal infrastructure `/52`. This is enforced by import policy at the VRF handoff point — both the `[INFRA-BLOCK]` aggregate and the internal infrastructure `/52` range must appear in a prefix deny list applied to all tenant VRF imports. Backbone link `/127` subnets are similarly unreachable from tenant networks, enforced by the same policy.

**Backbone links are not advertised externally.** Backbone link subnets are only needed for underlay routing between routers. External peers do not need to reach individual backbone link addresses; only loopbacks (for BGP peering) and SIDs (for SRv6 forwarding) require external reachability.

**External advertisement uses aggregates only.** For PoPs with Internet transit that require loopback reachability from external paths, only the PoP-level aggregate (the `/48` covering that PoP's loopbacks) may be advertised. Individual `/128`s must never appear in the global DFZ.

---

## Capacity Summary

| Resource                            | Prefix | Source                      | Capacity                                                       |
|-------------------------------------|--------|-----------------------------|----------------------------------------------------------------|
| Per-PoP ULA internal infrastructure | `/48`  | From `fd00::/8`             | One per PoP                                                    |
| Per-PoP internal infra subnets      | `/52`  | From PoP ULA `/48`          | `/64` per compute node, per management subnet, plus reserved   |
| Per-PoP backbone link subnets       | `/127` | From PoP ULA `/48`          | Effectively unlimited per PoP                                  |
| Per-PoP SRv6 locator (uSID Block)   | `/48`  | RIR PI block                | One per PoP; additional blocks required as the platform scales |
| Per-Node SRv6 locator               | `/64`  | uSID Block + 16-bit Node-ID | 57,343 usable Node-IDs per uSID Block (GIB `0x0001–0xDFFF`)    |
| Per-PoP loopback block              | `/48`  | RIR PI (infra loopback)     | One per PoP; additional blocks required as the platform scales |
| Per-Node loopback                   | `/128` | From PoP loopback `/48`     | One per node                                                   |

Additional SRv6 locator and infrastructure loopback blocks must be registered with a RIR before existing allocations are exhausted.
