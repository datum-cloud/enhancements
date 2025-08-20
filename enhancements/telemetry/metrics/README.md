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

<!-- omit from toc -->
# Policy-Driven Metrics Platform

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
  - [Concepts.](#concepts)
  - [Architecture \& flexibility.](#architecture--flexibility)
  - [Desired outcome \& success measures.](#desired-outcome--success-measures)
- [Design Details](#design-details)
  - [Metric Definitions](#metric-definitions)
    - [Complete MetricDefinition example](#complete-metricdefinition-example)
  - [Metric Policies](#metric-policies)
  - [Scenarios](#scenarios)
    - [Control Plane Metrics](#control-plane-metrics)
    - [Data Plane Metrics](#data-plane-metrics)
- [Implementation History](#implementation-history)

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

This enhancement adds a **policy-driven metrics platform** to Milo that
introduces a stable contract for metric identity and production so service
providers can declare *what* metrics a registered resource exposes and *how*
those metrics are produced—independent of the telemetry stack in use. Today,
operators hand-craft backend-specific configs (e.g., kube-state-metrics,
Prometheus scrapes). With this change, providers author portable **definitions**
and **policies** that the platform translates into whatever
collectors/processors/exporters are deployed, allowing the implementation to
evolve without API churn.

The result is consistent metric discovery, naming, units, and labels across the
platform. This foundation also enables future **alerting** (policy-driven
conditions over standardized metrics) and **metering for billing and cost
attribution** (well-defined counters/histograms per tenant/resource) without
revisiting every workload or dashboard.

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this Enhancement.  Describe why the change is important and the benefits to users.
-->

Operators and service providers need a consistent, declarative way to define
resource metrics and to produce them from either fields on control-plane
resources or data-plane telemetry. Backend-specific configuration creates drift
in names/units/labels and makes migration costly. A portable contract avoids
per-backend rewrites, improves discoverability, and lets the underlying
telemetry components change over time without breaking users.

### Goals

<!--
List the specific goals of the Enhancement. What is it trying to achieve? How will we
know that this has succeeded?
-->

- Provide a stable, declarative contract to:
  - Define metric identity, description, instrument type, unit, and label schema
    per resource kind.
  - Declare policies that map resource fields and/or data-plane telemetry to
    metric values.
- Ensure portability across implementations (OTLP, Prometheus-compatible, and
  future backends) with no API changes.
- Standardize naming/units/labels to eliminate drift and enable metric
  discovery.
- Support multi-tenant/authorization-aware exposure aligned with Milo IAM.
- Establish a forward path for alerting and metering (billing/cost attribution)
  to consume these standardized metrics.
- Measure success by: reduced bespoke configs, ability to swap
  collectors/exporters without changing API objects, and adoption across core
  Milo resources.

### Non-Goals

<!--
What is out of scope for this Enhancement? Listing non-goals helps to focus discussion
and make progress.
-->

- Building a new time-series database or UI dashboard.
- Replacing external alerting systems; only enabling consistent inputs to them.
- Defining billing business logic; only enabling reliable meters for it.
- Introducing a new telemetry format or standard

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

Introduce a contract that separates *what a metric is* from h*ow it is
produced*, and a translation layer that compiles that contract into the active
telemetry implementation.

### Concepts.

- **MetricDefinition** — Declares a single metric for a resource kind: name,
  description, instrument (counter/gauge/histogram), unit (spelled out, e.g.,
  `seconds`, `bytes`, `1`), and label schema. It is the discoverable, stable surface
  for consumers.
- **MetricPolicy** — Binds resource instances to a definition and specifies
  production rules:
  - **Resource-fields source**: derive values from control-plane objects
    (including list expansion for conditions/arrays, timestamp extraction,
    simple transforms).
  - **Data-plane source**: select incoming telemetry (e.g., from Envoy or edge
    router) by attributes and transform samples/units; attach resource identity
    labels.

### Architecture & flexibility.

- **Controller/translator**. A controller watches definitions/policies,
  validates them, and produces backend-specific configs for the active stack
  (e.g., OTel Collector pipelines, Prometheus relabeling/recording rules, or
  other processors). Swapping collectors/exporters or moving between “pull” and
  “push” models is an implementation detail behind the translator.

- **Runtime surfaces**. The platform can support multiple runtimes without
  changing API objects:
  - An **OTLP** path for modern backends.
  - A **Prometheus-compatible** path (name/label transforms only) for existing dashboards/alerts.

- **Source neutrality**. Policies can combine control-plane fields and
	data-plane telemetry. Implementations may realize this via:
  - Direct emission from the controller (control-plane metrics).
  - OTel Collector processors for attribute-based selection and unit conversion.
  - Adapters for existing components (e.g., translating policies into
    kube-state-metrics configuration or Prometheus rules) to ease migration.
- **Change management**. Definitions are stable; changes that would break
  consumers (name/unit/label changes) are treated as new definitions. Status
  conditions (e.g., Accepted) communicate readiness or validation errors to
  producers/consumers. Stability indicators are included on metric definitions.

### Desired outcome & success measures.

- Core Milo resources (e.g. Project, Groups) publish metric definitions;
  policies generate series from both control-plane fields and data-plane
  telemetry.
- Backends/collectors can be changed with no API edits; consumers see uninterrupted, stable metrics.

<!-- ### User Stories (Optional) -->

<!--
Detail the things that people will be able to do if this Enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system. The goal here is to make this feel real for users without getting
bogged down.
-->

<!-- #### Story 1 -->

<!-- #### Story 2 -->

<!-- ### Notes/Constraints/Caveats (Optional) -->

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

<!-- ### Risks and Mitigations -->

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

This section details the resources, fields, and concrete examples that implement
the policy-driven metrics platform. It uses real HTTPRoute examples from the
control plane and Envoy metrics from the data plane.

### Metric Definitions

**MetricDefinition** declares **what** a single metric is for a given resource kind: identity (name), description, instrument type, unit, and the **label schema** consumers can rely on. It does **not** describe how values are generated.

Key points:
- Metric names follow OTel style: `<api-group>.<singular-resource>.<metric>[.<subparts>]`.
- Units must be spelled out (`seconds`, `bytes`, `1` for dimensionless).
- Status conditions communicate acceptance and readiness.

#### Complete MetricDefinition example

```yaml
apiVersion: telemetry.miloapis.com/v1alpha1
kind: MetricDefinition
metadata:
  # Name of this definition object (cluster-unique, kebab-case recommended)
  name: gateway-httproute-delivery-upstream-request-latency
  labels:
    service.miloapis.com: gateway.networking.k8s.io
spec:
  # --- Unversioned reference to the owning resource kind ---
  resource:
    # API group of the resource that owns this metric
    group: gateway.networking.k8s.io
    # Plural name of the resource
    kind: HTTPRoute

  # --- Metric identity and instrument configuration ---
  metric:
    # Fully-qualified OTel-style name: <group>.<singular>.<metric>[...]
    # NOTE: singular form "httproute" is used here.
    name: gateway.networking.k8s.io.httproute.delivery.upstream_request.latency
    # Human-readable purpose for docs and discovery UIs
    description: Upstream request latency attributed to an HTTPRoute.
    instrument:
      # Enum: Counter | UpDownCounter | Gauge | Histogram
      type: Histogram
      # Unit spelled out; seconds for latency; use "1" for dimensionless
      unit: "seconds"
      # Enum: cumulative | delta (applies where meaningful; optional for histogram)
      aggregationTemporality: delta
      # Optional, forward-looking: processor-specific histogram hints
      # (e.g., defaultBuckets). Omitted here to keep the definition portable.

  # --- Label schema advertised by this metric ---
  labels:
    # Keys must follow OTel attribute naming; do not encode units in labels.
    - key: resource_name
      # Describe where the label value originates from conceptually
      description: The metadata.name of the HTTPRoute
      # Whether producers must provide this label for every point
      required: true
    - key: resource_namespace
      description: The metadata.namespace of the HTTPRoute
      required: true
    - key: resource_uid
      description: The metadata.uid of the HTTPRoute
      required: true
    - key: gateway_name
      description: Gateway associated with the route (if known)
      required: false
    - key: upstream_cluster
      description: Upstream cluster identifier from data plane telemetry
      required: false

status:
  conditions:
    - type: Accepted
      status: "True"
      reason: TelemetryRegistered
      message: Metric will be exposed when a matching MetricPolicy emits series.
```

### Metric Policies

**MetricPolicy** declares **how** to produce values for a single
`MetricDefinition` from either:

- **Control-plane fields** (`ResourceFields`): read k8s object fields and
  transform them.
- **Data-plane telemetry** (`DataPlane`): select incoming OTel/Prometheus
  samples and transform/attribute them.

Policies bind a subject (the resource instances) to a targetMetric (by name). A
policy uses one source type at a time.

**Complete MetricPolicy example — ResourceFields source**

```yaml
apiVersion: telemetry.miloapis.com/v1alpha1
kind: MetricPolicy
metadata:
  # Unique policy name
  name: httproute-condition-status-policy
spec:
  # --- Subject (what objects this policy applies to) ---
  subject:
    # Versioned reference to the resource kind so that different policies can be
    # configured per version in case field paths change between resources.
    group: gateway.networking.k8s.io
    version: v1
    kind: HTTPRoute

  # --- Target metric (must match a MetricDefinition.spec.metric.name) ---
  targetMetric:
    name: gateway.networking.k8s.io.httproute.status.condition

  # --- Source: derive values from control-plane fields ---
  source:
    type: ResourceFields
    resourceFields:
      # seriesFrom: explode an array to multiple series; "item" is the element
      seriesFrom:
        # JSONPath-like path into "resource" (the api object)
        path: "resource.status.conditions"
        # "value" is a CEL expression evaluated per "item"
        value:
          # Emit 1 when condition is True, else 0
          cel: "item.status == 'True' ? 1 : 0"
        # Map labels using CEL against "item" and "resource"
        labels:
          condition: "item.type"
          reason: "item.reason"
          resource_name: "resource.metadata.name"
          resource_namespace: "resource.metadata.namespace"
          resource_uid: "string(resource.metadata.uid)"

      # Optional: guard emission for the whole policy (drop series if true)
      # dropIf:
      #   cel: "resource.metadata.deletionTimestamp != null"

      # Optional: static labels always attached by the policy
      # staticLabels:
      #   environment: "prod"
```

**Complete MetricPolicy example — DataPlane source**

```yaml
apiVersion: telemetry.miloapis.com/v1alpha1
kind: MetricPolicy
metadata:
  name: httproute-upstream-latency-policy
spec:
  subject:
    group: gateway.networking.k8s.io
    version: v1
    kind: HTTPRoute

  targetMetric:
    name: gateway.networking.k8s.io.httproute.delivery.upstream_request.latency

  source:
    type: DataPlane
    dataPlane:
      # Select incoming telemetry to consume
      selector:
        otel:
          # Backend-native metric name to read (e.g., Envoy stat or OTEL metric)
          metricName: "envoy.cluster.upstream_rq_time"
          # Enum: cumulative | delta (match the source’s temporality)
          temporality: delta
          # Attribute selectors bind samples to the k8s resource identity
          attributeSelectors:
            # Match HTTPRoute name/namespace carried on the sample
            - key: "k8s.httproute.name"
              valueFrom: "resource.metadata.name"
            - key: "k8s.namespace.name"
              valueFrom: "resource.metadata.namespace"
          # Optional additional filters (prefix/regex on attribute values)
          # attributeFilters:
          #   - key: "http.route"
          #     regex: "^/api/.*"

      # Transform the selected samples into the target metric’s unit/labels
      transform:
        # CEL over "value" (numeric), "attributes" (map), and "resource"
        value:
          # Convert milliseconds -> seconds
          cel: "value / 1000.0"
        labels:
          resource_name: "resource.metadata.name"
          resource_namespace: "resource.metadata.namespace"
          resource_uid: "string(resource.metadata.uid)"
          gateway_name: "attributes['k8s.gateway.name']"
          upstream_cluster: "attributes['envoy.cluster_name']"

      # Optional: sampling and rate controls (implementation-dependent hints)
      # sampling:
      #   maxSamplesPerSecond: 500
      #   dropOnBackpressure: true
```

### Scenarios

#### Control Plane Metrics

**Goal**: Use resource fields on HTTPRoute to produce identity, timestamps, and
condition metrics.

**MetricDefinition — HTTPRoute condition status (1/0)**

```yaml
apiVersion: telemetry.miloapis.com/v1alpha1
kind: MetricDefinition
metadata:
  name: gateway-httproute-condition-status
spec:
  resource:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
  metric:
    name: gateway.networking.k8s.io.httproute.status.condition
    description: Status for each HTTPRoute condition (1=True, 0 otherwise).
    instrument:
      type: gauge
      unit: "1"
  labels:
    - key: resource_name
      description: HTTPRoute name
      required: true
    - key: resource_namespace
      description: HTTPRoute namespace
      required: true
    - key: resource_uid
      description: HTTPRoute UID
      required: true
    - key: condition
      description: Condition type (e.g., Accepted, Programmed)
      required: true
    - key: reason
      description: Condition reason string (may be empty)
      required: false
status:
  conditions:
    - type: Accepted
      status: "True"
      reason: TelemetryRegistered
```

**MetricPolicy — produce per-condition series from control-plane fields**

```yaml
apiVersion: telemetry.miloapis.com/v1alpha1
kind: MetricPolicy
metadata:
  name: gateway-httproute-condition-status-policy
spec:
  subject:
    group: gateway.networking.k8s.io
    version: v1
    kind: HTTPRoute
  targetMetric:
    name: gateway.networking.k8s.io.httproute.status.condition
  source:
    type: ResourceFields
    resourceFields:
      seriesFrom:
        path: "resource.status.conditions"
        value:
          cel: "item.status == 'True' ? 1 : 0"
        labels:
          condition: "item.type"
          reason: "item.reason"
          resource_name: "resource.metadata.name"
          resource_namespace: "resource.metadata.namespace"
          resource_uid: "string(resource.metadata.uid)"
```

> [!NOTE]
>
> Similar patterns can define creation_time, deletion_time, and an info metric with value 1.

#### Data Plane Metrics

**Goal**: Attribute Envoy gateway telemetry to `HTTPRoute` and expose standardized metrics.

**MetricDefinition — upstream request count (counter)**

```yaml
apiVersion: telemetry.miloapis.com/v1alpha1
kind: MetricDefinition
metadata:
  name: gateway-httproute-upstream-request-count
spec:
  resource:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
  metric:
    name: gateway.networking.k8s.io.httproute.delivery.upstream_request.count
    description: Upstream request count attributed to an HTTPRoute.
    instrument:
      type: counter
      unit: "1"
      aggregationTemporality: cumulative
  labels:
    - key: resource_name
      description: HTTPRoute name
      required: true
    - key: resource_namespace
      description: HTTPRoute namespace
      required: true
    - key: resource_uid
      description: HTTPRoute UID
      required: true
    - key: gateway_name
      description: Owning Gateway (if known)
      required: false
    - key: upstream_cluster
      description: Upstream cluster identifier
      required: false
    - key: response_code_family
      description: HTTP status code class (e.g., 2xx, 5xx)
      required: false
status:
  conditions:
    - type: Accepted
      status: "True"
      reason: TelemetryRegistered
```

**MetricPolicy — map Envoy counter to the platform metric**

```yaml
apiVersion: telemetry.miloapis.com/v1alpha1
kind: MetricPolicy
metadata:
  name: gateway-httproute-upstream-request-count-policy
spec:
  subject:
    group: gateway.networking.k8s.io
    version: v1
    resource: HTTPRoute
  targetMetric:
    name: gateway.networking.k8s.io.httproute.delivery.upstream_request.count
  source:
    type: DataPlane
    dataPlane:
      selector:
        otel:
          metricName: "envoy.cluster.upstream_rq_total"
          temporality: cumulative
          attributeSelectors:
            - key: "k8s.httproute.name"
              valueFrom: "resource.metadata.name"
            - key: "k8s.namespace.name"
              valueFrom: "resource.metadata.namespace"
      transform:
        value:
          # Pass-through counter sample
          cel: "value"
        labels:
          resource_name: "resource.metadata.name"
          resource_namespace: "resource.metadata.namespace"
          resource_uid: "string(resource.metadata.uid)"
          gateway_name: "attributes['k8s.gateway.name']"
          upstream_cluster: "attributes['envoy.cluster_name']"
          response_code_family: "attributes['http.response.status_code_class']"
```

**MetricDefinition — upstream request latency (histogram in seconds)**

```yaml
apiVersion: telemetry.miloapis.com/v1alpha1
kind: MetricDefinition
metadata:
  name: gateway-httproute-upstream-latency
spec:
  resource:
    group: gateway.networking.k8s.io
    resource: HTTPRoute
  metric:
    name: gateway.networking.k8s.io.httproute.delivery.upstream_request.latency
    description: Upstream request latency attributed to an HTTPRoute.
    instrument:
      type: histogram
      unit: "seconds"
      aggregationTemporality: delta
  labels:
    - key: resource_name
      description: HTTPRoute name
      required: true
    - key: resource_namespace
      description: HTTPRoute namespace
      required: true
    - key: resource_uid
      description: HTTPRoute UID
      required: true
    - key: gateway_name
      description: Owning Gateway (if known)
      required: false
    - key: upstream_cluster
      description: Upstream cluster identifier
      required: false
status:
  conditions:
    - type: Accepted
      status: "True"
      reason: TelemetryRegistered
```

**MetricPolicy — convert Envoy latency to seconds and attribute**

```yaml
apiVersion: telemetry.miloapis.com/v1alpha1
kind: MetricPolicy
metadata:
  name: gateway-httproute-upstream-latency-policy
spec:
  subject:
    apiGroup: gateway.networking.k8s.io
    resource: httproutes
    scope: Namespaced
  targetMetric:
    name: gateway.networking.k8s.io.httproute.delivery.upstream_request.latency
  source:
    type: DataPlane
    dataPlane:
      selector:
        otel:
          metricName: "envoy.cluster.upstream_rq_time"
          temporality: delta
          attributeSelectors:
            - key: "k8s.httproute.name"
              valueFrom: "resource.metadata.name"
            - key: "k8s.namespace.name"
              valueFrom: "resource.metadata.namespace"
      transform:
        value:
          # Convert milliseconds -> seconds to match MetricDefinition unit
          cel: "value / 1000.0"
        labels:
          resource_name: "resource.metadata.name"
          resource_namespace: "resource.metadata.namespace"
          resource_uid: "string(resource.metadata.uid)"
          gateway_name: "attributes['k8s.gateway.name']"
          upstream_cluster: "attributes['envoy.cluster_name']"
```

<!-- ## Production Readiness Review Questionnaire -->

<!--

Production readiness reviews are intended to ensure that features are observable,
scalable and supportable; can be safely operated in production environments, and
can be disabled or rolled back in the event they cause increased failures in
production.

See more in the PRR Enhancement at https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved
for the Enhancement to move to `implementable` status and be included in the release.
-->

<!-- ### Feature Enablement and Rollback -->

<!--
This section must be completed when targeting alpha to a release.
-->

<!-- #### How can this feature be enabled / disabled in a live cluster? -->

<!--
Pick one of these and delete the rest.
-->
<!--
- [ ] Feature gate
  - Feature gate name:
  - Components depending on the feature gate:
- [ ] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control plane?
  - Will enabling / disabling the feature require downtime or reprovisioning of a node?
-->

<!-- #### Does enabling the feature change any default behavior? -->

<!--
Any change of default behavior may be surprising to users or break existing
automations, so be extremely careful here.
-->

<!-- #### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)? -->

<!--
Describe the consequences on existing workloads (e.g., if this is a runtime
feature, can it break the existing applications?).

Feature gates are typically disabled by setting the flag to `false` and
restarting the component. No other changes should be necessary to disable the
feature.
-->

<!-- #### What happens if we reenable the feature if it was previously rolled back? -->

<!-- #### Are there any tests for feature enablement/disablement? -->

<!-- ### Rollout, Upgrade and Rollback Planning -->

<!--
This section must be completed when targeting beta to a release.
-->

<!-- #### How can a rollout or rollback fail? Can it impact already running workloads? -->

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->

<!-- #### What specific metrics should inform a rollback? -->

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

<!-- #### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested? -->

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

<!-- #### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.? -->

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

<!-- ### Monitoring Requirements -->

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

<!-- #### How can an operator determine if the feature is in use by workloads? -->

<!--
Ideally, this should be a metric. Operations against the API (e.g., checking if
there are objects with field X set) may be a last resort. Avoid logs or events
for this purpose.
-->

<!-- #### How can someone using this feature know that it is working for their instance? -->

<!--
For instance, if this is an instance-related feature, it should be possible to
determine if the feature is functioning properly for each individual instance.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so
that they can verify correct enablement and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->
<!--
- [ ] Events
  - Event Reason:
- [ ] API .status
  - Condition name:
  - Other field:
- [ ] Other (treat as last resort)
  - Details:
-->

<!-- #### What are the reasonable SLOs (Service Level Objectives) for the enhancement? -->

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

<!-- #### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service? -->

<!--
Pick one more of these and delete the rest.
-->
<!--
- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:
-->

<!-- #### Are there any missing metrics that would be useful to have to improve observability of this feature? -->

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

<!-- ### Dependencies -->

<!--
This section must be completed when targeting beta to a release.
-->

<!-- #### Does this feature depend on any specific services running in the cluster? -->

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

<!-- ### Scalability -->

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

<!-- #### Will enabling / using this feature result in any new API calls? -->

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

<!-- #### Will enabling / using this feature result in introducing new API types? -->

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

<!-- #### Will enabling / using this feature result in any new calls to the cloud provider? -->

<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->

<!-- #### Will enabling / using this feature result in increasing size or count of the existing API objects? -->

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

<!-- #### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs? -->

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

<!-- #### Will enabling / using this feature result in non-negligible increase of resource usage in any components? -->

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

<!-- #### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)? -->

<!--
Focus not just on happy cases, but primarily on more pathological cases.

Are there any tests that were run/should be run to understand performance
characteristics better and validate the declared limits?
-->

<!-- ### Troubleshooting -->

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

<!-- #### How does this feature react if the API server is unavailable? -->

<!-- #### What are other known failure modes? -->

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

<!-- #### What steps should be taken if SLOs are not being met to determine the problem? -->

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

<!-- ## Drawbacks -->

<!--
Why should this Enhancement _not_ be implemented?
-->

<!-- ## Alternatives -->

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

<!-- ## Infrastructure Needed (Optional) -->

<!--
Use this section if you need things from another party. Examples include a
new repos, external services, compute infrastructure.
-->
