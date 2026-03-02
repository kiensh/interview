# Database Interview Questions

A collection of database-related interview questions organized by topic.

## Contents

| File | Topics | Questions |
|------|--------|-----------|
| [fundamentals.md](fundamentals.md) | Data models, normalization, consistency models, CAP trade-offs | 7 |
| [indexing.md](indexing.md) | Index fundamentals, composite design, PostgreSQL index types, physical layout | 5 |
| [transactions.md](transactions.md) | Isolation, anomalies, propagation, distributed transactions, locking, deadlocks | 16 |
| [scaling.md](scaling.md) | Partitioning, sharding, replication and availability | 5 |
| [optimization.md](optimization.md) | Tuning workflow, EXPLAIN analysis, pooling, joins, DB-side logic | 5 |
| [redis.md](redis.md) | Redis fundamentals, data structures, persistence, clustering, caching, streams, operations | 42 |

**Total: 80 Questions**

## Quick Navigation

### Fundamentals
- Data models and workload fit (SQL vs NoSQL)
- Schema design and normalization/denormalization
- ACID vs BASE and consistency semantics
- CAP/PACELC-style distributed trade-offs

### Indexing
- How indexes work and when they help
- Composite indexes and leftmost-prefix design
- Index family trade-offs (B-Tree/Hash/Full-text/Spatial)
- PostgreSQL specifics (GIN/GiST)
- Clustered vs non-clustered physical layout behavior

### Transactions & Locking
- ACID foundations and isolation-level behavior
- Dirty read/non-repeatable/phantom + lost update/write skew
- Spring propagation and savepoints
- 2PC, saga, and outbox patterns for distributed workflows
- Lock types, optimistic vs pessimistic strategies, and deadlock prevention

### Scaling
- Intra-node partitioning strategies (range/list/hash/composite)
- Cross-node sharding strategies and rebalancing concerns
- Replication topologies and consistency/latency trade-offs

### Optimization
- Query tuning workflow: measure -> diagnose -> optimize -> validate
- Reading EXPLAIN plans in MySQL/PostgreSQL
- Connection pool sizing and operational guardrails
- JOIN strategy/performance pitfalls
- Stored procedures: when to use and when to avoid

### Redis
- Core architecture, key design, and data-structure modeling
- TTL semantics, memory management, and eviction policies
- RDB/AOF persistence trade-offs and restore strategy
- Replication, Sentinel failover, Cluster sharding/resharding
- Atomic ops, transactions, Lua/functions, and pipelining
- Caching patterns, stampede prevention, Pub/Sub vs Streams
- Security hardening and incident troubleshooting commands

---

[← Back to Main Index](../README.md)
