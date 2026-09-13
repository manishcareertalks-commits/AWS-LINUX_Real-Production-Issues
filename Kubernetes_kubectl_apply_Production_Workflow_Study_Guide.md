# Kubernetes `kubectl apply` Workflow --- Production & Interview Study Guide

<img width="965" height="837" alt="image" src="https://github.com/user-attachments/assets/1121c294-c421-4a66-b497-8c83dbc62fb9" />

**Question:**

> What happens in the backend when you execute  `kubectl apply -f deployment.yaml`?

A strong interview answer should explain the request lifecycle from the
developer/DevOps engineer's terminal through the Kubernetes Control
Plane and finally to the Worker Node where the Pod runs.

The simplified flow is:

``` text
User
  |
  | kubectl apply -f deployment.yaml
  v
kubectl
  |
  | kubeconfig + API request
  v
kube-apiserver
  |
  +--> Authentication
  |
  +--> Authorization
  |
  +--> Validation / Admission
  |
  v
etcd
  |
  v
Controllers
  |
  v
Scheduler
  |
  v
Worker Node
  |
  v
kubelet
  |
  v
Container Runtime
  |
  v
Pod
  |
  v
Application Container
```

> **Important:** This is a simplified mental model. Kubernetes is
> asynchronous and several components continuously watch and reconcile
> state rather than executing one long synchronous chain.

------------------------------------------------------------------------

# 2. Sample `deployment.yaml`

Use this simple Deployment for understanding the workflow:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
```

Execute:

``` bash
kubectl apply -f deployment.yaml
```

This YAML tells Kubernetes:

-   Create a Deployment named `nginx-deployment`
-   Maintain **3 replicas**
-   Create Pods with the label `app: nginx`
-   Run the `nginx:1.27` container image
-   Expose container port `80` inside the Pod

The important concept is:

> **The YAML describes the desired state. Kubernetes works continuously
> to make the actual state match that desired state.**

------------------------------------------------------------------------

# 3. Step 1 --- `kubectl` Starts the Request

When the engineer executes:

``` bash
kubectl apply -f deployment.yaml
```

`kubectl` first needs to understand:

1.  What resource is being submitted?
2.  Which Kubernetes cluster should receive the request?
3.  Which API Server should be contacted?
4.  Which identity/credentials should be used?

`kubectl` parses the YAML and prepares a Kubernetes API request.

`kubectl` itself does **not** create the Pod directly on a Worker Node.

Instead:

``` text
kubectl
   |
   v
Kubernetes API Server
```

The API Server becomes the entry point into the cluster.

------------------------------------------------------------------------

# 4. Step 2 --- kubeconfig

Before contacting the cluster, `kubectl` uses its kubeconfig
configuration.

Common location:

``` bash
~/.kube/config
```

The kubeconfig commonly contains information about:

-   Clusters
-   API Server endpoints
-   Users / authentication configuration
-   Contexts
-   Current context

Useful command:

``` bash
kubectl config current-context
```

To inspect configuration:

``` bash
kubectl config view
```

To see available contexts:

``` bash
kubectl config get-contexts
```

Mental model:

``` text
kubectl
   |
   +--> kubeconfig
           |
           +--> Which cluster?
           +--> Which API Server?
           +--> Which identity?
```

### Production Note

In production environments, authentication can be integrated with cloud
IAM, certificates, OIDC, identity providers, or other mechanisms
depending on the Kubernetes platform.

------------------------------------------------------------------------

# 5. Step 3 --- Request Reaches kube-apiserver

`kubectl` sends the API request to the Kubernetes **API Server**.

The API Server is the primary entry point for Kubernetes API operations.

Other Kubernetes components also communicate through the Kubernetes API.

Conceptually:

``` text
kubectl
   |
   v
+------------------+
|   API Server     |
+------------------+
       |
       +--> etcd
       |
       +--> Controllers
       |
       +--> Scheduler
       |
       +--> Other Kubernetes clients
```

### Important Interview Point

Do not say:

> "`kubectl` directly talks to the kubelet."

For the normal Kubernetes API workflow, `kubectl` communicates with the
**API Server**.

------------------------------------------------------------------------

# 6. Step 4 --- Authentication

The API Server needs to determine:

> **Who are you?**

This is **authentication**.

Authentication establishes the identity associated with the request.

Examples of authentication mechanisms can include:

-   Client certificates
-   Bearer tokens
-   OIDC
-   Cloud-provider authentication integrations
-   Other configured authentication mechanisms

Mental model:

``` text
Authentication
       |
       v
"Who are you?"
```

### Authentication vs Authorization

Authentication:

> **Who are you?**

Authorization:

> **What are you allowed to do?**

These are different security stages.

------------------------------------------------------------------------

# 7. Step 5 --- Authorization

After authentication, Kubernetes needs to determine:

> **Is this identity allowed to perform this action?**

For example:

``` bash
kubectl apply -f deployment.yaml
```

may result in an operation similar to:

``` text
User/Identity
    |
    +--> create/update Deployment
    |
    +--> Namespace: production
```

Kubernetes authorization mechanisms determine whether that action is
permitted.

A very common mechanism is **RBAC --- Role-Based Access Control**.

Useful commands:

``` bash
kubectl auth can-i create deployments
```

For a namespace:

``` bash
kubectl auth can-i create deployments -n production
```

Mental model:

``` text
Authentication
     |
     v
Who are you?
     |
     v
Authorization
     |
     v
Are you allowed?
```

------------------------------------------------------------------------

# 8. Step 6 --- Validation

Kubernetes validates the incoming object.

Examples of validation include checking:

-   API version
-   Resource kind
-   Required fields
-   Field types
-   Resource structure
-   Whether submitted values are valid for the resource

For example:

``` yaml
replicas: 3
```

is structurally different from:

``` yaml
replicas: "three"
```

The API Server validates whether the object conforms to the Kubernetes
API.

------------------------------------------------------------------------

# 9. Step 7 --- Admission Control

After authentication, authorization and request validation, Kubernetes
can apply **admission control**.

Admission controllers can:

-   Reject requests
-   Mutate requests
-   Apply policies
-   Enforce organizational requirements
-   Validate security or governance rules

Examples of production policies may include:

-   Required labels
-   Security restrictions
-   Image policies
-   Namespace policies
-   Resource requirements
-   Organization-specific controls

A simplified view:

``` text
Request
   |
   v
Authentication
   |
   v
Authorization
   |
   v
Validation
   |
   v
Admission
   |
   v
Accepted Object
```

> Exact internal processing details can vary by Kubernetes version and
> configuration, so this should be treated as an interview-level
> lifecycle model rather than an implementation trace.

------------------------------------------------------------------------

# 10. Step 8 --- Desired State Is Persisted in etcd

Once the API request is accepted, Kubernetes persists cluster state in
**etcd**.

`etcd` is a distributed, consistent key-value store used by Kubernetes
for cluster state.

It stores Kubernetes API data such as:

-   Resource configuration
-   Desired state
-   Metadata
-   Cluster objects and their state

For our example:

``` yaml
replicas: 3
```

represents the desired state of the Deployment.

Important:

> **etcd is not the database used by the application's business logic.**

For example, an e-commerce application may use PostgreSQL, MySQL,
DynamoDB, MongoDB, etc. That is separate from Kubernetes' etcd.

------------------------------------------------------------------------

# 11. Why etcd Is Critical in Production

etcd is extremely important because Kubernetes depends on its cluster
state.

A simplified concept:

``` text
Kubernetes API State
        |
        v
      etcd
```

Production considerations include:

-   High availability
-   Backup
-   Restore testing
-   Monitoring
-   Storage performance
-   Secure communication
-   Access control

A common operational principle is:

> **If you operate your own Kubernetes control plane, protect and
> regularly back up etcd according to your recovery strategy.**

Managed Kubernetes services may handle much of the control-plane
infrastructure for you.

------------------------------------------------------------------------

# 12. Step 9 --- Controllers Observe Desired State

After the Deployment exists in the Kubernetes API, controllers
continuously work toward the desired state.

For our example:

``` text
Desired State:
3 Pods
```

Suppose:

``` text
Actual State:
0 Pods
```

The Kubernetes controllers detect the difference and take action to move
the cluster toward:

``` text
Actual State → Desired State

0 Pods → 3 Pods
```

This process is called **reconciliation**.

------------------------------------------------------------------------

# 13. Deployment → ReplicaSet → Pods

A Deployment does not normally create the Pods directly.

A simplified relationship is:

``` text
Deployment
     |
     v
ReplicaSet
     |
     v
Pods
```

For:

``` yaml
replicas: 3
```

the Deployment controller works with a ReplicaSet so that the desired
number of Pods is maintained.

Mental model:

``` text
Deployment
   |
   | "I want 3 replicas"
   v
ReplicaSet
   |
   | "Maintain 3 Pods"
   v
Pods
```

### Production Importance

If one Pod crashes or disappears:

``` text
Desired = 3
Actual  = 2
```

Kubernetes works to restore:

``` text
Actual = 3
```

This is one of the most important Kubernetes concepts:

> **Kubernetes is continuously reconciling desired state with actual
> state.**

------------------------------------------------------------------------

# 14. Step 10 --- Scheduler Selects a Worker Node

When a new Pod does not yet have a node assigned, the **kube-scheduler**
looks for a suitable Worker Node.

The scheduler considers scheduling constraints such as:

-   CPU requests
-   Memory requests
-   Node selectors
-   Node affinity
-   Pod affinity
-   Pod anti-affinity
-   Taints and tolerations
-   Topology constraints
-   Other scheduling requirements

Conceptually:

``` text
Pod
 |
 | "I need a suitable node"
 v
Scheduler
 |
 +--> Worker Node 1
 +--> Worker Node 2
 +--> Worker Node 3
        |
        v
   Selected Node
```

### Important Interview Point

The Scheduler decides:

> **WHERE should the Pod run?**

It does not:

-   Start the container
-   Pull the container image
-   Execute the application

------------------------------------------------------------------------

# 15. Step 11 --- kubelet Takes Responsibility

Once the Pod is assigned to a Worker Node, the kubelet on that node is
responsible for making sure the Pod is running according to its
specification.

The kubelet is the primary node agent.

Simplified:

``` text
Scheduler
    |
    | Pod assigned to Node-2
    v
Node-2
    |
    v
kubelet
```

The kubelet works with the container runtime to ensure the required
containers are running.

### Important Interview Distinction

Remember:

``` text
Scheduler = WHERE?
kubelet   = Make sure the Pod runs there
```

------------------------------------------------------------------------

# 16. Step 12 --- Container Runtime

The kubelet interacts with the container runtime through the Kubernetes
**Container Runtime Interface (CRI)**.

A common runtime is:

``` text
containerd
```

The runtime is responsible for container lifecycle operations such as:

-   Creating containers
-   Starting containers
-   Stopping containers
-   Removing containers
-   Managing container images as required

Simplified:

``` text
kubelet
   |
   v
Container Runtime
   |
   v
Container
```

### Important Accuracy Point

Do not describe kubelet as the container runtime.

They have different responsibilities:

``` text
kubelet
  = Node agent

containerd
  = Container runtime
```

------------------------------------------------------------------------

# 17. Step 13 --- Pod Is Created and Application Container Runs

The container runtime starts the container inside the Pod.

Our final state becomes approximately:

``` text
Worker Node
    |
    +--> kubelet
    |
    +--> containerd
    |
    +--> Pod
           |
           +--> nginx container
```

For our Deployment:

``` yaml
replicas: 3
```

Kubernetes aims to maintain:

``` text
Pod 1 → nginx
Pod 2 → nginx
Pod 3 → nginx
```

------------------------------------------------------------------------

# 18. Complete End-to-End Workflow

The complete simplified workflow is:

``` text
                 USER / DEVOPS ENGINEER
                           |
                           |
             kubectl apply -f deployment.yaml
                           |
                           v
                      kubectl
                           |
                           v
                      kubeconfig
                           |
                           v
                    +-------------+
                    | API SERVER  |
                    +-------------+
                           |
                    Authentication
                           |
                    Authorization
                           |
                  Validation/Admission
                           |
                           v
                       +------+
                       | etcd |
                       +------+
                           |
                           v
                     Controllers
                           |
                           v
                      Deployment
                           |
                           v
                      ReplicaSet
                           |
                           v
                      Unschedulable
                           |
                           v
                      Scheduler
                           |
                           v
                   Selected Worker Node
                           |
                           v
                        kubelet
                           |
                           v
                    containerd/runtime
                           |
                           v
                          Pod
                           |
                           v
                    nginx Container
```

This diagram is intentionally simplified. Kubernetes controllers,
scheduler, kubelet and other components operate through the API and
watch/reconcile state asynchronously.

------------------------------------------------------------------------

# 19. A More Accurate Mental Model

Instead of thinking:

``` text
kubectl
  ↓
API Server
  ↓
Scheduler
  ↓
kubelet
```

as one synchronous request, think:

``` text
             Desired State
                   |
                   v
              API Server
                   |
                   v
                 etcd
                   |
          +--------+--------+
          |                 |
          v                 v
     Controllers        Scheduler
          |                 |
          |                 v
          |          Pod gets node
          |                 |
          +--------+--------+
                   |
                   v
                kubelet
                   |
                   v
            Container Runtime
                   |
                   v
                  Pod
```

Kubernetes is fundamentally a **declarative and reconciliation-based
system**.

You declare what you want.

Kubernetes continuously works to make the actual environment match that
desired state.

------------------------------------------------------------------------

# 20. What Happens if the Pod Fails?

Suppose the Deployment requires:

``` text
replicas: 3
```

Current state:

``` text
Pod-1 = Running
Pod-2 = Running
Pod-3 = Running
```

Now Pod-2 crashes.

Actual state becomes:

``` text
2 healthy/running Pods
```

Desired state remains:

``` text
3 replicas
```

Kubernetes controllers detect the difference and work to restore the
desired state.

Conceptually:

``` text
Desired = 3
Actual  = 2

        ↓

Reconciliation

        ↓

Create/recover workload

        ↓

Desired = 3
Actual  = 3
```

This is why Kubernetes is called a self-healing platform in many common
scenarios.

------------------------------------------------------------------------

# 21. What Happens if a Worker Node Fails?

Suppose:

``` text
Node-1 → Pods
Node-2 → Pods
Node-3 → Pods
```

If Node-2 becomes unavailable, Kubernetes detects the node condition
through its control-plane mechanisms.

The exact recovery behavior depends on workload configuration and
cluster conditions, but controllers can work to recreate required Pods
on suitable available nodes.

The important interview concept is:

> **The desired workload state is maintained independently of one
> individual Worker Node.**

However, recovery is not instantaneous and depends on node failure
detection, scheduling constraints, available capacity, storage, topology
rules and workload configuration.

------------------------------------------------------------------------

# 22. Where Do Services Fit?

The `Deployment` example creates Pods, but it does not automatically
create a Kubernetes Service.

If you also create:

``` yaml
kind: Service
```

the Service provides a stable virtual networking abstraction for
reaching backend Pods.

A simplified application architecture becomes:

``` text
Client
   |
   v
Service
   |
   +----> Pod
   |
   +----> Pod
   |
   +----> Pod
```

Service traffic is related to Kubernetes networking and may involve:

-   kube-proxy
-   CNI/network plugin
-   Service implementation
-   EndpointSlice objects

This is separate from the basic `kubectl apply` request lifecycle.

------------------------------------------------------------------------

# 23. Where Does kube-proxy Fit?

`kube-proxy` runs on Worker Nodes in many Kubernetes installations and
helps implement Kubernetes Service networking.

Simplified:

``` text
Client
   |
   v
Service
   |
   v
Service networking
   |
   v
Backend Pods
```

Important:

> **kube-proxy is not the entire Kubernetes networking system.**

Modern Kubernetes networking also depends heavily on the cluster's
CNI/network plugin.

Do not oversimplify this in a production interview.

------------------------------------------------------------------------

# 24. Readiness and Liveness Probes

Once the container starts, application health can be checked using
probes.

### Readiness Probe

Answers:

> Is this Pod ready to receive traffic?

If readiness fails, the Pod may remain running but should not receive
Service traffic through normal endpoint selection.

### Liveness Probe

Answers:

> Is this container healthy enough to continue running?

If a liveness probe repeatedly fails, Kubernetes can restart the
container depending on the configuration.

Example:

``` yaml
readinessProbe:
  httpGet:
    path: /health
    port: 80
  initialDelaySeconds: 5
  periodSeconds: 10
```

These probes are extremely important in production because:

> **A running container is not necessarily a ready application.**

------------------------------------------------------------------------

# 25. `kubectl apply` vs `kubectl create`

`kubectl apply` is commonly used for declarative configuration
management.

For example:

``` bash
kubectl apply -f deployment.yaml
```

The configuration describes the desired state.

If the configuration is changed:

``` yaml
replicas: 5
```

running `apply` again tells Kubernetes about the new desired state.

Kubernetes then reconciles the cluster toward the new state.

This is one reason declarative configuration is central to Kubernetes
and GitOps workflows.

------------------------------------------------------------------------

# 26. What Exactly Does `kubectl apply` Change?

For the Deployment example:

``` yaml
replicas: 3
```

you are effectively declaring:

> "I want Kubernetes to maintain this Deployment with three replicas."

Kubernetes then determines how to achieve that state.

You are **not** manually telling:

``` text
Create container on Worker Node 2.
```

Instead, you declare:

``` text
I want this application state.
```

Kubernetes decides how to achieve it within the cluster's constraints.

------------------------------------------------------------------------

# 27. Useful Commands for Understanding the Workflow

## Check Deployment

``` bash
kubectl get deployment
```

Detailed Deployment:

``` bash
kubectl describe deployment nginx-deployment
```

## Check ReplicaSets

``` bash
kubectl get rs
```

## Check Pods

``` bash
kubectl get pods
```

Detailed Pod information:

``` bash
kubectl describe pod <pod-name>
```

## Check Pod Node Placement

``` bash
kubectl get pods -o wide
```

This is useful because it shows which Worker Node is running each Pod.

## Check Cluster Nodes

``` bash
kubectl get nodes
```

Detailed node information:

``` bash
kubectl describe node <node-name>
```

## Check Events

``` bash
kubectl get events --sort-by=.lastTimestamp
```

Events are extremely useful when a Pod cannot start or cannot be
scheduled.

------------------------------------------------------------------------

# 28. Useful Debugging Flow

If someone says:

> "I executed `kubectl apply`, but my application isn't running."

Do not immediately assume the Deployment failed.

Check layer by layer:

``` text
Deployment
    ↓
ReplicaSet
    ↓
Pod
    ↓
Scheduling
    ↓
Node
    ↓
kubelet
    ↓
Container Runtime
    ↓
Container
    ↓
Readiness
    ↓
Service / Traffic
```

Commands:

``` bash
kubectl get deployment
kubectl get rs
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl get events --sort-by=.lastTimestamp
```

This layered approach is useful in production troubleshooting.

------------------------------------------------------------------------

# 29. Common Failure Points

## API Server

Possible issues:

-   API endpoint unavailable
-   Authentication failure
-   Authorization failure
-   Admission policy rejection
-   Invalid manifest

## etcd

Possible issues:

-   Control-plane state problems
-   Storage/performance issues
-   Availability problems
-   Backup/recovery concerns

## Scheduler

Possible issues:

-   Insufficient CPU/memory
-   Node selector mismatch
-   Affinity constraints
-   Taints without matching tolerations
-   Topology constraints

## kubelet

Possible issues:

-   Node unhealthy
-   Kubelet problems
-   Runtime communication issues
-   Image pull problems

## Container Runtime

Possible issues:

-   Image pull failure
-   Container startup failure
-   Runtime errors

## Application

Possible issues:

-   Application crash
-   Incorrect configuration
-   Missing dependencies
-   Failed readiness probe
-   Failed liveness probe

------------------------------------------------------------------------

# 30. Important Interview Distinctions

  -----------------------------------------------------------------------
  Concept                             Correct Mental Model
  ----------------------------------- -----------------------------------
  `kubectl`                           CLI used to interact with
                                      Kubernetes API

  kubeconfig                          Tells kubectl which cluster/context
                                      and authentication configuration to
                                      use

  API Server                          Kubernetes API entry point

  Authentication                      Who are you?

  Authorization                       Are you allowed?

  Admission                           Can the request be
                                      accepted/modified by admission
                                      policy?

  etcd                                Stores Kubernetes cluster state

  Controller                          Reconciles desired and actual state

  Deployment                          Manages application rollout through
                                      ReplicaSets

  ReplicaSet                          Maintains desired number of Pods

  Scheduler                           Selects a suitable node

  kubelet                             Node agent that manages Pods

  Container Runtime                   Runs/manages containers

  Pod                                 Smallest deployable Kubernetes
                                      workload unit

  kube-proxy                          Helps implement Service networking

  CNI                                 Provides cluster networking
                                      functionality
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 31. Common Interview Mistakes

### Mistake 1

> "kubectl directly creates the container."

Incorrect.

Better:

> `kubectl` sends the desired configuration to the API Server.
> Kubernetes components then reconcile the desired state and eventually
> the kubelet and container runtime create/run the workload on a Worker
> Node.

------------------------------------------------------------------------

### Mistake 2

> "Scheduler creates the Pod."

Better:

> The scheduler selects a suitable Worker Node for an unscheduled Pod.

------------------------------------------------------------------------

### Mistake 3

> "kubelet decides where the Pod should run."

Better:

> The scheduler selects the node; kubelet manages the Pod on that node.

------------------------------------------------------------------------

### Mistake 4

> "etcd stores my application database."

Incorrect.

Better:

> etcd stores Kubernetes cluster state. The application's database is
> separate.

------------------------------------------------------------------------

### Mistake 5

> "kube-proxy handles all Kubernetes networking."

Too broad.

Better:

> kube-proxy helps implement Service networking, while the CNI/network
> plugin provides important cluster networking functionality.

------------------------------------------------------------------------

### Mistake 6

> "A Pod is a container."

Incorrect.

Better:

> A Pod is the smallest deployable unit in Kubernetes and can contain
> one or more containers.

------------------------------------------------------------------------

# 32. Production-Level View

In a real production environment, the workflow can involve additional
components and policies:

``` text
Developer / CI/CD
       |
       v
kubectl / Automation
       |
       v
API Server
       |
       +--> Authentication
       +--> Authorization
       +--> Admission Policies
       +--> Validation
       |
       v
      etcd
       |
       +--> Controllers
       +--> Scheduler
       |
       v
Worker Nodes
       |
       +--> kubelet
       +--> Container Runtime
       +--> CNI
       +--> kube-proxy / Service networking
       +--> Pods
       |
       v
Application
```

Production clusters may additionally use:

-   Multiple control-plane nodes
-   Highly available etcd
-   Load-balanced Kubernetes API endpoints
-   Cloud-provider integrations
-   Network policies
-   Pod Security controls
-   Resource requests and limits
-   Horizontal Pod Autoscaling
-   Pod Disruption Budgets
-   Ingress or Gateway APIs
-   Persistent storage
-   Secrets/configuration management
-   Monitoring and logging
-   GitOps or CI/CD automation

These are related to the broader Kubernetes architecture but are not all
required to understand the basic `kubectl apply` flow.

------------------------------------------------------------------------

# 33. Managed Kubernetes

With managed Kubernetes services such as Amazon EKS, Azure AKS or Google
GKE, the cloud provider manages significant portions of the Control
Plane infrastructure.

The exact division of responsibility varies by service and
configuration.

The important interview concept remains the same:

``` text
API Server
    ↓
Cluster State
    ↓
Controllers / Scheduler
    ↓
Worker Nodes
    ↓
kubelet
    ↓
Container Runtime
    ↓
Pods
```

Managed Kubernetes does not eliminate the need to understand Kubernetes
architecture.

------------------------------------------------------------------------

# 34. Declarative Kubernetes --- The Core Concept

Kubernetes is fundamentally declarative.

You specify:

``` text
What I WANT
```

rather than manually specifying every action required to get there.

Example:

``` yaml
replicas: 3
```

You are declaring:

> Maintain three replicas.

Kubernetes continuously observes the actual state and reconciles it
toward the desired state.

This gives us:

``` text
Desired State
      |
      v
Kubernetes API
      |
      v
Controllers / Scheduler / Node Agents
      |
      v
Actual State
      |
      +------ reconciliation ------+
```

------------------------------------------------------------------------

# 35. The Most Important Mental Model

Remember these questions:

### API Server

> **Can Kubernetes accept and process my request?**

### etcd

> **What is the cluster's persisted state?**

### Controller

> **What should Kubernetes do to reach the desired state?**

### Scheduler

> **Which node should this Pod run on?**

### kubelet

> **Is the Pod actually running correctly on my node?**

### Container Runtime

> **Can I create and run the containers?**

### Pod

> **Where does the application workload run?**

------------------------------------------------------------------------

# 36. 30-Second Interview Answer

> "When I execute `kubectl apply -f deployment.yaml`, kubectl reads my
> kubeconfig and sends the request to the Kubernetes API Server. The API
> Server authenticates and authorizes the request, performs validation
> and admission processing, and persists the accepted state in etcd.
> Controllers then reconcile the desired state. If new Pods are
> required, the scheduler selects suitable Worker Nodes. Once a Pod is
> assigned to a node, the kubelet works with the container runtime, such
> as containerd, to start the containers. Kubernetes then continuously
> monitors and reconciles the actual state against the desired state."

------------------------------------------------------------------------

# 37. 60-Second Production Interview Answer

> "Let me take `kubectl apply -f deployment.yaml` as an example. First,
> kubectl reads the kubeconfig to identify the cluster, API Server and
> authentication configuration. It sends the request to the API Server,
> which authenticates and authorizes the caller and performs validation
> and admission processing. Once accepted, Kubernetes persists the
> relevant cluster state in etcd. Controllers then reconcile the desired
> state described by the Deployment. For example, if I specify three
> replicas, the Deployment and ReplicaSet controllers work toward having
> three Pods. Pods that need placement are handled by the scheduler,
> which selects suitable Worker Nodes based on resource requirements and
> scheduling constraints. The kubelet on the selected node then works
> with the container runtime, such as containerd, to start the
> containers. Finally, Kubernetes continuously reconciles the cluster so
> that the actual state remains aligned with the desired state."

------------------------------------------------------------------------

# 38. Quick Revision Flow

``` text
kubectl
  ↓
kubeconfig
  ↓
API Server
  ↓
Authentication
  ↓
Authorization
  ↓
Validation / Admission
  ↓
etcd
  ↓
Controllers
  ↓
ReplicaSet / Pod creation
  ↓
Scheduler
  ↓
Worker Node
  ↓
kubelet
  ↓
Container Runtime
  ↓
Pod
  ↓
Application Container
```

------------------------------------------------------------------------

# 39. Quick Revision Table

  ----------------------------------------------------------------------------------
  Problem / Component     Primary Question        Useful Command / Keyword
  ----------------------- ----------------------- ----------------------------------
  kubeconfig              Which cluster/context?  `kubectl config current-context`

  API Server              Can the API request be  API endpoint
                          processed?              

  Authentication          Who are you?            Identity / credentials

  Authorization           Are you allowed?        `kubectl auth can-i`

  Admission               Should request be       Admission policy
                          accepted/modified?      

  etcd                    Where is cluster state  etcd
                          persisted?              

  Deployment              What workload should be `kubectl get deploy`
                          maintained?             

  ReplicaSet              How many Pods should    `kubectl get rs`
                          exist?                  

  Scheduler               Which node should run   Scheduling constraints
                          the Pod?                

  kubelet                 Is the Pod running on   `kubectl describe node`
                          this node?              

  Container Runtime       Can the container run?  containerd / CRI

  Pod                     Where does workload     `kubectl get pods -o wide`
                          run?                    

  Events                  Why did something fail? `kubectl get events`

  Readiness               Is the app ready for    Readiness probe
                          traffic?                

  Liveness                Should the container be Liveness probe
                          restarted?              
  ----------------------------------------------------------------------------------

------------------------------------------------------------------------

# 40. Final Takeaway

The most important concept is not memorizing every component.

Understand the relationship:

``` text
I DECLARE
     |
     v
Desired State
     |
     v
API Server
     |
     v
Cluster State
     |
     v
Controllers
     |
     v
Scheduler
     |
     v
Worker Node
     |
     v
kubelet
     |
     v
Container Runtime
     |
     v
Pod
     |
     v
Application
```

And remember:

> **kubectl does not directly deploy the container to a node. It submits
> the desired state to the Kubernetes API. Kubernetes then uses its
> control-plane and node components to reconcile that desired state into
> a running workload.**

------------------------------------------------------------------------

# 41. Reel 2 vs Reel 1

## Reel 1 --- Kubernetes Cluster Architecture

Focus:

``` text
What components exist?
```

Topics:

-   Control Plane
-   API Server
-   etcd
-   Scheduler
-   Controller Manager
-   Worker Node
-   kubelet
-   kube-proxy
-   Container Runtime
-   Pods

## Reel 2 --- kubectl Apply Workflow

Focus:

``` text
What happens when I deploy?
```

Topics:

``` text
kubectl
→ kubeconfig
→ API Server
→ Authentication
→ Authorization
→ Validation / Admission
→ etcd
→ Controllers
→ Scheduler
→ kubelet
→ Container Runtime
→ Pod
```

Keeping these as two separate topics makes the concepts easier to learn
and easier to explain during interviews.

------------------------------------------------------------------------

# 42. Final One-Line Revision

> **`kubectl` sends the desired state to the API Server; Kubernetes
> stores it, reconciles it, schedules Pods, and uses kubelet plus the
> container runtime to run the workload on a Worker Node.**
