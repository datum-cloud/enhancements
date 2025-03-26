---
status: provisional
stage: alpha|beta|stable
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

# Enterprise Ready for Datum Cloud

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

The purpose of this enhancement is to collect, define, and prioritize a series of features related to "enterprise readiness" for Datum Cloud. Think of this as a broad meta issue with many small but related features.

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

Datum Cloud will be used by small, medium, and large organizations. Organizations of all sizes need various platform features for integrating a new product offering into their business lifecycle and IT systems. The expected impact of this enhancement is to reduce product selection and on-boarding friction that may be present in Datum Cloud.

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this Enhancement.  Describe why the change is important and the benefits to users.
-->

### Goals

<!--
List the specific goals of the Enhancement. What is it trying to achieve? How will we
know that this has succeeded?
-->

- Define key product features for Datum Cloud that will make our offering compelling to an Enterprise or Service Provider.
- The items listed in this enhancement are designed to reduce the effort
  associated with product adoption, product on-boarding and integration, and
  compliance/risk management. 

### Non-Goals

<!--
What is out of scope for this Enhancement? Listing non-goals helps to focus discussion
and make progress.
-->

- This enhancement should not be defining the core offering of Datum Cloud in
  any way.
- This enhancement purposefully deems the following areas of Enterprise Ready
  out of scope:
  - Product Assortment: Handled by the plethora of capability offered in Datum
    Cloud.
  - Change Management: Handled by the website / docs 2.0 project
  - Deployment Options: An intrinsic promise of Datum Cloud that did not seem
    appropriate for this 2025 Q2 fast follow initiative
  - Integrations: A wide topic that did not seem appropriate for this 2025 Q2
    fast follow initiative
  - GDPR: A wide topic that did not seem appropriate for this 2025 Q2 fast
    follow initiative

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

Using https://www.enterpriseready.io/#features as inspiration (and in some cases, a direct copy/paste of thier suggestions), we have selected 7x key platform features to implement to be "Enterprise Ready" with Datum Cloud.

Each of the 7x key platform features is defined below as a User Story.

### User Stories (Optional)

<!--
Detail the things that people will be able to do if this Enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

#### Single Sign On (SSO)

Single Sign On (SSO) allows our customers to manage their team’s users outside of our built-in user database. SSO centralizes the database of users into a single service that controls authorization to all accounts and applications.

Enterprises use a large number of services with a large number of end users. It is nearly impossible to manage hundreds, thousands or tens of thousands of users across tens or hundreds of different applications. For this reason, enterprises have been using Single Sign On (or directory services) to manage the provisioning, deprovisioning and permissioning of accounts and privileges for many years.

Detailed specification for SSO can be located in an existing enhancement: https://github.com/datum-cloud/enhancements/tree/main/enhancements/business-os/identity-and-access-management/authentication

#### Team Management

Teams describes the functionality that enables modern software to be collaborative and managed. Through Teams, users invite others to collaborative in application by creating or linking an account. Apps that don’t have team functionality often require password sharing for multiple users to access the account, which is an obvious security hurdle for most business users.

Key elements for teams feature:
- Create - during signup a team/organization is created & named by the initial user.
- Invite - emails sent to users to invite them to the team. Invites are accepted or declined.
- List - users should be able to view the team, see pending invites and resend invitations.
- Remove - users can be removed from teams and logged out of current sessions related to that team. 
- Resource discovery - users can find other users and resources created by those users easily.
- Profiles - each user has a profile that lists the resources they’ve created or contributed to.
- Access Review - it should be easy to review who has access to resources to support compliance audits.

Key elements for teams being applied to Datum Cloud Projects
- Add Permissions - It should be easy to assign permissions for a Datum Cloud project to a team
- View Permissions - Allow team members to view the permissions for an associated project.
- Remove Permissions - Allow administrators to remove permissions for a team from an associated project.

Possible advanced features for Teams:
- Ownership - resources are created by a single user and then shared with others. Ownership of a resource can be transferred to another user, including the bulk transfer of all owned resources when a user is deleted.
- Sub-teams - for large companies a single team would become too unwieldy. Instead, these enterprises will require that they can create a federated account of multiple teams with different settings (but likely unified billing) for each.

#### Role Based Access Control (RBAC)

Datum Cloud must have an authorization model that enables the administrator of the application to control the permissions that users have inside the app. This is colloquially known as “RBAC” (which stands for Role-Based Access Control), but often goes much deeper in terms of enterprise requirements.

Larger customers have compliance requirements such as SOC 2 or ISO 27001, requiring their admins to maintain fine-grained access control for their users. Along the same lines, enterprise customers often want audit trails of every authorization decision made by your application.

The minimal set of functionality to have an Enterprise authorization model:

- Define a permission for each distinct operation in your system (the cartesian product of verbs and nouns). This gives your authorization model room to evolve without any additional re-architecture.
- Define some common roles that are “roll-ups” of these permissions (RBAC), and allow customers to assign these roles to their users.
- Create at least one tier of “resource groups” (organizations, teams, projects) and allow admins to restrict roles/permissions to specific resource groups.
- Allow your customers to create custom roles, which are their own “roll-ups” of
  your permissions, fitting their business purposes.
- Offer your customers an audit trail of every authorization decision that your system makes around their user access.

Detailed specification for RBAC can be located in an existing enhancement: https://github.com/datum-cloud/enhancements/tree/main/enhancements/business-os/identity-and-access-management

#### Audit Logging

Audit logs are the centralized stream of all user activity within a team. Part of the security and compliance program of any large enterprise is designed to control and monitor the access of information within the organization. This drives the need for enterprise buyers to ask for a detailed audit trail of all activity that happens within their account. An audit trail can be used to prevent suspicious activity when it starts (if actively monitored), or to play back account activity during an incident review.

Audit logs ≠ Event logs: An audit logging feature is built from the ground up to log the relevant activity into a system that is immutable, time synced and accessible by account admins. Additionally, the best audit logs (as implemented by companies like GitHub and Google) are fully exportable, available from an API, searchable and include well documented changes.

Events to include in the audit log: 

- Authentication Events: Access granted, access rejected, account lock out
- User Events: Create, update, delete
- Team Events: Create, update, delete
- Project Events: Create, update, delete (at the project metadata level, not specific event logs within a project)
- API Key and Service Accounts: Create, update, delete
- Platform Events: Changes related to SSO, RBAC, or the Event Log configuration itself.

Fields to audit log for each event

When an event is logged it should have details that provide enough information about the event to provide the necessary context of who, what, when and where etc. Specifically, the following fields are critical to an audit log:

- Actor - The username, uuid, API token name of the account taking the action.
- Group - The group (aka organization, team, account) that the actor is a member of (needed to show admins the full history of their group).
- Where - IP address, device ID, country.
- When - The NTP synced server time when the event happened.
- Target - the object or underlying resource that is being changed (the noun) as well as the fields that include a key value for the new state of the target.
- Action - the way in which the object was changed (the verb).
- Action Type - the corresponding C``R``U``D category.
- Event Name - Common name for the event that can be used to filter down to similar events.
- Description - A human readable description of the action taken, sometimes includes links to other pages within the application.
- Version of the code that is sending the events.

Key functionality:
- Immutability: Data in an audit log should never change. Deleted objects should maintain a separately logged record of actions associated with the object (including its creation and deletion). External facing APIs should only be able to read the audit log, not write to it.
- Time Synced: Individual application audit logs will likely be combined with other audit log data, this makes it important to ensure that the timestamp on each event is accurate. The standard is to use the server time from a regularly NTP synced server, generally stored in GMT down to the millisecond.
- Exportable Logs: The activity should be exportable to a CSV format and API accessible so that it can be centralized into an organization wide SIEM logging system like Splunk. It’s advisable to offer both the ability to poll for new events and to be able to push new events to the remote system. When polling, use standards such as etag headers to prevent duplicate events from being received. When pushing, use standards such as webhooks to minimize the amount of custom work required to ingest these events.
- Account admin viewable: An audit log viewer should be embedded into the application to make the audit log accessible to enterprise account admins at all times. This viewer should be the central place to access all account activity logs.
- Searchable & filterable: Events should be indexed for searchability and filtering. Generally, actors, event names, IPs are all linked to filter down to related activity. The viewer should allow the account admin to specify a date range to filter in conjunction with other filters and searches.
- Change log: When new events and actions are captured in the audit log, it is important to publish the date at which each became available to appear in the audit log.
- Retention time: By default an audit log should generally be kept for 1-3 years. 

#### Reporting

If system logs are for developers and audit logs are for admin record keeping then reporting is for business. In other words, data related to the business purpose of the application, not the nuts and bolts.

Advanced reporting is an important feature for enterprise buyers because it allows them (or the administrator) to prove that the software in itself was effective (justifying the price they paid). It isn’t enough for software to be effective, it has to be provably effective. The more ways you enable customers to slice and manipulate the data that your application produces, the more likely that they’ll be able to prove effectiveness and derive value from your apps.

Reports Offered

- System objects defined: Report creation / deletion of system objects (workloads, instances, gateways, telemetry exporters, secrets) in a time series graph.
- Platform objects defined: Users, teams, API tokens created / deleted in a time series graph.

Reporting Features

- Export to different formats: Export to CSV or via API.
- Schedule/Email Reports: Some people want to get reports output via email. Define a screen that allows users to schedule a frequency, select parameters and select a format (you’ll need the export functionality mentioned in the previous section). Scheduling with time based parameters means you’re going to have to implement relative time logic (like give me data for last 7 days, where last 7 days is relative to when the report runs).
- Alerts on exceptions: Similar to scheduling, alerting is a schedule that only happens when a specific condition is run. This is an extension to the scheduling functionality above. Note that people will want to select the smallest increment for their alerts (i.e. check for the condition every 30 seconds) - with many users this will obviously slow down your database. 
- RBAC on Reports: Reports can only be run on resources to which the report request has "view" permissions.

#### Security and Trust

Datum Cloud will be responsible for critical business data, so security is paramount to our platform promises. 

Detailed specification for Security and Trust can be located in an existing enhancement: https://github.com/datum-cloud/enhancements/issues/54

Topical areas include:

- Software Development Lifecycle: Secure Development Practices, OWASP Top 10, CI/CD Pipeline Security, Code Reviews
- Platform Security: System to System Communication (TLS), Least Privilege, Rate Limiting, Secret Management, Passwords and Session Management
- Infrastrucuture Security: Network and System Hardening, Segementation, DDoS Resilience
- Data Security: Encryption at rest, least privilege, deletion, auditing.
- Physical Security:
- Vulnerability Scanning and Management
- Risk Management: Threat Modeling and Simulation
- Multi-Factor Authentication / Seperation of Duties

#### Support

To be Enterprise Ready, Datum must define levels of support it will offer. 

Types of Support:
- Community-based support: Generally for lower-tier customers, who will be able to ask questions of other community members in a forum.
- Email support: Email support is generally seen as a low-cost way to offer direct support to your customers, as you can create a queue of incoming emails and prioritize them based on the support contract.
- Phone support: Offered as a bit more of an “on-demand” or “instant-response” support option for customers who prefer to talk to someone immediately when they have a question.
- Dedicated customer success manager: A single point of contact for all account-related questions. Aside from answering questions and trying to resolve issues, this person serves as an advocate for the customer’s requests and issues within the SaaS provider.

Attributes of Support:
- Response Time based upon Severity
- Hours of Operation

Other support-related practices and offerings
- Documentation
- Change Log
- Product Roadmap
- Sample Code and Templates

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
