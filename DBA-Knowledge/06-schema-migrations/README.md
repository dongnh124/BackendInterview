# Schema Migrations & Change Management

Safe, zero-downtime schema changes for production databases.

## 📚 Core Topics

1. **Migration Tools** — Flyway, Liquibase, sqitch
2. **Expand-Contract Pattern** — Zero-downtime changes
3. **Forward-Only Migrations** — No rollback philosophy
4. **Validation** — Data integrity checks
5. **Deployment** — Coordinating with application
6. **Rollback Strategies** — Recovery procedures
7. **Testing** — Staging validation
8. **Communication** — Team coordination

## 🎯 Migration Types by Risk

```
LOW RISK (Can do anytime, no locking concerns):
✓ Add nullable column with default
✓ Add index (may briefly lock, but online in modern DB)
✓ Add stored procedure/function
✓ Add check constraint (for new rows)

MEDIUM RISK (Can do but need coordination):
⚠ Rename column (with alias)
⚠ Rename table (application change needed)
⚠ Add NOT NULL column (must backfill first)
⚠ Increase column size (usually safe)

HIGH RISK (Needs zero-downtime strategy):
❌ Drop column (breaking change)
❌ Drop table (breaking change)
❌ Change column type (data loss possible)
❌ Add FOREIGN KEY to large table
❌ Mass data deletion
```

## 🛠️ Expand-Contract Pattern

### Example: Add required column to users table

**The Problem:**

```
Add "status" column with NOT NULL constraint
If we just ADD COLUMN with NOT NULL, it locks the table for everyone
```

**Solution: Three Phase Approach**

#### PHASE 1: EXPAND (Deploy)

```sql
-- 1. Add nullable column
ALTER TABLE users ADD COLUMN status VARCHAR(50);

-- 2. Add default for future rows
ALTER TABLE users ALTER COLUMN status SET DEFAULT 'active';

-- 3. Create index if needed (can be concurrent)
CREATE INDEX CONCURRENTLY idx_users_status ON users(status);

-- Application continues working (old code ignores new column)
-- Migration script: takes < 1 second, non-blocking
```

**Deployment:**

- Schema change: Easy, non-blocking
- Application: No change needed (backward compatible)

#### PHASE 2: MIGRATE (Data Backfill)

```sql
-- 1. Backfill data in batches (avoid locking entire table)
BEGIN;
UPDATE users SET status = 'active' WHERE status IS NULL AND id BETWEEN 1 AND 10000;
COMMIT;

BEGIN;
UPDATE users SET status = 'active' WHERE status IS NULL AND id BETWEEN 10001 AND 20000;
COMMIT;
-- ... repeat for all rows

-- 2. Monitor completion
SELECT COUNT(*) WHERE status IS NULL FROM users;  -- Should be 0

-- 3. Optional: Add check constraint (doesn't lock existing rows)
ALTER TABLE users ADD CONSTRAINT users_status_not_null
  CHECK (status IS NOT NULL) NOT VALID;
ALTER TABLE users VALIDATE CONSTRAINT users_status_not_null;
```

**Timing:**

- Backfill happens gradually (minutes to hours)
- Can run during business hours (batches avoid locks)
- Application still working

#### PHASE 3: CONTRACT (Cleanup)

```sql
-- 1. Add NOT NULL constraint (only affects new rows)
ALTER TABLE users ALTER COLUMN status SET NOT NULL;

-- 2. Remove temporary data/indexes if any
-- Application updated to expect new column

-- 3. If rollback needed, recreate column from backups
```

**Result:**

- Zero downtime
- Rollback possible at each phase
- Application can be tested with new column in place

## ✅ Migration Runbook Template

```sql
-- ============================================
-- Migration: YYYY-MM-DD-HH-MM Add status column
-- Author: [Name]
-- Risk Level: LOW
-- Estimated Time: 5 minutes
-- ============================================

-- Tested on: Staging environment on 2024-01-15
-- Rollback: Simple (DROP COLUMN if needed)

BEGIN;

-- Phase 1: Expand
ALTER TABLE users ADD COLUMN status VARCHAR(50);
ALTER TABLE users ALTER COLUMN status SET DEFAULT 'active';

-- Verify
SELECT * FROM users LIMIT 1;

COMMIT;

-- Phase 2: Migrate (separate transaction for safety)
-- This can be run asynchronously
BEGIN;
UPDATE users SET status = 'active' WHERE status IS NULL;
COMMIT;

-- Phase 3: Contract (in future migration)
-- ALTER TABLE users ALTER COLUMN status SET NOT NULL;

-- Verification steps:
-- 1. Check row count before and after: SELECT COUNT(*) FROM users;
-- 2. Check data: SELECT DISTINCT status FROM users;
-- 3. Check index: SELECT * FROM users WHERE status = 'active' LIMIT 1;
```

## 🚀 Migration Tools

### Flyway

```bash
# Initialize
flyway init

# Create migration
# File: V1__Create_users_table.sql

# Migrate
flyway migrate

# Info
flyway info
```

**Migration file structure:**

```
db/migration/
├── V1__Create_initial_schema.sql
├── V2__Add_users_table.sql
├── V3__Add_index_on_email.sql
├── V4__Expand_status_column.sql  ← Our migration
└── V5__Add_metadata_column.sql
```

### Liquibase

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog>
    <changeSet id="1" author="john">
        <createTable tableName="users">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true"/>
            </column>
            <column name="name" type="VARCHAR(255)"/>
        </createTable>
    </changeSet>

    <changeSet id="2" author="john">
        <addColumn tableName="users">
            <column name="status" type="VARCHAR(50)" defaultValue="active"/>
        </addColumn>
    </changeSet>
</databaseChangeLog>
```

## 🔄 Forward-Only Philosophy

```
Traditional approach (rollback-friendly):
- Can undo any change
- Complex tool (versioning up and down)
- Risky (rollback can introduce new bugs)

Forward-only approach:
- Changes only go forward
- Simpler versioning (never undo)
- Safer (deploy fix forward, not backward)
```

**Why forward-only?**

```
Scenario: Bad migration deployed, must rollback
- Traditional: Rollback schema, app still expects new schema → Chaos
- Forward-only: Fix schema with new migration (safer controlled change)

Scenario: Data loss migration, want to undo
- Backup is still available (independent of schema versioning)
- Restore from backup if data recovery needed
- Schema versioning is separate concern
```

## 🧪 Testing Migration Procedures

### Pre-Migration Checklist

```
48 Hours Before:
☐ Migration tested in staging (with production-like data volume)
☐ Rollback plan written and tested
☐ Performance impact assessed (long-running migrations?)
☐ Communication plan created
☐ On-call engineer identified

4 Hours Before:
☐ Database backup completed
☐ Migration script reviewed by peer
☐ Rollback scripts ready
☐ Team members in place (app eng, DBA, oncall)

During Migration:
☐ Run on non-production first? (if timeline allows)
☐ Monitor lock contention
☐ Monitor error rates in application
☐ Have rollback ready to execute
```

### Validation After Migration

```sql
-- Data integrity checks:
SELECT COUNT(*) AS total_users FROM users;
-- Compare with before: should match

SELECT COUNT(DISTINCT status) FROM users;
-- Should have expected statuses

SELECT * FROM users WHERE status IS NULL;
-- Should be empty (if constraint added)

-- Index verification:
EXPLAIN ANALYZE SELECT * FROM users WHERE status = 'active';
-- Should use index, not seq scan
```

## 🎯 Common Patterns

### Rename Column (Zero-Downtime)

```sql
-- Phase 1: Add new column with same data
ALTER TABLE users ADD COLUMN email_new VARCHAR(255);
UPDATE users SET email_new = email;
CREATE INDEX idx_email_new ON users(email_new);

-- Phase 2: Dual-write (app writes to both columns)
-- Application code:
UPDATE users SET email = ?, email_new = ? WHERE id = ?;

-- Phase 3: Switch reads to new column
-- Application code:
SELECT email_new AS email FROM users WHERE id = ?;

-- Phase 4: Drop old column
ALTER TABLE users DROP COLUMN email;
ALTER TABLE users RENAME COLUMN email_new TO email;
```

### Add Foreign Key to Large Table

```sql
-- BAD: Direct add locks table
-- ALTER TABLE orders ADD CONSTRAINT fk_user
-- FOREIGN KEY (user_id) REFERENCES users(id);

-- GOOD: Not valid, then validate
ALTER TABLE orders ADD CONSTRAINT fk_user
FOREIGN KEY (user_id) REFERENCES users(id) NOT VALID;

-- Validate separately (can use concurrent index)
ALTER TABLE orders VALIDATE CONSTRAINT fk_user;
```

## 📋 Interview Questions

1. **How do you add a NOT NULL column to a table with 100M rows without downtime?**
   - Use expand-contract pattern
   - Add nullable column first
   - Backfill in batches during business hours
   - Finally add constraint

2. **Your migration failed halfway through. What do you do?**
   - Assess blast radius (which rows affected?)
   - Decide: Continue forward or restore backup
   - If continue: Identify what failed, fix in new migration
   - If restore: Restore backup, run fix, test again
   - Document root cause and prevention

3. **Design a migration strategy for renaming a table in production**
   - Create new table with new name (empty)
   - Add trigger to copy writes to both
   - Backfill existing data
   - Switch application to read from new
   - Remove old table

## ✅ Checklist

- [ ] Migration tools set up (Flyway/Liquibase)
- [ ] Migration versioning in source control
- [ ] Tested on staging first
- [ ] Peer review of SQL changes
- [ ] Rollback procedure documented
- [ ] Data validation steps written
- [ ] Backward-compatible approach used
- [ ] Team communication plan
- [ ] Monitoring enabled during migration
- [ ] Post-mortem process for any issues

## 🔗 Related

- [Backup & Recovery](../02-backup-recovery/)
- [Performance Tuning](../04-performance-tuning/)
- [PostgreSQL Guide](../07-platform-guides/postgresql.md)

---

**Key Insight:** The best migration is one you never have to rollback. Design for backward compatibility, test exhaustively in staging, and always have a way to go forward even if things go wrong.
