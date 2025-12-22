# Database Interview Questions & Answers

---

## Table of Contents

### [SQL vs NoSQL](#sql-vs-nosql)
- [Q62: What are the differences between SQL and NoSQL databases?](#q62)
- [Q63: When would you choose NoSQL over SQL?](#q63)

### [Normalization](#normalization)
- [Q64: What is database normalization and what are the normal forms?](#q64)
- [Q65: What is denormalization and when would you use it?](#q65)

### [Indexing](#indexing)
- [Q66: What is a database index and how does it work?](#q66)
- [Q67: What is a composite index and what is the leftmost prefix rule?](#q67)

### [ACID vs BASE](#acid-vs-base)
- [Q68: What are ACID properties?](#q68)
- [Q69: What is BASE?](#q69)

### [Transactions & Isolation Levels](#transactions--isolation-levels)
- [Q70: What is a database transaction?](#q70)
- [Q71: What are Dirty Read, Non-Repeatable Read, and Phantom Read?](#q71)
- [Q72: What are the database isolation levels?](#q72)
- [Q73: What is the difference between Read Committed and Read Uncommitted?](#q73)
- [Q74: What is the difference between Repeatable Read and Serializable?](#q74)
- [Q75: How do you choose the right isolation level?](#q75)
- [Q76: What are Lost Update and Write Skew anomalies?](#q76)

### [Joins](#joins)
- [Q77: Explain the different types of JOINs.](#q77)

### [Stored Procedures](#stored-procedures)
- [Q78: What are stored procedures and when should you use them?](#q78)

### [Caching (Redis, Memcached)](#caching-redis-memcached)
- [Q79: What is Redis and what are its use cases?](#q79)
- [Q80: What is the difference between Redis and Memcached?](#q80)

### [Connection Pooling](#connection-pooling)
- [Q81: What is connection pooling and why is it important?](#q81)

### [CAP Theorem](#cap-theorem)
- [Q82: What is the CAP theorem?](#q82)

### [Sharding](#sharding)
- [Q83: What is database sharding?](#q83)

### [Replication](#replication)
- [Q84: What is database replication?](#q84)

### [Deadlocks](#deadlocks)
- [Q85: What is a database deadlock and how can you prevent it?](#q85)

### [Query Optimization](#query-optimization)
- [Q86: How do you optimize slow SQL queries?](#q86)
- [Q87: What should you look for in an EXPLAIN output?](#q87)

---

## SQL vs NoSQL

<a id="q62"></a>
### Q62: What are the differences between SQL and NoSQL databases?
**Answer:**
| SQL | NoSQL |
|-----|-------|
| Relational (tables) | Non-relational (various models) |
| Fixed schema | Dynamic/flexible schema |
| ACID compliant | BASE (eventually consistent) |
| Vertical scaling | Horizontal scaling |
| Complex queries with JOINs | Limited JOIN support |
| Examples: MySQL, PostgreSQL | Examples: MongoDB, Cassandra, Redis |

<a id="q63"></a>
### Q63: When would you choose NoSQL over SQL?
**Answer:**
**Choose NoSQL when:**
- Handling large volumes of unstructured data
- Need horizontal scalability
- Schema changes frequently
- High write throughput required
- Data is naturally hierarchical/document-oriented

**Choose SQL when:**
- Data is highly structured
- Complex queries and JOINs needed
- ACID transactions required
- Data integrity is critical
- Reporting/analytics needs

---

## Normalization

<a id="q64"></a>
### Q64: What is database normalization and what are the normal forms?
**Answer:**
Normalization organizes data to reduce redundancy and improve integrity.

| Normal Form | Rule |
|-------------|------|
| 1NF | Atomic values, no repeating groups |
| 2NF | 1NF + no partial dependencies (all non-key attributes depend on full PK) |
| 3NF | 2NF + no transitive dependencies (non-key depends only on PK) |
| BCNF | 3NF + every determinant is a candidate key |

<a id="q65"></a>
### Q65: What is denormalization and when would you use it?
**Answer:**
Denormalization intentionally adds redundancy for performance.

**Use when:**
- Read performance is critical
- Complex JOINs are slow
- Data warehouse/reporting scenarios
- Caching frequently accessed data

**Trade-offs:**
- Faster reads, slower writes
- More storage space
- Risk of data inconsistency

---

## Indexing

<a id="q66"></a>
### Q66: What is a database index and how does it work?
**Answer:**
An index is a data structure that improves query speed at the cost of additional storage and slower writes.

**Common types:**
- **B-Tree**: Default, good for range queries and equality
- **Hash**: Fast equality lookups, no range queries
- **Bitmap**: Low cardinality columns
- **Full-text**: Text search

```sql
-- Create index
CREATE INDEX idx_user_email ON users(email);

-- Composite index
CREATE INDEX idx_user_name_date ON users(last_name, created_at);
```

<a id="q67"></a>
### Q67: What is a composite index and what is the leftmost prefix rule?
**Answer:**
Composite index indexes multiple columns. The order matters!

```sql
CREATE INDEX idx_abc ON table(a, b, c);
```

**Leftmost prefix rule** - index can be used for queries on:
- (a)
- (a, b)
- (a, b, c)

**Cannot use index for:**
- (b)
- (c)
- (b, c)

---

## ACID vs BASE

<a id="q68"></a>
### Q68: What are ACID properties?
**Answer:**
| Property | Description |
|----------|-------------|
| **Atomicity** | All operations succeed or all fail |
| **Consistency** | Database moves from one valid state to another |
| **Isolation** | Concurrent transactions don't interfere |
| **Durability** | Committed data survives system failure |

<a id="q69"></a>
### Q69: What is BASE?
**Answer:**
BASE is an alternative to ACID for distributed systems:
- **Basically Available**: System guarantees availability
- **Soft state**: State may change over time (due to eventual consistency)
- **Eventually consistent**: System becomes consistent over time

Used in NoSQL databases prioritizing availability over consistency.

---

## Transactions & Isolation Levels

<a id="q70"></a>
### Q70: What is a database transaction?
**Answer:**
A transaction is a unit of work that is either completely executed or not at all.

**Properties (ACID):**
- **Atomicity**: All or nothing
- **Consistency**: Valid state to valid state
- **Isolation**: Concurrent transactions don't interfere
- **Durability**: Committed data persists

```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT; -- or ROLLBACK if error
```

**Transaction states:**
```
Active → Partially Committed → Committed
   ↓              ↓
Failed ←──────────┘
   ↓
Aborted (Rolled Back)
```

<a id="q71"></a>
### Q71: What are Dirty Read, Non-Repeatable Read, and Phantom Read?
**Answer:**
These are **read phenomena** that can occur when transactions run concurrently:

**1. Dirty Read**
Reading uncommitted data from another transaction.

```
Transaction A                    Transaction B
─────────────                    ─────────────
                                 UPDATE balance = 500
SELECT balance → 500 (dirty!)    
                                 ROLLBACK (balance reverts to 1000)
Uses 500 (wrong value!)
```

**2. Non-Repeatable Read**
Same query returns different values within a transaction.

```
Transaction A                    Transaction B
─────────────                    ─────────────
SELECT balance → 1000
                                 UPDATE balance = 500
                                 COMMIT
SELECT balance → 500 (different!)
```

**3. Phantom Read**
New rows appear in repeated queries.

```
Transaction A                    Transaction B
─────────────                    ─────────────
SELECT COUNT(*) WHERE age > 20 → 5
                                 INSERT INTO users (age=25)
                                 COMMIT
SELECT COUNT(*) WHERE age > 20 → 6 (phantom row!)
```

| Phenomenon | Problem | Example |
|------------|---------|---------|
| Dirty Read | Reading uncommitted data | See rolled-back values |
| Non-Repeatable Read | Data changes between reads | Balance changed |
| Phantom Read | New rows appear | COUNT increased |

<a id="q72"></a>
### Q72: What are the database isolation levels?
**Answer:**
Isolation levels control the visibility of changes made by concurrent transactions.

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|-----------------|------------|---------------------|--------------|-------------|
| READ UNCOMMITTED | ✅ Possible | ✅ Possible | ✅ Possible | Fastest |
| READ COMMITTED | ❌ Prevented | ✅ Possible | ✅ Possible | Fast |
| REPEATABLE READ | ❌ Prevented | ❌ Prevented | ✅ Possible | Medium |
| SERIALIZABLE | ❌ Prevented | ❌ Prevented | ❌ Prevented | Slowest |

**Setting isolation level:**
```sql
-- SQL Standard
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- MySQL
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- PostgreSQL (default is READ COMMITTED)
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

**Spring Boot:**
```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void transferMoney() { }
```

<a id="q73"></a>
### Q73: What is the difference between Read Committed and Read Uncommitted?
**Answer:**

| READ UNCOMMITTED | READ COMMITTED |
|------------------|----------------|
| Can read uncommitted (dirty) data | Only reads committed data |
| Fastest, no locks for reads | Slightly slower, uses locks |
| Risk of seeing rolled-back data | Safe from dirty reads |
| Rarely used in production | **Default in most databases** |

**READ UNCOMMITTED example:**
```sql
-- Transaction A
BEGIN;
UPDATE products SET price = 100 WHERE id = 1;  -- Not committed yet

-- Transaction B (READ UNCOMMITTED)
SELECT price FROM products WHERE id = 1;  -- Returns 100 (dirty read!)

-- Transaction A
ROLLBACK;  -- Price reverts to original

-- Transaction B used wrong price!
```

**READ COMMITTED example:**
```sql
-- Transaction A
BEGIN;
UPDATE products SET price = 100 WHERE id = 1;  -- Not committed

-- Transaction B (READ COMMITTED)
SELECT price FROM products WHERE id = 1;  -- Waits or returns OLD value

-- Transaction A
COMMIT;

-- Transaction B now sees 100
```

**Use cases:**
- **READ UNCOMMITTED**: Approximate counts, reporting (rare)
- **READ COMMITTED**: Most OLTP applications (PostgreSQL, Oracle default)

<a id="q74"></a>
### Q74: What is the difference between Repeatable Read and Serializable?
**Answer:**

| REPEATABLE READ | SERIALIZABLE |
|-----------------|--------------|
| Prevents dirty & non-repeatable reads | Prevents ALL anomalies |
| Phantom reads possible | Phantom reads prevented |
| Row-level locking | Range locking or MVCC |
| **MySQL default** | Highest isolation |
| Good balance of safety/performance | Slowest, highest contention |

**REPEATABLE READ (phantom read possible):**
```sql
-- Transaction A
BEGIN;
SELECT * FROM orders WHERE amount > 100;  -- Returns 5 rows

-- Transaction B
INSERT INTO orders (amount) VALUES (200);
COMMIT;

-- Transaction A
SELECT * FROM orders WHERE amount > 100;  -- Returns 6 rows (phantom!)
COMMIT;
```

**SERIALIZABLE (no phantoms):**
```sql
-- Transaction A
BEGIN;
SELECT * FROM orders WHERE amount > 100;  -- Locks the range

-- Transaction B
INSERT INTO orders (amount) VALUES (200);  -- BLOCKED until A commits

-- Transaction A
SELECT * FROM orders WHERE amount > 100;  -- Still 5 rows
COMMIT;

-- Transaction B now proceeds
```

**Implementation differences:**
- **MySQL**: Uses gap locks in REPEATABLE READ (reduces phantoms)
- **PostgreSQL**: REPEATABLE READ uses snapshot isolation (phantoms possible)
- **SERIALIZABLE**: True serializability via locking or SSI (Serializable Snapshot Isolation)

<a id="q75"></a>
### Q75: How do you choose the right isolation level?
**Answer:**

| Use Case | Recommended Level | Reason |
|----------|-------------------|--------|
| Financial transactions | SERIALIZABLE | Maximum consistency |
| E-commerce checkout | REPEATABLE READ | Prevent price changes mid-transaction |
| General web apps | READ COMMITTED | Good balance |
| Analytics/Reporting | READ UNCOMMITTED | Approximate data OK |
| Inventory management | REPEATABLE READ or SERIALIZABLE | Prevent overselling |

**Decision factors:**
1. **Data consistency requirements** - How critical is accuracy?
2. **Concurrency needs** - How many simultaneous users?
3. **Performance requirements** - Can you afford locking overhead?
4. **Conflict likelihood** - How often do transactions touch same data?

**Best practices:**
```java
// Default to READ COMMITTED
@Transactional(isolation = Isolation.READ_COMMITTED)
public void normalOperation() { }

// Use SERIALIZABLE for critical operations
@Transactional(isolation = Isolation.SERIALIZABLE)
public void financialTransfer() { }

// Consider optimistic locking as alternative
@Entity
public class Account {
    @Version
    private Long version;  // Optimistic locking
}
```

**Alternatives to high isolation:**
- **Optimistic locking**: Version field, retry on conflict
- **Pessimistic locking**: `SELECT ... FOR UPDATE`
- **Application-level checks**: Validate before commit

<a id="q76"></a>
### Q76: What are Lost Update and Write Skew anomalies?
**Answer:**

**Lost Update**
Two transactions read the same value, both update it, one overwrites the other.

```
Transaction A                    Transaction B
─────────────                    ─────────────
SELECT balance → 1000            SELECT balance → 1000
balance = 1000 + 100             balance = 1000 + 200
UPDATE balance = 1100            UPDATE balance = 1200
COMMIT                           COMMIT

Result: 1200 (A's update lost!)
Expected: 1300
```

**Prevention:**
```sql
-- Use SELECT FOR UPDATE
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;

-- Or use optimistic locking
UPDATE accounts SET balance = 1100, version = 2 
WHERE id = 1 AND version = 1;
```

**Write Skew**
Two transactions read overlapping data, make decisions based on it, but the combined result violates a constraint.

```
-- Constraint: At least one doctor must be on call
-- Currently: Doctor A and Doctor B are on call

Transaction A                    Transaction B
─────────────                    ─────────────
SELECT COUNT(*) on_call → 2      SELECT COUNT(*) on_call → 2
"2 doctors, I can leave"         "2 doctors, I can leave"
UPDATE set on_call=false         UPDATE set on_call=false
  WHERE doctor='A'                 WHERE doctor='B'
COMMIT                           COMMIT

Result: 0 doctors on call! (constraint violated)
```

**Prevention:**
- SERIALIZABLE isolation level
- Explicit locking: `SELECT ... FOR UPDATE`
- Materialized constraints in database

| Anomaly | Description | Prevention |
|---------|-------------|------------|
| Lost Update | Concurrent updates, one lost | FOR UPDATE, optimistic locking |
| Write Skew | Valid reads, invalid combined writes | SERIALIZABLE, explicit locks |

---

## Joins

<a id="q77"></a>
### Q77: Explain the different types of JOINs.
**Answer:**

```sql
-- Sample data
-- Users: {1, Alice}, {2, Bob}, {3, Charlie}
-- Orders: {101, user_id=1}, {102, user_id=1}, {103, user_id=4}
```

| JOIN Type | Description | Result |
|-----------|-------------|--------|
| INNER JOIN | Only matching rows | Alice's orders |
| LEFT JOIN | All left + matching right | All users + their orders (if any) |
| RIGHT JOIN | All right + matching left | All orders + user info (if exists) |
| FULL OUTER JOIN | All rows from both tables | All users + all orders |
| CROSS JOIN | Cartesian product | Every user × every order |

```sql
-- Get all users with their orders (including users with no orders)
SELECT u.name, o.order_id
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
```

---

## Stored Procedures

<a id="q78"></a>
### Q78: What are stored procedures and when should you use them?
**Answer:**
Stored procedures are precompiled SQL code stored in the database.

**Advantages:**
- Reduced network traffic
- Better security (can grant execute without direct table access)
- Code reuse
- Precompiled (better performance)

**Disadvantages:**
- Database vendor lock-in
- Harder to version control
- Debugging can be difficult
- Business logic split between app and DB

```sql
CREATE PROCEDURE GetUserOrders(IN userId INT)
BEGIN
    SELECT * FROM orders WHERE user_id = userId;
END;
```

---

## Caching (Redis, Memcached)

<a id="q79"></a>
### Q79: What is Redis and what are its use cases?
**Answer:**
Redis is an in-memory data structure store.

**Use cases:**
- Caching (most common)
- Session management
- Rate limiting
- Leaderboards (sorted sets)
- Pub/Sub messaging
- Distributed locks

**Data structures:**
- Strings, Lists, Sets, Sorted Sets, Hashes, Bitmaps, HyperLogLog

<a id="q80"></a>
### Q80: What is the difference between Redis and Memcached?
**Answer:**
| Redis | Memcached |
|-------|-----------|
| Rich data structures | Only strings |
| Persistence options | No persistence |
| Replication & clustering | Simple scaling |
| Pub/Sub support | No pub/sub |
| Lua scripting | No scripting |
| Single-threaded | Multi-threaded |

---

## Connection Pooling

<a id="q81"></a>
### Q81: What is connection pooling and why is it important?
**Answer:**
Connection pooling maintains a cache of database connections for reuse.

**Without pooling:**
- Create connection → Execute query → Close connection
- Expensive: TCP handshake, authentication every time

**With pooling:**
- Get connection from pool → Execute query → Return to pool
- Connections reused, much faster

**Common libraries:**
- HikariCP (recommended, default in Spring Boot 2+)
- Apache DBCP
- C3P0

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 30000
```

---

## CAP Theorem

<a id="q82"></a>
### Q82: What is the CAP theorem?
**Answer:**
In a distributed system, you can only guarantee 2 out of 3:

- **Consistency**: Every read receives the most recent write
- **Availability**: Every request receives a response
- **Partition Tolerance**: System operates despite network partitions

**In practice** (network partitions happen):
- **CP**: Choose consistency over availability (MongoDB, HBase)
- **AP**: Choose availability over consistency (Cassandra, DynamoDB)

---

## Sharding

<a id="q83"></a>
### Q83: What is database sharding?
**Answer:**
Sharding horizontally partitions data across multiple databases.

**Strategies:**
1. **Range-based**: Shard by date range, ID range
2. **Hash-based**: Hash of shard key
3. **Directory-based**: Lookup service maps key to shard

**Challenges:**
- Cross-shard queries
- Rebalancing shards
- Maintaining referential integrity
- Increased complexity

```
Shard 1: Users A-M
Shard 2: Users N-Z
```

---

## Replication

<a id="q84"></a>
### Q84: What is database replication?
**Answer:**
Replication copies data across multiple database servers.

**Types:**
1. **Master-Slave**: Writes to master, reads from slaves
2. **Master-Master**: Writes to any master
3. **Synchronous**: Waits for all replicas
4. **Asynchronous**: Doesn't wait (may have lag)

**Benefits:**
- High availability
- Read scalability
- Disaster recovery
- Geographic distribution

---

## Deadlocks

<a id="q85"></a>
### Q85: What is a database deadlock and how can you prevent it?
**Answer:**
Deadlock: Two transactions each waiting for resources held by the other.

```
Transaction A: Locks Row 1, waits for Row 2
Transaction B: Locks Row 2, waits for Row 1
→ Deadlock!
```

**Prevention:**
1. Always access tables/rows in same order
2. Keep transactions short
3. Use appropriate isolation level
4. Implement retry logic
5. Use `SELECT ... FOR UPDATE` carefully

**Detection:**
- Most databases detect and kill one transaction
- Configure deadlock timeout

---

## Query Optimization

<a id="q86"></a>
### Q86: How do you optimize slow SQL queries?
**Answer:**
1. **Use EXPLAIN/EXPLAIN ANALYZE** to understand query plan
2. **Add appropriate indexes** for WHERE, JOIN, ORDER BY columns
3. **Avoid SELECT *** - only select needed columns
4. **Use pagination** for large result sets
5. **Optimize JOINs** - join on indexed columns
6. **Avoid functions on indexed columns** in WHERE clause
7. **Use query hints** when optimizer chooses wrong plan
8. **Consider denormalization** for read-heavy scenarios
9. **Use connection pooling**
10. **Cache frequently accessed data**

```sql
-- Bad: Function on indexed column
SELECT * FROM users WHERE YEAR(created_at) = 2023;

-- Good: Range query uses index
SELECT * FROM users WHERE created_at >= '2023-01-01' AND created_at < '2024-01-01';
```

<a id="q87"></a>
### Q87: What should you look for in an EXPLAIN output?
**Answer:**
- **Type**: ALL (table scan) → index → ref → eq_ref → const (best)
- **Key**: Which index is used
- **Rows**: Estimated rows examined
- **Extra**: 
  - "Using index" (good - index-only scan)
  - "Using filesort" (bad - extra sorting)
  - "Using temporary" (bad - temp table needed)

---

Good luck with your interview! 🚀
