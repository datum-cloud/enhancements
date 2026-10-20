---
status: provisional
stage: alpha
latest-milestone: "TBD"
---

# Apps

## Summary

Apps give customers one place to deploy and manage a complete application on
Datum Cloud. An **App Template** is a versioned recipe that describes the
resources an application needs, the services it uses, and the choices a customer
must make. An **App** is a project-scoped deployment of that recipe. A
customer selects a template, supplies a few inputs, reviews the resources and
expected costs, and deploys. Datum then shows one app-level view of readiness,
endpoints, usage, and changes across the underlying resources.

The first use case is a web API: a Workload, an HTTP route through a Gateway,
a domain, and an optional connection to a managed service. The same model should
also support simpler apps, such as a Function with a public endpoint, as those
services become available.

## Motivation

Customers currently assemble compute, networking, DNS, and service connections
one resource at a time. They must understand creation order, carry values
between services, and inspect several pages to answer whether their application
is ready. Repeating a known deployment is especially tedious. Service providers
also lack a way to turn a working combination of capabilities into a repeatable
customer experience.

The Service Catalog helps customers
discover and enable individual services. App Templates build on those services
to describe a working application; they do not create a new service type or
replace the services' own APIs.

Teams should be able to turn an application they have tested in a project into
a reusable offering that other projects can discover and install.

### Goals

- Deploy a useful multi-resource application through one guided flow or a
  declarative request.
- Let template authors compose Datum resources and consume outputs from Apps in
  other projects through explicit inputs and outputs.
- Give customers one app-level view of deployment progress, failures, endpoints,
  component resources, and estimated and actual spend.
- Make updates, rollbacks, and deletion predictable while keeping each
  underlying resource inspectable in its native service.
- Let an App consume approved outputs from another App without changing either
  App's ownership or lifecycle.

### Non-Goals

- A general-purpose infrastructure language or arbitrary scripts in templates.
- Automatically migrating application data or rolling back stateful services.
- Creating and owning resources in more than one project from one App installation.
- Hiding native resource controls from advanced users.
- A marketplace for paid third-party templates.

## Proposal

### Product model

| Concept | Customer meaning |
| --- | --- |
| App Template | A reusable, versioned recipe with a description, required services, configurable inputs, components, outputs, and an upgrade path. |
| App | One installation of a specific template version in a project, with its supplied inputs and owned resources. |
| Component | A Datum resource created by the app, or a reference to an existing resource or managed service in its project. |

A template can connect components by passing declared outputs into later
inputs, such as a Workload's service endpoint into an HTTP route. It can ask for
an existing network or domain instead of creating one. The template declares
which components it owns; referenced resources remain under their existing
owners. Required service enablement uses the Service Catalog's entitlement
flow, with dependencies and price implications shown before deployment.

An App may consume declared outputs from another App, including one in a
different project. The producing App grants access to specific outputs; the
consumer sees the producer, its readiness, and the dependency in its plan.
Each App manages only its own resources. A reference never makes the consumer
the producer's parent or owner, and deleting either App does not cascade to
the other. Direct resource references remain available within an App's project.
For example, a database App can share its endpoint with an API App in another
project while retaining its own deployment and deletion controls.

### Author and publish

An author develops and tests an app recipe in a project. They can start from a
draft template or derive a draft from a working App revision, then choose which
values become inputs. Publishing validates the recipe and creates an immutable
template version owned by the organization. Datum uses the same path to
publish platform-owned curated versions. A published template keeps the source
project and revision as provenance, but installations depend only on the
published version. Project-specific values and secret contents never enter the
published template.

One discovery experience combines Datum-curated templates with those published
by the customer's organization. It shows the publisher, version, required
services and permissions, example outputs, and lifecycle state. Project drafts
remain visible only to their authors; other projects discover published
versions. An organization can manage its own publications without maintaining
a separate catalog resource. Publishing a new version does not change existing
installations.

### Customer experience

1. **Discover:** Browse templates by use case and see the services, required
   permissions, expected resources, and pricing dimensions before choosing one.
2. **Configure:** Select a project, name the app, choose a deployment location,
   provide a container image or artifact, and fill in template-specific inputs.
   Secret values are supplied through secret references.
3. **Preview and deploy:** Review the resource plan, new service enablements,
   external resources the app will use, and estimated recurring cost. Deployment
   reports progress per component and gives a useful error when action is needed.
4. **Operate:** Open the app to find its endpoint, health, recent activity,
   component resources, and costs. Update inputs or select a newer template
   version after reviewing the resulting plan. Revert to a prior successful
   version when an update fails.
5. **Delete:** Preview what will be removed. App-owned resources are deleted;
   referenced resources and shared service entitlements are retained. Choose
   **keep resources** before deletion to graduate owned resources to direct
   management after the App installation is gone.

For example, a team selects **Web API**, chooses an image and domain, and
deploys. The app creates a Workload and route, connects the route to the chosen
domain, and reports a single URL and readiness state. If a managed database is
part of the recipe, the team can choose an existing database or request a new
one, subject to that service's availability and permissions.

### Initial scope

- Start with platform-curated templates installed into a single project, then
  allow organization-owned publication once publishing validation is proven.
  Start with Workload, Gateway/HTTP route, domain, and references to existing
  services; add Function and new managed services as they mature.
- Publish immutable template versions. Existing apps stay on their selected
  version until an owner chooses an update.
- Validate inputs and permissions, show a plan, reconcile created resources,
  and expose partial failure and retry at the component level.
- Record app ownership on created resources so updates and deletion can
  distinguish them from referenced or manually created resources, and prevent
  direct changes while the App manages them.

### Success measures

- A new customer can deploy the Web API example and reach a working endpoint
  without manually creating its component resources.
- The app page explains which component blocks readiness and links to that
  resource for diagnosis.
- Customers can preview and complete an app update or deletion without losing
  referenced resources.
- Customers can keep an App's resources when removing it, then manage those
  resources directly with no period of simultaneous App and customer control.
- Track deployment completion, time to first healthy endpoint, failed steps,
  template reuse, and update or rollback rates.

### Risks and Mitigations

- **Privilege escalation:** Validate installer permissions and run reconciliation
  with project-scoped rights; every generated resource still passes its native
  admission checks.
- **Unexpected deletion or drift:** Track created resource identities, never
  adopt references, preview destructive changes, and reconcile App resources
  against their declared spec.
- **Incomplete service signals:** Show unknown readiness or cost explicitly
  when a service has not published the corresponding App capability.

## Design Details

### API and ownership contract

The proposed API includes immutable `AppTemplate` versions, a project-scoped
`App` installation, side-effect-free `AppPlan` previews, and immutable
`AppRevision` history. Templates declare the resources they create or reference,
typed inputs and outputs, and required services. A plan shows the expected
resource changes and available cost estimates before the App is applied.

An App owns only resources it creates. Customers change those resources through
the App while it is installed; existing referenced resources remain under their
current owners. On deletion, customers can remove owned resources or choose to
keep them. Kept resources become directly manageable after the App installation
is removed. The [Apps architecture](../../../architecture/experiences/apps/README.md)
defines the API shapes, Milo multicluster runtime, admission enforcement, and
release sequence that preserves single ownership.

### Delivery sequence

1. **First vertical slice:** One platform-curated Web API template with a
   Workload, an existing Gateway, and an HTTP route. Ship `AppTemplate`, `App`,
   a shared planner, and ownership admission through API and CLI before adding
   a template gallery. Resolve the Workload-to-route backend contract with
   Networking first.
2. **Safe lifecycle:** Add plans, component status, deletion, retention and
   release, immutable revisions, and rollback. Verify
   direct update/delete denial, marker forgery, status and finalizer exceptions,
   name reuse, partial failure, retry, drift, and preservation of referenced
   resources in end-to-end tests.
3. **Broader composition:** Allow organization-owned templates after publishing
   validation is proven. Add cross-project App output references and richer
   readiness, outputs, and cost estimates as services publish App capability
   contracts.

The first release succeeds when a project owner can preview, deploy, inspect,
update, and remove the curated Web API without manually coordinating its
component resources.

### Open decisions

- Which components should the first curated template create versus require as
  existing resources, particularly domains and Gateways?
- What cost estimate is useful and reliable before deployment when service usage
  depends on traffic?
- Which networking resource should an HTTPRoute target for a Workload component,
  and what output should that component expose to the template?

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

Enable Apps by project during the initial rollout; existing resource behavior
does not change. Disabling new installs must leave reconciliation and deletion
available for existing Apps. Reenabling resumes normal reconciliation from the
stored App spec and resource inventory.

### Rollout, Upgrade and Rollback Planning

Roll out the curated Web API template to pilot projects first. Template versions
are immutable, so publishing a new version does not alter existing Apps. Keep
the controller compatible with stored revisions during upgrades and rollbacks.
Block broader rollout if ownership, deletion, or status reporting fails in the
pilot; test controller restart and upgrade, downgrade, upgrade paths.

### Monitoring Requirements

Operators need counts of Apps by condition, plan validation failures,
reconciliation lag, retry rates, and dependency-blocked Apps. Customers see
per-component conditions, resource links, outputs, and activity. Set numeric
plan latency and time-to-ready objectives before this proposal becomes
implementable.

### Dependencies

Apps depend on Milo project APIs and IAM, the Service Catalog entitlement flow,
and the native APIs used by each template. The first template also depends on a
defined Workload-to-route backend contract. Cost estimates and richer readiness
depend on service-published capability contracts.

### Scalability

Plans and installs must remain responsive as projects and apps grow. Bound
template size, rate-limit reconciliation per project, and establish scale
targets for project onboarding and admission latency before broad rollout.

### Troubleshooting

An unavailable API or service pauses the affected component and retries without
discarding known state. App conditions identify the blocked component, reason,
and last transition; the inventory and audit events show which native resources
were changed. Operators can inspect the linked resource in its owning service.

## Implementation History

- 2026-10-11: Initial provisional product proposal.
- 2026-10-13: API and implementation approach revised; product named Apps.
- 2026-10-20: Added Milo multicluster architecture, ownership admission, and
  graduation after installation removal.
- 2026-10-20: Moved API and runtime design to the linked Apps architecture.
- 2026-10-20: Defined project authoring, publication, and catalog discovery.

## Drawbacks

Apps add a controller and a cross-service lifecycle to operate. Plans cannot
guarantee a transaction across services, and some resource kinds cannot provide
meaningful health or cost signals without an additional capability contract.

## Alternatives

- **Client-side manifest bundles:** Simple to publish, but no durable App status,
  ownership, or safe update and deletion flow.
- **One App API per service:** Keeps service-specific logic local, but makes
  multi-service deployment and a unified customer view difficult.

## Infrastructure Needed (Optional)

An Apps API and controller, storage for templates, Apps, plans, revisions, and
release records, project-scoped service identity, and project API ownership
admission are required. Existing services continue to own their runtimes and
data planes.
