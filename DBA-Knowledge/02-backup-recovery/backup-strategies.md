# Backup Strategies

Choosing the right backup strategy depends on database size, change rate, RPO requirements, and infrastructure.

## 📦 Backup Types Overview

### 1. Full Backup

**Complete database snapshot at point-in-time**

#### Pros

- Complete recovery, standalone
- Can restore entire database from one file
- Simplest to understand

#### Cons

- Large size (entire database)
- Long time to complete
- Resource intensive (CPU, disk I/O)
- Not practical for huge databases

#### Use When

- First backup (required baseline)
- Before major changes
- Baseline for incremental backups
- Small to medium databases

#### Example

```bash
# PostgreSQL
pg_dump -U postgres mydb > full_backup.sql

# MySQL
mysqldump -u root -p mydb > full_backup.sql
```

---

### 2. Incremental Backup

**Only blocks changed since last backup**

#### Pros

- Small size (only changed blocks)
- Fast to complete
- Less resource intensive
- Ideal for large, slowly changing databases

#### Cons

- Complex restore (need full + all incrementals in order)
- Dependent chain (lose one, lose all after it)
- Storage of multiple files

#### Use When

- Database large and frequently changing
- Storage/bandwidth constraints
- Can afford restore complexity
- Multiple incremental backups per day

#### Restore Complexity

```
Day 1: Full backup (100GB)
Day 2: Incremental #1 (5GB of changes)
Day 3: Incremental #2 (4GB of changes)

To restore to Day 3:
1. Restore full backup
2. Apply incremental #1
3. Apply incremental #2
= Takes long time
```

---

### 3. Differential Backup

**All changes since last FULL backup**

#### Pros

- Faster restore than incremental (only full + last differential)
- Smaller than full backup
- No dependent chain issue

#### Cons

- Larger than incremental
- Less efficient for rapidly changing databases

#### Use When

- Need better restore speed than incremental
- SQL Server (native support)
- Daily backup schedule works

#### Restore Simplicity

```
Day 1: Full backup (100GB)
Day 2: Differential (5GB of all changes)
Day 3: Differential (8GB of all changes)

To restore to Day 3:
1. Restore full backup
2. Apply latest differential (Day 3)
= Faster than incremental
```

---

### 4. Logical Backup

**SQL statements to recreate database**

Tools: `pg_dump`, `mysqldump`, `expdp`

#### Pros

- Portable across platforms
- Human-readable (can edit SQL)
- Can exclude specific tables/schemas
- Database-agnostic
- Can be version-forward compatible

#### Cons

- Slower restore (re-execute SQL)
- RPO depends on dump schedule
- Larger for large databases
- Cannot do PITR easily

#### Use When

- Migrating between versions
- Portability needed
- Export subset required
- Small to medium databases

#### Example

```bash
# PostgreSQL full dump
pg_dump -U postgres mydb > backup.sql

# With compression
pg_dump -U postgres -F c mydb > backup.dump

# Specific tables only
pg_dump -U postgres -t users -t orders mydb > subset.sql
```

---

### 5. Physical Backup

**Byte-for-byte copy of data files**

Tools: filesystem snapshots, `xtrabackup`, `pg_basebackup`

#### Pros

- Faster restore than logical
- Can do PITR with WAL files
- Can set up replicas
- Copy entire database quickly

#### Cons

- Engine-specific
- Needs crash recovery
- Larger files
- Less portable

#### Use When

- Large databases
- Need fast RTO
- PITR required
- Replication setup needed

#### Example

```bash
# PostgreSQL physical backup
pg_basebackup -U postgres -D /path/to/backup

# MySQL Percona
innobackupex --user=root --password=/path/to/backup
```

---

## 🛠️ Recommended Backup Strategy (OLTP)

For typical OLTP database with 99.99% uptime requirement:

```
┌──────────────────────────────────────────┐
│ FULL BACKUP (Weekly Sunday 2 AM)        │
│ RPO: 7 days worst case                  │
├──────────────────────────────────────────┤
│ DIFFERENTIAL BACKUPS (Daily 3 AM)       │
│ RPO: 1 day worst case                   │
├──────────────────────────────────────────┤
│ TRANSACTION LOG BACKUP (Every 15 min)   │
│ RPO: 15 minutes                         │
├──────────────────────────────────────────┤
│ Offsite Copy (Daily)                    │
│ Protection against site failure         │
├──────────────────────────────────────────┤
│ RESTORE TEST (Monthly)                  │
│ Verify backups work                     │
└──────────────────────────────────────────┘

Result: RPO ≤ 15 minutes, RTO ≤ 1 hour
```

### Why This Stack?

1. **Full backup (weekly)**: Baseline for recovery, fits weekly maintenance window
2. **Differential daily**: Fast restore (only full + latest diff)
3. **Transaction logs (15 min)**: Point-in-time recovery without replicas
4. **Offsite (daily)**: Protection against site loss (fire, natural disaster)
5. **Test (monthly)**: Verify everything works before crisis

---

## 📊 Backup Strategy Comparison

| Factor           | Full    | Incremental | Differential | Logical | Physical  |
| ---------------- | ------- | ----------- | ------------ | ------- | --------- |
| Backup Size      | 100%    | 5-10%       | 15-25%       | 80%     | 100%      |
| Backup Speed     | Medium  | Very Fast   | Fast         | Slow    | Medium    |
| Restore Speed    | Fast    | Very Slow   | Fast         | Slow    | Very Fast |
| Complexity       | Simple  | Very High   | Medium       | Simple  | Medium    |
| Storage Required | Highest | Low         | Medium       | High    | High      |
| PITR Support     | No      | Limited     | Limited      | No      | Yes       |
| Portability      | Low     | Low         | Low          | High    | Low       |

---

## 🎯 Choosing Your Strategy

### Decision Tree

```
1. Database Size?
   - < 50GB → Use full backup daily
   - > 50GB → Use differential or physical

2. Change Rate?
   - Low (< 5% daily) → Use differential
   - High (> 20% daily) → Use incremental or physical + WAL

3. PITR Needed?
   - Yes → Use physical + WAL archiving
   - No → Use full/differential/logical

4. Portability Needed?
   - Yes → Use logical backup
   - No → Use physical backup

5. RTO Target?
   - > 4 hours → Logical backup okay
   - 1-4 hours → Differential backup
   - < 1 hour → Physical backup + WAL
```

---

## 💾 Storage Capacity Planning

### Calculate Total Backup Storage

```
Daily_Backup_Size = Database_Size × (1 + Change_Rate)
Weekly_Backup_Size = Database_Size × 1.05
Monthly_Retention = (Daily × 30) + (Weekly × 13)

Example: 100GB database, 2% daily change
- Daily differential: 100GB × 0.02 = 2GB
- 30 daily backups × 2GB = 60GB
- 13 weekly full: 13 × 100GB = 1.3TB
- Monthly total: 60GB + 1.3TB = 1.36TB
- Yearly: 16TB (without compression)

With 50% compression:
- Yearly: 8TB (realistic)
```

---

## ✅ Backup Strategy Checklist

- [ ] Define RPO requirement
- [ ] Define RTO requirement
- [ ] Determine database size and growth
- [ ] Calculate change rate
- [ ] Choose backup type (full/diff/incremental)
- [ ] Determine backup frequency
- [ ] Calculate total storage required
- [ ] Plan retention period
- [ ] Include transaction log backups
- [ ] Include offsite copy
- [ ] Schedule restore tests
- [ ] Document restore procedure
- [ ] Monitor backup success rate

---

## 🔗 Related Topics

- [RPO & RTO Explained](rpo-rto-explained.md) — Understand your requirements
- [Restore Testing](restore-testing.md) — Verify your backups work
- [PostgreSQL Backup](postgresql-backup.md) — PostgreSQL-specific strategies
- [MySQL Backup](mysql-backup.md) — MySQL-specific strategies

---

**Key Insight:** The best backup strategy balances RPO/RTO requirements with cost and complexity. Test it regularly—a backup you haven't restored is just hope.
