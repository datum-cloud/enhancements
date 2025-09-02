# Resource Federation

> [!IMPORTANT] This document is in in DRAFT

- Need to be able to connect to multiple "source" control planes and propagate
  resources to a "cell" control plane.
- Must prevent collision of resource names across projects, as namespaces are
  not globally unique.
- Federator will interact with Edge clusters to manage resources.
- Make sure that controllers are built such that they are blind to federation
  being in place.
- Take care to ensure resource names and structure make sense, particularly for
  resources created by operators that drive platform changes.

## Design Details

- **Milo Core API Server**: Organization level stuff - projects / etc.
- **Milo Project API Server**: Project specific API server, for project scoped things.
- **Datum Platform API Server**:  API server that operators create resources in to satisfy
  the intent of resources defined in the Project APIs.
- **Federation API Server**: API servers that a “scheduler” will place resources in for
  tools like Karmada to sync out to edge clusters
- **Edge Control API Server**:  API servers that live in edge clusters. These are
  specifically disconnected from the underlying k8s control plane. Operators in
  the clusters will read from these API servers, and program the Edge Cluster
  API server if necessary (I say if necessary, because it might make sense to
  have a vcluster kinda thing - sorta getting into the weeds)
- **Edge Cluster API Server**: Actual API server for Kubernetes that kubelets & such will
  connect to. The “bottom layer”.

### Resource Propagation

- Plan for multiple federator deployments for fault isolation purposes
- Operator integrates with multiple upstream project control planes, creates
  resources in federation control plane which are required to fulfill
  expectations of upstream resources.
- Scheduler processes resources in federation API and schedules onto a target
  downstream cluster.
- Federation API server, possibly based on [KCP](https://www.kcp.io/),
  leveraging workspaces for each upstream project, and each downstream edge
  cluster.
- Federators are either off the shelf tools like Karmada, or custom tools.

Datum is targeting a [Cell-Based Architecture][aws-cell] for both the control
plane and data plane.

From AWS:

> Today, modern organizations face an increasing number of challenges related to
> resiliency, be they scalability or availability, especially when customer
> expectations shift to an always on, always available mentality. More and more,
> we have remote teams and complex distributed systems, along with the growing
> need for frequent launches and an acceleration of teams, processes, and
> systems moving from a centralized model to a distributed model. All of this
> means that an organization and its systems need to be more resilient than
> ever.

- **Control Plane**: API Servers, Operators, Resource definitions.
- **Data Plane**: Services such as network connectivity, workload instances,
  DNS, L4 and L7 proxies.

[aws-cell]: https://docs.aws.amazon.com/wellarchitected/latest/reducing-scope-of-impact-with-cell-based-architecture/reducing-scope-of-impact-with-cell-based-architecture.html

#### Target Architecture

```mermaid
flowchart LR

  subgraph core["Core Control Plane"]
    datum-core-api["Datum API"]@{shape: procs}
    project-operator["Project Operator"]@{shape: procs}

    datum-core-api <--> project-operator
  end

  subgraph upstream-cell-a["Control Plane Cell A"]
    subgraph project-control-planes-a["Projects (upstream)"]
      project-api-servers-a["Project A - API Server"]
      project-api-servers-b["Project B - API Server"]
    end

    operators-a["Operators"]@{shape: procs}

    subgraph platform["Datum Platform API Servers <br /><br /><br /><br /> (downstream)"]
      datum-apiserver-a["API Server"]@{shape: procs}
    end

    scheduler-a["Scheduler"]@{shape: procs}
  end

  project-operator --Creates--> project-control-planes-a & project-control-planes-b

  subgraph upstream-cell-b["Control Plane Cell B"]
    subgraph project-control-planes-b["Projects (upstream)"]
      project-api-servers-c["Project C - API Server"]
    end

    operators-b["Operators"]@{shape: procs}

    subgraph platform-b["Datum Platform API Servers <br /><br /><br /><br /> (downstream)"]
      datum-apiserver-b["API Server"]@{shape: procs}
    end

    scheduler-b["Scheduler"]@{shape: procs}
  end

  subgraph federation["Federation"]

    federation-apiserver-a["API Server"]@{shape: procs}
    federator-a["Federator A"]@{shape: procs}

    federation-apiserver-b["API Server"]@{shape: procs}
    federator-b["Federator B"]@{shape: procs}
  end

  datum-apiserver-a <--> scheduler-a
  datum-apiserver-b <--> scheduler-b

  scheduler-a <--> federation-apiserver-a & federation-apiserver-b
  scheduler-b <--> federation-apiserver-a & federation-apiserver-b

  subgraph datum-pop-a["Datum POP"]
    datum-pop-a-cell-a["Cell A"]
    datum-pop-a-cell-b["Cell B"]
  end

  subgraph datum-pop-b["Datum POP"]
    datum-pop-b-cell-a["Cell A"]
    datum-pop-b-cell-b["Cell B"]
  end

  project-api-servers-a & project-api-servers-b <--> operators-a
  project-api-servers-c <--> operators-b

  operators-a <--> datum-apiserver-a
  operators-b <--> datum-apiserver-b

  federation-apiserver-a <--> federator-a
  federation-apiserver-b <--> federator-b

  federator-a  <--> datum-pop-a-cell-a & datum-pop-b-cell-a
  federator-b <--> datum-pop-a-cell-b & datum-pop-b-cell-b
```

#### MVP Architecture

```mermaid
flowchart LR

  subgraph core["Core Control Plane"]
    datum-core-api["Datum API"]@{shape: procs}
    project-operator["Project Operator"]@{shape: procs}

    datum-core-api <--> project-operator
  end

  subgraph project-control-planes-a["Projects (upstream)"]
    project-api-servers-a["Project A - API Server"]
    project-api-servers-b["Project B - API Server"]
  end

  subgraph upstream-cell-a["Control Plane"]
      operators-a["Operators"]@{shape: procs}
  end

  project-operator --Creates--> project-control-planes-a

  subgraph platform["Datum Platform API Servers <br />(downstream)"]
    datum-apiserver-a["API Server"]@{shape: procs}
  end

  subgraph datum-pop-a["Datum POP"]
    datum-pop-a-cell-a["Cell A"]
  end

  subgraph datum-pop-b["Datum POP"]
    datum-pop-b-cell-a["Cell A"]
  end

  project-api-servers-a & project-api-servers-b <--> operators-a

  operators-a <--> datum-apiserver-a
  datum-apiserver-a <--> datum-pop-a-cell-a & datum-pop-b-cell-a
```
