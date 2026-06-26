# Kubernetes Operator

**Datum Cloud connectivity, the Kubernetes-native way.**

The Kubernetes Operator is how teams bring Datum Cloud into their clusters
without leaving the Kubernetes workflow. It speaks the standard Gateway API and
Ingress, so developers expose services with the resources they already know,
while the operator handles all connectivity to Datum Cloud's edge network behind
the scenes — no inbound firewall rules, public IPs, or cluster exposure required.
It's a first-class experience, and this section collects the enhancements that
shape it: how it integrates, how it scales, and how it grows alongside the
platform.

## Enhancements

- [`service-exposure/`](./service-exposure) — expose Kubernetes cluster services
  through Datum Cloud's edge network using standard Gateway and Ingress APIs,
  with outbound-only connectivity, no inbound cluster exposure, and automatic
  TLS on custom domains when DNS is hosted in Datum Cloud.

_New Kubernetes Operator enhancements are added here as they're proposed._
