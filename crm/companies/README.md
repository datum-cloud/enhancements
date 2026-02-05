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

# CRM Companies Management

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
  - [User Stories](#user-stories)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [API Overview](#api-overview)
  - [Custom Resource Definitions](#custom-resource-definitions)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)

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

This enhancement introduces the **Companies** component for the Datum Cloud CRM system. The Companies module provides the capability to manage organizational entities, track their details (domains, LinkedIn profiles), categorize them via labels, and view associated people.

Like the People module, all Company resources are **cluster-scoped**, meaning they exist at the cluster level rather than being namespaced. This allows for a centralized view of all organizational relationships across the platform.

The data is intended to be manually managed by staff users rather than being auto-populated or linked by the system.

## Motivation

<!--
This section is for explixcitly listing the motivation, goals, and non-goals of
this Enhancement.  Describe why the change is important and the benefits to users.
-->

To effectively manage B2B relationships, the team needs a centralized record of the companies we interact with, distinct from the specific individuals (People) we talk to. Currently, there is no unified way to group contacts by their organization or track company-level details.

### Goals

<!--
List the specific goals of the Enhancement. What is it trying to achieve? How will we
know that this has succeeded?
-->

1. **Create a Company resource** that stores core firmographic information:
    - Company Name
    - Domains
    - Description
    - LinkedIn URL
    - Creator information
2. **Use standard Kubernetes labels** (`metadata.labels`) for tracking acquisition sources and segments. Common labels include:
    - `crm.milo.io/source=product-signup`
    - `crm.milo.io/source=newsletter-subscribe`
    - `crm.milo.io/source=referral`
    - `crm.milo.io/source=inbound-inquiry`
    - `crm.milo.io/source=website`
    - `crm.milo.io/source=existing-client-expansion`
    - `crm.milo.io/source=partner-introduction`
    - `crm.milo.io/source=organic-search`
    - `crm.milo.io/source=marketing-campaign`
    - `crm.milo.io/source=event`
3. **Establish relationships**:
    - Allow `Person` resources to reference `Company` resources.
    - Allow `Company` resources to optionally reference system `Organization` resources.
4. **Cluster-scoped management**: Centralized view across the platform.

### Non-Goals

<!--
What is out of scope for this Enhancement? Listing non-goals helps to focus discussion
and make progress.
-->

The following are explicitly out of scope for this enhancement:

- Automatic enrichment of company data (e.g., Clearbit integration).
- Automatic linking to system Organizations (Datum Cloud tenants).
- Hierarchy/Parent-Child company relationships (for now).
- Automatic duplicate detection.

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

This proposal introduces a `Company` Custom Resource Definition (CRD) to store firmographic data. This resource acts as the parent entity for `Person` resources (which reference the Company via `companyRef`).

### User Stories

<!--
Detail the things that people will be able to do if this Enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1: Creating a Company

As a staff member, I want to record a new potential customer company so I can group all contacts from that organization.

**Workflow:**

1. Create a `Company` resource.
2. Add labels like `crm.milo.io/source: inbound-inquiry`.
3. Add details like website domains and description.

#### Story 2: Viewing Company Details

As a staff member, I want to see an overview of a company and everyone we know there.

**Workflow:**

1. Retrieve the `Company` resource.
2. View metadata (Creation time, Creator).
3. List all `Person` resources where `spec.companyRef.name` matches this company.

### Notes/Constraints/Caveats (Optional)

1. **Standard Kubernetes Labels**: Instead of introducing a custom `CompanyLabel` CRD, we use standard Kubernetes `metadata.labels` for categorization. This approach:
   - Aligns with existing Kubernetes patterns and tooling
   - Allows flexible querying via label selectors
   - Enables the UI to query the telemetry system to discover which labels are in use across Company resources
   - Recommended label prefix: `crm.milo.io/`

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
software ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside of your immediate team.
-->

| Risk | Impact | Mitigation |
| ---- | ------ | ---------- |
| Name collisions | Two companies with similar names | Use unique `metadata.name` (slugs) and human-readable `spec.displayName`. |
| Data staleness | Manual data becomes outdated | Encourage staff updates; future integrations for enrichment. |
| Access control complexity | Sensitive company data access | Implement RBAC policies for CRM resources. |

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

### API Overview

| Resource | Kind | Description |
| -------- | ---- | ----------- |
| `companies` | `Company` | Represents a business entity. |

**Leveraging Existing Infrastructure:**

| Existing Resource/System | API Group / System | How It's Used |
| ------------------------ | ------------------ | ------------- |
| `notes` | `crm.milo.io` | Notes can reference a Company via `subjectRef` (extended to support Company) |

### Custom Resource Definitions

#### Company CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: companies.crm.milo.io
spec:
  group: crm.milo.io
  names:
    kind: Company
    plural: companies
    singular: company
  scope: Cluster
---
apiVersion: crm.milo.io/v1alpha1
kind: Company
metadata:
  name: acme-corp  # Unique slug
  labels:
    crm.milo.io/source: inbound-inquiry
    crm.milo.io/segment: existing-client-expansion
spec:
  displayName: "Acme Corporation"   # Required - Human readable name
  domains:                          # Optional - Associated domains
    - "acme.com"
    - "acme.io"
  description: "Global provider of anvils." # Optional
  linkedinProfile: "https://linkedin.com/company/acme-corp" # Optional
  organizationRef:                  # Optional - Reference to a system Organization
    name: "acme-verify-org"
  creatorRef:                       # Required
    name: staff-user-alice
status:
  conditions: []
```

### Example Resource

```yaml
apiVersion: crm.milo.io/v1alpha1
kind: Company
metadata:
  name: planet-express
  labels:
    crm.milo.io/source: referral
spec:
  displayName: "Planet Express, Inc."
  domains:
    - "planetexpress.com"
  description: "Interstellar delivery crew."
  linkedinProfile: "https://linkedin.com/company/planet-express"
  organizationRef:
    name: "planet-express-org"
  creatorRef:
    name: staff-user-fry
```

#### Using Existing Note CRD for Company Notes

The existing `Note` resource in `crm.milo.io` will be extended to support referencing a `Company` via `subjectRef`:

```yaml
apiVersion: crm.milo.io/v1alpha1
kind: Note
metadata:
  name: acme-corp-note-001
spec:
  subjectRef:
    apiGroup: crm.milo.io
    kind: Company
    name: acme-corp
  content: |
    Discussed potential partnership opportunities. They are interested in
    our enterprise tier features.
  creatorRef:
    name: staff-user-alice

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

<!--xrf
Pick one of these and delete the rest.
-->

- [x] Other
  - Describe the mechanism: CRDs are installed/uninstalled via kubectl or Helm.

#### Does enabling the feature change any default behavior?

<!--
Any change of default behavior may be surprising to users or break existing
automations, so be extremely careful here.
-->

No.

#### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

<!--
Describe the consequences on existing workloads (e.g., if this is a runtime
feature, can it break the existing applications?).

Feature gates are typically disabled by setting the flag to `false` and
restarting the component. No other changes should be necessary to disable the
feature.
-->

Yes, CRDs can be removed, but this results in data loss for Company resources.

#### What happens if we reenable the feature if it was previously rolled back?

Data is lost if CRDs were deleted; otherwise, it resumes.

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

Checking for existence of Company objects.

#### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is an instance-related feature, it should be possible to
determine if the feature is functioning properly for each individual instance.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so
that they can verify correct enablement and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

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

None.

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

- API call type: CRUD operations on Company resources.
- Estimated throughput: Low (manual entry).

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

No.

#### Will enabling / using this feature result in increasing size or count of the existing API objects?

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

No.

#### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

No.

#### Will enabling / using this feature result in non-negligible increase of resource usage in any components?

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

Minimal etcd storage (~2KB per object).

#### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

<!--
Focus not just on happy cases, but primarily on more pathological cases.

Are there any tests that were run/should be run to understand performance
characteristics better and validate the declared limits?
-->

No.

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

CRUD operations will fail.

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
