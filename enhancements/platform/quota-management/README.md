---
status: provisional
stage: alpha
latest-milestone: "v0.1"
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

- [x] **Make a copy of this template directory.**
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

# Quota Management


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

This enhancement proposes the development and implementation of a comprehensive quota management system within the Datum Cloud platform. This system will empower platform administrators to define, enforce, and manage resource consumption limits for tenants at both the organizational and project levels. Datum employees will have the ability to view and modify these quota levels, while Datum Cloud users will be able to see their allocated quotas and current resource usage. The system aims to provide predictable capacity management, enable customer tier enforcement, offer transparency to customers regarding their resource limits, and include enforcement mechanisms to reject API requests that would exceed these limits. Furthermore, services offered on the Datum Cloud platform will be able to register the resources they manage for quota protection and set default quota levels.

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



## Motivation

The ability to create, observe, and self-manage resource quotas within organizations and their projects provides numerous benefits to both internal and external administrators of the system. By providing full transparency and observability into key metrics and their limits, quota management also ensures operational stability and reliability, enables accurate cost predictability, prevents accidental or abusive overuse, and instills confidence in resource planning and the enforcement of internal and regulatory policies. The safeguards put in place through quota management will enable users to fully explore the Datum Cloud ecosystem and variety of functionality it provides, without worrying about exceeding the thresholds that have been set within their organzation and projects.

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this Enhancement.  Describe why the change is important and the benefits to users.
-->

### Goals

- Provide clear system context and architectural approach to the creation of a quota management system within Datum Cloud, including new Custom Resource Definitions
- Outline the ability of **organizational and project administrators** to create and manage specific resource quota limits within their organization or projects.
- Outline the ability of **Datum Cloud platform administrators** to create and manage global quota limits applied to all organizations and projects within the system.
- Enable Datum Cloud users to view their allocated quota levels and the number of resources consumed against each quota.
- Define how services on the Datum Cloud platform can register resources for quota protection and set default quota levels.
- Ensure the system can enforce defined quota limits, for example, by rejecting API requests that would exceed these limits.
- Facilitate predictable capacity management for the platform.
- Support customer tier enforcement (e.g., free vs. paid tiers) through configurable quotas.
- Define the API design for services registering quotas and for managing quota thresholds for projects and organizations.
- Enable enhancement document handoff for implementation of the quota management enhancement within Datum Cloud.
- Remain service agnostic to not avoid tightly coupling the architecture to a specific SaaS vendor (amberflo, OpenMeter, etc)
- Define a `BillingAccount` CRD, as this definition is required as a part of registering new organizations within both Datum and the downstream SaaS integration handling billing and tracking resource allocations (amberflo, OpenMeter, etc)


<!-- 
- Enable full visibility into the consumption metrics of provisioned workloads running in Datum Cloud in relation to set quota limits (???) -->



### Non-Goals

- Provide detailed implementation specifics of how the metering and billing components of the system will work, outside of the acknowledgement of their overall role in system architecture from a quota management perspective. This includes how resource consumption is translated into actual billable units and invoices. 
      - An exception to this is creating a new `BillingAccount` custom resource that ties an organization to a billing account, which is a pre-requisite for downstream functionality.
-   Provide implementation specifics of any third-party SaaS integration in regards to quota management, metering and billing.
-   Define the future Milo Service Catalog and service registration.
-   Define the exact user interface (UI) mockups or user experience (UX) flows for managing or viewing quotas, beyond the functional requirements.
-   Define how time-series metrics (e.g. CPU hours, data written, etc) will be implemented by the data plane.
-   Define how alerts can be created and sent organizational and project administrators to inform them that they are approaching the quota threshholds they set for the resources. 

<!--
What is out of scope for this Enhancement? Listing non-goals helps to focus discussion
and make progress.
-->


## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

This enhancement proposes the design and architectural foundation for a quota management system integrated into Datum Cloud. The system will allow for the definition and enforcement of resource limits at both organizational and project levels by both external administrators as well as Datum employees.

### Definition of Success



### Key Components and Capabilities

1.  **`ResourceQuotaGrant` Definition:**
    -   Platform administrators can define global default quota grants for various resources.
    -   Organizational administrators can define organization-specific quotas (e.g., number of projects, number of collaborators/team-members).
    -   Project administrators can define project-specific quotas (e.g., number of workloads, CPU/memory limits, potentially leveraging Kubernetes ResourceQuotas for namespace-level control).
    -   Quotas will be designed to support different customer tiers and plans, allowing for varied limits based on subscription levels (e.g., "Free Tier gets 1 collaborator, Pro Tier gets unlimited").

2.  **Quota Enforcement:**
    -   The system will include mutating admission webhooks and a quota-controller reconciler to check against defined quotas during resource creation, modification, or deletion.
    -   Requests exceeding the defined limits will be rejected with appropriate error messaging.

3.  **Quota Visibility:**
    -   Datum Cloud users (tenants) will be able to view their current quota allocations and their consumption of resources against these quotas.
    -   Datum Employees (internal administrators) will have the ability to view and modify quota levels for all projects and organizations.

4.  **Service Integration:**
    -   Services offered on the Datum Cloud platform will need a mechanism to register the resources they manage that should be subject to quotas.
    -   Services will define default quota levels for the resources they register.
    -   An API will be designed for services to register these quotable resources and for the management (CRUD operations) of quota thresholds for projects and organizations.
    -   The integration of the downstream metering and billing SaaS platform will remain service-agnostic to ensure flexibility of implementation in the future.

5.  **Architectural Considerations:**
    -   Initially, [Kubernetes ResourceQuotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/) was explored for project-level resource control, while acknowledging its potential limitations for project-wide (cross-namespace) totals. Due to this limitation, Kubernetes ResourceQuotas was determined to not provide the functionality desired for cross-namespace resource quota management.
    -   The concept of "Resource Grants" tied to product plans, as discussed in issue #78, will be considered to dynamically configure quotas based on purchased plans.

**Proposal Deliverables**
- An updated enhancement document (this document) detailing the system architecture
- A Proof of Concept (POC) demonstrating functional quota system integration with a Datum Cloud project, and a complete API design for service quota registration and management.
- Information outlining on an architectural level how Quota Management will be integrated into Milo in the near future.
This enhancement proposal aims to provide system architecture of a new quota management system within Datum Cloud.

### User Stories (Optional)

<!--
Detail the things that people will be able to do if this Enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

#### Story 2

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

Different risks must be taken into account when considering implementation of the Quota Management system to ensure the system is working as expected and the risks are mitigated.

- The potential to block all resource creation, preventing administration of resources due to network failure or timeouts.
- The potential for actual resource usage being out-of-sync with the cluster-state, leading to:
      * Allowing allocation of resources beyond the set quota limits on both external and internal levels, bypassing enforcement.
      - Denying resource allocation when there are enough free resources to allow the request the ability to proceed.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

### Custom Resource Definitions 

Two main CRDs will be created as a direct part of the Quota Management implementation: `ResourceQuotaClaim` and `ResourceQuotaGrant`. Ancillary to the system, a third CRD will be created that represents billing accounts for organizations.

#### `ResourceQuotaClaim`

The `ResourceQuotaClaim` CRD represents the *intent* of the request to create, update/scale, or delete resources. This CRD contains the resources being requested, along with their quantity. The below `yaml` snippet shows an claim created by an incoming request tied to one Instance, which requests the additional allocation of 8 CPU cores and 32GiB of memory, along with the instance count:

```yaml
apiVersion: quota.datumapis.com/v1alpha1
kind: ResourceQuotaClaim
metadata:
  # Connect the claim’s lifetime to the workload that needs the quota
  name: instance-abc123-claim
  namespace: proj-abc 
  ownerReferences: 
  - apiVersion: compute.datumapis.com/v1
    kind: Instance
    name: instance-abc123
    uid: <uuid>
  finalizers:
  - quota.datumapis.com/usage-release
spec:
  # Resources being requested
  resources:
  - name: compute.datumapis.com/instances/cpu
    quantity: "8”
  - name: compute.datumapis.com/instances/memoryAllocated
    quantity: "32GiB"
  - name: compute.datumapis.com/instances/count
    quantity: "1"


Status:
  # Pending | Granted | Denied
  phase: Pending    
  grantedResources: []  
  reason: ""             
  observedGeneration: 1
```

#### `ResourceQuotaGrant`

The `ResourceQuotaGrant` CRD declares the resource limits for a project or organization (it can be bound to either via `spec.subjectRef`). More granularly, it declares which specific resources are limited by existing quotas, how the those limits vary by different dimensions (such as `dimensionLabels` and `buckets`), and how much of each resource is still free (which is maintained by the controller).

```yaml
apiVersion: quota.datumapis.com/v1alpha1
kind: ResourceQuotaGrant
metadata:
  name: proj-abc
spec:
  subjectRef:
    apiGroup: quota.datumapis.com
    kind: Project
    name: proj-abc

  limits:
  # 1. CPU cores allocated per project / location / instance type
  - name: compute.datumapis.com/instances/cpu
    # Dimension labels are the same as the resource name.
    dimensionLabels:
      - resources.datumapis.com/project
      - compute.datumapis.com/location
      - compute.datumapis.com/instanceType
    buckets:
    # Default limit for all locations and instance types across the project.
    # This is a global limit, and will apply to all locations, since we
    # don't specify a location dimension label within the bucket selector.
    - value: "1000"
      selector: {}
    # Override for multi-location projects.
    - value: "300"
      selector:
        matchExpressions:
        - key: compute.datumapis.com/location
          operator: In
          values: ["dfw", "lhr", "dls"]

    # Single location and instance type override,
    # more specific than the global limit and multi-location override.
    - value: "40"
      selector:
        matchLabels:
          compute.datumapis.com/location: dfw
          compute.datumapis.com/instanceType: c3.small

  # 2. Memory (GiB) allocated per project / location / instance type
  - name: compute.datumapis.com/instances/memoryAllocated
    dimensionLabels:
      - resources.datumapis.com/project
      - compute.datumapis.com/location
      - compute.datumapis.com/instanceType
    buckets:
    - value: "4096"
      selector: {}

    - value: "1024"
      selector:
        matchLabels:
          compute.datumapis.com/location: dfw

  # 3. Instance count per project / location / instance type
  - name: compute.datumapis.com/instances/count
    dimensionLabels:
      - resources.datumapis.com/project
      - compute.datumapis.com/location
      - compute.datumapis.com/instanceType
    buckets:
    - value: "200"
      selector: {}
    - value: "20"
      selector:
        matchLabels:
          instanceType: 
    - value: "5"
      selector:
        matchLabels:
          compute.datumapis.com/location: dfw
          compute.datumapis.com/instanceType: c3.small

  # 4. Gateway count per project
  - name: network.datumapis.com/gateways
    dimensionLabels:
      - resources.datumapis.com/project
    buckets:
    - value: "15"
      selector: {}

    - value: "3"
      selector:
        matchExpressions:
        - key: compute.datumapis.com/location
          operator: Exists

```


#### `BillingAccount`

```yaml

```

### Quota Operator Controller

A `quota-operator` will be created which implements logic to convert the *intent* of the incoming `ResourceQuotaClaim` object into the *actual allocation* of resources. It maintains accurate usage totals against each `ResourceQuotaGrant` via the `ResourceQuotaGrant.status.usage` field containing the totals of currently allocated resources. The controller is what enforces per-project or per-organization (tenant) resource quota limits that are declared in the `ResourceQuotaGrant` objects.

The reconciliation loop for this controller will contain the following logic:

1. Watches newly generated `ResourceQuotaClaim` objects that were automatically generated by the admission webhook whenever a new resource is created/scaled that will count against the quota.
2. Validates the `ResourceQuotaClaim` structure and fails early if it is incorrect.
3. Queries external source (e.g. amberflo) via API to determine the actual usage of the requested resource to accurately inform the decision to be made for the claim.
4. Makes ultimate decision on whether to `Grant` or `Deny` the claim request based on synced data with the external source.
5. Reconciles quota thresholds when a `ResourceQuotaGrant` limit changes (e.g. when an administrator raises or lowers quota limits, or the organization undergoes a plan/tier upgrade)
6. Exposes the resulting status of the request

**Failure Blast Radius**

If the controller is down, new Claims will stall; however workloads are not affected and there is no blocking of writes to the cluster.

### Admission Webhooks

A stateless mutating admission webhook will be created that serves as the initial interaction between the incoming claim request and the `quota-operator` controller. While this is webhook is optional, it provides additional safety of the workflow at claim creation time. 

The admission webhook is responsible for and executes the following steps:

1. Synchronously intercepts incoming requests for resource creation/scaling/deletion that are sent to the `quota-operator` controller.
2. Automatically creates a `ResourceClaimQuota` object based on the intent of the intercepted request.
3. Attaches labels and annotations to the object to enable easy claim deletion via a finalizer when workloads are torn down.

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