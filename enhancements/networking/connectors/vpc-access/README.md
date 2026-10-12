---
title: Device Access to Galactic VPC
description: Let a consumer's own devices (phones, laptops, CI runners) join a Galactic VPC over an authenticated tunnel, so they reach VPC workloads on their private addresses. Extends Connectors from "the edge reaches a service on a device" to "a device reaches a network", and names the platform pieces that do not exist yet.
updated: 2026-10-12
tags: [connectors, galactic-vpc, masque, connect-ip, wireguard, iroh, srv6, mobile]
status: provisional
stage: alpha
latest-milestone: "TBD"
---

# Device Access to Galactic VPC

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [What exists today](#what-exists-today)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Shape of the system](#shape-of-the-system)
  - [API](#api)
  - [Authorization](#authorization)
  - [Addressing and routing](#addressing-and-routing)
  - [The data plane: choosing a tunnel](#the-data-plane-choosing-a-tunnel)
  - [Session lifecycle](#session-lifecycle)
  - [Clients](#clients)
  - [Prerequisite: peer authorization in datum-connect](#prerequisite-peer-authorization-in-datum-connect)
  - [Phasing](#phasing)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed](#infrastructure-needed)

## Summary

Galactic VPC gives a consumer's workloads private connectivity across every
location they run in. Today the only things inside a VPC are the platform's own:
compute instances attached through the CNI, the edge's Envoy reaching a
`NetworkService` or `instance` backend, and the NAT tier on the way out. A
consumer has no way to put **their own device** on the network. To reach an
instance on its private address, they have to expose it publicly through an
`HTTPProxy` and an `instance` backend first.

This enhancement proposes **device access to a Galactic VPC**. A device such as
a phone, laptop or CI runner registers as a `Connector`, is attached to a VPC
it is authorized for, is given an address inside it, and exchanges IP packets
with the VPC's workloads through an authenticated, NAT-traversing tunnel that
terminates at a Datum gateway and is bridged into the SRv6 fabric.

It builds on two provisional designs that already point here: the
[Connectors proposal](../initial-proposal/README.md), which defines `layer3`
advertisements, the `CONNECT-IP` capability and a future `ConnectorAttachment`,
and Galactic's
[MASQUE gateway proposal](https://github.com/datum-cloud/galactic/blob/main/docs/enhancements/proposals/masque-gateway-design.md),
which lists "Mobile App" among its clients. This document turns those threads
into a single design, identifies which pieces are missing, and asks the
networking team for one decision, the tunnel protocol, before work starts.

The first client motivating it is the Datum iOS app (`datum-mobile`), which
already shows a project's VPC resources read-only and has reserved the
entitlements a VPN-style Network Extension needs. Nothing here is specific to
iOS. `datum-connect` on desktop and a headless agent are the same client
shape.

## Motivation

Consumers deploy workloads into a Galactic VPC precisely so those workloads are
*not* on the public internet. The moment an engineer needs to reach one, to
open an admin UI, query a database, hit an internal API from a phone, or run an
integration test from CI, they have to punch a public hole: an `HTTPProxy`, a
`TrafficProtectionPolicy`, a hostname. That is the opposite of what the VPC was
for, and it only works for HTTP.

The connectors work already lets a device publish a service *to* the edge.
`datum-connect-daemon`'s own README names the missing half plainly:
**"Whole-subnet/VPC routing — the main gap versus Tailscale."**

### Goals

- A device the consumer controls can reach workloads in a Galactic VPC on
  their private (`fd20::/20` ULA) addresses, over any IP protocol, from behind
  NAT, on Wi-Fi or cellular.
- Access is granted per VPC through Datum IAM, and is revocable. Revoking it
  tears down live sessions within seconds, not at token expiry.
- The device is a first-class object: listed in the portal and app, with an
  owner, an address, a last-seen time and a session status.
- Traffic from devices is isolated per tenant exactly as workload traffic is,
  carried in the tenant's VRF and never seen by another tenant's fabric state.
- One client contract serves iOS, desktop and headless agents.
- The design composes with existing objects (`Connector`,
  `ConnectorAdvertisement`, `Network`, `NetworkBinding`) instead of adding a
  parallel model.

### Non-Goals

- **Exposing services that run on the device.** That direction (edge → device)
  already exists through `HTTPProxy` `connector` backends and is unchanged.
- **Site-to-site connectivity**, meaning routing a whole office LAN into a VPC
  through a gateway device. The `layer3` advertisement in the Connectors
  proposal models it and this design leaves room for it, but it brings
  route-leak and overlap questions that deserve their own document.
- **Device-to-device traffic** between two attached devices. It falls out of
  the routing if allowed, but it is off by default and not designed for here.
- **Full-tunnel or internet egress through the VPC.** Clients route only the
  VPC's prefixes. Internet-bound traffic stays on the device's own path.
- **IPv4-only VPCs.** New networks are IPv6-only (`spec.ipFamilies` refuses
  IPv4). The optional shared `10.128.0.0/9` space is out of scope until its
  overlap story is settled.
- **Browser clients** (WebTransport). The MASQUE proposal discusses them. They
  are a later phase at best.

## Proposal

### User Stories

#### Story 1: reach an internal service from a phone

Priya runs a staging API as a compute workload in her project's VPC, with no
public `HTTPProxy`. In the Datum iOS app she opens the project, taps **Connect
to VPC**, and approves the system VPN prompt the first time. The app shows
`Connected · fd20:3c5:71e:7::a:1`. In Safari she opens the API's private
address and it loads. Her phone's other traffic is unaffected.

#### Story 2: SSH to an instance from a laptop

An on-call engineer runs `datum-connect attach --network prod` and gets an
address in the VPC. They SSH to an instance's private address. When their
access to the project is removed, the session drops and the attachment shows
`Revoked`.

#### Story 3: CI integration tests against a private environment

A CI job authenticates with a machine account, attaches a short-lived
connector to the `ci` network for the length of the job, runs tests against
private endpoints, and deletes the attachment on exit. A forgotten attachment
expires through its lease.

#### Story 4: an administrator sees who is on the network

A project admin opens the network in the portal and sees every attached device:
owner, device name and OS, address, connected since, last seen, bytes. They can
disconnect any device.

### What exists today

| Piece | Where | What it does today |
|---|---|---|
| Galactic VPC data plane | `galactic`: `galactic-cni`, `galactic-router` | SRv6 uSID fabric with per-tenant VRFs and EVPN/BGP route distribution. Pods attach through the CNI. |
| Ingress into a VPC | `galactic-gateway` (`NetworkRule`), NSO `vpcpod.go` | Public VIP:port, XDP DSR load-balanced to VPC backends. Envoy reaches VPC pods for `HTTPProxy` `instance`/`networkService` backends. Service-level only. |
| Egress from a VPC | `galactic-nat` (`EgressShard`) | Stateful NAT66 (and NAT64) to the internet. |
| `Network`, `NetworkBinding`, `NetworkContext` | NSO `networking.datumapis.com/v1alpha` | A network, a declaration that it is needed in a location (with `spec.consumer`), and its presence there. |
| `Connector`, `ConnectorAdvertisement`, `ConnectorClass` | NSO `networking.datumapis.com/v1alpha1` | An iroh/QUIC peer with a public-key identity and a lease. Capabilities: **`ConnectTCP` only**. Advertisements: **`layer4` only**. |
| `datum-connect` | desktop app/CLI (Rust, iroh 1.0) | Registers a Connector, renews its lease, and serves local TCP services to Datum's `iroh-gateway`. Direction: **edge → device only**. |
| Android proof of concept | `datum-connect-daemon/examples/android-poc` | The Rust daemon cross-compiled for Android in a foreground service. Outward only. |

**Nothing today lets a device reach into a VPC.** The `layer3` advertisement,
the `CONNECT-IP`/`MASQUE` capabilities and `ConnectorAttachment` exist only in
the provisional Connectors and MASQUE documents, not in any CRD or binary.

### Notes/Constraints/Caveats

- **Direction is the whole design.** Every connector path built so far is
  edge → device: Envoy dials the device's iroh endpoint. This proposal is
  device → network, so the device dials Datum. The existing `iroh-gateway`
  cannot be reused as-is. It proxies HTTP CONNECT to the device, while this
  needs packets from the device bridged into a VRF.
- **Addressing.** VPCs are carved from `fd20::/20`, `/48` per VPC, `/64` per
  region subnet and `/96` per endpoint
  ([addressing](../../../../architecture/design/network/addressing/README.md)).
  Devices need addresses that fit this plan and are routable back to the
  gateway that holds the session (see
  [Addressing and routing](#addressing-and-routing)).
- **MTU.** `Network.spec.mtu` defaults to **1440** to leave room for the
  40-byte outer IPv6/SRv6 header. A client tunnel adds its own overhead (QUIC
  or WireGuard) *before* that, so the client MTU must be negotiated down to
  roughly **1280–1360**, not assumed. The IPv6 minimum of 1280 is the floor.
- **Mobile clients constrain the client side, not the platform.** iOS packet
  tunnels run in a Network Extension with a memory ceiling of about 50 MB, are
  started and stopped by the OS, and see frequent path changes (Wi-Fi ↔
  cellular). The protocol must tolerate connection migration and cheap
  reconnects, and the client core must be small.
- **The relay probe is IPv4-only.** Today's iroh relays are probed over
  HTTPS/443 on IPv4 only (`infra/docs/runbooks/vpc/iroh-relay-unreachable.md`).
  IPv6-only mobile networks need relays reachable over IPv6.

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| **A device becomes a path into a tenant's private network.** A stolen phone or leaked key is network access, not just account access. | Sessions are bound to *both* the device key and a live user token. Re-authorization runs per session and on every token refresh, not only at attach time. Revocation is pushed to gateways (see [Authorization](#authorization)). Short lease, with no session without a heartbeat. |
| **Cross-tenant leakage** through a shared gateway. | The gateway only ever writes a session's packets into the VRF of the network the attachment names, resolved from the attachment and never from the packet. It reuses the SRv6 uSID/VRF isolation the fabric already relies on. Fuzz and conformance tests assert that a packet from session A cannot be emitted into VRF B. |
| **The existing iroh peer-authorization gap** (see [Prerequisite](#prerequisite-peer-authorization-in-datum-connect)) is copied into a design with far larger blast radius. | Fix it first. This design requires the gateway to authorize the *peer identity*, not just the requested target. |
| **Abuse** (scanning, exfiltration, crypto mining over the tunnel). | Per-attachment rate and connection limits at the gateway, `NetworkPolicy` applied to device addresses like any other endpoint, flow logs per attachment, and a kill switch per attachment and per project. |
| **Route leaks**: a device advertising prefixes it doesn't own. | Devices in this design advertise nothing. `layer3` advertisements from devices are a non-goal, so a device can only originate its own assigned address. |
| **Operational load**: a new stateful tier on the edge. | Sessions are soft state rebuilt by client reconnect. A gateway restart costs a reconnect, not an outage. The gateway reuses `galactic-router`'s `BGPAdvertisement` contract instead of speaking BGP itself, as `galactic-gateway` and `galactic-nat` do. |

**Security review:** Security must review before this moves to
`implementable`. **UX review:** the portal and the iOS app own the attach and
disconnect flows and should review [Story 4](#story-4-an-administrator-sees-who-is-on-the-network)
together.

## Design Details

### Shape of the system

```
 Device (phone / laptop / CI)                          Datum edge                         Galactic VPC
 ─────────────────────────────            ──────────────────────────────────            ────────────────
 client core (Rust)                        VPC access gateway (new)
   iroh endpoint (device key) ──QUIC──►     terminates the tunnel
   tunnel: CONNECT-IP | WireGuard           authorizes peer + user per session
   route: VPC prefixes only                 bridges packets into the tenant VRF ──SRv6──►  workloads
   addr: from ConnectorAttachment           advertises device /128s via BGPAdvertisement   (fd20::/48)
        ▲                                         ▲
        │ register, attach, heartbeat              │ watch attachments, sessions
        └──────────── Datum API (NSO) ─────────────┘
              Connector · ConnectorAttachment · NetworkBinding (consumer: Connector)
```

Three parts are new:
1. `ConnectorAttachment`, in NSO.
2. The **VPC access gateway**, a new edge component, probably in `galactic`
   beside `galactic-gateway` and `galactic-nat`.
3. A client tunnel capability in the `datum-connect` core.

Everything else is reused.

### API

#### Connector: a new capability

The Connectors proposal already lists `CONNECT-IP`. This design needs exactly
one new capability on `Connector.spec.capabilities`, named for what it does
rather than the protocol, so the protocol choice below doesn't leak into the
API:

```yaml
apiVersion: networking.datumapis.com/v1alpha1
kind: Connector
metadata:
  name: priya-iphone
  annotations:
    datum.net/device-name: Priya's iPhone
    datum.net/device-os: iOS 26.1
spec:
  connectorClassName: datum-connect
  capabilities:
    - type: ConnectTCP          # existing: edge → device services
    - type: NetworkAccess       # new: device → network
status:
  capabilities:
    - type: NetworkAccess
      conditions:
        - type: Ready
          status: "True"
  connectionDetails:            # unchanged
    type: PublicKey
    publicKey:
      id: 2ovpybgj3snjmchns44pfn6dbwmdiu4ogfd66xyu72ghexllv6hq
      homeRelay: https://iroh-relay.us-central-1.datumconnect.net
```

#### ConnectorAttachment: new

This is the "future state" object from the Connectors proposal, made concrete.
It is the grant of network access to one connector, in the project control
plane next to the `Network`.

```yaml
apiVersion: networking.datumapis.com/v1alpha1
kind: ConnectorAttachment
metadata:
  name: priya-iphone-staging
spec:
  connectorRef:
    name: priya-iphone
  networkRef:
    name: staging
  # Optional. Defaults to the location nearest the device at connect time
  # (see "Where a device lands").
  location: null
  # Optional narrowing of what the device may reach. Defaults to the whole
  # network. Never wider than the network.
  allowedPrefixes: []
status:
  conditions:
    - type: Accepted        # authorized and valid
    - type: Ready           # address allocated, gateway programmed
    - type: Connected       # a live session exists right now
  addresses:
    - fd20:3c5:71e:7:0:0:a:1/128
  routes:                   # what the client should send into the tunnel
    - fd20:3c5:71e::/48
  dnsServers: []            # reserved for a future VPC resolver
  session:
    gateway: edge-us-central-1-a
    since: "2026-10-12T17:05:00Z"
    lastSeen: "2026-10-12T17:09:30Z"
```

**Why a separate object and not a field on `Connector`:** a device can be
attached to more than one network, an attachment is a *grant* that an admin
may create or revoke without touching the device's own object, and its
lifecycle (revocation, expiry, audit) is independent of the device's
registration.

#### NetworkBinding: a connector is a consumer

NSO's [network-in-every-location](https://github.com/datum-cloud/network-services-operator/blob/main/docs/enhancements/network-in-every-location.md)
design anticipates this. A consumer that needs a network in a location creates
a `NetworkBinding` naming itself in `spec.consumer`, and "a connector names a
network and the location it terminates in". The attachment controller creates
one binding per (network, gateway location) a session lands in, with
`spec.consumer` pointing at the `ConnectorAttachment`. Presence, subnet
allocation and readiness then come from the existing machinery, and the
gateway reads the `NetworkContext` for the network's rules (MTU, families)
like any other attaching component.

This design adds **no** `VPCIngressPoint` object (proposed in the MASQUE
document). Which gateways serve a network in a location is answered by the
binding and context the platform already has.

### Authorization

Two independent checks, both required, both re-evaluated:

1. **The user may attach.** Creating a `ConnectorAttachment` requires a new
   permission on the network, `networking.datumapis.com/networks.attach`,
   granted to project admins by default, plus ownership of the connector (the
   connector's creator, or `connectors.manage`). This is the same
   SubjectAccessReview path the API server already uses.
2. **This session is that device, for that user, right now.** When a tunnel
   opens, the gateway requires all of the following:
   - the peer's iroh endpoint ID equals the attachment's
     `connector.status.connectionDetails.publicKey.id`. This is cryptographic,
     from the QUIC handshake.
   - a user access token presented in-band, validated against Zitadel. It is
     the same token the client already holds, and is checked through the
     existing authn webhook path, not a new verifier.
   - a SubjectAccessReview for `networks.attach` on the attachment's network
     for that token's subject.

   The gateway repeats the SubjectAccessReview on a short interval (around
   60 s) and whenever the client rotates its token. A `Deny` closes the
   session.

**Revocation:** deleting the attachment, removing the user's access, or
disabling the connector moves `Accepted` to `False`. The gateway watches
attachments, so revocation closes live sessions within one watch event, not at
token expiry.

### Addressing and routing

- **Allocation:** the attachment controller allocates each device a `/96` from
  the network's `/48`, using the same per-endpoint convention as workload
  interfaces, through the existing IPAM (milo-os/ipam). The device uses the
  first address of its `/96` as its `/128`. A dedicated per-region "devices"
  `/64` inside the VPC keeps device addresses easy to recognize in flow logs
  and `NetworkPolicy`.

  <<[UNRESOLVED allocation source]>>
  A per-region devices `/64` gives policy authors a stable prefix ("all
  devices"). Allocating straight from the network's general subnets is
  simpler. Which do we want?
  <<[/UNRESOLVED]>>

- **Return path:** the gateway holding a session writes a `BGPAdvertisement`
  for that device's `/128` (or `/96`) into the tenant VRF, the same contract
  `galactic-gateway` and `galactic-nat` use, so `galactic-router` originates it
  over EVPN. Workloads reply to the device address, and the fabric carries the
  reply to that gateway. When the session moves gateways (roaming,
  reconnect), the old advertisement is withdrawn and the new one originated;
  the address does not change.
- **Client routes:** the client installs only `status.routes`, the network's
  `/48`, narrowed by `allowedPrefixes`. iOS includes exactly these in
  `NEPacketTunnelNetworkSettings.ipv6Settings.includedRoutes`. Everything else
  stays off the tunnel.
- **NetworkPolicy:** device addresses are endpoints like any other, so
  existing `NetworkPolicy` applies, with no special-casing.

### The data plane: choosing a tunnel

This is the one decision the networking team needs to make. Both options sit
behind the same API, the same authorization and the same client contract (an
address, a route set, an MTU). Only the bytes on the wire differ.

| | **A. MASQUE CONNECT-IP over iroh** (Galactic's stated direction) | **B. WireGuard** (the gVPC research doc's remote-edge termination) |
|---|---|---|
| Standard | RFC 9484 (CONNECT-IP), on HTTP/3 and QUIC | De facto standard, very widely deployed |
| NAT traversal | iroh hole punching, relay fallback, QUIC connection migration (Wi-Fi ↔ cellular without reconnect) | UDP keepalives. No relay, so it fails on UDP-hostile networks unless wrapped |
| Firewall-friendliness | QUIC on 443, close to HTTPS | UDP on a custom port, often blocked on corporate and hotel networks |
| Identity | iroh endpoint key, already the Connector's identity | Separate WireGuard key, which needs distribution and rotation |
| Client cost on iOS | Needs a CONNECT-IP client inside the Network Extension. Rust (iroh plus an H3 datagram layer) through UniFFI. **No production CONNECT-IP client library is known to exist**, so we would build one. | `wireguard-apple` / boringtun are mature, small and proven inside Network Extensions |
| Gateway cost | New CONNECT-IP termination: quic-go- or Envoy-based (open question in the MASQUE doc) | Kernel or userspace WireGuard, multi-peer, keys from attachments. Well-trodden |
| Fit with existing connectors | Same transport and identity as `datum-connect` today, so one client core | A second transport beside iroh |
| Time to first packet | Longer | Shorter |

<<[UNRESOLVED tunnel protocol]>>
**Recommendation: A**, with a time-boxed spike before committing. It keeps one
transport and one identity across every connector capability, inherits iroh's
NAT traversal and relays (which matter more on phones than anywhere else), and
is the direction Galactic has already written down. The risk is the client:
there is no off-the-shelf CONNECT-IP client for an iOS Network Extension. The
spike should measure a minimal Rust CONNECT-IP-over-iroh client in a Network
Extension for memory, battery and reconnect time. If it can't stay well under
the ceiling, **B is the fallback**, wrapped over iroh's relay when UDP is
blocked, behind the same API.
<<[/UNRESOLVED]>>

**Bridging into the fabric** is the same in both options. The gateway
de-tunnels a device packet, looks up the session's attachment, which gives
the network and therefore the VRF and uSID, and SRv6-encapsulates toward the
destination exactly as `galactic-gateway` does for ingress. It starts as a
kernel VRF and seg6 bridge (simple and debuggable) and moves to eBPF/XDP once
correctness is settled, mirroring the MASQUE doc's phasing.

### Session lifecycle

1. **Register:** the client ensures its `Connector` (existing `datum-connect`
   flow) with the `NetworkAccess` capability.
2. **Attach:** the user or an admin creates a `ConnectorAttachment`. The
   controller authorizes it (`Accepted`), allocates the address, ensures the
   `NetworkBinding`(s), and marks it `Ready`.
3. **Connect:** the client dials the nearest gateway by anycast over QUIC,
   presents its token, and receives its address, routes and MTU. The gateway
   authorizes, programs the bridge, originates the return route and sets
   `Connected`.
4. **Heartbeat:** the existing Connector lease keeps the device "present".
   The session itself is kept alive by the transport. iOS may suspend the app,
   but a Network Extension keeps running while the tunnel is up.
5. **Roam:** a path change migrates the QUIC connection (option A) or
   re-handshakes (option B). If the client lands on a different gateway, the
   return route moves with it.
6. **Revoke or expire:** deleting the attachment, losing access, or a lapsed
   lease closes the session and withdraws the route. The address is released
   after a grace period, so a reconnecting device keeps it.

#### Where a device lands

Gateways are anycast like the rest of the edge, so a device lands at its
nearest site. The binding for that (network, location) pair is created on
first connect if the network is not already present there, so a network does
not need to run workloads in a location for a device to enter through it. The
fabric carries traffic to wherever the workloads are.
`ConnectorAttachment.spec.location` pins a location for consumers with data
residency needs.

### Clients

All clients share one Rust core in `datum-connect`. It holds the iroh
endpoint, the tunnel, the attach and heartbeat logic and the token refresh. It
is **split out of today's `lib`**, which carries kube, axum, sentry and
iroh-blobs and is far too heavy for a Network Extension.

- **iOS (`datum-mobile`):**
  - an `NEPacketTunnelProvider` app extension (added through an Expo config
    plugin), with the core linked as an xcframework through **UniFFI**
  - the app creates the attachment and starts and stops the tunnel with
    `NETunnelProviderManager`
  - the extension reads the user's refresh token and the device's iroh key
    from the shared Keychain access group and App Group, both already
    reserved in the app
  - "Connect on demand" is supported through `NEOnDemandRule` for a chosen
    network
  - requires the Network Extension (packet tunnel) entitlement on Datum's
    Apple Developer team
- **Desktop (`datum-connect`):** an `attach --network <name>` command and a UI
  toggle, using the platform's TUN (utun on macOS through a system extension,
  Wintun on Windows, `/dev/net/tun` on Linux).
- **Headless (CI and agents):** the same CLI with a machine account and a
  short-lived attachment (Story 3).

### Prerequisite: peer authorization in datum-connect

`datum-connect`'s listen-side authorization (`lib/src/node.rs`,
`impl AuthHandler for StateWrapper`, as of `main` at `1487415`) ignores the
remote peer's identity (`_remote_id`). It only checks that the requested
host:port is one the device advertises. Any iroh peer that learns a device's
endpoint ID can therefore reach that device's advertised services directly,
without passing through Datum's `iroh-gateway` and its WAF, rate limits or
logging.

This is a live issue for today's edge → device direction and should be fixed
independently of this proposal. The fix is to authorize only Datum's gateway
endpoint IDs, published through the existing DNS discovery. It matters
doubly here: this design's security rests on the gateway authorizing peer
identity, and the client core it builds on must not carry the opposite
assumption. **This enhancement treats that fix as a blocking prerequisite.**

### Phasing

| Phase | Platform | Clients |
|---|---|---|
| **0: Decide** | Tunnel spike for A vs B on iOS (memory, battery, reconnect). Fix datum-connect peer authorization. | — |
| **1: Alpha, one location** | `ConnectorAttachment` CRD, controller, `networks.attach` permission. Gateway with a kernel bridge in one edge site. BGP return path. Feature-gated per project. | `datum-connect attach` (desktop CLI). iOS packet tunnel behind a staging flag. |
| **2: Beta** | All edge sites, anycast, roaming between gateways. Flow logs and metrics per attachment. Portal device list and disconnect. | iOS on TestFlight; desktop UI toggle. |
| **3: GA** | eBPF/XDP bridge, per-attachment limits, security review sign-off, SLOs. | On-demand rules, CI mode documented. |

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

- **Enable or disable:**
  - Enabled per project with a feature flag on the attachment controller.
  - Gateways are deployed but serve no sessions for projects without the
    flag.
  - Disabling the flag sets every attachment's `Accepted` to `False`, which
    closes sessions and withdraws routes.
- **Default behavior:** unchanged. No existing object changes meaning;
  `Connector` gains an opt-in capability.
- **Rollback:** delete attachments or disable the flag. Device routes are
  withdrawn and no workload state is touched.

### Rollout, Upgrade and Rollback Planning

- **Gateway rollouts:** they drop sessions on the node being replaced.
  Clients reconnect to the next-nearest site. Roll one site at a time and
  watch session re-establishment.
- **Rollback signals:** a spike in session failures, VRF injection errors, or
  any cross-VRF emission counter above zero.

### Monitoring Requirements

- **Per gateway:**
  - active sessions
  - session setups and failures by reason (authn, authz, address, bridge)
  - bytes and packets per direction
  - per-attachment rate-limit drops
  - a **cross-VRF emission counter that must always be 0**
- **Per attachment:** the `Connected` condition and `session.lastSeen` in
  status. Flow logs go through the existing telemetry export path.
- **SLO candidates:**
  - session setup p95 under 1.5 s on a direct path and under 3 s through a
    relay
  - revocation-to-teardown p99 under 10 s

### Dependencies

- Galactic (`galactic-router` `BGPAdvertisement`, the SRv6/VRF fabric)
- NSO (`NetworkBinding` and `NetworkContext` from network-in-every-location)
- milo IAM (SubjectAccessReview)
- IPAM (milo-os/ipam)
- Zitadel (token validation)
- iroh relays reachable over IPv6

### Scalability

- **New API types:** `ConnectorAttachment`.
- **New API calls:**
  - one attachment create per device per network
  - a status patch per session change
  - SubjectAccessReviews on connect and about once a minute per session
- **Gateway state:** one session and one advertised route per connected
  device. Plan capacity per site against expected concurrent devices; the
  route count per VRF grows with devices, so check `galactic-router` limits.

### Troubleshooting

- **Attachment conditions** explain where a device is stuck:
  - `Accepted=False` with an authz reason
  - `Ready=False` with an address or binding reason
  - `Connected=False` with the last session error
- **Gateway inspection:** the gateway exposes a per-session inspection
  endpoint, the analogue of `fabric-api` (FabricQuery).

## Implementation History

- 2026-10-12: Provisional draft. Builds on the Connectors initial proposal and
  Galactic's MASQUE gateway proposal; motivated by the Datum iOS app.

## Drawbacks

- **A new stateful edge tier**, with its own on-call surface and capacity
  planning.
- **It turns account compromise into network compromise** for anyone granted
  `networks.attach`. The authorization design limits this but cannot remove
  it.
- **Option A means building a CONNECT-IP client** where none exists
  off-the-shelf.

## Alternatives

- **Keep exposing services publicly** (`HTTPProxy` plus WAF plus auth at the
  app). That is today's answer. It is HTTP-only and defeats the point of a
  private network.
- **A VPC-side connector as a subnet router** (Tailscale-style): run
  `datum-connect` as a workload inside the VPC, advertising the VPC's prefixes
  through a `layer3` `ConnectorAdvertisement`, with devices dialing it
  peer-to-peer.
  - Pros: no new edge tier, and it works with iroh as it is.
  - Cons: the consumer operates it, there is one per network, it bottlenecks
    and is a single point of failure, authorization has to be rebuilt inside
    that workload, and it needs source NAT, so the workload sees the router's
    address, not the device's.
  - Use: a reasonable stopgap or demo, not the product.
- **IPsec/IKEv2:** native iOS support without an extension, but it is poor at
  NAT and roaming and uses a separate identity system. Not pursued.
- **Per-application proxies** (MASQUE CONNECT-UDP/TCP, or iOS per-app VPN):
  lighter than full IP, but it doesn't cover arbitrary protocols (SSH, raw
  TCP/UDP, ICMP for debugging). It could be a later, narrower mode on the same
  gateway.

## Infrastructure Needed

- VPC access gateway nodes per edge site, likely the existing gateway-role
  edge nodes beside `galactic-gateway` and `galactic-nat`.
- iroh relays reachable over IPv6, with an IPv6 relay probe.
- The Apple Developer Network Extension entitlement for `net.datum.cloud` and
  `net.datum.cloud.staging`.
