---
uid: 1bbe7fec-4137-41ea-8844-93be54d940ba
---

# SQL Fundamentals — Interview Guide

**Topic:** [[solution-arch/topics/data-architecture]]
**Related:** [[solution-arch/concepts/acid-vs-base]], [[solution-arch/concepts/database-sharding]], [[solution-arch/patterns/api-composition]], [[solution-arch/concepts/idempotency]]

## What it is

The baseline SQL fluency almost every technical interview tests somewhere — backend, data, SRE, or solution architecture — regardless of whether the role is "a SQL role." This page covers the specific areas most commonly probed: joins, `GROUP BY`/`WHERE`/`HAVING`, counting and duplicates, composite keys and many-to-many relationships, and the N+1 query problem, with worked examples and ASCII diagrams throughout.

## How it works

### Sample data used in every example below

```
customers:
id  name 
--  -----
1   Alice
2   Bob  
3   Carol

orders:
id   customer_id  amount
---  -----------  ------
101  1            50    
102  1            20    
103  2            75    
104  9            15      ← orphan: customer_id 9 doesn't exist in customers
```

Note deliberately: Carol (id 3) has no orders, and order 104 references a customer that doesn't exist. This asymmetry is what makes the join examples below actually show something — a clean 1:1 dataset would make every join type look identical.

---

### JOINs — which rows survive

The fastest way to reason about a join: think of it as **which of the three zones** (left-only, matched, right-only) survive in the output.

```
                      A only         A ∩ B (match)        B only      
---------------------------------------------------------------------
INNER JOIN              .           [ INCLUDED ]            .        
LEFT JOIN         [ INCLUDED ]      [ INCLUDED ]            .        
RIGHT JOIN               .          [ INCLUDED ]      [ INCLUDED ]   
FULL OUTER JOIN   [ INCLUDED ]      [ INCLUDED ]      [ INCLUDED ]
```

Now the same four joins against the actual sample data (`customers c JOIN orders o ON o.customer_id = c.id`), so you can see exactly what NULL-filling looks like in practice:

```
INNER JOIN — only rows with a match on BOTH sides:
  SELECT c.name, o.id, o.amount
  FROM customers c
  JOIN orders o ON o.customer_id = c.id;

c.name  o.id  o.amount
------  ----  --------
Alice   101   50      
Alice   102   20      
Bob     103   75      
  → Carol dropped (no orders). Order 104 dropped (no matching customer).
```

```
LEFT JOIN — every row from customers, matched orders if any exist:
  SELECT c.name, o.id, o.amount
  FROM customers c
  LEFT JOIN orders o ON o.customer_id = c.id;

c.name  o.id  o.amount
------  ----  --------
Alice   101   50      
Alice   102   20      
Bob     103   75      
Carol   NULL  NULL    
  → Carol KEPT, with NULLs for the unmatched order columns.
    Order 104 still dropped — it's not reachable from the LEFT side.
```

```
RIGHT JOIN — every row from orders, matched customers if any exist:
  SELECT c.name, o.id, o.amount
  FROM customers c
  RIGHT JOIN orders o ON o.customer_id = c.id;

c.name  o.id  o.amount
------  ----  --------
Alice   101   50      
Alice   102   20      
Bob     103   75      
NULL    104   15      
  → Order 104 KEPT, with NULL for the unmatched customer name.
    Carol dropped — she has no reachable row from the RIGHT side.
```

```
FULL OUTER JOIN — everything from both sides, matched where possible:
  SELECT c.name, o.id, o.amount
  FROM customers c
  FULL OUTER JOIN orders o ON o.customer_id = c.id;

c.name  o.id  o.amount
------  ----  --------
Alice   101   50      
Alice   102   20      
Bob     103   75      
Carol   NULL  NULL    
NULL    104   15      
  → Both unmatched rows survive, each with NULLs on the side that
    has no match. (MySQL has no native FULL OUTER JOIN — emulate
    with LEFT JOIN UNION RIGHT JOIN, or UNION ALL + a NOT EXISTS
    filter to avoid double-counting the matched rows.)
```

**The one-line mental model:** `INNER` = intersection only. `LEFT`/`RIGHT` = one full side + whatever matches on the other. `FULL OUTER` = union of both sides, NULL-padded wherever there's no match.

**Self-join** — a table joined to itself, typically to compare rows within the same table (e.g. "find each employee's manager, where manager_id is a foreign key back into the same employees table"):
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

---

### `WHERE` vs `HAVING` — the filter-BEFORE vs filter-AFTER aggregation distinction

This is the single most common `GROUP BY` interview trap. The key fact: SQL doesn't execute in the order it's WRITTEN — it executes in this order:

```
1. FROM       (and JOINs)
2. WHERE       ← filters INDIVIDUAL ROWS, before any grouping happens
3. GROUP BY    ← rows are collapsed into groups here
4. HAVING      ← filters GROUPS, after aggregation has already run
5. SELECT      ← column list and aliases are computed here
6. DISTINCT
7. ORDER BY
8. LIMIT
```

This ordering is WHY `WHERE` can't reference an aggregate (`WHERE COUNT(*) > 1` is invalid — `COUNT(*)` doesn't exist yet when `WHERE` runs) and why `HAVING` can (`HAVING COUNT(*) > 1` is valid — by the time `HAVING` runs, groups and their aggregates already exist).

```sql
-- WHERE filters rows first (only 'shipped' orders count toward the total),
-- HAVING filters the resulting groups (only customers whose total is high)
SELECT customer_id, SUM(amount) AS total
FROM orders
WHERE status = 'shipped'        -- row-level filter, BEFORE grouping
GROUP BY customer_id
HAVING SUM(amount) > 100;       -- group-level filter, AFTER aggregation
```

Using `WHERE` when you meant `HAVING` (or vice versa) doesn't error in every case — sometimes it just silently computes the wrong number, which is what makes this a favorite interview trap.

---

### Finding duplicates

The canonical pattern: `GROUP BY` the column(s) that define a "duplicate," then `HAVING COUNT(*) > 1`:

```sql
-- Find duplicate email addresses
SELECT email, COUNT(*) AS occurrences
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

To find (or delete) the ACTUAL duplicate ROWS, not just the duplicate values, window functions are the modern standard approach:

```sql
-- Tag each row with its rank within its own duplicate group
SELECT id, email,
       ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn
FROM users;

-- rn = 1 is the row you'd KEEP; rn > 1 are the duplicates
-- Delete all duplicates, keeping the lowest id per email:
DELETE FROM users
WHERE id IN (
  SELECT id FROM (
    SELECT id, ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS rn
    FROM users
  ) t
  WHERE rn > 1
);
```

`PARTITION BY` is doing the same conceptual job as `GROUP BY` here — splitting rows into buckets — but `ROW_NUMBER()` operates WITHIN each partition without collapsing the rows, so you keep row-level detail (like `id`) that a plain `GROUP BY` would lose.

---

### Counting — `COUNT(*)` vs `COUNT(column)` vs `COUNT(DISTINCT column)`

```
COUNT(*)              → counts ALL rows in the group, including rows
                         where every column is NULL
COUNT(column)          → counts only rows where THAT COLUMN is NOT NULL
                         (NULLs are silently excluded — a common
                         source of an "off by a few" bug)
COUNT(DISTINCT column) → counts only the UNIQUE non-NULL values
```

```sql
-- Orders per customer (row count per group)
SELECT customer_id, COUNT(*) AS order_count
FROM orders
GROUP BY customer_id;

-- Customers with at least one order (unique customers, not order rows)
SELECT COUNT(DISTINCT customer_id) AS customers_with_orders
FROM orders;
```

`COUNT(1)` behaves identically to `COUNT(*)` in every modern query optimizer — the historical belief that `COUNT(1)` is faster is a myth worth being able to debunk if it comes up.

---

### Composite keys & many-to-many relationships

A **composite primary key** is a primary key made of more than one column — used almost always on a **junction/bridge table** that resolves a many-to-many relationship into two many-to-one relationships:

```
                       ┌─────────────────────────┐                      
┌─────────────────┐    │     enrollments (N)     │    ┌────────────────┐
│  students (1)   │    ├─────────────────────────┤    │  courses (1)   │
├─────────────────┤    │ student_id (FK)         │    ├────────────────┤
│ student_id (PK) │    │ course_id (FK)          │    │ course_id (PK) │
│ name            │    │ enrolled_date           │    │ title          │
└─────────────────┘    │ PRIMARY KEY             │    └────────────────┘
                       │ (student_id, course_id) │                      
                       └─────────────────────────┘                      

                            (one student has           (one course has  
                            many enrollments)         many enrollments) 
```

Why the composite key matters, not just where it goes: `PRIMARY KEY (student_id, course_id)` enforces — at the DATABASE level, not just in application code — that the same student can't be enrolled in the same course twice, because the combination must be unique. A single-column surrogate `id` primary key on `enrollments` would NOT enforce that on its own; you'd need a separate `UNIQUE (student_id, course_id)` constraint to get the same guarantee.

```sql
-- The many-to-many query: which courses is Alice enrolled in?
SELECT c.title
FROM students s
JOIN enrollments e ON e.student_id = s.student_id
JOIN courses c ON c.course_id = e.course_id
WHERE s.name = 'Alice';
```

**Composite foreign key** (less common, worth recognizing): a foreign key made of multiple columns, referencing a composite primary key elsewhere — e.g. an `order_item_shipments` table referencing `(order_id, item_id)` together as a unit, because neither column alone identifies the row it points to.

---

### The N+1 Problem in SQL

The N+1 problem: fetch N parent rows with one query, then issue **one additional query per parent row** to fetch related child data — turning 1 intended operation into N+1 actual round trips. It's the most common ORM footgun (Hibernate, Django ORM, SQLAlchemy, ActiveRecord all default to lazy-loading relationships unless told otherwise).

```
BAD (N+1) — 1 query for orders, then 1 query PER order for its customer:

  SELECT * FROM orders;                      -- 1 query → 10 rows
  -- ORM lazy-loads order.customer for each row in a loop:
  SELECT * FROM customers WHERE id = 1;      ┐
  SELECT * FROM customers WHERE id = 2;      │  10 more queries —
  SELECT * FROM customers WHERE id = 1;      │  one per order row,
  ...                                        │  even for repeated
  SELECT * FROM customers WHERE id = 3;      ┘  customer_ids

  Total: 1 + 10 = 11 round trips for data a single query could return
```

```
GOOD — collapse to a single query with a JOIN:

  SELECT o.*, c.name
  FROM orders o
  JOIN customers c ON c.id = o.customer_id;

  Total: 1 query
```

```
GOOD (when a JOIN would duplicate/distort parent rows — e.g. one
order joined to MANY line items would multiply the order row N times):

  SELECT * FROM orders;                             -- 1 query
  SELECT * FROM customers WHERE id IN (1,2,3,...);   -- 1 batched query
                                                       (dedupe IDs first)
  Total: 2 queries, application code stitches them together in memory
         — the "batch loading" / DataLoader pattern
```

**How ORMs let you fix this explicitly:** eager loading — `JOIN FETCH` in JPQL/Hibernate, `.select_related()`/`.prefetch_related()` in Django ORM, `.includes()` in ActiveRecord, `joinedload()` in SQLAlchemy — tells the ORM to fetch the relationship up front in the SAME query (or one batched query) instead of lazily, one row at a time, as each parent object is accessed in a loop.

**This is the same pattern one layer up the stack** as the API-composition N+1 problem already covered in [[solution-arch/patterns/api-composition]] — there, the loop makes N HTTP calls instead of N SQL queries, but the root cause (looping and calling out per-item instead of batching) and the fix (batch/JOIN instead of loop) are identical. Worth naming that connection explicitly in an interview — it signals you see the pattern, not just the SQL-specific symptom.

## Complexity

Not algorithmic. The N+1 problem's real cost is round-trip latency, not per-query CPU cost — even a "fast" 2ms query becomes a real problem at N=1000, because 1000 sequential round trips (each paying network latency, not just query execution time) can easily add seconds to a request, even though the database itself is barely working.

## When to use

```
✅ Reach for JOIN as the default when you need related data and the
   relationship is 1:1 or many:1 (joining doesn't multiply/distort
   the parent rows)
✅ Reach for a batched WHERE ... IN (...) query, stitched together in
   application code, when a JOIN would multiply parent rows you
   don't want multiplied (e.g. one order with many line items, but
   you only want ONE row per order plus its customer)
✅ Always enable query logging in development and count queries per
   request — N+1 bugs are invisible in code review but immediately
   obvious in a query log showing 1 query become 50
```

## Common interview angles — basic SQL question bank

```
Q: What's the difference between WHERE and HAVING?
A: WHERE filters individual rows BEFORE grouping/aggregation runs;
   HAVING filters GROUPS after aggregation has already happened.
   WHERE can't reference an aggregate function; HAVING can.

Q: What's the difference between INNER JOIN and LEFT JOIN?
A: INNER JOIN keeps only rows with a match on both sides. LEFT JOIN
   keeps every row from the left table, NULL-filling columns from
   the right table when there's no match.

Q: What's the difference between UNION and UNION ALL?
A: UNION removes duplicate rows from the combined result (costs a
   sort/dedup pass); UNION ALL keeps every row from both queries,
   including duplicates, and is faster since it skips that pass.
   Default to UNION ALL unless you specifically need deduplication.

Q: What's the difference between DELETE, TRUNCATE, and DROP?
A: DELETE removes rows (optionally filtered by WHERE), is logged
   row-by-row, and can be rolled back in a transaction. TRUNCATE
   removes ALL rows at once, is minimally logged (faster), resets
   auto-increment counters, and in many databases can't be rolled
   back once committed. DROP removes the entire table structure
   itself, not just its rows.

Q: What's a primary key vs. a foreign key vs. a unique key?
A: Primary key uniquely identifies each row in ITS OWN table (one
   per table, can't be NULL). Foreign key references a primary
   (or unique) key in ANOTHER table, enforcing referential
   integrity. Unique key enforces uniqueness like a primary key but
   a table can have several, and (in most databases) unique keys
   CAN contain NULL.

Q: What's a composite key?
A: A primary (or unique) key made of more than one column, where
   the COMBINATION of values must be unique even if no single
   column is unique alone — the standard way to model a many-to-many
   junction table. See the students/enrollments/courses example above.

Q: How do you find duplicate rows in a table?
A: GROUP BY the column(s) that define a duplicate, then
   HAVING COUNT(*) > 1. To find/delete the actual duplicate ROWS
   (not just the values), use ROW_NUMBER() OVER (PARTITION BY ...)
   and filter/delete where the row number is greater than 1.

Q: How do you find the 2nd highest salary in a table? (a classic)
A: Several valid approaches:
     SELECT MAX(salary) FROM employees
     WHERE salary < (SELECT MAX(salary) FROM employees);
   or, more general (works for Nth highest with a parameter):
     SELECT DISTINCT salary FROM employees
     ORDER BY salary DESC
     LIMIT 1 OFFSET 1;
   or with a window function (handles ties cleanly via DENSE_RANK):
     SELECT salary FROM (
       SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
       FROM employees
     ) t WHERE rnk = 2;

Q: What is the N+1 query problem, and how do you avoid it?
A: See the dedicated section above — fetching N parent rows then
   issuing one additional query per row for related data. Fix with
   a JOIN (when it won't distort parent-row cardinality) or a
   batched WHERE ... IN (...) query, or by enabling eager loading
   in whatever ORM is in use instead of relying on lazy loading.

Q: What's the difference between COUNT(*), COUNT(1), and
   COUNT(column)?
A: COUNT(*) and COUNT(1) are equivalent — both count all rows in
   the group, including rows with NULLs. COUNT(column) counts only
   rows where that specific column is NOT NULL — a real semantic
   difference, not just a style choice, and a common source of a
   count being unexpectedly lower than the row count.

Q: What is a correlated subquery, and how is it different from a
   regular subquery?
A: A regular (uncorrelated) subquery runs once, independently, and
   its result is used by the outer query. A correlated subquery
   references a column from the OUTER query, so it conceptually
   re-runs once PER ROW of the outer query — e.g. finding each
   employee who earns more than their own department's average:
     SELECT name FROM employees e1
     WHERE salary > (
       SELECT AVG(salary) FROM employees e2
       WHERE e2.department_id = e1.department_id   -- correlation
     );
   Correlated subqueries are often rewritable as a JOIN or a window
   function for better performance — worth mentioning that a naive
   correlated subquery can be a real per-row performance trap at
   scale, the same shape of problem as N+1 but expressed as a
   single (slow) query instead of literally N queries.

Q: What are the first three normal forms (1NF/2NF/3NF), briefly?
A: 1NF: every column holds a single, atomic value (no comma-
   separated lists in a cell). 2NF: 1NF, plus every non-key column
   depends on the WHOLE primary key, not just part of a composite
   key. 3NF: 2NF, plus no non-key column depends on another
   non-key column (no transitive dependency). In practice: 3NF is
   the common target for transactional (OLTP) schemas; deliberately
   denormalizing is a valid, common trade-off for read-heavy
   analytical (OLAP) workloads.

Q: What's the difference between a clustered and a non-clustered
   index, at a level worth knowing for an interview?
A: A clustered index determines the PHYSICAL storage order of the
   table's rows — a table can have at most one (often the primary
   key). A non-clustered index is a separate structure that stores
   pointers back to the actual rows, and a table can have many.
   Looking up via a non-clustered index costs an extra hop (index →
   pointer → actual row) unless the index itself contains every
   column the query needs (a "covering index").
```

## Examples

```sql
-- Full worked example combining several of the above: total spend
-- per customer, only customers with more than one shipped order,
-- highest spenders first.
SELECT c.name,
       COUNT(o.id)      AS order_count,
       SUM(o.amount)    AS total_spent
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'shipped'          -- row filter, before grouping
GROUP BY c.id, c.name
HAVING COUNT(o.id) > 1              -- group filter, after aggregation
ORDER BY total_spent DESC;
```

## Sources
- [[solution-arch/patterns/api-composition]] — the N+1 problem's application-layer counterpart
- [[solution-arch/concepts/acid-vs-base]]
- [[solution-arch/concepts/database-sharding]]
