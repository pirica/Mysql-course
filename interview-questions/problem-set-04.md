# 🧩 SQL Interview Practice — Problem Set 4

> **Part 4: Problem Set** — five problems that round out the pattern library: a real **multi-table join**, the classic **median** (both overall and per-group, since MySQL has no `MEDIAN()`), the flagship **gaps-and-islands** pattern with **two** solution methods, and a **win-percentage** report built on a self-`UNION`.
> Each problem follows the same flow: **Question → Dataset → Expected Output → Solution(s) → Explanation.**

Every problem is self-contained: run the *Dataset* block, then run the solution to reproduce the expected output. All queries target **MySQL 8.0+** and were executed and verified.

---

## 📋 Contents

| # | Problem | Core Concept |
|---|---------|--------------|
| 1 | [Revenue per customer across four tables](#q1--revenue-per-customer-across-four-tables) | Multi-table `LEFT JOIN` + aggregation |
| 2 | [Median salary of all employees](#q2--median-salary-of-all-employees) | Median via `ROW_NUMBER` + `COUNT` |
| 3 | [Median salary per department](#q3--median-salary-per-department) | Partitioned median |
| 4 | [Collapse consecutive check-ins into streaks](#q4--collapse-consecutive-check-ins-into-streaks) | Gaps-and-islands (2 methods) |
| 5 | [Matches played & winning % per team](#q5--matches-played--winning--per-team) | Self-`UNION ALL` + conditional agg |

---

## Q1 — Revenue per customer across four tables

**Question:** Compute the **total revenue** (Σ quantity × unit price) for every customer, joining `customers → orders → order_items → products`. Customers with no orders must still appear, with revenue `0`.

### 🗄️ Dataset

```sql
CREATE TABLE customers   (id INT PRIMARY KEY, name VARCHAR(20));
CREATE TABLE orders      (order_id INT PRIMARY KEY, customer_id INT);
CREATE TABLE order_items (order_id INT, product_id INT, qty INT);
CREATE TABLE products    (product_id INT PRIMARY KEY, price INT);

INSERT INTO customers   VALUES (1,'Alice'),(2,'Bob'),(3,'Carol');
INSERT INTO orders      VALUES (101,1),(102,1),(103,2);
INSERT INTO order_items VALUES (101,1,2),(101,2,1),(102,1,1),(103,3,3);
INSERT INTO products    VALUES (1,100),(2,50),(3,200);
```

### ✅ Expected Output

| name  | total_revenue |
|-------|---------------|
| Bob   | 600 |
| Alice | 350 |
| Carol | 0 |

> Revenue = quantity × price.

### 💡 Solution

```sql
SELECT c.name,
  COALESCE(SUM(oi.qty * p.price), 0) AS total_revenue
FROM customers c
LEFT JOIN orders o       ON c.id          = o.customer_id
LEFT JOIN order_items oi ON o.order_id    = oi.order_id
LEFT JOIN products p     ON oi.product_id = p.product_id
GROUP BY c.id, c.name
ORDER BY total_revenue DESC;
```

### 🧠 Explanation

- **Chain the joins in dependency order:** each table hangs off a key in the one before it (`customer → order → item → product`). The line item's money lives across two tables — `order_items.qty` and `products.price` — so revenue is `SUM(oi.qty * p.price)`.
- **Why every join is `LEFT`:** the moment one link is `INNER`, a customer with no orders (Carol) vanishes from the result. `LEFT JOIN` all the way down keeps her; her item/price columns are `NULL`, `SUM` of nothing is `NULL`, and **`COALESCE(..., 0)`** turns that into a clean `0`.
- **Group by the customer, not the join grain:** the join explodes to one row per line item; `GROUP BY c.id` collapses it back. Alice = order 101 (2×100 + 1×50 = 250) + order 102 (1×100 = 100) = **350**; Bob = order 103 (3×200 = **600**).
- 💡 Group by `c.id` (the key), including `c.name` for display — safe even under `ONLY_FULL_GROUP_BY` because `name` is functionally dependent on the primary key `id`.

---

## Q2 — Median salary of all employees

**Question:** Find the **median** salary across all employees. Handle both odd and even total counts (even → average of the two middle values). MySQL has no built-in `MEDIAN()`.

### 🗄️ Dataset

```sql
CREATE TABLE emp2 (name VARCHAR(10), dept VARCHAR(10), salary INT);

INSERT INTO emp2 VALUES
('A','Eng',60000),('B','Eng',70000),('C','Eng',80000),('D','Eng',90000),
('E','Sales',50000),('F','Sales',60000),('G','Sales',70000);
```

### ✅ Expected Output

| median_salary |
|---------------|
| 70000.0000 |

> Sorted salaries → `50000, 60000, 60000, 70000, 70000, 80000, 90000`. There are 7 values, so the middle (4th) value = **70000**.

### 💡 Solution

```sql
SELECT AVG(salary) AS median_salary
FROM (
  SELECT salary,
    ROW_NUMBER() OVER (ORDER BY salary) AS ranking,
    COUNT(*)     OVER ()                AS total
  FROM emp2
) t
WHERE ranking IN (FLOOR((total + 1) / 2),
                  FLOOR((total + 2) / 2));
```

### 🧠 Explanation

- **Rank + size in one pass:** `ROW_NUMBER() OVER (ORDER BY salary)` numbers salaries `1..n` from lowest to highest; `COUNT(*) OVER ()` (empty `OVER()` = whole table) tags every row with the total row count `total`.
- **The middle-position formula handles both parities.** The two middle ranks are `FLOOR((total+1)/2)` and `FLOOR((total+2)/2)`:
  - **Odd** (`total = 7`, here): both evaluate to `4` → the single middle row, `70000`.
  - **Even** (`total = 6`): they give `3` and `4` → the two middle rows, which `AVG` then averages.
- **`AVG` finishes the job:** for an odd count it averages one row (a no-op); for an even count it averages the two middles. One formula, no `CASE` on parity.
- 💡 On MySQL 8 there is **no** `PERCENTILE_CONT` / `MEDIAN` (unlike Postgres/Oracle), so this `ROW_NUMBER` + `COUNT` pattern is the standard, portable way to compute a median.

---

## Q3 — Median salary per department

**Question:** Same median logic as Q2, but computed **within each department** — just add a partition.

### 🗄️ Dataset

Reuses the `emp2` table from Q2.

### ✅ Expected Output

| dept  | median_salary |
|-------|---------------|
| Eng   | 75000.0000 |
| Sales | 60000.0000 |

> `Eng → 60000, 70000, 80000, 90000` → (70000 + 80000) / 2 = **75000**.
> `Sales → 50000, 60000, 70000` → **60000**.

### 💡 Solution

```sql
SELECT dept, AVG(salary) AS median_salary
FROM (
  SELECT dept, salary,
    ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary) AS ranking,
    COUNT(*)     OVER (PARTITION BY dept)                 AS total
  FROM emp2
) t
WHERE ranking IN (FLOOR((total + 1) / 2),
                  FLOOR((total + 2) / 2))
GROUP BY dept
ORDER BY dept;
```

### 🧠 Explanation

- **`PARTITION BY dept` is the only change from Q2.** It restarts the numbering *and* the count inside each department, so `ranking` and `total` are now per-department. The identical middle-position filter then finds each department's median independently.
- **Even vs odd, per group:** Eng has 4 salaries (even) → the filter keeps ranks 2 and 3 (`70000, 80000`), averaged to **75000**. Sales has 3 (odd) → it keeps rank 2 only (`60000`).
- **The outer `GROUP BY dept`** is what averages the (one or two) middle rows *per department* into a single median value each.
- 💡 This "add a `PARTITION BY`" step is the general recipe for turning any whole-table window calculation into a per-group one — median, running totals, ranks, and more.

---

## Q4 — Collapse consecutive check-ins into streaks

**Question:** A gym logs each member's visit dates. Collapse each run of **consecutive** calendar days into a single streak row: `streak_start`, `streak_end`, and `streak_length`. A one-day visit is a streak of length 1.

### 🗄️ Dataset

```sql
CREATE TABLE member_checkins (member_id VARCHAR(5), visit_date DATE);

INSERT INTO member_checkins VALUES
('M1','2024-03-01'),('M1','2024-03-02'),('M1','2024-03-03'),
('M1','2024-03-06'),('M1','2024-03-07'),
('M1','2024-03-10'),
('M2','2024-03-01'),('M2','2024-03-02');
```

### ✅ Expected Output

| member_id | streak_start | streak_end | streak_length |
|-----------|--------------|------------|---------------|
| M1 | 2024-03-01 | 2024-03-03 | 3 |
| M1 | 2024-03-06 | 2024-03-07 | 2 |
| M1 | 2024-03-10 | 2024-03-10 | 1 |
| M2 | 2024-03-01 | 2024-03-02 | 2 |

### 💡 Solution 1 — `date − ROW_NUMBER()` (the classic trick)

```sql
WITH ranking AS (
  SELECT member_id, visit_date,
    ROW_NUMBER() OVER (PARTITION BY member_id ORDER BY visit_date) AS rn
  FROM member_checkins
),
grouped AS (
  SELECT member_id, visit_date,
    DATE_SUB(visit_date, INTERVAL rn DAY) AS grp
  FROM ranking
)
SELECT member_id,
  MIN(visit_date) AS streak_start,
  MAX(visit_date) AS streak_end,
  COUNT(*)        AS streak_length
FROM grouped
GROUP BY member_id, grp
ORDER BY member_id, streak_start;
```

### 💡 Solution 2 — `LAG()` + running sum of "gap" flags

```sql
WITH marked AS (
  SELECT member_id, visit_date,
    CASE WHEN DATEDIFF(visit_date,
             LAG(visit_date) OVER (PARTITION BY member_id ORDER BY visit_date)) = 1
         THEN 0 ELSE 1 END AS is_new_streak
  FROM member_checkins
),
grouped AS (
  SELECT member_id, visit_date,
    SUM(is_new_streak) OVER (PARTITION BY member_id ORDER BY visit_date) AS grp
  FROM marked
)
SELECT member_id,
  MIN(visit_date) AS streak_start,
  MAX(visit_date) AS streak_end,
  COUNT(*)        AS streak_length
FROM grouped
GROUP BY member_id, grp
ORDER BY member_id, streak_start;
```

### 🧠 Explanation

This is the **gaps-and-islands** pattern — the most-tested advanced SQL idea (streaks, consecutive logins, uptime windows all reduce to it). Both solutions assign a stable **group id** to each consecutive run, then `GROUP BY` it; they differ only in *how* they compute that id.

**Solution 1 — `date − ROW_NUMBER()`:** for a run of consecutive dates, `date − row_number` is **constant**. Number each member's visits `1, 2, 3, …` by date; consecutive dates advance in lockstep with the row number, so subtracting it lands every date in the run on the same anchor `grp`. A skipped day makes the date jump ahead of the row number, so `grp` shifts — a new island begins:

| visit_date | rn | grp (`date − rn` days) |
|------------|----|------------------------|
| 03-01 | 1 | 02-29 |
| 03-02 | 2 | 02-29 |
| 03-03 | 3 | 02-29 |
| 03-06 | 4 | 03-02 |
| 03-07 | 5 | 03-02 |
| 03-10 | 6 | 03-04 |

**Solution 2 — `LAG()` + running sum:** compare each date to the previous one with `LAG`. If the gap is exactly 1 day it's a continuation (`is_new_streak = 0`); otherwise it's a break (`= 1`). The very first row per member has `LAG = NULL` → `DATEDIFF` is `NULL` → not equal to 1 → flagged `1` (correctly starts streak #1). A **running `SUM` of those flags** then produces an increasing group id: `1,1,1,2,2,3` for M1.

- **Both then do the same finish:** each distinct `grp` is one streak — `MIN`/`MAX` give its span, `COUNT(*)` its length.
- **When to prefer which:** Solution 1 is shorter and cheaper but assumes a fixed step (consecutive *days*). Solution 2 (`LAG`) is more flexible — change the `DATEDIFF(...) = 1` condition to define "consecutive" however you like (e.g. within 7 days, or same-week), which the arithmetic trick can't express.
- 💡 The `grp` value is an internal artifact — group on it, but never expose it. For integer sequences (invoice numbers, IDs) use `value − ROW_NUMBER()` directly instead of `DATE_SUB`.

---

## Q5 — Matches played & winning % per team

**Question:** Each row of `cricket_matches` records `team1`, `team2`, and the `winner`. For **every** team, report matches played, matches won, and winning percentage — remembering that a team can appear in *either* column.

### 🗄️ Dataset

```sql
CREATE TABLE cricket_matches (team1 VARCHAR(30), team2 VARCHAR(30), winner VARCHAR(30));

INSERT INTO cricket_matches VALUES
('India','Australia','India'),
('New Zealand','India','New Zealand'),
('India','England','India'),
('South Africa','India','India'),
('Australia','England','England'),
('India','Pakistan','Pakistan'),
('New Zealand','Australia','Australia'),
('England','India','India'),
('Pakistan','South Africa','South Africa'),
('India','New Zealand','India');
```

### ✅ Expected Output

| team | matches_played | matches_won | winning_percentage |
|------|----------------|-------------|--------------------|
| India        | 7 | 5 | 71.43 |
| Australia    | 3 | 1 | 33.33 |
| England      | 3 | 1 | 33.33 |
| New Zealand  | 3 | 1 | 33.33 |
| Pakistan     | 2 | 1 | 50.00 |
| South Africa | 2 | 1 | 50.00 |

### 💡 Solution

```sql
WITH all_matches AS (
  SELECT team1 AS team, winner FROM cricket_matches
  UNION ALL
  SELECT team2 AS team, winner FROM cricket_matches
)
SELECT team,
  COUNT(*) AS matches_played,
  SUM(CASE WHEN team = winner THEN 1 ELSE 0 END) AS matches_won,
  ROUND(
    SUM(CASE WHEN team = winner THEN 1 ELSE 0 END) * 100 / COUNT(*),
    2
  ) AS winning_percentage
FROM all_matches
GROUP BY team
ORDER BY matches_played DESC, team;
```

### 🧠 Explanation

- **The core idea: normalize two team columns into one.** A team's match can sit in either `team1` or `team2`, so a single `GROUP BY team1` would miss half its games. `UNION ALL` stacks both columns into one `team` column — now each match contributes **two** rows (one per participant), which is exactly what "matches played per team" needs. `India` surfaces in 7 matches total across both columns → `matches_played = 7`.
- **Use `UNION ALL`, not `UNION`.** `UNION` would de-duplicate identical `(team, winner)` rows and undercount teams that had the same outcome twice. You want every appearance counted, so `UNION ALL`.
- **Conditional aggregation for wins:** `SUM(CASE WHEN team = winner THEN 1 ELSE 0 END)` counts only the rows where this team was the winner — a one-pass way to count a subset alongside the total.
- **Winning %:** `wins * 100 / played`, wrapped in `ROUND(..., 2)`. Multiplying by 100 *before* dividing keeps integer division from collapsing to 0; India = 5 × 100 / 7 = **71.43**.
- 💡 `ORDER BY matches_played DESC, team` gives the ranking-style output (busiest teams first, alphabetical within ties). In production, guard against teams with zero games via `NULLIF(COUNT(*), 0)` to avoid division-by-zero.

---

<div align="center">

**More problem sets coming soon.** ⭐ the repo to follow along.

[⬅ Back to Course Home](../README.md) · [Problem Set 1](problem-set-01.md) · [Problem Set 2](problem-set-02.md) · [Problem Set 3](problem-set-03.md)

</div>
