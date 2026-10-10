# datumctl

**The official command-line tool for Datum Cloud.**

`datumctl` is how people work with Datum Cloud from the terminal — managing
organizations, projects, DNS, networking, and more, without needing to know
Kubernetes. It's a first-class experience, and this section collects the
enhancements that shape it: how it's structured, how it's extended, and how it
grows alongside the platform.

## Enhancements

- [`api-proxy/`](./api-proxy) — a local front door to the Datum Cloud API: one
  command gives every tool on your machine authenticated access, with
  credentials attached automatically and kept fresh in the background.
- [`plugin-marketplace/`](./plugin-marketplace) — open the plugin ecosystem so
  community authors and enterprise platform teams can publish their own catalogs
  of `datumctl` plugins, and users can register, search, browse, and install
  across them — with the curated Datum catalog staying the trusted default and
  clear trust signals throughout.
- [`vpc-plugin/`](./vpc) — a `kubectl`-modeled plugin for managing Virtual Private
  Clouds: `datumctl vpc create`, `get`, `describe`, and `delete`, with
  platform-assigned ULA IPv6, structured output formats, and confirmation-safe
  deletion.

_New datumctl enhancements are added here as they're proposed._
