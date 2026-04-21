# PostgreSQL Backup & Recovery

PostgreSQL-specific backup strategies, tools, and best practices.

## 🔄 WAL (Write-Ahead Log) Archiving

WAL (Write-Ahead Log) is fundamental to PostgreSQL durability and recovery. Archive WAL files to enable PITR (Point-in-Time Recovery).

### Setup WAL Archiving

Edit `/etc/postgresql/[version]/main/postgresql.conf`:

```ini
# Enable WAL archiving
wal_level = replica                    # Must be 'replica' or 'logical'
max_wal_senders = 3                    # Replication connections
wal_keep_size = 10GB                   # Keep recent WALs locally
archive_mode = on                      # Enable archiving
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
archive_timeout = 300                  # Archive at least every 5 minutes
```

### Verify WAL Setup

```sql
-- Check WAL directory
SELECT * FROM pg_ls_waldir();

-- Check current WAL LSN (Log Sequence Number)
SELECT pg_current_wal_lsn();

-- Check if archiving is working
SELECT * FROM pg_stat_archiver;
-- archived_count should be increasing
-- failed_count should be 0
```

### Important WAL Files

```
WAL file naming: 000000010000000000000001

Structure:
- 8 hex digits: timeline
- 8 hex digits: log file number
- 8 hex digits: segment number
- Each file ~16MB
```

---

## 💾 Logical Backup (pg_dump)

SQL statements to recreate database. Good for smaller databases, portability, and selective backups.

### Full Database Backup

```bash
# Basic backup (plaintext SQL)
pg_dump -U postgres mydb > backup.sql

# With compression (much smaller)
pg_dump -U postgres -F c mydb > backup.dump
# -F c: custom format (compressed binary)
# -F d: directory format (parallel)
# -F t: tar format

# With compression level
pg_dump -U postgres -F c -Z 9 mydb > backup.dump
# -Z 9: Maximum compression
```

### Selective Backups

```bash
# Backup specific tables
pg_dump -U postgres -t users -t orders mydb > tables.sql

# Backup specific schema
pg_dump -U postgres -n public mydb > schema_public.sql

# Exclude specific tables
pg_dump -U postgres --exclude-table=logs mydb > backup_no_logs.sql

# Only schema (no data)
pg_dump -U postgres -s mydb > schema_only.sql

# Only data (no schema)
pg_dump -U postgres -a mydb > data_only.sql
```

### Advanced Options

```bash
# Verbose output
pg_dump -U postgres -v mydb > backup.sql

# Skip column order preservation
pg_dump -U postgres --no-ordered-cte mydb > backup.sql

# Include DROP statements
pg_dump -U postgres --clean mydb > backup.sql

# Parallel dump (faster for large databases)
pg_dump -U postgres -F d -j 4 mydb > backup_dir/
# -F d: directory format
# -j 4: 4 parallel jobs
```

### Restore from Logical Backup

```bash
# Plaintext SQL restore
psql -U postgres -d mydb < backup.sql

# Custom format restore
pg_restore -U postgres -d mydb backup.dump

# Directory format restore (parallel)
pg_restore -U postgres -d mydb -j 4 backup_dir/

# Restore specific table
pg_restore -U postgres -d mydb -t users backup.dump
```

---

## 🔐 Physical Backup (pg_basebackup)

Byte-for-byte copy of data files. Faster restore, enables PITR and replication setup.

### Basic Physical Backup

```bash
# Simple backup
pg_basebackup -U postgres -D /path/to/backup -Ft -z -P

# Flags:
# -U: User
# -D: Destination directory
# -Ft: Tar format (default)
# -Fz: Compressed tar
# -P: Show progress
# -l: Label (name your backup)
# -c: Checkpoint mode (fast/spread)
```

### Physical Backup with Labels

```bash
# Named backup with label
pg_basebackup -U postgres \
  -D /backups/base_2024_04_20 \
  -Ft -z -P \
  -l "Full backup - 2024-04-20"

# Backup with WAL files included
pg_basebackup -U postgres \
  -D /backups/base_with_wal \
  -Ft -z -P \
  -X fetch              # Include WAL files
  -c spread             # Spread checkpoint load
```

### Restore from Physical Backup

```bash
# Extract backup (assuming tar format)
cd /var/lib/postgresql/14/main
tar -xzf /backups/base_2024_04_20/base.tar.gz

# Start PostgreSQL (recovery happens automatically)
pg_ctl start

# Monitor recovery progress
tail -f /var/log/postgresql/postgresql.log
```

---

## ⏱️ PITR (Point-in-Time Recovery)

Recover database to any point in time where WAL is available.

### Prerequisites for PITR

1. WAL archiving enabled
2. WAL files retained or archived
3. Physical backup available
4. Knowledge of recovery target time

### PITR Restore Procedure

```bash
# 1. Restore from physical backup
cd /var/lib/postgresql/14/main
tar -xzf /backups/base_2024_04_20/base.tar.gz

# 2. Create recovery.signal file (PostgreSQL 12+)
touch /var/lib/postgresql/14/main/recovery.signal

# 3. Create recovery configuration (in postgresql.conf or recovery.conf)
cat > /var/lib/postgresql/14/main/postgresql.conf << 'EOF'
# Enable recovery mode
restore_command = 'cp /wal_archive/%f %p'

# Target time (ISO 8601 format)
recovery_target_time = '2024-04-20 14:30:00'

# Or target LSN
# recovery_target_lsn = '0/1234567'

# What to do at target
recovery_target_timeline = 'latest'
recovery_target_action = 'promote'
EOF

# 4. Start PostgreSQL
pg_ctl start

# 5. Monitor recovery
tail -f /var/log/postgresql/postgresql.log
# When recovery reaches target time, database promotes to online

# 6. Verify recovery worked
psql -U postgres -c "SELECT NOW();"
```

### PITR Parameters

```sql
-- Available recovery targets:
recovery_target_time = '2024-04-20 14:30:00'  -- Specific timestamp
recovery_target_name = 'before_migration'     -- Named savepoint
recovery_target_lsn = '0/1234567'             # Log sequence number

-- Recovery target action:
recovery_target_action = 'promote'       -- Promote to primary (default)
recovery_target_action = 'pause'         -- Pause at target for verification
recovery_target_action = 'shutdown'      -- Shutdown at target
```

---

## 🔍 Backup Strategy for PostgreSQL

### Recommended Strategy

```
┌─────────────────────────────────────────┐
│ FULL PHYSICAL BACKUP (Weekly Sunday)   │
│ pg_basebackup → compressed tar         │
├─────────────────────────────────────────┤
│ WAL ARCHIVING (Continuous)             │
│ archive_command → offsite storage      │
├─────────────────────────────────────────┤
│ LOGICAL BACKUP (Monthly)               │
│ pg_dump → plaintext for portability    │
├─────────────────────────────────────────┤
│ RESTORE TEST (Monthly)                 │
│ Verify physical + WAL recovery works   │
└─────────────────────────────────────────┘

Result: RPO ≤ 5 minutes (WAL frequency), RTO ≤ 1 hour
```

---

## 🛠️ Backup Automation Script

```bash
#!/bin/bash
# postgres_backup.sh
# Purpose: Automated daily backup of PostgreSQL

set -e

BACKUP_DIR="/backups/postgresql"
DB_NAME="production"
BACKUP_DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_${BACKUP_DATE}.tar.gz"

# Create backup directory
mkdir -p $BACKUP_DIR

echo "Starting PostgreSQL backup at $(date)"

# Physical backup
pg_basebackup \
    -U backup_user \
    -D /tmp/backup_temp \
    -Ft -z -P \
    -l "Daily backup $BACKUP_DATE"

# Move to final location
mv /tmp/backup_temp/base.tar.gz $BACKUP_FILE

# Verify backup
BACKUP_SIZE=$(stat -c%s "$BACKUP_FILE" 2>/dev/null || stat -f%z "$BACKUP_FILE")
echo "Backup completed: $BACKUP_FILE ($(numfmt --to=iec $BACKUP_SIZE 2>/dev/null || echo $BACKUP_SIZE))"

# Keep only last 8 backups
cd $BACKUP_DIR
ls -t backup_*.tar.gz | tail -n +9 | xargs rm -f 2>/dev/null || true

echo "Backup finished at $(date)"
```

### Setup Cron Schedule

```bash
# Daily backup at 2 AM
0 2 * * * /scripts/postgres_backup.sh >> /var/log/postgres_backup.log 2>&1

# Weekly full backup at 1 AM Sunday
0 1 * * 0 /scripts/postgres_backup_full.sh >> /var/log/postgres_backup.log 2>&1
```

---

## 🚨 Common PostgreSQL Backup Issues

### Issue: WAL Archiving Failing

```sql
SELECT * FROM pg_stat_archiver;
```

Look for:

- `failed_count > 0`: Archive command failed
- `last_failed_wal`: Last failed WAL file
- `last_failed_time`: When it failed

**Solution:**

- Check archive_command permissions
- Verify destination space
- Check network connectivity (if remote archiving)

### Issue: Backup Too Large

**Solutions:**

- Use `-F c` (compressed format) for logical backups
- Exclude large non-critical tables
- Use `-Z 9` for maximum compression
- Consider incremental backups

### Issue: Restore Takes Too Long

**Solutions:**

- Use physical backup instead of logical
- Use parallel restore: `pg_restore -j 4`
- Consider warm standby for faster failover

---

## ✅ PostgreSQL Backup Checklist

- [ ] WAL archiving configured and working
- [ ] Daily physical backups scheduled
- [ ] Backups verified (size, checksums)
- [ ] Backup retention policy defined
- [ ] Offsite backups configured
- [ ] Monthly restore tests scheduled
- [ ] Recovery procedure documented
- [ ] Backup user created with minimal permissions
- [ ] Backup monitoring alerts configured
- [ ] Backup encryption enabled for sensitive data

---

## 🔗 Related Topics

- [RPO & RTO Explained](rpo-rto-explained.md) — Design your recovery targets
- [Backup Strategies](backup-strategies.md) — PostgreSQL in broader context
- [Restore Testing](restore-testing.md) — Test your PostgreSQL backups

---

**Key Insight:** PostgreSQL's WAL archiving enables powerful recovery capabilities. Combine physical backups with WAL archiving for flexible, fast recovery at any point in time.
