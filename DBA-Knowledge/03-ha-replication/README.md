# High Availability & Replication

Mastering failover, scaling, and redundancy for production systems.

## 📚 Core Topics

1. **Replication Types** — Single-leader, multi-leader, leaderless
2. **Sync vs Async** — Consistency vs performance trade-offs
3. **Failover** — Automatic vs manual, preventing split-brain
4. **Read Replicas** — Scaling reads and reporting
5. **Replication Lag** — Monitoring and impact
6. **HA Architecture** — Multi-AZ, multi-region
7. **Connection Routing** — DNS, VIP, proxy strategies

## 🏗️ Replication Architectures

### Single-Leader (Master-Slave)

```
         [Primary/Master]
         /      |      \
    [Replica1][Replica2][Replica3]

Writes go to Primary only
Reads can go to Replicas
Failover: Promote one Replica to Primary
```

**Pros:**

- Simple to understand and operate
- Clear write path
- Native support in all databases

**Cons:**

- Write bottleneck on primary
- Stale reads from replicas
- Failover needs careful planning

**Use case:** Most applications, default choice

### Multi-Leader (Active-Active)

```
[Leader A] ←→ [Leader B] ←→ [Leader C]
   ↓             ↓             ↓
Replicas     Replicas      Replicas

Writes to any leader
Leaders sync with each other
```

**Pros:**

- No single write bottleneck
- Local writes for multi-region
- Fault tolerance

**Cons:**

- Conflict resolution complexity
- Higher latency
- Write conflicts possible

**Use case:** Multi-datacenter deployments

### Leaderless (Quorum-based)

```
Client writes to multiple nodes (W out of N)
Client reads from multiple nodes (R out of N)
Consistency: W + R > N

Example: N=3, W=2, R=2
```

**Pros:**

- High availability (no leader failure)
- Tunable consistency
- No master bottleneck

**Cons:**

- Complex consistency semantics
- Anti-entropy/read repair needed
- Higher read latency (multiple nodes)

**Use case:** Highly distributed systems (Dynamo, Cassandra)

## 🔄 Synchronous vs Asynchronous

| Aspect         | Synchronous               | Asynchronous                          |
| -------------- | ------------------------- | ------------------------------------- |
| **Durability** | Guaranteed (both written) | Best-effort (may lose on master fail) |
| **Latency**    | Higher (waits for ack)    | Lower (returns immediately)           |
| **Throughput** | Lower                     | Higher                                |
| **RPO**        | Zero (no data loss)       | Non-zero (small loss possible)        |
| **Use case**   | Strong consistency needed | Performance critical                  |

### Synchronous Replication

```sql
-- PostgreSQL: Synchronous replication
synchronous_commit = remote_apply  -- Wait for replica apply
max_wal_senders = 3
synchronous_standby_names = 'standby1,standby2'
```

**Affects:**

- COMMIT latency increases (waits for replica ack)
- If replica disconnects, primary may block writes
- Strongest data durability

### Asynchronous Replication

```sql
-- PostgreSQL: Asynchronous (default)
synchronous_commit = off
```

**Trade-offs:**

- Fast commits (no wait)
- Replica may lag behind primary
- Data loss possible if primary fails

## ⏱️ Replication Lag

### Monitoring

```sql
-- PostgreSQL: Check replication lag
SELECT
    CASE
        WHEN pg_last_wal_receive_lsn() = pg_last_wal_replay_lsn() THEN 0
        ELSE EXTRACT(EPOCH FROM now()) - EXTRACT(EPOCH FROM pg_last_xact_replay_timestamp())
    END AS lag_seconds;

-- MySQL: Check slave lag
SHOW SLAVE STATUS\G
-- Check Seconds_Behind_Master
```

### Impact

```
Replication Lag = Time between write to primary and available on replica

Impact on application:
- User writes to primary
- User reads from replica
- Sees stale data (lag duration)
- May see their writes haven't been applied yet!
```

### Mitigation

```
1. Route critical reads to primary (read-after-write paths)
2. Use replicas only for non-critical reads
3. Monitor lag continuously
4. Alert on lag > threshold (e.g., 1 second)
5. Optimize replication (network, disk IO)
6. Consider synchronous replication for critical data
```

## 🔴 Failover Scenarios

### Automatic Failover

```
1. Primary failure detected (no heartbeat)
2. Replica elected as new primary
3. DNS/connection string updated
4. Applications reconnect
5. Old primary comes back as replica
```

**Requirements:**

- Automatic detection (monitoring, quorum)
- Promotion script (ready to run)
- Connection routing that supports failover
- Data consistency acceptable during failover

**Tools:**

- PostgreSQL: Patroni + etcd
- MySQL: MHA, Percona XtraDB Cluster
- SQL Server: Always On Availability Groups

### Manual Failover

```
1. DBA detects primary failure
2. DBA verifies replica is caught up
3. DBA runs promotion script
4. DBA updates connection strings
5. DBA monitors stability
```

**More controlled, but slower (requires human intervention)**

### Split-brain Prevention

```
PROBLEM: Network partition
Primary: thinks replicas died → continues accepting writes
Replica: thinks primary died → gets promoted

Result: Two primaries with conflicting data!

SOLUTION: Quorum-based decision
Need majority of nodes to agree before promotion
Prevents both sides acting as primary
```

## 🔗 HA Architecture Example

```
┌────────────────────────────────────────────┐
│          Application Instances             │
│  (NestJS, .NET, Python services)          │
└────────────┬─────────────────────────────┘
             │
    ┌────────┴─────────┐
    ▼                  ▼
┌──────────────────────────────┐
│   Connection Router/Proxy    │
│  (pgBouncer, MySQL Router,   │
│   DNS with failover logic)   │
└──────┬──────────────────┬────┘
       │                  │
       ▼                  ▼
  [PRIMARY]          [REPLICA1]
  PostgreSQL 14      PostgreSQL 14
  Region A           Region A

  Streaming Replication (async)

       ▼
  [REPLICA2] (cross-region)
  PostgreSQL 14
  Region B (DR)
```

**Key Components:**

- Primary handles all writes
- Replica1 in same region (fast failover)
- Replica2 in different region (disaster recovery)
- Connection router handles failover transparent to app
- Continuous backups + WAL archiving
- Regular backup restoration tests

## 🚨 Operational Challenges

### Replication Lag Growing

```sql
-- Symptom: Replica lag increasing
-- Check what's slow on replica
SELECT * FROM pg_stat_replication;

-- Possible causes:
-- 1. Slow disk on replica
-- 2. Network saturation
-- 3. Long-running queries blocking replay
-- 4. Replica underpowered

-- Solutions:
-- 1. Add more IO capacity
-- 2. Optimize primary queries
-- 3. Route heavy reads away from replica
-- 4. Upgrade replica hardware
```

### Connection Pool After Failover

```
Problem: All app instances try to reconnect → "thundering herd"
Solution:
- Use exponential backoff with jitter
- Connection pooler handles retry
- DNS TTL set appropriately
```

## 📋 Interview Questions

1. **Design HA for 99.99% uptime requirement**
   - Multi-AZ deployment with automatic failover
   - Synchronous replication for critical data
   - Connection pooling with failover logic
   - Monitoring and alerting on replication lag
   - RTO < 1 minute, RPO = 0

2. **What's the difference between replication and backup?**
   - Replication: Live standby, for failover (RTO: minutes)
   - Backup: Point-in-time recovery, for data recovery (RPO: hours)
   - Use both together

3. **How do you handle replication lag in your application?**
   - Route critical reads (after writes) to primary
   - Use replicas for non-critical/reporting reads
   - Monitor and alert on lag
   - Document acceptable lag limits

4. **Explain split-brain and how to prevent it**
   - Split-brain = multiple primaries accepting writes
   - Prevention: Quorum-based election (needs majority)
   - Fencing = force old primary offline
   - Test failover regularly

## ✅ HA Checklist

- [ ] Primary-replica setup with continuous replication
- [ ] Automated health checks and monitoring
- [ ] Tested failover procedure
- [ ] Connection routing for transparent failover
- [ ] Backup strategy independent of replication
- [ ] Replication lag monitoring and alerting
- [ ] Regular failover drills (game days)
- [ ] Documented runbooks for common failure scenarios
- [ ] Split-brain prevention configured
- [ ] Cross-region replica for disaster recovery

## 📖 Advanced Topics

- [ ] Logical replication (selective table replication)
- [ ] Cascade replication (replica-of-replica)
- [ ] Bi-directional replication and conflict resolution
- [ ] Change Data Capture (CDC) for ETL
- [ ] Sharded replication across multiple clusters
- [ ] Multi-version consistency guarantees

## 🔗 Related

- [Backup & Recovery](../02-backup-recovery/)
- [Performance Tuning](../04-performance-tuning/)
- [Monitoring](../08-monitoring/)
- [PostgreSQL Guide](../07-platform-guides/postgresql.md)

---

**Key Insight:** HA without backup is risky (replication protects from server failure, not from application errors); backup without HA is slow recovery (good for RTO but not RPO).
