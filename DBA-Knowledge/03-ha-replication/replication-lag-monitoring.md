# Replication Lag Monitoring: Detection & Resolution

Master the critical art of monitoring and resolving replication lag to ensure data consistency and application reliability.

---

## 🎯 What is Replication Lag?

**Replication Lag = Time between when transaction commits on primary and when it's applied on replica**

```
Timeline:
T0:     Client sends COMMIT to primary
T1ms:   Primary writes to disk (1-5ms)
T2ms:   Primary sends WAL to replica (network latency)
T3ms:   Replica receives and queues (typically <5ms)
T4ms:   Replica applies from queue (can be delayed)
T5ms:   Replica durably writes

Replication Lag = T5 - T0 = typically 5-50ms in healthy system
```

### Why It Matters

**Critical reads after writes**

```
User writes data to primary (COMMIT returns)
User immediately refreshes (expects to see their data)
But if read goes to replica with lag...
User sees stale/empty data
Bad UX: "Where did my data go?"
```

**Failover consistency**

```
Primary has: 1000 transactions
Replica has: 950 transactions (50 transactions behind)
Primary crashes
Failover to replica
Promoted replica missing: 50 transactions
Data loss!
```

**Compliance & auditing**

```
Regulatory requirement:
"All changes must be replicated within 10 seconds"
If lag > 10s: Compliance violation
Potential audit failure / fines
```

---

## 🔍 Measuring Replication Lag

### PostgreSQL

#### Method 1: Using LSN (Preferred)

```sql
-- On primary: Get current WAL position
SELECT pg_current_wal_lsn() as primary_lsn;
-- Output: 0/3C72908

-- On replica: Get last received/replayed LSN
SELECT
    pg_last_wal_receive_lsn() as replica_received,
    pg_last_wal_replay_lsn() as replica_replayed;
-- Output: 0/3C72908 (same = caught up)
--         0/3C72800 (behind = lag exists)

-- Calculate bytes behind
SELECT
    (pg_current_wal_lsn() - pg_last_wal_replay_lsn())
    ::text as bytes_lag;
-- Output: 10000 (bytes behind)

-- Better: On replica, query replication stats
SELECT
    client_addr,
    usename,
    application_name,
    backend_start,
    backend_xmin,
    write_lsn,
    flush_lsn,
    replay_lsn,
    sync_state
FROM pg_stat_replication;
```

#### Method 2: Using Timestamp

```sql
-- On replica: Get last transaction replay time
SELECT
    now() - pg_last_xact_replay_timestamp() as replication_lag
FROM pg_stat_replication;
-- Output: "00:00:02.123"  (2 seconds)

-- Disadvantage: If replica is idle, doesn't update
-- More useful for high-transaction systems
```

#### Method 3: Test with Marker Table

```sql
-- On primary: Create marker table
CREATE TABLE replication_lag_marker (
    id INT PRIMARY KEY,
    updated_at TIMESTAMPTZ DEFAULT now()
);

-- Every 5 seconds (via cron or app):
INSERT INTO replication_lag_marker VALUES (1, now())
ON CONFLICT (id) DO UPDATE SET updated_at = now();

-- On replica: Query marker
SELECT
    now() - updated_at as replication_lag
FROM replication_lag_marker;
-- Output: 2.5 seconds
```

### MySQL / MariaDB

#### Semi-sync replication status

```sql
-- On primary
SHOW STATUS LIKE 'Rpl_semi_sync%';
-- Rpl_semi_sync_master_yes_tx: Transactions with semi-sync
-- Rpl_semi_sync_master_no_tx: Transactions without semi-sync
-- Rpl_semi_sync_master_clients: Number of semi-sync clients

-- On replica
SHOW STATUS LIKE 'Rpl_semi_sync%';
-- Rpl_semi_sync_slave_send_ack: ACKs sent to master
```

#### Traditional replication lag

```sql
-- On replica
SHOW SLAVE STATUS\G

-- Key fields:
-- Seconds_Behind_Master: Seconds of lag
-- Master_Log_File: Current log file reading
-- Relay_Master_Log_File: Log file being executed
-- Master_Log_Pos vs Read_Master_Log_Pos: Position gap
-- Slave_IO_State: "Waiting for master to send event" (normal)
```

### MongoDB

```javascript
// On replica set member
db.admin.command({ replSetGetStatus: 1 });
// Check: "optime" fields - time difference between members

// Better: Use rs.status() in shell
rs.status();
// Shows: members, their lastHeartbeat, optime offsets
```

---

## 📊 Monitoring in Production

### Prometheus Query Examples

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "postgres"
    static_configs:
      - targets: ["localhost:9187"] # postgres_exporter
```

```promql
# Query: PostgreSQL replication lag bytes
pg_replication_lag_bytes

# Query: PostgreSQL replication lag seconds
pg_replication_lag_seconds

# Query: Alert if lag > 1 second
ALERT ReplicationLagTooHigh
  IF pg_replication_lag_seconds > 1
  FOR 5m
  ANNOTATIONS:
    summary: "Replication lag is {{ $value | humanizeDuration }}"
```

### Grafana Dashboards

```json
{
  "panels": [
    {
      "title": "Replication Lag (seconds)",
      "targets": [
        {
          "expr": "pg_replication_lag_seconds"
        }
      ],
      "alert": {
        "conditions": [
          {
            "evaluator": { "params": [1], "type": "gt" },
            "operator": { "type": "and" },
            "query": { "params": ["A", "now", ""] },
            "type": "query"
          }
        ]
      }
    }
  ]
}
```

### Custom Monitoring Script

```python
#!/usr/bin/env python3
import psycopg2
import time
from datetime import datetime

PRIMARY = "primary.example.com"
REPLICA = "replica.example.com"

def check_lag():
    """Check and report replication lag"""

    # Connect to primary
    with psycopg2.connect(f"host={PRIMARY}") as conn:
        cursor = conn.cursor()
        cursor.execute("SELECT pg_current_wal_lsn()::text")
        primary_lsn = cursor.fetchone()[0]

    # Connect to replica
    with psycopg2.connect(f"host={REPLICA}") as conn:
        cursor = conn.cursor()
        cursor.execute("SELECT pg_last_wal_replay_lsn()::text")
        replica_lsn = cursor.fetchone()[0]

    # Parse LSN and calculate difference
    # LSN format: 0/3C72908 = 16-bit / 32-bit
    primary_val = int(primary_lsn.split('/')[1], 16)
    replica_val = int(replica_lsn.split('/')[1], 16)

    lag_bytes = primary_val - replica_val

    print(f"[{datetime.now()}] Lag: {lag_bytes} bytes ({lag_bytes/1024:.2f} KB)")

    # Alert if too high
    if lag_bytes > 1_000_000:  # 1 MB
        print(f"⚠️  ALERT: Replication lag exceeds 1MB!")
        send_alert(f"Lag: {lag_bytes} bytes")

def send_alert(message):
    # Send to PagerDuty / Slack / etc
    pass

if __name__ == "__main__":
    while True:
        check_lag()
        time.sleep(5)
```

---

## ⚠️ Detecting Lag Issues

### Red Flags

| Signal                       | Cause                                       | Severity    |
| ---------------------------- | ------------------------------------------- | ----------- |
| **Lag growing continuously** | Replica can't keep up with writes           | 🔴 Critical |
| **Lag spikes periodically**  | Long queries block replication              | 🟡 Warning  |
| **Replica IO at 100%**       | Disk is bottleneck                          | 🔴 Critical |
| **Replica CPU at 100%**      | Complex queries, indexing slow              | 🔴 Critical |
| **Network saturation**       | WAN link overloaded                         | 🔴 Critical |
| **Replication apply lag**    | Replica can decode WAL, but applying slowly | 🟡 Warning  |
| **Lag after failover**       | New primary catching up to promoted replica | ℹ️ Normal   |

### Query: Find What's Causing Lag

```sql
-- PostgreSQL: Check for long-running queries on replica
SELECT
    pid,
    usename,
    query,
    query_start,
    NOW() - query_start as duration,
    state
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- If replication process is blocked:
SELECT
    pid,
    usename,
    query,
    state
FROM pg_stat_activity
WHERE backend_type = 'walsender' OR application_name = 'replication';

-- Check replication process lag specifically
SELECT
    client_addr,
    backend_type,
    state,
    backend_start,
    wait_event,
    wait_event_type
FROM pg_stat_activity
WHERE backend_type IN ('walsender', 'wal_receiver');
```

---

## 🔧 Troubleshooting Common Causes

### 1. Replica Disk I/O Bottleneck

**Symptoms:**

```
Lag growing
iostat shows disk utilization 100%
iowait high
```

**Diagnosis:**

```bash
# Check disk I/O
iostat -x 1 5

# Check what's writing
lsof | grep postgresql | grep write

# Check dirty cache
cat /proc/sys/vm/dirty_ratio
```

**Solutions:**

```
1. Add more disk bandwidth (upgrade storage)
2. Tune OS I/O:
   - Increase write buffer (dirty_ratio)
   - Use faster disk (SSD instead of HDD)
3. Optimize queries (slow INDEX creation)
4. Reduce write load on primary
```

### 2. Replica CPU Bottleneck

**Symptoms:**

```
Lag growing
CPU utilization 100% on replica
Replica much slower than primary
```

**Diagnosis:**

```sql
-- PostgreSQL: Check what process is consuming CPU
SELECT
    pid,
    usename,
    query,
    backend_type,
    query_start
FROM pg_stat_activity
ORDER BY query_start;

-- Check for index creation in progress
SELECT
    schemaname,
    tablename,
    indexname
FROM pg_stat_user_indexes
WHERE idx_tup_read > 1000000;  -- Recently heavily used
```

**Solutions:**

```
1. Upgrade replica CPU (vertical scaling)
2. Optimize queries (add indexes)
3. Parallel query execution (work_mem, max_parallel_workers)
4. Reduce load: Move reporting queries elsewhere
```

### 3. Network Bandwidth Saturation

**Symptoms:**

```
Lag growing during high-write periods
Network utilization 95%+
Primary and replica close (not network latency issue)
```

**Diagnosis:**

```bash
# Check network throughput
iftop -n -P

# Check TCP retransmissions
netstat -s | grep retransmit

# Monitor primary -> replica traffic
ssh replica 'tcpdump -i eth0 src PRIMARY_IP'
```

**Solutions:**

```
1. Upgrade network (10Gbps instead of 1Gbps)
2. Compress replication traffic
3. Reduce write volume to primary
4. Use multi-region cascade (primary -> replica1 -> replica2)
```

### 4. Long-running Query Blocks Replication (PostgreSQL)

**Symptoms:**

```
Lag suddenly spikes to 10+ seconds
Lag plateaus at certain level
```

**Cause:**

```
PostgreSQL holds exclusive locks on pages during writes
Reader (even replication) must wait for locks
Long SELECT on replica blocks replay process
```

**Solution:**

```sql
-- Find blocking query
SELECT
    pid,
    usename,
    query,
    query_start,
    NOW() - query_start as duration
FROM pg_stat_activity
WHERE NOT query LIKE '%pg_sleep%'
    AND query_start < NOW() - '1 minute'::INTERVAL
ORDER BY query_start;

-- Kill long query (carefully!)
SELECT pg_terminate_backend(PID);

-- Prevent future issues: Set statement timeout
ALTER USER replica_user SET statement_timeout = '5min';
```

### 5. Replica Too Slow (Underpowered Hardware)

**Symptoms:**

```
Replica specs half of primary
Lag always exists
Lag proportional to write load
```

**Solution:**

```
Primary → Replica speeds should be similar
If primary: 32 CPU, 128GB RAM
Then Replica: 32 CPU, 128GB RAM (not 8 CPU, 32GB)

Otherwise: Replica will always lag under load
```

---

## 📈 Acceptable Lag Thresholds

### By Application Type

```
Financial transactions:
├─ Lag threshold: < 100ms
├─ Action: Promote to primary immediately
├─ Critical: Exact consistency

Social media (feed):
├─ Lag threshold: < 5 seconds
├─ Action: Use eventual consistency on replica
├─ Acceptable: Small delay in posts showing

Reporting/Analytics:
├─ Lag threshold: < 1 hour
├─ Action: Separate replica just for reporting
├─ No impact: User sees slightly stale reports

Caching layer:
├─ Lag threshold: < 30 seconds
├─ Action: Invalidate cache on lag
├─ Acceptable: Brief stale data
```

### SLA to Lag Threshold

```
99.9% uptime SLA (8.6 hours downtime/year):
├─ RTO requirement: < 15 minutes typically
├─ Lag threshold: < 5 seconds (preserve data)
├─ Failover strategy: Semi-automatic

99.99% uptime SLA (52 minutes downtime/year):
├─ RTO requirement: < 1 minute
├─ Lag threshold: < 100ms (near-zero loss)
├─ Failover strategy: Automatic

99.999% uptime SLA (5.2 minutes downtime/year):
├─ RTO requirement: < 10 seconds
├─ Lag threshold: < 10ms (synchronous)
├─ Failover strategy: Fully automatic + quorum
```

---

## 🛠️ Alert Configuration

### Alert: High Replication Lag

```yaml
# prometheus.yml
groups:
  - name: replication
    interval: 30s
    rules:
      - alert: ReplicationLagHigh
        expr: pg_replication_lag_seconds > 5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High replication lag"
          description: "Replica is {{ $value }}s behind primary"

      - alert: ReplicationLagCritical
        expr: pg_replication_lag_seconds > 30
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "CRITICAL: Very high replication lag"
          description: "Replica is {{ $value }}s behind. Check replica resources."

      - alert: ReplicationStalled
        expr: rate(pg_replication_lag_bytes[5m]) == 0 AND pg_replication_lag_bytes > 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Replication appears stalled"
          description: "Lag not decreasing for 5 minutes"
```

### Slack Notification Template

```
🚨 Replication Lag Alert

Replica: replica-prod-1.example.com
Lag: 45 seconds (threshold: 5s)
Severity: WARNING

Quick actions:
1. Check replica resources: CPU, disk, network
2. Run: SELECT * FROM pg_stat_activity (find blocking query)
3. Investigate: Top queries on primary
4. Escalate if: lag continues growing

See dashboard: https://grafana/d/replication
```

---

## 📋 Monitoring Checklist

- [ ] Replication lag metric exported (Prometheus/cloudwatch)
- [ ] Dashboard shows lag trend over time
- [ ] Alert triggered if lag > threshold (and stays > 2 min)
- [ ] Alert triggered if replication stalled (lag not decreasing)
- [ ] Runbook available for "high lag" alert
- [ ] On-call knows how to diagnose and fix lag issues
- [ ] Threshold documented (why chosen, who set it)
- [ ] Tested alert (fire manually, receive notification)
- [ ] Tested remediation (kill query, restart replication)
- [ ] Dashboards accessible to developers

---

## 🚨 Emergency Procedures

### If Lag Keeps Growing

```
1. MONITOR
   Check lag trend - is it accelerating?

2. CHECK REPLICA RESOURCES
   CPU? Disk I/O? Memory? Network?

3. FIND BLOCKING QUERY (if PostgreSQL)
   SELECT * FROM pg_stat_activity;
   Kill if safe: SELECT pg_terminate_backend(PID);

4. PAUSE WRITES (if critical)
   Set primary read-only (extreme measure!)
   ALTER SYSTEM SET default_transaction_read_only = ON;

5. REBUILD REPLICA (if needed)
   Stop replication
   pg_basebackup or equivalent
   Restart replication

6. POST-INCIDENT
   Review: Why did this happen?
   Document: Prevention for future
   Alert: Can we catch this earlier?
```

### If Replication Stops Entirely

```
1. Check replica connectivity
   ssh replica "psql -c 'SELECT 1'"

2. Check replication process
   ps aux | grep 'wal receiver|walsender'

3. Check primary export
   SELECT * FROM pg_stat_replication;

4. Check for WAL files on primary
   ls -la /var/lib/postgresql/pg_wal/

5. Manual restart
   On replica: pg_ctl promote -D $PGDATA  (make standalone)
   Then: Set up replication again
```

---

## 🔗 Related Topics

- [Sync vs Async Replication](sync-vs-async.md) — Understanding lag guarantees
- [Failover Strategies](failover-strategies.md) — What happens when lag > 0
- [Replication Types](replication-types.md) — Architectural considerations
- [Performance Tuning](../04-performance-tuning/) — Optimizing query speed

---

**Key Insight:** Replication lag is inevitable. The question isn't "how to eliminate it" but "how to detect it early, measure it accurately, and respond before it causes business impact."
