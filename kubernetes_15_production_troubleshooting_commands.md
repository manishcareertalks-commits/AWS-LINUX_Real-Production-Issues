# Top 15 Kubernetes Troubleshooting Commands for Production

**Practical DevOps / SRE notes**  
Examples use namespace `prod`, application `webapp`, and placeholder Pod/node names. Replace these with your environment's values. Run read-only diagnostics first and follow production access controls.

## Quick reference

| # | Purpose | Command |
|---|---|---|
| 1 | Pod status, IP and node placement | `kubectl get pods -o wide -n prod` |
| 2 | Previous terminated container logs | `kubectl logs <pod-name> -n prod --previous` |
| 3 | Execute a shell inside a container | `kubectl exec -it <pod-name> -n prod -- /bin/sh` |
| 4 | HPA scaling status | `kubectl get hpa -n prod` |
| 5 | Deployment replica status | `kubectl get deploy -n prod` |
| 6 | Worker node readiness | `kubectl get nodes` |
| 7 | Node conditions, taints and events | `kubectl describe node <node-name>` |
| 8 | Recent namespace events | `kubectl get events -n prod --sort-by='.metadata.creationTimestamp'` |
| 9 | Pod CPU and memory usage | `kubectl top pods -n prod` |
| 10 | Node CPU and memory usage | `kubectl top nodes` |
| 11 | Service selectors and ports | `kubectl describe svc webapp-svc -n prod` |
| 12 | Service EndpointSlices | `kubectl get endpointslices -n prod -l kubernetes.io/service-name=webapp-svc` |
| 13 | Deployment rollout progress | `kubectl rollout status deployment/webapp -n prod` |
| 14 | Deployment revision history | `kubectl rollout history deployment/webapp -n prod` |
| 15 | PodDisruptionBudget status | `kubectl get pdb -n prod` |

---

## 1. Inspect Pods with detailed placement

```bash
kubectl get pods -o wide -n prod
```

**Use when:** A Pod is Pending, restarting, or unreachable. Look at `READY`, `STATUS`, `RESTARTS`, Pod `IP`, and `NODE`. `-o wide` adds placement/network fields; it does not show all container details.

## 2. Retrieve logs from the previous container instance

```bash
kubectl logs <pod-name> -n prod --previous
# For multi-container Pods:
kubectl logs <pod-name> -c <container-name> -n prod --previous
```

**Use when:** A container restarted and its current logs no longer show the original error, such as during CrashLoopBackOff. **Important:** `--previous` retrieves the previous *terminated container instance within the same Pod*, if its logs are retained. It **does not** recover logs from an entirely deleted Pod. For deleted Pods, use centralized logging such as CloudWatch Logs, Loki, or OpenSearch if logs were collected before deletion.

## 3. Execute commands inside a running container

```bash
kubectl exec -it <pod-name> -n prod -- /bin/sh
# Or run one command without an interactive shell:
kubectl exec <pod-name> -n prod -- printenv
```

**Use when:** You need to inspect files, DNS resolution, or connectivity. The image must contain the requested executable; minimal/distroless images may not include `/bin/sh`. Use `-c <container-name>` for multi-container Pods. Avoid printing sensitive environment variables in shared terminals or recordings.

## 4. Check Horizontal Pod Autoscaler (HPA)

```bash
kubectl get hpa -n prod
kubectl describe hpa webapp-hpa -n prod
```

**Use when:** Traffic increases but replicas do not scale. Review targets, current metrics, min/max replicas, conditions, and events. HPA resource metrics typically require a working metrics API and relevant resource requests.

## 5. Check Deployment health

```bash
kubectl get deploy -n prod
kubectl describe deployment webapp -n prod
```

**Use when:** A release has fewer ready replicas than expected. Compare `READY`, `UP-TO-DATE`, and `AVAILABLE` and inspect Deployment conditions/events.

## 6. Check worker node readiness

```bash
kubectl get nodes
```

**Use when:** Pods cannot schedule or nodes stop serving workloads. Look for `NotReady` and `SchedulingDisabled`. A node being `Ready` does not guarantee every workload on it is healthy.

## 7. Inspect node conditions and events

```bash
kubectl describe node <node-name>
```

**Use when:** Pods are evicted, scheduling fails, or a node becomes NotReady. Inspect `MemoryPressure`, `DiskPressure`, `PIDPressure`, taints, allocated resources, and events.

## 8. Review Kubernetes events

```bash
kubectl get events -n prod --sort-by='.metadata.creationTimestamp'
```

**Use when:** You see `FailedScheduling`, `FailedMount`, `BackOff`, or image pull problems. Events are retained for a limited time and are not a permanent audit log.

## 9. Check Pod resource usage

```bash
kubectl top pods -n prod
kubectl top pods -n prod --containers
```

**Use when:** Investigating high CPU/memory consumption. Requires Metrics Server or another compatible Metrics API provider. `top` shows recent usage, not historical trends or CPU throttling directly.

## 10. Check node resource usage

```bash
kubectl top nodes
```

**Use when:** Investigating resource hotspots or overloaded nodes. Compare usage with node capacity/allocatable resources and longer-term monitoring.

## 11. Inspect Service configuration

```bash
kubectl describe svc webapp-svc -n prod
```

**Use when:** Pods are Running but the application is inaccessible. Verify selectors, Service type, ports, target ports, and reported endpoints. Follow up with EndpointSlices and Pod readiness.

## 12. Check Service EndpointSlices

```bash
kubectl get endpointslices -n prod -l kubernetes.io/service-name=webapp-svc
kubectl describe endpointslice <slice-name> -n prod
```

**Use when:** Traffic is not reaching expected backends. Verify endpoint addresses, ports, and readiness conditions. A Service may have multiple EndpointSlices.

## 13. Monitor a Deployment rollout

```bash
kubectl rollout status deployment/webapp -n prod
```

**Use when:** Checking whether a rolling update is progressing or stuck. For more context, inspect Deployment conditions, ReplicaSets, Pods, and events.

## 14. Inspect Deployment revision history

```bash
kubectl rollout history deployment/webapp -n prod
kubectl rollout history deployment/webapp --revision=2 -n prod
```

**Use when:** Investigating which Deployment revision introduced a problem. History depends on retained ReplicaSets and may not include meaningful change-cause annotations. **It does not contain historical Pod logs.** A rollback is a change operation requiring production approval.

## 15. Check PodDisruptionBudget (PDB)

```bash
kubectl get pdb -n prod
kubectl describe pdb webapp-pdb -n prod
```

**Use when:** Node draining or voluntary disruption is blocked. Check `ALLOWED DISRUPTIONS`, desired/current healthy Pods, and the PDB selector. PDBs govern certain voluntary disruptions, not every outage or node failure.

---

## Practical incident workflow

1. **Identify impact:** `kubectl get pods -o wide -n prod`, `kubectl get deploy -n prod`, `kubectl get nodes`.
2. **Find why:** `kubectl get events -n prod --sort-by='.metadata.creationTimestamp'`, `kubectl describe node <node-name>` or `kubectl describe deployment webapp -n prod`.
3. **Check application failures:** `kubectl logs <pod-name> -n prod --previous` for restarted containers; use centralized logs for deleted Pods.
4. **Investigate resources and scaling:** `kubectl top pods -n prod`, `kubectl top nodes`, `kubectl describe hpa webapp-hpa -n prod`.
5. **Trace traffic:** `kubectl describe svc webapp-svc -n prod` and inspect its EndpointSlices.
6. **Verify release health:** `kubectl rollout status deployment/webapp -n prod`, `kubectl rollout history deployment/webapp -n prod`.
7. **Before node maintenance:** `kubectl get pdb -n prod` and verify disruption budgets.

> **Interview takeaway:** `kubectl logs --previous` is for the prior container instance in an existing Pod—not for an already deleted Pod. Always distinguish container restarts from Pod replacement.

**Suggested reel CTA:** Comment **KUBERNETES** + Follow to get detailed notes.
