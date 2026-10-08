---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Looking glass system architecture

## Summary

Looking glass lets a project member run a live diagnostic from a named Datum
edge location. The project control plane authorizes and records the session;
the selected cell runs it and streams results to the client. The proposed
`milo-os/looking-glass` service owns this customer-facing path. Galactic
supplies fabric router observations through its existing gRPC service. The
[product proposal](../../enhancements/networking/looking-glass/README.md)
defines the experience and initial scope.

## Motivation

The project control plane knows who a customer is and what they may access.
The edge cell has the network vantage point and router state. The service
needs both without granting customers node access or routing every result
through the control plane.

### Goals

- Authorize and audit each diagnostic in its project.
- Stream results from the chosen cell to the portal and CLI.
- Reuse Galactic's bounded fabric diagnostics and permit other vantage points
  behind the same user workflow later.

### Non-goals

- An arbitrary shell on Datum infrastructure.
- Treating a fabric router probe as a test from a customer VPC or workload.
- Persisting complete session output in the project API.

## Proposal

The customer creates one short-lived session for one location and diagnostic.
The project control plane admits it and binds it to a cell. The client then
connects through Datum Connect with a key generated for that session. The cell
starts the diagnostic only after the client authenticates. This follows the
[Compute instance shell pattern](https://github.com/datum-cloud/compute/blob/main/docs/enhancements/instance-shell-sessions/README.md):
project authority, a direct client-to-cell stream, and an agent that reads
only its cell's session copy.

### Container view

![C4 container diagram of the client, project API, looking glass controller, Karmada, cell gateway, Datum Connect endpoint, and Galactic fabric diagnostics](./looking-glass-containers.png)

[PlantUML source](./looking-glass-containers.puml)

| Container | Deployed in | Responsibility |
| --- | --- | --- |
| Portal / `datumctl` | Customer browser or device | Creates sessions and displays results. |
| Project API and looking glass controller | Project control plane | Authorizes sessions, binds a cell, reflects status, and records activity. |
| Karmada hub | Federation control plane | Delivers the cell copy and returns cell status. |
| Datum Connect endpoint and looking glass gateway | Selected edge cell | Accepts the client stream, authenticates the session, and runs diagnostics. |
| Galactic fabric gateway and router agents | Selected edge cell, with agents beside fabric routers | Fans out queries and collects routing observations or probes. |

The cell gateway needs no project or hub credentials, and the Datum Connect
endpoint has no Kubernetes credentials. Galactic's internal calls use mTLS
gRPC.

### Risks and mitigations

- Active probes could scan or load other networks. Admission and the cell
  both enforce destination, traffic, time, concurrency, and output limits.
- Route results can reveal platform topology. Customer access to BGP details
  needs an explicit visibility policy before launch.
- Endpoint reachability is not authorization. The gateway checks the session
  key, bound cell, current state, and single-use claim before execution.

## Design details

### Consumer-facing API

The proposed `LookingGlassSession` is a project-scoped resource. This example
is illustrative; the API version and field names remain provisional:

```yaml
apiVersion: network.datumapis.com/v1alpha1
kind: LookingGlassSession
metadata:
  generateName: edge-trace-
spec:
  location: us-central-1
  vantagePoint: FabricEdge
  diagnostic:
    type: Traceroute
    target: 1.1.1.1
  clientPublicKey: "<ephemeral-public-key>"
```

Creating a session requires a dedicated project permission. The spec is
immutable, and deletion revokes the session. Portal and CLI clients use the
same API to create it and watch status, and keep the private key in memory.
Status reports pending, connectable, connected, or terminal state. Once
connectable, it carries the endpoint ID, relay URLs, connection target, and
connection deadline. Terminal status carries an ending reason and coverage
counts, rather than raw probe output. The project activity log records the
requester, location, diagnostic, start, and outcome.

One session targets one location; clients may create several to compare
locations. Admission checks project access to the chosen location and vantage
point. The selected cell is fixed before connection details are exposed, and
status from another cell cannot redirect the client.

### Execution and result stream

The cell gateway checks the live session when the client connects, consumes
its single use, then calls a typed diagnostic backend. For `FabricEdge`, it
calls Galactic's cell fabric gateway, which fans out to router agents. The
looking glass stream identifies each router and sample time, reports each
completed observation, and ends with coverage and typed errors. Galactic's
current gRPC calls return completed operations; live per-hop traceroute
output would require a streaming backend contract.

Galactic's debug gRPC endpoint uses operator-oriented credentials today. The
looking glass service needs a dedicated service identity and scoped policy at
that gateway, rather than inheriting broad operator access. A future VPC
vantage point needs a worker in that VPC's network context behind the same
session API.

### Lifecycle and failure handling

The cell advertises a session only when its endpoint is ready. No probe
starts until the client connects. An unclaimed session ends after a deadline;
disconnect, expiry, and revocation cancel in-flight work. Cell status returns
through Karmada, and the project API records the outcome and audit event. A
detached diagnostic that runs without a client would be a separate mode with
stored results.

## Alternatives

An asynchronous query resource suits saved results but does not offer a live
session. An HTTPS proxy through the control plane could stream results without
a native Iroh client, but would put every diagnostic response on that path.
The portal and CLI should share the session contract even if their transport
implementations differ.

Galactic's [fabric API](https://github.com/datum-cloud/galactic/pull/792) and
Compute's [cell session implementation](https://github.com/datum-cloud/compute/pull/390)
are the backend and transport precedents.
