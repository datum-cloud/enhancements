# The Beard Project: DDoS Detection and Scrubbing

**Parent:** [Total Load Balancing](total-load-balancing.md)  
**Status:** Early definition  
**Codename:** Internal project name — not a go-to-market product name.

---

Named after Coach Beard — the one who operates quietly in the background, is always there when things go wrong, and who you barely notice until you need him. When an attack lands, Beard has already been watching, has already classified it, and has already started doing something about it. Nobody runs toward the chaos the way he does.

> "I believe in process."

DDoS mitigation is the same. The moment an attack is visible to the operator, it should already be half over.

---

## What Beard Does

Beard is Datum Cloud's DDoS detection and scrubbing system. It continuously monitors traffic at every edge PoP, detects attack patterns using flow telemetry and inline telemetry, and applies the right mitigation at the right layer — inline scrubbing, surgical upstream filtering, or blackhole routing as a last resort.

The primary goal is **cleaned traffic delivery**: legitimate traffic reaches the customer's application even during an active attack. Blackholing a destination is always available but is a last resort — it ends the attack by ending the service, which is rarely what a customer wants.

---

## Architecture

Datum's anycast network is the structural advantage here. Attack traffic is naturally distributed across PoPs by BGP — no single point absorbs the full volume. Datum's anycast architecture makes inline scrubbing the preferred mitigation model for the majority of attacks, minimizing diversion complexity and avoiding centralized bottlenecks. Traffic arrives at the nearest PoP, is cleaned there, and flows directly to the next layer. No GRE tunnels, no scrubbing center RTT, no re-injection complexity.

```
                      ┌────────────────────────────┐
                      │      Internet Traffic      │
                      │   Legitimate + Attack Flow │
                      └─────────────┬──────────────┘
                                    │
                                    │  anycast — attack volume
                                    │  distributed across PoPs
                                    │
                    ┌───────────────┼───────────────┐
                    │               │               │
                    ▼               ▼               ▼
             ┌──────────┐    ┌──────────┐    ┌──────────┐
             │  PoP A   │    │  PoP B   │    │  PoP C   │
             └────┬─────┘    └────┬─────┘    └────┬─────┘
                  │               │               │
         ─ ─ ─ ─ ─│─ ─ per-PoP pipeline ─ ─ ─ ─ │─ ─ ─ ─
                  │                               │
                  ▼ (one PoP shown)               │
    ┌─────────────────────────────────────────┐   │
    │             Anycast Edge PoP            │   │
    │                                         │   │
    │  ┌──────────────┐   ┌────────────────┐  │   │
    │  │Flow Telemetry│   │ Inline Fast    │  │   │
    │  │sFlow/NetFlow │   │ Path           │  │   │
    │  │IPFIX/XDP     │   │ XDP/eBPF       │  │   │
    │  │counters      │   │                │  │   │
    │  └──────┬───────┘   │ · SYN cookies  │  │   │
    │         │           │ · UDP valid.   │  │   │
    │         ▼           │ · Bogon drop   │  │   │
    │  ┌──────────────┐   │ · IP repute.   │  │   │
    │  │  FastNetMon  │   │ · PPS / BPS    │  │   │
    │  │  Detection   │   │   rate limit   │  │   │
    │  └──────┬───────┘   └───────┬────────┘  │   │
    │         │                   │           │   │
    │         └─────────┬─────────┘           │   │
    │                   ▼                     │   │
    │       ┌───────────────────────┐         │   │
    │       │  Mitigation Control  │         │   │
    │       │       Plane          │         │   │
    │       │                      │         │   │
    │       │ · Classify attack    │         │   │
    │       │ · Select mode        │         │   │
    │       │ · Generate rules     │         │   │
    │       │ · Publish signals    │◄────────┘   │
    │       └──────────┬───────────┘             │
    │                  │                         │
    │       ┌──────────┼──────────┐              │
    │       │          │          │              │
    │       ▼          ▼          ▼              │
    │  ┌─────────┐ ┌────────┐ ┌───────┐         │
    │  │  GoBGP  │ │XDP/eBPF│ │ RTBH  │         │
    │  │FlowSpec │ │ Maps   │ │(last  │         │
    │  │upstream │ │inline  │ │resort)│         │
    │  │surgical │ │scrub   │ └───┬───┘         │
    │  └────┬────┘ └────┬───┘     │             │
    │       │           │         │             │
    └───────┼───────────┼─────────┼─────────────┘
            │           │         │
            ▼           ▼         ▼
       upstream    ┌─────────┐  upstream
       peers drop  │ Cleaned │  null-route
       attack      │ Traffic │  destination
       patterns    │         │
                   │ Envoy   │
                   │ L4 / App│
                   └─────────┘
```

Beard's effectiveness is bounded by the available ingress, transit, and compute capacity of the Datum edge. Anycast distribution and inline mitigation dramatically increase the volume the platform can absorb, but sufficiently large attacks may still require traffic engineering, upstream cooperation, or destination blackholing.

---

## Mitigation Modes

Beard selects the mitigation mode based on attack type, volume, and impact. Modes are not mutually exclusive — a large attack may use all three simultaneously at different layers.

### Mode 1: Inline Scrubbing (Primary)

XDP/eBPF programs run in the kernel fast path on each edge node, processing packets before they enter the networking stack. Rules are loaded into eBPF maps and updated dynamically by the mitigation control plane as attack signatures evolve. Enforcement is per-PoP, operating independently at each anycast ingress point.

| Technique | What it stops |
|---|---|
| SYN cookie proxying | SYN flood — validates TCP handshake before forwarding |
| UDP source validation | UDP reflection/amplification — drops spoofed-source UDP |
| Bogon filtering | Packets from invalid or reserved source addresses |
| IP reputation lookup | Known attack sources from Roy Kent lists and platform threat intel |
| PPS / BPS rate limiting | Volumetric floods — limits per-source and per-destination |
| Protocol validation | Malformed packets, invalid flag combinations, oversized fragments |
| Geo-based filtering | Drops from specific countries/regions per customer sovereignty or attack-origin policy |

Cleaned traffic passes directly to Envoy / L4 within the same PoP. No redirection, no tunneling, no added latency.

### Mode 2: BGP FlowSpec (Surgical Upstream)

GoBGP announces BGP FlowSpec rules (RFC 8955) to upstream transit peers and peering partners. FlowSpec encodes flow-matching conditions (source prefix, destination prefix, protocol, port, packet length, DSCP) and an associated action (discard, rate-limit, redirect, mark).

FlowSpec is the right tool when attack traffic can be identified by a stable pattern that doesn't overlap with legitimate traffic — a specific source prefix, a specific protocol/port combination, an anomalous packet size. The rules propagate to upstream routers and drop matching traffic before it consumes PoP ingress capacity.

FlowSpec rules are generated by the mitigation control plane and withdrawn automatically when the attack signature clears. Rule lifecycle is managed by GoBGP; the control plane does not manage BGP sessions directly.

### Mode 3: RTBH — Remotely Triggered Black Hole (Last Resort)

When a destination prefix is receiving more attack traffic than the platform can clean, and the attack is crowding out legitimate capacity, GoBGP announces a BGP community-tagged route that instructs upstream peers to null-route all traffic to the destination.

RTBH preserves platform stability by sacrificing reachability to the targeted destination. It is the option of last resort and requires either explicit customer opt-in or a platform-defined threshold policy. Customers can configure:

- Whether RTBH is ever permitted for their prefixes
- At what attack volume it is triggered automatically (if at all)
- Notification requirements before or concurrent with activation

RTBH announcements are time-bounded. The control plane sets an expiry and monitors for attack cessation, withdrawing the announcement and restoring service as soon as conditions allow.

---

## Detection: FastNetMon

FastNetMon monitors flow telemetry (sFlow, NetFlow, IPFIX) and XDP-derived counters from every edge node. It classifies attack patterns and generates alerts that the mitigation control plane acts on.

### What FastNetMon Detects

| Attack Class | Detection Signal |
|---|---|
| Volumetric (flood) | PPS and BPS thresholds per destination prefix |
| TCP SYN flood | SYN rate vs SYN-ACK rate anomaly |
| UDP reflection/amplification | UDP packet size distribution, response-to-request ratio |
| ICMP flood | ICMP PPS threshold |
| Protocol anomaly | Packet size distribution, flag pattern anomaly |
| Slowloris / low-and-slow | Connection rate vs completion rate — sourced from Coraza via Zava |
| HTTP flood | Request rate anomalies, repeated patterns — sourced from Coraza via Zava |

Detection thresholds have two levels: **Warning** (elevated monitoring, prepare mitigation) and **Attack** (active mitigation triggered). Both thresholds are configurable per customer and per protected prefix.

### Adaptive Baselines

FastNetMon Advanced (the commercial version) supports automatic threshold learning — it builds per-prefix traffic baselines from historical pps, bps, and flow data and derives detection thresholds from observed normal patterns. Fixed thresholds require manual tuning and produce false positives on traffic with natural volume spikes; baseline-derived thresholds adapt to each customer's actual traffic profile.

The open source version of FastNetMon supports fixed thresholds only. FastNetMon Advanced is required for adaptive baseline support — this informs the licensing decision (see Open Questions).

The baseline learning window is configurable. Until a stable baseline is established for a new customer or prefix, detection operates at conservative fixed thresholds with higher false-negative tolerance. Operators should set expectations with new customers about a profiling window before adaptive detection reaches full accuracy.

### Telemetry Sources

| Source | Layer | Granularity | Latency |
|---|---|---|---|
| sFlow / NetFlow / IPFIX | L3/L4 | Sampled — 1:N packets | 10–30s typical |
| XDP counters | L3/L4 | All packets, per-rule hit counts | Sub-second |
| eBPF ring buffer | L3/L4 | Per-packet metadata for classified flows | Sub-second |
| Coraza (via Zava) | L7 | Per-request rule matches, rate counters | Sub-second |

XDP-derived telemetry closes the detection gap for short-burst attacks that sampling misses. Coraza adds the L7 visibility that flow telemetry cannot provide.

---

## Signal Integration: Higgins Bus

Active attack state is a platform signal that other systems need. A PoP under active attack should influence GSLB, L4, and L7 routing decisions — traffic can be steered away from a PoP absorbing a large attack, or rate limits can be tightened automatically without waiting for operator intervention.

| Track | Publisher | Consumers | Object content |
|---|---|---|---|
| `platform/ddos/pop/{pop-id}` | Beard control plane | GSLB, Total Load Balancing, Ops | Attack state, attack class, intensity (bps/pps), active mitigation mode, affected prefixes |
| `org/{org-id}/ddos/{prefix}` | Beard control plane | Customer systems, alerting | Per-prefix attack state, mitigation mode, estimated traffic impact |

**Consumer behavior examples:**

- **GSLB** subscribes to `platform/ddos/pop/{pop-id}` — when a PoP is under a large volumetric attack it cannot fully absorb, GSLB deprioritizes that PoP in DNS answers and routes new sessions to less-affected PoPs
- **Total Load Balancing** adjusts routing signals for the affected PoP — the health signal contribution from a PoP under attack is interpreted in attack context, not as infrastructure failure
- **Customer alerting** subscribes to `org/{org-id}/ddos/{prefix}` — receives real-time notification of attacks against their prefixes, current mitigation mode, and estimated impact

---

## Relationship to Zava and Coraza

L7 attack detection and enforcement is Zava's responsibility. Beard handles L3/L4; Zava handles the application layer. The two systems share signals through Higgins Bus and act on their respective layers in coordination.

**Coraza** is the WAF layer running as a WebAssembly filter inside Envoy. It implements the OWASP ModSecurity Core Rule Set and inspects HTTP/HTTPS traffic inline. For DDoS purposes, Coraza's role is:

- **HTTP flood detection** — identifies request rate anomalies, repeated patterns, and malformed HTTP that characterise application-layer floods
- **Slowloris / slow-read detection** — connection hold patterns that consume server resources without generating volume detectable by flow telemetry
- **Signature-based blocking** — OWASP CRS rules block common attack tool fingerprints and payload patterns
- **Rate limiting enforcement** — Envoy's `local_ratelimit` and `ratelimit` filters act on signals Coraza surfaces, throttling or blocking attacking sources at the HTTP layer

**Signal flow between Beard and Zava:**

```
Coraza (Envoy WASM filter)
    │
    │ L7 attack signals — source IP, attack class,
    │ request rate, rule match counts
    ▼
Beard Mitigation Control Plane
    │
    ├─ Escalate to L3/L4: if L7 attack source IPs
    │  are generating volume, add to XDP block map
    │
    └─ Coordinate rate limits: signal Zava to tighten
       per-source rate limits below L7 attack threshold
```

When Coraza identifies a source IP generating HTTP flood traffic, Beard's control plane can escalate that signal to the XDP layer — blocking the IP at L3 before its packets even reach Envoy. This cross-layer escalation is more efficient than L7-only enforcement under sustained application-layer floods.

**L7 detection summary:**

| Attack Type | Detection | Enforcement |
|---|---|---|
| HTTP flood | Coraza / Envoy rate counters | Envoy `ratelimit` filter + XDP escalation |
| Slowloris | Coraza connection tracking | Envoy connection timeout tuning + source block |
| Malformed HTTP | Coraza CRS rules | Envoy drops connection |
| Bot / scraper | Coraza CRS + user-agent rules | Envoy block; future JS challenge |
| Credential stuffing | Coraza login endpoint rules | Envoy block + alerting |

---

## Relationship to Roy Kent

Roy Kent's GeoDB and named IP lists are direct inputs to Beard's inline scrubbing layer:

- **Geo filtering** — Beard's XDP programs perform geo lookups against the PoP-local Roy Kent GeoDB replica. A customer whose service has no legitimate users in a particular country can configure a geo block that Beard enforces at the XDP layer during normal operation and tightens automatically during an attack
- **Named IP lists** — customer-curated block lists (`org/{org-id}/lists/{list-id}`) distributed via Higgins Bus feed directly into XDP eBPF maps. A customer can add a newly identified attack source range to their block list and have it enforced at every PoP within seconds
- **Threat intel lists** — Datum-managed lists of known attack infrastructure (botnet C2, amplification reflectors, anonymizing proxies) are distributed the same way and applied across all customers

---

## Customer Configuration

```yaml
apiVersion: networking.datum.net/v1alpha
kind: DDoSProtectionPolicy
metadata:
  name: acme-ddos-policy
  namespace: acme-corp
spec:
  targetRef:
    kind: Zone                    # or IPPrefix
    name: acme-corp-prefixes
  mode: auto                      # auto | scrub-only | off
  scrubbing:
    synCookies: true
    udpValidation: true
    ipReputation: true
    geoFiltering:
      enabled: false              # customer enables; configured via GeoSteeringPolicy
  detection:
    sensitivity: medium           # low | medium | high
    customThresholds:
      ppsWarning: 500000
      ppsAttack: 2000000
      bpsWarning: 1000000000      # 1 Gbps
      bpsAttack: 5000000000       # 5 Gbps
  rtbh:
    permitted: true               # customer opts in to RTBH as last resort
    autoTriggerThreshold: 20Gbps  # trigger automatically above this volume
    requireNotification: true     # notify before triggering if possible
  notifications:
    onAttackStart: true
    onMitigationChange: true
    onAttackEnd: true
```

### What the Customer Controls vs What Datum Manages

| | Customer configures | Datum manages |
|---|---|---|
| Protection mode | `spec.mode` | Platform-wide baseline protection always active |
| Scrubbing techniques | Enable/disable per technique | Technique implementation, threshold tuning |
| Detection sensitivity | `spec.detection.sensitivity` | FastNetMon infrastructure, telemetry collection |
| Custom thresholds | `spec.detection.customThresholds` | Default thresholds based on traffic baseline |
| RTBH opt-in | `spec.rtbh.permitted` | BGP session management, announcement lifecycle |
| RTBH auto-trigger level | `spec.rtbh.autoTriggerThreshold` | — |
| Notifications | `spec.notifications` | Delivery via platform alerting pipeline |
| Block lists | Named IP lists (Roy Kent) | Datum-managed threat intel lists |
| Geo filtering rules | Via `GeoSteeringPolicy` | PoP-local GeoDB and enforcement |

### Status

```yaml
status:
  currentState: under-attack      # clean | elevated | under-attack | mitigating | blackholed
  activeAttacks:
    - prefix: 203.0.113.0/24
      class: udp-amplification
      intensity:
        bps: 12400000000          # 12.4 Gbps
        pps: 8200000
      mitigationMode: inline-scrub
      scrubEfficiency: 0.94       # 94% of attack traffic dropped
      startTime: "2026-06-04T09:12:00Z"
  affectedPoPs:
    - id: us-east-1
      state: mitigating
    - id: eu-west-1
      state: clean
```

---

## Open Questions

- **Customer isolation: detection namespacing.** FastNetMon tracks thresholds and baselines per prefix. Are those strictly namespaced per customer so that a large attack against one customer does not skew detection thresholds or consume baseline learning budget for another? How is per-customer prefix ownership enforced in the detection layer?

- **Customer isolation: XDP rule scoping.** When mitigation rules are pushed to XDP/eBPF maps in response to an attack against Customer A, are those rules strictly scoped to Customer A's prefixes? Could an overly broad drop or rate-limit rule affect Customer B's traffic landing at the same PoP?

- **Customer isolation: FlowSpec blast radius.** BGP FlowSpec rules propagate to upstream peers and are network-wide in effect. A FlowSpec rule targeting an attack against Customer A's prefix could affect Customer B if they share upstream paths, transit providers, or address space in the same aggregate. How are FlowSpec rules scoped to prevent collateral impact, and what is the review process before a rule is announced?

- **Customer isolation: RTBH blast radius.** A blackhole announcement for Customer A's destination prefix must not collaterally affect Customer B. What controls ensure RTBH announcements are scoped to the correct prefix length and do not aggregate into broader blocks that overlap other customers?

- **Customer isolation: resource contention.** XDP/eBPF map space, BGP FlowSpec rule table capacity, and mitigation control plane processing are shared resources at each PoP. Can a sustained large attack against one customer exhaust these resources and degrade protection availability for other customers on the same PoP? What quotas or isolation boundaries apply?

- **Customer isolation: visibility.** Customers subscribe to `org/{org-id}/ddos/{prefix}` for their own attack state. What ensures a customer cannot subscribe to — or infer — another customer's attack state, attack volume, or mitigation mode? This applies both to Higgins Bus subscription authorization and to any portal or API surface that exposes attack telemetry.

- **Baseline profiling.** Accurate detection requires a traffic baseline per protected prefix. How is the baseline established for new customers? How long does profiling take, and what happens during the profiling window?
- **Multi-vector attack handling.** Large attacks frequently combine multiple vectors simultaneously (volumetric + protocol + application layer). The mitigation control plane needs to classify and respond to each vector independently. How are conflicting mitigation actions across vectors resolved?
- **L7 attack visibility.** Resolved — Beard relies on Zava (Envoy + Coraza) for L7 detection and enforcement. See Relationship to Zava and Coraza section.
- **Scrub efficiency reporting.** Customers will want to know how much attack traffic was dropped vs passed. What is the measurement methodology and how is it surfaced — in the status subresource, in metrics, or both?
- **Cross-PoP attack coordination.** A large distributed attack may partially overwhelm multiple PoPs simultaneously. Should the mitigation control plane coordinate across PoPs to adjust GSLB steering during an active attack, or does each PoP mitigate independently? The Higgins Bus integration covers signaling; the coordination logic needs definition.
- **FastNetMon licensing.** FastNetMon Advanced is required — it provides adaptive baseline learning that the open-source version does not. Evaluation needed: does the per-PoP licensing model scale to Datum's PoP count, and what is the cost at full fleet scale?
- **FlowSpec peer coverage.** BGP FlowSpec is only effective where upstream peers support it. What percentage of Datum's upstream transit and peering relationships support FlowSpec? Where it is absent, what is the fallback?
