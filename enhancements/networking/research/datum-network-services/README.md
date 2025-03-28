---
title: Datum Network Services
description: Research into types of network services Datum may help enable.
weight: 10
status: provisional
stage: alpha
latest-milestone: "v0.x"
---

## Summary

This document explores various network services and capabilities which Datum may
offer, covers at a high level Datum VPC Networks, and enumerates a number of
connectivity approaches that may be taken to connect networks together.

## Motivation

### Goals

- Enumerate networking use cases and traffic paths.
- Define expectations on Datum Network connectivity.
- Define one or more additional enhancements to drive feature development.
- Determine initial expectations to be placed on infrastructure providers.

### Non-Goals

- Defining implementation details of network context infrastructure providers or
  how they might integrate with Datum APIs.

## Glossary

TODO(jreese) fill out glossary

## Edge Services

### Datum Edge

Requirements:

- Client facing Anycast IP architecture for client connectivity
- IPs for connectivity to endpoints defined with public addresses.
- Ability to connect to endpoints contained within Datum VPC Networks, which may
  be enabled via various interconnect techniques.
- Direct server return (DSR) from Layer 7 infrastructure.

```mermaid
%%{
  init: {
    'theme':'neutral',
    'themeVariables': {
      'lineColor': '#000'
    }
  }
}%%
flowchart TD
  subgraph external-clients["External Clients"]
    client-a["Client"]@{ shape: circ }
  end

  external-anycast["Anycast IPs"]@{ shape: procs }

  subgraph public-endpoints["Public Endpoints"]
    public-endpoint-a["Endpoint"]@{ shape: circ }
  end

  subgraph datum-edge["Datum Edge"]
    direction TB
    datum-l4lb@{label: "Layer 4 Load Balancer", shape: procs}

    datum-dns@{label: "DNS", shape: procs}
    datum-l7lb@{label: "Layer 7 Load Balancer", shape: procs}

    datum-l4lb --> datum-dns
    datum-l4lb --> datum-l7lb
  end

  subgraph user-network["Datum VPC Network"]
    subgraph user-network-context["Network Context"]
      direction TB
      subgraph network-context-interconnects["Context Providers"]
        direction LR

        datum
        hyperscaler

        wireguard
        ipsec
        evpn
        vlan

        tailscale
        other

        datum ~~~ hyperscaler
        wireguard ~~~ ipsec
        %% ipsec ~~~ evpn
        evpn ~~~ vlan
        %% vlan ~~~ tailscale
        tailscale ~~~ other
      end

      subgraph private-endpoints["Private Endpoints"]
        private-endpoint-a["Endpoint"]@{ shape: circ }
      end

      network-context-interconnects --> private-endpoints
    end
  end

  style datum-edge fill:#63597B,color:#ffffff,stroke-width:2px,stroke:#312847;
  style user-network fill:#63597B,color:#ffffff,stroke-width:2px,stroke:#312847;
  style user-network-context fill:#F4A89B,stroke-width:2px,stroke:#F27A67;
  style network-context-interconnects fill:#FFF6D7,stroke-width:2px,stroke:#F2E6BC;

  client-a --> external-anycast
  external-anycast --> datum-edge

  datum-l7lb --> public-endpoints
  datum-l7lb --> user-network

  style external-clients fill:#E1EBEA
  style public-endpoints fill:#E1EBEA
  style private-endpoints fill:#E1EBEA



  linkStyle 0,1,6,7,8,9,10 stroke-width:2px;

```

### Layer 4 Load Balancer with Layer 7 Direct Server Return

A layer 4 load balancer operates at the transport layer of the [OSI Model][osi],
leveraging information such as source/destination IP addresses, and
source/destination ports, to steer traffic to upstream systems.  A load balancer
implementation may proxy connections, establishing its own set of connections
to upstream systems. In this mode, all response traffic must pass through the
load balancer, placing scaling pressure on the system. A common method to avoid
this pressure is to leverage Direct Server Return (DSR), where response traffic
is sent directly from upstream services through the network, bypassing the load
balancer.

[osi]: https://en.wikipedia.org/wiki/OSI_model

```mermaid
%%{init: {'theme':'neutral'}}%%
flowchart LR
    client[Client]
    subgraph datum-edge["Datum Edge"]
      datum-anycast[Datum Anycast]@{ shape: procs }
      datum-l4lb[Layer 4 Load Balancer]@{ shape: procs }
      datum-l7lb[Layer 7 Load Balancer]@{ shape: procs }
    end

    subgraph "Internet"
      public-endpoints[Endpoints]@{ shape: procs }
    end

    client -->|GET /| datum-anycast
    datum-anycast -->|GET /| datum-l4lb
    datum-l4lb -->|forwards| datum-l7lb
    datum-l7lb -->|GET /| public-endpoints
    public-endpoints -->|200 OK| datum-l7lb
    datum-l7lb -->|200 OK| client

    linkStyle 4,5 stroke-width:4px,stroke:green
```

A layer 4 load balancer may also implement advanced traffic distribution
mechanisms such as [Maglev Consistent Hashing][maglev], which can help address
lumpy traffic distribution challenges presented in the datacenter when [ECMP]
is used by downstream network devices. In addition, Maglev allows for the
addition and removal of load balancer replicas with minimal impact to existing
connections, greatly improving operational complexity.

In addition to consistent hashing capabilities, a load balancer may support
weighted traffic distribution which can be adjusted on the fly to allow for
either draining or ramping up of traffic on an individual upstream server basis.

#### Additional Reading

- [High Availability Load Balancers with Maglev](https://blog.cloudflare.com/high-availability-load-balancers-with-maglev/)
- [Introducing the GitHub Load Balancer](https://github.blog/engineering/introducing-glb/)
- [Hashing on broken assumptions](https://archive.nanog.org/sites/default/files/1_Saino_Hashing_On_Broken_Assumptions.pdf)

[maglev]: https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/
[ecmp]:https://en.wikipedia.org/wiki/Equal-cost_multi-path_routing

### Layer 7 Proxy to public endpoints

Requirements:

- Endpoints are accessible over the public internet from Datum's IP ranges.

Benefits:

- Not dependent on vendor interconnect features.

Drawbacks:

- Each individual endpoint must be issued a public IP address.

```mermaid
%%{init: {'theme':'neutral'}}%%
flowchart LR
    client[Client]
    subgraph "Datum Edge"
      datum-anycast[Datum Anycast]@{ shape: procs }
      datum-proxy[Datum Proxy Infra]@{ shape: procs }
    end
    subgraph "Infrastructure Provider"
      public-endpoints[Public Endpoints]@{ shape: procs }
    end

    client -->|GET /| datum-anycast
    datum-anycast -->|GET /| datum-proxy
    datum-proxy -->|GET /| public-endpoints
    public-endpoints -->|200 OK| datum-proxy
    datum-proxy -->|200 OK| client

    linkStyle 3,4 stroke-width:4px,stroke:green
```

### Layer 7 proxy to endpoints within Datum VPC Network

<!-- TODO(jreese) Expand on:

- How proxies are represented within the VPC network. Some options are:
  - "Proxy subnets" - subnets allocated in VPC specifically for Datum's use in
    proxy connectivity. Benefit being flexibility in allowing the user to define
    their desired subnet range.
  - Static range -->

```mermaid
%%{init: {'theme':'neutral'}}%%
flowchart LR
    client[Client]
    subgraph "Datum Edge"
      datum-anycast[Datum Anycast]@{ shape: procs }
      datum-proxy[Datum Proxy Infra]@{ shape: procs }
    end

    interconnects["Interconnects"]@{ shape: procs }

    subgraph "Datum VPC Network"
      private-endpoints[Private Endpoints]@{ shape: procs }
    end

    client -->|GET /| datum-anycast
    datum-anycast -->|GET /| datum-proxy
    datum-proxy -->|GET /| interconnects
    interconnects -->|Get /| private-endpoints
    private-endpoints -->|200 OK| interconnects
    interconnects -->|200 OK| datum-proxy
    datum-proxy -->|200 OK| client

    linkStyle 4,5,6 stroke-width:4px,stroke:green
```

## Datum VPC Networks

A Datum VPC Network is composed of one or more network contexts. The data plane
for each network context may vary between infrastructure providers, with each
requiring unique connectivity approaches.

Each network context may exchange routing and endpoint information with a cloud
router, enabling connectivity across data planes.

## Datum Cloud Router

A cloud router will be needed to orchestrate exchange of routes and endpoints,
as well as ensuring the various data planes are programmed.

Requirements:

- Must be distributed, no single point of failure or hub and spoke movement of
  data.
- Must be capable of deploying tunnel endpoints where necessary.

> Note: The providers in the following diagrams are demonstrational, and may not
> be implemented.

### Cloud Router Control Plane

The diagram below demonstrates an example Datum VPC network composed of three
network contexts. Each network context is enabled via different providers, which
each exchange routing and endpoint information with the Cloud Router control
plane. See the [Route Exchange](#route-exchange) and
[Endpoint Exchange](#endpoint-exchange) sections for more thoughts on this
topic.

```mermaid
%%{init: {'theme':'neutral'}}%%
flowchart BT

  subgraph legend
    direction LR
    a@{ shape: sm-circ}-->blank1["Control Plane Traffic"]
    linkStyle 0 stroke-width:4px,stroke:#F27A67
  end

  subgraph Datum Services
    cloud-router["Datum Cloud Router"]@{ shape: procs }
  end

  subgraph datum-vpc-network["Datum VPC Network"]
    subgraph context-a["Context A"]
      subgraph "Datum Provider"
        endpoint-a["Datum Endpoint"]
      end
    end

    subgraph context-b["Context B"]
      subgraph "Tailscale Provider"
        endpoint-c["Tailscale Endpoint"]
      end
    end

    subgraph context-c["Context C"]
      subgraph "GCP Provider"
        endpoint-b["GCP Endpoint"]
      end
    end

    context-a <--Route/endpoint exchange--> cloud-router
    context-b <--Route/endpoint exchange--> cloud-router
    context-c <--Route/endpoint exchange--> cloud-router

    linkStyle 1,2,3 stroke-width:4px,stroke:#F27A67
  end

```

### Cloud Router Data Plane

The diagram below demonstrates how network contexts may interconnect with the
Cloud Router through [many different methods](#connectivity-approaches). Data
plane components will be geographically distributed,
with [traffic steering capabilities](#traffic-policies) that can be leveraged
to optimize traffic or restrict which paths a packet may take.

```mermaid
%%{init: {'theme':'neutral'}}%%
flowchart BT
  subgraph legend
    direction LR
    b@{ shape: sm-circ}-->blank2["Data Plane Traffic"]
    linkStyle 0 stroke-width:4px,stroke:green
  end

  subgraph datum-platform["Datum Platform"]


    cloud-router["Datum Cloud Router"]@{ shape: procs }

    interconnect-native["Datum"]@{ shape: procs }
    interconnect-ipsec["IPSec"]@{ shape: procs }

    interconnect-native <--> cloud-router
    interconnect-ipsec <--> cloud-router
  end

  subgraph datum-vpc-network["Datum VPC Network"]
    subgraph context-a["Context A"]
      subgraph "Datum Provider"
        endpoint-a["Datum Endpoint"]
      end
    end

    subgraph context-b["Context B"]
      subgraph "GCP Provider"
        endpoint-b["GCP Endpoint"]
      end
    end

    subgraph context-c["Context C"]
      subgraph "Tailscale Provider"
        endpoint-c["Tailscale Endpoint"]
      end
    end

    context-a <--> interconnect-native
    context-b <--> interconnect-ipsec
    context-c <--> interconnect-native


    linkStyle 1,2,3,4,5 stroke-width:4px,stroke:green
  end
```

### Route Exchange

Routes must be exchanged between individual network contexts in order to enable
connectivity across them. A single network may be composed of network
contexts which have differing infrastructure providers, placing a requirement
on individual providers to integrate with the Cloud Router.

Route exchange methods:

- [BGP] - Major cloud providers as well as many network overlays support
  exchanging routes via BGP.
- Introduction of a `Route` into the Datum Networking APIs to enable policy
  based routing.

[bgp]: https://en.wikipedia.org/wiki/Border_Gateway_Protocol

### Endpoint Exchange

In addition to exchanging routes, it is advantageous to also exchange endpoints.
An endpoint represents a single address in the VPC network, which may be
associated with an instance, service, gateway, or other resources. Each endpoint
may be decorated with additional information such as the zone which it exists in,
and the condition of the endpoint.

Endpoint exchange methods:

- Integration with DNS discovery features available in various providers.
- The [EndpointSlice] type available in the Kubernetes API is currently the API
type which Datum is planning to leverage for storing endpoint information.

Input on additional or alternative approaches to exchanging endpoint information
is very welcome!

[EndpointSlice]: https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/

### Interconnect Negotiation

Network contexts may be accessible via an assortment of technologies which a
cloud router must be able to integrate with. In order to reduce the burden of
establishing connections across network contexts, orchestration software should
be written to automate these connections as much as possible.

Integration with a hyperscale cloud provider would require, at minimum:

- Requesting an IPSec tunnel endpoint from Datum.
- Creating Cloud HA VPN resources in GCP, connecting them to the IPSec tunnel.
- Exchanging routes as desired.
- Propagation of Endpoint information to the cloud router.

Integration with an overlay network such as Tailscale or NetBird may need to:

- Deploy a Datum Agent with access to the overlay network, for example, as a
  sidecar to tailscaled.
- Configure the agent to connect to the desired Datum VPC network.
- Ensure routes are exchanged as desired.
- Propagate Endpoint information to the cloud router.

TODO(jreese) fill in thoughts on orchestration of circuits / cross connects / etc

### Traffic Policies

Traffic policies enable users to control how packets traverse the network. Some
policies may be for cost concerns, while others may be to restrict which paths
may be taken for data control and compliance needs.

#### Hot vs cold potato routing

From Wikipedia's page on [Hot-potato routing][hot-potato-routing]:

> In Internet routing between autonomous systems which are interconnected in
> multiple locations, **hot-potato routing** is the practice of passing traffic
> off to another autonomous system as quickly as possible, thus using their
> network for wide-area transit. **Cold-potato routing** is the opposite, where
> the originating autonomous system internally forwards the packet until it is
> as near to the destination as possible.

[hot-potato-routing]: https://en.wikipedia.org/wiki/Hot-potato_routing

```mermaid
%%{init: {'theme':'neutral'}}%%
flowchart LR

  subgraph legend
    direction LR
    a@{ shape: sm-circ}-->blank1["High Priority Traffic"]
    linkStyle 0 stroke-width:4px,stroke:green

    b@{ shape: sm-circ}-->blank2["Low Priority Traffic"]
    linkStyle 1 stroke-width:4px,stroke:#F27A67
  end

  subgraph region A
    direction LR
    client

    subgraph Datum PoP A
      tunnel
    end

    client <--> tunnel
  end

  subgraph region B
    direction LR

    datum-pop-b["Datum PoP B"]

    subgraph public internet
      direction LR

      internet-a["Hop A"]
      internet-b["Hop B"]

      internet-a <--> internet-b
    end
  end

  subgraph region C
    direction LR

    subgraph public internet
      direction LR
      internet-c["Hop C"]
      internet-b <--> internet-c
    end

    datum-pop-c["Datum PoP C"]
  end

  tunnel <--"Cold Potato"--> datum-pop-b
  tunnel <--"Hot Potato"--> internet-a

  datum-pop-b <--> datum-pop-c
  internet-c <--> datum-pop-c

  linkStyle 5,7 stroke-width:4px,stroke:green
  linkStyle 3,4,6,8 stroke-width:4px,stroke:#F27A67
```

#### Path restriction

```mermaid
%%{init: {'theme':'neutral'}}%%
flowchart LR

  subgraph legend
    direction LR
    a@{ shape: sm-circ}-->blank1["Permitted Path"]
    linkStyle 0 stroke-width:4px,stroke:green

    b@{ shape: sm-circ}-->blank2["Prohibited Path"]
    linkStyle 1 stroke-width:4px,stroke:red
  end

  subgraph region A
    direction LR
    client

    subgraph Datum PoP A
      tunnel
    end

    client <--> tunnel

    exchange-a["IX A"]
  end

  subgraph region B
    direction LR

    exchange-b["IX B"]

    datum-pop-b["Datum PoP B"]

    exchange-a <--> exchange-b
    exchange-b <--> datum-pop-b
  end

  subgraph region C
    direction LR

    datum-pop-c["Datum PoP C"]
  end

  subgraph region D
    direction LR

    datum-pop-d["Datum PoP D"]
  end

  tunnel <--"Required Path"--> exchange-a
  tunnel x--"Prohibited Path"---x datum-pop-c

  datum-pop-b <--> datum-pop-d

  linkStyle 6 stroke-width:4px,stroke:red
  linkStyle 3,4,5,7 stroke-width:4px,stroke:green

```

## Connectivity Approaches

> Note: An additional deployment model exists for agents where they are
> colocated with an application, either on the same node or in the same network
> namespace. In this model, dependencies on influencing provider networking
> routes are removed, but a requirement to expose methods of integration with
> applications is added. This integration could take various forms, such as the
> insertions of routes into the application's network namespace, exposing
> forward proxies, etc. The requirement for each agent to have public
> connectivity remains, which could lead to increased costs.

### Agent-managed tunnel

Requirements:

- Agent has the ability to establish external connections.
- Agent must be able to produce packets on the provider network with a source
  address not bound to the agent's machine.
- Routes must be configured in the provider network or individual endpoints to
  direct traffic destined to subnets in the Datum VPC Network via an agent.

Benefits:

- Not dependent on vendor interconnect features.
- Advertised subnets accessible from peer network contexts.
- Agents can be geographically distributed.
- Individual agent tunnel throughput may scale with agent resource footprint.
- Potential for autoscaling agents.

Drawbacks:

- Must run infrastructure for one or more agents.
- Provider must permit disabling source IP checks on individual instances or
  network interfaces.
- Provider must support either injection of routes or permit instances to
  leverage layer 2 next hop to transit traffic via an agent tunnel.

#### Request flow

```mermaid
%%{init: {'theme':'neutral'}}%%
flowchart LR

  subgraph internet
    internet-client["client"]
  end

  subgraph Datum Edge
    ingress-services["Ingress"]
    vpc-network["VPC Network"]

    ingress-services <--> vpc-network
  end

  subgraph Infrastructure Provider
    agent-tunnel["Agent Tunnel"]
    private-endpoints-b["Private Endpoints"]
  end

  internet-client <--Internet--> ingress-services

  vpc-network <--Encapsulated Traffic--> agent-tunnel
  agent-tunnel <--Provider Network--> private-endpoints-b
```

#### Tools

- Tailscale (as a Subnet Router)
- Datum Agent

### Agent-managed tunnel with ingress masquerading

Requirements:

- Agent has the ability to establish external connections.

Benefits:

- Not dependent on vendor interconnect features.
- Does not require extending the provider network into the Datum Network.
- Agent does not need to be able to produce packets with source addresses that
  do not match those assigned to the instance.
- No additional routes need to be inserted into the provider network.
- Agents can be geographically distributed.
- Individual agent tunnel throughput can scale with agent resource footprint.
- Potential for autoscaling agents.

Drawbacks:

- Must run infrastructure for one or more agents.
- Loss of client IP information when not using the PROXY protocol.
- Cannot access endpoints within the provider network via the addresses those
  endpoints are configured with.
- Private endpoints cannot establish connections to endpoints behind proxy.
- Added complexity in endpoint discovery, requiring either some form of NAT, or
  specific registration of endpoints.

#### Request flow

```mermaid
%%{init: {'theme':'neutral'}}%%
flowchart LR

  subgraph internet
    internet-client["client"]
  end

  subgraph Datum Edge
    ingress-services["Ingress"]
    vpc-network["VPC Network"]

    ingress-services <--> vpc-network
  end

  subgraph Infrastructure Provider
    agent-tunnel["Agent Tunnel"]
    private-endpoints-b["Private Endpoints"]
  end

  internet-client <--Internet--> ingress-services

  vpc-network <--Encapsulated Traffic--> agent-tunnel
  agent-tunnel <--Proxied Traffic--> private-endpoints-b
```

#### Tools

- Tailscale
- Datum Agent

### Agent-managed tunnel with egress proxy enabled

Requirements:

- Agent has the ability to establish external connections.
- Agent exposes a SOCKS5 and/or HTTP proxy endpoint.

Benefits:

- Not dependent on vendor interconnect features.
- Does not require extending the provider network into the Datum Network.
- Does not require any changes to the provider network.
- Endpoints can individually choose to use the proxy or not.
- Endpoints can access entire Datum VPC Network, and egress to the internet, if
  permitted by policies.

Drawbacks:

- Must run infrastructure for one or more agents.

#### Request flow

```mermaid
%%{init: {'theme':'neutral'}}%%
flowchart LR

  subgraph Infrastructure Provider
    agent-proxy["Agent Forward Proxy"]
    private-endpoints-a["Private Endpoints"]
  end

  subgraph Datum Edge
    egress-services["Egress"]
    vpc-network["VPC Network"]

    vpc-network <--> egress-services
  end

  subgraph internet
    internet-endpoint["endpoint"]
  end

  private-endpoints-a <--Proxied Traffic--> agent-proxy
  agent-proxy <--Encapsulated Traffic--> vpc-network

  egress-services <--> internet-endpoint
```

### Platform Specific IPsec Tunnels

Requirements:

- Infrastructure provider created to orchestrate tunnels.
- Routes must be configured in the provider network or individual endpoints to
  direct traffic destined to subnets in the Datum VPC Network via the tunnel.

Benefits:

- Ability to leverage vendor specific capabilities.
- Entire advertised subnets accessible from Datum VPC Network.
- Tunnels can be geographically distributed.

Drawbacks:

- Individual tunnel performance may be limited, requiring multiple tunnels to
  meet throughput needs.
- Individual tunnels may result in static costs.

#### Request flow

```mermaid
%%{init: {'theme':'neutral'}}%%
flowchart LR

  subgraph internet
    internet-client["client"]
  end

  subgraph Datum Edge
    ingress-services["Ingress"]
    vpc-network["VPC Network"]

    ingress-services <--> vpc-network
  end

  subgraph Infrastructure Provider
    agent-tunnel["Tunnel Endpoint"]
    private-endpoints-b["Private Endpoints"]
  end

  internet-client <--Internet--> ingress-services

  vpc-network <--Encapsulated Traffic--> agent-tunnel
  agent-tunnel <--Provider Network--> private-endpoints-b
```
