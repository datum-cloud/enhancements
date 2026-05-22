# The Roy Kent Project

**Parent:** [Total Load Balancing](total-load-balancing.md)  
**Status:** In progress  
**Codename:** Internal project name — not a go-to-market product name.

---

The first Total Load Balancing projects will focus on making geo data broadly available and reusable across the platform. DNS will consume it for Global Server Load Balancing (GSLB). Envoy will use it for ACL enforcement and endpoint load balancing decisions. UFO workloads will leverage it for localization, sovereignty awareness, and regional application behavior. Internally, we think of this as the "Roy Kent Project," inspired by Roy Kent:

> "He's here, he's there, he's every f@#%ing where."

Traffic Intelligence should behave the same way — present wherever decisions are being made across the network.

---

## Scope

The Roy Kent Project scopes to geography only. The goal is to make IP-to-geo data reliably available to every Datum system that needs it, and to build the operational foundation — database, update process, distribution — before layering in richer signals.

---

## Roadmap

Integration is delivered in six stages. The sequence is deliberate — each stage validates the GeoDB and distribution pipeline before the next stage relies on them for higher-stakes decisions.

| Stage | Integration | Rationale |
|---|---|---|
| 1 | **DNS / GSLB** | Lowest blast radius — a bad geo answer routes suboptimally for one TTL then self-heals. Validates GeoDB accuracy in production with no risk of locking users out. Most immediately visible feature to customers. |
| 2 | **ACLs (Geo Blocking + Named IP Lists)** | Security-sensitive — a bad geo entry can block legitimate traffic. By this stage the GeoDB is proven against real traffic from Stage 1. |
| 3 | **Application Load Balancing** | More complex routing logic. Builds on a distribution pipeline and DB already proven in production. |
| 4 | **Metrics and Logs Enrichment** | Geo-enriched telemetry. Once the DB is proven across the first three stages, enriching logs and metrics is low risk and high value for observability. |
| 5 | **Galactic VPC** | Distance-based PoP ranking for latency modeling. Depends on accurate lat/lon data validated in earlier stages. |
| 6 | **UFO Compute** | Most complex integration — Agent Router, AI Router, Tetrate Proxy, WAF. Benefits from all prior learnings. |

Rollout goes directly to production — no separate internal-only phase. Platform is early enough that the risk is manageable and speed of learning is more valuable than caution.

---

## GeoDB

A GeoDB maps IP addresses and prefixes to geographic and network attributes. At minimum it needs to resolve:

| Attribute | Used For |
|---|---|
| Country / Region / City | Sovereignty enforcement, geo blocking, GSLB proximity |
| Latitude / Longitude | Distance calculations for latency modeling |
| ASN + Carrier Name | Enrichment, risk context, routing preference |
| IP type (residential, datacenter, proxy, VPN, satellite) | Fraud / bot decisions, ALB policy |

**Update process:** IP-to-geo mappings shift constantly as prefixes are reassigned and carriers restructure. The DB must be refreshed on a defined schedule with a tested rollout process — a bad update can flip geo decisions globally.

**GeoDB Source — status: vendor eval required**

No vendor is selected. [ip66.dev](https://ip66.dev/) has been identified as a candidate but its dataset coverage is limited and needs validation. A formal evaluation is required before committing to a source.

The evaluation should assess vendors across:

| Criteria | Notes |
|---|---|
| Dataset coverage | % of global IP space mapped; accuracy by region |
| Attribute completeness | Country, city, lat/lon, ASN, IP type — which fields are present and reliable |
| Update frequency | How often the feed refreshes; how quickly real-world changes are reflected |
| Distribution format | MMDB, CSV, API — must fit our edge distribution model |
| Licensing terms | Can the data be embedded in edge nodes? Restrictions on redistribution? |
| Pricing model | Per-query, flat fee, volume tiers |
| SLA and support | Uptime guarantees, correction process for bad data |

**Issue [#732](https://github.com/datum-cloud/enhancements/issues/732):** Run GeoDB vendor evaluation against the criteria above. Produce a comparison matrix and a recommendation. Block distribution architecture decisions on this outcome. See [geodb-vendor-eval.md](geodb-vendor-eval.md) for the full vendor list and scoring matrix.

---

## Customer-Managed Named IP Lists

Alongside the authoritative GeoDB, customers need the ability to define their own named lists of IP addresses and CIDR ranges and have those lists distributed to the edge. This is customer intent — "these IPs are my office network", "these are trusted partner ranges", "block these addresses" — that no commercial geo feed will know about.

Named lists are referenced by name in policies rather than by raw CIDR, so rules stay readable and the underlying IPs can be updated without touching policy definitions.

**Example lists a customer might create:**

| List Name | Contents | Used For |
|---|---|---|
| `trusted-offices` | Corporate office egress IPs | Bypass CAPTCHA, relaxed WAF rules |
| `partner-networks` | Known partner AS ranges | Allow access to restricted APIs |
| `block-list` | Identified abusive IPs/ranges | Geo block equivalent, manually curated |
| `cdn-origins` | Upstream CDN egress IPs | Skip bot detection for known crawlers |
| `internal-monitors` | Synthetic monitoring agents | Exclude from analytics, suppress alerts |

---

**Ownership model**

Each list has a single owner — either the customer or Datum. Never both.

| Ownership | Example | Who manages | Access model |
|---|---|---|---|
| Customer-owned | "GP offices in the south east" — the customer's own egress IPs | Customer via self-serve API and portal | Customer has full CRUD access |
| Datum-owned | "Known bad actors from Canada" — threat intelligence curated by Datum | Datum | Customer has read access only; changes go through Datum |

Ownership determines who can edit a list — it does not determine precedence. IP and geo rules from all sources are evaluated together as a single ordered rule list. First match wins. Position in the list is what controls behavior.

---

**Quota limits**

List distribution carries real infrastructure cost — storage, replication, and lookup at every PoP. Each tenant has quota limits on the number of lists they can create and the size of each list. Quota limits are TBD and will be defined during platform capacity planning.

---

**Propagation SLA — near real-time**

List changes must propagate to all edge PoPs in near real-time. ACLs are frequently modified under active stress — an ongoing attack, a misconfigured partner range, a compromised egress IP. The time between a customer hitting save and that change being enforced at the edge must be measured in seconds, not minutes.

This is a hard requirement that drives the distribution architecture: list updates cannot rely on a scheduled poll cycle. Changes must be pushed to edge nodes immediately on commit.

---

**Management requirements**

- Create, update, and delete lists via API and UI
- Lists are versioned — a bad update can be rolled back
- Near-real-time push propagation to all PoPs on change
- Maximum list size and total list count enforced by quota
- Lists can reference individual IPs, CIDR blocks, ASNs, and ISPs

---

**Relationship to GeoDB**

The GeoDB and named lists are complementary but distinct:

| | GeoDB | Named IP Lists |
|---|---|---|
| Source | Commercial feed + authoritative registries | Customer or Datum defined |
| Updates | Vendor schedule (daily) | On-demand, near real-time propagation |
| Content | Geographic + network attributes | Operational intent (trust, block, group) |
| Consumers | All edge components | Policies that reference them by name |

Both are distributed through the same replication pipeline to PoP-local stores, so edge components make a single lookup regardless of whether the match comes from geo data or a customer list. Named list updates require a push path that bypasses the scheduled GeoDB refresh cycle.

**Decisions:**

- **Scope:** Lists are global per tenant — a list applies across all PoPs and is not scoped to a specific location or service.
- **Entry types:** Lists support individual IPs, CIDR blocks, ASNs, and ISPs. ASN support is confirmed; ISP-based entries to be defined during implementation.
- **Cross-product:** Lists are shared across all products — a list defined once is available to WAF, ALB, DNS, and any other consumer. No per-product duplication.

---

## Consumers

#### 1. DNS — Global Server Load Balancing (GSLB)

DNS is the first steering point for user traffic. With geo data, DNS can return the anycast address of the PoP geographically closest to the end user — or the PoP in the correct sovereignty zone.

Datum runs PowerDNS. The GSLB design should be based on what PowerDNS supports natively — not invented from scratch. Research required before this section can be fully defined. See issue [#733](https://github.com/datum-cloud/enhancements/issues/733).

- Input: end-user IP **or** end-user resolver IP (EDNS Client Subnet where available; resolver IP as fallback)
- Output: DNS answer biased toward the correct region/PoP
- Key consideration: resolver IP is often not the user IP — a user in Munich using a Google resolver (in the US) will get a wrong answer without EDNS Client Subnet handling

#### 2. Delivery Edge Proxy — Geo Blocking

The edge proxy enforces access rules combining IP and geo signals. Rules are evaluated as a single ordered list — first match wins, exactly like a traditional firewall ACL. Datum's role is to provide the capability and help customers configure it the way they want it to run. Compliance, sanctions enforcement, and regulatory obligations are the customer's responsibility to configure — not the platform's to impose.

- Input: connecting client IP → matched against the ordered rule list (named IP lists + geo rules combined)
- Output: allow / block decision per request based on first matching rule
- Key consideration: needs low-latency in-process lookup (not a remote call); the full rule list must be local to the proxy

**Rule ordering model:**

```
Rule 1:  ALLOW  ip-list:trusted-offices          → matches first, traffic allowed, stop
Rule 2:  BLOCK  country:RU                        → only reached if Rule 1 did not match
Rule 3:  BLOCK  ip-list:known-bad-actors          → only reached if Rules 1-2 did not match
Rule N:  ALLOW  any                               → default — last rule defines the default action
```

IP rules and geo rules intermingle freely in the list. The entire list is customer-orderable — there are no Datum-pinned positions. The last rule in the list defines the default action (allow all or deny all) and is set by the customer.

Datum provides a named list of its own infrastructure IPs (`datum-infrastructure`) that customers can reference in their rules. Customers can place this list anywhere in their rule order, or omit it entirely. Placing a BLOCK rule above it will deny Datum infrastructure traffic and may disrupt the customer's own service — but that is the customer's choice to make. The platform does not prevent configurations that may cause self-inflicted problems. Customers are expected to understand what they are configuring.

#### 3. Delivery Edge — Application Load Balancing (ALB)

The ALB uses geo to make L7 routing decisions: route a user to the right origin cluster, apply geo-specific rate limits, or steer to a localized version of an application.

- Input: client IP geo + request metadata
- Output: upstream selection, header injection (e.g. `X-Client-Country`, `X-Client-Region`)
- Key consideration: header injection here feeds downstream apps so they don't each need their own geo lookup

#### 4. Metrics and Logs — Enrichment

Every flow log and metric event gets geo attributes attached at ingest time so that dashboards, alerts, and anomaly detection can slice by region, country, or ASN without a join at query time.

- Input: source/destination IP from raw log events
- Output: enriched log record with country, city, ASN, IP type fields appended
- Key consideration: enrichment must happen close to ingest to avoid hot joins; a stale DB here produces incorrect historical attribution

#### 5. UFO (Unikraft Compute) — Application Geo Data

Applications deployed on UFO unikernel compute may need geo context at runtime — to localize content, enforce per-user rules, or make their own routing decisions. The GeoDB (or a lightweight API in front of it) must be reachable from the UFO execution environment.

- Input: IP passed by the application at runtime
- Output: geo record returned via API or shared memory
- Key consideration: latency sensitivity varies by app — some can tolerate a sidecar call, others need the data pre-injected in the request context

#### 6. Galactic VPC — Latency and Distance Mapping

Galactic VPC uses geographic coordinates (lat/lon of the client and candidate PoPs) as a starting signal for mapping latency and routing distance. Distance is a proxy for RTT until real measurement data is available in a later project.

- Input: client IP → lat/lon; PoP coordinates (static config)
- Output: ranked list of PoPs by estimated distance
- Key consideration: distance ≠ latency (undersea cables, IX routing, peering agreements all distort it) — this is an approximation until RTT signals are available

#### 7. Agent Router, AI Router, Tetrate Proxy, WAF

These components are hosted in UFO Compute. They each need geo context to enforce tenant policy, make model locality decisions, and apply WAF rules that vary by jurisdiction.

- Agent Router / AI Router: use country + ASN to select the appropriate inference endpoint (sovereignty-compliant, low-latency)
- Tetrate Proxy: injects geo headers into the service mesh so downstream microservices receive client context
- WAF: applies geo-specific rule sets (e.g. stricter rules for high-risk countries, different bot thresholds by region)

---

## Distribution Architecture

All seven consumers above need geo data, but they have different access patterns:

| Consumer | Latency need | Update tolerance | Deployment |
|---|---|---|---|
| DNS (GSLB) | sub-ms | daily | PoP-local |
| Edge Proxy (geo block) | sub-ms | hours | PoP-local |
| ALB | sub-ms | hours | PoP-local |
| Metrics enrichment | low (async) | daily | ingest pipeline |
| UFO apps | low-ms | hours | UFO runtime |
| Galactic VPC | low-ms | daily | control plane |
| Agent / AI Router, Tetrate, WAF | low-ms | hours | UFO Compute |

This suggests a **pub/sub distribution model** using [Higgins Bus](higgins-bus.md) (MOQT) as the underlying transport. One canonical store holds the authoritative GeoDB and named lists. When either changes, an event is published to a MoQ track. PoP-local subscribers receive the event immediately — no polling, no scheduled push cycle. Consumers that can't embed the DB directly subscribe to a lightweight query API that sits in front of the local replica.

**Named IP Lists** map directly to the MoQ object model. Each list is a track (`org/{org-id}/lists/{list-id}`), and every update is a new object with a monotonic sequence number and a TTL. Edge nodes subscribe to relevant tracks and receive changes within QUIC's latency budget — typically sub-second from commit to enforcement. This satisfies the near-real-time propagation requirement without a bespoke push system.

**GeoDB distribution** uses a hybrid model. The full database is large (potentially hundreds of MB) and updates on a vendor schedule — MOQT is not optimized for bulk transfer. Initial loads and daily snapshots are pulled from object storage as versioned artifacts. A dedicated MoQ track (`platform/geodb/version`) publishes a new object whenever a snapshot is ready, carrying the version identifier and snapshot URL. PoP nodes subscribe to this track and pull the artifact on notification. Incremental delta patches, where available from the vendor, can flow as MOQT objects directly.

This model handles both update cadences cleanly: GeoDB publishes once daily when a new snapshot is ready; named list changes publish on every write for near-real-time propagation. A new PoP or new consumer subscribes and receives updates without any change to the publishing side.

See [Higgins Bus](higgins-bus.md) for the full transport design, track namespace, relay infrastructure requirements, and failure handling.

- [ ] Define the canonical store format (MMDB, SQLite, custom binary?)
- [ ] Define the replication mechanism and health checks
- [ ] Define what happens when a PoP replica is stale or unreachable (fail open vs. fail closed)
- [ ] Define MoQ track namespace — see [Higgins Bus](higgins-bus.md)
