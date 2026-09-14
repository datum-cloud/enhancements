---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Per-region proxy reachability

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed](#infrastructure-needed)

## Summary

A proxy can look fully ready in Datum while real users still fail. Today's status tells you that configuration was accepted and programmed. It does not tell you whether the app answers through Datum, or whether that answer depends on which region the client hits.

This enhancement makes reachability a product signal. Datum checks each proxy's public hostname from every region we operate, and surfaces an aggregate "is it live?" answer plus a per-region picture in the console. Customers should not need to curl from three airports to learn what we already know from the edge.

## Motivation

People buy a global edge because they expect their app to work from many places. When it does not, they currently find out from their own monitors or from angry end users. Datum shows Accepted, Programmed, certificates ready, DNS programmed. None of those answer "can someone in this region actually use my proxy right now?"

That gap hurts in the cases that matter most:

- The origin is up for the team's home region and down for another.
- A partial deploy, firewall change, or TLS mistake only bites some paths.
- A connector or tunnel is fine while the public hostname path is not (or the reverse).

Competitors already treat this as table stakes. Cloudflare surfaces origin and pool health from many data centers. Fastly exposes backend health. Cloud load balancers run active target checks. Datum can do better than a single "up/down" bit because we run the edge: we can say *where* the path is broken, not only that something is wrong.

### Goals

- Every proxy gets an always-on answer to "is this live through Datum?" without the customer opting in or wiring their own synthetic monitors.
- That answer includes *where*: healthy, degraded, or unreachable per Datum region (rolled up for the console, e.g. by continent).
- The console shows a clear Regions view on the proxy detail page, with a short headline such as "live in all regions" or "live in X of Y".
- Configuration status stays what it is today. Reachability is a separate product signal so "programmed but unreachable" is visible instead of silent.
- Success means a customer can open a proxy and tell, within a short window of a real outage, which regions are affected without leaving Datum.

### Non-Goals

- Replacing Connector tunnel state. Tunnel readiness stays the source of truth for tunnel-backed connectivity.
- Customer-facing alerting, paging, or webhooks in v1. Insights and Activity can consume the signal later.
- Letting customers pick which regions to probe, or run custom multi-protocol checks (gRPC, raw TCP) in v1.
- Composing many checks into Cloudflare-style monitor groups.
- Failover policy configuration. Failover UI and behavior are complementary ([enhancements#573](https://github.com/datum-cloud/enhancements/issues/573)) and ship on their own timeline.
- Turning Envoy's internal active health checks into the customer-visible signal. Those remain a different (and weaker) product story for our use case.

## Proposal

Treat reachability as a default platform capability on every HTTPProxy.

**What the customer experiences**

1. They create or already have a proxy. They do nothing special to enable health signaling.
2. On the proxy detail page, a Regions card shows per-region (or per-continent) status with simple healthy / warning / down treatment, aligned with the product mock attached to the tracking issue.
3. A headline summarizes global state: live everywhere, live in some regions, or not live.
4. When a region is unhealthy, the UI gives enough context to act (for example, last checked time and a short reason such as unexpected status or no response). Deep historical charts can wait; the first job is the current picture.

**What "reachable" means for v1**

- We check the proxy the way a client would: the public hostname path, not a private poke at the origin URL alone. DNS, TLS, edge routing, WAF, and the origin all sit on that path.
- Defaults are boring and safe: HTTP(S) GET to `/`, expect success-class responses, run often enough that a regional outage shows up in the console without the customer refreshing for half an hour.
- Overrides (custom path, expected status) can come later. v1 is valuable with defaults alone.

**How we measure success**

- A known regional origin failure appears as unhealthy for that region in the console without a support ticket.
- Customers stop using "curl from my laptop vs a VPS" as the primary way to debug Datum-side reachability.
- Support can point at the Regions card instead of asking the customer to reproduce from multiple locations.

### User Stories

#### Story 1: Partial regional outage

A builder ships an API behind a Datum proxy. After a firewall change, clients in Europe fail while US traffic looks fine. They open the proxy in the console and see Europe unhealthy and other regions healthy. They fix the origin allowlist for the European path without opening a Datum support ticket first.

#### Story 2: "Is it us or Datum?"

A team gets reports that the site is down. The proxy shows Programmed and certificates ready. The Regions card shows all regions unreachable with failed origin responses. They stop blaming DNS/TLS configuration and fix the origin. If instead all regions were healthy, they would look at client-side or upstream CDN issues sooner.

#### Story 3: Quiet confidence after deploy

After enabling a new region or changing WAF mode, a platform engineer glances at Regions and sees every checked location healthy before telling the app team the change is safe.

### Notes/Constraints/Caveats

- Reachability is not the same as "origin server process is up." A 403 from WAF, a bad cert on the public name, or a DNS miss can all mark the path unhealthy even when the origin process is fine. That is intentional: customers care about the path users take.
- Probes must not become a new denial-of-service surface against customer origins, and they must not fight WAF. How probes pass protection (dedicated allow path, header, or similar) is an open product/engineering question; the customer-visible rule is that enabling protection does not permanently red-light the Regions card.
- Free-tier or abuse-sensitive accounts may need a probe budget. Universal default-on is the product intent; cost caps are a constraint, not a reason to hide the feature behind a toggle for everyone.
- Failover and multi-origin policy should eventually *use* this signal. This enhancement only makes the signal exist and visible.

### Risks and Mitigations

| Risk | Mitigation |
| --- | --- |
| False reds from probe path != real traffic (WAF, bot rules, geo blocks) | Design probes so protected proxies stay measurable; document what "unreachable" means; tune expected status defaults. |
| Customers confuse Programmed with Available | Keep the signals separate in API and UI copy; headline language says "live" / "reachable," not "configured." |
| Probe cost at scale | Bound frequency; share work per region; revisit idle vs busy proxies only if cost forces it. Product default stays on. |
| Alert fatigue if we page on every blip later | v1 is console visibility only; alerting thresholds are a separate decision. |

## Design Details

Product surfaces for v1 (implementation detail deferred to NSO work):

| Surface | Customer meaning |
| --- | --- |
| Aggregate availability | Is this proxy live through Datum overall? |
| Per-region reachability | For each Datum region, healthy / degraded / unreachable, with last checked time |
| Console Regions card | Continent or region rollup matching the product mock; TLS readiness can sit alongside where we already have certificate signals |
| CLI / API consumers | Same status a script or `datumctl` can read without scraping the UI |

<<[UNRESOLVED nso-design]>>
Exact API field names, storage of probe history, worker placement, and WAF allowlisting belong in the network-services-operator design that implements this enhancement. This document locks the product outcome, not the controller shape.
<<[/UNRESOLVED]>>

## Production Readiness Review Questionnaire

To be completed before moving this enhancement to `implementable`. Seed expectations for the review:

- Operators can see whether probing is running and whether results are stale.
- Customers can tell the feature is working from proxy status / the Regions card without reading platform logs.
- Disabling probing (break-glass) must not break proxy data-plane traffic; it only removes or freezes the reachability signal.

### Feature Enablement and Rollback

- [ ] Other
  - Describe the mechanism: platform capability default-on for HTTPProxies; exact gate TBD in implementation.
  - Will enabling / disabling the feature require downtime of the control plane? No expected.
  - Will enabling / disabling the feature require downtime or reprovisioning of a node? No expected.

#### Does enabling the feature change any default behavior?

Yes for visibility: proxies gain reachability status and console Regions content. Data-plane routing behavior does not change solely because probing is enabled.

#### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Rolling back should stop new probe results and clear or freeze customer-visible reachability status without removing the proxy or interrupting traffic.

### Monitoring Requirements

#### How can someone using this feature know that it is working for their instance?

- [ ] API .status
  - Condition name: aggregate availability (name TBD)
  - Other field: per-region reachability entries (shape TBD)
- [ ] Other
  - Details: Regions card on the proxy detail page in Cloud Portal

## Implementation History

- Earlier product ask: [enhancements#658](https://github.com/datum-cloud/enhancements/issues/658) (Proxy Origin Health Check).
- Related failover UI: [enhancements#573](https://github.com/datum-cloud/enhancements/issues/573).
- Implementation tracking: [network-services-operator#332](https://github.com/datum-cloud/network-services-operator/issues/332) (transferred from cloud-portal#1336).
- This document opened as the product-facing enhancement for per-region reachability.

## Drawbacks

- Building and operating synthetic checks from every region is real platform cost for a signal many quiet proxies will rarely need.
- A wrong probe definition (path, headers, WAF interaction) teaches customers the wrong lesson and erodes trust faster than showing nothing.
- Showing "unreachable" when the failure is actually WAF or customer bot rules will generate support load until copy and defaults are sharp.

## Alternatives

- **Customer-owned external monitors only.** Leaves the "is it Datum or me?" question unanswered inside the product, and loses the regional picture we can uniquely provide.
- **Single global check (one place, one bit).** Cheaper, but hides the partial-outage cases that make a global edge valuable.
- **Passive inference from real traffic only.** Fails when the proxy is idle, which is often when someone is debugging a fresh deploy.
- **Envoy-only active health checks as the product signal.** Useful for load balancing internals; a weaker answer for "what would a client in this region see on my hostname?"

## Infrastructure Needed

- Probe execution capacity in each Datum edge region (owned with platform/infra once NSO defines the contract).
- Console work in Cloud Portal for the Regions card once status exists.
- Optional later: Insights / Activity hooks when we choose to notify, not only display.
