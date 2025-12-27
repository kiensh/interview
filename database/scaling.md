# Database Scaling Interview Questions

---

## Table of Contents

### [Partitioning](#partitioning)
- [Q1: What is database partitioning and why use it?](#q1)
- [Q2: What are the different types of partitioning?](#q2)
- [Q3: What is the difference between partitioning and sharding?](#q3)

### [Sharding](#sharding)
- [Q4: What is database sharding?](#q4)

### [Replication](#replication)
- [Q5: What is database replication?](#q5)

---

## Partitioning

<a id="q1"></a>
### Q1: What is database partitioning and why use it?
**Answer:**
Partitioning divides a large table into smaller, manageable pieces (partitions) while appearing as a single table to applications.

**Benefits:**
1. **Improved query performance**: Partition pruning skips irrelevant partitions
2. **Easier maintenance**: Archive/drop old partitions without affecting others
3. **Parallel operations**: Operations can run on partitions simultaneously
4. **Better I/O distribution**: Spread data across storage devices

```sql
-- PostgreSQL partitioned table example
CREATE TABLE orders (
    id SERIAL,
    order_date DATE NOT NULL,
    customer_id INT,
    amount DECIMAL(10,2)
) PARTITION BY RANGE (order_date);

-- Create partitions
CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
    
CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Query automatically uses partition pruning
SELECT * FROM orders WHERE order_date = '2024-06-15';
-- Only scans orders_2024 partition
```

<a id="q2"></a>
### Q2: What are the different types of partitioning?
**Answer:**

| Type | Description | Use Case |
|------|-------------|----------|
| **Range** | Based on value ranges | Time-series data, dates |
| **List** | Based on discrete values | Geographic regions, categories |
| **Hash** | Based on hash of column | Even distribution, no natural range |
| **Composite** | Combination of above | Complex requirements |

**Range Partitioning:**
```sql
-- By date range
CREATE TABLE logs (
    id INT,
    log_date DATE,
    message TEXT
) PARTITION BY RANGE (log_date);

CREATE TABLE logs_jan PARTITION OF logs FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE logs_feb PARTITION OF logs FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');
```

**List Partitioning:**
```sql
-- By region
CREATE TABLE customers (
    id INT,
    name VARCHAR(100),
    region VARCHAR(20)
) PARTITION BY LIST (region);

CREATE TABLE customers_us PARTITION OF customers FOR VALUES IN ('US', 'CA');
CREATE TABLE customers_eu PARTITION OF customers FOR VALUES IN ('UK', 'DE', 'FR');
CREATE TABLE customers_asia PARTITION OF customers FOR VALUES IN ('JP', 'CN', 'KR');
```

**Hash Partitioning:**
```sql
-- Even distribution by user_id
CREATE TABLE user_sessions (
    session_id UUID,
    user_id INT,
    data JSONB
) PARTITION BY HASH (user_id);

CREATE TABLE user_sessions_0 PARTITION OF user_sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE user_sessions_1 PARTITION OF user_sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE user_sessions_2 PARTITION OF user_sessions FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE user_sessions_3 PARTITION OF user_sessions FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

<a id="q3"></a>
### Q3: What is the difference between partitioning and sharding?
**Answer:**

| Partitioning | Sharding |
|--------------|----------|
| Single database server | Multiple database servers |
| Logical division | Physical distribution |
| Managed by database | Managed by application/middleware |
| Transparent to application | Application aware of shards |
| Vertical scalability | Horizontal scalability |
| Single point of failure | Distributed, more resilient |

```
Partitioning (Single Server):
┌─────────────────────────────┐
│     Single Database         │
│  ┌─────┬─────┬─────┬─────┐ │
│  │ P1  │ P2  │ P3  │ P4  │ │
│  └─────┴─────┴─────┴─────┘ │
└─────────────────────────────┘

Sharding (Multiple Servers):
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Server 1 │  │ Server 2 │  │ Server 3 │
│ Shard 1  │  │ Shard 2  │  │ Shard 3  │
└──────────┘  └──────────┘  └──────────┘
```

**When to use each:**
- **Partitioning**: Large tables, easier maintenance, query optimization
- **Sharding**: Massive scale, geographic distribution, write scalability

---

## Sharding

<a id="q4"></a>
### Q4: What is database sharding?
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

<a id="q5"></a>
### Q5: What is database replication?
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

[← Back to Database Index](README.md)
