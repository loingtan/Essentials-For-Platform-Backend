# Deep Dive: Operating Redis in Kubernetes

## Executive Summary

Running Redis in Kubernetes requires careful consideration of architecture, persistence, high availability, and operational practices. This guide provides a comprehensive analysis of deploying and managing Redis on Kubernetes, covering deployment patterns, operators, storage strategies, monitoring, security, and production best practices.

---

## 1. Redis Deployment Architectures in Kubernetes

### 1.1 Architecture Patterns Overview

| Architecture | Use Case | Scalability | Complexity | Data Sharding |
|--------------|----------|-------------|------------|---------------|
| **Standalone** | Development, simple caching | None | Low | No |
| **Leader-Follower (Replication)** | Read-heavy workloads, HA | Read scaling | Medium | No |
| **Redis Sentinel** | Automatic failover, HA | Read scaling | Medium | No |
| **Redis Cluster** | Large datasets, write scaling | Full horizontal | High | Yes (16,384 slots) |
| **Proxy-based** | Simple client, complex backend | Via proxies | High | Transparent |

### 1.2 Redis Sentinel vs Redis Cluster

**Redis Sentinel** (High Availability Focus):
- Monitors master-replica topology
- Provides automatic failover
- Single master for writes (limits write throughput)
- Clients connect through Sentinel for service discovery
- Requires Sentinel-aware clients

**Redis Cluster** (Scaling Focus):
- Data sharded across 16,384 hash slots
- Multiple masters for parallel writes
- Built-in failover via replica promotion
- Requires Cluster-aware clients
- Supports multi-key operations only within same hash slot
- Uses hash tags for colocation: `{user:123}:profile` and `{user:123}:settings`

### 1.3 Hash Slots and Data Distribution

Redis Cluster uses CRC16 hashing for slot assignment:
```
SLOT = CRC16(key) mod 16384
```

With hash tags, only the tagged portion is hashed:
```
Key: user:{123}:profile → hashes "123"
Key: user:{123}:settings → hashes "123" (same slot)
```

This enables multi-key transactions for related data.

---

## 2. Kubernetes Deployment Patterns

### 2.1 StatefulSet-Based Deployment

Redis should use **StatefulSets** for:
- Stable network identities (persistent pod names)
- Ordered deployment and scaling
- Persistent storage per pod
- Stable DNS entries for service discovery

**Example StatefulSet Configuration:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  serviceName: redis-headless
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7.2
        ports:
        - containerPort: 6379
          name: redis
        - containerPort: 16379
          name: cluster-bus
        resources:
          requests:
            memory: "4Gi"
            cpu: "1"
          limits:
            memory: "6Gi"
            cpu: "2"
        volumeMounts:
        - name: data
          mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 50Gi
```

### 2.2 Headless Service for Pod Discovery

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
spec:
  clusterIP: None  # Headless service
  selector:
    app: redis
  ports:
  - port: 6379
    name: redis
  - port: 16379
    name: cluster-bus
```

### 2.3 Pod Anti-Affinity for HA

Prevent master and replica pods from running on the same node:

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values:
          - redis
      topologyKey: "kubernetes.io/hostname"
```

### 2.4 Topology Spread Constraints for Multi-AZ

Distribute pods across availability zones:

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: redis
```

---

## 3. Redis Operators Comparison

### 3.1 Popular Operators

| Operator | Maturity | Modes Supported | Key Features | License |
|----------|----------|-----------------|--------------|---------|
| **OT-Container-Kit (Opstree)** | High | Standalone, Sentinel, Cluster | StatefulSet-native, built-in metrics, backup/restore | Apache 2.0 |
| **Spotahome** | Mature | Sentinel-focused | Battle-tested, large community | Apache 2.0 |
| **Redis Enterprise** | Enterprise | All + Active-Active | Commercial support, advanced features | Commercial |
| **KubeDB** | Mature | All modes | Multi-database ecosystem | Enterprise/Apache |
| **HoWL (Howl Cloud)** | Modern | Standalone, Sentinel | Fencing-first failover, safety-focused | Apache 2.0 |

### 3.2 Operator Feature Comparison

| Feature | Opstree | Spotahome | HoWL | Redis Enterprise |
|---------|---------|-----------|------|------------------|
| Fencing-first failover | ❌ | ❌ | ✅ | ✅ |
| Boot-time split-brain guard | ❌ | ❌ | ✅ | ✅ |
| Supervised primary updates | ❌ | ❌ | ✅ | ✅ |
| Direct Pod/PVC management | ❌ | ❌ | ✅ | ⚠️ |
| Projected volume secrets | ⚠️ | ⚠️ | ✅ | ✅ |
| Per-pod status tracking | ⚠️ | ⚠️ | ✅ | ✅ |
| Built-in Prometheus metrics | ✅ | ✅ | ⚠️ | ✅ |
| Backup/Restore | ✅ | ⚠️ | ✅ | ✅ |
| Redis Cluster support | ✅ | ❌ | ⏳ | ✅ |

### 3.3 When to Choose Each Operator

**Choose Opstree if:**
- You need Redis Cluster sharding today
- You prefer StatefulSet-native workflows
- You want extensive Helm charts and documentation
- You need built-in Prometheus/Grafana integration

**Choose HoWL if:**
- Safety is paramount (fencing-first failover)
- You want predictable upgrades with approval gates
- You prefer pod-precise operations
- You're inspired by CloudNativePG design philosophy

**Choose Redis Enterprise if:**
- You need commercial support and SLAs
- You require Active-Active geo-replication
- You need Redis on Flash (RoF)
- Budget allows for enterprise licensing

---

## 4. Storage and Persistence

### 4.1 Persistence Options

| Method | Durability | Performance Impact | Use Case |
|--------|------------|-------------------|----------|
| **RDB (Snapshots)** | Point-in-time | Fork latency spikes | Backup, disaster recovery |
| **AOF (Append-Only)** | Near real-time | fsync overhead | Maximum durability |
| **RDB + AOF** | High | Combined overhead | Production safety |
| **No Persistence** | None | Best performance | Pure cache workloads |

### 4.2 StorageClass Configuration

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: redis-fast-ssd
provisioner: ebs.csi.aws.com  # Cloud-specific
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer  # Critical for multi-AZ
reclaimPolicy: Retain
allowVolumeExpansion: true
```

**Key Settings:**
- `volumeBindingMode: WaitForFirstConsumer` - Ensures PV is created in same AZ as pod
- `allowVolumeExpansion: true` - Enables online PVC resizing
- Use high-IOPS SSD for Redis workloads

### 4.3 PVC Expansion

```bash
# Edit PVC to increase size
kubectl edit pvc data-redis-0

# Update spec.resources.requests.storage
spec:
  resources:
    requests:
      storage: 100Gi  # Increased from 50Gi
```

**Requirements:**
- StorageClass must have `allowVolumeExpansion: true`
- Storage driver must support online expansion
- Backup before resizing (recommended)

### 4.4 Multi-AZ Storage Considerations

**Challenge:** Persistent volumes are zone-specific. If a Redis pod fails over to a different AZ, it cannot access its original PV.

**Solutions:**
1. **Disable persistence for cache workloads:**
   ```yaml
   redis:
     config:
       save: ""        # Disable RDB
       appendonly: "no" # Disable AOF
   persistentVolume:
     enabled: false
   ```

2. **Use zone-aware storage with replication:**
   - Redis replicas in different zones
- Async replication handles zone failover
- New replica promoted, data rebuilt from replication

3. **Use cross-zone storage solutions:**
   - EFS (AWS), Filestore (GCP), Azure Files
   - Higher latency, but zone-independent

---

## 5. High Availability Configuration

### 5.1 Pod Disruption Budget (PDB)

Ensure minimum availability during disruptions:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: redis-pdb
spec:
  minAvailable: 2  # Or use maxUnavailable
  selector:
    matchLabels:
      app: redis
```

**Recommendations by Replica Count:**
| Replicas | Recommended PDB | Rationale |
|----------|-----------------|-----------|
| 1 | maxUnavailable: 1 or none | Can't avoid downtime |
| 2 | maxUnavailable: 1 | Keep 1 running |
| 3 | minAvailable: 2 | Maintain majority/quorum |
| 5+ | maxUnavailable: 25% | Percentage-based |

### 5.2 Graceful Shutdown with PreStop Hooks

```yaml
lifecycle:
  preStop:
    exec:
      command:
      - /bin/sh
      - -c
      - |
        # Prevent new connections
        redis-cli CLIENT PAUSE 30000
        # Wait for replication sync
        sleep 10
```

### 5.3 Health Checks

```yaml
livenessProbe:
  exec:
    command:
    - redis-cli
    - ping
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  exec:
    command:
    - redis-cli
    - ping
  initialDelaySeconds: 5
  periodSeconds: 5

startupProbe:
  exec:
    command:
    - redis-cli
    - ping
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 30
```

### 5.4 Multi-AZ Deployment Pattern

```yaml
# Spread across 3 zones
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: redis

# Avoid same node
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchLabels:
          app: redis
      topologyKey: kubernetes.io/hostname
```

---

## 6. Resource Management

### 6.1 Resource Requests and Limits

**Redis is memory-bound - proper sizing is critical:**

```yaml
resources:
  requests:
    memory: "4Gi"    # Set to expected working set size
    cpu: "1000m"
  limits:
    memory: "6Gi"    # Headroom for spikes
    cpu: "2000m"
```

**Best Practices:**
- Set requests = limits for Guaranteed QoS class
- Memory request should fit working set + overhead
- Leave 20-30% headroom in limits
- CPU limits can cause throttling; consider unsetting for latency-sensitive workloads

### 6.2 Memory Optimization

```yaml
# redis.conf settings
maxmemory 3gb
maxmemory-policy allkeys-lru  # or volatile-lru, allkeys-lfu
```

**Eviction Policies:**
| Policy | Description | Use Case |
|--------|-------------|----------|
| `allkeys-lru` | Evict least recently used | General cache |
| `allkeys-lfu` | Evict least frequently used | Cache with hot data |
| `volatile-lru` | Evict LRU with TTL | Session store |
| `noeviction` | Return errors at limit | Data store (no cache) |

### 6.3 Node Eviction Thresholds

Configure kubelet to prevent premature eviction:

```yaml
# kubelet configuration
evictionSoft:
  memory.available: "500Mi"
evictionSoftGracePeriod:
  memory.available: "1m"
evictionHard:
  memory.available: "100Mi"
```

---

## 7. Security Best Practices

### 7.1 Network Policies

Restrict traffic to Redis pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-allow
spec:
  podSelector:
    matchLabels:
      app: redis
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: my-app
    - namespaceSelector:
        matchLabels:
          name: trusted-namespace
    ports:
    - protocol: TCP
      port: 6379
    - protocol: TCP
      port: 16379  # Cluster bus
```

**Default Deny All Pattern:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

### 7.2 Authentication and ACLs

```yaml
# Redis 6.0+ ACL configuration
apiVersion: v1
kind: Secret
metadata:
  name: redis-auth
type: Opaque
stringData:
  redis.conf: |
    requirepass ${REDIS_PASSWORD}
    aclfile /etc/redis/users.acl
---
# users.acl
user default on >${DEFAULT_PASSWORD} ~* &* +@all
user app on >${APP_PASSWORD} ~app:* &* +@read +@write -@dangerous
user exporter on >${EXPORTER_PASSWORD} ~* &* +ping +info +config|get +cluster|info
```

### 7.3 TLS Configuration

```yaml
# redis.conf for TLS
tls-port 6379
port 0  # Disable non-TLS
tls-cert-file /tls/redis.crt
tls-key-file /tls/redis.key
tls-ca-cert-file /tls/ca.crt
tls-auth-clients optional  # or 'yes' for mTLS
```

**Mount TLS certificates:**
```yaml
volumeMounts:
- name: tls-certs
  mountPath: /tls
  readOnly: true
volumes:
- name: tls-certs
  secret:
    secretName: redis-tls
```

### 7.4 Disable Dangerous Commands

```yaml
# redis.conf
rename-command FLUSHDB ""
rename-command FLUSHALL ""
rename-command CONFIG "CONFIG_9f3b2a1e"
rename-command SHUTDOWN "SHUTDOWN_7c4d8e2f"
```

### 7.5 RBAC for Redis Resources

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: redis-operator
rules:
- apiGroups: ["redis.redis.opstreelabs.in"]
  resources: ["redisclusters", "redissentinels", "redisreplications"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

---

## 8. Monitoring and Observability

### 8.1 Redis Exporter Setup

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis-exporter
  template:
    metadata:
      labels:
        app: redis-exporter
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9121"
    spec:
      containers:
      - name: redis-exporter
        image: oliver006/redis_exporter:latest
        ports:
        - containerPort: 9121
        env:
        - name: REDIS_ADDR
          value: "redis://redis-master:6379"
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-auth
              key: password
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: redis-exporter
spec:
  selector:
    matchLabels:
      app: redis-exporter
  endpoints:
  - port: "9121"
    interval: 15s
```

### 8.2 Key Metrics to Monitor

| Category | Metric | Alert Threshold |
|----------|--------|-----------------|
| **Memory** | `redis_memory_used_bytes` | > 80% of maxmemory |
| | `redis_memory_fragmentation_ratio` | < 1 or > 1.5 |
| **Connections** | `redis_connected_clients` | > 80% of maxclients |
| | `redis_rejected_connections_total` | Any increase |
| **Performance** | `redis_instantaneous_ops_per_sec` | Baseline deviation |
| | `redis_slowlog_length` | > 100 |
| **Replication** | `redis_master_link_up` | != 1 |
| | `redis_master_last_io_seconds_ago` | > 10s |
| **Persistence** | `redis_rdb_last_save_timestamp_seconds` | > 1 hour ago |
| | `redis_aof_rewrite_in_progress` | Stuck for > 5min |

### 8.3 Prometheus Alert Rules

```yaml
groups:
- name: redis-alerts
  rules:
  - alert: RedisMemoryHigh
    expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.8
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Redis memory usage high"
      
  - alert: RedisConnectionsExhausted
    expr: redis_connected_clients / redis_config_maxclients > 0.9
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Redis connections near limit"
      
  - alert: RedisReplicationBroken
    expr: redis_master_link_up{role="slave"} == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Redis replication broken"
```

### 8.4 Custom Monitoring Script

```python
import redis
import time
from prometheus_client import Gauge, start_http_server

# Custom metrics
BIG_KEYS = Gauge('redis_big_keys_count', 'Keys over 1MB')
EXPIRED_RATIO = Gauge('redis_expired_keys_ratio', 'Expired to total ratio')

def monitor_redis(host='localhost', port=6379):
    r = redis.Redis(host=host, port=port, decode_responses=True)
    
    while True:
        info = r.info()
        
        # Find big keys (sample)
        big_keys = []
        cursor = 0
        for _ in range(10):  # Sample 1000 keys
            cursor, keys = r.scan(cursor, count=100)
            for key in keys:
                memory = r.memory_usage(key)
                if memory and memory > 1024 * 1024:  # 1MB
                    big_keys.append(key)
            if cursor == 0:
                break
        
        BIG_KEYS.set(len(big_keys))
        
        # Calculate expired ratio
        total = info.get('total_keys_processed', 1)
        expired = info.get('expired_keys', 0)
        if total > 0:
            EXPIRED_RATIO.set(expired / total)
        
        time.sleep(60)

if __name__ == '__main__':
    start_http_server(9122)
    monitor_redis()
```

---

## 9. Backup and Disaster Recovery

### 9.1 Backup Strategies

| Strategy | RPO | RTO | Implementation |
|----------|-----|-----|----------------|
| **RDB Snapshots** | Minutes-Hours | Minutes | Scheduled BGSAVE |
| **AOF + RDB** | Seconds | Minutes | Continuous AOF + periodic RDB |
| **Cross-site Replication** | Near-zero | Seconds | Multi-cluster replication |
| **Cloud Snapshots** | Hours | Hours | Volume snapshots |

### 9.2 Automated Backup with CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: redis-backup
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: redis:7.2
            command:
            - /bin/sh
            - -c
            - |
              TIMESTAMP=$(date +%Y%m%d-%H%M%S)
              redis-cli --rdb /backup/redis-$TIMESTAMP.rdb
              # Upload to S3
              aws s3 cp /backup/redis-$TIMESTAMP.rdb s3://my-backup-bucket/redis/
              # Cleanup old backups
              find /backup -name "redis-*.rdb" -mtime +7 -delete
            volumeMounts:
            - name: backup
              mountPath: /backup
          volumes:
          - name: backup
            emptyDir: {}
          restartPolicy: OnFailure
```

### 9.3 Point-in-Time Recovery

For AOF-based recovery:
```bash
# 1. Stop Redis
kubectl exec -it redis-0 -- redis-cli SHUTDOWN

# 2. Restore AOF from backup
kubectl cp redis-backup.aof redis-0:/data/appendonly.aof

# 3. Fix AOF (if needed)
kubectl exec -it redis-0 -- redis-check-aof --fix /data/appendonly.aof

# 4. Restart Redis
kubectl rollout restart statefulset/redis
```

### 9.4 Cross-Region DR with Redis Cluster

```
Region A (Primary)          Region B (DR)
┌─────────────┐            ┌─────────────┐
│ Redis Node 1│◄──────────►│ Redis Node 4│
│ (Master)    │  Replicate │ (Replica)   │
├─────────────┤            ├─────────────┤
│ Redis Node 2│◄──────────►│ Redis Node 5│
│ (Master)    │  Replicate │ (Replica)   │
├─────────────┤            ├─────────────┤
│ Redis Node 3│◄──────────►│ Redis Node 6│
│ (Master)    │  Replicate │ (Replica)   │
└─────────────┘            └─────────────┘
```

---

## 10. Performance Tuning

### 10.1 Kernel Tuning

```bash
# Disable Transparent Huge Pages
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag

# TCP tuning
sysctl -w net.core.somaxconn=65535
sysctl -w net.ipv4.tcp_max_syn_backlog=65535
sysctl -w net.ipv4.tcp_tw_reuse=1

# VM tuning
sysctl -w vm.overcommit_memory=1
sysctl -w vm.swappiness=1
```

### 10.2 Redis Configuration for Low Latency

```yaml
# redis.conf
# Disable persistence for pure cache
save ""
appendonly no

# Enable multi-threaded I/O (Redis 6+)
io-threads 4
io-threads-do-reads yes

# TCP settings
tcp-backlog 511
tcp-keepalive 60

# Memory
databases 1  # Reduce if only using db 0

# Slow log - only capture genuinely slow
slowlog-log-slower-than 10000  # 10ms
slowlog-max-len 128
```

### 10.3 CPU Affinity

```yaml
# Pin Redis to specific cores
spec:
  containers:
  - name: redis
    securityContext:
      capabilities:
        add:
        - SYS_NICE  # Required for CPU affinity
    command:
    - /bin/sh
    - -c
    - |
      taskset -c 0-3 redis-server /etc/redis/redis.conf
```

### 10.4 Client-Side Optimization

- **Connection pooling:** Reuse connections, avoid connection per request
- **Pipelining:** Batch commands to reduce RTT
- **Lua scripts:** Reduce round trips for multi-step operations
- **Hash tags:** Ensure related keys are on same shard

---

## 11. Common Pitfalls and Troubleshooting

### 11.1 Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **OOMKilled** | Memory limit too low | Increase memory limit or add eviction |
| **Fork failures** | Not enough memory for copy-on-write | Increase memory or disable RDB |
| **High latency** | THP enabled, slow commands, persistence | Disable THP, check slowlog, tune persistence |
| **Split brain** | Network partition, improper failover | Use fencing, proper Sentinel/Cluster config |
| **Replication lag** | High write rate, network issues | Monitor lag, add bandwidth, check slowlog |

### 11.2 Troubleshooting Commands

```bash
# Check cluster health
redis-cli --cluster check <node-ip>:6379

# Monitor latency
redis-cli --latency -h <host> -p 6379
redis-cli --latency-history -h <host> -p 6379 -i 5

# Check slow queries
redis-cli SLOWLOG GET 10

# Memory analysis
redis-cli --bigkeys
redis-cli --memkeys

# Replication status
redis-cli INFO replication

# Check for blocked clients
redis-cli INFO clients | grep blocked
```

### 11.3 Debugging Connection Issues

```bash
# Test connectivity from within cluster
kubectl run redis-test --image=redis:latest -it --rm -- \
  redis-cli -h redis-headless -p 6379 PING

# Check service endpoints
kubectl get endpoints redis-headless

# Verify DNS resolution
kubectl run -it --rm debug --image=busybox -- nslookup redis-headless

# Check network policies
kubectl get networkpolicies
kubectl describe networkpolicy redis-allow
```

---

## 12. Production Checklist

### 12.1 Pre-Deployment

- [ ] Choose appropriate architecture (Standalone/Sentinel/Cluster)
- [ ] Select operator or manual deployment
- [ ] Size nodes correctly (memory, CPU, storage)
- [ ] Configure storage class with proper IOPS
- [ ] Plan for multi-AZ deployment
- [ ] Design network policies
- [ ] Set up monitoring and alerting

### 12.2 Configuration

- [ ] Set appropriate resource requests/limits
- [ ] Configure pod anti-affinity
- [ ] Set topology spread constraints
- [ ] Configure PDB
- [ ] Enable authentication
- [ ] Configure TLS (production)
- [ ] Set up ACLs for different clients
- [ ] Disable dangerous commands
- [ ] Configure persistence appropriately
- [ ] Set memory policies and limits

### 12.3 Operations

- [ ] Document backup procedures
- [ ] Test restore procedures
- [ ] Set up log aggregation
- [ ] Configure alerts
- [ ] Create runbooks
- [ ] Plan for scaling procedures
- [ ] Document failover procedures

---

## 13. References

1. [Redis Cluster on Kubernetes: Ultimate Guide](https://www.plural.sh/blog/redis-cluster-on-kubernetes/)
2. [Redis Operator Comparison](https://mintlify.com/howl-cloud/redis-operator/concepts/comparison)
3. [Redis Sentinel vs Cluster](https://medium.com/@rhgustmfrh/redis-sentinel-vs-redis-cluster-choosing-the-right-solution-01d6f47ef766)
4. [Kuaishou's Large-Scale Redis on K8s](https://www.cncf.io/blog/2024/12/17/managing-large-scale-redis-clusters-on-kubernetes-with-an-operator-kuaishous-approach/)
5. [Redis Cluster Specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
6. [Redis Performance Tuning](https://redis.io/docs/latest/operate/oss_and_stack/management/optimization/latency/)
7. [Kubernetes Storage Best Practices](https://crawler.woehler.de/web-pulse/kubernetes-storage-best-practices-a-comprehensive-guide-1764799296)
8. [Redis Monitoring with Prometheus](https://oneuptime.com/blog/post/2026-01-21-redis-prometheus-grafana-monitoring/view)

---

*Document compiled from multiple authoritative sources including Redis official documentation, Kubernetes documentation, and production best practices from industry leaders.*
