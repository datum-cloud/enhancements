---
status: provisional
stage: alpha
latest-milestone: "v0.1"
---

# Headless Tunnel CLI

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Execution Modes](#execution-modes)
  - [Architecture: Go Plugin + Rust Binary](#architecture-go-plugin--rust-binary)
  - [Auth Integration](#auth-integration)
  - [Long-Running Token Refresh](#long-running-token-refresh)
  - [Command Surface](#command-surface)
  - [Background Daemon Mode](#background-daemon-mode)
  - [System Service Installation](#system-service-installation)
  - [State Isolation](#state-isolation)
  - [Exit Codes](#exit-codes)
  - [Required Library Changes](#required-library-changes)
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
- [Infrastructure Needed](#infrastructure-needed)

## Summary

Today, the only supported way to run a Datum tunnel is through the Datum
desktop application, which requires a display environment. This enhancement
adds a headless command-line experience for tunnels, distributed as a
[datumctl][datumctl] plugin (`datumctl connect tunnel ...`). The plugin
supports three execution modes against the same underlying tunnel runtime:

1. **Foreground** — blocks until interrupted, logs to stdout/stderr.
2. **Background daemon** — detached process, PID file, log file, managed with
   `tunnel ps` / `tunnel stop` / `tunnel logs`.
3. **System service** — registered with systemd (Linux), launchd (macOS), or
   Windows Service Manager so tunnels survive reboot and run unattended.

For unattended modes, authentication is provided by Datum Cloud service
accounts (`datumctl login --credentials key.json --session <name>`), which
datumctl already supports. The plugin invokes datumctl's credentials helper
for both initial token acquisition and long-running token refresh, so a single
installed tunnel can run for days or years without manual intervention.

The plugin wraps the existing `datum-connect` Rust binary, which holds the
iroh data plane and the K8s control-plane integration. The Go plugin layer
handles process lifecycle, configuration persistence, and service-manager
integration; the Rust binary handles the tunnel itself.

[datumctl]: https://github.com/datum-cloud/datumctl

## Motivation

The Datum desktop application requires a display environment. That excludes
a large set of legitimate Datum use cases:

- Headless servers (cloud VMs, on-prem infrastructure)
- CI/CD pipelines
- Container workloads
- SSH-only remote machines
- Automated infrastructure

Operators who want a persistent tunnel on a server today have no supported
path. They either run the GUI app in a degraded mode, cobble together
workarounds, or do not use Datum at all. The system-service capability
directly addresses the operational gap where tunnels need to survive reboots
and run unattended — the same problem Docker and Podman solve for container
workloads with `systemctl`.

Without a headless CLI and service model, Datum is positioned as a developer
convenience tool rather than as production infrastructure.

### Goals

- Users can invoke `datumctl connect tunnel ...` from any terminal to manage
  tunnels with no display environment.
- The CLI exposes every tunnel configuration option the desktop UI exposes
  today (label, endpoint, enabled state).
- Users can run a tunnel in the foreground (blocking, logs to stdout/stderr)
  or detach it as a background daemon.
- Users can install a named tunnel as a persistent system service that
  auto-starts at boot, using the OS-native service manager (systemd on Linux,
  launchd on macOS, Windows Service Manager on Windows).
- Service lifecycle management commands are supported: `install`,
  `uninstall`, `start`, `stop`, `status`.
- The CLI exits with standard POSIX exit codes so it composes cleanly with
  shell scripts and process supervisors.
- Long-running tunnels refresh credentials automatically and indefinitely.
- A single named tunnel can run for days, weeks, or years without manual
  re-auth.

### Non-Goals

- This does not replace or deprecate the existing desktop GUI. The GUI
  remains the primary UX for interactive desktop users.
- No new tunnel types or networking capabilities are introduced. The CLI
  exposes existing functionality only.
- Container-native packaging (an official OCI image with an entrypoint) is
  out of scope for this enhancement, though the headless CLI makes it
  straightforward to build one separately. See
  [Implementation History](#implementation-history) for a follow-up note on
  container distribution.
- Cross-platform service installation is not required to ship simultaneously.
  Linux/systemd ships first; macOS/launchd and Windows SCM follow in
  subsequent milestones.
- Remote management of tunnels (controlling a daemon over a network socket
  or API) is out of scope.
- Identity material baked directly into a per-user binary download
  ("Identity Packet") is out of scope for the MVP; see
  [Alternatives](#alternatives).

## Proposal

A new datumctl plugin, `datumctl-connect`, ships as a release of this
repository and is registered in
[datum-cloud/datumctl-plugins](https://github.com/datum-cloud/datumctl-plugins).
The plugin binary bundles the existing `datum-connect` Rust binary in the
same release tarball. The Go plugin handles command-line surface, process
lifecycle, and service-manager integration; the Rust binary runs the actual
tunnel.

The Rust binary is refactored to support a "plugin mode" in which it
receives its bearer token via env var and renews it by invoking datumctl's
credentials helper — never opening a browser, never running OIDC discovery,
never writing OAuth state to disk. The datumctl credentials helper supports
both interactive sessions and Datum Cloud service-account sessions
transparently, so the same code path serves long-running unattended tunnels.

### User Stories

#### Story 1: Developer runs a quick tunnel from a terminal

> As a developer, I want to expose my local dev server to the Datum network
> from my terminal, without launching a desktop app.

```sh
datumctl connect tunnel listen --label my-dev --endpoint 127.0.0.1:3000
```

The tunnel blocks the terminal, prints the assigned hostname, and tears down
cleanly on Ctrl+C.

#### Story 2: Operator runs a background tunnel on a workstation

> As an operator, I want to start a tunnel that survives me closing my
> terminal, but doesn't require root or a service installation.

```sh
datumctl connect tunnel listen --detach --name dev-app \
  --label dev-app --endpoint 127.0.0.1:3000
datumctl connect tunnel ps
datumctl connect tunnel logs --name dev-app --follow
datumctl connect tunnel stop --name dev-app
```

#### Story 3: Operator installs a persistent production tunnel on a server

> As an operator, I want a tunnel on my production server to start at boot,
> recover from crashes, and authenticate without anyone being logged in.

```sh
# One-time setup using a service-account credential file.
datumctl login --credentials ./prod-tunnel-sa.json --session prod-tunnel

# Install as a system-scoped service, owned by the service account.
sudo datumctl connect tunnel install \
  --name prod-app \
  --label prod-app \
  --endpoint 127.0.0.1:8080 \
  --project proj-prod \
  --session prod-tunnel \
  --system

sudo datumctl connect tunnel start --name prod-app --system
```

The tunnel now starts on boot, refreshes credentials automatically by
re-minting the service-account JWT, and is supervised by systemd (or
launchd / Windows SCM).

#### Story 4: CI pipeline runs a tunnel for the duration of a job

> As a CI author, I want my pipeline to expose a service to Datum for the
> duration of a test job, then tear it down.

```sh
datumctl login --credentials ${{ secrets.DATUM_SA }} --session ci
datumctl connect tunnel listen --detach --name ci-job \
  --label ci-job --endpoint 127.0.0.1:8080 --session ci
# ... run integration tests against the assigned hostname ...
datumctl connect tunnel stop --name ci-job
```

#### Story 5 (Future): Container / sidecar deployment

> As a platform engineer, I want to drop a Datum tunnel into a container or
> as a Kubernetes sidecar so my app gets a public hostname with no extra
> infrastructure.

This is a non-goal for this enhancement (per issue #698 non-goals and the
discussion thread), but the architecture supports it: a container image can
bundle datumctl + the plugin + the Rust binary, mount a service-account
credentials file, and run `datumctl connect tunnel run` as the entrypoint.
We may publish an official image in a follow-up enhancement.

### Notes/Constraints/Caveats

- **Auth is owned by datumctl.** The headless tunnel never runs its own
  OAuth flow, never opens a browser, and never writes OAuth state to disk.
  This is a deliberate architectural constraint: it prevents auth-state
  drift between the headless CLI and the rest of datumctl, and it makes the
  service-account integration trivial (since datumctl already implements
  it).
- **System-scoped service installation requires service-account auth.**
  Interactive OAuth refresh tokens depend on a logged-in user session. A
  service that may start before any user logs in cannot rely on that. The
  plugin refuses to `install --system` against an interactive session.
- **Tunnel configuration parity is small.** The desktop UI exposes three
  user-facing options today: label, endpoint, and enabled state. All other
  tunnel concerns (traffic protection, HTTPS redirects, connector
  registration, discovery mode) are handled automatically by `TunnelService`.
  The headless CLI matches this surface exactly.
- **Cross-platform service installation phases.** Linux/systemd ships with
  the v0.1 release. macOS/launchd and Windows SCM follow once the systemd
  path is stable in production. The same Go API (`kardianos/service`)
  abstracts all three, so the platform-specific code is a single backend
  swap, not a redesign.

### Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Token refresh fails (helper missing, service account revoked) | Refresh runs on a schedule with exponential backoff; failures emit clear stderr warnings; `tunnel status` calls out the failure mode with a remediation hint. |
| Heartbeat lease expires while the host sleeps | On wake, the binary detects lease loss and re-enables the tunnel rather than exiting. |
| datumctl binary moves after install (e.g., `brew upgrade`) | Installed services capture the absolute path at install time; if it stops resolving, `tunnel status` surfaces the failure clearly and the user re-installs. |
| Long-running unattended interactive session expires | The plugin warns at detach/install time when the session is interactive; `install` outright refuses (exit 78) for system-scoped services. |
| Concurrent unnamed `listen` invocations collide on `listen_key` | Each named tunnel has its own subdirectory and key; the unnamed slot serializes via file lock. |
| `kardianos/service` platform quirks | Each platform phase requires hands-on validation against a real service manager; integration tests use stubs for unit coverage, real systems for end-to-end. |
| Rust binary size (~50–100 MB) inflates the plugin tarball | LTO, strip, exclude `ui/` dependencies; document the tarball size in release notes; consider a slim binary in a future iteration. |
| Cross-compiling Rust for Windows arm64 (tier-2 target) | Verify in CI before publishing the windows/arm64 manifest entry; omit from the first release if the build is unstable. |

## Design Details

### Execution Modes

```
                          datumctl connect tunnel <subcommand>
                                       │
            ┌──────────────────────────┼──────────────────────────┐
            │                          │                          │
       Foreground                Background daemon          System service
        (listen)                 (listen --detach)         (install/start)
            │                          │                          │
            └──────────────────────────┴──────────────────────────┘
                                       │
                          Rust binary in plugin mode:
                          datum-connect tunnel listen ...
                              ┌──────────────────────┐
                              │ ExternalTokenSource  │ ← exec
                              │ + refresh loop       │   $DATUM_CREDENTIALS_HELPER
                              ├──────────────────────┤   auth get-token
                              │ ProjectControlPlane  │   --session $DATUM_SESSION
                              │ Client (kube)        │
                              ├──────────────────────┤
                              │ TunnelService CRUD   │
                              ├──────────────────────┤
                              │ HeartbeatAgent       │
                              ├──────────────────────┤
                              │ ListenNode (iroh)    │
                              └──────────────────────┘
```

All three modes converge on the same Rust subprocess. Only the Go plugin's
invocation surface differs:

- **Foreground**: `exec`, stream stdio, forward signals, exit with child's
  code.
- **Detach**: double-fork (unix) or detached spawn (Windows), write a PID
  file, redirect stdio to a log file, return once the tunnel is ready.
- **Service**: `kardianos/service` writes a platform unit pointing at
  `datumctl-connect tunnel run --name <name>`, which loads persisted config
  and execs the Rust binary identically.

### Architecture: Go Plugin + Rust Binary

The Go plugin (`datumctl-connect`) and the Rust binary (`datum-connect`)
ship in the same release tarball. The Go plugin is the user-facing surface;
it locates the Rust binary either next to itself or on `PATH`.

Responsibilities split:

| Concern | Owner |
|---|---|
| Plugin manifest, cobra command surface | Go |
| Reading `DATUM_*` context env vars | Go |
| Acquiring initial bearer token via datumctl helper | Go |
| Output format mapping (`table`/`json`/`yaml`) | Go (table passthrough; yaml conversion) |
| Process lifecycle (foreground / detach / service) | Go |
| PID files, log files, log rotation | Go |
| Service-manager integration (`kardianos/service`) | Go |
| iroh `ListenNode` (data plane) | Rust |
| K8s control-plane client (`kube-rs`) | Rust |
| `TunnelService` CRUD | Rust |
| `HeartbeatAgent` (lease renewal) | Rust |
| Token refresh loop (exec helper) | Rust |
| JSON output of tunnel state | Rust |

### Auth Integration

The Rust binary in plugin mode receives a fresh bearer token from the Go
plugin via env (`DATUM_ACCESS_TOKEN`) at startup and renews it by execing
`$DATUM_CREDENTIALS_HELPER auth get-token --session $DATUM_SESSION` on a
refresh schedule. The credentials helper is datumctl itself; the plugin
captures its absolute path at install time.

datumctl supports two session types and both flow through this helper
transparently:

- **Interactive sessions** (`datumctl login`): OAuth2 + refresh tokens in
  the OS keyring.
- **Service-account sessions** (`datumctl login --credentials key.json
  --session <name>`): RS256 JWT minted from a private key, exchanged via
  JWT-bearer grant. No refresh token; re-minted on each refresh.

For the plugin and the Rust binary, both look identical:
`datumctl auth get-token --session <name>` returns a fresh access token.
This is the unification point that makes detached daemons and service
installations work — the same code path serves an interactive developer
session and an unattended production server.

| Mode | Required session type |
|---|---|
| Foreground `listen` / `list` / `update` / `delete` | interactive **or** service-account |
| `listen --detach` | interactive **or** service-account (warning if interactive) |
| `install` (user scope) | service-account (recommended); warning if interactive |
| `install --system` | service-account (required; exit 78 otherwise) |

### Long-Running Token Refresh

The Rust binary holds an `ExternalTokenSource` that:

1. Stores the current bearer token and its expiry (parsed from the JWT
   `exp` claim).
2. Runs a background task that refreshes when the token is within 5 minutes
   of expiry.
3. Refreshes opportunistically on 401 responses from the control plane.
4. On refresh failure, retries with exponential backoff (1s → 60s capped).
5. Notifies `ProjectControlPlaneClient::rebuild_if_changed` so the kube
   client picks up the new bearer.

A service-installed tunnel can run indefinitely. The service account's JWT
exchange has no inherent expiry beyond the lifetime of the account itself.

### Command Surface

**Tunnel CRUD** (forwarded to Rust):

```
datumctl connect tunnel list
datumctl connect tunnel listen [--name N] --label L --endpoint E [--yes]
datumctl connect tunnel update --id X [--label Y] [--endpoint Z]
datumctl connect tunnel delete --id X
```

**Daemon process management** (Go-only):

```
datumctl connect tunnel listen --detach --name N ...
datumctl connect tunnel ps
datumctl connect tunnel stop --name N
datumctl connect tunnel logs --name N [--follow]
datumctl connect tunnel status --name N
```

**Service installation** (Go-only, per-OS backend):

```
datumctl connect tunnel install --name N --label L --endpoint E \
  --session S [--project P] [--system]
datumctl connect tunnel uninstall --name N [--system]
datumctl connect tunnel start --name N [--system]
datumctl connect tunnel stop --name N [--system]
datumctl connect tunnel status --name N
datumctl connect tunnel run --name N      # internal; invoked by service unit
```

**Output formatting**:

| `--output` value | Behavior |
|---|---|
| `table` (default) | Human-readable table from Rust |
| `json` | JSON from Rust (`--json` forwarded) |
| `yaml` | Rust emits JSON; Go converts to YAML |

### Background Daemon Mode

`listen --detach --name N` daemonizes the same Rust subprocess:

- **Unix**: double-fork from the Go plugin; the grandchild calls `setsid`,
  closes stdin, redirects stdout/stderr to a log file, chdir's into the
  plugin runtime dir, writes its PID, and execs the Rust binary.
- **Windows**: spawn the Rust binary with
  `CREATE_NEW_PROCESS_GROUP | DETACHED_PROCESS | CREATE_NO_WINDOW`. Logs
  redirected via `cmd.Stdout`/`cmd.Stderr`. Job-object semantics isolate the
  daemon from the parent's signal handling.

The parent Go process waits up to 10 seconds for the tunnel to reach
ready, prints the assigned hostname, and exits 0. If readiness takes
longer, the parent exits 0 with a hint to run `tunnel status`.

Log files rotate at 10 MB with 5 retained files. Inside the log file,
Rust traces are JSON-line-formatted for greppable structured logs.

### System Service Installation

`install` writes a persisted YAML config and registers a platform service
unit via [`github.com/kardianos/service`][kardianos], which abstracts
systemd / launchd / Windows SCM behind a single API.

[kardianos]: https://github.com/kardianos/service

**Persisted config** (`<config-dir>/services/<name>.yaml`):

```yaml
name: prod-app
label: prod-app
endpoint: 127.0.0.1:8080
project: proj-prod
session: prod-tunnel
org: org-prod
api_host: https://api.datum.net
created_at: 2026-06-05T12:00:00Z
created_by: ops@example.com
```

**User vs. system scope**:

- **User-scoped** (default): systemd `--user`, launchd `LaunchAgent`,
  Windows interactive-user service. No elevated privileges.
- **System-scoped** (`--system`): systemd system unit, launchd
  `LaunchDaemon`, Windows SCM service. Starts at boot regardless of user
  login. Requires root / Administrator.

**Service unit** (rendered by `kardianos/service`):

- Name: `datumctl-connect-<scope>-<name>`
- Executable: absolute path to `datumctl-connect` captured at install time
- Arguments: `["tunnel", "run", "--name", "<name>"]` (plus `--system` for
  system-scoped)
- Restart policy: restart on non-zero exit with 5s/30s/60s backoff; clean
  shutdown (exit 0) does not restart

**Install validations** (all must pass; failure exits non-zero with a
remediation hint and leaves no state):

1. `--session` resolves to an active session in datumctl's keyring.
2. The session's credential type is `service_account` for `--system`
   (exit 78 otherwise).
3. The session can mint a fresh token now (helper smoke test).
4. The service account has permission to create tunnels in the target
   project (control-plane dry-run).
5. `--system` paired with root / Administrator.
6. No existing service with the same name.

**Uninstall** stops the service, removes the unit, and deletes the
persisted YAML. By default it does **not** delete the tunnel from the
Datum control plane (opt-in via `--purge`).

### State Isolation

The plugin uses a separate state directory from the standalone
`datum-connect` invocations and from the desktop app:

| Platform | User-scope state dir | System-scope state dir |
|---|---|---|
| Linux | `$XDG_DATA_HOME/datumctl/connect/` | `/var/lib/datumctl/connect/` |
| macOS | `~/Library/Application Support/datumctl/connect/` | `/Library/Application Support/datumctl/connect/` |
| Windows | `%LOCALAPPDATA%\datumctl\connect\` | `%PROGRAMDATA%\datumctl\connect\` |

Each named tunnel has its own subdirectory (`<repo>/tunnels/<name>/`)
holding its own `listen_key` and proxy state. The plugin sets
`DATUM_CONNECT_REPO` before exec'ing the Rust binary; the Rust binary uses
that env var to scope its `Repo` open.

No OAuth files are ever written here. Auth state lives in datumctl's
keyring; the plugin reads tokens through datumctl, not from disk.

### Exit Codes

POSIX-conformant so the CLI composes with shell scripts and process
supervisors. Both the Go plugin and the Rust binary use the same conventions
and the Go plugin propagates the child's code verbatim.

| Code | Meaning |
|---|---|
| 0 | Success (including clean signal-driven shutdown) |
| 1 | Generic runtime error |
| 2 | Misuse: invalid flags, missing required args, conflicting options |
| 64 | Usage error (per `sysexits.h`): semantically rejected (e.g., name conflict) |
| 65 | Data error: malformed config file or control-plane response |
| 69 | Service unavailable: control plane unreachable, network error |
| 75 | Temporary failure: tunnel exists but is still provisioning |
| 77 | Permission denied: not authenticated, or `--system` requires root |
| 78 | Configuration error: missing service-account session for `install` |

### Required Library Changes

Implementation requires modest, additive changes to the `datum-connect`
codebase:

1. **`app/lib/src/external_token.rs`** (new) — `ExternalTokenSource`
   struct that holds the current bearer, schedules refresh, execs the
   credentials helper, and notifies subscribers on rotation.

2. **`ProjectControlPlaneClient::new_with_token_source(...)`** — new
   constructor that wires the kube client to an `ExternalTokenSource`,
   reusing the existing `rebuild_if_changed` rotation path.

3. **`TunnelService::with_pcp_client(...)` and
   `HeartbeatAgent::with_pcp_client(...)`** — bypass `DatumCloudClient` so
   plugin mode never instantiates `AuthClient` or OIDC.

4. **`ApiEnv`** — honor `DATUM_API_HOST` in plugin mode (today it only
   reads `DATUM_API_ENV`).

5. **`cli/src/main.rs`** — detect plugin mode via `DATUM_ACCESS_TOKEN`,
   route to a separate `tunnel` branch that constructs the token source,
   builds `ProjectControlPlaneClient` directly, treats `--project` as a
   transient override (no `set_selected_context`), and constructs
   `ListenNode` only for the `listen` subcommand (not `list`/`update`/
   `delete`).

6. **`--json` flag** on tunnel subcommands; JSON output for
   `list`/`listen`/`update`/`delete`.

7. **Signal handling on Windows** — listen for `CTRL_BREAK_EVENT` in
   addition to `CTRL_C_EVENT`.

8. **Tracing layer for daemonized mode** — JSON-line tracing layer
   selectable via env var so daemon logs are greppable.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

#### How can this feature be enabled / disabled in a live cluster?

- [x] Other
  - **Describe the mechanism**: The headless tunnel CLI is an opt-in
    datumctl plugin. Users install it via the existing plugin install flow
    (`datumctl plugin install connect`). Disabling means uninstalling the
    plugin and stopping any installed services. No cluster-side flag is
    involved.
  - **Will enabling/disabling require control plane downtime?** No. The
    plugin uses existing control-plane APIs (`TunnelService` →
    `ConnectorAdvertisement` / `HTTPProxy` resources).
  - **Will enabling/disabling require node downtime or reprovisioning?**
    No. It's a client-side tool.

#### Does enabling the feature change any default behavior?

No. The plugin is opt-in. The desktop app remains unchanged. The
standalone `datum-connect` binary continues to function for users who
prefer it. Datum Cloud users who never install the plugin see no change.

#### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Users `uninstall` any installed services, stop any running tunnels,
and remove the plugin (`datumctl plugin uninstall connect`). Existing
tunnels in the Datum control plane are unaffected by plugin removal
unless `uninstall --purge` was used.

#### What happens if we reenable the feature if it was previously rolled back?

Re-installing the plugin restores the CLI surface. Persisted service
configs (if not removed) are picked up on the next `tunnel ps` / `tunnel
status`. Installed system services that survived the uninstall continue
to run (they reference the plugin binary by absolute path captured at
install time; if the path is restored, they keep working; if not, they
fail cleanly and the user re-installs).

#### Are there any tests for feature enablement/disablement?

- Plugin install / uninstall is covered by datumctl's existing plugin
  tests.
- This enhancement adds integration tests that install the plugin into a
  test environment, run each subcommand against a mocked control plane,
  uninstall, and verify no leftover state.

### Rollout, Upgrade and Rollback Planning

#### How can a rollout or rollback fail? Can it impact already running workloads?

The plugin is end-user software, not a cluster-side service. A failed
rollout to one user does not impact any other user. Installed services on
a given host continue to run with the previous plugin version (they
exec'd the Rust binary at the path captured at install time); an upgrade
that changes that path requires a re-install.

#### What specific metrics should inform a rollback?

- User-reported failures in the headless tunnel mode (auth refresh
  failures, service install errors, signal handling regressions).
- Control-plane error rates on `TunnelService` CRUD operations
  attributable to the headless client (identifiable via user agent;
  see [Monitoring Requirements](#monitoring-requirements)).

#### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Plugin upgrades follow datumctl's existing plugin lifecycle. Service
units capture an absolute path; downgrading the plugin without
re-installing the service results in the service running the old binary
(which may be removed or replaced). The release notes will direct users
to `uninstall` and re-`install` named services when upgrading across
breaking changes.

#### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No deprecations. The plan adds new functionality alongside existing
desktop and standalone-CLI paths.

### Monitoring Requirements

#### How can an operator determine if the feature is in use by workloads?

The Rust binary sends a distinctive User-Agent header
(`datum-connect/<version> datumctl-plugin/<version>`) on all
control-plane requests. Datum Cloud operators can count requests with
that User-Agent to gauge adoption.

#### How can someone using this feature know that it is working for their instance?

- [x] Other
  - **Details**: `datumctl connect tunnel status --name N` reports
    runtime state (installed/running/PID/uptime), auth state (session,
    last refresh, token expiry), control-plane state (resource ID,
    accepted/programmed flags, hostnames), and recent log entries.
    `datumctl connect tunnel logs --name N --follow` tails the structured
    log file.

#### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

- 99% of `tunnel list` invocations complete in under 2 seconds (network
  latency + control-plane round trip).
- 99% of `tunnel listen` invocations reach ready state in under 30
  seconds.
- 99.9% of token refreshes succeed on first attempt over a 24-hour
  window.
- An installed tunnel survives a credential rotation (re-mint cycle)
  without observable downtime > 5 seconds.

These are client-side SLOs measured by end-to-end tests against staging,
not control-plane SLOs.

#### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [x] Other
  - **Details**: Per-tunnel structured log lines emit timing for control-
    plane round trips, refresh outcomes, and heartbeat results. The
    plugin does not expose a Prometheus endpoint (it's a CLI). For
    Datum-side monitoring, the User-Agent header lets operators count
    requests, errors, and latencies attributable to headless-CLI usage.

#### Are there any missing metrics that would be useful to have to improve observability of this feature?

A future improvement could expose a local HTTP endpoint
(`http://localhost:<port>/metrics`) on a daemonized tunnel for
Prometheus scraping. Out of scope for v0.1.

### Dependencies

#### Does this feature depend on any specific services running in the cluster?

- **Datum Cloud control plane** (`api.datum.net` or staging equivalent)
  - **Usage description**: All tunnel CRUD operations and heartbeat
    leases.
  - **Impact of outage**: New tunnels cannot be created; existing
    tunnels lose their lease eventually and are reaped server-side.
  - **Impact of degraded performance**: Slower `listen` setup; refresh
    operations may time out and retry.
- **datumctl on the local host**
  - **Usage description**: The credentials helper invoked for token
    acquisition and refresh.
  - **Impact of outage**: The plugin cannot acquire or refresh tokens.
    Currently-running tunnels keep working until their current token
    expires; then they enter backoff and emit warnings.
- **Platform service manager (systemd / launchd / SCM)** for installed
  tunnels
  - **Usage description**: Process supervision and boot-time startup.
  - **Impact of outage**: Installed tunnels can't be started until the
    service manager is healthy.

### Scalability

Not applicable in the conventional cluster-side sense. The plugin is a
per-user, per-host CLI. Some specific notes:

#### Will enabling / using this feature result in any new API calls?

No new API surface. The plugin uses existing `TunnelService` /
`ConnectorAdvertisement` / `HTTPProxy` CRUD that the desktop app already
exercises. One new request pattern: long-running heartbeat at the
existing cadence (already implemented in `HeartbeatAgent`).

#### Will enabling / using this feature result in introducing new API types?

No.

#### Will enabling / using this feature result in any new calls to the cloud provider?

No.

#### Will enabling / using this feature result in increasing size or count of the existing API objects?

No new schema. Per-tunnel object count is the same as today (one
tunnel == one `ConnectorAdvertisement` + one `HTTPProxy`).

#### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No.

#### Will enabling / using this feature result in non-negligible increase of resource usage in any components?

On the user's host: a daemonized tunnel runs one Rust process holding
an iroh endpoint, the heartbeat loop, and the kube client. Memory is
~30–50 MB per tunnel. CPU is negligible at rest (event-driven).

On the control plane: no different from existing tunnel usage.

#### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

A user with many named tunnels accumulates one PID file, one log file,
and one repo subdirectory per tunnel. Log files cap at ~50 MB each
(10 MB × 5 retained). No reasonable usage hits node limits.

### Troubleshooting

#### How does this feature react if the API server is unavailable?

- `tunnel list` / `update` / `delete` fail with exit 69 (Service
  unavailable). The user retries when the API returns.
- A running `tunnel listen` continues to serve traffic (iroh data plane
  is independent of the control plane) but its heartbeat starts failing.
  After a few heartbeat windows, the control plane reaps the lease; on
  reconnect, the binary detects this and re-enables the tunnel.

#### What are other known failure modes?

- **Credentials helper missing or not on the recorded path**
  - **Detection**: Refresh attempts fail with `exec: file not found`;
    the Rust binary logs `credentials helper not found at <path>` to
    stderr / log file.
  - **Mitigations**: Re-install datumctl at the original path, or
    `uninstall` + `install` the named tunnel to capture the new path.
  - **Diagnostics**: `tunnel status --name N` surfaces this state.
  - **Testing**: Integration test that points the helper at a
    non-existent path and asserts the warning + status surfaces.
- **Service account revoked**
  - **Detection**: Refresh attempts return non-zero exit from the
    helper; logs include the helper's stderr.
  - **Mitigations**: Re-register a service account session under the
    same `--session` name.
  - **Diagnostics**: `tunnel status --name N` shows the auth section
    with `last_refresh: failed (revoked)`.
- **Host sleep > lease TTL**
  - **Detection**: On wake, heartbeat fails; the Rust binary detects
    the lease is gone and re-enables.
  - **Mitigations**: Automatic; no user action needed.
  - **Diagnostics**: Log line `lease lost during sleep; re-enabling`.
- **`listen_key` collision with standalone `datum-connect`**
  - **Detection**: iroh endpoint reports identity conflict.
  - **Mitigations**: The plugin uses a distinct repo path
    (`DATUM_CONNECT_REPO`) so this should not happen; if the user has
    set `DATUM_CONNECT_REPO` to the standalone path, they get a clear
    error directing them to unset it.
- **Service stuck in restart loop**
  - **Detection**: Service manager backs off after N restarts;
    `tunnel status` shows `Running: no` and the last exit reason.
  - **Mitigations**: User investigates via `tunnel logs` (or the
    platform service-manager logs surfaced in `status`) and either fixes
    the underlying cause or `uninstall`s the service.

#### What steps should be taken if SLOs are not being met to determine the problem?

1. `datumctl connect tunnel status --name N` — runtime + auth +
   control-plane state.
2. `datumctl connect tunnel logs --name N --tail 200` — recent
   structured log entries.
3. Verify datumctl auth: `datumctl auth get-token --session <name>` —
   should print a fresh token.
4. Verify control-plane reachability: `datumctl connect tunnel list`.
5. For installed services: `systemctl --user status
   datumctl-connect-<name>` (or launchd / SCM equivalent).

## Implementation History

- 2026-06-05: Initial draft submitted (this enhancement).

Planned milestones:

- v0.1 (Q3 2026): Foreground + detach + Linux/systemd service installation.
- v0.2 (Q4 2026): macOS/launchd service backend.
- v0.3 (Q4 2026): Windows Service Manager backend.
- Future: Container image distribution (separate enhancement, anticipated
  per the issue thread discussion).

## Drawbacks

- **Binary size.** The Rust binary is ~50–100 MB. Bundled with the Go
  plugin, the release tarball is large compared to other datumctl plugins.
  This is acceptable for a one-time install but worth noting.
- **Maintenance surface.** Three platform service-manager backends
  (systemd / launchd / SCM) means three places to debug platform-specific
  quirks. The `kardianos/service` abstraction helps but doesn't eliminate
  the cost.
- **Service-account onboarding step.** System-scoped installation requires
  the user to first generate a service account in Datum Cloud and download
  a credentials file. This is an extra step compared to "just log in." The
  step is unavoidable for boot-time unattended auth, but it's friction.
- **Dependence on datumctl for refresh.** A tunnel installed today is
  permanently tied to the datumctl binary at the recorded path. Moving or
  removing datumctl breaks installed tunnels. A future improvement could
  make the Rust binary natively service-account-aware to remove this
  dependency.

## Alternatives

### Alt 1: Standalone `headless-tunnel` binary (no datumctl plugin)

Build a single self-contained Rust binary that handles auth, control
plane, and data plane independently. This is what richardhenwood proposed
in the issue thread.

**Pros**: Trivial to drop into a Dockerfile, no datumctl dependency,
clean 12-factor surface.

**Cons**: Duplicates datumctl's auth surface (OIDC, keyring, service
account JWT minting). Long-term that's two implementations to keep in
sync. Also leaves desktop-app users + headless-CLI users with two
disjoint auth states.

**Why rejected for MVP**: We have an existing datumctl auth implementation
and a working credentials helper protocol. Reusing it is much cheaper than
re-implementing it, and the integration is the cleanest way to keep auth
state singular across all user experiences.

**When to revisit**: If container/sidecar usage becomes the dominant mode
and the plugin overhead is a real adoption blocker, fork the Rust binary
into a slim standalone variant that does the JWT mint itself. The
underlying `lib/` crate already separates auth from tunnel; the
refactoring would be additive.

### Alt 2: Pure-Go reimplementation

Re-implement the tunnel (iroh + kube client) in Go, eliminating the Rust
subprocess.

**Pros**: Single language, smaller binary, simpler distribution.

**Cons**: iroh's Go bindings are immature compared to the Rust crate. The
entire `lib/` crate would need a Go port. Multi-year project.

**Why rejected**: Out of scope. The existing Rust binary is production-ready.

### Alt 3: socket-based control of a running daemon

Run a long-lived `datum-connectd` daemon; control it via a local Unix
socket / named pipe. CLI invocations are clients.

**Pros**: Mid-tunnel reconfiguration possible without restart.

**Cons**: Significant complexity (IPC, daemon lifecycle, version skew
between CLI and daemon). The use cases listed in the issue don't require
this.

**Why rejected**: Over-engineered for the stated goals.

### Alt 4: Identity-Packet-baked-into-binary distribution

Bake per-user identity into a per-user binary download (richardhenwood's
"Identity Packet" idea, deferred by privateip).

**Pros**: Truly zero-config for end users.

**Cons**: Significant infrastructure (per-user signed binary build
pipeline). Hard to debug.

**Why rejected for MVP**: Both issue participants agreed this is a
post-MVP follow-up.

## Infrastructure Needed

- **Release pipeline** in this repository:
  - GitHub Actions matrix builds for linux/{amd64,arm64},
    darwin/{amd64,arm64}, windows/{amd64,arm64}.
  - Cross-compilation tooling for Rust (`cargo-zigbuild` for Linux;
    native runners for darwin/windows; verify aarch64-pc-windows-msvc
    as a tier-2 target).
  - Tarball + zip packaging that bundles both binaries plus LICENSE.
  - SHA256 sum generation for the plugin manifest.

- **Plugin registry entry** in
  [datum-cloud/datumctl-plugins](https://github.com/datum-cloud/datumctl-plugins):
  - A `plugins/connect.yaml` manifest.
  - `index.yaml` updated to include the new plugin.

- **Staging-credentials secret** for the E2E test in CI (the existing
  pattern used by other datumctl-adjacent tests).
