# Database Indexing Interview Questions

---

## Table of Contents

### [Index Fundamentals](#index-fundamentals)
- [Q1: What is a database index and how does it work?](#q1)

### [Index Design Rules](#index-design-rules)
- [Q2: What is a composite index and what is the leftmost prefix rule?](#q2)

### [Index Types & Trade-offs](#index-types--trade-offs)
- [Q3: What are the different types of database indexes?](#q3)
- [Q4: What are PostgreSQL-specific index types (GIN, GiST)?](#q4)

### [Physical Layout](#physical-layout)
- [Q5: What is the difference between clustered and non-clustered indexes?](#q5)

---

## Index Fundamentals

<a id="q1"></a>
### Q1: What is a database index and how does it work?
**Answer:**
An index is a secondary data structure that lets the optimizer locate rows quickly without scanning the full table.

Think of it as a sorted lookup structure:
1. Optimizer evaluates query predicates (`WHERE`, `JOIN`, `ORDER BY`).
2. If index selectivity is good, it performs index lookup/scan.
3. It either returns rows directly (index-only/covering scan) or does a table lookup for remaining columns.

**Why indexes help:**
- Reduce I/O by reading fewer pages.
- Speed up equality and range predicates.
- Can avoid explicit sorting if index order matches `ORDER BY`.

**Why indexes hurt:**
- Every insert/update/delete must maintain index entries.
- Extra disk + memory pressure.
- Too many indexes can slow writes significantly.

**Interview phrase to use:** "Indexes optimize reads by trading off write cost and storage."

```sql
-- Single-column index for frequent equality lookup
CREATE INDEX idx_user_email ON users(email);

-- Composite index for common filter + sort pattern
CREATE INDEX idx_user_name_date ON users(last_name, created_at);
```

---

## Index Design Rules

<a id="q2"></a>
### Q2: What is a composite index and what is the leftmost prefix rule?
**Answer:**
A composite index contains multiple columns in a fixed order, so column order determines which queries can use it efficiently.

```sql
CREATE INDEX idx_abc ON table(a, b, c);
```

**Leftmost prefix rule** means the index can support:
- (a)
- (a, b)
- (a, b, c)

It usually cannot support efficiently:
- (b)
- (c)
- (b, c)

**Design strategy (production):**
- Put highest selectivity and most frequent filter column first.
- Align with common `WHERE + ORDER BY` pattern to avoid extra sort.
- Avoid low-cardinality columns first unless combined with a selective prefix.
- Re-check with `EXPLAIN ANALYZE` after creation; assumptions are often wrong.

**Example query patterns:**
```sql
-- Uses idx_abc effectively
SELECT * FROM table WHERE a = 10 AND b = 20 ORDER BY c;

-- Often cannot use leading part efficiently
SELECT * FROM table WHERE b = 20;
```

---

## Index Types & Trade-offs

<a id="q3"></a>
### Q3: What are the different types of database indexes?
**Answer:**

| Index Type | What it is good at | Trade-offs / limitations |
|------------|---------------------|-------------------------|
| **B-Tree** | Equality + range + ordering | Default choice, but larger than hash |
| **Hash** | Exact equality lookups | No range/order support |
| **Bitmap** | Low-cardinality analytical filters | Expensive on heavy OLTP writes |
| **Full-text** | Tokenized text search | Language-specific config, ranking complexity |
| **Spatial** | Geo/geometric predicates | Specialized operators and maintenance |

**Rule of thumb:** start with B-Tree unless a query type clearly needs another structure.

```sql
-- B-Tree (default)
CREATE INDEX idx_name ON users(name);

-- Hash (PostgreSQL)
CREATE INDEX idx_email_hash ON users USING HASH (email);

-- Full-text (PostgreSQL)
CREATE INDEX idx_content_fts ON articles USING GIN (to_tsvector('english', content));
```

<a id="q4"></a>
### Q4: What are PostgreSQL-specific index types (GIN, GiST)?
**Answer:**

**GIN (Generalized Inverted Index)**
- Best for: full-text, arrays, JSONB containment, `hstore`
- Optimized for membership/containment lookups
- Usually larger index and slower writes than B-Tree

```sql
-- Full-text search
CREATE INDEX idx_fts ON articles USING GIN (to_tsvector('english', body));

-- JSONB containment
CREATE INDEX idx_jsonb ON products USING GIN (attributes);
SELECT * FROM products WHERE attributes @> '{"color": "red"}';

-- Array containment
CREATE INDEX idx_tags ON posts USING GIN (tags);
SELECT * FROM posts WHERE tags @> ARRAY['java', 'spring'];
```

**GiST (Generalized Search Tree)**
- Best for: ranges, geometric data, nearest-neighbor classes of queries
- Flexible framework for many operator classes
- Usually faster updates than GIN, often less exact/selective for some containment use cases

```sql
-- Range types
CREATE INDEX idx_dates ON events USING GIST (date_range);
SELECT * FROM events WHERE date_range && '[2024-01-01, 2024-12-31]'::daterange;

-- Geometric data
CREATE INDEX idx_location ON places USING GIST (coordinates);
SELECT * FROM places WHERE coordinates <@ box '((0,0),(100,100))';
```

| Feature | GIN | GiST |
|---------|-----|------|
| Typical read pattern | Exact containment / term match | Overlap, nearest, geometric |
| Write/update cost | Higher | Lower |
| Index size | Larger | Smaller |
| Typical examples | JSONB/array/full-text | PostGIS, ranges, spatial |

---

## Physical Layout

<a id="q5"></a>
### Q5: What is the difference between clustered and non-clustered indexes?
**Answer:**

| Clustered Index | Non-Clustered Index |
|-----------------|---------------------|
| Defines physical row order | Separate structure pointing to row locations |
| Usually one per table | Many per table |
| Great for range scans by key | May require extra heap lookup |
| Strong locality for sequential access | More random I/O on fetch |

**MySQL/InnoDB:**
```sql
-- Primary key IS the clustered index
CREATE TABLE users (
    id INT PRIMARY KEY,  -- Clustered index
    email VARCHAR(255),
    INDEX idx_email (email)  -- Non-clustered (secondary)
);
```

**SQL Server:**
```sql
-- Explicitly create clustered index
CREATE CLUSTERED INDEX idx_date ON orders(order_date);
CREATE NONCLUSTERED INDEX idx_customer ON orders(customer_id);
```

**PostgreSQL:**
- No true clustered indexes (heap tables)
- `CLUSTER` command reorders once but doesn't maintain

```sql
-- PostgreSQL CLUSTER (one-time reorder)
CLUSTER orders USING idx_order_date;
```

**Interview nuance:** in PostgreSQL, "clustered behavior" is an operational optimization, not a persistent storage guarantee like InnoDB's primary key clustering.

---

[← Back to Database Index](README.md)
