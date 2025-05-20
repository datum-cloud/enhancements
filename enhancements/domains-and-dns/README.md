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

# Domains and DNS

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
  - [Problem Space Exploration](#problem-space-exploration)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
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

A logical entry point to Datum's Services exists with a domain name and the
domain name system. However, domain name and DNS based workflow entry points
have historically created significant on-boarding barriers to IaaS, PaaS, and
SaaS applications due to the complexity of the DNS system, and its error prone
workflows. We seek to provide a streamlined method of domain on-boarding at
Datum.

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this Enhancement.  Describe why the change is important and the benefits to users.
-->

The purpose of this enhancement is to explore how a Datum Customer could
streamline their on-boarding to our platform with a domain name as the starting
point.

## Problem Space Exploration

When using SaaS applications like Loveable, Cloudflare, or Mailchimp, improper
DNS configuration becomes a blocking hurdle to progress. There are many reasons
why these hurdles exist, and the goal of this section is to help enumerate those
problems and where Datum can help solve them.

In other words, the "stars must align perfectly" to get a new or existing domain
to flawlessly on-board to a new service platform.

Some of the common problems include:

### Proof of Domain Ownership

Proof of domain ownership is designed to ensure that only authorized personnel
can configure a domain name on a platform such as Datum. Proof of domain
ownership generally uses an out of band validation technique, such as asking the
requesting party to place DNS TXT records in the domain in question, place a
specific nonce at a well known web service in the domain, or others. 

Allowing a domain to be hijacked by an unauthorized organization in the Datum
platform will erode trust with your customer base. For example, if Prospect
"Cyber Corp, Inc" is the rightful owner of the domain "cybercorp.com", and
existing Customer "Hacker Corp, LLC" adds cybercorp.com to their Datum
organization, the domain cybercorp.com will no longer be available for "Cyber
Corp, Inc." to use in our system. This is an example of a domain hijacking
attempt. 

### Domain Name Health

- As a service provider, I want to monitor the health of customer domains through WHOIS checks so that I can proactively identify and address potential issues
- As a service provider, I want to track domain expiration dates so that I can prevent accidental domain loss
- As a service provider, I want to monitor domain status flags (like clientTransferProhibited, clientDeleteProhibited) so that I can ensure proper domain security
- As a service provider, I want to verify nameserver configurations so that I can maintain proper DNS resolution
- As a service provider, I want to track domain modification dates so that I can detect unauthorized changes
- As a service provider, I want to receive alerts about critical domain status changes so that I can take immediate action

The WHOIS health check system will:
1. **Expiration Monitoring**
  - Track domain expiration dates
  - Calculate days until expiration
  - Alert on approaching expiration (30, 15, 7, 3, 1 days)
  - Flag domains with auto-renewal status

2. **Status Monitoring**
  - Track domain status flags:
    - clientTransferProhibited
    - clientDeleteProhibited
    - clientUpdateProhibited
    - serverTransferProhibited
    - serverDeleteProhibited
    - serverUpdateProhibited
    - pendingDelete
    - pendingTransfer
    - redemptionPeriod
  - Alert on status changes
  - Maintain status history

3. **Nameserver Verification**
  - Validate nameserver configurations
  - Check for proper delegation
  - Alert on misconfigurations

4. **Modification Tracking**
  - Monitor last modification date
  - Track registrar changes
  - Alert on modifications

### Improper DNS Configuration

Improper DNS configuration can contribute to platform setup problems. Typical
systems ask users to copy and paste DNS A, AAAA, CNAME, and TXT records from the
new platform into an existing DNS service provider. These steps are error prone
for users. Problems can include:

- Improperly copy/pasting values
- Placing a CNAME at the root of a domain name
- Creating new records with high TTLs, that are then cached (with incorrect
  data) upon DNS query

### Effects of DNS TTLs and "negative caching" (a.k.a DNS "propagation")

TTLs control the longevity of cached records in the DNS system. When an end
client queries it's recursive DNS server, if the records in question are not
available in the cache, the domain's authoritative server is queried for the
data. One of two things tends to happen after that: 

a) the records (hopefully with correct data) are fetched, and cached according
to the record's TTL; b) the record is not found, in which case an NXDOMAIN
response is given, and the negative response is cached by the recursive server
for the time period in the by the domain's negative cache TTL value (part of the
domain's SOA record); c) the same occurs, however, the negative cache TTL is set
to value defined as part of the local configuration of the recursive DNS server,
which may be longer than that of the domain's SOA record.

### Improper TLS/SSL Configuration

TLS certificates must be setup for all Subject Alternative Names (SANs) being
used by a client. If a CNAME is set into an endpoint, that endpoint must be
configured to accept traffic (using SNI / Host headers) and related TLS/SSL
certificates must be updated to accept such traffic.

### Reverse Proxy Backend is Not Ready for Traffic

For traffic ingress, any reverse proxy involved must be ready to accept traffic,
and hand that traffic to backends. Backends must be configured and also ready
for traffic.

### Configuration Validation

(Most) SaaS applications do not perform continuous configuration validation to
ensure that the desired data for the system is in place.

### Debugging Tools

DNS debugging tools, such as [WhatsMyDNS](https://www.whatsmydns.net/), are
useful for validating that authoritative DNS servers are vending proper zone
data around the world, but they do not provide full visibility of an end
client's state of the cache.

This creates opportunities where improper zone data or negative cached records,
could be present in an end user's local recursive, but these debugging tools do
not have awareness to the state of the user's cache. In other words, the
configuration might be totally valid and correct according to the debugging
tool, but the local state of cache causes the experience to be broken.

### Legacy DNS Service Platforms

As Datum offers a high performance Anycast edge, performance for a domain will
be constrained by the weakest performing authoritative DNS service in the chain
of delegation. Multiple legacy platforms do not use anycast DNS service,
creating high latency conditions.

## Goals

<!--
List the specific goals of the Enhancement. What is it trying to achieve? How will we
know that this has succeeded?
-->

- Identify common DNS related configuration mistakes that could impact setup of
  services on Datum Cloud.
- Identify 3rd party tools such as GoDaddy Domain Connect, CloudValid, and Entri
  to simplify domain setup on Datum Cloud. 


## Non-Goals

<!--
What is out of scope for this Enhancement? Listing non-goals helps to focus discussion
and make progress.
-->

- Creation of mitigations that require Datum to become a full ICANN-accredited
  registrar.
- Creation of a passive DNS monitoring service to identify each and every DNS
  change made.


## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

To mitigate common domain name and DNS configuration errors, Datum can provide a
series of common tools that can be used to set expectations about the DNS
configuration of a domain name. The proposal below attempts to create a series
of tools that could be used to inform a user about making DNS changes.

In the User Stories below, we describe a number of tools that should be able to
be run from Datum Cloud, Datum OS, and the Datum Website for debugging purposes.

Common commercial mitigation approaches to these problems include:

- [GoDaddy Domain Connect](https://www.domainconnect.org/)
- [CloudValid](https://www.domainconnect.org/)
- [Entri](https://www.entri.com/)

For reference, some best practice implementation examples include:

- [Let's Encrypt Challenges](https://letsencrypt.org/docs/challenge-types/)
- [February 2023 RFC Proposal to the DNSOp WG](https://datatracker.ietf.org/doc/draft-ietf-dnsop-domain-verification-techniques/)

Interesting related open source DNS projects:
- [DNSControl by StackExchange](https://dnscontrol.org/)
- [OctoDNS by Github](https://github.com/octodns/octodns)
- [DNSLexicon](https://github.com/dns-lexicon/dns-lexicon)

### User Stories

<!--
Detail the things that people will be able to do if this Enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

### Domain Name Ownership Verification Tool

As a Datum administrator, I want to verify ownership of domain names before they
are used in our platform. The tool will accept a domain name as input. The tool
will provide a nonce in the form of a TXT record to be placed at the APEX of the
domain. The tool will prompt the user to modify the domain so that the nonce
will be served from the TXT record. Once the user confirms changes have been
made, the tool will detect and query the authoritative DNS servers for the
domain to validate that the nonce was deployed as a TXT record, thereby
confirming domain ownership.

#### Domain Name Scanning Tool

As a user, I want a microservice that will scan and report information about my
domain name. The tool will accept an existing domain name is the input.

The output will include:

- WHOIS Data including Registrar and Nameservers
- Domain Name SOA data, including Negative Cache TTL. The negative cache TTL
  should have a prominent explanation about its role in new DNS record creation.
  If a record is queried in the DNS that does not exist, then that answer will
  be negatively cached up to the Negative Cache TTL. This may cause a user to
  believe that their data does not exist in the DNS after making DNS updates.

#### Datum Configuration Generator / Analyzer

As a user, I want to tell this tool about a specific Datum service (e.g.
instance of the gateway API). The tool will then lookup data in the Datum system
to determine the best practice DNS configuration (e.g. CNAMEs, etc), and query
the live DNS system without caching (e.g. dig +trace) to determine if DNS
records for that service are properly setup. If the service is not correctly
setup, the system will tell me the specific DNS records I need to go create with
recommended initial TTL values (along with consideration for expected time to
expire cache for existing TTLs or negative caching).

#### FQDN Analyzer

As a user, I want a microservice that will scan a given FQDN and tell me
interesting things about that hostname. The tool will accept an FQDN as input.

The output will include:

- Results of a non-cached DNS request made from Datum POPs around the world, to
  test for DNS record "propagation" differences. Return the data to the user
  with a summary of what is identified.
- If the record results in a CNAME or CNAME chain to be followed, collect and
  provide data about every step of the CNAME chain.
- Provide detailed data about DNS TTLs, and how making a record change could be
  impacted by having a high DNS TTL. Advise the user that if changes are being
  made, that we recommend dropping TTLs, waiting a TTL cycle, making the change,
  and then raising the TTL after the change is confirmed working.

#### DNS Cache vs. Authoritative Query Tool

As a user, I want to know if my authoritative DNS data is properly available in
various Recursive DNS platforms. Given an FQDN as input:

- Locate and query the authoritative DNS service for the FQDN.
- Globally query worldwide recursive DNS platforms, such as Cloudflare, Google
  DNS, Quad9, OpenDNS, etc.
- Compare the resulting data and present to the client.

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

Users will be able to add domain names to their inventory on Datum Cloud. Once a domain is added, its ownership will be verified using common domain ownership techniques (at the moment, we plan to use Entri). Once verified, in accordance with our User Stories above, Datum will be able to perform various health checks on those domains.

### Adding a Domain and Verifying Ownership

To start, a domain needs to be added to the system. The workflow to add a domain name is featured in the state diagram below:

```mermaid
sequenceDiagram
  autonumber

  participant User
  participant Portal
  participant Entri
  
  User ->> Portal: Add example.com
  Portal ->> Entri: Verify example.com
  Entri ->> User: Modal Dialog for example.com
  User ->> Entri: Run Entri Domain Ownership Sequence
  Entri ->> Portal: Domain Ownership Verified
  Portal ->> User: Domain example.com added to Inventory

```

Once a domain name is added to Datum Cloud, it is stored as a **Domain** object in our Kubernetes API service. An initial representation for a Domain object may look like:

```yaml
apiVersion: datumapis.com/v1alpha
kind: Domain
metadata:
  name: q7.io
spec:
  name: q7.io
```

### Gathering Initial Domain Name State

Once a domain is added a verified, details about the domain name should be fetched from public available WHOIS data and inserted into the Domain resource object. Datum will need to crawl WHOIS data, or find a method to obtain WHOIS data, to populate information into the Domain resource. The purpose of collecting this data is to make it available for viewing via API or Portal, and to be used to detection of events against the domain name (which ultimately could lead to outages or misconfiguration events).

For example, the domain q7.io would like the following after being crawled:

```yaml
apiVersion: datumapis.com/v1alpha
kind: Domain
metadata:
  name: q7.io
spec:
  name: q7.io
  registrarIANAID: 1068
  registrarIANAname: NAMECHEAP INC
  createdDate: 2012-07-06T13:58:57.00Z
  modifiedDate: 2019-08-17T16:33:24.36Z
  expirationDate: 2029-07-06T13:58:57.00Z
  nameservers: ns1.digitalocean.com, ns2.digitalocean.com, ns3.digitalocean.com
  DNSSEC: unsigned
  statusFlags: clientDeleteProhibited, clientTransferProhibited, clientUpdateProhibited
```

### Performing Routine Domain Name Health Checks

After gathering the initial domain state, we can now perform routine health checks for the domain as a convenience for Datum Cloud users. The sequence diagram below indicates this flow:

```mermaid
sequenceDiagram
  autonumber

  participant User
  participant Portal
  participant Cron
  participant Registry
  participant Event

  loop Hourly
    activate Cron
    Cron ->> Registry: Get domain example.com
    Registry ->> Cron: Details for example.com
    Cron ->> Portal: Check domain name data against Domain resource object
    Cron ->> Event: Create event if domain was modified
    deactivate Cron
  end
  User ->> Portal: View example.com
```

If a change is detected (compare the registry results to the data in the Domain resource object), then an event can be triggered to flag the change to a User. The Domain object should get updated to reflect the change, and the prior data should be stored for a period of time so that the user is aware of the change. 

Depending upon the capability of Datum's eventing systems, we may want to cause a real time notification to be triggered, but that should be out of scope at this time.

### Domain Inventory View

The domain inventory view will use a table showing all domain names available in the organization.

| Domain Name | Expiration  | Registrar     | DNS Provider   | Status | Last Change Detected | Actions     |
| ----------- | ----------- | ------------- | -------- | -------------- | ------------------- | ----------- |
| q7.io       | Jul 6, 2029 | NAMECHEAP INC | Digital Ocean | Normal | Aug 17, 2019         | Details |


- UX Details:
  - Sortable Columns: Name, Expiration, Status, Last Change Detected
  - Search & Filter: Text search by domain name; filter by name, registrar, DNS provider, or expiration date
  - Row Highlighting for Changes: Use a subtle yellow background tint (e.g., bg-yellow-50) on any row with detected WHOIS changes in the last 7 days

### Domain Detail View

View additional details about a domain name by clicking the "Details" button in the Inventory View. A possible view is:

```yaml
------------------------------------------
| q7.io                                  |
| Registered with: NAMECHEAP INC         |
| DNS Provider: DigitalOcean             |
| Last Scan: Aug 17, 2019   🔁 Rescan    |
------------------------------------------

[Tabs]: [Overview] [History] [Event Log]

Overview Tab:
------------------------------------------
| 🌐 Domain Info                         |
| Created: July 6, 2012                 |
| Modified: Aug 17, 2019               |
| Expiration: July 6, 2029             |
| Registrar IANA ID: 1068              |
| Status Flags:                        |
|  • clientDeleteProhibited            |
|  • clientTransferProhibited          |
|  • clientUpdateProhibited            |
| 🔧 Nameservers:                      |
|  • ns1.digitalocean.com              |
|  • ns2.digitalocean.com              |
|  • ns3.digitalocean.com              |
|  🔐 DNSSEC: Unsigned                 |
------------------------------------------
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
