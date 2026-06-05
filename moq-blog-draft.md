# All Together Now: Why MoQ Is the Edge Protocol We've All Been Building Separately

*Part 1 of 3 — Platform Coordination*

> *"One, two, three, four — can I have a little more?"*
> — The Beatles, "All Together Now" (1969)

Across the last four companies I've worked at, I've watched at least twenty different systems get built to solve the same fundamental problem: getting information from a control plane out to the edge, fast. Some of them were genuinely beautiful — elegant architectures, carefully designed consistency models, the kind of engineering you feel proud to show people. Some of them were ugly in the way that only production systems under pressure get ugly — polling loops with jittered sleeps, hand-rolled TCP push protocols, database tables treated as message queues. All of them were bespoke. Every single one. Twenty different bands, all writing the same song from scratch — and none of them were the Beatles.

And every time I joined a new one, I'd eventually learn how long it actually took for an update to propagate. The answer was almost always measured in minutes. Not milliseconds. At least twice across those twenty systems, the honest answer was hours. There were one or two that actually got it right — sub-second propagation, the kind of thing you'd demo proudly. But they were bespoke too, which meant nobody else got to learn from them. The gap between "a thing changed" and "the edge knows about it" had quietly grown to the size of a coffee break, and stayed there.

The solutions in use today are mostly compromises. Scheduled polling is simple but slow. Proprietary push protocols are fast but brittle. Centralized message brokers introduce a hub-and-spoke bottleneck that becomes the thing you're protecting at 3am.

The hard problem in large-scale distributed infrastructure isn't compute or storage. It's coordination. The reason we keep building bespoke solutions isn't that the problem is unsolvable — it's that until recently, there wasn't a protocol designed specifically for this shape of problem.

MoQ Transport might be that protocol. Do this once, use it everywhere: for us building a distributed platform, for customers facing the same fan-out problems, and eventually as a cross-vendor standard that lets us all play the same tune. MoQ is an emerging IETF protocol designed for live media distribution, but the properties that make it good for media are exactly the properties you want for distributing platform intelligence to the edge.

---

## What Makes MoQ Different

MoQ is a publish/subscribe protocol built natively on QUIC ([IETF draft](https://datatracker.ietf.org/doc/draft-ietf-moq-transport/)). That single sentence contains several meaningful implications.

**QUIC gives you the network properties you actually want for WAN edge distribution.** No head-of-line blocking means a slow subscriber doesn't stall others. Connection migration means a PoP that briefly loses its upstream path reconnects without dropping subscriptions. On WAN, this typically translates to 20 to 100ms end-to-end delivery — orders of magnitude better than polling.

**The relay topology solves the fan-out problem.** The control plane publishes once; relays distribute to subscribers at the edge. New nodes connect to the nearest relay and immediately begin receiving updates — no changes to the publisher, no registration step.

**The track model fits naturally onto "one stream of updates per named thing."** A track is a named sequence of objects with a monotonic sequence number and an optional TTL. A subscriber receives the latest object on connect, then new objects as they're published — one track per named IP list, one per PoP health state, one per cache purge scope.

**Object TTLs are a first-class primitive.** If a subscriber loses connectivity, it enforces the last-received object until the TTL expires, then falls back to a safe default. Stale data is better than no data up to a point — the TTL is where you draw the line.

---

## Real-Time Routing: Health, Performance, and Capacity

Every routing decision at the edge is only as good as the data behind it. Send a request to a PoP that's overloaded, unhealthy, or running a cold model and you pay in errors or latency. The data isn't hard to produce — every PoP knows its own health, every compute node knows its capacity, every inference worker knows what models it has loaded. The problem is getting that data to the routing decision fast enough to matter.

A monitoring component publishes to platform/health/pop/{pop-id} when health state changes. Compute nodes publish to platform/capacity/{pop-id}. Inference workers publish model availability to platform/model-locality/{model-id}/{worker-id}. DNS resolvers, load balancers, AI gateways — each subscribes to exactly the tracks it needs and receives updates within tens of milliseconds. No polling cycle, no telemetry query at request time. The routing decision uses current state because that is what the relay holds.

The same infrastructure carries signals in both directions. Edge nodes report health upward through the same topology that delivers updates downward. Automated failover and the ops team subscribe to the same streams. No separate telemetry pipeline to operate.

---

## Geo and Rules: Scale and Speed

Two problems look similar from the outside but have completely different shapes.

A commercial IP-to-geography database is hundreds of megabytes and updates on a schedule. You can't stream it as MoQ objects. The answer is a split: the control plane validates a new snapshot, stores it in object storage, and publishes a single small notification to platform/geodb/version — version identifier, artifact URL, checksum. Every subscribed edge node receives that signal within milliseconds and pulls the file over HTTP. ASN data and IP reputation databases follow the same pattern. MoQ carries the signal; object storage carries the bytes.

IP block lists during an attack are a different problem entirely. When a range gets flagged as malicious, when a customer triggers an emergency block mid-DDoS, when rate limits need tightening before the next request wave — these updates are kilobytes, not gigabytes, and they need to reach every enforcement point in seconds. Each list is a track: org/{org-id}/lists/{list-id}. A new object publishes and every enforcement node has it within milliseconds. The TTL ensures that a node that loses connectivity fails to a safe default rather than enforcing stale rules indefinitely.

The relay handles both. Large files on a schedule via the notification-and-pull model. Small urgent rules enforced in near real time as direct objects. The protocol doesn't distinguish between them.

---

## The Customer Layer: Across the Event Horizon

There is a boundary in every customer relationship between what Datum manages and what the customer manages. On our side: edge PoPs, routing, network infrastructure. On theirs: origin servers, application logic, internal services. That boundary is mostly opaque today — customers see aggregate metrics through polling APIs, and we see what happens at the edge but not inside their network.

A shared pub/sub relay changes that geometry. Datum publishes performance and availability signals that customer applications subscribe to and act on in real time — the same data Datum's own routing systems use, at the same latency. The customer publishes signals from their side — origin health, backend capacity, application performance — that Datum's routing can incorporate into forwarding decisions. The information gap across that boundary collapses from minutes to milliseconds, in both directions.

Because it's pub/sub and not a point-to-point integration, third parties can subscribe too. If a customer's analytics platform speaks the protocol — Hydrolix, or any other tool that can consume from the relay — it subscribes to the same tracks alongside Datum's own systems. The data flows to wherever it's useful without a bespoke pipeline on either side.

Tenant isolation on a shared relay requires real engineering. A namespace prefix alone doesn't enforce the boundary — the relay must validate the subscriber's identity token before accepting a subscription. The mechanism exists, but multi-tenant relay authorization isn't yet standardized. This is work we need to own.

---

## What to Be Honest About

MoQ is an active IETF draft. The core — QUIC transport, object model, relay protocol — has stabilized, but namespace conventions and priority signaling are still evolving. Teams building on MoQ today should expect to contribute to that process, not just consume from it.

MoQ relays don't store message history. A subscriber that reconnects gets the most recent object, not a replay of what it missed. For routing signals and health data, current state is usually all that matters. Part 2 covers where that tradeoff gets sharper.

Subscriber authorization isn't defined by the spec. Implementations layer it on via mutual TLS or application-level tokens — the mechanism exists, but the conventions for multi-tenant use are still taking shape.

---

## Where This Goes

Routing signals first. Then health data. Then policy distribution. Then model distribution. Each one is more tracks on the same relay, more subscribers on the same namespace convention, the same freshness guarantees throughout. One relay infrastructure, one operational model, no matter how many use cases you stack on top.

Most of the operational complexity in large distributed platforms comes from the proliferation of slightly different bespoke solutions to the same underlying coordination problem — each with its own failure modes, its own on-call runbooks. A shared transport layer designed for this problem is worth paying attention to. That's Part 2.

If you're building on MoQ, fighting the same coordination problems, or have a verse of your own to add to the song, I'd like to hear from you. [Reach out.](INSERT-LINK-HERE)

All together now.
