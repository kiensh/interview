# Database Indexing Interview Questions

---

## Table of Contents

- [Q1: What is a database index and how does it work?](#q1)
- [Q2: What is a composite index and what is the leftmost prefix rule?](#q2)
- [Q3: What are the different types of database indexes?](#q3)
- [Q4: What are PostgreSQL-specific index types (GIN, GiST)?](#q4)
- [Q5: What is the difference between clustered and non-clustered indexes?](#q5)

---

<a id="q1"></a>
### Q1: What is a database index and how does it work?
**Answer:**
An index is a data structure that improves query speed at the cost of additional storage and slower writes.

**Common types:**
- **B-Tree**: Default, good for range queries and equality
- **Hash**: Fast equality lookups, no range queries
- **Bitmap**: Low cardinality columns
- **Full-text**: Text search
- **Spatial**: R-tree for geographic/geometric data

```sql
-- Create index
CREATE INDEX idx_user_email ON users(email);

-- Composite index
CREATE INDEX idx_user_name_date ON users(last_name, created_at);
```

<a id="q2"></a>
### Q2: What is a composite index and what is the leftmost prefix rule?
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

<a id="q3"></a>
### Q3: What are the different types of database indexes?
**Answer:**

| Index Type | Description | Best For |
|------------|-------------|----------|
| **B-Tree** | Balanced tree structure, default in most DBs | Range queries, equality, sorting |
| **Hash** | Hash table based | Exact equality lookups only |
| **Bitmap** | Bit arrays for each distinct value | Low cardinality columns (gender, status) |
| **Full-text** | Inverted index for text search | Text search, keywords |
| **Spatial** | R-tree or similar for geographic data | Geographic/geometric queries |

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
- Best for: Full-text search, JSONB, arrays, hstore
- Faster reads, slower writes
- Larger index size

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
- Best for: Geometric/spatial data, range types, full-text (with lower precision)
- More flexible, supports various operators
- Good for overlapping ranges

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
| Read speed | Faster | Slower |
| Write speed | Slower | Faster |
| Index size | Larger | Smaller |
| Best for | Exact containment | Overlapping/nearest |

<a id="q5"></a>
### Q5: What is the difference between clustered and non-clustered indexes?
**Answer:**

| Clustered Index | Non-Clustered Index |
|-----------------|---------------------|
| Determines physical order of data | Separate structure pointing to data |
| Only ONE per table | Multiple allowed per table |
| Faster for range queries | Requires additional lookup |
| Table IS the index | Index is separate from table |
| Slower inserts (maintains order) | Faster inserts |

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

---

[← Back to Database Index](README.md)
