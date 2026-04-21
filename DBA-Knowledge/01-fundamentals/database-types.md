# Database Types & Selection

## Overview

Different database types serve different purposes. Choosing the right one is critical for system performance and scalability.

---

## Database Categories

### 1. Relational (RDBMS) 📊

**Characteristics:**

- Structured data (tables, rows, columns)
- ACID guarantees
- SQL language
- Joins across tables
- Indexes for performance

**Best for:**

- ✓ Transactional systems (banking, orders)
- ✓ Complex queries with joins
- ✓ Data integrity is critical
- ✓ Structured, well-defined schema

**Popular Options:**

| Database       | Pros                                       | Cons                          | Best For                     |
| -------------- | ------------------------------------------ | ----------------------------- | ---------------------------- |
| **PostgreSQL** | Advanced features, JSONB, full-text search | Slightly slower writes        | Complex queries, startups    |
| **MySQL**      | Fast reads, simple, common                 | Limited features, slow writes | Web apps, standard workloads |
| **SQL Server** | Enterprise features, Windows integration   | Expensive licensing           | Large enterprises            |
| **Oracle**     | Maximum scalability, performance           | Very expensive, complex       | Financial institutions       |

**Example (PostgreSQL):**

```sql
-- Structured schema
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email VARCHAR(255) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INTEGER REFERENCES users(id),
  total DECIMAL(10, 2),
  created_at TIMESTAMP
);

-- ACID transaction
BEGIN;
  INSERT INTO users VALUES (DEFAULT, 'john@example.com', NOW());
  INSERT INTO orders VALUES (DEFAULT, 1, 99.99, NOW());
COMMIT;

-- Complex join
SELECT u.email, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id
HAVING COUNT(o.id) > 5;
```

---

### 2. NoSQL - Document (MongoDB, Firebase)

**Characteristics:**

- Flexible schema (JSON documents)
- Horizontal scaling
- Denormalized data
- No ACID (initially), single-document ACID (4.0+)
- Query language varies

**Best for:**

- ✓ Flexible/evolving schema
- ✓ Semi-structured data
- ✓ Horizontal scaling
- ✓ Real-time applications

**MongoDB Example:**

```javascript
// Flexible schema
db.users.insertOne({
  _id: ObjectId("..."),
  email: "john@example.com",
  profile: {
    firstName: "John",
    lastName: "Doe",
  },
  tags: ["premium", "early-adopter"],
  settings: {
    notifications: true,
    theme: "dark",
  },
});

// Query with aggregation
db.users.aggregate([
  { $match: { "profile.firstName": "John" } },
  { $group: { _id: "$tags", count: { $sum: 1 } } },
  { $sort: { count: -1 } },
]);

// Multi-document ACID (4.0+)
session = db.getMongo().startSession();
session.startTransaction();
db.users.updateOne({ _id: 1 }, { $inc: { balance: -100 } }, { session });
db.orders.insertOne({ user_id: 1, amount: 100 }, { session });
session.commitTransaction();
```

**Pros:**

- ✓ Schema flexibility (good for rapid development)
- ✓ Horizontal scaling
- ✓ Fast reads (denormalized)

**Cons:**

- ❌ Data duplication
- ❌ No joins (must denormalize)
- ❌ Eventual consistency issues
- ❌ Complex multi-document transactions

---

### 3. NoSQL - Key-Value (Redis, Memcached)

**Characteristics:**

- Simple key → value mapping
- In-memory (usually)
- Very fast
- Limited query capabilities
- TTL support

**Best for:**

- ✓ Caching
- ✓ Session storage
- ✓ Rate limiting
- ✓ Real-time data (leaderboards, counts)
- ✓ Message queues

**Redis Example:**

```bash
# Simple key-value
SET user:1 '{"name":"John","email":"john@example.com"}'
GET user:1

# Counters
INCR page:views
GET page:views  # 1
INCR page:views
GET page:views  # 2

# Expiration (cache)
SET session:abc123 '{...}' EX 3600  # Expires in 1 hour

# Data structures
LPUSH queue:jobs '{"task":"email"}'
RPOP queue:jobs

# Leaderboard
ZADD leaderboard 100 user:1
ZADD leaderboard 150 user:2
ZRANGE leaderboard 0 -1 WITHSCORES
# user:1 100, user:2 150
```

**Pros:**

- ✓ Extremely fast (in-memory)
- ✓ Simple operations
- ✓ Atomic operations

**Cons:**

- ❌ Limited query language
- ❌ Data volatility (must backup)
- ❌ Not suitable for primary data store
- ❌ Memory limitations

---

### 4. NoSQL - Time-Series (InfluxDB, TimescaleDB)

**Characteristics:**

- Optimized for timestamped data
- Compression
- Fast aggregations
- Special time-based queries

**Best for:**

- ✓ Metrics (CPU, memory)
- ✓ Monitoring data
- ✓ Financial charts
- ✓ IoT sensor data

**Example (InfluxDB):**

```javascript
// Write metric
write_api.write(
  (bucket = "monitoring"),
  (record = Point("cpu_usage")
    .tag("host", "server1")
    .field("value", 75.5)
    .time(time.time())),
);

// Query
query_api.query(
  (org = "myorg"),
  (query = `
    from(bucket:"monitoring")
    |> range(start: -1h)
    |> filter(fn: (r) => r._measurement == "cpu_usage")
    |> aggregateWindow(every: 5m, fn: mean)
  `),
);
```

---

### 5. Search (Elasticsearch)

**Characteristics:**

- Full-text search optimized
- Complex text queries
- Distributed
- JSON documents
- Real-time indexing

**Best for:**

- ✓ Full-text search
- ✓ Log analysis
- ✓ Complex text queries
- ✓ Autocomplete

**Example:**

```json
// Index document
PUT /products/_doc/1
{
  "name": "PostgreSQL Database",
  "description": "High-performance relational database",
  "tags": ["database", "relational", "open-source"]
}

// Full-text search
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "database performance",
      "fields": ["name^2", "description"]
    }
  }
}
```

---

## Selection Decision Tree

```
Do you need ACID?
├─ YES → RDBMS (PostgreSQL/MySQL/SQL Server)
├─ NO → Continue...

Do you need complex queries?
├─ YES → RDBMS or Search (Elasticsearch)
├─ NO → Continue...

Is data structured & fixed?
├─ YES → RDBMS
├─ NO → Continue...

Need horizontal scaling?
├─ YES → NoSQL (MongoDB, Cassandra)
├─ NO → RDBMS

Data is timestamped metrics?
├─ YES → Time-series DB (InfluxDB)
├─ NO → Continue...

Need full-text search?
├─ YES → Elasticsearch
├─ NO → Continue...

Caching/sessions?
├─ YES → Redis
└─ NO → Evaluate with team
```

---

## RDBMS vs NoSQL Trade-offs

| Aspect             | RDBMS               | NoSQL                |
| ------------------ | ------------------- | -------------------- |
| **Schema**         | Fixed               | Flexible             |
| **Consistency**    | ACID                | Eventual             |
| **Scalability**    | Vertical (hard)     | Horizontal (easy)    |
| **Joins**          | Fast                | Manual (denormalize) |
| **Transactions**   | Strong              | Limited/None         |
| **Query Language** | SQL (standard)      | Varies               |
| **Learning Curve** | Moderate            | Depends on type      |
| **Cost**           | Lower (open-source) | Varies               |

---

## Platform-Specific Strengths

### PostgreSQL

```
✓ Best relational + JSON support
✓ Full-text search (built-in)
✓ Advanced features (arrays, ranges, custom types)
✓ Great for complex queries
✓ Excellent documentation

Ideal for: Web applications, data warehousing, complex analytics
```

### MySQL

```
✓ Fast, simple, reliable
✓ Great for read-heavy workloads
✓ Easiest to scale horizontally (sharding)
✓ Lower resource requirements

Ideal for: High-traffic web apps, fast reads
```

### MongoDB

```
✓ Easy to start with (flexible schema)
✓ Horizontal scaling built-in
✓ Multi-document transactions (4.0+)
✓ Good for real-time data

Ideal for: Rapid development, flexible data, microservices
```

### Redis

```
✓ Fastest possible (in-memory)
✓ Perfect for caching/sessions
✓ Atomic operations
✓ Message queues

Ideal for: Caching, sessions, real-time features
```

---

## Common Combinations

### Web Application Architecture

```
┌─ PostgreSQL (main database)
│  Primary: User data, orders, transactions
│
├─ Redis (cache)
│  Session data, frequently accessed objects
│
├─ Elasticsearch (search)
│  Full-text search on products/content
│
└─ TimescaleDB (metrics)
   Application metrics, monitoring
```

### Microservices Pattern

```
Service A: PostgreSQL (orders)
Service B: MongoDB (user profiles)
Service C: Cassandra (event log)
Shared: Redis (cache)
```

---

## Interview Questions

1. **Compare RDBMS vs NoSQL. When would you use each?**
   - Pattern: ACID + structured → RDBMS, flexible + scale → NoSQL

2. **Why would you choose MongoDB over PostgreSQL?**
   - Pattern: Schema flexibility, easier horizontal scaling, real-time data

3. **Design the database selection for [specific system]**
   - Pattern: Analyze requirements, map to database strengths

4. **What are limitations of NoSQL?**
   - Pattern: No joins, eventual consistency, no ACID, complex transactions

5. **When is caching critical, and which technology would you use?**
   - Pattern: High reads, low writes → Redis + RDBMS

---

## Practical Comparison Lab

### Same Data in Different Databases

**PostgreSQL (Relational):**

```sql
CREATE TABLE users (id SERIAL PRIMARY KEY, email VARCHAR, name VARCHAR);
CREATE TABLE posts (id SERIAL PRIMARY KEY, user_id INT REFERENCES users);

INSERT INTO users VALUES (1, 'john@example.com', 'John');
INSERT INTO posts VALUES (1, 1, 'My first post');

SELECT u.name, COUNT(p.id) FROM users u LEFT JOIN posts p ON u.id = p.user_id GROUP BY u.id;
```

**MongoDB (Document):**

```javascript
db.users.insertOne({
  _id: 1,
  email: "john@example.com",
  name: "John",
  posts: [1], // Denormalized
});

db.posts.insertOne({
  _id: 1,
  user_id: 1,
  title: "My first post",
});

// Must manually join
```

**Key Difference:**

- PostgreSQL: Normalized, join on query
- MongoDB: Denormalized, pre-calculated in document

---

## Quick Reference

```
Need ACID?           → PostgreSQL
Need scale?          → Cassandra/MongoDB
Need speed (cache)?  → Redis
Need full-text?      → Elasticsearch
Need timeseries?     → InfluxDB
Default choice?      → PostgreSQL + Redis
```
