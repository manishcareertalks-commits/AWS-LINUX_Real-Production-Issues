# Kubernetes Pod is Running but Application is Not Accessible
## Production Troubleshooting Guide

> **Core production lesson:**  
> A Kubernetes Pod being `Running` does **not** mean that the application is reachable.
>
> The request must successfully travel through the complete traffic path:
>
> **User → Ingress / Load Balancer → Service → TargetPort → Pod → Application**

---

## 1. The Production Problem

A very common Kubernetes incident looks like this:

```text
Pod Status: Running
Application: Not Accessible
```

You run:

```bash
kubectl get pods
```

and see:

```text
NAME                    READY   STATUS    RESTARTS   AGE
payment-api-7d9f...     1/1     Running   0          20m
```

At first glance, everything appears healthy.

But users receive:

- Connection timeout
- Connection refused
- HTTP 502
- HTTP 503
- HTTP 504
- DNS resolution errors
- Empty response
- Intermittent failures

The important distinction is:

```text
Running = container process is running
Ready   = Pod is considered ready to receive traffic
Reachable = the complete network/application path works
```

These are **not the same thing**.

---

# 2. Understand the Complete Traffic Flow

For a typical production application:

```text
                    INTERNET / USER
                           |
                           v
                    Load Balancer
                           |
                           v
                       Ingress
                           |
                           v
                      Kubernetes
                        Service
                           |
                    Service Port
                           |
                       TargetPort
                           |
                           v
                         Pod
                           |
                     Container Port
                           |
                           v
                     Application
```

For an internal application, the flow may simply be:

```text
Client
  |
  v
Service
  |
  v
TargetPort
  |
  v
Pod
  |
  v
Application
```

The troubleshooting strategy should follow this path rather than randomly checking Kubernetes objects.

---

# 3. First Principle: Running ≠ Accessible

A Pod can be `Running` while:

- The application failed to start correctly.
- The application is listening on a different port.
- The application is listening only on `127.0.0.1`.
- The Service selector does not match the Pod.
- The Service has no endpoints.
- The `targetPort` is incorrect.
- The readiness probe is failing.
- NetworkPolicy blocks traffic.
- Ingress routing is incorrect.
- The Load Balancer is unhealthy.
- DNS points to the wrong endpoint.
- The application is overloaded or returning errors.

Therefore:

> **Never conclude that the application is healthy simply because the Pod status says `Running`.**

---

# 4. Step 1 — Check the Pod Status

Start with:

```bash
kubectl get pods -o wide
```

Example:

```text
NAME                    READY   STATUS    RESTARTS   AGE   IP
payment-api-7d9f...     1/1     Running   0          20m   10.244.1.25
```

Check:

- `STATUS`
- `READY`
- Restart count
- Pod IP
- Node
- Age

Then inspect the Pod:

```bash
kubectl describe pod <pod-name>
```

Look especially at:

```text
Conditions:
  Initialized
  Ready
  ContainersReady
  PodScheduled
```

### Important distinction

A Pod may show:

```text
STATUS: Running
READY: 0/1
```

This means the container is running, but the Pod is not considered ready.

That is a major production clue.

---

# 5. Step 2 — Check Application Logs

Run:

```bash
kubectl logs <pod-name>
```

For a specific container:

```bash
kubectl logs <pod-name> -c <container-name>
```

For recent logs:

```bash
kubectl logs <pod-name> --since=10m
```

For a previous crashed container:

```bash
kubectl logs <pod-name> --previous
```

Look for:

- Startup errors
- Configuration errors
- Database connection failures
- Authentication failures
- Port binding errors
- Dependency failures
- Exceptions
- Out-of-memory symptoms
- Application crashes
- Incorrect environment variables

Example:

```text
ERROR: Unable to connect to database
ERROR: Application startup failed
```

The Pod may still exist and the container may still be running, but the application may not be serving requests correctly.

---

# 6. Step 3 — Verify Which Port the Application Is Actually Listening On

This is one of the most important checks.

Suppose the Kubernetes Service expects:

```text
targetPort: 8080
```

But the application is actually listening on:

```text
3000
```

Traffic will fail.

Inside the container, check listening sockets:

```bash
kubectl exec -it <pod-name> -- sh
```

Then depending on the image:

```bash
ss -lntp
```

or:

```bash
netstat -lntp
```

You may see:

```text
LISTEN  0  128  0.0.0.0:8080
```

This tells you the application is listening on port `8080`.

### Check the bind address too

This is a common production issue.

Bad for normal Pod networking:

```text
127.0.0.1:8080
```

Usually expected:

```text
0.0.0.0:8080
```

Why?

`127.0.0.1` means the application is listening only on the Pod's loopback interface.

Traffic arriving through the Pod IP cannot reach an application bound only to localhost.

---

# 7. Step 4 — Test the Application From Inside the Pod

Enter the Pod:

```bash
kubectl exec -it <pod-name> -- sh
```

Then:

```bash
curl localhost:<port>
```

Example:

```bash
curl localhost:8080
```

If the application exposes a health endpoint:

```bash
curl localhost:8080/health
```

### Interpret the result

If this works:

```text
curl localhost:8080
HTTP/1.1 200 OK
```

the application is responding locally.

The problem is likely somewhere outside the application itself.

If this fails:

```text
curl: Connection refused
```

focus on:

- Application startup
- Listening port
- Bind address
- Application configuration
- Process health

---

# 8. Step 5 — Check the Kubernetes Service

List Services:

```bash
kubectl get svc
```

Inspect the relevant Service:

```bash
kubectl describe svc <service-name>
```

Or:

```bash
kubectl get svc <service-name> -o yaml
```

Look at:

```yaml
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
```

Understand the difference:

### Service Port

The port exposed by the Kubernetes Service.

```text
Service Port = 80
```

### TargetPort

The port on the backend Pod where traffic should be delivered.

```text
TargetPort = 8080
```

Traffic conceptually becomes:

```text
Client
   |
   | Service:80
   v
Service
   |
   | targetPort:8080
   v
Pod:8080
```

A wrong `targetPort` can cause traffic to fail even when the Pod is perfectly healthy.

---

# 9. Step 6 — Verify Labels and Selectors

This is one of the most common Kubernetes Service problems.

Check Pod labels:

```bash
kubectl get pod <pod-name> --show-labels
```

Example:

```text
app=payment-api
version=v1
```

Now inspect the Service:

```bash
kubectl describe svc <service-name>
```

Look for:

```text
Selector: app=payment-api
```

The Service selector must match the Pod labels.

Example:

### Pod

```yaml
metadata:
  labels:
    app: payment-api
```

### Service

```yaml
spec:
  selector:
    app: payment-api
```

This matches.

But if the Service says:

```yaml
selector:
  app: payments-api
```

and the Pod says:

```yaml
labels:
  app: payment-api
```

there is no match.

Result:

```text
Service
   |
   X
No matching Pods
```

---

# 10. Step 7 — Check Endpoints

Run:

```bash
kubectl get endpoints <service-name>
```

Also useful on modern Kubernetes versions:

```bash
kubectl get endpointslice
```

A healthy Service should have backend addresses.

Example:

```text
NAME           ENDPOINTS
payment-api    10.244.1.25:8080,10.244.2.31:8080
```

If you see:

```text
NAME           ENDPOINTS
payment-api    <none>
```

the Service currently has no usable backend endpoints.

Investigate:

1. Service selector
2. Pod labels
3. Pod readiness
4. Pod status
5. EndpointSlice configuration

### Production interpretation

No endpoints means:

```text
Client
  |
  v
Service
  |
  X
No backend endpoint
```

The Service cannot forward traffic to a backend that it does not have available.

---

# 11. Step 8 — Check Readiness Probes

A Pod can be:

```text
Running
```

but:

```text
Not Ready
```

Readiness probes determine whether the Pod should receive traffic from Kubernetes Services.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
```

If this endpoint fails, Kubernetes can keep the container running while removing the Pod from the Service's ready backend set.

Check:

```bash
kubectl describe pod <pod-name>
```

Look for:

```text
Readiness probe failed
```

Example:

```text
Readiness probe failed:
HTTP probe failed with statuscode: 503
```

### Important production lesson

A readiness failure does **not necessarily mean the application process has crashed**.

It can mean:

```text
Application process = Running
Application = temporarily not ready
Traffic = should not be sent
```

This is particularly important during:

- Startup
- Deployment
- Rolling updates
- Dependency outages
- Database migrations
- High load
- Graceful shutdown

---

# 12. Liveness vs Readiness

These probes have different purposes.

## Readiness

Question:

> "Should this Pod receive traffic?"

If readiness fails:

```text
Pod remains running
Pod should not receive normal Service traffic
```

## Liveness

Question:

> "Is the container/application still alive enough to continue running?"

If liveness repeatedly fails, Kubernetes may restart the container.

### Simple mental model

```text
Liveness
   |
   +---- Is the application alive?
              |
              v
          Restart if needed


Readiness
   |
   +---- Should traffic be sent here?
              |
              v
          Remove from ready backends
```

---

# 13. Step 9 — Verify Service Type and Access Method

Kubernetes Services can have different types.

## ClusterIP

```yaml
type: ClusterIP
```

Used primarily for internal cluster communication.

A common mistake is trying to access a ClusterIP directly from the public internet.

---

## NodePort

```yaml
type: NodePort
```

Exposes the Service through a port on Kubernetes nodes.

---

## LoadBalancer

```yaml
type: LoadBalancer
```

Typically integrates with the cloud provider to provision an external load balancer.

Check:

```bash
kubectl get svc <service-name>
```

Verify:

- External IP / hostname
- Assigned ports
- Load Balancer status

---

## Ingress

Ingress provides HTTP/HTTPS routing to Services.

Example flow:

```text
Internet
   |
   v
Load Balancer
   |
   v
Ingress Controller
   |
   v
Service
   |
   v
Pod
```

If the Pod works internally but users cannot access it externally, investigate:

- Ingress rules
- Hostname
- Path
- TLS configuration
- Ingress Controller
- Load Balancer
- DNS

---

# 14. Step 10 — Test the Service From Inside the Cluster

Do not test only from your laptop.

First test the Pod directly.

For example, from a debugging Pod:

```bash
kubectl run network-debug \
  --rm -it \
  --image=curlimages/curl \
  -- sh
```

Then:

```bash
curl http://<service-name>:<service-port>
```

Example:

```bash
curl http://payment-api:80
```

You can also test the Service IP:

```bash
curl http://<cluster-ip>:80
```

This helps isolate the problem.

### Example diagnostic logic

```text
curl Pod IP directly
        |
        +---- FAIL → Pod/application/network issue
        |
        +---- SUCCESS
                |
                v
        curl Service
                |
                +---- FAIL → Service/endpoints/network policy issue
                |
                +---- SUCCESS
                        |
                        v
                External access fails
                        |
                        v
             Ingress/LB/DNS/TLS issue
```

---

# 15. Step 11 — Check DNS

If users access:

```text
https://api.example.com
```

test DNS:

```bash
nslookup api.example.com
```

or:

```bash
dig api.example.com
```

Inside the cluster, test Kubernetes DNS:

```bash
nslookup payment-api
```

A DNS issue can make a healthy application appear inaccessible.

Check:

- DNS record
- Service DNS name
- Namespace
- CoreDNS
- External DNS configuration
- Ingress hostname

---

# 16. Step 12 — Check NetworkPolicies

Even when:

- Pod is Running
- Pod is Ready
- Service has endpoints
- Application responds locally

traffic can still be blocked by a NetworkPolicy.

Check:

```bash
kubectl get networkpolicy
```

Then:

```bash
kubectl describe networkpolicy <policy-name>
```

Think about both directions:

```text
Client → Pod
Pod → Dependency
```

For example, the application may receive traffic correctly but fail because it cannot connect to:

- Database
- Redis
- Another microservice
- External API

NetworkPolicy can be responsible for either ingress or egress restrictions.

---

# 17. Step 13 — Check Ingress

If direct Service access works but external access fails, inspect Ingress:

```bash
kubectl get ingress
```

Then:

```bash
kubectl describe ingress <ingress-name>
```

Verify:

- Host
- Path
- Backend Service
- Backend Service Port
- TLS
- Ingress Class

Example:

```text
api.example.com
       |
       v
Ingress
       |
       v
payment-api:80
       |
       v
Pod:8080
```

A wrong backend Service name or port can result in errors such as:

```text
404
502
503
```

---

# 18. Step 14 — Check HTTP Status Codes

The error code itself provides useful information.

## 502 Bad Gateway

Often indicates an upstream/proxy problem.

Investigate:

- Ingress
- Load Balancer
- Backend connectivity
- Service
- Application port

## 503 Service Unavailable

Often indicates no usable backend or an unavailable upstream.

Investigate:

- Readiness
- Endpoints
- Service selector
- Backend Pods

## 504 Gateway Timeout

Often indicates that the proxy/load balancer could not get a response in time.

Investigate:

- Application latency
- Network connectivity
- Dependency latency
- Load Balancer timeout
- Ingress timeout

## Connection Refused

Usually means something actively rejected the connection.

Investigate:

- Wrong port
- Application not listening
- Wrong targetPort
- Application bound incorrectly

## Connection Timeout

Often points toward:

- Network path
- Firewall
- Security rules
- NetworkPolicy
- Load Balancer
- Routing

Do not treat these mappings as absolute; always verify the actual traffic path.

---

# 19. Step 15 — Check Events

Kubernetes events can reveal issues that are not obvious from `kubectl get pods`.

Run:

```bash
kubectl get events --sort-by=.lastTimestamp
```

Or:

```bash
kubectl describe pod <pod-name>
```

Look for:

- Failed probe
- Failed mount
- Image problems
- Scheduling failures
- Network issues
- Container restarts
- Permission problems

In production, events are often one of the fastest ways to understand what changed.

---

# 20. Step 16 — Check Resource Pressure

An application can technically be running but still behave badly under resource pressure.

Check:

```bash
kubectl top pod
```

and:

```bash
kubectl top node
```

Look for:

- High CPU
- High memory
- CPU throttling
- Memory pressure
- OOMKilled containers

Inspect:

```bash
kubectl describe pod <pod-name>
```

Look for:

```text
Reason: OOMKilled
```

Also review the application's own metrics.

---

# 21. Step 17 — Check Recent Deployment Changes

Production incidents often happen immediately after a deployment.

Check:

```bash
kubectl rollout history deployment/<deployment-name>
```

Check the Deployment:

```bash
kubectl describe deployment <deployment-name>
```

Look for changes to:

- Container image
- Environment variables
- Ports
- Readiness probe
- Liveness probe
- Service selector
- ConfigMap
- Secret
- Resource limits
- Ingress configuration

If the application worked before the deployment, compare the old and new versions.

---

# 22. Step 18 — Check ConfigMaps and Secrets

The application may start but fail to serve requests because of incorrect configuration.

Check:

```bash
kubectl get configmap
```

and:

```bash
kubectl get secrets
```

Do not print secret values into terminals, logs, tickets, or chat.

Inspect the Pod configuration safely:

```bash
kubectl describe pod <pod-name>
```

Look for:

- Environment variable names
- Mounted ConfigMaps
- Mounted Secrets
- Incorrect configuration references

Typical production example:

```text
Application starts
        |
        v
Database configuration is wrong
        |
        v
Readiness fails
        |
        v
Pod remains Running
        |
        v
Service has no ready backend
```

---

# 23. A Practical Production Troubleshooting Decision Tree

Use this order.

```text
START
  |
  v
Is Pod Running?
  |
  +-- NO --> Check scheduling/events/container
  |
  +-- YES
       |
       v
Is Pod Ready?
       |
       +-- NO --> Check readiness probe/application health
       |
       +-- YES
            |
            v
Does application respond on localhost?
            |
            +-- NO --> Check application/process/port/bind address
            |
            +-- YES
                 |
                 v
Does Service have endpoints?
                 |
                 +-- NO --> Check selectors/readiness
                 |
                 +-- YES
                      |
                      v
Can another Pod reach the Service?
                      |
                      +-- NO --> Check Service/NetworkPolicy/DNS
                      |
                      +-- YES
                           |
                           v
Can external client reach it?
                           |
                           +-- NO --> Check Ingress/LB/DNS/TLS
                           |
                           +-- YES
                                |
                                v
                         Application works
```

---

# 24. The Most Useful Commands

## Pod

```bash
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
```

## Service

```bash
kubectl get svc
kubectl describe svc <service-name>
kubectl get svc <service-name> -o yaml
```

## Endpoints

```bash
kubectl get endpoints <service-name>
kubectl get endpointslice
```

## Labels

```bash
kubectl get pod <pod-name> --show-labels
kubectl get pods --show-labels
```

## Probes

```bash
kubectl describe pod <pod-name>
```

## Ingress

```bash
kubectl get ingress
kubectl describe ingress <ingress-name>
```

## NetworkPolicy

```bash
kubectl get networkpolicy
kubectl describe networkpolicy <policy-name>
```

## Events

```bash
kubectl get events --sort-by=.lastTimestamp
```

## Resource usage

```bash
kubectl top pod
kubectl top node
```

## Application test

```bash
kubectl exec -it <pod-name> -- sh
curl localhost:<port>
```

## Cluster-side Service test

```bash
kubectl run network-debug \
  --rm -it \
  --image=curlimages/curl \
  -- sh
```

Then:

```bash
curl http://<service-name>:<service-port>
```

---

# 25. Production Example

Assume we have:

```text
Application: payment-api
Pod port: 8080
Service port: 80
TargetPort: 8080
Service type: ClusterIP
```

Architecture:

```text
Client
  |
  v
payment-api:80
  |
  | targetPort 8080
  v
payment-api Pod
  |
  v
Application :8080
```

The Pod shows:

```text
1/1 Running
```

but users cannot access the application.

### Investigation

#### Check logs

```bash
kubectl logs payment-api-xxxxx
```

No obvious error.

#### Test locally

```bash
kubectl exec -it payment-api-xxxxx -- sh
curl localhost:8080
```

Response:

```text
200 OK
```

Application is working.

#### Check Service

```bash
kubectl describe svc payment-api
```

Output shows:

```text
Port:       80
TargetPort: 8080
Selector:   app=payment-api
```

Looks correct.

#### Check endpoints

```bash
kubectl get endpoints payment-api
```

Output:

```text
<none>
```

Now we have a strong lead.

#### Check Pod labels

```bash
kubectl get pod payment-api-xxxxx --show-labels
```

Output:

```text
app=payment
```

But Service expects:

```text
app=payment-api
```

There is a selector mismatch.

### Root Cause

```text
Service selector
app=payment-api

        X

Pod label
app=payment
```

Therefore:

```text
Service
   |
   X
No matching endpoint
   |
   X
No traffic reaches Pod
```

The Pod was healthy.

The application was healthy.

The Service had no matching backend.

### Production lesson

The problem was **not the Pod**.

The problem was the **Service-to-Pod mapping**.

---

# 26. Another Common Production Scenario: Wrong TargetPort

Suppose:

```yaml
containers:
  - name: api
    ports:
      - containerPort: 8080
```

The application listens on:

```text
8080
```

But the Service is configured as:

```yaml
ports:
  - port: 80
    targetPort: 9090
```

Traffic becomes:

```text
Service:80
     |
     v
Pod:9090
     |
     X
Application is actually on 8080
```

Result:

```text
Connection refused / failed request
```

The fix is to align the Service `targetPort` with the actual application listening port.

---

# 27. Another Common Scenario: Running but Not Ready

Pod:

```text
READY   STATUS
0/1     Running
```

Describe Pod:

```bash
kubectl describe pod <pod-name>
```

shows:

```text
Readiness probe failed
HTTP probe failed with statuscode: 503
```

The application process is alive, but its readiness endpoint says:

```text
Service dependencies are not ready
```

Therefore Kubernetes does not consider the Pod ready for normal Service traffic.

Production approach:

1. Check application logs.
2. Check the readiness endpoint manually.
3. Check dependencies.
4. Verify probe path.
5. Verify probe port.
6. Verify probe timing.
7. Confirm whether the application is actually ready.

---

# 28. Another Common Scenario: Works Inside Pod but Not From Service

Suppose:

```bash
curl localhost:8080
```

works.

But:

```bash
curl http://payment-api:80
```

fails.

Now the application itself is less likely to be the primary problem.

Focus on:

```text
Service
TargetPort
Endpoints
EndpointSlices
NetworkPolicy
DNS
```

This is why testing from multiple points is extremely valuable.

---

# 29. Layer-by-Layer Troubleshooting Model

A useful production mental model is:

```text
Layer 1
Application
   |
   v
Layer 2
Container / Port
   |
   v
Layer 3
Pod
   |
   v
Layer 4
Readiness
   |
   v
Layer 5
Service
   |
   v
Layer 6
Endpoints / EndpointSlices
   |
   v
Layer 7
NetworkPolicy / Networking
   |
   v
Layer 8
Ingress
   |
   v
Layer 9
Load Balancer
   |
   v
Layer 10
DNS / TLS / Client
```

Do not jump directly to Layer 8 when Layer 2 has not been validated.

---

# 30. What I Would Do During a Real Production Incident

A practical sequence:

### Step 1

Confirm the impact.

```text
Is everyone affected?
One endpoint?
One namespace?
One Pod?
One node?
```

### Step 2

Check Pods.

```bash
kubectl get pods -o wide
```

### Step 3

Check readiness.

```bash
kubectl get pods
kubectl describe pod <pod-name>
```

### Step 4

Check logs.

```bash
kubectl logs <pod-name> --since=10m
```

### Step 5

Test the application locally.

```bash
kubectl exec -it <pod-name> -- curl localhost:<port>
```

### Step 6

Verify Service.

```bash
kubectl describe svc <service-name>
```

### Step 7

Verify selectors.

```bash
kubectl get pod <pod-name> --show-labels
```

### Step 8

Verify endpoints.

```bash
kubectl get endpoints <service-name>
kubectl get endpointslice
```

### Step 9

Test Service from inside the cluster.

```bash
curl http://<service-name>:<port>
```

### Step 10

If internal traffic works, investigate:

```text
Ingress
Load Balancer
DNS
TLS
External routing
```

### Step 11

Check recent changes.

```text
Deployment
Service
Ingress
ConfigMap
Secret
NetworkPolicy
```

### Step 12

Only after identifying the failure point, make a controlled change.

---

# 31. Avoid These Production Mistakes

## Mistake 1: Assuming Running means healthy

```text
Running ≠ Ready ≠ Reachable
```

---

## Mistake 2: Restarting Pods immediately

Running:

```bash
kubectl delete pod <pod-name>
```

may temporarily hide the real problem.

First identify the failure.

---

## Mistake 3: Changing multiple things at once

If you simultaneously modify:

- Service
- Deployment
- Ingress
- Probe

you may lose the ability to identify the actual root cause.

Change one logical layer at a time.

---

## Mistake 4: Testing only from your laptop

External access can fail even when the Service and Pod are healthy.

Always test from:

```text
Pod
  ↓
Service
  ↓
Ingress
  ↓
External client
```

---

## Mistake 5: Ignoring readiness

A Pod can look completely normal:

```text
STATUS = Running
```

while receiving no traffic because:

```text
READY = 0/1
```

---

## Mistake 6: Ignoring the application bind address

This is a classic issue:

```text
127.0.0.1:8080
```

instead of:

```text
0.0.0.0:8080
```

---

# 32. Observability in Production

For serious production environments, do not depend only on `kubectl`.

Use three major observability signals:

## Logs

Tell you:

```text
What happened?
```

Examples:

- Application errors
- Exceptions
- Startup failures

## Metrics

Tell you:

```text
How much?
How often?
How long?
```

Examples:

- Request rate
- Error rate
- Latency
- CPU
- Memory
- Saturation

## Traces

Tell you:

```text
Where did the request spend time?
```

For microservices, distributed tracing can show:

```text
API Gateway
   |
   v
Service A
   |
   v
Service B
   |
   v
Database
```

and identify where the request is failing or becoming slow.

---

# 33. Production Monitoring Signals

For a Kubernetes application, monitor:

### Golden Signals

```text
Latency
Traffic
Errors
Saturation
```

### Kubernetes signals

```text
Pod readiness
Pod restarts
CPU
Memory
OOMKilled
Deployment availability
Service endpoints
Ingress errors
Load Balancer health
```

### Application signals

```text
HTTP 4xx
HTTP 5xx
Request latency
Dependency errors
Database connectivity
Thread/connection pool saturation
```

---

# 34. Interview Answer — Short Version

If asked:

> "Your Kubernetes Pod is Running, but the application is not accessible. How would you troubleshoot it?"

A strong answer is:

> "I would not assume that Running means accessible. I would troubleshoot the complete traffic path step by step. First, I would check the application logs and verify that the application is actually listening on the expected port and bind address. Then I would check the Service configuration, especially the port and targetPort, and verify that the Service selector matches the Pod labels. Next, I would check the Service endpoints or EndpointSlices and confirm that the Pod is Ready and that its readiness probe is passing. I would test the application from inside the Pod using curl, then test the Service from inside the cluster. If internal access works but external access fails, I would investigate the Ingress, Load Balancer, DNS, TLS, and network policies. Finally, I would check recent deployments and configuration changes to identify the root cause."

---

# 35. The Golden Troubleshooting Flow

Memorize this:

```text
                 USER
                   |
                   v
             INGRESS / LB
                   |
                   v
                SERVICE
                   |
                   v
              TARGETPORT
                   |
                   v
                  POD
                   |
                   v
             APPLICATION
```

At each layer ask:

```text
Does it exist?
Does it point to the right place?
Is it healthy?
Can traffic reach it?
Is it responding?
```

---

# 36. Quick Production Checklist

Use this checklist during an incident:

```text
[ ] Pod exists
[ ] Pod is Running
[ ] Pod is Ready
[ ] No unexpected restarts
[ ] Application logs look healthy
[ ] Application process is running
[ ] Application is listening on expected port
[ ] Application is listening on correct bind address
[ ] localhost curl works
[ ] Service exists
[ ] Service port is correct
[ ] Service targetPort is correct
[ ] Service selector matches Pod labels
[ ] Endpoints exist
[ ] EndpointSlices look correct
[ ] Service works from inside cluster
[ ] NetworkPolicy allows required traffic
[ ] Ingress configuration is correct
[ ] Load Balancer is healthy
[ ] DNS resolves correctly
[ ] TLS configuration is correct
[ ] No recent broken deployment/configuration
[ ] CPU/memory/resource pressure checked
[ ] Application dependencies are healthy
```

---

# 37. Final Takeaway

The most important concept is:

> **A Pod being `Running` only tells you that the container process is running. It does not prove that the application is reachable.**

Always follow the traffic path:

```text
User
  ↓
Ingress / Load Balancer
  ↓
Service
  ↓
TargetPort
  ↓
Pod
  ↓
Application
```

And troubleshoot from the inside out:

```text
Application
    ↓
Port
    ↓
Pod
    ↓
Readiness
    ↓
Service
    ↓
Endpoints
    ↓
Network
    ↓
Ingress
    ↓
Load Balancer
    ↓
DNS / Client
```

## The production mindset

```text
RUNNING
   ≠
READY
   ≠
REACHABLE
   ≠
HEALTHY
```

The goal is not to simply restart the Pod.

The goal is to identify **exactly where the traffic path breaks**, prove the failure with a targeted test, fix the root cause, and verify the complete request path again.

---

## One-Line Interview Takeaway

> **"Don't troubleshoot Kubernetes availability by looking only at Pod status. Follow the request path from the client to the application and validate every hop."**
