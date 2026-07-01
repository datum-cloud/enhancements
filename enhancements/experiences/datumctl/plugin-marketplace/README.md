---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

# Third-Party Plugin Indexes and Marketplaces for datumctl

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Registering a catalog](#registering-a-catalog)
  - [Installing with catalog-aware addressing](#installing-with-catalog-aware-addressing)
  - [Searching and browsing across catalogs](#searching-and-browsing-across-catalogs)
  - [Trust signals everywhere a plugin appears](#trust-signals-everywhere-a-plugin-appears)
  - [Publishing a catalog](#publishing-a-catalog)
  - [Enterprise and platform teams](#enterprise-and-platform-teams)
  - [Catalog manifest format](#catalog-manifest-format)
  - [Plugin binaries and platform-shared plugins](#plugin-binaries-and-platform-shared-plugins)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed](#infrastructure-needed)

## Summary

`datumctl` already lets users extend the CLI with plugins — standalone binaries
that add commands for Datum Cloud workflows — and ships a single, curated,
Datum-operated catalog of those plugins that users can `search` and `install`
from. Today that catalog is the *only* one a user can discover plugins through:
a third party who wants to publish a plugin must either get it merged into
Datum's repository or hand users a raw `owner/repo` string with no description,
no search, and no signal about whether it is safe.

Opening the plugin ecosystem changes that. It lets **anyone** — a community
author, an open-source project, or an enterprise platform team — publish their
own catalog of `datumctl` plugins, and lets users **register** those catalogs
and **browse, search, and install** from them through the same friendly
experience they already use for official plugins. A user can add a company's
internal catalog or a community catalog with one command, see its plugins
side-by-side with Datum's in search and in an interactive browser, and install
with confidence because every plugin carries a clear badge telling them where it
came from and whether Datum vouches for it.

The result is a marketplace experience that mirrors what developers expect from
modern tooling — `kubectl krew` custom indexes, GitHub CLI extensions, Homebrew
taps, Claude Code plugin marketplaces — while keeping Datum's curated catalog as
the trusted default and making the moment a user adopts a third-party catalog an
explicit, informed choice.

Datum Cloud is also built on the **milo-os** platform, and that shapes the
design: many of the services a user works with — and the CLI plugins that drive
them — originate as milo-os services. A milo-os service should be able to publish
a single plugin that installs into `datumctl`, and into any other milo-os-based
CLI, straight from a catalog — rather than shipping a separate host-branded binary
for each. To the user the command is simply `datumctl ipam`. The first concrete
case is the milo-os IPAM service.

## Motivation

Plugins are how `datumctl` grows beyond what Datum can build directly. The
connectivity platform touches many adjacent workflows — DNS tooling, network
automation, integrations with a team's own internal systems — and not all of
them belong in the core CLI or in a catalog Datum has to personally review and
host. The single-catalog model creates a structural bottleneck:

- **Community authors have no front door.** A developer who builds a useful
  `datumctl` plugin can publish a binary, but they cannot make it
  *discoverable*. Users won't find it via `datumctl plugin search`, won't see a
  description, and have no way to tell a real project from a typo-squat. The only
  path to discoverability is a pull request into Datum's repository — a review
  burden that does not scale and that Datum should not have to own for every
  third-party tool.
- **Enterprises have nowhere to put internal tooling.** A platform team that
  builds `datumctl` plugins wrapping their own conventions, internal services, or
  approval workflows cannot distribute them as a first-class, browseable catalog.
  They are stuck emailing binaries or documenting raw install strings.
- **The "raw install" escape hatch has no trust story.** Installing directly from
  a source URL works mechanically, but it strips away everything that makes a
  catalog useful: curated metadata, search, and — most importantly — a clear
  signal about provenance and trust.

Meanwhile, the technical foundation for solving this already exists in
`datumctl`. The catalog is already a structured document, installs already
record where each plugin came from, downloads are already integrity-checked, and
the on-disk layout already separates "what is installed" from "where it came
from." What is missing is not plumbing — it is the *product*: the ability to have
more than one catalog, and an experience that makes adopting additional catalogs
discoverable, friendly, and safe.

### Goals

- **Let third parties publish discoverable plugin catalogs** without going
  through Datum, using the same catalog format Datum uses.
- **Let users register additional catalogs** with a single, memorable command and
  a clear one-time trust decision.
- **Make search, browse, and install span every registered catalog**, so
  third-party plugins are first-class citizens of the discovery experience, not a
  second-tier escape hatch.
- **Make provenance and trust legible at every step** — users should always be
  able to tell, at a glance, whether a plugin comes from Datum's curated catalog
  or a third party they chose to trust.
- **Serve enterprise platform teams** as a first-class use case: private internal
  catalogs, pre-seeded onto workstations, with the option to constrain which
  catalogs are allowed.
- **Keep the curated default catalog the trusted, zero-configuration default** —
  opening the ecosystem must not dilute the safety of the out-of-the-box
  experience.
- **Let plugins be shared across milo-os-based platforms.** Because Datum Cloud is
  built on milo-os, a service built for milo-os should be able to publish one
  plugin that installs into `datumctl` — and any other milo-os-based CLI — from a
  catalog, rather than shipping a separate host-branded binary for each.

### Non-Goals

- **Replacing the curated Datum catalog.** The official catalog remains the
  default and the home of Datum-vetted plugins. Third-party catalogs sit
  alongside it, not in place of it.
- **Becoming a package manager for arbitrary software.** This is about `datumctl`
  plugins specifically, not a general-purpose distribution channel.
- **Hosting third-party plugin binaries.** Catalog authors host their own
  artifacts; Datum is not in the business of storing or serving third-party
  plugin binaries.
- **Guaranteeing the quality or safety of third-party plugins.** Datum provides
  trust *signals* and integrity verification, not an endorsement of every plugin
  in every catalog a user chooses to add.
- **Standardizing the full cross-platform plugin protocol.** Decoupling a plugin's
  *binary name* from the host CLI — so milo-os plugins install into `datumctl` — is
  in scope. Standardizing the entire plugin *contract* across every milo-os-based
  platform is a larger, separate effort, noted here only as a direction.
- **Defining the implementation rollout or sequencing.** This document describes
  the intended product experience and vision; engineering sequencing is out of
  scope here.

## Proposal

Introduce the concept of multiple, named **plugin catalogs** (called *indexes*
in command syntax, consistent with the existing internal model and with the
`kubectl krew` ecosystem familiar to this audience). Datum's curated catalog,
named `datum`, is the reserved default; users never have to think about it.
Anyone can stand up an
additional catalog by hosting a manifest file, and users register it by name:

```console
$ datumctl plugin index add acme https://plugins.acme.example/index.yaml
```

From that moment, the catalog's plugins are part of the user's world — they show
up in search, appear in the interactive browser, and install by a short,
catalog-qualified name. The first time a user adds a third-party catalog, they
see a clear, one-time explanation of what they are trusting. Every place a plugin
is shown, a badge tells the user where it came from.

The full experience — registration, addressing, search, browse, trust badges,
the publisher's authoring flow, and the enterprise story — is detailed in
[Design Details](#design-details). The sections below make it concrete through
the people it serves.

### User Stories

#### Story 1: A community author publishes a plugin people can find

Priya builds `datumctl-zonesync`, a plugin that bulk-imports DNS zones from a
CSV. Today she can publish a binary, but nobody can *discover* it. With this
enhancement she creates a small catalog repository describing her plugin and its
releases, and shares one line in her README:

```console
$ datumctl plugin index add priya github.com/priya/datumctl-plugins
```

Now anyone who has added her catalog finds `zonesync` in `datumctl plugin
search`, sees its description and homepage, and installs it by name. Priya didn't
need Datum's permission or a pull request, and her users get the same install
experience — integrity-checked download, version tracking, clean upgrades — that
official plugins get.

#### Story 2: A platform team ships an internal catalog to its engineers

ACME's platform team maintains three `datumctl` plugins that encode their
internal conventions and wrap their service catalog. They host a private catalog
manifest behind their own authenticated endpoint. New engineers run a single
documented command (or have it pre-seeded by their managed configuration), and
ACME's plugins appear right alongside Datum's:

```console
$ datumctl plugin search deploy
NAME          INDEX     VERSION  TRUST         DESCRIPTION
deploy        acme      v2.1.0   third-party   ACME guided deploy workflow
deploy-check  acme      v1.4.0   third-party   Pre-flight policy checks
```

The platform team controls their own release cadence and never has to route
internal tooling through Datum.

#### Story 3: A consumer browses and installs with confidence

Dana is new to `datumctl` and wants to see what's available. They open the
interactive browser:

```console
$ datumctl plugin browse
```

Dana filters by typing, reads descriptions, and sees a clear badge on each
plugin: **official** for Datum's curated catalog, **third-party** for catalogs
they've added. They select one, see exactly what it will install, choose to
install it, and move on — never having to copy a URL or wonder whether a plugin
is the "real" one.

#### Story 4: A user adopts a third-party catalog as an informed choice

Sam wants a community plugin. Adding the catalog is a deliberate, legible moment:

```console
$ datumctl plugin index add community github.com/datum-community/plugins

  You're adding a third-party plugin catalog: "community"

  Datum does not review, verify, or endorse plugins from this catalog.
  Plugins are programs that run on your machine with your Datum credentials.
  Only add catalogs you trust.

  Add this catalog? [y/N]
```

Sam understands what they're accepting, says yes once, and from then on the
community catalog is just part of their search and browse experience — with a
persistent badge so they never lose track of which plugins are third-party.

#### Story 5: A milo-os service ships one plugin that works in Datum Cloud

The milo-os IPAM service publishes its CLI plugin to a catalog, and a Datum Cloud
user installs it like any other plugin:

```console
$ datumctl plugin install ipam
$ datumctl ipam prefix claim --pool staging-backbone --length 24
```

The IPAM team didn't build or maintain a `datumctl`-specific release — one plugin
artifact installs into `datumctl`, and into any future milo-os-based CLI, straight
from the catalog. To the user the command is simply `datumctl ipam`, inheriting
their Datum identity, organization, and project. (The naming convention that keeps
one artifact portable across hosts is covered in
[Design Details](#plugin-binaries-and-platform-shared-plugins).)

### Notes/Constraints/Caveats

- **One catalog format for everyone.** Third parties author the exact same
  manifest format that Datum's curated catalog uses. There is no "lesser"
  third-party format. This keeps the ecosystem coherent and means a plugin can
  graduate from a personal catalog into Datum's curated one without changing its
  manifest.
- **The curated catalog stays the default and stays special.** It is the only
  catalog that is trusted with no action from the user, the only one that carries
  the **official** badge, and the home of plugins Datum has reviewed. Opening the
  ecosystem is strictly additive to this.
- **Terminology.** Command syntax uses `index` (matching the existing internal
  model and the `kubectl krew` vocabulary this audience knows), while
  user-facing prose can speak of "catalogs" and "the marketplace." A friendly
  display name, description, and owner travel with each catalog so listings read
  like a marketplace, not a list of URLs.
- **Catalog-qualified naming resolves ambiguity.** A bare plugin name resolves
  against the official `datum` catalog first; when the same name exists in more
  than one
  catalog, `datumctl` does not guess — it shows the user the options and asks them
  to qualify the name (e.g. `acme/deploy`).
- **A central hub is a possible future, not a requirement.** The decentralized,
  user-registers-a-catalog model stands on its own. A Datum-hosted aggregator
  that lets users discover public third-party catalogs without adding them first
  (in the spirit of Artifact Hub) is a natural extension if demand warrants, but
  is not required for this vision to deliver value.

### Risks and Mitigations

#### Risk: Users run untrusted third-party code

A plugin is a native program that runs with the user's credentials. A malicious
catalog could advertise a malicious plugin.

*Mitigations:* Adding a third-party catalog is an explicit, one-time trust
decision with plain-language consequences (Story 4). Every download continues to
be fetched over HTTPS and verified against a checksum in the manifest, so a
plugin cannot be silently swapped after the catalog author publishes it. Every
plugin shows a persistent **official** vs **third-party** badge wherever it
appears, so provenance is never ambiguous. The curated default catalog — the only
one trusted by default — remains Datum-reviewed. Stronger publisher-verification
signals (proving a catalog's ownership) and optional plugin signing are available
avenues to deepen this trust model over time.

#### Risk: Enterprises need to constrain what their users can add

In regulated or locked-down environments, free-for-all catalog registration is
unacceptable.

*Mitigations:* Managed configuration can both pre-seed approved catalogs onto
workstations and restrict which catalogs users may add (an allow-list by name or
host). This lets a platform team say "here are the approved Datum plugins" with
no per-user setup, while individuals on unmanaged machines keep the open
experience.

#### Risk: Name collisions and impersonation

Two catalogs could ship a plugin with the same name, or a hostile catalog could
try to impersonate the official one.

*Mitigations:* Catalog-qualified addressing makes every plugin unambiguously
referrable (`acme/deploy`), and `datumctl` refuses to guess on a bare-name
collision. The `datum` catalog name (and its `default` alias) and Datum's catalog
identity are reserved, so third parties cannot present themselves as official.

#### Risk: Ecosystem fragmentation and uneven quality

A long tail of low-quality or abandoned catalogs could erode user trust in
plugins generally.

*Mitigations:* The curated catalog remains the prominent, recommended default,
so the high-quality path is the obvious one. Trust badges make the distinction
between curated and third-party plugins constant and visible, so a poor
experience with a third-party plugin does not read as a failure of Datum's
plugins. Publisher-verification signals give good-faith catalog authors a way to
stand out.

#### Risk: UX and security review

This feature changes the trust surface of the CLI and introduces new user-facing
flows.

*Mitigations:* The trust prompt, badge language, and managed-configuration
controls should be reviewed jointly by the CLI maintainers and a
security-minded reviewer before stabilization; the browse and search UX should be
validated with real users (including a first-time community-catalog adopter and a
platform-team operator).

## Design Details

This section describes the product experience. It is intentionally focused on
what users, authors, and operators see and do, rather than on implementation
sequencing.

### Registering a catalog

Users manage catalogs through a small, predictable command set:

```console
# Add a catalog. Sources can be an HTTPS manifest URL, a GitHub owner/repo,
# or a local path (for development or air-gapped use).
$ datumctl plugin index add acme https://plugins.acme.example/index.yaml
$ datumctl plugin index add community datum-community/datumctl-plugins
$ datumctl plugin index add local ./my-catalog

# See what's registered, with a friendly, marketplace-style listing.
$ datumctl plugin index list
NAME        TYPE       PLUGINS  TRUST         DESCRIPTION
datum       official   12       official      Datum-curated plugins
acme        custom     3        third-party   ACME internal Datum tooling
community   custom     31       third-party   Community-contributed plugins

# Refresh catalog metadata, or remove a catalog.
$ datumctl plugin index update
$ datumctl plugin index remove community
```

The first time a user adds a third-party catalog, they see the one-time trust
explanation from Story 4. Downloads remain HTTPS-only and checksum-verified
regardless of how a catalog is rated.

### Installing with catalog-aware addressing

```console
$ datumctl plugin install zonesync          # bare name -> datum (official) catalog
$ datumctl plugin install acme/deploy       # catalog-qualified
$ datumctl plugin install owner/repo@v1.2.0 # direct from a release (unchanged)
```

When a bare name exists in more than one catalog, `datumctl` surfaces the options
instead of guessing:

```console
$ datumctl plugin install deploy
Error: "deploy" is available from multiple catalogs. Choose one:
  datumctl plugin install datum/deploy     Datum guided deploy        (official)
  datumctl plugin install acme/deploy      ACME guided deploy         (third-party)
```

### Searching and browsing across catalogs

Search spans every registered catalog and makes provenance a column, not a
guess:

```console
$ datumctl plugin search dns
NAME    INDEX      VERSION  TRUST         DESCRIPTION
dns     datum      v1.2.3   official      Manage Datum Cloud DNS zones
zonex   community  v2.1.0   third-party   Bulk zone import/export
$ datumctl plugin search dns --index acme   # scope to one catalog
```

An interactive browser turns discovery into a true marketplace experience —
type-to-filter, inspect a plugin's description, homepage, version, source
catalog, and trust badge, see exactly what will be installed, and install in
place:

```console
$ datumctl plugin browse
┌─ Datum Plugin Catalog ───────────────────────────────────┐
│ Filter: dns_                                             │
│ › dns     datum      official      Manage DNS zones       │
│   zonex   community  third-party   Bulk zone import       │
│ ↑/↓ move   / filter   enter details   i install   q quit  │
└──────────────────────────────────────────────────────────┘
```

### Trust signals everywhere a plugin appears

Provenance is never ambiguous. Every surface — `search`, `index list`, `browse`,
the install confirmation, and the list of installed plugins — shows a badge:

- **official** — Datum's reserved, curated catalog, named `datum`.
- **third-party** — any catalog the user has added.

The installed-plugins listing additionally shows which catalog each plugin came
from, so a user can audit their tools at a glance and tell which were Datum-vetted
and which they adopted themselves. A future **verified** badge can distinguish
third-party catalogs that have proven their ownership from anonymous ones.

### Publishing a catalog

Publishing should feel as light as adding a Homebrew tap or a krew index:

1. **Author a manifest** describing the catalog and its plugins (same format
   Datum's curated catalog uses; see below).
2. **Host it** at any HTTPS URL or in a git repository.
3. **Share one line:** `datumctl plugin index add <name> <source>`.
4. **Validate before shipping** with a built-in linter
   (`datumctl plugin index validate ./index.yaml`) so authors catch mistakes —
   bad platform selectors, missing checksums, non-HTTPS URLs — before users do.

The path into Datum's *curated* catalog remains a deliberate, reviewed process.
Third-party catalogs relieve the pressure on that path without removing the
curated tier; a successful community plugin can be promoted into the curated
catalog without changing its manifest.

### Enterprise and platform teams

Enterprises are a first-class audience:

- **Private internal catalogs.** A platform team hosts a manifest behind their own
  authenticated endpoint; existing token-based authentication for fetching from
  private sources applies.
- **Zero-touch provisioning.** Managed configuration can pre-register the company
  catalog so every engineer has it on first run, with no manual `index add`.
- **Guardrails.** Managed configuration can restrict which catalogs users may add
  (an allow-list by name or host pattern), so a regulated org can ship a closed,
  curated set of approved plugins while still benefiting from the catalog model.

### Catalog manifest format

Third parties produce the **same** structured catalog document that Datum's
curated catalog uses, so there is one format to learn and one set of tooling. The
per-plugin entries — name, description, homepage, version, and per-platform
download locations with checksums — are unchanged from today's catalog. The only
addition is an optional catalog-level header that gives a catalog a friendly
identity in listings and the browser:

```yaml
# Optional catalog identity, surfaced in `index list` and `browse`.
name: acme
description: ACME internal Datum tooling
owner: ACME Platform Team
homepage: https://plugins.acme.example

# Plugin entries (existing format, unchanged).
plugins:
  - name: deploy
    description: ACME guided deploy workflow
    homepage: https://plugins.acme.example/deploy
    version: v2.1.0
    platforms:
      - os: linux
        arch: amd64
        uri: https://plugins.acme.example/deploy_linux_amd64.tar.gz
        sha256: "…"
```

Because the format is shared and backward-compatible, every catalog that exists
today keeps working, and a plugin never has to be rewritten to move between a
personal catalog, a company catalog, and Datum's curated catalog.

### Plugin binaries and platform-shared plugins

A milo-os service can ship one CLI plugin that installs into `datumctl` and into
any other milo-os-based CLI. For that to work, the plugin's binary name has to stay
independent of any single host's brand while still identifying itself as a plugin —
a service repository often ships several binaries, including the service itself.

milo-os plugins use a `milo-<command>` convention: the IPAM plugin is `milo-ipam`,
distinct from the service's own `ipam` binary and recognized by any milo-os-based
CLI. datumctl-native plugins keep the existing `datumctl-<command>` form. Either
way the user types the unprefixed command (`datumctl ipam`), and a catalog entry
can name exactly which binary in a release archive is the plugin.

The prefix is also a security boundary for binaries found on `PATH`: `datumctl`
treats a `PATH` binary as a plugin only when it carries a known prefix, so an
unrelated program named `ipam` is never run as a command. Catalog-installed plugins
are additionally integrity-checked and recorded.

Standardizing the plugin *contract itself* — the manifest, and the context and
credentials a host passes to a plugin — across every milo-os platform is a natural
next step beyond this document, so "build once, run on any milo-os platform" holds
end-to-end, not just for the binary name.

## Production Readiness Review Questionnaire

This is a **client-side `datumctl` feature**. It introduces no Datum
control-plane components, API types, or server-side services, so most of the
cluster-oriented PRR questionnaire does not apply. The relevant readiness
considerations are captured below.

### Feature enablement and rollback

- The capability ships as part of `datumctl`. Existing single-catalog behavior is
  preserved as the default: a user who never runs `plugin index add` sees no
  change beyond the new catalog badges in existing output.
- The default curated catalog continues to work exactly as it does today; the
  feature is purely additive.
- Rolling back is a matter of installing a prior `datumctl` version. Registered
  third-party catalogs and installed plugins are recorded in the user's local
  configuration; a downgrade ignores the additional-catalog configuration and
  falls back to default-catalog behavior, without affecting already-installed
  plugins.

### Monitoring and supportability

- Users can always determine which catalogs they have registered
  (`datumctl plugin index list`) and which catalog each installed plugin came
  from (the installed-plugins listing), which is the primary support and audit
  surface.
- Catalog fetches degrade gracefully: an unreachable third-party catalog produces
  a clear, actionable message and does not break the rest of the CLI or the
  default catalog.

### Dependencies

- Third-party catalogs depend on infrastructure operated by their own authors
  (an HTTPS endpoint or git host). An outage of a third-party catalog affects only
  that catalog's discovery and installs; the CLI and the default catalog are
  unaffected.

### Security

- The trust prompt, badge language, and managed-configuration guardrails should
  receive a joint CLI + security review before the feature stabilizes (see
  [Risks and Mitigations](#risks-and-mitigations)).

## Implementation History

- (provisional) Enhancement drafted, focusing on the product experience and
  vision for multi-catalog plugin discovery and installation.

## Drawbacks

- **It expands the CLI's trust surface.** Today, every plugin a user can discover
  comes from a catalog Datum operates. Opening the ecosystem means users can opt
  into running third-party code discovered through third-party catalogs. This is
  mitigated by explicit trust decisions, persistent provenance badges, integrity
  verification, and enterprise guardrails, but it is a real change in posture.
- **It introduces ecosystem-management responsibility.** Reserved names, trust
  badges, publisher verification, and impersonation defenses are ongoing
  concerns once third parties can publish.
- **It adds surface area to learn.** Users now have catalogs as a concept. This is
  mitigated by keeping the default catalog invisible to users who don't need more,
  and by making the new concepts mirror tools (krew, Homebrew, gh) the audience
  already knows.

## Alternatives

- **Keep the single curated catalog and grow it via pull requests.** This keeps
  the trust model simple but does not scale, cannot serve private/internal
  plugins at all, and makes Datum the bottleneck and reviewer for every
  third-party tool. Rejected as the primary model; it remains the curated *tier*
  within the broader ecosystem.
- **Rely on direct source installs (`owner/repo`) for everything third-party.**
  This works mechanically but provides no discovery, no curated metadata, and no
  trust signal — exactly the gaps named-catalog discovery closes. It remains available as
  a power-user escape hatch.
- **Build a centralized Datum-hosted marketplace first.** A hosted aggregator
  (à la Artifact Hub) is attractive but is a larger investment that presumes
  demand. The decentralized, register-a-catalog model delivers the core value
  without it, and a hub can be layered on later as an enhancement to discovery.
- **Require cryptographic signing of all third-party plugins up front.** Signing
  meaningfully strengthens trust, but mandating it as a precondition would raise
  the bar for community authors and slow adoption. Checksum-verified downloads
  plus explicit, badged trust match the bar set by comparable ecosystems (krew,
  gh, Homebrew); opt-in signing is a natural way to deepen the model later.
- **Name plugins with the host prefix (`datumctl-<command>`) or with a bare,
  unprefixed name.** A host prefix forces every milo-os service to ship a
  `datumctl`-branded binary — and a separate one for every other milo-os-based CLI —
  defeating build-once. A bare name (`ipam`) collides with the service's own
  binaries and can't be safely discovered on `PATH`. The `milo-<command>`
  convention avoids both: host-agnostic yet plugin-identifying. The
  `datumctl-<command>` prefix is retained for datumctl-native plugins and manual
  installs.

## Infrastructure Needed

- A documented, recommended layout for hosting a catalog manifest (HTTPS endpoint
  or git repository), with examples authors can copy.
- Optional, future: a Datum-hosted catalog aggregator ("hub") for discovering
  public third-party catalogs without registering them first, and the supporting
  publisher-verification signals.
