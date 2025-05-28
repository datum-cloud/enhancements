# Galactic VPCs - NPMS Overview

## High-Level Overview

The Network Performance Management System (NPMS) is a programmable, SLA-aware overlay that provides deterministic traffic routing between edge, core, and cloud environments. Built on Segment Routing over IPv6 (SRv6), WireGuard, and high-performance VPP routers, NPMS enables centralized control of path selection, observability, and performance optimization across distributed infrastructures.

This system is designed to serve multi-tenant environments where customers need application-specific traffic steering (e.g., ultra-low latency, lossless delivery, or high-throughput paths). All routing decisions are computed centrally and enforced via programmable sidecars, enabling proactive SLA enforcement and visibility across every link and tunnel in the system.

This document describes the architecture and technical design of a centralized Network Performance Management System (NPMS) responsible for probing, measuring, and programming an SRv6-based overlay network built on VPP routers. The system supports a multi-tenant environment, offers policy-based traffic steering (e.g., low latency, lossless, high throughput), and enables deterministic end-to-end path computation.

Edge nodes are securely connected to the core fabric via WireGuard tunnels, and all path computation and policy enforcement is performed out-of-band via a centralized controller. The system supports geographic path exclusions, synthetic bandwidth testing via iPerf, and configurable Forward Error Correction (FEC) per traffic class. A ten-minute control loop ensures consistent and predictable data plane updates across the fabric.

---

### Business Benefits

NPMS transforms connectivity from a fixed, opaque network into a programmable, real-time service layer. It enables:

- **Reduced Cost of Connectivity**  
  Replace MPLS or private circuits with SLA-aware paths over public infrastructure.

- **Policy-Driven Path Control**  
  Route traffic based on business needs: performance, compliance, or cost.

- **SLA-Based Productization**  
  Offer differentiated tiers like “low latency,” “zero loss,” or “geo-fenced” connectivity.

- **Multi-Cloud Reliability**  
  Optimize east-west and hybrid-cloud paths in real time.

---

### Why Developers Love It

For developers and infrastructure engineers, NPMS behaves like “network as code”:

- **Real-Time Network Observability**  
  Query latency, loss, and throughput metrics via API.

- **Declarative Control**  
  Route traffic per app or traffic class without BGP hacks.

- **Programmable Sidecars**  
  Integrate probing, policy, and telemetry into cloud-native pipelines.

---

### Why Executives Trust It

For technology leaders, NPMS delivers agility, visibility, and control:

- **Centralized Management**  
  Enforce policy and routing from a single controller.

- **SLA Accountability**  
  Monitor compliance and route around degraded infrastructure.

- **Multi-Tenant and Secure by Design**  
  Each customer operates in an isolated data plane environment.

---

NPMS brings the best of modern infrastructure—APIs, automation, real-time feedback loops—to network performance and delivery.

## Multi-Tenant Architecture

The NPMS is designed from the ground up to support a multi-tenant operating
model. Each tenant has isolated data plane resources and private performance
visibility, while sharing control plane logic and core network intelligence.

### Refefence Architectural Concept

Similar to MPLS BGP VPNs, NPMS is built upon a nearly state-free core network
built on SRv6 (similar in fashion to MPLS "P" routers) with a distributed L3
cloud router using BGP built at the edge (similar in fashion to MPLS "PE"
routers). Unlike MPLS BGP VPNs, routing and path information is forwarded to a
centrlaized controller for complete network topology computation and control.

### Shared Components

- **Control Plane**  
  The centralized controller, API service, database, and ingestion pipeline are shared across all tenants. This ensures efficient use of global resources and centralized decision-making.

- **Core-to-Core Performance Data**  
  Synthetic probing data between core nodes is shared. All tenants benefit from a global view of available core paths, and this data informs baseline segment path selection.

### Tenant-Isolated Components

- **Edge Probing and Measurement**  
  Edge-to-core and core-to-edge telemetry is isolated per tenant. Since edge deployments may be in overlapping or tenant-specific infrastructure, performance results are not assumed to be globally applicable.

- **Data Plane Instantiation**  
  Each tenant has their own fully isolated data plane:
  - A distinct set of VPP routers
  - Dedicated WireGuard tunnels
  - An exclusive SRv6 segment routing domain
  - Per-tenant routing and SLA policy profiles

- **L3 Cloud Router**
  We need to learn a tenant's L3 topology, therefore, each tenant receives an L3 control plane inside the tenant, used for route learning and advertisments.

- **Path Computation Engine**
  Each tenant receives a dedicated path computation engine. The path computation engine:
  - Ingests shared core to core performance data.
  - Ingests a tenant's edge to core and core to edge performance data.
  - Ingests a tenants routes.
  - Understands a tenants policy and SLA profiles.
  - Creates end to end deterministic routing parameters.

- **Policy and SLA Profiles**  
  Traffic classes, SLA preferences, geo-exclusions, and path scoring logic are scoped per tenant and applied independently to their network graph slice.

  Some example traffic classes that tenants can assign a class per tenant:
  - Best Effort for Lowest Cost: Prefer the path that is the lowest cost through the network.
  - Lowest latency: Prefer the lowest latency path through the network.
  - Zero packet loss: Prefer the path that has no packet loss.
  - Lossless with FEC: Prefer the path that has no packet loss, adding Forward Error Correction, at the expense of overhead for FEC.
  - Sustained Bandwidth: Prefer the path with validated available bandwith (using iPerf)

### Summary of Isolation

| Component                             | Shared / Per Tenant |
|--------------------------------------|----------------------|
| Control Plane Logic (API, DB, Engine) | Shared               |
| Core-to-Core Probe Data              | Shared               |
| Edge Probe Data                      | Per Tenant           |
| VPP Routers                          | Per Tenant           |
| WireGuard Tunnels                    | Per Tenant           |
| SRv6 Domain                          | Per Tenant           |
| L3 Cloud Router                      | Per Tenant           |
| Path Computation Engine              | Per Tenant           |
| Policy Profiles                      | Per Tenant           |

This design ensures strong tenant isolation at the data plane and SLA level, while leveraging shared control and observability infrastructure for scalability and efficiency.

## Probing and Topology Discovery

The system builds a real-time model of network performance through active synthetic probing. Every node (core or edge) is registered in a centralized database instance, tagged with metadata such as location, discovered network interfaces, network interface IP addresses, and role.

### Core Nodes

- Fully meshed active probing between all core sites
- Probe types include:
  - ICMP for reachability
  - UDP/TCP/HTTP for latency, jitter, and loss similating real user traffic patterns
  - Scheduled iPerf runs for bandwidth estimation

### Edge Nodes

- Each edge node connects to the core via at least three WireGuard tunnels
- Edge nodes determine upstream core nodes through initial triangulation probes; they query an IP with their current WAN IP address, Datum geolocates that IP address, and hands back 6x POP IPs. ICMP probes are used to determine the 3x best performing POPs.
- A direct WireGuard tunnel between edge sites enable baseline internet path testing

### Configuration Delivery

- Configs are polled every minute via a Go-based API service behind  CDN
- Instant Purge ensures rapid config propagation when policies change
- API responses are tenant-scoped and authenticated

## Telemetry Ingestion Architecture

### Sidecar Agents

- Each VPP node has a sidecar container that runs probes and gathers telemetry
- Metrics include RTT, loss, jitter, and iPerf-based bandwidth
- Core nodes probe all other core nodes
- Edge nodes probe all other related edge nodes, plus the 3x "upstream" connected POPs (via wireguard)
- Sidecars batch and transmit data to a dedicated ingestion service

### Ingestion Service

- Handles high-volume telemetry submission
- Performs validation, tagging, rate limiting, and aggregation
- Supports future integration with stream processing tools like Kafka or NATS

### Data Storage and Windowing

- Metrics are stored as time-series records
- Controller evaluates performance data in ten-minute windows
- This windowed model underpins SLA enforcement and routing decisions

## Customer Route Learning and Forwarding (BGP Integration)

To support dynamic route discovery from tenant networks, NPMS uses BGP at the edge to learn and advertise customer L3 routes. This mechanism is used solely for reachability exchange—**not** for determining forwarding paths.

### BGP for Customer-Facing Interfaces

Each tenant’s edge routers (VPP forwarding plane) runs BGP on interfaces connected to the customer’s network:

- Establishes BGP peering sessions with customer routers
- Learns customer prefixes (e.g., `10.0.0.0/24`)
- Advertises prefixes the overlay can reach on behalf of the tenant
- Operates within a tenant-specific ASN to enforce routing isolation
- As VPP does not provide a control plane function, we will need to evaluate a
  suitable BGP speaking daemon (examples include GoBGP, ExaBGP, FRR, Bird, etc.).

This BGP function is confined to edge nodes and is **not used** for internal core fabric routing.

### Separation of Reachability and Forwarding

Although the VPP router learns routes via BGP, **it does not autonomously install next-hops based on BGP**. Instead:

- A sidecar or local BGP daemon collects learned routes and submits them to the centralized controller
- The **controller** is responsible for determining:
  - Whether a path to the destination prefix exists
  - Which SLA-compliant path (segment list or tunnel) should be used
- The controller **programs explicit forwarding entries** into the VPP router
  - These entries override BGP-inferred next-hops with centrally computed paths

### Outbound Flow Behavior

1. Customer sends traffic to a remote destination.
2. VPP receives the traffic and performs a route lookup.
3. If the prefix exists, the controller-installed next-hop is used (e.g., SRv6 or WireGuard).
4. The packet is forwarded into the overlay using the correct SLA-compliant path.

### Inbound Flow Behavior

1. Customer prefixes are advertised by the edge VPP via BGP.
2. The controller collects these and distributes routing reachability knowledge across the tenant’s edge footprint.
3. Return traffic from remote edge sites is routed using paths selected by the controller.

This architecture ensures a flexible and policy-driven approach to overlay routing, while maintaining interoperability with existing customer routing systems.

## Overlay Network Programming

### WireGuard Edge Attachment

- Each edge has three or more WireGuard tunnels to diverse core sites
- Each edge also have wireguard tunnels to other edges, representing "the Internet"
- Tunnels are treated as individual interfaces in VPP
- Controller selects optimal ingress and egress tunnels based on telemetry

### Direct Edge-to-Edge Optimization

- Optional WireGuard tunnels between edge nodes allow for baseline testing
- If the internet path meets SLA criteria, the controller may select it over the overlay
- Use is policy-controlled per traffic class

### SRv6 Core Fabric

- Core VPP routers operate in an SRv6 domain
- No BGP or distributed routing protocols are used; all paths are centrally defined
- Segment lists are computed by the controller and downloaded to VPP nodes

### FEC-Enabled Transport

- Traffic classes may enable FEC for proactive loss recovery
- Sidecars manage packet duplication and deduplication at the endpoints
- FEC overhead is considered in bandwidth and path scoring

## Path Computation Engine

### Dynamic Network Graph

- Controller builds a graph with nodes (VPP instances) and edges (observed links)
- Each edge is weighted based on active probe metrics

### SLA-Driven Scoring Functions

- Tenants assign a mode per traffic class:
 - Best Effort for Lowest Cost: Prefer the path that is the lowest cost through the network.
  - Lowest latency: Prefer the lowest latency path through the network.
  - Zero packet loss: Prefer the path that has no packet loss.
  - Lossless with FEC: Prefer the path that has no packet loss, adding Forward Error Correction, at the expense of overhead for FEC.
  - Sustained Bandwidth: Prefer the path with validated available bandwith (using iPerf)
- Each mode maps to a scoring profile that converts probe metrics into graph weights

### Constraints and Filtering

- Graph is pruned or weighted based on:
  - Tenant geographic exclusions
  - Bandwidth thresholds
  - Hop count limits

### Routing Algorithms

- Dijkstra (or K-shortest path variants) used to compute segment lists
- Optional fallback logic:
  - Try secondary SLA profile
  - Use best-effort path and notify tenant
  - Flag unmet SLA via API or alert

## VPP and Sidecar Integration

### VPP Router Responsibilities

- Forward traffic via SRv6 or WireGuard interfaces
- Maintain static tunnel interfaces and apply centrally-issued segment lists
- Apply updates every ten minutes via config polling

### Sidecar Agent Responsibilities

- Execute all probing tasks
- Poll for config from Fastly-backed API
- Report telemetry to ingestion service
- Manage FEC and deduplication logic if required

### Policy Sync and Control Loop

- Each VPP node pulls a full config bundle every ten minutes
- Late updates are rejected to maintain state consistency
- Config includes segment lists, tunnel usage policies, FEC toggles, and SLA mode definitions

## Observability and Monitoring

NPMS includes built-in observability features to ensure that developers, operators, and tenants can monitor network performance in real-time and over time. Observability is scoped per tenant and integrates seamlessly with SLA and policy logic.

### Core Observability Capabilities

- **Real-Time Metrics**  
  - Per-path: latency, jitter, packet loss, bandwidth (iPerf)
  - Exported via Prometheus-compatible endpoints
  - Queryable per traffic class and per node pair

- **SLA Monitoring**  
  - Automatic detection of SLA violations
  - Root-cause hints (e.g., “loss >1% on segment Core 3 → Core 7”)
  - Alert hooks via webhook or API

- **Graph Views and Visualizations**  
  - Real-time graph view of active network topology per tenant
  - Segment overlays for SRv6 path inspection
  - Tunnel status and availability

- **Historical Analysis**  
  - SLA compliance reports over time
  - Visualization of path changes and churn reasons
  - Exportable telemetry logs for audits or debugging

- **APIs and Integrations**  
  - REST API for querying metrics and paths
  - Webhooks for SLA violation or path reprogram events
  - Prometheus metrics export for use in Grafana or other tools

These observability features ensure that the system remains transparent, debuggable, and trusted—critical for both developers and executives.

### Future Ideas

This document lays down a basis for a Galactic VPC - effectively a "backbone as
a service." We have envisioned the concept and initial behaviour of core nodes
and edge nodes. Future work for us to consider:

- How does traffic arrive at a Galactic VPC from a Datum Gateway (Anycast Envoy
  Proxy)?
  How can traffic egress from a Galactic VPC to the Internet (e.g. Datum CGNAT)?
  How can traffic ingress/egress from other application providers (e.g. AWS
  PrivateLink)?
  How can traffic ingress/egress from other physical networks via cross
  connects, IXPs, NNIs, or Network as a Service Providers?

## Glossary

- **API Service**: HTTP interface for sidecar config polling and tenant policy updates
- **CockroachDB**: Distributed SQL store for topology, telemetry, and policies
- **Edge Node**: A VPP router deployed adjacent to a customer's workload
- **FEC**: Forward Error Correction: Adds redundancy to recover from packet loss without retransmission
- **iPerf**: Tool used to measure available bandwidth
- **Overlay Network**: Path-controlled fabric using SRv6 and WireGuard
- **Path Computation Engine**: Computes best paths using performance graph and policies
- **Probing**: Active synthetic measurements between nodes
- **Sidecar**: Container handling telemetry, probing, and config sync
- **SRv6**: Segment Routing over IPv6
- **Tenant**: A customer with defined SLA and policy
- **Traffic Class**: A category of traffic with a specific SLA
- **Telemetry**: Time-series data collected from probes
- **VPP**: Vector Packet Processing, the high-performance data plane
- **WireGuard**: VPN protocol for tunneling over the public internet

# Business Benefits & Messaging Strategy for NPMS

## Business Benefits of the Network Performance Management System (NPMS)

### 🔐 1. Improved Application Performance and Reliability
- Deliver deterministic, SLA-backed paths for critical applications
- Guarantee low latency or lossless delivery per traffic class
- Ensure high bandwidth or FEC-assisted delivery for bulk and sensitive flows

### 🌐 2. Geopolitical Routing Control
- Enforce customer-specific regional exclusions (e.g., "never traverse India")
- Comply with data sovereignty and privacy requirements
- Enable secure routing for regulated industries (finance, healthcare)

### 🧩 3. Multi-Cloud Connectivity with Predictable Performance
- Seamlessly connect AWS, Azure, GCP, and on-prem environments
- Use public internet paths with private-network-like assurance
- Optimize per flow using real-time performance data

### 💸 4. Reduce MPLS or Private Backbone Spend
- Avoid costly telco contracts by leveraging smart overlay routing
- Deliver SLA-driven performance over commodity paths
- Reduce network TCO while improving end-user experience

### ⚙️ 5. Self-Healing, Programmable Network Infrastructure
- Central controller continuously evaluates and reprograms paths
- Autonomous reaction to latency spikes, jitter, or loss
- Configuration via APIs, ideal for DevOps and NetOps integration

### 🧾 6. Tenant-Aware, Service-Based Billing
- Productize connectivity: offer tiers like "Ultra Low Latency", "Lossless", or "Max Bandwidth"
- Meter performance per tenant, per class, per region
- Monetize premium network experiences

---

## Messaging Strategy

### 👩‍💻 For Software Developers and Network Engineers

> "This is the network as code. Think: a programmable substrate that reacts in real time to conditions like loss, latency, or bandwidth bottlenecks."

**Key Selling Points:**
- Full API control
- Modern tech stack (SRv6, WireGuard, VPP, sidecars)
- Real-time observability and control
- Integrates with CI/CD, infra-as-code, and cloud-native tools

**Phrases to use:**
- “No more manually tuning BGP.”
- “Your app declares what it needs—then the network figures it out.”
- “Programmable reliability, as a service.”

---

### 👨‍💼 For Technology Executives and Infrastructure Leaders

> "This gives your teams centralized visibility and control over global application delivery. You’ll reduce outages, cut telco costs, and move faster—without compromising user experience or SLA."

**Key Selling Points:**
- Strategic control of all network paths across clouds
- Real-time SLA enforcement
- Reduced complexity and operational cost
- Differentiation through superior performance guarantees

**Phrases to use:**
- “End-to-end control of application traffic across all clouds.”
- “Deliver better performance than MPLS for a fraction of the cost.”
- “Multi-tenant ready and ready to productize.”

**Service Plane features:**

| Feature name | Description |  Demo  | Use Case | 
| :---- | :---- | :---- | :----|
| L3VPN | Layer 3 VPN | Prefix reachability and shoot packets to private prefixes| NaaS providers |
| Deterministic routing | Applications/services drive path selection with their specifications/profiles | Run a ping with different DSCP codepoints that take different paths in the network | Real-time inference |
| Partner Peering| Private and encrypted connectivity with partners across VPCs | Get routes via APIS from AWS and GCP Leak routes between partner routing tables, compute paths and shoot packets | Replace PTP MPLS connectivity with IP connectivity with all the features of MPLS L3VPN including QoS and minus the cost and complexity |  
| Provider peering | Private and encrypted connectivity between providers for B2B networking across Provider VPCs| Create shared routing tables between 2 providers and provide route exchange, installation and traffic on different types of paths between then them, traffic generation is via APIs between providers | Allow for secure B2B connectivity for Agent2Agent and API access
|Global span | Multi-cloud secured connectivity across Geographies between AWS and GCP VPCs | Get Routes using APIs, Exchange routes, compute paths, Shoot Ipv4 packets between GCP and AWS using private overlapping prefixes | MPLS like private connectivity at lower cost with QoS |
|Customer connect | Access Provider VPC via private, encrypted connectivity using private attachment addresses from provider IP pool |  Private address from provider allocated to customer attachment | No attackable, visible public addresses ever|
|Rate-limiting, policing| Allow rate-limiting and policing policies to be applied on Edge | use iPerf and drop packets on ingress to Datum Edge at predefined limits| Customers cannot exceed specific consumption limits set for traffic classes|  
|Dashboard - Observability| Be able to view L3 prefixes exchanged and calculated paths and their classes for those prefixes | Show L3 prefixes being exchanged and paths computed per-VPC and across VPC peerings | Give customers deep insight into path computation,selection and metrics |
|Traceability| L3-L7 probes for tracing traffic to different prefixes | show L3-L7 behavior using per-packet/session/connection/transaction traceability|
|Debuggability| Packet traces and per-hop treatment of packets for different classes of traffic | Collect packet traces and render them in wireshark |


---

