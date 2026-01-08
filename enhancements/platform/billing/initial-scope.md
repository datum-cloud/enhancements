---
status: provisional
stage: alpha
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

# Initial Scope for Billing Datum Cloud Consumers

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

- [Initial Scope for Billing Datum Cloud Consumers](#initial-scope-for-billing-datum-cloud-consumers)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals](#non-goals)
  - [Glossary](#glossary)
  - [Functional Requirements](#functional-requirements)
  - [Proposal](#proposal)
    - [Billing](#billing)
      - [Out of Scope](#out-of-scope)
    - [Telemetry](#telemetry)
      - [In Scope](#in-scope-1)
      - [Out of Scope](#out-of-scope-1)
    - [Service Catalog](#service-catalog)
    - [Price Book](#price-book)
    - [Quota](#quota)
    - [User Stories](#user-stories)
      - [Define service pricing](#define-service-pricing)
  - [Implementation History](#implementation-history)

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

This enhancement is focused on getting alignment around the functionality that
needs to be designed and developed in order to bill consumers of Datum Cloud's
services.

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this Enhancement.  Describe why the change is important and the benefits to users.
-->



### Goals

<!--
List the specific goals of the Enhancement. What is it trying to achieve? How will we
know that this has succeeded?
-->

- Define requirements for functionality that needs to be enabled to support
  billing consumer's of Datum Cloud services
- Identify dependencies on other systems / capabilities that need to be built
  into Milo to enable billing on Datum Cloud
- Get alignment around what we're billing for, how consumers will be billed, and

### Non-Goals

<!--
What is out of scope for this Enhancement? Listing non-goals helps to focus discussion
and make progress.
-->

- Define requirements for individual systems and components. Additional
  enhancements will be created to flesh our details of each component.

## Glossary

This glossary helps define terms used throughout the rest of the document that
readers may find helpful.

- **Service Provider**: A provider that is using Milo to offer their services to
  **Consumer**(s) (e.g. Datum Technology, Inc)
- **Consumer**: An entity (*Business* or *Individual*) that is consuming a
  service offered by a **Service Provider**.
- **Service**: A discrete service that is offered to **Consumer**(s) by a
  **Service Provider** (e.g. *Datum Cloud DNS*)

## Functional Requirements

This functional requirements provide clarity over functionality various users of
the platform would expect at the completion of this enhancement.

- A **Service Provider** can register a **Service** and configure the pricing
  that can be used to charge **Consumers**. The **Service Provider** can also
  configure every feature available with from the **Service** that may be
  enabled for **Consumers**.
- A **Service Provider** can charge a **Consumer** for a **Service** using
  one-time charges, recurring-charges, or usage-based charges.
- A **Service Provider** is able to configure usage reported by the
  **Telemetry** system to be billed to a **Consumer**.
- A **Consumer** is able to create a billing account for their organization and
  attach a payment profile that configures how they should be billed.
- A **Consumer** is able to attach a project to a billing account so any usage
  consumed by the project is billed correctly.
- A **Service Provider** is able to create multiple offers that **Consumers**
  can choose from to get access to **Services** (e.g. *Free*, *Pro*,
  *Enterprise*). An offer may provide access to multiple **Services** sold as a
  single offer and may offer a sub-set of features available with a service.
- A **Consumer** is able to view all publicly available offers so they can
  choose the best one that meets their needs.
- A **Consumer** will be invoiced at the end of each month that includes all
  charges from **Services** they consumed.

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

This proposal section covers the functionality or concepts that have been
identified as in scope or out of scope of this enhancement. This list is
expected to shift as we get alignment around the functionality offered.

### Billing

<!-- omit from toc -->
#### In Scope

- **Billing Accounts** are organization-level resources that define how an
  organization expects to be billed for **Services**. Organization admins will
  be required to create at least one billing account when setting up an
  organization. Existing organizations will need to create a billing account to
  avoid service disruption. Organization admins will also be required to
  associate projects with billing accounts to configure how a project's
  consumption is billed to the organization. Refer to the [billing account
  enhancement](https://github.com/datum-cloud/enhancements/pull/253/files) for
  more detail.

- **Entitlements** are resources tied to a billing account that determines what
  the billing account is entitled to consume from the platform. Entitlements
  contain the pricing configuration a billing account will pay for any
  **Service** it consumes. This includes one time charges, recurring static
  charges, and usage based charges.

- **Payment Profile** are resources associated with a billing account that
  defines how the billing account will pay for any charges attached to
  entitlements. For the scope of this effort, we will only support Credit Card
  based payment profiles. Future efforts may add support for ACH / Wire Transfer
  / Other payment methods.

- **Invoices** will be generated every month for each billing account. Invoices
  will contain line items for any charges the billing account owes for the
  billing period. Users will be expected to pay for invoices using the **Payment
  Profile** configured on their billing account.

- **Usage** will need to be collected by the **Telemetry** service and
  aggregated for billing purposes. For the scope of this effort, we will **not**
  support detailed usage reports. Users will only receive invoices with total
  usage per line-item.

#### Out of Scope

- **Commitments** are used to bind consumers to termed contracts, typically used
  when engaging with large / enterprise consumers. Commitments are often used to
  offer lower pricing for committing to longer-term deals. Without commitments,
  consumers will be able to terminate their entitlements at any time.

- **Budgets** allow billing account admins to configure threshold based alerts
  when spend limits are approaching or have been exceeded.

- **Multi-Currency** support will allows **Service Providers** to offer their
  **Services** in currencies other than **USD**.

- **Hierarchical Billing** will add support for sub-account billing for advanced
  reseller / partner use-cases.

- **Discounting** will enable **Service Providers** to offer their **Consumers**
  discounts on **Service Pricing**.

- **Detailed Reporting** would provide **Consumers** with detailed billing
  reports so their can understand their billing accounts usage and which
  projects and services consumed it.

### Telemetry

#### In Scope

- **Metric Definitions** will be defined by **Service Providers** for every
  metric that's available to **Consumers** of a **Service**. Metrics can be used
  for *operational visibility* into the service or be used for *billing usage*.
  This will be required so we know what metrics are available from a **Service**
  enabling a **Service Provider** to configure a price for the metric.
- **Metric Policies** will be defined by **Service Providers** for every metric
  that defines how metrics are produced from resources in the control plane or
  data-plane telemetry. See [policy-driven metrics
  platform](https://github.com/datum-cloud/enhancements/pull/252) for more
  information.
- **Metric Exporting** will allow systems like the **Billing** platform to
  subscribe to real-time metrics reporting from services so it can be consumed
  and sent to configured billing providers.

#### Out of Scope

### Service Catalog

- **Services** must be configured by **Service Providers** to register the
  services that will be available to their **Consumers**.

- **Service Pricing** must be configured for each **Service** a **Service
  Provider** is offering to their **Consumer**(s). Pricing must be configured
  for one-time charges, recurring charges, and usage based charges. Usage based
  charges will be associated with a **Metric Definition** to define how the
  metric is billed to **Consumers**.

### Price Book

- **Offers** configure how **Service Pricing** configurations can be combined
  into a single offer that can be accepted by consumers. Service providers may
  have multiple offers with varying level of usage / pricing (e.g. Free, Pro,
  Enterprise). **Consumers** will be expected to accept an offer which will
  result in an **Entitlement** being created for their billing account.

### Quota

- **Entitlements** would need to be integrated with the quota system to enable
  users access to features / resources after an entitlement has been activated
  on their account. This helps ensure that a user cannot consume a service until
  there's an active entitlement for the service. This _may_ require us to add
  support for feature flagging in the quota system.

### User Stories

<!--
Detail the things that people will be able to do if this Enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Define service pricing

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

<!-- ## Design Details -->

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

<!-- ## Production Readiness Review Questionnaire -->

<!--

Production readiness reviews are intended to ensure that features are observable,
scalable and supportable; can be safely operated in production environments, and
can be disabled or rolled back in the event they cause increased failures in
production.

See more in the PRR Enhancement at https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved
for the Enhancement to move to `implementable` status and be included in the release.
-->

<!-- ### Feature Enablement and Rollback -->

<!--
This section must be completed when targeting alpha to a release.
-->

<!-- #### How can this feature be enabled / disabled in a live cluster? -->

<!--
Pick one of these and delete the rest.
-->

<!--
- [ ] Feature gate
  - Feature gate name:
  - Components depending on the feature gate:
- [ ] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control plane?
  - Will enabling / disabling the feature require downtime or reprovisioning of a node? -->

<!-- #### Does enabling the feature change any default behavior? -->

<!--
Any change of default behavior may be surprising to users or break existing
automations, so be extremely careful here.
-->

<!-- #### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)? -->

<!--
Describe the consequences on existing workloads (e.g., if this is a runtime
feature, can it break the existing applications?).

Feature gates are typically disabled by setting the flag to `false` and
restarting the component. No other changes should be necessary to disable the
feature.
-->

<!-- #### What happens if we reenable the feature if it was previously rolled back? -->

<!-- #### Are there any tests for feature enablement/disablement? -->

<!-- ### Rollout, Upgrade and Rollback Planning -->

<!--
This section must be completed when targeting beta to a release.
-->

<!-- #### How can a rollout or rollback fail? Can it impact already running workloads? -->

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->

<!-- #### What specific metrics should inform a rollback? -->

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

<!-- #### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested? -->

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

<!-- #### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.? -->

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

<!-- ### Monitoring Requirements -->

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

<!-- #### How can an operator determine if the feature is in use by workloads? -->

<!--
Ideally, this should be a metric. Operations against the API (e.g., checking if
there are objects with field X set) may be a last resort. Avoid logs or events
for this purpose.
-->

<!-- #### How can someone using this feature know that it is working for their instance? -->

<!--
For instance, if this is an instance-related feature, it should be possible to
determine if the feature is functioning properly for each individual instance.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so
that they can verify correct enablement and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

<!--
- [ ] Events
  - Event Reason:
- [ ] API .status
  - Condition name:
  - Other field:
- [ ] Other (treat as last resort)
  - Details: -->

<!-- #### What are the reasonable SLOs (Service Level Objectives) for the enhancement? -->

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

<!-- #### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service? -->

<!--
Pick one more of these and delete the rest.
-->

<!--
- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details: -->

<!-- #### Are there any missing metrics that would be useful to have to improve observability of this feature? -->

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

<!-- ### Dependencies -->

<!--
This section must be completed when targeting beta to a release.
-->

<!-- #### Does this feature depend on any specific services running in the cluster? -->

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

<!-- ### Scalability -->

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

<!-- #### Will enabling / using this feature result in any new API calls? -->

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

<!-- #### Will enabling / using this feature result in introducing new API types? -->

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

<!-- #### Will enabling / using this feature result in any new calls to the cloud provider? -->

<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->

<!-- #### Will enabling / using this feature result in increasing size or count of the existing API objects? -->

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

<!-- #### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs? -->

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

<!-- #### Will enabling / using this feature result in non-negligible increase of resource usage in any components? -->

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

<!-- #### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)? -->

<!--
Focus not just on happy cases, but primarily on more pathological cases.

Are there any tests that were run/should be run to understand performance
characteristics better and validate the declared limits?
-->

<!-- ### Troubleshooting -->

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

<!-- #### How does this feature react if the API server is unavailable? -->

<!-- #### What are other known failure modes? -->

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

<!-- #### What steps should be taken if SLOs are not being met to determine the problem? -->

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

<!-- ## Drawbacks -->

<!--
Why should this Enhancement _not_ be implemented?
-->

<!-- ## Alternatives -->

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

<!-- ## Infrastructure Needed (Optional) -->

<!--
Use this section if you need things from another party. Examples include a
new repos, external services, compute infrastructure.
-->
