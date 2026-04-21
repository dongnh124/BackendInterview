# MySQL Backup & Recovery

MySQL-specific backup strategies, tools, and best practices.

## 📝 Binary Log Setup

Binary logs record all database changes and are essential for replication and point-in-time recovery.

### Configure Binary Logging

Edit `/etc/mysql/my.cnf`:

```ini
[mysqld]
# Enable binary logging
log_bin = /var/log/mysql/mysql-bin
binlog_format = ROW              # ROW, STATEMENT, or MIXED
max_binlog_size = 100M           # Rotate every 100MB
expire_logs_days = 7             # Keep 7 days of logs
binlog_expire_logs_seconds = 604800  # Keep 7 days (seconds)
server_id = 1                    # For replication
```

### Verify Binary Logging

```sql
-- Check if binary logging enabled
SHOW VARIABLES LIKE 'log_bin%';

-- List binary logs
SHOW BINARY LOGS;

-- Show current binary log status
SHOW MASTER STATUS;

-- Check binlog format
SHOW VARIABLES LIKE 'binlog_format';
```

### Binlog Format Comparison

| Format    | What's Logged            | Pros                | Cons                     |
| --------- | ------------------------ | ------------------- | ------------------------ |
| ROW       | Changed rows as binary   | Safe, deterministic | Larger log files         |
| STATEMENT | SQL statements           | Smaller log files   | Non-deterministic issues |
| MIXED     | Mix of row and statement | Balanced            | Complexity               |

**Recommendation:** Use `ROW` format for reliability

---

## 💾 Logical Backup (mysqldump)

SQL statements to recreate database. Good for smaller databases, portability, and selective backups.

### Full Database Backup

```bash
# Basic backup (plaintext SQL)
mysqldump -u root -p mydb > backup.sql

# With all databases
mysqldump -u root -p --all-databases > all_backups.sql

# With compression
mysqldump -u root -p mydb | gzip > backup.sql.gz

# Include stored procedures and triggers
mysqldump -u root -p -R mydb > backup_with_procs.sql
# -R: Include routines (stored procedures, functions)
```

### Selective Backups

```bash
# Backup specific tables
mysqldump -u root -p mydb users orders > tables.sql

# Exclude specific tables
mysqldump -u root -p mydb --ignore-table=mydb.logs > backup_no_logs.sql

# Only schema (no data)
mysqldump -u root -p -d mydb > schema_only.sql
# -d: Dump schema only

# Only data (no schema)
mysqldump -u root -p -t mydb > data_only.sql
# -t: Dump data only
```

### Advanced Options

```bash
# Consistent point-in-time (for InnoDB)
mysqldump -u root -p --single-transaction mydb > backup.sql
# --single-transaction: Atomic snapshot for InnoDB

# Include binary log coordinates
mysqldump -u root -p --master-data=2 mydb > backup.sql
# --master-data=2: Comment out CHANGE MASTER TO

# Lock tables (for MyISAM)
mysqldump -u root -p --lock-tables mydb > backup.sql

# Disable keys during restore (faster)
mysqldump -u root -p --disable-keys mydb > backup.sql
```

### Restore from Logical Backup

```bash
# Restore plaintext SQL
mysql -u root -p mydb < backup.sql

# Restore compressed backup
gunzip < backup.sql.gz | mysql -u root -p mydb

# Restore specific database (from all-databases dump)
mysql -u root -p < all_backups.sql

# Restore with progress (uses pv)
pv backup.sql | mysql -u root -p mydb
```

---

## 🔐 Physical Backup (Percona XtraBackup)

Byte-for-byte copy of data files. Faster restore, enables PITR and replication setup.

### Install Percona XtraBackup

```bash
# Ubuntu/Debian
sudo apt-get install percona-xtrabackup-80

# CentOS/RHEL
sudo yum install percona-xtrabackup-80
```

### Basic Physical Backup

```bash
# Simple backup
innobackupex --user=root --password /path/to/backups/

# Or newer syntax (XtraBackup 8.0+)
xtrabackup --backup --target-dir=/path/to/backups/ --user=root --password

# Backup with compression
xtrabackup --backup --target-dir=/path/to/backups/ \
    --compress \
    --user=root \
    --password

# Backup specific to timestamp
xtrabackup --backup \
    --target-dir=/backups/backup_20240420 \
    --user=root \
    --password
```

### Prepare Physical Backup

```bash
# Before restore, must prepare the backup
xtrabackup --prepare --target-dir=/path/to/backup

# This replays transactions and makes backup consistent
```

### Restore from Physical Backup

```bash
# 1. Stop MySQL
sudo systemctl stop mysql

# 2. Clean data directory
sudo rm -rf /var/lib/mysql/*

# 3. Copy backup to data directory
xtrabackup --copy-back --target-dir=/path/to/backup

# 4. Fix permissions
sudo chown -R mysql:mysql /var/lib/mysql

# 5. Start MySQL (recovery happens automatically)
sudo systemctl start mysql

# 6. Verify recovery
mysql -u root -p -e "SELECT NOW();"
```

---

## ⏱️ PITR (Point-in-Time Recovery) with Binary Logs

Recover database to any point in time where binary logs exist.

### Prerequisites for PITR

1. Binary logging enabled
2. Backup with binary log coordinates
3. All binary logs between backup and target time retained
4. Knowledge of recovery target time

### PITR Restore Procedure

```bash
# 1. Restore from backup
mysql -u root -p mydb < backup.sql

# 2. Identify binary log coordinates from backup
# Check backup file for CHANGE MASTER TO comments
grep "CHANGE MASTER" backup.sql

# 3. Export binary logs after backup point
# Find which binary log and position to start from
mysqlbinlog --base64-output=decode-rows mysql-bin.000001 | grep -i "CHANGE MASTER" | head -1

# 4. Apply binary logs up to target time
mysqlbinlog \
    --start-datetime="2024-04-20 00:00:00" \
    --stop-datetime="2024-04-20 14:30:00" \
    mysql-bin.000001 mysql-bin.000002 | mysql -u root -p

# 5. Verify recovery
mysql -u root -p -e "SELECT NOW();"
```

### Important Binary Log Parameters

```bash
# Export specific binary log
mysqlbinlog mysql-bin.000001 > binlog.sql

# Start from specific position
mysqlbinlog --start-position=1000 mysql-bin.000001 | mysql -u root -p

# Stop at specific position
mysqlbinlog --stop-position=2000 mysql-bin.000001 | mysql -u root -p

# Start from specific time
mysqlbinlog --start-datetime="2024-04-20 00:00:00" mysql-bin.000001 > binlog.sql

# Stop at specific time
mysqlbinlog --stop-datetime="2024-04-20 14:30:00" mysql-bin.000001 > binlog.sql

# Database filter (specific database)
mysqlbinlog --database=mydb mysql-bin.000001 > binlog_filtered.sql
```

---

## 🔍 Backup Strategy for MySQL

### Recommended Strategy

```
┌─────────────────────────────────────────┐
│ FULL PHYSICAL BACKUP (Weekly Sunday)   │
│ xtrabackup → compressed backup         │
├─────────────────────────────────────────┤
│ BINARY LOG ARCHIVING (Continuous)      │
│ Binary logs → offsite storage          │
├─────────────────────────────────────────┤
│ LOGICAL BACKUP (Monthly)               │
│ mysqldump → plaintext for portability  │
├─────────────────────────────────────────┤
│ RESTORE TEST (Monthly)                 │
│ Verify physical + binary log recovery  │
└─────────────────────────────────────────┘

Result: RPO ≤ 1 minute (binary log frequency), RTO ≤ 1 hour
```

---

## 🛠️ Backup Automation Script

```bash
#!/bin/bash
# mysql_backup.sh
# Purpose: Automated daily backup of MySQL

set -e

BACKUP_DIR="/backups/mysql"
DB_NAME="production"
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_PATH="$BACKUP_DIR/backup_${BACKUP_DATE}"

# Create backup directory
mkdir -p $BACKUP_DIR

echo "Starting MySQL backup at $(date)"

# Physical backup with XtraBackup
innobackupex \
    --user=backup_user \
    --password=backup_pass \
    --compress \
    $BACKUP_PATH

echo "Backup completed: $BACKUP_PATH"

# Prepare backup
innobackupex --apply-log $BACKUP_PATH

# Compress for storage
cd $BACKUP_DIR
tar -czf backup_${BACKUP_DATE}.tar.gz backup_${BACKUP_DATE}/
rm -rf backup_${BACKUP_DATE}/

echo "Compressed backup: backup_${BACKUP_DATE}.tar.gz"

# Keep only last 8 backups
cd $BACKUP_DIR
ls -t backup_*.tar.gz | tail -n +9 | xargs rm -f 2>/dev/null || true

echo "Backup finished at $(date)"
```

### Setup Cron Schedule

```bash
# Daily backup at 2 AM
0 2 * * * /scripts/mysql_backup.sh >> /var/log/mysql_backup.log 2>&1

# Archive binary logs daily at 3 AM
0 3 * * * /scripts/archive_binlogs.sh >> /var/log/mysql_backup.log 2>&1
```

---

## 🚨 Common MySQL Backup Issues

### Issue: Binary Logs Not Being Created

```sql
SHOW VARIABLES LIKE 'log_bin%';
SHOW VARIABLES LIKE 'binlog_format';
```

**Solution:**

- Check my.cnf has `log_bin` enabled
- Restart MySQL after changing config
- Verify MySQL has write permissions to log directory

### Issue: Backup Too Large

**Solutions:**

- Use compression: `--compress` with xtrabackup
- Use incremental backups
- Exclude large non-critical tables
- Implement table archiving

### Issue: Restore Takes Too Long

**Solutions:**

- Use physical backup instead of logical
- Use parallel restore for logical backups
- Consider warm standby for faster failover

---

## ✅ MySQL Backup Checklist

- [ ] Binary logging configured and enabled
- [ ] Daily physical backups scheduled
- [ ] Backups verified (size, integrity)
- [ ] Backup retention policy defined
- [ ] Offsite backups configured
- [ ] Binary logs archived
- [ ] Monthly restore tests scheduled
- [ ] Recovery procedure documented
- [ ] Backup user created with minimal permissions
- [ ] Backup monitoring alerts configured
- [ ] Backup encryption enabled for sensitive data

---

## 🔗 Related Topics

- [RPO & RTO Explained](rpo-rto-explained.md) — Design your recovery targets
- [Backup Strategies](backup-strategies.md) — MySQL in broader context
- [Restore Testing](restore-testing.md) — Test your MySQL backups

---

**Key Insight:** MySQL's binary logs enable powerful recovery capabilities. Combine physical backups with binary log archiving for flexible, fast recovery at any point in time.
