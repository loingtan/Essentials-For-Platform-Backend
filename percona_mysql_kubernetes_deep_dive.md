
# Deep Dive: Operating Percona MySQL in Kubernetes

## Executive Summary

Running Percona MySQL (Percona XtraDB Cluster and Percona Server for MySQL) in Kubernetes has evolved from experimental to production-ready. With the Percona Operators, organizations can now deploy, manage, and scale enterprise-grade MySQL clusters with the same cloud-native principles applied to stateless applications. This deep dive covers architecture, deployment patterns, operational best practices, and production considerations.

---

## 1. Percona MySQL Kubernetes Operators Overview

### 1.1 Two Operator Options

Percona provides two distinct Kubernetes operators for MySQL:

#### Percona Operator for MySQL based on Percona XtraDB Cluster (PXC)
- **Replication Model**: Synchronous multi-master replication using Galera
- **Best For**: Applications requiring strong consistency and true multi-master capabilities
- **Current Version**: 1.19.0 (as of January 2026)
- **Status**: Production-ready (GA)

#### Percona Operator for MySQL based on Percona Server for MySQL (PS)
- **Replication Models**: 
  - Group Replication (GA, recommended for production)
  - Asynchronous Replication (Tech Preview)
- **Best For**: Traditional primary-replica setups with read scaling
- **Current Version**: 1.0.0
- **Status**: GA for Group Replication

### 1.2 Key Features Comparison

| Feature | PXC Operator | PS Operator |
|---------|--------------|-------------|
| Replication | Synchronous (Galera) | Group/Async |
| Multi-Master | Yes (true multi-master) | Single primary |
| Max Nodes | 5 recommended | 9 (Group Replication) |
| Automatic Failover | Yes | Yes |
| Read Scaling | Yes | Yes |
| Write Scaling | Horizontal | Vertical only |

---

## 2. Architecture Deep Dive

### 2.1 Percona XtraDB Cluster Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │  PXC Node 0 │  │  PXC Node 1 │  │  PXC Node 2 │         │
│  │  (Primary)  │  │  (Primary)  │  │  (Primary)  │         │
│  │  + PMM      │  │  + PMM      │  │  + PMM      │         │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │
│         │                │                │                │
│         └────────────────┼────────────────┘                │
│                          │ Galera Replication              │
│  ┌───────────────────────┴───────────────────────┐         │
│  │           HAProxy / ProxySQL Layer            │         │
│  │     (Load Balancing & Connection Pooling)     │         │
│  └───────────────────────┬───────────────────────┘         │
│                          │                                  │
│                   ┌──────┴──────┐                          │
│                   │  Application │                          │
│                   └─────────────┘                          │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 High Availability Design

**Pod Distribution Strategy**:
- Uses node affinity to distribute instances across separate worker nodes
- Prevents single node failure from affecting multiple database instances
- Automatic pod rescheduling on node failures

**Synchronous Replication (PXC)**:
- All writes are certified by majority of nodes before commit
- Eliminates replication lag
- Guarantees data consistency across all nodes
- Supports true multi-master (any node can accept writes)

**Quorum Requirements**:
- Odd number of nodes required (3, 5 recommended)
- Even numbers not recommended (split-brain risk)
- Maximum 5 nodes for optimal performance (more nodes = higher latency)

### 2.3 Storage Architecture

**Persistent Volume Strategy**:
```yaml
volumeSpec:
  persistentVolumeClaim:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 100Gi
    storageClassName: fast-ssd  # Critical for performance
```

**Storage Requirements**:
- Must support PVC expansion for growth
- SSD/NVMe storage strongly recommended
- Separate storage class for databases recommended
- CSI driver must support volume attachment to different nodes

---

## 3. Deployment Guide

### 3.1 Prerequisites

**Kubernetes Requirements**:
- Kubernetes 1.28+ (tested on GKE, EKS, AKS, OpenShift 4.13+)
- kubectl configured
- StorageClass with AllowVolumeExpansion: true
- Minimum 3 worker nodes for HA

**Resource Planning**:
```yaml
# Minimum production resources per PXC node
resources:
  requests:
    memory: 4Gi
    cpu: 2
  limits:
    memory: 8Gi
    cpu: 4
```

### 3.2 Installation Steps

**Step 1: Create Namespace**
```bash
kubectl create namespace percona-mysql
```

**Step 2: Deploy PXC Operator**
```bash
kubectl apply --server-side -f   https://raw.githubusercontent.com/percona/percona-xtradb-cluster-operator/v1.19.0/deploy/bundle.yaml   -n percona-mysql
```

**Step 3: Configure Secrets**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster1-secrets
stringData:
  root: root_password
  xtrabackup: backup_password
  monitor: monitor_password
  clustercheck: clustercheck_password
  proxyadmin: proxyadmin_password
  pmmserver: pmmserver_password
  operator: operator_password
  replication: replication_password
```

**Step 4: Deploy Cluster**
```yaml
apiVersion: pxc.percona.com/v1
kind: PerconaXtraDBCluster
metadata:
  name: cluster1
spec:
  crVersion: 1.19.0
  secretsName: cluster1-secrets

  pxc:
    size: 3
    image: percona/percona-xtradb-cluster:8.0.42-33.1
    resources:
      requests:
        memory: 4Gi
        cpu: 2
      limits:
        memory: 8Gi
        cpu: 4
    volumeSpec:
      persistentVolumeClaim:
        storageClassName: fast-ssd
        resources:
          requests:
            storage: 100Gi
    configuration: |
      [mysqld]
      innodb_buffer_pool_size=4G
      innodb_log_file_size=1G
      max_connections=500
      wsrep_provider_options="gcache.size=2G"

  haproxy:
    enabled: true
    size: 3
    image: percona/haproxy:2.8.15

  pmm:
    enabled: true
    image: percona/pmm-client:2.44.1-1
    serverHost: monitoring-service
```

### 3.3 Verification

```bash
# Check cluster status
kubectl get pxc -n percona-mysql

# Expected output:
# NAME       ENDPOINT                   STATUS   PXC   HAPROXY   AGE
# cluster1   cluster1-haproxy.default   ready    3     3         5m51s

# Check pod status
kubectl get pods -n percona-mysql

# Check logs
kubectl logs cluster1-pxc-0 -n percona-mysql -c pxc
```

---

## 4. Security Configuration

### 4.1 TLS/SSL Encryption

**Automatic Certificate Generation** (Default):
- Operator generates long-term certificates automatically
- Requires manual renewal

**Cert-Manager Integration** (Recommended):
```yaml
spec:
  tls:
    enabled: true
  unsafeFlags:
    tls: false
```

**Custom CA Certificates**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cluster1-ssl-internal
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-cert>
  tls.key: <base64-encoded-key>
  ca.crt: <base64-encoded-ca>
```

### 4.2 Data-at-Rest Encryption (PXC 8.4+)

```yaml
spec:
  pxc:
    configuration: |
      [mysqld]
      early-plugin-load=keyring_vault.so
      keyring_vault_config=/etc/mysql/keyring_vault.conf
```

### 4.3 Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: pxc-network-policy
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/component: pxc
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/component: haproxy
      ports:
        - protocol: TCP
          port: 3306
```

### 4.4 RBAC Configuration

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: percona-mysql-operator
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "endpoints", "persistentvolumeclaims"]
    verbs: ["*"]
  - apiGroups: ["pxc.percona.com"]
    resources: ["perconaxtradbclusters"]
    verbs: ["*"]
```

---

## 5. Backup and Recovery

### 5.1 Backup Architecture

**Supported Storage Types**:
- S3-compatible storage (AWS S3, MinIO, etc.)
- Azure Blob Storage
- Persistent Volumes (PVC)

**Backup Types**:
- Full backups (physical with XtraBackup)
- Incremental backups
- Point-in-Time Recovery (PITR) with binary logs

### 5.2 Configuring Scheduled Backups

```yaml
spec:
  backup:
    image: percona/percona-xtradb-cluster-operator:1.19.0-pxc8.0-backup

    pitr:
      enabled: true
      storageName: s3-backup
      timeBetweenUploads: 60  # seconds

    schedule:
      - name: daily-backup
        schedule: "0 2 * * *"  # Daily at 2 AM
        keep: 7  # Keep 7 backups
        storageName: s3-backup

      - name: weekly-backup
        schedule: "0 3 * * 0"  # Weekly on Sunday
        keep: 4  # Keep 4 backups
        storageName: s3-backup

    storages:
      s3-backup:
        type: s3
        s3:
          bucket: xtradb-backups
          region: us-west-2
          credentialsSecret: backup-s3-credentials
          endpointUrl: https://s3.us-west-2.amazonaws.com
          verifyTLS: true
          caBundle:
            name: s3-ca-bundle
            key: ca.crt
```

### 5.3 Point-in-Time Recovery

**Enable PITR**:
```yaml
spec:
  backup:
    pitr:
      enabled: true
      storageName: s3-backup
      timeBetweenUploads: 60
```

**Restore to Specific Time**:
```yaml
apiVersion: pxc.percona.com/v1
kind: PerconaXtraDBClusterRestore
metadata:
  name: restore1
spec:
  pxcCluster: cluster1
  backupName: backup1
  pitr:
    type: date
    date: "2026-01-15 14:30:00"
```

**Restore Types**:
- `date`: Roll back to specific datetime
- `transaction`: Roll back to specific GTID
- `latest`: Recover to most recent transaction
- `skip`: Skip specific transaction

### 5.4 Cross-Cluster Restore

```yaml
apiVersion: pxc.percona.com/v1
kind: PerconaXtraDBClusterRestore
metadata:
  name: restore-new-cluster
spec:
  pxcCluster: cluster2  # Different cluster
  backupSource:
    destination: s3://bucket/backup-name
    s3:
      bucket: xtradb-backups
      region: us-west-2
      credentialsSecret: backup-s3-credentials
```

---

## 6. Monitoring with Percona Monitoring and Management (PMM)

### 6.1 PMM Server Deployment

**Helm Installation**:
```bash
helm repo add percona https://percona.github.io/percona-helm-charts/
helm repo update

helm install pmm percona/pmm   --namespace monitoring   --set service.type=LoadBalancer   --set storage.storageClassName=fast-ssd   --set storage.size=100Gi   --set resources.requests.memory=4Gi   --set resources.requests.cpu=2
```

### 6.2 PMM Client Configuration

**Create Service Account Token** (PMM 3.x):
```bash
# Generate token in PMM UI or via API
# Token format: glsa_*************************_9e35351b
```

**Configure Cluster Monitoring**:
```yaml
spec:
  pmm:
    enabled: true
    image: percona/pmm-client:3.4.1
    serverHost: pmm-server.monitoring.svc.cluster.local
    serverUser: admin
    customClusterName: production-pxc
    resources:
      requests:
        memory: 150M
        cpu: 300m
```

**Add Token to Secrets**:
```bash
kubectl patch secret/cluster1-secrets -p   '{"data":{"pmmservertoken":"'$(echo -n <token> | base64 --wrap=0)'"}}'
```

### 6.3 Key Metrics to Monitor

**Cluster Health**:
- `wsrep_cluster_size`: Number of nodes in cluster
- `wsrep_cluster_status`: Primary/Non-Primary
- `wsrep_local_state`: Node state (Joined, Synced, etc.)
- `wsrep_flow_control_paused`: Flow control percentage

**Replication**:
- `wsrep_local_recv_queue`: Receive queue length
- `wsrep_local_send_queue`: Send queue length
- `wsrep_replicated_bytes`: Bytes replicated
- `wsrep_received_bytes`: Bytes received

**Performance**:
- `mysql_global_status_innodb_buffer_pool_reads`
- `mysql_global_status_slow_queries`
- `mysql_global_status_threads_connected`
- `mysql_global_status_threads_running`

---

## 7. Scaling Strategies

### 7.1 Horizontal Scaling (Adding Nodes)

```bash
# Scale from 3 to 5 nodes
kubectl patch pxc cluster1 --type=merge   -p '{"spec":{"pxc":{"size":5}}}}'
```

**Considerations**:
- PXC: Use odd numbers only (3, 5)
- Group Replication: Maximum 9 nodes
- New nodes join via SST (State Snapshot Transfer)
- Initial sync may impact performance

### 7.2 Vertical Scaling (Resource Changes)

```yaml
spec:
  pxc:
    resources:
      requests:
        memory: 8Gi  # Increased from 4Gi
        cpu: 4       # Increased from 2
      limits:
        memory: 16Gi
        cpu: 8
```

**Update Strategy**:
- Smart Update (default): Controlled rolling restart
- Rolling Update: Kubernetes-controlled restart
- OnDelete: Manual pod deletion

### 7.3 Storage Scaling

**Automatic Volume Expansion**:
```yaml
spec:
  enableVolumeExpansion: true
  pxc:
    volumeSpec:
      persistentVolumeClaim:
        resources:
          requests:
            storage: 200Gi  # Increased from 100Gi
```

**Manual Resizing** (if VolumeExpansion not supported):
1. Update CR with new storage size
2. Delete pods one by one
3. Let Kubernetes recreate with new PVCs
4. Data resyncs from other nodes

---

## 8. Connection Management

### 8.1 HAProxy Configuration

```yaml
spec:
  haproxy:
    enabled: true
    size: 3
    image: percona/haproxy:2.8.15

    # Read/write splitting
    exposeReplicas:
      enabled: true
      onlyReaders: true  # Exclude writer from read pool
```

**Connection Flow**:
```
App → HAProxy Service (3306) → Backend PXC Nodes
     ↓
   Health checks every 2s
   Automatic failover on failure
```

### 8.2 ProxySQL Configuration (Advanced)

```yaml
spec:
  proxysql:
    enabled: true
    size: 2
    image: percona/proxysql2:2.7.3

    configuration: |
      mysql_variables=
      {
        threads=4
        max_connections=2048
        default_query_timeout=3600000
        have_compress=true
        poll_timeout=2000
      }

    # External scheduler for intelligent routing
    scheduler:
      enabled: true
      writerIsAlsoReader: false
      checkTimeoutMilliseconds: 2000
      successThreshold: 1
      failureThreshold: 3
```

**ProxySQL Benefits**:
- Connection pooling (multiplexing)
- Query caching
- Read/write splitting
- Query rewriting
- SQL-aware routing

### 8.3 Application Connection Best Practices

```python
# Connection string example
# Use HAProxy service for general connections
mysql://user:pass@cluster1-haproxy:3306/database

# For read-only workloads
mysql://user:pass@cluster1-haproxy-replicas:3306/database
```

**Connection Pool Settings**:
- Pool size: 10-20 connections per app instance
- Max connections in MySQL: 500+ (depending on RAM)
- Connection timeout: 30 seconds
- Read timeout: 30 seconds

---

## 9. Upgrade Procedures

### 9.1 Upgrade Strategies

**Smart Update** (Recommended):
```yaml
spec:
  updateStrategy: SmartUpdate
```
- Operator-controlled rolling restart
- Primary updated last
- Minimizes connection issues

**Rolling Update**:
```yaml
spec:
  updateStrategy: RollingUpdate
```
- Kubernetes-controlled restart
- May not be optimal order

**OnDelete** (Manual):
```yaml
spec:
  updateStrategy: OnDelete
```
- Manual pod deletion required
- Full control over timing

### 9.2 Version Upgrade Steps

```bash
# 1. Check version compatibility
curl https://check.percona.com/versions/v1/pxc-operator/1.19.0 | jq

# 2. Update CR with new versions
kubectl patch pxc cluster1 --type=merge -p '{
  "spec": {
    "crVersion":"1.19.0",
    "pxc":{"image": "percona/percona-xtradb-cluster:8.0.42-33.1"},
    "haproxy": {"image": "percona/haproxy:2.8.15"},
    "backup": {"image": "percona/percona-xtrabackup:8.0.35-34.1"}
  }}'

# 3. Monitor rollout
kubectl rollout status sts cluster1-pxc
```

### 9.3 Major Version Upgrades (8.0 → 8.4)

**Prerequisites**:
- Must be on latest 8.0 version first
- Backup before upgrade
- Test in staging environment

**Rolling Upgrade Process**:
1. Stop one 8.0 node
2. Remove 8.0 packages
3. Install 8.4 packages
4. Restart node (auto-upgrades data directory)
5. Repeat for each node

**Key Changes in 8.4**:
- Keyring component replaces keyring plugin
- Default auth: caching_sha2_password
- PXC Strict Mode enabled by default
- Encrypted cluster traffic by default

---

## 10. Performance Tuning

### 10.1 MySQL Configuration

```yaml
spec:
  pxc:
    configuration: |
      [mysqld]
      # Buffer Pool (70-80% of available RAM)
      innodb_buffer_pool_size=6G

      # Log Files
      innodb_log_file_size=1G
      innodb_log_files_in_group=2

      # Flush Behavior
      innodb_flush_log_at_trx_commit=2
      innodb_flush_method=O_DIRECT

      # Connections
      max_connections=500

      # Galera Tuning
      wsrep_provider_options="gcache.size=2G;gcs.fc_limit=100"
      wsrep_slave_threads=8

      # Query Cache (disabled in 8.0+)
      query_cache_type=0

      # Table Performance
      table_open_cache=4000
      table_definition_cache=2000
```

### 10.2 Resource Allocation Guidelines

| Workload Type | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---------------|-------------|-----------|----------------|--------------|
| Small/Dev | 1 | 2 | 2Gi | 4Gi |
| Medium | 2 | 4 | 4Gi | 8Gi |
| Large/Production | 4 | 8 | 8Gi | 16Gi |
| XL/Analytics | 8 | 16 | 16Gi | 32Gi |

### 10.3 Storage Performance

**Storage Class Requirements**:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com  # Cloud-specific
parameters:
  type: gp3  # or io1/io2 for higher IOPS
  iopsPerGB: "50"
  encrypted: "true"
allowVolumeExpansion: true
mountOptions:
  - noatime
```

**I/O Optimization**:
- Use SSD/NVMe storage
- Enable volume encryption
- Set `innodb_flush_method=O_DIRECT`
- Monitor IOPS utilization

---

## 11. Troubleshooting Guide

### 11.1 Common Issues

**Pod Stuck in Pending**:
```bash
# Check events
kubectl describe pod cluster1-pxc-0

# Common causes:
# - Insufficient resources
# - PVC not binding
# - Node affinity constraints
```

**Cluster Not Syncing**:
```bash
# Check node status
kubectl exec -it cluster1-pxc-0 -- mysql -e "SHOW STATUS LIKE 'wsrep%';"

# Key indicators:
# wsrep_local_state_comment: Synced
# wsrep_cluster_size: 3
# wsrep_connected: ON
```

**Flow Control Issues**:
```bash
# Check flow control
mysql -e "SHOW STATUS LIKE 'wsrep_flow_control_paused';"

# If > 0.1 consistently:
# - Increase gcache.size
# - Add more CPU resources
# - Check network latency
```

### 11.2 Crash Recovery

```bash
# Check for grastate.dat
kubectl exec -it cluster1-pxc-0 -- cat /var/lib/mysql/grastate.dat

# If safe_to_bootstrap: 1 on any node
# That node can bootstrap the cluster

# Force bootstrap if needed
kubectl patch pxc cluster1 --type=merge   -p '{"spec":{"pxc":{"autoRecovery":true}}}}'
```

### 11.3 Log Collection

```bash
# Operator logs
kubectl logs deployment/percona-xtradb-cluster-operator

# Database logs
kubectl logs cluster1-pxc-0 -c pxc

# Log collector (if enabled)
kubectl logs cluster1-pxc-0 -c logs
```

---

## 12. Production Best Practices

### 12.1 Deployment Checklist

**Pre-Deployment**:
- [ ] Resource quotas defined
- [ ] Storage class configured with expansion
- [ ] Network policies defined
- [ ] TLS certificates ready
- [ ] Backup storage configured
- [ ] PMM server deployed
- [ ] Monitoring alerts configured

**Post-Deployment**:
- [ ] Verify cluster status: `kubectl get pxc`
- [ ] Test failover scenarios
- [ ] Verify backup completion
- [ ] Test restore procedure
- [ ] Configure log aggregation
- [ ] Document runbooks

### 12.2 Day-2 Operations

**Regular Tasks**:
- Monitor cluster health daily
- Review slow query logs weekly
- Test backup restores monthly
- Update passwords quarterly
- Review resource utilization monthly

**Alerting Rules**:
```yaml
# Example Prometheus alerts
- alert: PXCClusterDown
  expr: mysql_global_status_wsrep_cluster_size < 3
  for: 5m

- alert: PXCNodeNotSynced
  expr: mysql_global_status_wsrep_local_state != 4
  for: 10m

- alert: PXCFlowControlHigh
  expr: mysql_global_status_wsrep_flow_control_paused > 0.1
  for: 5m
```

### 12.3 Disaster Recovery

**RPO/RTO Targets**:
- RPO: 5 minutes (with PITR)
- RTO: 15 minutes (automated failover)

**DR Strategies**:
1. **Same-Region HA**: Multi-AZ deployment
2. **Cross-Region DR**: Backup replication to secondary region
3. **Active-Active**: Multi-region cluster (complex)

---

## 13. Cost Optimization

### 13.1 Right-Sizing

- Start with smaller instances
- Monitor actual usage
- Scale based on metrics, not guesses
- Use spot instances for dev/test

### 13.2 Storage Optimization

- Enable compression
- Archive old data
- Use lifecycle policies for backups
- Right-size PVC requests

### 13.3 Resource Scheduling

```yaml
spec:
  pxc:
    nodeSelector:
      workload-type: database
    tolerations:
      - key: "dedicated"
        operator: "Equal"
        value: "mysql"
        effect: "NoSchedule"
```

---

## 14. References and Resources

### Official Documentation
- [Percona Operator for MySQL (PXC)](https://docs.percona.com/percona-operator-for-mysql/pxc/)
- [Percona Operator for MySQL (PS)](https://docs.percona.com/percona-operator-for-mysql/ps/)
- [Percona Monitoring and Management](https://docs.percona.com/percona-monitoring-and-management/)

### GitHub Repositories
- [PXC Operator](https://github.com/percona/percona-xtradb-cluster-operator)
- [PS Operator](https://github.com/percona/percona-server-mysql-operator)

### Community Resources
- [Percona Community Forum](https://forums.percona.com/)
- [Percona Blog](https://www.percona.com/blog/)

---

## 15. Summary

Operating Percona MySQL in Kubernetes requires understanding both MySQL internals and Kubernetes primitives. The Percona Operators abstract much of the complexity, but production success depends on:

1. **Proper Planning**: Resource allocation, storage selection, and architecture design
2. **Security**: TLS encryption, network policies, and access controls
3. **Monitoring**: Comprehensive observability with PMM
4. **Backup Strategy**: Automated backups with PITR capability
5. **Operational Excellence**: Regular testing, documentation, and runbooks

With these practices in place, Percona MySQL on Kubernetes delivers enterprise-grade reliability with cloud-native agility.

---

*Document Version: 1.0*
*Last Updated: April 2026*
