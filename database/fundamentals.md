# Database Fundamentals Interview Questions

---

## Table of Contents

### [Data Models & Workload Fit](#data-models--workload-fit)
- [Q1: What are the differences between SQL and NoSQL databases?](#q1)
- [Q2: When would you choose NoSQL over SQL?](#q2)

### [Schema Design & Normalization](#schema-design--normalization)
- [Q3: What is database normalization and what are the normal forms?](#q3)
- [Q4: What is denormalization and when would you use it?](#q4)

### [Consistency Models](#consistency-models)
- [Q5: What are ACID properties?](#q5)
- [Q6: What is BASE?](#q6)

### [Distributed Trade-offs](#distributed-trade-offs)
- [Q7: What is the CAP theorem?](#q7)

---

## Data Models & Workload Fit

<a id="q1"></a>
### Q1: What are the differences between SQL and NoSQL databases?
**Answer:**
The core difference is not "old vs new", but **how data is modeled and queried under scale and consistency constraints**.

| Dimension | SQL (Relational) | NoSQL (Non-relational) |
|-----------|------------------|-------------------------|
| Data model | Tables + relations | Document, key-value, wide-column, graph |
| Schema | Strong, explicit schema (migrations) | Flexible/schema-on-read in many engines |
| Query power | Rich joins, aggregations, window functions | Often optimized for specific access patterns |
| Transactions | Strong multi-row ACID is common | Often per-document/partition ACID; cross-entity ACID varies |
| Consistency defaults | Usually stronger consistency | Often tunable/eventual consistency options |
| Scaling style | Scale up first, then read replicas/sharding | Designed for horizontal partitioning early |
| Typical strengths | Complex reporting, integrity-heavy domains | High write throughput, low-latency at large scale |

**Important nuance:** modern systems blur the line:
- PostgreSQL/MySQL can scale horizontally with replicas and sharding middleware.
- NoSQL engines (for example MongoDB) support ACID transactions for many workloads.
- Real production systems often use **polyglot persistence** (SQL + NoSQL per use case).

<a id="q2"></a>
### Q2: When would you choose NoSQL over SQL?
**Answer:**
Choose based on **access patterns, consistency requirements, and operational constraints**, not hype.

| Signal | Prefer SQL | Prefer NoSQL |
|--------|------------|--------------|
| Data relationships are complex | Strong fit | Often awkward unless graph/document model matches |
| Strict integrity is mandatory | Strong fit | Possible, but may add complexity |
| Schema changes are frequent | Migrations needed | Flexible model helps iteration |
| Very high write throughput | Possible but may need careful partitioning | Often easier at very large scale |
| Cross-entity joins are common | Excellent support | Usually application-side joins |
| Global distribution + low latency | Requires architecture work | Many engines offer built-in partition/replication patterns |

**NoSQL is usually a better fit when:**
- Data is naturally nested (events, product catalogs, user activity streams).
- You can design by key-based access patterns up front.
- Eventual consistency is acceptable for parts of the system.

**SQL is usually a better fit when:**
- You need strong constraints (`FOREIGN KEY`, unique invariants).
- Queries are ad hoc and analytics-heavy.
- Business rules require complex multi-row transactions.

**Interview tip:** mention migration path. Teams commonly start with SQL for correctness, then offload specific high-scale features to NoSQL.

---

## Schema Design & Normalization

<a id="q3"></a>
### Q3: What is database normalization and what are the normal forms?
**Answer:**
Normalization reduces redundancy and update anomalies by structuring data so each fact is stored in one place.

| Normal Form | Practical rule | Common anomaly prevented |
|-------------|----------------|--------------------------|
| 1NF | Atomic columns, no repeating groups | Hard-to-query arrays/repeated columns |
| 2NF | Non-key columns depend on full composite key | Partial dependency duplication |
| 3NF | Non-key columns depend only on key | Transitive updates (city/state copied everywhere) |
| BCNF | Every determinant is a candidate key | Edge-case dependency anomalies |

**Example (before normalization):**
```sql
-- Repeats customer fields for every order row
orders(order_id, customer_id, customer_name, customer_city, product_id, qty)
```

**After normalization (typical 3NF design):**
```sql
customers(customer_id, customer_name, customer_city)
orders(order_id, customer_id, order_date)
order_items(order_id, product_id, qty)
```

This improves consistency and keeps writes safer, but often increases join cost for read-heavy queries.

<a id="q4"></a>
### Q4: What is denormalization and when would you use it?
**Answer:**
Denormalization intentionally duplicates or precomputes data to make read paths faster and simpler.

**Common denormalization patterns:**
- Store derived counters (`comment_count`) instead of counting each request.
- Keep "read models" for dashboards/search.
- Materialized views or pre-joined summary tables.
- Embed small immutable attributes (for example `customer_tier`) for faster reads.

**Use denormalization when:**
- Latency targets are strict and joins/aggregations are the bottleneck.
- Workload is read-heavy and stale data tolerance is acceptable.
- You can define clear refresh/update rules.

**Guardrails to mention in interviews:**
- Treat one location as the source of truth.
- Update duplicated fields through a single workflow (events, triggers, background jobs).
- Monitor divergence with reconciliation jobs.

| Benefit | Cost |
|---------|------|
| Faster reads, fewer joins | More write amplification |
| Better cache locality | Higher storage usage |
| Easier read APIs | Risk of stale/inconsistent duplicates |

---

## Consistency Models

<a id="q5"></a>
### Q5: What are ACID properties?
**Answer:**
ACID defines correctness guarantees for transactional systems.

| Property | Meaning | Typical implementation mechanism |
|----------|---------|----------------------------------|
| **Atomicity** | All operations succeed or none do | Undo logs / rollback segments |
| **Consistency** | Constraints/invariants remain valid | Constraints, triggers, app rules |
| **Isolation** | Concurrent transactions behave safely | MVCC, locks, isolation levels |
| **Durability** | Committed data survives crashes | WAL/redo logs + fsync/replication |

**Practical example:** in a bank transfer, debit and credit must commit together. If a crash occurs after debit but before credit, recovery logic replays/rolls back so balances remain valid.

**Interview depth point:** ACID does not mean "no bugs." It guarantees transactional semantics, but bad business logic can still write wrong values.

<a id="q6"></a>
### Q6: What is BASE?
**Answer:**
BASE is a distributed-systems mindset that favors availability and partition tolerance, with convergence over time.

- **Basically Available**: system keeps serving requests.
- **Soft State**: replicas may temporarily diverge.
- **Eventually Consistent**: replicas converge without constant global coordination.

BASE is common in large distributed systems where low latency and uptime matter more than immediate global consistency.

| Consistency approach | Typical guarantee | Example use case |
|----------------------|-------------------|------------------|
| Strong consistency | Read sees latest committed write | Payment settlement |
| Eventual consistency (BASE style) | Data converges after replication delay | Feed timelines, analytics counters |
| Session-level guarantees | Read-your-writes for one user/session | Profile edits in user-facing apps |

**Key trade-off:** BASE improves resilience and scalability, but pushes conflict handling/idempotency into application design.

---

## Distributed Trade-offs

<a id="q7"></a>
### Q7: What is the CAP theorem?
**Answer:**
CAP says that when a **network partition** happens, a distributed system must choose between:

- **Consistency (C):** every read sees the latest write.
- **Availability (A):** every request gets a non-error response.
- **Partition Tolerance (P):** system continues operating despite network splits.

Because partitions are inevitable at scale, real systems typically choose **CP** or **AP** behavior under partition.

| Under partition | Behavior | Example trade-off |
|-----------------|----------|-------------------|
| CP | Reject/timeout some requests to keep data consistent | Fewer stale reads, lower availability |
| AP | Serve requests despite replica divergence | Higher availability, temporary stale/conflicting data |

**Interview nuance:** CAP only describes partition scenarios. In normal operation, latency trade-offs are better captured by **PACELC**:
- If Partitioned: choose A vs C
- Else: choose Latency vs Consistency

This is why globally distributed databases offer consistency levels (quorum, local read, strong read) instead of one fixed mode.

---

[← Back to Database Index](README.md)
