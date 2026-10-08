# Looking glass system architecture

## Overview

Looking glass owns the customer-facing diagnostic session. The project control
plane authorizes a request and reports its lifecycle; a cell service runs it
and sends results directly to the client. Galactic remains the provider of
fabric router diagnostics through its existing gRPC service. The proposed
service can live in `milo-os/looking-glass`, with its project controller and
cell gateway in one repository. The [product proposal](../../enhancements/networking/looking-glass/README.md)
defines the user experience and scope.

```text
Client -- create session / watch status --> Project control plane
                                               |
                                               | session copy via hub / Karmada
                                               v
Client -- Iroh --> Datum Connect endpoint --> Cell session gateway
                                                | gRPC
                                                v
                                       Galactic fabric diagnostics

Cell status -- Karmada / hub --> Project control plane
Cell session gateway -- result stream --> Client
```

## Session path

1. The client generates a temporary key pair and creates a project-scoped,
   immutable session naming one location, vantage point, diagnostic, target,
   and its public key. Project admission checks permissions and policy.
2. The control plane binds the session to an eligible cell and propagates it
   there. Only that cell may publish a connection endpoint or terminal status.
   The cell gateway claims the session when its endpoint is ready.
3. The client reads the endpoint and connects through Datum Connect. It proves
   possession of the private key and consumes the single-use session. The
   gateway checks the current session state again before starting work.
4. After connection, the gateway invokes the diagnostic backend. For the
   fabric vantage point, it fans out to Galactic's gRPC service and forwards
   each completed router observation to the client. It sends a final status
   with coverage and typed errors.
5. The cell writes lifecycle status to its local session copy. Federation
   reflects it to the project control plane for the client and activity log.
   Revocation, expiry, disconnect, and cell loss end the session.

The project control plane is the authority for *who may run what*. The Iroh
endpoint identifies the cell and carries bytes; its address is not an access
grant. The cell holds no project or federation-hub credentials. The client
private key stays in client memory, as it does for [Compute instance shells](https://github.com/datum-cloud/compute/blob/main/docs/enhancements/instance-shell-sessions/README.md).

## Execution boundary

The cell gateway exposes typed operations, not a shell. For fabric queries it
uses Galactic's existing gRPC contract, validation, budgets, and router-local
readers. The first stream can report session progress and each router's final
observation. Live ping replies or traceroute hops would require a streaming
backend contract; Iroh alone does not make a completed gRPC call incremental.

A future VPC vantage point needs a separate worker in that VPC's network
context. It must not be represented as a fabric probe. If later diagnostics
need user-supplied programs, their execution belongs in an isolated runtime
with its own resource and network policy, behind the same session gateway.

## Limits and failure handling

- Admission and the cell both validate the operation, target, vantage point,
  and project scope. The cell enforces destination policy and hard execution,
  traffic, concurrency, and output limits even if upstream validation fails.
- Sessions have a short connection window. No probe starts until a client
  authenticates. Disconnect cancels in-flight work; timeout and revocation
  close the stream. A detached, asynchronous diagnostic would be a separate
  mode with stored results.
- The gateway reports which routers were selected, answered, failed, or were
  omitted. A missing cell endpoint ends clearly rather than waiting without
  a deadline. Audit records outlive the short-lived session resource.
- Datum Connect endpoint pods have no cluster credentials and may reach only
  their paired gateway. Endpoint readiness includes a client-path check so a
  gateway does not advertise a session that users cannot reach.

This follows Compute's [project-to-cell session pattern](https://github.com/datum-cloud/compute/pull/387)
and [Datum Connect endpoint deployment](https://github.com/datum-cloud/compute/blob/main/config/components/shell-agent/endpoint.yaml).
Galactic's [fabric API](https://github.com/datum-cloud/galactic/pull/792) remains
the first diagnostic backend, not the customer session authority.
