# Phase 2: Per-zone nameservers

Part of [Hosted zone claims and ownership by delegation](README.md), which has
the background, goals, and user stories. Phase 2 is designed here and deferred;
[Phase 1](phase1.md) is the current ask.

Phase 2 is designed here so the decision to fund it can be made on its merits.
Its main argument is cost of delay: moving existing customers means a registrar
change for each of them, and that cost grows with the customer base.

## Model

- **Each zone gets its own set of four nameservers**, one from each of four
  address blocks, each with its own hostname and address.
- **Zones serve immediately on their own addresses**, as Route 53 does.
- **Delegation proves ownership.** A domain is owned by the project whose zone's
  addresses appear in the registry delegation. One matching address is enough,
  because zones for the same name never share an address.
- **Names stop being exclusive.** Several projects can have zones for the same
  name. Only the one the registry points to receives queries. Claims,
  reservations, and contests from Phase 1 are retired.
- **The dependency becomes strictly one-way.** dns-operator allocates, serves,
  and publishes each zone's set, and never reads `Domain`.
  network-services-operator reads the set as evidence.

## Safeguards

All three are required for serving immediately to be safe:

1. **A unique, reversible assignment.** A keyed format-preserving Feistel
   permutation (NIST SP 800-38G FF1) maps a counter to a 4-tuple of block
   indexes. Every zone gets a unique tuple, and a tuple maps back to its zone
   for attack attribution.
2. **No overlap between zones of the same name, or parent and child zones.** The
   allocator skips any tuple that shares an address with such a zone. Route 53
   enforces the same rule: two hosted zones "that have the same name or that
   have a parent/child relationship ... can't have any common name servers."
3. **Per-address serving.** Each address answers only for the zones assigned to
   it.

## Allocation

| Property         | Value                                                                                                                                                                                                     |
| ---------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Blocks           | Four /24s out of `67.14.160.0/22` (AS33438), plus four /48s out of `2607:ed40:0::/46`. The same index maps to one IPv4 and one IPv6 address.                                                              |
| Radix            | **252 per block, fixed permanently.** `.0`, `.1`, `.254`, and `.255` are reserved in every block. Changing the radix later would remap every counter value.                                               |
| Capacity         | 252⁴, about 4.03 billion zones                                                                                                                                                                            |
| Legacy addresses | `.128` in each block serves today's `ns1` to `ns4`. Tuples containing it are skipped until the migration deadline.                                                                                        |
| Key              | Stored in Secret Manager. The key version is recorded with each assignment, so rotation affects only new zones. An unkeyed permutation would let anyone recover allocation order and the number of zones. |
| Counter          | One counter, updated by compare-and-swap. Tuples are never reused.                                                                                                                                        |
| Record           | The assignment and key version are stored on the `DNSZone` status.                                                                                                                                        |

## Naming

Each block uses a different TLD from a different registry operator, so one
registry's outage can't take out every nameserver. Each block also has its own
theme, so you can tell the block from a nameserver's name.

| Block | TLD      | Registry operator                                  | Theme                 | Example                     |
| ----- | -------- | -------------------------------------------------- | --------------------- | --------------------------- |
| A     | `.net`   | Verisign                                           | animals               | `otter.datumdomains.net`    |
| B     | `.org`   | Public Interest Registry (run by Identity Digital) | colors                | `cobalt.datumdomains.org`   |
| C     | `.co.uk` | Nominet                                            | fruits and vegetables | `quince.datumdomains.co.uk` |
| D     | `.ch`    | SWITCH                                             | space                 | `vega.datumdomains.ch`      |

- `.com` is ruled out because Verisign also runs `.net`.
- `.io` and `.info` are ruled out because Identity Digital also runs `.org`'s
  registry.
- Names are never renamed or reused.

## Migration

Every existing zone moves to its own set by a deadline after launch, when its
customer changes their registrar's nameservers. After the deadline, the legacy
`ns1` to `ns4` addresses stop serving. Datum has few customers today, so one
model at the end is worth the one-time change.

## Open questions

> [!IMPORTANT]
>
> These points are unresolved and open for debate:
>
> - **Per-address serving mechanism.** Each address must answer only for its own
>   zones. The candidates are dnsdist in front of PowerDNS
>   ([dns-operator#125](https://github.com/datum-cloud/dns-operator/pull/125))
>   and PowerDNS views.
> - **Glue records and registrars.** Each nameserver name sits inside its own
>   domain, so each needs a glue record at its registry: 252 per domain, 1,008
>   in total, across four registrars. Registrar limits and per-host costs need
>   confirming.
> - **Address binding.** Cilium LB IPAM would bind about 1,008 addresses per
>   address family to the authoritative DNS Service. This needs testing at that
>   size.
> - **Migration deadline.** Proposed: 60 days after launch.
