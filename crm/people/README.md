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

# CRM People (Contacts) Management

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
  - [API Overview](#api-overview)
  - [Custom Resource Definitions](#custom-resource-definitions)
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

This enhancement introduces the **People (Contacts)** component for the Datum Cloud CRM system. The People module provides core contact management capabilities, enabling staff users to manually manage contact information, track acquisition sources through labels, manage communication subscriptions, log activities, and maintain notes for each contact.

All resources in this module are **cluster-scoped**, meaning they exist at the cluster level rather than being namespaced. This design decision aligns with the CRM's role as a centralized system for managing contacts across the entire organization.

The data is intended to be manually managed by staff users rather than being auto-populated or linked by the system. There is no automatic linking between People (contacts) and Users, or between Companies and Organizations—these are intentionally separate concepts.

## Motivation

<!--
This section is for explixcitly listing the motivation, goals, and non-goals of
this Enhancement.  Describe why the change is important and the benefits to users.
-->

Currently, there is no centralized system within Datum Cloud to manage contacts. This creates significant challenges for:

- **Tracking leads and contacts** — Sales and marketing teams have no unified view of people interacting with the platform
- **Understanding acquisition sources** — Without labels, we cannot measure which channels (waitlist, newsletter, events, etc.) drive the most engagement
- **Managing relationships** — No way to associate contacts with their companies or track interaction history

Building the People component will enable the team to effectively manage contact relationships and support growth.

### Goals

<!--
List the specific goals of the Enhancement. What is it trying to achieve? How will we
know that this has succeeded?
-->

1. **Create a Person resource** that stores core contact information:
   - Person Name (required)
   - Job Title (optional)
   - Company (optional)
   - Created By (required)
   - References to existing Contact resources (for email/subscription management)
   - Labels (via `metadata.labels`)

2. **Use standard Kubernetes labels** (`metadata.labels`) for tracking contact acquisition sources:
   - Leverages existing Kubernetes label semantics and tooling
   - Common labels include: `crm.milo.io/source=website-waitlist`, `crm.milo.io/source=newsletter-signup`, etc.
   - Fully extensible without requiring new CRDs
   - UI can query telemetry system to discover all labels in use across Person resources

3. **Leverage existing Contact CRD** (`notification.milo.io`) for email addresses and subscription management:
   - Person resources reference existing Contact resources
   - Contacts handle integration with email providers (Resend, Loops)

4. **Use Loki/activity logs** for tracking product-level actions and events associated with people:
   - Leverages existing activity log infrastructure used for staff user views
   - No new CRD required for activity tracking

5. **Leverage existing Note CRD** (`crm.milo.io`) for storing staff-entered information about people:
   - Extend Note's SubjectReference to support Person kind
   - Reuse existing note features (follow-up actions, creator tracking)

6. **Ensure new resources are cluster-scoped** for centralized management

### Non-Goals

<!--
What is out of scope for this Enhancement? Listing non-goals helps to focus discussion
and make progress.
-->

The following are explicitly out of scope for this enhancement:

- Deal/Opportunity pipeline functionality
- Email sending or inbox integration
- Calendar/meeting integration
- Advanced analytics dashboards
- Bulk import/export functionality
- Automatic duplicate detection
- Custom field builder
- Third-party integrations (Salesforce, HubSpot, etc.)
- Native mobile application
- Complex automation rules
- Automatic linking between People and Users
- Automatic linking between Companies and Organizations

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

This proposal introduces a set of Kubernetes Custom Resource Definitions (CRDs) to manage People (contacts) within the Datum Cloud CRM. The resources are designed to be:

1. **Cluster-scoped** — All resources exist at the cluster level for centralized management
2. **Manually managed** — Staff users create and maintain contact data; no automatic population
3. **Extensible** — Labels and subscriptions can be extended as business needs evolve
4. **Relational** — Resources reference each other (e.g., Company references Person, Activity references Person)

### User Stories (Optional)

<!--
Detail the things that people will be able to do if this Enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1: Adding a New Contact

As a staff member, I want to add a new person to the CRM so that I can track their information and interactions with our platform.

**Workflow:**

1. Create a `Person` resource with name and email addresses
2. Apply relevant labels via `metadata.labels` (e.g., `crm.milo.io/source: newsletter-signup`)
3. Add subscriptions for communication lists they've opted into
4. Add initial notes about how we met or relevant context

#### Story 2: Tracking Person Activity

As a staff member, I want to log activities for a person so that the team has visibility into their engagement with our platform.

**Workflow:**

1. Create a `PersonActivity` resource referencing the Person
2. Specify activity type (e.g., "demo-requested", "trial-started", "support-ticket")
3. Add details and timestamp
4. View all activities for a person in chronological order

#### Story 4: Viewing Contact Details

As a staff member, I want to view all information about a contact in one place so that I can understand their full relationship with our company.

**Workflow:**

1. Retrieve the `Person` resource
2. List all `PersonActivity` resources for that person
3. List all `Notes` resources for that person
4. List all `ContactGroupMemberships` resources for that person

### Notes/Constraints/Caveats (Optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

1. **No User/Organization Linking**: The naming convention "People" (not "Users") and "Companies" (not "Organizations") was intentionally chosen to avoid confusion. These CRM entities are separate from system users and organizations.

2. **Manual Data Entry**: All data is entered by staff based on their knowledge of companies and people. This is by design—we want precise, curated data rather than auto-populated information.

3. **Standard Kubernetes Labels**: Instead of introducing a custom `PersonLabel` CRD, we use standard Kubernetes `metadata.labels` for categorization. This approach:
   - Aligns with existing Kubernetes patterns and tooling
   - Allows flexible querying via label selectors
   - Enables the UI to query the telemetry system to discover which labels are in use across Person resources
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

| Risk                                     | Impact                                                 | Mitigation                                                                                                           |
| ---------------------------------------- | ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------- |
| Label sprawl with too many custom labels | Difficult to analyze and report                        | Define recommended label key conventions (e.g., `crm.milo.io/source`, `crm.milo.io/status`); document best practices |
| No duplicate detection                   | Same contact may be entered multiple times             | Document best practices; consider future enhancement for duplicate detection                                         |
| Access control complexity                | Sensitive contact data may be accessed inappropriately | Implement RBAC policies for CRM resources                                                                            |

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable. This may include API specs (though not always
required) or even code snippets. If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

### API Overview

The People module introduces the following cluster-scoped resource under the `crm.milo.io` API group:

| Resource  | Kind     | Description                                                         |
| --------- | -------- | ------------------------------------------------------------------- |
| `people`  | `Person` | Core person resource with name, job title, contact references, etc. |

**Leveraging Existing Infrastructure:**

| Existing Resource/System | API Group / System     | How It's Used                                                                  |
| ------------------------ | ---------------------- | ------------------------------------------------------------------------------ |
| `notes`                  | `crm.milo.io`          | Notes can reference a Person via `subjectRef` (extended to support Person)     |
| `contacts`               | `notification.milo.io` | Person references existing Contact resources for email/subscription management |
| Activity Logs            | Loki                   | Activity tracking for people uses existing Loki/activity log infrastructure    |

**Note on Labels**: Categorization uses standard Kubernetes `metadata.labels` on the `Person` resource. The UI can query the telemetry system to discover all labels currently in use.

### Custom Resource Definitions

#### Person CRD

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: people.crm.milo.io
spec:
  group: crm.milo.io
  names:
    kind: Person
    plural: people
    singular: person
  scope: Cluster
---
# Person Resource Structure
apiVersion: crm.milo.io/v1alpha1
kind: Person
metadata:
  name: <person-name>
  labels:
    crm.milo.io/source: <acquisition-source>  # e.g., newsletter-signup, website-waitlist
    crm.milo.io/event: <event-name>           # e.g., altcloud-meetup-2024
    crm.milo.io/status: <status>              # e.g., active, lead, customer
spec:
  name: string                    # required - Full name of the person
  jobTitle: string                # optional - Job title
  creatorRef:                     # required - Reference to the user that created this person
    name: string                  # required - Name of the User resource
  userRef:                        # optional - Reference to an existing User if this person is also a system user
    name: string                  # required - Name of the User resource
  companyRef:                     # optional - Reference to the Company this person belongs to
    name: string                  # required - Name of the Company resource
  contactRefs:                    # optional - References to existing Contact resources
    - name: string                # required - Name of the Contact resource
      namespace: string           # required - Namespace of the Contact resource
      type: string                # optional - Type of contact: work, personal, other
      primary: boolean            # optional - Whether this is the primary contact
status:
  createdBy: string               # Email of the user that created this person
  conditions: []                  # Standard Kubernetes conditions
```

#### Activity Tracking via Loki

Activity tracking for people uses the existing Loki/activity log infrastructure that is already in place for staff user views. Activities are logged with the person's resource name as a label, enabling queries like:

```logql
{app="crm"} | json | person="john-doe-acme"
```

This approach:

- Avoids creating a new CRD for activity tracking
- Leverages existing observability infrastructure
- Provides the same activity view experience as staff user activity logs

#### Using Existing Note CRD for Person Notes

The existing `Note` resource in `crm.milo.io` already supports referencing different subjects. We will extend the `SubjectReference` to include `Person`:

```yaml
# Note referencing a Person (using existing Note CRD)
apiVersion: crm.milo.io/v1alpha1
kind: Note
metadata:
  name: john-doe-note-001
spec:
  subjectRef:
    apiGroup: crm.milo.io         # Extended to support crm.milo.io
    kind: Person                  # Extended to support Person
    name: john-doe-acme
  content: |
    Met John at Alt Cloud Meetup in San Francisco. He mentioned they're
    evaluating cloud platforms for their infrastructure migration project.
  creatorRef:
    name: staff-user-alice
  followUp: true
  nextAction: "Schedule demo call"
  nextActionTime: "2024-07-01T10:00:00Z"
```

#### Using Existing Contact CRD for Email/Subscriptions

The existing `Contact` resource in `notification.milo.io` handles email addresses and subscription management. Person resources reference Contacts:

```yaml
# Existing Contact resource (notification.milo.io)
apiVersion: notification.milo.io/v1alpha1
kind: Contact
metadata:
  name: john-doe-work-email
  namespace: crm-contacts
spec:
  email: "john.doe@acme.com"
  givenName: "John"
  familyName: "Doe"
```

### Example Resources

#### Example Person

```yaml
apiVersion: crm.milo.io/v1alpha1
kind: Person
metadata:
  name: john-doe-acme
  labels:
    crm.milo.io/source: newsletter-signup
    crm.milo.io/event: altcloud-meetup-2024
    crm.milo.io/status: active
spec:
  name: "John Doe"
  jobTitle: "VP of Engineering"
  creatorRef:
    name: staff-user-alice
  userRef:                          # Optional - if this person is also a system user
    name: john-doe-user
  companyRef:                       # Optional - if this person is associated with a company
    name: acme-corp
  contactRefs:
    - name: john-doe-work-email
      namespace: crm-contacts
      type: work
      primary: true
    - name: john-doe-personal-email
      namespace: crm-contacts
      type: personal
      primary: false
```

#### Example Note (referencing Person)

```yaml
apiVersion: crm.milo.io/v1alpha1
kind: Note
metadata:
  name: john-doe-acme-note-001
spec:
  subjectRef:
    apiGroup: crm.milo.io
    kind: Person
    name: john-doe-acme
  content: |
    Met John at Alt Cloud Meetup in San Francisco. He mentioned they're
    evaluating cloud platforms for their infrastructure migration project
    scheduled for Q1 2025. Key decision maker along with their CTO.
  creatorRef:
    name: staff-user-alice
  followUp: true
  nextAction: "Schedule follow-up demo"
  nextActionTime: "2024-07-15T14:00:00Z"
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
  - Describe the mechanism: CRDs are installed/uninstalled via kubectl or Helm. The feature is enabled when CRDs are present and the CRM controller is deployed.
  - Will enabling / disabling the feature require downtime of the control plane? No
  - Will enabling / disabling the feature require downtime or reprovisioning of a node? No

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

TBD - Metrics will be defined in a future iteration.

#### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is an instance-related feature, it should be possible to
determine if the feature is functioning properly for each individual instance.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so
that they can verify correct enablement and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

TBD - Will be defined in a future iteration.

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

TBD - Will be defined in a future iteration.

#### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

TBD - Will be defined in a future iteration.

#### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

TBD - Will be defined in a future iteration.

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

- **Telemetry System** (optional)
  - Usage description: The UI queries the telemetry system to discover which labels are in use across Person resources for dynamic filtering
  - Impact of its outage on the feature: UI label discovery/filtering may be limited; core Person CRUD operations unaffected
  - Impact of its degraded performance or high-error rates on the feature: Slower label discovery in UI

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

- API call type: CRUD operations on Person-related resources
- Estimated throughput: Low (manual data entry by staff)
- Originating component(s): CRM Controller, UI/CLI

#### Will enabling / using this feature result in introducing new API types?

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

- `Person` - Estimated 1,000-10,000 per cluster (uses standard `metadata.labels` for categorization)

**Existing resources/infrastructure leveraged (no new objects created by this feature):**

- `Note` (`crm.milo.io`) - Extended to reference Person; estimated 1-20 notes per Person
- `Contact` (`notification.milo.io`) - Referenced by Person; no additional objects created
- Activity Logs (Loki) - Activity tracking uses existing log infrastructure

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

The existing `Note` CRD will be extended to support `Person` as a subject kind. No size increase for existing resources.

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

Minimal increase:

- etcd storage for Person resources (~5KB per Person average)
- Controller memory for watches (~10MB for 10K people)

#### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

<!--
Focus not just on happy cases, but primarily on more pathological cases.

Are there any tests that were run/should be run to understand performance
characteristics better and validate the declared limits?
-->

No. Resources are stored in etcd, not on nodes.

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

CRUD operations will fail until the API server is available. No data loss occurs.

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

- 2026-01-12: Initial enhancement proposal created

## Drawbacks

<!--
Why should this Enhancement _not_ be implemented?
-->

1. **Additional Complexity**: Introduces 1 new CRD (Person) and extends the existing Note CRD
2. **Manual Data Entry**: Requires staff time to maintain contact data
3. **No Integration**: Does not integrate with existing contact data sources
4. **Cluster-Scoped**: May complicate multi-tenant scenarios in the future

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

### Alternative 1: Use an External CRM

Instead of building custom CRM functionality, use an external service like Salesforce or HubSpot.

**Rejected because:**

- Vendor lock-in
- Cost at scale
- Limited customization for our specific workflows
- Data sovereignty concerns

### Alternative 2: Namespace-Scoped Resources

Make resources namespace-scoped for better isolation.

**Rejected because:**

- CRM data is organization-wide, not team-specific
- Simpler access control with cluster-scoped resources
- Matches the centralized nature of CRM

### Alternative 3: Single Resource with Embedded Data

Use a single `Person` resource with embedded labels, activities, and notes.

**Rejected because:**

- Resource size would grow unbounded
- Harder to query activities/notes independently
- More flexible to have separate resources with references

## Infrastructure Needed (Optional)

<!--
Use this section if you need things from another party. Examples include a
new repos, external services, compute infrastructure.
-->

- CRM Controller deployment
- Validation webhooks for reference checking
- RBAC policies for CRM resources
- Optional: UI/dashboard for managing People
