# 🎁 Bonus Set 1 — The Rolling Sum Pattern

> **Patterns the LeetCode SQL 50 doesn't teach properly.**
> Every problem follows the same flow: **Problem → Schema → Sample Input → Expected Output → Approach → Solution → Explanation.**

The SQL 50 contains exactly **one** rolling-window problem — [Restaurant Growth](../leetcode-problems/set-04-advanced-windows-and-conditional-logic.md#q10--restaurant-growth) — and it's the gentlest possible version: one row per day already, no gaps in the calendar, no per-customer split. Real reporting is never that tidy. **Rolling sums are the most-requested metric in analytics work** (trailing revenue, 7-day active users, 30-day spend per account) and the three ways they go wrong are the three questions below.

The set is a deliberate progression:

1. **Q1** establishes the base shape — *aggregate to one row per period first, then window over that*.
2. **Q2** adds `PARTITION BY` for a per-entity rolling sum, and reveals that `ROWS` counts **rows**, not days.
3. **Q3** fixes exactly that, with `RANGE` over a generated calendar — the version you'd actually ship.

All queries target **MySQL 8.0+** and were executed against a local MySQL 9.6 server. The Expected Output blocks are real query output, and every "here's what goes wrong instead" table is real output too.

---

## 📋 Contents

| # | Problem | Core Concept |
|---|---------|--------------|
| 1 | [3-Day Rolling Sum of Sales](#q1--3-day-rolling-sum-of-sales-for-each-day) | Aggregate first, then window — `ROWS BETWEEN 2 PRECEDING` |
| 2 | [5-Day Rolling Purchases Per Customer](#q2--5-day-rolling-purchase-amount-per-customer) | `PARTITION BY` + a frame; rows ≠ days |
| 3 | [7-Day Calendar Rolling Sales With Gaps](#q3--7-day-calendar-rolling-sales-with-missing-dates) | `RANGE … INTERVAL` + a recursive date spine |

---

## Q1 — 3-Day Rolling Sum of Sales For Each Day

**Problem:** The `daily_sales` table records **one row per store per day**. For each calendar day, report the day's total sales and a **3-day rolling sum** covering that day and the two before it.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS daily_sales;
CREATE TABLE daily_sales (
    sale_id   INT PRIMARY KEY,
    store     VARCHAR(10),
    sale_date DATE,
    sales     INT
);

INSERT INTO daily_sales VALUES
(1,'North','2024-01-01',100),(2,'South','2024-01-01',50),
(3,'North','2024-01-02',200),
(4,'North','2024-01-03',150),(5,'South','2024-01-03',50),
(6,'North','2024-01-04',300),
(7,'North','2024-01-05',250);
```

### ✅ Expected Output

| sale_date | sales | rolling_sum |
|-----------|-------|-------------|
| 2024-01-01 | 150 | 150 |
| 2024-01-02 | 200 | 350 |
| 2024-01-03 | 200 | 550 |
| 2024-01-04 | 300 | 700 |
| 2024-01-05 | 250 | 750 |

### 🎯 How to approach it

**Two steps, and the order is not negotiable: collapse to one row per day, then slide a window across those days.**

The table has **seven rows but only five days** — 2024-01-01 and 2024-01-03 each have a North and a South sale. That mismatch is the entire trap. A window function operates on *rows*, so if you window straight over `daily_sales`, "the previous two rows" sometimes means "the previous two stores on the same day" rather than "the previous two days".

So the CTE does the collapsing:

```
raw rows (7)                    daily totals (5)
North 01-01 100  ┐
South 01-01  50  ┘─────────────► 2024-01-01  150
North 01-02 200  ──────────────► 2024-01-02  200
North 01-03 150  ┐
South 01-03  50  ┘─────────────► 2024-01-03  200
...
```

Then `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` sums the current day plus the two before it. **`2`, not `3`** — the current row is one of the three, the same inclusive-count arithmetic as every window in [Set 4](../leetcode-problems/set-04-advanced-windows-and-conditional-logic.md).

### 💡 Solution

```sql
WITH daily_rolling_sales AS (
    SELECT sale_date,
           SUM(sales) AS sales
    FROM daily_sales
    GROUP BY sale_date
)
SELECT *,
       SUM(sales) OVER (ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS rolling_sum
FROM daily_rolling_sales;
```

### 🧠 Explanation

- **Trace 2024-01-03:** the window covers 01-01, 01-02, 01-03 → `150 + 200 + 200 = 550`. By 01-05 the window has slid to 01-03…01-05 → `200 + 300 + 250 = 750`. The window's *width* is fixed at three; its *position* moves with every row.
- ⚠️ **Skip the `GROUP BY` and the answer is silently wrong.** Windowing over the raw seven rows:

  | sale_date | sales | rolling over raw rows | correct |
  |---|---|:---:|:---:|
  | 2024-01-01 | 100 | 100 | — |
  | 2024-01-01 | 50 | **150** | 150 |
  | 2024-01-02 | 200 | **350** | 350 |
  | 2024-01-03 | 150 | **400** | — |
  | 2024-01-03 | 50 | **400** | 550 ❌ |
  | 2024-01-04 | 300 | **500** | 700 ❌ |
  | 2024-01-05 | 250 | **600** | 750 ❌ |

  That's verified output. Seven rows come back instead of five, and from 01-03 onward every total is wrong, because the frame is counting *store-days* while the question asks about *days*. **The moment the grain of your table is finer than the grain of your metric, aggregate before you window.**
- **The first two rows have incomplete frames and that's fine.** 2024-01-01 has no preceding rows, so its "3-day" sum is just its own 150. MySQL doesn't pad or error, it uses whatever rows exist. If a partial window is misleading for your report, filter them out with a `ROW_NUMBER() >= 3` guard, exactly as [Restaurant Growth](../leetcode-problems/set-04-advanced-windows-and-conditional-logic.md#q10--restaurant-growth) does.
- **`SELECT *` works here because the CTE has only two columns.** In production, name them — a later `ALTER TABLE` shouldn't change your report's shape.
- 💡 **Why not `GROUP BY` and window in one query?** Because window functions are evaluated *after* `GROUP BY`, so you can legally write `SUM(SUM(sales)) OVER (ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)` with a `GROUP BY sale_date` and no CTE at all. It's valid, it's one pass, and the doubled `SUM` reads like a typo to everyone who maintains it later. **The CTE version is the same plan and a fraction of the confusion.**

---

## Q2 — 5-Day Rolling Purchase Amount Per Customer

**Problem:** For each purchase, report a **rolling sum of the last five purchases for that customer**, including the current one. Customers must not affect each other's totals.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS customer_purchases;
CREATE TABLE customer_purchases (
    purchase_id     INT PRIMARY KEY,
    customer_id     INT,
    purchase_date   DATE,
    purchase_amount INT
);

INSERT INTO customer_purchases VALUES
(1,1,'2024-03-01',10),(2,1,'2024-03-02',20),(3,1,'2024-03-03',30),(4,1,'2024-03-04',40),
(5,1,'2024-03-05',50),(6,1,'2024-03-06',60),(7,1,'2024-03-07',70),
(8,2,'2024-03-02',100),(9,2,'2024-03-05',200),(10,2,'2024-03-09',300);
```

### ✅ Expected Output

| purchase_id | customer_id | purchase_date | purchase_amount | rolling_sum |
|---|---|---|---|---|
| 1 | 1 | 2024-03-01 | 10 | 10 |
| 2 | 1 | 2024-03-02 | 20 | 30 |
| 3 | 1 | 2024-03-03 | 30 | 60 |
| 4 | 1 | 2024-03-04 | 40 | 100 |
| 5 | 1 | 2024-03-05 | 50 | 150 |
| 6 | 1 | 2024-03-06 | 60 | 200 |
| 7 | 1 | 2024-03-07 | 70 | 250 |
| 8 | 2 | 2024-03-02 | 100 | 100 |
| 9 | 2 | 2024-03-05 | 200 | 300 |
| 10 | 2 | 2024-03-09 | 300 | 600 |

### 🎯 How to approach it

**Same frame as Q1, plus one clause: `PARTITION BY customer_id`.**

`PARTITION BY` restarts the window at every customer boundary. Customer 2's first purchase sees an empty frame behind it, not the tail of customer 1's history. That is the whole difference between a per-customer metric and a meaningless one.

Watch customer 1's window fill and then slide. Purchases 1 to 5 accumulate (10, 30, 60, 100, 150) because the frame isn't full yet. From purchase 6 the window is saturated at five and starts **dropping from the back**: `20+30+40+50+60 = 200`, then `30+40+50+60+70 = 250`. **A rolling sum stops growing once the window fills** — if your numbers keep climbing forever, you've written a cumulative total by accident.

No `GROUP BY` is needed here: the grain of the table is already one row per purchase, and the metric is per purchase.

### 💡 Solution

```sql
SELECT *,
       SUM(purchase_amount) OVER (
           PARTITION BY customer_id
           ORDER BY purchase_date
           ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
       ) AS rolling_sum
FROM customer_purchases;
```

### 🧠 Explanation

- ⚠️ **Drop `PARTITION BY` and customers bleed into each other.** Verified output with the partition removed:

  | customer_id | purchase_date | amount | no `PARTITION BY` | correct |
  |---|---|---|:---:|:---:|
  | 1 | 2024-03-03 | 30 | **160** ❌ | 60 |
  | 2 | 2024-03-02 | 100 | **130** ❌ | 100 |
  | 2 | 2024-03-09 | 300 | **680** ❌ | 600 |

  Customer 1's 03-03 total picks up customer 2's 100, and customer 2's very first purchase starts at 130 because customer 1's earlier rows sit inside its frame. The query still returns ten tidy-looking rows, which is what makes this failure dangerous.
- ⚠️ **`ROWS` counts purchases, not days — so "5-day" is only accurate when there's one purchase per customer per day.** Customer 1 buys daily, so five rows genuinely are five days. **Customer 2 buys on the 2nd, 5th, and 9th**, and their three purchases span eight calendar days:

  | customer 2 | amount | `ROWS 4 PRECEDING` (5 purchases) | `RANGE INTERVAL 4 DAY` (5 days) |
  |---|---|:---:|:---:|
  | 2024-03-02 | 100 | 100 | 100 |
  | 2024-03-05 | 200 | 300 | 300 |
  | 2024-03-09 | 300 | **600** | **500** |

  Verified. On 03-09 the `ROWS` frame reaches all the way back to 03-02, seven days earlier, and includes a purchase that a true five-day window excludes. **Decide which one the question means before you write the frame.** "Last 5 transactions" is `ROWS`. "Last 5 days" is `RANGE … INTERVAL`. They only agree on dense, one-row-per-day data.
- **`ORDER BY purchase_date` inside `OVER()` orders the frame, not the output.** The result rows came back grouped by customer only because that's how the plan read them. Add a real `ORDER BY customer_id, purchase_date` to the query if the output order matters.
- **Ties on `purchase_date` would make `ROWS` non-deterministic** — two purchases the same day have no defined order, so which one lands "before" the other can change between runs. `ORDER BY purchase_date, purchase_id` makes it stable. This is the frame-level cousin of the tie-breaking issue in [Set 3 Q1](../leetcode-problems/set-03-window-functions-and-first-rows.md#q1--immediate-food-delivery-ii).
- 💡 **Swap `SUM` for `AVG` and you have a rolling average; swap for `COUNT` and it's rolling frequency.** The frame is the pattern — the aggregate is just which question you're asking of it.

---

## Q3 — 7-Day Calendar Rolling Sales With Missing Dates

**Problem:** `sales_with_gaps` has **no row at all on days with no sales**. Report a **true 7-day calendar rolling sum** for **every** day between the first and last sale, including the days that are missing from the table.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS sales_with_gaps;
CREATE TABLE sales_with_gaps (
    sale_date DATE PRIMARY KEY,
    sales     INT
);

INSERT INTO sales_with_gaps VALUES
('2024-01-01',100),('2024-01-02',200),('2024-01-05',300),
('2024-01-08',400),('2024-01-09',500),('2024-01-12',600);
```

Six rows spanning twelve days. **2024-01-03, 04, 06, 07, 10 and 11 don't exist.**

### ✅ Expected Output

| sales_date | sales | rolling_sales |
|------------|-------|---------------|
| 2024-01-01 | 100 | 100 |
| 2024-01-02 | 200 | 300 |
| 2024-01-03 | 0 | 300 |
| 2024-01-04 | 0 | 300 |
| 2024-01-05 | 300 | 600 |
| 2024-01-06 | 0 | 600 |
| 2024-01-07 | 0 | 600 |
| 2024-01-08 | 400 | 900 |
| 2024-01-09 | 500 | 1200 |
| 2024-01-10 | 0 | 1200 |
| 2024-01-11 | 0 | 1200 |
| 2024-01-12 | 600 | 1500 |

### 🎯 How to approach it

**Two independent problems wearing one costume**, and separating them is the insight:

1. **The frame must measure calendar time, not row count.** `ROWS BETWEEN 6 PRECEDING` counts six *rows* back. With only six rows in the whole table, that reaches across the entire twelve-day span and degenerates into a plain running total. `RANGE BETWEEN INTERVAL 6 DAY PRECEDING AND CURRENT ROW` measures in **days**, so MySQL includes exactly the rows whose date falls in `[current − 6 days, current]` — however many that turns out to be.

2. **Days with no sales still need output rows.** `RANGE` fixes the arithmetic but cannot invent 2024-01-03: a window function only ever emits rows that already exist. So the calendar has to be generated and the sales joined onto it.

Compare the two on the raw table, before any spine is added:

| sale_date | sales | `ROWS 6 PRECEDING` | `RANGE INTERVAL 6 DAY` |
|---|---|:---:|:---:|
| 2024-01-01 | 100 | 100 | 100 |
| 2024-01-02 | 200 | 300 | 300 |
| 2024-01-05 | 300 | 600 | 600 |
| 2024-01-08 | 400 | **1000** ❌ | **900** ✅ |
| 2024-01-09 | 500 | **1500** ❌ | **1200** ✅ |
| 2024-01-12 | 600 | **2100** ❌ | **1500** ✅ |

Verified output. `ROWS` ends at 2100 — the sum of everything — because six preceding rows is the whole table. `RANGE` correctly drops 2024-01-01 once the window passes it.

**Note what that table proves: `RANGE` alone already gives the right number for every date that exists.** The recursive CTE isn't fixing the arithmetic — it's supplying the six missing calendar rows, so the report has a value for 01-03, 01-06, 01-10 and the rest.

**Building the spine** is a recursive CTE in two halves: an anchor (`MIN(sale_date)`) and a recursive member that keeps adding a day until it reaches `MAX(sale_date)`. Then `LEFT JOIN` the sales on, `COALESCE` the misses to `0`, and window over the result.

### 💡 Solution

```sql
WITH RECURSIVE missing_sales_date AS (
    SELECT MIN(sale_date) AS sales_date
    FROM sales_with_gaps

    UNION ALL

    SELECT DATE_ADD(sales_date, INTERVAL 1 DAY)
    FROM missing_sales_date
    WHERE sales_date < (SELECT MAX(sale_date) FROM sales_with_gaps)
)
SELECT m.sales_date,
       COALESCE(s.sales, 0) AS sales,
       SUM(COALESCE(s.sales, 0)) OVER (
           ORDER BY m.sales_date
           RANGE BETWEEN INTERVAL 6 DAY PRECEDING AND CURRENT ROW
       ) AS rolling_sales
FROM missing_sales_date m
LEFT JOIN sales_with_gaps s
    ON m.sales_date = s.sale_date;
```

### 🧠 Explanation

- **Trace 2024-01-08:** the frame is `[2024-01-02, 2024-01-08]`, which contains sales of 200 (01-02), 300 (01-05) and 400 (01-08) → **900**. The 100 from 01-01 has just fallen out of the window, which is exactly the behaviour `ROWS` failed to produce.
- **Trace 2024-01-10, a day with no sales:** the frame `[01-04, 01-10]` still holds 300 + 400 + 500 = **1200**. The row exists, the metric is meaningful, and a dashboard plotting this gets a flat line rather than a hole. **That's what the spine buys you.**
- **`COALESCE(s.sales, 0)` appears twice on purpose** — once to display the day's sales as `0`, once inside the `SUM`. The second is defensive rather than strictly required, since `SUM` ignores `NULL`s anyway, but writing it makes the intent explicit and keeps the two columns consistent.
- **`UNION ALL`, never `UNION`, in a recursive CTE.** MySQL requires it, and `UNION` would try to deduplicate the growing result on every iteration.
- ⚠️ **`cte_max_recursion_depth` defaults to 1000, and a date spine hits it fast.** A three-year calendar fails outright:

  ```
  ERROR 3636 (HY000): Recursive query aborted after 1001 iterations.
  Try increasing @@cte_max_recursion_depth to a larger value.
  ```

  Verified: 2021-01-01 to 2024-01-01 is 1096 days and errors at the default; `SET SESSION cte_max_recursion_depth = 100000;` makes it return all 1096. **Twelve days is safe, three years is not** — and this is the single thing most likely to break this query when you move it to real data. A permanent `calendar` table avoids the problem entirely and is what most warehouses keep for exactly this reason.
- ⚠️ **`RANGE … INTERVAL` is fussy about its `ORDER BY`, and the errors are precise:**

  | Mistake | MySQL says |
  |---|---|
  | `ORDER BY sale_date, sales` | `ERROR 3587: … requires exactly one deterministic ORDER BY expression, of numeric or temporal type` |
  | `ORDER BY sales` (an `INT`) with an `INTERVAL` bound | `ERROR 3589: … ORDER BY expression of numeric type, INTERVAL bound value not allowed` |

  Both verified. **One column, and it must be a date or time type** for `INTERVAL` bounds to be legal. A numeric `ORDER BY` takes a plain number instead: `RANGE BETWEEN 6 PRECEDING AND CURRENT ROW`.
- **The `LEFT JOIN` direction matters.** The generated calendar must be on the **left** so unmatched days survive. Reverse it and you're back to six rows.
- 💡 **This is the production-grade shape of every "daily metric over time" report.** The spine also fixes the sibling bug nobody notices: without it, a chart drawn straight from the table connects 01-02 to 01-05 with a straight line, implying sales happened on the 3rd and 4th. **Missing rows are zeros, and a time series has to say so.**

---

## 🧠 Bonus Set 1 — Patterns Worth Memorising

| Pattern | Trigger phrase | Tool |
|---|---|---|
| N-period rolling sum | "3-day rolling total" | `SUM(x) OVER (ORDER BY d ROWS BETWEEN N-1 PRECEDING AND CURRENT ROW)` |
| Table grain finer than metric grain | multiple rows per day/entity | `GROUP BY` in a CTE **first**, then window over it |
| Rolling metric per entity | "per customer", "per product" | Add `PARTITION BY entity_id` to the `OVER()` |
| Rolling over **calendar** time | "last 7 days", not "last 7 rows" | `RANGE BETWEEN INTERVAL N DAY PRECEDING AND CURRENT ROW` |
| Rolling over **event** count | "last 5 transactions" | `ROWS BETWEEN N-1 PRECEDING AND CURRENT ROW` |
| Days missing from the data | gaps in a time series | `WITH RECURSIVE` date spine + `LEFT JOIN` + `COALESCE(x, 0)` |
| Cumulative rather than rolling | "running total to date" | `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` |

### The five mistakes this set exists to prevent

1. **Windowing over rows that are finer than your metric.** Two stores on one day makes "the last 3 rows" mean something other than "the last 3 days". Aggregate in a CTE first. (Q1)
2. **Forgetting `PARTITION BY`.** The output still looks plausible — every customer just silently inherits the previous customer's tail. (Q2)
3. **Assuming `ROWS` means days.** It means rows. They coincide only on dense, one-row-per-day data, and diverge the moment someone skips a day. (Q2, Q3)
4. **Expecting a window function to invent missing dates.** It can only emit rows that exist; a gap needs a generated calendar. (Q3)
5. **Shipping a recursive date spine without raising `cte_max_recursion_depth`.** It works on your twelve-day test and dies at 1001 days in production. (Q3)

---

<div align="center">

**More bonus sets coming.** ⭐ the repo to follow along.

[⬅ Back to Course Home](../README.md) · [LeetCode SQL 50](../leetcode-problems/) · [Interview Problem Sets](../interview-questions/) · [Window Functions guide](../docs/05-window-functions.md) · [Recursive CTE guide](../docs/12-recursive-cte.md)

</div>
