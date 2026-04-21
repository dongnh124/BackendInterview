# Platform Guides

Database-specific deep dives and operational guidance.

## 🗂️ Available Platforms

### PostgreSQL

- [ ] [Full Guide](./postgresql.md)
- WAL, MVCC, Replication
- Extensions ecosystem
- Performance tuning (autovacuum, vacuum_analyze)
- Scaling strategies (sharding, partitioning)

**When to use:** Complex queries, JSON data, geospatial (PostGIS), want advanced features
**Typical workload:** Analytics, microservices, general purpose
**Companies:** Spotify, Airbnb, Instagram

### MySQL / InnoDB

- [ ] [Full Guide](./mysql.md)
- Storage engine internals
- Replication setup (Master-Slave, Group)
- Cluster options (Galera, NDB)
- Performance tuning

**When to use:** Web applications, simplicity, wide ecosystem support
**Typical workload:** Content management, e-commerce, web apps
**Companies:** Facebook, Twitter, Uber (initially)

### SQL Server

- [ ] [Full Guide](./sqlserver.md)
- Always On Availability Groups
- T-SQL specific features
- DMVs and query store
- Windows authentication

**When to use:** Enterprise .NET ecosystems, Windows servers
**Typical workload:** Enterprise business apps, reporting
**Companies:** Microsoft, Enterprise .NET shops

### MongoDB

- [ ] [Full Guide](./mongodb.md)
- Document model & BSON
- Sharding & replication
- Aggregation pipeline
- Index strategies

**When to use:** Flexible schema, rapid development, horizontal scaling
**Typical workload:** Mobile apps, real-time analytics, content platforms
**Companies:** Uber (current), Sap Concur, Dropbox

### Redis

- [ ] [Full Guide](./redis.md)
- In-memory data structures
- Persistence options
- Replication & cluster
- Pub/Sub, transactions

**When to use:** Caching, sessions, rate limiting, real-time features
**Typical workload:** Cache layer, session store, leaderboards
**Companies:** GitHub, Pinterest, Shopify

### Cloud-Managed Options

- [ ] [AWS RDS Guide](./aws-rds.md)
- [ ] [GCP Cloud SQL Guide](./gcp-cloud-sql.md)
- [ ] [Azure SQL Guide](./azure-sql.md)

---

## 🔄 Comparison Matrix

| Aspect               | PostgreSQL         | MySQL            | SQL Server          | MongoDB               | Redis                |
| -------------------- | ------------------ | ---------------- | ------------------- | --------------------- | -------------------- |
| **Type**             | RDBMS              | RDBMS            | RDBMS               | Document              | In-Memory            |
| **Scaling**          | Vertical, Sharding | Replication      | Availability Groups | Sharding              | Replication, Cluster |
| **ACID**             | Full               | Full             | Full                | Multi-doc (recent)    | Limited              |
| **Schema**           | Strict             | Strict           | Strict              | Flexible              | N/A                  |
| **Query Complexity** | Very complex       | Moderate         | Complex             | Limited (aggregation) | Simple (key-value)   |
| **Throughput**       | High               | Very high        | High                | Very high             | Extreme              |
| **Latency**          | Low                | Very low         | Low                 | Low                   | Ultra-low            |
| **Learn Time**       | Medium             | Easy             | Hard                | Easy                  | Easy                 |
| **Best For**         | Analytics, Complex | Web apps, Simple | Enterprise, .NET    | Rapid dev, Flexible   | Cache, Real-time     |

---

## 🎯 Decision Tree

```
Is your data schema changing frequently?
├─ Yes → MongoDB (flexible schema) or PostgreSQL (still flexible)
└─ No → Any RDBMS

Do you need complex queries / joins?
├─ Yes → PostgreSQL or SQL Server
└─ No → MySQL or MongoDB

Is this primarily a cache/session store?
├─ Yes → Redis
└─ No → Primary database

Do you need sub-millisecond latency?
├─ Yes → Redis
└─ No → Any other option

Do you need horizontal scaling?
├─ Yes → MongoDB (native sharding) or PostgreSQL (sharding yourself)
└─ No → Any RDBMS (vertical scaling fine)

Are you in a .NET environment?
├─ Yes → SQL Server
└─ No → PostgreSQL or MySQL more common

Is this a startup (rapid iteration)?
├─ Yes → PostgreSQL or MongoDB
└─ No → Depends on workload

Is cost critical?
├─ Yes → PostgreSQL or MySQL (open-source)
└─ No → Any, choose best fit
```

---

## 📚 What's in Each Guide

### PostgreSQL

```
1. Installation & setup
2. Architecture (MVCC, WAL, heap)
3. Configuration parameters
4. Performance tuning (autovacuum, shared_buffers)
5. Replication (streaming, logical)
6. Backup & recovery (pg_dump, WAL archiving)
7. Extensions (PostGIS, uuid-ossp, hstore)
8. High availability (Patroni, etcd)
9. Monitoring (pg_stat_statements)
10. Common issues & solutions
```

### MySQL

```
1. Installation & setup
2. Storage engines (InnoDB, MyISAM)
3. Configuration (my.cnf)
4. Performance tuning (buffer pool, key cache)
5. Replication (master-slave, group)
6. Backup & recovery (mysqldump, binary logs)
7. High availability (MHA, Percona XtraDB Cluster)
8. Security (user privileges, SSL)
9. Monitoring (slow log, Performance Schema)
10. Common issues & solutions
```

### SQL Server

```
1. Installation & Express vs Enterprise
2. Architecture (log buffer, buffer pool)
3. Configuration (SQL Server Management Studio)
4. Performance tuning (indexes, statistics, DMVs)
5. Always On Availability Groups (AOAG)
6. Backup & recovery (full, differential, log)
7. Security (Windows auth, SQL auth, TDE)
8. Monitoring (Query Store, Extended Events)
9. T-SQL specifics (CTE, window functions)
10. Common issues & solutions
```

### MongoDB

```
1. Installation & setup
2. Document model & BSON
3. Replication (replica sets)
4. Sharding (chunk distribution)
5. Indexing (single, compound, geo)
6. Aggregation pipeline
7. Transactions (single-doc, multi-doc)
8. Backup & recovery (mongodump, snapshots)
9. Performance tuning (explain plans)
10. Common issues & solutions
```

### Redis

```
1. Installation & setup
2. Data structures (string, hash, list, set, zset)
3. Persistence (RDB, AOF)
4. Replication & cluster
5. Lua scripting
6. Pub/Sub messaging
7. Transactions & watch
8. Memory management (eviction policies)
9. Monitoring & slow log
10. Common issues & solutions
```

---

## 💡 Usage Guide

### For Learning

```
1. Read the Comparison Matrix to understand trade-offs
2. Use Decision Tree for your use case
3. Read the specific platform guide(s)
4. Follow the "Getting Started" section
5. Do hands-on labs
```

### For System Design Interviews

```
1. Clarify requirements (scale, consistency, write-heavy?)
2. Use Decision Tree or Comparison Matrix
3. Explain your choice with trade-offs
4. Mention backup/HA strategy
5. Discuss monitoring approach
```

### For Operational Issues

```
1. Go to your platform guide
2. Find "Common Issues" section
3. Follow troubleshooting steps
4. Document findings for future reference
```

---

## 🚀 Next Steps

Choose your primary platform and deep dive:

### Option 1: Generalist

- Learn PostgreSQL (most features, good default choice)
- Learn MySQL (different approach, good for scale)
- Understand trade-offs between them

### Option 2: Specialist

- Master one platform deeply (PostgreSQL recommended)
- Understand when to use alternatives
- Know that platform's ecosystem well

### Option 3: Role-Specific

- **Backend Engineer**: PostgreSQL + MongoDB
- **Data Engineer**: PostgreSQL + Spark
- **DevOps**: All platforms for operations
- **Site Reliability Engineer**: All platforms for scale

---

## 📖 Additional Resources

### Official Documentation

- [PostgreSQL](https://www.postgresql.org/docs/)
- [MySQL](https://dev.mysql.com/doc/)
- [SQL Server](https://docs.microsoft.com/sql/)
- [MongoDB](https://docs.mongodb.com/)
- [Redis](https://redis.io/documentation)

### Essential Books

- Database Internals by Alex Petrov
- PostgreSQL 14 Internals by Egor Bogomolov
- MySQL 8.0 Reference Manual
- MongoDB: The Definitive Guide

### Blogs & Newsletters

- Use The Index, Luke! (query optimization)
- PostgreSQL Weekly
- Planet MySQL
- High Scalability
- AWS Database Blog

---

## 🔗 Quick Reference

| Platform   | Port  | Default DB | CLI Tool  |
| ---------- | ----- | ---------- | --------- |
| PostgreSQL | 5432  | postgres   | psql      |
| MySQL      | 3306  | mysql      | mysql     |
| SQL Server | 1433  | master     | sqlcmd    |
| MongoDB    | 27017 | admin      | mongosh   |
| Redis      | 6379  | (none)     | redis-cli |

---

**Key Insight:** There's no "best" database—only the best choice for your specific constraints (latency, throughput, consistency, cost, team expertise, existing infrastructure). Understand the trade-offs.
