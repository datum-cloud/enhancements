# Apps architecture

This document describes the proposed APIs, project runtime, resource ownership
protection, and graduation flow for [Apps on Datum Cloud](../../../enhancements/experiences/apps/README.md).
It is a design proposal; API names and fields remain subject to review.

## API design

The API uses `apps.miloapis.com/v1alpha1` as a proposed group. Apps coordinates
project resources across services, like other Milo platform capabilities; its
API group does not require its implementation to live in the `milo-os/milo`
repository. The Apps service and schemas remain owned by `datum-labs/apps` and
are served through Milo's normal service API and discovery integration. Names
and field shapes below are for review; service-specific manifests still use
their native APIs. Authoring begins in a project. Every published
`AppTemplate` version is stored and discoverable through the Platform context.
Its `publisherRef` identifies Datum or an organization; publisher identity is
metadata and authorization, not a second API scope. Apps and plans live in a
project. The Platform list and get APIs filter published versions by visibility
and caller authorization, without searching projects or creating an
`OrganizationCatalog` resource. A project resolves a published version by its
qualified identity. Authorization to use it is checked when planning and
deploying.

| Resource | Purpose |
| --- | --- |
| `AppTemplateDraft` | Editable recipe in an authoring project; supports preview and test installations there. |
| `AppPublication` | Project-scoped request to validate an exact draft revision and publish an immutable version in Platform context for an authorized publisher. |
| `AppTemplate` | Platform-scoped immutable published version. `spec.family` groups versions for discovery; `spec.version` identifies one release. |
| `App` | Desired installation: template reference and input values. Its status reports the observed deployment. |
| `AppPlan` | Side-effect-free preview of a proposed App create or update. Creating a plan never enables services or creates components. |
| `AppRevision` | Immutable record of an accepted App spec, used for history and rollback. It stores references rather than secret values. |

`AppRelease` is an internal, durable handoff record created when a customer
deletes an App with `Retain`. It is not a second installation.

### Project authoring and catalog discovery

A project-scoped `AppTemplateDraft` is the working recipe. An author may build
it directly or derive it from an App's accepted revision, then replace
project-specific values with typed inputs and identify references that
installers must supply. The draft can be planned and installed for testing only
in its source project. An `AppPublication` names the exact draft revision,
publisher identity, visibility, family, and version. Publishing checks the
author's rights in the source project and publisher organization, target API
schemas, bindings, required services, referenced-resource rules, and the
absence of embedded credentials. It then stores an immutable published snapshot
in Platform context. A live App or its resource
state is never the template's ongoing source of truth.

The published version records source project and revision, publisher identity,
content digest, compatibility requirements, and lifecycle state. The discovery
API exposes these along with required permissions, inputs, and outputs.
Project drafts are private to their project. Platform discovery filters
published versions by visibility, publisher policy, and service availability;
private organization versions are visible only to authorized callers in that
organization. List, get, plan, and install must enforce the same visibility
rules so direct lookup cannot bypass discovery filtering. A template reference
includes publisher identity and immutable version identity, so equal family
names cannot collide. The service assigns a unique Platform object name from
that qualified identity; consumers do not infer publisher identity from a
display name.
Deprecation hides a version from recommended discovery while existing pinned
Apps continue to work. Emergency revocation blocks new installs and upgrades;
existing installations retain the snapshot needed for safe reconciliation,
deletion, and graduation. The source project may be removed without changing
published versions.

An `AppTemplate` declares typed inputs, required catalog services, components,
and public outputs. Each component either **creates** a service-native resource
or **references** an existing one. Typed bindings copy an input or another
component's declared output into a field accepted by the target API. Publishing
validates binding types, target schemas, and dependency cycles. This keeps
composition explicit without allowing arbitrary template code. The example
below omits most of the service-native Workload and HTTPRoute specs to focus on
the app contract:

```yaml
apiVersion: apps.miloapis.com/v1alpha1
kind: AppTemplate
metadata:
  name: web-api-v1
spec:
  publisherRef:
    kind: Platform
    name: datum
  visibility: Public
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
      type: string
      exportable: true
      fromInput: hostname
```

An `App` contains only customer choices; the controller resolves the pinned
template and creates the declared components. Direct resource inputs include
group, kind, name, and scope and resolve within the installation project.
Cross-project composition uses a typed `appRef` input naming the producing
project and App plus a declared output. The producing template defines the
output's type and whether it may be exported; the producing App explicitly
grants access to the consuming project. The planner checks both the consumer's
right to use it and the producer's grant. The consumer records the producer's
UID and output contract, detects name reuse, and watches for relevant output
or readiness changes. A missing producer, revoked grant, incompatible output,
or dependency cycle blocks affected reconciliation with an actionable
condition. The consumer never adopts or deletes the producer or its resources.
For example, a database App in project `data` could export an `endpoint` to
project `orders`; an API App in `orders` could bind that output to one of its
inputs:

```yaml
# Consumer template input
inputs:
  - name: databaseEndpoint
    type: appOutputRef
    outputType: string
---
# Producer App spec, in project data
outputGrants:
  - output: endpoint
    projects: [orders]
---
# Consumer App input, in project orders
databaseEndpoint:
  appRef:
    project: data
    name: shared-db
    output: endpoint
```

Exports contain typed, non-secret values or service-defined scoped references,
never credential contents. A consumer cannot re-export an output without its
own explicit authorization. The producer may change an output value within its
declared type; the consumer plans and reconciles that change under its own
revision and permissions. Output grants do not transfer ownership.

```yaml
apiVersion: apps.miloapis.com/v1alpha1
kind: App
metadata:
  name: orders-api
  namespace: default
spec:
  templateRef:
    publisherRef:
      kind: Platform
      name: datum
    name: web-api-v1
  uninstallPolicy: Delete # Set to Retain before deleting to keep owned resources.
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
resources carry a protected App installation UID marker and appear in the App's
inventory; referenced resources are never adopted.

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

`spec.uninstallPolicy` defaults to `Delete`. An owner can change it to `Retain`
after reviewing a plan that lists every resource to keep, then delete the App.
The App is the installation: there is no separate installation object to leave
behind. Retained resources become directly editable only after that App has
been removed and their ownership markers have been released. A failed release
shows progress and retries; it never silently grants direct control.

## Runtime and reconciliation

Build Apps in the new [`datum-labs/apps`](https://github.com/datum-labs/apps)
repository as a control-plane service that registers the APIs above with Milo
and reconciles project Apps through existing Datum service APIs. The Apps API,
planner, controllers, curated templates, packaging, and service-owned tests
live there. Milo admission changes stay in `milo-os/milo`; the automated
end-to-end environment and Flux deployment fixtures live in
[`datum-cloud/test-infra`](https://github.com/datum-cloud/test-infra). Apps does
not run application traffic or replace the Workload, Networking, DNS, or
managed-service controllers. Milo's [multicluster runtime provider](https://github.com/milo-os/milo/blob/main/pkg/multicluster-runtime/milo/provider.go)
provides project discovery and per-project clients and caches. It watches
ready Projects through the core API and connects to their project control
planes; internal deployments can instead watch ready ProjectControlPlanes and
use service addresses. The Apps controller uses the mode appropriate to its
deployment, scopes each reconciliation to one project, and disengages when a
project becomes unavailable. Cluster engagement must run on every replica,
using Milo's `WithoutAutoStart` and `EngageAlways` wiring until the upstream
leader-election issue is fixed.

Apps deployment manifests live under `datum-labs/apps/config` and use
Kustomize bases, optional components, and environment overlays, following the
relevant layout in [`datum-cloud/compute/config`](https://github.com/datum-cloud/compute/tree/main/config).
The automated test environment consumes an Apps overlay or published Kustomize
bundle through Flux, so tests exercise the same deployment manifests intended
for release.

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
controller receives only the corresponding project-scoped rights. Define the
durable delegation behind that statement: record the template version,
component kinds, verbs, project, and authorizing grant for each installation;
check authority on install, spec changes, and later reconciliation. Revoking
the grant must prevent new privileged actions while preserving safe cleanup
or graduation. Privileged kinds such as RBAC and admission policy resources
require an explicit platform allowlist before publication. Generic
integration can report that a resource was accepted, but cannot infer health
or usage cost from an unfamiliar kind. Services can register optional App
capabilities for readiness conditions, outputs, cost estimates, and special
deletion behavior. A required component without a readiness contract leaves
the App's readiness unknown, and a template cannot bind to an undeclared
output. Plans and status show when capabilities are unavailable. Platform
policy may still deny sensitive kinds or App output exports; those are
authorization decisions, not a fixed list of app-capable resources.

Each generated resource gets a stable name and a protected owner marker
containing the App's project and immutable UID. The App stores an inventory of
component GVK, name, namespace, and resource UID so it can detect drift, name
reuse, and partial failures. The marker and inventory must agree before an
update or deletion. On restart, the controller also discovers marked resources
to recover a create that succeeded before its inventory write. A Kubernetes
owner reference alone cannot establish ownership across control planes. Use
server-side apply with a dedicated field manager for the desired fields; field
ownership does not authorize direct customer edits while the App manages the
resource. Referenced resources are checked for access and readiness but never
marked, adopted, or deleted. A cross-project `appRef` consumes only approved
outputs from another App; it does not give the consumer direct access to that
App's component resources. The producer and consumer each retain one owner and
an independent lifecycle. Apps does not grant itself write access in the
producer's project.

## Ownership protection

One App installation owns a resource at a time. The controller refuses to
create over an existing unmarked resource or one marked by a different App.
Future adoption of an existing resource requires a separate, explicit transfer
flow; it is not part of the first release. Only dedicated Apps controller and
release identities may create or remove an owner marker. Project users,
including project admins, cannot update or delete a marked resource directly;
they update the App instead.
Normal status updates and necessary service-controller finalizer writes must
continue through narrowly scoped subresource or service-identity rules. No
general exception for privileged project users or field managers is allowed.

Enforce this at the project API's admission boundary, after authentication and
authorization, for every supported component kind. Check the old object on
update and delete so a client cannot strip the marker in the same request.
Reject marker forgery on create and marker changes by other identities. Give
the Apps controller a dedicated credential with scoped permissions; customers
must not be able to impersonate it. Keep a project-level admission guard
active even when new App installations are disabled, or existing resources
would become unprotected.

A shared CEL `ValidatingAdmissionPolicy` and `Deny` binding could make the
common marker and caller checks local to each API server, with constant policy
count per project rather than one policy per App or component. However,
[Milo currently exposes admissionregistration resources only in Platform
context](https://github.com/milo-os/milo/blob/main/config/discovery/core/discovery-context-policy.yaml),
and [its project control planes are not seeded with
policies](https://github.com/milo-os/milo/blob/main/internal/apiserver/admission/plugin/emailverification/admission.go).
Enabling the plugin alone therefore does not protect project writes. Start
with a small Milo admission guard on project API servers, or first add reliable
per-project policy bootstrap and restricted policy administration. A VAP rollout
must prove policy availability before the API accepts App ownership, fail
closed when policy distribution fails, and measure admission latency and policy
sync at project scale. CEL can inspect the request and old object but cannot
look up the live App, so the controller remains responsible for the ownership
inventory and release state. Recheck any component APIs that bypass Milo's
project admission path before allowing them in templates.

Policy failure behavior needs a separate bootstrap guarantee: Kubernetes
[ignores a binding whose policy is missing](https://kubernetes.io/docs/reference/kubernetes-api/admissionregistration/validating-admission-policy-binding-v1/),
and [admissionregistration resources are exempt from validating admission
policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/).
`failurePolicy: Fail` alone cannot secure a project where the policy or binding
was never installed or was removed. Gate marker creation and project readiness
on a verified guard, protect guard administration with platform authorization,
and independently deny writes to *existing marked resources* while that guard
is absent. If VAP distribution cannot maintain both guarantees, use a compiled
Milo guard for the first release. Exercise startup, reconnection, and deletion
of the policy and binding on a real project API server.

## Graduation and removal

There are two deletion outcomes, shown in the App plan:

1. **Delete resources (default):** The App finalizer deletes owned components
   in reverse dependency order, verifies their resource UIDs, and then removes
   the installation. Referenced resources and shared entitlements stay.
2. **Keep resources:** The user first sets `spec.uninstallPolicy: Retain` and
   reviews the retained inventory, then deletes the App. The App controller
   freezes reconciliation, persists an `AppRelease` record with the exact
   inventory, and removes the App finalizer. The release worker waits until the
   App UID is absent, then clears owner markers only on matching resource UIDs.
   It records per-resource progress and retries failures. A resource remains
   protected until its marker is cleared, and an incomplete release is visible
   for repair. The release record is removed only after all resources are free.

Only the trusted App controller may create an `AppRelease`, and only the
release worker may advance it or clear ownership markers. Persist a snapshot
of the uninstall policy and exact App and resource UIDs before App removal;
customers cannot forge or edit that snapshot. The worker independently checks
the App UID is absent and each resource UID and marker match before release.

The release worker must not apply App templates. This ordering ensures that a
user can take over a resource only after deleting the managing installation;
there is no dual ownership during a partial release. A new App cannot claim a
marked resource. Direct edits and deletes are denied throughout the handoff
until each resource is released. The release record is durable and separate
from the deleted App so retries survive controller restarts and App garbage
collection. Retaining a stateful component keeps its data but does not promise
the App's former readiness or connectivity after graduation.
An App may be removed even when another App consumes one of its outputs; there
is no cascading deletion or implicit finalizer on the producer. Deletion
preview lists known consumers where the caller may see them, and consumers
report a missing dependency after removal. The producer's output grant is
removed with it. A replacement App using the same name has a different UID
and requires the consumer to accept a new reference.

Default deletion runs the graph in reverse dependency order and keeps the App
finalizer until owned resources are removed or a clear intervention condition
is set.
Shared service entitlements remain enabled. Rollback selects a prior successful
revision and reconciles its desired spec; it cannot undo external side effects
or stateful data changes. A failed rollout preserves the previous revision
record and exposes exactly which component needs attention.

## Scale and rollout checks

Apps add several customer-facing API types, an internal release record, and API
calls proportional to component count. Bound template size and plan work,
rate-limit reconciliation per project, and restrict per-project caches to
needed resource kinds. Milo's provider currently registers project control
planes serially; measure project engagement time, memory, watch count, and API
request rates before expanding adoption. Ownership admission must use request
metadata and identity without per-write App lookups. Measure deny rate and
latency separately from reconciliation work.

Before enabling a component kind, verify that all writes pass through the
project ownership guard and that its normal status and finalizer updates still
work. Test controller restart, a create that succeeds before inventory is
written, project disconnection, name reuse, partial deletion, and partial
release. Keep reconciliation and release enabled when new installations are
disabled.

## Open question

Can project policy bootstrap and administration meet fail-closed availability
and latency targets, allowing the ownership guard to move from a Milo plugin
to one shared validating admission policy and binding per project?
