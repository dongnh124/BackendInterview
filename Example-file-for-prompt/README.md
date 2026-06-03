# 🗄️ Database Administrator (DBA) Knowledge Roadmap

> Comprehensive guide to Database Administration covering all core competencies from fundamentals to advanced operations.

## 📚 Table of Contents

1. [Learning Path](#learning-path)
2. [Core Competencies](#core-competencies)
3. [Database Platforms](#database-platforms)
4. [Topics Overview](#topics-overview)

---

## 🎯 Learning Path

### **Phase 1: Fundamentals (Weeks 1-2)**

- [ ] Database Basics & ACID Properties
- [ ] Transaction Management & Isolation Levels
- [ ] Indexing Strategies
- [ ] Basic Monitoring

### **Phase 2: Core DBA Skills (Weeks 3-6)**

- [ ] Backup & Recovery (RPO/RTO)
- [ ] High Availability & Replication
- [ ] Performance Tuning & Optimization
- [ ] Security & Access Control

### **Phase 3: Advanced Operations (Weeks 7-10)**

- [ ] Schema Migration & Change Management
- [ ] Capacity Planning
- [ ] Incident Response & Troubleshooting
- [ ] Cloud Database Management

### **Phase 4: Specialization (Weeks 11+)**

- [ ] Platform-specific Deep Dives
- [ ] Advanced Replication Strategies
- [ ] Database Design Patterns
- [ ] Compliance & Governance

---

## 🏢 Core Competencies

| Competency                     | Priority | Time    | Status |
| ------------------------------ | -------- | ------- | ------ |
| **Backup & Disaster Recovery** | ⭐⭐⭐   | 2 weeks | -      |
| **Performance Tuning**         | ⭐⭐⭐   | 3 weeks | -      |
| **High Availability**          | ⭐⭐⭐   | 2 weeks | -      |
| **Security & Compliance**      | ⭐⭐⭐   | 2 weeks | -      |
| **Replication & Failover**     | ⭐⭐⭐   | 2 weeks | -      |
| **Monitoring & Alerting**      | ⭐⭐⭐   | 1 week  | -      |
| **Schema Migrations**          | ⭐⭐⭐   | 2 weeks | -      |
| **Troubleshooting**            | ⭐⭐⭐   | 2 weeks | -      |
| **Capacity Planning**          | ⭐⭐     | 1 week  | -      |
| **Cloud Databases**            | ⭐⭐     | 2 weeks | -      |

---

## 🗂️ Topics Overview

### 📁 **1. Fundamentals** (`01-fundamentals/`)

- ACID Properties & Transactions
- Transaction Isolation Levels
- Indexing Basics (B-Tree, Hash, Full-Text)
- Database Types (RDBMS, NoSQL)
- Connection Pooling

### 📁 **2. Backup & Recovery** (`02-backup-recovery/`)

- **RPO vs RTO** - Business Requirements
- Backup Strategies (Full, Incremental, Differential)
- WAL Archiving & Point-in-Time Recovery
- Restore Testing & Runbooks
- Backup Encryption & Security
- Cloud Backup Solutions

### 📁 **3. High Availability & Replication** (`03-ha-replication/`)

- Replication Types (Single-Leader, Multi-Leader, Leaderless)
- Synchronous vs Asynchronous Replication
- Replication Lag Monitoring
- Failover Mechanisms
- Read Replicas Strategy
- Split-brain Prevention

### 📁 **4. Performance Tuning** (`04-performance-tuning/`)

- Query Analysis & EXPLAIN Plans
- Index Design & Optimization
- Autovacuum & Bloat Management
- Statistics & Query Plans
- Connection Pool Saturation
- Slow Query Detection & Analysis
- Lock Contention & Deadlocks

### 📁 **5. Security & Compliance** (`05-security-compliance/`)

- Least Privilege Access Model
- Role-Based Access Control (RBAC)
- Row-Level Security (RLS)
- Encryption (At-rest & In-transit)
- Audit Logging
- GDPR & PCI-DSS Compliance
- Secret Rotation & Vault Management

### 📁 **6. Schema Evolution & Migrations** (`06-schema-migrations/`)

- Forward-Only Migrations
- Expand-Contract Pattern
- Zero-Downtime Deployments
- Migration Tools (Flyway, Liquibase)
- Data Validation & Checksums
- Rollback Strategies

### 📁 **7. Platform-Specific Guides** (`07-platform-guides/`)

- **PostgreSQL** - WAL, MVCC, Extensions
- **MySQL/InnoDB** - Storage Engine, Replication
- **SQL Server** - Always On, T-SQL, DMVs
- **MongoDB** - Document Model, Aggregation
- **Redis** - In-Memory Cache, Persistence
- Cloud Options (RDS, Cloud SQL, Azure SQL)

### 📁 **8. Monitoring & Observability** (`08-monitoring/`)

- Key Metrics Dashboard
- Alerting Thresholds & SLOs
- Cloud Monitoring (CloudWatch, Datadog)
- Prometheus & Grafana Setup
- Performance Baselines
- Capacity Trending

### 📁 **9. Troubleshooting & Incident Response** (`09-troubleshooting/`)

- Slow Query Diagnosis
- Blocking & Deadlock Analysis
- Database Corruption Detection
- Replication Lag Issues
- Storage & Disk Issues
- Memory Pressure & OOM
- Post-Incident Review Process

### 📁 **10. Advanced Topics** (`10-advanced/`)

- Sharding & Partitioning
- Multi-datacenter Replication
- Distributed Transactions
- Change Data Capture (CDC)
- Logical Decoding
- Serverless Databases

### 📁 **11. Interview Prep** (`11-interview-prep/`)

- Top 20 DBA Interview Questions
- System Design Scenarios
- Incident Response Stories (STAR method)
- SQL Optimization Challenges
- Design Patterns for Scale

---

## 🎓 By Database Platform

### **PostgreSQL**

```
Strengths: ACID, Extensions, JSONB, PostGIS, Replication
Ideal for: Complex queries, analytics, geospatial data
Covered in: 01-fundamentals, 04-performance-tuning, 07-platform-guides/postgresql
```

### **MySQL/InnoDB**

```
Strengths: Reliability, Speed, Ecosystem, Replication
Ideal for: Web applications, simple CRUD, read-heavy workloads
Covered in: 01-fundamentals, 04-performance-tuning, 07-platform-guides/mysql
```

### **SQL Server**

```
Strengths: Enterprise features, Always On, DMVs, Reporting
Ideal for: Enterprise systems, .NET ecosystem
Covered in: 04-performance-tuning, 07-platform-guides/sqlserver
```

### **MongoDB**

```
Strengths: Document model, Flexibility, Horizontal scaling
Ideal for: Flexible schemas, rapid development
Covered in: 01-fundamentals, 07-platform-guides/mongodb
```

### **Redis**

```
Strengths: Ultra-low latency, In-memory, Multiple data structures
Ideal for: Caching, sessions, rate limiting, pub/sub
Covered in: 07-platform-guides/redis
```

---

## 🔗 Quick Links

| Topic                | Folder                                                                                     | Priority          |
| -------------------- | ------------------------------------------------------------------------------------------ | ----------------- |
| How to become DBA    | [Roadmap](./ROADMAP.md)                                                                    | Start here        |
| Interview Questions  | [11-interview-prep](./11-interview-prep/)                                                  | Before interviews |
| RPO & RTO Explained  | [02-backup-recovery/rpo-rto.md](./02-backup-recovery/rpo-rto.md)                           | Essential         |
| PostgreSQL Tuning    | [07-platform-guides/postgresql.md](./07-platform-guides/postgresql.md)                     | Platform-specific |
| Production Checklist | [09-troubleshooting/production-checklist.md](./09-troubleshooting/production-checklist.md) | For cutover       |

---

## 📊 Skill Matrix

### Beginner (0-1 years)

- [ ] ACID properties and transactions
- [ ] Basic index types
- [ ] Simple backup procedures
- [ ] SQL query basics
- [ ] Connection pooling concept

### Intermediate (1-3 years)

- [ ] Replication setup and monitoring
- [ ] Performance analysis with EXPLAIN
- [ ] Autovacuum tuning
- [ ] WAL archiving
- [ ] HA architecture design
- [ ] Schema migration planning

### Advanced (3-5+ years)

- [ ] Multi-datacenter strategies
- [ ] Sharding architecture
- [ ] Incident command & post-mortems
- [ ] Capacity planning models
- [ ] Cloud database optimization
- [ ] Compliance frameworks

---

## 🚀 Getting Started

### Step 1: Set Learning Goals

```
Choose your path:
- Generalist DBA (all platforms)
- Specialist (PostgreSQL/MySQL/SQL Server expert)
- Cloud-focused (AWS RDS, Azure SQL, Cloud SQL)
```

### Step 2: Build Lab Environment

```bash
# Docker compose stack to experiment
docker-compose up -d

# Includes: PostgreSQL, MySQL, Redis, MongoDB
```

### Step 3: Study + Practice

```
1. Read a module (30 min)
2. Set up lab instance (30 min)
3. Practice the skill (30-60 min)
4. Review checklist (10 min)
```

### Step 4: Prepare Interview Stories

```
For each topic, prepare STAR stories:
- Situation
- Task
- Action
- Result
```

---

## 📖 Reference Materials

### Essential Reading

- **"Database Internals"** by Alex Petrov — Deep dive into storage engines
- **"PostgreSQL 14 Internals"** by Egor Bogomolov — PG specific
- **"MySQL Crash Course"** — MySQL fundamentals
- **"Designing Data-Intensive Applications"** — System design patterns

### Official Documentation

- [PostgreSQL Docs](https://www.postgresql.org/docs/)
- [MySQL Docs](https://dev.mysql.com/doc/)
- [SQL Server Docs](https://docs.microsoft.com/en-us/sql/)
- [MongoDB Manual](https://docs.mongodb.com/manual/)

### Key Articles & Blogs

- Postgres Weekly Newsletter
- Planet MySQL
- Use The Index, Luke!
- High-Scalability Blog

---

## 🎯 Interview Preparation

### Top Questions by Category

#### Backup & Recovery

- [ ] Explain RPO vs RTO
- [ ] Design backup strategy for OLTP
- [ ] How to perform restore testing
- [ ] Difference between logical and physical backup

#### Performance

- [ ] How to troubleshoot slow queries
- [ ] Explain deadlock vs blocking
- [ ] Index design principles
- [ ] Autovacuum tuning

#### HA/Replication

- [ ] Synchronous vs asynchronous replication
- [ ] Design HA architecture
- [ ] Handle replication lag
- [ ] Failover strategies

#### Real-world Incident

- [ ] Tell about a database incident (STAR)
- [ ] Root cause analysis
- [ ] Prevention steps implemented

See `11-interview-prep/` for full Q&A guide.

---

## ✅ Self-Assessment Checklist

Before interviews or new role, verify:

- [ ] Can explain ACID properties from memory
- [ ] Can design backup strategy for given SLA
- [ ] Can read and interpret EXPLAIN plans
- [ ] Can troubleshoot slow query
- [ ] Can set up replication
- [ ] Can perform database migration safely
- [ ] Can design for scaling (sharding, partitioning)
- [ ] Can respond to common incidents
- [ ] Can discuss security best practices
- [ ] Can explain CAP theorem trade-offs

---

## 📞 Support & Resources

### Learning

- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Use The Index, Luke](https://use-the-index-luke.com/)
- [Database Reliability Engineering](https://www.databasereliability.com/)

### Tools

- **pgAdmin** — PostgreSQL GUI
- **DBeaver** — Universal database tool
- **Percona Toolkit** — MySQL utilities
- **pgBouncer** — Connection pooler

### Community

- r/dba (Reddit)
- DBA StackExchange
- PostgreSQL Slack
- MySQL Community Forum

---

## 📋 How to Use This Guide

### For Self-Study

1. Start with [Learning Path](#learning-path)
2. Go through each phase sequentially
3. Do the practical exercises
4. Build a portfolio project

### For Interview Preparation

1. Focus on [11-interview-prep](./11-interview-prep/)
2. Study your target platform deeply
3. Prepare incident stories (STAR)
4. Practice explaining concepts clearly

### For On-the-Job Learning

1. Refer to [Platform Guides](./07-platform-guides/)
2. Use [Troubleshooting](./09-troubleshooting/) for problems
3. Check [Monitoring](./08-monitoring/) for setup
4. Validate with [Production Checklist](./09-troubleshooting/production-checklist.md)

---

## 🗺️ Next Steps

```
├─ 1️⃣  Read this README fully
├─ 2️⃣  Choose your learning path (Beginner/Intermediate/Advanced)
├─ 3️⃣  Start with 01-fundamentals/
├─ 4️⃣  Set up lab environment with Docker
├─ 5️⃣  Complete exercises for each topic
├─ 6️⃣  Build a portfolio project
└─ 7️⃣  Prepare for interviews using 11-interview-prep/
```

---

**Last Updated:** 2026-04-21
**Version:** 1.0
**Maintainer:** Backend Interview Prep
