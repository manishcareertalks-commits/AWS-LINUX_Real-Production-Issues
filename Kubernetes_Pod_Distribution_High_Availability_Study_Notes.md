# Kubernetes Pod Distribution Across 3 Worker Nodes

## Interview question

**Suppose a Kubernetes cluster has 3 worker nodes. For high availability, you want the Pods of a Deployment distributed across all 3 nodes. How would you achieve this?**

Two Kubernetes scheduling features address this requirement:

1. **Topology Spread Constraints** — control how evenly matching Pods are distributed across topology domains.
2. **Pod Anti-Affinity** — keep matching Pods apart, including on separate worker nodes.

In both examples below, `topologyKey: kubernetes.io/hostname` identifies each worker node as a separate topology domain.

## 1. Topology Spread Constraints

Topology Spread Constraints tell the Kubernetes scheduler how to distribute matching Pods across nodes (or other topology domains, such as zones).

### Example Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: my-app
      containers:
        - name: my-app
          image: nginx:1.27
          ports:
            - containerPort: 80
```

### Important fields

| Field | Meaning |
|---|---|
| `topologySpreadConstraints` | Scheduling rules for spreading matching Pods. |
| `topologyKey: kubernetes.io/hostname` | Treat each worker node as a separate topology domain. |
| `maxSkew: 1` | Limit the permitted imbalance in matching Pod counts between eligible domains to 1 under the scheduler's skew calculation. |
| `whenUnsatisfiable: DoNotSchedule` | Leave a new Pod Pending if scheduling it would violate the spread constraint. |
| `labelSelector.matchLabels.app: my-app` | Count matching Pods labeled `app: my-app` for the constraint. |

**Expected distribution:** With 3 replicas and 3 eligible worker nodes, and assuming sufficient resources and no conflicting scheduling rules, Kubernetes can place one Pod on each node.

```text
Worker Node 1    Worker Node 2    Worker Node 3
   my-app           my-app           my-app
   Pod 1            Pod 2            Pod 3
```

**What happens if a node is unavailable?** With `DoNotSchedule`, a replacement Pod can remain Pending when placing it on an available node would break the configured spread rule. The scheduler does not move already-running Pods merely because the distribution later changes.

**Alternative:** `whenUnsatisfiable: ScheduleAnyway` makes spreading a preference rather than a hard scheduling requirement; the scheduler can place a Pod even when the desired balance cannot be maintained.

## 2. Pod Anti-Affinity

Pod Anti-Affinity instructs Kubernetes to avoid placing matching Pods together in the same topology domain. Using the hostname topology key means separating them by node.

### Example Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: my-app
              topologyKey: kubernetes.io/hostname
      containers:
        - name: my-app
          image: nginx:1.27
          ports:
            - containerPort: 80
```

### Important fields

| Field | Meaning |
|---|---|
| `podAntiAffinity` | Defines rules for separating a Pod from other matching Pods. |
| `requiredDuringSchedulingIgnoredDuringExecution` | A hard rule checked when scheduling; existing Pods are not evicted just because conditions later change. |
| `labelSelector.matchLabels.app: my-app` | Identifies the Pods that must not share the selected topology domain. |
| `topologyKey: kubernetes.io/hostname` | Apply separation at the worker-node level. |

**Expected distribution:** With 3 replicas and 3 eligible worker nodes, the scheduler can place one matching Pod per node.

```text
Worker Node 1    Worker Node 2    Worker Node 3
   my-app           my-app           my-app
   Pod 1            Pod 2            Pod 3
```

**What happens if a node is unavailable?** If only 2 eligible nodes are available and each already hosts a matching Pod, a third Pod cannot be scheduled under this required anti-affinity rule and remains Pending.

**Alternative:** `preferredDuringSchedulingIgnoredDuringExecution` expresses a preference to separate matching Pods, but permits co-location if needed.

## 3. Topology Spread Constraints vs. Pod Anti-Affinity

| Aspect | Topology Spread Constraints | Pod Anti-Affinity |
|---|---|---|
| Main purpose | Balance matching Pods across topology domains. | Separate matching Pods across topology domains. |
| Core configuration | `maxSkew`, `topologyKey`, `whenUnsatisfiable`, `labelSelector` | `podAntiAffinity`, `labelSelector`, `topologyKey` |
| Example on 3 nodes | Aim for 1 Pod on each node with 3 replicas. | Prevent 2 matching Pods from sharing a node with required anti-affinity. |
| With more replicas than nodes | Can distribute multiple Pods per node while respecting the skew limit. | Required node-level anti-affinity can leave extra Pods Pending. |
| Strict vs. flexible | `DoNotSchedule` vs. `ScheduleAnyway` | `requiredDuringSchedulingIgnoredDuringExecution` vs. `preferredDuringSchedulingIgnoredDuringExecution` |

### Important distinction with 6 replicas and 3 nodes

- **Topology Spread Constraints (`maxSkew: 1`):** Can distribute 2 Pods per node, assuming sufficient resources and eligible nodes.
- **Required Pod Anti-Affinity (same app label, hostname topology):** Allows at most one matching Pod per node; with only 3 eligible nodes, additional replicas remain Pending.

## 4. Apply and verify

Save either example as `deployment.yml` and run:

```bash
kubectl apply -f deployment.yml
kubectl get pods -o wide
kubectl get nodes -L kubernetes.io/hostname
```

Inspect the `NODE` column in `kubectl get pods -o wide` to confirm that the 3 Pods run on 3 different worker nodes. If a Pod remains Pending, inspect scheduling events:

```bash
kubectl describe pod <pending-pod-name>
```

## 5. Interview-ready summary

**Topology Spread Constraints** spread matching Pods as evenly as the configured skew allows. Setting `topologyKey: kubernetes.io/hostname` spreads them by node, and `maxSkew: 1` limits imbalance.

**Pod Anti-Affinity** separates matching Pods. With required anti-affinity and `topologyKey: kubernetes.io/hostname`, two matching Pods cannot be scheduled on the same node.

Both can improve availability by reducing the risk that one worker-node failure affects all replicas. Neither alone guarantees application high availability: the cluster still needs enough eligible nodes and capacity, and the application must be able to serve traffic through its remaining healthy Pods.
