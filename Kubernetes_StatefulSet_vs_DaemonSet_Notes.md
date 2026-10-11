# Kubernetes StatefulSet vs DaemonSet — Quick Study Notes

## 1. StatefulSet

A **StatefulSet** manages applications that need stable pod identities and may need persistent storage.

- Pods get stable, ordered names such as `kafka-0`, `kafka-1`, and `kafka-2`.
- Can use `volumeClaimTemplates` to provision a separate PersistentVolumeClaim for each pod.
- Useful when applications depend on stable identity or persistent data.
- **Production examples:** Kafka brokers, database clusters, and distributed data systems (depending on their architecture).

**Example:** A Kafka cluster has three broker pods. StatefulSet provides predictable pod names and can associate each broker with persistent storage.

## 2. DaemonSet

A **DaemonSet** ensures that a copy of a pod runs on every **eligible node** in the cluster.

- When a new eligible node joins, Kubernetes schedules the DaemonSet pod on it.
- Pods may not run on every node if scheduling rules or node eligibility prevent placement.
- Commonly used for node-level monitoring, logging, networking, and security agents.
- **Production examples:** Prometheus Node Exporter, Fluent Bit, and node-level security agents.

**Example:** A cluster has 10 eligible worker nodes. A Node Exporter DaemonSet runs one monitoring-agent pod on each eligible node.

## 3. Difference Table

| Feature | StatefulSet | DaemonSet |
|---|---|---|
| Main purpose | Manage stateful workloads with stable identities | Run a pod on every eligible node |
| Pod identity | Stable ordinal names, e.g. `app-0`, `app-1` | No StatefulSet-style ordinal identity |
| Scheduling | According to normal scheduling constraints | Targets each eligible node |
| Storage | Supports per-pod PVCs using `volumeClaimTemplates` | Can use volumes, depending on configuration |
| Scaling | Change replicas to add or remove pods | Pods follow the eligible node population |
| Typical use | Kafka, databases, distributed data systems | Node metrics, log collection, node agents |
| Example | `kafka-0`, `kafka-1`, `kafka-2` | One Node Exporter pod per eligible node |

## 4. Production Monitoring Flow

**Node Exporter → Prometheus → Grafana**

- **Node Exporter:** Exposes host-level metrics such as CPU, memory, and disk.
- **Prometheus:** Scrapes and stores metrics from configured targets.
- **Grafana:** Displays metrics in dashboards.

Node Exporter is commonly deployed as a **DaemonSet** so each eligible node is monitored. Prometheus is commonly deployed as a **Deployment or StatefulSet**, depending on the architecture. Grafana is commonly deployed as a **Deployment**. Prometheus and Grafana do not inherently require a DaemonSet.

## 5. Interview Takeaways

- **StatefulSet = stable pod identity + persistent storage when configured.**
- **DaemonSet = one pod on every eligible node.**
- StatefulSet does **not** automatically provide database high availability; replication, quorum, and application configuration still matter.
- DaemonSet does **not** guarantee a pod on absolutely every node; selectors, taints, tolerations, and other scheduling rules affect eligibility.
