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

### Goals

- Deploy a useful multi-resource application through one guided flow or a
  declarative request.
- Let template authors compose Datum resources and reference managed services
  through explicit inputs and outputs.
- Give customers one app-level view of deployment progress, failures, endpoints,
  component resources, and estimated and actual spend.
- Make updates, rollbacks, and deletion predictable while keeping each
  underlying resource inspectable in its native service.

### Non-Goals

- A general-purpose infrastructure language or arbitrary scripts in templates.
- Automatically migrating application data or rolling back stateful services.
- Deployments spanning multiple projects or organizations.
- Hiding native resource controls from advanced users.
- A marketplace for paid third-party templates.

## Proposal

### Product model

| Concept | Customer meaning |
| --- | --- |
| App Template | A reusable, versioned recipe with a description, required services, configurable inputs, components, outputs, and an upgrade path. |
| App | One installation of a specific template version in a project, with its supplied inputs and owned resources. |
| Component | A Datum resource created by the app, or a reference to an existing resource or managed service. |

A template can connect components by passing declared outputs into later
inputs, such as a Workload's service endpoint into an HTTP route. It can ask for
an existing network or domain instead of creating one. The template declares
which components it owns; referenced resources remain under their existing
owners. Required service enablement uses the Service Catalog's entitlement
flow, with dependencies and price implications shown before deployment.

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
   referenced resources and shared service entitlements are retained.

For example, a team selects **Web API**, chooses an image and domain, and
deploys. The app creates a Workload and route, connects the route to the chosen
domain, and reports a single URL and readiness state. If a managed database is
part of the recipe, the team can choose an existing database or request a new
one, subject to that service's availability and permissions.

### Initial scope

- Start with platform-curated templates installed into a single project, then
  allow organization-owned templates once publishing validation is proven.
  Start with Workload, Gateway/HTTP route, domain, and references to existing
  services; add Function and new managed services as they mature.
- Publish immutable template versions. Existing apps stay on their selected
  version until an owner chooses an update.
- Validate inputs and permissions, show a plan, reconcile created resources,
  and expose partial failure and retry at the component level.
- Record app ownership on created resources so updates and deletion can
  distinguish them from referenced or manually created resources.

### Success measures

- A new customer can deploy the Web API example and reach a working endpoint
  without manually creating its component resources.
- The app page explains which component blocks readiness and links to that
  resource for diagnosis.
- Customers can preview and complete an app update or deletion without losing
  referenced resources.
- Track deployment completion, time to first healthy endpoint, failed steps,
  template reuse, and update or rollback rates.

### Risks and Mitigations

- **Privilege escalation:** Validate installer permissions and run reconciliation
  with project-scoped rights; every generated resource still passes its native
  admission checks.
- **Unexpected deletion or drift:** Track created resource identities, never
  adopt references, preview destructive changes, and reconcile only fields owned
  by the App.
- **Incomplete service signals:** Show unknown readiness or cost explicitly
  when a service has not published the corresponding App capability.

## Design Details

### Proposed API design

The API uses `apps.datumapis.com/v1alpha1` as a proposed group. Names and field
shapes below are for review; service-specific manifests still use their native
APIs. Templates are published at organization scope (Datum publishes the
curated set), while apps and plans live in a project. Authorization to use a
template is checked when planning and deploying.

| Resource | Purpose |
| --- | --- |
| `AppTemplate` | Immutable, published recipe version. `spec.family` groups versions for discovery; `spec.version` identifies one release. |
| `App` | Desired installation: template reference and input values. Its status reports the observed deployment. |
| `AppPlan` | Side-effect-free preview of a proposed App create or update. Creating a plan never enables services or creates components. |
| `AppRevision` | Immutable record of an accepted App spec, used for history and rollback. It stores references rather than secret values. |

An `AppTemplate` declares typed inputs, required catalog services, components,
and public outputs. Each component either **creates** a service-native resource
or **references** an existing one. Typed bindings copy an input or another
component's declared output into a field accepted by the target API. Publishing
validates binding types, target schemas, and dependency cycles. This keeps
composition explicit without allowing arbitrary template code. The example
below omits most of the service-native Workload and HTTPRoute specs to focus on
the app contract:

```yaml
apiVersion: apps.datumapis.com/v1alpha1
kind: AppTemplate
metadata:
  name: web-api-v1
spec:
  family: web-api
  version: 1.0.0
  inputs:
    - name: image
      type: string
      required: true
    - name: gateway
      type: resourceRef
      resourceKind: Gateway
      required: true
    - name: hostname
      type: string
      required: true
  requiredServices:
    - compute.datumapis.com
    - networking.datumapis.com
  components:
    - name: api
      create:
        apiVersion: compute.datumapis.com/v1alpha
        kind: Workload
        # Service-native Workload spec omitted.
      bindings:
        - fromInput: image
          toPath: /spec/template/spec/runtime/sandbox/containers/0/image
    - name: route
      dependsOn: [api]
      create:
        apiVersion: gateway.networking.k8s.io/v1
        kind: HTTPRoute
        # Service-native HTTPRoute spec omitted.
      bindings:
        - fromInput: gateway
          fromField: metadata.name
          toPath: /spec/parentRefs/0/name
        - fromInput: hostname
          toPath: /spec/hostnames/0
        # The backend binding targets the networking service's supported
        # resource for the api component.
  outputs:
    - name: hostname
      fromInput: hostname
```

An `App` contains only customer choices; the controller resolves the pinned
template and creates the declared components. Resource inputs include full
group, kind, and name in the actual API, and must resolve within the project
unless a service explicitly supports cross-scope references.

```yaml
apiVersion: apps.datumapis.com/v1alpha1
kind: App
metadata:
  name: orders-api
  namespace: default
spec:
  templateRef:
    name: web-api-v1
  inputs:
    image:
      value: registry.example.com/orders@sha256:...
    gateway:
      resourceRef:
        apiVersion: gateway.networking.k8s.io/v1
        kind: Gateway
        name: shared-edge
    hostname:
      value: orders.example.com
```

The controller populates status; clients do not set it:

```yaml
status:
  observedGeneration: 1
  templateVersion: 1.0.0
  currentRevision: orders-api-r1
  conditions:
    - type: Ready
      status: "True"
  components:
    - name: api
      resourceRef:
        apiVersion: compute.datumapis.com/v1alpha
        kind: Workload
        name: orders-api-api
      ready: true
  outputs:
    hostname: orders.example.com
```

The app controller reports `status.observedGeneration`, the resolved template
version, conditions (`Accepted`, `Progressing`, `Ready`), per-component
resource references and conditions, and declared outputs such as the hostname. A
condition identifies the failed component and its actionable reason. Created
resources carry an app owner reference or equivalent cross-service ownership
record; referenced resources are never adopted.

To preview a change, a client creates an `AppPlan` with `spec.app` containing
the proposed App name and spec. The plan compares it with any existing App of
that name in the project. Its `status` returns the resources to create, update,
or delete; required service enablements; permission or quota failures; and
available cost estimates with their assumptions or an explicit unknown.
Applying the App repeats validation because a plan can become stale. Every
accepted App spec is retained as an `AppRevision`
for audit; rollback reapplies a prior successful revision's template reference
and inputs.
Secrets are accepted only as references, never copied into plans, revisions,
status, or generated activity messages.

Updating `spec.templateRef` or `spec.inputs` reconciles only app-owned
components. A failed component leaves the app `Ready=False` with partial
progress visible and can be retried. Deleting an App removes its owned
components after a deletion preview; existing referenced resources and shared
service entitlements remain. Native resource admission, permissions, and
service-specific safety checks still apply to every generated change.

### Implementation approach

Build Apps as a control-plane service that registers the APIs above with Milo
and reconciles project Apps through existing Datum service APIs. It does not run
application traffic or replace the Workload, Networking, DNS, or managed-service
controllers. The managed service pattern
provides the model for watching project resources and reporting status across
control planes.

The service has three parts:

1. **Template API and admission:** Store immutable template versions. Check
   publisher permissions, target API discovery, typed bindings, dependency
   cycles, reference scope, and secret handling before publication. Require
   target APIs to validate their own resource specs. Templates cannot execute
   scripts or bypass service admission.
2. **Shared planner:** Resolve a template version and App inputs into a stable
   component graph. Validate references, project permissions, service
   entitlements, quotas, and service-specific admission. Diff that graph against
   the App's owned resources. Use the same planner for `AppPlan` and controller
   reconciliation; the controller rechecks the plan at apply time. Plans expire
   and are advisory because resources and prices can change.
3. **App controller:** Enable required services through the Service Catalog's
   normal entitlement flow, wait for them to become usable, then apply the graph
   in dependency order. Watch native resource conditions where their meaning is
   known, publish component status and outputs, and retry failed steps with
   backoff. Persist an `AppRevision` for each accepted spec and mark it
   successful only when the App is ready.

Composition should be open to any project resource reachable through Milo's
standard API, subject to project authorization, target admission, and an Apps
service identity scoped to that project. Generic integration provides create,
update, reference, and delete using API discovery and an ownership inventory.
The installer must have permission for the resulting resource actions, or a
project administrator must grant a template-specific install permission; the
controller receives only the corresponding project-scoped rights. Generic
integration can report that a resource was accepted, but cannot infer health
or usage cost from an unfamiliar kind. Services can register optional App
capabilities for readiness conditions, outputs, cost estimates, and special
deletion behavior. A required component without a readiness contract leaves
the App's readiness unknown, and a template cannot bind to an undeclared
output. Plans and status show when capabilities are unavailable. Platform
policy may still deny sensitive kinds or cross-project
references; those are authorization decisions, not a fixed list of app-capable
resources.

Each generated resource gets a stable name and an App identity label. The App
keeps an inventory of created resource identities so it can detect drift and
clean up after partial failures. Where resources live in another control plane,
that inventory and a finalizer provide ownership; a Kubernetes owner reference
alone cannot cross control-plane boundaries. Use server-side apply with a
dedicated field manager so the App controller owns only template-managed fields.
Changes to those fields are reconciled back to the App spec; other native fields
remain available to service-specific tooling. Referenced resources are checked
for access and readiness but never adopted or deleted.

Deletion runs the graph in reverse dependency order and keeps the App finalizer
until owned resources are removed or a clear intervention condition is set.
Shared service entitlements remain enabled. Rollback selects a prior successful
revision and reconciles its desired spec; it cannot undo external side effects
or stateful data changes. A failed rollout preserves the previous revision
record and exposes exactly which component needs attention.

#### Delivery sequence

1. **First vertical slice:** One platform-curated Web API template with a
   Workload, an existing Gateway, and an HTTP route. Ship `AppTemplate`, `App`,
   and a shared planner through API and CLI before adding a template gallery.
   Resolve the Workload-to-route backend contract with Networking first.
2. **Safe lifecycle:** Add plans, component status, deletion, immutable
   revisions, and rollback. Verify partial failure, retry, drift, permissions,
   and preservation of referenced resources in end-to-end tests.
3. **Broader composition:** Allow organization-owned templates after publishing
   validation is proven. Add richer readiness, outputs, and cost estimates as
   services publish App capability contracts.

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

Apps add four API types and API calls proportional to their component count.
Bound template size and plan work, rate-limit reconciliation per project, and
watch only the resources needed by active Apps. Measure controller work queue
depth and API request rates as adoption grows.

### Troubleshooting

An unavailable API or service pauses the affected component and retries without
discarding known state. App conditions identify the blocked component, reason,
and last transition; the inventory and audit events show which native resources
were changed. Operators can inspect the linked resource in its owning service.

## Implementation History

- 2026-10-11: Initial provisional product proposal.
- 2026-10-13: API and implementation approach revised; product named Apps.

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

An Apps API and controller, storage for templates, Apps, plans, and revisions,
and project-scoped service identity are required. Existing services continue to
own their runtimes and data planes.
