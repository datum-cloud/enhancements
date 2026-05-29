# With a Little Help from My Friends: Memory for the MoQ Fabric

*Part 3 of 3 — Durable Intelligence*

> *"I get by with a little help from my friends."*
> — The Beatles, "With a Little Help from My Friends" (1967)

Parts 1 and 2 of this series ended with the same honest admission: MoQ relays don't store history. A subscriber that reconnects after an extended outage gets the most recent state, not a replay of what it missed. We said this was usually fine. We said the edge cases were real. We said we didn't have clean answers yet. Acknowledging the gap without exploring what fills it felt like stopping half a song. This is Part 3.

---

## Why Not Solve It in the Standard?

The obvious response to a protocol with a memory gap is to add memory to the protocol. I've watched that argument play out in enough standards discussions to recognize the pattern: scope grows, implementations diverge on optional storage features, and the properties that made the protocol attractive get harder to reason about. MoQ's decision not to define durable replay is deliberate. Pairing MoQ with something built for persistence is cleaner than bending the relay into something it wasn't designed to be.

I had coffee recently with some friends who work on Hydrolix, a streaming analytics database built for exactly this shape of problem: high-volume time-series data. I haven't had a chance to run any of this by them yet, and none of what follows has been validated on their side. But the conversation got me thinking about whether the combination, live delivery through MoQ and durable intelligence through a time-series store built for it, could cover the full problem more cleanly than either a MoQ extension or a standard database. That's the idea I've been exploring.

---

## The Gap in Practice

The easy answer, "just use current state," is correct more often than you'd expect. A new edge node comes online, subscribes to the relevant tracks, receives current state, and starts operating. But a node that was offline for two hours during an incident has the answer without the context. And current state tells you nothing useful for capacity planning, incident reviews, or catching problems that develop gradually. MoQ discards what it has delivered; nothing in the protocol is meant to answer "what did this track look like three hours ago?" Something else has to carry the memory.

The harder case is the AI scenario from Part 2. An agent doing anomaly detection or trend analysis needs access to what came before. Live streams alone don't provide that, and stitching a baseline from current state is an approximation at best.

---

## Why Hydrolix Is Interesting Here

Standard relational databases struggle with ingest rate at scale. Message queues with retention swap one operational dependency for another without adding an analytics layer. Object storage is cheap but makes bounded time queries across track namespaces awkward.

Hydrolix ingests from Kafka and streaming sources at production scale, stores data in a columnar format that excels at bounded time queries, and speaks SQL. It could absorb the firehose of MoQ objects without ingest becoming a bottleneck, and answer queries like "give me everything on platform/health/pop/# between 14:00 and 14:30 UTC yesterday" efficiently.

Worth being clear about what we're storing: platform signals and inference events — small, structured, time-stamped records. MoQ also carries video segments, and whether the same architecture makes sense for storing those is a different question and probably worth its own post.

Every object on every MoQ track already carries a monotonic sequence number. That turns out to be a natural join condition between historical data and the live stream, a detail that isn't obvious until you start sketching the replay architecture and then feels almost too convenient.

---

## A Sketch of the Pattern

An adapter on the relay subscribes to the relevant tracks as an ordinary subscriber and writes each received object to Hydrolix: track namespace, sequence number, publish timestamp, and payload. The write is asynchronous; Hydrolix ingest doesn't block relay delivery to other subscribers.

When a new or recovering subscriber needs to bootstrap, it queries Hydrolix for the relevant objects ordered by sequence number, processes them, then subscribes to the live relay starting from the sequence number immediately following the last one Hydrolix returned. The handoff is exact because both sides speak the same sequence space. No gap, no overlap, no separate coordination mechanism. The MoQ protocol already provides the thread that stitches them together.

---

## What I'm Still Working Through

The time-series layer would be a real operational dependency, though not in the critical path for live delivery. If it's unavailable, new subscribers can't bootstrap from history and analytics queries fail — but existing live subscribers are unaffected. It's a system you operate, and you should go in with clear eyes about that.

The sequence number handoff has an edge case I haven't fully resolved. It assumes the relay has recent objects buffered at the point of handoff. If a subscriber was offline long enough that the last Hydrolix sequence number is older than the relay's buffer, a gap opens. The obvious mitigation is a Hydrolix retention window that exceeds the relay buffer by a meaningful margin, with a defined fallback for subscribers that exceed it. The right margins for different signal types would need calibration.

---

## Where This Points

The fit feels surprisingly clean: a transport protocol that already carries sequence numbers pairs naturally with a columnar store optimized for historical queries. I'm not ready to call this solved, but the direction feels right.

If you've paired MoQ with a storage layer, built replay into a relay, or think I've got this completely wrong, I'd love to hear from you. And if you have a verse of your own to add to the song, [reach out.](INSERT-LINK-HERE)

Ringo couldn't carry "With a Little Help from My Friends" alone. Neither could Paul. The song needed both.

*With a little help from my friends.*
