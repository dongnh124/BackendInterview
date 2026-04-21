# Transaction Isolation Levels

## Overview

Isolation levels control the balance between **concurrency** (multiple transactions) and **safety** (preventing anomalies). Higher isolation = safer but slower.

---

## Four Isolation Levels

### 1. Read Uncommitted ❌

**Definition:** Transactions can read data from uncommitted transactions.

**Problems Allowed:**

- ✅ Dirty reads
- ✅ Non-repeatable reads
- ✅ Phantom reads

**Example:**

```
Transaction A (Writer)          Transaction B (Reader)
BEGIN;
UPDATE balance = 0 WHERE id=1;
                                BEGIN;
                                SELECT balance;  -- Returns 0 (dirty read!)
ROLLBACK;  -- Never happened!
                                SELECT balance;  -- Returns 100 (original)
```

**When to use:**

- ❌ Almost never (too dangerous)
- Only for non-critical analytics with ancient databases

**Performance:** Fastest (no locks)

---

### 2. Read Committed ✓

**Definition:** Only committed data is read. Most common default.

**Problems Prevented:**

- ✓ Dirty reads

**Problems Still Allowed:**

- ✅ Non-repeatable reads
- ✅ Phantom reads

**Example:**

```
Transaction A (Writer)          Transaction B (Reader)
BEGIN;
UPDATE balance = 0 WHERE id=1;
                                BEGIN;
                                SELECT balance;  -- Blocks! Waits for A
COMMIT;  -- Now committed
SELECT balance;  -- Returns 0 (committed data only)
                                ROLLBACK;
```

**Implementation:**

- Row-level locks held only for duration of read
- Prevents dirty reads by holding lock until write committed

**When to use:**

- ✓ Default for MySQL, SQL Server
- ✓ Most web applications
- ✓ Good balance of performance + safety

**PostgreSQL default:** Read Committed

**MySQL/SQL Server default:** Read Committed (SQL Server) / Repeatable Read (MySQL)

---

### 3. Repeatable Read 🔒

**Definition:** Same query returns same data throughout transaction. No new rows appear/disappear.

**Problems Prevented:**

- ✓ Dirty reads
- ✓ Non-repeatable reads

**Problems Still Allowed:**

- ✅ Phantom reads (SQL Standard definition)
- ⚠️ PostgreSQL doesn't allow phantom reads (different implementation)

**Example - Non-repeatable Read (Read Committed has this issue):**

```
Transaction A (Modifier)        Transaction B (Reader)
                                BEGIN;
                                SELECT * FROM orders WHERE user_id = 1;
                                -- Returns: order1, order2
BEGIN;
INSERT INTO orders (user_id = 1);
COMMIT;
                                SELECT * FROM orders WHERE user_id = 1;
                                -- Returns: order1, order2, order3 (PHANTOM!)
                                COMMIT;
```

**With Repeatable Read:**

```
                                BEGIN REPEATABLE READ;
                                SELECT * FROM orders;  -- Snapshot taken
BEGIN;
INSERT INTO orders;
COMMIT;
                                SELECT * FROM orders;  -- Still sees old snapshot
                                COMMIT;
```

**Implementation Methods:**

**Method 1: Snapshot Isolation (PostgreSQL, MySQL 8.0+)**

- Each transaction gets consistent snapshot at START time
- Reads always see same data

**Method 2: Locking (Old approach)**

- Locks all read rows for duration of transaction
- Prevents modifications

**When to use:**

- ✓ Financial transactions
- ✓ Reports that need consistency
- ✓ Default in PostgreSQL

---

### 4. Serializable 🔐

**Definition:** Transactions execute as if they were serial (one after another). Strictest isolation.

**Problems Prevented:**

- ✓ Dirty reads
- ✓ Non-repeatable reads
- ✓ Phantom reads
- ✓ Serialization anomalies

**Example - Serialization Anomaly:**

```
Transaction A                  Transaction B
BEGIN;
SELECT SUM(balance) FROM accounts;
-- Result: 1000
                               BEGIN;
                               INSERT INTO accounts (100);
                               COMMIT;
SELECT SUM(balance);
-- Result: 1100
-- A saw inconsistent state!
COMMIT;
```

**With Serializable:**

- One transaction blocks until other commits
- Data is fully consistent

**Implementation:**

- SSI (Serializable Snapshot Isolation) in PostgreSQL
- Predicate locking in traditional systems
- Can be expensive

**When to use:**

- ✓ Critical financial systems
- ✓ Regulatory requirements
- ⚠️ Expect performance impact
- ✓ Use wisely (not all transactions need it)

---

## Platform Defaults & Capabilities

| Platform       | Default             | Supports All 4 | Notes                        |
| -------------- | ------------------- | -------------- | ---------------------------- |
| **PostgreSQL** | Read Committed      | ✅ Yes         | SSI for Serializable         |
| **MySQL**      | Repeatable Read     | ⚠️ Limited     | Missing true Serializable    |
| **SQL Server** | Read Committed      | ✅ Yes         | Snapshot isolation available |
| **Oracle**     | Read Committed      | ✅ Yes         | Multi-versioning             |
| **MongoDB**    | Session consistency | ✅ Limited     | Per-document ACID only       |

---

## Anomalies Prevented

### Dirty Read

```sql
-- Read Uncommitted allows this:
Transaction A:
  UPDATE users SET credit = 0 WHERE id = 1;

Transaction B:
  SELECT credit FROM users WHERE id = 1;  -- Sees 0 (not committed!)

Transaction A:
  ROLLBACK;  -- Never happened!

-- Result: B read data that was never committed
```

**Fixed by:** Read Committed+

---

### Non-Repeatable Read

```sql
-- Read Committed allows this:
Transaction A:
  SELECT salary FROM employees WHERE id = 1;  -- Returns 50000

Transaction B:
  UPDATE employees SET salary = 60000 WHERE id = 1;
  COMMIT;

Transaction A:
  SELECT salary FROM employees WHERE id = 1;  -- Returns 60000!

-- Result: Same SELECT got different result
```

**Fixed by:** Repeatable Read+

---

### Phantom Read

```sql
-- Repeatable Read (SQL Standard) allows this:
Transaction A:
  SELECT * FROM employees WHERE salary > 50000;  -- 5 rows

Transaction B:
  INSERT INTO employees (salary = 60000);
  COMMIT;

Transaction A:
  SELECT * FROM employees WHERE salary > 50000;  -- 6 rows!

-- Result: Phantom row appeared
```

**Fixed by:** Serializable

**Note:** PostgreSQL's Repeatable Read actually prevents this using conflict detection.

---

## Choosing the Right Level

### Decision Matrix

```
Do you need:
  1. Speed? (Many concurrent transactions)
     → Read Committed

  2. Accuracy? (Financial, audit)
     → Repeatable Read or Serializable

  3. Legal compliance? (Can't have any anomalies)
     → Serializable

  4. Default that works? (Safest without overhead)
     → Repeatable Read
```

### By Use Case

| Scenario                | Level           | Reason                  |
| ----------------------- | --------------- | ----------------------- |
| Web requests (REST API) | Read Committed  | Speed + adequate safety |
| Financial transfers     | Repeatable Read | Consistency needed      |
| Bank clearing           | Serializable    | No anomalies allowed    |
| Analytics report        | Repeatable Read | Consistent snapshot     |
| Real-time dashboard     | Read Committed  | Freshness > accuracy    |

---

## Setting Isolation Levels

### PostgreSQL

```sql
-- Session level (applies to all transactions)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Transaction level (applies to next transaction)
BEGIN ISOLATION LEVEL REPEATABLE READ;
  SELECT * FROM accounts;
  -- All queries in this transaction use Repeatable Read
COMMIT;

-- Check current level
SHOW TRANSACTION ISOLATION LEVEL;
```

### MySQL

```sql
-- Session level
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Global level (affects new connections)
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- Transaction level
START TRANSACTION;
  SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
  SELECT * FROM accounts;
COMMIT;

-- Check current level
SELECT @@transaction_isolation;
```

### SQL Server

```sql
-- Connection level
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SET TRANSACTION ISOLATION LEVEL SNAPSHOT;

-- Transaction level
BEGIN TRANSACTION;
  SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
  SELECT * FROM accounts;
COMMIT;
```

---

## Performance Impact

| Level            | Locks  | Snapshots               | Speed  | Safety |
| ---------------- | ------ | ----------------------- | ------ | ------ |
| Read Uncommitted | None   | None                    | ⚡⚡⚡ | ❌     |
| Read Committed   | Short  | Per-statement           | ⚡⚡   | ✓      |
| Repeatable Read  | Medium | Per-transaction         | ⚡     | ✓✓     |
| Serializable     | Long   | Full conflict detection | 🐢     | ✓✓✓    |

**Optimization Tips:**

```sql
-- Use READ COMMITTED for most work
BEGIN READ COMMITTED;
  -- Fast queries here
COMMIT;

-- Use REPEATABLE READ only when needed
BEGIN REPEATABLE READ;
  -- Slower but consistent
COMMIT;

-- Minimize scope of SERIALIZABLE
-- ❌ WRONG
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- Entire session serialized!

-- ✓ CORRECT
BEGIN ISOLATION LEVEL SERIALIZABLE;
  SELECT SUM(balance) FROM accounts;  -- Only this transaction serialized
COMMIT;
```

---

## Common Pitfalls

### 1. Assuming Serializable Prevents All Errors

```sql
-- ❌ WRONG
BEGIN SERIALIZABLE;
  INSERT INTO users VALUES (...);  -- Might still fail if duplicate key
  -- Serializable only prevents anomalies, not constraint violations
COMMIT;
```

### 2. Long Transactions at High Isolation

```sql
-- ❌ WRONG
BEGIN REPEATABLE READ;
  SELECT * FROM big_table;  -- Holds snapshot
  -- ... application thinks for 30 minutes ...
  UPDATE big_table SET status = 'processed';
COMMIT;
-- Locks held entire time, blocks other transactions

-- ✓ CORRECT
SELECT * FROM big_table;  -- No transaction
-- ... application thinks ...
BEGIN REPEATABLE READ;
  UPDATE big_table SET status = 'processed';
COMMIT;
```

### 3. Ignoring Platform Defaults

```sql
-- MySQL defaults to REPEATABLE READ
-- PostgreSQL defaults to READ COMMITTED
-- SQL Server defaults to READ COMMITTED

-- Application works on one but fails on another!
-- Solution: Explicitly set isolation level in application code
```

---

## Interview Questions

1. **Explain isolation levels and why they matter**
   - Pattern: Read Uncommitted → Serializable, trade-offs

2. **What anomalies does Repeatable Read prevent?**
   - Pattern: Dirty reads + non-repeatable reads, but NOT phantoms (in SQL standard)

3. **When would you use Serializable?**
   - Pattern: Critical financial transactions + regulatory requirements

4. **How does PostgreSQL achieve Repeatable Read?**
   - Pattern: MVCC + snapshot isolation

5. **Design a transaction that needs Serializable**
   - Pattern: Find anomaly only Serializable prevents

---

## Quick Reference

```
        Anomaly Prevention
Level                Dirty    Non-Rep    Phantom
─────────────────────────────────────────────────
Read Uncommitted      ❌       ❌          ❌
Read Committed        ✓        ❌          ❌
Repeatable Read       ✓        ✓           ⚠️*
Serializable          ✓        ✓           ✓

* PostgreSQL prevents, SQL Standard allows
```
