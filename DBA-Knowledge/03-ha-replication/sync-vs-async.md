# Synchronous vs Asynchronous Replication: Trade-offs Deep Dive

Understanding the fundamental trade-off between durability and performance in database replication.

---

## 🎯 The Core Trade-off

```
SYNCHRONOUS                          ASYNCHRONOUS
┌─────────────┐                     ┌─────────────┐
│  Primary    │                     │  Primary    │
│ writes data │                     │ writes data │
└──────┬──────┘                     └──────┬──────┘
       │ wait for ack                      │ continue immediately
       ↓                                   ↓
┌──────────────┐                    ┌──────────────┐
│   Replica    │                    │   Replica    │
│ receives+    │                    │ receives+    │
│  applies     │                    │  applies(lag)│
└──────────────┘                    └──────────────┘
       │                                   │
   [slow]                            [fast for primary]
  [safe]                              [risky]
```

---

## 📊 Comparison Matrix

| Aspect                     | Synchronous            | Asynchronous                |
| -------------------------- | ---------------------- | --------------------------- |
| **Primary latency**        | Higher (blocks on ack) | Lower (no wait)             |
| **Throughput**             | Lower (waits per txn)  | Higher (fire & forget)      |
| **RPO**                    | Zero (no data loss)    | Non-zero (minutes possible) |
| **RTO**                    | Depends on failover    | Depends on failover         |
| **Data durability**        | Guaranteed             | Best-effort                 |
| **Network sensitivity**    | High (blocks on lag)   | Low (doesn't block)         |
| **Replica load**           | Can impact primary     | Independent                 |
| **Operational complexity** | Lower                  | Higher (lag management)     |
| **Best for**               | Strong consistency     | High performance            |

---

## 1️⃣ Synchronous Replication

### What Happens

```
Primary receives COMMIT request
    ↓
Write to primary's local storage
    ↓
Send WAL/changes to replica
    ↓
Replica receives and applies changes
    ↓
Replica sends ACK back
    ↓
Primary receives ACK
    ↓
Primary returns "success" to client
    ↓
Primary commits transaction (ACID durable)
```

**Total time = primary write + network latency + replica write + return latency**

### Configuration

#### PostgreSQL

```sql
-- Synchronous replication: Wait for replica ACK
synchronous_commit = remote_apply

-- Options:
-- OFF: async (default)
-- LOCAL: wait for local write (no replica)
-- REMOTE_WRITE: replica writes to storage (not applied)
-- REMOTE_APPLY: replica applies (safest, slowest)
-- ON: same as remote_apply (PostgreSQL 9.2+)

-- Specify which replicas must acknowledge
max_wal_senders = 3              -- Allow 3 replica connections
synchronous_standby_names = 'replica1,replica2'  -- Need both to ack
```

#### MySQL

```sql
-- MySQL 5.7+: Semisynchronous replication
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';

-- Master settings
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 10000;  -- 10 seconds before async

-- Slave settings
SET GLOBAL rpl_semi_sync_slave_enabled = 1;

-- Monitor
SHOW STATUS LIKE 'Rpl_semi_sync%';
```

#### Oracle Data Guard

```sql
-- Synchronous (SYNC) mode
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2='SERVICE=dr_db SYNC';

-- Asynchronous (ASYNC) mode
ALTER SYSTEM SET LOG_ARCHIVE_DEST_2='SERVICE=dr_db ASYNC';
```

### Performance Impact

```
Scenario: Single transaction commit latency

Primary in NYC, Replica in California (50ms network latency)

ASYNCHRONOUS:
  Primary write: 1ms
  Total: ~1ms (immediate)

SYNCHRONOUS (remote_apply):
  Primary write: 1ms
  Send to replica: 50ms
  Replica write: 1ms
  Replica ack: 50ms
  Total: ~102ms (100x slower!)
```

### Latency Trade-offs

```
synchronous_commit levels (PostgreSQL):

OFF (~1ms)
│
LOCAL (~5ms - includes local fsync)
│
REMOTE_WRITE (~50-100ms - replica receives)
│
REMOTE_APPLY (~100-200ms - replica applies, safest)
```

### Durability Guarantees

**Synchronous = "I won't tell you success until it's on the replica too"**

```
Primary crash scenario:

SYNC: All ACK'd transactions on replica
   → No data loss
   → Replica becomes primary safely
   → 0 RPO

ASYNC: Only transactions on primary (lag)
   → Up to [lag_duration] of data loss
   → Replica may be behind
   → RPO = replication_lag
```

### When Network Conditions Worsen

```
If replica becomes slow (e.g., network congestion):

SYNC problem: Primary blocks on all writes
- Commits timeout
- Application times out
- Writes queue up
- Potential outage on primary side

Solution: Set timeout to fallback to async
```

### Advanced: Conditional Synchronous

#### PostgreSQL Quorum Sync

```sql
-- Need 1 out of 2 replicas to acknowledge (k-safe)
synchronous_standby_names = 'ANY 1 (replica1, replica2)'

-- vs need both
synchronous_standby_names = 'replica1, replica2'
```

#### Behavior with timeouts

```sql
-- If replica doesn't respond in 30 seconds
-- automatically fall back to async
-- (PostgreSQL 9.2+)
```

### Best Practices for Sync Replication

1. **Keep primary and replica close** (same region)
   - Latency should be <10ms

2. **Monitor synchronization status**

   ```sql
   -- PostgreSQL
   SELECT * FROM pg_stat_replication;
   ```

3. **Have fallback strategy**
   - Timeout → async mode
   - Alert ops → investigate network

4. **Size appropriately**
   - Don't sync more replicas than needed
   - Each one adds latency
   - Quorum sync (ANY 1 of 3) is better than all 3

5. **Document acceptable latency**
   - Business requirement (SLA)
   - Not just technical constraint

---

## 2️⃣ Asynchronous Replication

### What Happens

```
Primary receives COMMIT request
    ↓
Write to primary's local storage
    ↓
Primary returns "success" to client
    ↓
Primary commits transaction (ACID on primary)
    ↓
Primary sends WAL/changes to replica (background)
    ↓
Replica receives and applies (in background)
    ↓
[No ACK back to primary]
```

**Total time = primary write only (~1-5ms)**

### Configuration

#### PostgreSQL

```sql
-- Asynchronous replication (default)
synchronous_commit = OFF

-- or more explicitly
synchronous_commit = LOCAL
synchronous_standby_names = ''
```

#### MySQL

```sql
-- Asynchronous replication (default)
-- No special config needed

-- Verify async is in use
SHOW SLAVE STATUS\G
-- Check: Master_User, Relay_Master_Log_File
```

### Replication Lag

```
Async replication lag = time between write on primary and apply on replica

                       time
Primary:  T1: INSERT   T2: DELETE   T3: UPDATE
          ←──────────────────────→
Replica:             T1: INSERT   T2: DELETE   T3: UPDATE

Lag = T2 (on replica) - T1 (on primary)
Typically: 0-5ms in healthy system, seconds under load
```

### Data Loss Risk

**Key trade-off: You can lose data**

```
Scenario: Primary crashes during async replication

Time    Primary                 Replica
0ms:    Write commit success    [async queue]
1ms:    Write 2                 [still in queue]
5ms:    CRASH! ← Primary fails
10ms:   Old data lost           Replica still applying

Result: Committed data is GONE
        Application was told "success"
        But data wasn't replicated yet

RPO = network + primary write time ≈ 0-50ms typical
```

### When to Use Async

✅ **High-write-throughput** systems (don't want blocking)
✅ **Performance critical** (application latency matters)
✅ **Acceptable RPO** (small data loss tolerable)
✅ **Combined with backups** (replication not sole safety net)
✅ **Read-heavy** (replicas lag but available)

### Monitoring Async Replication

```sql
-- PostgreSQL: Check lag
SELECT
    client_addr,
    write_lsn,
    replay_lsn,
    (write_lsn - replay_lsn) as bytes_behind,
    CASE
        WHEN replay_lsn = write_lsn THEN '0'
        ELSE pg_wal_lsn_diff(write_lsn, replay_lsn)::text
    END AS bytes_lag
FROM pg_stat_replication;

-- MySQL: Check lag
SHOW SLAVE STATUS\G
-- Look for: Seconds_Behind_Master
```

### The "Thundering Herd" Problem

```
With async replication:
- Replica is always slightly behind
- If primary fails
- Applications have brief stale data
- When replica promoted, there's a period where:
  - Some servers point to old primary (failed)
  - Some servers point to new replica-turned-primary
  - Inconsistent reads across cluster

Solution:
- Use connection pooler with failover
- Update DNS aggressively
- Use service discovery
```

---

## 3️⃣ Hybrid: Semisynchronous Replication

### Concept

"Synchronous by default, async when needed"

```
Normal operation:
  Primary waits for ≥1 replica ACK (SYNC)

If replica fails/is slow:
  Primary times out (e.g., 10 seconds)
  Falls back to ASYNC
  Alerts fired
  Operations investigates
```

### Configuration (MySQL)

```sql
-- Install semisync plugin (if not built-in)
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';

-- On master
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 10000;  -- 10 sec timeout

-- On slave
SET GLOBAL rpl_semi_sync_slave_enabled = 1;

-- Monitor
SHOW STATUS WHERE variable_name LIKE 'Rpl_semi_sync%';
-- If Rpl_semi_sync_master_clients = 0 → no slaves ready
-- If Rpl_semi_sync_master_yes_tx = 0 → no semi-sync happening
```

### Behavior

```
Transaction flow:

Primary: COMMIT
  ↓
  Wait for replica (up to 10 seconds)
  ↓
  ✓ Replica ACK received within timeout
    → Commit succeeds (SYNC)
  ✗ Timeout (replica down/slow)
    → Fallback to ASYNC
    → Commit returns (but risky)
    → Alert: semi-sync not working!
```

### When to Use Semisync

✅ **Want safety** but not full latency cost
✅ **Can tolerate occasional async** when replicas are down
✅ **Want alerting** when things go wrong
✅ **Most production systems** benefit from this
❌ **Guaranteed 0 RPO** (some async periods)
❌ **Guaranteed low latency** (waits in normal case)

---

## 4️⃣ Group Replication (Automatic Consensus)

### PostgreSQL Synchronous Replication with Quorum

```sql
-- Multiple replicas, need 2 to acknowledge
synchronous_standby_names = 'FIRST 2 (replica1, replica2, replica3)'

-- or ANY 1 of them
synchronous_standby_names = 'ANY 1 (replica1, replica2, replica3)'
```

### MySQL Group Replication

```sql
-- Install plugin
INSTALL PLUGIN group_replication SONAME 'group_replication.so';

-- Setup for automatic consensus
SET GLOBAL group_replication_consistency = 'BEFORE';
SET GLOBAL group_replication_single_primary_mode = ON;

-- Start replication
START GROUP_REPLICATION;

-- Status
SELECT * FROM performance_schema.replication_group_members;
```

---

## 📈 Performance Comparison (Real Numbers)

### Test: 1000 transactions, 4KB each

```
Scenario: Primary (10ms CPU) + Replica (20ms away, 5ms CPU)

ASYNC:
  Primary latency: ~10ms (just local write + WAL)
  Throughput: ~100 txn/sec
  RPO: ~100ms (typical lag)

SYNC (remote_apply):
  Primary latency: ~50-60ms (2x network + replica)
  Throughput: ~15 txn/sec
  RPO: 0

SEMISYNC (1 replica, 10s timeout):
  Normal latency: ~50-60ms (same as sync)
  Under load: Drops to ~10ms (fallback to async)
  Throughput: ~15-100 txn/sec (varies)
  RPO: Usually 0, occasionally ~100ms
```

---

## 🔧 Decision Framework

### Choose ASYNC if:

- Performance > safety
- RPO of seconds-to-minutes acceptable
- Writes are high-volume
- Combined with regular backups
- Read replicas more important than failover

### Choose SYNC if:

- Data loss unacceptable (financial txn, compliance)
- Latency <50ms acceptable
- Small cluster (1-2 replicas)
- Replicas in same region

### Choose SEMISYNC if:

- Most systems! (good balance)
- Want safety with fallback
- Tolerance for occasional async
- Can handle 50ms latency spikes

---

## 🚨 Operational Gotchas

### Sync Replication Blocked Primary

```
Problem: Replica goes down, primary blocks waiting

pg_stat_replication: no rows (replica disconnected)
SELECT * FROM pg_stat_activity;  -- All queries blocked!

Solution:
1. Fix replica connectivity
2. Or set synchronous_standby_names = '' (remove replica)
3. Or increase rpl_semi_sync_master_timeout
```

### Async Replication Gap During Failover

```
Problem: Promoted replica missing recent transactions

Primary had: INSERT 1, INSERT 2, INSERT 3
Replica had: INSERT 1, INSERT 2
Failover: Replica promoted
Result: INSERT 3 lost

Solution: Have better failover detection
```

### Network Partition Split-Brain

```
Problem: Slow network but not disconnected

Primary waits indefinitely for replica (with sync)
Replica thinks primary dead
Failover happens
Two primaries accepting writes!

Solution: Use quorum (ANY 1 of 3) or timeout
```

---

## 📋 Configuration Checklist

**For High-Availability Production System:**

- [ ] Decide: Async vs Sync vs Semisync
- [ ] Document decision and why
- [ ] Configure timeout for sync (if used)
- [ ] Monitor replication lag continuously
- [ ] Alert on lag > threshold
- [ ] Test failover quarterly
- [ ] Verify backup strategy independent of replication
- [ ] Know what happens on replica disconnect
- [ ] Test disaster recovery with async lag
- [ ] Document acceptable RPO

---

## 🔗 Related Topics

- [Replication Types](replication-types.md)
- [Replication Lag Monitoring](replication-lag-monitoring.md)
- [Failover Strategies](failover-strategies.md)
- [Backup & Recovery (independent of replication)](../02-backup-recovery/)

---

**Key Insight:** Sync = safety at latency cost. Async = performance at safety cost. Semisync = best of both with fallback. Choose based on your SLA, not your gut.
