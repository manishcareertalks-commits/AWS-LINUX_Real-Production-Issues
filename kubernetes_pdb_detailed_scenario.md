# Kubernetes PodDisruptionBudget (PDB)

## Production Scenario: Healthy Pods but Pod Eviction Is Blocked

### 1. Scenario

Imagine a production application running on Kubernetes with **6
replicas**.

All 6 Pods are healthy and `Ready`.

Now the infrastructure team needs to perform **planned maintenance** on
a node. The node needs to be drained, so Kubernetes attempts to evict
the Pods running on that node.

However, the eviction is blocked.

The key question is:

> **If the application is healthy, why can't Kubernetes evict the Pod?**

A common thing to check first is the application's **PodDisruptionBudget
(PDB)**.

------------------------------------------------------------------------

## 2. What is a PodDisruptionBudget?

A **PodDisruptionBudget** is a Kubernetes policy that limits how many
Pods from a selected application can be voluntarily disrupted at the
same time.

In simple terms:

> **PDB protects application availability during planned/voluntary
> disruptions.**

Examples of voluntary disruptions include:

-   `kubectl drain`
-   Planned node maintenance
-   Cluster/node upgrades
-   Other operations that use the Kubernetes Eviction API

A PDB does **not** guarantee that the application will always have the
configured number of healthy Pods. Unexpected events such as a node
crash can still make Pods unavailable.

------------------------------------------------------------------------

## 3. Why does PDB matter during node maintenance?

Consider this application:

``` text
Deployment
replicas: 6

Pod 1  Pod 2  Pod 3
Pod 4  Pod 5  Pod 6

All Pods: Ready
```

During planned maintenance, suppose one of these Pods needs to be
evicted.

If the PDB allows one disruption, Kubernetes can evict one Pod while
preserving the required availability.

If another eviction would violate the PDB, the eviction request can be
rejected temporarily.

This is why you can see:

``` text
Application: Healthy
        +
Planned Maintenance
        +
Pod Eviction
        ↓
     BLOCKED
```

The Pods themselves may be completely healthy. The **disruption policy**
is what is preventing the voluntary eviction.

------------------------------------------------------------------------

# 4. How is a PDB configured?

A PDB primarily uses:

-   `selector`
-   `minAvailable` **OR**
-   `maxUnavailable`

You normally configure **one of `minAvailable` or `maxUnavailable`**,
not both.

------------------------------------------------------------------------

## 5. Example using `minAvailable`

Suppose the application has 6 replicas and we want at least 5 Pods to
remain available during voluntary disruption.

``` yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  minAvailable: 5
  selector:
    matchLabels:
      app: my-app
```

The important part is:

``` yaml
minAvailable: 5
```

This means:

> At least 5 selected Pods must remain available after a voluntary
> eviction.

### Example

Initial state:

``` text
6 healthy Pods

🟢 🟢 🟢 🟢 🟢 🟢
```

Evict one:

``` text
5 healthy Pods

🟢 🟢 🟢 🟢 🟢 ❌
```

This satisfies:

``` text
minAvailable = 5
```

So the eviction can be allowed.

But if another eviction would result in only 4 available Pods:

``` text
🟢 🟢 🟢 🟢 ❌ ❌

Available = 4
Required  = 5
```

The next voluntary eviction is blocked because it would violate the PDB.

------------------------------------------------------------------------

# 6. Example using `maxUnavailable`

Instead of saying how many Pods must remain available, you can specify
how many can be unavailable.

For example:

``` yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
spec:
  maxUnavailable: 1
  selector:
    matchLabels:
      app: my-app
```

This means:

> At most 1 selected Pod can be unavailable due to the voluntary
> disruption being evaluated.

For a 6-replica application, this allows one disruption while preventing
a second disruption if it would exceed the budget.

`maxUnavailable` can also be expressed as a percentage.

Example:

``` yaml
maxUnavailable: 25%
```

------------------------------------------------------------------------

# 7. `minAvailable` vs `maxUnavailable`

  Configuration           Meaning
  ----------------------- ------------------------------------------------
  `minAvailable: 5`       Keep at least 5 selected Pods available
  `maxUnavailable: 1`     Allow at most 1 selected Pod to be unavailable
  `minAvailable: 80%`     Keep at least 80% available
  `maxUnavailable: 20%`   Allow up to 20% unavailable

For a fixed-size application, both styles can be useful.

A percentage or `maxUnavailable` can be convenient when the workload's
replica count changes because the budget can adapt to the desired
replica count.

------------------------------------------------------------------------

# 8. The `selector` is extremely important

The PDB needs to know **which Pods it protects**.

That is why the PDB contains a label selector:

``` yaml
selector:
  matchLabels:
    app: my-app
```

The selected Pods need to have the matching label:

``` yaml
metadata:
  labels:
    app: my-app
```

Conceptually:

``` text
PDB
 |
 | selector: app=my-app
 ↓
Pods with app=my-app
 |
 ├── Pod 1
 ├── Pod 2
 ├── Pod 3
 ├── Pod 4
 ├── Pod 5
 └── Pod 6
```

If the selector does not match the intended Pods, the PDB will not
protect the workload as expected.

------------------------------------------------------------------------

# 9. What happens during `kubectl drain`?

A typical maintenance operation may look like:

``` bash
kubectl drain <node-name> --ignore-daemonsets
```

`kubectl drain` attempts to evict Pods from the node.

Kubernetes uses the **Eviction API** for these voluntary disruptions,
and the eviction process respects PodDisruptionBudgets.

Conceptually:

``` text
kubectl drain
     |
     ↓
Eviction request
     |
     ↓
Check PDB
     |
     ├── Disruption allowed → Evict Pod
     |
     └── Disruption not allowed → Eviction rejected/retried
```

Therefore, a node drain can appear to be "stuck" when the PDB does not
currently allow another disruption.

------------------------------------------------------------------------

# 10. How do you troubleshoot the scenario?

If an interviewer gives you this scenario:

> "The application is healthy, but pod eviction is blocked during
> planned maintenance."

A good troubleshooting sequence is:

### Step 1 --- Check the PDB

``` bash
kubectl get pdb -n <namespace>
```

Look at:

``` text
NAME
MIN AVAILABLE
MAX UNAVAILABLE
ALLOWED DISRUPTIONS
```

The most important field for this scenario is often:

``` text
ALLOWED DISRUPTIONS
```

If it is `0`, Kubernetes currently does not have budget for another
voluntary disruption.

------------------------------------------------------------------------

### Step 2 --- Describe the PDB

``` bash
kubectl describe pdb <pdb-name> -n <namespace>
```

Check:

-   Selector
-   Min available
-   Max unavailable
-   Current healthy
-   Desired healthy
-   Expected Pods
-   Allowed disruptions
-   Events

------------------------------------------------------------------------

### Step 3 --- Check the complete PDB YAML

``` bash
kubectl get pdb <pdb-name> -n <namespace> -o yaml
```

Look at the `spec` and `status`.

A useful status section may look like:

``` yaml
status:
  currentHealthy: 6
  desiredHealthy: 5
  disruptionsAllowed: 1
  expectedPods: 6
```

This tells us:

``` text
currentHealthy      = 6
desiredHealthy      = 5
disruptionsAllowed  = 1
expectedPods        = 6
```

So one voluntary disruption is currently allowed.

------------------------------------------------------------------------

# 11. Why might `ALLOWED DISRUPTIONS` be zero?

For example:

``` text
Expected Pods:       6
Current Healthy:     5
Desired Healthy:     5
Allowed Disruptions: 0
```

The application is already at its minimum required healthy count.

If Kubernetes evicted another Pod:

``` text
Current healthy = 5
Required        = 5

Evict one
   ↓
Healthy = 4
   ↓
PDB violated
   ↓
Eviction blocked
```

This is a very common explanation for the interview scenario.

------------------------------------------------------------------------

# 12. Important: PDB does not protect against everything

A PDB primarily controls **voluntary disruptions**.

### Voluntary disruption

Examples:

-   Node drain
-   Planned maintenance
-   Cluster upgrades
-   Eviction API requests

PDB can influence these.

### Involuntary disruption

Examples:

-   Node suddenly crashes
-   Hardware failure
-   Network failure
-   Unexpected infrastructure failure

A PDB cannot prevent these failures.

For example:

``` text
Node crashes
     ↓
Pods disappear
     ↓
PDB cannot prevent the crash
```

However, those unavailable Pods can still affect the application's
disruption budget and recovery behavior.

------------------------------------------------------------------------

# 13. PDB does not control Deployment rolling updates

Another important interview point:

A PDB is not the mechanism that controls a Deployment's normal rolling
update behavior.

For example, a Deployment's rolling update is controlled by fields such
as:

``` yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```

PDB and Deployment rolling-update settings solve different problems.

### PDB

Controls **voluntary disruptions** to maintain application availability.

### Deployment RollingUpdate

Controls **how a new application version replaces the old version**.

Do not confuse:

``` text
PDB maxUnavailable
```

with:

``` text
Deployment rollingUpdate.maxUnavailable
```

They are different settings with different purposes.

------------------------------------------------------------------------

# 14. Unhealthy Pods and `unhealthyPodEvictionPolicy`

Modern Kubernetes also provides:

``` yaml
unhealthyPodEvictionPolicy:
```

Two supported policies are:

``` yaml
IfHealthyBudget
```

and

``` yaml
AlwaysAllow
```

The default behavior corresponds to `IfHealthyBudget`.

`AlwaysAllow` can be useful for allowing unhealthy running Pods to be
evicted during node drains rather than having a misbehaving application
prevent the drain.

Example:

``` yaml
spec:
  minAvailable: 5
  unhealthyPodEvictionPolicy: AlwaysAllow
  selector:
    matchLabels:
      app: my-app
```

This is an advanced troubleshooting point and usually does not need to
be included in a short interview answer unless unhealthy Pods are part
of the scenario.

------------------------------------------------------------------------

# 15. Common mistakes when configuring PDBs

### Mistake 1 --- PDB is too restrictive

For example:

``` yaml
minAvailable: 6
```

with a 6-replica application.

This effectively requires all 6 Pods to remain available, so voluntary
eviction may not be possible.

Similarly:

``` yaml
maxUnavailable: 0
```

allows zero voluntary disruption.

This can make node draining impossible for nodes hosting those Pods.

------------------------------------------------------------------------

### Mistake 2 --- Incorrect selector

Example:

``` yaml
selector:
  matchLabels:
    app: frontend
```

but the Pods actually have:

``` yaml
labels:
  app: backend
```

The PDB will not target the intended Pods.

Always verify:

``` bash
kubectl get pods --show-labels -n <namespace>
```

------------------------------------------------------------------------

### Mistake 3 --- Not enough replicas

Suppose an application has only one replica:

``` text
1 replica
+
minAvailable: 1
```

There is no room for voluntary disruption.

For highly available production workloads, the application architecture
should be designed with enough replicas and appropriate distribution
across failure domains.

------------------------------------------------------------------------

# 16. Practical troubleshooting commands

### List PDBs

``` bash
kubectl get pdb -A
```

### Check PDB in a namespace

``` bash
kubectl get pdb -n <namespace>
```

### Describe PDB

``` bash
kubectl describe pdb <pdb-name> -n <namespace>
```

### Get complete YAML

``` bash
kubectl get pdb <pdb-name> -n <namespace> -o yaml
```

### Check Pods and labels

``` bash
kubectl get pods -n <namespace> --show-labels
```

### Check a Deployment

``` bash
kubectl get deployment <deployment-name> -n <namespace>
```

### Check the Deployment's replica count

``` bash
kubectl get deployment <deployment-name> -n <namespace> \
  -o jsonpath='{.spec.replicas}'
```

### Check node drain

``` bash
kubectl drain <node-name> --ignore-daemonsets
```

------------------------------------------------------------------------

# 17. Interview Answer

If asked:

> **"A production application is healthy, but during planned
> infrastructure maintenance, Pod eviction is blocked. What would you
> check first?"**

A strong answer would be:

> "I would first check the PodDisruptionBudget. PDB controls voluntary
> disruptions and defines how many Pods must remain available using
> `minAvailable` or `maxUnavailable`. I would check the PDB selector and
> its status, especially `currentHealthy`, `desiredHealthy`, and
> `disruptionsAllowed`. If `disruptionsAllowed` is zero, the eviction
> can be blocked because another voluntary disruption would violate the
> application's availability requirement."

Then add:

> "I would also verify that the PDB selector matches the intended Pods
> and that the PDB isn't overly restrictive for the application's
> replica count."

------------------------------------------------------------------------

# 18. Simple mental model

Remember this:

``` text
Application
    |
    | 6 replicas
    ↓
Pod Pod Pod Pod Pod Pod
    |
    ↓
   PDB
    |
    ├── minAvailable
    │
    └── maxUnavailable
    |
    ↓
Planned disruption?
    |
    ├── YES → Check disruption budget
    |
    └── NO  → PDB cannot prevent the failure
```

### One-line takeaway

> **Healthy Pods + planned eviction blocked → check the
> PodDisruptionBudget and especially `disruptionsAllowed`.**

------------------------------------------------------------------------

## Key points to remember

1.  **PDB = PodDisruptionBudget**
2.  It protects application availability during **voluntary
    disruptions**.
3.  Use **`minAvailable` OR `maxUnavailable`**.
4.  PDB uses a **label selector** to identify the Pods it protects.
5.  `kubectl drain` uses the **Eviction API**, which respects PDBs.
6.  `disruptionsAllowed: 0` can cause eviction to be blocked.
7.  PDB does **not** prevent unexpected node/infrastructure failures.
8.  PDB is different from Deployment `RollingUpdate.maxUnavailable`.
9.  Always verify the PDB selector matches the application's Pod labels.
10. Avoid overly restrictive PDBs that make planned maintenance
    impossible.

------------------------------------------------------------------------

## Official Kubernetes References

-   Kubernetes PodDisruptionBudget documentation
-   Kubernetes PodDisruptionBudget API reference
-   Kubernetes API-initiated Eviction documentation

These references describe the current `policy/v1` PDB behavior and
eviction semantics.
