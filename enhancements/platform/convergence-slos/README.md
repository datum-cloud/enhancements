---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Bound convergence to a reasonable-expectation threshold

## Summary

Measure user-facing convergence against the latency a customer can reasonably expect, not the latency the underlying systems happen to take at their worst.

For each user-facing convergence path, define a **reasonable-expectation threshold** — the latency beyond which the delay exceeds what a customer could reasonably expect, even when the operation eventually succeeds — and make that budget the shared yardstick every latency gate enforces: end-to-end tests, alerts, and dashboards.

## Motivation

Timeouts and alerts today are anchored to worst-case system latency: propagation plus reconcile backlog plus headroom. That measures what the systems can get away with, not what a customer could reasonably expect.

The result is a blind spot for **slow-but-eventual** success. A hostname, certificate, or route that takes many minutes to begin working still "succeeds," and every gate stays green — while the customer experiences a product that is effectively broken.

Concrete case: a DNS-reap end-to-end test sat at a 16-minute timeout and passed on latency far beyond anything a customer could reasonably expect. In production, an operation that takes too long is a defect even when it eventually completes.

### Goals

- Define a reasonable-expectation threshold for each user-facing convergence path.
- Establish that threshold as the single yardstick across every surface that asserts or observes latency — end-to-end tests, alerting, and dashboards.
- Make slow-but-eventual convergence a detectable failure rather than a silent pass.

### Non-Goals

- Datum's own enforcement wiring — the end-to-end timeout pass, alert rules, and dashboards. This is owned in Datum's infrastructure repository.
- Race conditions that a wall-clock budget cannot detect. A defect whose outcome is eventually correct but which a single missed step could strand is not distinguishable by any timeout; that belongs in unit tests.
- Controller reconcile or requeue behavior. This is owned by the component repositories (for example, network-services-operator).

## Proposal

Treat each user-facing convergence path as a small service-level objective with one number: the reasonable-expectation threshold. That number is a product decision, set to what a reasonable customer would expect the operation to take — not derived from observed system latency. Observed latency confirms the threshold is achievable; it does not define it.

Enforcement is layered, and every layer reads the same number:

- **Tests** — an end-to-end test for a path fails when convergence exceeds its threshold, catching a regression before release.
- **Alerts** — production alerts fire when convergence exceeds the threshold, not only when convergence never happens. Existing route and gateway alerts are never-converge shaped and miss the slow-but-succeeds case.
- **Dashboards** — convergence-latency percentiles are shown against the threshold, so drift becomes visible before a customer feels it.

### Notes/Constraints/Caveats

- The threshold is an upper bound on acceptable latency, not a target. Healthy convergence should sit well inside it; the gate exists to catch the tail.
- A threshold set too tight produces false alarms on normal variance; too loose reproduces the current blind spot. Each is calibrated against observed healthy latency plus headroom, then held under what a reasonable customer would expect.

### Risks and Mitigations

- **False alarms on normal variance** — calibrate against observed healthy percentiles with headroom, and alert on sustained breach rather than single samples.
- **Threshold drift** — thresholds are reviewed as product service-level objectives, not quietly raised to turn a failing gate green.

## Design Details

The initial set of user-facing convergence paths and their thresholds:

| Convergence path | Signal that it converged | Reasonable-expectation threshold (proposed) |
|---|---|---|
| Custom hostname resolves | authoritative answer for the hostname | TBD |
| TLS certificate issues | certificate ready and serving on the listener | TBD |
| HTTPRoute programs and serves | route accepted and traffic served | TBD |
| DNS record reaps on hostname removal | authoritative answer gone, no SERVFAIL | TBD |

Budgets are left `TBD` pending calibration against observed healthy latency per path. The table is the artifact this enhancement delivers.

Enforcement surfaces in Datum's deployment are owned by Datum's infrastructure repository:

- End-to-end timeouts bounded to the threshold.
- Alert rules and dashboards keyed to the threshold — these build on the telemetry system (see [telemetry](../../telemetry/)) for the underlying metrics.

## Production Readiness Review Questionnaire

### Monitoring Requirements

Each path exposes a convergence-latency metric — the duration from request to the converged signal. Alerts and dashboards are derived from those metrics against the threshold. Metric definition and export follow the telemetry system.

### Dependencies

- The telemetry system, for the convergence-latency metrics that alerts and dashboards consume.
- Component controllers, to emit or make observable the converged signal for each path.

## Implementation History

- Tracking issue opened in datum-cloud/enhancements.
- Datum-side implementation tracked separately in Datum's infrastructure repository.

## Alternatives

- **Keep worst-case timeouts and rely on customer reports** — the status quo; the failure stays invisible until a customer complains.
- **Drop timeouts to seconds everywhere** — over-tightens, producing false alarms on normal propagation without catching anything a calibrated threshold would miss.
