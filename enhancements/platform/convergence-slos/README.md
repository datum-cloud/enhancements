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

- Enforcement wiring — the end-to-end timeout pass, alert rules, and dashboards. This is owned by the deployment's infrastructure repository.
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
| Project creates and becomes ready | project control plane accepts requests | 30s |
| Organization onboarding completes | organization ready for use | 60s |
| IAM policy binding takes effect | granted access is enforced | 30s |
| Gateway programs | gateway Programmed with address assigned | 30s |
| Custom hostname resolves | authoritative answer for the hostname | 60s |
| Managed DNS zone and record sets program | record served by the authoritative nameservers | 60s |
| TLS certificate issues | certificate ready and serving on the listener | 120s (provisional) |
| HTTPRoute programs and serves | route accepted and traffic served | 30s |
| Workload deploys and serves | workload available in each target location | 300s (provisional) |
| Custom domain verifies, verification record in place | domain marked verified once the verification record is answerable | 300s |
| DNS record reaps on hostname removal | authoritative answer gone, no SERVFAIL | 60s |

The thresholds are derived from reasonable expectation, not from observation. Observed healthy convergence suggests a calibration; each number is then ratified by judging whether the wait is one a customer could reasonably expect. Where observed latency sits above what is reasonable to expect, the threshold holds and the gap is a defect to fix, not headroom to grant.

The proposed numbers are seed calibrations from observed healthy convergence and existing per-path budgets; each should be validated against latency percentiles over more runs before enforcement, so the gate bounds expectation without introducing flake. Rows marked provisional have no healthy measurement yet and are set from the reasonable-expectation judgment alone. Where a path's observed latency today sits above its threshold — for example, domain verification rechecks that back off to hours — the threshold holds and the gap is a defect to fix. A composite journey (hostname created through resolving and serving traffic) is bounded by the sum of its component thresholds. The table is the artifact this enhancement delivers.

Enforcement surfaces are owned by the deployment's infrastructure repository:

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
- Deployment-side implementation tracked separately in the deployment's infrastructure repository.

## Alternatives

- **Keep worst-case timeouts and rely on customer reports** — the status quo; the failure stays invisible until a customer complains.
- **Drop timeouts to seconds everywhere** — over-tightens, producing false alarms on normal propagation without catching anything a calibrated threshold would miss.
