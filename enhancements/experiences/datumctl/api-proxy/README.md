---
status: implementable
stage: alpha
latest-milestone: "v0.x"
---

# A Local Front Door to the Datum Cloud API

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [The command](#the-command)
  - [What the local address serves](#what-the-local-address-serves)
  - [Live results arrive in real time](#live-results-arrive-in-real-time)
  - [Credentials and errors](#credentials-and-errors)
  - [Security model](#security-model)
  - [The request log](#the-request-log)
  - [Starting and stopping](#starting-and-stopping)
  - [Prior art](#prior-art)
  - [How we know it works](#how-we-know-it-works)
  - [The first release](#the-first-release)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed](#infrastructure-needed)

## Summary

`datumctl api proxy` gives every tool on your machine an authenticated
front door to the Datum Cloud API. It starts a small local server that
forwards each request it receives to the API, signed in as you.
Credentials are attached automatically and kept fresh in the background,
so the tools behind the proxy never see a token, never store one, and
never break when one expires:

```console
$ datumctl api proxy --port 8001
$ curl http://127.0.0.1:8001/apis/resourcemanager.miloapis.com/v1alpha1/organizations
```

By default the local address is a faithful stand-in for the real API: any
request that works against `https://api.datum.net` works against
`http://127.0.0.1:8001` unchanged — including live *watch* requests, whose
updates arrive the instant the platform sends them. Alternatively, point
the proxy at a single project or organization and URLs get much shorter:

```console
$ datumctl api proxy --port 8001 --project my-project
$ curl http://127.0.0.1:8001/apis/networking.datumapis.com/v1alpha/dnszones
```

datumctl already knows who you are, which environment you use, and how to
keep your credentials current. This feature puts all of that behind one
local address, in the spirit of `kubectl proxy` and `cloud-sql-proxy`.

## Motivation

Today, a local tool that calls the Datum Cloud API has to manage
credentials itself. In practice that means copying a token out of
`datumctl auth get-token` into an environment variable or a config file —
and copying it again when it expires, because tokens are short-lived by
design. Every team building against the platform locally rediscovers this
friction:

1. **Local development of apps built on the platform.** A web app or
   service under development needs platform credentials on every request
   its dev server makes. With the proxy, a developer points the app's API
   base URL at `http://127.0.0.1:8001` once and datumctl owns
   authentication for the whole workday — no tokens in `.env` files, no
   mystery "unauthorized" failures an hour into work.
2. **Live-updating clients.** Dashboards, controllers, and registries
   keep a *watch* open so changes appear the moment they happen. Watches
   deliver results piece by piece over a long-lived request; a proxy that
   held results back until the request ended would silently break them.
   **Delivering live results in real time is a hard requirement**, not an
   optimization.
3. **End-to-end test suites.** A harness starts a proxy, reads the
   address it prints, and points every test at it. No test process ever
   holds a credential, and tearing down auth for the whole suite means
   stopping one process.
4. **Scripting and exploration.** `curl` or a notebook against a local
   address beats copy-pasting tokens that expire mid-investigation.

None of these are hypothetical — Datum's own cloud portal, whose dev
server calls the platform on every request and keeps live watches open,
is simply the first consumer. The friction is the same for anything
built against the platform.

### Goals

- One command that gives any local tool an authenticated view of the
  Datum Cloud API for the user's session.
- Credentials handled entirely by datumctl — attached per request,
  refreshed before they expire, machine accounts included — with
  refreshed credentials saved back to the session exactly as every other
  command does.
- Live results (watches and other streaming responses) delivered to the
  local client the moment the platform sends them.
- A conservative security posture: reachable only from the local machine,
  protected against malicious websites, credentials never exposed to the
  tools behind the proxy and never written to logs.
- A predictable lifecycle: the session is pinned when the proxy starts,
  the address is printed in a machine-readable way, and failures say what
  to do next.

### Non-Goals

- **Exposing the proxy beyond the local machine.** There is no option to
  do so (see [Security model](#security-model)). Users who want remote
  access can run their own tunnel and own that decision.
- **Being an API gateway.** No caching, no rewriting of responses, no
  rate limiting — the proxy adds credentials and otherwise stays out of
  the way.
- **Serving raw tokens.** `datumctl auth get-token` exists for that, and
  handing a token to the program that asked is a deliberately narrower
  door than serving it to anything that can reach a local port.
- **Guessing scope from the current context.** Short URLs come from the
  explicit `--project`/`--organization` mode, never from silently reading
  the active context (see
  [What the local address serves](#what-the-local-address-serves)).
- **A one-shot request command** (in the style of `gh api`). A natural
  future sibling under the same `api` command group, but a separate
  enhancement.

## Proposal

Add a new `api` command group with a single subcommand, `proxy`. There is
no top-level `datumctl proxy` shortcut: the `api` group earns its keep by
leaving room for a one-shot `api request` sibling, and a shortcut is cheap
to add later if usage shows the extra word hurts.

When the command starts it:

1. Picks a session — the active one by default, or one named with
   `--session` — and from it the API environment to talk to, resolved the
   same way every other datumctl command resolves it.
2. Decides what its root address means: the whole API by default, or a
   single project/organization when a scope flag is given.
3. Prepares the same self-refreshing credentials the rest of the CLI uses.
4. Starts listening on the local machine (a random free port unless
   `--port` says otherwise), prints a human-readable banner and a
   machine-readable URL, and serves until interrupted.

From then on every request is forwarded with fresh credentials attached,
and every response flows back in real time.

### User Stories

#### Story 1: An app developer stops thinking about tokens

Maya builds a web app on Datum Cloud. Today her dev server needs a
platform token injected per session, and when it expires she restarts
things and mutters. Now her dev script assumes a proxy:

```console
$ datumctl api proxy --port 8001
  Session:    maya@datum.net (api.datum.net)
  Upstream:   https://api.datum.net
  Listening:  http://127.0.0.1:8001
```

She sets `API_URL=http://127.0.0.1:8001` once in `.env.local`. The app
sends ordinary requests; datumctl owns auth for the whole workday. If her
session ever genuinely needs a re-login, the proxy says exactly that — in
its log and in the error the app receives — and `datumctl login` fixes
it without restarting anything.

#### Story 2: A dashboard watches for changes through the proxy

A team dashboard keeps a watch open on a project so changes show up the
moment they happen. Through the proxy it uses exactly the URL it uses in
production — only the host changes — and each update arrives as the
platform sends it. When the platform eventually ends the watch (they are
periodically recycled by design), the dashboard's normal reconnect opens
a new one, which automatically carries freshly refreshed credentials.
Nobody wrote any auth code.

If the client prefers a dedicated base URL, a proxy started with
`--project my-project` serves that project at its root and the watch URL
shrinks to `/apis/networking.datumapis.com/v1alpha/dnszones?watch=true`.

#### Story 3: An E2E suite gets auth for free

An E2E harness starts `datumctl api proxy` with no `--port`, so it gets a
random free port and can never collide with another suite:

```console
$ datumctl api proxy --quiet --project e2e-project
http://127.0.0.1:52713
```

The harness reads that first line as both its readiness signal and the
address, and hands it to every test worker. No test process ever holds a
credential; teardown is stopping one process.

#### Story 4: Pinning a non-active session

Sam is logged into both production and staging and keeps a proxy running
against staging while their *active* session stays on production:

```console
$ datumctl auth list
$ datumctl api proxy --session sam@datum.net@api.staging.env.datum.net --port 8002
```

Running `datumctl auth switch` in another terminal does not silently
repoint Sam's staging proxy — the session is pinned at startup.

### Notes/Constraints/Caveats

- **The proxy pins its session at startup.** A proxy that followed the
  active session would change identity *and* environment under a running
  dev server. The proxy states its session in the banner and serves that
  identity until it exits; users who want a different session restart it.
- **The default mode is a faithful stand-in for the real API.** Anything
  recorded against the real API — docs examples, captured traffic, SDK
  output — replays against the proxy unchanged. This is what makes
  `API_URL` swapping work.
- **Scope is explicit and pinned, never inherited from the current
  context.** Rationale in
  [What the local address serves](#what-the-local-address-serves).
- **A new built-in `api` command shadows any user plugin named `api`.**
  No known plugin uses the name; worth a release-note line.
- **Responses are never rewritten.** If some platform response ever
  embedded a full API URL in its body, clients would see the real
  endpoint's address — acceptable and documented, not silently patched.

### Risks and Mitigations

#### Risk: Anything on the machine can act as the user while the proxy runs

The proxy deliberately does not ask local programs to authenticate —
requiring a local password would just recreate the credential plumbing
this command exists to remove. That is the same trust boundary as
`kubectl proxy` and `cloud-sql-proxy`, but it means any local program
(or another user on a shared machine) can use the proxy while it runs.

*Mitigations:* the proxy is reachable only from the local machine and
only exists while the user runs it; its banner names the identity it
serves; every request is logged by default, so misuse is visible; and the
credential itself is never exposed — a local program can act *through*
the proxy but cannot take the token with it, which is strictly safer than
the token-in-environment practice it replaces. A follow-up will add a
socket-based mode that only the same OS user can reach; it is first on
the deferred list as the designated answer for shared machines.

#### Risk: Malicious websites probing the local proxy

A hostile web page can make a visitor's browser send requests to local
addresses, and known tricks exist for smuggling responses back.

*Mitigations:* the proxy uses both standard defenses. It refuses any
request that does not address it by its own local name, which defeats the
smuggling tricks; and it never opts into cross-site sharing, so browsers
block web pages from reading its responses. Local web development is
unaffected — it is the app's *dev server* that talks to the proxy, not
the browser. Browser-direct use would need an explicit opt-in flag,
deferred.

#### Risk: Buffering silently breaks watch clients

An innocent-looking default — collecting a response before relaying it,
or a response time limit — would make watches appear to hang, and only
under real use.

*Mitigations:* real-time delivery is a stated hard requirement with a
dedicated test designed to fail if results are held back even slightly
(see [How we know it works](#how-we-know-it-works)), and nothing in the
proxy limits how long a response may last.

#### Risk: A dead session causing a flood of sign-in attempts

A polling dev server can send many requests per second; if the session's
refresh credential has been revoked, each request could trigger a fresh
sign-in attempt against the auth service.

*Mitigations:* refresh attempts are shared — concurrent requests wait on
one attempt rather than starting their own — and after a failure the
proxy waits a few seconds before trying again, answering requests in the
meantime with the same actionable error. However hard the local client
polls, the auth service sees at most one attempt per cooldown.

## Design Details

### The command

```console
$ datumctl api proxy --help
Start a local proxy that authenticates requests to the Datum Cloud API.

The proxy listens on 127.0.0.1 and forwards every request to the API
endpoint of your datumctl session, adding your credentials automatically
and refreshing them as needed. Point any local tool at the printed URL —
no tokens to copy, no expiry to manage.

By default the proxy serves the full API endpoint, so requests use the
same paths as the real API. Pass --project or --organization to serve a
single control plane instead, with shorter paths.

The session and scope are pinned when the proxy starts. Switching your
active account or context does not affect a running proxy.

Usage:
  datumctl api proxy [flags]

Flags:
      --port int            Local port to listen on (default: a random free port)
      --session string      Pin a specific session by name (defaults to the
                            active session; see 'datumctl auth list')
      --project string      Serve this project's control plane as the proxy root
      --organization string Serve this organization's control plane as the proxy root
  -q, --quiet               Suppress per-request log lines
```

```console
  # Start a proxy on a fixed port for a dev server
  datumctl api proxy --port 8001

  # Start on a random free port; the URL is printed on the first stdout line
  datumctl api proxy

  # List organizations through the proxy
  curl http://127.0.0.1:8001/apis/resourcemanager.miloapis.com/v1alpha1/organizations

  # Serve one project directly, for shorter URLs
  datumctl api proxy --port 8001 --project my-project
  curl "http://127.0.0.1:8001/apis/networking.datumapis.com/v1alpha/dnszones?watch=true"

  # Pin a non-active session
  datumctl api proxy --session sam@datum.net@api.staging.env.datum.net
```

Three flag decisions worth stating:

- **`--port` defaults to a random free port.** A fixed default fails the
  moment a second proxy starts and trains tools to assume a well-known
  port. Random never fails at startup, and the chosen address is always
  printed, so nothing is guessing. Stable setups opt in with `--port`.
- **`--session` names a session** exactly as `datumctl auth list` shows
  it, matching the existing `--session` flag on `datumctl auth get-token`.
  The same email can be logged in to several environments, so the session
  name — not the email — is the unambiguous handle.
- **`--project`/`--organization`/`--platform-wide` are the same scope
  flags every datumctl command accepts**, read at launch. Unlike resource
  commands, the proxy does not fall back to the active context when no
  flag is given — see the next section.

### What the local address serves

The proxy picks its session and environment the way every other datumctl
command does — same session selection, same endpoint lookup, same
settings for staging environments, same credentials. It does that once,
at startup, and pins the result. Two guarantees follow: the proxy can
never disagree with the rest of the CLI about where a session points, and
if `datumctl get` works, the proxy works — there is nothing
proxy-specific to configure or debug.

The only real choice is what the *root* of the local address means.
Everything after the root is forwarded exactly as the client sent it.

**Default: the whole API.** Organizations, projects, and their individual
control planes are all reachable as paths under the one API endpoint, so
one unscoped proxy reaches everything — and every URL matches the real
API exactly.

**Scoped: one project or organization.** With
`--project`/`--organization`/`--platform-wide`, the proxy serves just
that scope at its root, behaving like a dedicated API server for it:
short paths, and a client library can be handed `http://127.0.0.1:PORT`
as its base URL with no path assembly at all.

**Scope is explicit, never inherited from the current context.** It is
tempting to default to the active context, the way resource commands do,
so URLs are short out of the box. Rejected, deliberately:

- *Matching production is the flagship requirement.* An app talks to the
  real API in production; swapping in the proxy with one base-URL change
  only works if the proxy accepts the very same URLs. A context-scoped
  default would make development URLs differ from production URLs.
- *The most obvious first request would break.* Listing organizations
  happens at the top of the API, not inside any project; a project-scoped
  default would turn the natural first `curl` into "not found."
- *Ambient state is poison for a proxy.* Tools cache the proxy URL. If
  its meaning depended on whichever context happened to be active at
  launch — or worse, followed `datumctl ctx use` live — the same URL
  would quietly mean different things on different days. Scope, like the
  session, is pinned at startup, stated on the command line, and shown in
  the banner.

If interactive users turn out to consistently expect context inheritance,
an explicit opt-in flag can add that convenience later without changing
what a bare invocation means. A mixed mode — one proxy serving short
paths for several scopes at once — is deferred, with a URL prefix (`/-/`)
reserved so adding it later breaks nothing.

### Live results arrive in real time

Watches and other streaming responses must flow through the proxy as if
it weren't there:

- **Every piece of a response is passed to the local client the moment it
  arrives.** The proxy never collects a response before relaying it. This
  single property is the entire streaming design; it is the same approach
  `kubectl proxy` uses to support watches.
- **Nothing limits how long a response may last.** Time limits apply only
  to *starting* a request (connecting, waiting for the first response);
  an open watch can run for hours. Watches are infinite on purpose.
- **Credentials are checked before a request is sent, never mid-response.**
  An open stream is never cut off just because the token that started it
  has since expired; when the platform itself ends a long watch (they are
  recycled by design), the client's normal reconnect gets fresh
  credentials automatically.

### Credentials and errors

- **Fresh before expiry.** Tokens are refreshed shortly before they
  expire, and the refreshed token is saved back to the session — so a
  long proxy run keeps the user's session fresh for every other datumctl
  command too. Machine accounts renew the same way they do everywhere
  else in datumctl; nothing is proxy-specific.
- **A dead session produces one clear answer.** When the session can no
  longer be refreshed — revoked, or expired for good — the proxy answers
  with an error of its own that names the problem and the fix: run
  `datumctl login`. That error is explicitly marked as coming from the
  proxy, and it deliberately does *not* imitate a platform "unauthorized"
  response: when a tool behind the proxy is told its request was
  rejected, that answer must genuinely come from the platform, or every
  client's re-auth logic becomes untrustworthy.
- **Platform rejections pass through untouched — and are never retried.**
  If the platform says no, the proxy relays exactly that, and does not
  refresh-and-retry on the client's behalf: the proxy cannot know a
  request is safe to replay, and silent retries would mask real problems.
  (`kubectl` makes the same choice.)
- **Failure is polite under load.** After a failed refresh, the proxy
  waits a few seconds before trying again and answers requests in the
  meantime with the same actionable error — fast, and without hammering
  the sign-in service.
- **Logging out doesn't strand the proxy.** After `datumctl logout` the
  proxy stays up and serves the actionable error instead of vanishing
  mid-workday; if the user logs back in to the same session, the proxy
  picks it up again without a restart.

### Security model

Conservative by default, with the reasoning stated so future changes are
deliberate:

- **Local machine only, no exceptions.** The proxy listens on the local
  address `127.0.0.1` and there is no flag to open it wider — not even a
  scary one — because every legitimate "remote proxy" story is better
  served by the user running their own tunnel, whose security model they
  already own. (It binds the IPv4 local address specifically: that is the
  address it prints, clients told `localhost` connect fine on their own,
  and doubling the setup for a failure nobody has observed isn't worth
  it. Revisit only if a real client cannot connect.) datumctl already
  takes this posture elsewhere: the browser login flow runs its own
  local-only listener.
- **Same-user trust, eyes open.** The proxy does not ask local programs
  to authenticate — see
  [Risks and Mitigations](#risks-and-mitigations) for why, what that
  exposes, and the socket-based hardening follow-up.
- **Web pages get nothing.** Both standard browser defenses are on and
  not configurable: requests that don't address the proxy by its local
  name are refused, and cross-site sharing is never enabled, so browsers
  block web pages from reading proxy responses. If browser-direct access
  is ever wanted, it arrives as an explicit opt-in flag — off by default,
  exact origins only.
- **Local clients cannot smuggle credentials.** Any credentials a local
  client attaches to its request are dropped before the session's own are
  added — a stale token baked into some tool's config is ignored rather
  than half-working, and nothing a client sends can impersonate anyone.
- **Tokens never appear in output.** The request log records method,
  path, status, timing, and size — never credentials. Anything
  token-shaped in a URL is redacted defensively, and the higher-verbosity
  debug output masks credentials the same way it does across the rest of
  datumctl.

### The request log

One line per request, on by default, silenced by `--quiet`:

```
10:42:03 GET  /apis/resourcemanager.miloapis.com/v1alpha1/organizations 200 143ms 8.1kB
10:42:05 GET  /apis/…/dnszones?watch=true 200 …streaming
```

Streaming responses log once when they start (marked `…streaming`) and
again when they end, with total duration and size — so an abruptly closed
watch is visible, not mysterious.

A failed credential refresh is the one event the user must act on, so it
is logged even under `--quiet` — once per cooldown, not once per request.
Log lines for *successful* refreshes, and per-request correlation IDs,
are deferred to a richer-diagnostics follow-up.

### Starting and stopping

- **The banner** (on the log side of the output) names the three things a
  user should be able to verify at a glance — who, where, and at what
  address:

  ```
    Session:    maya@datum.net (api.datum.net)
    Upstream:   https://api.datum.net
    Scope:      full endpoint (use --project/--organization to serve one control plane)
    Listening:  http://127.0.0.1:52347

    Press Ctrl+C to stop. Requests are logged below (silence with --quiet).
  ```

  In scoped mode the `Scope` line names the project or organization and
  `Upstream` shows its full address.

- **The address doubles as the readiness signal.** The bare URL is
  printed as the first and only line on standard output, after the proxy
  is actually accepting requests. Scripts and harnesses read one line and
  go — no port files, no retry loops.
- **Startup failures say what to do next:** not logged in → run
  `datumctl login`; unknown `--session` → check `datumctl auth list`;
  port already in use → pick another `--port` or omit the flag.
- **Ctrl+C stops cleanly.** In-flight ordinary requests get a moment to
  finish; open watches are then closed, which watch clients treat as a
  normal stream end. A second Ctrl+C exits immediately.
- **`datumctl auth switch` has no effect on a running proxy** — by
  design. The banner is the contract: the proxy serves the identity it
  printed until it exits.

### Prior art

| Tool | Take | Leave |
| --- | --- | --- |
| `kubectl proxy` | Real-time watch streaming; local-only default; refusing requests that misname the proxy | Fixed default port (collides); flags that loosen the local-only posture (documented incidents when used); path-filtering machinery Datum's API doesn't need |
| `cloud-sql-proxy` | Machine-readable readiness; pin the target at startup; background credential refresh as a first-class concern | Its tunnel is opaque — working at the request level lets us attach credentials, refuse bad requests, and log usefully |
| `gh api` | Proof that "the CLI owns auth, tools speak plain HTTP" is the right developer contract; the `api` command-group naming | It's one request per invocation — useless as a dev server's `API_URL`; a future `datumctl api request` can cover that case |
| aws-vault's credential server | The cautionary tale: it serves the *credential itself* over a local port, so anything local can steal it | Our proxy never serves the token — local programs can act through it, but can't take the credential with them |

### How we know it works

The whole suite runs against stand-in credentials and a stand-in API, so
it needs no real account and no network access.

The load-bearing test is the streaming one: the stand-in API sends a
single watch update and then refuses to produce the next one until the
test has seen the first arrive through the proxy. If the proxy held
results back even slightly, the test would deadlock and fail — real-time
delivery is proven by construction, with no timing luck involved. The
same test runs against a scoped proxy.

Around it, the suite pins every promise this document makes: credentials
are attached upstream and never revealed to local clients; whatever
credentials a client sends are dropped; requests that misname the proxy
are refused; platform rejections pass through unmarked while
proxy-generated errors are clearly marked and actionable; a session that
can't refresh produces at most one sign-in attempt per cooldown no matter
how hard clients poll; the printed address is usable the moment it
appears; Ctrl+C exits cleanly; session resolution matches the rest of the
CLI exactly; and a bare invocation ignores the active context even when
one is set.

Before release, the proxy is verified manually against staging with a
real watch left running past a token expiry.

### The first release

In scope:

- `datumctl api proxy` with `--port`, `--session`, `--quiet`, and
  launch-time scoping via `--project`/`--organization`/`--platform-wide`
- Faithful passthrough by default; scoped mode serving one control plane;
  real-time streaming
- Automatic credential attachment, refresh, and persistence; machine
  accounts; clearly-marked actionable errors when the session dies;
  refresh cooldown
- Local-only listener, browser defenses, client-credential stripping,
  credential-free request log
- Banner, machine-readable address line, graceful shutdown, pinned session
- The validation suite above and a user-facing docs page

Deferred, in likely priority order:

1. A socket-based listener only the same OS user can reach (the
   shared-machine hardening step; also nice for hermetic E2E)
2. Serving several scopes' short paths from one proxy (the reserved `/-/`
   prefix)
3. Opt-in browser access for browser-direct use cases
4. `datumctl api request` — a one-shot sibling in the style of `gh api`
5. A tested WebSocket guarantee, if a platform feature comes to need one
6. Richer diagnostics: per-request correlation IDs and successful-refresh
   log lines

## Production Readiness Review Questionnaire

### Feature enablement and rollback

- **How can this feature be enabled / disabled?** It is a new opt-in
  command; not running it disables it. It leaves no state behind after
  exit — the only persistent side effect (refreshed credentials saved to
  the session) is identical to running any other datumctl command.
- **Can the feature be rolled back?** Yes; removing the command removes
  the feature. Tools pointed at a stopped proxy fail the same way they
  would if it had simply never been started.

### Monitoring and supportability

- The request log (with streaming start/end markers) is the primary
  diagnostic; higher-verbosity debugging reuses datumctl's existing
  credential-masking output.
- Errors generated by the proxy are explicitly marked as such, so support
  can tell "your session is dead" from "the platform said no" from a
  single client screenshot.

### Dependencies

- No new third-party dependencies, services, or endpoints; the proxy is
  built entirely from what datumctl already ships and requires only the
  same local state every authenticated command already uses.

### Security

- See [Security model](#security-model): local machine only with no
  override, browser defenses on and not configurable, local programs
  trusted (documented, with a hardening follow-up), client credentials
  stripped, tokens never logged or served. Versus the status quo, a
  credential formerly exposed *as a value* (a token pasted into a dev
  server's environment) becomes an ability to act that exists only while
  the proxy runs — an intentional improvement.

## Implementation History

- 2026-07-11: Proposal drafted; implementation opened as
  [datum-cloud/datumctl#247](https://github.com/datum-cloud/datumctl/pull/247).

## Drawbacks

- **A standing capability on the local machine.** While the proxy runs,
  anything local can act as the user against the pinned environment. This
  is the price of removing credential plumbing, shared with every tool in
  the prior-art table; the mitigations and the hardening follow-up are
  covered above.
- **A long-running surface to support.** Streaming and shutdown behavior
  must be defended against regressions; the streaming test is the
  tripwire.
- **`api` becomes a reserved name**, shadowing any user plugin called
  `api`.
- **Faithful passthrough has edges:** a response that embedded the real
  API's address in its body (none known today) would show clients that
  address rather than the proxy's.

## Alternatives

- **Status quo: `datumctl auth get-token` plus an environment variable.**
  Works for one-shots; fails the dev-server case (the token expires
  mid-session and is exposed to the whole process tree) and the watch
  case (nothing refreshes credentials across reconnects).
- **The plugin credentials-helper protocol.** Right answer for datumctl
  plugins, but only available to processes datumctl itself launches — not
  an app's dev server, not `curl` — and requires integration code in
  every client that the proxy makes unnecessary.
- **A one-shot `datumctl api request <path>`.** Solves scripting, not the
  dev-server or watch cases; remains a good future sibling.
- **Teach each app to invoke datumctl itself.** Couples every consumer to
  CLI internals, one integration at a time; the proxy solves the whole
  class at once.
- **`auth update-kubeconfig` plus an external kubectl proxy.** For
  kubectl users only, and drags Kubernetes tooling into a workflow this
  product deliberately keeps Kubernetes-free.
- **Serve the token itself on a local port** (aws-vault's model).
  Strictly weaker: anything local could steal the credential and keep it
  after the proxy exits. Rejected on principle; `auth get-token` remains
  the only raw-token path, granted per invocation rather than to an open
  port.

## Infrastructure Needed

None. The feature is entirely client-side; no new services, endpoints, or
repositories.
