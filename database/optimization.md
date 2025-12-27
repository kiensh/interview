# Database Optimization Interview Questions

---

## Table of Contents

### [Query Optimization](#query-optimization)
- [Q1: How do you optimize slow SQL queries?](#q1)
- [Q2: What should you look for in an EXPLAIN output?](#q2)

### [Connection Pooling](#connection-pooling)
- [Q3: What is connection pooling and why is it important?](#q3)

### [Joins](#joins)
- [Q4: Explain the different types of JOINs.](#q4)

### [Stored Procedures](#stored-procedures)
- [Q5: What are stored procedures and when should you use them?](#q5)

---

## Query Optimization

<a id="q1"></a>
### Q1: How do you optimize slow SQL queries?
**Answer:**
1. **Use EXPLAIN/EXPLAIN ANALYZE** to understand query plan
2. **Add appropriate indexes** for WHERE, JOIN, ORDER BY columns
3. **Avoid SELECT \*** - only select needed columns
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

<a id="q2"></a>
### Q2: What should you look for in an EXPLAIN output?
**Answer:**
- **Type**: ALL (table scan) → index → ref → eq_ref → const (best)
- **Key**: Which index is used
- **Rows**: Estimated rows examined
- **Extra**: 
  - "Using index" (good - index-only scan)
  - "Using filesort" (bad - extra sorting)
  - "Using temporary" (bad - temp table needed)

---

## Connection Pooling

<a id="q3"></a>
### Q3: What is connection pooling and why is it important?
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

## Joins

<a id="q4"></a>
### Q4: Explain the different types of JOINs.
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

<a id="q5"></a>
### Q5: What are stored procedures and when should you use them?
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

[← Back to Database Index](README.md)
