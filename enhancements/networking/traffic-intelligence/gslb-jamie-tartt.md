# Jamie Tartt: Global Server Load Balancing

**Parent:** [Total Load Balancing](total-load-balancing.md)  
**Status:** Early definition  
**Codename:** Internal project name — not a go-to-market product name.

---

Named after Jamie Tartt — the striker who started his career thinking only about himself. Fastest on the pitch, most technically gifted, and completely useless until he learned to stop optimizing locally and start playing for the whole team.

> "Stop thinking locally and start optimizing for the whole team."

GSLB is the same evolution applied to DNS. The internet's early routing model was local: BGP picks the nearest PoP by AS-path, and that's where you land. It worked. Then the internet scaled, applications went global, and "nearest" turned out to mean "nearest to the resolver" — which is often in a different country from the user. GSLB is what happened when DNS grew up. It stopped answering "where is the server?" and started answering "where is the *right* server for *this* user, right now?"

As GSLB has matured it has become a foundation of global applications on the internet. Every major CDN, cloud provider, and SaaS platform uses it. Datum's edge platform requires it to steer users to the right PoP based on geography, health, and policy — not just BGP topology.

---

## What GSLB Does

DNS is the first steering point for every user session. Before a TCP connection is opened, before TLS is negotiated, before Envoy sees a request — DNS answers. GSLB makes that answer intelligent:

- **Geo-aware** — return the PoP closest to the user, not just the nearest by AS-path
- **Health-aware** — exclude unhealthy or degraded PoPs from the answer set
- **Policy-aware** — respect sovereignty constraints; never route a user to a jurisdiction their data cannot enter
- **Latency-proxied** — use geographic distance as a latency proxy until real RTT signals are available

The output is a DNS answer — one or more A/AAAA records pointing to the PoP anycast or unicast address that best serves this user. Every other layer (Cilium, Envoy, origin) receives traffic that has already been steered by GSLB.

---

## Role in the Total Load Balancing Stack

GSLB is the outermost steering layer. It operates before any connection is established and shapes the entire traffic distribution across Datum's PoP fleet.

```
User DNS Query
      │
      ▼
Jamie Tartt (GSLB / PowerDNS)   ← geo + health + policy → PoP selection
      │
      ▼
Selected PoP Anycast Address
      │
      ▼
Dani Rojas (Cilium L4)  →  Zava (Envoy L7)  →  Origin / UFO Compute
```

GSLB consumes two signal types from Total Load Balancing:

| Signal | Source | Used For |
|---|---|---|
| Geography (IP→geo) | [Roy Kent](ip-geo-roy-kent.md) GeoDB | Map client IP to location; select nearest PoP |
| PoP Health | [Nate](health-checks-nate.md) via [Higgins Bus](signal-distribution-higgins-bus.md) | Exclude unhealthy PoPs from answer set |

---

## PowerDNS

Datum runs PowerDNS Authoritative Server. GSLB geo routing is implemented using PowerDNS's native GeoIP backend — the design must be grounded in what PowerDNS actually supports, not invented around it.

### Native GeoIP Backend

PowerDNS includes a `geoip` backend that routes DNS responses based on the querying client's geographic location. Key properties:

- **Database format:** MaxMind MMDB (GeoIP2). This aligns with the Roy Kent GeoDB evaluation — the selected vendor must provide MMDB output or equivalent that can be converted. See [#732](https://github.com/datum-cloud/enhancements/issues/732).
- **Geographic granularity:** Continent, country, and region (subdivision) supported natively. City-level routing is supported where the database includes city data.
- **Routing rules:** Expressed per-zone or per-record in YAML configuration. A zone can return different records based on the client's continent, country, or region.
- **Weighted responses:** Supported within a geo rule — multiple PoPs in the same region can be returned with weights for load distribution.
- **EDNS Client Subnet:** PowerDNS supports ECS natively. When a resolver includes an ECS option, PowerDNS uses the client subnet for the geo lookup rather than the resolver IP. This is essential for accuracy — without ECS, a user in Munich behind Google's resolver (8.8.8.8, anycast in the US) gets routed to a US PoP.

### EDNS Client Subnet Handling

| Scenario | PowerDNS Behavior |
|---|---|
| Resolver sends ECS option | Geo lookup uses client subnet from ECS — accurate |
| Resolver does not send ECS | Geo lookup falls back to resolver IP — potentially inaccurate |
| Client uses a local ISP resolver | Resolver IP is usually geographically close — acceptable accuracy |
| Client uses a public resolver (Google, Cloudflare) | Resolver IP is wrong; ECS absence means degraded accuracy |

ECS is not universally supported by resolvers. Accuracy for users behind ECS-absent resolvers will be reduced. This is a known limitation of DNS-based geo steering and not specific to Datum — it affects every GSLB implementation.

### What PowerDNS Does Not Do Natively

| Gap | Notes |
|---|---|
| Active health checks | PowerDNS does not probe PoPs. Health-aware routing requires an external mechanism to update routing rules when a PoP is unhealthy. |
| Near-real-time rule updates without restart | Configuration reload is required to pick up zone file changes. The PowerDNS API supports runtime record updates; whether this is sufficient for PoP health failover without a reload needs confirmation — see [#733](https://github.com/datum-cloud/enhancements/issues/733). |
| Latency-based routing | PowerDNS routes by geography, not measured latency. Distance is used as a proxy until RTT signals are available from a future Total Load Balancing project. |

---

## Health-Aware Failover

GSLB must exclude unhealthy PoPs from DNS answer sets. PowerDNS does not probe PoPs itself — health state comes from the [Nate Project](health-checks-nate.md), which publishes `HealthStatus` objects to Higgins Bus on the `platform/health/pop/{pop-id}` track.

The integration path:

1. Nate probes each PoP continuously and publishes `HealthStatus` to Higgins Bus
2. The Datum control plane subscribes to `platform/health/pop/{pop-id}` tracks
3. When a PoP transitions to UNHEALTHY or DEGRADED, the control plane updates PowerDNS routing rules to remove or deprioritize that PoP
4. When the PoP recovers, the control plane restores it to the answer set

**Update mechanism:** The control plane uses the PowerDNS API to update records at runtime. Whether this requires a zone reload or takes effect immediately is a key open question — see [#733](https://github.com/datum-cloud/enhancements/issues/733).

**TTL behavior on failover:** When a PoP is removed from the answer set, clients that have already cached the old answer will continue routing to that PoP until the TTL expires. Short TTLs reduce this window but increase DNS query volume. TTL strategy needs to be defined.

**DDoS-aware steering:** GSLB also subscribes to `platform/ddos/pop/{pop-id}` from [Beard](ddos-scrubbing-beard.md). When Beard reports a PoP as `mitigating` or `blackholed`, the control plane deprioritizes that PoP in DNS answers, steering new sessions to less-affected PoPs. This signal is distinct from Nate health — a PoP under active attack may show degraded health metrics without being infrastructure-failed. Treating these signals separately avoids incorrectly retiring a healthy PoP that is simply absorbing attack traffic.

---

## GeoDB Integration

The PowerDNS GeoIP backend reads the GeoDB from a local file in MMDB format. Roy Kent distributes GeoDB snapshots to PoP-local storage via Higgins Bus — the same distribution model serves both edge nodes (for Envoy geo blocking and ALB) and the control plane nodes running PowerDNS.

When Roy Kent installs a new GeoDB snapshot:
1. The snapshot is written to the local path PowerDNS is configured to read
2. PowerDNS must be notified to reload the database — either via a SIGHUP, a zone reload, or the API
3. Until reload, PowerDNS continues serving responses from the previous snapshot

The reload mechanism and any gap between snapshot install and PowerDNS picking it up needs to be defined. A stale GeoDB in PowerDNS routes users to the wrong PoP — for the duration of their DNS TTL.

---

## PoP Topology Configuration

Each Datum PoP has a geographic coordinate (lat/lon) and a set of anycast or unicast addresses. GSLB routing rules map geographic regions to PoP addresses. The topology is maintained by the control plane and expressed as PowerDNS zone configuration.

When a new PoP is brought online or taken offline, the control plane updates the PowerDNS zone configuration. This is a control plane operation, not a customer-facing action.

Customer-visible behavior: DNS answers for customer zones return PoP addresses selected by GSLB. Customers do not configure GSLB directly — they configure whether geo steering is enabled for their zone, and GSLB handles the rest.

---

## Customer Configuration

Customers do not configure PowerDNS directly. The customer-facing surface is a `GeoSteeringPolicy` resource that targets a DNS zone. The control plane translates the policy into PowerDNS zone configuration.

### GeoSteeringPolicy

```yaml
apiVersion: networking.datum.net/v1alpha
kind: GeoSteeringPolicy
metadata:
  name: example-com-geo
  namespace: acme-corp
spec:
  targetRef:
    group: dns.datum.net
    kind: Zone
    name: example.com
  enabled: true
  ttl: 60s                        # DNS TTL for geo-steered records
  fallback:
    behavior: nearest-healthy     # nearest-healthy | round-robin | specific-pop
    pop: us-east                  # only used when behavior: specific-pop
  sovereignty:
    rules:
      - clientRegions: [EU]
        allowedJurisdictions: [EU]
      - clientRegions: [US, CA]
        allowedJurisdictions: [US, CA]
```

When `enabled: false` or no `GeoSteeringPolicy` exists for a zone, DNS answers are returned without geo steering — standard round-robin across all healthy PoPs.

### What the Customer Controls vs What Datum Manages

| | Customer configures | Datum manages |
|---|---|---|
| Geo steering on/off | `spec.enabled` | — |
| DNS TTL | `spec.ttl` | Minimum TTL floor enforced by platform |
| Fallback behavior | `spec.fallback` | PoP health state that drives which PoPs qualify as "healthy" |
| Sovereignty rules | `spec.sovereignty.rules` | PoP jurisdiction registry (which PoP is in which legal jurisdiction) |
| PoP selection logic | — | Geographic proximity scoring, PowerDNS GeoIP rule generation |
| Health failover | — | Nate signal subscription, PowerDNS record updates on health transitions |
| GeoDB updates | — | Roy Kent snapshot distribution and PowerDNS reload |
| EDNS Client Subnet | — | PowerDNS ECS configuration and fallback behavior |

### TTL Guidance

The `spec.ttl` field sets the DNS TTL for geo-steered records. This directly controls the failover window: a client that has cached a PoP address continues routing to that PoP until the TTL expires, even if the PoP is removed from the GSLB answer set due to a health event.

| TTL | Tradeoff |
|---|---|
| 30–60s | Fast failover; higher DNS query volume |
| 60–120s | Balanced — recommended default |
| 300s+ | Lower query volume; 5-minute window during which clients route to a failed PoP |

The platform enforces a minimum TTL floor (TBD) to prevent abuse of the DNS infrastructure. The right default and minimum values depend on Nate failover propagation time — see open question in the Open Questions section.

### Visibility

Customers can inspect the current GSLB answer for their zone using `datumctl`:

```
datumctl get geosteeringpolicy example-com-geo
datumctl describe zone example.com --gslb
```

The `describe` output will show which PoPs are currently in the answer set, which are excluded due to health, and which are excluded due to sovereignty rules. This is the primary debugging surface for "why is traffic going to the wrong PoP?" questions.

Status on the GeoSteeringPolicy resource reflects current control plane state:

```yaml
status:
  activePops:
    - id: us-east-1
      status: HEALTHY
      inAnswerSet: true
    - id: eu-west-1
      status: HEALTHY
      inAnswerSet: true
    - id: ap-southeast-1
      status: DEGRADED
      inAnswerSet: false
      excludedReason: health
  lastUpdated: "2026-05-29T14:00:00Z"
```

---

## Sovereignty Constraints

Some customers or workloads have data residency requirements that prohibit routing through certain jurisdictions. GSLB must respect these constraints as hard rules — a non-compliant PoP must be excluded regardless of proximity or health.

Sovereignty enforcement at the GSLB layer means: if a user's request must not be processed in jurisdiction X, and the closest PoP is in jurisdiction X, GSLB must return the next-best compliant PoP instead. This requires:

- Per-customer or per-zone sovereignty configuration
- A PoP jurisdiction registry (which PoP is in which legal jurisdiction)
- Rule evaluation before proximity scoring

Sovereignty signals are listed in the Total Load Balancing roadmap as a future project. Until those signals are available, sovereignty constraints at GSLB will require manual configuration. The architecture should accommodate sovereignty rules in the routing logic from the start.

---

## Open Questions

Research for [#733](https://github.com/datum-cloud/enhancements/issues/733) is required to close the following before the GSLB design can be finalized:

**PowerDNS API runtime updates.** Can the PowerDNS API update geo routing rules (add/remove a PoP from an answer set) without a zone reload? If a reload is required, what is the reload latency and does it cause a service interruption? This determines whether Nate-driven failover can operate in near-real-time or has a hard floor imposed by reload time.

**GeoDB reload mechanism.** How does PowerDNS detect and load a new MMDB snapshot? SIGHUP? API call? File watch? What is the reload latency, and what happens to in-flight queries during reload?

**TTL strategy.** What TTL should GSLB zones use? Short TTLs (30–60s) reduce the failover window but increase query volume. Long TTLs (300s+) reduce query volume but extend the window during which clients route to a failed PoP. The right value depends on Nate failover propagation time and acceptable impact duration.

**ECS scope.** Should Datum's PowerDNS configuration actively request or encourage ECS from upstream resolvers? Are there privacy or compliance implications of logging ECS client subnets that need to be addressed?

**Weighted multi-PoP answers.** When multiple PoPs serve the same region, should GSLB return all of them (letting the client pick via round-robin or connection preference) or return a single best PoP? Returning multiple reduces the blast radius of a single PoP failure but requires clients to try alternates — which not all do.

**Failover logging.** When the control plane removes a PoP from the GSLB answer set due to a Nate health signal, where is that event recorded? What is visible to Datum operators and to affected customers?
