# SOP: Kubernetes Traffic Reaching Only 3 Out of 10 Pods

## 1. Purpose

This Standard Operating Procedure (SOP) provides a structured
troubleshooting approach for a production issue where:

-   10 Kubernetes application pods are running.
-   Traffic is expected to be distributed across all healthy pods.
-   Only 3 pods are receiving traffic.
-   The remaining 7 pods are running but are not receiving requests.

The objective is to identify **at which layer the remaining pods are
being excluded from the traffic path** and restore normal traffic
distribution safely.

------------------------------------------------------------------------

# 2. Problem Statement

## Incident Scenario

> **You have 10 Kubernetes pods running for your application, but
> traffic is reaching only 3 pods. How would you troubleshoot this
> issue?**

A typical traffic flow may look like:

``` text
Users
  |
  v
Ingress / Load Balancer
  |
  v
Kubernetes Service
  |
  v
EndpointSlices / Endpoints
  |
  +-----------------------------+
  | Pod 1  - Receiving Traffic  |
  | Pod 2  - Receiving Traffic  |
  | Pod 3  - Receiving Traffic  |
  | Pod 4  - No Traffic         |
  | Pod 5  - No Traffic         |
  | Pod 6  - No Traffic         |
  | Pod 7  - No Traffic         |
  | Pod 8  - No Traffic         |
  | Pod 9  - No Traffic         |
  | Pod 10 - No Traffic         |
  +-----------------------------+
```

------------------------------------------------------------------------

# 3. Expected Traffic Flow

``` text
Client / User
      |
      v
External Load Balancer
      |
      v
Ingress Controller
      |
      v
Kubernetes Service
      |
      | Matches Pods Using Labels + Selectors
      v
EndpointSlice / Endpoints
      |
      v
Ready Pods
      |
      +--> Pod 1
      +--> Pod 2
      +--> Pod 3
      +--> Pod 4
      +--> Pod 5
      +--> Pod 6
      +--> Pod 7
      +--> Pod 8
      +--> Pod 9
      +--> Pod 10
```

## Important Principle

A Kubernetes Service does **not** send traffic simply because a pod is
in the `Running` state.

For a pod to receive Service traffic, the following generally need to be
correct:

1.  The Service selector must match the pod labels.
2.  The pod must be eligible to become a Service backend.
3.  The pod should normally be Ready.
4.  The application must be healthy.
5.  The application must listen on the expected port.
6.  Service port and targetPort must be correctly configured.
7.  Traffic policies, session affinity, Ingress, or Load Balancer
    configuration must not prevent distribution.

------------------------------------------------------------------------

# 4. Impact Assessment

## Possible Production Impact

This issue can cause:

-   Uneven traffic distribution.
-   High CPU or memory usage on only a few pods.
-   Increased latency.
-   Request timeouts.
-   Application instability.
-   Reduced fault tolerance.
-   Possible outage if the 3 active pods become overloaded.

## Example

Assume the application receives:

``` text
3,000 requests per second
```

Expected distribution across 10 pods:

``` text
Approximately 300 requests per pod
```

Actual distribution across only 3 pods:

``` text
Approximately 1,000 requests per active pod
```

This can overload the three active pods while the remaining seven pods
remain underutilized.

------------------------------------------------------------------------

# 5. Prerequisites

Before troubleshooting, collect:

-   Kubernetes cluster name.
-   Namespace.
-   Application name.
-   Deployment name.
-   Service name.
-   Ingress name.
-   Names of pods not receiving traffic.
-   Approximate time when the issue started.
-   Recent deployment or configuration changes.

Set variables where appropriate:

``` bash
NAMESPACE=<namespace>
SERVICE=<service-name>
APP=<application-name>
INGRESS=<ingress-name>
```

------------------------------------------------------------------------

# 6. High-Level Troubleshooting Flow

Follow this order:

``` text
Service Selector
      |
      v
Pod Labels
      |
      v
Endpoints / EndpointSlices
      |
      v
Pod Readiness
      |
      v
Application Health
      |
      v
Service Ports / targetPort
      |
      v
Session Affinity / Sticky Sessions
      |
      v
Ingress / Load Balancer
      |
      v
Network Policies / CNI (if required)
```

The goal is to identify:

> **At which layer are the other 7 pods being removed from the traffic
> flow?**

------------------------------------------------------------------------

# 7. Step 1 - Check the Kubernetes Service

## Objective

Verify that the correct Service is routing traffic to the application.

## Command

``` bash
kubectl get svc -n <namespace>
```

Inspect the Service:

``` bash
kubectl describe svc <service-name> -n <namespace>
```

Or view the complete YAML:

``` bash
kubectl get svc <service-name> -n <namespace> -o yaml
```

## What to Check

Look for:

-   `selector`
-   `port`
-   `targetPort`
-   `sessionAffinity`
-   Service type

Example:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
```

## Key Question

> Does the Service selector match all 10 application pods?

------------------------------------------------------------------------

# 8. Step 2 - Verify Pod Labels

## Objective

Ensure that all intended pods have labels matching the Service selector.

## Command

``` bash
kubectl get pods -n <namespace> --show-labels
```

To inspect a specific pod:

``` bash
kubectl get pod <pod-name> -n <namespace> --show-labels
```

Example output concept:

``` text
NAME       LABELS
pod-1      app=my-app
pod-2      app=my-app
pod-3      app=my-app
pod-4      app=my-app-v2
pod-5      app=my-app-v2
```

If the Service selector is:

``` yaml
selector:
  app: my-app
```

Then only Pods 1, 2, and 3 match.

## Root Cause

The Service will send traffic only to pods matching its selector.

## Resolution

Correct the labels or the Service selector.

Example:

``` yaml
metadata:
  labels:
    app: my-app
```

After changing configuration, verify again:

``` bash
kubectl get pods -n <namespace> --show-labels
kubectl describe svc <service-name> -n <namespace>
```

------------------------------------------------------------------------

# 9. Step 3 - Check Endpoints and EndpointSlices

## Objective

Determine how many pod IP addresses Kubernetes has registered behind the
Service.

This is one of the most important troubleshooting steps.

## Check Endpoints

``` bash
kubectl get endpoints <service-name> -n <namespace>
```

Detailed output:

``` bash
kubectl describe endpoints <service-name> -n <namespace>
```

## Check EndpointSlices

``` bash
kubectl get endpointslices -n <namespace> \
-l kubernetes.io/service-name=<service-name>
```

Detailed view:

``` bash
kubectl describe endpointslice <endpointslice-name> -n <namespace>
```

## Expected Result

All healthy and eligible pods should normally appear as backends.

## Scenario A: Only 3 Pod IPs Are Present

Example:

``` text
10 Pods Running
       |
       v
Only 3 Pod IPs Registered in EndpointSlice
```

This means the problem is likely **before or at the endpoint
registration layer**.

Investigate:

-   Service selector.
-   Pod labels.
-   Pod readiness.
-   Pod health.
-   Endpoint conditions.

## Scenario B: All 10 Pod IPs Are Present

If all 10 pods are registered, move to:

-   Session affinity.
-   Ingress.
-   Load Balancer.
-   Traffic source behavior.
-   External routing.

------------------------------------------------------------------------

# 10. Step 4 - Check Pod Readiness

## Important Concept

A pod can be:

``` text
Running = True
Ready   = False
```

A `Running` pod is not automatically guaranteed to receive Service
traffic.

## Check Pod Status

``` bash
kubectl get pods -n <namespace>
```

Example:

``` text
NAME        READY   STATUS
pod-1       1/1     Running
pod-2       1/1     Running
pod-3       1/1     Running
pod-4       0/1     Running
```

The `READY` column is important.

## Inspect the Problematic Pod

``` bash
kubectl describe pod <pod-name> -n <namespace>
```

Check:

-   Conditions.
-   Events.
-   Readiness failures.
-   Probe failures.

## Example Readiness Probe

``` yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
```

## Possible Failures

The readiness probe may fail because:

-   Application is not started.
-   Wrong health check path.
-   Wrong port.
-   Database dependency is unavailable.
-   External dependency is unavailable.
-   Application startup is slow.
-   Timeout is too short.

## Check Events

``` bash
kubectl get events -n <namespace> \
--sort-by='.metadata.creationTimestamp'
```

## Resolution

Correct the readiness probe or application issue.

Do not disable readiness probes in production as a shortcut unless there
is a validated emergency procedure.

------------------------------------------------------------------------

# 11. Step 5 - Check Application Health

## Objective

Verify that the application inside the pods is healthy.

## Check Logs

``` bash
kubectl logs <pod-name> -n <namespace>
```

For a multi-container pod:

``` bash
kubectl logs <pod-name> \
-c <container-name> \
-n <namespace>
```

For previous container logs after a restart:

``` bash
kubectl logs <pod-name> \
--previous \
-n <namespace>
```

## Compare Working and Non-Working Pods

Compare logs from:

``` text
Pod Receiving Traffic
        VS
Pod Not Receiving Traffic
```

Look for:

-   Startup failures.
-   Database connection failures.
-   Configuration errors.
-   Port binding errors.
-   Dependency failures.
-   Health check failures.
-   Authentication failures.

## Check Environment Differences

``` bash
kubectl describe pod <pod-name> -n <namespace>
```

Verify that the affected pods have the expected:

-   Environment variables.
-   ConfigMaps.
-   Secrets.
-   Volume mounts.
-   Image version.

------------------------------------------------------------------------

# 12. Step 6 - Verify Application Port and Service targetPort

## Objective

Ensure the Service forwards traffic to the correct application port.

## Check Service Configuration

``` bash
kubectl get svc <service-name> \
-n <namespace> \
-o yaml
```

Example:

``` yaml
ports:
  - port: 80
    targetPort: 8080
```

This means:

``` text
Service Port
    80
     |
     v
Pod Application Port
    8080
```

## Verify Application Port

Check the container configuration:

``` bash
kubectl describe pod <pod-name> -n <namespace>
```

You can also test connectivity from inside the cluster using a temporary
troubleshooting pod:

``` bash
kubectl run debug \
-it \
--rm \
--restart=Never \
--image=curlimages/curl \
-n <namespace> \
-- sh
```

Then test the Service:

``` bash
curl http://<service-name>
```

Or test a specific pod IP when appropriate:

``` bash
curl http://<pod-ip>:<port>
```

## Important

Direct Pod IP testing should be used carefully in production
troubleshooting and only from authorized network locations.

------------------------------------------------------------------------

# 13. Step 7 - Check Session Affinity

## Objective

Determine whether Kubernetes is intentionally keeping traffic on the
same pods.

## Check Service

``` bash
kubectl get svc <service-name> \
-n <namespace> \
-o yaml
```

Look for:

``` yaml
sessionAffinity: ClientIP
```

## Default Behavior

Typically:

``` yaml
sessionAffinity: None
```

## Problem Scenario

If:

``` yaml
sessionAffinity: ClientIP
```

is enabled and traffic comes from a small number of client IPs, requests
may repeatedly go to the same pods.

## Important Production Observation

Uneven traffic does not always mean Kubernetes is broken.

It can also happen because:

-   Only a few clients are generating traffic.
-   Session affinity is enabled.
-   Cookies are creating sticky sessions at the Ingress layer.
-   The Load Balancer uses connection reuse.
-   Long-lived connections exist.

------------------------------------------------------------------------

# 14. Step 8 - Check Ingress Configuration

## Objective

Verify that the Ingress is routing traffic correctly.

## Check Ingress

``` bash
kubectl get ingress -n <namespace>
```

Inspect configuration:

``` bash
kubectl describe ingress <ingress-name> \
-n <namespace>
```

View YAML:

``` bash
kubectl get ingress <ingress-name> \
-n <namespace> \
-o yaml
```

## Check Ingress Controller Pods

``` bash
kubectl get pods -n <ingress-namespace>
```

## Check Ingress Controller Logs

``` bash
kubectl logs <ingress-controller-pod> \
-n <ingress-namespace>
```

Look for:

-   Upstream errors.
-   Health check failures.
-   Routing errors.
-   Connection errors.
-   Timeout errors.

## Check Sticky Session Configuration

Depending on the Ingress Controller, check for sticky session
annotations or configuration.

------------------------------------------------------------------------

# 15. Step 9 - Check External Load Balancer

## Objective

Determine whether an external Load Balancer is causing uneven
distribution.

Possible components:

-   Cloud Load Balancer.
-   Application Load Balancer.
-   Network Load Balancer.
-   Reverse Proxy.
-   API Gateway.

## Check

Verify:

-   Target health.
-   Backend health checks.
-   Target registration.
-   Connection draining.
-   Sticky sessions.
-   Load balancing algorithm.

## Important

A Load Balancer may consider some backends unhealthy even when the
Kubernetes pod is running.

------------------------------------------------------------------------

# 16. Step 10 - Check Network Policies

If all configuration appears correct but specific pods are unreachable,
inspect NetworkPolicies.

## Command

``` bash
kubectl get networkpolicy -n <namespace>
```

Inspect:

``` bash
kubectl describe networkpolicy <policy-name> \
-n <namespace>
```

## Investigate

Check whether:

-   Ingress traffic is blocked.
-   Only specific pods are allowed.
-   Pod labels affect NetworkPolicy rules.
-   Namespace selectors are incorrect.

------------------------------------------------------------------------

# 17. Step 11 - Check Deployment and Replica Consistency

Verify that all 10 pods belong to the expected Deployment or ReplicaSet.

## Commands

``` bash
kubectl get deployment <deployment-name> \
-n <namespace>
```

``` bash
kubectl describe deployment <deployment-name> \
-n <namespace>
```

Check ReplicaSets:

``` bash
kubectl get rs -n <namespace>
```

## Possible Issue

During a rollout, pods may belong to:

``` text
Old ReplicaSet
New ReplicaSet
```

Labels or versions may differ.

This can result in unexpected traffic distribution.

------------------------------------------------------------------------

# 18. Troubleshooting Decision Tree

``` text
START
  |
  v
10 Pods Running, Only 3 Receiving Traffic
  |
  v
Check Service Selector
  |
  +--> Selector does not match all pods
  |         |
  |         v
  |      Fix Labels / Selector
  |
  v
Check Endpoints / EndpointSlices
  |
  +--> Only 3 endpoints registered
  |         |
  |         v
  |      Check Readiness + Labels
  |
  v
All 10 Endpoints Registered?
  |
  +--> No --> Check Readiness / Health
  |
  +--> Yes
         |
         v
Check Session Affinity
         |
         v
Check Ingress / Load Balancer
         |
         v
Check Sticky Sessions / Traffic Pattern
         |
         v
Check Network Policies
         |
         v
RESOLVE ROOT CAUSE
```

------------------------------------------------------------------------

# 19. Recommended Troubleshooting Commands

## Service

``` bash
kubectl get svc -n <namespace>

kubectl describe svc <service-name> \
-n <namespace>

kubectl get svc <service-name> \
-n <namespace> \
-o yaml
```

## Pods and Labels

``` bash
kubectl get pods -n <namespace>

kubectl get pods \
-n <namespace> \
--show-labels

kubectl describe pod <pod-name> \
-n <namespace>
```

## Endpoints

``` bash
kubectl get endpoints <service-name> \
-n <namespace>

kubectl get endpointslices \
-n <namespace> \
-l kubernetes.io/service-name=<service-name>
```

## Logs

``` bash
kubectl logs <pod-name> \
-n <namespace>

kubectl logs <pod-name> \
--previous \
-n <namespace>
```

## Events

``` bash
kubectl get events \
-n <namespace> \
--sort-by='.metadata.creationTimestamp'
```

## Ingress

``` bash
kubectl get ingress -n <namespace>

kubectl describe ingress <ingress-name> \
-n <namespace>
```

## Network Policy

``` bash
kubectl get networkpolicy \
-n <namespace>
```

------------------------------------------------------------------------

# 20. Common Root Causes

## Root Cause 1 - Label and Selector Mismatch

### Symptoms

-   10 pods are running.
-   Only 3 pods appear in EndpointSlices.

### Cause

Only 3 pods have labels matching the Service selector.

### Resolution

Correct pod labels or Service selector.

------------------------------------------------------------------------

## Root Cause 2 - Readiness Probe Failure

### Symptoms

-   Pods are `Running`.
-   Pods are not `Ready`.
-   Pods do not receive traffic.

### Cause

Readiness probe fails.

### Resolution

Fix:

-   Health endpoint.
-   Port.
-   Application startup.
-   Dependency issue.
-   Probe timeout.

------------------------------------------------------------------------

## Root Cause 3 - Incorrect targetPort

### Symptoms

-   Pods are registered.
-   Application does not receive expected requests.
-   Connection failures occur.

### Cause

Service forwards traffic to the wrong application port.

### Resolution

Correct:

``` yaml
targetPort
```

and verify the application listener.

------------------------------------------------------------------------

## Root Cause 4 - Session Affinity

### Symptoms

-   All pods are healthy.
-   All endpoints are registered.
-   Traffic repeatedly reaches the same few pods.

### Cause

Client IP affinity or sticky sessions.

### Resolution

Review whether sticky sessions are required.

------------------------------------------------------------------------

## Root Cause 5 - Ingress Sticky Sessions

### Symptoms

-   Service configuration looks correct.
-   Endpoints are correct.
-   Traffic is still uneven.

### Cause

Ingress cookie affinity or sticky session configuration.

### Resolution

Review Ingress annotations and controller configuration.

------------------------------------------------------------------------

## Root Cause 6 - Load Balancer Health Checks

### Symptoms

-   Kubernetes pods appear healthy.
-   External traffic reaches only selected backends.

### Cause

External Load Balancer marks some targets as unhealthy.

### Resolution

Review:

-   Health check path.
-   Health check port.
-   Security rules.
-   Backend registration.

------------------------------------------------------------------------

# 21. Production Investigation Checklist

## Service Layer

-   [ ] Correct Service identified.
-   [ ] Service selector verified.
-   [ ] Service ports verified.
-   [ ] targetPort verified.

## Pod Layer

-   [ ] All 10 pods are Running.
-   [ ] All intended pods are Ready.
-   [ ] Labels are consistent.
-   [ ] Containers are healthy.

## Endpoint Layer

-   [ ] All expected pods are registered.
-   [ ] EndpointSlice conditions are correct.
-   [ ] Pod IPs match expected backends.

## Application Layer

-   [ ] Application logs checked.
-   [ ] Health endpoint checked.
-   [ ] Application port verified.
-   [ ] Dependencies checked.

## Traffic Layer

-   [ ] Session affinity checked.
-   [ ] Sticky sessions checked.
-   [ ] Ingress configuration checked.
-   [ ] Load Balancer health checked.

## Network Layer

-   [ ] NetworkPolicies checked.
-   [ ] Pod connectivity tested.
-   [ ] Security rules checked.

------------------------------------------------------------------------

# 22. Safe Production Remediation Strategy

Before making changes:

1.  Identify the root cause.
2.  Avoid changing multiple layers at once.
3.  Record the current configuration.
4.  Make one controlled change.
5.  Validate the result.
6.  Monitor traffic distribution.
7.  Roll back if the change introduces impact.

## Recommended Validation

After remediation:

``` bash
kubectl get pods -n <namespace>
```

Check endpoints:

``` bash
kubectl get endpoints <service-name> \
-n <namespace>
```

Check EndpointSlices:

``` bash
kubectl get endpointslices \
-n <namespace> \
-l kubernetes.io/service-name=<service-name>
```

Verify application metrics:

-   Requests per pod.
-   Error rate.
-   Latency.
-   CPU utilization.
-   Memory utilization.

------------------------------------------------------------------------

# 23. How to Confirm Resolution

The incident can be considered resolved when:

-   All intended pods match the Service selector.
-   All healthy pods are Ready.
-   Expected pod IPs are registered in EndpointSlices.
-   Application health checks pass.
-   Traffic distribution is behaving as expected.
-   Error rates are stable.
-   No pods are overloaded because of uneven traffic.

## Important Note

Traffic may not always be perfectly equal.

For example:

``` text
Pod 1 = 98 requests
Pod 2 = 102 requests
Pod 3 = 95 requests
```

is normal.

However:

``` text
Pod 1 = 5,000 requests
Pod 2 = 4,800 requests
Pod 3 = 4,900 requests
Pods 4-10 = 0 requests
```

requires investigation.

------------------------------------------------------------------------

# 24. Recommended Monitoring and Alerting

To detect this issue early, monitor:

## Per-Pod Metrics

-   Requests per pod.
-   CPU usage.
-   Memory usage.
-   Error rate.
-   Latency.

## Kubernetes Metrics

-   Ready pods.
-   Desired replicas.
-   Available replicas.
-   Endpoint count.

## Recommended Alerts

Create alerts when:

``` text
Desired Replicas != Ready Replicas
```

Or:

``` text
Available Endpoints < Expected Healthy Pods
```

Or when:

``` text
Traffic distribution becomes significantly imbalanced
```

------------------------------------------------------------------------

# 25. Incident Documentation Template

## Incident Title

``` text
Traffic Reaching Only 3 Out of 10 Kubernetes Pods
```

## Detection Time

``` text
<Date and Time>
```

## Impact

``` text
<Application impact>
```

## Root Cause

``` text
<Example: Service selector matched only 3 pods>
```

## Resolution

``` text
<Configuration or application change>
```

## Validation

``` text
<Endpoint count and traffic metrics after fix>
```

## Preventive Action

``` text
<Monitoring / deployment validation / alerting improvement>
```

------------------------------------------------------------------------

# 26. Senior DevOps Interview Summary

A strong troubleshooting approach is:

> **I would first verify the Service selector and pod labels, because a
> Service only routes traffic to pods that match its selector. Then I
> would check EndpointSlices to confirm how many pod IPs are registered.
> If only three pods are registered, I would investigate labels and
> readiness. If all ten are registered, I would move up the traffic path
> and check session affinity, sticky sessions, Ingress, and Load
> Balancer configuration.**

## Final Troubleshooting Order

``` text
Service Selector
        ↓
Pod Labels
        ↓
Endpoints / EndpointSlices
        ↓
Readiness Probe
        ↓
Application Health
        ↓
Service Port / targetPort
        ↓
Session Affinity / Sticky Sessions
        ↓
Ingress / Load Balancer
        ↓
Network Policies (if required)
```

# Key Takeaway

> **Do not start by randomly checking logs. First identify at which
> layer the other pods are being removed from the traffic path.**

That structured approach makes troubleshooting faster, safer, and more
effective in production.
