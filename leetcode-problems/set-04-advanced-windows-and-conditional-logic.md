# 🟪 LeetCode SQL 50 — Set 4: Advanced Windows & Conditional Logic

> **Problems 31–40 of the [LeetCode SQL 50](https://leetcode.com/studyplan/top-sql-50/) study plan.**
> Every problem follows the same flow: **Problem → Schema → Sample Input → Expected Output → Approach → Solution → Explanation.**

Set 3 established the first-row-per-group pattern. **Set 4 is where the window *frame* starts to matter** — running totals, 7-day moving averages, and looking two rows ahead. Up to now every window function used the default frame and you could ignore it. Here, `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` *is* the answer, and the difference between `ROWS` and `RANGE` is the difference between a correct moving average and a plausible-looking wrong one.

The other half is **conditional logic that has to produce rows that aren't in the data**: a salary bucket with zero accounts, a price for a product that was never repriced, a seat swap at the end of an odd-length row. `GROUP BY` cannot invent a row — and knowing when to reach for `UNION ALL` instead is what these problems are really testing.

All queries target **MySQL 8.0+** and were executed against LeetCode's own sample data on a local MySQL 9.6 server — the Expected Output blocks below are real query output, not hand-written. Where two solutions look equivalent but diverge on data LeetCode's samples don't contain, the divergence is shown with real output too.

---

## 📋 Contents

| # | Problem | Difficulty | Core Concept |
|---|---------|:---:|--------------|
| 1 | [Primary Department for Each Employee](#q1--primary-department-for-each-employee) | 🟩 Easy | `COUNT(*) OVER (PARTITION BY)` as a group-size tag |
| 2 | [Triangle Judgement](#q2--triangle-judgement) | 🟩 Easy | `CASE` vs `IF` — row-level branching |
| 3 | [Consecutive Numbers](#q3--consecutive-numbers) | 🟨 Medium | `LEAD(col, n)` with an offset |
| 4 | [Product Price at a Given Date](#q4--product-price-at-a-given-date) | 🟨 Medium | Latest-before-a-date + a default for the missing |
| 5 | [Last Person to Fit in the Bus](#q5--last-person-to-fit-in-the-bus) | 🟨 Medium | Running total, then the last row under a cap |
| 6 | [Count Salary Categories](#q6--count-salary-categories) | 🟨 Medium | `UNION ALL` to force zero-count buckets |
| 7 | [Employees Whose Manager Left the Company](#q7--employees-whose-manager-left-the-company) | 🟩 Easy | Self anti-join |
| 8 | [Exchange Seats](#q8--exchange-seats) | 🟨 Medium | Arithmetic on `id` + an odd-tail edge case |
| 9 | [Movie Rating](#q9--movie-rating) | 🟨 Medium | Two unrelated top-1 queries, stacked |
| 10 | [Restaurant Growth](#q10--restaurant-growth) | 🟨 Medium | 7-day moving window over pre-aggregated days |

---

## Q1 — Primary Department for Each Employee

🔗 **[leetcode.com/problems/primary-department-for-each-employee](https://leetcode.com/problems/primary-department-for-each-employee/description/)**

**Problem:** Employees can belong to multiple departments; the primary one is flagged `'Y'`. An employee belonging to **only one** department has that department as primary **even though it's flagged `'N'`**. Report each employee's primary department.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Employee;
CREATE TABLE Employee (
    employee_id   INT,
    department_id INT,
    primary_flag  ENUM('Y','N'),
    PRIMARY KEY (employee_id, department_id)
);

INSERT INTO Employee VALUES
(1,1,'N'),(2,1,'Y'),(2,2,'N'),(3,3,'N'),(4,2,'N'),(4,3,'Y'),(4,4,'N');
```

### ✅ Expected Output

| employee_id | department_id |
|-------------|---------------|
| 1 | 1 |
| 2 | 1 |
| 3 | 3 |
| 4 | 3 |

### 🎯 How to approach it

**Two rules that can't be expressed as one filter.** `WHERE primary_flag = 'Y'` gets employees 2 and 4 and silently loses 1 and 3, who have a single department each flagged `'N'`. The single-department case has to be rescued separately.

The obstacle is that "how many departments does this employee have?" is a **group-level fact** that has to be attached to **each row** before you can filter on it. That's precisely what `COUNT(*) OVER (PARTITION BY employee_id)` does: it computes the group size and stamps it on every row of the group without collapsing anything. Then a single `WHERE` handles both branches.

### 💡 Solution

```sql
WITH count_of_employee AS (
    SELECT *,
           COUNT(*) OVER (PARTITION BY employee_id) AS count
    FROM Employee
)
SELECT employee_id,
       department_id
FROM count_of_employee
WHERE count = 1
   OR (count > 1 AND primary_flag = 'Y');
```

### 🧠 Explanation

- **`COUNT(*) OVER (PARTITION BY employee_id)` with no `ORDER BY`** counts the *whole* partition, not a running count. Employee 4's three rows each get `count = 3`. Adding `ORDER BY` would turn it into a running count (1, 2, 3) and break the logic — a genuinely easy mistake, and the reason the omission is deliberate rather than accidental.
- **The `count > 1` half of the predicate is redundant but honest.** `WHERE count = 1 OR primary_flag = 'Y'` returns the same rows, because LeetCode guarantees at most one `'Y'` per employee. Keeping `count > 1` makes the two business rules explicit in the SQL, which is what you want when the guarantee is a constraint someone can drop later.
- **`count` is not a reserved word in MySQL** so it works unquoted as a column alias — but `COUNT` is a function name, and the resemblance is close enough that `dept_count` would be kinder to the next reader.
- 💡 **The no-window alternative** is worth knowing for MySQL 5.7:

  ```sql
  SELECT employee_id, department_id FROM Employee WHERE primary_flag = 'Y'
  UNION ALL
  SELECT employee_id, department_id FROM Employee
  WHERE employee_id IN (
      SELECT employee_id FROM Employee GROUP BY employee_id HAVING COUNT(*) = 1
  );
  ```

  It reads as the two rules literally — and needs `UNION ALL`, not `UNION`, since the branches are provably disjoint and deduplication would be wasted work.

---

## Q2 — Triangle Judgement

🔗 **[leetcode.com/problems/triangle-judgement](https://leetcode.com/problems/triangle-judgement/description/)**

**Problem:** For each row of three side lengths, report whether they can form a triangle.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Triangle;
CREATE TABLE Triangle (
    x INT,
    y INT,
    z INT,
    PRIMARY KEY (x, y, z)
);

INSERT INTO Triangle VALUES (13,15,30),(10,20,15);
```

### ✅ Expected Output

| x | y | z | triangle |
|---|---|---|----------|
| 10 | 20 | 15 | Yes |
| 13 | 15 | 30 | No |

### 🎯 How to approach it

Pure **row-level branching** — no grouping, no joins, one output row per input row. The only thing to get right is the maths.

The **triangle inequality** says the sum of any two sides must exceed the third. All three comparisons are required:

```
x + y > z    AND    y + z > x    AND    z + x > y
```

Checking only the largest pair is a common shortcut that works *if* you know which side is largest — but SQL doesn't sort the columns for you, so write all three. For 13/15/30: `13 + 15 = 28`, not greater than 30 → **No**. For 10/20/15: 30 > 15, 35 > 10, 25 > 20 → **Yes**.

Note **`>` and not `>=`**: sides of 1, 2, 3 are collinear, a degenerate triangle with zero area. LeetCode counts that as `No`.

### 💡 Solution

```sql
SELECT *,
       CASE
           WHEN x + y > z AND y + z > x AND z + x > y
           THEN 'Yes'
           ELSE 'No'
       END AS triangle
FROM Triangle;
```

### 🔄 Alternative — `IF()`

```sql
SELECT *,
       IF(x + y > z AND y + z > x AND z + x > y, 'Yes', 'No') AS triangle
FROM Triangle;
```

Identical result, verified. `IF()` is more compact for a two-way branch; **`CASE` is ANSI SQL and `IF()` is MySQL-only**, so `CASE` is what ports to PostgreSQL, SQL Server, Snowflake, and BigQuery unchanged. For a two-outcome test either is fine — reach for `CASE` the moment a third branch appears, since nested `IF()` gets unreadable fast.

### 🧠 Explanation

- **`CASE` returns the first matching branch**, so ordering matters when conditions overlap. Here there's one condition and an `ELSE`, so it can't bite — but in [Q6](#q6--count-salary-categories) the branch order does real work.
- **`ELSE` is not optional in spirit.** Omit it and non-matching rows get `NULL`, not `'No'` — the output would be blank where the answer should be a word. Always write the `ELSE` unless `NULL` is genuinely what you mean.
- **The rows came back in `(10,20,15)` order** because the primary key on `(x,y,z)` makes 10 sort before 13. The problem accepts any order and the output above is what MySQL actually returned — a reminder that **result order without `ORDER BY` is an artifact of the plan**, not a promise.
- 💡 **This generalises to any row-level classification** — risk tiers, pass/fail, shipping bands. The pattern is one `CASE` in the `SELECT` list and nothing else; if you find yourself adding a `GROUP BY`, you've drifted into [Q6](#q6--count-salary-categories) territory.

---

## Q3 — Consecutive Numbers

🔗 **[leetcode.com/problems/consecutive-numbers](https://leetcode.com/problems/consecutive-numbers/description/)**

**Problem:** Find all numbers that appear **at least three times consecutively**.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Logs;
CREATE TABLE Logs (
    id  INT PRIMARY KEY,
    num INT
);

INSERT INTO Logs VALUES (1,1),(2,1),(3,1),(4,2),(5,1),(6,2),(7,2);
```

### ✅ Expected Output

| ConsecutiveNums |
|-----------------|
| 1 |

### 🎯 How to approach it

**Flatten the vertical run into a horizontal row.** Comparing a row to its neighbours is hard while the neighbours live in other rows — so pull them onto the current row first, then the test is a plain `WHERE`.

`LEAD(num, 1)` fetches the next row's number and `LEAD(num, 2)` the one after. Once every row carries `(num, next, next2)`, "three in a row" is literally `num = next AND num = next2`:

| id | num | next_num | next_num2 | three in a row? |
|---|---|---|---|---|
| 1 | 1 | 1 | 1 | ✅ |
| 2 | 1 | 1 | 2 | ❌ |
| 5 | 1 | 2 | 2 | ❌ |
| 6 | 2 | 2 | NULL | ❌ |

Only id 1 qualifies, so the answer is `1`. **`DISTINCT` is required** — a run of five identical values would produce three qualifying rows and report the same number three times.

The rows near the end get `NULL` from `LEAD` (nothing to look ahead to), and `NULL = 2` is `NULL`, which `WHERE` discards. The boundary handles itself.

### 💡 Solution

```sql
WITH logs_next_num AS (
    SELECT *,
           LEAD(num, 1) OVER (ORDER BY id) AS next_num,
           LEAD(num, 2) OVER (ORDER BY id) AS next_num2
    FROM Logs
)
SELECT DISTINCT num AS ConsecutiveNums
FROM logs_next_num
WHERE num = next_num
  AND num = next_num2;
```

### 🧠 Explanation

- **`LEAD(col, n)` takes an offset** — the second argument is how many rows forward to look, defaulting to 1. There's also an optional third argument for the default when the offset runs off the end: `LEAD(num, 2, -1)` would return `-1` instead of `NULL`. Rarely needed, but it's the clean way to avoid `NULL`-handling in the predicate.
- **No `PARTITION BY` here** because the run must span the whole table. If the problem were "three consecutive readings per sensor", you'd add `PARTITION BY sensor_id` and everything else would stay identical — that one clause is the difference between a global streak and a per-entity streak.
- ⚠️ **`ORDER BY id` defines "consecutive" as *row adjacency*, not *id adjacency*** — and that distinction is real. The classic alternative joins the table to itself on `b.id = a.id + 1` and `c.id = a.id + 2`, which defines consecutive as *ids differing by one*. On a table with gapped ids (1, 2, 9 — all the same number) the two disagree:

  | Method | Result |
  |---|---|
  | `LEAD` over `ORDER BY id` | **5** — three adjacent rows |
  | Self-join on `id+1`, `id+2` | *(no rows)* — ids aren't contiguous |

  That's verified output. `Logs.id` is an auto-increment column with no gaps in LeetCode's tests so both are accepted, but **deleted rows create gaps in any real table**, and the self-join then silently reports nothing. The `LEAD` version is the one that survives contact with production data.
- 💡 **Three copies of `LEAD` is fine; thirty isn't.** For "at least *N* consecutive", switch to the **gaps-and-islands** technique — `ROW_NUMBER()` minus a second `ROW_NUMBER()` partitioned by value gives a constant per run, then `GROUP BY` that constant and use `HAVING COUNT(*) >= N`. See [Problem Set 4](../interview-questions/problem-set-04.md) for the streak walkthrough.

---

## Q4 — Product Price at a Given Date

🔗 **[leetcode.com/problems/product-price-at-a-given-date](https://leetcode.com/problems/product-price-at-a-given-date/description/)**

**Problem:** Find the price of every product on **2019-08-16**. Products with no price change on or before that date have a price of **10**.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Products;
CREATE TABLE Products (
    product_id  INT,
    new_price   INT,
    change_date DATE,
    PRIMARY KEY (product_id, change_date)
);

INSERT INTO Products VALUES
(1,20,'2019-08-14'),(2,50,'2019-08-14'),(1,30,'2019-08-15'),
(1,35,'2019-08-16'),(2,65,'2019-08-17'),(3,20,'2019-08-18');
```

### ✅ Expected Output

| product_id | price |
|------------|-------|
| 1 | 35 |
| 2 | 50 |
| 3 | 10 |

### 🎯 How to approach it

**A slowly-changing-dimension lookup** — "what was the value as of date D?" — and it's the most production-relevant problem in this set. The table stores *changes*, not *states*, so the price on any date is the most recent change at or before it.

Three moving parts:

1. **Discard the future.** `WHERE change_date <= '2019-08-16'` throws away product 2's 08-17 change and product 3's 08-18 change. Product 3 now has no rows at all — which is exactly the case the default of 10 exists for.
2. **Keep only the latest surviving change per product.** `ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY change_date DESC)` and take rank 1 — the same first-row pattern as [Set 3 Q1](set-03-window-functions-and-first-rows.md#q1--immediate-food-delivery-ii), just pointed at the *newest* row instead of the oldest via `DESC`.
3. **Get product 3 back.** Steps 1–2 eliminated it entirely, so the product list has to come from somewhere that wasn't filtered: `SELECT DISTINCT product_id FROM Products`, `LEFT JOIN`ed to the ranked prices, with `COALESCE(new_price, 10)` supplying the default.

**Step 3 is the whole difficulty.** Everything else is a standard latest-row lookup; the trap is that the filter in step 1 removes not just the wrong *rows* but the entire *product*.

### 💡 Solution

```sql
WITH ranking_price AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY product_id ORDER BY change_date DESC) AS ranking
    FROM Products
    WHERE change_date <= '2019-08-16'
)
SELECT p.product_id,
       COALESCE(r.new_price, 10) AS price
FROM (SELECT DISTINCT product_id FROM Products) p
LEFT JOIN ranking_price r
    ON p.product_id = r.product_id
   AND r.ranking = 1;
```

### 🧠 Explanation

- **`ORDER BY change_date DESC` + `ranking = 1` is "the latest".** Flipping the sort direction is the entire difference between first-row and last-row logic — the rest of the pattern is unchanged.
- ⚠️ **`AND r.ranking = 1` must live in the `ON` clause, not in `WHERE`.** This is the [Set 2 Q1](set-02-joins-and-aggregation.md#q1--employee-bonus) trap in a new costume: product 3 has no matching row, so `r.ranking` is `NULL` for it, and `WHERE r.ranking = 1` evaluates to `NULL` and discards the row — turning the `LEFT JOIN` back into an inner one. Verified:

  | Filter placement | Rows returned |
  |---|---|
  | `AND r.ranking = 1` inside `ON` | **1, 2, 3** ✅ |
  | `WHERE r.ranking = 1` | 1, 2 ❌ — product 3 vanishes, default never applied |

  **A predicate on the right-hand table of a `LEFT JOIN` belongs in `ON`.** That single rule prevents this bug permanently.
- **`SELECT DISTINCT product_id FROM Products` is the product master list.** In a real schema you'd join to an actual `Product` table; here the source of truth for "which products exist" is the unfiltered change log. Note it must be **unfiltered** — deriving it from the already-filtered CTE would lose product 3 again.
- **`COALESCE` vs `IFNULL`:** identical here. `COALESCE` is ANSI standard and takes any number of arguments; `IFNULL` is MySQL-only and takes exactly two. Prefer `COALESCE` out of habit.
- 💡 **This is the shape behind every "price/status/plan as of date X" report.** Swap `new_price` for `subscription_tier` and it's customer state on a billing date; swap in `exchange_rate` and it's currency conversion for a historical order.

---

## Q5 — Last Person to Fit in the Bus

🔗 **[leetcode.com/problems/last-person-to-fit-in-the-bus](https://leetcode.com/problems/last-person-to-fit-in-the-bus/description/)**

**Problem:** People board in `turn` order. The bus has a **1000 kg** weight limit. Find the **last person** who can board without exceeding it.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Queue;
CREATE TABLE Queue (
    person_id   INT PRIMARY KEY,
    person_name VARCHAR(30),
    weight      INT,
    turn        INT
);

INSERT INTO Queue VALUES
(5,'Alice',250,1),(4,'Bob',175,5),(3,'Alex',350,2),
(6,'John Cena',400,3),(1,'Winston',500,6),(2,'Marie',200,4);
```

### ✅ Expected Output

| person_name |
|-------------|
| John Cena |

### 🎯 How to approach it

**A running total, then the last row still under the cap.**

| turn | person | weight | cumulative |
|---|---|---|---|
| 1 | Alice | 250 | 250 |
| 2 | Alex | 350 | 600 |
| 3 | John Cena | 400 | **1000** ✅ |
| 4 | Marie | 200 | 1200 ❌ |
| 5 | Bob | 175 | 1375 ❌ |
| 6 | Winston | 500 | 1875 ❌ |

Boarding stops at 1000 exactly — **`<=`, not `<`**, since hitting the limit precisely is still fitting. Marie would push it to 1200, so she and everyone behind her stay off.

`SUM(weight) OVER (ORDER BY turn)` produces the cumulative column: adding `ORDER BY` to an aggregate window function turns it from a whole-partition total into a **running** total. Then filter to rows at or under 1000 and take the largest.

### 💡 Solution

```sql
WITH bus_weight AS (
    SELECT *,
           SUM(weight) OVER (ORDER BY turn) AS running_weight
    FROM Queue
)
SELECT person_name
FROM bus_weight
WHERE running_weight <= 1000
ORDER BY running_weight DESC
LIMIT 1;
```

### 🧠 Explanation

- **`ORDER BY` inside `OVER()` is what creates the running total.** Without it you'd get 1875 on every row — the grand total. This is the same clause that turned `COUNT(*) OVER()` into a running count back in [Q1](#q1--primary-department-for-each-employee), where it had to be *omitted*. Same knob, opposite requirement.
- **`ORDER BY running_weight DESC LIMIT 1` finds the last boarder** because the running total is monotonically increasing (weights are positive), so the largest total under the cap is also the latest turn under the cap. `ORDER BY turn DESC LIMIT 1` is equivalent here and states the intent more directly — **it's the one to prefer if weights could ever be zero or negative**, since the running total would no longer be monotonic.
- ⚠️ **The default frame is `RANGE`, and `RANGE` treats ties as one unit.** `SUM(...) OVER (ORDER BY turn)` expands to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which includes **every row with the same `turn` value**, not just the rows up to this one. With two people sharing a turn:

  | person | weight | turn | `RANGE` (default) | `ROWS UNBOUNDED PRECEDING` |
  |---|---|:---:|---|---|
  | A | 600 | 1 | 600 | 600 |
  | B | 300 | 2 | **1200** | 900 |
  | C | 300 | 2 | **1200** | 1200 |

  That's verified output — B's running total jumps to 1200 because C is a *peer*, and B is wrongly reported as unable to board. `turn` is unique in this problem so the default is safe, but **`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` is the frame you almost always mean** for a running total. Writing it explicitly costs eight words and removes an entire class of silent bug.
- 💡 **Same shape, other names:** inventory allocation against available stock, budget consumption until a cap, cumulative refunds against a limit. Running total → filter on the cap → take the boundary row.

---

## Q6 — Count Salary Categories

🔗 **[leetcode.com/problems/count-salary-categories](https://leetcode.com/problems/count-salary-categories/description/)**

**Problem:** Count accounts in each salary category — **Low Salary** (`< 20000`), **Average Salary** (`20000`–`50000` inclusive), **High Salary** (`> 50000`). **Every category must appear, even with a count of zero.**

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Accounts;
CREATE TABLE Accounts (
    account_id INT PRIMARY KEY,
    income     INT
);

INSERT INTO Accounts VALUES (3,108939),(2,12747),(8,87709),(6,91796);
```

### ✅ Expected Output

| category | accounts_count |
|----------|----------------|
| Low Salary | 1 |
| Average Salary | 0 |
| High Salary | 3 |

### 🎯 How to approach it

**The bolded requirement is the entire problem: a category with no accounts must still produce a row reading `0`.**

The instinctive answer is `CASE` in the `SELECT` and `GROUP BY category`. It's shorter, it's what most people write first, and **it is wrong here** — `GROUP BY` can only emit groups that exist in the data, and no account earns between 20000 and 50000. Verified output from the grouped version:

| category | accounts_count |
|---|---|
| Low Salary | 1 |
| High Salary | 3 |

Two rows. **"Average Salary" is missing entirely**, not zero. No amount of `IFNULL` fixes it, because there is no row to apply `IFNULL` to.

So the categories have to come from the *query text* rather than from the data. **`UNION ALL` of three one-row queries** does exactly that: each branch hardcodes its label and counts its own condition, so all three rows exist unconditionally.

### 💡 Solution

```sql
SELECT "Low Salary" AS category,
       COUNT(CASE WHEN income < 20000 THEN 1 END) AS accounts_count
FROM Accounts

UNION ALL

SELECT "Average Salary" AS category,
       COUNT(CASE WHEN income BETWEEN 20000 AND 50000 THEN 1 END) AS accounts_count
FROM Accounts

UNION ALL

SELECT "High Salary" AS category,
       COUNT(CASE WHEN income > 50000 THEN 1 END) AS accounts_count
FROM Accounts;
```

### 🧠 Explanation

- **An aggregate with no `GROUP BY` always returns exactly one row** — even when nothing matches. That's the property doing the work: the Average branch scans four accounts, matches none, and still emits a row containing `0`. (Same principle that made `MAX()` return `NULL` rather than no row in [Set 3 Q8](set-03-window-functions-and-first-rows.md#q8--biggest-single-number).)
- **`COUNT(CASE WHEN cond THEN 1 END)` counts non-`NULL` results.** With no `ELSE`, non-matching rows yield `NULL` and `COUNT` skips them. Three equivalent spellings, all verified to give 1 / 0 / 3:

  | Form | Notes |
  |---|---|
  | `COUNT(CASE WHEN cond THEN 1 END)` | ANSI, no `ELSE` needed — `COUNT` ignores `NULL` |
  | `SUM(CASE WHEN cond THEN 1 ELSE 0 END)` | ANSI, needs `ELSE 0` or an all-`NULL` group returns `NULL` |
  | `SUM(cond)` | MySQL shorthand — booleans coerce to 1/0. Shortest, least portable |

  **The `SUM` form has a sharper edge**: over zero matching rows `SUM` returns `NULL` where `COUNT` returns `0`. It doesn't bite here because the table is non-empty and `ELSE 0` covers it, but on an empty table `SUM(income < 20000)` gives `NULL` and `COUNT(CASE ...)` gives `0`. The `COUNT` form is the safest of the three.
- **`UNION ALL`, never `UNION`.** The three labels are distinct so deduplication would change nothing — but `UNION` sorts or hashes the whole result to find duplicates, and you'd pay for it every run. **Default to `UNION ALL` and only reach for `UNION` when you actually need duplicates removed.** See [Union and Union All](../docs/11-union-and-union-all.md).
- **Column names come from the first branch only.** The later branches' aliases are ignored, which is why they can be omitted — though writing them keeps the query readable.
- 💡 **The scalable version** replaces three table scans with one, by joining a derived list of categories to the data. Worth mentioning in an interview once the table is large:

  ```sql
  SELECT c.category, COUNT(a.account_id) AS accounts_count
  FROM (SELECT 'Low Salary' AS category UNION ALL
        SELECT 'Average Salary' UNION ALL
        SELECT 'High Salary') c
  LEFT JOIN Accounts a
      ON c.category = CASE WHEN a.income < 20000 THEN 'Low Salary'
                           WHEN a.income <= 50000 THEN 'Average Salary'
                           ELSE 'High Salary' END
  GROUP BY c.category;
  ```

  Same scaffold-then-`LEFT JOIN` idea as [Set 2 Q2](set-02-joins-and-aggregation.md#q2--students-and-examinations) — **when a required row doesn't exist in the data, generate it and join to it.**

---

## Q7 — Employees Whose Manager Left the Company

🔗 **[leetcode.com/problems/employees-whose-manager-left-the-company](https://leetcode.com/problems/employees-whose-manager-left-the-company/description/)**

**Problem:** Find employees with **salary under 30000** whose **manager has left the company** — the manager's `employee_id` no longer appears in the table. Order by `employee_id`.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Employees;
CREATE TABLE Employees (
    employee_id INT PRIMARY KEY,
    name        VARCHAR(30),
    manager_id  INT,
    salary      INT
);

INSERT INTO Employees VALUES
(3,'Mila',9,60301),(12,'Antonella',NULL,31000),(13,'Emery',NULL,67084),
(1,'Kalel',11,21241),(9,'Mikaela',NULL,50937),(11,'Joziah',6,28485);
```

### ✅ Expected Output

| employee_id |
|-------------|
| 11 |

### 🎯 How to approach it

A **self anti-join** — find rows whose reference points at nothing. Three conditions, and all three are load-bearing:

1. **`salary < 30000`** — Kalel (21241) and Joziah (28485) qualify; everyone else is out on money alone.
2. **`manager_id IS NOT NULL`** — employees with no manager haven't been orphaned, they were never assigned one. Without this, Antonella (31000) would be excluded by salary anyway, but on other data the `NULL`s would flood in.
3. **The manager is gone** — `LEFT JOIN` the table to itself on `e.manager_id = m.employee_id`, then keep only rows where the match **failed**: `m.employee_id IS NULL`.

Kalel's manager is 11 (Joziah), who is still on the payroll — so Kalel is filtered out. Joziah's manager is 6, who appears nowhere in the table. **Only employee 11 survives.**

### 💡 Solution

```sql
SELECT e.employee_id
FROM Employees e
LEFT JOIN Employees m
    ON e.manager_id = m.employee_id
WHERE e.salary < 30000
  AND e.manager_id IS NOT NULL
  AND m.employee_id IS NULL
ORDER BY e.employee_id;
```

### 🧠 Explanation

- **`LEFT JOIN … WHERE right_table.key IS NULL` is the anti-join idiom.** The join tries to find a manager row; when it can't, every `m.*` column comes back `NULL`, and testing the *joined key* for `NULL` is how you ask "did the match fail?" Same pattern as [Set 1 Q8](set-01-easy-basics.md#q8--customer-who-visited-but-did-not-make-any-transactions).
- **This is the one place where a `WHERE` on the right table is correct** — and it's not a contradiction of the [Q4](#q4--product-price-at-a-given-date) rule. Filtering the right table for *values* undoes the outer join; filtering it for `IS NULL` is *interrogating* the outer join's result. The first is a bug, the second is the point.
- **`m.employee_id IS NULL` rather than `m.name IS NULL`** — test the joined key, which is guaranteed non-`NULL` in any real row. A nullable column would be ambiguous: `NULL` could mean "no match" or "matched a row with a `NULL` name".
- **`NOT EXISTS` is the cleaner equivalent** and often faster, since it can stop at the first match instead of materialising joined rows:

  ```sql
  SELECT employee_id FROM Employees e
  WHERE e.salary < 30000 AND e.manager_id IS NOT NULL
    AND NOT EXISTS (SELECT 1 FROM Employees m WHERE m.employee_id = e.manager_id)
  ORDER BY employee_id;
  ```

- ⚠️ **`NOT IN` is the trap here.** `manager_id NOT IN (SELECT employee_id FROM Employees)` looks equivalent but returns **zero rows** if the subquery contains a single `NULL` — because `x NOT IN (1, NULL)` evaluates to `NULL`, never `TRUE`. `employee_id` is a primary key so it's safe in this schema, but **`NOT IN` against a nullable column is one of SQL's nastiest silent failures**. Use `NOT EXISTS`.

---

## Q8 — Exchange Seats

🔗 **[leetcode.com/problems/exchange-seats](https://leetcode.com/problems/exchange-seats/description/)**

**Problem:** Swap seats for every **pair of consecutive students**. If the number of students is **odd**, the last one keeps their seat. Order by `id`.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Seat;
CREATE TABLE Seat (
    id      INT PRIMARY KEY,
    student VARCHAR(30)
);

INSERT INTO Seat VALUES (1,'Abbot'),(2,'Doris'),(3,'Emerson'),(4,'Green'),(5,'Jeames');
```

### ✅ Expected Output

| id | student |
|----|---------|
| 1 | Doris |
| 2 | Abbot |
| 3 | Green |
| 4 | Emerson |
| 5 | Jeames |

### 🎯 How to approach it

**Don't move the students — renumber their seats and re-sort.** Trying to physically swap rows leads to self-joins and temp tables; changing each row's `id` and letting `ORDER BY` do the shuffling is far simpler.

The mapping is pure arithmetic on `id`:

- **Even `id` → `id - 1`.** Seat 2 becomes 1, seat 4 becomes 3.
- **Odd `id` → `id + 1`.** Seat 1 becomes 2, seat 3 becomes 4.
- **Odd `id` that is also the last seat → unchanged.** Seat 5 has no partner, so it stays 5. Without this branch it would become 6 and the output would have a seat that doesn't exist.

Branch order matters: **the odd-and-last case must be tested first**, because seat 5 also satisfies "odd", and `CASE` takes the first match. Reorder those two branches and Jeames lands in seat 6.

### 💡 Solution

```sql
SELECT CASE
           WHEN id % 2 = 1 AND id = (SELECT MAX(id) FROM Seat) THEN id
           WHEN id % 2 = 0 THEN id - 1
           ELSE id + 1
       END AS id,
       student
FROM Seat
ORDER BY id;
```

### 🧠 Explanation

- **`CASE` evaluates top-down and stops at the first `TRUE`** — which is why the odd-tail guard sits at the top. This is the ordering sensitivity that [Q2](#q2--triangle-judgement) didn't have to worry about.
- **The even count needs no special handling.** With four seats, no odd id is the maximum, so the first branch never fires and every seat pairs off. Verified on a 4-row table: `1→B, 2→A, 3→D, 4→C` — correct, and the guard costs nothing.
- **`(SELECT MAX(id) FROM Seat)` is an uncorrelated scalar subquery** — it doesn't reference the outer row, so MySQL evaluates it once and reuses the constant. `(SELECT COUNT(*) FROM Seat)` works identically *because the problem guarantees ids are continuous from 1*. **`MAX(id)` is the safer of the two**: it stays correct if a seat is ever deleted, while `COUNT(*)` would then point at the wrong row.
- **`ORDER BY id` sorts by the *computed* alias, not the original column.** MySQL resolves `ORDER BY` against the `SELECT` list, which is what makes the renumbering actually rearrange the output. That's also why `ORDER BY` can see aliases while `WHERE` cannot — `ORDER BY` runs last.
- 💡 **The `LEAD`/`LAG` alternative** keeps ids fixed and moves the names instead: `COALESCE(IF(id % 2 = 1, LEAD(student) OVER (ORDER BY id), LAG(student) OVER (ORDER BY id)), student)`. The `COALESCE` handles the odd tail — `LEAD` returns `NULL` past the last row, so the student falls back to themselves. Elegant, and it avoids the `MAX(id)` subquery entirely.

---

## Q9 — Movie Rating

🔗 **[leetcode.com/problems/movie-rating](https://leetcode.com/problems/movie-rating/description/)**

**Problem:** Report two things in one result set:
1. The **user who rated the most movies** — ties broken by lexicographically smaller name.
2. The **movie with the highest average rating in February 2020** — ties broken by lexicographically smaller title.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS MovieRating;
DROP TABLE IF EXISTS Movies;
DROP TABLE IF EXISTS Users;

CREATE TABLE Movies (
    movie_id INT PRIMARY KEY,
    title    VARCHAR(30)
);
INSERT INTO Movies VALUES (1,'Avengers'),(2,'Frozen 2'),(3,'Joker');

CREATE TABLE Users (
    user_id INT PRIMARY KEY,
    name    VARCHAR(30)
);
INSERT INTO Users VALUES (1,'Daniel'),(2,'Monica'),(3,'Maria'),(4,'James');

CREATE TABLE MovieRating (
    movie_id   INT,
    user_id    INT,
    rating     INT,
    created_at DATE,
    PRIMARY KEY (movie_id, user_id)
);
INSERT INTO MovieRating VALUES
(1,1,3,'2020-01-12'),(1,2,4,'2020-02-11'),(1,3,2,'2020-02-12'),(1,4,1,'2020-01-01'),
(2,1,5,'2020-02-17'),(2,2,2,'2020-02-01'),(2,3,2,'2020-03-01'),
(3,1,3,'2020-02-22'),(3,2,4,'2020-02-25');
```

### ✅ Expected Output

| results |
|---------|
| Daniel |
| Frozen 2 |

### 🎯 How to approach it

**Two completely unrelated questions glued into one result set.** There's no join between them, no shared logic — just two independent top-1 queries stacked with `UNION ALL`. Recognising that immediately is most of the work; trying to answer both in one pass leads nowhere good.

**Part 1 — busiest rater.** Daniel and Monica both rated 3 movies. The tie-break is alphabetical, so `ORDER BY COUNT(*) DESC, name ASC LIMIT 1` → **Daniel**.

**Part 2 — best February movie.** Restrict to February, then average per movie: Avengers `(4+2)/2 = 3.0`, Frozen 2 `(5+2)/2 = 3.5`, Joker `(3+4)/2 = 3.5`. Frozen 2 and Joker tie, and `'Frozen 2' < 'Joker'` alphabetically → **Frozen 2**.

**Both tie-breaks are the point of the problem.** Drop the secondary `ORDER BY` and MySQL returns whichever row its plan happens to reach first — which will pass the sample and fail the hidden tests.

### 💡 Solution

```sql
(SELECT u.name AS results
 FROM Users u
 JOIN MovieRating m
     ON u.user_id = m.user_id
 GROUP BY u.user_id, u.name
 ORDER BY COUNT(*) DESC, u.name ASC
 LIMIT 1)

UNION ALL

(SELECT m.title
 FROM Movies m
 JOIN MovieRating mr
     ON m.movie_id = mr.movie_id
 WHERE created_at >= '2020-02-01'
   AND created_at <  '2020-03-01'
 GROUP BY m.movie_id, m.title
 ORDER BY AVG(mr.rating) DESC, m.title ASC
 LIMIT 1);
```

### 🧠 Explanation

- **The parentheses are mandatory.** Without them, MySQL attaches the trailing `ORDER BY … LIMIT 1` to the *combined* result instead of the second branch, and you get one row total. Wrapping each branch scopes its own `ORDER BY` and `LIMIT` — this is the single most common way this query is written wrong.
- ⚠️ **`UNION ALL`, not `UNION` — and here it's a correctness issue, not just performance.** If the top rater's name happened to equal the top movie's title, `UNION` would deduplicate them into one row and the answer would be incomplete. Verified with two identical values:

  | Operator | Rows returned |
  |---|---|
  | `UNION ALL` | **2** ✅ |
  | `UNION` | 1 ❌ |

  A user named "Joker" is unlikely but not impossible, and **the query shouldn't depend on that**.
- **`ORDER BY COUNT(*) DESC` works without selecting the count.** `ORDER BY` runs after `GROUP BY`, so aggregates are available whether or not they appear in the `SELECT` list — same principle as `HAVING` in [Set 3 Q6](set-03-window-functions-and-first-rows.md#q6--classes-with-at-least-5-students).
- **`GROUP BY u.user_id, u.name` groups by the key, not the label** — two users named "Daniel" would otherwise merge and win a contest neither entered.
- **`created_at >= '2020-02-01' AND created_at < '2020-03-01'` is the right way to write a month filter.** The half-open range keeps the column bare so an index on `created_at` can be used, and it stays correct if the column ever becomes a `DATETIME` — `BETWEEN '2020-02-01' AND '2020-02-29'` silently drops everything after midnight on the 29th. `LIKE '2020-02%'` and `MONTH(created_at) = 2` both work here but wrap the column in a function or a cast, forcing a full scan.
- 💡 **The output column is named `results` from the first branch alone.** `UNION` takes its column names from the first `SELECT` and ignores the rest — which is why the second branch's `m.title` needs no alias.

---

## Q10 — Restaurant Growth

🔗 **[leetcode.com/problems/restaurant-growth](https://leetcode.com/problems/restaurant-growth/description/)**

**Problem:** For every day from the **7th day onward**, report the **7-day rolling total** and **rolling average** of customer spend, rounded to 2 decimals. Order by `visited_on`.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Customer;
CREATE TABLE Customer (
    customer_id INT,
    name        VARCHAR(30),
    visited_on  DATE,
    amount      INT,
    PRIMARY KEY (customer_id, visited_on)
);

INSERT INTO Customer VALUES
(1,'Jhon','2019-01-01',100),(2,'Daniel','2019-01-02',110),(3,'Jade','2019-01-03',120),
(4,'Khaled','2019-01-04',130),(5,'Winston','2019-01-05',110),(6,'Elvis','2019-01-06',140),
(7,'Anna','2019-01-07',150),(8,'Maria','2019-01-08',80),(9,'Jaze','2019-01-09',110),
(1,'Jhon','2019-01-10',130),(3,'Jade','2019-01-10',150);
```

### ✅ Expected Output

| visited_on | amount | average_amount |
|------------|--------|----------------|
| 2019-01-07 | 860 | 122.86 |
| 2019-01-08 | 840 | 120.00 |
| 2019-01-09 | 840 | 120.00 |
| 2019-01-10 | 1000 | 142.86 |

### 🎯 How to approach it

**Aggregate to one row per day first, then window over days.** This ordering is not optional — 2019-01-10 has *two* customers (Jhon 130 and Jade 150). Windowing over raw rows would treat them as two separate days, and every rolling total from that point on would be wrong.

So: a CTE collapsing `Customer` to `(visited_on, daily_total)`, then a window over that.

The frame is the new idea. `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` means **this row plus the 6 before it** — seven rows, which given one row per day is seven days. `6`, not `7`, because the current row is one of the seven; this off-by-one is the same inclusive-range arithmetic as the 30-day window in [Set 3 Q4](set-03-window-functions-and-first-rows.md#q4--user-activity-for-the-past-30-days-i).

Finally, **days 1–6 have an incomplete window** — 2019-01-03 would report a 3-day total, not a 7-day one — so they must be dropped. `ROW_NUMBER()` over the same ordering numbers the days, and `ranking >= 7` keeps only the days with a full window behind them. Filtering on `visited_on >= <7th date>` would need to know the date in advance; the row number derives it.

### 💡 Solution

```sql
WITH daily_sums AS (
    SELECT visited_on,
           SUM(amount) AS amount
    FROM Customer
    GROUP BY visited_on
)
SELECT visited_on,
       amount,
       ROUND(average_amount, 2) AS average_amount
FROM (
    SELECT visited_on,
           SUM(amount) OVER (ORDER BY visited_on ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS amount,
           AVG(amount) OVER (ORDER BY visited_on ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS average_amount,
           ROW_NUMBER() OVER (ORDER BY visited_on) AS ranking
    FROM daily_sums
) t
WHERE ranking >= 7;
```

### 🧠 Explanation

- **Trace 2019-01-07:** days 01-01 through 01-07 sum to `100+110+120+130+110+140+150 = 860`, and `860 / 7 = 122.857…` → **122.86**. On 01-10 the window has slid to 01-04 … 01-10, where 01-10 contributes `130 + 150 = 280` as a single day — giving 1000. **That day is the reason the CTE exists.**
- **`ROUND(average_amount, 2)` runs in the outer query, after the window.** Rounding inside the window would round each day's contribution before averaging. Note that `amount` needs no rounding — it's a sum of integers.
- **The window functions can't be filtered in the same `SELECT`**, hence the extra nesting level: `ROW_NUMBER()` is computed inside, `WHERE ranking >= 7` applied outside. The rule from [Set 3](set-03-window-functions-and-first-rows.md) again — `WHERE` runs before window functions exist.
- ⚠️ **`ROWS` counts rows; it does not count days.** They coincide here only because the restaurant had customers every single day. **Delete one day from the data and the seven "rows" span eight calendar dates** — verified on a table missing 2019-01-07:

  | visited_on | `ROWS BETWEEN 6 PRECEDING` | `RANGE BETWEEN INTERVAL 6 DAY PRECEDING` |
  |---|---|---|
  | 2019-01-08 | **700** — 7 rows spanning 8 days ❌ | **600** — a true 7-day window ✅ |

  MySQL 8 supports `RANGE BETWEEN INTERVAL 6 DAY PRECEDING AND CURRENT ROW` on a date-ordered window, which measures the frame in **calendar time** rather than row count and is therefore gap-proof. LeetCode's data has no gaps so `ROWS` is accepted — but **"7-day moving average" almost always means calendar days**, and a closed-for-the-holidays day is exactly when the two answers diverge and nobody notices.
- 💡 **A gap-proof frame doesn't fix the `ranking >= 7` filter**, which still assumes 7 rows = 7 days. With real gaps you'd filter on `visited_on >= (SELECT MIN(visited_on) FROM daily_sums) + INTERVAL 6 DAY` instead — the honest way to say "only days with a full week behind them."

---

## 🧠 Set 4 — Patterns Worth Memorising

| Pattern | Trigger phrase | Tool |
|---|---|---|
| Tag each row with its group's size | "if they belong to only one…" | `COUNT(*) OVER (PARTITION BY k)` — **no** `ORDER BY` |
| Running total | "cumulative", "until the limit is reached" | `SUM(x) OVER (ORDER BY k ROWS UNBOUNDED PRECEDING)` |
| Rolling N-period window | "7-day moving average" | `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` (or `RANGE … INTERVAL 6 DAY`) |
| Look ahead more than one row | "three times consecutively" | `LEAD(col, 1)`, `LEAD(col, 2)` |
| Latest value as of a date | "the price on 2019-08-16" | Filter `<= D`, `ROW_NUMBER() … ORDER BY d DESC`, take rank 1 |
| Supply a default for missing entities | "products with no change cost 10" | `LEFT JOIN` from the full entity list + `COALESCE(x, default)` |
| Force a bucket that has no rows | "every category, even if zero" | `UNION ALL` of one-row aggregates — **not** `GROUP BY` |
| Row-level classification | "return Yes/No per row" | `CASE WHEN … THEN … ELSE … END` |
| Reference points at a deleted row | "whose manager left" | `LEFT JOIN … WHERE m.key IS NULL`, or `NOT EXISTS` |
| Pairwise swap over a sequence | "swap every two consecutive" | Arithmetic on `id` in a `CASE`, then `ORDER BY` the alias |
| Two unrelated answers, one output | "report the top user **and** the top movie" | Parenthesised branches joined by `UNION ALL` |
| Aggregate before windowing | any daily metric with repeat entities | CTE with `GROUP BY day`, then the window over that |

### Five mistakes that cost the most submissions

1. **`GROUP BY` to produce a zero-count bucket.** It can only emit groups that exist — the empty category vanishes, and `IFNULL` can't rescue a row that was never created. Use `UNION ALL`. (Q6)
2. **Filtering the right table of a `LEFT JOIN` in `WHERE`.** `WHERE r.ranking = 1` drops the very rows the default was meant for. Put it in `ON`. (Q4)
3. **Trusting the default window frame.** `ORDER BY` alone means `RANGE`, which lumps tied rows together; a running total over non-unique keys silently over-counts. Write `ROWS` explicitly. (Q5)
4. **`ROWS` where you meant calendar days.** Seven rows equal seven days only if no day is missing. (Q10)
5. **Missing parentheses or `UNION` instead of `UNION ALL` on a two-part answer.** The first collapses two `LIMIT 1` branches into one; the second silently dedupes a legitimate duplicate. (Q9)

---

<div align="center">

**[Continue to Set 5 — Strings, Regex & Set Operations ➡](set-05-strings-regex-and-set-operations.md)**

[⬅ Back to Course Home](../README.md) · [Set 1](set-01-easy-basics.md) · [Set 2](set-02-joins-and-aggregation.md) · [Set 3](set-03-window-functions-and-first-rows.md) · [Set 5](set-05-strings-regex-and-set-operations.md) · [Interview Problem Sets](../interview-questions/) · [LeetCode SQL 50 study plan ↗](https://leetcode.com/studyplan/top-sql-50/)

</div>
