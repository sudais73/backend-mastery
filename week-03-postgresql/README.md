# Week 3: PostgreSQL Data Layer Mastery

## Overview

**Goal:** Understand PostgreSQL as a correctness engine, not just a data bucket. Master schema design, indexing, transactions, and query optimization.

**Why it matters:** The gap between a mid-level dev and a senior shows up here. Juniors treat the database as a dumb bucket. Seniors treat it as the most trustworthy part of the system.

---

## The Mental Model (Foundation)

### **Think Like a Filing Cabinet**

Imagine a physical filing cabinet:

```
┌─────────────────────────────────────────┐
│ TABLE (The Heap - actual data)          │
│ ┌─────────────────────────────────────┐ │
│ │ Row 1: Customer "Alice"             │ │
│ │ Row 2: Customer "Bob"               │ │
│ │ Row 3: Customer "Charlie"           │ │
│ │ ...rows in random order...          │ │
│ └─────────────────────────────────────┘ │
│                                         │
│ PROBLEM: Finding "Alice" requires      │
│ reading every row (slow!)              │
└─────────────────────────────────────────┘
```

**Add an Index:**

```
┌─────────────────────────────────────────┐
│ INDEX (Alphabetical dividers)           │
│                                         │
│ A → points to Row 1 (Alice)            │
│ B → points to Row 2 (Bob)              │
│ C → points to Row 3 (Charlie)          │
│                                         │
│ BENEFIT: Finding "Alice" is instant!   │
│ (Jump directly to Row 1)               │
└─────────────────────────────────────────┘
```

### **The Four Core Layers**

**1. TABLE (The Heap)**
- Where rows actually live
- Unordered (no guaranteed sequence)
- Direct access is slow for large tables
- **Example:** Millions of customer records scattered on disk

**2. INDEX**
- Sorted reference structure
- Says "value X is at row location Y"
- Lets database jump straight to matching rows
- **Example:** "Customer name 'Alice' is at row 5000"

**3. CONSTRAINT**
- Rules the database enforces
- Bad data can never enter
- Catches bugs in your code
- **Examples:** PRIMARY KEY, NOT NULL, UNIQUE, FOREIGN KEY, CHECK

**4. TRANSACTION**
- All-or-nothing envelope
- Everything commits together or rolls back together
- Other connections don't see half-finished work
- **Example:** Transfer $100 between accounts — both updates succeed or both fail

---

## Core Concept 1: Relational Modeling & Normalization (3NF)

### **The Problem: Duplicate Data**

❌ **BAD - Denormalized (Duplicated Data):**

```sql
CREATE TABLE orders (
  id          bigint PRIMARY KEY,
  customer_id bigint,
  customer_name text,      -- ← Duplicated across 1000s of rows!
  customer_email text,     -- ← Same customer = same email repeated
  status text,
  total numeric
);
```

**Problem:**
- Customer "Alice" has email in 1000 rows
- Update Alice's email? Must update 1000 rows
- Miss one? Database is now corrupt
- Wasted disk space

✅ **GOOD - Normalized (3NF - Each Fact Once):**

```sql
CREATE TABLE customers (
  id bigint PRIMARY KEY,
  name text NOT NULL,
  email text NOT NULL
);

CREATE TABLE orders (
  id bigint PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers(id),
  status text,
  total numeric
);
```

**Benefit:**
- Alice's email in ONE row
- Update once, everywhere reflects it
- No duplication, no data corruption risk

### **Third Normal Form (3NF) Rule**

> "Every non-key column depends on the key, the whole key, and nothing but the key."

**Translation:**
- Each fact lives in exactly one place
- Every column in a table describes the primary key, not something else
- If you can remove a column and the data still makes sense, it belongs in another table

### **When to Denormalize**

Only denormalize AFTER:
1. Schema is normalized (3NF)
2. You measure a slow read path
3. You prove denormalization helps
4. You document why

**Example:**
```sql
-- Normalized (slow for reads)
SELECT orders.id, customers.name, customers.email
FROM orders
JOIN customers ON orders.customer_id = customers.id;

-- Denormalized (fast reads, but harder to maintain)
ALTER TABLE orders ADD COLUMN customer_name text;
-- Now duplicate data, but fewer joins
```

---

## Core Concept 2: Data Types & Constraints

### **Pick the Tightest Type That Fits**

**IDs:**
```sql
id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY
-- NOT: id int (too small, only 2 billion max)
-- NOT: id serial (deprecated, use GENERATED ALWAYS AS IDENTITY)
```

**Strings:**
```sql
-- GOOD: text (no arbitrary limit)
email text NOT NULL

-- BAD: varchar(255) (arbitrary limit, Postgres treats same as text anyway)
email varchar(255) NOT NULL

-- GOOD: Use enum for fixed set of values
status status_enum NOT NULL  -- 'open', 'paid', 'shipped'
```

**Dates & Times:**
```sql
-- GOOD: Always use timestamptz (timezone-aware)
created_at timestamptz NOT NULL DEFAULT now()

-- BAD: timestamp (naive, no timezone, causes bugs!)
created_at timestamp NOT NULL
```

**Money:**
```sql
-- GOOD: numeric (exact decimal, no floating-point errors)
total numeric(12,2) NOT NULL  -- $999,999,999.99 with 2 decimals

-- BAD: float (inexact, causes $0.01 errors in financial systems!)
total float NOT NULL
```

**Booleans:**
```sql
is_active boolean NOT NULL DEFAULT true
```

**Semi-Structured Data:**
```sql
-- GOOD: jsonb (indexed, queryable)
metadata jsonb

-- BAD: text (just a string, can't query inside)
metadata text
```

### **Constraints: Rules the Database Enforces**

**NOT NULL:**
```sql
email text NOT NULL
-- Database rejects any row where email is NULL
```

**UNIQUE:**
```sql
email text UNIQUE
-- Database rejects duplicate emails
-- (only one "alice@example.com" allowed)
```

**CHECK:**
```sql
age int CHECK (age >= 18)
-- Database rejects age < 18
-- Business logic enforced at database level!
```

**PRIMARY KEY:**
```sql
id bigint PRIMARY KEY
-- Unique + Not Null + indexed by default
```

**FOREIGN KEY:**
```sql
customer_id bigint NOT NULL REFERENCES customers(id) ON DELETE CASCADE
-- Database rejects orders for non-existent customers
-- ON DELETE CASCADE: delete customer → delete their orders
```

### **Why Constraints Matter**

Your application code might be buggy:
```javascript
// Buggy code might do this:
db.query("INSERT INTO orders (customer_id, total) VALUES (999, -500)")
// Negative total, invalid customer!
```

But constraints catch it:
```
ERROR: new row violates check constraint "orders_total_check"
ERROR: insert or update on table "orders" violates foreign key constraint
```

**Result:** Bad data never enters the database, no matter how buggy your code is.

---

## Core Concept 3: Indexes (B-tree, Partial, Composite, Covering)

### **B-tree Index (Default)**

```sql
CREATE INDEX ON orders (customer_id);
```

**What it does:**
- Sorts customer_id values alphabetically
- Database jumps straight to matching rows
- Handles: `=`, `<`, `>`, `BETWEEN`, `ORDER BY`

**Without index (Sequential Scan):**
```
Read row 1, check customer_id
Read row 2, check customer_id
Read row 3, check customer_id
... (read all 1 million rows!)
```

**With index (Index Scan):**
```
Jump directly to customer_id = 42
Found! Read those 5 rows only.
```

**Speed difference:** 1 million rows = 1000x faster with index!

---

### **Composite Index (Multiple Columns)**

```sql
CREATE INDEX ON orders (customer_id, created_at);
```

**What it does:**
- Sorted first by customer_id, then by created_at
- Handles the **leftmost prefix rule**

**Works for:**
```sql
-- ✅ Filter on customer_id
SELECT * FROM orders WHERE customer_id = 42

-- ✅ Filter on customer_id AND created_at
SELECT * FROM orders WHERE customer_id = 42 AND created_at > '2024-01-01'

-- ✅ Filter on customer_id AND ORDER BY created_at
SELECT * FROM orders 
WHERE customer_id = 42 
ORDER BY created_at DESC
```

**DOESN'T work for:**
```sql
-- ❌ Filter only on created_at (without customer_id first)
SELECT * FROM orders WHERE created_at > '2024-01-01'
-- Violates leftmost prefix rule!
-- Need separate index: CREATE INDEX ON orders (created_at)
```

**Why column order matters:**
- `(customer_id, created_at)` = useful
- `(created_at, customer_id)` = different usefulness

Choose order based on your actual queries.

---

### **Partial Index (Conditional)**

```sql
CREATE INDEX ON orders (created_at) WHERE status = 'open';
```

**What it does:**
- Indexes ONLY rows where status = 'open'
- Stays small (maybe 10% of table size)
- Faster queries on open orders

**Use case:**
```sql
-- ✅ Very fast (uses partial index)
SELECT * FROM orders WHERE status = 'open' ORDER BY created_at

-- ❌ Doesn't use partial index (status is not 'open')
SELECT * FROM orders WHERE status = 'paid' ORDER BY created_at
```

**Benefit:** Indexes don't bloat with historical/closed data.

---

### **Covering Index (Index-Only Scan)**

```sql
CREATE INDEX ON orders (customer_id) INCLUDE (total);
```

**What it does:**
- Index stores customer_id for lookup
- Index ALSO stores total as "extra baggage"
- Database can answer queries from index alone, without touching heap

**Example:**
```sql
-- ✅ Index-only scan (super fast!)
SELECT customer_id, total 
FROM orders 
WHERE customer_id = 42
-- Postgres finds all data in index, never touches table!

-- ❌ Non-covering (must touch table)
SELECT customer_id, total, status
FROM orders
WHERE customer_id = 42
-- Index has customer_id + total, but needs status from table
```

---

### **When to Index**

✅ **Index these columns:**
- Columns in WHERE clauses (filters)
- Columns in JOINs (foreign keys)
- Columns in ORDER BY / GROUP BY

❌ **Don't index:**
- Every column (bloats indexes, slows writes)
- Low-cardinality columns (boolean, status with 3 values)
- Columns you never query

### **Index Cost**

Indexes speed reads but slow writes:
```sql
INSERT — must update indexes too
UPDATE — must update indexes too
DELETE — must update indexes too
SELECT — faster!
```

**Trade-off:** Choose wisely. 80/20 rule: index the hot path (queries that run 1000s of times), not every column.

---

## Core Concept 4: Transactions & ACID

### **ACID Guarantees**

**Atomic (All or Nothing):**
```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

**Guarantees:**
- Both updates succeed, or both fail
- Never one succeeds and the other fails
- No half-finished state

**Use case:** Transfer money — both accounts update or neither does.

---

**Consistent (Constraints Hold):**
```sql
-- Constraint: total >= 0
INSERT INTO orders (total) VALUES (-500);
-- ❌ Rejected! Constraint violated
```

**Guarantees:**
- Database rejects any write that violates constraints
- Data is always in a valid state

---

**Isolated (Concurrent Transactions Don't Interfere):**

Transaction A:
```sql
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- $1000
-- Meanwhile, Transaction B runs...
SELECT balance FROM accounts WHERE id = 1;  -- Still $1000!
COMMIT;
```

Transaction B (running simultaneously):
```sql
BEGIN;
UPDATE accounts SET balance = 1500 WHERE id = 1;
COMMIT;
```

**Guarantee:** Transaction A sees a consistent snapshot from when it started. B's updates don't show up mid-transaction.

---

**Durable (Survives Crashes):**
```sql
BEGIN;
UPDATE accounts SET balance = 900 WHERE id = 1;
COMMIT;  -- Writes to disk
-- Server crashes!
-- Data is still there (survived the crash)
```

**Guarantee:** Once you COMMIT, data survives any failure.

---

### **Isolation Levels (Trade-off: Correctness vs Concurrency)**

**READ COMMITTED (Default):**
```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- $1000
-- Another transaction updates: balance = 1500
SELECT balance FROM accounts WHERE id = 1;  -- Sees $1500!
COMMIT;
```

**Property:** Sees latest committed data (may change within transaction)
**Use:** Most applications, good balance of safety and concurrency

---

**REPEATABLE READ:**
```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- $1000
-- Another transaction updates: balance = 1500
SELECT balance FROM accounts WHERE id = 1;  -- Still $1000!
COMMIT;
```

**Property:** Sees frozen snapshot from start (doesn't see mid-transaction updates)
**Use:** Need consistent data throughout transaction (analytics, reports)

---

**SERIALIZABLE:**
```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- $1000
-- Another transaction tries: UPDATE balance = 1500
-- Postgres detects conflict!
-- Your transaction ABORTS with "serialization failure"
-- (Must retry the whole transaction)
COMMIT;
```

**Property:** Transactions behave as if they ran one at a time (safest)
**Cost:** Aborts conflicts, you must retry
**Use:** Critical operations (financial transfers, ticket booking)

---

### **When to Use Transactions**

**Always wrap multi-step writes:**

✅ **GOOD:**
```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- Both succeed or both fail
```

❌ **BAD:**
```sql
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- If server crashes between, money disappears!
```

---

## Core Concept 5: MVCC & Concurrency (Why Long Transactions Are Dangerous)

### **How PostgreSQL Handles Concurrency**

**Multi-Version Concurrency Control (MVCC):**

Instead of locking rows (which blocks readers), Postgres creates new versions:

```
Transaction A starts, reads row version 1
Transaction B starts, reads row version 1

Transaction B: UPDATE (creates version 2)
Transaction B: COMMIT

Transaction A: Still sees version 1 (consistent snapshot)
```

**Benefit:** Readers never block writers, writers never block readers.

---

### **The Cost: Dead Tuples**

When you UPDATE:
```sql
UPDATE orders SET status = 'shipped' WHERE id = 1;
```

PostgreSQL doesn't overwrite the old row. Instead:
```
Old version (marked dead): status = 'open'
New version (current):     status = 'shipped'
```

**Both versions stay in table!**

---

### **VACUUM Cleans Up**

PostgreSQL runs VACUUM (usually automatic) to clean dead versions:
```
Removes old versions (frees space)
Reclaims disk
Keeps table healthy
```

---

### **The Danger: Long-Running Transactions**

```sql
BEGIN;
SELECT COUNT(*) FROM orders;  -- Starts at 3:00 PM
-- ... do other stuff for 30 minutes ...
SELECT * FROM orders WHERE id = 42;  -- Still 3:30 PM
COMMIT;
```

**Problem:**
- This transaction holds open a snapshot from 3:00 PM
- VACUUM can't remove rows changed after 3:00 PM
- Dead tuples pile up for 30 minutes
- Table bloats, gets slow

**Danger:** Overnight batch jobs that run for hours bloat the table.

**Solution:** Keep transactions short.

---

## Core Concept 6: EXPLAIN & Query Optimization

### **Read a Query Plan**

```sql
EXPLAIN ANALYZE 
SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC LIMIT 20;
```

**Output:**
```
Limit  (cost=0.29..1.50 rows=20 width=100)
  ->  Index Scan Backward using orders_customer_id_created_at on orders
        Index Cond: (customer_id = 42)
        (actual time=0.025..0.150 rows=20 loops=1)
```

**What this means:**
- **Limit:** Show first 20 rows
- **Index Scan Backward:** Using index in reverse order (efficiently!)
- **Index Cond:** Using index for customer_id = 42 filter
- **actual time=0.025ms:** Query ran in 0.025 milliseconds (fast!)

---

### **Red Flags: Seq Scan (Sequential Scan)**

```sql
EXPLAIN ANALYZE 
SELECT * FROM orders WHERE created_at > '2024-01-01';
```

**Output:**
```
Seq Scan on orders  (cost=0.00..50000.00 rows=500000 width=100)
  Filter: (created_at > '2024-01-01')
  (actual time=1234.567..5678.901 rows=500000 loops=1)
```

**Translation:** "Read every single row (500,000 rows), took 5+ seconds"

**Problem:** No index on created_at, so must read all rows.

**Solution:**
```sql
CREATE INDEX ON orders (created_at);
```

Re-run EXPLAIN ANALYZE:
```
Index Scan using orders_created_at on orders
  (actual time=0.025..12.345 rows=500000 loops=1)
```

**Result:** 5+ seconds → 0.025ms (200x faster!)

---

### **Other Red Flags**

**1. Big gap between estimated and actual rows:**
```
(cost=0.29..1.50 rows=20 width=100)
  (actual time=0.025..0.150 rows=5000)
-- Estimated 20 rows, got 5000!
-- Planner statistics are stale
-- Solution: ANALYZE orders;
```

**2. Expensive Sort:**
```
Sort  (cost=50000.00..60000.00 rows=500000)
```

**Solution:** Add index matching sort order
```sql
CREATE INDEX ON orders (customer_id, created_at DESC);
```

**3. Nested Loop:**
```
Nested Loop  (cost=1.00..10000.00)
  ->  Seq Scan on orders
  ->  Index Scan on order_items
```

**Translation:** "For each order (500,000), scan order_items" = slow!

**Solution:** Use JOIN instead of nested loop, add indexes

---

### **How to Read EXPLAIN ANALYZE**

**Rule: Read inside-out (deepest node runs first)**

```
Limit
  ->  Sort
    ->  Seq Scan
```

**Execution order:**
1. Seq Scan (read all rows)
2. Sort (sort them)
3. Limit (take first 20)

**Optimization:** Add index to avoid Sort, use LIMIT with index

---

## Core Concept 7: Connection Pooling (PgBouncer)

### **Why Connection Pooling Matters**

**Without pooling:**
```
Each HTTP request → new database connection
1000 requests/second → 1000 connections!
```

**Problem:**
- PostgreSQL processes each connection in a separate OS process
- 1000 processes = 10GB+ RAM
- Server exhausted, new connections rejected
- App crashes

**With PgBouncer (connection pool):**
```
HTTP requests → connection pool (10 connections reused)
1000 requests/second → rotate through 10 connections
```

**Benefit:**
- Server uses 10 connections, not 1000
- Uses 100MB RAM, not 10GB
- Handles 1000s of requests/second smoothly

---

### **How to Use PgBouncer**

**Setup:**
```bash
# Install
sudo apt install pgbouncer

# Configure /etc/pgbouncer/pgbouncer.ini
[databases]
myapp = host=localhost port=5432 dbname=myapp

[pgbouncer]
pool_mode = transaction  # Reuse connection per transaction
max_client_conn = 1000
default_pool_size = 10
```

**In production:**
```javascript
// Connect to PgBouncer (port 6432), not PostgreSQL (5432)
const connection = new Client({
  host: 'localhost',
  port: 6432,  // PgBouncer, not 5432!
  database: 'myapp'
});
```

**Result:** App scales to 1000s of concurrent requests

---

### **Modes**

**Session mode:**
- Connection reused per session (login → logout)
- Slower handoff, safer (full PostgreSQL state preserved)

**Transaction mode (recommended):**
- Connection reused per transaction
- Faster, works for most apps
- Each transaction gets fresh connection state

---

## Core Concept 8: N+1 Queries & Needless SELECT *

### **The N+1 Problem**

❌ **BAD - N+1 Loop:**

```javascript
// Get all customers (1 query)
const customers = await db.query("SELECT id, name FROM customers");

// For each customer, get their orders (N more queries!)
for (const customer of customers) {
  const orders = await db.query(
    "SELECT * FROM orders WHERE customer_id = $1",
    [customer.id]
  );
  customer.orders = orders;
}

// Total: 1 + N queries (if 1000 customers, 1001 queries!)
```

**Problem:** With 1000 customers, 1001 database hits!

✅ **GOOD - Single Join:**

```javascript
// Get customers with orders (1 query, 1 join)
const results = await db.query(`
  SELECT 
    customers.id, 
    customers.name,
    json_agg(orders.*) as orders
  FROM customers
  LEFT JOIN orders ON orders.customer_id = customers.id
  GROUP BY customers.id
`);

// Total: 1 query!
```

**Speed:** 1001 hits → 1 hit (1000x faster!)

---

### **Needless SELECT ***

❌ **BAD:**
```javascript
// Fetch ALL columns
const orders = await db.query("SELECT * FROM orders WHERE customer_id = 42");

// Use only 2 columns
orders.forEach(order => console.log(order.id, order.status));
```

**Problems:**
- Wasted bandwidth (fetched 50 columns, used 2)
- Defeats covering indexes (index has id + status, SELECT * needs table)
- Breaks silently when schema changes (new column added → code breaks)

✅ **GOOD:**
```javascript
// Fetch only needed columns
const orders = await db.query(
  "SELECT id, status FROM orders WHERE customer_id = $1",
  [42]
);
```

**Benefits:**
- Faster (less data over wire)
- Covering indexes work
- Schema changes don't break code

---

## Build This: Practice Project

### **Step 1: Design a Normalized Schema**

Model: **E-commerce System** (products, customers, orders, order_items)

```sql
-- Customers table
CREATE TABLE customers (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email text NOT NULL UNIQUE,
  name text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);

-- Products table
CREATE TABLE products (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  name text NOT NULL,
  price numeric(12,2) NOT NULL CHECK (price >= 0),
  stock int NOT NULL CHECK (stock >= 0),
  created_at timestamptz NOT NULL DEFAULT now()
);

-- Orders table
CREATE TABLE orders (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  status text NOT NULL CHECK (status IN ('open', 'paid', 'shipped')),
  total numeric(12,2) NOT NULL CHECK (total >= 0),
  created_at timestamptz NOT NULL DEFAULT now()
);

-- Order items table (junction table for order-product relationship)
CREATE TABLE order_items (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  order_id bigint NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
  product_id bigint NOT NULL REFERENCES products(id),
  quantity int NOT NULL CHECK (quantity > 0),
  unit_price numeric(12,2) NOT NULL CHECK (unit_price >= 0)
);
```

**Why this is normalized (3NF):**
- Customer data in one place (customers table)
- Product data in one place (products table)
- Order data separate from items
- No duplicated info

---

### **Step 2: Seed with Volume**

```sql
-- Insert 10,000 customers
INSERT INTO customers (email, name)
SELECT 
  'customer_' || i || '@example.com',
  'Customer ' || i
FROM generate_series(1, 10000) AS i;

-- Insert 100 products
INSERT INTO products (name, price, stock)
SELECT 
  'Product ' || i,
  (random() * 1000)::numeric(12,2),
  (random() * 1000)::int
FROM generate_series(1, 100) AS i;

-- Insert 100,000 orders
INSERT INTO orders (customer_id, status, total)
SELECT 
  (random() * 9999 + 1)::int,
  CASE (random() * 3)::int
    WHEN 0 THEN 'open'
    WHEN 1 THEN 'paid'
    ELSE 'shipped'
  END,
  (random() * 10000)::numeric(12,2)
FROM generate_series(1, 100000) AS i;

-- Insert 500,000 order items
INSERT INTO order_items (order_id, product_id, quantity, unit_price)
SELECT 
  (random() * 99999 + 1)::int,
  (random() * 99 + 1)::int,
  (random() * 10 + 1)::int,
  (random() * 1000)::numeric(12,2)
FROM generate_series(1, 500000) AS i;
```

---

### **Step 3: Find Three Slow Queries**

```sql
-- Query 1: Filter by customer
EXPLAIN ANALYZE 
SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC LIMIT 20;

-- Query 2: Join products with orders
EXPLAIN ANALYZE
SELECT o.id, o.status, p.name, oi.quantity
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
WHERE o.customer_id = 42;

-- Query 3: Aggregate with sorting
EXPLAIN ANALYZE
SELECT customer_id, COUNT(*) as order_count, SUM(total) as total_spent
FROM orders
WHERE created_at > now() - interval '30 days'
GROUP BY customer_id
ORDER BY total_spent DESC
LIMIT 20;
```

**Note the timings and Seq Scan nodes.**

---

### **Step 4: Make Them Fast with Indexes**

```sql
-- Index for Query 1
CREATE INDEX idx_orders_customer_created 
ON orders (customer_id, created_at DESC);

-- Index for Query 2
CREATE INDEX idx_order_items_order 
ON order_items (order_id);

-- Index for Query 3
CREATE INDEX idx_orders_created 
ON orders (created_at) WHERE created_at > now() - interval '30 days';
```

**Re-run EXPLAIN ANALYZE:**

Before indexes:
```
Seq Scan on orders (cost=0.00..50000.00 rows=100000)
(actual time=1234.567..5678.901 rows=100000 loops=1)
```

After indexes:
```
Index Scan using idx_orders_customer_created on orders
(actual time=0.025..12.345 rows=20 loops=1)
```

**Improvement:** 5+ seconds → 0.025ms (200x faster!)

---

## Mastery Checklist (0/8 → 8/8)

### **1. ✅ I can design a normalized (3NF) schema and know when to denormalize deliberately**

**What this means:**
- Design tables so each fact exists exactly once
- No duplicated data across rows
- Know when to break 3NF for performance (measure first!)

**How to practice:**
- Design schema for: Blog (posts, comments, authors)
- Normalize to 3NF
- Identify which denormalizations would help reads

---

### **2. ✅ I choose tight data types and enforce rules with NOT NULL, CHECK, UNIQUE, and FOREIGN KEY**

**What this means:**
- Use `bigint` for IDs, `text` for strings, `timestamptz` for dates
- Use `numeric` for money (never float)
- Let constraints enforce business rules

**How to practice:**
- Create table with 10 columns
- Pick optimal type for each
- Add 5 constraints (NOT NULL, CHECK, UNIQUE, FK)
- Try to insert bad data — watch it get rejected

---

### **3. ✅ I can explain B-tree, partial, composite, and covering indexes and when each helps**

**What this means:**
- Know why each index type exists
- Know which columns to index
- Know that column order matters in composite indexes

**How to practice:**
- Write 4 queries
- Pick best index type for each
- Use EXPLAIN ANALYZE to verify

---

### **4. ✅ I understand ACID and can pick the right isolation level for a workload**

**What this means:**
- Know ACID guarantees protect you
- Know READ COMMITTED vs REPEATABLE READ vs SERIALIZABLE
- Know when to use transactions

**How to practice:**
- Write multi-step transaction (transfer money)
- Wrap in BEGIN/COMMIT
- Understand what goes wrong without it

---

### **5. ✅ I can explain MVCC and why long transactions and dead tuples cause bloat**

**What this means:**
- MVCC = multiple versions, readers/writers don't block each other
- Long transactions prevent cleanup, cause bloat
- VACUUM cleans up

**How to practice:**
- Run a 30-minute transaction
- Check table size before/after
- See bloat happen
- Run VACUUM, see it clean up

---

### **6. ✅ I read EXPLAIN ANALYZE output and turn a Seq Scan into an Index Scan**

**What this means:**
- Understand query plans
- Spot Seq Scan (bad) vs Index Scan (good)
- Add indexes to flip the plan

**How to practice:**
- Run EXPLAIN ANALYZE on a slow query
- Predict what index would help
- Add it, re-run, verify speedup

---

### **7. ✅ I know why connection pooling matters and use PgBouncer in production**

**What this means:**
- Without pooling, 1000 connections = server exhaustion
- PgBouncer reuses 10-20 connections
- Always use in production

**How to practice:**
- Set up PgBouncer locally
- Connect through it instead of direct PostgreSQL
- See it reuse connections in logs

---

### **8. ✅ I can spot and fix N+1 queries, missing indexes, and needless SELECT ***

**What this means:**
- Recognize N+1 loop (1 query + N per row)
- Fix with JOIN or WHERE id = ANY(...)
- Use SELECT only_needed_columns, not SELECT *

**How to practice:**
- Write N+1 code
- Run EXPLAIN ANALYZE, see 1001 queries
- Fix to join, re-run, see 1 query
- Measure speedup

---

## Cheat Sheet (Quick Reference)

### **Schema & Constraints**

```sql
-- Create table with constraints
CREATE TABLE orders (
  id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
  status text NOT NULL CHECK (status IN ('open', 'paid', 'shipped')),
  total numeric(12,2) NOT NULL CHECK (total >= 0),
  created_at timestamptz NOT NULL DEFAULT now()
);
```

---

### **Indexes**

```sql
-- B-tree (default)
CREATE INDEX ON orders (customer_id);

-- Composite (leftmost prefix)
CREATE INDEX ON orders (customer_id, created_at DESC);

-- Partial (conditional)
CREATE INDEX ON orders (created_at) WHERE status = 'open';

-- Covering (index-only scan)
CREATE INDEX ON orders (customer_id) INCLUDE (total);
```

---

### **Transactions**

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- or ROLLBACK;
```

---

### **Read Query Plans**

```sql
-- See the plan
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- See the plan WITH actual timings
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;

-- Refresh statistics (if plan looks wrong)
ANALYZE orders;
```

---

### **Connection Pooling**

```bash
# Install PgBouncer
sudo apt install pgbouncer

# Configure (port 6432 listens, forwards to 5432)
# [pgbouncer]
# max_client_conn = 1000
# default_pool_size = 10
```

```javascript
// Connect through PgBouncer
const client = new Client({
  host: 'localhost',
  port: 6432,  // PgBouncer!
  database: 'myapp'
});
```

---

### **Avoid N+1**

```javascript
// ❌ BAD: N+1
const customers = await db.query("SELECT * FROM customers");
for (const c of customers) {
  c.orders = await db.query("SELECT * FROM orders WHERE customer_id = ?", [c.id]);
}

// ✅ GOOD: Single join
const result = await db.query(`
  SELECT c.*, array_agg(o.*) as orders
  FROM customers c
  LEFT JOIN orders o ON o.customer_id = c.id
  GROUP BY c.id
`);
```

---

## Interview Signals & Pitfalls

### **Signal of Mastery**

You can:
- ✅ Look at a slow query and predict what EXPLAIN ANALYZE shows
- ✅ Name the exact index that will flip Seq Scan → Index Scan
- ✅ Explain why composite index column order matters
- ✅ Spot N+1 in code instantly
- ✅ Design a normalized schema in your head
- ✅ Explain MVCC and why long transactions are dangerous

### **Common Pitfalls (Don't Do These)**

**Pitfall 1: The N+1 Query**

❌ **Loop and query per row:**
```javascript
for (const customer of customers) {
  const orders = await db.query(...);  // 1000 queries!
}
```

✅ **Fix: Join**
```sql
SELECT * FROM customers
JOIN orders ON ...
-- 1 query
```

---

**Pitfall 2: Missing Indexes on Foreign Keys**

❌ **No index:**
```sql
-- Every lookup is a table scan
SELECT * FROM orders WHERE customer_id = 42
```

✅ **Fix: Add index**
```sql
CREATE INDEX ON orders (customer_id);
```

---

**Pitfall 3: SELECT * Everywhere**

❌ **Fetch all columns:**
```sql
SELECT * FROM orders
-- Defeats covering indexes, breaks on schema changes
```

✅ **Fix: Select needed columns**
```sql
SELECT id, status FROM orders
```

---

**Pitfall 4: Long-Running Connections**

❌ **30-minute transaction:**
```sql
BEGIN;
  ... do stuff for 30 minutes ...
COMMIT;
-- Table bloats, VACUUM can't clean up
```

✅ **Fix: Keep transactions short**
```sql
BEGIN;
  ... do stuff quickly ...
COMMIT;
-- VACUUM can clean up immediately
```

---

**Pitfall 5: Running Without Connection Pooling in Production**

❌ **Direct connection per request:**
```
1000 requests → 1000 connections → server exhausted
```

✅ **Fix: Use PgBouncer**
```
1000 requests → 10 pooled connections → scales smoothly
```

---

## Real-World Example: E-commerce Query Optimization

### **Scenario: "Show customer's recent orders with products"**

### **Initial (Slow) Version:**

```javascript
// N+1 query!
const customer = await db.query("SELECT * FROM customers WHERE id = $1", [42]);

const orders = await db.query(
  "SELECT * FROM orders WHERE customer_id = $1 ORDER BY created_at DESC LIMIT 10",
  [42]
);

for (const order of orders) {
  const items = await db.query(
    "SELECT * FROM order_items WHERE order_id = $1",
    [order.id]
  );
  
  for (const item of items) {
    const product = await db.query(
      "SELECT * FROM products WHERE id = $1",
      [item.product_id]
    );
    item.product = product;
  }
  
  order.items = items;
}

customer.orders = orders;
```

**Problems:**
- 1 query for customer
- 10 queries for orders
- 50+ queries for items (10 orders × ~5 items each)
- 500+ queries for products
- **Total: 600+ database hits!**

### **Optimized (Fast) Version:**

```sql
SELECT 
  c.id as customer_id,
  c.email,
  c.name,
  json_agg(json_build_object(
    'id', o.id,
    'status', o.status,
    'total', o.total,
    'items', (
      SELECT json_agg(json_build_object(
        'id', oi.id,
        'quantity', oi.quantity,
        'product', json_build_object(
          'id', p.id,
          'name', p.name,
          'price', p.price
        )
      ))
      FROM order_items oi
      JOIN products p ON p.id = oi.product_id
      WHERE oi.order_id = o.id
    )
  )) as orders
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id AND o.created_at > now() - interval '90 days'
LEFT JOIN order_items oi ON oi.order_id = o.id
WHERE c.id = 42
GROUP BY c.id
```

**Result:**
- 1 single query
- Returns customer + orders + items + products in one hit
- 600 hits → 1 hit (600x faster!)

### **Add Supporting Indexes:**

```sql
CREATE INDEX ON orders (customer_id, created_at DESC);
CREATE INDEX ON order_items (order_id, product_id);
```

---

## What's Next: Week 4

Now that you understand PostgreSQL at depth:
- APIs (REST, GraphQL)
- Authentication (JWT, OAuth)
- Scaling (caching, replication, partitioning)

You're building the **data layer** — the most critical part of any backend system.

---

## Summary

**PostgreSQL is not a dumb bucket:**
- ✅ It's a correctness engine (constraints enforce business rules)
- ✅ It's a query optimizer (indexes make reads 100x faster)
- ✅ It's a concurrency manager (MVCC, transactions, isolation)

**Master these 8 items and you're a database expert:**
1. Normalize to 3NF
2. Pick tight types + add constraints
3. Index wisely (B-tree, partial, composite, covering)
4. Understand ACID + isolation levels
5. Explain MVCC + bloat
6. Read EXPLAIN ANALYZE
7. Use connection pooling (PgBouncer)
8. Eliminate N+1 + SELECT *

**Senior engineers don't just store data — they architect systems that can't fail.**
