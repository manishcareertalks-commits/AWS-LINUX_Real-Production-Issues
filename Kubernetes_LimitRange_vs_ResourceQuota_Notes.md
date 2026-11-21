# Kubernetes LimitRange vs ResourceQuota

**Interview question:** What is the difference between LimitRange and ResourceQuota, and how do we use them in production?

<img width="638" height="631" alt="image" src="https://github.com/user-attachments/assets/7e7e6e53-78b8-4d2a-b186-ea538df66f7c" />


<img width="589" height="620" alt="image" src="https://github.com/user-attachments/assets/415fc120-bccf-4897-ab22-e565f2aca175" />


## 1. Production scenario

Imagine Development, Testing, and Production teams sharing an Amazon EKS cluster, each with its own Kubernetes namespace. We need to stop individual workloads from specifying unreasonable CPU/memory resources **and** stop any one namespace from consuming the team's entire resource budget.

- **LimitRange:** Applies minimum, maximum, and optional default resource requests/limits to individual containers or Pods within a namespace.
- **ResourceQuota:** Enforces an aggregate budget for resources such as CPU requests, memory requests, CPU/memory limits, and Pod counts within a namespace.

Both are **namespace-scoped**. They complement each other rather than replace each other.

## 2. LimitRange example (`limitrange.yaml`)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: container-limits
  namespace: development
spec:
  limits:
    - type: Container
      min:
        cpu: "100m"
        memory: "128Mi"
      max:
        cpu: "1"
        memory: "1Gi"
      defaultRequest:
        cpu: "200m"
        memory: "256Mi"
      default:
        cpu: "500m"
        memory: "512Mi"
```

**What it means:** Each container is constrained by the configured minimum and maximum values. For newly admitted containers that omit resource settings, Kubernetes can apply the configured default requests and limits. `100m` = 0.1 CPU core; `1Gi` = 1,024 MiB.

## 3. ResourceQuota example (`resourcequota.yaml`)

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: development-quota
  namespace: development
spec:
  hard:
    requests.cpu: "4"
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "10"
```

**What it means:** Across the **development** namespace, the sum of CPU requests cannot exceed 4 cores, memory requests cannot exceed 8 GiB, CPU limits cannot exceed 8 cores, memory limits cannot exceed 16 GiB, and no more than 10 non-terminal Pods can count toward the Pod quota. Creating or updating an object that would exceed a quota is rejected by admission control.

## 4. Key differences

| Feature | LimitRange | ResourceQuota |
|---|---|---|
| Scope | Individual container/Pod resource settings | Total namespace budget |
| Main purpose | Enforce min/max and set defaults | Cap aggregate requests, limits, object counts |
| Defaults | Can inject default requests/limits | Does not inject container defaults |
| Example | Container CPU limit <= 1 core | Namespace CPU requests <= 4 cores |
| When exceeded | Noncompliant Pod admission rejected | Creation/update exceeding quota rejected |

## 5. Apply and verify

```bash
kubectl create namespace development
kubectl apply -f limitrange.yaml
kubectl apply -f resourcequota.yaml

kubectl get limitrange,resourcequota -n development
kubectl describe limitrange container-limits -n development
kubectl describe resourcequota development-quota -n development
kubectl get pods -n development
```

## 6. Production best practices

1. Create separate namespaces for teams/environments, with suitable RBAC permissions.
2. Set realistic container requests/limits using observed CPU and memory usage; avoid unnecessarily tight limits.
3. Use LimitRange defaults to make resource configuration more consistent, and ResourceQuota to enforce namespace budgets.
4. Monitor quota usage, pending/rejected deployments, and actual application performance. Plan headroom for rollouts and autoscaling.
5. Remember: these controls do **not** reserve cluster capacity or guarantee scheduling. CPU limits may cause throttling, and memory limits can cause OOM kills.

**Interview-ready summary:** *LimitRange controls resource settings for individual containers or Pods, while ResourceQuota limits the aggregate resources available to a namespace. In production, we use both together for workload governance and fairer resource allocation across teams.*
