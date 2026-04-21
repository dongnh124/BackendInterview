# Performance Tuning & Optimization

Mastering query optimization, indexing, and database tuning for production systems.

## 📚 Core Topics

1. **Query Analysis** — EXPLAIN ANALYZE
2. **Index Design** — Choosing right indexes strategically
3. **Statistics** — Keeping query planner informed
4. **Autovacuum** — Maintenance automation
5. **Lock Management** — Preventing contention
6. **Connection Pooling** — Efficient resource use
7. **Slow Query Detection** — Finding problems
8. **Caching Strategies** — Application-level optimization

## 🔍 Query Analysis Workflow

### Step 1: Identify Slow Queries

```sql
-- PostgreSQL: pg_stat_statements
CREATE EXTENSION pg_stat_statements;

SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### Step 2: Get Query Plan

```sql
EXPLAIN ANALYZE
SELECT o.*, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending'
AND o.created_at > '2024-01-01';
```

### Step 3: Interpret Plan

```
Seq Scan on orders   ← BAD (full table scan)
├─ Filter: (status = 'pending') AND (created_at > ...)
└─ Rows: 50000

vs

Index Scan using idx_orders_status_date on orders  ← GOOD
├─ Index Cond: (status = 'pending') AND (created_at > ...)
└─ Rows: 150
```

**Key metrics to look for:**

- **Seq Scan** → Look for missing index
- **Filter** → Can this be an index condition?
- **Rows** → Estimate vs actual (big difference = stale stats)
- **Buffers** → Hits vs reads (high hits = efficient)

### Step 4: Optimize

```sql
-- Option 1: Add index
CREATE INDEX idx_orders_status_date
ON orders(status, created_at);

-- Option 2: Improve join
-- Maybe users table should have been pre-joined
-- Or use denormalization

-- Option 3: Rewrite query
-- Sometimes query logic can be simplified
```

## 📊 Index Design Principles

### Composite Index Ordering

```sql
-- BAD: Wrong order
CREATE INDEX idx_bad ON orders(created_at, status);

-- Query: WHERE status = 'pending' AND created_at > '2024-01-01'
-- Can't use index efficiently (created_at first in index)

-- GOOD: Equality first, range second
CREATE INDEX idx_good ON orders(status, created_at);

-- Now query uses index efficiently:
-- 1. Find status = 'pending' (index range)
-- 2. Within that range, find created_at > '2024-01-01'
```

### Covering Index

```sql
-- Query only needs 3 columns
SELECT id, status, total FROM orders WHERE user_id = 1;

-- Without covering index:
-- 1. Scan index on user_id
-- 2. Look up table for each row (status, total)

-- With covering index (includes needed columns):
CREATE INDEX idx_covering ON orders(user_id)
INCLUDE (status, total);  -- PostgreSQL 11+

-- Benefits: Index-only scan, no table lookup
```

### Partial Index

```sql
-- Common filter: active users
CREATE INDEX idx_active_users ON users(email)
WHERE status = 'active';

-- Only indexes active users (smaller, faster)
-- Query: SELECT * FROM users WHERE email = 'test@email.com' AND status = 'active'
-- Uses index ✓

-- Query: SELECT * FROM users WHERE email = 'test@email.com' AND status = 'deleted'
-- Doesn't use index (predicate doesn't match) ✗
```

## 🧹 Maintenance Tasks

### Autovacuum Tuning

```sql
-- Why vacuum matters:
-- PostgreSQL uses MVCC (multiple versions)
-- Old versions accumulate (bloat)
-- VACUUM removes dead rows

-- Check current settings:
SHOW autovacuum_vacuum_scale_factor;  -- Default: 0.2 (20% change)
SHOW autovacuum_analyze_scale_factor; -- Default: 0.1 (10% change)
SHOW autovacuum_vacuum_cost_limit;    -- Default: 200

-- For large, high-update tables:
ALTER TABLE hot_table SET (
  autovacuum_vacuum_scale_factor = 0.05,  -- Vacuum at 5% change
  autovacuum_vacuum_cost_limit = 1000      -- More aggressive
);

-- Check bloat:
SELECT schemaname, tablename, pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size
FROM pg_tables
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```

### Statistics Updates

```sql
-- Query planner needs accurate statistics
-- To estimate rows correctly

-- Manual update:
ANALYZE;  -- Update stats on all tables

-- For specific table:
ANALYZE hot_table;

-- Check stats age:
SELECT schemaname, tablename, last_vacuum, last_autovacuum, last_analyze
FROM pg_stat_user_tables
ORDER BY last_analyze ASC;
```

## 🔒 Lock Management

### Identify Long-running Queries

```sql
-- PostgreSQL: Find blocking queries
SELECT pid, usename, state, wait_event_type, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY backend_start ASC;

-- Kill problematic query:
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE pid = 12345;
```

### Deadlock Detection

```sql
-- Log deadlocks:
SET log_min_duration_statement = 1000;  -- Log slow queries
-- Look for deadlock messages in log

-- Reduce deadlock chance:
-- 1. Access tables in same order always
-- 2. Keep transactions short
-- 3. Use appropriate indexes (reduce lock footprint)
-- 4. Lower isolation level if acceptable
```

## 🌡️ Connection Pool Sizing

### Formula

```
connections = ((core_count * 2) + effective_disk_count)

Example: 4-core CPU, 1 disk = (4 * 2) + 1 = 9 connections

Rule of thumb: Start small (5-10), increase based on monitoring
```

### PgBouncer Configuration

```ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction         -- One connection per transaction
max_client_conn = 1000          -- Max clients
default_pool_size = 25          -- Connections to backend
min_pool_size = 5               -- Minimum connections
```

## 📈 Performance Dashboard Metrics

| Metric                | Query                              | Target  | Alert    |
| --------------------- | ---------------------------------- | ------- | -------- |
| Slow queries (> 1s)   | pg_stat_statements                 | < 5     | > 10     |
| Sequential scans      | pg_stat_user_tables                | < 100   | > 1000   |
| Index bloat           | pg_stat_user_indexes               | < 50%   | > 70%    |
| Autovacuum lag        | pg_stat_user_tables                | < 1 day | > 3 days |
| Connection pool usage | current_setting('max_connections') | < 80%   | > 90%    |
| Replication lag       | pg_last_wal_receive_lsn()          | < 1 sec | > 5 sec  |

## 🔧 Common Optimization Scenarios

### N+1 Query Problem

```sql
-- BAD: 1 + N queries
SELECT * FROM orders;
-- For each order: SELECT * FROM users WHERE id = ?

-- GOOD: Single join
SELECT o.*, u.name FROM orders o
JOIN users u ON o.user_id = u.id;
```

### Avoid Functions on Indexed Columns

```sql
-- BAD: Can't use index
SELECT * FROM users WHERE LOWER(email) = 'test@email.com';

-- GOOD: Use functional index
CREATE INDEX idx_email_lower ON users(LOWER(email));

-- Or better: normalize data at insert
SELECT * FROM users WHERE email = 'test@email.com';
```

### Pagination at Scale

```sql
-- SLOW: Large offset
SELECT * FROM orders ORDER BY id LIMIT 10 OFFSET 10000;

-- FAST: Keyset pagination
SELECT * FROM orders WHERE id > last_id ORDER BY id LIMIT 10;
```

## ✅ Optimization Checklist

- [ ] Most-run queries analyzed with EXPLAIN
- [ ] Appropriate indexes created (no unused indexes)
- [ ] Statistics up-to-date (ANALYZE completed)
- [ ] Autovacuum tuned for workload
- [ ] Connection pool right-sized
- [ ] Long-running queries identified and optimized
- [ ] No N+1 query patterns
- [ ] Replication lag monitored
- [ ] Slow query log enabled and reviewed
- [ ] Performance baselines documented

## 📖 Advanced Topics

- [ ] Query plan cache and prepared statements
- [ ] Bloom filters and JIT compilation
- [ ] Partitioning strategy
- [ ] Materialized views for complex reports
- [ ] Column-level statistics
- [ ] Cost-based optimizer tuning
- [ ] Parallel query execution

## 🔗 Related

- [Fundamentals](../01-fundamentals/)
- [Monitoring](../08-monitoring/)
- [PostgreSQL Guide](../07-platform-guides/postgresql.md)

---

**Key Insight:** The best index is the one that prevents full table scans for your most-run queries. Always measure before and after optimizing to ensure improvement is real.
