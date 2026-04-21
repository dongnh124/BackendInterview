# Indexing Strategies

## Overview

Indexes are the primary tool for query optimization. They trade write performance for read speed by maintaining data structures that enable fast lookups.

---

## Index Fundamentals

### Why Indexes Matter

**Without Index (Full Table Scan):**

```
1 million rows → Check every row → O(n) = 1,000,000 checks
```

**With Index (B-Tree Search):**

```
1 million rows → Binary search → O(log n) = 20 checks
```

**Real Cost:**

```
10GB table, full scan: ~1 second per query
Same table, indexed: ~10 milliseconds
```

### Index Trade-offs

| Aspect               | Cost                          |
| -------------------- | ----------------------------- |
| **Index storage**    | 10-20% of table size          |
| **Write overhead**   | 5-10% slower (maintain index) |
| **Read improvement** | 100-1000x faster (typical)    |
| **Memory pressure**  | Index cached in RAM (good)    |

**Rule of thumb:** Benefits > Costs for most columns

---

## Index Types

### 1. B-Tree (General Purpose) ⭐

**Structure:** Balanced tree, log(n) lookups

**When to use:**

- ✓ Default index type
- ✓ Equality searches (id = 5)
- ✓ Range searches (salary BETWEEN 50k AND 100k)
- ✓ Sorting (ORDER BY)
- ✓ Prefix matching (name LIKE 'John%')

**Example:**

```sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_created ON orders(created_at);
CREATE INDEX idx_salary_range ON employees(salary);

-- Queries that use these indexes:
SELECT * FROM users WHERE email = 'john@example.com';          -- ✓
SELECT * FROM orders WHERE created_at > '2026-01-01';         -- ✓
SELECT * FROM employees WHERE salary BETWEEN 50000 AND 100000; -- ✓
SELECT * FROM employees ORDER BY salary;                      -- ✓
```

**Leftmost Rule (Composite Indexes):**

```sql
CREATE INDEX idx_users_name_email ON users(first_name, last_name, email);

-- ✓ Uses index
SELECT * FROM users WHERE first_name = 'John' AND last_name = 'Doe';

-- ✓ Uses index
SELECT * FROM users WHERE first_name = 'John';

-- ❌ SKIP! Doesn't start with first_name
SELECT * FROM users WHERE last_name = 'Doe';

-- ✓ Uses index (can skip middle column)
SELECT * FROM users WHERE first_name = 'John' AND email = 'john@example.com';
```

**B-Tree Performance:**

```
Lookup:     O(log n)
Insert:     O(log n)
Delete:     O(log n)
Range scan: O(log n + result_count)
```

---

### 2. Hash Index

**Structure:** Hash table (very fast for exact match)

**When to use:**

- ✓ Exact equality only (id = 5)
- ✓ When you need guaranteed O(1) lookup
- ❌ Ranges (NOT for BETWEEN)
- ❌ Sorting

**Example:**

```sql
-- PostgreSQL (some databases support hash indexes)
CREATE INDEX idx_user_id_hash ON users USING HASH (id);

-- ✓ Very fast
SELECT * FROM users WHERE id = 12345;

-- ❌ Cannot use hash index (not supported for ranges)
SELECT * FROM users WHERE id BETWEEN 100 AND 200;
```

**Hash vs B-Tree:**

```
Hash:   O(1) lookup, but not for ranges
B-Tree: O(log n) lookup, works for everything
```

**Note:** Most databases prefer B-Tree because it handles both cases.

---

### 3. Bitmap Index

**Structure:** Stores presence/absence of values (bitmap)

**When to use:**

- ✓ Low cardinality columns (few distinct values)
- ✓ Gender (M/F), Status (Active/Inactive)
- ✓ Analytical queries (not OLTP)

**Example:**

```sql
CREATE BITMAP INDEX idx_status ON users(status);

-- ✓ Good (low cardinality)
SELECT * FROM users WHERE status = 'active' AND region = 'us';

-- ❌ Bad (high cardinality, too many bitmaps)
SELECT * FROM users WHERE email = 'john@example.com';
```

---

### 4. Full-Text Index 🔍

**Structure:** Inverted index (word → document mapping)

**When to use:**

- ✓ Text search (LIKE '%term%')
- ✓ Natural language queries
- ✓ Search engines

**Example (PostgreSQL):**

```sql
CREATE INDEX idx_posts_content ON posts USING GIN (
  to_tsvector('english', title || ' ' || content)
);

-- Fast text search
SELECT * FROM posts WHERE to_tsvector('english', content) @@ to_tsquery('english', 'database & performance');
```

**Example (MySQL):**

```sql
CREATE FULLTEXT INDEX idx_posts_content ON posts(title, content);

SELECT * FROM posts WHERE MATCH(title, content) AGAINST('+database -performance' IN BOOLEAN MODE);
```

---

### 5. Composite (Multi-Column) Index

**Structure:** B-Tree on multiple columns

**When to use:**

- ✓ Queries that filter on multiple columns
- ✓ Follow leftmost rule

**Example:**

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);

-- ✓ Uses index (filters on user_id first)
SELECT * FROM orders WHERE user_id = 5 AND status = 'pending';

-- ✓ Uses index (only first part)
SELECT * FROM orders WHERE user_id = 5;

-- ❌ Doesn't use index (wrong column order)
SELECT * FROM orders WHERE status = 'pending';
```

**Composite Index Strategy:**

```sql
-- Don't create too many indexes!
-- ❌ WRONG
CREATE INDEX idx1 ON orders(user_id);
CREATE INDEX idx2 ON orders(status);
CREATE INDEX idx3 ON orders(user_id, status);
CREATE INDEX idx4 ON orders(status, user_id);

-- ✓ CORRECT - choose one that matches query patterns
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

---

### 6. Partial Index (Filtered)

**Structure:** Index only rows matching condition

**When to use:**

- ✓ Index only active rows (not inactive)
- ✓ Reduces index size
- ✓ Common WHERE clause

**Example:**

```sql
-- Index only active users (saves space)
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;

-- ✓ Uses index
SELECT * FROM users WHERE email = 'john@example.com' AND is_active = true;

-- ❌ Doesn't use index (condition doesn't match)
SELECT * FROM users WHERE email = 'john@example.com' AND is_active = false;
```

**Real-world:** Soft-deleted records

```sql
CREATE INDEX idx_orders_active ON orders(user_id, created_at)
  WHERE deleted_at IS NULL;

-- ✓ Fast (uses partial index)
SELECT * FROM orders WHERE user_id = 5 AND deleted_at IS NULL;

-- ❌ Slow (full table scan needed for deleted orders)
SELECT * FROM orders WHERE user_id = 5 AND deleted_at IS NOT NULL;
```

---

### 7. Covering Index (Index-Only Scan)

**Structure:** All columns needed in query included in index

**When to use:**

- ✓ Query needs: WHERE + ORDER BY + SELECT columns all in index
- ✓ Eliminates table lookup

**Example:**

```sql
-- Include user email in index on user_id
CREATE INDEX idx_orders_covering ON orders(user_id, created_at) INCLUDE (user_email, status);

-- ✓ Index-only scan (doesn't need to touch table)
SELECT user_email, status FROM orders WHERE user_id = 5 ORDER BY created_at DESC;

-- ❌ Must hit table (phone_number not in index)
SELECT user_email, phone_number FROM orders WHERE user_id = 5;
```

**PostgreSQL Syntax:**

```sql
CREATE INDEX idx_orders_covering ON orders(user_id) INCLUDE (email, status);
```

---

## Indexing Strategies

### Strategy 1: Index for WHERE Clauses

```sql
-- Analyze common queries
SELECT * FROM users WHERE email = ? AND status = ?;
SELECT * FROM users WHERE created_at > ?;

-- Create indexes
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_created ON users(created_at);

-- Better: composite
CREATE INDEX idx_users_email_status ON users(email, status);
```

### Strategy 2: Index for JOIN Columns

```sql
-- Slow: Full table scan on right side
SELECT o.* FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.status = 'active';

-- Solution: Index both sides of join
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_users_id_status ON users(id, status);
```

### Strategy 3: Index for ORDER BY / GROUP BY

```sql
-- Slow: Sort 1M rows in memory
SELECT * FROM orders WHERE user_id = 5 ORDER BY created_at DESC;

-- Solution: Index can provide sorted data
CREATE INDEX idx_orders_user_created ON orders(user_id, created_at DESC);
```

### Strategy 4: Avoid Over-Indexing

```sql
-- ❌ TOO MANY INDEXES
CREATE INDEX idx1 ON users(email);
CREATE INDEX idx2 ON users(phone);
CREATE INDEX idx3 ON users(username);
CREATE INDEX idx4 ON users(email, phone);
-- Slower inserts, more memory, maintenance nightmare

-- ✓ BALANCED
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_phone ON users(phone);
-- Add more only if needed
```

### Strategy 5: Selectivity Matters

**High Selectivity (good index):**

```sql
-- Email: millions of values, each appears once
CREATE INDEX idx_users_email ON users(email);

-- Gender: 2-3 values, each appears many times
-- ❌ Not worth indexing (or use bitmap)
CREATE INDEX idx_users_gender ON users(gender);  -- Bad!

-- Orders per user: 0-1000 per user
-- ✓ Good to index
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

---

## Finding & Analyzing Indexes

### PostgreSQL

```sql
-- All indexes on a table
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'users';

-- Index sizes
SELECT indexname, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_indexes
JOIN pg_class ON indexname = relname
WHERE tablename = 'orders';

-- Unused indexes (candidates for deletion)
SELECT schemaname, tablename, indexname
FROM pg_indexes
WHERE indexname NOT IN (
  SELECT indexrelname FROM pg_stat_user_indexes WHERE idx_scan > 0
);

-- Index scan stats (how often used)
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

### MySQL

```sql
-- All indexes
SELECT TABLE_NAME, INDEX_NAME, COLUMN_NAME, SEQ_IN_INDEX
FROM INFORMATION_SCHEMA.STATISTICS
WHERE TABLE_SCHEMA = 'mydb' AND TABLE_NAME = 'users';

-- Index sizes
SELECT TABLE_NAME, INDEX_NAME, STAT_VALUE * @@innodb_page_size / 1024 / 1024 AS size_mb
FROM mysql.innodb_index_stats
WHERE STAT_NAME = 'size';

-- Unused indexes
SELECT * FROM sys.schema_unused_indexes;
```

---

## Common Mistakes

### 1. Indexing the Wrong Columns

```sql
-- ❌ WRONG
SELECT * FROM users WHERE status = 'active' AND email = ?;
CREATE INDEX idx_users_active ON users(status);  -- Too many rows!

-- ✓ CORRECT
CREATE INDEX idx_users_email ON users(email);  -- More selective
```

### 2. Index Doesn't Match Query

```sql
-- ❌ WRONG
CREATE INDEX idx_users_name ON users(last_name, first_name);
SELECT * FROM users WHERE first_name = 'John';  -- Doesn't use index

-- ✓ CORRECT (leftmost rule)
CREATE INDEX idx_users_name ON users(first_name, last_name);
```

### 3. LIKE Without Prefix Wildcard

```sql
-- Uses index
SELECT * FROM users WHERE name LIKE 'John%';

-- ❌ Doesn't use index (starts with wildcard)
SELECT * FROM users WHERE name LIKE '%John%';  -- Full table scan

-- Solution: Use full-text search for middle wildcards
```

### 4. NULL Handling

```sql
-- Indexes may not include NULLs (depends on DB)
CREATE INDEX idx_users_deleted ON users(deleted_at);

-- ❌ May not use index
SELECT * FROM users WHERE deleted_at IS NULL;

-- ✓ Explicit is better
SELECT * FROM users WHERE deleted_at IS NULL AND status = 'active';
CREATE INDEX idx_users_status_deleted ON users(status, deleted_at);
```

---

## Interview Questions

1. **What's the difference between B-Tree and Hash indexes?**
   - Pattern: B-Tree for ranges, Hash for exact match only

2. **Explain the leftmost rule for composite indexes**
   - Pattern: Order matters, first column must be in WHERE clause

3. **When would you use a partial index?**
   - Pattern: Soft-deleted records, active-only data

4. **How do you find unused indexes?**
   - Pattern: Query system tables (pg_stat_user_indexes, sys.schema_unused_indexes)

5. **Design indexes for this query pattern**
   - Pattern: Analyze WHERE, JOIN, ORDER BY, SELECT columns

---

## Indexing Checklist

```
□ Identified all WHERE clause columns
□ Analyzed JOIN columns
□ Checked ORDER BY / GROUP BY columns
□ Verified selectivity (not indexing low-cardinality)
□ Considered composite index order
□ Measured index size vs benefit
□ Checked for unused indexes
□ Monitored index stats
□ Documented index purpose
□ Tested performance improvement
```

---

## Quick Reference

| Type      | Speed    | Range | Joins | Cost    |
| --------- | -------- | ----- | ----- | ------- |
| B-Tree    | O(log n) | ✓     | ✓     | Medium  |
| Hash      | O(1)     | ❌    | ✓     | Low     |
| Bitmap    | O(1)     | ✓     | ✓     | Low     |
| Full-Text | O(?)     | ✓     | ❌    | Medium  |
| Partial   | O(log n) | ✓     | ✓     | Low     |
| Covering  | O(log n) | ✓     | ✓     | Medium+ |
