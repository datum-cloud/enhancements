---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# VPC Plugin for datumctl

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Command tree](#command-tree)
  - [Create](#create)
  - [Get](#get)
  - [Describe](#describe)
  - [Delete](#delete)
  - [Output formats](#output-formats)
  - [Error handling and exit codes](#error-handling-and-exit-codes)
  - [Plugin manifest](#plugin-manifest)
  - [API contract](#api-contract)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed](#infrastructure-needed)

## Summary

This enhancement introduces a `datumctl` plugin that lets users manage Virtual
Private Clouds (VPCs) from the command line. The plugin exposes a
`kubectl`-inspired command tree — `datumctl vpc create`, `get`, `describe`, and
`delete` — providing a consistent mental model for users already familiar with
Kubernetes CLI conventions. Users can provision a VPC with a platform-assigned
ULA `/48`, inspect existing VPCs in tabular or structured output, drill into the
full detail of a single VPC including region attachments and address allocations,
and tear down VPCs they no longer need.

The plugin is a standalone binary (`datumctl-vpc`) that uses the `datumctl`
plugin SDK, inherits the user's organization and project context, and
communicates with the Datum Cloud API using the credentials helper for
token acquisition.

## Motivation

VPCs are the foundational networking primitive in Datum Cloud. Every compute
workload, DNS zone, and connectivity service attaches to a VPC. Without a CLI
plugin, users must manage VPCs exclusively through the web console or raw API
calls — neither of which integrates into automation pipelines, CI/CD workflows,
or terminal-first development habits.

A `kubectl`-modeled command tree reduces the learning curve for the primary
audience (platform engineers and developers) and establishes a pattern that
future resource plugins (subnets, instances, DNS zones) can follow.

### Goals

- **Let users create VPCs from the command line** with a platform-assigned ULA
  `/48` and a human-readable name.
- **Let users list VPCs** in a concise tabular format suitable for scripting,
  with filtering by organization or project.
- **Let users inspect a single VPC** in detail, including address allocations,
  region attachments, and status.
- **Let users delete VPCs** with confirmation prompts and cascading behavior
  that is clear before execution.
- **Provide consistent output formats** (table, JSON, YAML) across all commands,
  driven by the `--output` flag wired by `plugin.NewRootCmd`.
- **Follow the `kubectl` mental model** for action names, flag conventions, and
  error messages so that users familiar with Kubernetes CLI patterns feel at
  home.

### Non-Goals

- **Custom IPv6 address space (GUA).** Users bringing their own GUA `/48` block
  is a future capability. The initial release only supports platform-assigned
  ULA.
- **IPv4 support.** IPv4 addressing for VPCs is a future capability. The initial
  release is IPv6-only.
- **Managing VPC subnets or endpoints.** Subnet and endpoint lifecycle is a
  separate concern and belongs in a future `datumctl vpc subnet` or
  `datumctl compute` plugin.
- **VPC peering, Private Link, or Galactic VPC services.** These are advanced
  connectivity features covered by the Galactic VPC enhancement and belong in
  dedicated commands.
- **Defining the VPC API itself.** This plugin consumes an existing (or
  forthcoming) Datum Cloud API for VPC lifecycle. The API contract is a
  dependency, not a deliverable of this enhancement.
- **Interactive VPC wizard or guided setup.** The plugin is designed for
  explicit, scriptable commands, not interactive onboarding.
- **Defining the implementation rollout or sequencing.** This document describes
  the intended product experience; engineering sequencing is out of scope.

## Proposal

Introduce `datumctl-vpc`, a standalone Go binary distributed as a `datumctl`
plugin, that implements four top-level actions under the `vpc` command:

```
datumctl vpc create [name] [flags]
datumctl vpc get [flags]
datumctl vpc describe [name-or-id]
datumctl vpc delete [name-or-id] [flags]
```

The plugin uses the `datumctl` plugin SDK (`go.datum.net/datumctl/plugin`) for
manifest serving, root command wiring, context passthrough, and token
acquisition. Subcommands are standard Cobra commands.

### User Stories

#### Story 1: A developer provisions a VPC for a new project

Maya is scaffolding a new project and needs an isolated network. She creates a
VPC with a single command, letting the platform assign a ULA `/48`:

```console
$ datumctl vpc create maya-project
VPC "maya-project" created (id: vpc_a1b2c3d4, ipv6: fd20:0a1b::/48)
```

She gets back a stable identifier and the assigned address block, which she can
pipe into her provisioning scripts.

#### Story 2: A platform engineer audits VPCs across their organization

James needs to see what VPCs exist in his organization. He lists them in table
format, scoped to his current project:

```console
$ datumctl vpc get
NAME            ID            IPV6              AGE
maya-project    vpc_a1b2c3d4  fd20:0a1b::/48    2d
staging-net     vpc_e5f6a7b8  fd20:0c3d::/48    5d
```

He switches to JSON output to feed the list into a compliance script:

```console
$ datumctl vpc get -o json
```

#### Story 3: A user investigates a VPC's region attachments

Sam needs to understand which regions a VPC is active in and what address space
it holds. He describes the VPC by name:

```console
$ datumctl vpc describe maya-project
Name:           maya-project
ID:             vpc_a1b2c3d4
Organization:   acme
Project:        production
Status:         active
IPv6:           fd20:0a1b::/48 (ULA)
Created:        2026-08-05T14:32:00Z

Regions:
  NAME          IPV6_SUBNET         STATUS
  us-east-1     fd20:0a1b:0001::/64  active
  eu-west-1     fd20:0a1b:0002::/64  active
```

#### Story 4: A user tears down a staging VPC

Dana is done with a staging environment and wants to remove its VPC. The delete
command prompts for confirmation, showing what will be affected:

```console
$ datumctl vpc delete staging-net

  This will delete VPC "staging-net" (vpc_e5f6a7b8).
  All region attachments and address allocations will be released.

  Delete this VPC? [y/N]
```

In a CI pipeline, she skips the prompt:

```console
$ datumctl vpc delete staging-net --force
VPC "staging-net" deleted
```

### Notes/Constraints/Caveats

- **Built-in shadowing.** The plugin command is `vpc`, which must not conflict
  with any built-in `datumctl` command. If a built-in `vpc` command is added in
  the future, it takes precedence over the plugin.
- **Organization and project scoping.** The plugin inherits `DATUM_ORG` and
  `DATUM_PROJECT` from the `datumctl` context. Users can override with
  `--org` and `--project` flags (wired by `plugin.NewRootCmd`). The `get`
  command may list across projects within an organization when scoped
  appropriately.
- **Token freshness.** The plugin calls `plugin.Token()` immediately before
  each API request. Tokens are short-lived and must not be cached across
  requests.
- **VPC deletion is destructive.** The `delete` command requires explicit
  confirmation (`--force` to bypass) and should surface dependent resources
  before proceeding.
- **The VPC API is a dependency.** This plugin assumes the existence of a
  Datum Cloud REST API supporting VPC CRUD operations. The exact endpoints
  and request/response shapes are defined by that API, not by this plugin.

### Risks and Mitigations

#### Risk: Users delete VPCs with active workloads

A VPC deletion that cascades to running instances or active subnets could cause
unexpected outages.

*Mitigations:* The `delete` command requires confirmation by default
(`--force` to bypass). Before proceeding, the API should return any dependent
resources, and the plugin surfaces them in the confirmation prompt. The API
itself may reject deletion if active workloads exist (a server-side safety
net).

#### Risk: Inconsistent behavior across output formats

JSON/YAML output may leak internal fields or differ in structure from table
output.

*Mitigations:* The plugin uses a single internal struct per resource and
serializes it to the requested format. Table output is a curated projection;
structured output (`-o json`, `-o yaml`) emits the full API response envelope.

## Design Details

### Command tree

```
datumctl vpc
├── create [name] [flags]
├── get [flags]
├── describe [name-or-id]
└── delete [name-or-id] [flags]
```

All commands inherit `--org`, `--project`, `--output`, and `--session` from
`plugin.NewRootCmd`.

### Create

```
datumctl vpc create <name> [flags]
```

Provisions a new VPC with the given name. The name must be unique within the
target project.

**Flags:**

| Flag | Default | Description |
|------|---------|-------------|
| `--label` | — | Key-value label to attach (repeatable, e.g. `--label env=staging`) |
| `--output` | `text` | Output format: `text`, `json`, `yaml` |

**Behavior:**

- Sends a `POST` request to the VPC creation endpoint.
- On success, prints a confirmation line with the assigned VPC ID and IPv6 CIDR
  in text mode, or the full resource object in JSON/YAML mode.
- On failure (e.g., name collision), prints the API error message and exits
  with code 1.

**Examples:**

```console
# Create with default ULA IPv6
$ datumctl vpc create my-vpc

# Create with labels
$ datumctl vpc create my-vpc --label env=staging --label team=platform

# Create and emit JSON
$ datumctl vpc create my-vpc -o json
```

### Get

```
datumctl vpc get [flags]
```

Lists VPCs in the current project (or organization, if project-scoped listing is
not supported by the API).

**Flags:**

| Flag | Default | Description |
|------|---------|-------------|
| `--output` | `table` | Output format: `table`, `json`, `yaml` |
| `--label` | — | Filter by label key-value (e.g., `--label env=staging`) |

**Table output columns:**

```
NAME            ID            IPV6              LABELS         AGE
maya-project    vpc_a1b2c3d4  fd20:0a1b::/48    env=staging    2d
```

**Behavior:**

- Sends a `GET` request to the VPC list endpoint.
- In table mode, displays a curated set of columns (name, ID, IPv6 CIDR,
  labels, age). The `AGE` column uses relative time (e.g., `2d`, `5h`, `3m`),
  matching `kubectl` convention.
- In JSON/YAML mode, emits the full array of VPC objects as returned by the API.
- Exit code 0 on success, even if the list is empty.

**Examples:**

```console
# List all VPCs in current project
$ datumctl vpc get

# List as JSON
$ datumctl vpc get -o json

# Filter by label
$ datumctl vpc get --label env=staging
```

### Describe

```
datumctl vpc describe <name-or-id>
```

Displays detailed information about a single VPC, identified by name or ID.

**Arguments:**

| Argument | Description |
|----------|-------------|
| `name-or-id` | VPC name (human-readable slug) or VPC ID (e.g., `vpc_a1b2c3d4`) |

**Behavior:**

- Sends a `GET` request to the VPC detail endpoint.
- Renders a multi-section, human-readable display including:
  - **Identity** — name, ID, organization, project, status, labels
  - **Addressing** — IPv6 CIDR and type (ULA)
  - **Regions** — table of region attachments with subnet allocations and status
  - **Timestamps** — created, last modified
- Falls back to JSON/YAML output if `--output` flag is set.

**Output (text mode):**

```console
$ datumctl vpc describe maya-project
Name:           maya-project
ID:             vpc_a1b2c3d4
Organization:   acme
Project:        production
Status:         active
Labels:         env=staging, team=platform
IPv6:           fd20:0a1b::/48 (ULA)
Created:        2026-08-05T14:32:00Z
Modified:       2026-08-06T09:15:00Z

Regions:
  NAME          IPV6_SUBNET         STATUS
  us-east-1     fd20:0a1b:0001::/64  active
  eu-west-1     fd20:0a1b:0002::/64  active
```

**Examples:**

```console
# Describe by name
$ datumctl vpc describe maya-project

# Describe by ID
$ datumctl vpc describe vpc_a1b2c3d4

# Describe as JSON
$ datumctl vpc describe maya-project -o json
```

### Delete

```
datumctl vpc delete <name-or-id> [flags]
```

Deletes a VPC and releases its address allocations.

**Arguments:**

| Argument | Description |
|----------|-------------|
| `name-or-id` | VPC name or ID to delete |

**Flags:**

| Flag | Default | Description |
|------|---------|-------------|
| `--force` | `false` | Skip confirmation prompt |

**Behavior:**

- Resolves the VPC by name or ID.
- Without `--force`, prints a confirmation prompt listing the VPC and its
  region attachments, then waits for `y`/`yes` (case-insensitive). Any other
  input aborts the operation with exit code 1.
- Sends a `DELETE` request to the VPC endpoint.
- On success, prints a confirmation line.
- On failure (e.g., VPC not found, active workloads preventing deletion), prints
  the error and exits with code 1.

**Examples:**

```console
# Delete with confirmation
$ datumctl vpc delete maya-project

# Delete without confirmation (CI/CD)
$ datumctl vpc delete maya-project --force
```

### Output formats

All commands support the `--output` (short: `-o`) flag wired by
`plugin.NewRootCmd`:

| Format | Flag | Description |
|--------|------|-------------|
| `text` | `-o text` | Human-readable confirmation line (create, delete) |
| `table` | `-o table` | Tabular listing (get default) |
| `json` | `-o json` | Structured JSON, machine-parseable |
| `yaml` | `-o yaml` | Structured YAML, machine-parseable |

The `describe` command uses a custom multi-section text layout by default and
falls back to JSON/YAML when `-o json` or `-o yaml` is specified.

### Error handling and exit codes

| Exit code | Meaning |
|-----------|---------|
| 0 | Success |
| 1 | General error (API failure, validation error, not found, confirmation denied) |

Error messages are printed to stderr and follow the pattern:

```
Error: <human-readable message>
```

When the API returns a structured error (e.g., `422 Unprocessable Entity` with
field-level validation details), the plugin surfaces the API's error message
directly rather than rephrasing it.

### Plugin manifest

The plugin serves the following manifest in response to `--plugin-manifest`:

```json
{
  "name": "vpc",
  "version": "v0.1.0",
  "description": "Manage Virtual Private Clouds (VPCs) — create, list, describe, and delete",
  "min_datumctl_version": "v0.10.0",
  "api_version": 1,
  "min_api_version": 1
}
```

### API contract

The plugin consumes the following Datum Cloud API endpoints (subject to the
actual API specification):

| Action | Method | Path | Description |
|--------|--------|------|-------------|
| Create | `POST` | `/v1/organizations/{org}/projects/{project}/vpcs` | Provision a new VPC |
| List | `GET` | `/v1/organizations/{org}/projects/{project}/vpcs` | List VPCs in a project |
| Get | `GET` | `/v1/organizations/{org}/projects/{project}/vpcs/{id}` | Retrieve VPC detail |
| Delete | `DELETE` | `/v1/organizations/{org}/projects/{project}/vpcs/{id}` | Delete a VPC |

The plugin constructs the API host from `DATUM_API_HOST`, the organization from
`DATUM_ORG` (or `--org`), and the project from `DATUM_PROJECT` (or
`--project`). Authentication is via Bearer token obtained from
`plugin.Token()`.

## Production Readiness Review Questionnaire

This is a **client-side `datumctl` plugin**. It introduces no Datum
control-plane components, API types, or server-side behavior. The questionnaire
below is adapted for a CLI tool.

### Feature Enablement and Rollback

#### How can this feature be enabled / disabled?

The plugin is installed by the user via `datumctl plugin install vpc` (from the
curated index) or `datumctl plugin install owner/repo`. Uninstalling the plugin
(`datumctl plugin uninstall vpc`) removes the command entirely. No feature gate
or server-side flag is involved.

#### Does enabling the feature change any default behavior?

No. The plugin adds a new subcommand tree under `datumctl vpc`; it does not
modify existing commands or default behavior.

#### Can the feature be disabled once enabled?

Yes — uninstalling the plugin removes it. No residual state is left on the
client.

### Rollout, Upgrade and Rollback Planning

#### How can a rollout or rollback fail?

Plugin distribution relies on the curated index or GitHub Releases. A broken
release (wrong binary name, missing platform) would fail at install time.
Checksums in the index prevent silent corruption.

#### What specific metrics should inform a rollback?

Install failure rate reported through the plugin index telemetry (if available)
and user-reported issues.

### Monitoring Requirements

As a client-side tool, the plugin does not expose server-side metrics.
Observability is provided through:

- **Structured error messages** — API errors are surfaced verbatim, allowing
  users to diagnose failures.
- **Exit codes** — Scriptable success/failure signaling for CI/CD integration.

### Dependencies

#### External dependencies

- **Datum Cloud API** — The VPC CRUD endpoints must be available and stable. An
  API outage or breaking change would cause plugin requests to fail.
- **`datumctl` host binary** — The plugin requires `datumctl` v0.10.0 or later
  for context passthrough and credentials helper support.

### Scalability

The plugin makes at most one API call per invocation. There is no polling,
watching, or background process. Scalability is bounded by the API's rate limits
and the user's invocation pattern.

### Troubleshooting

#### How does the plugin react if the API is unavailable?

The plugin prints the HTTP error (e.g., connection timeout, 5xx status) and
exits with code 1. No retry logic is implemented in the plugin; users or
orchestrators are responsible for retry semantics.

#### What are known failure modes?

- **Authentication failure** — expired or invalid token. The credentials helper
  returns an error; the plugin surfaces it.
- **VPC not found** — `describe` or `delete` with an invalid name or ID returns
  404. The plugin prints `Error: VPC "<name>" not found`.
- **Deletion blocked by active workloads** — The API returns 422 or 409. The
  plugin surfaces the API's error message.
- **Name collision on create** — The API returns 409. The plugin surfaces the
  conflict message.

## Implementation History

- **2026-08-07** — Enhancement document created, status `provisional`.

## Drawbacks

- **Another binary to maintain.** The plugin adds a release surface (build, test,
  distribute) that the team must maintain alongside the core `datumctl` binary
  and other plugins.
- **API coupling.** The plugin is tightly coupled to the VPC API contract. A
  breaking API change requires a coordinated plugin update.
- **kubectl convention may not fit all users.** While the `kubectl` mental model
  is familiar to the primary audience, users coming from other cloud providers
  (e.g., `aws ec2 create-vpc`) may find the action names (`create` vs
  `create-vpc`) less intuitive initially.

## Alternatives

- **Built-in `datumctl` command.** Instead of a plugin, the VPC commands could
  be implemented as a built-in subcommand tree. This eliminates the plugin
  distribution surface but increases the core binary's scope and release
  coupling. The plugin model was chosen to keep the core lean and allow
  independent versioning.
- **Verb-noun syntax (`vpc-create`, `vpc-get`).** Mirrors AWS CLI convention
  (`aws ec2 create-vpc`). This was rejected in favor of the `kubectl`-style
  `vpc create` because the target audience is more familiar with Kubernetes
  patterns and the noun-verb structure scales better as subcommands are added
  (e.g., `vpc subnet create`, `vpc peer create`).
- **JSON-only API wrapper.** A minimal plugin that simply forwards arguments to
  the API and emits raw JSON. This was rejected because users need a
  human-readable default experience (table listings, describe output,
  confirmation prompts) alongside structured output for scripting.

## Infrastructure Needed

- **Datum Cloud VPC API.** The CRUD endpoints described in the [API
  contract](#api-contract) must be available before the plugin can be
  implemented.
- **Curated plugin index entry.** A catalog entry describing the plugin, its
  versions, and per-platform download artifacts must be added to Datum's curated
  plugin index for `datumctl plugin install vpc` to work.
- **Go release pipeline.** A `goreleaser` configuration (or equivalent) to build
  `datumctl-vpc` binaries for all supported platforms, produce checksums, and
  publish GitHub Releases.
