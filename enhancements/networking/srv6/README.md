# Platform — SRv6 uSID Plan

## Overview

This document defines the SRv6 micro-SID (uSID) structure and management for the platform. It covers the compressed uSID carrier layout, behavior registry, VRF ID slot semantics, and scaling requirements.

The SRv6 overlay for tenant VPCs is modeled on RFC 9252 (BGP Overlay Services over SRv6), which defines the Service SID as the VRF/service identifier — locally significant at the egress node. The compressed SID encoding follows RFC 9800 (Compressed SRv6 Segment List Encoding).

For the underlying address allocation (SRv6 uSID locator blocks, per-PoP `/48` assignments), see [IP Addressing Plan](../ipv6/README.md).

---

## SID Structure (uSID Carrier)

SIDs within a PoP's `/48` block follow the compressed SRv6 segment list encoding (C-SID) specified in RFC 9800. Multiple 16-bit micro-SIDs are packed into a single 128-bit IPv6 destination address container.

```
|<------- 48 bits ------->|<-- 16 bits -->|<-- 16 bits -->|<------- 48 bits ------->|
  uSID Block              Active uSID      Next uSID          Padding
  [LOCATOR-BLOCK]::/48    Node + Func      VRF ID             ::0
```

| Field                  | Bits | Description                                                                       |
| ---------------------- | ---- | --------------------------------------------------------------------------------- |
| uSID Block             | 48   | Identifies the PoP domain; drawn from the SRv6 locator block.                     |
| Active uSID            | 16   | Combines node routing and the endpoint behavior code.                             |
| Next uSID              | 16   | PoP-local VRF ID; part of the opaque SID used for FIB lookup.                     |
| Padding                | 48   | Zero; automatically filled from the right as uSIDs are shifted left.              |

The Active uSID is split into two sub-fields:

| Sub-field | Bits | Range       | Purpose                                                                    |
|-----------|------|-------------|----------------------------------------------------------------------------|
| Node-ID   | 8    | 0x01-0xFF   | Identifies the target node (thread) within the PoP. 0x00 is reserved.      |
| Function  | 8    | 0x01-0xFE   | Selects the endpoint behavior. 0x00 and 0xFF are reserved.                 |

An 8-bit Node-ID supports up to 255 nodes per PoP, which covers the expected scale of any single PoP deployment. An 8-bit Function field supports 254 usable behavior codes, which exceeds the current registry (3 active, 251 reserved or unallocated). The Node-ID occupies the upper 8 bits of the Active uSID; the Function occupies the lower 8 bits.

The uSID block is PoP-scoped, allowing aggregate advertisement and clean per-PoP filtering in the underlay via BGP IPv6 Unicast. When a packet reaches the active node, the hardware processes the **Active uSID**, shifts the subsequent bits left by 16 bits, and uses the full compressed SID (including the Next uSID slot) as an exact-match key for VRF and tenant prefix table selection.

---

## Behavior Registry

The Function sub-field of the Active uSID encodes the compressed SRv6 operation applied at the endpoint. Per RFC 9800, C-SID values are scoped per Locator-Block (i.e., per PoP), which allows the Function space to be reused across sites without exhaustion. The Function codes below are 8-bit platform-wide constants; the full Active uSID (Node-ID + Function) is only guaranteed unique within a PoP's `/48` locator block.

| Function | Behavior  | Description                                                     |
|----------|-----------|-----------------------------------------------------------------|
| 0x00     | Reserved  | Invalid / null; must not be allocated.                          |
| 0x01     | uEnd.DT4  | Shift uSID, decapsulate, and lookup in tenant IPv4 VRF table.   |
| 0x02     | uEnd.DT6  | Shift uSID, decapsulate, and lookup in tenant IPv6 VRF table.   |
| 0x03     | uEnd.DT46 | Shift uSID, decapsulate, and lookup in unified tenant IP table. |
| 0x04-0x7F| Reserved  | Reserved for future platform-wide behaviors.                    |
| 0x80-0xFE | Unallocated | Available for future assignment.                              |
| 0xFF     | Reserved  | Must not be allocated.                                          |

> [!WARNING]
> `uEnd.DT46` is the platform default. However, its use requires hardware/DPU pipeline capability to support a unified, single-table lookup for both address families. If the underlying ASIC infrastructure maintains separate IPv4 and IPv6 forwarding tables, the per-AF deployment configuration (`uEnd.DT4` + `uEnd.DT6`) must be explicitly enforced.

---

## VRF ID (Next uSID Slot)

The 16-bit slot immediately following the Active uSID carries the VRF ID. VRF IDs are **PoP-local**; the same value may be reused across different PoPs because the 48-bit uSID block disambiguates the SID globally.

VRF ID `0x0000` is reserved and must not be allocated.

The platform allocates VRF IDs sequentially within each PoP's namespace when a tenant VPC is first instantiated at that PoP. The platform tracks the mapping of tenant VPC to VRF ID independently per PoP.

The VRF ID is a component of the compressed SID used as an opaque FIB lookup key. Endpoint behaviors (End.DT4/DT6/DT46 per RFC 8986 §4.16-4.18) do not decode the VRF ID as a standalone argument. Per RFC 9800, the compressed REPLACE-CSID processing performs an exact-match table lookup on the full compressed SID to select the VRF and tenant prefix table. The VRF ID slot contributes to that match key but is never arithmetically decoded by the data plane.

### VRF ID to Infrastructure Mapping

1. **The platform** allocates a VRF ID when a tenant VPC is instantiated at a PoP.
2. **The compressed SID** `[uSID Block][Active uSID][VRF ID]` is programmed into each node's FIB as an exact-match lookup key that resolves to a VRF instance and its tenant prefix table.
3. **The VRF instance** is selected by the full SID match, not by decoding the VRF ID field in isolation.
4. **Tenant routes** are imported into this VRF instance by the control plane.

The VRF ID is a platform-local integer managed by the provisioning layer and does not directly encode any external record or attachment ID.

---

## SIDs per Tenant VRF

A tenant VRF requires one active uSID sequence per behavior instantiated at the PoP. All sequences for a given VRF share the same VRF ID in the adjacent slot and differ only in the Function code.

| VRF Type                       | SIDs Required                                      |
| ------------------------------ | -------------------------------------------------- |
| IPv4-only                      | 1 — `uEnd.DT4` (Function `0x01`) + VRF ID          |
| IPv6-only                      | 1 — `uEnd.DT6` (Function `0x02`) + VRF ID          |
| Dual-stack (single VRF table)  | 1 — `uEnd.DT46` (Function `0x03`) + VRF ID         |
| Dual-stack (per-AF VRF tables) | 2 — `uEnd.DT4` + `uEnd.DT6`                        |

---

## Control Plane Mechanics (BGP EVPN)

The 16-bit VRF ID (Next uSID slot) is distributed across the PoP fabric via BGP EVPN, following the L2VPN service model defined in RFC 7432. The control plane maintains a deterministic mapping between BGP route identifiers and SRv6 VRF IDs, ensuring consistent SID construction at every ingress node.

### VRF ID Signaling

The VRF ID is signaled as part of the BGP EVPN Route Distinguisher (RD). Per RFC 4364, an RD is an 8-byte value composed of a 2-byte Type field and a 6-byte Value field. The platform uses RD Type 1 (RFC 4364 §4.2), where the 6-byte Value field is split into a 4-byte PoP router-ID (IPv4 loopback address) and a 2-byte VRF ID:

```
RD (Type 1) — 8 bytes total:
├── Type: 0x0001 (2 bytes)
└── Value (6 bytes):
     ├── PoP router-ID (4 bytes)
     └── VRF ID (2 bytes)
```

The 2-byte VRF ID field yields 65,535 usable distinguishers per PoP (value `0x0000` is reserved), matching the 16-bit Next uSID capacity of the compressed carrier. The PoP router-ID component is required because VRF IDs are PoP-local and reused across PoPs. An RD based on ASN plus VRF ID alone would produce duplicate RDs across PoPs, which under RFC 4364 causes BGP to treat distinct tenant routes as the same NLRI and suppress one via best-path selection. Global RD uniqueness is provided by the per-PoP router-ID component, not the VRF ID itself. The same VRF ID value is used consistently across all nodes within a PoP, as the uSID Block provides PoP-level scoping.

### Route Target Distribution

Route Targets (RTs) define the route import/export policy for each tenant VRF. Each VRF is assigned a unique import RT and export RT (or a shared set for full mesh). The RTs are carried in BGP EVPN Type-1 (Ethernet Auto-Discovery) and Type-5 (IPv4/IPv6 Route Distribution) routes.

The control plane programs the VRF instance with both the RT set and the corresponding VRF ID. When a node receives an EVPN route, it extracts the VRF ID from the RD, validates the RT membership, and programs the tenant prefix into the VRF forwarding table indexed by that VRF ID.

### Type-5 Route Injection

Tenant prefixes are distributed via BGP EVPN Type-5 routes (MP_UNICAST_NLFI with AFI/SAFI = IPv4/IPv6 Unicast). Each Type-5 route carries:

- **Route Distinguisher** — identifies the VRF (embeds VRF ID)
- **Route Targets** — control route import/export scope
- **Prefix** — the tenant subnet (inner payload address)
- **Next Hop** — the SRv6 SID of the egress node (uSID Block + Active uSID + VRF ID)

The next-hop SID encodes the full decapsulation target: the underlay routes the packet to the egress node via the `/64` locator, and the egress node uses the full compressed SID to select the VRF and perform the tenant prefix lookup.

### SID Programming Flow

1. The control plane allocates a VRF ID for a new tenant VPC at a PoP.
2. EVPN Type-1 routes establish the VRF's RT set across all nodes in the PoP.
3. EVPN Type-5 routes inject tenant prefixes, carrying the VRF ID in the RD and the egress node's SRv6 SID as next hop.
4. Each node in the PoP programs a forwarding entry: `[uSID Block][Active uSID][VRF ID] → VRF instance → tenant prefix table`.
5. Ingress nodes construct the SRv6 outer header using the egress SID from the Type-5 next hop, encapsulating the tenant packet.

---

## VRF ID Space and Scale

Because the 16-bit VRF ID slot is part of an opaque SID matched by exact lookup rather than decoded as a standalone field, a single `/48` uSID block provides 65,535 usable VRF IDs per PoP (excluding reserved `0x0000`). This limit applies per uSID Block, not per node — all nodes within a PoP share the same VRF ID namespace under a given locator block.

### Scaling Beyond 64k VRFs

When a PoP approaches the VRF ID ceiling, the platform supports two mitigation paths that preserve the tenant experience:

**Secondary uSID Block allocation.** A second `/48` locator block is assigned to the same physical PoP. This creates a new, independent VRF ID namespace of 65,535 VRF IDs. Each block allocates its full range independently: the primary block serves VRF IDs 1-65,535 and the secondary block serves VRF IDs 1-65,535, for a combined capacity of 131,070 VRFs on the same hardware. Tenant SIDs referencing VRFs on the secondary block use the secondary `/48` prefix with the same Active uSID and VRF ID encoding. The underlay routes to the secondary block's per-node `/64` locators, which resolve to the same physical nodes. This requires a supplementary RIR PI allocation but involves no tenant-visible renumbering.

**Logical PoP partitioning.** The physical PoP is split into two logical domains (e.g., PoP-1A, PoP-1B), each with its own `/48` locator block and independent VRF ID namespace. Cross-domain steering is handled by the control plane: ingress nodes in one logical domain can reach VRFs in the other via inter-domain SRv6 adjacency. This approach is preferable when hardware forwarding table limits (not just VRF ID space) constrain a single logical domain.

### VRF Migration Continuity

When a tenant VRF is migrated to a secondary block or logical partition, its RD changes because the RD encodes the locator block identity (via the PoP router-ID component) and the VRF ID within the new namespace. Route continuity during migration is maintained via a make-before-break procedure: the control plane advertises the tenant's routes under both the old and new RD simultaneously for a defined overlap window, allowing remote peers to install the new next-hop SID alongside the existing entry. Once all peers have converged on the new SID, the old RD is withdrawn. The full migration sequence, including rollback criteria and overlap window sizing, is defined in the platform's VRF migration runbook.

### Operational Thresholds

| Threshold | Action |
|-----------|--------|
| 70% (45,874 VRFs) | IPAM warning — begin evaluating secondary block procurement |
| 80% (52,428 VRFs) | IPAM alert — initiate RIR PI allocation for secondary block or logical partition design |
| 90% (58,981 VRFs) | Capacity critical — freeze new VRF allocations at this PoP until scaling path is active |
| 95% (62,258 VRFs) | Emergency — activate pre-provisioned secondary block or enforce logical partition cutover |
