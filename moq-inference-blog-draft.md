# Come Together: What MoQ Makes Possible for Distributed AI Agents

*Part 2 of 3 — Distributed AI Inference*

> *"Come together, right now."*
> — The Beatles, "Come Together" (1969)

Part 1 made the case that MoQ is the right transport layer for the distribution problems every distributed platform eventually builds bespoke solutions for: routing signals, health state, cache invalidation, model distribution. This post is the other half: what we're building next for AI inference, and why the same infrastructure handles both.

Inference is not an API you call over HTTP. It's a distributed workload: token streams that fan out to multiple consumers, workers that need to be discovered without a registry, pipelines of agents that hand off work without an orchestrator in the critical path. These are the same shapes as the problems in Part 1. They map onto the same relay infrastructure and the same track model, built once and used for both.

---

## Agent-to-Agent Pipelines: Composition Without Orchestration

Building a pipeline of agents — summarization passes to translation, translation passes to formatting — almost always means building an orchestrator: a component that knows the pipeline shape, calls each stage in sequence, and handles errors across the chain. Orchestrators work for fixed, synchronous pipelines. They accumulate complexity fast when pipelines are dynamic, when stages can run in parallel, or when a downstream agent needs to start processing before the upstream one finishes.

Each agent publishes its output as a track: {org}/agents/{agent-id}/output/{job-id}. Any downstream agent subscribes. A translation agent can begin processing the first tokens of a summarization before the summarization is complete — real parallelism across stages, without a coordinator managing the handoff. The output of one agent becomes the input to another through subscription, not a function call.

The pipeline topology is defined by namespace conventions and subscription configuration, not by code in a central coordinator. Adding a consumer is a subscription. Routing output to a different downstream agent is a configuration change. The pipeline isn't declared anywhere central. It emerges from the subscriptions.

---

## Token Streaming: The Building Block

Tokens are a stream — literally what a language model produces, one at a time, in sequence. The dominant delivery pattern today is Server-Sent Events over HTTP: a single persistent connection per inference request, no multiplexing, head-of-line blocking built in. When a slow request shares a connection with a fast one, the fast one waits. QUIC eliminates this.

In the MoQ model, each inference request gets a track: {org}/inference/{request-id}/tokens. The worker publishes objects to that track as tokens arrive. The client that initiated the request subscribes. A downstream agent subscribes. A logging pipeline subscribes. None of them block the others, none require the worker to maintain separate connections per consumer, and a subscriber that loses connectivity mid-stream reconnects and resumes from its last sequence number without rerunning inference.

---

## Inference Discovery: No Registry Required

The usual solution to distributed inference discovery is a service registry: a central catalog of workers, loaded models, and current capacity. The catalog becomes a thing you operate, keep consistent, and page someone about at 3am.

MoQ offers discovery via subscription instead. A worker with the summarization-v2 model loaded publishes its availability to platform/inference/capacity/{model-id}/{region}/{worker-id} — model identifier, worker endpoint, current load, maximum concurrent jobs. The TTL is the heartbeat: a worker that dies is automatically removed from any scheduler's view. A scheduler subscribing to platform/inference/capacity/{model-id}/# gets a live view of every available worker for that model. Workers that also publish to platform/model-locality/{model-id}/{worker-id} let schedulers route to nodes that already have the required weights resident, avoiding cold starts that can add seconds to a request.

---

## Distributed Scheduling: Work as a Stream

Centralized scheduling creates a bottleneck and a failure domain. The MoQ scheduling model inverts the dependency.

A coordinator publishes work items to platform/inference/queue/{model-id}. Each object is a job: prompt, parameters, callback address, priority, TTL. Workers subscribed to the queue claim jobs by publishing to an acknowledgment track. Jobs whose TTL expires without completion are re-queued. A coordinator handling ten thousand requests per second doesn't need ten thousand open connections — it publishes to a track, and the relay delivers to whoever is subscribed. MoQ objects carry a priority field, so high-priority jobs reach workers first without a separate scheduling algorithm. The operational surface doesn't grow; this is an application built on relay infrastructure you're already running.

---

## What to Be Honest About

Short synchronous request and response fits HTTP well. MoQ's advantages compound with long token streams, parallel consumers, and pipelines with multiple stages. If your inference is mostly synchronous and single hop, you're already using the right protocol.

The absence of durable replay is a sharper tradeoff for inference than for platform signals. A job that expires unacknowledged while a worker is offline is gone. For inference where freshness matters, that's fine. A stale job is usually worthless. For batch workloads where durability matters, a message queue with replay semantics is a better fit. Part 3 explores what a durable memory layer for the MoQ fabric actually looks like.

Operational tooling is still maturing. Client libraries and relay implementations are usable. Observability, quota enforcement, and debugging at the track level are not yet at the maturity you'd expect from production HTTP infrastructure.

---

## The Convergence That's Been Coming

Live video streaming and AI inference have historically run on separate infrastructure. But a video frame is an object in a named sequence, and a token is an object in a named sequence. The network properties you want for both are identical: QUIC stream independence, relay distribution, connection migration.

Platforms already running MoQ relay infrastructure can extend it for inference scheduling and agent communication without building something new. The relay doesn't know or care what it's carrying. Token streams fan out the same way video frames do. Agents come together around shared tracks, without anyone running the meeting.

If you're building distributed inference pipelines or experimenting with MoQ for agent communication, I'd like to hear about it. [Reach out.](INSERT-LINK-HERE)

Come together.
