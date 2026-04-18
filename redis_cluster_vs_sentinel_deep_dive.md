# Redis Cluster vs Redis Sentinel: A Deep Dive Research

## Executive Summary

Redis offers two distinct high-availability solutions that serve different architectural needs: **Redis Sentinel** and **Redis Cluster**. While both provide automatic failover capabilities, they address fundamentally different scaling challenges:

| Aspect | Redis Sentinel | Redis Cluster |
|--------|---------------|---------------|
| **Primary Purpose** | High Availability | Horizontal Scaling + HA |
| **Data Distribution** | Single dataset (replication) | Sharded across 16,384 hash slots |
| **Write Scaling** | Single master only | Multiple masters (shards) |
| **Memory Limit** | Single node capacity | Aggregate of all nodes |
| **Multi-Key Operations** | Full support | Same-slot only |
| **Minimum Nodes** | 3 (1 master + 2 replicas + 3 sentinels) | 6 (3 masters + 3 replicas) |

**Key Rule of Thumb**: Use Sentinel when you need high availability without changing your data model. Use Cluster when you must scale Redis beyond one machine's capacity.

---

## 1. Redis Sentinel Deep Dive

### 1.1 Architecture Overview

Redis Sentinel is a distributed monitoring and failover system designed for Redis master-replica deployments. It was introduced in Redis 2.4 and provides high availability without data sharding.

```
┌─────────────────────────────────────────────────────────────────┐
│                     Redis Sentinel Architecture                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┐         ┌─────────────┐         ┌───────────┐ │
│   │  Sentinel 1 │◄───────►│  Sentinel 2 │◄───────►│ Sentinel 3│ │
│   │  (monitor)  │         │  (monitor)  │         │ (monitor) │ │
│   └──────┬──────┘         └──────┬──────┘         └─────┬─────┘ │
│          │                       │                      │       │
│          │    monitors           │                      │       │
│          ▼                       ▼                      ▼       │
│   ┌─────────────┐         ┌─────────────┐                      │
│   │   Master    │◄────────┤   Replica 1 │                      │
│   │  (Primary)  │         │  (Standby)  │                      │
│   │  Read/Write │         │  Read-only  │                      │
│   └──────┬──────┘         └──────┬──────┘                      │
│          │                       │                               │
│          └───────────────────────┘                               │
│                    Replication                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Core Capabilities

| Capability | Description |
|------------|-------------|
| **Monitoring** | Continuously checks master and replica health via PING commands |
| **Notification** | Alerts administrators via API when issues are detected |
| **Automatic Failover** | Promotes replica to master when master is deemed unavailable |
| **Configuration Provider** | Acts as service discovery for clients to find current master |

### 1.3 Failure Detection Mechanism

Sentinel uses a two-stage failure detection process:

#### Stage 1: Subjectively Down (SDOWN)
- A single Sentinel marks a master as SDOWN when it doesn't receive valid PING replies for `is-master-down-after-milliseconds` (default: 30 seconds)
- Valid replies: `+PONG`, `-LOADING`, `-MASTERDOWN`
- SDOWN alone does NOT trigger failover

#### Stage 2: Objectively Down (ODOWN)
- Reached when at least `quorum` number of Sentinels agree the master is down
- Quorum is configurable per monitored master
- ODOWN triggers the failover authorization process

```
┌──────────────────────────────────────────────────────────────┐
│              Sentinel Failure Detection Flow                  │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│   PING failures ──► SDOWN (single Sentinel)                  │
│        │                                                      │
│        ▼                                                      │
│   Sentinel asks other Sentinels:                              │
│   "Is master X down?"                                         │
│        │                                                      │
│        ▼                                                      │
│   Quorum reached? ──► ODOWN (consensus)                      │
│        │                                                      │
│        ▼                                                      │
│   Leader election ──► Failover authorized                    │
│        │                                                      │
│        ▼                                                      │
│   Replica promoted to master                                  │
└──────────────────────────────────────────────────────────────┘
```

### 1.4 Quorum and Majority Requirements

Understanding quorum is critical for proper Sentinel deployment:

| Setting | Purpose | Example (5 Sentinels) |
|---------|---------|----------------------|
| `quorum` | Number of Sentinels needed to detect ODOWN | `quorum = 2` |
| `majority` | Number of Sentinels needed to authorize failover | `majority = 3` |

**Important**: Sentinel NEVER performs failover in a minority partition to prevent split-brain scenarios.

### 1.5 Sentinel Deployment Patterns

#### Pattern 1: Co-located (Basic Setup)
```
Box 1: Master + Sentinel
Box 2: Replica + Sentinel  
Box 3: Replica + Sentinel
```
**Pros**: Simple, fewer servers needed
**Cons**: Host failure loses both Redis and Sentinel

#### Pattern 2: Dedicated Sentinel Servers (Recommended)
```
Redis Servers:     Master, Replica-1, Replica-2
Sentinel Servers:  Sentinel-1, Sentinel-2, Sentinel-3 (separate boxes)
```
**Pros**: Better fault isolation, independent scaling
**Cons**: Requires more infrastructure

#### Pattern 3: Client-Side Sentinels
```
App Server 1: App + Sentinel
App Server 2: App + Sentinel
App Server 3: App + Sentinel
Redis Servers: Master, Replica (dedicated)
```
**Pros**: No dedicated Sentinel servers needed
**Cons**: Network partitions can cause issues

### 1.6 Configuration Epochs and Consensus

Sentinel uses configuration epochs to version cluster state:

1. Each successful failover gets a unique `configEpoch`
2. Higher epochs always win over lower epochs
3. Configurations propagate via Pub/Sub on `__sentinel__:hello` channel
4. This ensures all Sentinels eventually converge to the same configuration

```
Initial: Master at 192.168.1.50:6379 (epoch 1)
            │
            ▼
Failover: Replica promoted (epoch 2)
            │
            ▼
Propagation: All Sentinels update to new master 192.168.1.51:6379
```

---

## 2. Redis Cluster Deep Dive

### 2.1 Architecture Overview

Redis Cluster is a distributed implementation of Redis that provides automatic sharding and high availability. Introduced in Redis 3.0, it partitions data across multiple nodes using hash slots.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Redis Cluster Architecture                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│   ┌─────────────────────────────────────────────────────────────────┐   │
│   │                    16,384 Hash Slots (0-16383)                   │   │
│   └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                     │
│        ┌───────────────────────────┼───────────────────────────┐         │
│        ▼                           ▼                           ▼         │
│   ┌─────────┐                ┌─────────┐                ┌─────────┐     │
│   │ Master A │◄──────────────►│ Master B │◄──────────────►│ Master C │     │
│   │Slots    │                │Slots    │                │Slots    │     │
│   │0-5460   │                │5461-10922│               │10923-16383│    │
│   └────┬────┘                └────┬────┘                └────┬────┘     │
│        │                          │                          │          │
│        │ Replicates               │ Replicates               │ Replicates│
│        ▼                          ▼                          ▼          │
│   ┌─────────┐                ┌─────────┐                ┌─────────┐     │
│   │Replica A1│               │Replica B1│               │Replica C1│     │
│   │(Standby) │               │(Standby) │               │(Standby) │     │
│   └─────────┘                └─────────┘                └─────────┘     │
│                                                                          │
│   Gossip Protocol: All nodes communicate to maintain cluster state      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Hash Slot Model

Redis Cluster uses 16,384 hash slots to partition data:

```
HASH_SLOT = CRC16(key) mod 16384
```

**Why 16,384 slots?**
- Provides even distribution across nodes
- Small enough for efficient metadata storage
- Large enough to support up to ~1,000 nodes (recommended max)
- 14 bits of CRC16 output used (2^14 = 16,384)

**Example Key Distribution:**
| Key | CRC16 | Hash Slot | Node |
|-----|-------|-----------|------|
| `user:1001` | 20,234,847 | 12,479 | Master C |
| `session:abc` | 5,432,109 | 5,021 | Master A |
| `order:5678` | 12,876,543 | 8,991 | Master B |

### 2.3 Hash Tags for Data Locality

Hash tags force related keys to the same slot:

```
Key Pattern:        {user:123}.profile  {user:123}.settings  {user:123}.orders
Hash Input:         user:123            user:123             user:123
Result:             Same slot (e.g., 5474) for all keys
```

**Use Cases for Hash Tags:**
- Multi-key transactions (MSET, MGET)
- Lua scripts requiring multiple keys
- Related data that must be accessed atomically

**Warning**: Poorly chosen hash tags can create hot spots:
```
# BAD: All user sessions on one node
{session}:user1, {session}:user2, ... {session}:user1000000

# GOOD: Distribute by user ID
session:{user1}, session:{user2}, ... session:{user1000000}
```

### 2.4 Client Redirection Mechanism

Cluster-aware clients maintain a slot-to-node mapping:

```
┌─────────────────────────────────────────────────────────────┐
│                  Client Request Flow                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Client sends: GET user:1234                              │
│                                                              │
│  2. Client calculates: CRC16("user:1234") % 16384 = 8921    │
│                                                              │
│  3. Client checks slot map: Slot 8921 → Master B            │
│                                                              │
│  4. Request sent directly to Master B                        │
│                                                              │
│  ─────────────────────────────────────────────────────────  │
│                                                              │
│  If slot moved (resharding):                                 │
│  • Server returns -MOVED 8921 192.168.1.12:6379             │
│  • Client updates slot map                                   │
│  • Request retried to new node                               │
│                                                              │
│  If migration in progress:                                   │
│  • Server returns -ASK redirect                              │
│  • Client sends ASKING command, then request                 │
└─────────────────────────────────────────────────────────────┘
```

### 2.5 Resharding and Slot Migration

Resharding moves hash slots between nodes without downtime:

#### Migration States
1. **Normal**: Node owns slot exclusively
2. **MIGRATING**: Source node moving slot to target
3. **IMPORTING**: Target node receiving slot from source

#### Migration Process
```
Step 1: Set migration state
  Source: CLUSTER SETSLOT <slot> MIGRATING <target-id>
  Target: CLUSTER SETSLOT <slot> IMPORTING <source-id>

Step 2: Migrate existing keys
  For each key in slot:
    MIGRATE <target> <port> "" 0 <timeout> KEYS <key>

Step 3: Complete migration
  Both: CLUSTER SETSLOT <slot> NODE <target-id>
```

#### Automated Resharding
```bash
# Interactive resharding
redis-cli --cluster reshard 127.0.0.1:7000

# Non-interactive resharding
redis-cli --cluster reshard 127.0.0.1:7000 \
  --cluster-from <source-node-id> \
  --cluster-to <target-node-id> \
  --cluster-slots 1000 \
  --cluster-yes

# Automatic rebalancing
redis-cli --cluster rebalance 127.0.0.1:7000
```

### 2.6 Failover in Redis Cluster

When a master fails, its replica is promoted:

```
┌─────────────────────────────────────────────────────────────┐
│              Cluster Failover Process                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. Replica detects master is unreachable                    │
│                                                              │
│  2. Replica requests failover authorization from masters    │
│     (majority of masters must agree)                        │
│                                                              │
│  3. If authorized, replica increments its configEpoch       │
│                                                              │
│  4. Replica promotes itself to master                       │
│                                                              │
│  5. New configuration propagated via gossip                 │
│                                                              │
│  6. Other replicas reconfigure to follow new master         │
│                                                              │
│  Result: Slot ownership transferred with new epoch          │
└─────────────────────────────────────────────────────────────┘
```

### 2.7 Replica Migration

Cluster automatically moves replicas to cover orphaned masters:

```
Before:
  Master A ← Replica A1, Replica A2
  Master B ← (no replica)

After Replica Migration:
  Master A ← Replica A1
  Master B ← Replica A2 (migrated from A)
```

---

## 3. Detailed Comparison

### 3.1 Feature Comparison Matrix

| Feature | Redis Sentinel | Redis Cluster |
|---------|---------------|---------------|
| **High Availability** | ✅ Automatic failover | ✅ Automatic failover |
| **Horizontal Scaling** | ❌ Single master | ✅ Multiple masters |
| **Write Scaling** | ❌ Limited to single core | ✅ Distributed writes |
| **Memory Scaling** | ❌ Single node limit | ✅ Aggregate memory |
| **Read Scaling** | ✅ Read replicas | ✅ Read replicas |
| **Multi-Key Operations** | ✅ Full support | ⚠️ Same-slot only |
| **Transactions (MULTI/EXEC)** | ✅ Full support | ⚠️ Same-slot only |
| **Lua Scripts** | ✅ Full support | ⚠️ Same-slot keys only |
| **Pub/Sub** | ✅ Full support | ✅ Full support (cluster-wide) |
| **Multiple Databases** | ✅ 0-15 supported | ❌ Database 0 only |
| **SELECT Command** | ✅ Supported | ❌ Not allowed |
| **Key Scanning** | ✅ Full DB scan | ⚠️ Per-node scan |
| **Geospatial Commands** | ✅ Full support | ✅ Full support |
| **Stream Commands** | ✅ Full support | ⚠️ Stream must be on single node |

### 3.2 Multi-Key Operations Limitations

This is often the biggest surprise when migrating to Redis Cluster:

#### Commands Affected by CROSSSLOT Error
| Command | Sentinel | Cluster |
|---------|----------|---------|
| `MGET key1 key2 key3` | ✅ Works | ❌ CROSSSLOT if different slots |
| `MSET key1 val1 key2 val2` | ✅ Works | ❌ CROSSSLOT if different slots |
| `DEL key1 key2 key3` | ✅ Works | ❌ CROSSSLOT if different slots |
| `RENAME key1 key2` | ✅ Works | ❌ CROSSSLOT if different slots |
| `SINTER set1 set2` | ✅ Works | ❌ CROSSSLOT if different slots |
| `SUNION set1 set2` | ✅ Works | ❌ CROSSSLOT if different slots |
| `ZINTER zset1 zset2` | ✅ Works | ❌ CROSSSLOT if different slots |
| `PFMERGE hll1 hll2` | ✅ Works | ❌ CROSSSLOT if different slots |
| `WATCH key1 key2` | ✅ Works | ❌ CROSSSLOT if different slots |
| `EVAL` with multiple keys | ✅ Works | ❌ CROSSSLOT if different slots |

#### Solutions for Cluster Multi-Key Operations

**1. Hash Tags (Recommended)**
```
# Before: Different slots
MSET user:1:name "Alice" user:1:email "alice@example.com"
# ERROR: CROSSSLOT

# After: Same slot using hash tag
MSET {user:1}:name "Alice" {user:1}:email "alice@example.com"
# SUCCESS
```

**2. Pipeline Individual Commands**
```python
# Instead of MGET (which may fail)
pipeline = redis.pipeline()
for key in keys:
    pipeline.get(key)
results = pipeline.execute()
```

**3. Redesign Data Model**
```
# Before: Separate keys
SET user:1001:name "Alice"
SET user:1001:email "alice@example.com"

# After: Hash with all user data
HSET user:1001 name "Alice" email "alice@example.com"
# Now HMGET works on single key
```

### 3.3 Performance Characteristics

| Metric | Redis Sentinel | Redis Cluster |
|--------|---------------|---------------|
| **Write Throughput** | ~100K-1M ops/sec (single core) | N × single node (scales linearly) |
| **Read Throughput** | ~100K-1M ops/sec per replica | N × single node (scales linearly) |
| **Latency (Normal)** | Sub-millisecond | Sub-millisecond (direct routing) |
| **Latency (MOVED redirect)** | N/A | +1 RTT (temporary) |
| **Memory Capacity** | Single node RAM | Sum of all nodes RAM |
| **Network Overhead** | Replication traffic | Replication + Gossip traffic |

### 3.4 Operational Complexity

| Aspect | Redis Sentinel | Redis Cluster |
|--------|---------------|---------------|
| **Initial Setup** | Moderate | Complex |
| **Client Changes** | Minimal (Sentinel-aware client) | Significant (Cluster-aware client) |
| **Monitoring** | Standard Redis monitoring | Cluster-aware monitoring required |
| **Scaling** | Vertical only (bigger server) | Horizontal (add/remove nodes) |
| **Resharding** | N/A | Required when adding/removing nodes |
| **Backup Strategy** | Single node backup | Multi-node backup coordination |
| **Troubleshooting** | Simpler (single dataset) | Complex (distributed state) |

---

## 4. Decision Framework

### 4.1 When to Choose Redis Sentinel

**Ideal Use Cases:**
- ✅ Dataset fits on a single server (< 100GB typically)
- ✅ Write throughput within single-core limits
- ✅ High availability is required
- ✅ Heavy use of multi-key operations
- ✅ Frequent transactions or Lua scripts
- ✅ Legacy application with minimal changes allowed
- ✅ Strong consistency requirements

**Real-World Scenarios:**
- Session stores with moderate data size
- Caching layers where all data fits in memory
- Real-time leaderboards (small to medium scale)
- Rate limiting systems
- Application configuration stores

### 4.2 When to Choose Redis Cluster

**Ideal Use Cases:**
- ✅ Dataset exceeds single server memory
- ✅ Write throughput exceeds single-core capacity
- ✅ Horizontal scaling is required
- ✅ Read-heavy workloads (8:1 read/write ratio)
- ✅ Acceptable minor data loss during failover
- ✅ Application can be designed for sharding

**Real-World Scenarios:**
- Large-scale caching (100GB+ datasets)
- High-throughput write workloads
- Multi-tenant applications with sharded data
- Geographic distribution requirements
- Big data ingestion pipelines

### 4.3 Decision Flowchart

```
┌─────────────────────────────────────────────────────────────────┐
│                    Redis Deployment Decision                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │ Does your data   │
                    │ fit on one node? │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                              │
              ▼ YES                          ▼ NO
    ┌─────────────────┐            ┌─────────────────┐
    │ Do you need     │            │ Use Redis       │
    │ high availability?│           │ Cluster         │
    └────────┬────────┘            └─────────────────┘
             │
    ┌────────┴────────┐
    │                 │
    ▼ YES             ▼ NO
┌─────────┐     ┌─────────┐
│ Use     │     │ Single  │
│ Sentinel│     │ Redis   │
│ +       │     │ Instance│
│ Replica │     │         │
└─────────┘     └─────────┘
```

### 4.4 Data Size Guidelines

| Data Size | Recommendation | Rationale |
|-----------|---------------|-----------|
| < 10GB | Single Redis or Sentinel | Simplest solution |
| 10-50GB | Sentinel with large instance | HA without complexity |
| 50-100GB | Sentinel or Small Cluster | Depends on write throughput |
| 100GB-1TB | Redis Cluster | Horizontal scaling needed |
| > 1TB | Redis Cluster + Proxy | Consider managed solutions |

---

## 5. Production Deployment Best Practices

### 5.1 Redis Sentinel Best Practices

#### Minimum Production Setup
```
Infrastructure:
- 1 Master (dedicated server)
- 2 Replicas (dedicated servers)
- 3 Sentinels (dedicated servers or co-located)

Network:
- All nodes in same data center (low latency)
- Separate network segments for Redis and Sentinel
```

#### Key Configuration Parameters
```conf
# sentinel.conf
sentinel monitor mymaster 192.168.1.10 6379 2
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1

# Redis replication settings
min-replicas-to-write 1
min-replicas-max-lag 10
```

#### Sentinel Deployment Checklist
- [ ] Deploy at least 3 Sentinels (never 2)
- [ ] Place Sentinels on different physical boxes
- [ ] Set appropriate quorum (majority of Sentinels)
- [ ] Configure `down-after-milliseconds` based on network latency
- [ ] Enable `min-replicas-to-write` to prevent split-brain writes
- [ ] Monitor Sentinel logs for failover events
- [ ] Test failover procedures regularly

### 5.2 Redis Cluster Best Practices

#### Minimum Production Setup
```
Infrastructure:
- 3 Master nodes (dedicated servers)
- 3 Replica nodes (dedicated servers)
- Total: 6 Redis instances across 6 servers

Slot Distribution:
- Master A: Slots 0-5460 (5,461 slots)
- Master B: Slots 5461-10922 (5,462 slots)
- Master C: Slots 10923-16383 (5,461 slots)
```

#### Key Configuration Parameters
```conf
# redis.conf for cluster nodes
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-require-full-coverage no
cluster-replica-validity-factor 10

# Memory and persistence
maxmemory 32gb
maxmemory-policy allkeys-lru
appendonly yes
appendfsync everysec
```

#### Cluster Deployment Checklist
- [ ] Deploy minimum 3 masters + 3 replicas
- [ ] Distribute masters across availability zones
- [ ] Ensure replicas are on different AZs than their masters
- [ ] Configure `cluster-require-full-coverage` based on tolerance
- [ ] Set appropriate `cluster-node-timeout`
- [ ] Design key schema with hash tags for related data
- [ ] Test resharding procedures before production
- [ ] Monitor slot distribution and hot spots

### 5.3 Common Pitfalls and Solutions

#### Pitfall 1: Split-Brain in Sentinel
**Symptom**: Two nodes claim to be master after network partition
**Solution**: 
- Always use odd number of Sentinels (3, 5, 7)
- Configure `min-replicas-to-write` to prevent writes during partition
- Ensure Sentinels can communicate with majority

#### Pitfall 2: CROSSSLOT Errors After Migration
**Symptom**: Application fails with CROSSSLOT errors after moving to Cluster
**Solution**:
- Audit all multi-key operations before migration
- Implement hash tags for related keys
- Redesign data model to use fewer multi-key operations

#### Pitfall 3: Hot Slots in Cluster
**Symptom**: Uneven load distribution, one node overloaded
**Solution**:
- Review hash tag design
- Distribute high-cardinality tags
- Consider resharding to balance load

#### Pitfall 4: Long Resharding Times
**Symptom**: Resharding takes hours or days
**Solution**:
- Schedule resharding during low-traffic periods
- Monitor `migrate` command performance
- Consider using `redis-cli --cluster rebalance` with thresholds

---

## 6. Client Library Support

### 6.1 Sentinel-Aware Clients

| Language | Library | Sentinel Support |
|----------|---------|------------------|
| Python | redis-py | ✅ `Sentinel` class |
| Java | Jedis | ✅ `JedisSentinelPool` |
| Java | Lettuce | ✅ Built-in support |
| Node.js | ioredis | ✅ Built-in support |
| Go | go-redis | ✅ `FailoverClient` |
| Ruby | redis-rb | ✅ Built-in support |

#### Python Example (redis-py)
```python
from redis.sentinel import Sentinel

sentinel = Sentinel([
    ('sentinel-1', 26379),
    ('sentinel-2', 26379),
    ('sentinel-3', 26379)
])

# Get master connection
master = sentinel.master_for('mymaster', socket_timeout=0.5)
master.set('key', 'value')

# Get replica connection (read-only)
replica = sentinel.slave_for('mymaster', socket_timeout=0.5)
value = replica.get('key')
```

#### Java Example (Jedis)
```java
import redis.clients.jedis.JedisSentinelPool;
import redis.clients.jedis.Jedis;
import java.util.Set;
import java.util.HashSet;

Set<String> sentinels = new HashSet<>();
sentinels.add("sentinel-1:26379");
sentinels.add("sentinel-2:26379");
sentinels.add("sentinel-3:26379");

JedisSentinelPool pool = new JedisSentinelPool(
    "mymaster",
    sentinels,
    "password"
);

try (Jedis jedis = pool.getResource()) {
    jedis.set("key", "value");
}
```

### 6.2 Cluster-Aware Clients

| Language | Library | Cluster Support |
|----------|---------|-----------------|
| Python | redis-py | ✅ `RedisCluster` class |
| Java | Jedis | ✅ `JedisCluster` class |
| Java | Lettuce | ✅ Built-in support |
| Node.js | ioredis | ✅ Built-in support |
| Go | go-redis | ✅ `ClusterClient` |
| Ruby | redis-rb | ✅ Built-in support |

#### Python Example (redis-py)
```python
from redis.cluster import RedisCluster

rc = RedisCluster(
    host='localhost',
    port=7000,
    decode_responses=True
)

# Automatic slot routing
rc.set('key', 'value')  # Routed to correct node
value = rc.get('key')

# Check slot for key
slot = rc.cluster_keyslot('key')
print(f"Key 'key' is in slot {slot}")
```

#### Java Example (Jedis)
```java
import redis.clients.jedis.JedisCluster;
import redis.clients.jedis.HostAndPort;
import java.util.Set;
import java.util.HashSet;

Set<HostAndPort> nodes = new HashSet<>();
nodes.add(new HostAndPort("node-1", 6379));
nodes.add(new HostAndPort("node-2", 6379));
nodes.add(new HostAndPort("node-3", 6379));

JedisCluster cluster = new JedisCluster(nodes);
cluster.set("key", "value");
String value = cluster.get("key");
```

---

## 7. Cost Analysis

### 7.1 Infrastructure Costs

| Deployment Type | Nodes | Typical AWS Cost (monthly) |
|-----------------|-------|---------------------------|
| Single Redis | 1 | $100-500 |
| Sentinel (3 nodes) | 3 | $300-1,500 |
| Cluster (6 nodes) | 6 | $600-3,000 |
| Cluster (9 nodes) | 9 | $900-4,500 |

### 7.2 Operational Costs

| Aspect | Sentinel | Cluster |
|--------|----------|---------|
| **Setup Time** | Hours | Days |
| **Monitoring Complexity** | Low | High |
| **Scaling Effort** | High (vertical) | Medium (horizontal) |
| **Troubleshooting Skill** | Basic | Advanced |
| **Downtime Risk** | Low | Low-Medium |

### 7.3 Hidden Costs

**Redis Cluster:**
- Client library upgrades may be required
- Application refactoring for multi-key operations
- Additional monitoring tools (cluster-aware)
- Training for operations team

**Redis Sentinel:**
- Eventually requires migration to Cluster if data grows
- Vertical scaling has hardware limits
- May need larger instances (more expensive per GB)

---

## 8. Migration Strategies

### 8.1 From Single Redis to Sentinel

```
Phase 1: Set up replicas
  - Configure replica nodes
  - Start replication
  - Verify sync

Phase 2: Deploy Sentinels
  - Configure Sentinel instances
  - Start monitoring
  - Verify quorum

Phase 3: Update clients
  - Switch to Sentinel-aware client
  - Configure with all Sentinel addresses
  - Test failover

Phase 4: Decommission old setup
  - Remove direct connections
  - Monitor for issues
```

### 8.2 From Sentinel to Cluster

```
Phase 1: Application audit
  - Identify all multi-key operations
  - Plan hash tag usage
  - Refactor data model if needed

Phase 2: Deploy Cluster
  - Set up cluster nodes
  - Configure slots
  - Test with sample data

Phase 3: Data migration
  - Use redis-cli --cluster import
  - Or dual-write during transition
  - Verify data consistency

Phase 4: Client migration
  - Switch to cluster-aware client
  - Update connection configuration
  - Gradually shift traffic

Phase 5: Decommission Sentinel
  - Remove old infrastructure
  - Update documentation
```

---

## 9. Conclusion

### Key Takeaways

1. **Different Problems, Different Solutions**
   - Sentinel solves high availability
   - Cluster solves horizontal scaling + HA

2. **No Universal Winner**
   - Many applications work perfectly with Sentinel
   - Cluster adds complexity that may not be needed

3. **Plan for Growth**
   - Design key schemas with potential clustering in mind
   - Use hash tags proactively for related data

4. **Test Failover**
   - Both solutions require regular failover testing
   - Document and practice recovery procedures

5. **Monitor Everything**
   - Cluster requires more sophisticated monitoring
   - Track slot distribution, hot spots, and migration progress

### Final Recommendations

| Scenario | Recommendation |
|----------|---------------|
| Small dataset, need HA | **Redis Sentinel** |
| Large dataset, high write throughput | **Redis Cluster** |
| Uncertain about future growth | Start with **Sentinel**, design for **Cluster** |
| Heavy multi-key operations | **Sentinel** (unless you can redesign) |
| Multi-tenant SaaS | **Redis Cluster** with tenant sharding |
| Cache layer only | Either works; **Sentinel** is simpler |
| Primary data store | **Sentinel** for stronger consistency |

---

## References

1. [Redis Sentinel Documentation](https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/)
2. [Redis Cluster Specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
3. [Redis High Availability Tutorial](https://redis.io/tutorials/operate/redis-at-scale/high-availability/)
4. [Redis Scalability Tutorial](https://redis.io/tutorials/operate/redis-at-scale/scalability/)
5. [Multi-key Operations in Redis](https://redis.io/docs/latest/develop/using-commands/multi-key-operations/)
6. [Redis Cluster Scaling](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/)

---

*Document Version: 1.0*  
*Last Updated: April 2025*
