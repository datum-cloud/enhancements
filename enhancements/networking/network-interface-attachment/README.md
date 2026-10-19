---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Attaching instance network interfaces to tenant VPCs

## Summary

A compute Instance reaches a tenant VPC (virtual private cloud) through four
layers. The user picks a capability class. The location binds that class to a
handler it offers. The runtime class states the consequences, including how a
guest consumes a network interface card (NIC). The data plane realizes it.

A VPC controller in the cell owns the bridge between Datum networking APIs and
the data plane. A cell is the Kubernetes cluster in a point of presence (POP)
that runs tenant workloads. The controller creates the `VPCAttachment` and the
`NetworkAttachmentDefinition` (NAD) when a network interface claim is fulfilled,
then injects the Pod annotation that delivers that NAD to Multus.

An infrastructure provider stamps one opt-in label on the Pod. It never learns
what a VPC, a NAD, or a tap device is. Galactic's container network interface
(CNI) chain is unchanged.

## Motivation

Five repositories each hold one piece of instance-to-VPC attachment.

| Repository | Piece it holds |
| --- | --- |
| `compute` | Instance intent, the `Network` scheduling gate |
| `network-services-operator` (NSO) | `NetworkInterfaceClaim`, `NetworkInterface`, addresses |
| `cloud` | The `VPC` and `VPCAttachment` types, with no controller |
| `galactic` | The CNI data plane, which assumes the NAD already exists |
| `unikraft-provider` | The Pod that becomes a microVM |

Nothing authors the NAD. Galactic removed its VPC CRDs, operator, and Pod
webhook, and now names an external companion operator as the owner. That
operator does not exist, so an instance cannot join a tenant VPC.

### Goals

- Name one owner for NAD create, update, and delete, with a clear contract to
  the other four components.
- Preserve the ordering guarantee: the NAD exists and is complete before the
  Pod is created.
- Keep compute free of networking internals. Keep galactic free of Datum APIs.
- Support containers (`galactic-veth`) and microVMs (`galactic-tap`) alike.

### Non-Goals

- Choosing addresses. NSO decides those and this design consumes the answer.
- Programming the data plane. Galactic's CNI chain and `galactic-router` do that.
- Retiring `NetworkBinding`, `NetworkContext`, `SubnetClaim`, or `Subnet`.

## Proposal

### The four layers

Each layer decides one thing and reads only the layer above it.

| Layer | Decides | Example |
| --- | --- | --- |
| User | The capability the workload needs | `isolated` |
| Location | Which handler offers that capability here | The Unikraft runtime |
| Runtime class | The consequences of that handler, including how the guest consumes a NIC | `Hypervisor` |
| Data plane | How to realize the attachment | `galactic-tap` |

A user never names an implementation. `isolated` binds to a different handler in
each location, the way a `StorageClass` name binds to a different provisioner in
each cloud. That indirection lets one workload span locations.

The capability class supplies the handler, so compute sets the consumption mode
without knowing the runtime's implementation. The runtime discriminator says
what spec the user writes: containers, or a boot image and volumes. The class
says how it executes. Unikraft proves the axes are independent: a sandbox by
contract, a microVM by implementation.

### Ownership

| Component | Owns |
| --- | --- |
| compute | Instance intent, the resolved capability class, `Instance.status.networkInterfaces[]` |
| NSO | Claims, interfaces, addresses. Carries `attachmentMode` without interpreting it |
| VPC controller (cell) | Base62 identifiers, `VPCAttachment`, the NAD, the Pod webhook, `Prepared` |
| galactic | Kernel state, the NAD `host-interface` annotation, BGP and SRv6 |
| Infrastructure provider | One opt-in label on the Pod, and the scheduling gate it already honors |

The NAD is an implementation detail of the galactic data plane, so it belongs to
the controller that owns the galactic control-plane objects, not the one that
owns the Pod. Multus knowledge lives in exactly one repository.

### Scope of the first implementation

One path: the Unikraft runtime provider, `Hypervisor` mode, `galactic-tap`.

- No `AttachmentClass` object. A cell hosts one networking implementation today,
  so an object that chooses between them has nothing to choose. Add it when a
  second one exists. Nothing here forecloses that.
- The consumption mode is a required setting on the VPC controller with no
  default. A cell states what it is rather than falling back to veth and handing
  a microVM an interface it cannot use.
- The capability class is a compute product API tracked separately. The
  controller's setting stands in for it until then.

## Design Details

### API contract

- `NetworkInterfaceClaim.spec.attachmentMode`: `Netns` or `Hypervisor`,
  defaulting to `Netns`. Compute sets it from the resolved capability class. NSO
  copies it to `NetworkInterface.spec.attachmentMode` and never interprets it.
  The field describes how a guest consumes a NIC. It names no CNI, no Multus,
  and no Linux device type.
- `Instance.status.networkInterfaces[].networkInterfaceRef`: the bound
  `NetworkInterface`, so nothing re-derives compute's claim-name convention.
- `VPCAttachment.spec` gains `interfaceRef {name, uid}` and
  `interface.type: veth | tap`, and allows zero addresses for a guest that
  manages its own addressing.
- `VPCAttachmentStatus` makes every field except `conditions` optional. Eight
  are required today, so the controller cannot record an identifier before a
  Pod attaches.

### Conditions

Gate workload creation on `Prepared`, not on `Programmed`.

| Condition | Meaning | Owner |
| --- | --- | --- |
| `Prepared` | The data plane's pre-Pod artifacts exist: identifiers allocated, `VPCAttachment` created, NAD written | VPC controller |
| `Programmed` | The attachment is live in the kernel | galactic, projected by the VPC controller |

`Programmed` becomes true at CNI ADD, which happens at sandbox creation, so any
component that blocks Pod creation on `Programmed` deadlocks. NSO seeds both
conditions and leaves them alone. Compute's gate stays on Bound plus Allocated.

### Pod mutating webhook

The VPC controller serves a mutating admission webhook on Pods. It matches Pods
carrying an opt-in label, resolves them to their Instance and its interfaces,
and injects whatever the delivery mechanism requires. Today that is the
`k8s.v1.cni.cncf.io/networks` annotation.

The opt-in must be a label, not an annotation. A webhook `objectSelector` is a
`LabelSelector` and cannot match annotations. Get that wrong and the selector
never matches, the webhook never fires, and Pods come up with no interface while
every object looks healthy.

The opt-in earns its place even though the webhook could key off the Pod's owner
reference. A narrow `objectSelector` makes `failurePolicy: Fail` safe: an outage
blocks exactly the Pods that need an interface, loudly, rather than every Pod in
the cell.

### Object lifecycle and ownership

The VPC controller creates the `VPCAttachment` and the NAD when the claim is
fulfilled, before any Pod exists. The `NetworkInterface` owns the attachment and
the attachment owns the NAD, so deletion cascades without a finalizer.

The attachment is per-interface, not per-Pod. A `NetworkInterface` is
slot-stable, so the attachment identifier survives instance replacement: a
replacement returns to the same tap name and reuses its `BGPAdvertisement`.

One controller creates the attachment, the NAD, and the annotation, so no lookup
crosses a component boundary and no two repositories disagree about naming.

### Constraints that bind any implementation

1. **The kernel limits an interface name to 15 characters.** Galactic spends 9
   on a 48-bit VPC identifier and 3 on the attachment identifier, which is why
   both are base62. A UUID does not fit.
2. **Base62 is not a valid Kubernetes object name.** The digit set includes
   uppercase; `metadata.name` must be a lowercase RFC 1123 subdomain. Galactic
   interpolates base62 straight into `BGPAdvertisement` and `BGPVRFInstance`
   names. Roughly 99% of random 48-bit VPC identifiers contain uppercase, so
   this breaks on the first generated VPC, independent of this design.
3. **One NAD serves one attachment.** Kraftlet, the Unikraft node agent, reads
   the tap device name from the NAD's `k8s.v1.cni.cncf.io/host-interface`
   annotation, and galactic writes one such annotation per NAD.
4. **The NAD must exist and be complete before the sandbox.** Multus resolves
   the annotation at sandbox creation, and `VerifyChainComplete` turns a missing
   or incomplete NAD into a failed ADD.
5. **A NAD is a raw CNI config, so it is a privilege boundary.** Anyone who can
   create one can name any VPC and attach to another tenant's VRF. NAD write
   access in cell workload namespaces stays restricted to the controller's
   service account.
6. **Static IPAM must carry the address NSO decided.** `galactic-ipam` takes one
   IPv6 address, forces a `/64` mask, and allocates no IPv4. Without a
   prefix-preserving dual-stack path, NSO's allocation is decorative.

### Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| The webhook is an availability dependency on Pod creation | A narrow `objectSelector` bounds the blast radius to Pods that need an interface |
| The applied Pod is not the created Pod | The webhook records what it injected |
| Two live Pods on one interface collide on the tap name | Compute replaces by delete-then-recreate and the provider holds a finalizer until the Pod is gone. Cover with an end-to-end test |

## Implementation History

Nothing is implemented. No part of this design has run on hardware, and no
envtest exercises the admission path. The Unikraft provider's share is one label
and roughly 26 lines of code. The largest open item is naming the owner of the
VPC controller binary: `cloud`, or NSO's cell manager.

## Drawbacks

The design adds a mutating webhook to the Pod creation path in every cell that
runs galactic, plus a new controller in a repository that has never shipped a
binary.

## Alternatives

- **Galactic owns the NAD again.** Galactic deleted its CRDs, operator, and
  controller-runtime dependency on purpose; its code now assumes an external
  author.
- **NSO owns the NAD.** Programming the data plane is an explicit non-goal of
  NSO's own enhancement, and an NSO-written NAD makes every future data plane an
  NSO release.
- **Compute owns the NAD.** Against the goal of keeping compute out of
  networking internals; compute does not know the runtime is a microVM.
- **The infrastructure provider owns the NAD.** The shortest path to a lab demo,
  but it hard-codes galactic's conflist into a per-vendor provider and leaves
  identifier allocation unowned.
- **An opaque annotation map on `NetworkInterface.status`.** Puts an instruction
  in a status field, assumes the consumer is a Pod, and leaks Multus in plain
  sight while claiming to hide it.
- **A typed `consumerRef` to the NAD.** Makes the provider map a kind to a
  mechanism, compressing Multus knowledge into a switch statement rather than
  removing it. With a webhook, the field has no external reader.
- **One NAD per VPC instead of one per attachment.** Cuts object count by orders
  of magnitude and removes the ordering constraint. It requires kraftlet to read
  the tap name from the CNI result instead of the NAD annotation, a vendor change
  Datum does not control, plus identifier allocation inside the CNI and Multus
  `ips` runtime-capability support in galactic. Revisit if the vendor change
  lands.

## Open Questions

- Does a microVM attach as a secondary network with Cilium still the default,
  keeping its Pod IP and Service, or does it need
  `v1.multus-cni.io/default-network`? Untested.
- Which scope allocates the VPC identifier: cell, metro, or global? A network
  spanning locations needs one stable value, which argues for the `Network`
  rather than the per-location `NetworkContext`.
- Is `NetworkContext` the right trigger for `VPC` creation?
- Which component writes `VPCAttachment.status`? NSO's design names a node
  agent. `galactic-router` is the only candidate and does not know these types.
- Should the controller reject mutations to a live NAD?
