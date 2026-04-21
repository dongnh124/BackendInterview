# Backup & Disaster Recovery

Critical knowledge for database reliability, recovery, and compliance.

## 📚 Core Topics

1. **RPO & RTO** — Business requirements for recovery
2. **Backup Types** — Full, incremental, differential, logical, physical
3. **Backup Strategy** — Choosing the right approach for your database
4. **Restore Testing** — Verifying backups actually work
5. **PITR (Point-in-Time Recovery)** — Recovering to specific moments
6. **WAL Archiving** — PostgreSQL's durability mechanism
7. **Backup Encryption** — Securing sensitive backups
8. **Cloud Backup** — RDS, Cloud SQL managed backups

## 🎯 RPO vs RTO - The Foundation

### RPO (Recovery Point Objective)

```
"Maximum acceptable data loss"
Measured in TIME: minutes/hours/days

Example: RPO = 1 hour
→ Can lose up to 1 hour of transactions
→ Need backups at least hourly
→ Or continuous replication
```

### RTO (Recovery Time Objective)

```
"Maximum acceptable downtime"
Measured in TIME to restore service

Example: RTO = 30 minutes
→ Database must be online within 30 min after failure
→ Need automated failover OR fast restore procedure
→ Requires backup accessible and tested
```

### Business Impact

```
RPO ← DATA LOSS     vs     UPTIME → RTO
↑ Tight RPO = More expensive (continuous backup/replication)
                    ↑ Tight RTO = More expensive (HA/failover automation)

Business decides acceptable trade-off
```

## 🛠️ Backup Strategy

### Full Backup

```
Complete database snapshot at point-in-time
Pros: Complete recovery, standalone
Cons: Large size, long time, resource intensive

Use when:
- First backup
- Before major changes
- Baseline for incremental
```

### Incremental Backup

```
Only blocks changed since last backup
Pros: Small size, fast
Cons: Complex restore (need full + all incrementals), dependent chain

Use when:
- Database large and frequently changing
- Storage/bandwidth constraints
```

### Differential Backup

```
All changes since last FULL backup
Pros: Faster restore than incremental (only need full + last diff)
Cons: Larger than incremental

Use when:
- Need better restore speed than incremental
- SQL Server (native support)
```

### Logical Backup (pg_dump, mysqldump)

```
SQL statements to recreate database
Pros: Portable, human-readable, can exclude tables, database-agnostic
Cons: Slower restore, RPO depends on dump schedule

Use when:
- Migrating between versions
- Portability needed
- Subset export required
```

### Physical Backup (snapshot, WAL)

```
Byte-for-byte copy of data files
Pros: Faster restore, can do PITR, replica setup
Cons: Engine-specific, needs crash recovery

Use when:
- Large databases
- Need fast RTO
- PITR required
```

## 📊 Recommended Strategy (OLTP)

```
┌─────────────────────────────────────┐
│ FULL BACKUP (Weekly Sunday 2 AM)   │  RPO: 7 days worst case
├─────────────────────────────────────┤
│ DIFF BACKUPS (Daily 3 AM)          │  RPO: 1 day worst case
├─────────────────────────────────────┤
│ TRANSACTION LOG BACKUP (Every 15 min) │ RPO: 15 minutes
├─────────────────────────────────────┤
│ Offsite Copy (Daily)               │  Protection against site loss
├─────────────────────────────────────┤
│ RESTORE TEST (Monthly)             │  Verify backups work
└─────────────────────────────────────┘

Result: RPO ≤ 15 minutes, RTO ≤ 1 hour for most failures
```

## 🔄 PostgreSQL Specific

### WAL Archiving (for PITR)

```sql
-- Setup WAL archiving in postgresql.conf
wal_level = replica
max_wal_senders = 3
wal_keep_size = 10GB

-- Check WAL
SELECT * FROM pg_ls_waldir();
SELECT pg_current_wal_lsn();
```

### Logical Backup

```bash
# Full logical backup
pg_dump -U postgres mydb > backup.sql

# With compression
pg_dump -U postgres -F c mydb > backup.dump

# Specific tables only
pg_dump -U postgres -t table1 -t table2 mydb > subset.sql
```

### PITR Restore

```sql
-- Restore to specific point in time
SELECT pg_wal_replay_pause();
-- When ready to resume
SELECT pg_wal_replay_resume();
```

## 🔄 MySQL Specific

### Binary Log Backups

```bash
# MySQL continuous backup with binary logs
-- In my.cnf
binlog_format = ROW
expire_logs_days = 10

# Show binary logs
SHOW BINARY LOGS;

# Restore with binary logs for point-in-time
```

### Logical Backup

```bash
# Full backup
mysqldump -u root -p mydb > backup.sql

# Specific tables
mysqldump -u root -p mydb table1 table2 > tables.sql

# With compression
mysqldump -u root -p mydb | gzip > backup.sql.gz
```

## ✅ Restore Testing

### Essential Checks

```
1. Backup file integrity
   - Check file size is reasonable
   - Verify checksums if available

2. Restore to isolated environment
   - Separate instance (never overwrite prod!)
   - Same OS, version, configuration

3. Functional validation
   - Database online and responsive
   - Table counts match source
   - Checksums/row counts verified
   - Sample data spot-checks

4. Application testing
   - App can connect and query
   - Known queries execute
   - Results match expectations

5. Document findings
   - Restore time in each phase
   - Data validation results
   - Any issues encountered
   - Proof for audit/compliance
```

### Automated Restore Test Script

```bash
#!/bin/bash
# Restore backup and validate

BACKUP_FILE=$1
TEST_DB="test_restore_$(date +%s)"

echo "Starting restore test at $(date)"

# Restore
psql -U postgres -f $BACKUP_FILE

# Validate
echo "Connecting to $TEST_DB..."
psql -U postgres -d $TEST_DB -c "SELECT COUNT(*) as table_count FROM information_schema.tables;"

echo "Checking row counts..."
psql -U postgres -d $TEST_DB -c "
  SELECT tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
  FROM pg_tables WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
  ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;"

# Cleanup
dropdb -U postgres $TEST_DB

echo "Restore test completed at $(date)"
```

## 🔐 Security Checklist

- [ ] Backups encrypted in transit (TLS/SSH)
- [ ] Backups encrypted at rest (AES-256)
- [ ] Offsite copy in separate account/region
- [ ] Immutable storage (WORM, object lock)
- [ ] Access logging enabled
- [ ] Minimal permissions for backup user
- [ ] Backup encryption key rotation scheduled
- [ ] Restore access restricted to DBA team

## 📋 Interview Questions

1. **Explain your backup strategy for a 500GB OLTP database with 99.99% uptime requirement**
   - Calculate appropriate RPO (suggest 15-30 min)
   - Calculate appropriate RTO (suggest < 1 hour)
   - Design: Full weekly + daily differential + hourly transaction log
   - Include restore testing

2. **What's the difference between backup and replication?**
   - Backup: Point-in-time copy, good for old data recovery, ransomware protection
   - Replication: Live copy, fast failover, read scaling
   - Use both: replication for HA, backup for DR

3. **How do you verify a backup is good?**
   - Restore to test environment
   - Verify checksums
   - Test application connectivity
   - Document restore time

4. **If you have 1TB database, what's best backup method?**
   - Physical snapshot (fast restore) + WAL archiving (PITR)
   - Or incremental backups if snapshots not available
   - Logical dump too large for frequent full backups

## 🎯 Key Metrics

| Metric               | Target         | How to Achieve                    |
| -------------------- | -------------- | --------------------------------- |
| RPO                  | ≤ 1 hour       | Hourly backups or replication     |
| RTO                  | ≤ 30 min       | Automated restore, tested runbook |
| Backup Size          | < 50% raw data | Compression, incremental, dedup   |
| Restore Time         | < RTO          | Test regularly, validate          |
| Restore Success Rate | 100%           | Test every month minimum          |

## 📖 Advanced Topics

- [ ] Incremental forever backups
- [ ] ZFFS snapshots & cloning
- [ ] Backup deduplication
- [ ] Cross-region backup replication
- [ ] Ransomware-resistant backup
- [ ] Cost optimization (tiered storage, archival)

## ✅ Practical Exercises

1. **Design Backup for Your Database**
   - Define RPO based on business need
   - Define RTO based on criticality
   - Choose backup types
   - Schedule backups
   - Document restore procedure

2. **Set Up Automated Restore Test**
   - Create backup script
   - Create restore script
   - Create validation script
   - Schedule to run weekly

3. **Calculate Backup Capacity**
   - Current database size
   - Growth rate
   - Retention period (compliance)
   - Total storage needed
   - Cost estimation

## 🔗 Related

- [High Availability](../03-ha-replication/)
- [Performance Tuning](../04-performance-tuning/)
- [PostgreSQL Guide](../07-platform-guides/postgresql.md)
- [MySQL Guide](../07-platform-guides/mysql.md)

---

**Key Insight:** Backup strategy isn't just about having files—it's about ensuring you can recover within your business's acceptable RPO/RTO, and you prove it works with regular testing.
