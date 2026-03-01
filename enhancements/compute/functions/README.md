---
status: provisional
stage: alpha
latest-milestone: "v0.1"
---

<!-- omit from toc -->
# Functions - Serverless Edge Compute

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Building on Workload](#building-on-workload)
  - [Unikraft Runtime](#unikraft-runtime)
  - [Control Plane Architecture](#control-plane-architecture)
  - [Private Connectivity via Service Connect](#private-connectivity-via-service-connect)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Dependencies](#dependencies)
- [Alternatives](#alternatives)
  - [Extend Workload Resource](#extend-workload-resource)
  - [Direct Runtime Integration](#direct-runtime-integration)

## Summary

Functions is a serverless compute capability that enables developers to run
scripts at the edge for common use cases like request interception, simple APIs,
and webhooks. It provides a simplified interface with automatic scaling
(including scale-to-zero) and sub-50ms cold starts via Unikraft unikernels.

## Motivation

Many developers have simple use cases that don't require the full control that
Workload provides. Common scenarios include:

- **Request interception**: Modify headers, rewrite URLs, add authentication
- **API endpoints**: Simple CRUD operations, webhooks, form handlers
- **Server-side rendering**: Generate HTML at the edge for low-latency responses

These use cases benefit from a streamlined interface where the platform handles
infrastructure decisions automatically.

### Goals

- Enable developers to deploy functions with minimal configuration
- Support true scale-to-zero with automatic scaling on traffic
- Provide multi-language support (Go, Node.js, Python, Rust)
- Integrate with Gateway API for HTTP routing

### Non-Goals

- Replacing Workload for advanced infrastructure use cases
- Supporting stateful workloads
- GPU workloads or specialized hardware
- Region selection in MVP (automatic placement only)

## Proposal

Functions introduces a `Function` resource that abstracts infrastructure
complexity. Developers specify their container image and port; the platform
handles everything else.

<p align="center">
  <img src="./architecture-context.png" alt="System Context" />
</p>

### Building on Workload

Rather than building compute capabilities from scratch, Functions generates
[Workload][workload-enhancement] resources with serverless-optimized defaults.
Workload is the platform's foundational compute primitive, providing instance
lifecycle management, placement, scaling, and runtime configuration.

This approach:

- **Reduces duplication**: Functions inherits Workload's battle-tested
  infrastructure
- **Ensures consistency**: All compute services use the same underlying
  primitives
- **Simplifies operations**: One set of compute infrastructure to monitor and
  maintain

As Workload gains new capabilities, Functions automatically benefits. For
example, when Workload adds direct VPC connectivity, Functions will inherit that
capability without requiring changes to the Functions service itself.

Workload also enables deployment across multiple infrastructure providers. The
long-term vision is for the platform's networking layer to stitch together
connectivity across providers, allowing Functions to run closer to users
regardless of where the underlying compute is provisioned. By building on
Workload, Functions will inherit this multi-provider capability as it matures.

Functions deploys these Workloads in the Functions service control plane, not in
the consumer's project. This design choice enables:

- **Simplified user experience**: Consumers interact only with their Function
  resource; they never see or manage the underlying Workloads
- **Platform-managed infrastructure**: The platform controls placement, scaling,
  and runtime configuration without consumer involvement
- **Operational efficiency**: The Functions team can upgrade, patch, and
  optimize infrastructure across all consumers without requiring consumer action
- **Resource optimization**: The platform can make scheduling decisions across
  all consumers rather than within isolated consumer projects

Functions establishes the pattern for managed services on Datum Cloud. Future
services (Databases, Caches, ML Inference) will follow the same
approach—building on Workload rather than implementing compute from scratch.

### Unikraft Runtime

Functions achieves sub-50ms cold starts through Unikraft unikernels integrated
at the Workload layer. Unikraft provides:

- **Minimal images**: 1-10MB unikernels vs 100MB+ containers
- **Fast boot**: 10-50ms cold starts via snapshot/restore
- **VM isolation**: MicroVMs provide hardware-level security and full isolation
- **Multi-language**: Go, Node.js, Python, Rust support

The Workload controller handles Unikraft configuration transparently. Function
users provide standard container images; the platform optimizes them for the
Unikraft runtime automatically.

### Control Plane Architecture

The Function controller runs in the Functions service control plane and directly
accesses consumer project APIs:

<p align="center">
  <img src="./architecture-control-plane.png" alt="Control Plane Architecture" />
</p>

1. **Watches** Function resources across consumer projects
2. **Creates** corresponding Workloads in per-consumer namespaces
3. **Updates** Function status directly in consumer projects

This direct access pattern (rather than federation-based sync) provides:

- **Simpler architecture**: Single component handles watch and status updates
- **Lower latency**: No intermediate sync layer
- **Full context**: Controller has complete information for reconciliation
- **Easier debugging**: Single point of control

Authentication uses service identity with PolicyBindings created automatically
when consumers enable the Functions service.

### Private Connectivity via Service Connect

Because Workloads run in the service control plane, [Service
Connect][service-connect-enhancement] provides private connectivity between the
consumer's applications and their Functions:

<p align="center">
  <img src="./architecture-service-connect.png" alt="Service Connect" />
</p>

When a consumer creates a Function:

1. The Function controller creates a Workload in the service control plane
2. A ServicePublication exposes the Workload for private access
3. A ServiceEndpoint appears in the consumer's project with a private IP
4. The consumer's applications connect via the private IP or DNS name

This model provides:

- **Private networking**: Traffic never traverses the public internet
- **No VPC peering**: Consumers don't need to peer networks with the platform
- **Overlapping IP support**: NAT handles address conflicts between consumers
- **Consistent pattern**: All managed services use the same connectivity model

### User Stories

#### Deploy a Simple API

As a developer, I want to deploy an API endpoint without configuring
infrastructure. I create a Function with my container image, and within seconds
it's deployed and accessible. No network configuration, instance sizing, or
placement decisions required.

#### Scale to Zero When Idle

As a developer, I want my function to scale to zero when not receiving traffic
to minimize costs. The function automatically scales down after an idle period
and scales back up when traffic arrives.

#### Route Traffic via Gateway

As a developer, I want to expose my function via an HTTPRoute so it's accessible
through my project's Gateway.

### Notes/Constraints/Caveats

- **Automatic Placement**: MVP supports only automatic placement. Region
  selection is planned for a future phase.
- **Stateless Only**: Functions are designed for stateless workloads. Once
  Workload gains VPC connectivity, Functions will be able to integrate with
  external storage services (databases, caches, object storage).
- **HTTP-Only Triggers**: MVP supports HTTP triggers only. Cron and event
  triggers are planned for future phases.

### Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Cold start latency | Unikraft snapshot/restore targets sub-50ms P95 |
| Scale-from-zero delays | Queue requests during cold start; target <100ms scale decision |
| Unikraft build failures | Provide tested base images; clear error messages for unsupported configurations |

## Dependencies

- **Compute (Workload)**: Functions generates Workload resources. The Workload
  controller handles instance lifecycle, placement, and runtime configuration
  including Unikraft integration.

- **Unikraft Runtime**: Workload integrates Unikraft for unikernel builds,
  snapshot/restore, and Firecracker MicroVM execution. This is transparent to
  Function users.

- **Networking (Gateway)**: Gateway routes HTTP traffic to Function backends via
  HTTPRoute resources.

- **Service Connect**: Provides private connectivity between consumer
  applications and Functions running in the service control plane.

## Alternatives

### Extend Workload Resource

Add "serverless mode" to existing Workload.

**Rejected because:** Workload is designed for advanced use cases. Adding
serverless semantics would complicate Workload's mental model.

### Direct Runtime Integration

Have Function controller manage instances directly without Workload.

**Rejected because:** Duplicates infrastructure already built in Workload. Loses
benefits of Workload's placement, scaling, and runtime management.
