---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Datum Connectors

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Connector](#connector)
  - [ConnectorAdvertisement](#connectoradvertisement)
  - [ConnectorAttachment](#connectorattachment)
  - [Integration with Proxies](#integration-with-proxies)
  - [Connector Controllers](#connector-controllers)
  - [Additional Reading](#additional-reading)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)

## Summary

Datum Connectors introduce a method to model outbound connectivity as first
class resources in the control plane. A Connector resource lives in a project
and captures what kinds of network access it can facilitate, while
ConnectorAdvertisement resources describe the networks and endpoints that are
reachable through that connector. Other platform components, such as proxies,
can then target backends via these connectors, allowing the control plane to
reason about and route traffic to remote or otherwise inaccessible environments.

A Connector will always be managed by a controller responsible for maintaining
its lifecycle, capabilities, advertisements, and health state in the control
plane. The first controller implementation will be the [Datum Connect][datum-connect]
application, which will register and manage Connector, ConnectorAdvertisement,
and HTTPProxy resources.

## Motivation

Datum Connectors aim to make outbound connectivity a first class, declarative
part of the control plane. By introducing resources that describe which
connectors exist, what capabilities they support, and what networks and
endpoints they can reach, other platform components can reliably use this
information to route traffic and present clear views of reachable environments.

### Goals

- Define API resources that describe connectors, their capabilities, and their
  reachable networks and endpoints.
- Allow platform components such as proxies to target backends via connectors,
  using names or label selectors.
- Support multiple connector classes behind a common Connector API, and expose
  connector health and capability status in a consistent way.

### Non-Goals

- Specifying how individual connector implementations establish or transport
  traffic.
- Replacing existing Datum networking abstractions; connectors are an additional
  way to represent outbound connectivity, not a full networking redesign.
- Providing a generic extension mechanism for arbitrary, user-defined connector
  capabilities. Capabilities are explicitly modeled in the API.

## Proposal

### User Stories

> [!NOTE]
> This section needs more content.

#### Portal - List Connectors

- Show name, health, advertisement count

#### Portal - View Connector

- Detailed information (capabilities, connection info)
- Advertisements
- Telemetry

#### Portal - Create connector

- Wizard to get users going with the [Datum Connect][datum-connect] application.

#### Portal - Proxies

- Add the ability to target an endpoint _via_ a connector.
- Leave the door open for defining settings when using the connector, such as
  how a selector based lookup will choose one or more connectors.

### Notes/Constraints/Caveats (Optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
software ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside of your immediate team.
-->

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

### Connector

- Lives within a Project.
- Facilitates connections from L1 up.
- Different connector classes to be aligned with Datum Connect, Connection
  Coordinator API, Tailscale, Netbird, etc.
- Managed by a connector controller. This may be a daemon on a user controlled
  machine, or internal control plane software.
- Defines capabilities and settings supported by the connector. For example:
  - `MASQUE`: Supports MASQUE connections (QUIC, HTTP3, MASQUE protocols).
  - `HTTP2`: Supports HTTP2 connections, useful for clients that don't support
    QUIC/HTTP3.
  - `CONNECT-TCP`: Supports accepting HTTP CONNECT requests.
  - `CONNECT-UDP`: Supports accepting HTTP CONNECT-UDP requests.
  - `CONNECT-IP`: Supports accepting HTTP CONNECT-IP requests.
  - `OTEL`: Supports requesting OTEL (Metrics, Traces, Logs) to be pushed back
    over the requesting stream.
  - `GRPC_HEALTH`: Supports requesting health information via the [health/v1][grpc-health]
    service API.
- Capabilities are defined explicitly in the API structure. Each capability may
  have additional settings that can be defined.
- A connector's controller is responsible for setting capabilities.
- Defines connection details, if any: `PublicKey` (iroh), `IPAddress`, `Hostname`
- Connector status communicated via conditions.

#### Example Connector Manifest

```yaml
apiVersion: networking.datumapis.com/v1alpha1
kind: Connector
metadata:
  name: datum-connect-sk5xg
  namespace: default
spec:
  connectorClassName: datum-connect
  # Sparse list of capabilities. If a capability is omitted, it will not be
  # known as available for the connector.
  capabilities:
    # Enables QUIC listener that handles MASQUE protocols
    - type: MASQUE
      enabled: true
    # No HTTP2 listener. Can also simply be omitted from the capabilities list.
    - type: HTTP2
      enabled: false
    - type: CONNECT-TCP
      enabled: true
    - type: CONNECT-UDP
      enabled: true
    - type: CONNECT-IP
      enabled: true
      connectIP:
        mtu: 1400
    - type: OTEL
      otel:
        metrics:
          enabled: true
        traces:
          enabled: false
        logs:
          enabled: true
    - type: GRPC_HEALTH
      enabled: true
status:
  capabilities:
    - type: MASQUE
      conditions:
        - lastTransitionTime: "2025-11-19T14:57:03Z"
          message: The connector is ready to accept MASQUE requests
          observedGeneration: 1
          reason: ListenerReady
          status: "True"
          type: Ready
    - type: CONNECT-IP
      conditions:
        - lastTransitionTime: "2025-11-19T14:57:03Z"
          message: The connector does not support CONNECT-IP
          observedGeneration: 1
          reason: NotSupportedByVersion
          status: "False"
          type: Ready
    - type: OTEL
      conditions:
        - lastTransitionTime: "2025-11-19T14:57:03Z"
          message: The connector has established an OTLP connection and is publishing telemetry.
          observedGeneration: 1
          reason: ConnectionEstablished
          status: "True"
          type: Connected
    # ... remaining capabilities
  leaseRef:
    name: datum-connect-sk5xg
  conditions:
    - lastTransitionTime: "2025-11-19T14:57:03Z"
      message: The connector is ready to tunnel traffic.
      observedGeneration: 1
      reason: ConnectorReady
      status: "True"
      type: Ready
  connectionDetails:
    type: PublicKey
    publicKey:
      value: 2ovpybgj3snjmchns44pfn6dbwmdiu4ogfd66xyu72ghexllv6hq
      discoveryMode: DNS
```

[grpc-health]: https://github.com/grpc/grpc-proto/blob/master/grpc/health/v1/health.proto

### ConnectorAdvertisement

- One or more advertisements associated with a Connector.
- Must live in the same namespace as the Connector it's attached to.
- Communicates what the connector can facilitate connections to.
  - L3 CIDRs
  - L3 Interconnect (Connection Coordinator API)
  - L4 endpoints

#### Example ConnectorAdvertisement Manifest

```yaml
apiVersion: networking.datumapis.com/v1alpha1
kind: ConnectorAdvertisement
metadata:
  name: datum-connect-sk5xg
  namespace: default
spec:
  connectorRef:
    name: datum-connect-sk5xg
  layer3:
  # Communicates what networks can be reached via the connector
    - name: local-network
      cidrs:
        - 10.0.0.0/8
        - fd20:3c5:71e:0:0:0:0:0/48
  # Communicates that the connector can reach a TCP service at 127.0.0.1:80
  layer4:
    - name: local-dev
      services:
        # Address can be an IPv4, IPv6, or a DNS address. A DNS address may
        # contain wildcards. A DNS address acts as an allow list for what
        # addresses the Connector will allow to be requested through it.
        #
        # DNS resolution is the responsibility of the connector.
        - address: 127.0.0.1
          ports:
            - name: http
              port: 80
              protocol: TCP
        - address: internal-service.example.com
          ports:
            - name: https
              port: 443
              protocol: TCP
```

### ConnectorAttachment

> [!NOTE]
> Future state

- Association of a Connector with a MeetMeRoom (MMR), or Galactic VPC.
- For future connectors, such as one that facilitates physical connections, an
  attachment will allow the connector's controller to know what it needs to
  provision.

### Integration with Proxies

Reference a connector by name:

```yaml
apiVersion: networking.datumapis.com/v1alpha
kind: HTTPProxy
metadata:
  name: proxy-to-connector-backend
spec:
  rules:
    - backends:
        # Connect to 127.0.0.1:80 via the referenced connector.
        - endpoint: http://127.0.0.1:80
          connector:
            name: datum-connect-sk5xg
```

Or reference one or more connectors by label:

```yaml
apiVersion: networking.datumapis.com/v1alpha
kind: HTTPProxy
metadata:
  name: proxy-to-connector-backend
spec:
  rules:
    - backends:
        # Connect to 127.0.0.1:80 via a connector with matching labels.
        # Behavior of matching multiple connectors is TBD.
        - endpoint: http://127.0.0.1:80
          connector:
            selector:
              matchLabels:
                local-dev: user1
```

### Connector Controllers

A connector controller is responsible for facilitating connections, maintaining
heartbeats, and potentially creating the connector resource and maintaining
announcements as well.

#### Datum Connect

[Datum Connect][datum-connect] is an application capable of tunneling traffic
over HTTP tunnels via either HTTP/2 or HTTP/3. Additionally, the connect process
can negotiate peer to peer connections via the use of the [iroh][iroh] p2p
library.

The Datum Connect application will be expanded to:

- Register a `Connector` with the Datum control plane, in a target project and
  namespace, which will allow discovery of the connector, and enable platform
  features to target endpoints available behind the connector.
- Create a `Lease` for the connector, and maintain periodic heartbeats.
- Maintain zero or more `ConnectorAdvertisements`, to let the platform and other
  services know what connections the connector can facilitate.
- Maintain zero or more `HTTPProxy` resources with backends that are accessed
  by the Datum Proxy fleet via the connector.

In the future, the application can be expanded to support the OTEL and health
reporting information described in the [Connector API design](#connector).
Additionally, the application can be updated to establish direct connections
between different instances if allowed by policy.

[datum-connect]: https://github.com/datum-cloud/datum-connect
[iroh]: https://www.iroh.computer/

### Additional Reading

- AWS and GCP [recently announced a partnership][cc-api-announcement] to align
  on a [Connection Coordinator API Specification][cc-api]. Building a connector
  that can orchestrate interconnects via this API could make sense.

[cc-api-announcement]: https://cloud.google.com/blog/products/networking/aws-and-google-cloud-collaborate-on-multicloud-networking
[cc-api]: https://github.com/aws/Interconnect/tree/main

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features are observable,
scalable and supportable; can be safely operated in production environments, and
can be disabled or rolled back in the event they cause increased failures in
production.

See more in the PRR Enhancement at https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved
for the Enhancement to move to `implementable` status and be included in the release.
-->

### Feature Enablement and Rollback

<!--
This section must be completed when targeting alpha to a release.
-->

#### How can this feature be enabled / disabled in a live cluster?

<!--
Pick one of these and delete the rest.
-->

- [ ] Feature gate
  - Feature gate name:
  - Components depending on the feature gate:
- [ ] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control plane?
  - Will enabling / disabling the feature require downtime or reprovisioning of a node?

#### Does enabling the feature change any default behavior?

<!--
Any change of default behavior may be surprising to users or break existing
automations, so be extremely careful here.
-->

#### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

<!--
Describe the consequences on existing workloads (e.g., if this is a runtime
feature, can it break the existing applications?).

Feature gates are typically disabled by setting the flag to `false` and
restarting the component. No other changes should be necessary to disable the
feature.
-->

#### What happens if we reenable the feature if it was previously rolled back?

#### Are there any tests for feature enablement/disablement?

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

#### How can a rollout or rollback fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->

#### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

#### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

#### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

#### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the API (e.g., checking if
there are objects with field X set) may be a last resort. Avoid logs or events
for this purpose.
-->

#### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is an instance-related feature, it should be possible to
determine if the feature is functioning properly for each individual instance.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so
that they can verify correct enablement and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

- [ ] Events
  - Event Reason:
- [ ] API .status
  - Condition name:
  - Other field:
- [ ] Other (treat as last resort)
  - Details:

#### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

<!--
This is your opportunity to define what "normal" quality of service looks like
for a feature.

It's impossible to provide comprehensive guidance, but at the very
high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99.9% of /health requests per day finish with 200 code

These goals will help you determine what you need to measure (SLIs) in the next
question.
-->

#### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:

#### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

#### Does this feature depend on any specific services running in the cluster?

<!--
Think about both cluster-level services (e.g. metrics-server) as well
as node-level agents (e.g. specific version of CRI). Focus on external or
optional services that are needed. For example, if this feature depends on
a cloud provider API, or upon an external software-defined storage or network
control plane.

For each of these, fill in the following—thinking about running existing user workloads
and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:
-->

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

#### Will enabling / using this feature result in any new API calls?

<!--
Describe them, providing:
  - API call type (e.g. PATCH workloads)
  - estimated throughput
  - originating component(s) (e.g. Workload, Network, Controllers)
Focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)
-->

#### Will enabling / using this feature result in introducing new API types?

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

#### Will enabling / using this feature result in any new calls to the cloud provider?

<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->

#### Will enabling / using this feature result in increasing size or count of the existing API objects?

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

#### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

#### Will enabling / using this feature result in non-negligible increase of resource usage in any components?

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

#### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

<!--
Focus not just on happy cases, but primarily on more pathological cases.

Are there any tests that were run/should be run to understand performance
characteristics better and validate the declared limits?
-->

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

#### How does this feature react if the API server is unavailable?

#### What are other known failure modes?

<!--
For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.
-->

#### What steps should be taken if SLOs are not being met to determine the problem?

## Implementation History

<!--
Major milestones in the lifecycle of a Enhancement should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first release where an initial version of the Enhancement was available
- the version where the Enhancement graduated to general availability
- when the Enhancement was retired or superseded
-->

## Drawbacks

<!--
Why should this Enhancement _not_ be implemented?
-->

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

## Infrastructure Needed (Optional)

<!--
Use this section if you need things from another party. Examples include a
new repos, external services, compute infrastructure.
-->
