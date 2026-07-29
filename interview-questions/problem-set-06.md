# 🧩 SQL Interview Practice — Problem Set 6

> **Part 6: Problem Set** — seven problems mixing query puzzles (consecutive days, "bought all", missing IDs, set difference) with the theory questions interviewers use as filters (`GROUP BY` rules, `DELETE`/`TRUNCATE`/`DROP`, `IFNULL` vs `COALESCE`).
> Each problem follows the same flow: **Question → Dataset → Expected Output → Solution → Explanation.**

Every problem is self-contained: run the *Dataset* block, then run the solution to reproduce the expected output. All queries target **MySQL 8.0+** and were executed and verified.

---

## 📋 Contents

| # | Problem | Core Concept |
|---|---------|--------------|
| 1 | [Customers who ordered on consecutive days](#q1--customers-who-ordered-on-consecutive-days) | `LAG()` + `DATEDIFF` |
| 2 | [Customers who purchased all products](#q2--customers-who-purchased-all-products) | Relational division (`HAVING` count-match) |
| 3 | [Find missing order IDs](#q3--find-missing-order-ids) | Recursive CTE + anti-join |
| 4 | [Bought product A but never B](#q4--bought-product-a-but-never-b) | Conditional `HAVING` / `NOT EXISTS` |
| 5 | [Why can't we use `SELECT *` with `GROUP BY`?](#q5--why-cant-we-use-select--with-group-by) | `ONLY_FULL_GROUP_BY` |
| 6 | [`DELETE` vs `TRUNCATE` vs `DROP`](#q6--delete-vs-truncate-vs-drop) | DML vs DDL |
| 7 | [`IFNULL` vs `COALESCE`](#q7--ifnull-vs-coalesce) | NULL handling |

---

## Q1 — Customers who ordered on consecutive days

**Question:** Find every customer who placed orders on two or more **consecutive calendar days**.

### 🗄️ Dataset

```sql
CREATE TABLE orders6 (
    order_id INT,
    customer VARCHAR(5),
    order_date DATE
);

INSERT INTO orders6 VALUES
(101,'C1','2024-01-01'),
(102,'C1','2024-01-02'),
(103,'C1','2024-01-03'),
(104,'C1','2024-01-06'),
(105,'C2','2024-01-03'),
(106,'C2','2024-01-05'),
(107,'C2','2024-01-08'),
(108,'C3','2024-02-10'),
(109,'C3','2024-02-11'),
(110,'C3','2024-02-12'),
(111,'C4','2024-03-15');
```

### ✅ Expected Output

| customer |
|----------|
| C1 |
| C3 |

### 💡 Solution

```sql
SELECT DISTINCT customer
FROM (
  SELECT customer, order_date,
    LAG(order_date) OVER (PARTITION BY customer ORDER BY order_date) AS previous_order_date
  FROM orders6
) t
WHERE DATEDIFF(order_date, previous_order_date) = 1
ORDER BY customer;
```

### 🧠 Explanation

- **`LAG()` pulls the previous row's value onto the current row.** `PARTITION BY customer` restarts the lookback for each customer (so C2's first order never compares against C1's last), and `ORDER BY order_date` defines what "previous" means. Every row now carries both its own date and the date before it.
- **`DATEDIFF(order_date, previous_order_date) = 1` is the consecutive test.** `DATEDIFF` returns whole days between two dates, so a difference of exactly `1` means back-to-back days. C1 gets `1` twice (Jan 1→2, Jan 2→3) and `3` once (Jan 3→6); C2 gets `2` and `3` — never adjacent; C4 has a single order, so its `LAG` is `NULL`.
- **Why the subquery is required:** window functions are evaluated *after* `WHERE`, so you cannot filter on `LAG(...)` in the same `SELECT` that computes it. Wrap it in a derived table (or CTE) and filter on the outer level.
- **`DISTINCT` collapses multiple hits.** C1 satisfies the condition on two rows but should appear once. `NULL` rows (each customer's earliest order) fail the comparison automatically — `DATEDIFF(x, NULL)` is `NULL`, and `NULL = 1` is never true, so no extra `IS NOT NULL` guard is needed.
- 💡 To find customers with a streak of **3+** days rather than any 2, switch to gaps-and-islands: `DATE_SUB(order_date, INTERVAL ROW_NUMBER() OVER (PARTITION BY customer ORDER BY order_date) DAY)` stays constant within a run, so `GROUP BY` that expression and add `HAVING COUNT(*) >= 3`.

---

## Q2 — Customers who purchased all products

**Question:** Find the customers who have purchased **every** product in the catalogue.

### 🗄️ Dataset

```sql
CREATE TABLE purchases (
    customer VARCHAR(5),
    product VARCHAR(10)
);

INSERT INTO purchases VALUES
('C1','Laptop'),
('C1','Mouse'),
('C1','Keyboard'),

('C2','Laptop'),
('C2','Mouse'),

('C3','Laptop'),
('C3','Mouse'),
('C3','Keyboard'),

('C4','Keyboard');

CREATE TABLE products (
    product VARCHAR(10)
);

INSERT INTO products VALUES
('Laptop'),
('Mouse'),
('Keyboard');
```

### ✅ Expected Output

| customer |
|----------|
| C1 |
| C3 |

### 💡 Solution

```sql
SELECT customer, COUNT(DISTINCT product) AS product_count
FROM purchases
GROUP BY customer
HAVING COUNT(DISTINCT product) = (
  SELECT COUNT(*)
  FROM products
)
ORDER BY customer;
```

### 🧠 Explanation

- **This is *relational division*** — "find the rows that pair with *all* values in another set." The count-matching trick is the standard SQL answer: if a customer's distinct product count equals the catalogue size, they must have bought everything.
- **`COUNT(DISTINCT product)`, not `COUNT(*)`.** If the same customer buys a Laptop twice, `COUNT(*)` would be 4 for three products and the comparison would silently fail. `DISTINCT` counts *which* products, not how many purchase rows.
- **The scalar subquery `(SELECT COUNT(*) FROM products)` is the target number** — here `3`. It's evaluated once, and because it's driven by the `products` table the query keeps working when the catalogue grows; nothing is hard-coded.
- **`HAVING`, not `WHERE`:** the filter compares an aggregate, and `WHERE` runs before grouping. C1 and C3 reach 3; C2 reaches 2; C4 reaches 1.
- 💡 The alternative is a **double `NOT EXISTS`** ("no product exists that this customer hasn't bought"), which is textbook-correct even when `purchases` may reference products *not* in the catalogue — a case the count method can be fooled by:

  ```sql
  SELECT DISTINCT p.customer
  FROM purchases p
  WHERE NOT EXISTS (
    SELECT 1 FROM products pr
    WHERE NOT EXISTS (
      SELECT 1 FROM purchases p2
      WHERE p2.customer = p.customer AND p2.product = pr.product
    )
  );
  ```

---

## Q3 — Find missing order IDs

**Question:** `order_id` should be a gapless sequence. Report the IDs missing between the smallest and largest existing IDs.

### 🗄️ Dataset

```sql
CREATE TABLE orders7 (
    order_id INT
);

INSERT INTO orders7 VALUES
(101),
(102),
(104),
(105),
(107);
```

### ✅ Expected Output

| missing_order_id |
|------------------|
| 103 |
| 106 |

### 💡 Solution

```sql
WITH RECURSIVE numbers AS (
  SELECT MIN(order_id) AS order_id
  FROM orders7

  UNION ALL

  SELECT order_id + 1
  FROM numbers
  WHERE order_id < (SELECT MAX(order_id) FROM orders7)
)
SELECT n.order_id AS missing_order_id
FROM numbers n
LEFT JOIN orders7 o
  ON n.order_id = o.order_id
WHERE o.order_id IS NULL
ORDER BY n.order_id;
```

### 🧠 Explanation

- **You can't find what isn't there — so first *generate* what should be there.** The recursive CTE builds the complete sequence `101 … 107`, then an anti-join subtracts the IDs that actually exist. Missing rows are whatever's left.
- **How the recursive CTE runs:** the **anchor** `SELECT MIN(order_id)` seeds it with `101`. The **recursive member** takes the previous row and emits `order_id + 1`, repeatedly, and the `WHERE order_id < (SELECT MAX(order_id))` **stops it at 107** — without that guard it would run until MySQL's `cte_max_recursion_depth` (default 1000) aborts the query.
- **`UNION ALL`, not `UNION`:** the recursive member must not deduplicate, and `UNION` would also add pointless sort/dedup work on every iteration.
- **`LEFT JOIN … WHERE o.order_id IS NULL` is the anti-join.** Every generated number is kept; those with a match in `orders7` get a real value in `o.order_id`, those without get `NULL` — so the `IS NULL` filter keeps exactly the missing IDs, `103` and `106`.
- 💡 Note the boundary semantics: this finds gaps **inside** the observed range only. It cannot know that `100` or `108` are missing, because `MIN`/`MAX` come from the data itself. If the sequence should start at a fixed value, seed the anchor with that literal (`SELECT 100`) instead of `MIN(order_id)`. `NOT IN (SELECT order_id FROM orders7)` is an equally valid substitute for the anti-join — but beware `NOT IN` with a `NULL`-able column, which returns no rows at all.

---

## Q4 — Bought product A but never B

**Question:** Find customers who bought product **A** but have never bought product **B**.

### 🗄️ Dataset

```sql
CREATE TABLE customer_products (
    customer VARCHAR(5),
    product CHAR(1)
);

INSERT INTO customer_products VALUES
('C1','A'),
('C1','B'),

('C2','A'),

('C3','B'),

('C4','A'),
('C4','C'),

('C5','A'),
('C5','B'),
('C5','C');
```

### ✅ Expected Output

| customer |
|----------|
| C2 |
| C4 |

### 💡 Solution — conditional aggregation

```sql
SELECT customer
FROM customer_products
GROUP BY customer
HAVING SUM(product = 'A') > 0
   AND SUM(product = 'B') = 0
ORDER BY customer;
```

### 💡 Solution — `NOT EXISTS`

```sql
SELECT c1.customer
FROM customer_products c1
WHERE c1.product = 'A'
  AND NOT EXISTS (
    SELECT 1
    FROM customer_products c2
    WHERE c2.customer = c1.customer
      AND c2.product = 'B'
  )
ORDER BY c1.customer;
```

### 🧠 Explanation

- **Both queries answer one condition per customer: "A present, B absent."** The trap is doing it row-by-row — `WHERE product = 'A' AND product <> 'B'` is always just "product = A", because a single row can't be two products at once. The test has to span *all* of a customer's rows.
- **Conditional aggregation:** in MySQL a boolean expression evaluates to `1`/`0`, so `SUM(product = 'A')` counts A-rows and `SUM(product = 'B')` counts B-rows. `> 0` means "at least one A", `= 0` means "not a single B". C1 and C5 fail the second test; C3 fails the first; C2 and C4 pass both. (`SUM(CASE WHEN product = 'A' THEN 1 ELSE 0 END)` is the portable spelling of the same idea.)
- **`NOT EXISTS` version:** the outer query keeps only A-rows, and the correlated subquery — linked by `c2.customer = c1.customer` — asks "does this same customer have a B row?" `NOT EXISTS` keeps them only when the answer is no. It short-circuits on the first match, so it's typically the faster plan when `(customer, product)` is indexed.
- **Which to reach for:** conditional aggregation scales cleanly to compound rules (`AND SUM(product = 'C') > 0`, counts, thresholds) in one pass; `NOT EXISTS` is the more direct read of "never bought" and is the idiom to know for anti-join questions generally.
- 💡 `NOT IN (SELECT customer FROM customer_products WHERE product = 'B')` reads well too — but if that subquery can ever produce a `NULL` customer, `NOT IN` returns **zero rows**. `NOT EXISTS` is `NULL`-safe; prefer it.

---

## Q5 — Why can't we use `SELECT *` with `GROUP BY`?

**Question:** Why does `SELECT * FROM employees GROUP BY dept` fail, and what's the correct way to write it?

### 🗄️ Dataset

```sql
CREATE TABLE employees (
    dept VARCHAR(20),
    employee VARCHAR(20),
    salary INT
);

INSERT INTO employees VALUES
('HR','Amit',50000),
('HR','Rohit',55000),
('IT','Vivek',70000),
('IT','Ankit',75000);
```

### ✅ Expected Output

| dept | total_count |
|------|-------------|
| HR | 2 |
| IT | 2 |

### 💡 Solution

```sql
-- ❌ Fails under ONLY_FULL_GROUP_BY (the MySQL 5.7+ default)
SELECT * FROM employees GROUP BY dept;

-- ✅ Every selected column is either grouped or aggregated
SELECT dept, COUNT(*) AS total_count
FROM employees
GROUP BY dept
ORDER BY dept;
```

### 🧠 Explanation

- **`GROUP BY` collapses many rows into one per group — so every selected column must be answerable *for the group as a whole*.** That means each column has to be either in the `GROUP BY` list or wrapped in an aggregate (`COUNT`, `SUM`, `MIN`, …). `SELECT *` drags in `employee` and `salary`, which are neither.
- **The real problem is ambiguity, not syntax.** The `HR` group contains `Amit` and `Rohit`. Asked for one `employee` value, the engine has no defensible answer — so `ONLY_FULL_GROUP_BY` (enabled by default since MySQL 5.7) rejects the query instead of silently inventing one.
- **Older MySQL used to allow it**, returning an arbitrary row's value per group. That's why legacy queries "work" on old servers and break after an upgrade — the data was never wrong-proof, the server just stopped guessing. Disabling `ONLY_FULL_GROUP_BY` to make such a query run is hiding a bug, not fixing one.
- **Fix by deciding what you actually mean:** aggregate it (`MAX(salary)`), group by it (`GROUP BY dept, employee`), or — if you genuinely want *whole rows* alongside a group-level number — use a window function instead: `SELECT *, COUNT(*) OVER (PARTITION BY dept) FROM employees` keeps all four rows and attaches the per-dept count.
- 💡 `ANY_VALUE(employee)` is the escape hatch: it tells MySQL "I know this is arbitrary and I accept it," passing the check without turning the SQL mode off globally. Use it only when the column is functionally dependent on the grouping key (e.g. grouping by `employee_id` and selecting `employee_name`).

---

## Q6 — `DELETE` vs `TRUNCATE` vs `DROP`

**Question:** Explain the difference between `DELETE`, `TRUNCATE`, and `DROP`.

### 🗄️ Dataset

```sql
CREATE TABLE students (
    id INT,
    name VARCHAR(20)
);

INSERT INTO students VALUES
(1,'Amit'),
(2,'Bob'),
(3,'Rahul');
```

### ✅ Expected Output

| Command | One-line Explanation |
|---------|----------------------|
| `DELETE` | Removes selected rows and can use a `WHERE` clause. |
| `TRUNCATE` | Removes all rows quickly and resets the auto-increment value. |
| `DROP` | Deletes the entire table including its structure and data. |

### 💡 Solution

```sql
-- Rows only, filtered, undoable inside a transaction
DELETE FROM students
WHERE id = 2;

-- All rows, fast, resets AUTO_INCREMENT, cannot be rolled back
TRUNCATE TABLE students;

-- Table itself is gone — data, structure, indexes, triggers
DROP TABLE students;
```

### 🧠 Explanation

- **The one-line framing interviewers want:** `DELETE` removes **rows you choose**, `TRUNCATE` removes **all rows**, `DROP` removes **the table**.
- **`DELETE` is DML.** It deletes row by row, writes each removal to the transaction log, fires `DELETE` triggers, and is **rollback-able** inside a transaction. That makes it the slowest option on large tables — and the only one that accepts a `WHERE`.
- **`TRUNCATE` is DDL.** MySQL implements it by dropping and recreating the table, so it's dramatically faster on big tables, but: no `WHERE`, no row-level triggers, **auto-commits** (an open transaction can't roll it back), and it **resets `AUTO_INCREMENT` to 1**. `DELETE FROM t` with no `WHERE` empties the table too but keeps the counter climbing — the classic follow-up question.
- **`DROP` is DDL and total.** Structure, data, indexes, triggers, and privileges all go; the table name no longer exists, so subsequent queries error out rather than returning zero rows. You must `CREATE TABLE` again to reuse it.
- **Foreign keys behave differently:** `TRUNCATE` is **blocked** on a table referenced by a foreign key, and `DROP` is too (unless the constraint is removed or `foreign_key_checks` is off). `DELETE` respects the constraint's `ON DELETE` action (`CASCADE`, `RESTRICT`, `SET NULL`) instead.
- 💡 Rule of thumb: **filtered delete → `DELETE`. Emptying a staging/temp table → `TRUNCATE`. Removing the object entirely → `DROP`.** Only `DELETE` is safely reversible — run the equivalent `SELECT` first, and wrap it in a transaction before touching production.

---

## Q7 — `IFNULL` vs `COALESCE`

**Question:** Explain the difference between `IFNULL()` and `COALESCE()`, and use each to supply a default bonus.

### 🗄️ Dataset

```sql
CREATE TABLE employee_bonus (
    id INT,
    name VARCHAR(20),
    bonus INT,
    incentive INT
);

INSERT INTO employee_bonus VALUES
(1,'Amit',NULL,1000),
(2,'Bob',500,NULL),
(3,'Rahul',NULL,NULL);
```

### ✅ Expected Output

| Function | One-line Explanation |
|----------|----------------------|
| `IFNULL()` | Returns an alternative value if the first expression is `NULL`. |
| `COALESCE()` | Returns the first non-`NULL` value from multiple expressions. |

**`IFNULL` — two arguments only:**

| bonus | bonuses |
|-------|---------|
| NULL | 10000 |
| 500  | 500   |
| NULL | 10000 |

**`COALESCE` — falls back through a chain:**

| bonus | incentive | bonuses |
|-------|-----------|---------|
| NULL | 1000 | 1000  |
| 500  | NULL | 500   |
| NULL | NULL | 10000 |

### 💡 Solution

```sql
SELECT bonus, IFNULL(bonus, 10000) AS bonuses
FROM employee_bonus
ORDER BY id;

SELECT bonus, incentive, COALESCE(bonus, incentive, 10000) AS bonuses
FROM employee_bonus
ORDER BY id;
```

### 🧠 Explanation

- **Same job, different arity.** `IFNULL(a, b)` takes **exactly two** arguments: return `a` unless it's `NULL`, else `b`. `COALESCE(a, b, c, …)` takes **any number** and returns the first non-`NULL` one, left to right. `IFNULL(a, b)` is exactly `COALESCE(a, b)`.
- **Read the rows:** for Amit, `IFNULL` jumps straight to the literal `10000` because it never sees `incentive`, while `COALESCE` finds `incentive = 1000` first and stops there. Bob has a real bonus, so both return `500`. Rahul has neither, so both fall through to `10000` — the last argument is the safety net that guarantees a non-`NULL` result.
- **`COALESCE` is standard SQL; `IFNULL` is MySQL-specific** (SQL Server's equivalent is `ISNULL`, Oracle's is `NVL`). If portability matters at all, default to `COALESCE` — it's also strictly more capable, which is why it's the better habit.
- **Short-circuit behaviour matters with expensive arguments:** both stop evaluating once they find a non-`NULL`, so put the cheap, most-likely-populated column first — especially when a later argument is a subquery.
- 💡 Two gotchas worth saying out loud. **`NULL` is not `0` or `''`** — `IFNULL(bonus, 0)` only replaces true `NULL`s, so a bonus stored as `0` stays `0`. And the **result type is the widest of the arguments**, so `COALESCE(int_col, 'N/A')` quietly returns a string and can break downstream arithmetic or sorting; keep the fallback the same type as the column. Don't confuse either with `NULLIF(a, b)`, which does the reverse — it *produces* `NULL` when `a = b`.

---

<div align="center">

**More problem sets coming soon.** ⭐ the repo to follow along.

[⬅ Back to Course Home](../README.md) · [Problem Set 1](problem-set-01.md) · [Problem Set 2](problem-set-02.md) · [Problem Set 3](problem-set-03.md) · [Problem Set 4](problem-set-04.md) · [Problem Set 5](problem-set-05.md)

</div>
