# Database Fundamentals
### A Complete Reference for DevOps Engineers

> **How to use this document:** Databases are the most critical piece of most production systems — they hold all the state. As a DevOps engineer you're not expected to write complex queries, but you absolutely must understand how databases work, why they fail, and how to operate them safely. This document gives you that foundation.

---

## Table of Contents

1. [What is a Database?](#1-what-is-a-database)
2. [How a Database Engine Works Internally](#2-how-a-database-engine-works-internally)
3. [SQL Fundamentals](#3-sql-fundamentals)
4. [Joins — The Most Important SQL Concept](#4-joins--the-most-important-sql-concept)
5. [Indexes — The Difference Between Fast and Broken](#5-indexes--the-difference-between-fast-and-broken)
6. [Transactions & ACID](#6-transactions--acid)
7. [Locks & Concurrency](#7-locks--concurrency)
8. [Query Execution & EXPLAIN](#8-query-execution--explain)
9. [Normalization](#9-normalization)
10. [RDBMS vs NoSQL — When to Use Which](#10-rdbms-vs-nosql--when-to-use-which)
11. [CAP Theorem](#11-cap-theorem)
12. [Replication](#12-replication)
13. [Sharding & Partitioning](#13-sharding--partitioning)
14. [Connection Pooling](#14-connection-pooling)
15. [PostgreSQL — The DevOps Engineer's Database](#15-postgresql--the-devops-engineers-database)
16. [MySQL / MariaDB](#16-mysql--mariadb)
17. [Redis — In-Memory Data Store](#17-redis--in-memory-data-store)
18. [MongoDB — Document Store](#18-mongodb--document-store)
19. [Database Backups & Recovery](#19-database-backups--recovery)
20. [Database Performance & Monitoring](#20-database-performance--monitoring)
21. [Database Security](#21-database-security)
22. [Migrations — Changing Schemas Safely](#22-migrations--changing-schemas-safely)
23. [Quick Reference Cheat Sheet](#23-quick-reference-cheat-sheet)

---

## 1. What is a Database?

A **database** is an organized collection of structured data, managed by software called a **Database Management System (DBMS)**. The DBMS handles storing, retrieving, updating, and securing that data.

### Why Not Just Use Files?

You could store data in CSV or JSON files. But as soon as you have:
- Multiple users reading/writing simultaneously
- Need to find data quickly without reading the whole file
- Need to ensure partial writes don't corrupt data
- Need to enforce relationships between data

...you need a database. A DBMS solves all of these problems systematically.

### What a DBMS Provides

| Capability | What it means |
|---|---|
| **Persistence** | Data survives process restarts and crashes |
| **Concurrent access** | Many clients read/write simultaneously without corruption |
| **Query language** | Structured way to find and manipulate data |
| **Indexing** | Fast lookups without scanning all data |
| **Transactions** | Group operations — all succeed or all fail |
| **Access control** | Who can read, write, or modify what |
| **Backup & recovery** | Restore data after failure |
| **Integrity constraints** | Enforce rules on data (NOT NULL, UNIQUE, FOREIGN KEY) |

### Types of Databases

```
Databases
├── Relational (SQL)
│   ├── PostgreSQL      ← general purpose, most powerful
│   ├── MySQL/MariaDB   ← widely deployed, especially in web apps
│   ├── SQLite          ← embedded, single file, no server
│   └── Oracle/SQL Server ← enterprise
│
├── Document Store
│   ├── MongoDB         ← JSON-like documents
│   └── CouchDB
│
├── Key-Value Store
│   ├── Redis           ← in-memory, blazing fast, caching
│   ├── DynamoDB        ← AWS managed, scales massively
│   └── etcd            ← Kubernetes uses this for all cluster state
│
├── Column-Family
│   ├── Apache Cassandra ← massive scale, time-series data
│   └── HBase
│
├── Graph Database
│   ├── Neo4j           ← highly connected data (social networks)
│   └── Amazon Neptune
│
├── Time Series
│   ├── InfluxDB        ← metrics, IoT
│   ├── TimescaleDB     ← PostgreSQL extension for time series
│   └── Prometheus      ← metrics (also a monitoring tool)
│
└── Search Engine
    ├── Elasticsearch   ← full-text search
    └── OpenSearch
```

---

## 2. How a Database Engine Works Internally

Understanding database internals makes you better at every operational task — tuning, troubleshooting, backups, replication. This section is dense but worth it.

### The Storage Layer

Databases store data in **pages** (also called blocks) — fixed-size units of disk I/O, typically 8KB (PostgreSQL default) or 16KB (MySQL default).

A **table** is stored as a collection of pages on disk. Each page holds multiple rows. When the database needs a row, it reads the entire page containing that row into memory (the buffer pool).

```
Disk:
┌─────────────┬─────────────┬─────────────┬─────────────┐
│   Page 1    │   Page 2    │   Page 3    │   Page 4    │
│ (rows 1-50) │(rows 51-100)│(rows 101-50)│(rows 151-..)|
└─────────────┴─────────────┴─────────────┴─────────────┘
                    8KB each

Memory (Buffer Pool):
┌─────────────┬─────────────┐
│   Page 1    │   Page 3    │  (frequently accessed pages cached here)
└─────────────┴─────────────┘
```

### The Buffer Pool (Buffer Cache)

The **buffer pool** is RAM that the database uses to cache disk pages. This is the single most important performance variable in a database.

- When a query needs a row, the database first checks if the page is in the buffer pool (**cache hit** — fast)
- If not (**cache miss**), it reads from disk (slow) and loads the page into the buffer pool
- When the buffer pool is full, the **LRU (Least Recently Used)** algorithm evicts the oldest pages

```
Query hits row in page 42
  ↓
Is page 42 in buffer pool?
  Yes → return row (microseconds)
  No  → read page 42 from disk (milliseconds — 1000x slower)
        → put page 42 in buffer pool
        → return row
```

**Key insight:** A database that fits entirely in RAM is much faster than one that must hit disk regularly. This is why PostgreSQL's `shared_buffers` and MySQL's `innodb_buffer_pool_size` are the first things you tune — make them as large as you can afford.

```bash
# PostgreSQL: Check buffer pool hit rate (should be >99%)
SELECT
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) * 100 AS cache_hit_rate
FROM pg_statio_user_tables;
```

### The Write Ahead Log (WAL) / Redo Log

This is how databases survive crashes without losing data or corrupting themselves.

**The problem:** Writing to disk is not atomic. If the database crashes mid-write (power failure, OOM kill), the data file could be in a corrupt, half-written state.

**The solution — WAL (Write-Ahead Logging):**
1. Before modifying any data page, first write the change to the **WAL** (an append-only log file)
2. Once the WAL record is safely on disk → then update the actual data page in memory
3. The modified page is written to disk later (asynchronously) by the **background writer**

On crash recovery:
- Database reads the WAL
- Replays any changes that made it to the WAL but not yet to the data files
- Database is restored to a consistent state

```
Transaction commits:
  1. WAL record written to WAL buffer
  2. WAL buffer flushed to disk (fsync) ← this is what makes the commit durable
  3. Respond "committed" to client
  4. Data page updated in buffer pool (async)
  5. Eventually written to disk by background writer
```

The WAL is also the foundation of **replication** — standby servers receive and replay the primary's WAL stream.

### The Query Planner / Optimizer

When you submit a SQL query, the database doesn't just execute it literally. The **query planner** figures out the most efficient way to get the results.

For a query like:
```sql
SELECT * FROM orders WHERE customer_id = 123 AND status = 'pending';
```

The planner considers:
- Should it use an index on `customer_id`? On `status`? On both? On neither (full table scan)?
- What order should it evaluate conditions?
- If joining tables, which table should be the outer loop?
- Should it sort or hash for GROUP BY and ORDER BY?

The planner estimates the cost of each option based on statistics it maintains about data distribution (using `pg_statistic` / `information_schema`). It picks the **lowest estimated cost** plan.

This is why statistics being up-to-date matters — stale statistics lead to bad query plans, which cause slow queries.

```bash
# PostgreSQL: Update statistics manually
ANALYZE tablename;
ANALYZE;   # All tables

# MySQL: Update statistics
ANALYZE TABLE tablename;
```

---

## 3. SQL Fundamentals

**SQL (Structured Query Language)** is the language used to interact with relational databases. Every DevOps engineer should be comfortable reading and writing basic SQL.

### The Four Core Operations — CRUD

```sql
-- CREATE (INSERT)
INSERT INTO users (name, email, created_at)
VALUES ('Alice', 'alice@example.com', NOW());

-- READ (SELECT)
SELECT id, name, email
FROM users
WHERE created_at > '2024-01-01'
ORDER BY name ASC
LIMIT 10;

-- UPDATE
UPDATE users
SET email = 'newalice@example.com'
WHERE id = 42;   -- ALWAYS include WHERE — otherwise updates ALL rows!

-- DELETE
DELETE FROM users
WHERE id = 42;   -- ALWAYS include WHERE — otherwise deletes ALL rows!
```

> **The most dangerous SQL mistakes:** Running UPDATE or DELETE without a WHERE clause. Always write the WHERE clause FIRST, verify it, then add the rest.

### SELECT in Detail

```sql
SELECT
    u.id,
    u.name,
    u.email,
    COUNT(o.id) AS order_count,
    SUM(o.total) AS total_spent
FROM users u                           -- Main table, aliased as 'u'
JOIN orders o ON o.customer_id = u.id  -- Join to orders table
WHERE u.created_at > '2024-01-01'      -- Filter rows BEFORE grouping
  AND u.active = true
GROUP BY u.id, u.name, u.email         -- Group for aggregation
HAVING COUNT(o.id) > 5                 -- Filter AFTER grouping (unlike WHERE)
ORDER BY total_spent DESC              -- Sort results
LIMIT 20 OFFSET 0;                     -- Pagination: 20 results, starting at 0
```

### Execution Order — Not the Same as Written Order

SQL looks like you're writing it top-to-bottom but the database executes it in a different order:

```
1. FROM / JOIN    ← which tables and how to combine them
2. WHERE          ← filter individual rows
3. GROUP BY       ← group remaining rows
4. HAVING         ← filter groups
5. SELECT         ← choose columns, calculate expressions
6. DISTINCT       ← remove duplicates
7. ORDER BY       ← sort
8. LIMIT / OFFSET ← trim results
```

This is why you can't use a SELECT alias in a WHERE clause — WHERE runs before SELECT.

```sql
-- This FAILS — alias 'full_name' doesn't exist yet when WHERE runs
SELECT first_name || ' ' || last_name AS full_name
FROM users
WHERE full_name = 'Alice Smith';   -- Error!

-- This WORKS — use the original expression in WHERE
SELECT first_name || ' ' || last_name AS full_name
FROM users
WHERE first_name || ' ' || last_name = 'Alice Smith';
-- OR use a subquery / CTE
```

### Data Types — Know the Common Ones

| Type | Description | Example |
|---|---|---|
| `INTEGER` / `INT` | 32-bit integer | `42`, `-7` |
| `BIGINT` | 64-bit integer | Large IDs, counters |
| `SERIAL` / `BIGSERIAL` | Auto-incrementing integer (PostgreSQL) | Primary keys |
| `DECIMAL(p,s)` / `NUMERIC` | Exact decimal (precision, scale) | Money: `DECIMAL(10,2)` |
| `FLOAT` / `DOUBLE` | Approximate floating point | Scientific data — NOT for money |
| `VARCHAR(n)` | Variable-length string, max n chars | Names, emails |
| `TEXT` | Unlimited length string | Long text, content |
| `CHAR(n)` | Fixed-length string, padded with spaces | Country codes |
| `BOOLEAN` | True/False | `true`, `false` |
| `DATE` | Calendar date | `2024-01-15` |
| `TIME` | Time of day | `14:30:00` |
| `TIMESTAMP` | Date + time | `2024-01-15 14:30:00` |
| `TIMESTAMPTZ` | Timestamp with timezone | Recommended for most timestamps |
| `UUID` | 128-bit unique identifier | `550e8400-e29b-41d4-a716-446655440000` |
| `JSON` / `JSONB` | JSON data (JSONB is binary, indexable) | Flexible structured data |
| `ARRAY` | Array of another type | `{1, 2, 3}` |

> **Never use FLOAT for money.** `FLOAT` is approximate — `0.1 + 0.2 = 0.30000000000000004`. Use `DECIMAL(10,2)` for currency.

### Constraints — Enforcing Data Rules

```sql
CREATE TABLE users (
    id          BIGSERIAL    PRIMARY KEY,           -- Unique, not null, auto-increment
    email       VARCHAR(255) NOT NULL UNIQUE,       -- Cannot be null, must be unique
    age         INTEGER      CHECK (age >= 0),      -- Must satisfy condition
    role        VARCHAR(50)  DEFAULT 'user',        -- Default value if not specified
    team_id     INTEGER      REFERENCES teams(id)   -- Foreign key
                             ON DELETE SET NULL     -- What to do if referenced row deleted
);
```

Constraint types:
- **PRIMARY KEY** — Unique identifier for each row. Automatically NOT NULL + UNIQUE. Table can only have one.
- **NOT NULL** — Column must have a value
- **UNIQUE** — All values in column must be different
- **CHECK** — Custom condition must be true
- **DEFAULT** — Value to use if none provided
- **FOREIGN KEY** — References a row in another table — enforces referential integrity

---

## 4. Joins — The Most Important SQL Concept

A **JOIN** combines rows from two or more tables based on a related column. This is the heart of relational data modeling.

### Visual Guide to Join Types

```
Table A:              Table B:
id | name             id | city
---|-----             ---|--------
 1 | Alice             1 | New York
 2 | Bob               2 | London
 3 | Carol             4 | Paris   ← no matching user
```

#### INNER JOIN — Only matching rows from both tables
```sql
SELECT a.name, b.city
FROM users a
INNER JOIN locations b ON a.id = b.id;

-- Result:
name  | city
------|---------
Alice | New York
Bob   | London
-- Carol is excluded (no matching row in B)
-- Paris is excluded (no matching row in A)
```

#### LEFT JOIN — All rows from left table, matching from right (NULL if no match)
```sql
SELECT a.name, b.city
FROM users a
LEFT JOIN locations b ON a.id = b.id;

-- Result:
name  | city
------|---------
Alice | New York
Bob   | London
Carol | NULL      ← Carol included, city is NULL
```

#### RIGHT JOIN — All rows from right table, matching from left (NULL if no match)
```sql
SELECT a.name, b.city
FROM users a
RIGHT JOIN locations b ON a.id = b.id;

-- Result:
name  | city
------|---------
Alice | New York
Bob   | London
NULL  | Paris     ← Paris included, name is NULL
```

#### FULL OUTER JOIN — All rows from both tables
```sql
SELECT a.name, b.city
FROM users a
FULL OUTER JOIN locations b ON a.id = b.id;

-- Result:
name  | city
------|---------
Alice | New York
Bob   | London
Carol | NULL
NULL  | Paris
```

#### CROSS JOIN — Every row in A paired with every row in B (Cartesian product)
```sql
SELECT a.name, b.city
FROM users a
CROSS JOIN locations b;
-- 3 users × 3 locations = 9 rows
-- Rarely intentional — be careful
```

### Self Join — Joining a Table to Itself

```sql
-- Find employees and their manager (manager is also an employee)
SELECT
    e.name AS employee,
    m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

### Multiple Joins

```sql
SELECT
    u.name,
    o.order_date,
    p.product_name,
    oi.quantity
FROM users u
JOIN orders o       ON o.customer_id = u.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p     ON p.id = oi.product_id
WHERE u.id = 42
  AND o.order_date > '2024-01-01';
```

### Subqueries

A subquery is a query nested inside another query:

```sql
-- Find users who have placed more than 10 orders
SELECT name
FROM users
WHERE id IN (
    SELECT customer_id
    FROM orders
    GROUP BY customer_id
    HAVING COUNT(*) > 10
);
```

### CTEs (Common Table Expressions) — Named Subqueries

CTEs make complex queries readable by breaking them into named steps:

```sql
WITH
-- Step 1: Find high-value customers
high_value AS (
    SELECT customer_id, SUM(total) AS lifetime_value
    FROM orders
    GROUP BY customer_id
    HAVING SUM(total) > 1000
),
-- Step 2: Get their details
customer_details AS (
    SELECT u.name, u.email, hv.lifetime_value
    FROM users u
    JOIN high_value hv ON hv.customer_id = u.id
)
-- Step 3: Final result
SELECT *
FROM customer_details
ORDER BY lifetime_value DESC;
```

---

## 5. Indexes — The Difference Between Fast and Broken

An **index** is a separate data structure that the database maintains to speed up data lookups. It's a sorted reference to rows, allowing the database to find data without scanning every row in the table.

### The Problem Without Indexes

```sql
SELECT * FROM users WHERE email = 'alice@example.com';
```

Without an index on `email`:
- The database reads **every single row** in the `users` table
- Checks if each row's email matches
- This is called a **full table scan (Seq Scan)**
- If `users` has 10 million rows → reads 10 million rows to find 1

With an index on `email`:
- Database looks up `alice@example.com` in the B-Tree index
- Finds the location of the matching row(s) in microseconds
- Reads only that row (or a handful of pages)

### The B-Tree Index (Default Index Type)

Most database indexes use a **B-Tree (Balanced Tree)** structure.

```
                  [M]
                /     \
            [F, J]    [R, V]
           /  |  \   /  |  \
        [A-E][G-I][K-L][N-Q][S-U][W-Z]
```

- Sorted — allows range queries (`WHERE age BETWEEN 20 AND 30`)
- Balanced — every lookup takes the same number of steps (O(log n))
- Supports equality (`=`), ranges (`<`, `>`, `BETWEEN`), sorting (ORDER BY)

### Types of Indexes

| Type | Best for | Notes |
|---|---|---|
| **B-Tree** | Equality, range, sort — default | Most common |
| **Hash** | Equality lookups only | Faster for =, useless for ranges |
| **GIN** | Arrays, JSONB, full-text search | PostgreSQL-specific |
| **GiST** | Geometric, full-text, custom types | PostgreSQL-specific |
| **Partial index** | A subset of rows | Smaller, faster for specific queries |
| **Composite index** | Multiple columns | Order matters! |
| **Covering index** | Includes all queried columns | Avoids table lookup (index-only scan) |

### Creating Indexes

```sql
-- Basic index
CREATE INDEX idx_users_email ON users(email);

-- Unique index (also enforces uniqueness constraint)
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Composite index (column order matters!)
CREATE INDEX idx_orders_customer_date ON orders(customer_id, created_at);

-- Partial index (only index rows matching condition — smaller, faster)
CREATE INDEX idx_active_users ON users(email) WHERE active = true;

-- Covering index (include extra columns so table doesn't need to be accessed)
CREATE INDEX idx_users_email_covering ON users(email) INCLUDE (name, status);

-- PostgreSQL: Create without blocking writes (takes longer but safe in production)
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```

### Composite Index Column Order

For composite indexes, column order determines which queries benefit:

```sql
CREATE INDEX idx_orders_composite ON orders(customer_id, status, created_at);

-- This USES the index (starts with leftmost column)
WHERE customer_id = 42
WHERE customer_id = 42 AND status = 'pending'
WHERE customer_id = 42 AND status = 'pending' AND created_at > '2024-01-01'

-- This DOES NOT use the index efficiently (skips leftmost column)
WHERE status = 'pending'                        -- can't use index
WHERE created_at > '2024-01-01'                -- can't use index
WHERE status = 'pending' AND created_at > ...  -- can't use index
```

**Rule:** Design composite indexes to match your most common query patterns. Put equality columns first, range columns last.

### When Indexes Hurt

Indexes are not free:
- **Write overhead:** Every INSERT, UPDATE, DELETE must also update all indexes on the table
- **Storage:** Indexes use disk space (sometimes as much as the table itself)
- **Planning overhead:** Too many indexes confuse the query planner

Don't index every column. Index columns that appear in:
- WHERE clauses (equality and range lookups)
- JOIN conditions (`ON a.id = b.user_id` — foreign keys should be indexed)
- ORDER BY / GROUP BY clauses

### Finding Missing and Unused Indexes

```sql
-- PostgreSQL: Find tables doing too many sequential scans (potential missing index)
SELECT relname, seq_scan, seq_tup_read, idx_scan
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;

-- PostgreSQL: Find unused indexes (wasting write overhead)
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND schemaname = 'public';

-- PostgreSQL: Find duplicate/redundant indexes
SELECT pg_size_pretty(pg_relation_size(idx)) AS size, idx
FROM (SELECT indexrelid::regclass AS idx FROM pg_index) t
ORDER BY pg_relation_size(idx) DESC;
```

---

## 6. Transactions & ACID

A **transaction** is a sequence of database operations that are treated as a single unit of work — either all succeed or all fail together.

### The Classic Example — Bank Transfer

```sql
BEGIN;  -- Start transaction

-- Step 1: Deduct from Alice's account
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

-- Step 2: Add to Bob's account
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;  -- Both succeed → commit

-- OR if something went wrong:
ROLLBACK;  -- Neither change is applied
```

Without transactions: if the system crashes between step 1 and step 2, Alice loses $100 and Bob gets nothing.

### ACID Properties

**ACID** defines the guarantees a reliable database transaction must provide:

#### A — Atomicity
"All or nothing." A transaction either fully completes or has zero effect. There is no partial success.

If a transaction has 5 operations and the 3rd one fails, the database rolls back the 2 that already succeeded. You never see intermediate state.

#### C — Consistency
A transaction brings the database from one valid state to another valid state. All constraints, rules, and business logic must hold before and after.

Example: If a `CHECK (balance >= 0)` constraint exists, no transaction can make a balance negative — the database rejects it.

#### I — Isolation
Concurrent transactions don't interfere with each other. Each transaction sees a consistent view of the database as if it ran alone.

Without isolation: Two transactions reading and writing the same row simultaneously could cause all kinds of corruption.

#### D — Durability
Once a transaction is committed, it will survive any subsequent failure (crash, power outage). This is why the WAL exists — committed data is written to the WAL before confirming the commit.

### Transaction Isolation Levels

Full isolation is expensive. Most databases offer configurable isolation levels with different performance/safety tradeoffs:

#### Read Phenomena (Problems That Isolation Levels Protect Against)

**Dirty Read:** Transaction A reads data that Transaction B has written but not yet committed. If B rolls back, A has read data that never officially existed.

**Non-Repeatable Read:** Transaction A reads a row, Transaction B updates and commits it, Transaction A reads the same row again and gets different data.

**Phantom Read:** Transaction A runs a query and gets N rows, Transaction B inserts new rows matching the same criteria and commits, Transaction A runs the same query and gets N+M rows.

#### Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|---|---|---|---|---|
| **READ UNCOMMITTED** | Possible | Possible | Possible | Highest |
| **READ COMMITTED** | Prevented | Possible | Possible | High |
| **REPEATABLE READ** | Prevented | Prevented | Possible | Medium |
| **SERIALIZABLE** | Prevented | Prevented | Prevented | Lowest |

```sql
-- Set isolation level for a transaction
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
...
COMMIT;

-- PostgreSQL default: READ COMMITTED
-- MySQL InnoDB default: REPEATABLE READ
```

**In practice:**
- **READ COMMITTED** is fine for most web applications
- **REPEATABLE READ / SERIALIZABLE** for financial calculations, inventory systems where consistency is critical

---

## 7. Locks & Concurrency

When multiple transactions access the same data simultaneously, the database uses locks to prevent conflicts.

### Types of Locks

**Row-level locks (most common in practice):**
- **Shared lock (S / Read lock):** Multiple transactions can hold shared locks simultaneously — they all want to read the same row.
- **Exclusive lock (X / Write lock):** Only one transaction can hold an exclusive lock — it's writing to the row. No other locks (shared or exclusive) allowed simultaneously.

**Table-level locks:**
- **ACCESS SHARE:** Acquired by SELECT — allows other reads, blocks DDL (ALTER TABLE etc.)
- **ROW EXCLUSIVE:** Acquired by INSERT/UPDATE/DELETE — blocks competing writes
- **ACCESS EXCLUSIVE:** Acquired by ALTER TABLE, DROP, VACUUM FULL — blocks everything

### Deadlock

A **deadlock** occurs when two transactions are each waiting for the other to release a lock — neither can proceed.

```
Transaction A:
  LOCK row 1 (success)
  Waiting for row 2... ← blocked by B

Transaction B:
  LOCK row 2 (success)
  Waiting for row 1... ← blocked by A

Both waiting forever → DEADLOCK
```

The database's **deadlock detector** runs periodically, detects the cycle, and kills one of the transactions (returning an error). The killed transaction's work is rolled back.

```bash
# PostgreSQL: Check for locks and waiting queries
SELECT
    pid,
    now() - pg_stat_activity.query_start AS duration,
    query,
    state,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

# Find blocking locks
SELECT
    blocked.pid AS blocked_pid,
    blocking.pid AS blocking_pid,
    blocked.query AS blocked_query,
    blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

**Preventing deadlocks:**
- Always acquire locks in the same order across transactions
- Keep transactions short — the less time a lock is held, the less chance of deadlock
- Use `NOWAIT` or `SKIP LOCKED` in non-critical operations

```sql
-- Try to lock but fail immediately instead of waiting
SELECT * FROM jobs WHERE status = 'pending' LIMIT 1 FOR UPDATE SKIP LOCKED;
-- Useful for job queues — skip rows locked by other workers
```

### MVCC — Multi-Version Concurrency Control

**MVCC** is how PostgreSQL (and most modern databases) implement isolation without readers blocking writers.

Instead of locking rows for reads:
- Every update creates a **new version** of the row
- The old version is kept until no transaction needs it anymore
- Readers see a consistent snapshot as of when their transaction started
- Writers create new versions, don't block readers

```
Time →
Row versions:    [v1: Alice, $100]   [v2: Alice, $200]   [v3: Alice, $150]

Transaction A started at T1 → always sees v1 (its snapshot)
Transaction B started at T2 → sees v2
Transaction C started at T3 → sees v3
```

**Consequence:** Old row versions accumulate. **VACUUM** (PostgreSQL) cleans up old versions that no transaction needs anymore.

```bash
# PostgreSQL: Check for table bloat (too many dead rows)
SELECT relname, n_dead_tup, n_live_tup, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

# Manual VACUUM
VACUUM tablename;        # Clean dead rows
VACUUM ANALYZE;          # Clean + update statistics (safe in production)
VACUUM FULL tablename;   # Reclaim disk space (locks table! use carefully)
```

---

## 8. Query Execution & EXPLAIN

**EXPLAIN** shows you how the database plans to execute a query. This is the single most important tool for database performance debugging.

### Reading EXPLAIN Output

```sql
EXPLAIN SELECT * FROM users WHERE email = 'alice@example.com';

-- Output (PostgreSQL):
Index Scan using idx_users_email on users  (cost=0.43..8.45 rows=1 width=120)
  Index Cond: ((email)::text = 'alice@example.com'::text)
```

```sql
EXPLAIN SELECT * FROM users WHERE active = true;

-- Output (no index on 'active'):
Seq Scan on users  (cost=0.00..2453.00 rows=81234 width=120)
  Filter: (active = true)
```

### EXPLAIN ANALYZE — Actually Run It and Measure

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;

-- Output:
Index Scan using idx_orders_customer on orders  (cost=0.43..12.45 rows=3 width=89)
                                                (actual time=0.028..0.035 rows=3 loops=1)
  Index Cond: (customer_id = 42)
Planning Time: 0.088 ms
Execution Time: 0.053 ms
```

Key terms:
- **cost=X..Y** — Estimated startup cost .. total cost (in arbitrary units)
- **rows=N** — Estimated number of rows (planner's guess based on statistics)
- **actual time=X..Y** — Real time in milliseconds (only with ANALYZE)
- **actual rows=N** — Real number of rows (if very different from estimated → stale statistics)
- **loops=N** — How many times this node was executed

### Common Plan Nodes

| Node | Meaning | Fast? |
|---|---|---|
| **Seq Scan** | Read entire table from disk | Slow on large tables |
| **Index Scan** | Use index to find rows, then fetch from table | Fast |
| **Index Only Scan** | All data in index — no table access needed | Very fast |
| **Bitmap Heap Scan** | Use index to build bitmap, then batch-read pages | Good for many rows |
| **Nested Loop** | For each row in outer → scan inner | Good for small result sets |
| **Hash Join** | Build hash table of smaller table, probe with larger | Good for larger tables |
| **Merge Join** | Both tables sorted on join key, merge | Good when both pre-sorted |
| **Sort** | Sort rows (may spill to disk if large) | Expensive if large |
| **Hash Aggregate** | Group using hash table | Common for GROUP BY |

### When Seq Scan Is Actually Fine

Don't panic when you see a Seq Scan. The planner chooses Seq Scan when:
- The table is tiny (faster than index overhead)
- The query returns a large fraction of the table (index not helpful)
- There's no suitable index

It's only a problem when you see Seq Scan on a large table for a selective query.

```sql
-- Visualize EXPLAIN output beautifully
-- Use https://explain.dalibo.com/ or https://explain.depesz.com/
-- Paste your EXPLAIN (FORMAT JSON) output there
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;
```

---

## 9. Normalization

**Normalization** is the process of organizing a database schema to reduce data redundancy and improve data integrity.

### Why Normalization Matters

Without normalization:
```
orders table:
id | customer_name | customer_email    | customer_city | product_name | price
1  | Alice         | alice@example.com | New York      | Laptop       | 999
2  | Alice         | alice@example.com | New York      | Mouse        | 29
3  | Bob           | bob@example.com   | London        | Keyboard     | 79
```

Problems:
- **Update anomaly:** If Alice moves to Boston, must update every row
- **Insertion anomaly:** Can't add a product without an order
- **Deletion anomaly:** Delete the only order for a product → lose the product info

### Normal Forms

**1NF (First Normal Form):**
- Each column contains atomic (indivisible) values
- No repeating groups
- Each row is unique

```sql
-- Violates 1NF: multiple values in one column
id | name  | phone_numbers
1  | Alice | "555-1234, 555-5678"  ← not atomic!

-- 1NF compliant: separate table for phone numbers
users: id | name
phones: user_id | phone_number
```

**2NF (Second Normal Form):**
- Must be in 1NF
- Every non-key column must depend on the ENTIRE primary key (not just part of it)
- Only relevant when using composite primary keys

```sql
-- Violates 2NF: product_name depends only on product_id, not on (order_id, product_id)
order_items: order_id | product_id | product_name | quantity
-- product_name is a partial dependency — move it to products table

-- 2NF compliant:
order_items: order_id | product_id | quantity
products:    product_id | product_name | price
```

**3NF (Third Normal Form):**
- Must be in 2NF
- No transitive dependencies (non-key column depends on another non-key column)

```sql
-- Violates 3NF: zip_code → city (city depends on zip_code, not on user_id)
users: id | name | zip_code | city

-- 3NF compliant:
users: id | name | zip_code
zip_codes: zip_code | city
```

### When to Denormalize

Normalization is not always the right answer for every situation:

**Normalize** (up to 3NF): For transactional systems (OLTP) where data is frequently written and must be consistent.

**Denormalize** (intentionally break normal forms): For read-heavy reporting or analytics (OLAP). Precompute joins by storing redundant data for faster reads.

> In practice, most web application databases are normalized to 3NF. Reporting databases and data warehouses are often intentionally denormalized (star schema, fact tables).

---

## 10. RDBMS vs NoSQL — When to Use Which

### Relational Databases (SQL)

**Strengths:**
- Strong consistency (ACID transactions)
- Powerful query language (SQL — JOINs, aggregations, subqueries)
- Schema enforced — data integrity by design
- Mature, well-understood, excellent tooling
- Best for complex relationships between data

**Weaknesses:**
- Harder to scale writes horizontally (sharding is complex)
- Schema changes can be disruptive (migrations required)
- Not great for unstructured or highly variable data

**Use when:**
- Data has clear relationships (users → orders → products)
- Data integrity is critical (financial systems, inventory)
- You need complex queries across multiple data types
- The schema is relatively stable

### NoSQL Databases

"NoSQL" means "not only SQL" — it's an umbrella term for non-relational databases. Different types solve different problems.

#### Document Stores (MongoDB, CouchDB)
Store JSON-like documents. Schema is flexible — different documents can have different fields.

**Use when:**
- Data is naturally document-shaped (product catalog with varying attributes)
- Schema evolves rapidly
- You need to store and retrieve entire objects
- Not much need for joins

**Avoid when:**
- Data has strong relationships (many-to-many)
- You need ACID transactions across multiple documents (improved in MongoDB 4.0+ but still complex)

#### Key-Value Stores (Redis, DynamoDB)
Map keys to values. Extremely simple interface, extremely fast.

**Use when:**
- Caching (most common Redis use case)
- Session storage
- Rate limiting counters
- Leaderboards (Redis sorted sets)
- Simple lookups where you always know the exact key

#### Column-Family Stores (Cassandra, HBase)
Store data in columns rather than rows. Optimized for writing and reading specific columns across many rows.

**Use when:**
- Massive scale (trillions of rows)
- Write-heavy workloads
- Time-series data
- Wide rows with many optional columns
- Geographic distribution required

**Avoid when:**
- Complex queries with JOINs
- Strong consistency required
- Low volume — complexity not worth it

#### Graph Databases (Neo4j, Neptune)
Store data as nodes and edges. Optimized for traversing relationships.

**Use when:**
- Highly connected data (social networks, recommendation engines, fraud detection)
- The relationships between data are as important as the data itself
- Need to traverse multiple levels of relationships efficiently

### The Honest Comparison

| Aspect | Relational (PostgreSQL) | Document (MongoDB) | Key-Value (Redis) |
|---|---|---|---|
| **Consistency** | Strong (ACID) | Eventually consistent (by default) | Depends on mode |
| **Query flexibility** | Very high (SQL) | Medium (limited aggregation) | Very low (key only) |
| **Horizontal scaling** | Harder | Easier | Easy |
| **Schema** | Rigid (good for integrity) | Flexible (good for iteration) | None |
| **Speed** | Fast with indexes | Fast for document retrieval | Fastest (in-memory) |
| **Data size** | Any | Any | Limited by RAM (Redis) |
| **Use case** | Transactions, reporting | Content, catalogs | Cache, sessions |

> **Reality check:** PostgreSQL can do most of what MongoDB does (JSONB column with GIN index), plus it gives you ACID and SQL. Most "we chose MongoDB for flexibility" decisions are later regretted when the data grows complex and joins become impossible. Default to PostgreSQL unless you have a specific reason not to.

---

## 11. CAP Theorem

The **CAP Theorem** (Brewer's Theorem, 2000) states that a distributed data store can only guarantee **two of three** properties simultaneously:

```
         Consistency
              /\
             /  \
            /    \
           / CA   \
          /  (RDBMS)\
         /    zone   \
        ──────────────
       /               \
      / CP               AP \
     /  (HBase,            (Cassandra,\
    /   Zookeeper)          CouchDB)   \
   ──────────────────────────────────────
Availability              Partition Tolerance
```

### The Three Properties

**Consistency (C):**
Every read receives the most recent write or an error. All nodes in the distributed system see the same data at the same time.

*Not the same as ACID Consistency — this is about distributed system consistency.*

**Availability (A):**
Every request receives a non-error response (though it may not contain the most recent data). The system is always operational.

**Partition Tolerance (P):**
The system continues operating even when network partitions occur (some nodes can't communicate with others).

### The Critical Insight

In a distributed system, network partitions **will happen** — this is a fact of distributed computing. So you cannot give up Partition Tolerance.

**Therefore, in practice, you're always choosing between C and A when a partition occurs:**

**CP System** (Consistency + Partition Tolerance):
When a partition occurs, the system refuses to respond (sacrifices Availability) rather than return potentially stale data.
- Examples: HBase, Zookeeper, etcd, Redis (in certain configs)
- Use when: Data accuracy is critical — banking, inventory where stale reads cause real harm

**AP System** (Availability + Partition Tolerance):
When a partition occurs, nodes continue serving requests (sacrifices Consistency) — some nodes may return stale data.
- Examples: Cassandra, CouchDB, DynamoDB (eventually consistent), DNS
- Use when: Availability is critical, stale reads are acceptable — social media, caching, DNS

**CA System** (Consistency + Availability):
Only possible if you never have network partitions — meaning it's a single node or you trust your network completely.
- Traditional RDBMS on a single server
- Once you distribute it, you must choose between C and A when a partition occurs

### PACELC — The More Practical Extension

CAP only describes behavior during partitions. **PACELC** extends this:

> If there's a Partition, choose between Availability and Consistency. Else (no partition), choose between Latency and Consistency.

Even without network failures, there's a tradeoff between consistency and latency in distributed systems — stronger consistency requires more coordination between nodes (higher latency).

---

## 12. Replication

**Replication** copies data from one database server (primary/master) to one or more other servers (replicas/standbys). This provides:
- **High availability:** If the primary fails, a replica can take over
- **Read scalability:** Direct read queries to replicas, reducing load on the primary
- **Disaster recovery:** Geographically distributed replicas

### Synchronous vs Asynchronous Replication

**Asynchronous Replication (default in most databases):**
- Primary commits the transaction and immediately tells the client "committed"
- Changes are sent to replicas in the background
- **Pros:** Low latency (primary doesn't wait for replica confirmation)
- **Cons:** If primary fails before the replica gets the changes, those changes are **lost** (replication lag)

```
Client → Primary COMMIT → "OK"
                ↓ (background, delayed)
            Replica
```

**Synchronous Replication:**
- Primary sends the WAL to the replica and waits for acknowledgment before confirming the commit
- **Pros:** Zero data loss (if primary fails, replica has all committed data)
- **Cons:** Every commit waits for network round-trip to replica → higher latency

```
Client → Primary COMMIT → Wait for replica ACK → "OK"
                ↓ ←─────────────────────────────────
            Replica (confirms receipt of WAL)
```

```sql
-- PostgreSQL: Configure synchronous replication
-- In postgresql.conf on primary:
synchronous_standby_names = 'replica1'
synchronous_commit = on   -- default; 'remote_write' or 'remote_apply' for other levels
```

### Replication Lag

**Replication lag** is how far behind a replica is from the primary. Measured in bytes (WAL bytes) or time.

Causes:
- Slow network between primary and replica
- Replica is under heavy query load (can't replay WAL fast enough)
- Long-running queries on the replica that block WAL replay (some configs)

```bash
# PostgreSQL: Check replication lag on primary
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    (sent_lsn - replay_lsn) AS replication_lag_bytes
FROM pg_stat_replication;

# On replica: Check lag
SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag_time;
```

### Primary-Replica Architecture

```
         ┌──────────────────────────────────────┐
Writes → │          PRIMARY DATABASE             │
Reads ↗  │  (handles writes + some reads)        │
         └──────────────┬───────────────────────┘
                        │ WAL stream
            ┌───────────┴────────────┐
            ↓                        ↓
   ┌────────────────┐      ┌────────────────┐
   │   REPLICA 1    │      │   REPLICA 2    │
   │ (read traffic) │      │  (standby/DR)  │
   └────────────────┘      └────────────────┘
```

### Failover

**Failover** is the process of promoting a replica to become the new primary when the original primary fails.

**Manual failover:** A DBA manually promotes a replica.
**Automatic failover:** Tools like Patroni (PostgreSQL), MHA (MySQL), or managed cloud services automatically detect primary failure and promote a replica.

```bash
# PostgreSQL: Promote a replica to primary
pg_ctl promote -D /var/lib/postgresql/data

# Or create the trigger file (older method)
touch /var/lib/postgresql/data/failover.trigger
```

### Read Replicas in Cloud Environments

AWS RDS, GCP Cloud SQL, Azure Database all support read replicas out of the box. Key consideration:

**Replication lag** means reads from replicas may return slightly stale data. This is acceptable for:
- Analytics dashboards
- Report generation
- Non-critical reads

Not acceptable for:
- Reading data immediately after writing it ("read your own writes" scenarios)
- Inventory levels where accuracy is critical

---

## 13. Sharding & Partitioning

When a single database server can't handle the load or data volume, you distribute the data across multiple servers.

### Vertical Scaling vs Horizontal Scaling

**Vertical scaling:** Bigger server — more CPU, more RAM, faster SSD.
- Simple, no application changes
- Has limits — you can only buy so much hardware
- A single point of failure

**Horizontal scaling (sharding):** More servers, each holding a subset of data.
- Near-unlimited scale
- Complex — application must know which server has which data
- Queries spanning multiple shards are expensive

### Table Partitioning

**Partitioning** divides one large table into smaller physical pieces (partitions), but presents them as one logical table. Done on a single server.

Benefits:
- Query performance — queries can skip partitions that don't match (partition pruning)
- Maintenance — vacuum, backup individual partitions independently
- Data lifecycle — drop old time-based partitions instead of DELETE (much faster)

```sql
-- PostgreSQL: Range partitioning by month
CREATE TABLE events (
    id        BIGSERIAL,
    event_time TIMESTAMPTZ NOT NULL,
    data      JSONB
) PARTITION BY RANGE (event_time);

-- Create monthly partitions
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE events_2024_02 PARTITION OF events
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Query uses partition pruning automatically:
SELECT * FROM events WHERE event_time >= '2024-01-01' AND event_time < '2024-02-01';
-- Only reads events_2024_01, skips all other partitions
```

### Sharding

**Sharding** splits data across multiple database servers. Each server (shard) holds a subset of the total data.

**Shard key:** The column(s) used to determine which shard a row lives on.

**Hash sharding:**
```
shard_id = hash(user_id) % number_of_shards

user_id 1001 → hash → shard 0
user_id 1002 → hash → shard 1
user_id 1003 → hash → shard 2
```
Even distribution, but range queries span all shards.

**Range sharding:**
```
user_id 1-1000000    → shard 0
user_id 1000001-2000000 → shard 1
user_id 2000001+     → shard 2
```
Range queries stay on one shard, but can create hot spots.

**Challenges of sharding:**
- JOINs across shards are very expensive (or impossible)
- Transactions spanning multiple shards are complex (distributed transactions)
- Resharding when adding more shards is painful
- Application must be aware of sharding logic (unless using a proxy)

> **Advice:** Don't shard until you absolutely must. Vertical scaling, read replicas, caching, and better indexing can get you much farther than people expect. Sharding adds immense operational complexity.

---

## 14. Connection Pooling

Every database connection consumes resources — memory on the database server (~5-10MB per connection in PostgreSQL), CPU for connection setup, and file descriptors.

A web application with 100 worker processes, each opening 5 connections = 500 database connections. This can overwhelm the database.

### What Connection Pooling Does

A **connection pool** maintains a cache of pre-established database connections. Application requests borrow a connection from the pool, use it, return it.

```
Without pooling:
  App Worker 1 → [open connection] → DB query → [close connection]
  App Worker 2 → [open connection] → DB query → [close connection]
  (Connection setup overhead every time)

With pooling:
  Pool pre-opens 20 connections to DB
  App Worker 1 → [borrow connection from pool] → DB query → [return connection]
  App Worker 2 → [borrow connection from pool] → DB query → [return connection]
  (Instant — connection already established)
```

### Types of Connection Pooling

**Application-level pooling:**
Each application instance maintains its own pool.
- Simple, no extra component
- Can result in too many total connections (10 app instances × 20 connections = 200)
- Libraries: PgBouncer, HikariCP (Java), pgx (Go), SQLAlchemy (Python)

**External pooler (PgBouncer for PostgreSQL):**
A separate process that pools connections for all application instances.
```
1000 App Workers ──→ PgBouncer (20 pooled connections) ──→ PostgreSQL
```
PostgreSQL only sees 20 connections regardless of app worker count.

### PgBouncer Modes

| Mode | Behavior | Best for |
|---|---|---|
| **Session pooling** | Connection held for entire session | Compatibility — stateful sessions |
| **Transaction pooling** | Connection returned to pool after each transaction | Most web apps (recommended) |
| **Statement pooling** | Connection returned after each statement | Rare — limited compatibility |

```ini
# PgBouncer config: /etc/pgbouncer/pgbouncer.ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
pool_mode = transaction
max_client_conn = 10000    # App connections to PgBouncer
default_pool_size = 25     # Connections to actual PostgreSQL
```

```bash
# Check PgBouncer stats
psql -p 6432 -U pgbouncer pgbouncer -c "SHOW POOLS;"
psql -p 6432 -U pgbouncer pgbouncer -c "SHOW STATS;"
psql -p 6432 -U pgbouncer pgbouncer -c "SHOW CLIENTS;"
```

### Checking Current Connections

```sql
-- PostgreSQL: Current connections by state
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;

-- PostgreSQL: Max connections setting
SHOW max_connections;

-- PostgreSQL: Connection details
SELECT pid, usename, application_name, client_addr, state, query
FROM pg_stat_activity
WHERE state != 'idle';

-- Kill a stuck connection
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND query_start < now() - interval '10 minutes';
```

---

## 15. PostgreSQL — The DevOps Engineer's Database

PostgreSQL is the most feature-rich open-source relational database. It's the default choice for new projects.

### Key Configuration Files

```bash
# Main config file
/etc/postgresql/15/main/postgresql.conf   # Debian/Ubuntu
/var/lib/pgsql/15/data/postgresql.conf    # RHEL/CentOS

# Authentication config
/etc/postgresql/15/main/pg_hba.conf

# Replication config
/etc/postgresql/15/main/pg_ident.conf
```

### Critical postgresql.conf Settings

```ini
# Memory — most important
shared_buffers = 4GB          # 25% of total RAM — PostgreSQL's buffer pool
work_mem = 64MB               # Per-sort/hash operation (per query can use multiple)
                              # Set too high → OOM when many queries run simultaneously
maintenance_work_mem = 1GB    # For VACUUM, CREATE INDEX, ALTER TABLE

# Write performance
wal_level = replica           # Minimum for replication
synchronous_commit = on       # off for max performance (risk: lose last ~100ms of commits on crash)
checkpoint_completion_target = 0.9  # Spread checkpoint writes to reduce I/O spikes
max_wal_size = 4GB            # Before forcing a checkpoint

# Connections
max_connections = 200         # Keep reasonable — use PgBouncer for more

# Query planner
effective_cache_size = 12GB   # OS + PostgreSQL cache — help planner estimate I/O cost
random_page_cost = 1.1        # For SSDs (default 4.0 is for spinning disks)

# Logging
log_min_duration_statement = 1000  # Log queries taking more than 1 second
log_line_prefix = '%t [%p]: [%l-1] user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_lock_waits = on
log_temp_files = 0            # Log temp file creation (usually indicates work_mem too low)
```

### pg_hba.conf — Client Authentication

```
# TYPE  DATABASE    USER        ADDRESS         METHOD
local   all         postgres                    peer        # Unix socket, postgres user
local   all         all                         md5         # Unix socket, password
host    all         all         127.0.0.1/32    md5         # Local TCP, password
host    mydb        appuser     10.0.0.0/8      scram-sha-256  # Network, SCRAM
hostssl all         all         0.0.0.0/0       scram-sha-256  # SSL required
```

### Essential psql Commands

```bash
# Connect
psql -U postgres                          # Local as postgres user
psql -h hostname -U user -d database      # Remote

# psql meta-commands (run inside psql)
\l                  # List databases
\c dbname           # Connect to database
\dt                 # List tables
\dt schema.*        # List tables in schema
\d tablename        # Describe table (columns, indexes, constraints)
\di                 # List indexes
\df                 # List functions
\x                  # Toggle expanded output (useful for wide tables)
\timing             # Toggle query timing
\e                  # Open editor for complex queries
\q                  # Quit
\!                  # Run shell command
```

### Essential PostgreSQL Admin Queries

```sql
-- Database sizes
SELECT datname, pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database ORDER BY pg_database_size(datname) DESC;

-- Table sizes with indexes
SELECT
    tablename,
    pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total,
    pg_size_pretty(pg_relation_size(tablename::regclass)) AS table,
    pg_size_pretty(pg_indexes_size(tablename::regclass)) AS indexes
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(tablename::regclass) DESC;

-- Long-running queries
SELECT pid, now() - query_start AS duration, query, state
FROM pg_stat_activity
WHERE state != 'idle' AND now() - query_start > interval '30 seconds'
ORDER BY duration DESC;

-- Cache hit ratio (should be >99%)
SELECT
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) * 100 AS hit_rate
FROM pg_statio_user_tables;

-- Index usage stats
SELECT relname, indexrelname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;
```

---

## 16. MySQL / MariaDB

MySQL is the most widely deployed open-source database — found in the vast majority of LAMP stack applications. MariaDB is a community fork with additional features.

### InnoDB — The Default Storage Engine

MySQL uses **InnoDB** as its default storage engine. Key properties:
- ACID-compliant transactions
- Row-level locking
- Foreign key support
- MVCC for concurrency
- The buffer pool (like PostgreSQL's shared_buffers)

```ini
# Critical MySQL settings (my.cnf)
[mysqld]
innodb_buffer_pool_size = 4G       # 60-80% of total RAM
innodb_log_file_size = 1G          # Larger = less checkpoint pressure, slower crash recovery
innodb_flush_log_at_trx_commit = 1 # 1=durable, 2=fast (lose 1s of commits on crash), 0=fastest (unsafe)
innodb_flush_method = O_DIRECT     # Bypass OS cache — avoid double buffering
max_connections = 200
query_cache_size = 0               # Disable query cache — it's deprecated and a bottleneck
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
```

### Essential MySQL Commands

```bash
mysql -u root -p                      # Connect as root
mysql -h hostname -u user -p database # Remote connect

# In mysql shell:
SHOW DATABASES;
USE dbname;
SHOW TABLES;
DESCRIBE tablename;
SHOW CREATE TABLE tablename\G        # Full table DDL
SHOW PROCESSLIST;                    # Running queries
SHOW ENGINE INNODB STATUS\G          # InnoDB internals — deadlocks, transactions
SHOW VARIABLES LIKE 'innodb%';       # InnoDB settings
SHOW STATUS LIKE 'Threads%';         # Connection stats
```

### MySQL vs PostgreSQL — Key Differences

| Aspect | PostgreSQL | MySQL |
|---|---|---|
| **Default isolation** | READ COMMITTED | REPEATABLE READ |
| **JSON support** | JSONB (binary, indexable) | JSON (text, limited indexing) |
| **Full-text search** | Built-in, powerful | Basic |
| **Extensions** | Rich ecosystem (PostGIS, TimescaleDB) | Limited |
| **Replication** | WAL streaming, logical | Binary log, row/statement based |
| **Sequences** | Native SERIAL/SEQUENCE | AUTO_INCREMENT |
| **Case sensitivity** | Case-sensitive by default | Case-insensitive by default |
| **NULL handling** | Strict SQL standard | Some quirks |

---

## 17. Redis — In-Memory Data Store

**Redis** (Remote Dictionary Server) is an in-memory data structure store. It's extraordinarily fast because everything is in RAM. Used for caching, sessions, queues, leaderboards, pub/sub.

### Redis Data Structures

Redis is not just a key-value store — it has rich data types:

```bash
# String — most basic
SET username "alice"
GET username              # → "alice"
INCR page_views           # Atomic increment — perfect for counters
EXPIRE session:abc123 3600  # Set TTL — auto-delete after 1 hour

# Hash — object/dictionary
HSET user:1001 name "Alice" email "alice@example.com" age "30"
HGET user:1001 name       # → "Alice"
HGETALL user:1001         # All fields
HINCRBY user:1001 age 1   # Increment a field

# List — ordered, allows duplicates
LPUSH queue:jobs "job1" "job2"   # Push to left
RPOP queue:jobs                   # Pop from right (FIFO queue)
LRANGE queue:jobs 0 -1           # Get all items
LLEN queue:jobs                  # List length

# Set — unordered, unique values
SADD online_users "alice" "bob" "carol"
SISMEMBER online_users "alice"   # → 1 (true)
SMEMBERS online_users            # All members
SCARD online_users               # Count

# Sorted Set — unique, ordered by score
ZADD leaderboard 1500 "alice" 2300 "bob" 980 "carol"
ZRANK leaderboard "alice"        # Rank (0-indexed)
ZRANGE leaderboard 0 2 WITHSCORES  # Top 3 with scores
ZINCRBY leaderboard 100 "alice"   # Add to score
```

### Redis Use Cases

**Caching:**
```python
# Pseudo-code: cache-aside pattern
def get_user(user_id):
    # Try cache first
    cached = redis.get(f"user:{user_id}")
    if cached:
        return json.loads(cached)

    # Cache miss — get from DB
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)

    # Store in cache, expire after 5 minutes
    redis.setex(f"user:{user_id}", 300, json.dumps(user))
    return user
```

**Session storage:**
```bash
SET session:abc123 '{"user_id": 1001, "role": "admin"}' EX 86400
GET session:abc123
DEL session:abc123  # Logout — delete the session
```

**Rate limiting:**
```bash
# Allow 100 requests per minute per IP
INCR ratelimit:192.168.1.1:1609459200  # Key includes minute timestamp
EXPIRE ratelimit:192.168.1.1:1609459200 60
# If value > 100 → reject request
```

**Pub/Sub — simple message bus:**
```bash
# Subscriber (in one terminal)
SUBSCRIBE channel:notifications

# Publisher (in another terminal)
PUBLISH channel:notifications "New order placed: #12345"
```

### Redis Persistence

Redis is in-memory — by default data is lost on restart. Two persistence options:

**RDB (Redis Database Backup):**
- Point-in-time snapshot saved to disk periodically
- Fast restarts, compact files
- Risk: data between last snapshot and crash is lost

**AOF (Append Only File):**
- Logs every write operation
- More durable — at most 1 second of data loss (configurable)
- Larger files, slower restart

```ini
# /etc/redis/redis.conf
# RDB snapshots
save 900 1       # Save if at least 1 key changed in 900 seconds
save 300 10      # Save if at least 10 keys changed in 300 seconds
save 60 10000    # Save if at least 10000 keys changed in 60 seconds

# AOF
appendonly yes
appendfsync everysec   # fsync every second (good balance)
```

### Redis Cluster and Sentinel

**Redis Sentinel:** Provides high availability for a primary-replica setup. Monitors instances, handles automatic failover, notifies clients of new primary.

**Redis Cluster:** Automatic sharding across multiple nodes. Data is divided into 16,384 hash slots distributed across cluster nodes.

```bash
# Check Redis info
redis-cli INFO server
redis-cli INFO memory
redis-cli INFO stats
redis-cli INFO replication

# Monitor commands in real-time (use carefully in production)
redis-cli MONITOR

# Slow log
redis-cli SLOWLOG GET 10   # Last 10 slow commands
```

---

## 18. MongoDB — Document Store

**MongoDB** stores data as BSON (Binary JSON) documents in collections (equivalent of tables). Documents in the same collection can have different fields.

### Core Concepts

| MongoDB | Relational |
|---|---|
| Database | Database |
| Collection | Table |
| Document | Row |
| Field | Column |
| `_id` field | Primary key |
| Index | Index |

### Basic MongoDB Operations

```javascript
// Insert
db.users.insertOne({ name: "Alice", email: "alice@example.com", age: 30, tags: ["admin", "dev"] })
db.users.insertMany([{...}, {...}])

// Find (SELECT)
db.users.find({ age: { $gt: 25 } })           // WHERE age > 25
db.users.find({ tags: "admin" })               // Array contains "admin"
db.users.find({}, { name: 1, email: 1 })       // Projection (SELECT name, email)
db.users.find().sort({ age: -1 }).limit(10)    // Sort + Limit
db.users.findOne({ email: "alice@example.com" })

// Update
db.users.updateOne(
    { email: "alice@example.com" },            // Filter
    { $set: { age: 31 } }                      // Update operation
)
db.users.updateMany({ active: false }, { $set: { archived: true } })

// Delete
db.users.deleteOne({ _id: ObjectId("...") })
db.users.deleteMany({ active: false })

// Aggregation Pipeline (equivalent of GROUP BY + JOIN)
db.orders.aggregate([
    { $match: { status: "completed" } },       // WHERE
    { $group: { _id: "$customer_id", total: { $sum: "$amount" } } },  // GROUP BY
    { $sort: { total: -1 } },                  // ORDER BY
    { $limit: 10 }                             // LIMIT
])
```

### When MongoDB Makes Sense (and When It Doesn't)

**Good fits:**
- Product catalogs (each product type has different attributes)
- Content management (articles with varying metadata)
- Event logging (each event has different fields)
- Rapid prototyping (schema evolution without migrations)

**Bad fits:**
- Financial transactions (ACID across documents is complex)
- Highly relational data (multi-level JOINs are awkward with `$lookup`)
- Reporting with complex aggregations (SQL is much more expressive)
- When you later realize you need referential integrity (no real foreign keys)

---

## 19. Database Backups & Recovery

Backups are the only thing standing between you and catastrophic data loss. In DevOps, if you didn't test the restore, you don't have a backup.

### Backup Types

**Full backup:** Complete copy of the database.

**Incremental backup:** Only changes since the last backup. Faster to take, slower to restore (must apply base + all incrementals).

**WAL archiving (continuous backup):** Continuously archive WAL files. Allows point-in-time recovery (PITR) — restore to any point in time, not just when backups were taken.

### PostgreSQL Backup

```bash
# pg_dump — logical backup of a single database
pg_dump -h localhost -U postgres mydb > mydb_backup.sql

# Compressed dump
pg_dump -h localhost -U postgres -Fc mydb > mydb_backup.dump

# Restore from SQL dump
psql -h localhost -U postgres mydb < mydb_backup.sql

# Restore from custom format
pg_restore -h localhost -U postgres -d mydb mydb_backup.dump

# pg_dumpall — dump all databases including roles and settings
pg_dumpall -h localhost -U postgres > all_databases.sql

# PITR with pgBaseBackup + WAL archiving
pg_basebackup -h primary -U replication -D /backup/base -Fp -Xs -P -R
```

### MySQL Backup

```bash
# mysqldump — logical backup
mysqldump -u root -p mydb > mydb_backup.sql

# All databases
mysqldump -u root -p --all-databases > all_databases.sql

# Compressed
mysqldump -u root -p mydb | gzip > mydb_backup.sql.gz

# Restore
mysql -u root -p mydb < mydb_backup.sql
zcat mydb_backup.sql.gz | mysql -u root -p mydb

# XtraBackup — hot physical backup (no locks)
xtrabackup --backup --target-dir=/backup/full
xtrabackup --prepare --target-dir=/backup/full
xtrabackup --copy-back --target-dir=/backup/full
```

### Backup Best Practices

```
3-2-1 Rule:
  3 copies of data
  2 different media types
  1 offsite (different geographic location)
```

**Test your backups regularly.** A backup that can't be restored is worthless. Schedule regular test restores to a staging environment.

**Automate:**
```bash
# Cron job for daily PostgreSQL backup
0 2 * * * pg_dump -U postgres mydb | gzip > /backup/mydb_$(date +%Y%m%d).sql.gz

# S3 upload after backup
0 3 * * * aws s3 cp /backup/mydb_$(date +%Y%m%d).sql.gz s3://my-backups/postgres/
```

**Monitor backup age and size:**
- Alert if no backup has run in 25 hours
- Alert if backup size is <50% of yesterday's (might indicate data loss)
- Alert if backup size is >200% of yesterday's (might indicate data corruption or dump issue)

**Retention policy:**
- Daily backups for 7 days
- Weekly backups for 4 weeks
- Monthly backups for 12 months
- Yearly backups for 7 years (regulatory compliance)

### RTO and RPO — Two Critical Metrics

**RPO (Recovery Point Objective):** Maximum acceptable data loss. If RPO = 1 hour, your backup frequency must be at most 1 hour.

**RTO (Recovery Time Objective):** Maximum acceptable downtime during recovery. If RTO = 4 hours, your restore process must complete within 4 hours.

These define your backup strategy:
- RPO = 0 (no data loss) → Synchronous replication required
- RPO = 1 hour → Hourly backups or WAL archiving
- RTO = 1 hour → Large databases might not restore in time → consider standby replica for faster failover

---

## 20. Database Performance & Monitoring

### The Key Metrics to Watch

**PostgreSQL:**
```sql
-- Queries per second (connections * avg query rate)
SELECT sum(xact_commit + xact_rollback) FROM pg_stat_database;

-- Cache hit rate (should be >99%)
SELECT round(sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) * 100, 2) AS cache_hit_pct
FROM pg_statio_user_tables;

-- Transactions per second
SELECT sum(xact_commit) + sum(xact_rollback) FROM pg_stat_database;

-- Top 10 slowest queries (requires pg_stat_statements extension)
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Enable pg_stat_statements
-- In postgresql.conf: shared_preload_libraries = 'pg_stat_statements'
-- Then: CREATE EXTENSION pg_stat_statements;
```

**Key PostgreSQL metrics to monitor:**

| Metric | Healthy | Warning |
|---|---|---|
| Cache hit rate | >99% | <95% |
| Active connections | <80% of max | >90% of max |
| Replication lag | <1 second | >30 seconds |
| Transaction rate | Stable | Sudden spikes/drops |
| Dead rows (bloat) | Low | >20% dead rows |
| Temp file creation | None/low | Any frequent temp files |
| Long-running transactions | None >1hr | Any >5 minutes |
| Lock wait time | Near zero | Any significant waits |

### EXPLAIN for Performance Debugging

```sql
-- Find the slow part of a query
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;

-- Look for:
-- 1. Seq Scan on large tables → add an index
-- 2. Estimated rows vs actual rows differ greatly → run ANALYZE to update statistics
-- 3. Sort nodes → add index on ORDER BY column, or increase work_mem
-- 4. Hash Batches > 1 → increase work_mem (hash spilling to disk)
-- 5. Nested Loop with large row counts → consider different join strategy
```

### Tools

```bash
# pgBadger — analyze PostgreSQL logs and produce HTML report
pgbadger /var/log/postgresql/postgresql.log -o report.html

# pg_activity — top-like tool for PostgreSQL
pg_activity -U postgres

# Percona Monitoring and Management (PMM) — full monitoring stack
# mysqltop / mytop — like top for MySQL
mytop -u root -p

# Monitoring via Prometheus exporters:
# postgres_exporter → Prometheus → Grafana
# mysqld_exporter → Prometheus → Grafana
```

---

## 21. Database Security

### Authentication & Access Control

```sql
-- PostgreSQL: Create application user with minimal privileges
CREATE USER appuser WITH PASSWORD 'strongpassword';
GRANT CONNECT ON DATABASE mydb TO appuser;
GRANT USAGE ON SCHEMA public TO appuser;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO appuser;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO appuser;

-- Read-only user for reporting/analytics
CREATE USER readonly WITH PASSWORD 'strongpassword';
GRANT CONNECT ON DATABASE mydb TO readonly;
GRANT USAGE ON SCHEMA public TO readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly;

-- MySQL: Create user with specific host restriction
CREATE USER 'appuser'@'10.0.0.%' IDENTIFIED BY 'strongpassword';
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'appuser'@'10.0.0.%';
FLUSH PRIVILEGES;
```

### Encryption

**Encryption in transit:** Always use TLS for database connections.
```bash
# PostgreSQL: Force SSL for all connections
# In pg_hba.conf:
hostssl all all 0.0.0.0/0 scram-sha-256

# Connect with SSL
psql "sslmode=require host=dbhost dbname=mydb user=appuser"
```

**Encryption at rest:** Use encrypted storage volumes (AWS EBS encryption, dm-crypt on Linux).

**Column-level encryption:** For extra-sensitive data (SSNs, credit cards), encrypt individual columns in the application before storing.

### Audit Logging

```sql
-- PostgreSQL: Enable audit logging with pgAudit extension
-- In postgresql.conf:
shared_preload_libraries = 'pgaudit'
pgaudit.log = 'write, ddl'   # Log all writes and schema changes
pgaudit.log_catalog = on

-- MySQL: Enable general query log (careful — high volume)
SET GLOBAL general_log = 'ON';
SET GLOBAL general_log_file = '/var/log/mysql/general.log';
```

---

## 22. Migrations — Changing Schemas Safely

A **database migration** is a controlled, versioned change to your database schema (adding columns, creating tables, modifying indexes, etc.).

### Why Migrations Are Hard

In production with live traffic:
- Adding a column is usually safe (defaults to NULL for existing rows)
- Adding a NOT NULL column without a default → FAILS on existing rows
- Renaming a column → breaks existing queries immediately
- Adding an index → locks the table on MySQL, can use CONCURRENTLY on PostgreSQL
- Dropping a column → breaks code still referencing it

### Safe Migration Patterns

**Expand-Contract (Blue-Green migrations):**

Adding a new column:
```sql
-- Step 1 (Expand): Add nullable column, backfill it
ALTER TABLE users ADD COLUMN phone VARCHAR(20);    -- Safe, no lock needed
UPDATE users SET phone = '' WHERE phone IS NULL;   -- Backfill in batches

-- Step 2: Add NOT NULL constraint after backfill complete
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
```

Renaming a column (no downtime):
```
Step 1: Add new_column, write to both old and new
Step 2: Backfill new_column from old_column
Step 3: Update all reads to use new_column
Step 4: Update all writes to write only new_column
Step 5: Drop old_column
```

**Large table migrations:**
```sql
-- Never run this on a large table in production (locks table for minutes/hours):
ALTER TABLE events ADD COLUMN processed BOOLEAN DEFAULT false;

-- Instead, add without default first:
ALTER TABLE events ADD COLUMN processed BOOLEAN;         -- Fast, no lock
UPDATE events SET processed = false WHERE processed IS NULL LIMIT 10000;  -- Batch update
-- Repeat the UPDATE in batches until complete
ALTER TABLE events ALTER COLUMN processed SET DEFAULT false;
ALTER TABLE events ALTER COLUMN processed SET NOT NULL;
```

### Migration Tools

**Flyway:**
```sql
-- V1__create_users_table.sql
CREATE TABLE users (...);

-- V2__add_phone_column.sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- V3__create_index_on_email.sql
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```

**Liquibase, Alembic (Python), ActiveRecord Migrations (Rails), Django Migrations** — all follow the same versioned migration concept.

**Best practices:**
- Every migration must be reversible (have a down migration)
- Never edit a migration that has already run in production — create a new one
- Test migrations on a production-sized copy of data before running in production
- Run migrations during low-traffic windows when possible
- For large tables, test with `EXPLAIN` first, then run in a transaction, verify, then commit

---

## 23. Quick Reference Cheat Sheet

### PostgreSQL Quick Reference
```bash
# Connect
psql -U postgres -d mydb
psql "postgres://user:pass@host:5432/dbname"

# Common psql commands
\l          # List databases
\dt         # List tables
\d table    # Describe table
\di         # List indexes
\timing     # Toggle timing

# Admin queries
SELECT pg_size_pretty(pg_database_size('mydb'));     # DB size
SELECT * FROM pg_stat_activity WHERE state != 'idle'; # Active queries
SELECT * FROM pg_locks WHERE NOT granted;             # Blocked locks

# Backup/restore
pg_dump -Fc mydb > mydb.dump
pg_restore -d mydb mydb.dump
```

### MySQL Quick Reference
```bash
# Connect
mysql -u root -p mydb

# Show
SHOW DATABASES; SHOW TABLES; DESCRIBE table;
SHOW PROCESSLIST; SHOW ENGINE INNODB STATUS\G

# Backup/restore
mysqldump -u root -p mydb > backup.sql
mysql -u root -p mydb < backup.sql
```

### Redis Quick Reference
```bash
redis-cli
SET key value EX 3600    # Set with 1hr expiry
GET key
DEL key
KEYS pattern             # Find keys (use SCAN in production)
TTL key                  # Time to live
INFO memory              # Memory usage
MONITOR                  # Watch live commands
```

### Index Cheat Sheet
```sql
-- Create
CREATE INDEX idx_name ON table(column);
CREATE INDEX CONCURRENTLY idx_name ON table(column);  -- Non-blocking
CREATE UNIQUE INDEX idx_name ON table(column);
CREATE INDEX idx_name ON table(col1, col2, col3);     -- Composite

-- Drop
DROP INDEX idx_name;
DROP INDEX CONCURRENTLY idx_name;  -- Non-blocking

-- Find missing indexes
SELECT relname, seq_scan FROM pg_stat_user_tables ORDER BY seq_scan DESC;

-- Find unused indexes
SELECT indexrelname, idx_scan FROM pg_stat_user_indexes WHERE idx_scan = 0;
```

### Transactions
```sql
BEGIN;
-- operations
COMMIT;   -- or ROLLBACK;

SAVEPOINT my_savepoint;
ROLLBACK TO SAVEPOINT my_savepoint;   -- Partial rollback
```

---

## Database Concepts Map

```
STORAGE
  Pages → Buffer Pool (RAM cache) → Disk
  WAL → Crash recovery + Replication
  Indexes (B-Tree) → Fast lookups

SQL LANGUAGE
  SELECT / INSERT / UPDATE / DELETE
  JOINs → Combine tables
  Transactions → ACID → Consistency
  EXPLAIN → Query planning → Performance

SCALING
  Single server → Vertical scaling
  Read replicas → Read scaling
  Partitioning → Large table management (single server)
  Sharding → Write scaling (multi-server)
  Connection pooling → Many clients, fewer DB connections

DATABASE TYPES
  Relational (PostgreSQL, MySQL) → Transactions, JOINs, schema
  Document (MongoDB) → Flexible schema, object storage
  Key-Value (Redis) → Cache, sessions, counters
  Column (Cassandra) → Massive write scale
  Graph (Neo4j) → Connected data

OPERATIONS
  Backups → pg_dump, mysqldump, WAL archiving
  Replication → Primary → Replica(s)
  Migrations → Versioned schema changes
  Monitoring → Cache hit rate, slow queries, replication lag, connections
```

---

*Document version 1.0 — Database Fundamentals for DevOps Engineers. Pair with OS, Networking, and Security Fundamentals for a complete foundation.*
