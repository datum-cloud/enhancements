# Tenant Addressing — Subnet and Endpoint Allocation

## Overview

This document defines how tenant VPC address space is structured and subdivided down to the individual endpoint, for both IPv6 and IPv4. The two are not the same kind of address family in this design: IPv6 (ULA or GUA) is the VPC's base address space, per the [Addressing Plan](README.md#vpc-address-space-options); IPv4 is a separate, additional address family a VPC may also use. Whether IPv4 is turned on for a given VPC by default is a platform provisioning decision outside the scope of this document — this document only defines the addressing scheme IPv4 follows when it is in use.

- **IPv6** follows the ULA option (`fd00::/8` family) described in the [Addressing Plan](README.md#vpc-address-space-options): each VPC receives its own unique, non-overlapping `/48`, subdivided down to a `/96` per endpoint. GUA tenant addressing (Option B) is out of scope here — see [GUA Tenant Addressing Policy](README.md#gua-tenant-addressing-policy).
- **IPv4** uses a fixed, platform-wide RFC 1918 layout reused identically across every tenant VPC that uses it — not a unique-per-VPC allocation like IPv6 — because IPv4 address scarcity makes per-tenant uniqueness impractical at scale. See [Shared, Not Unique, Per VPC](#shared-not-unique-per-vpc-ipv4) for why that trade-off is necessary.

---

## IPv6 Tenant Addressing (ULA)

Rather than assigning individual `/128` addresses to each attached interface, every VM or load balancer interface receives an entire `/96` block. This gives each endpoint a large local address space of its own — for secondary IPs, containers, or pods — without requiring an additional subnet allocation from IPAM every time an endpoint needs more addresses.

> [!NOTE]
> This section covers ULA tenant addressing only. GUA tenant addressing (Option B) follows the tenant's own RIR-allocated block and is out of scope here — see [GUA Tenant Addressing Policy](README.md#gua-tenant-addressing-policy).

### IPv6 Address Hierarchy

```
fd20::/20                         Tenant VPC ULA pool
└── /48   per VPC                 IPAM-assigned
     └── /64   per region         tenant subnet
          └── /96   per endpoint  VM / load balancer interface
               ├── ::1 of the /64   subnet router / gateway
               ├── ::/96            primary interface address (endpoint's first address)
               └── remainder of /96  secondary IPs, containers, pods
```

| Level                   | Prefix | Bits from parent | Count within parent                            | Purpose                                                |
|-------------------------|--------|------------------|------------------------------------------------|--------------------------------------------------------|
| Tenant VPC ULA pool     | `/20`  | —                | —                                              | `fd20::/20` — platform-managed pool for tenant VPCs    |
| VPC                     | `/48`  | 28               | ~268,435,456 (2^28) `/48`s                     | One tenant VPC                                         |
| Region subnet           | `/64`  | 16               | 65,536 (2^16) `/64`s per VPC                   | One tenant subnet within a region                      |
| Endpoint                | `/96`  | 32               | ~4,294,967,296 (2^32) `/96`s per region subnet | One VM or load balancer network interface              |
| Address within endpoint | `/128` | 32               | ~4,294,967,296 (2^32) addresses per endpoint   | Primary address plus secondary/pod/container addresses |

### VPC Allocation (`/48`)

Each tenant VPC is assigned a `/48` out of the platform's tenant ULA pool, `fd20::/20`. Allocation is performed and tracked by IPAM at VPC creation time.

`fd20::/20` is a fixed, platform-managed sub-range of the broader ULA space (`fd00::/8`, RFC 4193). This is a deliberate departure from the general guidance in the [main addressing plan](README.md#vpc-address-space-options), which — per RFC 4193 §3.2.1 — recommends tenants generate their own pseudo-random Global ID rather than drawing from a narrow shared sub-range, since a narrow range reduces entropy and increases collision probability.

> [!NOTE]
> That guidance addresses collision risk when tenants *self-generate* a Global ID. Here, uniqueness is instead guaranteed structurally: `/48` VPC allocations are assigned centrally by IPAM out of `fd20::/20`, not randomly chosen by tenants. Because the allocator itself guarantees no two VPCs receive the same `/48`, the collision risk RFC 4193 §3.2.1 is concerned with does not apply in the same way. This model is a deliberate trade — structured, IPAM-issued allocation in exchange for the dense, predictable subnetting described below — and it depends on IPAM remaining the sole issuer of `/48`s from this range.

### Regional Subnet Allocation (`/64`)

Each VPC subdivides its `/48` into per-region `/64` subnets — one `/64` per region the VPC is attached to. This mirrors the per-PoP tenant subnet model in the [main addressing plan](README.md#tenant-subnet-allocation); "region" here corresponds to a platform PoP (or availability zone within one).

A VPC's `/48` supports up to 65,536 region subnets, which is not a practical constraint for any foreseeable deployment footprint.

#### Subnet infrastructure addresses

- **Subnet router / gateway (`::1`)** — the first address of the `/64` (e.g., `fd20:0a1b:2c3d:0001::1`) is reserved for the subnet's default gateway (virtual router).
- **No broadcast or network-address reservation** — IPv6 has no broadcast address, so unlike IPv4 there is no equivalent of a reserved `.0` network address or `.255` broadcast address at the subnet boundary.

### Endpoint Allocation (`/96`)

Instead of handing out individual `/128` addresses directly from the `/64`, the `/64` region subnet is divided into 2^32 (~4.29 billion) distinct `/96` blocks. When a VM instance or load balancer interface attaches to the subnet, it is assigned one whole `/96` block (e.g., `fd20:0a1b:2c3d:0001:0000:0000::/96`), not a single address.

- **Primary address (`::` of the `/96`)** — assigned to the endpoint's main network interface.
- **Remaining address space of the `/96`** — reserved exclusively for that endpoint to self-assign locally to secondary interfaces, containers, or pods, without requesting an additional subnet from IPAM.

This gives up density at the `/64` level (4.29 billion endpoints per region subnet, rather than an effectively unlimited number of `/128`s) in exchange for every endpoint having a large, pre-allocated local address block it can use on its own without a control-plane round trip.

### IPv6 Capacity Summary

| Resource                                 | Capacity       |
|-------------------------------------------|----------------|
| `/48` VPCs available in `fd20::/20`      | ~268.4 million |
| `/64` region subnets per VPC             | 65,536         |
| `/96` endpoints per region subnet        | ~4.29 billion  |
| Addresses available per endpoint (`/96`) | ~4.29 billion  |

### Relationship to the Main Addressing Plan

This IPv6 model refines, for the ULA option only, the endpoint-level detail of [Tenant Subnet Allocation](README.md#tenant-subnet-allocation) — replacing the `/128`-per-instance recommendation with a `/96`-per-endpoint model. The higher-level structure (VPC pool independence from infrastructure locator space, per-PoP/region subnetting, IPAM as system of record) is unchanged from the main plan.

Address space is only half of how a VPC reaches a region: each region subnet (`/64`) a VPC attaches to is backed by one VRF instance in the platform's SRv6 forwarding plane. See [SRv6 uSID Plan — VRF / EVI ID](srv6.md#vrf-evi-id-argument) for how that VRF Instance ID is assigned and encoded in the data plane.

---

## IPv4 Tenant Addressing

This section defines the platform's **default** tenant IPv4 scheme: a fixed, well-known layout reused identically across tenant VPCs, not a unique-per-VPC allocation like the [IPv6 tenant addressing model](#ipv6-tenant-addressing-ula) above. See [Shared, Not Unique, Per VPC](#shared-not-unique-per-vpc-ipv4) for why that's an acceptable — and necessary — trade-off for IPv4.

### IPv4 Address Hierarchy

```
10.128.0.0/9                Tenant IPv4 pool (platform-wide)
└── /12   per macro-region   one contiguous block per continent/macro-region
     └── /20   per region site    one subnet per regional site (PoP)
          └── /32  per endpoint   VM / load balancer interface, assigned directly from the site's /20
```

| Level                 | Prefix | Bits from parent | Count within parent                                                      | Purpose                                         |
|-----------------------|--------|------------------|--------------------------------------------------------------------------|-------------------------------------------------|
| Tenant IPv4 pool      | `/9`   | —                | —                                                                        | `10.128.0.0/9` — platform-wide default pool     |
| Macro-region supernet | `/12`  | 3                | 8 `/12`s in the `/9`, assigned sequentially as macro-regions come online | One contiguous block per continent/macro-region |
| Region site subnet    | `/20`  | 8                | 256 `/20`s per macro-region (2,048 total in the `/9`)                    | One subnet per regional site                    |
| Endpoint              | `/32`  | 12               | 4,096 `/32`s per site subnet (4,092 usable)                              | One VM or load balancer network interface       |

### Platform-Level Pool (`/9`)

The platform reserves `10.128.0.0/9` for default tenant VPC networks. This is the platform's default base pool and is the source for every macro-region supernet described below. A VPC may select an alternate base pool instead — see [Configurable Base Pool](#configurable-base-pool) — in which case the same macro-region/site/endpoint carving logic applies relative to that base rather than to `10.128.0.0/9`.

### Configurable Base Pool

`10.0.0.0/8` is the most heavily used RFC 1918 block in enterprise networking, and the platform's default base pool (`10.128.0.0/9`) sits inside it. A tenant setting up a VPN or Interconnect to their own on-premises network frequently finds their internal addressing already overlaps this range.

**The fix:** the base pool is a single choice made at VPC creation time, not a hardcoded constant. A VPC may instead be created with an alternate RFC 1918 base — `172.16.0.0/12` or `192.168.0.0/16` — while keeping the same automatic macro-region/region-site/endpoint carving logic described in this document. The base is fixed for the lifetime of the VPC once chosen.

**The benefit:** this retains the zero-friction, fully-automatic per-region subnet model — a tenant never manually carves subnets regardless of which base is selected — while avoiding an immediate collision with a tenant's existing internal addressing during VPN or Interconnect setup.

Because `172.16.0.0/12` and `192.168.0.0/16` are smaller than the default `/9`, the same fixed `/20` site subnet and `/32` endpoint sizing leaves less room for the macro-region layer:

| Base pool                | Host bits available (base → `/32`) | Macro-region layer                                                              | Site `/20` subnets available |
|--------------------------|------------------------------------|---------------------------------------------------------------------------------|------------------------------|
| `10.128.0.0/9` (default) | 23                                 | Yes — 8 possible `/12`s, 256 sites each                                         | 2,048 total                  |
| `172.16.0.0/12`          | 20                                 | None — the `/12` itself is the only supernet; sites are carved directly from it | 256 total                    |
| `192.168.0.0/16`         | 16                                 | None                                                                            | 16 total                     |

In both alternate cases, the entire base pool is itself a single contiguous block, so it is already advertisable as one aggregate route without needing a macro-region subdivision — the macro-region layer's BGP-summarization benefit only matters at the scale of the default `/9`, where multiple continents share one pool.

### Macro-Region Supernet (`/12`)

This section describes the default base pool. When a VPC selects an alternate base instead (see [Configurable Base Pool](#configurable-base-pool)), the macro-region layer is skipped and site subnets are carved directly from the alternate base.

The `/9` is first carved into `/12` supernets, one per continent/macro-region (e.g., Americas, EMEA, APAC), before any regional site subnets are assigned. Region site subnets are only ever carved out of their macro-region's `/12` — a site is never assigned a `/20` from a macro-region other than the one it belongs to.

The `/9` holds 8 possible `/12` supernets, assigned sequentially as macro-regions are brought online; unassigned supernets remain reserved for future macro-regions. Actual macro-region → `/12` assignments are operational data tracked in the platform's IPAM service, not fixed by this design.

**Why this matters:** without this layer, region site `/20`s are handed out from the `/9` in whatever order regions are brought online, with no relationship between a region's geography and its position in the address block. That leaves an upstream or on-premises BGP router with no way to summarize — it must carry a distinct route for every region's `/20` individually. Fixing macro-region boundaries at `/12` means every region within a macro-region is numerically contiguous, so a router advertising or filtering routes for an entire continent can do so with a single `/12` aggregate instead of dozens of discontiguous `/20` host routes.

### Region Site Subnet (`/20`)

Each regional site (PoP) is assigned a non-overlapping `/20` carved out of its macro-region's `/12`, fixed and identical across every tenant VPC using the default addressing scheme. "Region site" here is the same PoP-scoped concept as "region" in the [IPv6 Address Hierarchy](#ipv6-address-hierarchy) above — the IPv4 model just adds the macro-region layer on top of it. Actual site → `/20` assignments are operational data tracked in the platform's IPAM service, not fixed by this design.

A `/20` provides 4,096 host addresses (2^12), of which 4,092 are usable by endpoints once the four reserved addresses below are excluded.

#### Reserved addresses

Every tenant IPv4 subnet reserves four `/32` addresses that cannot be assigned to an endpoint:

| Position                            | Example (`/20`) | Purpose                                              |
|-------------------------------------|-----------------|------------------------------------------------------|
| First address (`.0`)                | `10.128.0.0`    | Subnet network address                               |
| Second address (`.1`)               | `10.128.0.1`    | Default gateway (virtual router)                     |
| Second-to-last address (`.255.254`) | `10.128.15.254` | Reserved for platform internal maintenance/expansion |
| Last address (`.255.255`)           | `10.128.15.255` | Subnet broadcast address                             |

Every other `/32` in the site's `/20` — 4,092 of the 4,096 total — is fully usable for tenant endpoints.

### Just-in-Time Regional Subnet Instantiation

The macro-region and region-site layers above define a *reservation* scheme, not a provisioning trigger. A region's `/20` has a fixed, permanent position in the address plan from the moment the region is added — it will never be renumbered or reused for another region — but reserving that position is not the same as instantiating it.

**The fix:** subnet creation is lazy. A region's `/20` is only actually created — instantiated in the VPC's route table, with its gateway and reserved addresses provisioned — the first time a tenant provisions a resource in that region. Regions with no live resources have a reserved position in the addressing plan but no instantiated subnet and no control-plane state.

**The benefit:**

- **Cleaner route tables** — a VPC's route table only contains entries for the regions it actually uses, not a preallocated subnet for every region the platform supports.
- **Reduced control-plane noise** — no gateway, reservation, or route-advertisement state is created for a region until something actually needs it.
- **Easier external peering** — a peer only needs visibility into the regions a VPC is actively using, not every region that merely exists in the addressing scheme.

> [!NOTE]
> This is distinct from the [macro-region aggregation](#macro-region-supernet-12) described above. Aggregation at the `/12` boundary is about what an external BGP router needs to carry — a single continent-wide route regardless of which sites within it are active. Lazy instantiation is about what a tenant VPC's own route table and control plane need to hold — only the sites it actually uses. The two combine: an external peer sees one `/12` per macro-region; a VPC's own route table sees only the `/20`s it has actually instantiated within that `/12`.

### Endpoint Allocation (`/32`)

Each VM instance or load balancer interface created within a regional site receives a single `/32` address assigned directly from that site's `/20` range (e.g., `10.128.0.2`). Unlike the IPv6 model, there is no per-endpoint sub-block — IPv4's address scarcity means a `/32` is the unit of assignment, and any additional addresses an endpoint needs (secondary IPs, alias IP ranges) must be separately allocated from the same site subnet.

### Shared, Not Unique, Per VPC (IPv4)

The `/9` pool and its regional `/20`s are **not** allocated uniquely per tenant VPC. Every tenant VPC using the default IPv4 scheme reuses the same fixed regional `/20` — e.g., every tenant's subnet at a given site is the same `/20` — rather than each VPC receiving its own non-overlapping block, as happens in the [IPv6 tenant addressing model](#ipv6-tenant-addressing-ula) above.

This is a deliberate consequence of IPv4 scarcity, not an oversight: the `/9` only contains 2,048 `/20`s across all macro-regions combined, nowhere near enough to hand out a unique block per tenant VPC per region at any meaningful scale. Because this space is private (RFC 1918) and carried strictly as inner payload inside the overlay — the same isolation property that applies to ULA tenant space in the IPv6 plan — reuse across tenants is safe as long as:

- The address space never leaves the overlay or is exposed to another tenant's VRF (same import/export isolation requirements as ULA).
- Tenants that require non-overlapping IPv4 ranges — for direct VPC-to-VPC peering, or interconnects that can't tolerate NAT — must be assigned a custom, non-default IPv4 block instead of the shared default range. Selecting one of the [alternate base pools](#configurable-base-pool) only avoids collisions with the platform's own default `/9`; it does not make a VPC's block unique among tenants sharing that same alternate base. A block that must be non-overlapping across tenants is a fully custom allocation, which remains out of scope for this document.

### IPv4 Capacity Summary

| Resource                                   | Capacity |
|--------------------------------------------|----------|
| `/12` macro-region supernets in the `/9`   | 8        |
| `/20` region site subnets per macro-region | 256      |
| `/20` region site subnets in the `/9`      | 2,048    |
| Addresses per site subnet (`/20`)          | 4,096    |
| Usable endpoint addresses per site subnet  | 4,092    |
