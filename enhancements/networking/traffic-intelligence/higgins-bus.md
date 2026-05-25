# Higgins Bus

**Parent:** [Total Load Balancing](total-load-balancing.md)  
**Status:** In progress  
**Codename:** Internal project name — not a go-to-market product name.

---

Higgins Bus is the pub/sub transport layer for Total Load Balancing, built on MoQ Transport (MOQT). It carries signals — geo data, named IP lists, and future routing intelligence — from authoritative sources to every edge PoP and consumer in near real-time.

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

The [Nate Project](nate.md) introduces health signals — availability, latency, and throughput measurements from distributed active probes. Three track namespaces cover Datum infrastructure, Datum-managed endpoints, and customer-defined checks.

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

## Caveats

**IETF draft status.** MOQT is an active IETF draft, not a finalized RFC. The core object model, subscription semantics, and relay protocol have stabilized, but track namespace conventions and priority signaling are still evolving. Pin the moqstream implementation to a specific draft version and plan for an upgrade pass when the final RFC is published.

**No durable replay.** MOQT relays do not provide durable message replay. A PoP that was offline for an extended period will receive the most recent object on reconnect but will not receive intermediate objects it missed. For named lists this is acceptable — the most recent object is always the full current state. For GeoDB deltas, a PoP that has missed a delta sequence falls back to pulling the full snapshot (see above).

**Object storage dependency.** The GeoDB hybrid model introduces a dependency on object storage for snapshot distribution. This is a deliberate trade-off: it keeps MOQT in its sweet spot (real-time, small-to-medium objects) while using the right tool for bulk artifact distribution.
