# Experiences

**The different ways you work with Datum.**

People reach Datum Cloud through several distinct experiences — a web portal, a
command-line tool, an API, a Kubernetes operator, CI/CD integrations, and more.
Each one is a product in its own right, with its own users, its own ergonomics,
and its own roadmap.

This section organizes enhancements by the experience they shape, rather than by
the underlying platform capability they touch. It's the home for proposals that
make a given way of working with Datum better — clearer, friendlier, more
powerful — independent of which domain feature is behind it.

## How this is organized

- Enhancements about **an experience itself** — how it looks, feels, and behaves
  for the people using it — live here, grouped by experience.
- Enhancements about **a platform capability** (for example, DNS zone transfers
  or BGP control-plane behavior) live under their domain section
  (`networking/`, `compute/`, `telemetry/`, and so on), even when a particular
  experience surfaces them.

When something genuinely spans both, file it where the primary intent lives — the
experience if the proposal is fundamentally about the way people interact with
Datum, the domain if it's fundamentally about what the platform can do.

## Experiences

- [`datumctl/`](./datumctl) — the official command-line tool for Datum Cloud.
- [`kubernetes-operator/`](./kubernetes-operator) — Datum Cloud connectivity
  brought into Kubernetes through standard Gateway and Ingress APIs.

Other experiences (Cloud Portal, API, CI/CD, …) can be added here as their
enhancements are proposed.
