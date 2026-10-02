---
status: provisional
stage: alpha
latest-milestone: "v0.1"
---
<!-- omit from toc -->
# IdentitySet and Policy

These resource schemas are not defined yet. These examples capture intent, so that the schemas can be tested
against them later. Field names in the sketches are illustrative only. 

<!-- omit from toc -->
## Contents

- [Why an idSet](#why-an-idset)
- [What an idSet is](#what-an-idset-is)
- [Policy](#policy)
- [Candidate actions](#candidate-actions)
- [Examples](#examples)
- [Sketches](#sketches)

---

## Why an idSet

Agents and other workloads arrive at Datum's edge from many platforms, over differing transport types (iroh
tunnels, MASQUE tunnels, mTLS transport, IPsec tunnels) and with many credential forms: tokens from
different issuers, certificates, and tunnel keys. Many are transient: a sandbox may live for seconds, so
its instances cannot be registered one by one, and policy written against individual principals would
never keep up. The same workload is also recognised differently at each enforcement point: by a token claim
at the ALB, and at the MASQUE Gateway by its registered key or the labels it enrolled with.

A stable name for **who** is therefore required, one that holds while members come and go and means the
same thing everywhere. And the components that enforce it, such as the gateways and the Credential broker,
each understand only their own vocabulary. An idSet is that stable name, and is translated into a particular
component's vocabulary by a controller.

## What an idSet is

- **A named group of identities, in one project.** An idSet refers to a group of potentially disparate
  identities within a project, much as a label selector refers to a group of workloads within a network
  policy. A Policy does not change the idSet, and one principal may belong to several idSets.
- **It selects on the principal:** who it is, and later what it runs. What it is doing, and for whom, are
  conditions in the Policy.
- **It selects on identity attributes:** a token's issuer, subject and claims; a certificate's URI or
  DNS name; the labels of a registered `Connector`; or the labels of the `EnrollmentProfile` a sandbox's
  token came from.
- **Its members are rules or named identities, never instances.** A rule is an issuer plus claim values,
  or a selector on labels; a named identity is an agent's identifier, or a durable host's key. E.g., a sandbox
  that starts matches a rule without being registered.
- **The controller expands an idSet into its members** in the control plane, and renders them into what
  each component understands: gateway rules, the MASQUE Gateway's configuration, the Credential broker's
  configuration. No component outside Datum's control plane ever sees an idSet. A future extension could
  have edge controllers perform the expansion at the edge.
- **An idSet can be deleted and recreated with the same name and different members.** The controller
  keeps a ledger of every membership it evaluates, keyed by the idSet's UID and generation, so audit can
  tell them apart.

## Policy

**The Policy resource will be defined in the future.** A Policy binds an idSet, as its subject, to a
target and an action. It refers only to idSets in its own project, and each party's Policies apply only to
the objects its own project owns. More sketches, including a Datum-hosted target, are in
[`Connections-IAM.md`](Connections-IAM.md).

```yaml
# Tunnelled researchers may reach only the orders database, behind acme's datacenter.
kind: Policy
metadata: { name: researchers-reach-orders-db, namespace: project-blue }
spec:
  subject: { identitySetRef: { name: researcher-tunnels } }
  target:                                  # instances on Datum compute: a networkServiceRef instead
    networkRef: { name: acme-network }
    prefixes: [ 10.200.1.50/32 ]
    ports: [ 5432 ]
  action: Allow
```

## Candidate actions

Future possibilities only, to be considered when customer requirements shape the Policy resource. Beyond
allow and deny, a seam that has identified the caller can treat its traffic by identity.

| Action | What it does | Enforceable at |
|---|---|---|
| **Allow / Deny / Drop** | Admit or refuse; on the network path, refused packets are dropped | Every seam |
| **Steer** | Send the idSet's traffic or tunnels to a named PoP or MASQUE Gateway | Inbound seam, or DNS answers |
| **Redirect** | Send the idSet to an alternate target, such as a maintenance endpoint or an inspection sandbox | ALB and AI Gateway for HTTP; MASQUE Gateway by route |
| **Mirror** | Copy the idSet's traffic to an inspection point while serving it normally | ALB and AI Gateway; MASQUE Gateway for packets |
| **Pin egress** | Let the idSet leave Datum only from a named PoP or a fixed address pool | Inbound seam, which chooses the egress |
| **Deliver via** | Reach an organization's own datacenter only through the PoP holding its `Connector` | Inbound MASQUE Gateway |
| **Quarantine** | Move the idSet into an isolated network instead of dropping it, keeping evidence | MASQUE Gateway, by the address and routes it assigns |
| **Rate-limit** | Cap requests or tokens, or set a traffic class, per idSet | ALB and AI Gateway |

## Examples

The names below are defined by each organization and have significance only to it. Datum keys idSets
instead by their UID and generation.

- **`researcher-sandboxes`**: a service provider's sandboxes running researcher agents for one
  organization. At stage 1 they are matched by the labels of the `EnrollmentProfile` their tokens came
  from; at stage 2, by the workload identity each presents when its tunnel is set up. Used to let them
  reach one database in the organization's datacenter, and nothing else.
- **`researcher-sandboxes`**, again: the same idSet, reaching the datacenter only through the PoP holding
  the organization's `Connector`.
- **`model-callers`**: agents in a partner's sandboxes that call a model provider through the AI Gateway.
- **`build-agents`** and **`ci-sandboxes`**: two idSets in one project, one allowed to push to a repository
  and the other only to pull.
- **`support-agents`**: a subset of an organization's agents that act for members of one user group. Acting
  for that group, and read-only access, are conditions in the Policy, not part of the idSet.
- **`provider-sandboxes`** and **`acme-researchers`**: a service provider's idSet in its own project, and
  an organization's idSet in its own. Each party's rules apply only to the objects its own project owns.
- **`agents-under-review`**: agents whose traffic is mirrored to an inspection point while they keep
  working.
- **`attested-compute`**: workloads on Datum compute, admitted only with runtime attestation.
- **Deferred:** a session valid only while its task is live. The task arrives as a transaction token, as a
  Policy condition; see [`dataplane-workload-identity-framework.md`](dataplane-workload-identity-framework.md).

## Sketches

**Not a schema.** Each sketch is one idSet in one project, holding members of one kind.

**An idSet of workload identities**, matched on the credentials they present:

```yaml
kind: IdentitySet
metadata: { name: researcher-sandboxes, namespace: project-blue }
spec:
  members:
    - workloadIdentity:                    # agents from a registered issuer, by claims
        issuerRef: { name: sandbox-platform }
        claims: { org: acme, agentClass: researcher }
    - workloadIdentity:                    # a named agent, by subject
        issuerRef: { name: acme-ci }
        subjects: [ "spiffe://acme.example/agent/triage" ]
    - workloadIdentity:                    # a certificate, by its URI SAN
        certificate: { uriSAN: "spiffe://acme.example/agent/researcher" }
```

**An idSet of transport identities**, matched on labels Datum recorded, never on labels the endpoint
asserts:

```yaml
kind: IdentitySet
metadata: { name: researcher-tunnels, namespace: project-blue }
spec:
  members:
    - transportIdentity:                   # durable hosts registered as Connectors
        connectorSelector:
          matchLabels: { datum.net/agent-class: researcher }
    - transportIdentity:                   # sandboxes enrolled with tokens from matching profiles
        enrollmentProfileSelector:
          matchLabels: { datum.net/agent-class: researcher }
```

- **The first member** suits durable hosts, and sandboxes their platform registers in advance. The labels
  are on the `Connector` object.
- **The second** suits sandboxes nobody registers. The labels are on the `EnrollmentProfile`. A sandbox
  presents a single-use token minted from the profile when its tunnel is set up, and is a member until the
  tunnel closes.
