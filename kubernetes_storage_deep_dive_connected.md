# Kubernetes Storage Deep Dive
## A connected explanation from operating system storage to Kubernetes stateful workloads

This guide is written as one continuous story.

The problem with many Kubernetes storage explanations is that they start in the middle. They begin with `PersistentVolume`, `PVC`, or `StorageClass` as if those objects explain themselves. They do not. Those objects only make sense when you already understand what storage is doing underneath at the operating system level.

So this guide uses a different order.

It starts with the basic question: **why does storage exist at all?** Then it moves through operating system storage, container filesystems, Kubernetes abstractions, stateful workload design, and finally troubleshooting and architectural trade-offs.

If you read the sections in order, each one should feel like the next logical step.

---

# Part 1 — Why storage exists

A running program needs two different kinds of places to keep data:

1. **Memory**
2. **Storage**

Memory is fast, but temporary. If a process crashes or the machine powers off, memory contents disappear.

Storage is slower, but durable. If data is written properly, it can survive process restart, machine reboot, and sometimes even movement of the workload to another machine.

That is the first principle:

> Storage exists because applications need data to outlive execution.

Some applications do not care much about durability. A stateless HTTP server can restart and continue serving requests because its important state lives elsewhere. A database is the opposite. Its entire purpose is to preserve state across time. If its data disappears on restart, the application has failed at its core job.

So before Kubernetes, before containers, before cloud, the original problem is simple:

- where does data live,
- how is it organized,
- how is it accessed,
- and how long does it survive?

Those are storage questions.

---

# Part 2 — Operating system storage: the real foundation

Kubernetes does not invent storage. It sits on top of operating system storage behavior.

To understand Kubernetes storage, you first need to understand what the operating system sees.

## 2.1 Block devices

At a low level, storage often begins as a **block device**.

A block device is disk-like storage that the operating system can read and write in chunks called blocks. This could be:

- a local SSD
- an HDD
- an NVMe drive
- a SAN LUN
- a remote cloud disk
- a distributed block device such as Ceph RBD

In Linux, these often appear as names such as:

- `/dev/sda`
- `/dev/sdb`
- `/dev/nvme0n1`

The important idea is that a block device is not yet a friendly directory tree. It is just raw addressable storage.

Applications usually do not interact with raw block devices directly. They interact with filesystems built on top of them.

## 2.2 Filesystems

A filesystem is the layer that organizes raw storage into something applications can use.

A filesystem gives you:

- files
- directories
- metadata
- permissions
- timestamps
- allocation rules
- free space tracking

Common Linux filesystems include:

- ext4
- xfs
- btrfs

Without a filesystem, a disk is just raw space. With a filesystem, it becomes a structured namespace of files and directories.

That gives us a natural chain:

```text
Physical disk
  ↓
Block device
  ↓
Filesystem
  ↓
Mounted path
  ↓
Application files
```

Example:

```text
NVMe SSD
  ↓
/dev/nvme0n1p1
  ↓
ext4
  ↓
mounted at /data
  ↓
/data/postgresql/base/...
```

This chain matters because Kubernetes never escapes it. It only automates how that chain is delivered to applications.

## 2.3 Mounting

A filesystem becomes accessible through a **mount point**.

When the OS mounts a filesystem, it attaches that storage to a directory path.

For example:

```bash
mount /dev/sdb1 /data
```

After this, the application does not think about `/dev/sdb1`. It thinks about `/data`.

That is exactly the same mental move Kubernetes later makes for containers. The container does not care which backend storage exists underneath. It cares that some path, such as `/var/lib/postgresql/data`, is mounted and usable.

So when Kubernetes says it mounts a volume into a Pod, it is not inventing a new concept. It is extending the old operating system concept of mounting storage to a path.

## 2.4 Local and remote storage

The operating system can use storage that is either local or remote.

### Local storage

This is physically attached to the machine:

- SSD
- HDD
- NVMe

Properties:

- low latency
- high throughput in many cases
- physically tied to one node

### Remote storage

This is reached over a network or a storage fabric:

- NFS
- CephFS
- SAN
- cloud block disk
- distributed storage systems

Properties:

- can be shared or reattached
- can survive workload movement better
- often higher latency than truly local storage

This distinction becomes crucial in Kubernetes because Pods can move between nodes. If storage is local to one node, mobility is limited. If storage is remote and attachable, workloads may move more easily.

## 2.5 Permissions and ownership

Filesystems are not just space. They also enforce access rules.

Linux storage includes:

- user IDs
- group IDs
- permission bits
- ownership
- sometimes ACLs or SELinux labels

This matters because applications do not merely need storage to exist. They need permission to use it.

A mounted disk that is owned by the wrong UID may behave like “broken storage” from the application’s point of view.

That is why storage debugging often turns into:

- `permission denied`
- `read-only file system`
- `cannot create directory`
- `cannot write file`

These are not abstract Kubernetes problems. These are still operating system filesystem problems.

## 2.6 Durability and flushing

A final OS concept matters a lot: writing data is not the same as safely persisting it.

The operating system may buffer writes in memory before they are truly committed to disk. Filesystems may journal metadata or data differently. Some backends may acknowledge writes before data is fully durable. Databases often call `fsync` or equivalent mechanisms because they need stronger guarantees.

So two storage systems can both be “persistent,” yet behave very differently under load or failure.

That is why the storage question is never just “does the Pod have a volume?” It is also:

- how fast are writes,
- when are writes durable,
- what happens during crash,
- what does `fsync` cost,
- what consistency guarantees does the backend provide?

At this point, the real foundation is in place.

Storage is:

- a real device or remote system,
- organized by filesystem semantics,
- mounted into paths,
- protected by permissions,
- and governed by durability behavior.

Now containers complicate the picture.

---

# Part 3 — Containers changed the default assumption

Before containers, many applications assumed they ran directly on a machine or VM that had a stable filesystem.

Containers break that assumption.

## 3.1 The container filesystem is not a normal durable machine disk

A container usually runs with:

- image layers that are read-only
- one writable layer on top

That writable layer is convenient for runtime changes, but it is not a good place for durable state.

If the container is recreated, the writable layer is typically lost.

So now we get a tension:

- applications still need durable storage,
- but the container’s default writable filesystem is ephemeral.

This is why storage becomes such a central concept in container orchestration.

## 3.2 What this means in practice

Inside a container, a path might look normal:

- `/app/data`
- `/var/lib/postgresql/data`
- `/uploads`

But if that path lives only in the container writable layer, the data may disappear when the container is replaced.

So the rule becomes:

> The container filesystem is convenient for runtime execution, but not a safe default for durable application state.

This is the direct reason volumes exist in Kubernetes.

## 3.3 Three different lifetimes

At this point it helps to distinguish three storage lifetimes.

### Container lifetime
Storage in the writable layer may vanish if the container is rebuilt or recreated.

### Pod lifetime
Some storage lives as long as the Pod exists, but not beyond.

### Persistent lifetime
Some storage is intended to outlive Pod replacement and be reattached later.

That gives us a simple model:

| Storage location | Survives container restart | Survives Pod replacement | Typical use |
|---|---:|---:|---|
| Container writable layer | not reliable | no | temporary local runtime files |
| `emptyDir` | yes | no | scratch, cache, intermediate data |
| PVC-backed storage | yes | yes | durable application state |

Once you understand these different lifetimes, Kubernetes storage abstractions become much less mysterious.

---

# Part 4 — Kubernetes storage abstractions exist to bridge application needs to real storage

Kubernetes needs to solve a very specific problem:

> Applications need storage, but the platform must schedule workloads dynamically across nodes while still preserving data semantics.

That is why Kubernetes introduces a layered model.

The main objects are:

- Volume
- PersistentVolume (PV)
- PersistentVolumeClaim (PVC)
- StorageClass
- CSI driver

These objects look complicated only if you see them as separate definitions. They become clearer if you see them as a chain.

## 4.1 Volume: storage attached to the Pod

A **volume** is the Pod-facing concept.

It says: this Pod has access to some storage, and that storage can be mounted into one or more containers.

So the volume is the direct bridge between Kubernetes and the container path.

Example idea:

```yaml
volumes:
  - name: app-data
    persistentVolumeClaim:
      claimName: my-data

containers:
  - name: app
    volumeMounts:
      - name: app-data
        mountPath: /data
```

The application sees `/data`. That is the important part for the app.

But where did that storage come from? That is where PVs and PVCs enter.

## 4.2 PersistentVolume: the supply side

A **PersistentVolume** is Kubernetes’ representation of an actual storage resource.

Think of it as the supply side. It describes storage that exists or has been provisioned for use.

A PV usually carries information such as:

- capacity
- access mode
- volume mode
- reclaim policy
- backend details
- storage class

It is the Kubernetes object that corresponds to “a real piece of storage has been made available.”

## 4.3 PersistentVolumeClaim: the demand side

A **PersistentVolumeClaim** is the workload’s request for storage.

Think of it as the demand side.

Instead of saying “use disk X,” the application says something closer to:

- I need 20Gi
- I need ReadWriteOnce
- I want the `fast-ssd` class

The PVC does not want to know the physical identity of the storage. It wants matching properties.

This separation is useful because it decouples workloads from storage provisioning details.

## 4.4 StorageClass: the provisioning policy

A **StorageClass** tells Kubernetes how to create or manage storage.

It can define:

- which provisioner to use
- parameters for the backend
- reclaim policy defaults
- binding mode
- expansion behavior

So if a PVC asks for a certain StorageClass, Kubernetes knows what provisioning behavior to apply.

This is how the platform turns a high-level request into backend-specific storage.

## 4.5 CSI driver: the translator

A **CSI driver** is the standardized plugin that lets Kubernetes talk to storage backends.

Kubernetes itself does not want deep vendor-specific code for every storage platform.

Instead, Kubernetes issues standard requests like:

- create volume
- delete volume
- attach volume
- mount volume
- resize volume
- snapshot volume

The CSI driver translates those into backend-specific actions.

This is very similar to how operating systems use drivers to talk to devices while exposing standard interfaces to the rest of the system.

## 4.6 The whole chain together

Now all the pieces can be connected.

```text
Application writes to /data
  ↓
Container sees mounted path
  ↓
Pod volume refers to PVC
  ↓
PVC requests storage properties
  ↓
StorageClass defines provisioning policy
  ↓
CSI driver provisions or maps backend storage
  ↓
PV represents actual storage resource
  ↓
Node OS attaches and mounts storage
  ↓
Backend storage serves the real data
```

This is the first complete end-to-end Kubernetes storage story.

It is still the same old storage path from the OS world. Kubernetes just inserts orchestration objects in the middle.

---

# Part 5 — Static and dynamic provisioning: two ways the PV appears

Once you understand PV and PVC, the next question is how the PV gets there.

There are two major models.

## 5.1 Static provisioning

In static provisioning, an administrator creates the PV first.

That means someone has already prepared the storage and described it to the cluster.

Then a PVC is created later and binds to it if the properties match.

This is common when:

- storage is hand-managed
- an NFS export already exists
- on-prem environments want strict control
- the storage lifecycle is managed outside Kubernetes

The flow is:

```text
Admin prepares storage
  ↓
Admin creates PV
  ↓
Workload creates PVC
  ↓
PVC binds to matching PV
```

## 5.2 Dynamic provisioning

In dynamic provisioning, the user creates only a PVC.

Then Kubernetes uses the StorageClass and CSI driver to create the actual backing storage automatically.

The flow is:

```text
Workload creates PVC
  ↓
StorageClass selected
  ↓
CSI driver provisions storage
  ↓
PV created automatically
  ↓
PVC binds to PV
```

This is the modern default because it scales operationally and removes manual PV management for many environments.

At this point we have the lifecycle of storage creation. The next question is what kind of storage semantics the workload needs.

---

# Part 6 — Access modes and volume modes: the semantics of use

A volume is not defined only by size. It is also defined by how it can be used.

## 6.1 Volume modes: block versus filesystem

At the OS level, storage can be exposed as a raw block device or as a filesystem.

Kubernetes reflects the same distinction.

### Filesystem mode
The volume is mounted as a normal filesystem. The application sees files and directories.

This is the most common mode.

### Block mode
The volume is presented as a raw block device.

This is less common and is used by specialized workloads that want direct device access.

So again, Kubernetes is not creating a new idea. It is preserving an old one.

## 6.2 Access modes

Kubernetes also describes how a volume may be mounted.

### ReadWriteOnce (RWO)
Typically writable by one node at a time.

This is very common for block storage and databases.

### ReadOnlyMany (ROX)
Can be mounted read-only by many nodes.

Useful for shared reference data.

### ReadWriteMany (RWX)
Can be mounted read-write by many nodes.

Usually provided by shared filesystems such as NFS or CephFS.

The most useful rule of thumb is:

> Block storage usually maps well to RWO. Shared filesystems usually provide RWX.

This is not a law of physics, but it is a very strong practical default.

---

# Part 7 — Common volume types make sense once lifetimes are clear

Now that the framework is established, common Kubernetes volume types are much easier to reason about.

## 7.1 emptyDir

`emptyDir` exists for the lifetime of the Pod.

It is created when the Pod starts and removed when the Pod is gone.

It is good for:

- scratch space
- caches
- intermediate files
- sharing temporary data between containers in the same Pod

It is not good for durable business data.

The key insight is that `emptyDir` solves a Pod-lifetime storage need, not a persistent-lifetime storage need.

## 7.2 hostPath

`hostPath` mounts a directory from the node directly into the Pod.

This is useful when the application genuinely needs node-local host data, for example:

- log collectors
- node agents
- system integrations
- debugging tools

But it comes with real costs:

- tight coupling to a specific node
- weaker portability
- security implications
- unpredictable behavior if rescheduled elsewhere

`hostPath` is therefore powerful, but it is close to exposing raw host storage to the application.

## 7.3 PVC-backed volumes

PVC-backed volumes are the normal answer when an application needs persistence that outlives the Pod.

This is the general-purpose production pattern for durable storage.

## 7.4 ConfigMap and Secret volumes

These look like mounted files, but they are not a replacement for persistent application storage.

They are meant for:

- configuration files
- certificates
- secrets
- small settings data

They answer a configuration-distribution problem, not a durability problem.

---

# Part 8 — Backend categories: local disk, shared filesystem, remote block

At the Kubernetes layer, storage looks abstract. At the backend layer, it is still a real storage system with real trade-offs.

There are three broad backend categories worth mastering.

## 8.1 Local storage

Local storage is physically attached to the node.

Examples:

- SSD
- NVMe
- local PV
- local-path provisioners

Advantages:

- low latency
- high performance
- simple data path

Disadvantages:

- tied to one node
- poor mobility
- node failure can make the data inaccessible
- scheduling must respect placement constraints

This is best when performance matters and the application or cluster design can tolerate node locality.

## 8.2 Network filesystems

These expose a shared filesystem over the network.

Examples:

- NFS
- CephFS
- EFS-like systems

Advantages:

- natural fit for RWX
- many Pods can share the same file tree
- easier for shared assets

Disadvantages:

- metadata overhead
- contention
- often higher latency
- sometimes poor fit for transactional databases

This is best for shared content, user uploads, and collaborative access patterns.

## 8.3 Remote block storage

This behaves like a disk, but the disk is remote.

Examples:

- EBS
- GCE PD
- Azure Disk
- Ceph RBD
- SAN LUN

Advantages:

- disk-like semantics
- strong fit for single-writer databases
- often simpler mental model for stateful workloads

Disadvantages:

- commonly one writable attachment at a time
- attach/detach behavior matters
- still network-backed, so latency is not the same as local NVMe

This is usually the default good answer for databases that want one volume per replica.

At this point, we can finally discuss workload design properly.

---

# Part 9 — Stateless and stateful workloads are different storage problems

A stateless workload mainly needs code and configuration. A stateful workload needs continuity of data identity.

## 9.1 Stateless workloads

Examples:

- web frontends
- many API servers
- retryable workers
- pure compute jobs

These often do not require durable volumes because the important state is elsewhere.

## 9.2 Stateful workloads

Examples:

- PostgreSQL
- MySQL
- Redis with persistence
- Elasticsearch
- Kafka
- file upload services

These workloads depend on data surviving replacement and often need stable storage semantics.

That is why stateful workloads are where Kubernetes storage design really matters.

## 9.3 Why databases care so much

A database still experiences the world through operating system storage behavior:

- latency
- write ordering
- fsync cost
- filesystem semantics
- locking behavior
- read/write throughput

The database does not care that Kubernetes objects exist. Kubernetes only influences how the database reaches its data path.

So if the storage backend is a poor fit, no amount of elegant YAML changes the underlying fact.

---

# Part 10 — StatefulSet: the Kubernetes pattern that respects storage identity

This is the natural next step once you accept that stateful workloads need stable identity.

A `Deployment` is good when replicas are interchangeable.

A `StatefulSet` is better when replicas need:

- stable names
- stable identity
- stable per-replica storage

## 10.1 One replica, one volume

The defining storage pattern of StatefulSet is that each replica typically gets its own PVC.

Example:

```text
postgres-0 → pvc data-postgres-0
postgres-1 → pvc data-postgres-1
postgres-2 → pvc data-postgres-2
```

This matches how databases actually think.

Each instance wants its own data directory, its own WAL files, its own storage ownership, and its own failure boundary.

## 10.2 Why not one shared writable volume for all replicas?

Because that would ignore the application’s real storage model.

Most databases are not designed for multiple independent replicas writing freely to one shared filesystem path. They want clearly defined replication semantics and separate storage identity.

So StatefulSet plus one PVC per replica is not arbitrary Kubernetes tradition. It is a reflection of how stateful software actually works.

---

# Part 11 — What really happens on the node when a Pod mounts storage

This is the layer many explanations skip, and it is where a lot of confusion comes from.

When a Pod uses a volume, the node OS still performs real storage work.

A typical sequence looks like this:

1. The scheduler places the Pod on a node.
2. The kubelet on that node sees which volumes are needed.
3. The CSI driver prepares, attaches, or maps storage if required.
4. The node mounts the filesystem or network share to a kubelet-managed path.
5. The container runtime bind-mounts that path into the Pod’s filesystem namespace.
6. The application sees the final mount path.

Conceptually:

```text
Backend storage
  ↓
Node sees device or network endpoint
  ↓
Node OS mount under kubelet management
  ↓
Bind mount into Pod
  ↓
Application sees /data
```

This matters because storage debugging is often partly a node problem. The YAML may be correct, but the node may have failed to attach or mount the storage.

That is why Kubernetes storage is never purely a control-plane topic. It is also a runtime operating system topic.

---

# Part 12 — Reclaim policy, binding mode, expansion, and snapshots are lifecycle controls

So far we have discussed how storage exists and gets mounted. Now we need to discuss what happens across time.

## 12.1 Reclaim policy

When a claim is deleted, what should happen to the underlying storage?

### Delete
The backing storage is removed.

Useful for:

- temporary environments
- disposable test systems

Danger:
- easy data loss if used carelessly

### Retain
The backing storage remains even after the claim is deleted.

Useful for:

- production data
- manual recovery workflows
- conservative operational environments

This is really a storage lifecycle decision, not just a Kubernetes detail.

## 12.2 Volume binding mode

Some environments need storage to be provisioned only when the workload is actually scheduled.

### Immediate
Provision or bind as soon as the PVC exists.

### WaitForFirstConsumer
Delay provisioning until a Pod that uses the claim is scheduled.

This matters when topology is important, such as:

- local storage
- zone-aware storage
- backends with node or region constraints

## 12.3 Expansion

Some volumes can be resized.

But resizing actually has layers:

- the Kubernetes request changes
- the backend volume grows
- the filesystem may need to grow too

So this is another example where Kubernetes is coordinating a process that still depends on lower-level storage behavior.

## 12.4 Snapshots

Snapshots are point-in-time views of storage state.

They are useful for:

- backup workflows
- cloning
- restore tests
- operational recovery

But they do not automatically equal application-consistent backup.

A database snapshot may be crash-consistent rather than logically clean. So snapshots are powerful, but they are not the entire backup story.

---

# Part 13 — Persistence is not backup, and this is one of the most dangerous confusions

A PVC provides persistence.

That means data can survive normal restart or replacement events.

But persistence is not backup.

If the application corrupts the data, a PVC will faithfully preserve the corrupted data. If an operator deletes important records, the PVC will faithfully preserve the deletion.

Backup is about recoverability after bad events, not just continuity through normal lifecycle events.

So a real storage strategy for stateful workloads usually includes both:

- persistent storage
- backup and restore design

That may involve:

- snapshots
- logical dumps
- off-cluster copies
- tested restore procedures

Without restore testing, backup claims are often aspirational rather than real.

---

# Part 14 — Performance and correctness still come from the backend

Kubernetes abstracts storage, but it does not remove backend physics.

Applications still feel:

- latency
- throughput
- IOPS
- network jitter
- metadata cost
- flush behavior

This is where many poor designs reveal themselves.

A database placed on slow RWX shared storage may technically work, but latency and consistency behavior may be terrible for the workload.

A shared uploads service placed on a single-writer RWO block disk may become awkward to scale horizontally.

So storage design is always a matching exercise:

- what access pattern does the application need,
- what consistency model does it assume,
- what performance profile does it demand,
- and which backend actually supplies that?

---

# Part 15 — The clean workload mapping

Now we can summarize the best-fit patterns in a way that follows all the earlier logic.

## 15.1 PostgreSQL or MySQL

Recommended pattern:

- StatefulSet
- one PVC per replica
- RWO block storage
- backups and usually replication strategy

Why:
The application wants single-writer disk-like semantics and stable storage identity.

## 15.2 Shared uploads or media

Recommended pattern:

- PVC on RWX-capable shared filesystem

Why:
Multiple Pods need to see the same files at the same time.

## 15.3 Temporary batch processing

Recommended pattern:

- `emptyDir`

Why:
The data is temporary and tied to the Pod lifecycle.

## 15.4 Node agents or system daemons

Recommended pattern:

- `hostPath`

Why:
They genuinely need access to host files, sockets, or logs.

These patterns are not just recipes. They fall naturally out of the earlier layers of reasoning.

---

# Part 16 — The most useful troubleshooting ladder

When storage breaks, the fastest path is to debug from the bottom upward.

Do not start with application logs alone.

Use this ladder:

1. Is the backend storage healthy?
2. Is the CSI driver or provisioner healthy?
3. Is the PV or PVC created and bound correctly?
4. Is the Pod scheduled to a valid node?
5. Did attach or mount succeed on the node?
6. Are permissions and ownership correct?
7. Is the application using the expected path correctly?

This order works because storage problems often originate below the application, even when the application is where the symptom appears.

A clean mental model is:

```text
backend
→ node mount
→ Kubernetes object binding
→ Pod mount
→ permissions
→ application behavior
```

That is usually a faster and more reliable method than reading logs and guessing.

---

# Part 17 — The final integrated mental model

At the beginning, storage looked like many disconnected Kubernetes nouns.

Now it can be described as one connected system:

An application needs data to survive execution. That requires durable storage. Durable storage begins as real backend storage, exposed to an operating system as a block device or networked storage service. The operating system organizes it through filesystems, mount points, permissions, and flushing behavior. Containers complicate things because their default writable layers are ephemeral. Kubernetes therefore adds abstractions so workloads can request durable storage without binding themselves directly to backend details. PVCs express storage demand, PVs represent available storage, StorageClasses define provisioning policy, and CSI drivers translate platform requests into backend operations. The node then performs real attach and mount work, and the application finally sees a path inside the container. Stateful workloads need this chain to preserve identity and correctness; stateless workloads often do not. Performance, durability, and recovery still depend on the real storage backend, not on abstraction alone.

That is the whole story.

---

# Part 18 — Memorization version

If you want one diagram to remember everything, use this:

```text
Application data
  ↓
Container mount path
  ↓
Pod volume
  ↓
PVC
  ↓
PV
  ↓
StorageClass + CSI
  ↓
Node OS attach / mount / permissions
  ↓
Backend storage
```

Or even shorter:

```text
Real storage
→ OS semantics
→ Kubernetes abstraction
→ Pod mount
→ application state
```

Once you internalize that, Kubernetes storage stops feeling magical and starts feeling predictable.
