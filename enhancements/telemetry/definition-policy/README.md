---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Telemetry Definition and Policy

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Design Details](#design-details)
  - [Log signals](#log-signals)
    - [LogDefinition](#logdefinition)
    - [LogCollectionPolicy](#logcollectionpolicy)
    - [LogRedactionPolicy](#logredactionpolicy)
    - [Collection policy reconciler](#collection-policy-reconciler)
    - [Redaction](#redaction)
    - [Query layer extensions](#query-layer-extensions)
  - [Metric signals](#metric-signals)
    - [Architecture](#architecture)
    - [MetricDefinition](#metricdefinition)
    - [MetricPolicy](#metricpolicy)
    - [Translation controller](#translation-controller)
    - [Scenarios](#scenarios)
- [Open Questions](#open-questions)
- [Alternatives](#alternatives)
- [Implementation History](#implementation-history)

## Summary

This enhancement defines the declarative API layer for both log and metric
signals. Service providers author `LogDefinition` and `MetricDefinition`
resources to declare what logs and metrics their resource kinds publish.
Operators and customers use `LogCollectionPolicy` and `MetricPolicy` to declare
production and collection rules. `LogRedactionPolicy` enforces attribute-level
redaction before logs reach storage.

The result is a stable, discoverable contract between signal producers and
consumers. What a resource publishes is declared once; how it is collected,
transformed, and retained can evolve without breaking the contract.

## Motivation

Without declarative definitions, service providers hand-craft backend-specific
configs (Prometheus scrapes, OTel Collector pipelines, custom formatters).
Consumers can't discover what signals are available or rely on stable
names/units/labels. Adding a new signal or changing a backend requires touching
every collector and dashboard.

A portable contract separates *what a signal is* from *how it is produced*. The
platform translates the contract into whatever backend is active — log pipeline
today, metrics rollup schema tomorrow — without requiring API changes on the
producer or consumer side.

### Goals

- Declarative contract for log identity: `LogDefinition` declares what log types
  a resource kind publishes, their categories, and attribute schemas
- Declarative collection opt-in: `LogCollectionPolicy` lets customers opt
  resources into platform log collection without manual Collector config
- Attribute-level redaction: `LogRedactionPolicy` enforces drop/hash rules
  before logs reach storage or export
- Declarative contract for metric identity: `MetricDefinition` declares metric
  name, instrument type, unit, and label schema per resource kind
- Declarative production rules: `MetricPolicy` binds resource instances to a
  definition via control-plane fields or data-plane telemetry sources
- Backend portability: swapping collectors, exporters, or storage backends
  requires no change to definition or policy objects
- Foundation for alerting and metering (billing/cost attribution) via
  well-defined, standardized signal contracts

### Non-Goals

- Building a new time-series database or log store — those are the pipeline docs
- Replacing external alerting systems; only enabling consistent inputs to them
- Defining billing business logic; only enabling reliable meters for it
- Full log query support — a useful subset is sufficient for the initial
  implementation; see [query-layer](../query-layer/)
- Message body redaction — attribute-level only; body redaction is a follow-on
  if compliance requirements (GDPR, HIPAA) mandate it
- Long-term retention policy — see [retention](../retention/)

## Design Details

> [!WARNING]
>
> **Open question — API group boundary:** `ExportPolicy` lives under
> `telemetry.datumapis.com` (Datum Cloud product layer). The examples below use
> `telemetry.miloapis.com` for log and metric definition/policy CRDs on the
> basis that they are platform-layer primitives (service providers declare signal
> types; collection policy is a platform-operator concern). This must be
> confirmed before any CRD is registered.

### Log signals

#### LogDefinition

`LogDefinition` declares what log types a resource kind publishes, their
category, and the attribute schema consumers can rely on. It does not describe
how logs are collected or routed.

```yaml
apiVersion: telemetry.miloapis.com/v1alpha1
kind: LogDefinition
metadata:
  name: networking-httproxy-access-logs
spec:
  resource:
    group: networking.datumapis.com
    kind: HTTPProxy
  logs:
    - name: access
      description: HTTP access logs for traffic passing through the proxy.
      category: allLogs
      attributes:
        - key: http.method
          description: HTTP request method
        - key: http.target
          description: HTTP request target (path + query)
        - key: http.status_code
          description: HTTP response status code
    - name: waf-events
      description: WAF block and allow events.
      category: allLogs
      attributes:
        - key: waf.rule_id
          description: WAF rule that triggered the event
        - key: waf.action
          description: Action taken (block, allow, log)
```

Log names follow the resource kind and log type: `access`, `waf-events`,
`audit`. Category `allLogs` enables platform collection for all declared types
when a `LogCollectionPolicy` references it.

#### LogCollectionPolicy

`LogCollectionPolicy` opts a project into platform log collection for one or
more resource kinds and categories. Without a policy, no logs are collected —
this prevents unbounded storage growth and gives operators control over what
enters the pipeline.

> [!NOTE]
>
> **Phases 1–4 collect unconditionally.** The `LogCollectionPolicy` controller
> does not exist until Phase 5. Before it ships, the pipeline collects all log
> records that arrive at the bridge with a valid `datum.project.id` — there is
> no opt-in gate. "Without a policy, no logs are collected" becomes the enforced
> behavior only once the Phase 5 controller is deployed.

```yaml
apiVersion: telemetry.miloapis.com/v1alpha1
kind: LogCollectionPolicy
metadata:
  name: collect-gateway-logs
  namespace: acme-prod
spec:
  # Select which resource kinds to enable log collection for.
  resourceSelectors:
    - group: networking.datumapis.com
      kind: HTTPProxy
  # Which log categories to collect. allLogs enables all declared log types
  # for matched resources.
  categories:
    - allLogs
```

#### LogRedactionPolicy

`LogRedactionPolicy` declares attribute-level redaction rules. Rules are applied
at the OTel Collector processing tier before the bridge, so redacted values
never enter NATS or ClickHouse.

```yaml
apiVersion: telemetry.miloapis.com/v1alpha1
kind: LogRedactionPolicy
metadata:
  name: redact-sensitive-headers
  namespace: acme-prod
spec:
  resourceSelectors:
    - group: networking.datumapis.com
      kind: HTTPProxy
  rules:
    - attribute: http.request.header.authorization
      action: drop
    - attribute: http.request.header.cookie
      action: hash
```

Supported actions: `drop` removes the attribute entirely; `hash` replaces the
value with a one-way SHA-256 hash.

#### Collection policy reconciler

The telemetry operator watches `LogCollectionPolicy` resources and reconciles
them against the set of `LogDefinition` resources for matching resource kinds.
For each matched kind and category, the reconciler configures the OTel Collector
to route matching log records into the platform pipeline (`telemetry.logs.<project_id>`
via the NATS bridge). See [logs](../logs/) and
[ingest-pipeline](../ingest-pipeline/) for the pipeline design.

#### Redaction

Redaction is applied at the OTel Collector processing tier, before logs reach
the NATS bridge:

- `drop` — the attribute is removed from the log record
- `hash` — the attribute value is replaced with a one-way hash (SHA-256 hash)

Body redaction is deferred. If compliance requirements mandate it, a future
enhancement will add a `body.mask` action with CEL-based expression matching.

#### Query layer extensions

Once `LogDefinition` resources exist for a project, the query layer and
`datumctl logs` can expose filtering by log type and category. Customers can
filter by the log type names declared for their resources (e.g. `access`,
`waf-events` for an `HTTPProxy`) in addition to the resource attribute filters
available from the base [logs](../logs/) pipeline. No new query protocol is
introduced.

---

### Metric signals

#### Architecture

The metric definition and policy layer introduces a stable contract for metric
identity and production so service providers can declare *what* metrics a
registered resource exposes and *how* those metrics are produced — independent
of the telemetry stack in use.

Key concepts:

- **MetricDefinition** — Declares a single metric for a resource kind: name,
  description, instrument (counter/gauge/histogram), unit (e.g. `seconds`,
  `bytes`, `1`), and label schema. The stable, discoverable surface for
  consumers.
- **MetricPolicy** — Binds resource instances to a definition and specifies
  production rules from either control-plane fields (`ResourceFields`) or
  data-plane telemetry (`DataPlane`).
- **Translation controller** — Watches definitions and policies, validates them,
  and produces backend-specific configs for the active stack (OTel Collector
  pipelines, ClickHouse routing, etc.).

Runtime surfaces the platform supports without changing API objects:

- An **OTLP** path for modern backends
- A **Prometheus-compatible** path (name/label transforms only) for existing
  dashboards and alerts

Definitions are stable; changes that would break consumers (name/unit/label
changes) are treated as new definitions. Status conditions communicate readiness
or validation errors to producers and consumers.

#### MetricDefinition

`MetricDefinition` declares what a single metric is for a given resource kind.
It does not describe how values are generated.

Metric names follow OTel style: `<api-group>.<singular-resource>.<metric>[.<subparts>]`.
Units must be spelled out (`seconds`, `bytes`, `1` for dimensionless).

```yaml
apiVersion: telemetry.miloapis.com/v1alpha1
kind: MetricDefinition
metadata:
  name: gateway-httproute-delivery-upstream-request-latency
  # k8s object labels — not the metric label schema (see spec.labels below)
  labels:
    service.miloapis.com: gateway.networking.k8s.io
spec:
  resource:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
  metric:
    name: gateway.networking.k8s.io.httproute.delivery.upstream_request.latency
    description: Upstream request latency attributed to an HTTPRoute.
    instrument:
      type: histogram
      unit: "seconds"
      aggregationTemporality: delta
  # Label schema: keys that consumers depend on when querying this metric
  labels:
    - key: resource_name
      description: The metadata.name of the HTTPRoute
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

#### MetricPolicy

`MetricPolicy` declares how to produce values for a `MetricDefinition` from
either data-plane telemetry or control-plane fields.

**DataPlane source** — selects incoming OTel/Prometheus samples and
transforms/attributes them:

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
          cel: "value / 1000.0"   # milliseconds → seconds
        labels:
          resource_name: "resource.metadata.name"
          resource_namespace: "resource.metadata.namespace"
          resource_uid: "string(resource.metadata.uid)"
          gateway_name: "attributes['k8s.gateway.name']"
          upstream_cluster: "attributes['envoy.cluster_name']"
```

**ResourceFields source** — derives values from control-plane object fields:

```yaml
apiVersion: telemetry.miloapis.com/v1alpha1
kind: MetricPolicy
metadata:
  name: httproute-condition-status-policy
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

#### Translation controller

A controller watches `MetricDefinition` and `MetricPolicy` resources, validates
them, and produces backend-specific configs:

- **OTel Collector pipelines** — filter, transform, routing, and batch
  processors for data-plane metric policies
- **Status conditions** — both definition and policy objects are updated to
  indicate acceptance or validation errors

By Phase 5, VictoriaMetrics has been decommissioned (Phase 4). The controller
targets the NATS + ClickHouse backend. The exact output format for
control-plane metric policies (e.g. scheduled ClickHouse queries, OTel
Collector transform pipelines) is TBD at implementation.

#### Scenarios

**Control plane — HTTPRoute condition status**

```yaml
# MetricDefinition
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
      required: true
    - key: resource_namespace
      required: true
    - key: resource_uid
      required: true
    - key: condition
      description: Condition type (e.g., Accepted, Programmed)
      required: true
    - key: reason
      required: false
```

```yaml
# MetricPolicy
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

**Data plane — HTTPRoute upstream request count**

```yaml
# MetricDefinition
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
      required: true
    - key: resource_namespace
      required: true
    - key: resource_uid
      required: true
    - key: gateway_name
      required: false
    - key: upstream_cluster
      required: false
    - key: response_code_family
      description: HTTP status code class (e.g., 2xx, 5xx)
      required: false
```

```yaml
# MetricPolicy
apiVersion: telemetry.miloapis.com/v1alpha1
kind: MetricPolicy
metadata:
  name: gateway-httproute-upstream-request-count-policy
spec:
  subject:
    group: gateway.networking.k8s.io
    version: v1
    kind: HTTPRoute
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
          cel: "value"
        labels:
          resource_name: "resource.metadata.name"
          resource_namespace: "resource.metadata.namespace"
          resource_uid: "string(resource.metadata.uid)"
          gateway_name: "attributes['k8s.gateway.name']"
          upstream_cluster: "attributes['envoy.cluster_name']"
          response_code_family: "attributes['http.response.status_code_class']"
```

## Alternatives

**Bespoke backend configs per signal** — each service provider adds a custom
Prometheus scrape job, OTel Collector processor, or Loki rule. Rejected because
it produces drift in names/units/labels, makes migration costly, and provides no
discoverable contract for consumers. The definition/policy layer is the
alternative.

**Embed definitions in service provider code** — service providers register
metrics by calling a platform API at startup rather than authoring CRDs.
Rejected because it ties signal registration to deployment lifecycles, prevents
static analysis and discovery, and requires live service instances to exist
before a signal contract can be consulted.

## Implementation History

- 2025-03 — MetricDefinition and MetricPolicy concepts proposed in the
  policy-driven metrics platform enhancement
- 2026-06 — Log signals (LogDefinition, LogCollectionPolicy, LogRedactionPolicy)
  combined with metric signals into this unified definition-policy enhancement;
  separated from the pipeline docs (logs, metrics) and from retention
