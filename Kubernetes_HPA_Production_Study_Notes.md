# Kubernetes Horizontal Pod Autoscaler (HPA) — Production Study Notes

## Interview scenario
**Question:** An application is running in a Kubernetes cluster with 3 Pods. Traffic suddenly spikes and the Pods struggle to handle the load. How would you automatically scale the Pods in production?

**Answer:** Configure a Horizontal Pod Autoscaler (HPA) on the application's Deployment. HPA adjusts the Deployment's desired replica count based on observed CPU utilization (or configured memory, custom, or external metrics). CPU-based HPA is an indirect response to traffic: high traffic must actually increase CPU demand for this metric to trigger scaling.

## 1. Investigate and verify prerequisites

```bash
kubectl get pods -n production
kubectl top pods -n production
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
kubectl describe deployment myapp -n production
```

- `kubectl top` reports CPU usage in millicores and memory usage, **not CPU utilization percentages**. CPU utilization for HPA is measured relative to configured CPU **requests**.
- Ensure the resource metrics API is available (typically via Metrics Server) and that the relevant containers have CPU requests set; otherwise CPU-based HPA may not calculate a valid metric.
- Investigate application latency, errors, throttling, and available node capacity as well as CPU usage.

Example container resource configuration (inside the Deployment Pod template):

```yaml
resources:
  requests:
    cpu: "1000m"
    memory: "512Mi"
  limits:
    memory: "1Gi"
```

If a container requests `1000m` CPU and consumes `900m`, its CPU utilization relative to its request is **90%**.

## 2. Configure HPA (hpa.yaml)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

```bash
kubectl apply -f hpa.yaml
kubectl get hpa -n production
kubectl describe hpa myapp-hpa -n production
kubectl get pods -n production -w
```

**Interpretation:** HPA attempts to keep average CPU utilization near 70%, while requesting between 3 and 20 replicas. It does **not** mean that every reading above 70% immediately adds a Pod.

## 3. How HPA calculates desired Pods

**Simplified formula:**

`desiredReplicas = ceil(currentReplicas × currentUtilization / targetUtilization)`

For **3 Pods**, **90% current average CPU utilization**, and **70% target**:

`ceil(3 × 90 / 70) = ceil(3.857) = 4 Pods`

So HPA may update the Deployment's desired replicas from **3 to 4**. Kubernetes schedules the new Pod, subject to available capacity and readiness.

| Current average CPU | Simplified calculation | Desired replicas |
|---|---|---:|
| 70% | ceil(3 × 70 / 70) | 3 |
| 80% | ceil(3 × 80 / 70) | 4 |
| 90% | ceil(3 × 90 / 70) | 4 |
| 100% | ceil(3 × 100 / 70) | 5 |
| 120% | ceil(3 × 120 / 70) | 6 |

**Real-world nuance:** HPA has a default 10% tolerance around the target and accounts for missing metrics, not-yet-ready Pods, scale policies, and stabilization. For example, 70% → 75% does not necessarily trigger scaling. The table is illustrative, not a guarantee of immediate changes.

## 4. Node capacity and safe production scaling

- **HPA scales Pods, not worker nodes.** If new Pods cannot be scheduled, they may remain `Pending`. On Amazon EKS, use **Karpenter** or **Cluster Autoscaler** to add node capacity where configured.
- Configure **readiness probes** so Services route traffic only to ready Pods; use graceful termination and suitable rolling-update settings to protect availability.
- Monitor CPU, request latency, error rates, HPA events, pending Pods, and cluster capacity using tools such as Prometheus/Grafana.
- HPA scales down when demand decreases, subject to its scale-down stabilization behavior (commonly a 5-minute default window).
- HPA does not guarantee zero downtime; scaling and node provisioning take time. Keep enough baseline capacity for sudden spikes.

## Quick interview conclusion

“We would verify Pod resource usage and Metrics Server, ensure CPU requests are configured, and apply an HPA with 3 minimum replicas, 20 maximum replicas, and a 70% CPU target. If average CPU reaches 90% with 3 Pods, the simplified HPA calculation gives 4 desired Pods. We would verify scaling with `kubectl get hpa`, monitor application health, and use Karpenter or Cluster Autoscaler if additional worker-node capacity is required.”

**Reference:** Kubernetes documentation — Horizontal Pod Autoscaling: https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/
