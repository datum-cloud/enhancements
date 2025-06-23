---
status: provisional|implementable|implemented|deferred|rejected|withdrawn|replaced
stage: alpha|beta|stable
latest-milestone: "v0.x"
---
<!--
Inspired by https://github.com/kubernetes/enhancements/tree/master/keps/NNNN-kep-template

Goals are aligned in principle with those described at https://github.com/kubernetes/enhancements/blob/master/keps/sig-architecture/0000-kep-process/README.md

Recommended reading:
  - https://developers.google.com/tech-writing
-->

<!--
**Note:** When your Enhancement is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Make a copy of this template directory.**
  Copy this template into the desired path and name it `short-descriptive-title`.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary" and "Motivation" sections.
  These should be easy if you've preflighted the idea of the Enhancement with the
  appropriate stakeholders.
- [ ] **Create a PR for this Enhancement.**
  Assign it to stakeholders who are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the Enhancement clarified and merged quickly. The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a Enhancement is merged does not mean it is complete or approved. Any Enhancement
marked as `provisional` is a working document and subject to change. You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing RFCs, aim for tightly-scoped, single-topic PRs to keep discussions
focused. If you disagree with what is already in a document, open a new PR
with suggested changes.

One Enhancement corresponds to one "feature" or "enhancement" for its whole lifecycle.
You do not need a new Enhancement to move from beta to GA, for example. If
new details emerge that belong in the Enhancement, edit the Enhancement. Once a feature has
become "implemented", major changes should get new RFCs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/docs/rfcs/template/README.md).

**Note:** Any PRs to move a Enhancement to `implementable`, or significant changes once
it is marked `implementable`, must be approved by each of the Enhancement approvers.
If none of those approvers are still appropriate, then changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross-cutting RFCs).
-->

# Milo Control Plane

<!--
This is the title of your Enhancement. Keep it short, simple, and descriptive. A good
title can help communicate what the Enhancement is and should be considered as part of
any review.
-->

<!--
A table of contents is helpful for quickly jumping to sections of a Enhancement and for
highlighting any additional information provided beyond the standard Enhancement
template.
-->

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (Optional)](#user-stories-optional)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
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

<!--
This section is incredibly important for producing high-quality, user-focused
documentation such as release notes or a development roadmap. It should be
possible to collect this information before implementation begins, in order to
avoid requiring implementors to split their attention between writing release
notes and implementing the feature itself. Enhancement editors should help to ensure
that the tone and content of the `Summary` section is useful for a wide audience.

A good summary is probably at least a paragraph in length.

Both in this section and below, follow the guidelines of the [documentation
style guide]. In particular, wrap lines to a reasonable length, to make it
easier for reviewers to cite specific portions, and to minimize diff churn on
updates.

[documentation style guide]: https://github.com/kubernetes/community/blob/master/contributors/guide/style-guide.md
-->
This document details a generic control plane for Software Service Providers
(SSPs) and Alt Clouds. It enables extensible service offerings with minimal
overhead, featuring service discovery, standard APIs, CRD-based extensibility,
hierarchical resource model (Organizations, Folders, Projects), IAM with
inheritance, and telemetry integration (metrics, logs, traces).

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this Enhancement.  Describe why the change is important and the benefits to users.
-->
To accelerate SSP and Alt Cloud service delivery by providing a versatile,
robust, and extensible platform. This reduces foundational engineering burdens,
allowing providers to focus on core service logic. Standardized interfaces and
discovery simplify integration and consumption.

### Goals

<!--
List the specific goals of the Enhancement. What is it trying to achieve? How will we
know that this has succeeded?
-->
- Deliver a control plane for SSPs/Alt Clouds to efficiently offer services.
- Enable low-burden integration of new services via an extensible architecture.
- Automate discovery of services, resources, and APIs.
- Provide standardized platform capabilities:
    - Consistent API experience.
    - CRD-based API extensibility.
    - Hierarchical resource management (Organizations, Folders, Projects).
    - IAM with resource hierarchy inheritance.
    - Integrated telemetry (metrics, events/logs, traces).

### Non-Goals

<!--
What is out of scope for this Enhancement? Listing non-goals helps to focus discussion
and make progress.
-->
<!-- TODO: Define specific non-goals for this enhancement. -->

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->
Develop a generic control plane founded on the Kubernetes APIServer library
([`kubernetes/apiserver`][apiserver-lib]). This foundation is crucial for its
robust, battle-tested nature, providing core API machinery: RESTful resource
handling, versioning, validation, admission control, and OpenAPI schema
generation. It enables rapid development of new APIs and ensures a consistent,
Kubernetes-native experience. The generalization of the APIServer for such uses
is an ongoing effort within the Kubernetes community (see [KEP-4080][kep-4080]).
Adherence to this API model unlocks compatibility with a vast ecosystem of
existing Kubernetes tooling (e.g., `kubectl`, client libraries, IDE
integrations) for any generalized API, not just native Kubernetes types.

The control plane serves as the central API for **Service Users** to manage
resources and for **Services** to register, expose their APIs, and reconcile
their state. It will facilitate service registration and discovery. Registered
services leverage core platform functionalities built atop this Kubernetes
APIServer base:
- Unified API gateway and standard patterns inherent from the APIServer.
- CRD-based custom resource definition and extension, a native APIServer
  feature.
- Pre-defined resource hierarchy (Organizations, Folders, Projects).
- IAM with hierarchical permissions and inheritance, integrated with APIServer
  authentication and authorization mechanisms, and validated by the **IAM**
  system.

![Control Plane System Context Diagram](./system-context.png)

[apiserver-lib]: https://github.com/kubernetes/apiserver
[kep-4080]: https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/4080-generic-controlplane
[apiserver-runtime]: https://github.com/kubernetes-sigs/apiserver-runtime
[sample-apiserver]: https://github.com/kubernetes/sample-apiserver

<!-- ### User Stories (Optional) -->

<!--
Detail the things that people will be able to do if this Enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

<!-- #### Story 1 -->

<!-- #### Story 2 -->

<!-- ### Notes/Constraints/Caveats (Optional) -->

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

<!-- ### Risks and Mitigations -->

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

The Milo control plane is built on top of the [Kubernetes APIServer
library][apiserver-lib]. Milo leverages the APIServer library to provide a
foundation for building an extensible control plane that can be used to expose
APIs for services registered on the platform. The APIServer library provides all
of the core functionality required to build a control plane, including:
- RESTful resource handling
- Versioning
- Validation
- Admission control
- OpenAPI schema generation
- Authentication and authorization
- Storage layer
- Event handling
- Audit logging


### APIServer Architecture

The Milo Control Plane includes a resource hierarchy to enable organizations to
organize and manage their resources. The primary entities in the resources hierarchy
are:
- **Organizations**: A top-level entity that represents a group of users,
  projects, and resources.
- **Folders**: A sub-entity of an organization that represents a group of
  projects and resources. (Will be added in the future).
- **Projects**: A sub-entity of a folder or an organization that represents a
  group of resources.

The APIServer library does not natively support a resource hierarchy or
multi-tenant system. To support our resource hierarchy, we use multiple
deployments of the Milo APIServer. The deployments of these APIServers are
managed by [Kyverno] and [FluxCD].

- **Core APIServer**: The core APIServer is responsible for serving
  organization-level resources or global resources to configure the platform.
- **Project APIServer**: The project APIServer is responsible for serving
  project-level resources.

The API Gateway is configured to route requests to the appropriate APIServer
based on path prefix matching. Today a single etcd cluster is used to store all
state for resources managed by the control plane. In the future we will
investigate using [Kine] to store state in a traditional relational database to
improve scalability.

![apiserver-architecture](./apiserver-architecture.png)

> [!IMPORTANT]
>
> In the future we will invest in the Milo APIServer to virtualize the
> project-level APIServer and make it possible to serve multiple projects from
> a single APIServer instance.

[Kyverno]: https://kyverno.io/
[FluxCD]: https://fluxcd.io/
[Kine]: https://github.com/k3s-io/kine

### Authentication and Authorization

The Milo Control Plane system uses the [OIDC Authentication] and [webhook
authorization] capabilities built into the APIServer library to authorize
requests to resources managed by the control plane.

Our initial implementation supports authenticating users and machines with
[Zitadel] and authorizing requests with [OpenFGA].

These two components work together to provide a robust authentication and
authorization system that's hierarchy aware and can be used to authorize
requests across all resources managed by the control plane.

![authentication-and-authorization](./auth-architecture.png)

> [!NOTE]
>
> We're architecting the Milo Control Plane to support multiple authentication
> and authorization mechanisms to allow service providers to choose the best fit
> for their use case.

[OpenFGA]: https://openfga.dev/
[Zitadel]: https://zitadel.com/
[OIDC Authentication]: https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens
[webhook authorization]: https://kubernetes.io/docs/reference/access-authn-authz/webhook/

### Control PlaneTelemetry


![telemetry-architecture](./telemetry-architecture.png)

### Resource Creation Flow

The following sequence diagram illustrates the complete flow of creating a
resource through the Milo APIServer, including all validation, authorization,
and notification steps:

```mermaid
sequenceDiagram
  participant CLI as CLI Client
  participant API as APIServer
  participant IAM as IAM System
  participant MutatingWH as Mutating Webhook
  participant ValidatingWH as Validating Webhook
  participant Storage as Storage Layer (etcd)
  participant Watcher as Event Watchers
  participant Audit as Telemetry System

  Note over CLI,Audit: Resource Creation Request Flow

  CLI->>+API: POST /api/v1/namespaces/{ns}/resources
  Note right of CLI: Create resource request with manifest

  API->>Audit: Audit Stage: RequestReceived
  Note right of API: Audit: Request received by audit handler

  API->>API: Parse & validate request format
  Note right of API: Basic request validation

  API->>+IAM: SubjectAccessReview request
  Note right of IAM: RBAC/ABAC authorization check
  IAM-->>-API: Authorization result (allow/deny)

  alt Authorization denied
      API->>Audit: Audit Stage: ResponseComplete (403 Forbidden)
      API-->>CLI: 403 Forbidden
  else Authorization allowed
      API->>Audit: Log authorization success

      API->>+MutatingWH: Call mutating admission webhooks
      Note right of MutatingWH: Modify resource spec if needed
      MutatingWH-->>-API: Modified resource spec

      API->>+ValidatingWH: Call validating admission webhooks
      Note right of ValidatingWH: Validate resource against policies
      ValidatingWH-->>-API: Validation result (accept/reject)

      alt Validation failed
          API->>Audit: Audit Stage: ResponseComplete (422 Unprocessable Entity)
          API-->>CLI: 400 Bad Request / 422 Unprocessable Entity
      else Validation passed
          API->>Audit: Log validation success

          API->>+Storage: Store resource in etcd
          Note right of Storage: Persist resource with metadata
          Storage-->>-API: Storage confirmation

          API->>Audit: Audit Stage: ResponseComplete (201 Created)
          API-->>CLI: 201 Created (resource manifest)

          Note over API,Watcher: Event Notification Phase
          API->>+Watcher: Notify watchers (ADDED event)
          Note right of Watcher: Controllers, operators, etc.

          loop For each watcher
              Watcher->>Watcher: Process resource event
              Note right of Watcher: Reconcile desired state
          end

          deactivate Watcher

          API->>Audit: Log event notifications sent
      end
  end

  deactivate API

  Note over CLI,Audit: Request Complete
```

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
