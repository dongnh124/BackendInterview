# Restore Testing

A backup is only as good as your ability to restore it. Restore testing proves your backups actually work and helps identify gaps before a real disaster.

## 🎯 Why Restore Testing Matters

### The Problem

```
Many organizations have backups but can't restore them:
- Backup software license expired
- Backup file corrupted
- Incompatible OS/database version
- Insufficient disk space
- Missing documentation
- Team doesn't know the procedure

Result: Disaster strikes, and backups are useless
```

### The Cost

```
Failed restore attempt during disaster = Business loss
vs.
Regular restore tests = Minimal cost
```

---

## ✅ Essential Restore Test Checks

### 1. Backup File Integrity

**Before attempting restore:**

- [ ] Check file size is reasonable (not 0 bytes or suspiciously small)
- [ ] Verify checksum if available:

  ```bash
  # PostgreSQL logical backup
  sha256sum backup.sql

  # MySQL dump
  md5sum backup.sql
  ```

- [ ] Verify backup is readable and not corrupted:
  ```bash
  # Test file accessibility
  file backup.sql
  head -n 10 backup.sql  # Should show SQL statements
  tail -n 10 backup.sql  # Should show valid ending
  ```
- [ ] Verify backup age (is it recent?)
- [ ] Verify backup location accessible (especially for remote/cloud backups)

---

### 2. Restore to Isolated Environment

**NEVER restore to production!**

#### Setup Requirements

- [ ] Separate instance (never overwrite production)
- [ ] Same OS (Linux vs. Windows)
- [ ] Same database version or compatible
- [ ] Similar configuration (memory, disk, CPU)
- [ ] Labeled as TEST/STAGING (prevent accidents)
- [ ] Network isolated if possible

#### Example: Docker Test Environment

```bash
# Spin up isolated test database
docker run -d \
  --name test-restore \
  -e POSTGRES_PASSWORD=test \
  -v /path/to/backup:/backup \
  postgres:14

# Restore from backup
docker exec test-restore \
  psql -U postgres -f /backup/backup.sql

# After testing, cleanup
docker rm -f test-restore
```

---

### 3. Functional Validation

#### 3a. Database Online and Responsive

```sql
-- PostgreSQL
SELECT version();
SELECT current_database();

-- MySQL
SELECT @@version;
SELECT database();
```

#### 3b. Table Structure Verification

```sql
-- PostgreSQL: Count tables
SELECT COUNT(*) as table_count
FROM information_schema.tables
WHERE table_schema NOT IN ('pg_catalog', 'information_schema');

-- MySQL: Count tables
SELECT COUNT(*) as table_count
FROM information_schema.tables
WHERE table_schema NOT IN ('information_schema', 'mysql');
```

#### 3c. Row Count Verification

```sql
-- PostgreSQL: Total rows across all tables
SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;

-- MySQL: Total rows
SELECT TABLE_NAME, TABLE_ROWS, DATA_LENGTH
FROM information_schema.TABLES
WHERE TABLE_SCHEMA NOT IN ('information_schema', 'mysql')
ORDER BY TABLE_ROWS DESC;
```

#### 3d. Checksums/Data Integrity

```sql
-- PostgreSQL: Checksum all tables
SELECT schemaname, tablename,
       (SELECT COUNT(*) FROM (SELECT * FROM ONLY schemaname.tablename) x) as row_count
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema');

-- MySQL: Sample query across tables
SELECT COUNT(*) FROM users;
SELECT COUNT(*) FROM orders;
SELECT COUNT(*) FROM products;
```

#### 3e. Sample Data Spot Checks

```sql
-- Verify key columns exist and have data
SELECT * FROM users LIMIT 1;
SELECT * FROM orders LIMIT 1;
SELECT * FROM products LIMIT 1;

-- Verify data types are correct
DESCRIBE users;
DESCRIBE orders;
```

---

### 4. Application Testing

#### 4a. Application Can Connect

```bash
# From application code
python -c "import psycopg2; conn = psycopg2.connect('dbname=restored_db'); print('Success')"

# Or test application startup
docker run --link test-restore app:latest  # Verify app connects
```

#### 4b. Known Queries Execute

```sql
-- Run queries that application typically runs
SELECT * FROM active_users WHERE status = 'active';
SELECT SUM(total) FROM orders WHERE created_at > NOW() - INTERVAL '30 days';
SELECT COUNT(*) FROM products WHERE price > 100;
```

#### 4c. Results Match Expectations

```sql
-- Compare against production query results
-- Use same query on:
-- 1. Original production database
-- 2. Restored test database
-- Results should match exactly
```

---

### 5. Document Findings

**Create restore test report:**

```markdown
# Restore Test Report - [Date]

## Backup Information

- Backup File: backup_2024_04_20.sql
- Backup Date: 2024-04-20 02:00 UTC
- Backup Size: 125GB
- Backup Checksum: abc123def456

## Restore Environment

- OS: Ubuntu 20.04
- Database: PostgreSQL 14.5
- Instance: test-restore
- Date Restored: 2024-04-21

## Restore Process

- Restore Start: 10:00 UTC
- Restore End: 10:15 UTC
- Restore Duration: 15 minutes

## Validation Results

- Database Online: ✅ Yes
- Table Count: ✅ 247 tables (matches production)
- Total Rows: ✅ 125M (matches production)
- Sample Queries: ✅ All returned expected results

## Application Testing

- Application Startup: ✅ Success
- Database Connection: ✅ Success
- Query Execution: ✅ All test queries passed
- Performance: ✅ Acceptable

## Issues Found

- None

## Conclusion

✅ Backup is valid and restorable. RTO estimate: 15 minutes for similar-sized hardware.

## Signed By

- Tested By: [Name]
- Date: 2024-04-21
```

---

## 🤖 Automated Restore Test Script

### PostgreSQL Example

```bash
#!/bin/bash
# automated_restore_test.sh
# Purpose: Automate restore testing, run via cron weekly

BACKUP_FILE=$1
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
TEST_DB="restore_test_${BACKUP_DATE}"
LOG_FILE="/var/log/restore_test_${BACKUP_DATE}.log"

echo "Starting restore test at $(date)" | tee -a $LOG_FILE

# Check backup file exists
if [ ! -f "$BACKUP_FILE" ]; then
    echo "ERROR: Backup file not found: $BACKUP_FILE" | tee -a $LOG_FILE
    exit 1
fi

# Check backup file size
SIZE=$(stat -f%z "$BACKUP_FILE" 2>/dev/null || stat -c%s "$BACKUP_FILE")
if [ "$SIZE" -lt 1000000 ]; then
    echo "WARNING: Backup file suspiciously small: ${SIZE} bytes" | tee -a $LOG_FILE
fi

echo "Backup file: $BACKUP_FILE ($(numfmt --to=iec $SIZE 2>/dev/null || echo $SIZE))" | tee -a $LOG_FILE

# Restore database
echo "Creating test database: $TEST_DB" | tee -a $LOG_FILE
createdb -U postgres $TEST_DB || exit 1

echo "Restoring backup..." | tee -a $LOG_FILE
START_TIME=$(date +%s)
psql -U postgres -d $TEST_DB -f "$BACKUP_FILE" >> $LOG_FILE 2>&1
RESTORE_RESULT=$?
END_TIME=$(date +%s)
DURATION=$((END_TIME - START_TIME))

if [ $RESTORE_RESULT -ne 0 ]; then
    echo "ERROR: Restore failed" | tee -a $LOG_FILE
    dropdb -U postgres $TEST_DB
    exit 1
fi

echo "Restore completed in ${DURATION} seconds" | tee -a $LOG_FILE

# Validate restored database
echo "Running validation checks..." | tee -a $LOG_FILE

# Check table count
TABLE_COUNT=$(psql -U postgres -d $TEST_DB -t -c "
    SELECT COUNT(*) FROM information_schema.tables
    WHERE table_schema NOT IN ('pg_catalog', 'information_schema')"
)
echo "  Tables: $TABLE_COUNT" | tee -a $LOG_FILE

# Check row counts
echo "  Table sizes:" | tee -a $LOG_FILE
psql -U postgres -d $TEST_DB -t -c "
    SELECT tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename))
    FROM pg_tables
    WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
    ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
    LIMIT 10" >> $LOG_FILE

# Run sample query
SAMPLE_ROWS=$(psql -U postgres -d $TEST_DB -t -c "SELECT COUNT(*) FROM (SELECT * FROM information_schema.tables LIMIT 1) x")
echo "  Sample validation: $SAMPLE_ROWS" | tee -a $LOG_FILE

# Cleanup
echo "Cleaning up test database..." | tee -a $LOG_FILE
dropdb -U postgres $TEST_DB

echo "Restore test completed successfully at $(date)" | tee -a $LOG_FILE
```

### Setup Automated Testing with Cron

```bash
# Weekly restore test every Sunday at 3 AM
0 3 * * 0 /scripts/automated_restore_test.sh /backups/latest_backup.sql

# Email results
0 3 * * 0 /scripts/automated_restore_test.sh /backups/latest_backup.sql | mail -s "Restore Test Results" dba@company.com
```

---

## 📊 Restore Test Frequency

### Minimum Recommendations

| Backup Type     | Test Frequency | Reason                         |
| --------------- | -------------- | ------------------------------ |
| Full backup     | Monthly        | Large, complete restore        |
| Incremental     | Quarterly      | Complex restore chain          |
| Logical dump    | Monthly        | May have portability issues    |
| Physical backup | Weekly         | PITR requires special handling |
| Cloud backup    | Monthly        | Access/permissions may change  |

---

## 🚨 Common Issues Found During Testing

### Issue: Restore Fails Partway Through

**Problem:** Backup corrupted, network interrupted, disk full

**Solution:**

- Verify backup file integrity
- Restore on system with sufficient disk space
- Check backup software logs

### Issue: Restored Database Won't Start

**Problem:** Incompatible version, corrupted data files, crash recovery needed

**Solution:**

- Verify database version matches
- Run crash recovery
- Check system logs for errors

### Issue: Application Can't Connect

**Problem:** Roles/permissions not restored, network isolation issue

**Solution:**

- Verify roles and grants restored
- Test database connectivity
- Check firewall rules

### Issue: Data Doesn't Match Production

**Problem:** Incomplete backup, concurrent writes during backup, backup lag

**Solution:**

- Verify backup strategy captures all data
- Understand expected lag
- Run comparison queries

---

## ✅ Restore Testing Checklist

- [ ] Backup file exists and is readable
- [ ] Backup file size is reasonable
- [ ] Backup file checksum verified
- [ ] Test environment created (not production!)
- [ ] Test environment matches production configuration
- [ ] Restore completed successfully
- [ ] Restore time documented
- [ ] Database online and responsive
- [ ] Table counts match source
- [ ] Row counts match source
- [ ] Sample queries return expected results
- [ ] Application can connect
- [ ] Test queries execute successfully
- [ ] Results match production expectations
- [ ] Test report generated and signed
- [ ] Cleanup completed

---

## 🔗 Related Topics

- [RPO & RTO Explained](rpo-rto-explained.md) — Understand your recovery targets
- [Backup Strategies](backup-strategies.md) — Design backups for testing
- [PostgreSQL Backup](postgresql-backup.md) — PostgreSQL-specific testing
- [MySQL Backup](mysql-backup.md) — MySQL-specific testing

---

**Key Insight:** A backup you haven't tested is just hope. Schedule restore tests regularly, automate them when possible, and treat them with the same seriousness as actual disaster recovery.
