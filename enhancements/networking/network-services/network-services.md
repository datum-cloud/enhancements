--- status: provisional stage: alpha latest-milestone: "v0.x" ---

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
  - [Draining](#draining)
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
in. Consumers select members by label; an HTTPProxy names the service as its backend.

Anycast brings each request to the closest edge. A NetworkService covers the rest: that
edge ranks the service's locations by distance from itself, serves the request from the
best one, and moves to the next if it fails. Consumers configure none of it.

Compute is the first consumer. Others follow. A NetworkService selects network interface
interfaces, which belong to the networking API. Nothing in it names a workload.

This proposal defines the consumer surface for
[Zava](../traffic-intelligence/envoy-routing-zava.md) feature #1, geo-aware upstream
selection, and answers the [Dani
Rojas](../traffic-intelligence/l4-load-balancing-dani-rojas.md) question of how a load
balancer discovers compute backends as instances start and stop.

## Motivation

HTTPProxy assumes one origin on the internet: the edge terminates TLS, protects the
origin, and forwards. One endpoint URL per backend is right for that.

Compute makes the origin a set of instances the platform places, replaces, scales, and
moves between locations. One URL cannot describe that set. Describing it falls to the
consumer: enumerate every instance address and keep it current, decide which location each
edge prefers, and detect a failed location fast enough to matter.

The platform already knows all three. Solving it once is cheaper than every consumer
solving it on every deployment change, and whatever needs traffic spread across locations
next reuses the answer.

### Goals

- Give consumers one resource meaning "my application's endpoints, wherever they run."
- Track membership automatically as instances appear, disappear, and move.
- Serve each request from the nearest members, failing over automatically and returning on
  recovery, with no configuration.
- Add and remove members without dropping requests.
- Report per-location member and health counts so consumers diagnose their own problems.

### Non-Goals

- **Modeling compute in the networking API.** A NetworkService selects network interface
  claims.
- **How the Layer 4 load balancer is configured.** [Dani Rojas](../traffic-intelligence/l4-load-balancing-dani-rojas.md) owns that. A
  NetworkService defines the membership that load balancer consumes. Its policy belongs
  there.
- **Per-request latency-based routing.** Steering here changes at control plane speed.
  Measured round-trip time is [Zava](../traffic-intelligence/envoy-routing-zava.md)
  feature #3.
- **Consumer-controlled steering and health checks.** `Nearest` is the only strategy, and
  the platform judges members by real request outcomes. Both are expected to gain controls;
  neither has them here.
- **An address of its own.** A virtual IP for east-west traffic is a natural extension this
  milestone does not deliver.

## Proposal

A consumer creates a `NetworkService` naming a label and a port. The platform resolves
that label into the endpoint set and keeps it current.

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
        compute.datumapis.com/workload-name: storefront
  ports:
    - name: http
      port: 8080
```

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

A shopper in Atlanta reaches Datum's Ashburn edge and is served from Dallas; a shopper in
Portland reaches Seattle and is served from San Jose. Priya configured neither route.

#### Story 2: Change capacity without editing anything

Priya scales Dallas from two replicas to twenty for Black Friday. The new interfaces carry
the same label, join the service, and take traffic once they are running. She scales back
down on Saturday, and the instances going away finish their in-flight requests first.
Later she adds Frankfurt, and European users start being served there.

None of it requires an edit to the NetworkService or the HTTPProxy.

#### Story 3: Survive a location failure at 3am

Dallas loses power. Health checks fail and its traffic moves to San Jose within seconds.
Users see higher latency. They see no errors. Dallas resumes serving its own region when
it returns. Priya reads about it in the morning.

### Notes/Constraints/Caveats

**The platform records a member's location.** An interface is served in one location, and
networking labels it with that location when it publishes the copy. Consumers never write
it.

**Selecting works without labeling anything first.** Claim-creating services stamp a
defined set of keys. The facts worth selecting on are already present, spelled the same
way for everyone. Consumers can still get their own labels later; see [Well-known
labels](#well-known-labels).

### Reaching a member

A member's addresses are inside its network. The tenant addresses it holds are unroutable
from outside the VPC, so an address a proxy is handed cannot reach an origin on its own.
Members are reached over the fabric, which costs nothing per instance and keeps origins
private.

The fabric carries a packet to a member by encapsulating it. The sending node wraps the
packet toward a segment identifying the receiving attachment, and the receiving node
unwraps it and looks the inner packet up in that attachment's table. The inner packet
still carries the tenant address, so the address a proxy dials never changes. What changes
is that the edge learns a segment to wrap toward.

**A cell composes that segment, because only a cell can.** The values it is built from are
allocated per location and mean nothing anywhere else. A cell knows which node holds a
member, and it knows that node's block, so it can compose a complete address that any edge
can route to without knowing how any of it was allocated.

**The segment never reaches a consumer.** A consumer's own view keeps the tenant address,
which is the honest one: it is the address their instance holds. The segment travels only
on the copies the platform propagates to its edges, and is one platform-internal step
between a member and the traffic it serves.

**Each member gets its own published entry.** One member per object keeps a segment and
the address it belongs to in one place, so the two cannot disagree. A segment paired with
the wrong address is a silent hole rather than a visible failure. Grouping members
together later is an optimisation that changes nothing a consumer or the edge can see.

Wrapping a packet is done by the host rather than by the proxy, and an edge can already do
it. Nothing new has to run there: an edge learns, per member, which segment to wrap
toward, and the rest is configuration it is already able to apply.

<<[UNRESOLVED identity collisions ]>>
Per-location allocation is safe for reaching a member, because nothing outside a location
interprets what it allocated. It is not safe for the identity a network is known by across
locations, which is allocated at random today and can collide. That is a separate problem
with its own proposal, and it does not block reaching a member.
<<[/UNRESOLVED]>>

**Traffic from the edge to a member crosses the public internet today.** A private
backbone is planned, and it will shorten those hops without changing anything here. Until
it lands, ranking locations by geography is a rough guide to latency: peering and cable
routes mean the closest location is sometimes not the fastest.

### Risks and Mitigations

- **An over-broad selector sends traffic to the wrong endpoints.** Per-location status makes
  resolved membership visible. A selector spanning networks is caught loudly. One matching
  nothing raises a condition, since a service may be written before its workload.
- **Failover moves traffic across a jurisdiction boundary.** No control declares spilling
  unacceptable in this milestone, so consumers with hard residency obligations should not
  use a NetworkService yet, and the product must say so. The durable answer is a sovereignty
  constraint in [Total Load Balancing](../traffic-intelligence/total-load-balancing.md).
- **Membership goes stale, or members are unreachable, while status looks healthy.** Report
  when membership was last confirmed rather than presenting it as live, and report
  reachability in the service's own status rather than only in proxy error rates.

Security review must cover selector scoping across namespace and project boundaries, and
confirm that a consumer cannot select interfaces they do not own.

## Design Details

A follow-on technical design owned by the network services operator covers how the edge
gets programmed.

![Components involved in a network service, grouped by the control plane each runs
in](./components.png)

Four control planes carry a share of the work. A cell binds a claim and records where it
is served. The federation hub carries a copy toward the consumer. The project resolves
membership and turns it into endpoints. The edge translates those endpoints into
localities and serves the request. No plane holds more than it needs, which is why the
copy a consumer reads is smaller than the claim a cell holds.

### NetworkService

```yaml
apiVersion: networking.datumapis.com/v1alpha
kind: NetworkService
metadata:
  name: storefront
spec:
  # A standard label selector.
  networkInterfaces:
    selector:
      matchLabels:
        compute.datumapis.com/workload-name: storefront

  ports:
    - name: http
      port: 8080
      protocol: TCP

  # Optional. Nearest is the only value this milestone accepts.
  trafficDistribution:
    strategy: Nearest
```

### Membership

An interface joins when it matches the selector, is held by a claim, and its holder
reports itself available to serve. An interface whose claim has been released is retained
capacity holding an address with nothing behind it, and is never a member. It leaves when
it stops matching or goes away, ordinarily after [draining](#draining) rather than
abruptly. A selector reaches only the consumer's own resources.

**A service spans one network.** A selector matching interfaces across two networks is a
configuration error. Whether the service names its network outright or reports the
mismatch is still open. The constraint holds either way.

A NetworkService is written in one control plane and its members live in others.
[Federation](#federation) covers how the facts travel.

### Well-known labels

Every interface carries a defined set of labels, applied by whichever service created it.

Networking applies this when it publishes an interface, where the location is known:

| Label | Value | Example |
|---|---|---|
| `networking.datumapis.com/location` | The location serving the interface. | `us-central-1` |

**The location key is authoritative, and stays that way without a webhook.** Networking
writes it where the truth lives, and publication rewrites the keys networking owns on
every pass, so an edit to a published copy is corrected on the next reconcile. A consumer
can add labels of their own; they cannot durably change this one. Correction is eventual
rather than immediate, so a wrong value survives until the next pass.

Compute applies these to the claim it creates, and they travel to the interface and its
published copy:

| Label | Value | Example |
|---|---|---|
| `compute.datumapis.com/workload-name` | The workload owning the interface. | `storefront` |
| `compute.datumapis.com/placement-name` | The placement within that workload. | `americas` |
| `compute.datumapis.com/city-code` | The city the workload was placed in. | `dfw` |
| `compute.datumapis.com/instance-index` | The instance's ordinal within the placement. | `0` |

These are the keys compute already stamps on every Instance, reused rather than
reinvented. Location appears twice: networking's `location` and compute's `city-code`.
Networking's own key is the one to select on, because networking sets it on the published
copy and guarantees it for claims no workload created.

**Networking never interprets a compute key.** A key's prefix records which service
guarantees it. Any service can populate keys under its own prefix without changing
networking.

Selecting a whole application is the common case, as shown above. Narrower selections add
keys — one placement, or one city:

```yaml
networkInterfaces:
  selector:
    matchLabels:
      compute.datumapis.com/workload-name: storefront
      networking.datumapis.com/location: us-central-1
```

Adding the location key restricts which claims are members. It does not steer traffic:
getting Dallas users served from Dallas requires nothing.

A service can also span more than one value of a key, which lets two workloads serve one
hostname as a blue/green rollout does. The field allows it from the start, because adding
it later would mean changing the field:

```yaml
networkInterfaces:
  selector:
    matchExpressions:
      - key: compute.datumapis.com/workload-name
        operator: In
        values: [storefront-blue, storefront-green]
```

**Managing addresses.** The same vocabulary pays off beyond load balancing: which
addresses an application holds, which it holds in one location, and which survive a
replacement all become the same query.

### Traffic distribution

`spec.trafficDistribution.strategy` defaults to `Nearest`, the only value this milestone
accepts; see [How nearest is decided](#how-nearest-is-decided).

The field carries one value deliberately: a consumer who sets it today keeps working when
we add strategies, at the cost of an enum that reads unfinished.

### Health

Two things have to be true before a member takes traffic, each established by whoever
can see it.

| Signal | Established by | Catches |
|---|---|---|
| The holder reports itself available | the workload holding the interface | An address with nothing serving behind it |
| Requests succeed | the edge, watching real traffic | A member that is up but broken |

The first decides whether a member is published. The second runs continuously afterwards
and is acted on where it is observed: an edge stops using a member that fails requests,
without asking anything else first. A service does not report it. Judging it centrally
would mean carrying an account of every request back to the control plane, which grows
with traffic rather than with the number of services, and the edge has already acted by
the time any of that could arrive.

Only the holder can answer the first. Networking sees an address and cannot tell whether
anything is listening on it, so it reads a single condition the holder writes and never
interprets what the holder is. A member is healthy while its holder vouches for it, and
stops being healthy the moment the holder stops, including while it is shutting down.

That an interface is programmed is a separate claim about whether its address can carry
packets at all, and it is tracked with the reachability question below rather than here.

Detection is reactive. A few requests fail before a member is ejected, and recovery is
tested with real requests. A member receiving no traffic is not assessed at all, which
matters most in a location sitting idle because its users are served closer.

Ejection also drives location failover: enough ejected members degrade a location far
enough that traffic moves down the ranking.

Edge health checking is independent of the platform-wide health signals
[Nate](../traffic-intelligence/health-checks-nate.md) publishes, which run on a different
timescale for a different purpose.

<<[UNRESOLVED ejection ]>>
Ejection is capped as a share of a location's members, and under a common default a
two-member location can eject neither. Two replicas per location is the ordinary case
here, so pick the defaults against that size: how many failed requests eject a member, how
long it stays out, and how much of a location may go at once. Allowing a whole location to
be ejected makes failover work when everything there is broken; capping below that stops a
location emptying on a correlated blip.
<<[/UNRESOLVED]>>

### Draining

A member that is going away leaves rotation before it stops answering. Requests already in
flight finish. No new ones arrive.

Without it, the edge finds out by dialing a member and failing. Health checking bounds
that to seconds, but those seconds are user-visible errors, and every scale-down and
rolling update produces one such window per member replaced. Because a member's address is
tied to its node, a rescheduled instance is a departure and an arrival rather than an
update, which makes routine deployments the common case for this rather than the rare one.

Draining depends on notice that a member is about to go away, given before it does. That
signal belongs to whatever retires the member, which also reports when a member has
started.

<<[UNRESOLVED holder lifecycle ]>>
Whatever holds a claim reports two transitions on it: started, and about to stop. For
compute that is an instance reaching running and an instance being retired. The retirement
notice has to reach the edge before the member stops answering, and how much warning that
needs depends on propagation, which is unmeasured, so settle the two together. A member
lost abruptly through node failure gives no notice and stays with the edge's health
checking.

Decide also how a not-yet-running member is represented. Publishing it as not-ready marks
it draining at the proxy, which withholds traffic correctly but counts it against its
location's health: a location scaling from two members to twenty would briefly look mostly
unhealthy and could spill traffic away for no reason. Omitting a member until it runs
avoids that and makes joining and leaving asymmetric.
<<[/UNRESOLVED]>>

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

A consumer should be able to answer "which location serves my users, and is any location
out of rotation" without contacting support.

### Consuming a network service

An HTTPProxy rule reaches a service through a `networkService` backend naming the service
and one of its ports. It sits alongside three existing forms: an endpoint URL, an endpoint
reached through a connector, and a single instance on a tenant network. It generalizes
that last one from one endpoint to a set.

Referencing by name rather than by API group and kind keeps HTTPProxy backends a curated
set. Naming a port rather than a number lets the reference survive a port change.

We intend the same resource to back a Layer 4 load balancer when [Dani
Rojas](../traffic-intelligence/l4-load-balancing-dani-rojas.md) becomes
customer-configurable. That is why ports carry a protocol and the resource is not named
for HTTP.

### Federation

A consumer writes a NetworkService in their project. Its members are claims served in POP
cells, which are separate clusters from the edge clusters serving traffic. **Each control
plane holds the minimum it needs.** A cell holds full-fidelity interfaces because
providers there configure real NICs. The edge holds addresses, ports, locations, and
health, because that is all a proxy needs to pick a member.

Controllers that predate this proposal already publish interfaces to consumers, so
membership reads a copy that is already there.

| Where | What happens |
|---|---|
| Cell | `NetworkInterfaceClaimReconciler` allocates addresses and binds an interface to the claim |
| Cell → hub | `NetworkInterfaceWriteBackReconciler` publishes a copy of the interface |
| Hub → project | `NetworkInterfaceProjector` publishes that copy into the project, owned by the consumer's `Network` |
| Hub | `NetworkInterfaceProjectionGCReconciler` removes a project copy once nothing is published behind it |
| **Cell → hub → project** | **Claim publication, on the same path, so a consumer sees the claim as well as the interface. This does not exist yet.** |
| Project | The consumer's `NetworkService`, with resolved membership in its status |
| Hub → edges | The resolved endpoint set per service, federated out |

A published copy carries the network, interface name, MTU, addresses, and reclaim policy.
It drops everything naming an object that does not exist where it lands: the claim
reference, the network context, the attachment, and the VPC identifier.

**Labels have to travel.** Compute stamps its keys on the claim it creates and networking
stamps the location in the cell, but no claim is published to the project, so nothing
carries either to where a consumer selects. Publication is the substantive federation work
this proposal requires.

<<[UNRESOLVED label propagation ]>>
Propagating labels blindly is the simple rule, and it lets a consumer's key collide with a
platform one. Restricting propagation to known prefixes avoids that and makes networking
hold a list it otherwise would not. Decide which. Decide also whether a label added to a
claim after creation is expected to reach the copy, since claims are written once and
never updated today.

Whichever rule is chosen, publication must keep overwriting the keys networking owns
rather than merging them, since that is what makes the location authoritative.
<<[/UNRESOLVED]>>

**A member joins once it can serve.** An interface holds an address long before anything
answers on it, so membership waits for the holder to say it is serving rather than for
allocation. Programming the data plane is not that signal: it says an address can carry
packets, not that a workload is ready to answer, and the two are far apart in time.

#### What happens when a control plane is unreachable

| Failure | Effect | Why this is correct |
|---|---|---|
| Karmada unavailable | Membership freezes. Programmed endpoints keep serving; joins and departures wait. | Fail-static. A federation outage must not change what traffic does. |
| POP cell unreachable, capacity healthy | Its members stay in rotation. | The alternative withdraws healthy capacity because a control plane became unreachable, turning a management outage into a customer outage. |
| POP cell unreachable, capacity down | Edge health checks eject the members within seconds. | Liveness is the edge's job. |
| Project control plane unavailable | Services cannot be created or edited; existing ones keep serving. | Consumers lose the ability to change things. Traffic keeps flowing. |
| Edge cluster isolated | That edge serves its last known configuration. | Anycast withdraws an edge that is badly broken. One that is merely stale still serves users correctly. |

**Never treat control plane reachability as data plane health.** Conflating them turns
routine control plane incidents into traffic incidents. Membership answers which members
exist. Only edge health checking answers which members work.

<<[UNRESOLVED staleness ]>>
Fail-static is right for a short outage and wrong for an indefinite one. Membership frozen
for a week is not a useful answer. Decide whether a staleness bound exists and what it
does when crossed. The obvious answer — expiring stale members — reintroduces the failure
the table above rejects.
<<[/UNRESOLVED]>>

### How nearest is decided

Each edge ranks the service's locations by proximity to itself, prefers its own top-ranked
location, and treats the rest as an ordered fallback. Because every edge ranks for itself,
one NetworkService produces correct local behavior everywhere. The consumer never
expresses a matrix of locations against edges, which is what makes hand-rolled
multi-region routing miserable.

Ranking starts from each location's coordinates, which every location a service can use is
expected to carry.

**An operator can override the computed order.** Geographic distance and network distance
are not the same thing: peering, backbone topology and transit cost all decide which
location an edge actually reaches fastest, and none of them are visible in a pair of
coordinates. The platform therefore keeps a per-location preference order that replaces
the computed one where it is set. An edge uses the row for the location it sits in.

Consumers never see this table and never write to it. It is operator-controlled platform
data, the same as the coordinates it overrides. Where no override exists, the computed
order stands, so the table holds only the cases where geography lies.

Measured signals replace both later as the [Total Load
Balancing](../traffic-intelligence/total-load-balancing.md) signal set matures, and that
substitution is invisible in this API.

<<[UNRESOLVED coordinates ]>>
No location carries coordinates today. The field is optional on `Location` and is unset on
all three that exist, so ranking cannot be computed until someone enters them. That is an
operator task rather than an engineering one, and it gates the feature's headline
behaviour. The override table does not remove the requirement: it is meant to correct a
computed order, not to substitute for having one.
<<[/UNRESOLVED]>>

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

**2. The network service and the proxy.** Exactly as shown in [What a consumer
writes](#what-a-consumer-writes). Neither changes because the workload runs in two cities
rather than one.

**What follows without being written.** Four instances come up, two per city. Compute
requests an interface for each and labels it `compute.datumapis.com/workload-name:
storefront`; networking labels it with its city. All four join the service. A request for
The consumer wrote no label, and neither networking resource mentions a city, an address,
or a workload.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

Additive, and in use only where a consumer created a `NetworkService` and referenced it.
Existing endpoint-URL, connector, and instance backends are untouched. Enabling changes no
default behavior and needs no downtime. Rolling back is consumer-visible: a proxy backed
by a NetworkService has no fallback backend. Those proxies stop serving. Re-enabling
converges on reality, because membership re-resolves from current state rather than a
stored list.

### Monitoring Requirements

Uptake is a count of `NetworkService` resources and of HTTPProxy rules referencing one.
Consumers see whether it works through `.status`: the `Ready`, `MembersResolved`, and
`EndpointsReachable` conditions, and per-location member and healthy counts. SLIs are
membership resolution latency, per-location healthy member count, edge requests by serving
location, and failover events per service.

<<[UNRESOLVED slos ]>>
Two numbers need agreement and neither should be guessed: how quickly a membership change
reaches the edge, and how quickly a health change moves traffic. Both are end-to-end
across the control plane and need measurement in the prod-fidelity environment first.
<<[/UNRESOLVED]>>

### Dependencies

| Dependency | Used for | If it is unavailable |
|---|---|---|
| Network interfaces | The unit of membership | Membership cannot change; programmed endpoints keep serving |
| The VPC data plane | Reporting that an interface is programmed, which is what makes a member healthy | No member ever becomes healthy, so nothing serves |
| Karmada federation | Carrying interfaces toward the project and endpoints out to the edges | Membership freezes, deliberately |
| Compute | Stamping the well-known labels most consumers select on | Existing interfaces keep their labels |
| Location coordinates | Ranking locations by proximity for each edge | Every location ranks equally, so nearest degrades to arbitrary. Coordinates are unset today, so this is a precondition rather than a runtime risk. |
| Edge proxy fleet | Load balancing and health checking | Traffic is not served, which is the existing edge failure mode |

Degraded performance in the first two shows up as membership lagging reality: scaled-up
members wait for traffic, scaled-down members linger. Edge health checking bounds the
latter.

### Scalability

Membership resolution watches claims, which the operator did not do before; that volume
scales with member count rather than request volume. One new namespaced type, one to a few
per application per project. Edge endpoint configuration grows from one endpoint per proxy
rule to one per member.

<<[UNRESOLVED scale ]>>
Endpoint fan-out grows differently from the gateway-count growth already measured and
needs its own measurement before limits are published. Test the configuration size ceiling
early: a prior scale run stopped programming silently once generated configuration crossed
the default message limit, and endpoints draw on the same budget. If this is what binds,
it is the trigger for the separate delivery path recorded in
[Alternatives](#alternatives).
<<[/UNRESOLVED]>>

## Implementation History

- 2026-08-25: Initial draft.

## Drawbacks

**A selector that stops matching produces no error.** Traffic goes missing where an
explicit list would fail visibly. That is the price of membership tracking reality, and it
puts a lot of weight on status being right.

**Consumers whose needs differ from the defaults have no recourse.** With one strategy
value and no health check configuration, a consumer cannot canary a location, drain one
for maintenance, hold traffic inside a jurisdiction, or tune how readily a member is
ejected. The only workaround is to stop using a NetworkService. A consumer who needs one
of these is blocked.

**Judging health by real traffic means some requests pay for it.** A member is ejected
after it has already failed requests. Only a declared health check would catch a broken
member before it serves anyone.

**Nearest-location routing depends on the platform knowing where capacity is.** If that
data is wrong, routing is wrong in a way that looks fine from the outside, which is much
harder to debug than an outright failure.

## Alternatives

**Point the proxy at a workload.** Puts a compute concept in the networking API, couples
the two services' release cycles, and offers nothing for endpoints compute did not create.

**Let consumers list endpoints with per-location weights.** Hands the consumer exactly the
work this proposal exists to absorb: a list that goes stale on the first capacity change,
and a weight matrix maintained against every edge.

**Select the interface rather than the claim that holds it.** A retained interface
outlives its claim, keeping its address and labels while nothing runs behind it, so
membership would include retired capacity that a label selector cannot filter out. Labels
would also have to survive binding and projection, which is three places to get right
instead of one.

**Deliver endpoints outside the resource path**, either on a purpose-built transport or by
pointing the proxy at a separate source. Both keep membership out of the configuration the
proxy is otherwise given. Churn costs less, and service size stops bearing on
configuration limits. Deferred rather than rejected: each adds a delivery mechanism to
operate, debug, and alert on, for a payload the existing path already carries. Revisit if
measurement shows membership size, churn, or propagation is the binding constraint.

**Solve it in DNS.** The wrong layer: it fails over at cache speed rather than connection
speed, and discards the edge's knowledge of where the request entered. DNS steering
remains valuable as [Jamie Tartt](../traffic-intelligence/gslb-jamie-tartt.md), a
complement rather than a substitute.

**Name the resource `Backend` or `EndpointGroup`.** `Backend` collides with an existing
type in the edge proxy stack and names a position rather than a thing. `EndpointGroup`
describes membership accurately but forecloses the address this resource may grow.
