---
status: provisional
stage: alpha
latest-milestone: "v0.1"
---
<!--
Inspired by https://github.com/kubernetes/enhancements/tree/master/keps/NNNN-kep-template

Goals are aligned in principle with those described at https://github.com/kubernetes/enhancements/blob/master/keps/sig-architecture/0000-kep-process/README.md

Recommended reading:
  - https://developers.google.com/tech-writing
-->

<!--
**Note:** When your Enhancement is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Make a copy of this template directory.**
  Copy this template into the desired path and name it `short-descriptive-title`.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary" and "Motivation" sections.
  These should be easy if you've preflighted the idea of the Enhancement with the
  appropriate stakeholders.
- [ ] **Create a PR for this Enhancement.**
  Assign it to stakeholders who are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the Enhancement clarified and merged quickly. The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a Enhancement is merged does not mean it is complete or approved. Any Enhancement
marked as `provisional` is a working document and subject to change. You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing RFCs, aim for tightly-scoped, single-topic PRs to keep discussions
focused. If you disagree with what is already in a document, open a new PR
with suggested changes.

One Enhancement corresponds to one "feature" or "enhancement" for its whole lifecycle.
You do not need a new Enhancement to move from beta to GA, for example. If
new details emerge that belong in the Enhancement, edit the Enhancement. Once a feature has
become "implemented", major changes should get new RFCs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/docs/rfcs/template/README.md).

**Note:** Any PRs to move a Enhancement to `implementable`, or significant changes once
it is marked `implementable`, must be approved by each of the Enhancement approvers.
If none of those approvers are still appropriate, then changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross-cutting RFCs).
-->

# Milo Performance Testing

<!--
This is the title of your Enhancement. Keep it short, simple, and descriptive. A good
title can help communicate what the Enhancement is and should be considered as part of
any review.
-->

<!--
A table of contents is helpful for quickly jumping to sections of a Enhancement and for
highlighting any additional information provided beyond the standard Enhancement
template.
-->

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)

## Summary

<!--
This section is incredibly important for producing high-quality, user-focused
documentation such as release notes or a development roadmap. It should be
possible to collect this information before implementation begins, in order to
avoid requiring implementors to split their attention between writing release
notes and implementing the feature itself. Enhancement editors should help to ensure
that the tone and content of the `Summary` section is useful for a wide audience.

A good summary is probably at least a paragraph in length.

Both in this section and below, follow the guidelines of the [documentation
style guide]. In particular, wrap lines to a reasonable length, to make it
easier for reviewers to cite specific portions, and to minimize diff churn on
updates.

[documentation style guide]: https://github.com/kubernetes/community/blob/master/contributors/guide/style-guide.md
-->


We want to establish repeatable, automated performance testing for the **Milo API server**, specifically around its multi-tenant constructs: **Projects** and their associated resources. Projects in Milo act as virtual control planes. They are cluster-scoped custom resources, and once created, expose their own API path (`/projects/{name}/control-plane`). Within each Project control plane, users can create standard Kubernetes resources such as Secrets and ConfigMaps.

Performance testing in this area has three main purposes:

1. **Scalability validation** – Ensure Milo can scale to a large number of Projects and Project resources without unacceptable latency or resource consumption.  
2. **Resource stress** – Measure how the API server and etcd behave under heavy load from Secrets/ConfigMaps churn (create, list, delete).  
3. **Garbage collection validation** – Confirm that cleanup of child resources when a Project is deleted is timely and does not leave orphaned objects.

We will explore two frameworks to drive this testing:

- **[ClusterLoader2 (CL2)](https://github.com/kubernetes/perf-tests/tree/master/clusterloader2)**: the upstream Kubernetes scalability and performance testing tool. It provides YAML-driven test definitions, standard measurements (API responsiveness, etcd metrics, resource usage), and integration with Prometheus.  
- **[k6](https://k6.io/)**: a modern load testing tool with a JavaScript-based DSL. It gives us flexibility for defining custom scenarios and is well-suited to exercising API endpoints directly, including Milo’s virtual API server paths.

The goal is to evaluate both, run representative workloads with each, and decide which is the better long-term fit for Milo’s perf testing.


## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this Enhancement.  Describe why the change is important and the benefits to users.
-->

Milo is intended to host many independent tenants through Projects. Each Project runs its own virtual API server surface area but shares the same backing etcd and controller stack. Before pushing toward production scale, we need data to answer:

- **How many Projects can Milo support concurrently?**  
- **How many Secrets/ConfigMaps per Project before etcd or API server usage becomes problematic?**  
- **What is the performance profile during resource churn (create → steady-state → delete)?**  
- **Does garbage collection of Project resources complete reliably under load?**

Without this testing, we risk regressions in latency, uncontrolled memory growth, and operational issues when cleaning up Projects.


### Goals

<!--
List the specific goals of the Enhancement. What is it trying to achieve? How will we
know that this has succeeded?
-->

- Automate creation of N Projects and high-volume Secrets/ConfigMaps workloads inside each Project.  
- Exercise deletion flows and verify cleanup of Project-scoped resources.  
- Collect metrics on API server responsiveness, availability, and etcd performance.  
- Provide repeatable developer workflows via Taskfile targets to run tests locally (kind) or against remote clusters (e.g., GKE).  
- Compare CL2 vs k6 in terms of:
  - Ease of expressing tests.  
  - Observability and built-in measurements.  
  - Ability to scale workloads.  
  - Integration into CI pipelines.  

### Non-Goals

<!--
What is out of scope for this Enhancement? Listing non-goals helps to focus discussion
and make progress.
-->

- Testing every type of Kubernetes object (we’ll focus on Projects + Secrets/ConfigMaps).  
- Full multi-tenant end-to-end validation (authn/authz, networking, etc.).  
- Benchmarking other Milo CRDs (Organizations, Users) beyond what is needed to bootstrap Projects.  


## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation. What is the desired outcome and how do we measure success?.
The "Design Details" section below is for the real
nitty-gritty.
-->

We will build test harnesses for both CL2 and k6:

1. **ClusterLoader2**  
   - Use modules to create/delete cluster-scoped `Organization` and `Project` objects.  
   - Run workloads in each Project control plane: Secrets/ConfigMaps create, steady-state, delete.  
   - Gather built-in CL2 measurements (APIAvailability, APIResponsiveness, EtcdMetrics, ResourceUsageSummary).  

2. **k6**  
   - Write scripts to directly call Milo’s management API (`/apis/resourcemanager.miloapis.com`) and virtual Project API endpoints (`/projects/{name}/control-plane`).  
   - Simulate user-like patterns (burst creation, list/get load, deletes).  
   - Capture latency percentiles and error rates with k6’s reporting.  

We’ll evaluate results from both and converge on one framework (or a hybrid approach) for Milo’s ongoing performance and scalability testing.  

Issue [#265](https://github.com/datum-cloud/enhancements/issues/265)
