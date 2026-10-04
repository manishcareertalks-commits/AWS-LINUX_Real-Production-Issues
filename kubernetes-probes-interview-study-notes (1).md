# Kubernetes Probes — Interview & Study Notes

<img width="605" height="815" alt="image" src="https://github.com/user-attachments/assets/f161aa91-d87c-41ef-b483-312b47c02f20" />


## 1. What are Kubernetes Probes?

Kubernetes **probes** are health checks used by the kubelet to determine the state of an application running inside a Pod.

Kubernetes provides three main probe types:

| Probe | Main Question | Typical Action |
|---|---|---|
| **Liveness** | Should the container be restarted? | Restart the container when the application is unhealthy |
| **Readiness** | Is the Pod ready to receive traffic? | Remove the Pod from Service endpoints while not ready |
| **Startup** | Has the application finished starting? | Delay liveness/readiness checks during slow startup |

### Why are probes important?

Without probes, Kubernetes may consider a container healthy simply because the process is running.

Example:

```text
Container process → Running
Application       → Stuck / Deadlocked
```

A properly configured probe allows Kubernetes to react to the **actual application state**.

---

# 2. Liveness Probe

## What does it do?

A **liveness probe** checks whether the application is still functioning correctly.

If the liveness probe fails repeatedly, Kubernetes considers the container unhealthy and the kubelet restarts the container according to the Pod's restart policy.

### Production scenario

A Java/Node.js application gets stuck because of:

- Deadlock
- Infinite loop
- Application thread becoming unresponsive
- Internal state becoming corrupted

The container process may still be running, but the application cannot serve requests.

```text
Pod
 └── Container
      └── Application
           └── Stuck / Unhealthy
                  ↓
             Liveness fails
                  ↓
          Container restarted
```

### Example

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

### Key interview point

> **Liveness = Should the container be restarted?**

Do not use a liveness probe to simply check whether the application is ready to receive traffic. That is the job of the readiness probe.

---

# 3. Readiness Probe

## What does it do?

A **readiness probe** checks whether the application is ready to handle requests.

If the readiness probe fails, Kubernetes marks the Pod as **NotReady** and the Pod is removed from the normal Service traffic path.

The container is **not restarted** just because the readiness probe fails.

### Production scenario

An application starts successfully but needs additional time to:

- Load configuration
- Establish connections
- Load cache
- Initialize dependencies
- Warm up

During this period:

```text
Pod → Running
Application → Not ready
       ↓
Readiness probe fails
       ↓
Pod excluded from Service traffic
       ↓
Application becomes ready
       ↓
Pod receives traffic
```

### Example

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

### Key interview point

> **Readiness = Is the Pod ready to receive traffic?**

A readiness failure normally affects **traffic routing**, not container lifecycle.

---

# 4. Startup Probe

## What does it do?

A **startup probe** is designed for applications that take a long time to initialize.

While the startup probe is running, Kubernetes does not run the configured **liveness and readiness probes for that container**.

Once the startup probe succeeds, the normal liveness and readiness checks begin.

### Production scenario

Suppose a large Java application takes **90 seconds** to start.

Without a startup probe, an aggressive liveness probe might consider the application unhealthy before initialization completes and repeatedly restart it.

With a startup probe:

```text
Container starts
      ↓
Startup probe checks application
      ↓
Application initializes
      ↓
Startup probe succeeds
      ↓
Liveness + Readiness probes start
```

### Example

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  periodSeconds: 10
  failureThreshold: 18

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  periodSeconds: 5
```

Here, the startup probe can allow approximately **180 seconds** for startup (`18 × 10s`) before the container is considered to have failed startup.

### Key interview point

> **Startup = Has the application finished starting?**

---

# 5. Probe Mechanisms

Kubernetes supports different ways to perform probe checks.

### HTTP GET

Used when the application exposes an HTTP health endpoint.

```yaml
httpGet:
  path: /health
  port: 8080
```

### TCP Socket

Checks whether a TCP connection can be established.

```yaml
tcpSocket:
  port: 8080
```

Useful when an application does not expose an HTTP health endpoint.

### Exec

Runs a command inside the container.

```yaml
exec:
  command:
    - cat
    - /tmp/healthy
```

The command's exit code determines success or failure.

### gRPC

For applications exposing a gRPC health-check endpoint.

```yaml
grpc:
  port: 50051
```

---

# 6. Important Probe Configuration

Common fields used to control probe behavior:

| Field | Purpose |
|---|---|
| `initialDelaySeconds` | Delay before the first probe |
| `periodSeconds` | How frequently the probe runs |
| `timeoutSeconds` | Maximum time allowed for a probe |
| `successThreshold` | Consecutive successes required |
| `failureThreshold` | Consecutive failures required |
| `terminationGracePeriodSeconds` | Grace period before forced termination |

### Example

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

---

# 7. Liveness vs Readiness vs Startup

| Feature | Liveness | Readiness | Startup |
|---|---|---|---|
| Main purpose | Detect unhealthy application | Control traffic | Handle slow startup |
| Main question | Restart? | Receive traffic? | Finished starting? |
| Failure effect | Container may restart | Pod removed from Service traffic | Container considered failed if startup never succeeds |
| Used during startup? | Normally after startup probe succeeds | Normally after startup probe succeeds | Yes |
| Typical example | Deadlock | Loading configuration | Large Java application |

### Easy way to remember

```text
STARTUP
   ↓
Has the application started?

READINESS
   ↓
Can it receive traffic?

LIVENESS
   ↓
Is it still healthy?
```

---

# 8. Common Interview Questions

### Q1. What is the difference between liveness and readiness?

**Liveness** determines whether the container should be restarted.

**Readiness** determines whether the Pod should receive traffic.

---

### Q2. What happens when a readiness probe fails?

The Pod is marked **NotReady** and is removed from the normal Service traffic path. The container is not automatically restarted just because readiness failed.

---

### Q3. Why do we need a startup probe if we already have liveness?

For slow-starting applications. Startup probes prevent liveness checks from restarting the application before it has finished initialization.

---

### Q4. Can we use all three probes together?

**Yes.** A common production pattern is:

```text
Startup Probe
      ↓
Liveness + Readiness
```

Startup protects the initialization phase, liveness detects stuck applications, and readiness controls traffic.

---

### Q5. Does a failed liveness probe kill the Pod?

The kubelet restarts the affected container according to the Pod's restart policy. The Pod object itself is not necessarily recreated.

---

# 9. Production Best Practices

- Keep **liveness** focused on whether the application is fundamentally stuck or unhealthy.
- Keep **readiness** focused on whether the application can safely receive traffic.
- Use **startup probes** for applications with unpredictable or long startup times.
- Avoid making health endpoints dependent on too many external services unless that behavior is intentional.
- Set realistic `timeoutSeconds`, `periodSeconds`, and `failureThreshold` values.
- Test probe behavior before deploying to production.
- Monitor probe failures using Kubernetes events and application/monitoring logs.

Useful commands:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl get events --sort-by=.lastTimestamp
kubectl logs <pod-name>
```

---

# Quick Revision

**Liveness** → *Should the container be restarted?*

**Readiness** → *Is the Pod ready to receive traffic?*

**Startup** → *Has the application finished starting?*

**Remember:**

> **Startup protects initialization → Readiness controls traffic → Liveness protects application health.**
