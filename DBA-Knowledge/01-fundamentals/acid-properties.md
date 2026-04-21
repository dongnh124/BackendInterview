# ACID Properties

## Overview

ACID represents four key properties that guarantee reliable database transactions. Understanding these is fundamental to designing robust systems.

## The Four Properties

### 1. Atomicity (All or Nothing)

**Definition:** A transaction is either completely committed or completely rolled back. Partial execution is impossible.

**Examples:**

```sql
-- Bank transfer: Atomicity ensures both debits AND credits happen
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- Alice loses $100
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- Bob gains $100
COMMIT;
```

If a crash occurs between the two updates:

- Database rolls back to pre-transaction state
- Alice keeps her $100, Bob doesn't get it
- No partial state where money disappears

**Implementation Details:**

- **Write-Ahead Logging (WAL):** Changes written to log before committed to disk
- **Rollback capability:** Undo logs stored for all changes
- **2-Phase Commit:** Ensures all-or-nothing across systems

**Common Issues:**

```
❌ Application handles SQL, but crashes before COMMIT
   → Database rolls back, but app thinks it succeeded

Solution: Always handle COMMIT/ROLLBACK in same transaction context
```

---

### 2. Consistency (Valid State)

**Definition:** Database moves from one valid state to another. No orphaned references, violated constraints, or corrupted data.

**Examples:**

```sql
-- Foreign key constraint ensures consistency
CREATE TABLE accounts (
  id INT PRIMARY KEY,
  balance DECIMAL NOT NULL CHECK (balance >= 0)
);

CREATE TABLE transactions (
  id INT PRIMARY KEY,
  account_id INT REFERENCES accounts(id),
  amount DECIMAL NOT NULL
);
```

**Consistency Rules:**

1. **Constraints enforced:**
   - Primary keys (no duplicates)
   - Foreign keys (no orphans)
   - Check constraints (business rules)
   - Unique constraints

2. **Triggers** can enforce complex consistency:

```sql
CREATE TRIGGER balance_check BEFORE INSERT ON transactions
BEGIN
  IF (SELECT balance FROM accounts WHERE id = NEW.account_id) < NEW.amount
  THEN RAISE ERROR 'Insufficient funds';
  END IF;
END;
```

**Durability relationship:** Consistency + Durability = No data loss/corruption

---

### 3. Isolation (Concurrent Independence)

**Definition:** Concurrent transactions don't interfere. Each transaction sees a consistent snapshot.

**Problems Isolation Prevents:**

| Problem                 | Description                  | Example                                                                  |
| ----------------------- | ---------------------------- | ------------------------------------------------------------------------ |
| **Dirty Read**          | Read uncommitted changes     | Transaction A reads partial update from B before B commits               |
| **Non-Repeatable Read** | Data changes mid-transaction | Same SELECT returns different results at start vs end of transaction     |
| **Phantom Read**        | New rows appear/disappear    | WHERE clause matches different rows during transaction                   |
| **Lost Update**         | Concurrent updates overwrite | Two transactions read `x=10`, both add 1, write `x=11` instead of `x=12` |

**Isolation Levels (Low to High):**

```
Read Uncommitted → Read Committed → Repeatable Read → Serializable
    ↓                    ↓                ↓                 ↓
  (none)         (block dirty reads)  (block non-repeatable)  (full lock)
  Performance    Standard             Default (PostgreSQL)    Safety
  Worst safety   ✓ Production         ✓ Most common           Slow
```

**Implementation Mechanisms:**

- **Locking:** Row/table-level locks
- **Versioning:** MVCC (Multi-Version Concurrency Control) - PostgreSQL/MySQL 8.0+
- **Snapshot Isolation:** Each transaction sees consistent view

---

### 4. Durability (Persistent)

**Definition:** Committed data survives any failure (crash, power loss, disk corruption).

**Mechanisms:**

1. **Write-Ahead Logging (WAL):**

   ```
   Step 1: Write to log on disk
   Step 2: Update in-memory buffer
   Step 3: Return success to client
   Step 4: Flush buffer to disk periodically
   ```

2. **Fsync guarantees:**
   - `fsync=on` (PostgreSQL): Every COMMIT waits for disk write
   - `fsync=off`: Much faster but risky on crashes

3. **Replication:** Data on multiple servers

**Example Timeline:**

```
08:00:00.000 - Client begins transaction
08:00:00.100 - INSERT/UPDATE sent to server
08:00:00.200 - Server writes to WAL on disk
08:00:00.300 - Client gets success response
08:00:00.500 - CRASH! Server loses power

Recovery: Database reads WAL, replays committed transactions
Result: Data is recovered and durable
```

---

## ACID Violations in Real Systems

### When ACID Breaks

**Application-level errors:**

```python
def transfer_money(from_id, to_id, amount):
    # ❌ BAD: Not in a transaction
    db.execute(f"UPDATE accounts SET balance = balance - {amount} WHERE id = {from_id}")

    if random() > 0.99:  # 1% of the time
        raise Exception("Network error")  # Rolled back, but...

    db.execute(f"UPDATE accounts SET balance = balance + {amount} WHERE id = {to_id}")
```

**Correct approach:**

```python
def transfer_money(from_id, to_id, amount):
    with db.transaction():  # ✓ GOOD
        db.execute(f"UPDATE accounts SET balance = balance - {amount} WHERE id = {from_id}")
        db.execute(f"UPDATE accounts SET balance = balance + {amount} WHERE id = {to_id}")
        # Both succeed together or both fail
```

**Cross-database transactions:**

```
Two separate databases (PostgreSQL + MongoDB) can't guarantee ACID together
→ Use distributed transactions or saga pattern
```

---

## Platform-Specific ACID Support

### PostgreSQL

- ✅ Full ACID at transaction level
- ✅ Supports Serializable isolation
- ✅ Synchronous replication for durability
- ✅ Crash-safe with WAL

### MySQL (InnoDB)

- ✅ Full ACID (InnoDB engine)
- ⚠️ Default Repeatable Read (not Serializable)
- ✅ Group Commit for durability
- ⚠️ MyISAM engine ≠ ACID (no longer used)

### MongoDB

- ✅ ACID at single document level (4.0+)
- ✅ Multi-document ACID (4.0+)
- ⚠️ Not cross-shard ACID
- ⚠️ Durability depends on `writeConcern`

---

## ACID Trade-offs

| Property        | Benefit            | Cost                | Default |
| --------------- | ------------------ | ------------------- | ------- |
| **Atomicity**   | No partial updates | Rollback overhead   | On      |
| **Consistency** | No invalid states  | Constraint checking | On      |
| **Isolation**   | No dirty data      | Locking contention  | Varies  |
| **Durability**  | Data persistence   | Disk I/O latency    | On      |

**Performance tune-up:**

```sql
-- PostgreSQL: Lower durability for speed (testing only!)
SET synchronous_commit = off;  -- Risk: data loss on crash
SET fsync = off;               -- Risk: corruption on crash

-- ✓ For production: Keep all ACID properties enabled
SET synchronous_commit = on;
```

---

## Interview Questions

1. **Explain ACID and why it matters**
   - _Pattern:_ Define each property + real-world impact

2. **What happens if a power outage occurs mid-transaction?**
   - _Pattern:_ Explain WAL → recovery process

3. **Can you have ACID without locks?**
   - _Pattern:_ Explain MVCC (PostgreSQL/MySQL) prevents many locks

4. **Design a system without ACID. What breaks?**
   - _Pattern:_ Distributed systems, eventual consistency trade-offs

---

## Practical Exercise

### Lab: Verify ACID Properties

```sql
-- Test 1: Atomicity
BEGIN;
  INSERT INTO accounts VALUES (1, 100);
  INSERT INTO accounts VALUES (1, 200);  -- Duplicate key! Will fail
COMMIT;  -- Entire transaction rolls back

SELECT * FROM accounts;  -- Empty (both inserts rolled back)

-- Test 2: Isolation
-- Terminal 1:
BEGIN;
  UPDATE accounts SET balance = 500 WHERE id = 1;
  -- Don't commit yet

-- Terminal 2:
SELECT * FROM accounts WHERE id = 1;  -- Still sees old value (not dirty read)

-- Test 3: Durability
INSERT INTO accounts VALUES (1, 100);
COMMIT;
-- Server crashes immediately after

-- After restart: Data still exists (durable)
SELECT * FROM accounts;  -- Row 1 still there
```

---

## Key Takeaways

- ✅ **Atomicity**: All-or-nothing transactions prevent corruption
- ✅ **Consistency**: Constraints keep database in valid state
- ✅ **Isolation**: Concurrent transactions don't interfere
- ✅ **Durability**: WAL ensures recovery from crashes
- ⚠️ **Trade-off**: Strict ACID has performance cost; tune based on risk tolerance
- ⚠️ **Not automatic**: Must use transactions properly in application code
