# Kubernetes Pod Stuck in Pending — Production Troubleshooting Guide

## 1. Introduction

When a Kubernetes Pod remains in the `Pending` state after a production deployment, the first question should be:

> **Why has Kubernetes not been able to schedule this Pod onto a suitable node?**

A Pod in `Pending` does **not automatically mean the application is broken**.

In many cases, the container has not started yet. Therefore, application logs may not be available or useful.

A production troubleshooting approach should separate:

- **Scheduling problems** — Kubernetes cannot place the Pod.
- **Container/startup problems** — the Pod is scheduled, but the container cannot start correctly.
- **Application problems** — the container starts, but the application is unhealthy.

A useful mental model is:

```text
Deployment
   |
   v
Pod Created
   |
   v
Can Kubernetes Schedule It?
   |
   +---- NO ----> Pending
   |                |
   |                +--> Resources
   |                +--> Node Selector / Affinity
   |                +--> Taints / Tolerations
   |                +--> Pod Affinity / Anti-Affinity
   |                +--> Topology Constraints
   |                +--> Storage
   |                +--> CNI / IP Capacity
   |                +--> Cluster Capacity
   |
   +---- YES ---> Pod Scheduled
                    |
                    v
              Container Starts
                    |
                    +--> Image Pull
                    +--> Secrets / Config
                    +--> Application Startup
                    +--> Readiness / Liveness
                    +--> Application Logs
```

---

# 2. Start With the Pod Status

Always establish what Kubernetes is actually reporting.

```bash
kubectl get pods -n <namespace>
```

For additional information:

```bash
kubectl get pods -n <namespace> -o wide
```

Example:

```text
NAME                     READY   STATUS    RESTARTS   AGE   IP       NODE
payment-api-7d9f8c       0/1     Pending   0          2m    <none>   <none>
```

The important clues are:

```text
STATUS = Pending
NODE   = <none>
```

If the Pod has no assigned node, scheduling is a major area to investigate.

---

# 3. Most Important Diagnostic Command: kubectl describe pod

Run:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Pay special attention to:

- `Node`
- `Conditions`
- `Volumes`
- `QoS Class`
- `Node-Selectors`
- `Tolerations`
- `Affinity`
- `Events`

The **Events** section is particularly useful for a genuinely Pending Pod.

Example:

```text
Events:
  Warning  FailedScheduling
  0/6 nodes are available:
  3 Insufficient cpu,
  2 node(s) had untolerated taint,
  1 node(s) didn't match Pod's node affinity
```

This is extremely valuable because Kubernetes is telling us which scheduling constraints are preventing placement.

## Production rule

> **Do not guess the root cause. Read the Events and then investigate the specific layer.**

Events are a diagnostic source, not a separate root-cause category.

---

# 4. Reason 1 — Insufficient CPU or Memory

## Problem

A Pod can remain Pending when its resource requests cannot be satisfied by any eligible node.

Example:

```yaml
resources:
  requests:
    cpu: "4"
    memory: "8Gi"
```

Suppose every eligible node has less than 4 CPUs or 8 GiB of allocatable memory available.

The scheduler cannot place the Pod.

## Check the Nodes

```bash
kubectl get nodes
```

For detailed capacity:

```bash
kubectl describe nodes
```

Look at:

```text
Capacity:
  cpu:
  memory:

Allocatable:
  cpu:
  memory:
```

## Check Current Utilization

If Metrics Server is available:

```bash
kubectl top nodes
```

Also check:

```bash
kubectl top pods -A
```

Important distinction:

### Requests vs actual usage

Kubernetes scheduling is primarily based on **resource requests**, not simply current CPU utilization.

For example:

```yaml
resources:
  requests:
    cpu: "2"
```

Even if the application currently uses only 500m CPU, the scheduler considers the requested amount when deciding whether the Pod fits.

## Check the Pod

```bash
kubectl get pod <pod-name> -n <namespace> -o yaml
```

Look for:

```yaml
resources:
  requests:
    cpu:
    memory:
  limits:
    cpu:
    memory:
```

## Typical Event

```text
0/3 nodes are available: 3 Insufficient cpu.
```

## Production Troubleshooting

Ask:

1. Are the resource requests realistic?
2. Are the nodes large enough?
3. Are existing workloads consuming the allocatable capacity?
4. Is the Pod restricted to a smaller set of nodes by affinity or taints?
5. Can the node group scale?

---

# 5. Reason 2 — Node Selector or Required Node Affinity

## Problem

A developer may tell Kubernetes:

> Run this Pod only on nodes with a specific label.

Example:

```yaml
nodeSelector:
  workload: production
```

If no available node has:

```text
workload=production
```

the Pod cannot be scheduled.

## Check Node Labels

```bash
kubectl get nodes --show-labels
```

Or:

```bash
kubectl get nodes -L workload
```

## Check the Pod

```bash
kubectl get pod <pod-name> -n <namespace> -o yaml
```

Look for:

```yaml
nodeSelector:
```

and:

```yaml
affinity:
```

## Required Node Affinity

Example:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: workload
          operator: In
          values:
          - production
```

The word **required** is important.

If no node satisfies the requirement, scheduling fails.

## Typical Event

```text
0/5 nodes are available:
5 node(s) didn't match Pod's node affinity/selector.
```

## Production Troubleshooting

Check:

```text
Pod requirement
      |
      v
Node label
      |
      v
Do they match?
```

A common real-world issue is a typo or outdated node label.

---

# 6. Reason 3 — Taints and Tolerations

## Problem

A node can be configured to reject Pods unless they explicitly tolerate its taint.

Example node taint:

```text
dedicated=production:NoSchedule
```

A Pod without the corresponding toleration may not be scheduled there.

## Check Node Taints

```bash
kubectl describe node <node-name>
```

Look for:

```text
Taints:
```

You can also use:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

## Example Taint

```text
dedicated=production:NoSchedule
```

## Matching Toleration

```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "production"
  effect: "NoSchedule"
```

## Important

A toleration does not force a Pod onto a node.

It means:

> The Pod is allowed to be scheduled onto a node with this matching taint.

Other scheduling rules still need to be satisfied.

## Typical Event

```text
node(s) had untolerated taint
```

## Production Troubleshooting

Check:

```text
Node Taint
     |
     v
Pod Toleration
     |
     v
Does it match?
```

Be especially careful in clusters with dedicated:

- GPU nodes
- production nodes
- system nodes
- compliance workloads
- high-memory nodes

---

# 7. Reason 4 — Pod Affinity and Anti-Affinity

## Pod Affinity

Pod affinity means:

> I want this Pod to be scheduled close to another Pod/workload according to specified topology rules.

Example concept:

```yaml
podAffinity:
```

This can be useful when workloads benefit from being placed together.

## Pod Anti-Affinity

Pod anti-affinity means:

> I don't want these Pods placed together according to a specified topology rule.

A common production example is spreading replicas so that multiple replicas do not land on the same node.

Example:

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: payment-api
      topologyKey: kubernetes.io/hostname
```

This can prevent multiple matching Pods from being placed on the same node.

## Check the Pod

```bash
kubectl get pod <pod-name> -n <namespace> -o yaml
```

Inspect:

```yaml
affinity:
  podAffinity:
  podAntiAffinity:
```

## Typical Event

You may see a scheduling message indicating that nodes do not satisfy the Pod affinity/anti-affinity requirements.

## Production Troubleshooting

Ask:

1. Is the rule `required` or `preferred`?
2. Are there enough eligible nodes?
3. Does the label selector match the intended Pods?
4. Is the topology key correct?
5. Has the cluster topology changed?

---

# 8. Reason 5 — Topology Spread Constraints

## Problem

Production workloads are often distributed across:

- nodes
- Availability Zones
- failure domains

Kubernetes provides `topologySpreadConstraints` for this.

Example:

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: payment-api
```

The important setting here is:

```text
whenUnsatisfiable: DoNotSchedule
```

This means Kubernetes should not schedule the Pod if the required distribution cannot be satisfied.

## Check

```bash
kubectl get pod <pod-name> -n <namespace> -o yaml
```

Look for:

```yaml
topologySpreadConstraints:
```

Check node topology:

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone \
  -L kubernetes.io/hostname
```

## Production Example

Suppose:

```text
AZ-a: 3 replicas
AZ-b: 3 replicas
AZ-c: 0 replicas
```

A new Pod may be constrained by the desired spread configuration.

## Troubleshooting

Check:

- `maxSkew`
- `topologyKey`
- `whenUnsatisfiable`
- label selector
- number of eligible nodes
- Availability Zone distribution

---

# 9. Reason 6 — PersistentVolumeClaim / Storage Problems

## Problem

A Pod may depend on persistent storage.

For example:

```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: payment-data
```

If the PVC cannot be properly bound or the required storage cannot be satisfied, scheduling can be affected.

## Check PVCs

```bash
kubectl get pvc -n <namespace>
```

Example:

```text
NAME           STATUS    VOLUME
payment-data   Pending
```

Detailed information:

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

## Check Persistent Volumes

```bash
kubectl get pv
```

Detailed:

```bash
kubectl describe pv <pv-name>
```

## Check Storage Classes

```bash
kubectl get storageclass
```

Detailed:

```bash
kubectl describe storageclass <storage-class-name>
```

## Important Production Concepts

Check:

- StorageClass
- access mode
- requested capacity
- dynamic provisioning
- CSI driver
- Availability Zone
- volume topology
- node compatibility

For example, a volume may be associated with one Availability Zone while the Pod is being considered for nodes in another zone.

## Typical Event

You may see events indicating an unbound PVC or storage topology issue.

## Production Troubleshooting

Follow:

```text
Pod
 |
 +--> PVC
       |
       +--> StorageClass
       |
       +--> CSI Driver
       |
       +--> Persistent Volume
       |
       +--> Availability Zone / Topology
```

---

# 10. Reason 7 — Subnet IP Exhaustion / AWS VPC CNI Capacity

## Why This Matters in AWS EKS

In Amazon EKS using the AWS VPC CNI, Pods can consume VPC IP addresses.

Therefore, a cluster may have:

```text
CPU available
Memory available
```

but still have a networking/IP capacity problem.

## Important Nuance

Do not say:

> "No subnet IP automatically means the Pod will always be Pending."

The exact observed state depends on where the failure occurs in the Pod networking/startup path.

Therefore:

> **Use Pod Events and CNI logs to confirm the exact failure.**

## Check AWS VPC CNI Pods

```bash
kubectl get pods -n kube-system -l k8s-app=aws-node
```

Check logs:

```bash
kubectl logs -n kube-system \
  -l k8s-app=aws-node \
  --tail=100
```

For a specific CNI Pod:

```bash
kubectl logs -n kube-system <aws-node-pod>
```

## AWS-Side Checks

Check:

- subnet available IP addresses
- subnet size
- number of Pods
- ENI capacity
- IP address allocation
- instance type networking limits
- VPC CNI configuration

## Production Thinking

Don't only ask:

> Do I have enough CPU?

Also ask:

> **Do I have enough network capacity to place and initialize more Pods?**

This is particularly important in high-density EKS clusters.

---

# 11. Reason 8 — No Suitable Node / Cluster Capacity

## Problem

Sometimes the problem is simply that the cluster does not have enough suitable capacity.

For example:

```text
Current nodes:
node-1 -> full
node-2 -> full
node-3 -> wrong labels
node-4 -> tainted
```

The Pod has nowhere to go.

## Check Nodes

```bash
kubectl get nodes
```

Check details:

```bash
kubectl describe node <node-name>
```

Check resource usage:

```bash
kubectl top nodes
```

## Cluster Autoscaler

If Cluster Autoscaler is being used, verify whether it can add nodes.

Potential problems include:

- node group reached maximum size
- node group cannot scale
- instance provisioning failure
- insufficient cloud capacity
- wrong instance type
- node labels do not satisfy the Pod
- node taints are incompatible
- affinity rules prevent the new node from being useful

## Production Example

Suppose:

```text
Node Group:
min = 3
desired = 3
max = 3
```

All three nodes are full.

The Pod cannot fit.

Even though Cluster Autoscaler may detect an unschedulable Pod, it cannot increase the node group because:

```text
desired = max
```

The result can remain:

```text
Pod = Pending
```

## Troubleshooting

Check:

```bash
kubectl get nodes
```

Then inspect your cloud-side:

- node group
- autoscaler
- instance provisioning
- scaling limits
- cloud capacity

---

# 12. Where Does kubectl logs Fit?

This is one of the most important distinctions for students.

For a Pod that has **never started its container**, application logs are usually not the first troubleshooting tool.

Use:

```bash
kubectl describe pod <pod-name>
```

and inspect Events.

Once the container has started, `kubectl logs` becomes important.

## Basic Command

```bash
kubectl logs <pod-name> -n <namespace>
```

For a specific container:

```bash
kubectl logs <pod-name> -c <container-name> -n <namespace>
```

## Previous Container

If the container has restarted:

```bash
kubectl logs <pod-name> --previous -n <namespace>
```

For a specific container:

```bash
kubectl logs <pod-name> \
  -c <container-name> \
  --previous \
  -n <namespace>
```

## What Can Logs Reveal?

Logs can reveal:

- application startup failure
- configuration errors
- database connection failures
- authentication errors
- AWS Secrets Manager access errors
- missing environment variables
- application exceptions
- dependency failures

---

# 13. AWS Secrets Manager — Where Does It Fit?

A missing secret, incorrect secret ARN, wrong IAM permission, KMS permission problem, or incorrect AWS region can absolutely cause a workload to fail.

However, be precise:

> **AWS Secrets Manager problems are generally not a direct Kubernetes scheduler reason for a genuinely Pending Pod.**

They usually become relevant when the Pod has been scheduled and the application/container or secret integration fails.

Depending on how secrets are integrated, investigate:

- External Secrets Operator
- Secrets Store CSI Driver
- Kubernetes Secret synchronization
- IAM Roles for Service Accounts / EKS Pod Identity
- KMS permissions
- secret ARN
- AWS region

Check Pod Events:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Then, if the container starts:

```bash
kubectl logs <pod-name> -n <namespace>
```

---

# 14. Health Checks — Where Do They Fit?

A missing or incorrect health-check endpoint is usually **not a reason for the scheduler to leave a Pod Pending**.

For example, if your application exposes:

```text
/health
```

but Kubernetes checks:

```text
/healthz
```

the Pod may start but fail its readiness probe.

You may see:

```text
Running 0/1
```

or readiness probe failures in Events.

## Readiness Probe

Readiness determines whether the Pod should receive traffic.

Example:

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
```

## Liveness Probe

Liveness determines whether Kubernetes should restart an unhealthy container.

Example:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
```

## Troubleshooting

Check:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

and:

```bash
kubectl logs <pod-name> -n <namespace>
```

The key distinction:

```text
Pending
   |
   +--> Scheduling investigation

Running 0/1
   |
   +--> Readiness / application investigation

CrashLoopBackOff
   |
   +--> Container / application investigation
```

---

# 15. ImagePullBackOff Is a Different Problem

If you see:

```text
ImagePullBackOff
```

the Pod has already gone beyond the scheduling problem.

Investigate:

- image name
- image tag
- registry availability
- image pull secret
- registry authentication
- network access to registry

Commands:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Then:

```bash
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

If the container actually starts and fails later:

```bash
kubectl logs <pod-name> -n <namespace>
```

---

# 16. CrashLoopBackOff Is a Different Problem

If you see:

```text
CrashLoopBackOff
```

the container is starting and repeatedly failing.

Start with:

```bash
kubectl logs <pod-name> -n <namespace>
```

Then:

```bash
kubectl logs <pod-name> --previous -n <namespace>
```

Also:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Investigate:

- application exceptions
- incorrect configuration
- missing secrets
- database connection
- dependency failures
- incorrect command/entrypoint
- permissions
- memory limits / OOMKilled

---

# 17. Complete Production Troubleshooting Flow

Use this flow when a production Pod is reported as Pending.

```text
                    Pod Pending
                         |
                         v
               kubectl get pods -o wide
                         |
                         v
                Is NODE assigned?
                  /             \
                NO               YES
                |                 |
                v                 v
       kubectl describe       Investigate
            pod               startup/container
                |             state
                v
             Events
                |
                v
       Identify the layer
                |
      +---------+---------+---------+---------+
      |         |         |         |         |
   CPU/RAM   Affinity   Taints   Storage   Network
      |         |         |         |         |
      +---------+---------+---------+---------+
                         |
                         v
                Cluster Capacity
                         |
                         v
                 Fix the root cause
                         |
                         v
                 Observe the Pod
```

---

# 18. Practical Command Checklist

## Step 1 — Check Pod Status

```bash
kubectl get pods -n <namespace> -o wide
```

## Step 2 — Describe the Pod

```bash
kubectl describe pod <pod-name> -n <namespace>
```

## Step 3 — Check Events

```bash
kubectl get events -n <namespace> --sort-by=.lastTimestamp
```

## Step 4 — Check Nodes

```bash
kubectl get nodes
```

## Step 5 — Check Node Labels

```bash
kubectl get nodes --show-labels
```

## Step 6 — Check Node Resources

```bash
kubectl top nodes
```

```bash
kubectl describe nodes
```

## Step 7 — Check Pod Configuration

```bash
kubectl get pod <pod-name> -n <namespace> -o yaml
```

Inspect:

```text
resources
nodeSelector
affinity
tolerations
topologySpreadConstraints
volumes
```

## Step 8 — Check Storage

```bash
kubectl get pvc -n <namespace>
kubectl get pv
kubectl get storageclass
```

## Step 9 — Check EKS CNI

```bash
kubectl get pods -n kube-system -l k8s-app=aws-node
```

```bash
kubectl logs -n kube-system -l k8s-app=aws-node --tail=100
```

## Step 10 — Check Application Logs When Appropriate

```bash
kubectl logs <pod-name> -n <namespace>
```

Previous container:

```bash
kubectl logs <pod-name> --previous -n <namespace>
```

---

# 19. Interview Answer

If an interviewer asks:

> **"A production Pod is stuck in Pending. How would you troubleshoot it?"**

A strong answer is:

> "First, I verify the Pod status using `kubectl get pods -o wide` and check whether a node has been assigned. If it is genuinely unscheduled, I run `kubectl describe pod` and inspect the Events because they usually tell me why the scheduler rejected the available nodes.
>
> Then I investigate the specific scheduling constraint: CPU or memory requests, node selectors and affinity, taints and tolerations, pod affinity or anti-affinity, topology spread constraints, PVC and storage topology, EKS networking/IP capacity, and finally overall cluster or node-group capacity.
>
> If the Pod is already scheduled and the container has started, I switch from scheduling troubleshooting to container troubleshooting and use `kubectl logs`, including `--previous` if the container has restarted.
>
> I don't treat application errors, health-check failures, Secrets Manager errors, CrashLoopBackOff, or ImagePullBackOff as the same problem as a scheduler Pending state. I first identify the actual Pod state and then troubleshoot the corresponding layer."

---

# 20. Key Takeaways

### Remember These 8 Root Causes

1. **Insufficient CPU / Memory**
2. **Node Selector / Required Node Affinity**
3. **Taints / Tolerations**
4. **Pod Affinity / Anti-Affinity**
5. **Topology Spread Constraints**
6. **PVC / Storage / Volume Topology**
7. **Subnet IP Exhaustion / CNI Capacity**
8. **No Suitable Node / Cluster Capacity**

### Remember the Diagnostic Tools

```text
kubectl get pods
        ↓
kubectl describe pod
        ↓
Events
        ↓
Identify root cause
        ↓
Investigate specific layer
        ↓
kubectl logs
```

### Most Important Rule

> **`kubectl describe` + Events help answer: "Why can't Kubernetes schedule this Pod?"**

> **`kubectl logs` helps answer: "Why is the container/application failing after it starts?"**

Do not mix these two troubleshooting paths.

---

# 21. Quick Revision Table

| Problem | Primary Check | Useful Command |
|---|---|---|
| CPU / Memory | Resource requests and node allocatable capacity | `kubectl describe nodes` |
| Node Selector | Node labels | `kubectl get nodes --show-labels` |
| Node Affinity | Pod affinity configuration | `kubectl get pod -o yaml` |
| Taints | Node taints + Pod tolerations | `kubectl describe node` |
| Pod Affinity | Matching Pods and topology | `kubectl get pod -o yaml` |
| Topology Spread | Zones/nodes and spread constraints | `kubectl get nodes -L topology.kubernetes.io/zone` |
| PVC | PVC/PV/StorageClass | `kubectl describe pvc` |
| EKS CNI/IP | Subnet IPs + `aws-node` | `kubectl logs -n kube-system -l k8s-app=aws-node` |
| Cluster Capacity | Nodes + autoscaler/node group | `kubectl get nodes` |
| Application failure | Container logs | `kubectl logs` |
| Previous crash | Previous container logs | `kubectl logs --previous` |

---

# Final Mental Model

When you see:

```text
Pending
```

think:

> **"Can Kubernetes find a suitable place to run this Pod?"**

When you see:

```text
Running 0/1
```

think:

> **"The Pod is running, but is the application ready?"**

When you see:

```text
CrashLoopBackOff
```

think:

> **"Why is the container repeatedly crashing?"**

When you see:

```text
ImagePullBackOff
```

think:

> **"Why can't the node pull the container image?"**

This state-based approach prevents random troubleshooting and helps you move systematically from **Kubernetes scheduler → infrastructure → networking/storage → container → application**.
