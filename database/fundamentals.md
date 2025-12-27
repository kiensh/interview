# Database Fundamentals Interview Questions

---

## Table of Contents

### [SQL vs NoSQL](#sql-vs-nosql)
- [Q1: What are the differences between SQL and NoSQL databases?](#q1)
- [Q2: When would you choose NoSQL over SQL?](#q2)

### [Normalization](#normalization)
- [Q3: What is database normalization and what are the normal forms?](#q3)
- [Q4: What is denormalization and when would you use it?](#q4)

### [ACID vs BASE](#acid-vs-base)
- [Q5: What are ACID properties?](#q5)
- [Q6: What is BASE?](#q6)

### [CAP Theorem](#cap-theorem)
- [Q7: What is the CAP theorem?](#q7)

---

## SQL vs NoSQL

<a id="q1"></a>
### Q1: What are the differences between SQL and NoSQL databases?
**Answer:**
| SQL | NoSQL |
|-----|-------|
| Relational (tables) | Non-relational (various models) |
| Fixed schema | Dynamic/flexible schema |
| ACID compliant | BASE (eventually consistent) |
| Vertical scaling | Horizontal scaling |
| Complex queries with JOINs | Limited JOIN support |
| Examples: MySQL, PostgreSQL | Examples: MongoDB, Cassandra, Redis |

<a id="q2"></a>
### Q2: When would you choose NoSQL over SQL?
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

<a id="q3"></a>
### Q3: What is database normalization and what are the normal forms?
**Answer:**
Normalization organizes data to reduce redundancy and improve integrity.

| Normal Form | Rule |
|-------------|------|
| 1NF | Atomic values, no repeating groups |
| 2NF | 1NF + no partial dependencies (all non-key attributes depend on full PK) |
| 3NF | 2NF + no transitive dependencies (non-key depends only on PK) |
| BCNF | 3NF + every determinant is a candidate key |

<a id="q4"></a>
### Q4: What is denormalization and when would you use it?
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

## ACID vs BASE

<a id="q5"></a>
### Q5: What are ACID properties?
**Answer:**
| Property | Description |
|----------|-------------|
| **Atomicity** | All operations succeed or all fail |
| **Consistency** | Database moves from one valid state to another |
| **Isolation** | Concurrent transactions don't interfere |
| **Durability** | Committed data survives system failure |

<a id="q6"></a>
### Q6: What is BASE?
**Answer:**
BASE is an alternative to ACID for distributed systems:
- **Basically Available**: System guarantees availability
- **Soft state**: State may change over time (due to eventual consistency)
- **Eventually consistent**: System becomes consistent over time

Used in NoSQL databases prioritizing availability over consistency.

---

## CAP Theorem

<a id="q7"></a>
### Q7: What is the CAP theorem?
**Answer:**
In a distributed system, you can only guarantee 2 out of 3:

- **Consistency**: Every read receives the most recent write
- **Availability**: Every request receives a response
- **Partition Tolerance**: System operates despite network partitions

**In practice** (network partitions happen):
- **CP**: Choose consistency over availability (MongoDB, HBase)
- **AP**: Choose availability over consistency (Cassandra, DynamoDB)

---

[← Back to Database Index](README.md)
