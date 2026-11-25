# Kubernetes Service Mesh (Istio) 
## Production interview study notes

# Instagram link -
https://www.instagram.com/manishcareertalks/?hl=en
# Youtube Link - 
https://www.youtube.com/@ManishCareerTalks

### 1. Interview scenario
**You have 4 microservices running in a production Kubernetes cluster. How would you secure, monitor and manage communication between them without modifying application code?**

<img width="672" height="672" alt="image" src="https://github.com/user-attachments/assets/bc3114d6-2658-4ab4-9fe8-e92a76a97030" />

Use a service mesh such as Istio to centrally manage networking policies. Consider an e-commerce application on AWS EKS with four representative services: Order, Payment, Inventory and Notification.

### 2. What is a Service Mesh?
A dedicated infrastructure layer for service-to-service communication. It provides security, traffic management, resilience and observability without implementing these networking features separately in each application.

**Istio sidecar architecture:**
- **Data plane (Envoy):** A proxy alongside the application container in each participating Pod intercepts configured inbound and outbound traffic and enforces policies.
- **Control plane (Istiod):** Distributes proxy configuration and manages workload identity certificates. Application requests do not flow through Istiod.
- **Kubernetes Services:** Continue to provide service discovery and stable service names.

```text
                 Istiod (Control Plane)
                   | Configuration
          +--------+---------+
          v                  v
  Order Pod              Payment Pod
  [App + Envoy] <--mTLS--> [Envoy + App]
```

### 3. Four production benefits

#### Security — mTLS and access control
Mutual TLS encrypts traffic and authenticates both workloads. Authorization policies restrict which identities may access a workload.

Example: allow Order to access Payment using its dedicated ServiceAccount; deny other callers unless another applicable policy permits them.

`peer-authentication.yaml`:
```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata: {name: default, namespace: production}
spec:
  mtls: {mode: STRICT}
```
`authorization-policy.yaml`:
```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata: {name: payment-access, namespace: production}
spec:
  selector:
    matchLabels: {app: payment}
  action: ALLOW
  rules:
    - from:
        - source:
            principals: [cluster.local/ns/production/sa/order-sa]
```

#### Traffic management — Canary deployment
Send 90% of requests to Payment v1 and 10% to Payment v2. Monitor the new version before increasing its traffic share.

`destination-rule.yaml` and `virtual-service.yaml`:
```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata: {name: payment, namespace: production}
spec:
  host: payment
  subsets:
    - {name: v1, labels: {version: v1}}
    - {name: v2, labels: {version: v2}}
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata: {name: payment, namespace: production}
spec:
  hosts: [payment]
  http:
    - route:
        - destination: {host: payment, subset: v1}
          weight: 90
        - destination: {host: payment, subset: v2}
          weight: 10
```
Traffic weights are independent of replica counts; observed percentages may vary over short intervals.

#### Reliability — Retries, timeouts and circuit breaking
If Inventory becomes slow or returns errors, configure timeouts and bounded retries for suitable requests. Connection-pool limits reduce overload; outlier detection temporarily ejects failing endpoints.

`inventory-reliability.yaml`:
```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata: {name: inventory, namespace: production}
spec:
  hosts: [inventory]
  http:
    - match:
        - method: {exact: GET}
      timeout: 3s
      retries: {attempts: 2, perTryTimeout: 1s, retryOn: "5xx,connect-failure,reset"}
      route:
        - destination: {host: inventory}
    - route:
        - destination: {host: inventory}
---
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata: {name: inventory, namespace: production}
spec:
  host: inventory
  trafficPolicy:
    connectionPool:
      tcp: {maxConnections: 100}
    outlierDetection:
      consecutive5xxErrors: 5
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```
Avoid blind retries of non-idempotent payment transactions. Tune limits to workload capacity and latency objectives.

#### Observability — Metrics, logs and traces
- **Prometheus:** Collects metrics such as request rate, latency and errors.
- **Grafana:** Visualizes metrics and traffic distribution.
- **Jaeger:** Analyzes distributed traces to identify slow or failing services.

`telemetry.yaml`:
```yaml
apiVersion: telemetry.istio.io/v1
kind: Telemetry
metadata: {name: mesh-telemetry, namespace: production}
spec:
  metrics:
    - providers: [{name: prometheus}]
  tracing:
    - providers: [{name: otel}]
      randomSamplingPercentage: 10
  accessLogging:
    - providers: [{name: envoy}]
```
Requires configured providers and monitoring backends. End-to-end traces generally require applications to propagate trace context headers.

### 4. Quick revision
| Requirement | Istio resource |
|---|---|
| Enforce mTLS | PeerAuthentication |
| Restrict access | AuthorizationPolicy |
| Traffic routing, retries, timeouts | VirtualService |
| Subsets, circuit breaking, outlier detection | DestinationRule |
| Metrics, logs, tracing | Telemetry |

### 5. Production considerations
These illustrative manifests assume Istio is installed, workloads are enrolled in the mesh, and matching Services, labels, ServiceAccounts and telemetry providers exist. Validate policies before rollout; monitor proxy resource overhead and latency. Manage configurations through GitOps (e.g., Argo CD). Istio also supports ambient mode as an alternative to sidecars.

**Interview takeaway:** Istio centralizes service-to-service networking policies, helping make microservice communication secure, reliable and observable.

Reference: https://istio.io/latest/docs/
