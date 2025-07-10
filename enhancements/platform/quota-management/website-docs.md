# Quotas

## Background

Before beginning to leverage the various functionality that quota management
provides, it's important to understand some general information and context
around quotas and the role they play in the management and protection of
infrastructure within Datum Cloud.

### What are Quotas?

Quotas are specific limits that can be set on different Datum Cloud resources at
either organization or project scopes and act as guard-rails around the usage of
those resources. Examples include setting a limit on the number of projects that
can be created within an organization, or the number of workloads that can be
deployed within each project.

### Why are they important?

Enforcing quota limits on resource usage provides key benefits to everyone using
or managing the platform:

1.  **Stability**: Prevents resource exhaustion scenarios caused by accidental
    or intentional overuse of resources that could affect the stability and
    reliability of overall platform or organizational operations.
2.  **Observability**: Provides clear visibility into set quota limits and
    accurate accounting of current usage.
3.  **Predictability**: Enables proactive capacity planning and financial
    forecasting, while preventing scenarios that could lead to significant and
    unexpected infrastructure costs.

### What resources are quotable?

Initially, a few key resources are enabled for quota management and enforcement.
As functionality is added over time, more resources will be enabled to be
managed by quotas, providing protection across a wider range of deployed
infrastructure. See the [feature roadmap](#feature-roadmap) for details
regarding the initial set of quotable resources, and the plans for growing that
list over time with iterative releases.

## Working with Quotas

Interacting with the quota system involves three main steps.

### 1. Resource Registration

To make a resource type quotable, a platform administrator submits a
`ResourceRegistration` manifest. This informs the quota system about the
existence of the resource, its unit of measurement (e.g., "projects", "cpu
cores"), and potential dimensions for more granular quota control (e.g.,
location and instance-type).

```yaml
apiGroup: quota.miloapis.com
kind: ResourceRegistration
metadata:
  # {service}-{resource-type}-{allocation-or-entity-type}-registration
  name: resourcemanager-organization-project-registration
  namespace: org-abc
spec:
  # Reference to the owning service and kind that will create claims for the type being registered.
  # Refer to note in CRD description regarding phase 1 approach
  ownerRef:
    apiGroup: resourcemanager.miloapis.com
    kind: Organization
  # - "Entity": Declares a new resource entity (e.g., User, Project) 
  #   that must be counted for quota.
  # - "Allocation": Declares a resource unit (e.g., CPU, memory, storage) 
  #   that can be reserved or consumed within a parent entity.
  # - "Feature": Declares feature flags (future enhancement)
  type: "Entity"
  # Fully qualified name of the resource type being registered, as defined by the owning service.
  # This field is used in the matching pattern to locate associated ResourceGrants and ResourceClaims
  resourceTypeName: "resourcemanager.miloapis.com/Project"
  # Description of the quota limit for the resource type
  description: "Number of projects that can be created within an organization."
  # The base unit of measurement for the resource
  baseUnit: "projects"
  # The unit of measurement UI will display to users
  displayUnit: "projects"
  # Defines how to convert between the base unit and the display unit to instruct Cloud and Staff portals on 
  # how to display values
 # Formula: baseValue / unitConversionFactor = displayValue
  unitConversionFactor: 1
  # Dimensions that can be used in ResourceGrant selectors
  # for this resource type. These should be fully qualified resource type names
  # from the owning service's API group.
  dimensions: []
```

### 2. Grant Creation

Once a resource is registered, administrators can set limits by creating
`ResourceGrant`s. A grant specifies an amount of a resource type that is allowed
to be claimed within a certain scope. These grants are additive in nature.

The following example grants an organization an allowance for 3 projects:

```yaml
apiGroup: quota.miloapis.com
kind: ResourceGrant
metadata:
  name: default-project-count-grant
  namespace: org-abc
spec:
  # Reference to the organization this grant applies to
  ownerRef:
    apiGroup: resourcemanager.miloapis.com
    kind: Organization
    name: org-abc
  # List of allowances this grant contains
  allowances:
    # Default number of projects allowed in this organization
    - resourceTypeName: "resourcemanager.miloapis.com/Project"
      buckets:
        - amount: 3
          dimensionSelector: {}

```

### 3. Claim Creation and Evaluation

The quota management system uses two specific enforcement patterns during the
evaluation of a claim, depending on the `type` defined in the
`ResourceRegistration` spec that is tied to the claim (`Allocation` or
`Entity`). Currently, only the synchronous enforcement pattern is enabled.

#### Entity-Based Enforcement (Synchronous Claim Decisions)

For entity-based registrations, a synchronous enforcement mechanism is used, leveraging
a validating admission webhook that blocks the API request from being returned
to the client until the claim is evaluated, a decision is made, and the status
of the claim is updated.

Using the creation of a `Project` as an example:

1. **Initial Event**: A client sends a request for the creation of a new
    `Project` to the Milo APIServer, initiating the workflow.
2. **Claim Creation**: The APIServer invokes the validating webhook as part of
    the admission chain. The webhook creates a `ResourceClaim` in the cluster
    and blocks the request while it watches the claim for the final decision.
3. **Claim Detection**: The `ResourceClaim` controller (watching claims) detects
    the new object and begins reconciliation. 
4. **Claim Evaluation**: The controller evaluates the claim against the
   configured quota limit and current usage before updating the claim's `status`
   once a decision is made.
5. **Webhook Decision**: When the webhook detects the transitioned claim's
   `status` to a terminal state, it proceeds to return the decision to the
   APIServer.
6. **Final Outcome**: If the claim was granted, the `Project` is created in the
   APIServer; if denied, the `Project` isn't created. The client receives a
   response from the APIServer indicating the final outcome.

The following is an example that attempts to claim 3 Projects for an
Organization. Since the current limit is set to a maximum of 3 Projects for the
Organization, this claim will be granted by the system.

```yaml
apiGroup: quota.miloapis.com
kind: ResourceClaim
metadata:
  name: additional-project-count-claim
  # Must match the namespace of applicable ResourceGrants for this claim
  # and is used in the matching pattern to locate associated EffectiveResourceGrants
  namespace: org-abc

spec:
  # Reference to the owner of the claim request
  ownerRef:
    apiGroup: resourcemanager.miloapis.com
    kind: Organization
    name: org-abc
  requests:
    # Project entity count (must reference an active ResourceRegistration)
    - resourceTypeName: "resourcemanager.miloapis.com/Project"
      amount: 3
      dimensions: {}
```

## Feature Roadmap

The quota management system is being implemented in multiple iterations to
provide additional functionality and rapid delivery.

### Initial MVP Release

The initial MVP focuses on delivering core functionality of the system:
- Registration of various resource types with the system.
- Management of quota limits for registered resource types.
- Claiming resources through the implementation of the synchronous, entity-based
  enforcement pattern.
- Support for quota enforcement on the following resource types:
  - Maximum number of projects within an organization
  - Maximum number of deployed HTTP proxies within a project

### Future Releases

Future releases will build upon previously released functionality:
- Implementation of the asynchronous, allocation-based enforcement pattern.
- Enhanced logic supporting multi-dimensional constraints (e.g., location,
  instance type).
- Advanced UI features for quota visualization at specific dimensional levels.
- Support for enforcing quota limits on additional resource types

## FAQ

### What happens if I exceed set quota limits?

Clear feedback will be provided in the claim evaluation response, specifying
that the claim was denied and the reason why. The resource will not be created in the
control plane until quota is released (by deleting existing resources) or the
limit for that resource type is increased.

### How do I increase my quota limits?

<!-- TODO -->