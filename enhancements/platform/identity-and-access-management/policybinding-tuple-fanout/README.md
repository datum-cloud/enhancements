---
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

<!-- omit from toc -->

# Reduce PolicyBinding tuple fan-out

Related: [datum-cloud/auth-provider-openfga](https://github.com/datum-cloud/auth-provider-openfga)

- [Summary](#summary)
- [Motivation](#motivation)
  - [Where the time goes](#where-the-time-goes)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [How it works today](#how-it-works-today)
  - [The change](#the-change)
- [Design Details](#design-details)
  - [Authorization model](#authorization-model)
  - [Policy reconciler](#policy-reconciler)
  - [Migration](#migration)
- [Risks and Mitigations](#risks-and-mitigations)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Open Questions](#open-questions)
- [Implementation History](#implementation-history)

## Summary

Every PolicyBinding is expanded into OpenFGA tuples by writing one tuple for
each pair of (subject, permission) on the target object. A role with a handful
of permissions is cheap. The organization owner role is not: it grants every
permission across the whole resource hierarchy, so a single owner binding turns
into 351 tuples.

That fan-out lands on the critical path of signup. A new user cannot use their
organization until the owner grant has propagated, and the owner grant is 351
sequential-ish tuple writes behind a reconciler that also writes them one
binding at a time.

This proposal moves the fan-out out of the tuple data and into the
authorization model. A binding would write a single "this subject has this role
on this object" tuple, and the model would express which permissions that role
grants. The model is written once; the per-binding cost drops from N tuples to
one.

## Motivation

Signup latency on staging is dominated by IAM propagation, and the largest
single contributor is the organization owner grant. We confirmed this from
controller and OpenFGA logs across several real signups.

### Where the time goes

The owner binding writes 351 tuples. We saw this directly in the controller
logs when the binding was torn down:

```
Successfully deleted tuples for PolicyBinding ... "tupleCount": 351
```

Time from user creation to the owner grant being ready, across three signups on
the same day:

| Signup      | Org owner ready | Project owner ready |
| ----------- | --------------- | ------------------- |
| `org-kzjnm` | +59s            | +82s                |
| `org-tbsxl` | +25s            | +47s                |
| `org-drc28` | +12s            | +37s                |

The spread (12s to 59s for the same operation) tracks OpenFGA write latency and
reconciler queue pressure, not anything the user did. Supporting numbers from
the same window:

- OpenFGA `Write` p99: about 990ms
- OpenFGA `Check` p99: about 10ms (checks are not the problem)
- `policybinding` reconcile p99: about 3.4s on a bad day, 0.9 to 1.1s on an
  average one

A smaller resource, a project owner binding, is noticeably faster because it
carries far fewer permissions. The pattern is consistent: cost scales with the
permission count of the bound role.

### Goals

- Make the number of OpenFGA tuples a PolicyBinding writes independent of how
  many permissions its role grants.
- Cut the time from organization creation to the owner grant being usable.
- Keep authorization decisions identical. A Check that passes today must pass
  after the change, and one that fails today must still fail.

### Non-Goals

- Changing the PolicyBinding, Role, or ProtectedResource APIs. This is an
  internal representation change in the OpenFGA provider.
- Changing how the portal waits for grants during onboarding. That work is
  tracked separately.
- Reworking the resource hierarchy or inheritance semantics.

## Proposal

### How it works today

For each resource type the model defines one relation per permission, keyed by a
hash of the permission name. A PolicyBinding reconciler expands the role's
effective permissions and writes a direct tuple per permission:

```
(InternalUser:alice, hash(org.get),  Organization:org-1)
(InternalUser:alice, hash(org.list), Organization:org-1)
... one per permission ...
```

For the owner role that expansion is 351 rows for one subject on one object.

### The change

Represent the grant as role membership instead of a flattened permission list.
A binding writes a single tuple that says "alice has role R on org-1", and the
authorization model already knows that role R implies `org.get`, `org.list`, and
the rest. The permission relations resolve through the role rather than through
351 direct assignments.

```
# today: 351 tuples per owner binding
(InternalUser:alice, hash(org.get),  Organization:org-1)
(InternalUser:alice, hash(org.list), Organization:org-1)
...

# proposed: 1 tuple per owner binding
(InternalUser:alice, role_<roleHash>, Organization:org-1)
```

The total amount of "which permission belongs to which role" information does
not disappear. It moves into the authorization model, which is computed once and
updated only when roles or protected resources change, rather than being copied
into tuple storage on every binding.

## Design Details

### Authorization model

Add a role-scoped relation to each resource type. For every role that can target
a resource type, the type gains a relation such as `role_<roleHash>` that is
directly assignable. Each existing permission relation is then defined as the
union of its current definition and the roles that include that permission:

```
hash(org.get) = direct_assignment
             or role_<ownerHash>       # owner includes org.get
             or role_<viewerHash>      # viewer includes org.get
             or tuple_to_userset(parent, hash(org.get))
```

The model grows with (roles x permissions), but it is written once by the
authorization model reconciler and does not touch the signup path. Direct
per-permission assignment stays supported so existing tuples keep resolving
during migration.

### Policy reconciler

`buildPermissionTuples` currently emits one tuple per (subject, permission). The
change is to emit one tuple per (subject, role) targeting the role relation,
when the bound role is eligible for the role-based path. The diff, sibling, and
delete logic stays the same; it simply operates over a much smaller desired set.

### Migration

The two representations resolve to the same Check results, so they can coexist:

1. Ship the model change first, adding role relations without removing the
   per-permission ones. No behavior change.
2. Switch the reconciler to write role tuples for new and re-reconciled
   bindings. Old bindings keep their permission tuples and still work.
3. Backfill: re-reconcile existing bindings to replace permission tuples with a
   role tuple. This is idempotent and can run in the background.
4. Once no permission-style tuples remain for role-eligible bindings, the direct
   per-permission assignment on permission relations can be reconsidered.

Each step is independently revertible.

## Risks and Mitigations

The main risk is an authorization regression: a subtle difference between the
old and new resolution paths could grant or deny something incorrectly.
Mitigations:

- Keep both representations valid at once so rollback is a reconciler flag, not a
  data migration.
- Add Check-equivalence tests that assert, for a representative set of subjects,
  roles, and resources, that the role-based model returns the same decision as
  the per-permission model.
- Roll out on staging first and diff authorization decisions against production
  behavior before backfilling.

A secondary risk is model size. Unioning every permission over every role could
make the model large. If that becomes a problem, scope role relations to the
resource types a role actually targets rather than all types.

## Drawbacks

The authorization model becomes more complex to read, since permission relations
now reference roles. Debugging a Check means understanding both the tuple and the
model, where today the tuple alone tells most of the story. The migration also
adds a period where both representations exist, which is more moving parts than a
single cutover.

## Alternatives

**Batch the existing writes.** Keep the per-permission tuples but write them in
larger OpenFGA `Write` batches. This lowers wall-clock time without changing the
model, but the cost still scales with permission count and storage still holds
351 rows per owner binding.

**Shrink the owner role.** Grant the owner fewer permissions. This changes
product behavior and does not address the general case of any broad role.

**Precompute owner tuples at org creation.** Write the owner grant as part of
org bootstrap rather than through a PolicyBinding. This narrows the fix to one
role and leaves the fan-out in place for every other broad binding.

## Open Questions

<<[UNRESOLVED role-scoping ]>>
Should role relations be added to every resource type, or only to the types a
given role actually targets? The first is simpler; the second keeps the model
smaller.
<<[/UNRESOLVED]>>

<<[UNRESOLVED groups ]>>
Group-based subjects already resolve through `member`. Confirm the role-based
path composes correctly with group membership so a group granted a role still
resolves every permission.
<<[/UNRESOLVED]>>

## Implementation History

- (provisional) Summary and motivation drafted from staging signup analysis.
