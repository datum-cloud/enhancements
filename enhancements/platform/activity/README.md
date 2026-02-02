---
status: provisional
stage: alpha
latest-milestone: "v0.1"
---

<!-- omit from toc -->
# Platform Activity

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Examples](#examples)
  - [User Stories](#user-stories)
  - [Key Capabilities](#key-capabilities)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)

## Summary

The Activity service provides consumers, service providers, and platform
administrators with visibility into changes and operations occurring across the
platform. The service transforms raw system events (audit logs, control plane
events, etc) into human-readable activity records, enabling users to understand
who made changes, what changed, and when changes occurred.

Activity records provide a complete history of operations performed within
organizations and projects. Users can view activity through the platform portal
or query activity programmatically through the API. The service supports
filtering, searching, and exporting activity data to meet debugging and
operational needs.

## Motivation

Understanding what is happening within a platform is essential for security,
troubleshooting, and operational visibility. Without a centralized activity
service, users must rely on fragmented logs across multiple systems, making it
difficult to answer fundamental questions like "who deleted this resource?" or
"what changes were made to my project last week?"

The Activity service addresses these needs by:

- **Providing transparency** - All platform operations are tracked and visible
  to authorized users
- **Enabling accountability** - Every action is attributed to a specific user,
  machine account, or system component
- **Accelerating troubleshooting** - Users can quickly trace the sequence of
  events that led to an issue

### Goals

- Provide human-readable activity records for all platform operations
- Enable consumers to view activity within their organizations and projects
- Enable service providers to monitor activity across their customer base
- Enable platform administrators to investigate issues and monitor system health
- Support filtering and searching activity by time range, actor, resource, and
  action type
- Provide activity data through both the portal UI and API
- Retain activity records for 31 days

### Non-Goals

- Real-time alerting or notifications based on activity (handled by separate
  alerting systems)
- Log aggregation from customer workloads (only platform-level operations)
- Metrics collection or performance monitoring
- Automated remediation or policy enforcement based on activity
- Compliance reporting (use raw audit logs for compliance requirements)
- Activity export (planned for a future release)

## Proposal

The Activity service captures operations performed across the platform and
presents them as human-readable activity records. Each record includes the actor
who performed the action, the resource affected, the type of change made, and
relevant metadata about the operation.

### Examples

The following examples illustrate how activity records tell the story of
resource changes over time.

#### DNS Zone Activity Timeline

This timeline shows typical operations a team might perform when setting up DNS
for a new domain.

| Time (UTC) | Actor | Activity |
|------|-------|----------|
| Jan 15, 9:00 AM | alice@example.com | Created public DNS zone **example.com** |
| Jan 15, 9:05 AM | alice@example.com | Granted **DNS Editor** role to bob@example.com on zone **example.com** |
| Jan 15, 9:10 AM | alice@example.com | Created **A** record for **example.com** pointing to 203.0.113.50 |
| Jan 15, 9:12 AM | alice@example.com | Created **CNAME** record for **www.example.com** pointing to example.com |
| Jan 15, 10:30 AM | bob@example.com | Created **A** record for **api.example.com** pointing to 203.0.113.51, 203.0.113.52 |
| Jan 15, 10:45 AM | bob@example.com | Created **MX** record for **example.com** with priority 10 pointing to mail.example.com |
| Jan 15, 2:15 PM | cert-manager | Created **TXT** record for **_acme-challenge.example.com** for certificate validation |
| Jan 15, 2:20 PM | cert-manager | Deleted **TXT** record for **_acme-challenge.example.com** |
| Jan 16, 8:00 AM | bob@example.com | Updated **A** record for **api.example.com**, changed TTL from 60 to 300 seconds |
| Jan 16, 3:30 PM | alice@example.com | Deleted **CNAME** record for **www.example.com** |

This timeline enables the team to:

- **Trace zone setup** - Understand who created the zone and established initial
  access
- **Review DNS changes** - See the sequence of record additions and
  modifications
- **Audit automated actions** - Verify that machine accounts like cert-manager
  performed expected operations
- **Investigate issues** - If DNS resolution fails, trace back through recent
  changes to identify potential causes

#### Proxy Activity Timeline

This timeline shows activity for configuring a proxy with routes and TLS
certificates.

| Time (UTC) | Actor | Activity |
|------|-------|----------|
| Jan 20, 10:00 AM | alice@example.com | Created proxy **web-proxy** in project **production** |
| Jan 20, 10:05 AM | alice@example.com | Granted **Proxy Editor** role to bob@example.com on proxy **web-proxy** |
| Jan 20, 10:15 AM | bob@example.com | Created route **/api/\*** on proxy **web-proxy** forwarding to backend **api-service:8080** |
| Jan 20, 10:20 AM | bob@example.com | Created route **/static/\*** on proxy **web-proxy** forwarding to backend **cdn-origin:443** |
| Jan 20, 10:30 AM | bob@example.com | Uploaded TLS certificate **example-com-cert** for domains example.com, www.example.com |
| Jan 20, 10:35 AM | bob@example.com | Attached certificate **example-com-cert** to proxy **web-proxy** |
| Jan 20, 11:00 AM | bob@example.com | Enabled health checks on route **/api/\*** with path /healthz and interval 30s |
| Jan 21, 9:00 AM | alice@example.com | Updated route **/api/\*** on proxy **web-proxy**, changed timeout from 30s to 60s |
| Jan 21, 2:00 PM | cert-manager | Renewed TLS certificate **example-com-cert**, new expiration Feb 20 |
| Jan 22, 11:30 AM | bob@example.com | Deleted route **/static/\*** on proxy **web-proxy** |

This timeline enables the team to:

- **Trace proxy configuration** - Understand how routes and backends were set up
- **Track certificate lifecycle** - See when certificates were uploaded,
  attached, and renewed
- **Review routing changes** - Identify modifications to routes and timeouts
- **Investigate traffic issues** - If requests fail, trace back through recent
  proxy and certificate changes

### User Stories

#### Consumer: Investigating unexpected changes

As a project member, I want to view recent activity in my project so I can
understand who made changes to resources and when those changes occurred.

**Example:** A developer notices that a network configuration changed
unexpectedly. They open the Activity view in the portal, filter by resource type
"Network", and discover that a colleague updated the configuration two hours
ago. The activity record shows exactly what fields changed.

#### Service Provider: Customer support

As a service provider support engineer, I want to view activity for a specific
customer organization so I can help troubleshoot issues they report.

**Example:** A customer reports that their deployment failed unexpectedly. The
support engineer views the customer's project activity and identifies that a
quota limit was reached just before the failure, helping them quickly resolve
the issue.

#### Service Provider: Usage monitoring

As a service provider, I want to understand how customers use my services so I
can identify adoption patterns and improvement opportunities.

**Example:** A service provider reviews aggregated activity data to understand
which API operations customers use most frequently, informing their product
roadmap priorities.

#### Platform Administrator: Incident investigation

As a platform administrator, I want to trace the sequence of operations that led
to a system issue so I can identify root cause and prevent recurrence.

**Example:** After an outage, a platform administrator queries activity across
all affected resources to reconstruct the timeline of events. They identify that
a configuration change triggered a cascade of failures.

### Key Capabilities

#### Activity Viewing

Users can view activity through the platform portal with an interface that
displays recent operations in chronological order. Each activity record
includes:

- **Timestamp** - When the action occurred (UTC)
- **Actor** - The user, machine account, or system component that performed the
  action
- **Activity** - A human-readable description of what occurred, including the
  resource affected and relevant details (e.g., "Created **A** record for
  **api.example.com** pointing to 203.0.113.50")

#### Filtering and Search

Users can filter activity by:

- **Time range** - View activity from a specific period (last hour, last day,
  custom range)
- **Actor** - Filter to actions by a specific user or machine account
- **Service** - Filter to actions from a specific service (DNS, Proxy, IAM,
  etc.)
- **Action type** - Filter by operation type (create, update, delete)
- **Source** - Filter by how the action was initiated (Portal, API, CLI,
  automation)
- **Outcome** - Filter by result (success, failure)
- **Full-text search** - Search activity descriptions for specific keywords or
  phrases

#### Retention

Activity records are retained for 31 days. Records older than 31 days are
automatically removed from the system.

#### Access Control

Activity visibility follows the platform's IAM model:

- Users see activity for resources they have permission to view
- Organization administrators see all activity within their organization
- Service providers see activity for their customers based on support agreements
- Platform administrators have visibility across the platform for operational
  needs

### Notes/Constraints/Caveats

#### Activity Availability

Activity records become available shortly after operations complete. Users
should expect a brief delay between performing an action and seeing it appear in
activity views. This delay is typically under one minute but may be longer
during periods of high platform activity.

#### Activity Record Immutability

Activity records cannot be modified or deleted. Platform administrators retain
the ability to manage activity policies, but individual records cannot be
selectively removed.
