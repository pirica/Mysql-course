# 🧩 SQL Interview Practice — Problem Set 5

> **Part 5: Problem Set** — five problems on string parsing, streak detection, pivot/unpivot reshaping, and first/last-per-group aggregation.
> Each problem follows the same flow: **Question → Dataset → Expected Output → Solution → Explanation.**

Every problem is self-contained: run the *Dataset* block, then run the solution to reproduce the expected output. All queries target **MySQL 8.0+** and were executed and verified.

---

## 📋 Contents

| # | Problem | Core Concept |
|---|---------|--------------|
| 1 | [Extract name & email components](#q1--extract-name--email-components) | `SUBSTRING_INDEX` (positive & negative) |
| 2 | [Longest win streak per player](#q2--longest-win-streak-per-player) | Gaps-and-islands with reset |
| 3 | [Monthly sales pivot](#q3--monthly-sales-pivot) | Rows → columns (conditional agg) |
| 4 | [Student marks unpivot](#q4--student-marks-unpivot) | Columns → rows (`UNION ALL`) |
| 5 | [First & last order per customer](#q5--first--last-order-per-customer) | `MIN`/`MAX` per group |

---

## Q1 — Extract name & email components

**Question:** From free-text `full_name` and `email`, derive first name, last name, initials, the email username (before `@`), the domain (after `@`), and the top-level domain (after the last `.`).

### 🗄️ Dataset

```sql
CREATE TABLE users6 (id INT, full_name VARCHAR(40), email VARCHAR(50));

INSERT INTO users6 VALUES
(1,'Ravi Kumar','ravi.kumar@gmail.com'),
(2,'Anita S Sharma','anita@yahoo.co.in'),
(3,'Vikram Singh Rao','vikram_rao@company.org');
```

### ✅ Expected Output

| first_name | last_name | initials | email_user | email_domain | tld |
|------------|-----------|----------|------------|--------------|-----|
| Ravi   | Kumar  | RK | ravi.kumar | gmail.com   | com |
| Anita  | Sharma | AS | anita      | yahoo.co.in | in  |
| Vikram | Rao    | VR | vikram_rao | company.org | org |

### 💡 Solution

```sql
SELECT
  SUBSTRING_INDEX(full_name, ' ',  1)                   AS first_name,
  SUBSTRING_INDEX(full_name, ' ', -1)                   AS last_name,
  CONCAT(LEFT(SUBSTRING_INDEX(full_name, ' ',  1), 1),
         LEFT(SUBSTRING_INDEX(full_name, ' ', -1), 1))  AS initials,
  SUBSTRING_INDEX(email, '@',  1)                        AS email_user,
  SUBSTRING_INDEX(email, '@', -1)                        AS email_domain,
  SUBSTRING_INDEX(email, '.', -1)                        AS tld
FROM users6
ORDER BY id;
```

### 🧠 Explanation

- **`SUBSTRING_INDEX(str, delim, n)` is the whole toolkit here.** A **positive** `n` returns everything *before* the n-th delimiter (from the left); a **negative** `n` returns everything *after* the n-th-from-last delimiter. Point it at different delimiters and it does every split in this problem.
- **First vs last name:** `(full_name, ' ', 1)` grabs the first token; `(full_name, ' ', -1)` grabs the last. That's why `'Vikram Singh Rao'` → first `Vikram`, last `Rao` — the middle token is skipped by both.
- **Initials:** take `LEFT(..., 1)` of each parsed name and `CONCAT` them → `RK`, `AS`, `VR`.
- **Email split:** `(email, '@', 1)` = username, `(email, '@', -1)` = domain. **Negative counting is what makes `co.in` safe:** `(email, '.', -1)` returns only the part after the *last* dot (`in`), regardless of how many dots the domain has — a naive "split on the first dot" would break on `yahoo.co.in`.
- 💡 Caveat worth stating aloud: the first-token/last-token split is lossy for `First Middle Last` and compound surnames. To pull the **middle** token, nest the calls: `SUBSTRING_INDEX(SUBSTRING_INDEX(full_name,' ',2),' ',-1)`.

---

## Q2 — Longest win streak per player

**Question:** For each player, find the length of their longest run of consecutive wins (matches ordered by match number).

### 🗄️ Dataset

```sql
CREATE TABLE matches (player VARCHAR(20), match_no INT, result CHAR(1));

INSERT INTO matches VALUES
('Kohli',1,'W'),('Kohli',2,'W'),('Kohli',3,'L'),('Kohli',4,'W'),
('Kohli',5,'W'),('Kohli',6,'W'),('Kohli',7,'L'),('Kohli',8,'W'),
('Rohit',1,'L'),('Rohit',2,'W'),('Rohit',3,'W');
```

### ✅ Expected Output

| player | longest_win_streak |
|--------|--------------------|
| Kohli  | 3 |
| Rohit  | 2 |

### 💡 Solution

```sql
WITH wins AS (
  SELECT player, match_no,
    match_no - ROW_NUMBER() OVER (PARTITION BY player ORDER BY match_no) AS grp
  FROM matches
  WHERE result = 'W'
),
streaks AS (
  SELECT player, grp, COUNT(*) AS len
  FROM wins
  GROUP BY player, grp
)
SELECT player, MAX(len) AS longest_win_streak
FROM streaks
GROUP BY player
ORDER BY player;
```

### 🧠 Explanation

- **This is gaps-and-islands on the wins only.** First `WHERE result = 'W'` throws away losses, so only winning matches remain — but with holes in `match_no` where losses used to be.
- **`match_no − ROW_NUMBER()` labels each winning run.** Within a consecutive block of wins, both `match_no` and the row number increase by 1 in lockstep, so their difference (`grp`) stays constant. A loss creates a hole in `match_no` but not in the row number, so after the gap `grp` jumps to a new value — starting a new streak. For Kohli's wins at matches `1,2` then `4,5,6` then `8`:

  | match_no | row_number | grp |
  |----------|-----------|-----|
  | 1 | 1 | 0 |
  | 2 | 2 | 0 |
  | 4 | 3 | 1 |
  | 5 | 4 | 1 |
  | 6 | 5 | 1 |
  | 8 | 6 | 2 |

- **Two aggregations:** `COUNT(*)` per `grp` gives each streak's length (`2, 3, 1`), then `MAX(len)` per player picks the longest → Kohli **3**.
- 💡 Because we filter on wins first and use the *integer* `match_no` (not dates), this is the pure-integer form of the pattern — no `DATE_SUB` needed. Same idea powers "longest login streak", "max consecutive on-time deliveries", etc.

---

## Q3 — Monthly sales pivot

**Question:** Pivot the long sales table so each product is one row with a column per month (`jan`, `feb`, `mar`); months with no sales show `0`.

### 🗄️ Dataset

```sql
CREATE TABLE monthly_sales (product VARCHAR(5), sale_month VARCHAR(3), amount INT);

INSERT INTO monthly_sales VALUES
('A','Jan',100),('A','Feb',200),('B','Jan',50),('B','Mar',70);
```

### ✅ Expected Output

| product | jan | feb | mar |
|---------|-----|-----|-----|
| A | 100 | 200 | 0 |
| B | 50  | 0   | 70 |

### 💡 Solution

```sql
SELECT product,
  SUM(CASE WHEN sale_month = 'Jan' THEN amount ELSE 0 END) AS jan,
  SUM(CASE WHEN sale_month = 'Feb' THEN amount ELSE 0 END) AS feb,
  SUM(CASE WHEN sale_month = 'Mar' THEN amount ELSE 0 END) AS mar
FROM monthly_sales
GROUP BY product
ORDER BY product;
```

### 🧠 Explanation

- **Pivot = "rows into columns" via conditional aggregation.** MySQL has no `PIVOT` keyword, so the idiom is one aggregate per target column: `SUM(CASE WHEN month = 'X' THEN amount ELSE 0 END)`.
- **How each cell fills:** for product A, the `jan` column sums `amount` only on rows where `sale_month = 'Jan'` (100) and contributes `0` everywhere else; the `feb` column does the same for February (200); March has no A row, so its `CASE` is always `0`.
- **`GROUP BY product`** collapses each product's several month-rows into the single pivoted row. `ELSE 0` (rather than leaving `NULL`) gives clean zeros for absent months.
- 💡 This works when the columns are **known in advance**. For an unknown/large set of months you'd generate the `CASE` list dynamically (build the SQL string with `GROUP_CONCAT` + a prepared statement) — but in interviews the fixed-column version is what's expected.

---

## Q4 — Student marks unpivot

**Question:** The reverse of a pivot — turn a wide marks table (one column per subject) into tall `(student, subject, marks)` rows.

### 🗄️ Dataset

```sql
CREATE TABLE student_marks (student VARCHAR(10), math INT, science INT, english INT);

INSERT INTO student_marks VALUES ('Amit',90,85,80),('Bob',70,60,95);
```

### ✅ Expected Output

| student | subject | marks |
|---------|---------|-------|
| Amit | Math    | 90 |
| Amit | Science | 85 |
| Amit | English | 80 |
| Bob  | Math    | 70 |
| Bob  | Science | 60 |
| Bob  | English | 95 |

### 💡 Solution

```sql
SELECT student, 'Math'    AS subject, math    AS marks FROM student_marks
UNION ALL
SELECT student, 'Science',           science         FROM student_marks
UNION ALL
SELECT student, 'English',           english         FROM student_marks
ORDER BY student, FIELD(subject, 'Math', 'Science', 'English');
```

### 🧠 Explanation

- **Unpivot = "columns into rows."** MySQL has no `UNPIVOT`, so the idiom is one `SELECT` per source column, each emitting a literal subject label plus that column's value, stitched together with `UNION ALL`.
- **Why `UNION ALL`, not `UNION`:** `UNION` would deduplicate identical `(student, subject, marks)` rows — if two students genuinely scored the same, one would vanish. `UNION ALL` keeps every row, which is what you want.
- **Column names come from the first branch.** The output columns (`student, subject, marks`) are taken from the first `SELECT`; later branches just need matching positions and types, so their literals/columns don't need aliases.
- 💡 **`ORDER BY student, FIELD(subject, 'Math','Science','English')`** restores the original column order. Plain `ORDER BY subject` would sort alphabetically (English, Math, Science); `FIELD()` returns each value's position in the list, giving a custom sort that matches the source layout.

---

## Q5 — First & last order per customer

**Question:** For each customer, find the date of their first and last order.

### 🗄️ Dataset

```sql
CREATE TABLE orders5 (order_id INT, customer VARCHAR(5), order_date DATE);

INSERT INTO orders5 VALUES
(101,'C1','2024-01-05'),(102,'C1','2024-02-10'),(103,'C1','2024-03-01'),
(104,'C2','2024-01-20');
```

### ✅ Expected Output

| customer | first_order_date | last_order_date |
|----------|------------------|-----------------|
| C1 | 2024-01-05 | 2024-03-01 |
| C2 | 2024-01-20 | 2024-01-20 |

### 💡 Solution

```sql
SELECT customer,
  MIN(order_date) AS first_order_date,
  MAX(order_date) AS last_order_date
FROM orders5
GROUP BY customer
ORDER BY customer;
```

### 🧠 Explanation

- **`MIN`/`MAX` over a date column give the earliest and latest** order per customer in a single grouped pass — no self-join or window needed. C1's three orders collapse to `2024-01-05` … `2024-03-01`; C2's single order is both its first and last.
- **`GROUP BY customer`** defines the grain — one output row per customer.
- 💡 If you need the whole first/last **order row** (order id, amount, …) rather than just the dates, `MIN`/`MAX` alone won't do it — you'd rank with `ROW_NUMBER() OVER (PARTITION BY customer ORDER BY order_date)` for the first and `... ORDER BY order_date DESC` for the last, then keep `rn = 1`. Use the aggregate form when you only need the boundary *values*, the window form when you need the boundary *rows*.

---

<div align="center">

**More problem sets coming soon.** ⭐ the repo to follow along.

[⬅ Back to Course Home](../README.md) · [Problem Set 1](problem-set-01.md) · [Problem Set 2](problem-set-02.md) · [Problem Set 3](problem-set-03.md) · [Problem Set 4](problem-set-04.md)

</div>
