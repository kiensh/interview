# Database Transactions Interview Questions

---

[Spring Transactional: Propagation and Isolation](https://www.baeldung.com/spring-transactional-propagation-isolation)

[Deep into Transaction](https://viblo.asia/p/deep-into-transaction-PwlVmeK1V5Z)

## Table of Contents

### [Transactions & Isolation Levels](#transactions--isolation-levels)
- [Q1: What is a database transaction?](#q1)
- [Q2: What are Dirty Read, Non-Repeatable Read, and Phantom Read?](#q2)
- [Q3: What are the database isolation levels?](#q3)
- [Q4: What is the difference between Read Committed and Read Uncommitted?](#q4)
- [Q5: What is the difference between Repeatable Read and Serializable?](#q5)
- [Q6: How do you choose the right isolation level?](#q6)
- [Q7: What are Lost Update and Write Skew anomalies?](#q7)

### [Transaction Propagation & Savepoints](#transaction-propagation--savepoints)
- [Q8: What is transaction propagation?](#q8)
- [Q9: What are savepoints in transactions?](#q9)

### [Distributed Transactions](#distributed-transactions)
- [Q10: What is the two-phase commit protocol?](#q10)
- [Q11: What are distributed transactions and their challenges?](#q11)

### [Locking](#locking)
- [Q12: What is database locking?](#q12)
- [Q13: What is the difference between optimistic and pessimistic locking?](#q13)
- [Q14: When should you use optimistic vs pessimistic locking?](#q14)
- [Q15: What are the different lock types in databases?](#q15)

### [Deadlocks](#deadlocks)
- [Q16: What is a database deadlock and how can you prevent it?](#q16)

---

## Transactions & Isolation Levels

<a id="q1"></a>
### Q1: What is a database transaction?
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

<a id="q2"></a>
### Q2: What are Dirty Read, Non-Repeatable Read, and Phantom Read?
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

<a id="q3"></a>
### Q3: What are the database isolation levels?
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

<a id="q4"></a>
### Q4: What is the difference between Read Committed and Read Uncommitted?
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

<a id="q5"></a>
### Q5: What is the difference between Repeatable Read and Serializable?
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

<a id="q6"></a>
### Q6: How do you choose the right isolation level?
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

<a id="q7"></a>
### Q7: What are Lost Update and Write Skew anomalies?

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

## Transaction Propagation & Savepoints

<a id="q8"></a>
### Q8: What is transaction propagation?
**Answer:**
Transaction propagation defines how a transactional method behaves when called from another transactional method.

| Propagation | Behavior |
|-------------|----------|
| **REQUIRED** (default) | Join existing or create new |
| **REQUIRES_NEW** | Always create new, suspend existing |
| **SUPPORTS** | Join if exists, non-transactional otherwise |
| **NOT_SUPPORTED** | Non-transactional, suspend existing |
| **MANDATORY** | Must join existing, error if none |
| **NEVER** | Non-transactional, error if exists |
| **NESTED** | Nested transaction (savepoint) |

```java
@Service
public class OrderService {
    
    @Transactional  // REQUIRED by default
    public void createOrder(Order order) {
        orderRepository.save(order);
        paymentService.processPayment(order);  // What happens here?
    }
}

@Service
public class PaymentService {
    
    @Transactional(propagation = Propagation.REQUIRED)
    public void processPayment(Order order) {
        // Joins the existing transaction from createOrder
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void processPaymentNew(Order order) {
        // Creates NEW transaction, suspends createOrder's transaction
        // If this fails, createOrder still commits
    }
    
    @Transactional(propagation = Propagation.MANDATORY)
    public void processPaymentMandatory(Order order) {
        // Must be called from within a transaction
        // Throws exception if no active transaction
    }
}
```

<a id="q9"></a>
### Q9: What are savepoints in transactions?
**Answer:**
Savepoints allow partial rollback within a transaction, rolling back to a specific point without aborting the entire transaction.

```sql
BEGIN;

INSERT INTO orders (id, customer_id) VALUES (1, 100);

SAVEPOINT before_items;

INSERT INTO order_items (order_id, product_id) VALUES (1, 'PROD1');
INSERT INTO order_items (order_id, product_id) VALUES (1, 'PROD2');

-- Something went wrong with items
ROLLBACK TO SAVEPOINT before_items;
-- Order is still saved, items are rolled back

INSERT INTO order_items (order_id, product_id) VALUES (1, 'PROD3');

COMMIT;  -- Commits order and PROD3 only
```

**Spring/JDBC:**
```java
@Transactional
public void processWithSavepoint() {
    Connection conn = dataSource.getConnection();
    
    try {
        // First operation
        jdbcTemplate.update("INSERT INTO orders ...");
        
        Savepoint savepoint = conn.setSavepoint("beforeItems");
        
        try {
            // Risky operation
            jdbcTemplate.update("INSERT INTO order_items ...");
        } catch (Exception e) {
            conn.rollback(savepoint);
            // Continue with alternative logic
        }
        
        // This will commit even if items failed
    } finally {
        conn.releaseSavepoint(savepoint);
    }
}
```

**Use cases:**
- Batch processing with partial failure handling
- Complex operations where some steps can fail
- Nested transaction-like behavior

---

## Distributed Transactions

<a id="q10"></a>
### Q10: What is the two-phase commit protocol?
**Answer:**
Two-phase commit (2PC) is a distributed algorithm ensuring all nodes in a distributed transaction either commit or abort together.

**Phase 1: Prepare (Voting)**
1. Coordinator sends PREPARE to all participants
2. Participants prepare (acquire locks, write to log)
3. Participants respond VOTE_COMMIT or VOTE_ABORT

**Phase 2: Commit/Abort**
1. If all voted COMMIT → Coordinator sends COMMIT to all
2. If any voted ABORT → Coordinator sends ABORT to all
3. Participants execute and acknowledge

```
        Coordinator
            │
     ┌──────┼──────┐
     ▼      ▼      ▼
   Node1  Node2  Node3

Phase 1 (Prepare):
Coordinator → "PREPARE" → All Nodes
All Nodes → "READY" → Coordinator

Phase 2 (Commit):
Coordinator → "COMMIT" → All Nodes
All Nodes → "ACK" → Coordinator
```

**Problems with 2PC:**
| Problem | Description |
|---------|-------------|
| Blocking | Participants block waiting for coordinator |
| Single point of failure | Coordinator failure blocks everyone |
| Network partitions | Uncertain state during partitions |
| Performance | Multiple round trips, lock holding |

**Alternatives:**
- **3PC (Three-Phase Commit)**: Adds pre-commit phase
- **Saga Pattern**: Compensating transactions
- **TCC (Try-Confirm-Cancel)**: Reservation pattern

<a id="q11"></a>
### Q11: What are distributed transactions and their challenges?
**Answer:**
Distributed transactions span multiple databases, services, or systems that must all commit or rollback together.

**Challenges:**

| Challenge | Description |
|-----------|-------------|
| Network failures | Nodes may become unreachable |
| Partial failures | Some nodes succeed, others fail |
| Latency | Multiple round trips increase time |
| Coordination overhead | Lock holding across network |
| CAP theorem trade-offs | Can't have consistency + availability + partition tolerance |

**Patterns for distributed transactions:**

**1. Saga Pattern:**
```
Order Saga:
1. Create Order → (compensate: Cancel Order)
2. Reserve Inventory → (compensate: Release Inventory)
3. Process Payment → (compensate: Refund Payment)
4. Ship Order → (compensate: Cancel Shipment)

If step 3 fails:
- Execute compensations in reverse: Release Inventory, Cancel Order
```

**2. Outbox Pattern:**
```sql
-- Write business data and event in same transaction
BEGIN;
INSERT INTO orders (id, status) VALUES (1, 'CREATED');
INSERT INTO outbox (event_type, payload) VALUES ('OrderCreated', '{"id":1}');
COMMIT;

-- Separate process polls outbox and publishes events
```

**3. TCC Pattern:**
```java
// Try: Reserve resources
paymentService.tryReserve(amount);
inventoryService.tryReserve(items);

// Confirm: Complete the operation
paymentService.confirm();
inventoryService.confirm();

// Cancel: If any try fails
paymentService.cancel();
inventoryService.cancel();
```

**Best practices:**
1. Prefer eventual consistency when possible
2. Design idempotent operations
3. Implement compensating transactions
4. Use correlation IDs for tracing
5. Consider event-driven architecture

---

## Locking

<a id="q12"></a>
### Q12: What is database locking?
**Answer:**
Locking is a mechanism to control concurrent access to data, ensuring data integrity when multiple transactions access the same data.

**Why locking is needed:**
- Prevent lost updates
- Ensure data consistency
- Implement isolation levels
- Coordinate concurrent transactions

**Lock granularity:**
| Level | Locks | Concurrency | Overhead |
|-------|-------|-------------|----------|
| Database | Entire database | Lowest | Lowest |
| Table | Entire table | Low | Low |
| Page | Data page (group of rows) | Medium | Medium |
| Row | Single row | Highest | Highest |

```sql
-- Explicit row-level locking
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- Exclusive lock
SELECT * FROM accounts WHERE id = 1 FOR SHARE;   -- Shared lock

-- Table-level lock
LOCK TABLE accounts IN EXCLUSIVE MODE;
```

<a id="q13"></a>
### Q13: What is the difference between optimistic and pessimistic locking?
**Answer:**

| Optimistic Locking | Pessimistic Locking |
|--------------------|---------------------|
| Assumes conflicts are rare | Assumes conflicts are common |
| No locks during read/modify | Locks data during entire operation |
| Checks for conflicts at commit | Prevents conflicts proactively |
| Uses version/timestamp column | Uses database locks |
| Better for read-heavy workloads | Better for write-heavy workloads |
| May fail at commit time | Guaranteed success (if lock acquired) |

**Optimistic Locking (Version-based):**
```java
@Entity
public class Account {
    @Id
    private Long id;
    
    @Version  // Hibernate auto-increments on update
    private Long version;
    
    private BigDecimal balance;
}

// Usage
Account account = repository.findById(1L);  // version = 5
account.setBalance(new BigDecimal("100"));
repository.save(account);  // UPDATE ... WHERE id = 1 AND version = 5
// If another transaction updated, version won't match → OptimisticLockException
```

**Pessimistic Locking:**
```java
// JPA pessimistic lock
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Account findByIdWithLock(@Param("id") Long id);

// Or native SQL
@Query(value = "SELECT * FROM accounts WHERE id = :id FOR UPDATE", nativeQuery = true)
Account findByIdForUpdate(@Param("id") Long id);
```

```sql
-- SQL pessimistic locking
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- Lock acquired
-- Other transactions block here
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;  -- Lock released
```

<a id="q14"></a>
### Q14: When should you use optimistic vs pessimistic locking?
**Answer:**

**Use Optimistic Locking when:**
- Read-heavy workload (many reads, few writes)
- Low contention (conflicts are rare)
- Short transactions
- Can handle retry logic
- Need high concurrency

**Use Pessimistic Locking when:**
- Write-heavy workload
- High contention on same data
- Conflicts are expensive to resolve
- Operations must succeed without retry
- Critical data integrity (financial transactions)

| Scenario | Recommended |
|----------|-------------|
| E-commerce product catalog | Optimistic |
| Inventory with limited stock | Pessimistic |
| User profile updates | Optimistic |
| Bank account transfers | Pessimistic |
| Blog post editing | Optimistic |
| Seat reservation | Pessimistic |

**Handling optimistic lock failures:**
```java
@Retryable(value = OptimisticLockingFailureException.class, maxAttempts = 3)
@Transactional
public void updateBalance(Long id, BigDecimal amount) {
    Account account = repository.findById(id).orElseThrow();
    account.setBalance(account.getBalance().add(amount));
    repository.save(account);
}
```

<a id="q15"></a>
### Q15: What are the different lock types in databases?
**Answer:**

| Lock Type | Description | Compatibility |
|-----------|-------------|---------------|
| **Shared (S)** | Read lock, multiple transactions can hold | Compatible with other Shared |
| **Exclusive (X)** | Write lock, only one transaction | Incompatible with all |
| **Update (U)** | Intent to update, prevents deadlock | Compatible with Shared |
| **Intent Shared (IS)** | Intent to acquire Shared at lower level | Compatible with IS, IX, S |
| **Intent Exclusive (IX)** | Intent to acquire Exclusive at lower level | Compatible with IS, IX |

**Lock compatibility matrix:**
```
        S    X    IS   IX
   S   ✓    ✗    ✓    ✗
   X   ✗    ✗    ✗    ✗
  IS   ✓    ✗    ✓    ✓
  IX   ✗    ✗    ✓    ✓
```

**PostgreSQL lock modes:**
```sql
-- Different lock modes
SELECT * FROM accounts FOR SHARE;           -- Shared lock
SELECT * FROM accounts FOR UPDATE;          -- Exclusive lock
SELECT * FROM accounts FOR NO KEY UPDATE;   -- Weaker exclusive
SELECT * FROM accounts FOR KEY SHARE;       -- Weaker shared

-- Skip locked rows (useful for job queues)
SELECT * FROM jobs WHERE status = 'pending' 
FOR UPDATE SKIP LOCKED LIMIT 1;
```

---

## Deadlocks

<a id="q16"></a>
### Q16: What is a database deadlock and how can you prevent it?
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

[← Back to Database Index](README.md)
