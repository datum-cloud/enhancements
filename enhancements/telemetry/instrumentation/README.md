---
status: provisional
stage: alpha
latest-milestone: "v0.1"
---

# Instrumentation library

- [Summary](#summary)
- [Motivation](#motivation)
- [Why adopt it](#why-adopt-it)
- [Design](#design)
- [Labels](#labels)
- [First adopters](#first-adopters)
- [Evaluation](#evaluation)
- [Out of scope](#out-of-scope)

## Summary

Our codebase has incomplete, inconsistent telemetry instrumentation. This
leads to mental toil by constantly re-implementing the same basic
boilerplate. Metrics sometimes never actually get exported. Labels are
inconsistent. Traces across services, if they existed, would be cumbersome.

This proposes [`go.miloapis.com/instrumentation`][repo], a small Go library
in `milo-os` (it's provider-agnostic), licensed Apache-2.0 so any repository
can import it. Version 1 covers logs and metrics only. If it isn't useful, we
drop it.

This is about how services emit telemetry. Where telemetry goes, and what
customers can see, belong to the rest of this telemetry system and are out of
scope here.

## Motivation

- **Unexported metrics.** A controller-runtime manager serves
  `ctrlmetrics.Registry`. Metrics created with `promauto` land on
  `prometheus.DefaultRegisterer`, which nothing serves. Nothing fails.
  - network-services-operator: 13 metric families. Three production alerts
    can never fire ([network-services-operator#359][nso-359]).
  - `milo-os/locations`: 11 metrics. Its metrics config looks copied from
    network-services-operator, bug included.
- **Colliding labels, in every repository checked.**
  network-services-operator keys gauges by `{namespace, name}`, so they
  overwrite each other across projects, and logs the project under `cluster`,
  which infra already uses for the physical cluster. `milo-os/activity`
  labels a metric `namespace`, which Prometheus renames to
  `exported_namespace`.
- **The check is hard to write correctly.** network-services-operator's own
  pending fix tests the default registry by gathering it, which cannot see an
  unset labelled metric, so it passes on the bug. A first prototype of this
  library did the same. Intercepting registrations at runtime saw none of
  them, depending on package initialization order. A review found the first
  logger check passed while a binary bypassed the logger. Each repository
  writing its own check would repeat these.

## Why adopt it

> Add one test. If your metrics aren't reaching Prometheus, it fails. Add
> another, and it fails if your logs bypass the standard logger.

The tests work before any migration. Migrating to the library is the fix
they point to.

One wrong key name can send internal operator logs into a customer's
telemetry. The library reserves that key: a metric using it panics at
startup. Log keys are not checked, so the reservation documents the rule for
logs rather than enforcing it.

## Design

| Package | Contents |
|---|---|
| `labels` | Label names for metrics and log keys alike, generated from `labels.yaml`; reserved-name lookup |
| `buildinfo` | The binary's `build_version` and the library's `instrumentation_version` |
| `metrics` | `client_golang`-style constructors that register on the served registry and refuse reserved names; `Handler` for servers that are not managers |
| `logging` | JSON at info level, no sampling, UTC timestamps, versions on every line. It does not redact; don't log secrets |
| `instrumenttest` | `AssertNothingStranded`, `AssertStandardLogger`, and `Lint` |

`AssertNothingStranded` reads the module's source and fails at each use of
the default registry, at each registration that bypasses the library's
label checks, and at a disabled manager metrics server when nothing else
serves metrics. `AssertStandardLogger` reads source the same way and fails
at any other logger constructor, at any `SetLogger` in a file that never
calls `logging.New`, and when no file sets a logger built with `logging.New`.
Reading source finds every use wherever the test lives, regardless of
initialization order. `Lint` runs `promlint`.

Rules:

- controller-runtime binaries only.
- Reuse upstream and return upstream types. If a file only re-exports
  upstream code, delete it.

## Labels

| | Reserved: where it ran, who receives it | Subject: what it's about | Build: what emitted it |
|---|---|---|---|
| Examples | `app`, `region`, `cluster`, `namespace`, `pod`, `project_name` | `organization`, `project`, `resource_kind`, `resource_name`, `reason` | `build_version`, `instrumentation_version` |
| Set by | Collection pipelines ([infra#4051][infra-4051], `milo-os/telemetry`) | Application code | This library |

- One vocabulary for metric labels and log keys, so a metric and the log
  lines about the same resource join in our Prometheus- and Loki-compatible
  query API. `labels.yaml` is the only source.
- A reserved name collides with the pipeline's value: Prometheus renames it
  `exported_<name>`, Loki `<name>_extracted`. `project_name` routes a record
  into that project's tenant, so internal code uses `project`.
- `organization` names the owner of an organization-scoped resource only. It
  is never derived from a project, which would copy hierarchy out of the core
  control plane, against this system's guiding principles.
- `resource_name` and `resource_namespace` match the label keys
  [`MetricDefinition`](../definition-policy/) already uses for customer
  metrics.
- Build labels go on every log line but on one metric only,
  `instrumentation_build_info`. On every series, a version label would replace
  each series on every deploy.
- Names are Prometheus-style. Other languages get generated constants from
  the same file when a non-Go service adopts it; evaluate OpenTelemetry Weaver
  as the generator first.

## First adopters

| Repository | Work |
|---|---|
| network-services-operator | Build on the pending fix for #359, which moves the 13 families to the served registry, adds `project`, and updates the alerts. Swap its local checks for the library's, serve the extension server's metrics with `metrics.Handler`, key logs with `labels`, and adopt `logging`. Done when a scrape of its e2e environment shows the gateway metrics and `GatewayListenerCertUnusable` fires there for a broken certificate |
| `milo-os/locations` | Add the test and confirm it names the 11 metrics. Migrate its metrics |

## Evaluation

Review the library once both first adopters have migrated. Keep it if it is
earning its place, judged on these signals:

- The tests found a real problem in each adopter that its own checks missed.
- A repository adopts it in an afternoon, without help from its authors.
- Teams adopt it without being asked.
  `count by (instrumentation_version) (instrumentation_build_info)` shows
  adoption across the fleet without anyone reporting it.

A weak signal means trim or fix the part that isn't working, not drop
everything. If the library as a whole isn't paying for itself, freeze it and
keep the tests as a snippet each repository copies.

## Out of scope

- Tracing and profiling. These are a later phase, if version 1 is kept.
- The internal operations pipeline (`datum-cloud/infra`).
- Customer-facing telemetry ([infra#3221][infra-3221]). The vocabulary
  reserves its routing key, `project_name`, so the two don't collide.

[repo]: https://github.com/milo-os/instrumentation
[nso-359]: https://github.com/datum-cloud/network-services-operator/issues/359
[infra-4051]: https://github.com/datum-cloud/infra/issues/4051
[infra-3221]: https://github.com/datum-cloud/infra/issues/3221
