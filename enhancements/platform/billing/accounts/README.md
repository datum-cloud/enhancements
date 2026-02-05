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

<!-- omit from toc -->
# Billing Accounts

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
  - [Billing Account Lifecycle](#billing-account-lifecycle)
    - [Account Creation and Activation](#account-creation-and-activation)
    - [Account Suspension and Reactivation](#account-suspension-and-reactivation)
    - [Account Closure](#account-closure)
  - [Binding Mechanics and Rules](#binding-mechanics-and-rules)
    - [Binding Creation and Validation](#binding-creation-and-validation)
    - [Immutability and Audit Trails](#immutability-and-audit-trails)
    - [Binding Transitions](#binding-transitions)
  - [Future Enhancement: Hierarchical Billing](#future-enhancement-hierarchical-billing)
  - [Payment Terms and Preferences](#payment-terms-and-preferences)
    - [Currency Configuration](#currency-configuration)
    - [Payment Terms Structure](#payment-terms-structure)
    - [Regional and Compliance Considerations](#regional-and-compliance-considerations)
- [Design Details](#design-details)
  - [Example BillingAccount Resource](#example-billingaccount-resource)
  - [Example BillingAccountBinding Resource](#example-billingaccountbinding-resource)
  - [Example Project Resource](#example-project-resource)
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

Introduce **Billing Accounts** as a new Milo platform primitive that identifies
the party responsible for service costs. This enhancement also adds
**project-to-account binding** so each project has a single, explicit payer.
Scope is limited to defining the Billing Account concept and enabling bindings;
it does not implement pricing, invoicing, or payments. The outcome is clear
ownership of costs and a consistent billing surface across all services.

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this Enhancement.  Describe why the change is important and the benefits to users.
-->

Milo currently lacks a platform-level way to declare financial responsibility
for usage. Teams need a simple, authoritative place to set who pays for a
project, support common commercial structures (departments, regions, resellers),
and track when responsibility changes. Establishing Billing Accounts and
bindings provides that foundation and enables rollups by account without
coupling to any specific billing backend.

### Goals

<!--
List the specific goals of the Enhancement. What is it trying to achieve? How will we
know that this has succeeded?
-->

- Make **Billing Account** the authoritative payer entity for the platform.
- Allow **multiple accounts per organization** to model departments, regions, or
  client accounts.
- Enable **one active binding per project** to make responsibility unambiguous.
- Support **account-level preferences** (e.g., currency and payment terms) to
  reflect business arrangements.
- Provide **account-scoped rollups** of usage across services for consolidated
  reporting.
- Ensure **binding lifecycle traceability** (when responsibility starts/ends)
  for audit and compliance.
- Improve **discoverability**: see all projects paid by an account, and the
  account paying a project.
- Defer **hierarchical billing** (reseller/sub-account scenarios) to a future
  enhancement while ensuring the design accommodates this extension.


### Non-Goals

<!--
What is out of scope for this Enhancement? Listing non-goals helps to focus discussion
and make progress.
-->

- Payment processing, tax calculation, invoicing, or integrations with payment
  gateways.
- Real-time rating/charging, price books, or SKU catalogs.
- Budgets, spending limits, credit checks, or commitment management.
- Detailed analytics/BI and deep metering design (covered by telemetry/metering
  work).
- Cross-organization sharing of Billing Accounts.
- Dispute handling, chargebacks, or revenue recognition.

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

The proposal introduces the **BillingAccount** as the core billing entity in
Milo's integrated service billing system. Billing accounts manage payment
profiles, currency preferences, and payment terms while serving as the billing
target for all service consumption charges. Projects within an organization's
resource hierarchy are linked to billing accounts through a separate
**BillingAccountBinding** resource, establishing the billing responsibility for
any resources consumed by services.

The system enforces clear financial ownership through explicit bindings,
supports hierarchical billing relationships for reseller scenarios, and
maintains comprehensive audit trails for compliance. Each project must have
exactly one active billing account binding, ensuring unambiguous financial
responsibility at all times.

<!-- ### User Stories -->

<!--
Detail the things that people will be able to do if this Enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

### Billing Account Lifecycle

The **BillingAccount** lifecycle ensures proper financial controls and clear
accountability throughout an account's existence:

#### Account Creation and Activation

Billing accounts begin in a "**Provisioning**" state when created. The system
validates required fields including currency code, payment terms, and initial
payment profile. Once validation completes, the account transitions to
"**Ready**" status, indicating it can accept project bindings. Accounts without
valid payment profiles remain in "**Incomplete**" status and cannot be bound to
projects.

#### Account Suspension and Reactivation

Accounts may be suspended for various reasons including payment failures,
compliance violations, or administrative actions. Suspension triggers:

- Status transition to "**Suspended**" with reason codes
- Notifications to all bound projects about potential service impacts
- Blocking of new project bindings
- Preservation of existing bindings for continuity

Reactivation requires resolving the underlying issue and administrative
approval. The account status history maintains a complete audit trail of all
suspensions and reactivations.

#### Account Closure

Closing a billing account requires:
- No active project bindings (all must be transitioned first)
- Settlement of outstanding charges
- Administrative approval for closure
- Archival of account data per retention policies

Closed accounts transition to "**Archived**" status and become read-only.
Historical data remains accessible for compliance and reporting purposes.

### Binding Mechanics and Rules

**BillingAccountBinding** resources establish the critical link between projects
and their financial responsibility:

#### Binding Creation and Validation

When creating a binding, the system enforces several invariants:

- **Single Binding Rule**: Each project may have at most one active binding
- **Organization Boundary**: Projects and billing accounts must belong to the
  same organization
- **Account Readiness**: Target billing account must be in "Ready" status
- **Project Existence**: Referenced project must exist and be active

The binding controller validates these rules atomically, rejecting invalid
bindings immediately. Successful bindings receive a unique identifier and
establishment timestamp for tracking.

#### Immutability and Audit Trails

Once created, bindings are immutable to maintain audit integrity. Key fields
like `projectRef` and `billingAccountRef` cannot be modified. This design:

- Ensures billing history remains intact
- Provides clear timeline of financial responsibility
- Satisfies compliance requirements for financial records
- Simplifies dispute resolution with immutable records

#### Binding Transitions

Changing a project's billing account requires creating a new binding rather than
modifying existing ones. The process involves:

1. Creating new binding with updated billing account reference
2. System validates no conflicts with existing active binding
3. Previous binding automatically transitions to "**Superseded**" status
4. New binding becomes active with fresh `establishedAt` timestamp
5. Both bindings retained for complete audit trail

This transition model ensures zero gaps in billing responsibility while
maintaining historical accuracy.

### Future Enhancement: Hierarchical Billing

A future enhancement will introduce hierarchical billing structures to support
sophisticated commercial arrangements including:

- **Reseller Models**: Parent accounts serving as payer of record for
  sub-accounts, enabling managed service providers and channel partners
- **Consolidated Billing**: Aggregating charges from multiple accounts for
  unified invoicing and payment
- **Sub-Account Management**: Creating child accounts that inherit payment
  responsibility while maintaining independent configurations

This capability will build upon the foundation established by this enhancement,
extending the billing account model to support parent-child relationships and
multi-level hierarchies. The design will ensure backward compatibility with
existing single-level billing accounts.

### Payment Terms and Preferences

Billing accounts encapsulate commercial terms that govern financial
transactions:

#### Currency Configuration

Each billing account specifies a `currencyCode` that determines:

- Invoice denomination for all bound projects
- Exchange rate application for multi-currency scenarios
- Reporting currency for financial statements
- Payment processing currency requirements

Currency codes follow ISO 4217 standards (USD, EUR, GBP, etc.) and are immutable
after account activation to prevent billing inconsistencies.

#### Payment Terms Structure

The `paymentTerms` object defines commercial payment agreements:

- **Net Days**: Payment due period (e.g., Net30, Net60, Net90)
- **Invoice Frequency**: Monthly, Quarterly, or Annual billing cycles
- **Invoice Day**: Specific day for invoice generation (1-28)
- **Early Payment Discounts**: Optional percentage for early settlement
- **Late Payment Penalties**: Interest rates for overdue balances

These terms flow through to invoice generation and payment processing systems,
ensuring consistent commercial treatment.

#### Regional and Compliance Considerations

Payment preferences may include region-specific requirements:

- Tax registration numbers for VAT/GST compliance
- Preferred invoice language and format
- Electronic invoicing standards
- Payment method restrictions per jurisdiction
- Data residency requirements for billing data

The system validates these preferences against regional compliance rules during
account configuration.

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


### Example BillingAccount Resource

```yaml
apiVersion: billing.milo.io/v1alpha1
kind: BillingAccount
metadata:
  name: acme-corp-us
  namespace: acme-org
  annotations:
    kubernetes.io/description: "Primary billing account for ACME Corp US operations"
spec:
  currencyCode: USD
  # Configures the payment terms of the account
  paymentTerms:
    netDays: 30
    # Monthly, Quarterly, etc
    invoiceFrequency: Monthly
    invoiceDayOfMonth: 1
  paymentProfile:
    creditCard:
      cardReference: cc-12345
status:
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2024-01-15T10:30:00Z"
    reason: BillingAccountActive
    message: "Billing account is active and ready for project linking"
  linkedProjectsCount: 8
```

### Example BillingAccountBinding Resource

```yaml
apiVersion: billing.milo.io/v1alpha1
kind: BillingAccountBinding
metadata:
  name: acme-corp-us-webapp-frontend-billing
  namespace: organization-acme
spec:
  projectRef:
    name: webapp-frontend
  billingAccountRef:
    name: acme-corp-us
status:
  conditions:
  - type: Bound
    status: "True"
    lastTransitionTime: "2024-01-15T14:22:05Z"
    reason: SuccessfulBinding
    message: "Project successfully bound to billing account"
  billingResponsibility:
    establishedAt: "2024-01-15T14:22:05Z"
    currentAccount: acme-corp-us
```

### Example Project Resource

```yaml
apiVersion: resources.milo.io/v1alpha1
kind: Project
metadata:
  name: webapp-frontend
  annotations:
    kubernetes.io/description: "Frontend services for the main web application"
spec:
  organizationRef:
    name: acme-corp
status:
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2024-01-15T14:22:00Z"
    reason: ProjectActive
    message: "Project is active"
  billingResponsibility:
    account: acme-corp-us
    establishedAt: "2024-01-15T14:22:05Z"
```

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
  - Will enabling / disabling the feature require downtime or reprovisioning of a node?
-->

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
  - Details:
-->

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
  - Details:
-->

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

- **8/21/2025** - Initial enhancement document created

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
