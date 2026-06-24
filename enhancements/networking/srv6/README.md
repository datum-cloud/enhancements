# Platform — SRv6 uSID Plan

## Overview

This document defines the SRv6 micro-SID (uSID) structure and management for the platform. It covers the compressed uSID carrier layout, behavior registry, VRF ID argument semantics, and scaling requirements.

For the underlying address allocation (SRv6 uSID locator blocks, per-PoP `/48` assignments), see [IP Addressing Plan](../ipv6/README.md).

---

## SID Structure (uSID Carrier)

SIDs within a PoP's `/48` block follow the standard **`uFMT 48+16`** compressed layout specified in RFC 9374. Multiple 16-bit micro-SIDs are packed into a single 128-bit IPv6 destination address container.

```
|<------- 48 bits ------->|<-- 16 bits -->|<-- 16 bits -->|<------- 48 bits ------->|
  uSID Block              Active uSID      Next uSID          Padding
  [LOCATOR-BLOCK]::/48    Node + Func      VRF ID             ::0
```

| Field                  | Bits | Description                                                                       |
| ---------------------- | ---- | --------------------------------------------------------------------------------- |
| uSID Block             | 48   | Identifies the PoP domain; drawn from the SRv6 locator block.                     |
| Active uSID            | 16   | Combines node routing and the endpoint behavior code.                             |
| Next uSID / Argument   | 16   | PoP-local VRF ID; functions as the argument for the tenant VRF.                   |
| Padding                | 48   | Zero; automatically filled from the right as uSIDs are shifted left.              |

The uSID block is PoP-scoped, allowing aggregate advertisement and clean per-PoP filtering in the underlay via BGP IPv6 Unicast. When a packet reaches the active node, the hardware processes the **Active uSID**, shifts the subsequent bits left by 16 bits, and interprets the incoming **Next uSID** slot as the VRF lookup argument.

---

## Behavior Registry

The uSID behavior field encodes the compressed SRv6 operation applied at the endpoint. These behavior identifiers are platform-wide constants.

| Code          | Behavior    | Description                                                     |
|---------------|-------------|-----------------------------------------------------------------|
| 0x0000        | Reserved    | Invalid / null; must not be allocated.                          |
| 0x0001        | uEnd.DT4    | Shift uSID, decapsulate, and lookup in tenant IPv4 VRF table.   |
| 0x0002        | uEnd.DT6    | Shift uSID, decapsulate, and lookup in tenant IPv6 VRF table.   |
| 0x0003        | uEnd.DT46   | Shift uSID, decapsulate, and lookup in unified tenant IP table. |
| 0x0004–0x00FF | Reserved    | Reserved for future platform-wide behaviors.                    |
| 0x0100–0xFFFE | Unallocated | Available for future assignment.                                |
| 0xFFFF        | Reserved    | Must not be allocated.                                          |

> [!WARNING]
> `uEnd.DT46` is the platform default. However, its use requires hardware/DPU pipeline capability to support a unified, single-table lookup for both address families. If the underlying ASIC infrastructure maintains separate IPv4 and IPv6 forwarding tables, the per-AF deployment configuration (`uEnd.DT4` + `uEnd.DT6`) must be explicitly enforced.

---

## Argument Semantics: VRF ID

The 16-bit slot immediately following the active behavior uSID carries the VRF ID. VRF IDs are **PoP-local**; the same value may be reused across different PoPs because the 48-bit uSID block ensures global uniqueness.

VRF ID `0x0000` is reserved and must not be allocated.

The platform allocates VRF IDs sequentially within each PoP's namespace when a tenant VPC is first instantiated at that PoP. The platform tracks the mapping of tenant VPC to VRF ID independently per PoP.

### VRF ID to Infrastructure Mapping

1. **The platform** allocates a VRF ID when a tenant VPC is instantiated at a PoP.
2. **The uSID Argument slot** carries the VRF ID. The endpoint behavior extracts this argument to select the correct forwarding table.
3. **The VRF instance** at the PoP uses this ID to isolate the lookup space.
4. **Tenant routes** are imported into this local VRF instance by the control plane.

The VRF ID is a platform-local integer managed by the provisioning layer and does not directly encode any external record or attachment ID.

---

## SIDs per Tenant VRF

A tenant VRF requires one active uSID sequence per behavior instantiated at the PoP. All sequences for a given VRF share the same argument (VRF ID) in the adjacent slot and differ only in the behavior code.

| VRF Type                       | SIDs Required                                      |
| ------------------------------ | -------------------------------------------------- |
| IPv4-only                      | 1 — `uEnd.DT4` (`0x0001`) + VRF ID                 |
| IPv6-only                      | 1 — `uEnd.DT6` (`0x0002`) + VRF ID                 |
| Dual-stack (single VRF table)  | 1 — `uEnd.DT46` (`0x0003`) + VRF ID                |
| Dual-stack (per-AF VRF tables) | 2 — `uEnd.DT4` + `uEnd.DT6`                        |

---

## VRF ID Space and Scale

Because the 16-bit argument field is evaluated locally by the node processor relative to the active uSID behavior rather than a fixed global IPv6 index, a single `/48` uSID block provides 65,535 usable VRF IDs per PoP node (excluding reserved `0x0000`).

This localized pipeline processing eliminates the need for secondary locator block steering or unsummarized underlay injects during scaling events. When a PoP reaches 80% utilization (52,429 VRF IDs allocated), IPAM must generate an alert to evaluate overall node capacity and hardware lookup table limitations.
