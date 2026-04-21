# Connection Pooling

## Overview

Connection pooling reduces overhead by reusing database connections instead of creating new ones for each request. Critical for performance in high-concurrency systems.

---

## Why Connection Pooling Matters

### Cost of a Fresh Connection

```
Creating new TCP connection:
1. TCP handshake (3 packets)       → ~1 ms
2. SSL/TLS negotiation             → ~5 ms
3. Authentication                  → ~1 ms
4. Database initialization         → ~1 ms
─────────────────────────
Total: ~10 ms per connection

vs.

Reusing pooled connection:
1. Checkout from pool               → <1 ms
```

**Impact on web server:**

```
100 concurrent requests × 10 ms per connection = 1 second overhead
With pooling: ~50 ms overhead
→ 20x improvement in connection overhead
```

### Resource Exhaustion

```
PostgreSQL max_connections = 100
Without pooling:
  100 concurrent requests → 100 connections
  101st request → Connection refused (ERROR)

With pooling (size=20):
  100 concurrent requests → 20 pooled connections (queued)
  101st request → Wait for connection to free up
```

---

## How Connection Pooling Works

### Architecture

```
Application Servers (many)
        ↓
Connection Pool (e.g., PgBouncer)
    [conn1][conn2][conn3]
        ↓
Database Server (limited connections)
```

### Connection States

```
Pool: [IDLE] [IDLE] [IDLE] [BUSY] [BUSY]
       Ready to use, waiting    In use
```

**Request flow:**

```
1. App requests connection
2. Pool checks: Idle connection available?
   YES → Return immediately
   NO → Queue request, wait for next idle
3. App uses connection
4. App closes connection (returns to pool)
5. Pool marks connection idle (ready for next app)
```

---

## Types of Connection Pooling

### 1. Application-Level Pooling

**Location:** Inside application server (Node.js, Python, Java)

**Examples:**

- HikariCP (Java)
- Node-postgres pool
- SQLAlchemy (Python)
- Django database pool

**Example (Node.js with node-postgres):**

```javascript
const pool = new Pool({
  user: "postgres",
  password: "secret",
  host: "localhost",
  port: 5432,
  database: "myapp",
  max: 20, // Pool size
  idleTimeoutMillis: 30000, // Close idle after 30s
  connectionTimeoutMillis: 2000,
});

app.get("/users/:id", async (req, res) => {
  const connection = await pool.connect();
  try {
    const result = await connection.query("SELECT * FROM users WHERE id = $1", [
      req.params.id,
    ]);
    res.json(result.rows[0]);
  } finally {
    connection.release(); // Return to pool
  }
});
```

**Pros:**

- ✓ Simple (library handles it)
- ✓ Low latency (local)
- ✓ Works with any database

**Cons:**

- ❌ Each app instance has its own pool (connection multiplication)
- ❌ Difficult to monitor across servers
- ❌ Each app needs overhead

---

### 2. Proxy-Level Pooling (Middleware)

**Location:** Between application and database (separate service)

**Examples:**

- PgBouncer (PostgreSQL)
- ProxySQL (MySQL)
- Pgpool-II (PostgreSQL)

**Architecture:**

```
[App Server 1] ┐
[App Server 2] ├─→ [Connection Pool] → [PostgreSQL]
[App Server 3] ┘      (PgBouncer)
```

**Example (PgBouncer configuration):**

```ini
[databases]
mydb = host=localhost port=5432 dbname=myapp

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
reserve_pool_size = 5
reserve_pool_timeout = 3
```

**Pros:**

- ✓ Single pool for all apps (connection efficiency)
- ✓ Central monitoring
- ✓ Transparent to application
- ✓ Can connect from many servers

**Cons:**

- ⚠️ Extra network hop (slightly higher latency)
- ⚠️ Requires additional infrastructure
- ⚠️ Complexity increases

---

## Pool Sizing

### Formula

```
Pool Size = ((Core Count * 2) + Effective Spindle Count)

Example:
- 8 core CPU, 1 disk
- Pool Size = (8 * 2) + 1 = 17
- Use 10-15 for safety
```

**Better rule of thumb:**

```
Pool Size ≈ (Expected Concurrent Queries) + Buffer

Example:
- Peak traffic: 50 concurrent requests
- DB query time: 50ms average
- Requests per connection: 50 / 0.05 = 1000 requests/sec
- Peak concurrent = 50 requests
- Pool = 50 + 5 (buffer) = 55

Reality: Start with 20-25, monitor, adjust
```

### Under-Sizing vs Over-Sizing

| Size            | Problem                | Symptom                                   |
| --------------- | ---------------------- | ----------------------------------------- |
| Too small (5)   | Connection queue grows | Requests timeout waiting for connection   |
| Optimal (20-25) | Efficient resource use | Balanced latency + DB load                |
| Too large (200) | Wasted resources       | High idle connections, DB memory pressure |

**Monitoring:** Track queue depth

```
If queue always empty: Size too large
If queue frequently filled: Size too small
```

---

## Pool Configuration

### Key Parameters

```
connection_timeout    → How long to wait for available connection
idle_timeout         → How long to keep idle connections
max_idle             → Max connections to keep idle
queue_timeout        → How long request waits in queue
validation_query     → Check connection health before use
```

### Example (HikariCP - Java)

```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
config.setUsername("postgres");
config.setPassword("secret");

// Pool sizing
config.setMaximumPoolSize(20);           // Max connections
config.setMinimumIdle(5);                // Min idle to keep ready

// Timeouts
config.setConnectionTimeout(30000);      // 30 seconds
config.setIdleTimeout(600000);           // 10 minutes
config.setMaxLifetime(1800000);          // 30 minutes

// Health checks
config.setConnectionTestQuery("SELECT 1");
config.setLeakDetectionThreshold(60000); // Warn on 60s+ open connections

HikariDataSource ds = new HikariDataSource(config);
```

### Common Issues & Fixes

| Issue                    | Cause                           | Fix                                        |
| ------------------------ | ------------------------------- | ------------------------------------------ |
| Connections leak         | App doesn't return connection   | Enable leak detection warnings             |
| Queue timeout            | Pool too small                  | Increase pool size                         |
| Idle timeout errors      | Connection stayed idle too long | Lower idle_timeout or use validation_query |
| Max connections exceeded | Pool + direct connections       | Stop direct connections                    |

---

## Pool Modes

### Transaction Mode (Recommended)

**Rule:** Pool connection per transaction (not per session)

```
Client A: BEGIN → GET conn1 → QUERY → COMMIT → RETURN conn1
Client B: BEGIN → GET conn1 (wait) → QUERY → COMMIT → RETURN conn1
```

**Pros:**

- ✓ Maximizes connection reuse
- ✓ Allows many clients with few connections
- ✓ Solves connection limit problem

**Implementation (PgBouncer):**

```ini
[pgbouncer]
pool_mode = transaction
```

**Gotchas:**

```sql
-- ❌ Won't work with prepared statements across transactions
PREPARE stmt AS SELECT * FROM users WHERE id = $1;
EXECUTE stmt;
-- Connection returned to pool
EXECUTE stmt;  -- ERROR: Prepared statement not found!

-- ✓ Solution: Use parameterized queries per transaction
SELECT * FROM users WHERE id = $1;  -- Fresh prepare each time
```

---

### Session Mode

**Rule:** Pool connection per session (entire connection lifecycle)

```
Client A: GET conn1 → QUERY → QUERY → QUERY → RETURN conn1 (when done)
Client B: Wait for conn1
```

**Pros:**

- ✓ Full PostgreSQL feature support
- ✓ Prepared statements work across transactions
- ✓ Session variables preserved

**Cons:**

- ❌ Less connection reuse (connection per client)

**Implementation (PgBouncer):**

```ini
[pgbouncer]
pool_mode = session
```

---

## Monitoring Connection Pools

### Key Metrics

```
Active connections:   Currently executing queries
Idle connections:     Waiting for requests
Queued requests:      Waiting for connection
Connection errors:    Failed acquisitions
Leak count:          Connections not returned
```

### PostgreSQL (PgBouncer) Monitoring

```sql
-- Connect to PgBouncer admin console
psql -U pgbouncer -d pgbouncer -h localhost -p 6432

-- View pool status
SHOW pools;

-- Per-database stats
SHOW stats;

-- Connection details
SHOW clients;
SHOW servers;

-- Reload config
RELOAD;
```

### Application-Level (HikariCP)

```java
HikariDataSource ds = new HikariDataSource(config);

// Get current pool stats
System.out.println("Active connections: " + ds.getHikariPoolMXBean().getActiveConnections());
System.out.println("Idle connections: " + ds.getHikariPoolMXBean().getIdleConnections());
System.out.println("Pending threads: " + ds.getHikariPoolMXBean().getThreadsAwaitingConnection());
```

---

## Best Practices

### 1. Choose the Right Pool Type

```
Single server app?         → Application-level pooling (HikariCP)
Multiple app servers?      → Proxy pooling (PgBouncer)
High connection churn?     → Use transaction mode
Need prepared statements?  → Use session mode
```

### 2. Right-Size the Pool

```
Start with: (CPU cores * 2) + spindles
Monitor: Queue depth + idle time
Adjust: Increase if queue fills, decrease if idle > 50%
```

### 3. Handle Connection Failures

```python
# ❌ WRONG
def get_user(user_id):
    conn = pool.get_connection()
    result = conn.query("SELECT * FROM users WHERE id = %s", user_id)
    conn.close()  # Doesn't return to pool on error!
    return result

# ✓ CORRECT
def get_user(user_id):
    conn = pool.get_connection()
    try:
        result = conn.query("SELECT * FROM users WHERE id = %s", user_id)
        return result
    finally:
        conn.close()  # Always returns to pool
```

### 4. Monitor Pool Health

```
□ Track connection acquisition time
□ Monitor queue depth
□ Alert on timeouts
□ Check for connection leaks
□ Review idle connection percentage
□ Test failover scenarios
```

---

## Common Mistakes

### 1. Pool Too Small

```
Symptom: "Unable to acquire connection" errors
→ Increase pool size
```

### 2. Connection Leaks

```javascript
// ❌ WRONG
app.get('/users', async (req, res) => {
  const conn = await pool.connect();
  if (!user_id) {
    res.send(400);  // Forgot to release!
  }
  const result = await conn.query(...);
  conn.release();
  res.json(result);
});

// ✓ CORRECT
app.get('/users', async (req, res) => {
  const conn = await pool.connect();
  try {
    if (!user_id) {
      return res.send(400);
    }
    const result = await conn.query(...);
    res.json(result);
  } finally {
    conn.release();  // Always called
  }
});
```

### 3. Prepared Statement Mode Issues

```sql
-- ❌ WRONG (with transaction mode pooling)
PREPARE my_stmt AS SELECT * FROM users WHERE id = $1;
-- Connection returned to pool after transaction

EXECUTE my_stmt(5);  -- ERROR: Prepared statement not found!
```

### 4. Long Transactions Hold Connections

```python
# ❌ WRONG
with db.connection() as conn:
    data = conn.query("SELECT * FROM big_table")
    # Long processing in app (5 seconds)
    time.sleep(5)
    # Connection held entire time!
    conn.execute("UPDATE table SET processed = true")

# ✓ CORRECT
data = db.query("SELECT * FROM big_table")
# Process data in app (5 seconds)
time.sleep(5)
# No connection held
with db.connection() as conn:
    conn.execute("UPDATE table SET processed = true")
```

---

## Interview Questions

1. **Why is connection pooling important?**
   - Pattern: Cost per connection creation + resource exhaustion prevention

2. **What's the difference between transaction and session mode pooling?**
   - Pattern: Transaction mode reuses per transaction, session mode per client

3. **How do you size a connection pool?**
   - Pattern: Formula + monitoring queue depth

4. **Design a pooling strategy for microservices**
   - Pattern: Proxy pooling (PgBouncer) vs app-level + considerations

5. **What causes connection leaks and how do you prevent them?**
   - Pattern: Not releasing in error paths + use try/finally

---

## Quick Reference

```
Metric                  Good Range      Warning
─────────────────────────────────────────────────
Pool Size               10-25           < 5 or > 100
Queue Depth             Mostly 0        Consistently > 5
Connection Acquire ms   < 1 ms          > 10 ms
Idle Connections       20-50%          < 5% or > 80%
Leak Detection Events   0               > 0 (investigate)
```
