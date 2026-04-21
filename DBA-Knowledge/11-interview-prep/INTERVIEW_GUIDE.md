# DBA Interview Preparation

Comprehensive guide to ace your Database Administrator interviews.

## 📊 Interview Format Overview

| Type              | Time      | Questions | Focus                            |
| ----------------- | --------- | --------- | -------------------------------- |
| Phone Screen      | 30-45 min | 2-3       | Background, motivation, overview |
| Technical Round 1 | 60 min    | 4-5       | Fundamentals, scenarios          |
| Technical Round 2 | 90 min    | 2-3       | Deep dive + incident story       |
| System Design     | 90 min    | 1-2       | Design complex system            |
| Culture/Fit       | 30 min    | -         | Values, teamwork, growth         |

## 🎯 Top 20 DBA Interview Questions

### Tier 1: Fundamentals (Must Know)

#### 1. **Explain RPO and RTO. How would you design a backup strategy for a mission-critical database?**

**What they're looking for:**

- Understanding of business requirements
- Ability to translate requirements into technical decisions
- Knowledge of backup types and trade-offs
- Hands-on experience with backup tools

**Structure your answer:**

```
1. Define RPO & RTO
   RPO = maximum acceptable data loss (time)
   RTO = maximum acceptable downtime (time)

2. Business context
   "For mission-critical systems:"
   - RPO typically < 1 hour (often 15-30 min)
   - RTO typically < 30 minutes

3. Design strategy
   - Full backup weekly (baseline)
   - Differential/incremental daily
   - Transaction log backups every 15 minutes
   - WAL archiving for PITR
   - Offsite copy (different region)

4. Implementation
   - Use managed backup if available (RDS, Cloud SQL)
   - Or self-managed with pgBackRest, Barman
   - Automate restore tests monthly

5. Verification
   - Test restore from backup quarterly
   - Verify checksums
   - Document RTO in real conditions
```

**Follow-up scenarios:**

- "What if RPO was 5 minutes instead of 1 hour?" → Synchronous replication
- "What if database was 500GB?" → Physical backups + WAL
- "What about compliance requirements?" → Immutable offsite, encryption

---

#### 2. **Describe a database performance issue you troubleshot. What metrics did you check?**

**This is your incident story (use STAR format)**

**S - Situation:**
"I had a PostgreSQL database supporting an e-commerce platform. During a peak sale event (high traffic), customers reported slow page loads and timeouts."

**T - Task:**
"As the DBA on-call, I needed to identify the root cause and restore performance within 30 minutes."

**A - Action:**
"My troubleshooting process:

1. Connected to database and checked top wait events
   - High IO wait on specific table

2. Analyzed slow queries
   - SELECT on 'orders' table scanning 10M rows
   - Missing index on filter column

3. Checked query plan
   - EXPLAIN ANALYZE showed sequential scan
   - Should use index but not created yet

4. Temporary fix
   - Added index on order_status column
   - Rebuild statistics
   - Tested query (1.5s → 50ms)

5. Long-term fix
   - Analyzed access patterns
   - Added composite index (user_id, order_status)
   - Ensured replica had same indexes

6. Monitoring
   - Added alert for sequential scans on large tables
   - Set threshold for query duration > 5s"

**R - Result:**
"Performance restored within 10 minutes. Response time went from 8s to 200ms. Prevention: Added automated index recommendations to deployment process. No performance regression in following 3 months."

**Why this works:**

- Shows problem-solving methodology
- Demonstrates specific tools/commands knowledge
- Shows preventive thinking
- Realistic timeline and resolution

---

#### 3. **What's the difference between blocking and deadlock? How do you minimize each?**

**Blocking:**

```
Transaction A locks row 1
Transaction B tries to access row 1
B has to WAIT (block) until A releases lock

Duration: Short (seconds) = normal
Duration: Long (minutes) = problem
```

**Causes:**

- Long-running transaction holding locks
- Missing indexes causing lock escalation
- Application bugs (connections not returned)

**Minimize blocking:**

- Keep transactions short
- Create appropriate indexes
- Use lower isolation level if acceptable
- Monitor and kill long-running queries

**Deadlock:**

```
Transaction A locks row 1, waits for row 2
Transaction B locks row 2, waits for row 1

Circular wait → DEADLOCK
→ Database detects and kills one transaction (victim)
```

**Causes:**

- Inconsistent lock ordering in application
- Complex transaction logic
- High concurrency

**Minimize deadlock:**

- Lock resources in same order always
- Keep transactions small
- Add appropriate indexes
- Implement retry logic (exponential backoff)
- Use lower isolation level if possible

---

#### 4. **Design HA architecture for a database with 99.99% uptime requirement**

**Uptime calculation:**

```
99.99% = 52.6 minutes downtime per year
     = ~8.64 seconds per day

This is VERY tight → requires automated failover
```

**Architecture:**

```
┌─────────────────────────────────────┐
│     Load Balancer (DNS/VIP)        │
└───────────────┬─────────────────────┘
                │
    ┌───────────┴───────────┐
    ▼                       ▼
[PRIMARY]              [REPLICA]
PostgreSQL 14         PostgreSQL 14
(Multi-AZ)           (Same AZ initially)
    │                      │
    └──────Streaming───────┘
           Replication

┌─────────────────────────────────────┐
│   Automated Failover (Patroni)      │
│   - Health check every 10s          │
│   - Promote replica if primary down │
│   - Update DNS/VIP                  │
│   - Prevent split-brain (etcd)      │
└─────────────────────────────────────┘

Additional:
- Backup to S3 (independent of replication)
- CloudWatch monitoring + alerts
- RTO < 1 minute (automatic failover)
- RPO < 15 seconds (sync replication)
```

**Key decisions explained:**

- **Synchronous replication** → RPO = 0 (no data loss)
- **Automatic failover** → RTO < 1 min (tight uptime budget)
- **Quorum + etcd** → Prevent split-brain
- **Independent backup** → Protection from application errors

---

### Tier 2: Intermediate Scenarios

#### 5. **Your database size is growing 20% per month. How do you plan capacity?**

**Trend analysis:**

```
Current size: 200GB
Growth rate: 20% per month

6 months: 200GB × 1.2^6 = 579GB
12 months: 200GB × 1.2^12 = 1.5TB
```

**Capacity planning factors:**

1. **Data growth** (as above)
2. **Index overhead** (typically 10-30% of data)
3. **WAL/logs** (consider log archiving strategy)
4. **Temporary space** (maintenance, temp tables)
5. **Replicas** (multiply by number of replicas)
6. **Backups** (retention period × daily backup size)
7. **Performance** (headroom for spikes)

**Scaling decisions:**

```
Vertical scaling (bigger box):
- Pros: Simple, no application change
- Cons: Gets expensive, single point of failure risk

Horizontal scaling (sharding):
- Pros: Unlimited scale, load distribution
- Cons: Complex, application-level changes, operational burden

Rule: Vertical first (easier), sharding when necessary (> 2TB typically)
```

**Action steps:**

1. Script trend analysis (automated weekly)
2. Set alert when reaching 80% capacity
3. Plan upgrade 3 months before hitting limit
4. Test upgrade in staging
5. Schedule maintenance window if needed

---

#### 6. **Walk me through a schema migration for production (zero-downtime)**

**The challenge:**
Add a new column to heavily-used table without downtime.

**Expand-Contract Pattern:**

```
PHASE 1: EXPAND (Deploy)
- Add new nullable column
- Add default value or NULL
- Existing app still uses old column
- Deploy code that ignores new column

PHASE 2: MIGRATE (Parallel)
- Data backfill in batches (avoid locks)
  UPDATE users SET new_column = old_column
  WHERE id BETWEEN 1 AND 100000;

- Dual-write: Code writes to BOTH columns
- Monitor for consistency issues
- Continue until fully backfilled

PHASE 3: CONTRACT (Cleanup)
- Change code to read from NEW column
- Deploy to all instances
- Wait for traffic to stabilize
- Monitor errors (wrong column access)
- Remove old column (after few days/weeks)
```

**Why this works:**

- No table locks
- Rollback possible at each phase
- App can recover if issues found
- Gradual, safe transition

**Tools to use:**

- Flyway / Liquibase (versioned migrations)
- Feature flags (rollback deployment without schema rollback)
- pg_partman or Oracle partitioning for huge tables

---

### Tier 3: Real-world Scenarios

#### 7. **You have a 48-hour cutover window for a major migration. What's your pre-cutover checklist?**

**Pre-cutover Checklist:**

```
72 Hours Before:
☐ Final full backup of source database
☐ Test restore to staging environment
☐ Run migration script on staging database
☐ Validate row counts match exactly
☐ Run application smoke tests
☐ Brief all teams on plan and runbook

24 Hours Before:
☐ Verify backup encryption keys
☐ Confirm external dependencies (no scheduled maintenance)
☐ Notify customers of maintenance window
☐ Brief incident commander and support team
☐ Set up monitoring dashboards
☐ Have rollback plan documented

4 Hours Before:
☐ Drain connection pools (graceful shutdown)
☐ Final checkpoint - no new writes to source
☐ Prepare communication channels
☐ Position team members

During Migration (Phase by phase):
☐ Stop application writes
☐ Final backup before cutover
☐ Run forward migration script
☐ Validate data integrity:
   ✓ Row count per table
   ✓ Checksums on key tables
   ✓ Referential integrity checks
   ✓ Index exists and valid
☐ Application connectivity test
☐ Run business logic smoke tests
☐ Prod environment application start
☐ Monitor for errors (first 30 minutes critical)

Post-cutover:
☐ Continuous monitoring (1 hour)
☐ Performance baseline (queries, latency)
☐ Backup after cutover
☐ Close communication channels
☐ Schedule post-mortem (next day)
```

**Runbook format:**

- Script each step when possible (automation reduces errors)
- Estimated time for each phase
- Success criteria before proceeding
- Rollback procedure if issues found
- Communication template for each outcome

---

#### 8. **What security measures do you implement for database access?**

**Layered Security:**

````
Layer 1: NETWORK
- Private subnet (no public IP)
- Security group limited to app IPs only
- VPN for DBA access
- TLS 1.2+ for all connections

Layer 2: AUTHENTICATION
- No shared passwords
- IAM/Active Directory for authentication
- Separate service account per application
- MFA for DBA console access
- SSH key rotation (90 days)

Layer 3: AUTHORIZATION
- Principle of least privilege
- Role-based access control (RBAC)
- Schema separation per application
- No superuser for applications
- Read-only role for reports

Layer 4: ENCRYPTION
- At-rest: AES-256 for data files
- In-transit: TLS for connections
- Transparent Data Encryption (TDE) if available
- Keystore/HSM for key management

Layer 5: AUDIT
- DDL change logging (who changed schema, when)
- Failed login attempts (5 strikes = lockout)
- Successful privileged commands logged
- Query auditing for PII/sensitive tables
- Logs shipped to SIEM

Example PostgreSQL setup:
```sql
-- Create application role (no superuser)
CREATE ROLE app_user WITH LOGIN PASSWORD 'secret';

-- Grant minimal needed permissions
GRANT CONNECT ON DATABASE mydb TO app_user;
GRANT USAGE ON SCHEMA public TO app_user;
GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO app_user;

-- Revoke dangerous operations
REVOKE ALL ON DATABASE template1 FROM app_user;

-- Enable auditing
CREATE EXTENSION IF NOT EXISTS pgaudit;
SET pgaudit.log = 'DDL, ROLE';
````

---

### Tier 4: Deep Technical

#### 9. **Explain transaction isolation levels and their performance implications**

**The Spectrum:**

```
Isolation Level         Problems?           Performance    Use Case
───────────────────────────────────────────────────────────────────
Read Uncommitted     ✗ Dirty reads         Very fast      Not recommended
Read Committed       ✗ Non-repeatable      Fast           Default PostgreSQL/SQL Server
Repeatable Read      ✗ Phantom reads       Medium         MySQL default
Serializable         ✗ None                Slow           Critical transactions
```

**Explanation with example:**

```sql
-- Example: Transaction A and B both running

ISOLATION LEVEL: Read Uncommitted
A: UPDATE balance SET = 500 (not yet committed)
B: SELECT balance → sees 500 (dirty read!) ❌

ISOLATION LEVEL: Read Committed (default)
A: UPDATE balance SET = 500
COMMIT
B: SELECT balance → sees 500 ✓

ISOLATION LEVEL: Repeatable Read
A: SELECT balance → 100
C: UPDATE balance SET = 200
C: COMMIT
A: SELECT balance again → still sees 100 (repeatable) ✓
→ But if A does INSERT into related table, might see phantom rows

ISOLATION LEVEL: Serializable
A and B run sequentially as if no concurrency
No anomalies but worst performance
```

**Performance implications:**

```
Lower isolation = Less locking = Better concurrency/throughput
Higher isolation = More locking = Fewer anomalies but slower

Most apps use Read Committed:
- Provides good safety (no dirty reads)
- Good performance (minimal locks)
- Anomalies rare in practice
- Application-level checks mitigate issues
```

---

## 🌟 Interview Tips

### Before Interview

```
1. PREPARE STORIES (not just knowledge)
   - 2-3 incident stories (STAR format)
   - Have metrics ready (resolved in X minutes, prevented Y problems)

2. RESEARCH THE COMPANY
   - What databases do they use? (Check job description)
   - Scale (number of users, data volume)
   - Infrastructure (on-prem, cloud, hybrid)

3. KNOW YOUR RESUME
   - Be ready to discuss any technology listed
   - Have examples of impact you made
   - Know what you want to do differently this time

4. PREPARE QUESTIONS
   - What's the typical on-call rotation?
   - How many databases in production? What's the scale?
   - What's your current biggest database pain point?
   - What tools do you use for monitoring?
```

### During Interview

```
1. CLARIFY BEFORE SOLVING
   "Let me make sure I understand the constraint..."
   - Clarify the problem scope
   - Ask about constraints
   - Understand trade-offs required

2. THINK OUT LOUD
   "Here's my approach, let me walk through it..."
   - Show reasoning, not just answers
   - Explain why you chose this approach
   - Point out trade-offs

3. ADMIT WHAT YOU DON'T KNOW
   "I haven't worked with Aurora specifically, but here's what I'd do..."
   - Honesty is valued over bluffing
   - Show you can learn and figure things out

4. USE EXAMPLES
   "In my previous role at [company]..."
   - Specific examples beat generic answers
   - Numbers/metrics make impact real

5. SPEAK TO THE INTERVIEWER'S LEVEL
   - Manager: Talk about impact, business context
   - Senior DBA: Technical depth, optimization
   - Engineer: Implementation details, trade-offs
```

### After Interview

```
1. SEND THANK YOU EMAIL within 24 hours
   - Reference specific conversation points
   - Reiterate interest
   - Ask about timeline

2. REFLECT ON GAPS
   - What questions felt weak?
   - What should you study more?
   - Practice for next round

3. PREPARE FOR NEXT ROUND
   - Ask what to expect
   - Study those areas
   - Prepare new stories
```

## 📚 Study Materials

### To Read Before Interview

- [ ] Database Internals (Ch 1-3)
- [ ] PostgreSQL Documentation (Backup/Recovery, Replication)
- [ ] MySQL High Availability
- [ ] Your target company's tech blog

### Hands-on Practice

- [ ] Set up PostgreSQL replication locally
- [ ] Practice restore from backup
- [ ] Analyze slow query with EXPLAIN
- [ ] Create migration script with zero downtime
- [ ] Set up monitoring dashboard

### Common Tools to Know

- **PostgreSQL:** pg_stat_statements, EXPLAIN ANALYZE, pg_dump, WAL archiving
- **MySQL:** slow query log, SHOW ENGINE INNODB STATUS, mysqldump, replication
- **General:** pgBouncer, pgBackRest, Barman, Prometheus, Grafana

## 🎯 90-Day Master Plan

| Week  | Focus              | Deliverable                                    |
| ----- | ------------------ | ---------------------------------------------- |
| 1-2   | Fundamentals       | Can explain ACID, isolation levels from memory |
| 3-4   | Backup & HA        | Design backup strategy for given SLA           |
| 5-6   | Performance        | Troubleshoot slow query end-to-end             |
| 7-8   | Platform deep dive | Know one platform really well                  |
| 9-10  | Practice stories   | Polish 3 interview stories (STAR format)       |
| 11-12 | Mock interviews    | Practice with someone                          |

---

**Remember:** Interviewers want to know if you can:

1. **Think systematically** about problems
2. **Consider trade-offs** not just one solution
3. **Work under pressure** and communicate
4. **Learn and adapt** to new technologies
5. **Protect data** (most important job of DBA)

Good luck! 🚀
