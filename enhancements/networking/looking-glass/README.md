---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Looking glass

## Summary

Looking glass lets a customer ask what Datum's network sees from a chosen edge
location. From the Cloud Portal, CLI, or API, they can run a bounded diagnostic
and watch results from the edge without obtaining access to infrastructure.
The first release covers fabric routing observations and ping or traceroute from
the fabric edge. Other vantage points can follow under the same experience.

## Problem

A customer investigating reachability can see that an application is failing,
but cannot tell whether a route reached a Datum edge, which routers see it, or
where a probe from that edge stops. Support has to collect this evidence for
them. Local ping and traceroute describe the customer's own network, which can
give a different answer.

## Experience

The user selects a project, edge location, diagnostic, and target. Looking
glass identifies the vantage point before the user runs the request. For a
fabric diagnostic, the result names each responding router and its sample time;
it never presents several routers' observations as one authoritative route.
Results appear as routers finish, with clear coverage when some routers are
unavailable or time out. The user can copy or export the response.

The initial diagnostics are route lookup, BGP summary, ping, and traceroute.
The product offers typed inputs and results rather than an arbitrary command
line. A user can repeat a request from another location to compare paths. A
single session addresses one location, so each result has an unambiguous
source.

The project records who started a session, its location and diagnostic, and
how it ended. Closing the client or revoking the session stops new work.
Session output is available to the connected client; saving a full diagnostic
history is a separate product decision.

## Boundaries

- A fabric edge probe describes reachability from a fabric router. It does not
  claim to test traffic from a customer's VPC or workload. Those require a
  distinct vantage point with the right network context.
- Project permissions govern who may start diagnostics. Route visibility may
  need narrower permissions than active probes; customer exposure of global
  BGP state must be decided before launch.
- Probe destinations, duration, packet volume, concurrency, and output are
  bounded to prevent the service from becoming a scanning or load tool.
  Private or platform addresses require explicit authorization tied to a
  suitable vantage point.
- A failed or partial diagnostic must say what was observed and what was not.
  Packet loss and a silent traceroute hop are observations, not service errors.

## Success criteria

A project member with permission can run a diagnostic from a named edge
location in the portal or CLI, understand which routers answered, and distinguish
a network observation from a platform failure. Support can use the same result
to investigate the incident without granting node access. Session setup time,
completion rate, and partial coverage are measured before wider rollout.

## Open product decisions

- Which roles may see route details, peer information, and active probe results?
- Should complete results be retained and shareable, and for how long?
- Which customer VPC vantage point should be offered after the fabric edge?

The [system architecture](../../../architecture/networking/looking-glass.md)
describes the proposed service boundaries and connection path.
