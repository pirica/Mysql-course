# 🧩 SQL Interview Practice — Problem Set 3

> **Part 3: Problem Set** — six *classic* SQL interview problems that show up in almost every screening round (Nth-highest salary, finding & deleting duplicates, manager comparisons, department averages, and month-over-month growth).
> Each problem follows the same flow: **Question → Dataset → Expected Output → Solution(s) → Explanation.**

Every problem is self-contained: run the *Dataset Creation* block, then run the solution to reproduce the expected output. All queries target **MySQL 8.0+**.

---

## 📋 Contents

| # | Problem | Core Concept |
|---|---------|--------------|
| 1 | [Find the Nth highest salary](#q1--find-the-nth-highest-salary) | `DISTINCT` + `LIMIT/OFFSET`, `DENSE_RANK()` |
| 2 | [Find duplicate records based on email](#q2--find-duplicate-records-based-on-email) | `GROUP BY` + `HAVING COUNT` |
| 3 | [Delete duplicate records of the same email](#q3--delete-duplicate-records-of-the-same-email) | Self-join `DELETE`, keep-one logic |
| 4 | [Employees earning more than their manager](#q4--employees-earning-more-than-their-manager) | Self-join on `manager_id` |
| 5 | [Employees earning more than the department average](#q5--employees-earning-more-than-the-department-average) | Correlated subquery / window `AVG()` |
| 6 | [Month-over-Month sales growth](#q6--month-over-month-sales-growth) | `LAG()` window function + growth % |

---

## Q1 — Find the Nth highest salary

**Question:** Find the **2nd highest** salary from the `employees` table (the solution must generalize to any *N*).

### 🗄️ Dataset

```sql
CREATE TABLE Employee (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(50),
    salary INT
);

INSERT INTO Employee VALUES
(1,'Ravi Kumar',90000),
(2,'Anita Sharma',85000),
(3,'Vikram Singh',85000),
(4,'Neha Joshi',70000),
(5,'Suresh Rao',60000);
```

### ✅ Expected Output

| second_highest_salary |
|-----------------------|
| 85000 |

### 💡 Solution 1 — Scalar subquery + `LIMIT / OFFSET` (NULL-safe)

```sql
SELECT (
    SELECT salary
    FROM Employee
    ORDER BY salary DESC
    LIMIT 1 OFFSET 1        -- OFFSET = N-1
) AS second_highest_salary;
```

### 💡 Solution 2 — Scalar subquery + `ROW_NUMBER()` (NULL-safe, generalizes)

```sql
SELECT (
    SELECT salary
    FROM (
        SELECT
            salary,
            ROW_NUMBER() OVER (ORDER BY salary DESC) AS ranking
        FROM Employee
    ) t
    WHERE ranking = 2      -- ranking = N
) AS second_highest_salary;
```

### 🧠 Explanation

- **Why wrap it in an outer `SELECT (...)`?** This is the key trick for **NULL handling**. A bare `... LIMIT 1 OFFSET 1` returns an *empty result set* when there is no Nth salary (e.g. asking for the 10th highest in a 5-row table). Wrapping it as a **scalar subquery** guarantees exactly one row back — the value if it exists, otherwise a clean `NULL`. Interviewers love this because "return NULL, not zero rows" is the correct, defensive behaviour.
- **`LIMIT 1 OFFSET 1`:** after sorting salaries high→low (`90000, 85000, 85000, 70000, 60000`), we skip the first row (`OFFSET 1`) and take one (`LIMIT 1`) → `85000`. For the *Nth* highest, use `OFFSET N-1`. (Add `DISTINCT` inside if the top salary itself could be tied and you want the 2nd *distinct* value.)
- **`ROW_NUMBER()` version** makes *N* an explicit parameter (`WHERE ranking = N`) and is easy to reason about: it numbers salaries `1, 2, 3, …` descending, so `ranking = 2` is the runner-up. Here `90000` is unique, so row 2 is `85000` — matching the expected output. Swap to `DENSE_RANK()` if you specifically want the *Nth distinct* salary and the higher salaries may contain ties.
- Edge case worth stating out loud: if *N* exceeds the number of rows, both scalar-subquery forms return **`NULL`** rather than erroring or returning nothing.

---

## Q2 — Find duplicate records based on email

**Question:** Find all email addresses that appear more than once in the `users` table.

### 🗄️ Dataset

```sql
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    user_name VARCHAR(50),
    email VARCHAR(100)
);

INSERT INTO users VALUES
(1,'Ravi Kumar','ravi@example.com'),
(2,'Anita Sharma','anita@example.com'),
(3,'Ravi K','ravi@example.com'),
(4,'Neha Joshi','neha@example.com'),
(5,'Anita S','anita@example.com');
```

### ✅ Expected Output

| email | occurrences |
|--------------------|-------------|
| ravi@example.com | 2 |
| anita@example.com | 2 |

### 💡 Solution

```sql
SELECT
    email,
    COUNT(*) AS occurrences
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

### 🧠 Explanation

- **`GROUP BY email`** collapses all rows sharing an email into one group; **`COUNT(*)`** tells you how big each group is.
- **`HAVING COUNT(*) > 1`** keeps only groups with duplicates. Recall the ordering: `WHERE` filters *rows before* grouping, `HAVING` filters *groups after* aggregation — since `COUNT(*)` is an aggregate, it can only live in `HAVING`.
- `ravi@example.com` (rows 1 & 3) and `anita@example.com` (rows 2 & 5) each appear twice; `neha@example.com` appears once and is dropped.
- 💡 If you need the actual duplicate *rows* (not just the emails), wrap this as a subquery: `SELECT * FROM users WHERE email IN (SELECT email FROM users GROUP BY email HAVING COUNT(*) > 1)`.

---

## Q3 — Delete duplicate records of the same email

**Question:** Delete duplicate users so that each email is kept **only once** — retain the row with the *smallest* `user_id`.

### 🗄️ Dataset

*(same `users` table as Q2)*

```sql
INSERT INTO users VALUES
(1,'Ravi Kumar','ravi@example.com'),
(2,'Anita Sharma','anita@example.com'),
(3,'Ravi K','ravi@example.com'),
(4,'Neha Joshi','neha@example.com'),
(5,'Anita S','anita@example.com');
```

### ✅ Expected Output — remaining rows after the `DELETE`

| user_id | user_name | email |
|---------|--------------|-------------------|
| 1 | Ravi Kumar | ravi@example.com |
| 2 | Anita Sharma | anita@example.com |
| 4 | Neha Joshi | neha@example.com |

### 💡 Solution 1 — CTE + `ROW_NUMBER()` (MySQL 8.0+)

```sql
WITH duplicate_ranking AS (
    SELECT
        user_id,
        email,
        ROW_NUMBER() OVER (PARTITION BY email ORDER BY user_id) AS ranking
    FROM users
)
DELETE FROM users
WHERE user_id IN (
    SELECT user_id
    FROM duplicate_ranking
    WHERE ranking > 1
);
```

### 💡 Solution 2 — Self-join `DELETE`

```sql
DELETE u1
FROM users u1
JOIN users u2
    ON  u1.email  = u2.email
    AND u1.user_id > u2.user_id;
```

### 🧠 Explanation

- **CTE approach (preferred):** `ROW_NUMBER() OVER (PARTITION BY email ORDER BY user_id)` numbers rows `1, 2, …` within each email, smallest id first. Row `1` is the keeper; anything with `ranking > 1` is a duplicate. The CTE lists those ids and the `DELETE` removes them (rows 3 and 5). This reads top-to-bottom and makes the keep-one rule explicit.
- ⚠️ **MySQL quirk that this sidesteps:** you can't `DELETE FROM users WHERE user_id IN (SELECT ... FROM users)` that references the *same* table directly. Naming the subquery as a **CTE** (`duplicate_ranking`) — like a derived table — gets MySQL to materialize it first, so the delete is allowed. That's why the ranking lives in a `WITH` block rather than inline.
- **Self-join approach:** joining `users` to itself on the same email while requiring `u1.user_id > u2.user_id` matches every row that has *a smaller-id twin with the same email*. Those larger-id rows (3 and 5) are exactly the duplicates; the smallest-id copy of each email has no smaller twin, so it survives. `DELETE u1` targets only the left alias.
- **Keep-one direction:** both keep the *smallest* id. Flip to `ranking` on `ORDER BY user_id DESC` (or `u1.user_id < u2.user_id` in the self-join) to keep the *largest* / most recent instead.

---

## Q4 — Employees earning more than their manager

**Question:** Find employees whose salary is higher than their direct manager's salary.

### 🗄️ Dataset

```sql
CREATE TABLE companyEmployee (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(50),
    salary INT,
    manager_id INT   -- references companyEmployee.emp_id; NULL for the CEO
);

INSERT INTO companyEmployee VALUES
(1,'Ravi Kumar',60000,NULL),
(2,'Anita Sharma',80000,1),
(3,'Vikram Singh',50000,1),
(4,'Neha Joshi',70000,2),
(5,'Suresh Rao',90000,2);
```

### ✅ Expected Output

| employee | employee_salary | manager | manager_salary |
|----------|-----------------|---------|----------------|
| Anita Sharma | 80000 | Ravi Kumar | 60000 |
| Suresh Rao | 90000 | Anita Sharma | 80000 |

### 💡 Solution

```sql
SELECT
    e.emp_name  AS employee,
    e.salary    AS employee_salary,
    m.emp_name  AS manager,
    m.salary    AS manager_salary
FROM companyEmployee e
JOIN companyEmployee m
    ON e.manager_id = m.emp_id
WHERE e.salary > m.salary;
```

### 🧠 Explanation

- This is a **self-join on a hierarchical (self-referencing) table**. Alias `e` is the *employee*, alias `m` is the *manager*; joining `e.manager_id = m.emp_id` pairs each employee with their boss's row.
- **`WHERE e.salary > m.salary`** keeps only pairs where the subordinate out-earns the manager.
- Walking the data: Anita (80k) reports to Ravi (60k) ✅; Suresh (90k) reports to Anita (80k) ✅; Vikram (50k < 60k) and Neha (70k < 80k) don't qualify. Ravi (the CEO) has `manager_id = NULL`, so the `JOIN` drops him entirely — correct, since he has no manager to compare against.
- 💡 Note the `INNER JOIN` naturally excludes the top-level manager. If you wanted to *list* the CEO too, you'd switch to a `LEFT JOIN` — but for "earns more than manager," excluding them is exactly right.

---

## Q5 — Employees earning more than the department average

**Question:** Find employees whose salary is above the average salary **of their own department**.

### 🗄️ Dataset

```sql
CREATE TABLE dept_staff (
    emp_id INT PRIMARY KEY,
    emp_name VARCHAR(50),
    department VARCHAR(30),
    salary INT
);

INSERT INTO dept_staff VALUES
(1,'Ravi Kumar','Sales',50000),
(2,'Anita Sharma','Sales',70000),
(3,'Vikram Singh','IT',90000),
(4,'Neha Joshi','IT',60000),
(5,'Suresh Rao','IT',60000);
```

### ✅ Expected Output

| emp_name | department | salary | dept_avg |
|----------|------------|--------|----------|
| Anita Sharma | Sales | 70000 | 60000.0000 |
| Vikram Singh | IT | 90000 | 70000.0000 |

### 💡 Solution 1 — Correlated subquery

```sql
SELECT
    e.emp_name,
    e.department,
    e.salary,
    (SELECT AVG(salary) FROM dept_staff d WHERE d.department = e.department) AS dept_avg
FROM dept_staff e
WHERE e.salary > (
    SELECT AVG(salary)
    FROM dept_staff d
    WHERE d.department = e.department
);
```

### 💡 Solution 2 — Window function (single pass)

```sql
SELECT emp_name, department, salary, dept_avg
FROM (
    SELECT
        emp_name,
        department,
        salary,
        AVG(salary) OVER (PARTITION BY department) AS dept_avg
    FROM dept_staff
) t
WHERE salary > dept_avg;
```

### 🧠 Explanation

- **Correlated subquery:** for each outer row, the inner query recomputes `AVG(salary)` *filtered to that row's department* (`d.department = e.department`). It's "correlated" because the inner query depends on the outer row — it re-runs per row.
- **Window-function version** is cleaner and usually faster: `AVG(salary) OVER (PARTITION BY department)` computes each department's average once and attaches it to every row *without collapsing rows* (unlike `GROUP BY`). We can't put a window function in `WHERE` directly, so we compute it in a subquery `t` and filter in the outer query.
- The math: Sales avg = (50k + 70k)/2 = **60k** → only Anita (70k) is above. IT avg = (90k + 60k + 60k)/3 = **70k** → only Vikram (90k) is above. Ravi, Neha, and Suresh sit at or below their department average and are excluded.
- `AVG()` returns a decimal in MySQL, hence `60000.0000`.

---

## Q6 — Month-over-Month sales growth

**Question:** For each month, compute total sales and the **month-over-month growth percentage** versus the previous month.

### 🗄️ Dataset

```sql
CREATE TABLE monthly_sales (
    sale_id INT PRIMARY KEY,
    sale_date DATE,
    amount INT
);

INSERT INTO monthly_sales VALUES
(1,'2024-01-05',6000),
(2,'2024-01-20',4000),
(3,'2024-02-10',15000),
(4,'2024-03-15',12000),
(5,'2024-04-08',10000),
(6,'2024-04-22',8000),
(7,'2024-05-15',9000),
(8,'2024-06-10',9000);
```

### ✅ Expected Output

| sales_month | total_sales | prev_month_sales | growth_pct |
|-------------|-------------|------------------|------------|
| 2024-01 | 10000 | NULL | NULL |
| 2024-02 | 15000 | 10000 | 50.00 |
| 2024-03 | 12000 | 15000 | -20.00 |
| 2024-04 | 18000 | 12000 | 50.00 |
| 2024-05 | 9000 | 18000 | -50.00 |
| 2024-06 | 9000 | 9000 | 0.00 |

### 💡 Solution

```sql
WITH monthly AS (
    SELECT
        DATE_FORMAT(sale_date, '%Y-%m') AS sales_month,
        SUM(amount) AS total_sales
    FROM monthly_sales
    GROUP BY DATE_FORMAT(sale_date, '%Y-%m')
)
SELECT
    sales_month,
    total_sales,
    LAG(total_sales) OVER (ORDER BY sales_month) AS prev_month_sales,
    ROUND(
        (total_sales - LAG(total_sales) OVER (ORDER BY sales_month))
        / LAG(total_sales) OVER (ORDER BY sales_month) * 100
    , 2) AS growth_pct
FROM monthly
ORDER BY sales_month;
```

### 🧠 Explanation

- **Step 1 — bucket by month:** the `monthly` CTE aggregates daily rows into monthly totals using `DATE_FORMAT(sale_date, '%Y-%m')` (January's two rows, 6000 + 4000, roll up to **10000**).
- **Step 2 — reach back one month:** `LAG(total_sales) OVER (ORDER BY sales_month)` pulls the *previous* month's total onto the current row. The first month has no predecessor, so `LAG` returns `NULL` — which correctly propagates to a `NULL` growth for January.
- **Step 3 — growth formula:** `(current − previous) / previous × 100`, rounded to 2 decimals. Feb = (15000 − 10000)/10000 × 100 = **+50.00%**; Mar = (12000 − 15000)/15000 × 100 = **−20.00%**; Apr rebounds to **+50.00%**; May halves to **−50.00%**; June is flat versus May, so growth is exactly **0.00%** (a useful reminder that `0` is a real value, distinct from January's `NULL` "no prior month").
- ⚠️ **Gotchas to mention:** (1) dividing by the previous month risks a **division-by-zero** if a month had `0` sales — guard with `NULLIF(prev, 0)` in production. (2) This only works when every month is present; **missing months** silently compare non-adjacent periods. For gap-safe growth, join against a generated calendar of months.

---

<div align="center">

**More problem sets coming soon.** ⭐ the repo to follow along.

[⬅ Back to Course Home](../README.md) · [Problem Set 1](problem-set-01.md) · [Problem Set 2](problem-set-02.md)

</div>
