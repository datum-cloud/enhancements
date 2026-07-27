# Platform — SRv6 uSID Plan (EVPN Services overlay)

## Overview

This document defines the SRv6 micro-SID (uSID) structure and management for the platform's tenant overlay services. It covers the compressed uSID carrier layout, behavior registry, VRF and EVI ID semantics, and scaling requirements.

The overlay services are built on BGP EVPN over SRv6 (RFC 9136 / RFC 9252), carried in the L2VPN EVPN address family (AFI=25, SAFI=70). The 16-bit Local Identifier Block (LIB) space is carved into two independent service blocks using the leading hex digit:

1. **The `0xE___` Universe: Layer 3 Services (EVPN Route Type 5)**
   * **Function Block**: `0xE` signifies a `uEnd.DT46` dual-stack Layer 3 VRF lookup.
   * **Instance Space**: The remaining 12 bits map directly to tenant L3 VRF IDs.
   * **Example**: VRF 10 maps directly to `0xE00A`. Packets landing here hit the L3 routing engine.
2. **The `0xF___` Universe: Layer 2 Bridging Services (EVPN Route Type 2 — Future Use)**
   * **Function Block**: `0xF` signifies a `uEnd.DT2` Layer 2 MAC table lookup.
   * **Instance Space**: The remaining 12 bits map directly to tenant EVPN Instance (EVI) / Bridge Domain IDs.
   * **Example**: EVI 10 maps directly to `0xF00A`. Packets landing here hit the Layer 2 bridging engine.

The compressed SID encoding follows RFC 9800 (Compressed SRv6 Segment List Encoding). For the underlying address allocation (SRv6 uSID locator blocks, per-PoP `/48` assignments), see [Addressing Plan](README.md).

---

## SID Structure (uSID Carrier)

SIDs within a PoP's `/48` block follow the compressed SRv6 segment list encoding (C-SID) REPLACE-CSID flavor specified in RFC 9800 §4.2 — the flavor RFC 9800 §4.2.7 defines for terminal `End.DT`/`End.DX`-family behaviors (RFC 9800 states no NEXT-CSID counterpart exists for these, since they're always the last SID in a container). Under the platform's selected design, the 16-bit Next uSID slot is shared between the 4-bit Function (FL = 4) and the 12-bit Instance ID (AL = 12).

```
|<------- 48 bits ------->|<-- 16 bits -->|<-- 16 bits -->|<------- 48 bits ------->|
  uSID Block              Node ID          Func + Instance  Padding
  [LOCATOR-BLOCK]::/48    Active uSID      Next uSID        ::0
                          (LNL=16)         (FL=4, AL=12)
```

| Field                 | Bits | Description                                                                                      |
|-----------------------|------|--------------------------------------------------------------------------------------------------|
| uSID Block            | 48   | Identifies the PoP domain; drawn from the SRv6 locator block.                                    |
| Node ID (Active uSID) | 16   | Identifies the specific node (Locator-Node / LNL = 16) within the PoP domain.                    |
| Function + Instance   | 16   | Shared slot (Next uSID) containing a 4-bit Function (FL = 4) and a 12-bit Instance ID (AL = 12). |
| Padding               | 48   | Zero; this carrier has only one Next uSID slot, so there is never a further C-SID to shift in.   |

This layout is referred to elsewhere in the platform's documentation (e.g. the Addressing Plan) as the `uFMT 48+16` (F4816) carrier format with a shared service slot. Bit positions cited in absolute terms (e.g. "bits 49–64") are 1-based and numbered from the most-significant bit of the 128-bit container — bit 1 is the first bit of the uSID Block, bit 49 is the first bit of the Node ID, and so on.

The uSID components correspond to RFC 9800's own Locator-Node (LNL), Function (FL), and Argument (AL) length parameters. The platform's parameters are:

| Component                    | Bits | Range         | Purpose                                                             |
|------------------------------|------|---------------|-----------------------------------------------------------------------|
| Node-ID (Locator-Node / LNL) | 16   | 0x0001-0xDFFF | Identifies the target node (GIB / Locator-Node).                    |
| Function (FL)                | 4    | 0xE-0xF       | Selects the behavior universe (in LIB).                             |
| Instance ID (Argument / AL)  | 12   | 0x001-0xFFF   | Identifies VRF ID (for 0xE) or EVI ID (for 0xF). 0x000 is reserved. |

The 16-bit C-SID space is partitioned into a Global ID Block (GIB) for Node-IDs (`0x0001–0xDFFF`) and a Local ID Block (LIB) for local endpoint functions (`0xE000–0xFFFF`). For the 16-bit Next uSID slot, this is partitioned into a 4-bit Function field (supporting L3 overlay `0xE` and future L2 overlay `0xF`) and a 12-bit Instance ID field (supporting up to 4,095 dynamic VRF/EVI IDs per node), preventing forwarding loops during C-SID shift operations.

The uSID block is PoP-scoped, which allows clean per-PoP filtering and isolation in the underlay. Each node individually advertises its own `/64` locator prefix (`[uSID Block][Node-ID]::/64`) into the underlay IGP and BGP IPv6 Unicast. Underlay routing resolves packets based on this `/64` prefix to deliver them to the exact egress node.

Per RFC 9800 §5.3 ("Recommended Installation of CSIDs in FIB"), the node installs a FIB/local-SID-table entry that matches only its own `/64` locator: `[uSID Block][Node-ID]::/64` via longest-prefix match. Function here always selects a terminal endpoint behavior (`uEnd.DT46` / `uEnd.DT2`), and RFC 9800 §4.2.7 states that no NEXT-CSID counterpart is defined for `End.DT`/`End.DX`-family behaviors — they instead run RFC 8986 §4.4–4.11's original decapsulation procedure under the REPLACE-CSID flavor, with the Argument value ignored by the SR segment endpoint node. RFC 9800 defines no destination-address shift or rewrite for this case at all: the shift/rewrite pseudocode in §4.1 (NEXT-CSID) and in §4.2.1 (REPLACE-CSID's own `End`/`End.X` forwarding case) only applies when there's a further C-SID to expose for a subsequent segment endpoint, which never happens here. The node therefore reads Function (bits 65–68) and Instance ID (bits 69–80) directly from the unmutated destination address to execute the local endpoint behavior (e.g. L3 routing table lookup or L2 MAC lookup) and select the correct tenant VRF or Bridge Domain table.

> [!NOTE]
> **This is RFC 9800 §4.2.7's own behavior, not a deviation from it.** NEXT-CSID (§4.1) and REPLACE-CSID's `End`/`End.X` forwarding case (§4.2.1) both shift the destination address to expose the *next* C-SID for a subsequent segment endpoint — but RFC 9800 explicitly carves out `End.DT`/`End.DX`-family behaviors as having no such shift step: per §4.2.7, they run RFC 8986's original procedure directly, ignoring the Argument bits for FIB purposes. Since Function here always selects one of these terminal behaviors (`uEnd.DT46` / `uEnd.DT2`), there is no further C-SID to advance to, and no shift is ever prescribed. (A literal 128-bit left-shift, had one been required, would in any case have corrupted the uSID Block in bits 1–48 — its low 32 bits would be overwritten by the Node-ID — which never squares with the uSID Block reappearing unchanged; reading Function/Instance ID directly at their fixed offsets was always the only coherent reading, and RFC 9800 §4.2.7 confirms it's also the RFC-prescribed one.)

---

## Behavior Registry

The Function field of the C-SID encodes the compressed SRv6 operation applied at the endpoint. RFC 9800 defines Locator-Block and Locator-Node as components of the SID address prefix, but does not itself define Locator-Block granularity as per-PoP or specify a per-Locator-Block Function-reuse rule — that mapping is this platform's own allocation policy: each PoP is assigned its own `/48` uSID/Locator-Block (see [Addressing Plan](README.md)), so identical Function codes in different PoPs address entirely different SIDs, letting the Function space be reused across PoPs without exhaustion.

The 4-bit Function field divides the `0xE000` starting LIB space into two entirely independent service universes:

| Function (Hex) | Function (Dec) | Active C-SID Representation | Decimal Range | Behavior    | Description                                                                                                |
|----------------|----------------|-----------------------------|---------------|-------------|--------------------------------------------------------------------------------------------------------------|
| `0x0 - 0xD`    | `0 - 13`       | GIB Reserved                | —             | Reserved    | Reserved for Node-IDs (Global ID Block).                                                                   |
| `0xE`          | `14`           | `0xE000 \| VRF_ID`          | `57344-61439` | `uEnd.DT46` | **L3 Service Universe (Type 5)**: Shift, decapsulate, and lookup in tenant VRF (both IPv4/IPv6).           |
| `0xF`          | `15`           | `0xF000 \| EVI_ID`          | `61440-65534` | `uEnd.DT2`  | **L2 Bridging Universe (Type 2)**: Shift, decapsulate, and lookup in Bridge Domain MAC table (Future Use). |

---

## VRF / EVI ID (Argument)

The 12-bit Argument (AL = 12) is embedded in the lower 12 bits of the shared 16-bit Next uSID slot. Its semantics depend on the active Function code:
* When Function is `0xE`, it represents the **VRF ID** for L3 overlays.
* When Function is `0xF`, it represents the **EVI ID / Bridge Domain ID** for L2 overlays.

Instance ID `0x000` is reserved and must not be allocated. Usable Instance IDs range from `0x001` (1) to `0xFFF` (4095).

The platform allocates IDs sequentially within each PoP's namespace when a tenant service is first instantiated at that PoP. The platform tracks these mappings independently per PoP.

Per RFC 9800 §5.3, the node installs its local-SID-table entry to match only the Locator-Block and Node-ID — `[uSID Block][Node-ID]::/64` — via longest-prefix match; this design never shifts the destination address to expose Function at a shorter prefix (see [SID Structure](#sid-structure-usid-carrier) for why), so Function and Instance ID are instead read directly from their fixed offsets in the unmutated address. Per §4.2.7, the Argument value is ignored by the SR segment endpoint node for `End.DT`/`End.DX`-family behaviors, so the Instance ID is never part of the local-SID-table match key itself — VRF/Bridge Domain selection happens as a separate step (below).

### Instance ID to Infrastructure Mapping

1. **The platform** allocates a VRF ID or EVI ID (1 to 4095) when a tenant service is instantiated at a PoP.
2. **The egress node** resolves the arriving compressed SID to a specific VRF or Bridge Domain instance as described in [SID Structure](#sid-structure-usid-carrier): match the `/64` locator, then read Function and Instance ID directly from their fixed offsets in the unmutated address to select the forwarding instance.
3. **Tenant routes / MAC addresses** are imported into the selected forwarding instance by the control plane.

The Instance ID is a platform-local integer managed by the provisioning layer and does not directly encode any external record or attachment ID.

---

## SIDs per Tenant Service

A tenant service requires active uSID sequences based on its overlay type:

| Service Type                               | SIDs Required                                         |
|--------------------------------------------|---------------------------------------------------------|
| L3 Tenant VRF (IPv4, IPv6, or Dual-Stack)  | 1 — `uEnd.DT46` (Function `0xE` / `0xE000 \| VRF_ID`) |
| L2 Tenant EVI (VPLS bridging — Future Use) | 1 — `uEnd.DT2` (Function `0xF` / `0xF000 \| EVI_ID`)  |

> [!NOTE]
> **Single-Stack L3 Efficiency & Future-Proofing.** Running a pure IPv6-only (or IPv4-only) tenant stack over the unified `uEnd.DT46` behavior carries zero performance or control plane overhead:
> *   **Data Plane**: The inner packet's IP version (IPv4 vs. IPv6) is inspected at line rate by the DPU or host eBPF parser. A pure IPv6 packet is processed in the same clock cycles under a `uEnd.DT46` instruction as it would be under a dedicated `uEnd.DT6` instruction.
> *   **Control Plane**: The BGP EVPN control plane only advertises routes for the active address families configured in the VRF.
> *   **Future-Proofing**: This design provides "future-proofing for free." If a tenant subsequently introduces the other address family (e.g. enabling IPv4 on an IPv6-only VRF), no SID renumbering or locator re-architecting is required. You simply enable BGP signaling for the new address family, and the same `0xE000 | VRF_ID` SID handles the traffic immediately on day one.

---

## Control Plane Mechanics (BGP EVPN L2VPN)

Tenant routing and bridging states are distributed across the PoP fabric via BGP EVPN under the L2VPN EVPN address family (AFI=25, SAFI=70). The control plane maintains a deterministic mapping between BGP route identifiers and SRv6 VRF/EVI IDs, ensuring consistent SID construction at every ingress node.

### Instance ID Signaling

The VRF ID or EVI ID is signaled as part of the BGP EVPN Route Distinguisher (RD), alongside the originating node's own identity, so the RD is unique per node. Per RFC 4364 §4, an RD is an 8-byte value composed of a 2-byte Type field and a 6-byte Value field. The platform uses RD Type 1 (RFC 4364 §4.2), the type RFC 7432 §7.9 recommends for EVPN, whose 6-byte Value field consists of a 4-byte Administrator subfield (the originating node's own IPv4 loopback address) and a 2-byte Assigned Number subfield carrying the Instance ID (bounded to `1–4095` to match the C-SID Argument capacity):

```
RD (Type 1) — 8 bytes total:
├── Type: 0x0001 (2 bytes)
└── Value (6 bytes):
     ├── Node loopback IPv4 address (4 bytes)
     └── VRF / EVI ID (2 bytes, range 1-4095)
```

Because the Administrator subfield is each node's own loopback address, the RD is naturally unique per node — no two nodes ever construct the identical RD for the same tenant instance, even though the Instance ID portion is the same. This is RFC-compliant: RFC 7432 §7.9 requires an RD to be unique per MAC-VRF *on a given PE* and recommends exactly this IP-address-based encoding. Per-node RD uniqueness means every node's advertisement for a given tenant instance is a structurally distinct NLRI, so a route reflector retains all of them without needing BGP ADD-PATH (RFC 7911) to preserve alternate paths — there is never a single contested NLRI in the first place. Route Targets ([below](#route-target-distribution)) still govern import/export scope; the RD's role here is per-node route distinctness, not scope.

### Route Target Distribution

Route Targets (RTs) define the route import/export policy for each tenant service. Each instance is assigned a unique import RT and export RT (or a shared set for full mesh).
* **For L3 Services (Type 5)**: RTs are carried on BGP EVPN Route Type 5 (IP Prefix) routes (RFC 9136 §3, §4.4.1–§4.4.3).
* **For L2 Services (Type 2)**: RTs are carried on BGP EVPN Route Type 2 (MAC/IP Advertisement) routes (RFC 7432).

The control plane programs the local service instance with its RT set and Instance ID. When a node's BGP process receives an EVPN route, it validates RT membership against the local instance's configured import targets to determine import eligibility. Once imported, the tenant prefixes or MAC addresses are programmed into the node's data-plane forwarding table under the matching instance (selected as described in [Instance ID to Infrastructure Mapping](#instance-id-to-infrastructure-mapping)), not by decoding the RD at the data plane.

### Route Injection

Tenant prefix reachability and MAC learning are distributed via BGP EVPN, carried in the MP_REACH_NLRI/MP_UNREACH_NLRI attribute (RFC 4760) using AFI=25 (L2VPN) / SAFI=70 (EVPN) per RFC 7432 §7.

#### L3 Services: EVPN Route Type 5 (IP Prefix Route)
Each Type-5 route carries:
- **Route Distinguisher** — identifies the originating node and tenant VRF instance (embeds the node's loopback address + VRF ID)
- **Route Targets** — control route import/export scope
- **Prefix** — the tenant subnet (inner payload address)
- **Next Hop** — the SRv6 SID of the egress node (`[uSID Block][Node-ID][Func (0xE) + VRF ID (12)]::0`)
- **BGP Prefix-SID Attribute** — the RFC 9252 SRv6 L3 Service TLV, describing the SID's internal structure (below)

#### L2 Services: EVPN Route Type 2 (MAC/IP Advertisement — Future Use)
Each Type-2 route carries:
- **Route Distinguisher** — identifies the originating node and tenant EVI instance (embeds the node's loopback address + EVI ID)
- **Route Targets** — control route import/export scope
- **MAC / IP Address** — the tenant hardware and binding IP addresses
- **Next Hop** — the SRv6 SID of the egress node (`[uSID Block][Node-ID][Func (0xF) + EVI ID (12)]::0`)
- **BGP Prefix-SID Attribute** — the RFC 9252 SRv6 L2 Service TLV, describing the SID's internal structure

#### RFC 9252 SRv6 Service Encoding

Per RFC 9252 §5 and §6, the BGP Next Hop is set directly to the SRv6 SID — here, the full 128-bit compressed SID `[uSID Block][Node-ID][Func (4) + Instance ID (12)]::0` — which RFC 9252 permits.

The route's BGP Prefix-SID Attribute (RFC 8669) carries an **SRv6 L3 Service TLV** (for Type 5) or **SRv6 L2 Service TLV** (for Type 2), containing one **SRv6 SID Structure Sub-Sub-TLV** (RFC 9252 §3.2.1) describing how to parse the SID already present in Next Hop:

| Sub-Sub-TLV Field          | Value | Maps to                                               |
|-----------------------------|-------|---------------------------------------------------------|
| Locator Block Length (LBL) | 48    | uSID Block (PoP domain prefix)                        |
| Locator Node Length (LNL)  | 16    | Node-ID — the routed portion of the locator           |
| Function Length (FL)       | 4     | Function — local behavior code (`0xE` or `0xF`)       |
| Argument Length (AL)       | 12    | Instance ID (Argument)                                |
| Transposition Length (TL)  | 0     | No transposition into an NLRI label field             |
| Transposition Offset (TO)  | 0     | Not applicable when TL = 0                            |

LNL and FL are reported separately because they carry different semantics per RFC 8986 §3.1: LNL covers the bits the underlay actually consumes to route to a node (forming the /64 locator prefix), while FL covers bits that are local and opaque to routing — read directly from the unmutated SID rather than exposed via any address shift (see [SID Structure](#sid-structure-usid-carrier)). **TL = 0** because the full 128-bit compressed SID is carried directly in the Next Hop field with no separate NLRI label transposition. The remaining 48 bits of zero padding sit outside the declared four fields by design and are fully conformant.

### SID Programming Flow

1. The control plane allocates a VRF ID or EVI ID for a new tenant service at a PoP.
2. EVPN routes carry each service's Route Targets, establishing import/export policy across all nodes in the PoP.
3. EVPN routes inject tenant prefixes (Type-5) or MACs (Type-2), carrying the Instance ID in the RD and the egress node's SRv6 SID as next hop.
4. Each node in the PoP programs its FIB locator entry and local-SID-table Function entries as described in [SID Structure](#sid-structure-usid-carrier), so the node is ready to dispatch to the correct endpoint behavior for this instance.
5. Ingress nodes construct the outer IPv6 header using the egress SID from the EVPN next hop, encapsulating the tenant packet.

---

## Instance ID Space and Scale

Because the 12-bit Instance ID is excluded from the FIB match key (see [VRF / EVI ID (Argument)](#vrf-evi-id-argument)) and instead read directly by the endpoint behavior, a single `/48` uSID block provides **4,095 usable VRF IDs** under the `0xE___` universe and **4,095 usable EVI IDs** under the `0xF___` universe. This limit applies per uSID Block, not per node or per PoP — all nodes within a PoP share the same Instance ID namespaces under a given locator block, but a PoP with more than one uSID Block (see [Scaling Beyond 4k Instances](#scaling-beyond-4k-instances)) has a multiple of this capacity.

This is a distinct ceiling from the Node-ID (Locator-Node) space: the GIB range `0x0001–0xDFFF` yields **57,343 usable Node-IDs per uSID Block**, bounding the number of physical nodes a single locator block can address, independent of how many tenant VRF/EVI instances those nodes host.

### Scaling Beyond 4k Instances

When a PoP approaches the Instance ID ceiling in either universe, the platform supports two mitigation paths that preserve the tenant experience:

**Secondary uSID Block allocation.** A second `/48` locator block is assigned to the same physical PoP. This creates a new, independent namespace of 4,095 VRF/EVI IDs. Each block allocates its full range independently: the primary block serves IDs 1-4,095 and the secondary block serves IDs 1-4,095, for a combined capacity of 8,190 instances on the same hardware. Tenant SIDs referencing the secondary block use the secondary `/48` prefix with the same Node-ID, Function, and Instance ID encoding. The underlay routes to the secondary block's per-node `/64` locators, which resolve to the same physical nodes. This requires a supplementary RIR PI allocation — RIR policy does not guarantee the secondary block will be contiguous with the primary one, so the platform's PoP-edge aggregation design must tolerate a non-contiguous secondary locator block — but involves no tenant-visible renumbering.

**Logical PoP partitioning.** The physical PoP is split into two logical domains (e.g., PoP-1A, PoP-1B), each with its own `/48` locator block and independent namespace. Cross-domain steering is handled by a small set of designated inter-domain gateway nodes — not every ingress node — that hold full reachability to the other domain's locators and instances; a domain's non-gateway nodes only program state for instances attached within that domain. This is what actually relieves per-node hardware forwarding-table pressure (distinct from Instance ID space): the aggregate FIB load for cross-domain tenants is concentrated on the gateway nodes rather than spread across every ingress node, so this approach is preferable when hardware forwarding table limits constrain a single logical domain.

### Instance Migration Continuity

When a tenant instance is migrated to a secondary block or logical partition, the platform must maintain route continuity via a make-before-break procedure: both the old and new SID contexts remain live in the data plane during the overlap window, and the old context is decommissioned only after traffic telemetry confirms zero hits on the legacy key. The migration sequence, including rollback criteria and overlap window sizing, is defined in the platform's service migration runbook.

---

## Alternatives Considered (Service & Instance ID Encoding)

The platform evaluated three architectural options for encoding the service function and the target Instance ID within the 128-bit SID container:

### Option 1: Independent 16-bit Slots (Separate Function and Argument)
This option allocates two independent 16-bit slots: one for the 16-bit Function C-SID (Next uSID) and one for the 16-bit Instance ID (Argument).
*   **Packet Layout**: `[uSID Block (48)][Node-ID (16)][Function (16)][Instance ID (16)][Padding (32)]`
*   **Pros**:
    *   Full 16-bit namespace for Node-IDs, Functions, and Instance IDs (supporting up to 65,535 instances per block).
    *   Clean separation of concerns: the Function specifies the behavior, and the Argument provides the instance parameter.
*   **Cons**:
    *   Consumes 32 bits of the C-SID container, leaving only two remaining 16-bit slots for other micro-SIDs (reducing shift-and-forward depth to 3 segments).
    *   Standard Linux kernel `seg6local` does not support dynamic table lookup from the C-SID Argument, requiring an eBPF/XDP workaround on the host CPU.

### Option 2: Shared 16-bit Slot (Sub-field Split / Selected Design)
This option packs both the Function and the Instance ID into a single 16-bit block by splitting it into two sub-fields: a 4-bit Function (FL) and a 12-bit Instance ID (AL).
*   **Packet Layout**: `[uSID Block (48)][Node-ID (16)][Func (4)+Instance (12)][Padding (48)]`
*   **Pros**:
    *   Saves space in the C-SID container, utilizing only a single 16-bit slot for both service and routing table parameters.
    *   **O(1) Egress FIB Complexity**: The egress DPU or ASIC only needs to program a single forwarding rule per behavior type (e.g., matching the prefix `[uSID Block][Function]::/52`), regardless of the number of instances. The hardware parser extracts the 12-bit Instance ID metadata on the fly.
*   **Cons**:
    *   Constrains the Instance ID namespace to 12 bits (maximum 4,095 VRF/EVI IDs per node).
    *   Requires a custom eBPF (tc/XDP) datapath on Linux hosts today, as vanilla Linux kernel `seg6local` cannot parse sub-field arguments dynamically.

### Option 3: Per-Service C-SID Allocation
In this option, there is no explicit Instance ID (AL = 0) in the packet header. Instead, the egress node allocates a unique 16-bit C-SID (from the Local ID Block range `0xE000–0xFFFF`) for each tenant instance, mapping it 1:1 to a specific behavior and table lookup.
*   **Packet Layout**: `[uSID Block (48)][Node-ID (16)][Service-SID (16)][Padding (48)]`
*   **Pros**:
    *   Fully standard and natively supported out-of-the-box by vanilla Linux kernels and major router OSes. No custom argument parsing required.
    *   Supports up to 8,191 local instances (within the standard `0xE000-0xFFFE` LIB range).
*   **Cons**:
    *   **O(N) Egress FIB Complexity**: The egress node must program one hardware TCAM/FIB entry per tenant instance, which can cause table exhaustion on scale-heavy PEs or DPUs.
