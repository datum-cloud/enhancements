# Higgins Bus

**Parent:** [Total Load Balancing](total-load-balancing.md)  
**Status:** In progress  
**Codename:** Internal project name — not a go-to-market product name.

---

Higgins Bus is the pub/sub transport layer for Total Load Balancing, built on MoQ Transport (MOQT). It carries signals — geo data, named IP lists, and future routing intelligence — from authoritative sources to every edge PoP and consumer in near real-time.

The decision to build on MOQT reflects a broader conviction about where this protocol is heading. MoQ is emerging as a foundation layer not just for platform coordination — routing signals, health state, geo data — but for the AI workloads that will run alongside them: token streaming from inference workers, worker discovery without a registry, job scheduling across a distributed compute fleet, and agent-to-agent pipeline handoffs. These are the same fan-out and coordination problems, carrying different payloads. Building Higgins Bus on MOQT now means the relay infrastructure established here extends naturally to serve inference and agent use cases as they mature — one fabric, not two systems built side by side.

---

## Why MOQT

Total Load Balancing has two hard requirements for its distribution layer:

1. **Low-latency fan-out** — changes must reach every edge PoP in seconds, not minutes
2. **No central broker bottleneck** — the distribution path must survive a single node failure and scale to hundreds of PoPs

MOQT is a pub/sub protocol built on QUIC and WebTransport, designed for exactly this shape of problem. Key properties:

- **QUIC-native** — no head-of-line blocking, connection migration, and low-latency stream setup make it well-suited for WAN distribution across geographically dispersed PoPs
- **Relay topology** — MOQT relays fan out subscriptions at the network edge, so a single publisher reaches many subscribers without a hub-and-spoke bottleneck
- **Track-based model** — named tracks with monotonic object sequences map cleanly to the "one named list = one update stream" shape of our distribution problem
- **Object TTLs** — built-in expiry semantics match the TTL requirements for named IP lists
- **Protocol maturity** — MOQT is an active IETF draft (moq-transport); the spec is stable enough to build on and aligns with the existing moqstream infrastructure investment

---

## Track Namespace

Each signal type is assigned a track namespace. Subscribers subscribe to the tracks they need; publishers write to the canonical track path.

### Roy Kent Project

| Track | Publisher | Consumers | Object content |
|---|---|---|---|
| `platform/geodb/version` | Control plane | All PoPs | Version ID + snapshot URL + checksum |
| `platform/geodb/delta/{version}` | Control plane | All PoPs | Delta patch from previous version |
| `org/{org-id}/lists/{list-id}` | Control plane (on customer write) | PoPs serving that org | Full list payload + sequence number + TTL |

### The Nate Project

The [Nate Project](health-checks-nate.md) introduces health signals — availability, latency, and throughput measurements from distributed active probes. Three track namespaces cover Datum infrastructure, Datum-managed endpoints, and customer-defined checks.

| Track | Publisher | Consumers | Object content |
|---|---|---|---|
| `platform/health/pop/{pop-id}` | Nate control plane | Total Load Balancing, GSLB, Galactic VPC | Aggregated HealthStatus for a Datum PoP — overall status, per-region availability and latency, last probe time |
| `platform/health/endpoint/{endpoint-id}` | Nate control plane | ALB, Connectors, Total Load Balancing | Aggregated HealthStatus for a Datum-managed upstream or endpoint |
| `org/{org-id}/health/{check-id}` | Nate control plane | Customer ALB policy, customer DNS, customer consumers | Aggregated HealthStatus for a customer-defined HealthCheck |

**Publish triggers:** objects are published immediately on any status transition (HEALTHY → DEGRADED, DEGRADED → UNHEALTHY, and the reverse) and on a heartbeat interval even when status is unchanged, so consumers can detect a failed publisher.

**Bootstrap:** a new subscriber receives the most recent HealthStatus object immediately on subscription — no separate initialization step. The most recent object is always the authoritative current state.

**Metrics path:** probe measurements (raw latency samples, availability counts, throughput) are also written to the Datum metrics pipeline in parallel with Higgins Bus publication. The metrics path serves dashboards and alerting; Higgins Bus serves real-time routing decisions.

### Future Projects

Each subsequent Total Load Balancing project extends the namespace:

| Track | Signal Group |
|---|---|
| `platform/rtt/{pop-id}` | RTT / Packet Loss / Congestion |
| `platform/sovereignty/{jurisdiction}` | Sovereignty |
| `platform/model-locality/{model-id}` | Model Locality |
| `platform/compute/{pop-id}` | Compute Availability |

Track namespaces are additive — adding a new signal type requires no changes to existing relay infrastructure or existing subscribers.

### Inference and Agent Workloads

Higgins Bus is not limited to routing signals. The same relay infrastructure extends to inference management and agent-to-agent communication — the same fan-out and coordination problems as Total Load Balancing, carrying different payloads. These namespaces are reserved as the inference layer matures; track definitions will be refined as implementation begins.

| Track | Publisher | Consumers | Object content |
|---|---|---|---|
| `platform/inference/capacity/{model-id}/{region}/{worker-id}` | Inference worker | AI gateway, job schedulers | Worker endpoint, current load %, max concurrent jobs — TTL acts as heartbeat; a dead worker disappears automatically |
| `platform/inference/queue/{model-id}` | Job coordinator | Inference workers | Job payload — prompt, parameters, callback address, priority, TTL; expired unclaimed jobs are re-queued by coordinator |
| `platform/model-locality/{model-id}/{worker-id}` | Inference worker | Schedulers, AI gateway | Model load state (warm / cold / loading), worker endpoint — lets schedulers avoid cold-start latency |
| `{org}/inference/{request-id}/tokens` | Inference worker | Initiating client, downstream agents, logging pipeline | Token batch with sequence number; subscriber that loses connectivity resumes from last sequence number without re-inference |
| `{org}/agents/{agent-id}/output/{job-id}` | Agent | Downstream agents, orchestrators, logging pipeline | Agent output object with sequence number; downstream agents subscribe directly — no orchestrator in the handoff path |

The TTL model applies uniformly: a subscriber that loses connectivity enforces the last-received object until TTL expires, then falls back to a safe default. For capacity and queue tracks, the safe default is "no workers available / no jobs pending" — the scheduler waits rather than routing to a potentially stale endpoint.

---

## GeoDB — Hybrid Distribution Model

GeoDB updates have a different shape than named IP list updates: the full database is large (potentially hundreds of MB), and updates arrive on a vendor schedule (daily). MOQT is optimized for real-time streaming of small-to-medium objects; bulk transfer of a full GeoDB snapshot does not fit the object model cleanly.

The hybrid model splits the concern.

### Initial Load and Daily Snapshots

1. The control plane fetches the new GeoDB snapshot from the vendor on schedule
2. The snapshot is validated (checksum, coverage sample), versioned, and stored in object storage
3. The control plane publishes a new object to `platform/geodb/version` containing the version ID, snapshot URL, and checksum
4. PoP nodes subscribe to this track; on receiving a new object, each PoP pulls the snapshot artifact from object storage and installs it locally
5. The installed replica is served in-process to local consumers (DNS, Envoy, ALB, metrics enrichment)

### Delta Patches (where supported by vendor)

Where the GeoDB vendor provides incremental delta feeds:

1. Deltas are published as objects to `platform/geodb/delta/{version}`
2. PoPs apply deltas to the current local replica without re-downloading the full snapshot
3. The full snapshot track remains the source of truth — a PoP that misses a delta sequence falls back to pulling the next full snapshot

### Failure Handling

| Scenario | Behavior |
|---|---|
| PoP offline during snapshot publish | On reconnect, subscribes to `platform/geodb/version`; receives the latest object and pulls the current snapshot |
| Snapshot pull from object storage fails | PoP retries with backoff; continues serving the last known-good replica until successful |
| Delta sequence gap | PoP discards delta state and pulls the current full snapshot |
| Vendor feed unavailable | Last known-good snapshot remains active; alerting fires on staleness threshold |

---

## Named IP Lists — Real-Time Distribution

Named IP lists map directly onto the MOQT object model:

- Each list is a track: `org/{org-id}/lists/{list-id}`
- Every update is a new object carrying the full list payload, a monotonic sequence number, and a TTL
- Edge PoPs subscribe to the tracks for all lists visible to their tenants
- Updates arrive within QUIC's latency budget — typically sub-second from commit to enforcement

### Bootstrap (new PoP or new subscriber)

A new PoP subscribing to a list track receives the most recent object immediately — no separate bootstrap step required. The most recent object is always the authoritative current state.

### TTL and Expiry

Each list object carries a TTL. If a PoP loses connectivity and cannot receive new objects, it enforces the last-received list until the TTL expires, then fails to the default policy configured by the customer (fail open or fail closed). A stale list is better than no list up to the TTL; beyond the TTL, a stale list risks enforcing outdated intent.

---

## Relay Infrastructure

MOQT relays sit between the control plane publisher and edge PoP subscribers. They are the fan-out layer — the control plane publishes once; relays replicate to all subscribers at the edge.

This relay infrastructure is provided by **moqstream** — the existing Datum MoQ platform. The Roy Kent Project is the first production use of moqstream for signal distribution.

**Deployment topology:**

```
Control Plane
    │
    ├─ publishes to moqstream relay cluster
    │
    ├─── Relay (region A) ──── PoP A1, PoP A2, PoP A3
    ├─── Relay (region B) ──── PoP B1, PoP B2
    └─── Relay (region C) ──── PoP C1, PoP C2, PoP C3
```

PoPs connect to their regional relay. The relay receives each published object once from the control plane and fans it out to all connected PoP subscribers. New PoPs coming online connect to the nearest relay and immediately begin receiving updates.

---

## Durable Replay

MOQT relays hold recent objects in memory but do not store message history. For most Higgins Bus use cases this is acceptable — the most recent geo snapshot, health status, or IP list is the authoritative current state, and intermediate updates missed during a brief outage are irrelevant. Current state is all that matters.

Two cases where current-state-only is not sufficient:

- **Extended outages** — a node offline for hours needs context to understand what changed during the gap, not just where things stand now. Incident review and post-mortem analysis depend on this.
- **AI workloads** — agents performing anomaly detection, trend analysis, or incident correlation require access to signal history, not just the most recent value. This use case will grow as Higgins Bus expands to carry inference management signals.

### Pattern

An adapter subscribes to the relay as an ordinary MOQT subscriber and writes each received object to a time-series database: track namespace, sequence number, publish timestamp, and payload. The write is asynchronous — the adapter is not in the critical path for relay delivery.

When a subscriber needs to bootstrap from history, it queries the TSDB for the relevant track namespace and time range, ordered by sequence number. After processing the historical records, it subscribes to the live relay starting from the sequence number immediately following the last TSDB record. The handoff is exact because both sides use the same sequence space — no gap, no overlap, no separate coordination step.

```
Historical query (TSDB)         Live relay
  seq 1 ... seq 847             seq 848, 849 ...
              └──── handoff ───┘
```

Every MoQ object already carries a monotonic sequence number. That is a natural join condition between the TSDB and the live relay — a property of the protocol that makes replay architecture cleaner than it would be with most alternatives.

### Storage

Signal data on Higgins Bus — health events, geo updates, IP list changes, capacity and model locality signals — is small, structured, time-stamped, and written at high volume. A time-series database with columnar storage handles this ingest rate without becoming a bottleneck and makes bounded time range queries efficient. "Give me everything on `platform/health/pop/#` between 14:00 and 14:30 UTC yesterday" is the natural query shape.

Hydrolix is an early candidate for evaluation — a streaming analytics database built for high-volume time-series ingest that speaks SQL. No vendor is selected; evaluation is required before committing to a storage layer.

### Retention per Signal Type

Not all signal types have equal replay value. Retention windows should reflect this:

| Signal type | Replay value | Suggested retention |
|---|---|---|
| PoP and endpoint health events | High — essential for incident review and post-mortem | 30–90 days |
| Customer health checks | Medium — useful for customer-facing incident timelines | 30 days |
| Inference capacity and model locality | Medium — trend analysis, scheduling tuning | 7–30 days |
| Named IP list updates | Low — current state is authoritative; history useful mainly for audit | 7 days |
| GeoDB version notifications | Very low — node should pull current snapshot; version history has minimal replay value | 7 days |

### Operational Characteristics

The TSDB is not in the critical path for live delivery. If unavailable:
- Existing live subscribers are unaffected
- New subscribers fall back to bootstrapping from current state only
- Historical queries and analytics fail

**Handoff edge case.** The TSDB retention window should exceed the relay's in-memory buffer by a meaningful margin. A recovering subscriber whose last known sequence number falls within the TSDB window but outside the relay buffer hands off cleanly. Beyond the TSDB retention window, the subscriber falls back to current state with no historical context. Calibrating the margin per signal type is part of the retention strategy above.

---

## Caveats

**IETF draft status.** MOQT is an active IETF draft, not a finalized RFC. The core object model, subscription semantics, and relay protocol have stabilized, but track namespace conventions and priority signaling are still evolving. Pin the moqstream implementation to a specific draft version and plan for an upgrade pass when the final RFC is published.

**No durable relay history.** MOQT relays do not store message history. The Durable Replay section above describes the time-series storage approach for subscribers that need historical context. For named lists, current state is always sufficient — the most recent object is the full authoritative state. For GeoDB deltas, a PoP that misses a delta sequence falls back to pulling the current full snapshot.

**Object storage dependency.** The GeoDB hybrid model introduces a dependency on object storage for snapshot distribution. This is a deliberate trade-off: it keeps MOQT in its sweet spot (real-time, small-to-medium objects) while using the right tool for bulk artifact distribution.
