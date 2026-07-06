# Kubernetes

**Bring Datum Cloud into your clusters with the Kubernetes APIs you already use.**

Kubernetes is how teams work with Datum Cloud without leaving their cluster
workflow — exposing services, wiring up connectivity, and more through standard
resources like the [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/)
and [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), with no
need to learn Datum-specific tooling. Today that experience is delivered by an operator that handles all
connectivity to Datum Cloud's edge network behind the scenes — no inbound firewall
rules, public IPs, or cluster exposure required. It's a first-class experience,
and this section collects the enhancements that shape it: how Datum integrates
with native Kubernetes APIs, how it scales, and how it grows alongside the
platform.

## Enhancements

- [`service-exposure/`](./service-exposure) — expose Kubernetes cluster services
  through Datum Cloud's edge network using standard Gateway and Ingress APIs,
  with outbound-only connectivity, no inbound cluster exposure, and automatic
  TLS on custom domains when DNS is hosted in Datum Cloud.

_New Kubernetes enhancements are added here as they're proposed._
