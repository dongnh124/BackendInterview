# Replication Types: Architecture & Trade-offs

Master the fundamental replication architectures used in modern databases and understand when to use each.

---

## 🎯 Overview

Replication types define how data flows from the source to replicas. Each architecture has different consistency guarantees, operational complexity, and failover capabilities.

**Quick Comparison:**

| Type              | Write Pattern           | Failover        | Complexity | Data Loss Risk  | Best For                           |
| ----------------- | ----------------------- | --------------- | ---------- | --------------- | ---------------------------------- |
| **Single-Leader** | Primary only            | Promote replica | Low        | Low (with sync) | Most apps (default)                |
| **Multi-Leader**  | Any leader              | Automatic       | High       | Medium          | Multi-region, active-active        |
| **Leaderless**    | Any node (quorum)       | Automatic       | Very High  | Low (tunable)   | Highly distributed systems         |
| **Cascading**     | Primary→Replica→Replica | Manual          | Medium     | Low             | Large replicas, bandwidth concerns |

---

## 1️⃣ Single-Leader (Master-Slave) Replication

### Architecture

```
         [PRIMARY/MASTER]
              ↓ (write-ahead log stream)
         [Replica 1]
         [Replica 2]
         [Replica 3]
```

**How it works:**

1. All writes go to the primary
2. Primary writes to local WAL (Write-Ahead Log)
3. WAL entries streamed to replicas
4. Replicas apply changes asynchronously or synchronously
5. Reads can go to primary or replicas

### Advantages

✅ **Simple & Intuitive** — Clear primary/replica roles
✅ **Native support** — All databases support it natively
✅ **Predictable** — Deterministic replication (replay same log)
✅ **Low latency** — Async replication has minimal impact
✅ **Flexible failover** — Easy to promote any replica
✅ **Read scaling** — Distribute reads across replicas

### Disadvantages

❌ **Single write bottleneck** — All writes must go through primary
❌ **Stale reads** — Replicas lag behind (potential consistency issues)
❌ **Manual failover required** — Without external orchestration
❌ **Replica resources** — Must handle full dataset size
❌ **Cascading failures** — If primary fails, no writes until failover

### Database Support

- **PostgreSQL**: Streaming replication (WAL level)
- **MySQL**: Binary log replication
- **MongoDB**: Replica sets (primary-secondary)
- **SQL Server**: Log shipping
- **Oracle**: Data Guard

### Configuration Examples

#### PostgreSQL

```sql
-- On primary
synchronous_commit = remote_apply    -- Wait for replica
max_wal_senders = 3
synchronous_standby_names = 'replica1,replica2'

-- On replica
primary_conninfo = 'host=primary dbname=postgres'
hot_standby = on  -- Allow reads on replica
```

#### MySQL

```sql
-- On primary
server-id = 1
log_bin = /var/log/mysql/mysql-bin.log
max_binlog_size = 1G
binlog-format = ROW

-- On replica
server-id = 2
relay-log = /var/log/mysql/mysql-relay-bin
relay-log-index = /var/log/mysql/mysql-relay-bin.index
read-only = ON
```

### When to Use

✅ **Default choice** for most applications
✅ **OLTP systems** with read scaling needs
✅ **Bounded write throughput** (single primary acceptable)
✅ **Geographic colocation** (all nodes in same region)
✅ **Teams familiar** with traditional HA

### Failover Process

```
1. Primary goes down
2. Monitor detects failure (3-5 seconds typical)
3. Election: Choose healthiest replica
4. Promotion: Replica becomes new primary
5. Demotion: Old primary rejoins as replica
6. DNS/Connection update
7. Applications reconnect

Typical RTO: 30 seconds - 2 minutes
RPO: 0 (sync) or seconds (async)
```

---

## 2️⃣ Multi-Leader (Active-Active) Replication

### Architecture

```
[Leader A] ←→ [Leader B] ←→ [Leader C]
   ↓ ↓          ↓ ↓          ↓ ↓
Replicas     Replicas      Replicas
```

**How it works:**

1. Every leader accepts writes (independently)
2. Writes replicated to all other leaders
3. Leaders act as each other's replicas
4. Leaderless from application perspective
5. Conflict resolution handles concurrent writes

### Advantages

✅ **No write bottleneck** — Write to any datacenter
✅ **Local latency** — Users write to nearest leader
✅ **High availability** — Any leader failure tolerated
✅ **Fault tolerance** — No single point of failure
✅ **Geo-distributed** — Perfect for multi-region

### Disadvantages

❌ **Conflict complexity** — Concurrent writes to same record
❌ **Harder to operate** — More moving parts
❌ **Higher latency** — Replication between leaders
❌ **Consistency challenges** — May need eventual consistency
❌ **Data divergence** — Temporary inconsistency between leaders
❌ **Limited DB support** — Not built-in for traditional RDBMS

### Conflict Scenarios

```
Time    Datacenter A              Datacenter B
1:00    UPDATE user SET age=30    UPDATE user SET age=31
        WHERE id=1                WHERE id=1

Replication delay: 100ms

Both leaders apply locally:
- A: age = 30
- B: age = 31

After replication:
- A receives: age = 31
- B receives: age = 30

CONFLICT! Which is correct?
```

### Resolution Strategies

1. **Last Write Wins (LWW)**

   ```
   Use timestamp on each write
   Higher timestamp wins
   Risk: Arbitrary choice, may lose data
   ```

2. **Multi-Version Concurrency Control (MVCC)**

   ```
   Keep both versions
   Application chooses during read
   Risk: Complex merge logic
   ```

3. **Custom Logic**

   ```
   Define business rules per field
   age: take maximum (older writes ignored safely)
   name: take from primary region
   status: merge flags bitwise
   ```

4. **Abort on Conflict**
   ```
   Detect conflicts, notify application
   Application must retry
   Risk: Poor UX
   ```

### Database Support

- **MongoDB**: Replica sets can be configured multi-leader
- **PostgreSQL**: Logical replication + custom logic (Bucardo, pglogical)
- **MySQL**: Group Replication (InnoDB Cluster)
- **CouchDB**: Native multi-master
- **Cassandra**: Fully distributed (all nodes write)

### Configuration Example (MongoDB)

```javascript
// Enable sharding + replication
use admin
db.adminCommand({
  configureReplicaSet: true,
  members: [
    { _id: 0, host: "leader-a:27017" },
    { _id: 1, host: "leader-b:27017" },
    { _id: 2, host: "leader-c:27017" }
  ]
})

// Multi-region setup
db.replSetStatus()  // Check replication lag
```

### When to Use

✅ **Multi-datacenter** deployments (primary + standby in different regions)
✅ **Global applications** (local writes in each region)
✅ **High availability** critical (no single failure tolerated)
✅ **Write throughput scaling** (distribute write load)
❌ **Strong consistency** critical (conflict-free guarantees)
❌ **Team new to distributed systems**

### Operational Complexity

```
Simple (1 leader):  Single failure point
Medium (2 leaders): Split-brain possible
Hard (3+ leaders):  Conflict resolution at scale
```

---

## 3️⃣ Leaderless (Quorum-Based) Replication

### Architecture

```
Client sends write to 3 nodes (W out of N)
Client sends read to 3 nodes (R out of N)

Consistency guaranteed if: W + R > N

Example: N=5, W=3, R=3
Write to 3: Guaranteed read will hit ≥2 written
```

**How it works:**

1. Client writes to W replicas (doesn't wait for all)
2. As long as W succeed, write is committed
3. Client reads from R replicas (takes quorum vote)
4. Most recent version wins (timestamp + version vector)
5. Replicas gossip changes (anti-entropy)

### Advantages

✅ **Highly available** — No single leader failure
✅ **No failover needed** — Automatic recovery
✅ **Tunable consistency** — Trade off via W and R
✅ **Scalable writes** — All nodes accept writes
✅ **No split-brain** — Quorum prevents conflicts
✅ **Symmetric nodes** — All nodes identical role

### Disadvantages

❌ **Complex semantics** — Quorum math non-intuitive
❌ **Read repair needed** — Anti-entropy background task
❌ **Higher read latency** — Must wait for quorum
❌ **Network partition issues** — Can't commit if minority
❌ **Debugging hard** — Eventual consistency issues subtle
❌ **Limited DB support** — Specialized systems only

### Quorum Math

```
N = total nodes
W = write quorum size
R = read quorum size

Strong Consistency: W + R > N
  - Example: N=5, W=3, R=3 ✓
  - Any write hits ≥3, any read hits ≥3
  - Overlap guaranteed

Weak Consistency: W + R ≤ N
  - Example: N=5, W=2, R=2 ✗
  - Write might miss nodes read from
  - Stale reads possible

Performance vs Consistency trade-off
- W=1, R=1: Fast but inconsistent
- W=N, R=1: Slow writes, fast consistent reads
- W=(N+1)/2, R=(N+1)/2: Balanced
```

### Database Support

- **DynamoDB**: Quorum-based (configurable W/R)
- **Cassandra**: Configurable consistency levels
- **Riak**: Vector clocks + quorum
- **Voldemort**: Amazon's Dynamo implementation
- **Memcached**: Not built-in (replication layer)

### Configuration Example (DynamoDB)

```python
# DynamoDB uses quorum internally
# You control read/write capacity units

import boto3
dynamodb = boto3.resource('dynamodb')

table = dynamodb.create_table(
    TableName='users',
    KeySchema=[
        {'AttributeName': 'user_id', 'KeyType': 'HASH'}
    ],
    AttributeDefinitions=[
        {'AttributeName': 'user_id', 'AttributeType': 'S'}
    ],
    BillingMode='PAY_PER_REQUEST'  # On-demand = automatic quorum sizing
)

# Read with eventual consistency (faster, cheaper)
response = table.get_item(
    Key={'user_id': 'user123'},
    ConsistentRead=False  # R=1
)

# Read with strong consistency (slower, more RCU)
response = table.get_item(
    Key={'user_id': 'user123'},
    ConsistentRead=True   # R=majority
)
```

### When to Use

✅ **Highly available systems** (tolerating failures)
✅ **Global distribution** (replicas across regions)
✅ **Write scalability** critical
✅ **Eventual consistency** acceptable (social media, caching)
❌ **Financial systems** (strong consistency required)
❌ **Audit logs** (immutable, ordered)
❌ **Banking/compliance** (legal consistency requirements)

---

## 4️⃣ Cascading (Multi-Level) Replication

### Architecture

```
[PRIMARY]
    ↓ (replicate)
[REPLICA 1]
    ↓ (replicate)
[REPLICA 2]
    ↓ (replicate)
[REPLICA 3]
```

**How it works:**

1. Primary replicates to replica 1
2. Replica 1 replicates to replica 2
3. Replica 2 replicates to replica 3
4. Reduces bandwidth on primary
5. Increases replication lag at each hop

### Advantages

✅ **Reduced primary load** — Primary only talks to R1
✅ **Bandwidth efficient** — Large fan-out over WAN
✅ **Scalable replicas** — More replicas than primary can handle
✅ **Hierarchical organization** — By region/purpose
✅ **Parallel recovery** — Replicas catch up in parallel

### Disadvantages

❌ **Increased lag** — Cumulative delay at each level
❌ **More failure points** — If R1 fails, R2/R3 don't catch up
❌ **Complex monitoring** — Track lag at each level
❌ **Harder to failover** — Must promote correct node
❌ **Debugging** — Issues multiply at each level

### When to Use

✅ **Many replicas** (10+) in different regions
✅ **Bandwidth constrained** (WAN replication)
✅ **Hierarchical deployment** (geographic structure)
❌ **Low replication lag** required
❌ **Automated failover** needed
❌ **High availability** critical

---

## 🔄 Comparison & Decision Tree

### By Use Case

**New project, standard OLTP?**
→ **Single-Leader** (Simple, proven, scalable reads)

**Multi-datacenter, global write traffic?**
→ **Multi-Leader** (Accept complexity for local writes)

**Extreme scale, eventual consistency OK?**
→ **Leaderless** (Netflix/Facebook scale)

**Large fan-out of replicas?**
→ **Cascading** (Keep primary load down)

**Hybrid: fast local reads + global consistency?**
→ **Single-Leader + cascading** (Primary in main region, cascade replicas)

### By Scale

```
0-10 nodes:           Single-Leader (simple, proven)
10-100 nodes:         Multi-Leader or Leaderless
100+ nodes:           Leaderless (designed for it)
Multi-region (few):   Multi-Leader
Multi-region (many):  Leaderless
```

---

## 📋 Operational Checklist

- [ ] Understand data consistency model for your architecture
- [ ] Know replication lag in your system
- [ ] Test failover procedure regularly
- [ ] Monitor replication health
- [ ] Have runbooks for lag > threshold
- [ ] Document write path (where writes go)
- [ ] Verify no applications writing to replicas
- [ ] Test backup + restore independent of replication
- [ ] Know how to stop replication in emergency
- [ ] Understand conflict resolution strategy

---

## 🔗 Related Topics

- [Sync vs Async Replication](sync-vs-async.md)
- [Failover Strategies](failover-strategies.md)
- [Replication Lag Monitoring](replication-lag-monitoring.md)
- [HA Architecture Design](README.md)

---

**Key Insight:** Replication type determines your operational model. Single-leader is default; multi-leader and leaderless are specialized solutions for specific scale/geography problems.
