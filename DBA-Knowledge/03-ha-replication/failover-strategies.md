# Failover Strategies: From Manual to Fully Automated

Master failover procedures to minimize downtime and prevent split-brain scenarios.

---

## 🎯 Overview

**Failover = Recovery from primary database failure by promoting a replica to primary**

### Key Metrics

| Metric   | Definition                    | Goal                                                 |
| -------- | ----------------------------- | ---------------------------------------------------- |
| **RTO**  | Recovery Time Objective       | How quickly after failure are writes available?      |
| **RPO**  | Recovery Point Objective      | How much data can you lose (seconds behind primary)? |
| **MTBx** | Mean Time Between/To failures | How often failures happen                            |
| **MTTR** | Mean Time To Repair           | How long to fully restore redundancy                 |

### Example: 99.99% Uptime SLA

```
Downtime budget: 52 minutes/year ≈ 4 minutes/month

If RTO = 1 minute:
  → ~52 failures/year tolerable
  → Feasible with semi-automated failover

If RTO = 30 seconds:
  → ~104 failures/year tolerable
  → Requires automatic detection + promotion

If RTO = 5 seconds:
  → ~625 failures/year tolerable
  → Requires instant automatic failover
  → Not realistic with human involvement
```

---

## 1️⃣ Manual Failover

### Process

```
1. Monitoring detects primary is down
   └─ Database connection fails
   └─ Health check timeout
   └─ Alert sent to on-call DBA

2. DBA verifies primary is truly down
   └─ SSH to primary, try to connect
   └─ Check logs: crash? hung process?
   └─ Decision: really gone? or temporary network blip?

3. DBA verifies replica is caught up
   └─ Check replication lag
   └─ If lag > 0: Decisions to make (lose data? wait?)

4. DBA promotes replica to primary
   └─ Run promotion script:
   └─ Stop replication
   └─ Set read-write mode
   └─ Verify new primary accepts writes

5. DBA updates connection strings
   └─ Update application config
   └─ Update DNS
   └─ Update monitoring targets

6. DBA monitors for issues
   └─ Verify applications connected
   └─ Check performance
   └─ Monitor system stability

7. DBA recovers old primary (much later)
   └─ Restart old primary as replica
   └─ Rebuild if needed
```

### Typical Timing

```
Normal database failure:
5s   → Problem detected (health check timeout)
30s  → Alert received by DBA
60s  → DBA confirmed failure
120s → DBA runs promotion script
180s → Applications notice DNS change (TTL=60s)
300s → System recovered, monitoring stable

Total RTO: ~5 minutes
Total impact: ~500 connections reconnecting = ~100 spike
```

### Advantages

✅ **Human judgment** — Can handle edge cases
✅ **Safety** — Explicit verification before promotion
✅ **Simplicity** — Minimal automation needed
✅ **Testing easy** — Practice runbook manually
✅ **Emergency brake** — DBA can stop promotion

### Disadvantages

❌ **Slow** — RTO measured in minutes
❌ **On-call overhead** — Requires live DBA
❌ **Human error** — Finger slip on rm -rf during incident
❌ **Sleep quality** — Oncall burden
❌ **Doesn't scale** — 10 failures/day = burnout

### Manual Failover Runbook Template

````markdown
# Production Database Failover Runbook

## Failure Detection

- Alerting system triggers: "Primary down"
- Acknowledge alert in PagerDuty
- Open war room (Slack channel)

## Verification (30 seconds)

```bash
# 1. Verify primary is truly down
ping -c 3 db-primary-prod.internal
timeout 5 psql -h db-primary-prod.internal -U postgres -c "SELECT 1" || echo "FAILED"

# 2. Check logs on primary (if accessible)
ssh db-primary-prod.internal
tail -100 /var/log/postgresql/postgresql.log | grep ERROR

# 3. Check replica status
ssh db-replica-prod.internal
psql -U postgres -c "SELECT now() - pg_last_xact_replay_timestamp() as replication_lag;"
```
````

## Promotion (2 minutes)

```bash
# 4. On replica, promote to primary
ssh db-replica-prod.internal
sudo systemctl stop postgresql
sudo -u postgres pg_ctl promote -D /var/lib/postgresql/14/main

# 5. Verify it's primary
psql -U postgres -c "SELECT pg_is_in_recovery();"
# Should return: f (false, meaning NOT in recovery = is primary)

# 6. Verify can accept writes
psql -U postgres -c "CREATE TABLE test_table (id INT); DROP TABLE test_table;"
```

## Connection Update (1 minute)

```bash
# 7. Update connection strings (update in config repo)
# OLD: dbhost=db-primary-prod.internal
# NEW: dbhost=db-replica-prod.internal (old replica now primary)

# 8. Update DNS
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123 \
  --change-batch "file://update-dns.json"

# 9. Restart applications (or trigger reconnect)
kubectl set env deployment/api \
  DB_HOST=db-replica-prod.internal \
  ROLLOUT_RESTART=true
```

## Verification (2 minutes)

- [ ] New primary accepting queries
- [ ] All replica applications connected
- [ ] No error logs
- [ ] Monitoring shows stable latency
- [ ] Backup system updated to backup new primary

## Recovery (later)

- [ ] Contact provider about old primary
- [ ] Review logs to understand failure cause
- [ ] Prepare old primary as new replica
- [ ] Test rebuild procedure
- [ ] Schedule maintenance to complete setup

```

---

## 2️⃣ Semi-Automated Failover

### Process

```

Automation detects failure + runs promotion script
DBA approves promotion
Applications notified + reconnect

````

### Implementation Example (Patroni)

```yaml
# patroni.yml configuration
scope: postgres-prod
namespace: /patroni

postgresql:
  parameters:
    max_connections: 200
    shared_buffers: 256MB

failover:
  loop_wait: 10
  ttl: 30
  retry_timeout: 10
  maximum_lag_on_failover: 1048576  # 1MB
  master_start_timeout: 300
  synchronous_mode: true
  synchronous_mode_strict: false

# Monitoring/alerting
watchdog:
  mode: automatic
  device: /dev/watchdog
  safety_margin: 5
````

### Automatic Detection & Promotion

```python
# Pseudocode: Semi-automated failover framework
class DatabaseFailoverManager:

    def detect_failure(self):
        """Check if primary is alive"""
        for attempt in range(3):
            if not self.health_check(primary):
                if not self.network_check():  # isn't network outage?
                    return True
        return False

    def promote_replica(self):
        """Execute promotion scripts"""
        replica = self.select_best_replica()
        # Verify caught up
        if replica.replication_lag > 0:
            self.notify_operators("Replica lag: {} bytes".format(...))
            if not self.wait_for_catchup(timeout=30):
                self.log_warning("Forcing promotion despite lag")

        # Run promotion
        result = replica.promote()
        if result.success:
            self.notify_operators("Promoted successfully")
            return True
        else:
            self.notify_operators("Promotion FAILED: {}".format(result.error))
            return False

    def update_routing(self):
        """Update where apps connect"""
        # Option 1: DNS failover
        self.update_dns_primary(new_primary=self.replica.hostname)

        # Option 2: Update service discovery
        self.consul.register_service(
            name='postgres-primary',
            address=self.replica.ip,
            port=5432
        )
```

### Timing

```
5s   → Health check fails (consensus)
15s  → Replica promotion started
30s  → Promotion complete
45s  → DNS/service discovery updated
60s  → Applications start reconnecting
90s  → Most connections migrated

Total RTO: ~2 minutes
```

### Advantages

✅ **Faster than manual** (RTO: 1-2 minutes)
✅ **Repeatable** — Automation reduces errors
✅ **24/7 without on-call burden** — Runs automatically
✅ **Still has safety** — Requires explicit triggers
✅ **Easy to test** — Run playbooks regularly

### Disadvantages

❌ **Complex setup** — Framework to maintain
❌ **Edge cases** — Still need human for unusual situations
❌ **False positives** — Network hiccup triggers failover
❌ **Rollback hard** — Once started, hard to stop

### Semi-Automated Tools

- **PostgreSQL**: Patroni + etcd, pg_auto_failover
- **MySQL**: Orchestrator, MHA (MySQL HA)
- **MongoDB**: Built-in replica set auto-failover
- **Cloud RDS**: AWS RDS Multi-AZ automatic failover
- **Kubernetes**: Operators (CrunchyData PostgreSQL Operator)

---

## 3️⃣ Fully Automatic Failover

### Process

```
Quorum detects failure
Replica automatically promoted
Applications reconnect transparently
Zero human involvement
```

### Architecture

```
┌─────────────────────────────────────────────────┐
│            Application Layer                     │
│  (Connection pooling, auto-reconnect)           │
└────────────┬─────────────────────────────────┬──┘
             │                                 │
    ┌────────▼────────┐              ┌─────────▼────────┐
    │  Read Replica 1 │              │ Read Replica 2   │
    └────────┬────────┘              └─────────┬────────┘
             │                               │
             └───────────┬───────────────────┘
                        │ Replication
                        ▼
             ┌──────────────────────┐
             │   Primary (Active)   │
             │   [with watchdog]    │
             └──────────┬───────────┘
                        │
           ┌────────────┼────────────┐
           │            │            │
          ▼             ▼            ▼
      [Replica1]   [Replica2]   [Replica3]
    (standby)      (standby)    (standby)
       │              │           │
       └──────────────┴───────────┘
              Quorum consensus
        (etcd/Zookeeper/other)
```

### How It Works

```
Normal State:
Primary: Leader (writes accepted)
Replicas: Standbys (replication streaming)
Quorum: Stores leader location

Primary Failure Detected:
Primary heartbeat expires
Quorum leader timeout (e.g., 30 seconds)
Quorum triggers failover election

Replica Promotion:
Highest priority replica elected
Replica stops replication (becomes leader)
Replica starts accepting writes
Quorum updated with new leader

Connection Rerouting:
Applications have smart driver (auto-failover driver)
Driver notifies quorum for new primary location
Driver automatically reconnects
Transparent to application!
```

### Implementation (PostgreSQL + Patroni + etcd)

```yaml
# patroni.yml - Enable automatic failover
patroni:
  ttl: 30
  loop_wait: 10
  retry_timeout: 10
  maximum_lag_on_failover: 1000000

  watchdog:
    mode: automatic
    device: /dev/watchdog
    safety_margin: 5

postgresql:
  synchronous_commit: remote_apply
  synchronous_standby_names: "patroni"
```

```bash
# Start Patroni on each node
patronictl -c patroni.yml start
patronictl members
```

### Timing

```
5s   → Primary heartbeat not detected by quorum
10s  → Quorum leader realizes primary gone (TTL expired)
15s  → Failover triggered, replicas notified
20s  → Best replica promoted
25s  → Quorum updated with new leader
30s  → Applications' drivers detect change
35s  → Applications reconnected
40s  → Read traffic flowing

Total RTO: ~30-40 seconds
Total RPO: ~0 (if sync replication)
```

### Advantages

✅ **Fastest** — RTO < 1 minute
✅ **Zero human involvement** — True hands-off
✅ **Scales** — Works with 10 failures/day
✅ **Predictable** — Deterministic election
✅ **Transparent** — Application unaware

### Disadvantages

❌ **Complex** — Significant infrastructure
❌ **Requires quorum** — At least 3 nodes
❌ **Network partition risk** — Can cause split-brain if quorum fails
❌ **Harder to debug** — More moving parts
❌ **Over-engineering** — Overkill for many apps

### Fully Automatic Tools

- **PostgreSQL**: Patroni, pg_auto_failover, Stolon
- **MySQL**: Percona XtraDB Cluster (PXC)
- **MongoDB**: Replica sets (built-in)
- **Cloud**: AWS Aurora Multi-AZ, Azure SQL HA

---

## 4️⃣ Split-Brain Prevention

### The Problem

```
Scenario: Primary and replica lose network connection

Before: [Primary] ←→ [Replica]
Network dies: [Primary] ✗ [Replica]

Both think "the other is dead"

Primary continues: "Accept writes, replica will catch up"
Replica continues: "Start accepting writes (I'm primary now)"

Result: TWO primaries with diverging data!

[Primary accepts write A]
[Replica (now thinks it's primary) accepts write B]

Conflict! Which data is correct?
```

### Prevention Strategies

#### Strategy 1: Quorum (Best)

```
Require majority agreement to promote

In 3-node cluster:
Primary + Replica1 + Replica2

Network partition: Primary ↔ [Replica1, Replica2]

Primary side:
  - Can see only itself
  - Needs 2/3 votes to be primary
  - Has only 1 vote → Cannot act as primary
  - BLOCKS writes

Replica side:
  - Sees Replica1 + Replica2
  - Has 2/3 votes → CAN act as primary
  - Promotes Replica1 to primary
  - Continues

Result: Only one side can write!
        No split-brain!
```

#### Strategy 2: Fencing (Force Old Primary Offline)

````
After promoting replica:

1. DNS updated to point to new primary
2. Run command on OLD primary (if accessible):
   - Revoke network access
   - Kill all connections
   - Set read-only = ON
   - Disable replication

Example: PostgreSQL
```sql
-- On old primary
ALTER SYSTEM SET default_transaction_read_only = ON;
SELECT pg_reload_conf();
-- Now it CANNOT accept writes
-- Even if it comes back online
````

#### Strategy 3: Lease-based Leadership (Advanced)

```
New primary must renew lease with quorum every N seconds

If promoted replica can't renew lease:
  - Immediately demotes itself
  - Becomes readonly
  - Rejoins as replica

Example: Kubernetes leader-election
```

### Split-Brain Detection

```sql
-- PostgreSQL: After failover, verify single primary
SELECT datname, usename, application_name, state
FROM pg_stat_replication;

-- Should show:
-- 1. New primary has replicas streaming from it
-- 2. Old primary (if up) shows 0 replicas
-- 3. Check logs for FATAL conflicting errors

-- If you see both accepting writes:
-- CRITICAL: Split-brain occurred!
-- Stop one immediately:
ALTER SYSTEM SET default_transaction_read_only = ON;
SELECT pg_reload_conf();
```

---

## 5️⃣ Choosing Failover Strategy

### Decision Matrix

```
Can tolerate 5-10 min RTO?
├─ YES → Manual failover (simple, tested)
└─ NO → Continue

Can tolerate 2-5 min RTO?
├─ YES → Semi-automated (Patroni, Orchestrator)
└─ NO → Continue

Can tolerate <1 min RTO?
├─ YES → Fully automatic + smart driver
└─ NO → Reconsider architecture

Need < 30 sec RTO?
├─ YES → Multi-region active-active + quorum
└─ Simpler options work
```

### By Organization Size

```
Startup:
├─ Manual failover OK (1-2 person ops team)
├─ Test it quarterly
└─ Acceptable 30min-1hr downtime

Scale-up (10-50 engineers):
├─ Semi-automated (Patroni)
├─ Some on-call involvement
└─ RTO: 5-10 minutes

Enterprise:
├─ Fully automatic + smart drivers
├─ Multi-region setup
└─ RTO: <1 minute

Financial/Healthcare:
├─ Extreme redundancy
├─ Multiple failover layers
└─ RTO: <30 seconds
```

---

## 📋 Testing Failover

### Quarterly Failover Drill (Game Day)

```bash
#!/bin/bash
# Simulate primary failure

set -e

echo "Starting failover game day..."
echo "1. Stopping primary PostgreSQL"
systemctl stop postgresql

echo "2. Waiting 30 seconds for monitoring to detect..."
sleep 30

echo "3. Checking if failover occurred automatically"
psql -h replica-endpoint -c "SELECT pg_is_in_recovery();"
# Should return 'f' (not in recovery = is primary)

echo "4. Attempting write to new primary"
psql -h replica-endpoint -c "INSERT INTO game_day_log VALUES (now(), 'Test write');"

echo "5. Verifying applications reconnected"
curl http://localhost:8080/health
# Should return healthy

echo "Game day successful!"
```

### Chaos Engineering

```python
# chaos.py - Automated failure injection
import random
import subprocess

def cause_network_partition():
    """Block all traffic to primary"""
    subprocess.run([
        "iptables", "-A", "INPUT", "-s", "PRIMARY_IP", "-j", "DROP"
    ])
    print("Network partition created")

def restore_network():
    """Restore connectivity"""
    subprocess.run([
        "iptables", "-D", "INPUT", "-s", "PRIMARY_IP", "-j", "DROP"
    ])
    print("Network restored")

# Run chaos test
cause_network_partition()
time.sleep(60)  # Let failover happen
verify_application_working()
restore_network()
```

---

## ⚠️ Failover Gotchas

### Quorum Lost = All Nodes Go Readonly

```
In 3-node cluster with 2 nodes down:
- Remaining node: has 1/3 votes
- Cannot become primary (needs majority)
- Sets itself readonly
- All writes fail!

Solution:
- Repair/bring back 2nd node quickly
- Have runbook for "emergency override"
```

### Old Primary Comes Back (Split Brain!)

```
Scenario: Primary was just network partition, not crash
Network heals
Old primary comes back up
Thinks it's still primary!
Starts accepting writes!

Solution:
- Quroum-based promotions (automatically demote)
- Fencing (automatically set readonly)
- Monitoring: alert on multiple primaries
```

### Wrong Replica Promoted

```
Scenario: Automatic failover chose lagged replica
Promoted replica missing 5 minutes of data!
Original data lost

Solution:
- Use "maximum_lag_on_failover" setting
- If lag > threshold, don't failover (manual intervention)
- Prefer replica with least lag
```

---

## 📋 Failover Checklist

- [ ] Know your failover strategy (manual/semi/automatic)
- [ ] RTO/RPO documented and communicated to business
- [ ] Promotion script tested monthly
- [ ] DNS failover verified working
- [ ] Split-brain prevention configured
- [ ] Replica catch-up tested (with lag > 0)
- [ ] Monitoring alerts configured for primary failure
- [ ] Oncall runbook prepared and reviewed
- [ ] Team trained on failover procedure (game day annually)
- [ ] Recovery time from failover to full HA restored (documented)
- [ ] Backup strategy independent of failover tested

---

## 🔗 Related Topics

- [Sync vs Async Replication](sync-vs-async.md)
- [Replication Lag Monitoring](replication-lag-monitoring.md)
- [Replication Types](replication-types.md)
- [Backup & Recovery](../02-backup-recovery/)

---

**Key Insight:** Manual failover is slow but safe (for small teams). Semi-automatic is the sweet spot (reduced oncall burden, tested easily). Fully automatic requires quorum + smart drivers (complex but handles massive scale).
