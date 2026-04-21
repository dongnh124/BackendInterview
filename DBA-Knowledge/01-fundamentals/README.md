# Database Fundamentals

Quick reference for core database concepts every DBA must know.

## 📚 Topics

1. **ACID Properties** — What guarantees databases provide
2. **Transaction Isolation Levels** — Concurrency trade-offs
3. **Indexing Strategies** — Query optimization basics
4. **Database Types** — RDBMS vs NoSQL
5. **Connection Pooling** — Resource management

## 🎯 Key Takeaways

### ACID Properties

- **Atomicity**: All or nothing transactions
- **Consistency**: Valid state before and after
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed data survives crashes (WAL)

### Isolation Levels (Low → High)

1. Read Uncommitted — Allows dirty reads ❌
2. Read Committed — No dirty reads ✓
3. Repeatable Read — No non-repeatable reads ✓
4. Serializable — Full isolation (slowest) ✓

### Index Types

- **B-Tree** — General purpose (range, sorting)
- **Hash** — Equality only (fastest single value)
- **Full-Text** — Text search optimization
- **Composite** — Multiple columns (follow leftmost rule)
- **Partial** — Filtered index (smaller, faster)
- **Covering** — Index-only scan (no table lookup)

### Connection Pooling

```
Benefits: Reduce TCP handshake overhead, avoid max_connections limit
Issues: Connection leaks, exhaustion, parameter sniffing
Solution: Right-size pool, use transaction mode pooler (PgBouncer)
```

## 📖 Study Guide

| Topic              | Time   | Difficulty | Labs                         |
| ------------------ | ------ | ---------- | ---------------------------- |
| ACID Properties    | 30 min | ⭐         | N/A                          |
| Isolation Levels   | 45 min | ⭐⭐       | Test each level              |
| Indexing Basics    | 1 hour | ⭐⭐       | Create & analyze indexes     |
| Database Types     | 45 min | ⭐         | Compare RDBMS vs Mongo       |
| Connection Pooling | 30 min | ⭐⭐       | Configure HikariCP/PgBouncer |

## 🔗 Related Topics

- [Backup & Recovery](../02-backup-recovery/)
- [Performance Tuning](../04-performance-tuning/)
- [Platform Guides](../07-platform-guides/)

## ✅ Checklist

- [ ] Can explain ACID without notes
- [ ] Know which isolation level PostgreSQL/MySQL use by default
- [ ] Understand B-Tree index structure
- [ ] Know connection pool sizing formula
- [ ] Can choose right index type for a query
