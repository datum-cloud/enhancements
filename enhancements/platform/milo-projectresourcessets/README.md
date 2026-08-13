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

When editing RFCs, aim for tightly-scoped, single-topic PRs to keep discussions
focused.
-->

# ProjectResourceSets: Bootstrap defaults into Project control planes

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

Milo Projects are virtualized control planes (reachable at `/projects/<id>/control-plane`)
that start minimally functional and often need baseline resources to be usable.
Today, some of those defaults are installed by the core Project controller via
ad-hoc logic (e.g. creating `GatewayClass` and `DNSZoneClass` objects inside each
Project control plane).

This enhancement introduces a generic, policy-driven mechanism to apply Kubernetes
objects into each Project control plane automatically:

- **`ProjectResourceSet`**: declares a set of manifests (stored in ConfigMaps/Secrets)
  and a selector for matching Projects.
- **`ProjectResourceSetBinding`**: records which `ProjectResourceSet`s and which
  resources were applied to a given Project, including hashes, timestamps, and
  success/failure state.

The initial use-cases are to replace the existing:

- `ensureGatewayClass(...)` (currently used to install the per-Project default `GatewayClass`)
- `ensureDNSZoneClass(...)` (currently used to install the per-Project default `DNSZoneClass`)

with equivalent manifests managed declaratively by `ProjectResourceSet`.

## Motivation

Milo manages customer **Organizations**, each of which has **Projects**. Each Project is
a separate, virtualized control plane. We need a flexible way to install default
behavior into Project control planes at creation time (and optionally keep it up to
date over time) without hard-coding per-service bootstrapping into the Project
lifecycle controller.

The motivating problems include:

- **Separation of concerns**: the Project lifecycle controller should focus on
  provisioning and readiness of Projects, not on installing service-specific resources.
- **Consistency and policy**: Organizations often want consistent defaults across
  all Projects they own.
- **Extensibility**: new services should not require editing the Project controller
  to install their defaults into every Project.

### Goals

- Provide a way to apply a set of Kubernetes objects automatically to:
  - newly-created Projects, and
  - existing Projects that match selection criteria.
- Provide an API surface that supports Organization-scoped policy (e.g. different
  defaults per Organization) without requiring per-service code changes.
- Provide visibility into:
  - which resources were applied to a Project,
  - when they were applied,
  - and whether they succeeded.
- Support both:
  - **ApplyOnce** semantics (install and then stop managing), and
  - **Reconcile** semantics (reapply when the resource definitions change).

### Non-Goals

- Lifecycle management of installed components (e.g. “install a controller deployment
  and keep it healthy”). This enhancement focuses on applying objects, not operating
  them.
- Deleting resources from Project control planes when removed from a set (pruning).
  Like Cluster API’s ClusterResourceSet, deletion is intentionally out of scope for
  the initial version.
- Drift correction based on live-object divergence. Reconcile mode is triggered by
  definition changes (hash changes), not by detecting mutations inside Project control
  planes.

## Proposal

Introduce a “resource set” mechanism analogous to Cluster API’s ClusterResourceSet,
adapted to Milo’s Projects and virtual control planes.

At a high level:

1. Operators define one or more `ProjectResourceSet`s that reference manifest bundles
   stored in ConfigMaps/Secrets.
2. A controller in the Milo management plane matches Projects using selectors and
   applies the manifests into each Project control plane using the existing per-Project
   API routing (`/projects/<id>/control-plane`).
3. A `ProjectResourceSetBinding` object tracks and exposes apply results for each
   Project.

### User Stories (Optional)

#### Story 1: Install a default GatewayClass into each Project control plane

As a platform operator, I want every Project in an Organization (or matching label selector)
to automatically have a default `GatewayClass` created, replacing the current
hard-coded behavior.

This should be implemented by:

- a `ProjectResourceSet` that selects the relevant Projects, and
- a manifest bundle containing the `GatewayClass` YAML.

#### Story 2: Install a default DNSZoneClass into each Project control plane

As a platform operator, I want every Project in an Organization (or matching label selector)
to automatically have a default `DNSZoneClass` created, replacing the current
hard-coded behavior.

This should be implemented by:

- a `ProjectResourceSet` that selects the relevant Projects, and
- a manifest bundle containing the `DNSZoneClass` YAML.

### Notes/Constraints/Caveats (Optional)

- **Virtualized control plane boundary**: objects applied into a Project control plane
  live in a different logical API surface than the management plane objects defining
  the sets. Cross-control-plane OwnerReferences are not meaningful; linkage should be
  via labels/annotations and via the `ProjectResourceSetBinding` status.
- **CRD definitions are global across all Projects**: Milo installs CRDs at the global
  (shared) level rather than per-Project. As a result, a missing GVK is typically a
  global installation/registration issue (or transient discovery lag), not a per-Project
  configuration difference. The controller should still use discovery and report a clear
  status in the binding (e.g. “GVK not served”) rather than failing catastrophically.
- **Idempotency**: application must be safe to retry. Prefer server-side apply (SSA)
  for patch-like behavior and stable “field manager” ownership.

### Risks and Mitigations

- **Accidental wide blast radius** (e.g. selector matches too many Projects):
  - Mitigation: require non-empty selectors; optionally gate with feature flags and
    RBAC so only privileged actors can create/modify sets.
  - Mitigation: record all applications in `ProjectResourceSetBinding` for audit and
    rollback planning.
- **Resource conflicts** when multiple sets try to manage the same object:
  - Mitigation: define deterministic ordering or explicit priority; ensure conflicts
    are surfaced in binding status.
- **Secrets distribution risk**:
  - Mitigation: only allow Secrets of a specific type to be referenced as bundles,
    and scope where those Secrets can live (e.g. `milo-system`).

## Design Details

### APIs

#### `ProjectResourceSet`

Cluster-scoped CRD (management plane) that selects Projects and references manifest bundles.

Key fields:

- `spec.projectSelector`: required label selector matching `Project` objects.
- `spec.mode`: one of:
  - `ApplyOnce` (default): apply each referenced bundle once per Project.
  - `Reconcile`: reapply when the bundle content hash changes.
- `spec.resources[]`: list of references to ConfigMaps/Secrets that contain YAML/JSON
  manifests (multiple keys; multiple YAML documents per key supported).

#### `ProjectResourceSetBinding`

Cluster-scoped CRD (management plane), one per Project, tracking application state.

Key fields:

- `spec.projectRef`: identifies the Project.
- `status.bindings[]`: one entry per matching `ProjectResourceSet`, including:
  - bundle references (ConfigMap/Secret)
  - `applied` boolean, last applied time
  - content hash
  - last error (if any)

### Controller integration point (current Milo architecture)

The `ProjectResourceSet` controller runs alongside the existing controllers and uses the
same mechanism already used by the Project lifecycle controller to talk to Project
control planes:

- Derive a per-Project `rest.Config` targeting:
  - `/projects/<id>/control-plane`
- Create dynamic/discovery clients
- Apply manifests into that Project control plane

The controller should gate application on the Project being Ready.

### Selection model: labels vs Organization spec vs Project spec

**Recommendation (initial / alpha): selection via labels (no Project/Organization API changes).**

Rationale:

- Projects already have an immutable organization label set by the Project admission webhook
  (`resourcemanager.miloapis.com/organization-name`).
- Label selectors are the simplest, most Kubernetes-native way to express “apply to all
  Projects in org X” or “apply to Projects matching capability Y”.
- This keeps the initial footprint small and avoids immediate API migrations.

**How Organization-specific defaults work with labels:**

- Create a `ProjectResourceSet` per Organization (or per Organization class), with
  `spec.projectSelector.matchLabels` including the organization label.

Future-friendly evolution paths are covered in [Alternatives](#alternatives).

### Apply modes

#### ApplyOnce

- If a bundle hash has already been recorded as applied for the Project, do not reapply.
- If the target object already exists in the Project control plane, do not overwrite it;
  record as applied and continue.

#### Reconcile

- Compute a deterministic hash for each bundle key (or entire referenced object).
- Reapply when the recorded hash differs from the current hash.
- Reconcile is driven by bundle definition changes, not by live drift.

### Applying resources

Implementation guidance:

- Parse YAML/JSON into `unstructured.Unstructured` objects.
- Apply with SSA (`PATCH ... apply`) using a stable field manager (e.g. `milo-projectresourceset`).
- Attach labels/annotations to applied objects for traceability, for example:
  - `resourcemanager.miloapis.com/managed-by=projectresourceset`
  - `resourcemanager.miloapis.com/projectresourceset=<name>`

### Example: replacing current hard-coded defaults

In the current system, the Project controller contains ad-hoc logic to create per-Project:

- `GatewayClass` for external global proxy
- `DNSZoneClass` for external global DNS

This enhancement replaces that logic with:

- one or more `ProjectResourceSet`s selecting the desired Projects, and
- ConfigMaps/Secrets containing the YAML for those objects.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

#### How can this feature be enabled / disabled in a live cluster?

- [ ] Feature gate
  - Feature gate name: `ProjectResourceSets`
  - Components depending on the feature gate: milo-controller-manager (PRS controller)
- [ ] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control plane?
  - Will enabling / disabling the feature require downtime or reprovisioning of a node?

#### Does enabling the feature change any default behavior?

Yes. When enabled and when `ProjectResourceSet`s are present, Projects may receive new
default resources automatically. Existing hard-coded defaults (if still present) must be
removed or carefully coordinated to avoid conflicts.

#### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Disabling stops further application/reconciliation. Previously applied resources remain
in Project control planes (pruning is out of scope).

#### What happens if we reenable the feature if it was previously rolled back?

The controller will resume reconciliation. In ApplyOnce mode it should not reapply resources
that were already recorded as applied; in Reconcile mode it may apply again if definitions
changed while disabled.

#### Are there any tests for feature enablement/disablement?

Planned:

- unit tests around mode behavior (ApplyOnce vs Reconcile) and binding status updates
- integration/e2e coverage for applying bundles into a sample Project control plane

### Rollout, Upgrade and Rollback Planning

#### How can a rollout or rollback fail? Can it impact already running workloads?

- Misconfigured selectors could apply resources broadly, potentially conflicting with existing
  objects in Project control planes.
- SSA conflicts could surface and block applying required defaults.

Existing workloads inside Project control planes should not be directly impacted unless the
applied resources overlap with names/GVKs they depend on.

#### What specific metrics should inform a rollback?

- sustained increases in PRS apply failures (per-project / per-resource)
- elevated API error rates for Project control plane requests from the controller
- reconciliation latency spikes (time-to-apply for newly created Projects)

#### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Not yet (alpha).

#### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

Yes: removal of hard-coded default installers in the Project controller, replaced by declarative
`ProjectResourceSet`s.

### Monitoring Requirements

#### How can an operator determine if the feature is in use by workloads?

- By listing `ProjectResourceSet` objects.
- By checking for `ProjectResourceSetBinding` objects and their `.status` entries.

#### How can someone using this feature know that it is working for their instance?

- [ ] Events
  - Event Reason: `Applied`, `ApplyFailed`, `Reconciled`
- [x] API .status
  - Condition name: (binding-level) `Applied` / `Degraded` (exact names TBD)
  - Other field: `ProjectResourceSetBinding.status.bindings[*]`
- [ ] Other (treat as last resort)
  - Details:

#### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

Alpha target:

- New Project becomes fully “bootstrapped” within N seconds after ProjectReady (N TBD),
  excluding retries due to missing CRDs or transient API failures.

#### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [ ] Metrics
  - Metric name: `milo_projectresourceset_apply_total` (success/fail labels), `milo_projectresourceset_apply_latency_seconds`
  - Components exposing the metric: milo-controller-manager
- [ ] Other (treat as last resort)
  - Details: binding status inspection

#### Are there any missing metrics that would be useful to have to improve observability of this feature?

Likely:

- per-project “time-to-first-apply” histogram
- per-GVK “missing CRD” counter to spot mismatched installations

### Dependencies

#### Does this feature depend on any specific services running in the cluster?

- Project control plane routing must be functional (the `/projects/<id>/control-plane` path).
- Referenced ConfigMaps/Secrets must be present and readable by the controller.

### Scalability

#### Will enabling / using this feature result in any new API calls?

Yes:

- Watch/list `ProjectResourceSet`s, referenced ConfigMaps/Secrets, and Projects.
- For each matching Project, apply N objects into its Project control plane via PATCH/POST.

Throughput will scale with:

- number of Projects,
- number of matching ProjectResourceSets,
- size of bundles, and
- frequency of bundle changes in Reconcile mode.

#### Will enabling / using this feature result in introducing new API types?

Yes:

- `ProjectResourceSet` (cluster-scoped)
- `ProjectResourceSetBinding` (cluster-scoped; one per Project)

#### Will enabling / using this feature result in increasing size or count of the existing API objects?

Yes:

- One `ProjectResourceSetBinding` per Project.
- Status grows with number of sets and bundles applied to that Project.

#### Will enabling / using this feature result in non-negligible increase of resource usage in any components?

Potentially:

- Controller CPU for parsing bundles and hashing.
- Increased API traffic to Project control planes.

Mitigations:

- bounded concurrency per controller
- backoff on per-Project errors
- avoid periodic drift correction in alpha

#### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

Possible with very large numbers of Projects if concurrency is unbounded.
The controller should enforce concurrency limits and reuse transports where possible.

### Troubleshooting

#### How does this feature react if the API server is unavailable?

- Retries with exponential backoff.
- Binding status reflects failures.

#### What are other known failure modes?

- Missing CRDs in a Project control plane for applied objects:
  - Detection: binding status indicates “GVK not served”.
  - Mitigation: install CRDs, or remove/update the set.
- Conflicting management of the same object by multiple sets:
  - Detection: apply conflict errors surfaced in binding status.
  - Mitigation: adjust selectors or introduce priorities.

#### What steps should be taken if SLOs are not being met to determine the problem?

- Inspect `ProjectResourceSetBinding` for the affected Project for last errors and timestamps.
- Check controller metrics/logs for apply failure rates.
- Validate bundle references and content hashing expectations.

## Implementation History

- (TBD) Initial proposal drafted.

## Drawbacks

- Adds new CRDs and controller complexity.
- ApplyOnce semantics may leave behind “orphaned” defaults over time (no pruning).
- Reconcile semantics can produce unexpected changes if bundles are updated broadly.

## Alternatives

### Keep hard-coded bootstrapping in the Project controller (status quo)

Pros:

- Simple for a small number of defaults.

Cons:

- Poor separation of concerns; Project lifecycle becomes a dumping ground for service-specific defaults.
- Requires code changes and releases to add/modify defaults.

### Make individual service operators install defaults into all Projects

Pros:

- Service ownership of defaults.

Cons:

- Forces each operator to discover Projects and create per-Project clients.
- N operators x M Projects scaling and duplicated policy/selection logic.

### Encode “desired defaults” directly on the Project API (Project spec)

Pros:

- Project-level explicitness.

Cons:

- Requires API changes and migrations.
- Pushes policy decisions into every Project object (no easy org-level policy).

### Encode “default sets” on the Organization API (Organization spec)

Pros:

- Natural match to Milo’s tenant model (Organizations own Projects).
- Central policy: change the org, affect future (and/or current) Projects.

Cons:

- Requires immediate API changes.
- Needs clear semantics for existing Projects vs new Projects.

### Labels-only selection (recommended initial approach)

Pros:

- No API changes required; leverages existing Project org label.
- Compatible with Kubernetes-native selectors.

Cons:

- Less discoverable than an explicit “defaults” field on Organization.
- Requires careful RBAC to prevent arbitrary broad selectors.

## Infrastructure Needed (Optional)

None beyond:

- new CRDs installed into the Milo management plane
- controller deployed in milo-controller-manager
