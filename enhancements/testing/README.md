# E2E Testing Strategy

## Goal

A concise, actionable guide for running and extending end‑to‑end (E2E) tests across backend controllers and UI/UX, both locally and in CI/CD.

---

## Current State

| Area | What We Have Today |
| ---- | ------------------ |
|      |                    |

| **Backend**  | Functional E2E tests in `network-services-operator/test/e2e` run against a local **Kind** cluster and in GitHub Actions. Infra‑level tests live in `datum-infra/tests`. **IAM/Auth Provider** tests reside in `auth-provider-openfga/test/iam`; the **test‑e2e** GitHub Action job builds the image, spins up a Kind cluster, deploys via Flux, and executes Chainsaw scenarios. |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Frontend** | Cypress tests in `cloud-portal/cypress`, executed locally (interactive) and in CI (headless).                                                                                                                                                                                                                                                                                    |
| **CI/CD**    | GitHub Actions for every PR; Flux deploys to **staging** ➜ **prod** clusters.                                                                                                                                                                                                                                                                                                    |

---

## Gaps

1. No automated performance or load tests.
2. No Lighthouse audits for UI performance & accessibility.
3. Staging cluster isn’t validated after each deploy.
4. IAM/Auth Provider tests lack performance metrics and post‑deploy smoke checks.

---

## Action Plan (Incremental)

1. **Document & Scripts**\
   • Add `make kind-test` (backend) and `npm run test:e2e` (frontend).\
   • Update READMEs with quick‑start commands.
2. **Lighthouse Audits**\
   • Add Lighthouse CI GitHub Action in portal repo (URLs: `/`, `/dashboard`).\
   • Store HTML reports; warn if score < 90.
3. **Backend Load Tests**\
   • Create `tests/perf/` with k6 scripts (e.g., `create_networks.js`).\
   • Add manual workflow `perf.yml` to run against Kind or staging.
4. **Staging Smoke Tests**\
   • Nightly GitHub Action runs minimal Cypress suite against staging.\
   • Send Slack alert on failure.
5. **Optional Nova AI Pilot**\
   • Enable on portal for top 3 user flows.\
   • Evaluate flake rate & coverage before gating merges.
6. **Continuous Improvement**\
   • Expand test cases with new features.\
   • Review flaky tests weekly; tighten perf budgets after optimizations.

---

## Local Quick Start

```bash
# Backend controllers
go test ./test/e2e -v        # quick run
make kind-test               # full Kind cluster run

# Cloud portal UI
yarn install                 # or npm ci
npm run cypress:open         # interactive
yarn cypress:run             # headless

# Performance (optional)
k6 run tests/perf/create_networks.js
```

---

**Keep this doc in sync** as new tests, tools, or workflows are added.

