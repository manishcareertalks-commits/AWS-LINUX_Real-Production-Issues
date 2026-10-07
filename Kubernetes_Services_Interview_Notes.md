# Kubernetes Services --- Interview Notes

## 1. Why do we need a Kubernetes Service?

Pods are **ephemeral**. Their IP addresses can change when Pods are
recreated.

A Kubernetes **Service** provides a stable network endpoint and routes
traffic to the appropriate Pods.

### Basic flow

``` text
Client / Application
        |
        v
     Service
        |
   +----+----+
   |    |    |
  Pod  Pod  Pod
```

A Service selects Pods using **labels**.

``` yaml
selector:
  app: backend
```

------------------------------------------------------------------------

# 2. ClusterIP

### Scenario

A frontend application needs to communicate with multiple backend Pods,
but the backend should **not be exposed to the internet**.

``` text
Kubernetes Cluster
┌─────────────────────────────────────────────┐
│                                             │
│  Frontend Pod                               │
│       |                                     │
│       v                                     │
│  Backend Service                            │
│     ClusterIP                               │
│       |                                     │
│   +---+---+---+                             │
│   |   |   |   |                             │
│  Pod Pod Pod Pod                            │
│                                             │
└─────────────────────────────────────────────┘
```

### Remember

**ClusterIP = Internal communication inside the cluster**

-   Default Service type.
-   Provides a stable virtual IP.
-   Traffic is routed to matching Pods.
-   Normally reachable only from within the cluster.

### YAML

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

Typical flow:

``` text
Frontend → ClusterIP Service → Backend Pods
```

------------------------------------------------------------------------

# 3. NodePort

### Scenario

You want to expose an application outside the cluster using a
**Kubernetes Node IP and a port**, without provisioning a cloud load
balancer.

``` text
User / Internet
       |
       | Node IP:30080
       v
┌──────────────────────────────┐
│      Kubernetes Cluster      │
│                              │
│   Node 1    Node 2   Node 3  │
│     |         |        |     │
│     +---------+--------+     │
│              |               │
│        NodePort Service      │
│              |               │
│          +---+---+           │
│          |   |   |           │
│         Pod Pod Pod           │
└──────────────────────────────┘
```

### Remember

**NodePort = Expose the application through a Kubernetes Node**

-   Opens the same NodePort on each node.
-   Default NodePort range is typically **30000--32767**.
-   Can be accessed using `NodeIP:NodePort`.
-   Useful for development, testing, or simple external access.
-   In production cloud environments, `LoadBalancer` or an Ingress-based
    architecture is usually preferred.

### YAML

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-nodeport
spec:
  type: NodePort
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

Typical flow:

``` text
User → NodeIP:30080 → NodePort Service → Pods
```

------------------------------------------------------------------------

# 4. LoadBalancer

### Scenario

Your application is running on **AWS EKS** and needs to be accessible
from the internet through a cloud load balancer.

``` text
Users
  |
  v
AWS Load Balancer
  |
  v
LoadBalancer Service
  |
  +--------+--------+
  |        |        |
 Pod      Pod      Pod
```

### Remember

**LoadBalancer = External access through a cloud load balancer**

In supported cloud environments, creating a `LoadBalancer` Service can
cause the cloud provider integration to provision an external load
balancer.

Examples:

-   AWS → Load Balancer
-   Google Cloud → Cloud Load Balancing
-   Azure → Azure Load Balancer

### YAML

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
spec:
  type: LoadBalancer
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
```

Typical production flow:

``` text
Internet
   ↓
Cloud Load Balancer
   ↓
Kubernetes Service
   ↓
Application Pods
```

> **Interview point:** A `LoadBalancer` Service is different from an
> application-level Ingress. Ingress provides HTTP/HTTPS routing
> capabilities, while a LoadBalancer Service primarily exposes a Service
> through a cloud load balancer.

------------------------------------------------------------------------

# 5. ExternalName

### Scenario

Your application is running inside Kubernetes, but the database or API
is **outside the Kubernetes cluster**.

For example, an external database has:

``` text
database.example.com
```

Instead of creating Pods for that database, Kubernetes can provide a
Service name that maps to the external DNS name.

``` text
Kubernetes Cluster
┌─────────────────────────────────┐
│                                 │
│  Application Pod                │
│        |                        │
│        v                        │
│  ExternalName Service           │
│        |                        │
└────────|────────────────────────┘
         |
         | DNS
         v
   External Database
   database.example.com
```

### Remember

**ExternalName = Access an external service using DNS**

### YAML

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com
```

### Important points

-   No Pods are created by this Service.
-   It maps a Kubernetes Service name to an external DNS name.
-   Useful for external databases, APIs, SaaS platforms, etc.
-   Helps applications use a Kubernetes service name instead of
    hard-coding the external endpoint.

Typical flow:

``` text
Application → external-db → database.example.com
```

------------------------------------------------------------------------

# 6. All Service Types --- Quick Comparison

  -----------------------------------------------------------------------
  Service Type            Main Purpose            Typical Access
  ----------------------- ----------------------- -----------------------
  **ClusterIP**           Internal communication  Inside cluster

  **NodePort**            Expose through          Node IP + Port
                          Kubernetes nodes        

  **LoadBalancer**        External access using   Internet / external
                          cloud LB                

  **ExternalName**        Reference external      External DNS
                          service using DNS       
  -----------------------------------------------------------------------

### Easy way to remember

``` text
ClusterIP     → Internal
NodePort      → Node
LoadBalancer  → Cloud Load Balancer
ExternalName  → External DNS
```

------------------------------------------------------------------------

# 7. Interview Scenarios

### Question 1

**Frontend Pods need to communicate with backend Pods internally. Which
Service?**

**Answer:** `ClusterIP`

``` text
Frontend → ClusterIP → Backend Pods
```

------------------------------------------------------------------------

### Question 2

**You want to expose an application using Node IP and port 30080.**

**Answer:** `NodePort`

``` text
NodeIP:30080 → NodePort → Pods
```

------------------------------------------------------------------------

### Question 3

**An application running on EKS needs internet access through a cloud
load balancer.**

**Answer:** `LoadBalancer`

``` text
Internet → AWS Load Balancer → Service → Pods
```

------------------------------------------------------------------------

### Question 4

**Your application needs to connect to a database outside Kubernetes
using its DNS name.**

**Answer:** `ExternalName`

``` text
Application → ExternalName → External DNS
```

------------------------------------------------------------------------

# 8. Important Interview Points

### Service vs Pod IP

**Pod IP:** Can change when a Pod is recreated.

**Service:** Provides a stable endpoint for accessing a group of Pods.

### Service selects Pods

A Service normally uses a label selector:

``` yaml
selector:
  app: backend
```

The Service routes traffic to matching backend endpoints.

### One-line interview answer

> **"Kubernetes Services provide stable network access to applications
> by exposing a logical set of Pods and routing traffic to their backend
> endpoints."**

------------------------------------------------------------------------

# 9. Final Cheat Sheet

``` text
                    Kubernetes Services

                         Service
                            |
       +--------------------+--------------------+
       |                    |                    |
   ClusterIP             NodePort          LoadBalancer
       |                    |                    |
   Internal             Node IP:Port        Cloud LB
   traffic              External access     External access


                    ExternalName
                         |
                  External DNS
                         |
                External Service
```

**Interview memory trick:**

> **ClusterIP → Internal**\
> **NodePort → Node**\
> **LoadBalancer → Cloud LB**\
> **ExternalName → External DNS**
