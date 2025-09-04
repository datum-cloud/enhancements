---
status: provisional
stage: alpha
latest-milestone: "v0.1"
---

# Project/Org Context & Generic CRUD (Discovery‑Driven)

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Resource Discovery & Compatibility](#resource-discovery--compatibility)
  - [Proposed Tool Surface](#proposed-tool-surface)
  - [Validation](#validation)
  - [Error Model](#error-model)
- [Phased Implementation](#phased-implementation)
- [Implementation History](#implementation-history)
- [References](#references)

## Summary

Extend `datumctl mcp` with (1) **project/org context switching** and (2) **generic CRUD** (create/read/update/delete)
for resources. The engine is **resource‑agnostic** and **discovery‑driven**, so new CRDs work without shipping a new CLI.
Initial examples below use **`HTTPProxy`** for clarity. Currently we are in **Phase 2**, building on **#255** (read‑only MCP MVP).
A subsequent **Phase 3** will add **safe change management** via `patch/plan/apply` and optional approvals.

## Motivation

With the read‑only MCP from **#255** shipped, the next step is to enable **context switching** and **CRUD** so IDE copilots
(e.g. Claude) can perform simple operations without leaving the prompt box.

### Goals

Add additional tools to datumctl mcp to enhance the developer experience.

- `datum/change_context(project, org)` sets the active context for the MCP session.
- Generic CRUD tools: `datum/create_resource`, `datum/get_resource`, `datum/update_resource`, `datum/delete_resource`.
- Keep the design **resource‑agnostic** and **discovery‑driven** in line with **#217**.

### Non-Goals

- Bulk operations, diff/patch planners, or cross‑project replication.
- Full AI UX for Datum (tracked separately).

## Proposal

Expose a small set of MCP tools. The session can carry a default **project/org** via
`datum/change_context`, while CRUD calls may also specify explicit `project`/`org` (which override session defaults).
The engine uses Kubernetes **API discovery** so newly added resource kinds are available automatically if RBAC permits.

### Resource Discovery & Compatibility

- **Resource‑agnostic engine**: CRUD relies on **API discovery** so the CLI does **not** require new releases for new CRDs.
  This aligns with **#217** (discovery‑driven `datumctl`, kubectl‑familiar UX).

### Proposed Tool Surface

> Tool names follow the MVP style (`datum/<verb>_<thing>`). Requests/Responses are JSON.

#### 1) `datum/change_context`

**Request**
```json
{ "project": "intro-project-3", "org": "datum" }
```
**Response**
```json
{ "ok": true, "sessionContext": {"project": "intro-project-3", "org": "datum"} }
```
**Errors**: `PROJECT_NOT_FOUND`, `ORG_NOT_FOUND`, `PERMISSION_DENIED`.

---

#### 2) `datum/create_resource`

**Example** (resource‑agnostic; using `HTTPProxy` for clarity)
```json
{
  "kind": "HTTPProxy",
  "project": "intro-project-3",
  "org": "datum",
  "name": "bittensor-dapp",
  "spec": { "rules": [ /* ... */ ] },
  "dryRun": true
}
```
**Dry‑run response**
```json
{ "ok": true, "dryRun": true, "validated": true }
```
**Applied response**
```json
{ "ok": true, "resourceVersion": "v1alpha/12345" }
```

---

#### 3) `datum/get_resource`
```json
{ "kind": "HTTPProxy", "project": "intro-project-3", "org": "datum", "name": "bittensor-dapp" }
```

---

#### 4) `datum/update_resource`
```json
{
  "kind": "HTTPProxy",
  "project": "intro-project-3",
  "org": "datum",
  "name": "bittensor-dapp",
  "spec": { /* full desired spec */ },
  "dryRun": false
}
```

---

#### 5) `datum/delete_resource`
```json
{ "kind": "HTTPProxy", "project": "intro-project-3", "org": "datum", "name": "bittensor-dapp", "dryRun": false }
```

---

#### Phase 3 (details TBD) additions (Safe Change Management — minimal surface)

- `datum/patch_resource` — JSONPatch/StrategicMerge/Server‑Side‑Apply with optional `expectedResourceVersion` and `dryRun`.
- `datum/plan_resource` — produce a stable `planId`/`planHash`, return diff and policy gate results (no writes).
- `datum/apply_plan` — apply only a previously reviewed plan; enforces `planHash` + TTL; optional human approval.

### Validation

- **Schema validation**: validate resources against CRD schemas before apply.
- **Dry‑run**: `dryRun=true` performs validation without persisting changes.
- **Explicit override**: CRUD requests may specify `project/org` to override session defaults.

### Error Model

```json
{ "ok": false, "error": { "code": "VALIDATION_ERROR", "message": "...", "details": {} } }
```
Canonical codes: `PROJECT_NOT_FOUND`, `ORG_NOT_FOUND`, `PERMISSION_DENIED`, `VALIDATION_ERROR`, `ALREADY_EXISTS`, `NOT_FOUND`, `CONFLICT`, `INTERNAL`.

## Phased Implementation

- **Phase 1 — Read‑Only MCP (shipped; #255)**  
  `list_crds`, `get_crd`, `validate_yaml`.

- **Phase 2 — Context + Generic CRUD**  
  `change_context`, `create_resource`, `get_resource`, `update_resource`, `delete_resource`; schema validation; dry‑run.

- **Phase 3 — Safe Change Management**  
  `patch_resource`, `plan_resource`, `apply_plan`; optional policy guardrails and approvals; optimistic concurrency on write paths.

## Implementation History

- 2025-09-04 — Added Phase 3 note and Phased Implementation section.
- 2025-09-03 — Phase 2 proposal drafted (this doc). Baseline: **#255** shipped read‑only MCP MVP.

## References

- Tracking discussion: **#267** — Discovery‑driven engine; MCP safety context.
- Baseline: **#255** — Introduce MCP server in `datumctl` (read‑only MVP).
- Related: **#217** — Enhance `datumctl` with a kubectl‑familiar, discovery‑driven UX.
- MVP PR: datumctl **PR #30**
