# Kubernetes Storage Research

## Table of Contents
1. [Introduction](#introduction)
2. [Core Storage Concepts](#core-storage-concepts)
3. [Persistent Volumes and Claims](#persistent-volumes-and-claims)
4. [Storage Classes](#storage-classes)
5. [Container Storage Interface (CSI)](#container-storage-interface-csi)
6. [Storage Solutions Comparison](#storage-solutions-comparison)
7. [Ephemeral Storage](#ephemeral-storage)
8. [StatefulSet Storage](#statefulset-storage)
9. [Security and Encryption](#security-and-encryption)
10. [Backup and Disaster Recovery](#backup-and-disaster-recovery)
11. [Monitoring and Metrics](#monitoring-and-metrics)
12. [Best Practices](#best-practices)

---

## Introduction

Kubernetes storage is a fundamental component for running stateful workloads in containerized environments. While containers are ephemeral by design, modern applications require persistent data storage that survives pod restarts, rescheduling, and failures.

**Key Challenge**: Containers are ephemeral by default. When a pod restarts, any data written to its local filesystem is lost. For stateful workloads like databases, message queues, and file stores, persistent storage is essential.

**Kubernetes Solution**: The platform decouples storage provisioning from consumption through an abstraction layer that allows cluster administrators to create storage resources while developers request storage without knowing underlying infrastructure details.

---

## Core Storage Concepts

### Storage Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Storage Flow                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Administrator          Developer           Pod                │
│        │                      │               │                 │
│        ▼                      ▼               ▼                 │
│   ┌─────────┐           ┌─────────┐     ┌─────────┐             │
│   │    PV   │◄─────────│   PVC   │────►│  Mount  │             │
│   │(Storage)│   Bind    │(Request)│     │(Access) │             │
│   └────┬────┘           └─────────┘     └─────────┘             │
│        │                                                         │
│        ▼                                                         │
│   ┌─────────────────────────────────────┐                       │
│   │      Physical Storage Backend        │                       │
│   │  (EBS, GCE PD, NFS, Ceph, Local)    │                       │
│   └─────────────────────────────────────┘                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Volume Types in Kubernetes

| Volume Type | Persistence | Use Case |
|-------------|-------------|----------|
| **emptyDir** | Pod lifetime only | Scratch space, caching |
| **hostPath** | Node-local | Single-node testing only |
| **PersistentVolume** | Beyond pod lifecycle | Production databases |
| **configMap/secret** | Config data | Configuration injection |
| **CSI Volumes** | Varies by driver | Cloud-native storage |

---

## Persistent Volumes and Claims

### Persistent Volume (PV)

A **PersistentVolume (PV)** is a cluster-level resource representing a piece of storage provisioned by an administrator or dynamically provisioned using StorageClasses.

**Key Characteristics:**
- Cluster-scoped resource (not namespaced)
- Lifecycle independent of any individual Pod
- Can be backed by NFS, iSCSI, cloud-provider storage, or CSI drivers

**Example PV Definition:**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-01
  labels:
    type: nfs
    environment: production
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  nfs:
    server: 192.168.1.100
    path: /exports/data
```

### Persistent Volume Claim (PVC)

A **PersistentVolumeClaim (PVC)** is a request for storage by a user, similar to how a Pod requests compute resources.

**Example PVC Definition:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-claim
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: fast-ssd
  selector:
    matchLabels:
      environment: production
```

### Access Modes

| Mode | Short | Description | Typical Backends |
|------|-------|-------------|------------------|
| **ReadWriteOnce** | RWO | Single node read-write | EBS, GCE PD, Azure Disk |
| **ReadOnlyMany** | ROX | Multiple nodes read-only | NFS, CephFS |
| **ReadWriteMany** | RWX | Multiple nodes read-write | NFS, CephFS, EFS |
| **ReadWriteOncePod** | RWOP | Single pod read-write | Modern CSI drivers |

### PV-PVC Lifecycle

```
┌──────────┐     Create      ┌──────────┐
│    PV    │ ──────────────► │ Available│
└──────────┘                 └────┬─────┘
                                  │
                    PVC requests  │
                    matching PV   ▼
                            ┌──────────┐
                            │  Bound   │
                            └────┬─────┘
                                 │
                    PVC deleted  │
                                 ▼
                            ┌──────────┐
                            │ Released │
                            └────┬─────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
              ▼                  ▼                  ▼
        ┌──────────┐      ┌──────────┐      ┌──────────┐
        │ Retained │      │ Deleted  │      │ Recycled │
        │ (Manual) │      │(Automatic)│     │(Deprecated)│
        └──────────┘      └──────────┘      └──────────┘
```

### Reclaim Policies

| Policy | Behavior | Use Case |
|--------|----------|----------|
| **Retain** | PV and data kept; manual cleanup required | Data preservation |
| **Delete** | PV and backing storage deleted automatically | Default for dynamic provisioning |
| **Recycle** | Volume scrubbed and reused | Deprecated (Kubernetes 1.35) |

---

## Storage Classes

**StorageClasses** enable dynamic provisioning of PersistentVolumes based on pre-defined policies. They abstract storage tier details from users.

### Key Benefits
- **Dynamic Provisioning**: Volumes created on-demand
- **Tiered Storage**: Different performance/cost options
- **Simplified Management**: No manual PV creation needed

### Example StorageClass Definitions

**AWS EBS gp3:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123456789:key/abc-123
```

**GCE Persistent Disk SSD:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gce-pd-ssd
provisioner: pd.csi.storage.gke.io
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
parameters:
  type: pd-ssd
```

### Volume Binding Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| **Immediate** | Volume provisioned immediately | Simple deployments |
| **WaitForFirstConsumer** | Volume provisioned when pod scheduled | Topology-aware scheduling |

---

## Container Storage Interface (CSI)

The **Container Storage Interface (CSI)** is a standard for exposing storage systems to containerized workloads. It replaced in-tree volume plugins for better extensibility.

### CSI Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CSI Architecture                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐      ┌──────────────┐                     │
│  │   External   │      │   External   │                     │
│  │ Provisioner  │      │  Attacher    │                     │
│  └──────┬───────┘      └──────┬───────┘                     │
│         │                     │                              │
│         └──────────┬──────────┘                              │
│                    │                                         │
│                    ▼                                         │
│  ┌────────────────────────────────────┐                     │
│  │         CSI Driver Controller       │                     │
│  │    (Create/Delete, Attach/Detach)   │                     │
│  └────────────────────────────────────┘                     │
│                    │                                         │
│                    │ gRPC                                    │
│                    ▼                                         │
│  ┌────────────────────────────────────┐                     │
│  │         CSI Driver Node             │                     │
│  │    (Mount/Unmount, Format)          │                     │
│  │         (DaemonSet)                 │                     │
│  └────────────────────────────────────┘                     │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Popular CSI Drivers

| Driver | Provider | Features |
|--------|----------|----------|
| **EBS CSI** | AWS | Block storage, encryption, snapshots |
| **GCE PD CSI** | GCP | Block storage, regional disks |
| **Azure Disk CSI** | Azure | Managed disks, encryption |
| **Longhorn** | Rancher/SUSE | Distributed block, replicas |
| **Rook-Ceph** | Ceph | Block, file, object storage |
| **NFS CSI** | Community | Shared file storage |

---

## Storage Solutions Comparison

### Decision Matrix

```
                    ┌─────────────────────────────────────┐
                    │    Where is your cluster running?   │
                    └───────────────┬─────────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
            ▼                       ▼                       ▼
    ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
    │  Cloud (AWS)  │      │  Cloud (GCP)  │      │  Bare Metal   │
    └───────┬───────┘      └───────┬───────┘      └───────┬───────┘
            │                       │                       │
            ▼                       ▼                       ▼
    ┌───────────────┐      ┌───────────────┐      ┌───────────────┐
    │   EBS CSI     │      │   GCE PD CSI  │      │ Need replication?│
    │  EFS CSI      │      │  Filestore    │      └───────┬───────┘
    └───────────────┘      └───────────────┘              │
                                                          │
                                    ┌─────────────────────┼─────────────────────┐
                                    │                     │                     │
                                    ▼                     ▼                     ▼
                            ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
                            │     Yes       │     │     Yes       │     │      No       │
                            │  Enterprise   │     │    Simple     │     │    NFS OK     │
                            └───────┬───────┘     └───────┬───────┘     └───────┬───────┘
                                    │                     │                     │
                                    ▼                     ▼                     ▼
                            ┌───────────────┐     ┌───────────────┐     ┌───────────────┐
                            │  Rook-Ceph    │     │   Longhorn    │     │   NFS CSI     │
                            └───────────────┘     └───────────────┘     └───────────────┘
```

### Longhorn vs Rook-Ceph Comparison

| Feature | Longhorn | Rook-Ceph |
|---------|----------|-----------|
| **Storage Types** | Block (RWO) | Block, File (RWX), Object |
| **ReadWriteMany** | No | Yes (CephFS) |
| **Object Storage** | No | Yes (RadosGW) |
| **Snapshots** | Yes | Yes |
| **Backups to S3** | Built-in | Via Velero |
| **Installation** | Simple (Helm) | Complex (Multi-step) |
| **Minimum Nodes** | 1 | 3 |
| **Memory Footprint** | Low | High |
| **CNCF Status** | Incubating | Graduated |
| **Best For** | Simplicity, Rancher | Enterprise, multi-protocol |

### Cloud-Native Storage Solutions Summary

| Solution | Best Fit | Strengths | Watch-outs |
|----------|----------|-----------|------------|
| **Ceph/Rook** | Bare metal/hybrid | Mature replication, erasure coding, multi-protocol | Operationally heavy, steep learning curve |
| **Longhorn** | Mid-sized clusters | Simple ops, built-in UI, edge-friendly | Only block storage, volume-level replication |
| **OpenEBS** | Mixed workloads | Multiple engines, flexible | Each engine has unique patterns |
| **Portworx** | Enterprise | 24/7 support, multi-cloud DR | License costs |
| **Cloud Block** | Single cloud | Managed HA, IOPS guarantees | Vendor lock-in |

---

## Ephemeral Storage

### Types of Ephemeral Volumes

| Type | Description | Use Case |
|------|-------------|----------|
| **emptyDir** | Empty at pod startup | Scratch space, caching |
| **configMap** | Inject config data | Application configuration |
| **secret** | Inject sensitive data | Credentials, certificates |
| **downwardAPI** | Pod metadata | Self-aware applications |
| **CSI Ephemeral** | Driver-provided ephemeral | Specialized storage needs |

### emptyDir Configuration

**Basic emptyDir:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache-pod
spec:
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: cache
      mountPath: /cache
  volumes:
  - name: cache
    emptyDir:
      sizeLimit: 500Mi
```

**Memory-backed emptyDir (tmpfs):**
```yaml
volumes:
- name: cache-volume
  emptyDir:
    medium: Memory
    sizeLimit: 500Mi
```

### Ephemeral Storage Limits

Configure ephemeral storage to prevent disk pressure:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    resources:
      requests:
        ephemeral-storage: "2Gi"
      limits:
        ephemeral-storage: "4Gi"
```

---

## StatefulSet Storage

StatefulSets use **volumeClaimTemplates** to provide stable, per-pod storage.

### How volumeClaimTemplates Work

```
┌─────────────────────────────────────────────────────────────┐
│              StatefulSet VolumeClaimTemplate Flow            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  StatefulSet with 3 replicas:                                │
│                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │   pod-0     │    │   pod-1     │    │   pod-2     │     │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘     │
│         │                  │                  │             │
│         ▼                  ▼                  ▼             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │  pvc-0      │    │  pvc-1      │    │  pvc-2      │     │
│  │ (from tpl)  │    │ (from tpl)  │    │ (from tpl)  │     │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘     │
│         │                  │                  │             │
│         ▼                  ▼                  ▼             │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │
│  │    pv-0     │    │    pv-1     │    │    pv-2     │     │
│  └─────────────┘    └─────────────┘    └─────────────┘     │
│                                                              │
│  Key: Each pod gets its own PVC, preserving identity         │
│       even after rescheduling                                │
└─────────────────────────────────────────────────────────────┘
```

### StatefulSet Example with Storage

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 1Gi
```

### Key Characteristics

- **Stable Identity**: Each pod gets a unique, persistent identity
- **Ordered Deployment**: Pods created sequentially (0, 1, 2...)
- **Persistent Storage**: PVCs survive pod deletion
- **Ordered Scaling**: Pods scaled down in reverse order

---

## Security and Encryption

### Encryption at Rest

By default, Kubernetes Secrets are stored as **base64-encoded** (not encrypted) in etcd. Encryption at rest is essential for production.

### Encryption Configuration

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: <base64-encoded-32-byte-key>
      - identity: {}  # Allow reading old unencrypted data
```

### Encryption Providers

| Provider | Algorithm | Notes |
|----------|-----------|-------|
| **aescbc** | AES-CBC | Recommended for most use cases |
| **aesgcm** | AES-GCM | Faster, requires careful key management |
| **secretbox** | XSalsa20 + Poly1305 | Alternative option |
| **kms** | External KMS | Best for enterprise (AWS KMS, Azure Key Vault) |

### Storage-Level Encryption

| Layer | Method | Implementation |
|-------|--------|----------------|
| **etcd** | Application-level | EncryptionConfiguration |
| **PV Data** | Backend encryption | Cloud provider encryption |
| **Node Disk** | Full-disk encryption | LUKS, BitLocker |

---

## Backup and Disaster Recovery

### Velero for Kubernetes Backup

**Velero** is the de facto standard for Kubernetes backup and restore, supporting:
- Resource manifest backups
- CSI volume snapshots
- Cross-cluster migration
- Scheduled backups

### Velero Installation

```bash
# Install Velero with CSI support
velero install \
  --features=EnableCSI \
  --use-volume-snapshots \
  --use-node-agent \
  --plugins=velero/velero-plugin-for-aws:v1.13.2 \
  --provider aws \
  --bucket my-backup-bucket \
  --secret-file /path/to/aws-credentials \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1
```

### Backup Operations

```bash
# Create a backup
velero backup create db-backup \
  --include-namespaces databases \
  --snapshot-volumes=true \
  --wait

# Schedule regular backups
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --include-namespaces production

# Check backup status
velero backup describe db-backup --details
```

### Restore Operations

```bash
# Restore from backup
velero restore create --from-backup db-backup --wait

# Restore to different namespace
velero restore create --from-backup db-backup \
  --namespace-mappings databases:databases-restored
```

### Volume Snapshots

CSI volume snapshots provide crash-consistent backups:

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-snapshot
spec:
  volumeSnapshotClassName: csi-snapclass
  source:
    persistentVolumeClaimName: mysql-pvc
```

### Restore from Snapshot

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  storageClassName: fast-ssd
  dataSource:
    name: mysql-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

---

## Monitoring and Metrics

### Key Storage Metrics

| Metric | Description | Importance |
|--------|-------------|------------|
| **Volume Usage** | Used vs available space | Capacity planning |
| **IOPS** | Read/write operations per second | Performance tuning |
| **Latency** | Response time for operations | User experience |
| **Throughput** | Data transfer rate | Workload optimization |
| **Inode Usage** | Filesystem inode consumption | Prevent exhaustion |

### Prometheus Storage Monitoring

```yaml
# PrometheusRule for storage alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: storage-alerts
spec:
  groups:
    - name: storage
      rules:
        - alert: PersistentVolumeFillingUp
          expr: |
            kubelet_volume_stats_available_bytes /
            kubelet_volume_stats_capacity_bytes < 0.1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "PV {{ $labels.persistentvolume }} is filling up"
            
        - alert: NodeDiskPressure
          expr: |
            kube_node_status_condition{
              condition="DiskPressure",
              status="true"
            } == 1
          for: 5m
          labels:
            severity: critical
```

### CSI Metrics

CSI drivers can expose metrics through the `csi-metrics` endpoint:

```yaml
# ServiceMonitor for CSI metrics
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: csi-metrics
spec:
  selector:
    matchLabels:
      app: csi-driver
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

---

## Best Practices

### General Guidelines

1. **Use Dynamic Provisioning**
   - Prefer StorageClasses over manual PV creation
   - Simplifies operations and enables self-service

2. **Right-Size Storage**
   - Monitor actual usage before provisioning
   - Use volume expansion when needed
   - Set appropriate resource quotas

3. **Implement Backup Strategies**
   - Regular automated backups with Velero
   - Test restore procedures
   - Store backups off-site

4. **Enable Encryption**
   - Encrypt secrets at rest in etcd
   - Enable storage backend encryption
   - Use KMS for key management

5. **Monitor Storage Health**
   - Track capacity, performance, and errors
   - Set up alerts for critical thresholds
   - Review metrics regularly

### StorageClass Best Practices

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: production-standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer  # Topology-aware
allowVolumeExpansion: true               # Enable resizing
reclaimPolicy: Delete                    # Auto-cleanup
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:region:account:key/id"
```

### Security Best Practices

1. **RBAC for Storage Resources**
   - Limit who can create/delete PVs and PVCs
   - Use least-privilege access

2. **Network Policies**
   - Control traffic to storage pods
   - Isolate storage traffic

3. **Pod Security**
   - Run storage components as non-root
   - Use read-only root filesystems where possible

4. **Encryption**
   - Enable encryption at rest for etcd
   - Use encrypted storage backends
   - Rotate encryption keys regularly

### Troubleshooting Common Issues

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **PVC Pending** | No matching PV | Check StorageClass, capacity, access modes |
| **Mount Failures** | Pod stuck ContainerCreating | Verify node access, permissions |
| **Disk Pressure** | Node tainted, pods evicted | Clean up images, logs; set limits |
| **Snapshot Failures** | Snapshot not ready | Check CSI driver, snapshot class |

---

## Summary

Kubernetes storage has evolved significantly with CSI becoming the standard interface. Key takeaways:

1. **CSI is the Future**: All new storage implementations should use CSI drivers
2. **Choose Based on Needs**: Cloud-native for simplicity, distributed for bare metal
3. **Plan for Growth**: Use volume expansion, monitor capacity
4. **Security First**: Enable encryption at all layers
5. **Backup Regularly**: Velero + snapshots for disaster recovery
6. **Monitor Everything**: Storage metrics are critical for reliability

### Quick Selection Guide

| Scenario | Recommended Solution |
|----------|---------------------|
| AWS-only workloads | EBS/EFS CSI drivers |
| GCP-only workloads | GCE PD/Filestore CSI |
| Simple bare metal | Longhorn |
| Enterprise multi-protocol | Rook-Ceph |
| Maximum simplicity | Cloud managed storage |
| Enterprise support | Portworx or similar |

---

*Research compiled from Kubernetes documentation, cloud provider guides, and industry best practices as of April 2026.*
