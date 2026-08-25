---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Network Services

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [What a consumer writes](#what-a-consumer-writes)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [NetworkService](#networkservice)
  - [Membership](#membership)
  - [Well-known labels](#well-known-labels)
  - [Traffic distribution](#traffic-distribution)
  - [Health](#health)
  - [What a consumer can see](#what-a-consumer-can-see)
  - [Consuming a network service](#consuming-a-network-service)
  - [Federation](#federation)
  - [How nearest is decided](#how-nearest-is-decided)
  - [End to end: a compute workload behind a proxy](#end-to-end-a-compute-workload-behind-a-proxy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)

## Summary

A **NetworkService** is a named set of endpoints spanning every location a consumer runs
in. Consumers select members by label, and an HTTPProxy names the service as its backend.

Anycast already gets each request to the point of presence nearest the user. A
NetworkService covers the rest of the path: the edge serves that request from the members
nearest that point of presence, and moves to the next-nearest location when the closest one
fails.

The consumer configures none of that. They name a label the platform already applies to
their interfaces, and membership follows whatever is actually running — new replicas join
as they come up, and a new location starts serving its own users once it has healthy
members.

This document defines the consumer surface for
[Zava](../traffic-intelligence/envoy-routing-zava.md) feature #1, geo-aware upstream
selection. It also answers the
[Dani Rojas](../traffic-intelligence/l4-load-balancing-dani-rojas.md) open question of how
a load balancer discovers compute backends as instances start and stop.

## Motivation

Nobody runs a multi-location application behind Datum's edge today. Compute is still early,
and the edge started as a reverse proxy to a single public origin. We should decide how
multi-location traffic works before consumers need it.

An HTTPProxy backend is one endpoint URL, but compute can place a workload in several
cities. One URL cannot point at all of them, so the consumer has to load balance across
locations themselves.

That means the consumer enumerates every instance address and keeps the list current as
instances are replaced and locations added. It means deciding which location each point of
presence prefers — a matrix of edges against locations. And it means detecting a failed
location and shifting traffic off it quickly. The first is tedious. The other two are easy
to get wrong and hard to notice being wrong.

It also wastes the edge. Anycast puts the request at the point of presence nearest the
user, then the proxy hauls it to whatever origin the URL names. The platform knows where
the request arrived and where the consumer's capacity is, but a consumer cannot connect
those two facts from outside.

The platform should do this because it already holds the inputs: where instances run, where
points of presence are, and which members are healthy. Solving it once is cheaper than
every consumer solving it separately and re-solving it on every deployment change.

### Goals

- Give consumers one resource meaning "my application's endpoints, wherever they run."
- Track membership automatically as instances appear, disappear, and move.
- Define a well-known label vocabulary that interface-creating services populate, so
  selection and address management work without a consumer inventing conventions.
- Serve each request from the nearest members, failing over to the next-nearest location
  automatically and returning on recovery, with no configuration.
- Report per-location member and health counts so consumers diagnose their own problems.
- Define one membership noun that serves the Layer 7 edge now and Layer 4 later.

### Non-Goals

| Out of scope | Why, and where it belongs |
|---|---|
| Modeling compute in the networking API | A NetworkService selects interfaces. It has no workload, deployment, or instance concept, which keeps it usable for endpoints compute did not create. |
| The Layer 4 load balancer's configuration surface | [Dani Rojas](../traffic-intelligence/l4-load-balancing-dani-rojas.md). This document defines the membership that load balancer consumes, not its policy. |
| Per-request latency-based routing | Steering here is topology- and health-driven and changes at control plane speed. Measured round-trip time is [Zava](../traffic-intelligence/envoy-routing-zava.md) feature #3. |
| Geo authorization | [Roy Kent](../traffic-intelligence/ip-geo-roy-kent.md) and `SecurityPolicy` decide whether to serve a request. This document decides which member serves it. |
| An address of its own | A virtual IP for east-west traffic is a natural extension. The name accommodates it; this milestone does not deliver it. |
| Consumer-controlled steering | `Nearest` is the only strategy value. Weighting, pinning, and explicit failover order are deferred. See [Drawbacks](#drawbacks). |
| Consumer-configured health checks | Health is platform-derived and not configurable, so this milestone cannot detect a member that is reachable but serving errors. |
| Replacing connector-backed origins | Consumers behind a Datum connector keep that path unchanged. |

## Proposal

A consumer creates a `NetworkService` naming a label and a port. The platform resolves that
label into the current endpoint set, records each member's location, and keeps it current.
It handles everything else too: traffic distribution has one behavior, and consumers can
inspect what it did but not change it.

### What a consumer writes

```yaml
apiVersion: networking.datumapis.com/v1alpha
kind: NetworkService
metadata:
  name: storefront
spec:
  networkInterfaces:
    selector:
      matchLabels:
        compute.datumapis.com/workload: storefront
  ports:
    - name: http
      port: 8080
```

Nearest-location routing, automatic failover, and health checking are on by default.

The proxy in front of it:

```yaml
apiVersion: networking.datumapis.com/v1alpha
kind: HTTPProxy
metadata:
  name: storefront
spec:
  hostnames:
    - shop.example.com
  rules:
    - backends:
        - networkService:
            name: storefront
            port: http
```

### User Stories

#### Story 1: Serve users from their nearest location

Priya's storefront runs in Dallas and San Jose. She creates the NetworkService above,
selecting the label compute already applies to her workload's interfaces.

A shopper in Atlanta reaches Datum's Ashburn point of presence and is served from Dallas;
a shopper in Portland reaches Seattle and is served from San Jose. Priya configured neither
route.

#### Story 2: Change capacity without editing anything

Priya scales Dallas from two replicas to twenty for Black Friday. The new interfaces carry
the same label, join the service, and take traffic as they become healthy. She scales back
down on Saturday. Later she adds Frankfurt, and European users start being served there.

None of this requires an edit to the NetworkService or the HTTPProxy.

#### Story 3: Survive a location failure at 3am

Dallas loses power. Health checks fail and its traffic moves to San Jose within seconds.
Users see higher latency, not errors. Dallas resumes serving its own region when it
returns. Priya reads about it in the morning.

#### Story 4: Diagnose a report of slowness

A user reports slowness from Berlin. Priya asks the platform what the service looks like
and gets per-location member and health counts, which tell her whether Frankfurt is in
rotation.

### Notes/Constraints/Caveats

**Consumers never write a member's location.** The platform reads it from where the member
runs. That is why the resource carries no location list, and why it stays correct when
capacity moves.

**Membership is a query over network interfaces.** Interfaces are networking's own
resource, so anything holding one can be a member: a compute instance today, a load
balancer or appliance later.

**Consumers label nothing.** Interface-creating services stamp a defined set of keys, so
the facts worth selecting on are already present and spelled the same way for everyone. See
[Well-known labels](#well-known-labels).

<<[UNRESOLVED labels ]>>
This document proposes the vocabulary but cannot assign the work. Each owning service must
commit to stamping its keys — the compute keys especially. Free-form consumer labels
alongside the well-known set are a separate, later question.
<<[/UNRESOLVED]>>

**The edge balances across individual members.** Layer 7 load balancing needs the proxy to
see each one: to eject a bad instance while its siblings keep serving, to retry a failed
request elsewhere, and to prefer a nearby location.

If a NetworkService later gains an address, the edge still routes to members directly.
Behavior attached to that address applies to clients that dial it, not to edge traffic.

**Nearest is computed from topology and geography.** Peering and cable paths mean the
geographically closest location is not always the fastest. Measured signals replace the
approximation later, through the
[Zava](../traffic-intelligence/envoy-routing-zava.md) latency workstream, and change no
API.

<<[UNRESOLVED reachability ]>>
What the edge dials to reach a member — a public address on the instance, or an address
routed over the platform fabric — remains unsettled and changes the product. Public
backhaul works today and crosses the internet. Fabric backhaul keeps traffic on Datum's
network end to end but depends on the underlay work. Settle this before consumers see the
feature, because it determines what a consumer must expose.
<<[/UNRESOLVED]>>

### Risks and Mitigations

**An over-broad selector sends production traffic to the wrong endpoints.** Per-location
status makes resolved membership visible, so a wrong selector shows up as a member count
that does not match expectations. Validation should also reject a selector matching
nothing.

**Failover moves traffic across a jurisdiction boundary.** There is no mitigation in this
milestone, because there is no control for declaring that spilling to another location is
unacceptable. Consumers with hard residency obligations should not use a NetworkService
yet, and the product surface must say so rather than let them discover it during a
failover. The durable answer is a sovereignty constraint in
[Total Load Balancing](../traffic-intelligence/total-load-balancing.md).

**Failover is gradual.** A degrading location bleeds a growing share of traffic to its
neighbor instead of switching at a threshold. That is correct for a location 30% down and
surprising to anyone expecting binary failover. Document it and make it legible in
per-location status.

**Membership can be stale while looking current.** Endpoint facts cross several control
planes, and a break anywhere freezes membership while traffic continues normally. Report
when membership was last confirmed rather than presenting it as live. See
[Federation](#federation).

**Members may be unreachable from the edge.** A consumer sees a healthy-looking resource
and broken traffic. Reachability must appear in the service's own status, not only in proxy
error rates.

**Security review** must cover selector scoping across namespace and project boundaries and
confirm that a consumer cannot select interfaces they do not own.

## Design Details

This section describes consumer-visible shape and behavior. A follow-on technical design
owned by the network services operator covers how the edge gets programmed.

### NetworkService

```yaml
apiVersion: networking.datumapis.com/v1alpha
kind: NetworkService
metadata:
  name: storefront
spec:
  # Which endpoints belong to this service.
  networkInterfaces:
    selector:
      matchLabels:
        compute.datumapis.com/workload: storefront

  # Ports this service exposes. Named, so consumers reference a name.
  ports:
    - name: http
      port: 8080
      protocol: TCP

  # Optional. Defaults to Nearest, the only value this milestone accepts.
  trafficDistribution:
    strategy: Nearest
```

### Membership

Membership is the set of interfaces matching the selector, resolved continuously. An
interface joins when it matches and is reachable, and leaves when it stops matching or goes
away.

Selectors are scoped to the consumer's own resources; selecting another project's
interfaces is not possible.

A NetworkService is written in one control plane and its members live in others.
[Federation](#federation) covers how the facts travel.

### Well-known labels

Every network interface carries a defined set of labels applied by whichever service
created it. Consumers select on these labels and create none of them.

Networking applies these to every interface, whatever created it:

| Label | Value | Example |
|---|---|---|
| `topology.datum.net/city-code` | The city holding the interface. Already the platform's well-known topology key. | `DFW` |
| `networking.datumapis.com/network` | The network the interface attaches to. | `default` |

Compute applies these to interfaces it requests:

| Label | Value | Example |
|---|---|---|
| `compute.datumapis.com/workload` | The workload owning the interface. | `storefront` |
| `compute.datumapis.com/placement` | The placement within that workload. | `americas` |
| `compute.datumapis.com/instance` | The instance holding the interface. | `storefront-americas-dfw-0` |

Two properties follow. **Networking never interprets a compute key.** It matches values as
opaque strings, exactly as a Kubernetes Service matches pods without knowing what a
Deployment is. A key's prefix records which service guarantees it, not which service
understands it. **Any service can populate its own keys** under its own prefix without
changing networking, which keeps the model open to endpoints no workload produced.

Selecting a whole application is the common case, as shown above. Narrower selections add
keys — one placement, or one city:

```yaml
networkInterfaces:
  selector:
    matchLabels:
      compute.datumapis.com/workload: storefront
      topology.datum.net/city-code: DFW
```

Adding the city key restricts which interfaces are members; it does not steer traffic.
Location steering is automatic and never appears in a selector. Restricting a service to
Dallas capacity happens here; getting Dallas users served from Dallas requires nothing.

**Managing addresses.** Defining these labels centrally pays off beyond load balancing.
"Which addresses does this application hold," "which does it hold in Frankfurt," and "which
will this instance keep if it is replaced" all become the same query against the same
labels.

### Traffic distribution

`spec.trafficDistribution.strategy` selects how traffic spreads across a service's
locations. It defaults to `Nearest`, and `Nearest` is the only value this milestone accepts.

| Strategy | Behavior |
|---|---|
| `Nearest` (default, only value today) | Serve each request from the members nearest where it entered the network. Shift to the next-nearest as health degrades, and return on recovery. |

Within a location, the edge balances requests across that location's members per request.

Carrying a field with one value is deliberate. Nearest-with-failover is what nearly every
consumer wants, so it is worth getting right before anything else. We expect to add more
strategies, and a consumer who sets this explicitly today keeps working when we do. The
cost is that a single-value enum reads like an unfinished API.

### Health

The platform assesses health using a check derived from the port's protocol. Consumers do
not configure it; the resource carries no health check field.

That check proves a member is reachable and accepting connections, not that the
application answers correctly. **A member serving errors on every request still looks
healthy, keeps its share of traffic, and never triggers failover.** Until application-level
checks exist, this milestone cannot detect a process that is up but broken.

The edge removes failing members from rotation and restores them automatically, within
seconds. This is deliberately independent of the platform-wide health signals
[Nate](../traffic-intelligence/health-checks-nate.md) publishes, which operate on a
different timescale for a different purpose. The two are complementary, as they are for the
Layer 4 load balancer.

### What a consumer can see

```console
$ datumctl get networkservice storefront
NAME         LOCATIONS   MEMBERS   HEALTHY   AGE
storefront   3           24        22        18d

$ datumctl describe networkservice storefront
Locations:
  dfw   members 20   healthy 18   serving
  sjc   members  2   healthy  2   serving
  fra   members  2   healthy  2   serving
Conditions:
  Ready               True
  MembersResolved     True    24 interfaces matched
  EndpointsReachable  True
```

Per-location counts are the primary diagnostic. A consumer must be able to answer "which
location serves my users, and is any location out of rotation" without contacting support.

### Consuming a network service

An HTTPProxy rule reaches a service through a `networkService` backend naming the service
and one of its ports. This sits alongside the existing backend forms — an endpoint URL, and
an endpoint reached through a connector.
Referencing by name rather than by API group and kind keeps HTTPProxy backends a curated
set; see [Alternatives](#alternatives). Naming a port rather than a number lets the
reference survive a port change.

The same resource is intended to back a Layer 4 load balancer when
[Dani Rojas](../traffic-intelligence/l4-load-balancing-dani-rojas.md) becomes
customer-configurable. That is why ports carry a protocol and the resource is not named for
HTTP.

### Federation

A consumer writes a NetworkService in their project. Its members are interfaces in POP
cells, which are separate clusters from the edge clusters serving traffic. This section
states which control plane holds what, and what happens when one becomes unreachable.

**Each control plane holds the minimum it needs.** A POP cell holds full-fidelity
interfaces because providers there configure real NICs. The edge holds addresses, ports,
cities, and health, because that is all a proxy needs to pick a member. Copying whole
interfaces everywhere would couple the edge to allocation details it never uses.

| Control plane | Holds | Written by |
|---|---|---|
| Project | `NetworkService` intent, resolved membership in status | Consumer writes spec; networking writes status |
| POP cell | `NetworkInterface` and its claim — addresses, gateway, MTU, attachment | Networking, fulfilling compute's claim |
| Karmada | Endpoint projections written back from each POP cell | Networking in the POP cell |
| Control plane cell | Nothing durable; aggregates projections into the project | Networking |
| Edge clusters | The resolved endpoint set per service | Networking |

**The endpoint projection.** When an interface becomes usable, its POP cell publishes a
small record — address, city, well-known labels, and programmed state — rather than the
interface itself. This reuses the write-back-and-project pattern compute already uses for
`Instance`, described in
[Federated Deployment Scheduling](https://github.com/datum-cloud/compute/blob/main/docs/enhancements/federated-deployment-scheduling.md),
including the `ns-<uid>` namespace mapping and `meta.datumapis.com/*` label tracking. It
introduces no new distribution mechanism.

**Labels travel with the projection.** Selection runs against the projected record, so the
well-known labels are part of it rather than re-derived downstream. A POP cell knows its own
city, so producing the city label needs no lookup.

**An endpoint joins when it can serve, not when it is allocated.** An interface holds an
address before its data plane is programmed and before its instance runs. Publishing on
allocation would put members into rotation that cannot carry traffic, so the projection
waits for programming. This is why the interface design keeps allocation and programming as
separate conditions.

#### What happens when a control plane is unreachable

| Failure | Effect | Why this is correct |
|---|---|---|
| Karmada unavailable | Membership freezes. Programmed endpoints keep serving; joins and departures wait. | Fail-static. A federation outage must not change what traffic does. |
| POP cell unreachable, capacity healthy | Its members stay in rotation. | The alternative withdraws healthy capacity because a control plane became unreachable, turning a management outage into a customer outage. |
| POP cell unreachable, capacity down | Edge health checks eject the members within seconds. | Liveness is the edge's job. |
| Project control plane unavailable | Services cannot be created or edited; existing ones keep serving. | Consumers lose the ability to change things, not the ability to serve traffic. |
| Edge cluster isolated | That edge serves its last known configuration. | Anycast withdraws an edge that is badly broken. One that is merely stale still serves users correctly. |

**Control plane reachability is never treated as data plane health.** These are independent
signals about different things, and conflating them turns routine control plane incidents
into traffic incidents. Membership answers which members exist. Only edge health checking
answers which members work.

<<[UNRESOLVED staleness ]>>
Fail-static is right for a short outage and wrong for an indefinite one. Membership frozen
for a week is not a useful answer. Decide whether a staleness bound exists and what it does
when crossed. Note that the obvious answer — expiring stale members — reintroduces the
failure the table above rejects.
<<[/UNRESOLVED]>>

### How nearest is decided

Each point of presence ranks the service's locations by proximity to itself, prefers its own
top-ranked location, and treats the rest as an ordered fallback. Because every point of
presence ranks for itself, one NetworkService produces correct local behavior everywhere.
The consumer never expresses a matrix of locations against points of presence, which is what
makes hand-rolled multi-region routing miserable.

Ranking starts from platform topology and geographic coordinates. Measured signals replace
them as the [Total Load Balancing](../traffic-intelligence/total-load-balancing.md) signal
set matures. That substitution is invisible in this API.

### End to end: a compute workload behind a proxy

Everything a consumer writes to put an application in two cities behind a hostname, with
nearest-location routing and automatic failover.

**1. The workload.** An unmodified compute workload. Nothing here exists because of this
proposal; compute applies the labels the service selects on without being asked.

```yaml
apiVersion: compute.datumapis.com/v1alpha
kind: Workload
metadata:
  name: storefront
spec:
  placements:
    - name: americas
      cityCodes:
        - DFW
        - SJC
      scaleSettings:
        minReplicas: 2
        instanceManagementPolicy: OrderedReady
  template:
    spec:
      networkInterfaces:
        - network:
            name: default
      runtime:
        resources:
          instanceType: datumcloud/d1-standard-2
        sandbox:
          containers:
            - name: app
              image: ghcr.io/acme/storefront:1.4.2
              ports:
                - name: http
                  port: 8080
                  protocol: TCP
```

**2. The network service.** Names the label to match and the port to reach.

```yaml
apiVersion: networking.datumapis.com/v1alpha
kind: NetworkService
metadata:
  name: storefront
spec:
  networkInterfaces:
    selector:
      matchLabels:
        compute.datumapis.com/workload: storefront
  ports:
    - name: http
      port: 8080
      protocol: TCP
```

**3. The proxy.** Terminates the hostname and forwards to the service.

```yaml
apiVersion: networking.datumapis.com/v1alpha
kind: HTTPProxy
metadata:
  name: storefront
spec:
  hostnames:
    - shop.example.com
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backends:
        - networkService:
            name: storefront
            port: http
```

**What follows without being written.** Four instances come up, two per city. Compute
requests an interface for each and labels it `compute.datumapis.com/workload: storefront`;
networking labels it with its city. All four join the service. A request for
`shop.example.com` from Atlanta is served from Dallas; the same request from Portland is
served from San Jose. Adding `FRA` to `cityCodes` serves European users from Frankfurt,
changing neither networking resource. Losing Dallas moves its traffic to San Jose and
returns it on recovery.

The consumer wrote no label, and neither networking resource mentions a city, an address,
or a workload.

This assumes compute applies the keys in [Well-known labels](#well-known-labels), which is
the open commitment noted there.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

#### How can this feature be enabled / disabled in a live cluster?

- [x] Other
  - Describe the mechanism: The feature is additive and in use only where a consumer
    created a `NetworkService` and referenced it. Disabling means not serving the new
    resource type. Endpoint-URL and connector backends are untouched.
  - Will enabling / disabling the feature require downtime of the control plane? No.
  - Will enabling / disabling the feature require downtime or reprovisioning of a node? No.

#### Does enabling the feature change any default behavior?

No. Existing HTTPProxy backends behave as before. Consumers opt in by writing a new
resource.

#### Can the feature be disabled once it has been enabled?

Yes, with consumer impact. A proxy backed by a NetworkService has no fallback backend, so
those proxies stop serving. Rollback is a consumer-visible operation.

#### What happens if we reenable the feature if it was previously rolled back?

Membership re-resolves from current state rather than a stored list, so re-enablement
converges on reality.

### Monitoring Requirements

#### How can an operator determine if the feature is in use by workloads?

Count `NetworkService` resources, and HTTPProxy rules referencing one.

#### How can someone using this feature know that it is working for their instance?

- [x] API .status
  - Condition name: `Ready`, `MembersResolved`, `EndpointsReachable`
  - Other field: per-location member and healthy counts

#### What are the reasonable SLOs for the enhancement?

<<[UNRESOLVED slos ]>>
Two numbers need agreement and neither should be guessed: how quickly a membership change
reaches the edge, and how quickly a health change moves traffic. Both are end-to-end across
the control plane and need measurement in the prod-fidelity environment first.
<<[/UNRESOLVED]>>

#### What are the SLIs an operator can use to determine the health of the service?

- [x] Metrics
  - Metric name: membership resolution latency; per-location healthy member count; edge
    requests by serving location; failover events per service
  - Components exposing the metric: network services operator, edge proxy fleet

### Dependencies

- Network interfaces as a selectable, labeled resource
  - Usage description: the unit of membership.
    - Impact of its outage on the feature: membership cannot change; programmed endpoints
      keep serving.
    - Impact of its degraded performance: membership goes stale, delaying scale-up and
      scale-down.
- Karmada federation
  - Usage description: carries endpoint projections toward the project and the resolved
    endpoint set out to edge clusters. See [Federation](#federation).
    - Impact of its outage on the feature: membership freezes, deliberately. Programmed
      endpoints keep serving.
    - Impact of its degraded performance or high-error rates: membership lags reality, so
      scaled-up members wait for traffic and scaled-down members linger. Edge health
      checking bounds the harm of the latter.
- Compute, for the well-known labels it applies
  - Usage description: `compute.datumapis.com/workload` and its siblings are what most
    consumers select on.
    - Impact of its outage: existing interfaces keep their labels.
    - Impact of its degraded performance: none on already-labeled interfaces.
- Location topology and coordinates
  - Usage description: ranking locations by proximity for each point of presence.
    - Impact of its outage: existing ranking continues to apply.
    - Impact of its degraded performance: negligible. This data changes rarely.
- Edge proxy fleet
  - Usage description: performs load balancing and health checking.
    - Impact of its outage: traffic is not served. This is the existing edge failure mode.

### Scalability

#### Will enabling / using this feature result in any new API calls?

Yes. Membership resolution watches network interfaces, which the operator did not do
before. Volume scales with total member count, not request volume.

#### Will enabling / using this feature result in introducing new API types?

Yes: `NetworkService`, namespaced. Expect one to a few per application per project.

#### Will enabling / using this feature result in increasing size or count of the existing API objects?

Yes. Edge endpoint configuration grows from one endpoint per proxy rule to one per member.
Size against a consumer with many replicas across many locations.

#### Will enabling / using this feature result in non-negligible increase of resource usage in any components?

<<[UNRESOLVED scale ]>>
Prior edge scale work established that control plane translation, not the extension path,
limits throughput. Endpoint fan-out grows differently from the gateway-count growth already
measured and needs its own measurement in the prod-fidelity environment before limits are
published.
<<[/UNRESOLVED]>>

## Implementation History

- 2026-08-25: Initial draft.

## Drawbacks

**Consumers running in one location gain nothing**, and now have two ways to express a
backend instead of one. The simple form is just a selector and a port, though, and most
consumers already know the concept from Kubernetes Services or cloud target groups.

**A selector that stops matching produces no error.** Traffic just goes missing, where an
explicit list would at least fail visibly. This is the price of membership tracking
reality, and it puts a lot of weight on the status surface being right.

**Consumers whose needs differ from the defaults have no recourse.** With one strategy
value and no health check configuration, canarying a location, draining one for maintenance,
holding traffic inside a jurisdiction, and ejecting a member that is up but broken are all
unavailable. In each case the only workaround is to stop using a NetworkService altogether,
so a consumer who needs one of these is blocked.

**Reachability-only health will cause a bad first impression.** Consumers read "automatic
failover" as covering a broken application. It covers an unreachable one. A consumer whose
instance returns 500s sees no failover and concludes the feature is broken. The product
surface must be precise about what healthy means.

**Nearest-location routing depends on the platform knowing where capacity is.** If that
data is wrong, routing is wrong in a way that looks fine from the outside, and that is much
harder to debug than an outright failure.

**Deferring the address may prove expensive.** The name leaves room for one. If east-west
addressing wants a different shape, the name will have promised something the resource does
not deliver.

## Alternatives

**Point the proxy at a workload.** Simpler to name, but it puts a compute concept in the
networking API, couples the two services' release cycles, and offers nothing for endpoints
compute did not create. Selecting interfaces delivers the same experience without the
coupling.

**Let consumers list endpoints with per-location weights.** Maximum control, no new
selection concept. Rejected because it hands the consumer exactly the work this proposal
exists to absorb: a list that goes stale on the first capacity change, and a weight matrix
maintained against every point of presence.

**Solve it in DNS.** Rejected as the wrong layer. It fails over at cache speed rather than
connection speed and discards the edge's knowledge of where the request entered. DNS
steering remains valuable as [Jamie Tartt](../traffic-intelligence/gslb-jamie-tartt.md) — a
complement, not a substitute.

**Extend the existing inline backend forms with location fields.** Smallest change.
Rejected because an inline backend cannot be shared across rules or proxies, inspected on
its own, or report per-location health. Consumers reason about the endpoint set, so it
should be a resource they can inspect.

**Reference the service with a Gateway API backend reference.**
`BackendObjectReference` accepts any group and kind, and this is how the reference is
realized internally, requiring no Gateway API extension. Rejected for the consumer surface
on two grounds. HTTPProxy exists because Gateway API's shape is more than most consumers
need, and putting a group and kind on its most-edited field reintroduces what the type
hides. And an open reference cannot be validated: any group and kind pair parses, so an
unsupported backend fails later as a status condition instead of immediately as a rejected
write. Each future backend kind therefore costs an HTTPProxy API change — the same trade
already made for connector-backed endpoints.

**Name the resource `Backend` or `EndpointGroup`.** `Backend` collides with an existing type
in the edge proxy stack that means something else, and it describes where something sits in
a request path rather than what it is. `EndpointGroup` describes membership accurately but
forecloses the address this resource may grow.
