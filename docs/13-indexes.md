# Part 13: Indexes in MySQL

An **index** is a separate data structure that MySQL keeps *alongside* your table to make **looking up rows faster** — the same way the index at the back of a book lets you jump straight to a page instead of reading every page.

Without an index, a query like `WHERE email = 'x'` forces MySQL to scan **every row** in the table (a *full table scan*). With an index on `email`, MySQL walks a sorted B-Tree structure and lands on the matching rows in a handful of steps.

> 💡 **The trade-off in one line:** an index makes **reads faster** but makes **writes slower** (every `INSERT`/`UPDATE`/`DELETE` must also update the index) and **costs extra storage**. You index for the queries you actually run, not "just in case".

![Indexing in MySQL](../assets/Indexing.png)

---

## How an Index Works (the mental model)

MySQL's default index is a **B-Tree** — a balanced, sorted tree. Because the values are kept **sorted**, MySQL can:

- **Find** a value with a binary-search-style descent (a few hops instead of N rows).
- **Range scan** efficiently (`BETWEEN`, `<`, `>`, `LIKE 'abc%'`) by walking the sorted leaves.
- **Avoid sorting** for `ORDER BY` when the order matches the index.

| Without an index | With an index |
| --- | --- |
| Full table scan — reads every row | Targeted lookup — reads only what matches |
| Cost grows linearly with table size | Cost grows *logarithmically* |
| Fine for tiny tables | Essential for large tables |

---

## When to Use an Index

Add an index when a column is **frequently used to find or order rows**:

1. **`WHERE` filters** — columns you search on constantly (`email`, `status`, `user_id`).
2. **`JOIN` keys** — foreign-key columns used to join tables.
3. **`ORDER BY` / `GROUP BY`** — columns you sort or group on often.
4. **Uniqueness rules** — enforce "no duplicates" with a `UNIQUE` index.

### When *not* to index

- Columns that are **rarely queried**.
- Columns with **very few distinct values** (e.g. a `gender` or boolean flag) — the index barely narrows anything.
- **Write-heavy** tables where the index-maintenance cost outweighs the read gain.
- **Small tables** — a full scan is already cheap.

---

## Types of Index Covered Here

| Type | Purpose |
| --- | --- |
| **Single-column** | Speeds up lookups on one column. |
| **Composite** | One index across **multiple columns** (order matters). |
| **Unique** | Speeds up lookups **and** forbids duplicate values. |
| **Prefix** | Indexes only the **first N characters** of a long text column. |

---

## Practice Questions

### Q1. Create a Single-Column Index

```sql
CREATE INDEX idx_employee_email
ON employees(email);
```

**🧠 Explanation**

- Creates a B-Tree index on the single column `email`.
- Any query that filters or sorts by `email` (`WHERE email = ...`, `ORDER BY email`) can now use it instead of scanning the whole table.
- Naming convention: prefixing with `idx_` makes indexes easy to spot when you list them.

---

### Q2. Create a Composite Index

```sql
CREATE INDEX idx_employee_dept
ON employees(employee_name, department);
```

**🧠 Explanation**

- A **composite** index covers **more than one column** in a single structure.
- **Column order matters.** This index helps queries that filter on `employee_name` **alone**, or `employee_name` **and** `department` together — because it's sorted by `employee_name` *first*, then `department`.
- It does **not** efficiently serve a query that filters on `department` alone (the second column). This is the **left-prefix rule**: an index on `(a, b)` works for `a` and `(a, b)`, but not `b` by itself.

> 💡 **Rule of thumb:** put the **most selective** / most-often-filtered column **first** in a composite index.

---

### Q3. Create a UNIQUE Index

```sql
CREATE UNIQUE INDEX idx_project_unique
ON projects(project_name);
```

A unique index does two jobs at once: it **speeds up lookups** *and* **guarantees no two rows share the same value**.

```sql
SHOW INDEXES FROM projects;

INSERT INTO projects(project_name) VALUES ('CRM System');
```

**🧠 Explanation**

- The first `INSERT` succeeds. Run it **a second time** and MySQL rejects it with a **duplicate-key error** — that's the uniqueness constraint doing its job.
- Use a unique index to enforce natural business rules: no two users with the same email, no two projects with the same name, etc.

---

### Q4. Create a Prefix Index

```sql
CREATE INDEX idx_email_prefix
ON employees(email(5));

SHOW INDEXES FROM employees;
```

A **prefix index** indexes only the **first N characters** of a column — here, the first 5 characters of `email`.

**When to use it**

- **Large text columns** (`VARCHAR(1000)`, `TEXT`) where a full-column index would be huge.
- **Saves storage** — the index only stores a slice of each value.
- **Faster index creation and updates** — less data to maintain.

**🧠 Explanation**

- `email(5)` tells MySQL: *"only index the first 5 characters."*
- The trade-off is **selectivity**. If many emails share the same first 5 characters, the prefix can't distinguish them and MySQL still has to check the actual rows. Pick a prefix length long enough to keep values mostly distinct.
- Example: if the emails `kunal.shah@...`, `kunal.jain@...`, `kunal.roy@...`, `kunal.dev@...` all start with `kunal`, a 5-char prefix collapses those **4 emails** into one bucket — so `WHERE email = 'kunal.shah@company.com'` still has to examine each `kunal*` row.

```sql
EXPLAIN ANALYZE
SELECT *
FROM employees
WHERE email = 'kunal.shah@company.com';
```

Run this to *see* how many rows the prefix index actually narrows the search down to.

---

### Q5. Remove an Index

```sql
DROP INDEX idx_employee_email
ON employees;
```

**🧠 Explanation**

- Drops the index by name from the specified table.
- Drop indexes that aren't being used — they cost storage and slow every write for no read benefit.

---

### Q6. Check Existing Indexes

```sql
SHOW INDEXES FROM employees;
```

**🧠 Explanation**

- Lists every index on the table, including the auto-created index behind the **primary key**.
- Useful columns in the output: `Key_name` (index name), `Column_name`, `Non_unique` (`0` = unique), and `Seq_in_index` (position of a column within a composite index).

---

### Q7. Check Query Performance with `EXPLAIN`

`EXPLAIN` shows how MySQL **plans** to execute a query — **without actually running it**.

```sql
EXPLAIN
SELECT *
FROM employees
WHERE employee_name = 'Neha Joshi' AND department = 'Marketing';
```

**🧠 Explanation**

- Reveals whether MySQL will use an **index** or fall back to a **full table scan**.
- Key columns to read: `type` (`ALL` = full scan ⚠️, `ref`/`range` = using an index ✅), `key` (which index it picked), and `rows` (estimated rows examined).
- This query is exactly what `idx_employee_dept` from Q2 was built for — `EXPLAIN` lets you confirm the optimizer actually uses it.

---

### Q8. Measure Real Performance with `EXPLAIN ANALYZE`

`EXPLAIN ANALYZE` **actually executes** the query and reports what *really* happened:

- the execution plan,
- **actual** execution time,
- **actual** rows processed,
- timing for **each step**.

```sql
EXPLAIN ANALYZE
SELECT *
FROM employees
WHERE employee_name = 'Neha Joshi' AND department = 'Marketing';
```

### `EXPLAIN` vs `EXPLAIN ANALYZE`

| `EXPLAIN` | `EXPLAIN ANALYZE` |
| --- | --- |
| Doesn't execute the query | Executes the query |
| Estimated execution plan | Actual execution plan |
| Estimated rows | Actual rows |
| Estimated cost | Actual execution time |

> 💡 **Use `EXPLAIN` to plan, `EXPLAIN ANALYZE` to prove.** `EXPLAIN` is safe and instant (nothing runs), so reach for it first. When estimates and reality disagree, `EXPLAIN ANALYZE` shows you the truth — at the cost of actually running the query.

---

## Key Points

- ✔ An index is a **sorted side structure** that turns full table scans into targeted lookups.
- ✔ Indexes make **reads faster** but **writes slower** and use extra **storage** — index deliberately.
- ✔ **Composite** indexes follow the **left-prefix rule**: `(a, b)` serves `a` and `(a, b)`, not `b` alone.
- ✔ **Unique** indexes speed up lookups *and* forbid duplicate values.
- ✔ **Prefix** indexes trade selectivity for smaller size on long text columns.
- ✔ Use **`EXPLAIN`** to see the *planned* execution, **`EXPLAIN ANALYZE`** to see the *actual* one.
- ✔ Don't index low-cardinality columns, rarely-queried columns, or tiny/write-heavy tables.

---

## Cheatsheet

```sql
-- Single-column index
CREATE INDEX idx_name ON table_name(column);

-- Composite index (order matters — most-filtered column first)
CREATE INDEX idx_name ON table_name(col_a, col_b);

-- Unique index (speeds lookups + blocks duplicates)
CREATE UNIQUE INDEX idx_name ON table_name(column);

-- Prefix index (first N chars of a long text column)
CREATE INDEX idx_name ON table_name(column(N));

-- Inspect indexes on a table
SHOW INDEXES FROM table_name;

-- Remove an index
DROP INDEX idx_name ON table_name;

-- Plan a query (does NOT run it)
EXPLAIN SELECT ... ;

-- Run a query and report actual timing + rows
EXPLAIN ANALYZE SELECT ... ;
```
