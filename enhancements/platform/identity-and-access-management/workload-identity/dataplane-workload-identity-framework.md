---
status: provisional
stage: alpha
latest-milestone: "v0.1"
---
<!-- omit from toc -->
# Data-plane workload identity framework

A framework for Identity and AuthN/Z across the dataplane, and mapped to the framing in 
standards. **It is a framework, not a design:** each Datum service instantiates it in its 
own design, as Datum Connections does in [`Connections-IAM.md`](Connections-IAM.md). The framework establishes
common terminology for concepts to avoid conflicts across different service implementations.

<!-- omit from toc -->
## Contents

- [Three kinds of identity](#three-kinds-of-identity)
- [The identity record](#the-identity-record)
- [Mapping to the standards](#mapping-to-the-standards)
- [Additional implications beyond the existing standards](#additional-implications-beyond-the-existing-standards)
- [Related documents](#related-documents)

---

## Three kinds of identity

Datum knows a workload by up to three kinds of identity, each answering one question:
- **Transport identity, which endpoint?** The tunnel key, proven by the handshake.
- **Workload identity, who is it?** A token naming the workload, bound to that key. The instance is the key.
- **Transaction identity, what is it doing and for whom?** A token naming the task and its root principal.

## The identity record

The IETF WIMSE WG provides a [framework for AI agent identity](https://www.ietf.org/archive/id/draft-ietf-wimse-aims-00.html), which can be leveraged here: a 
Workload Identity Token (WIT), a transaction token (Txn-Token), and a Workload Proof Token (WPT) that 
binds both to each request with the instance's key. Each is issued by a different party, on its own schedule.

```mermaid
block-beta
  columns 3
  wpt["<b>WPT: the seal on each request</b><br/>signed with the instance's key K, carrying hashes of the WIT and the Txn-Token"]:3
  q1["<b>Who is it?</b><br/>WIT<br/>the workload's identifier,<br/>bound to the instance's key K"]
  q2["<b>What is it running?</b> Future<br/>a digest of its code, configuration<br/>and models, and who vouched"]
  q3["<b>What is it doing, and for whom?</b><br/>Txn-Token<br/>the task, the root principal, this hop"]
  ledger[("<b>Datum's ledger</b>: where the digests and hashes resolve, and every token and fact Datum records")]:3
  classDef rec fill:#f6e7c1,stroke:#a8813a,color:#3b2a06
  classDef fut fill:#eeeeee,stroke:#999,color:#555
  classDef seal fill:#e8dcf0,stroke:#7a4fa8,color:#25062f
  classDef led fill:#dcefe0,stroke:#3a8a52,color:#062510
  class q1,q3 rec
  class q2 fut
  class wpt seal
  class ledger led
```

## Mapping to the standards

The IETF's AI identity management draft (AIMS, `draft-ietf-wimse-aims-00`) treats an agent as a workload
and layers existing standards. This Datum framework builds on this:

```mermaid
block-beta
  columns 3
  h1["Layer"] h2["What Datum builds"] h3["Standards"]
  z["<b>Authorization</b>"] zd["access rules per group, compiled once<br/>and enforced at each gateway"] zs["OAuth 2.0, token exchange (RFC 8693),<br/>identity chaining across domains"]
  n["<b>Authentication</b>"] nd["gateway: once per tunnel, the token and the key the handshake proved<br/>ALB: every request, with a proof of the key"] ns["WIMSE proof tokens, HTTP Message Signatures,<br/>MASQUE (RFC 9484)"]
  t["<b>Transaction identity</b>"] td["Txn-Token: the task, the root principal, this hop"] ts["OAuth transaction tokens"]
  w["<b>Workload identity</b>"] wd["WIT: the workload's identifier, bound to the instance's key"] ws["WIMSE workload credentials, SPIFFE"]
  r["<b>Transport identity</b>"] rd["the tunnel key: registered, enrolled,<br/>or named by the WIT that binds it"] rs["MASQUE over QUIC or TLS,<br/>X.509 workload certificates"]
  i["<b>Naming</b>"] id["a URI naming the workload, scoped to its project (scheme open)<br/>the instance is its key"] is["WIMSE identifier, SPIFFE ID"]
  classDef head fill:#444,stroke:#444,color:#fff
  classDef auth fill:#e8dcf0,stroke:#7a4fa8,color:#25062f
  classDef ident fill:#f6e7c1,stroke:#a8813a,color:#3b2a06
  classDef name fill:#dcefe0,stroke:#3a8a52,color:#062510
  classDef std fill:#d8e8f6,stroke:#3a6ea8,color:#062033
  class h1,h2,h3 head
  class z,zd,n,nd auth
  class t,td,w,wd,r,rd ident
  class i,id name
  class zs,ns,ts,ws,rs,is std
```

## Additional implications beyond the existing standards

AIMS does not cover attaching at the network layer, or a gateway that enforces access.

## Related documents

- [`Connections-IAM.md`](Connections-IAM.md): how Datum Connections instantiates the framework, in stages.
- `Workload-identity-attributes.md`: Examples of how the framework can be instantiated in different scenarios and mappings to existing standards and initiatives.
