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

# Contact Management System for Datum OS

## Summary

The Contact Management System is a core component of Datum OS that provides service providers with a robust system for managing marketing contacts, including dynamic list management and opt-in functionality. This enhancement proposes a comprehensive solution for handling contact data, preferences, and engagement tracking while ensuring compliance with privacy regulations and marketing best practices.

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

The Contact Management System serves as the central system of record for all
marketing contacts within Datum OS, providing service providers with a
comprehensive solution for managing customer relationships and communications.
This system enables businesses to maintain accurate contact information, track
engagement, manage preferences, and ensure compliance with privacy regulations.
By integrating directly into Datum OS, it eliminates the need for external
contact management systems while providing seamless integration with other
platform features.

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this Enhancement.  Describe why the change is important and the benefits to users.
-->

A robust contact management system is essential for service providers operating in the modern digital landscape. By building this system directly into Datum OS, we provide several key benefits:

1. **Unified Data Management**: Eliminates data silos by centralizing contact information within the platform
2. **Privacy Compliance**: Built-in support for GDPR, CCPA, and other privacy regulations
3. **Seamless Integration**: Native integration with other Datum OS features like scheduling, billing, and service delivery
4. **Enhanced Customer Experience**: Enables personalized communication and service delivery
5. **Operational Efficiency**: Reduces manual data entry and maintenance by automating contact management workflows

### Goals

<!--
List the specific goals of the Enhancement. What is it trying to achieve? How will we
know that this has succeeded?
-->

1. **Contact Data Management**
   - Store and manage comprehensive contact profiles
   - Support multiple contact types (clients, prospects, vendors)
   - Enable custom fields and tags for flexible categorization

2. **List Management**
   - Create and manage dynamic contact lists
   - Support automated list updates based on contact attributes
   - Enable list segmentation for targeted communications

3. **Opt-in Management**
   - Track and manage communication preferences
   - Support multiple communication channels
   - Maintain audit trail of consent changes

4. **Integration Capabilities**
   - Provide APIs for external system integration
   - Support webhook notifications for contact updates
   - Enable data import/export functionality

5. **Compliance & Privacy**
   - Implement privacy-by-design principles
   - Support data retention policies
   - Enable automated compliance reporting

### Non-Goals

<!--
What is out of scope for this Enhancement? Listing non-goals helps to focus discussion
and make progress.
-->

1. **CRM Functionality**
   - Complex sales pipeline management
   - Advanced lead scoring
   - Sales forecasting

2. **Marketing Automation**
   - Email campaign management
   - A/B testing
   - Marketing analytics

3. **Social Media Management**
   - Social media posting
   - Social media analytics
   - Social media engagement tracking

4. **Advanced Analytics**
   - Predictive analytics
   - Machine learning-based insights
   - Complex reporting dashboards

5. **External System Replacement**
   - Not intended to replace existing CRM systems
   - Not meant to replace email marketing platforms
   - Not designed to replace customer support systems

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

### Contact Management Stories

<!--
Detail the things that people will be able to do if this Enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

1. **Contact Creation and Self-Service**
   - As a potential client, I want to create my contact profile through a public form so that I can start receiving communications from the service provider
   - As a service provider admin, I want to manually create contact profiles for clients who prefer offline registration
   - As a system, I want to validate contact information during creation to ensure data quality
   - As a system, I want to maintain an audit log of all contact creation activities

2. **Contact Information Management**
   - As a contact, I want to update my personal information after verifying my identity so that my profile stays current
   - As a contact, I want to delete my profile after verifying my identity so that I can exercise my right to be forgotten
   - As a service provider admin, I want to update contact information on behalf of clients who request assistance
   - As a system, I want to maintain an audit log of all contact information changes

3. **List Management**
   - As a service provider admin, I want to create contact lists with specific purposes and distribution methods so that I can organize contacts for targeted communications
   - As a service provider admin, I want to add contacts to lists manually or through automated rules so that I can maintain up-to-date communication groups
   - As a contact, I want to opt out of specific lists while remaining subscribed to others so that I can control my communication preferences
   - As a system, I want to maintain an audit log of all list membership changes

4. **Communication Preferences**
   - As a contact, I want to specify my preferred communication channels (email, SMS, etc.) so that I receive information in my preferred format
   - As a contact, I want to set my communication frequency preferences so that I don't receive too many messages
   - As a service provider admin, I want to view and respect contact communication preferences so that I can maintain positive relationships
   - As a system, I want to enforce communication preferences and channel restrictions

5. **Data Privacy and Compliance**
   - As a contact, I want to view all the data stored about me so that I can verify its accuracy
   - As a contact, I want to export my data in a machine-readable format so that I can exercise my data portability rights
   - As a service provider admin, I want to generate compliance reports so that I can demonstrate adherence to privacy regulations
   - As a system, I want to automatically handle data retention and deletion requests

6. **Integration and Automation**
   - As a service provider admin, I want to import contacts from external systems so that I can consolidate my contact management
   - As a service provider admin, I want to export contact data to external systems so that I can use other tools
   - As a system, I want to provide webhook notifications for contact changes so that other systems can stay synchronized
   - As a system, I want to support API-based access to contact management features so that custom integrations can be built

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

The Contact Management System will be implemented as a core service within Datum OS, providing both internal APIs and external interfaces for contact management functionality. The design follows Datum OS's microservices architecture while maintaining data consistency and privacy compliance.

### Data Models

#### Contact
```typescript
interface Contact {
  id: string;                    // Unique identifier
  tenantId: string;             // Datum OS tenant identifier
  firstName: string;
  lastName: string;
  title?: string;
  company?: string;
  email: string;                // Primary contact method
  phone?: string;
  socialProfiles?: {
    linkedin?: string;
    github?: string;
  };
  status: 'active' | 'inactive' | 'unsubscribed';
  preferences: {
    communicationChannels: {
      email: boolean;
      sms: boolean;
      social: boolean;
    };
    frequency: 'daily' | 'weekly' | 'monthly' | 'quarterly';
    timezone: string;
  };
  metadata: {
    createdAt: Date;
    updatedAt: Date;
    lastEngagement?: Date;
    source: 'manual' | 'import' | 'form' | 'integration';
    tags: string[];
  };
  auditLog: AuditLogEntry[];
}

interface AuditLogEntry {
  timestamp: Date;
  action: string;
  actor: string;
  details: Record<string, any>;
}
```

#### ContactList
```typescript
interface ContactList {
  id: string;
  tenantId: string;
  name: string;
  description?: string;
  type: 'static' | 'dynamic';
  criteria?: ListCriteria;      // For dynamic lists
  communicationMethod: {
    type: 'email' | 'sms' | 'social';
    settings: Record<string, any>;
  };
  integrations: {
    externalSystem: string;
    syncDirection: 'push' | 'pull' | 'bidirectional';
    lastSync?: Date;
    mapping: Record<string, string>;
  }[];
  metadata: {
    createdAt: Date;
    updatedAt: Date;
    contactCount: number;
    tags: string[];
  };
}

interface ListCriteria {
  conditions: {
    field: string;
    operator: 'equals' | 'contains' | 'greaterThan' | 'lessThan' | 'in' | 'notIn';
    value: any;
  }[];
  logic: 'and' | 'or';
}

interface ContactListMembership {
  id: string;
  tenantId: string;
  contactId: string;
  listId: string;
  status: 'active' | 'pending' | 'unsubscribed';
  metadata: {
    joinedAt: Date;
    updatedAt: Date;
    lastEngagement?: Date;
    source: 'manual' | 'import' | 'dynamic' | 'integration';
    notes?: string;
  };
  preferences: {
    // Override list-level communication preferences for this specific contact
    communicationChannels?: {
      email?: boolean;
      sms?: boolean;
      social?: boolean;
    };
    frequency?: 'daily' | 'weekly' | 'monthly' | 'quarterly';
  };
  auditLog: AuditLogEntry[];
}
```

### API Design

#### Core APIs

1. **Contact Management**
   ```typescript
   // Contact CRUD operations
   POST /api/v1/contacts
   GET /api/v1/contacts/:id
   PUT /api/v1/contacts/:id
   DELETE /api/v1/contacts/:id
   
   // Contact search and filtering
   GET /api/v1/contacts
   POST /api/v1/contacts/search
   
   // Contact preferences
   PUT /api/v1/contacts/:id/preferences
   GET /api/v1/contacts/:id/preferences
   
   // Contact audit log
   GET /api/v1/contacts/:id/audit-log
   ```

2. **List Management**
   ```typescript
   // List CRUD operations
   POST /api/v1/lists
   GET /api/v1/lists/:id
   PUT /api/v1/lists/:id
   DELETE /api/v1/lists/:id
   
   // List membership
   POST /api/v1/lists/:id/contacts
   DELETE /api/v1/lists/:id/contacts/:contactId
   GET /api/v1/lists/:id/contacts
   
   // List sync status
   GET /api/v1/lists/:id/sync-status
   ```

3. **Integration APIs**
   ```typescript
   // Webhook management
   POST /api/v1/webhooks
   GET /api/v1/webhooks
   DELETE /api/v1/webhooks/:id
   
   // Data export
   POST /api/v1/export
   GET /api/v1/export/:id/status
   GET /api/v1/export/:id/download
   ```

### Data Storage

The system will use Datum OS's distributed storage system with the following considerations:

1. **Primary Storage**
   - Contact and list data stored in a distributed document store
   - Optimized for read-heavy workloads
   - Built-in support for tenant isolation

2. **Audit Log Storage**
   - Separate storage for audit logs
   - Immutable append-only structure
   - Configurable retention policies

3. **Cache Layer**
   - Distributed cache for frequently accessed data
   - Cache invalidation on updates
   - Support for tenant-specific caching

### Security & Privacy

1. **Authentication & Authorization**
   - Integration with Datum OS's identity management
   - Role-based access control (RBAC)
   - API key management for external integrations

2. **Data Protection**
   - Encryption at rest and in transit
   - PII data masking in logs
   - Automatic data retention enforcement

3. **Privacy Compliance**
   - Built-in support for GDPR/CCPA requirements
   - Automated data subject request handling
   - Consent management and tracking

### Integration Points

1. **Internal Datum OS Services**
   - Identity Management
   - Notification Service
   - Analytics Engine
   - Workflow Engine

2. **External Systems**
   - Email Service Providers
   - SMS Gateways
   - Social Media Platforms
   - CRM Systems

### Monitoring & Observability

1. **Metrics**
   - Contact operations (CRUD)
   - List operations
   - API performance
   - Integration sync status
   - Error rates

2. **Logging**
   - Structured logging
   - Correlation IDs
   - Audit trail
   - Error tracking

3. **Alerts**
   - Integration failures
   - API latency spikes
   - Error rate thresholds
   - Data sync issues

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
