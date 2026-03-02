# Database Optimization Interview Questions

---

## Table of Contents

### [Performance Tuning Workflow](#performance-tuning-workflow)
- [Q1: How do you optimize slow SQL queries?](#q1)
- [Q2: What should you look for in an EXPLAIN output?](#q2)

### [Connection Management](#connection-management)
- [Q3: What is connection pooling and why is it important?](#q3)

### [Query Shape & Data Access](#query-shape--data-access)
- [Q4: Explain the different types of JOINs.](#q4)

### [Database-side Logic](#database-side-logic)
- [Q5: What are stored procedures and when should you use them?](#q5)

---

## Performance Tuning Workflow

<a id="q1"></a>
### Q1: How do you optimize slow SQL queries?
**Answer:**
Treat query tuning as an iterative workflow, not a one-time checklist.

**Step-by-step approach:**
1. **Measure first**: capture slow query text, frequency, p95 latency, rows examined.
2. **Inspect plan**: use `EXPLAIN ANALYZE` (or DB equivalent) to find scan/sort/join hotspots.
3. **Fix query shape**: reduce rows early, avoid unnecessary columns, remove anti-patterns.
4. **Fix indexing**: add/adjust indexes that match predicate + join + sort access paths.
5. **Validate impact**: compare latency and resource usage before/after under realistic load.
6. **Guard against regressions**: monitor plan changes after schema/data growth.

**Common high-impact optimizations:**
- Replace non-SARGable predicates (functions on indexed columns).
- Convert wide `SELECT *` queries to targeted projections.
- Break expensive "one query does everything" into staged queries when cardinality explodes.
- Use keyset pagination for deep pages.
- Add covering indexes for hot read paths.

```sql
-- Bad: Function on indexed column
SELECT * FROM users WHERE YEAR(created_at) = 2023;

-- Good: SARGable range query can use index
SELECT * FROM users WHERE created_at >= '2023-01-01' AND created_at < '2024-01-01';
```

**Interview tip:** mention that the "fastest query" is often the one that scans the least data, not the one with the most hints.

<a id="q2"></a>
### Q2: What should you look for in an EXPLAIN output?
**Answer:**
Focus on whether the planner is reading too many rows, choosing wrong join order, or sorting/spilling unnecessarily.

| Signal | Why it matters | What to do |
|--------|----------------|------------|
| Access method (`ALL`, seq scan) | Full scan can be expensive on large tables | Add/select better index, tighten filters |
| Chosen index (`key`) | Wrong index often means bad cardinality assumptions | Refresh stats, reorder index columns |
| Estimated vs actual rows | Big mismatch indicates stale/insufficient stats | `ANALYZE`, improve predicates, adjust stats targets |
| Join strategy | Nested loop/hash/merge each has sweet spots | Re-check indexes and join cardinality |
| Sort/hash spill | Disk spills can dominate latency | Increase work memory carefully or avoid large intermediate sets |
| Extra flags (`Using filesort`, `Using temporary`) | Indicates additional operations | Align indexes with `ORDER BY`/`GROUP BY` |

**MySQL quick clues:**
- `type=ALL` is usually a red flag on large tables.
- `Using index` means covering access (often good).

**PostgreSQL quick clues:**
- Compare `rows=` estimate with `actual rows=`.
- Watch `Sort Method: external merge` or hash batch spills in `EXPLAIN ANALYZE`.

---

## Connection Management

<a id="q3"></a>
### Q3: What is connection pooling and why is it important?
**Answer:**
Connection pooling reuses existing DB connections so each request does not pay connection setup and auth overhead.

Without pooling, high traffic causes:
- Frequent TCP/TLS/auth handshakes.
- Connection storms under bursts.
- DB CPU wasted on connection churn instead of query execution.

With pooling:
- App borrows a connection, executes query, returns connection.
- Throughput and latency stabilize.
- You can enforce backpressure via max pool size.

**Common libraries:**
- HikariCP (recommended, default in Spring Boot 2+)
- Apache DBCP
- C3P0

**Sizing guidance (practical):**
- Pool too small -> queueing and timeouts.
- Pool too large -> DB context switching and lock contention.
- Start around `(cpu_cores * 2) + effective_spindle_count` as a rough heuristic, then tune by metrics.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      max-lifetime: 1800000
      leak-detection-threshold: 60000
```

**Interview depth point:** connection pool tuning must be coordinated with DB `max_connections` and thread model.

---

## Query Shape & Data Access

<a id="q4"></a>
### Q4: Explain the different types of JOINs.
**Answer:**
JOIN type controls which matched and unmatched rows are returned.

| JOIN Type | Description | Result |
|-----------|-------------|--------|
| INNER JOIN | Only matching rows | Alice's orders |
| LEFT JOIN | All left + matching right | All users + their orders (if any) |
| RIGHT JOIN | All right + matching left | All orders + user info (if exists) |
| FULL OUTER JOIN | All rows from both tables | All users + all orders |
| CROSS JOIN | Cartesian product | Every user × every order |

```sql
-- Sample data
-- users: (1,Alice), (2,Bob), (3,Charlie)
-- orders: (101,user_id=1), (102,user_id=1), (103,user_id=4)
```

```sql
-- Get all users with their orders (including users with no orders)
SELECT u.name, o.order_id
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
```

**Performance pitfalls to mention:**
- Missing join predicate -> accidental Cartesian explosion.
- Joining before filtering -> huge intermediate sets.
- Data type mismatch on join keys -> index not used.
- One-to-many chains can duplicate rows; aggregate carefully.

---

## Database-side Logic

<a id="q5"></a>
### Q5: What are stored procedures and when should you use them?
**Answer:**
Stored procedures are executable routines stored in the database that encapsulate SQL logic near the data.

**Advantages:**
- Reduced network traffic
- Better security (can grant execute without direct table access)
- Code reuse
- Stable execution path for repeated operations

**Disadvantages:**
- Database vendor lock-in
- Harder to version control
- Debugging can be difficult
- Business logic split between app and DB

**Use stored procedures when:**
- You need tightly controlled data access APIs for compliance/security.
- Logic is data-intensive and benefits from fewer app<->DB round trips.
- Batch jobs require efficient set-based operations inside DB.

**Avoid overusing them when:**
- Team prefers application-level testing and CI/CD pipelines.
- Multi-database portability is important.
- Business logic changes frequently.

```sql
CREATE PROCEDURE GetUserOrders(IN userId INT)
BEGIN
    SELECT * FROM orders WHERE user_id = userId;
END;
```

---

[← Back to Database Index](README.md)
