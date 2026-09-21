# Kubernetes Persistent Storage on AWS/EKS --- Interview-Focused Study Guide

> **Interviewer:** You are running your application inside Kubernetes
> Pods on AWS, and now the application needs to store persistent data.
> How will you do it?

## Quick Answer

I would use Kubernetes persistent storage rather than relying on the
Pod/container filesystem. On AWS/EKS, a common approach is to use the
**AWS EBS CSI Driver** with a **StorageClass**, create a
**PersistentVolumeClaim (PVC)** for the required capacity, and mount
that PVC into the application Pod.

With dynamic provisioning, the flow is:

``` text
Pod
 ↓
PVC
 ↓
StorageClass
 ↓
CSI Driver
 ↓
AWS EBS
```

The PVC requests storage, the StorageClass defines how it should be
provisioned, the CSI driver integrates Kubernetes with AWS storage, and
Kubernetes maintains the PVC → PV → underlying storage relationship.

------------------------------------------------------------------------

# 1. Why Pod Storage Is Not Enough

A container has a writable filesystem, but application data stored there
should generally be treated as **ephemeral**.

If a Pod is deleted and recreated, the new Pod does not automatically
get the old container filesystem contents.

For example, suppose an application writes:

``` text
/app/data/orders.db
```

inside the container filesystem. If that Pod is deleted and a
replacement Pod is created, that data should not be assumed to survive.

### Pod/container ephemeral storage vs persistent storage

  -----------------------------------------------------------------------
  Type                    Purpose                 Survives Pod deletion?
  ----------------------- ----------------------- -----------------------
  Container writable      Temporary               No
  filesystem              application/runtime     
                          data                    

  `emptyDir`              Temporary shared        No
                          storage for containers  
                          in one Pod              

  Persistent storage via  Application data that   Yes, subject to the
  PVC                     should survive Pod      PVC/PV/storage
                          lifecycle               lifecycle
  -----------------------------------------------------------------------

### What is `emptyDir`?

`emptyDir` creates a directory when a Pod is assigned to a node.
Containers in that Pod can use it as shared temporary storage.

Example:

``` yaml
volumes:
  - name: temp-data
    emptyDir: {}
```

It is useful for temporary files, caches, scratch space, or data shared
between containers in the same Pod.

**Interview point:** `emptyDir` survives a container restart within the
same Pod, but it is deleted when the Pod is removed from the node.

### Simple real-world example

Imagine an order-processing application:

``` text
Application Pod
      |
      +--> /tmp/cache       → temporary data
      |
      +--> /data/orders     → important persistent data
```

The cache can use ephemeral storage. Important order data should use a
persistent storage solution.

### Small architecture

``` text
                    Kubernetes
                        |
                     Pod/App
                    /       \
                   /         \
          Temporary data    Persistent data
               |                  |
           emptyDir              PVC
                                  |
                                  PV
                                  |
                              AWS Storage
```

------------------------------------------------------------------------

# 2. Kubernetes Persistent Storage Architecture

A useful interview mental model is:

``` text
Pod
 ↓
PVC
 ↓
PV
 ↓
CSI Driver
 ↓
AWS Storage
```

## Pod

The Pod is the consumer of storage.

The application container mounts storage using:

-   `volumeMounts` --- defines where storage appears inside the
    container.
-   `volumes` --- defines the volume source available to the Pod.

Example:

``` yaml
containers:
  - name: app
    image: nginx
    volumeMounts:
      - name: app-storage
        mountPath: /data

volumes:
  - name: app-storage
    persistentVolumeClaim:
      claimName: app-data
```

The application can then read and write files under:

``` text
/data
```

## PVC --- PersistentVolumeClaim

> **PVC is a request for storage.**

For example:

``` yaml
resources:
  requests:
    storage: 20Gi
```

A PVC can specify:

### `storage`

How much capacity the application requests.

``` yaml
storage: 20Gi
```

### `accessModes`

How the storage is intended to be accessed.

``` yaml
accessModes:
  - ReadWriteOnce
```

### `storageClassName`

Which StorageClass should satisfy the request.

``` yaml
storageClassName: ebs-sc
```

### Why does the Pod reference the PVC?

The application normally does not need to know the AWS EBS volume ID.

Instead:

``` text
Pod
 ↓
PVC
 ↓
PV
 ↓
Underlying storage
```

This separates the application workload from infrastructure-specific
storage details.

## PV --- PersistentVolume

> **PV is a Kubernetes object representing/providing persistent
> storage.**

**Important:** A PV is **not the physical EBS/EFS storage itself**.

The PV represents/connects Kubernetes to an underlying storage backend.

Examples:

``` text
PV
 ├── EBS backend
 ├── EFS backend
 ├── NFS backend
 └── Other supported storage backends
```

A useful analogy:

``` text
PVC = storage request
PV  = Kubernetes storage object
EBS = actual AWS block storage
CSI = integration layer between Kubernetes and AWS
```

------------------------------------------------------------------------

# 3. What Is CSI?

**CSI** stands for **Container Storage Interface**.

It provides a standard interface through which Kubernetes can work with
different storage systems.

Without needing Kubernetes itself to implement every storage provider's
proprietary API, a CSI driver handles provider-specific operations.

## CSI driver concept

A CSI driver is the integration component for a storage provider.

On AWS/EKS, common examples include:

-   AWS EBS CSI Driver
-   AWS EFS CSI Driver

### AWS EBS CSI Driver

The EBS CSI Driver allows Kubernetes to provision, attach, mount, and
manage AWS EBS-backed persistent storage for workloads.

### AWS EFS CSI Driver

The EFS CSI Driver allows Kubernetes workloads to use AWS EFS-backed
shared file storage.

### Architecture

``` text
                 Kubernetes
                     |
                     ▼
                CSI Driver
                  /     \
                 /       \
                ▼         ▼
              EBS        EFS
```

The key interview concept is:

> Kubernetes uses the CSI interface, while the CSI driver handles the
> storage-provider-specific integration.

------------------------------------------------------------------------

# 4. Dynamic Provisioning

Suppose a user creates a PVC requesting:

``` yaml
resources:
  requests:
    storage: 20Gi
```

The PVC references a StorageClass.

A simplified dynamic provisioning flow is:

``` text
User creates PVC requesting 20Gi
        ↓
PVC references StorageClass
        ↓
StorageClass identifies provisioner
        ↓
CSI Driver receives provisioning request
        ↓
AWS EBS CSI Driver calls AWS
        ↓
AWS creates EBS volume
        ↓
Kubernetes creates/binds PV
        ↓
PVC becomes Bound
        ↓
Pod mounts PVC
```

### Important interview statement

> **The user does not normally manually create the EBS volume first when
> using dynamic provisioning.**

The StorageClass and CSI driver provide the mechanism for automatically
provisioning the required storage.

------------------------------------------------------------------------

# 5. How Does the Pod Know Which EBS to Use?

This is an important interview concept.

Imagine AWS has 100 EBS volumes.

The Pod does **not** search through all 100 EBS volumes.

Instead, Kubernetes maintains the relationship between the PVC, PV, and
underlying storage.

Example:

``` text
Pod
 ↓
PVC: app-data
 ↓
PV: pv-app-data
 ↓
EBS Volume ID: vol-0abc123
```

The relationship can be thought of as:

``` text
PVC → PV → underlying storage
```

The Pod references:

``` yaml
persistentVolumeClaim:
  claimName: app-data
```

It does not normally specify:

``` text
vol-0abc123
```

### Analogy

> **PVC = storage request**\
> **PV = storage object provided by Kubernetes**\
> **EBS = actual storage**\
> **CSI = integration layer between Kubernetes and AWS**

This abstraction allows the application to consume storage without
hard-coding provider-specific storage identifiers.

------------------------------------------------------------------------

# 6. Complete YAML Example

The following example uses an AWS EBS CSI Driver and `gp3` as the
StorageClass.

## 6.1 StorageClass

``` yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-sc
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### What this defines

-   `provisioner`: AWS EBS CSI Driver
-   `type: gp3`: EBS volume type
-   `fsType: ext4`: filesystem used when formatting the volume
-   `reclaimPolicy: Delete`: dynamically provisioned storage can be
    deleted when its associated PVC/PV lifecycle reaches deletion
-   `volumeBindingMode: WaitForFirstConsumer`: delays volume
    binding/provisioning until a consuming Pod exists, helping account
    for topology
-   `allowVolumeExpansion: true`: permits supported volume expansion

## 6.2 PVC

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-sc
  resources:
    requests:
      storage: 20Gi
```

This says:

> "I need 20Gi of storage using the `ebs-sc` StorageClass."

## 6.3 Deployment

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: storage-app
  template:
    metadata:
      labels:
        app: storage-app
    spec:
      containers:
        - name: app
          image: nginx
          volumeMounts:
            - name: app-storage
              mountPath: /data
      volumes:
        - name: app-storage
          persistentVolumeClaim:
            claimName: app-data
```

### Complete relationship

``` text
Deployment
    |
    ▼
  Pod
    |
    ▼
PVC: app-data
    |
    ▼
StorageClass: ebs-sc
    |
    ▼
EBS CSI Driver
    |
    ▼
AWS EBS gp3 volume
```

------------------------------------------------------------------------

# 7. What Happens Behind the Scenes?

## When PVC is created

The high-level flow is:

``` text
PVC → StorageClass → CSI Driver → AWS API → EBS
```

The StorageClass identifies the provisioner and storage behavior. The
CSI driver handles the provider-specific provisioning request.

## When Pod is scheduled

EBS is generally **Availability Zone scoped**.

That means topology matters.

For example:

``` text
AZ-a
 ├── Node-A
 └── EBS-A

AZ-b
 ├── Node-B
 └── EBS-B
```

If an EBS volume exists in one AZ, Kubernetes cannot simply treat it as
if it were a normal multi-AZ shared filesystem.

`WaitForFirstConsumer` can help delay volume provisioning/binding until
scheduling information is available, allowing the storage to be
provisioned with the workload's topology in mind.

## When Pod starts

A simplified runtime flow is:

``` text
EBS Volume
   ↓
Attach to worker node
   ↓
Mount on node
   ↓
Mount into container
   ↓
Application reads/writes /data
```

------------------------------------------------------------------------

# 8. EBS vs EFS

  -----------------------------------------------------------------------
  Feature                 EBS                     EFS
  ----------------------- ----------------------- -----------------------
  Storage type            Block storage           Managed shared file
                                                  storage

  Access mode             Commonly RWO            Commonly RWX

  Multi-AZ                Volume is AZ-scoped     Designed for
                                                  regional/shared access

  Multiple Pods           Typically not used as   Designed for concurrent
                          shared multi-node       access
                          storage                 

  Typical use case        Databases, application  Shared application
                          block storage,          files, shared content,
                          workloads needing block workloads needing
                          devices                 concurrent file access

  Kubernetes CSI Driver   AWS EBS CSI Driver      AWS EFS CSI Driver

  Performance             Block-storage           Network file-system
  characteristics         performance             performance
                          characteristics         characteristics
  -----------------------------------------------------------------------

### When would you use EBS?

A workload may use EBS when it needs block storage and a workload's
access pattern fits EBS's characteristics.

Examples can include:

-   Database workloads
-   Applications needing a block device
-   Stateful workloads with per-instance storage

### When would you use EFS?

A workload may use EFS when multiple Pods/nodes need shared file access.

Examples can include:

-   Shared application files
-   Shared content
-   Workloads requiring concurrent file access

**Do not say one is universally better.** The choice depends on
application architecture, access pattern, topology, performance
requirements, and operational needs.

------------------------------------------------------------------------

# 9. Access Modes

Kubernetes access modes describe how a volume can be mounted/accessed.

## ReadWriteOnce --- RWO

The volume can be mounted read-write by a single node.

``` text
RWO
→ Read + Write
→ One node
```

EBS is commonly used with RWO-style workloads.

## ReadOnlyMany --- ROX

The volume can be mounted read-only by many nodes.

``` text
ROX
→ Read-only
→ Multiple nodes
```

The practical availability of a particular access mode depends on the
storage backend and CSI driver.

## ReadWriteMany --- RWX

The volume can be mounted read-write by many nodes.

``` text
RWX
→ Read + Write
→ Multiple nodes
```

EFS is commonly relevant when workloads need shared read/write access.

### Practical example

``` text
Database Pod
   |
   └── EBS-backed PVC
       └── RWO

Multiple application Pods
   |
   └── EFS-backed PVC
       └── RWX
```

------------------------------------------------------------------------

# 10. StorageClass

A **StorageClass** describes a class of storage and how dynamic
provisioning should occur.

## Why is StorageClass needed?

Instead of manually creating every PV and storage volume, a StorageClass
can define the provisioning behavior.

``` text
PVC
 ↓
StorageClass
 ↓
CSI Driver
 ↓
Storage
```

## Provisioner

The provisioner identifies the component responsible for provisioning
storage.

For EBS:

``` yaml
provisioner: ebs.csi.aws.com
```

## `reclaimPolicy`

Common values include:

``` text
Delete
Retain
```

It controls what happens to the provisioned persistent storage when the
associated Kubernetes storage object lifecycle reaches deletion.

## `volumeBindingMode`

Two commonly discussed modes are:

``` text
Immediate
WaitForFirstConsumer
```

`WaitForFirstConsumer` is especially useful for topology-sensitive
storage such as EBS because provisioning/binding can take the consuming
Pod's scheduling constraints into account.

## `allowVolumeExpansion`

``` yaml
allowVolumeExpansion: true
```

When supported, this permits a PVC's requested capacity to be increased.

### PVC vs StorageClass vs PV

  Object         Meaning
  -------------- -----------------------------------------------------------------
  PVC            Application's request for storage
  StorageClass   Defines/provides the storage provisioning behavior
  PV             Kubernetes object representing/providing the persistent storage

------------------------------------------------------------------------

# 11. PV/PVC Lifecycle

A simplified lifecycle is:

``` text
Available
   ↓
Bound
   ↓
Released
   ↓
Deleted
```

## Pod is deleted

Deleting a Pod normally does **not** mean deleting its PVC.

Example:

``` text
Pod deleted
   ↓
PVC remains
   ↓
PV remains
   ↓
Underlying EBS can remain
```

A replacement Pod can use the same PVC, subject to workload,
access-mode, topology, and storage constraints.

## PVC is deleted

Deleting the PVC starts the PVC/PV lifecycle toward release/deletion
according to the relevant reclaim policy and storage-controller
behavior.

## PV is deleted

Deleting the PV removes the Kubernetes PV object. The underlying storage
behavior depends on how the volume was provisioned and the applicable
reclaim policy/controller behavior.

## `Delete` reclaim policy

With:

``` yaml
reclaimPolicy: Delete
```

dynamically provisioned storage can be deleted as part of the associated
storage lifecycle.

## `Retain` reclaim policy

With:

``` yaml
reclaimPolicy: Retain
```

the underlying persistent storage is retained for recovery/manual
handling rather than automatically being deleted.

### Important distinction

**Pod deletion is not the same as PVC deletion.**

``` text
Delete Pod
   ≠
Delete PVC
   ≠
Delete EBS
```

**Interview point:** Deleting a Pod normally does not mean deleting the
persistent EBS volume.

------------------------------------------------------------------------

# 12. Persistent Storage vs Backup

> **Persistent storage is NOT the same thing as backup.**

For example:

``` text
EBS Volume
     ≠
EBS Snapshot
```

An EBS volume provides persistent storage for the workload.

A snapshot is a separate backup/recovery mechanism.

A production system can require backups because persistent storage can
still be affected by:

-   Application bugs
-   Accidental deletion
-   Data corruption
-   Operational mistakes
-   Security incidents
-   Other failure scenarios

Therefore:

``` text
Persistent storage
        +
Backup / snapshot strategy
```

are separate concerns.

------------------------------------------------------------------------

# 13. StatefulSet

StatefulSets are commonly used for stateful applications because they
provide stable Pod identity and support stable storage patterns.

Important concepts include:

-   Stable Pod identity
-   Stable storage
-   `volumeClaimTemplates`
-   Database workloads

## StatefulSet storage pattern

``` text
StatefulSet
 ├── Pod-0 → PVC-0 → PV → EBS
 ├── Pod-1 → PVC-1 → PV → EBS
 └── Pod-2 → PVC-2 → PV → EBS
```

Each Pod can receive its own PVC from a `volumeClaimTemplates`
definition.

This is useful for stateful workloads where each instance needs its own
persistent storage.

------------------------------------------------------------------------

# 14. Troubleshooting

## PVC stuck in `Pending`

### Commands

``` bash
kubectl get pvc
kubectl describe pvc <pvc-name>
kubectl get storageclass
```

### Possible reasons

-   StorageClass missing
-   CSI driver issue
-   AZ/topology issue
-   Insufficient capacity/resources
-   IAM permissions
-   Provisioner problem

Start with:

``` bash
kubectl describe pvc <pvc-name>
```

and inspect events and the reported provisioning error.

## Pod stuck in `ContainerCreating`

### Commands

``` bash
kubectl describe pod <pod-name>
kubectl get pv
kubectl get pvc
```

Possible causes include:

-   Volume attachment problem
-   Volume mount problem
-   CSI driver issue
-   Node/storage topology issue
-   Filesystem/mount-related issue

Check the Pod events and volume information.

## Application cannot write data

Check:

### Mount path

Does the application write to the same path where the volume is mounted?

``` text
Application
    ↓
/data
    ↓
PVC
```

### Permissions

Verify that the application process can write to the mounted filesystem.

### PVC status

``` bash
kubectl get pvc
```

The PVC should normally be:

``` text
Bound
```

### Filesystem

Check that the filesystem is mounted and usable.

### Storage capacity

Check whether the volume/filesystem has sufficient capacity.

------------------------------------------------------------------------

# 15. Useful Commands Cheat Sheet

## PVC

``` bash
kubectl get pvc
```

Shows PVCs and their status, volume, capacity, access mode, and
StorageClass.

``` bash
kubectl describe pvc <pvc-name>
```

Shows detailed PVC information, including events and
provisioning-related messages.

## PV

``` bash
kubectl get pv
```

Shows PersistentVolumes and their state, capacity, claim, access mode,
and reclaim policy.

``` bash
kubectl describe pv <pv-name>
```

Shows detailed PV configuration and volume-related information.

## StorageClass

``` bash
kubectl get storageclass
```

Shows available StorageClasses and their provisioners/binding behavior.

``` bash
kubectl describe storageclass <storageclass-name>
```

Shows detailed StorageClass configuration.

## Pods

``` bash
kubectl get pods
```

Shows Pod status.

``` bash
kubectl describe pod <pod-name>
```

Shows Pod configuration, volumes, scheduling information, and events.

## Events

``` bash
kubectl get events
```

Shows cluster events that can help identify scheduling, provisioning,
attachment, or mount problems.

### Fast troubleshooting sequence

``` bash
kubectl get pvc
kubectl describe pvc <pvc-name>
kubectl get pv
kubectl get storageclass
kubectl describe pod <pod-name>
kubectl get events
```

------------------------------------------------------------------------

# 16. Interview Questions

## 1. What happens when a Pod using a PVC is deleted?

**Expected interview answer:** The Pod is deleted, but the PVC normally
remains.

**Explanation:** The PVC is a separate Kubernetes object from the Pod.

**Production consideration:** A replacement workload can continue using
the PVC, subject to access mode and topology constraints.

------------------------------------------------------------------------

## 2. What happens when a PVC is deleted?

**Expected interview answer:** The PVC enters its deletion/lifecycle
process, and the associated PV/storage behavior depends on the reclaim
policy.

**Explanation:** PVC deletion is different from Pod deletion.

**Production consideration:** Verify the reclaim policy before deleting
a production PVC.

------------------------------------------------------------------------

## 3. Does deleting a Pod delete EBS?

**Expected interview answer:** Normally, no.

**Explanation:** Pod deletion and persistent storage deletion are
separate lifecycle events.

**Production consideration:** Protect production data by understanding
the PVC/PV lifecycle and reclaim policy.

------------------------------------------------------------------------

## 4. What is the difference between PV and PVC?

**Expected interview answer:** A PVC is a request for storage; a PV is a
Kubernetes object representing/providing persistent storage.

**Explanation:**

``` text
PVC → request
PV  → provided storage object
```

**Production consideration:** Applications generally reference PVCs
rather than infrastructure-specific storage IDs.

------------------------------------------------------------------------

## 5. Is PV the actual EBS volume?

**Expected interview answer:** No.

**Explanation:** The PV is a Kubernetes storage object
representing/providing persistent storage; the underlying backend can be
EBS, EFS, NFS, or another supported backend.

**Production consideration:** Keep the Kubernetes abstraction separate
from the physical/storage-provider resource.

------------------------------------------------------------------------

## 6. How does Kubernetes know which EBS volume belongs to a PVC?

**Expected interview answer:** Kubernetes maintains the relationship
between PVC, PV, and the underlying storage.

**Explanation:**

``` text
PVC → PV → EBS volume
```

**Production consideration:** The Pod does not search all AWS EBS
volumes.

------------------------------------------------------------------------

## 7. What is CSI?

**Expected interview answer:** CSI stands for Container Storage
Interface and provides a standard integration mechanism for Kubernetes
storage systems.

**Explanation:** CSI drivers implement provider-specific storage
operations.

**Production consideration:** On AWS/EKS, the EBS and EFS CSI drivers
integrate Kubernetes with those AWS storage services.

------------------------------------------------------------------------

## 8. What does the EBS CSI Driver do?

**Expected interview answer:** It integrates Kubernetes with AWS EBS for
persistent storage operations.

**Explanation:** It handles storage-provider-specific interactions
required for provisioning/attaching/mounting EBS-backed storage.

**Production consideration:** CSI driver health and permissions are
important troubleshooting areas.

------------------------------------------------------------------------

## 9. What is dynamic provisioning?

**Expected interview answer:** Dynamic provisioning allows Kubernetes to
automatically provision persistent storage when a PVC requests it
through a StorageClass.

**Explanation:**

``` text
PVC → StorageClass → CSI Driver → AWS EBS
```

**Production consideration:** The EBS volume does not normally need to
be manually created first.

------------------------------------------------------------------------

## 10. What happens when a PVC requests 20Gi?

**Expected interview answer:** The PVC requests 20Gi through its
StorageClass, which identifies the provisioner. The CSI driver can then
request an appropriate EBS volume and Kubernetes establishes the PV/PVC
relationship.

**Explanation:**

``` text
20Gi PVC
   ↓
StorageClass
   ↓
EBS CSI Driver
   ↓
EBS volume
   ↓
PV
   ↓
PVC Bound
```

**Production consideration:** Topology and driver/IAM configuration can
affect provisioning.

------------------------------------------------------------------------

## 11. Can two Pods use the same EBS volume?

**Expected interview answer:** It depends on the access pattern and
storage capabilities, but EBS is commonly used with `ReadWriteOnce`
workloads and is not the usual choice for shared multi-node RWX access.

**Explanation:** Access modes and the storage backend matter.

**Production consideration:** For shared multi-Pod read/write file
access, EFS is commonly considered.

------------------------------------------------------------------------

## 12. EBS vs EFS?

**Expected interview answer:** EBS provides block storage and is
commonly used for per-workload persistent block storage. EFS provides
shared file storage and is commonly used where multiple workloads need
shared file access.

**Explanation:** They have different storage models and access
characteristics.

**Production consideration:** Choose based on application access
pattern, topology, performance, and sharing requirements.

------------------------------------------------------------------------

## 13. RWO vs RWX?

**Expected interview answer:** RWO means ReadWriteOnce and is intended
for read/write access from a single node; RWX means ReadWriteMany and
supports read/write access from multiple nodes when the backend supports
it.

**Explanation:** EBS is commonly relevant to RWO workloads, while EFS is
commonly relevant to RWX workloads.

**Production consideration:** Match the access mode to the application's
architecture.

------------------------------------------------------------------------

## 14. What happens if a Pod moves to another AZ?

**Expected interview answer:** EBS is generally AZ-scoped, so topology
matters. Kubernetes scheduling and the CSI/storage configuration must
place the workload where its volume can be attached.

**Explanation:** An EBS volume is not a generic multi-AZ shared
filesystem.

**Production consideration:** `WaitForFirstConsumer` can help with
topology-aware provisioning.

------------------------------------------------------------------------

## 15. What is a StorageClass?

**Expected interview answer:** A StorageClass defines a class of storage
and the dynamic provisioning behavior for that storage.

**Explanation:** It identifies the provisioner and can define settings
such as volume type, reclaim policy, binding mode, and expansion
behavior.

**Production consideration:** StorageClass configuration directly
affects how persistent storage is provisioned.

------------------------------------------------------------------------

## 16. What is `reclaimPolicy`?

**Expected interview answer:** It defines what happens to dynamically
provisioned persistent storage when its Kubernetes storage lifecycle
reaches deletion.

**Explanation:** Common policies include `Delete` and `Retain`.

**Production consideration:** Review this setting carefully before
deleting production PVCs.

------------------------------------------------------------------------

## 17. What happens when a PVC is `Pending`?

**Expected interview answer:** Kubernetes has not successfully completed
the process of satisfying the storage claim.

**Explanation:** Investigate the StorageClass, CSI driver, topology,
capacity/resources, IAM permissions, and provisioner errors.

**Production consideration:**

``` bash
kubectl describe pvc <pvc-name>
```

is an important first diagnostic command.

------------------------------------------------------------------------

## 18. How do you troubleshoot volume attachment?

**Expected interview answer:** Start with the Pod, PVC, PV,
StorageClass, and events.

**Explanation:**

``` bash
kubectl describe pod <pod-name>
kubectl get pvc
kubectl get pv
kubectl get events
```

**Production consideration:** Look for attachment, mount, topology, and
CSI-related errors.

------------------------------------------------------------------------

## 19. Why use StatefulSet for databases?

**Expected interview answer:** StatefulSets provide stable Pod identity
and stable storage patterns, including per-Pod PVCs through
`volumeClaimTemplates`.

**Explanation:**

``` text
Pod-0 → PVC-0
Pod-1 → PVC-1
Pod-2 → PVC-2
```

**Production consideration:** StatefulSet solves Kubernetes
identity/storage orchestration concerns; it does not by itself make a
database highly available.

------------------------------------------------------------------------

## 20. Is persistent storage a backup?

**Expected interview answer:** No.

**Explanation:** Persistent storage is live storage; backups/snapshots
are separate recovery mechanisms.

**Production consideration:** Production workloads should have an
appropriate backup and recovery strategy.

------------------------------------------------------------------------

## 21. Why use a PVC instead of directly referencing an EBS volume?

**Expected interview answer:** PVC provides a Kubernetes-native storage
request and keeps the application decoupled from provider-specific
storage identifiers.

**Explanation:**

``` text
Application → PVC → PV → EBS
```

**Production consideration:** This improves abstraction and makes
storage consumption declarative.

------------------------------------------------------------------------

## 22. What is `volumeMounts`?

**Expected interview answer:** `volumeMounts` defines where a volume
appears inside a container.

**Explanation:**

``` yaml
volumeMounts:
  - name: app-storage
    mountPath: /data
```

**Production consideration:** The application must write to the correct
mount path.

------------------------------------------------------------------------

## 23. What is the purpose of `volumes` in a Pod?

**Expected interview answer:** `volumes` defines the volume source
available to the Pod.

**Explanation:**

``` yaml
volumes:
  - name: app-storage
    persistentVolumeClaim:
      claimName: app-data
```

**Production consideration:** The `name` connects the Pod volume
definition with its `volumeMounts`.

------------------------------------------------------------------------

## 24. Why use `WaitForFirstConsumer` with EBS?

**Expected interview answer:** It delays volume binding/provisioning
until there is a consuming Pod, helping Kubernetes account for topology
and scheduling constraints.

**Explanation:** EBS is generally AZ-scoped.

**Production consideration:** This can help avoid creating a volume in a
topology where the Pod cannot run.

------------------------------------------------------------------------

## 25. What is the difference between persistent storage and `emptyDir`?

**Expected interview answer:** `emptyDir` is temporary Pod-scoped
storage, while a PVC-backed volume is designed to provide persistent
storage beyond the lifetime of an individual Pod.

**Explanation:** `emptyDir` is removed when the Pod is removed.

**Production consideration:** Do not use `emptyDir` as the persistence
mechanism for important application data.

------------------------------------------------------------------------

## 26. What does `allowVolumeExpansion` do?

**Expected interview answer:** It enables supported PVC capacity
expansion.

**Explanation:**

``` yaml
allowVolumeExpansion: true
```

allows supported storage to be expanded through the PVC.

**Production consideration:** Expansion support depends on the storage
backend/driver and operational constraints.

------------------------------------------------------------------------

## 27. What is the relationship between StorageClass, PV, and PVC?

**Expected interview answer:**

``` text
StorageClass → defines provisioning
PVC          → requests storage
PV           → represents/provides the resulting persistent storage
```

**Explanation:** These objects work together during dynamic
provisioning.

**Production consideration:** Understanding their roles makes storage
troubleshooting much easier.

------------------------------------------------------------------------

## 28. How would you investigate an application that cannot write to `/data`?

**Expected interview answer:** Verify the PVC is Bound, confirm the
volume is mounted at `/data`, check permissions and filesystem state,
and verify available capacity.

**Explanation:** The problem may be at the Kubernetes storage layer or
inside the container.

**Production consideration:**

``` bash
kubectl get pvc
kubectl describe pod <pod-name>
```

Then inspect the container's mount and filesystem.

------------------------------------------------------------------------

## 29. What happens between an EBS volume and the container?

**Expected interview answer:**

``` text
EBS
 ↓
Attach to worker node
 ↓
Mount on node
 ↓
Mount into container
 ↓
Application uses /data
```

**Explanation:** The CSI/storage stack handles the storage integration
and mounting flow.

**Production consideration:** Attachment and mount failures commonly
surface through Pod events.

------------------------------------------------------------------------

## 30. How would you explain Kubernetes persistent storage in 30 seconds?

**Expected interview answer:** "I would not store important application
data only in the Pod filesystem because Pods are ephemeral. On EKS, I
can create a StorageClass backed by the AWS EBS CSI Driver, then create
a PVC requesting something like 20Gi. Kubernetes dynamically provisions
the underlying EBS-backed storage, creates/binds the PV, and I mount the
PVC into the Pod at a path such as `/data`. The application writes to
`/data`, while the storage lifecycle is managed separately from the Pod
lifecycle."

**Production consideration:** "For shared file access I would consider
EFS, and I would also keep backups/snapshots separate from persistent
storage."

------------------------------------------------------------------------

# 17. Final Mental Model

## Core definitions

``` text
PVC = Request for storage
PV = Kubernetes storage object
StorageClass = Defines/provides storage dynamically
CSI = Integration layer with storage provider
EBS = AWS block storage
EFS = AWS shared file storage
Pod = Consumer of the PVC
```

## Complete flow

``` text
                 Kubernetes
                     │
                     ▼
                    Pod
                     │
                     ▼
                    PVC
                     │
                     ▼
               StorageClass
                     │
                     ▼
                 CSI Driver
                  /       \
                 /         \
                ▼           ▼
              EBS           EFS
```

## Production mental model

``` text
Application
    │
    ▼
   Pod
    │
    │ volumeMounts
    ▼
   PVC
    │
    ▼
   PV
    │
    ▼
 CSI Driver
    │
    ├──────────► EBS
    │
    └──────────► EFS
```

### The most important interview distinctions

``` text
Pod deletion
    ≠
PVC deletion
    ≠
EBS deletion
```

And:

``` text
PVC
= request

PV
= Kubernetes storage object

EBS
= underlying AWS block storage

CSI
= integration layer
```

### 30-second interview answer

> "If my application running in an EKS Pod needs persistent data, I
> would use Kubernetes persistent storage instead of relying on the Pod
> filesystem. I would define a StorageClass using the AWS EBS CSI
> Driver, create a PVC requesting the required capacity, such as 20Gi,
> and mount that PVC into the application container at `/data`. With
> dynamic provisioning, the CSI driver provisions the EBS-backed storage
> and Kubernetes maintains the PVC-to-PV-to-storage relationship. I
> would consider EFS instead when the application needs shared file
> access, and I would treat backups or snapshots as a separate concern
> from persistent storage."
