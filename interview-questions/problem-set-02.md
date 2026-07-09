# 🧩 SQL Interview Practice — Problem Set 2

> **Part 2: Problem Set** — five real-world query problems inspired by product companies (hotels, retail, healthcare, SaaS, social media).
> Each problem follows the same flow: **Question → Dataset → Expected Output → Solution(s) → Explanation.**

Every problem is self-contained: run the *Dataset Creation* block, then run the solution to reproduce the expected output. All queries target **MySQL 8.0+**.

---

## 📋 Contents

| # | Problem | Core Concept |
|---|---------|--------------|
| 1 | [Double-booked (overlapping) rooms](#q1--double-booked-overlapping-rooms) | Self-join + interval overlap logic |
| 2 | [Top-selling product per category](#q2--top-selling-product-per-category) | `ROW_NUMBER()`, CTE, per-group top-N |
| 3 | [Patients with multiple same-day appointments](#q3--patients-with-multiple-same-day-appointments) | `GROUP BY` + `HAVING` |
| 4 | [Customers who downgraded to free](#q4--customers-who-downgraded-to-free) | `LEAD()` window function |
| 5 | [Likes, comments & shares per post](#q5--likes-comments--shares-per-post) | Conditional aggregation + `LEFT JOIN` |

---

## Q1 — Double-booked (overlapping) rooms

**Question:** Find rooms that have overlapping (double-booked) bookings.

### 🗄️ Dataset

```sql
CREATE TABLE bookings (
    booking_id INT PRIMARY KEY,
    room_id INT,
    guest_name VARCHAR(50),
    check_in DATE,
    check_out DATE
);

INSERT INTO bookings VALUES
(1,101,'Ravi Kumar','2024-04-01','2024-04-05'),
(2,101,'Anita Sharma','2024-04-04','2024-04-08'),
(3,102,'Vikram Singh','2024-04-01','2024-04-03'),
(4,102,'Neha Joshi','2024-04-03','2024-04-06'),
(5,103,'Suresh Rao','2024-04-01','2024-04-02'),
(6,101,'Priya Nair','2024-04-10','2024-04-12');
```

### ✅ Expected Output

| room_id | booking_id_1 | guest_1 | booking_id_2 | guest_2 |
|---------|--------------|---------|--------------|---------|
| 101 | 1 | Ravi Kumar | 2 | Anita Sharma |

### 💡 Solution

```sql
SELECT
    b.room_id,
    b.booking_id  AS booking_id_1,
    b.guest_name  AS guest_1,
    b1.booking_id AS booking_id_2,
    b1.guest_name AS guest_2
FROM bookings b
JOIN bookings b1
    ON  b.room_id = b1.room_id
    AND b.booking_id < b1.booking_id
    AND b.check_in  < b1.check_out
    AND b.check_out > b1.check_in;
```

### 🧠 Explanation

- This is a **self-join**: the `bookings` table is joined to itself so we can compare *pairs* of bookings for the same room.
- **`b.room_id = b1.room_id`** restricts comparisons to bookings of the same room.
- **`b.booking_id < b1.booking_id`** is the key to avoiding duplicates. Without it you'd get each pair twice (A–B *and* B–A) plus every row matched against itself. Requiring the left id to be smaller keeps exactly one ordered pair.
- The two date conditions are the **classic interval-overlap test**: two ranges overlap when *each starts before the other ends* — `b.check_in < b1.check_out AND b.check_out > b1.check_in`. Booking 1 (Apr 1–5) and Booking 2 (Apr 4–8) overlap on Apr 4–5, so they surface; Booking 6 (Apr 10–12) touches neither and is excluded.

---

## Q2 — Top-selling product per category

**Question:** Find the top-selling product (by quantity) in each category.

### 🗄️ Dataset

```sql
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(50),
    category VARCHAR(30)
);

CREATE TABLE sales (
    sale_id INT PRIMARY KEY,
    product_id INT,
    quantity_sold INT,
    sale_date DATE
);

INSERT INTO products VALUES
(1,'Colgate Toothpaste','Personal Care'),
(2,'Dove Soap','Personal Care'),
(3,'Tata Salt','Grocery'),
(4,'Aashirvaad Atta','Grocery'),
(5,'Lays Chips','Snacks'),
(6,'Kurkure','Snacks');

INSERT INTO sales VALUES
(1,1,50,'2024-01-01'),
(2,2,80,'2024-01-02'),
(3,3,120,'2024-01-03'),
(4,4,150,'2024-01-04'),
(5,5,200,'2024-01-05'),
(6,6,90,'2024-01-06'),
(7,1,30,'2024-01-07'),
(8,4,60,'2024-01-08');
```

### ✅ Expected Output

| category | product_name | total_qty |
|---------------|--------------------|-----------|
| Personal Care | Colgate Toothpaste | 80 |
| Grocery | Aashirvaad Atta | 210 |
| Snacks | Lays Chips | 200 |

### 💡 Solution

```sql
WITH total_quantity_sold AS (
    SELECT
        p.category,
        p.product_name,
        SUM(s.quantity_sold) AS total_qty
    FROM products p
    JOIN sales s
        ON p.product_id = s.product_id
    GROUP BY p.category, p.product_name
)
SELECT category, product_name, total_qty
FROM (
    SELECT
        category,
        product_name,
        total_qty,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY total_qty DESC) AS ranking
    FROM total_quantity_sold
) t
WHERE ranking = 1
ORDER BY total_qty;
```

### 🧠 Explanation

- **Step 1 — aggregate:** the CTE `total_quantity_sold` sums `quantity_sold` per product. Products sold on multiple days (Colgate = 50 + 30 = 80, Aashirvaad = 150 + 60 = 210) are correctly totaled by grouping on `category, product_name`.
- **Step 2 — rank within each category:** `ROW_NUMBER() OVER (PARTITION BY category ORDER BY total_qty DESC)` numbers products `1, 2, 3…` starting fresh for each category, highest quantity first.
- **Step 3 — filter:** keeping `ranking = 1` returns the single best-seller per category — the standard **"top-N-per-group"** pattern.
- **`ROW_NUMBER` vs `RANK` vs `DENSE_RANK`:** `ROW_NUMBER` always yields one winner even on a tie (arbitrary tiebreak). If you want *all* products tied for the top, swap in `RANK()` and keep `= 1`.

---

## Q3 — Patients with multiple same-day appointments

**Question:** Find patients who booked more than one appointment on the same day.

### 🗄️ Dataset

```sql
CREATE TABLE appointments (
    appointment_id INT PRIMARY KEY,
    patient_id INT,
    doctor_id INT,
    appointment_date DATE
);

INSERT INTO appointments VALUES
(1,1,10,'2024-05-01'),
(2,1,11,'2024-05-01'),
(3,2,10,'2024-05-01'),
(4,3,12,'2024-05-02'),
(5,3,12,'2024-05-02'),
(6,4,13,'2024-05-03'),
(7,1,10,'2024-05-04');
```

### ✅ Expected Output

| patient_id | appointment_date | appointment_count |
|------------|------------------|-------------------|
| 1 | 2024-05-01 | 2 |
| 3 | 2024-05-02 | 2 |

### 💡 Solution

```sql
SELECT
    patient_id,
    appointment_date,
    COUNT(*) AS appointment_count
FROM appointments
GROUP BY patient_id, appointment_date
HAVING COUNT(*) > 1;
```

### 🧠 Explanation

- **`GROUP BY patient_id, appointment_date`** creates one group per *(patient, day)* combination — exactly the unit we want to count.
- **`COUNT(*)`** tallies how many appointments fall in each group.
- **`HAVING COUNT(*) > 1`** keeps only the groups with duplicates. Remember: `WHERE` filters *rows before* grouping, while **`HAVING` filters *groups after* aggregation** — you can't use an aggregate like `COUNT(*)` in a `WHERE` clause, which is why `HAVING` is required here.
- Patient 1 has two appointments on 2024-05-01 (different doctors) and patient 3 has two on 2024-05-02, so both surface; single-appointment days are dropped.

---

## Q4 — Customers who downgraded to free

**Question:** Find customers who downgraded from a paid plan (`BASIC`/`PRO`) to the free plan.

### 🗄️ Dataset

```sql
CREATE TABLE subscriptions (
    subscription_id INT PRIMARY KEY,
    customer_id INT,
    plan_type VARCHAR(20),   -- FREE, BASIC, PRO
    start_date DATE,
    end_date DATE
);

INSERT INTO subscriptions VALUES
(1,1,'BASIC','2024-01-01','2024-02-01'),
(2,1,'FREE','2024-02-01',NULL),
(3,2,'PRO','2024-01-01','2024-03-01'),
(4,2,'PRO','2024-03-01',NULL),
(5,3,'FREE','2024-01-01','2024-02-01'),
(6,3,'BASIC','2024-02-01',NULL);
```

### ✅ Expected Output

| customer_id | from_plan | to_plan | downgrade_date |
|-------------|-----------|---------|----------------|
| 1 | BASIC | FREE | 2024-02-01 |

### 💡 Solution

```sql
SELECT
    customer_id,
    from_plan,
    next_plan AS to_plan,
    downgrade_date
FROM (
    SELECT
        customer_id,
        plan_type AS from_plan,
        LEAD(plan_type)  OVER (PARTITION BY customer_id ORDER BY start_date) AS next_plan,
        LEAD(start_date) OVER (PARTITION BY customer_id ORDER BY start_date) AS downgrade_date
    FROM subscriptions
) t
WHERE from_plan IN ('BASIC', 'PRO')
  AND next_plan = 'FREE';
```

### 🧠 Explanation

- **`LEAD()`** looks *forward* to the next row within the partition — it lets each subscription "see" the plan that came after it. `PARTITION BY customer_id` keeps each customer's timeline separate; `ORDER BY start_date` puts their plans in chronological order.
- For each row we capture both the **next plan** (`LEAD(plan_type)`) and **when it started** (`LEAD(start_date)`) — the effective downgrade date.
- The outer filter defines a *downgrade* precisely: current plan is paid (`from_plan IN ('BASIC','PRO')`) **and** the very next plan is `FREE`.
- Walking the data: customer 1 goes `BASIC → FREE` ✅ (a downgrade). Customer 2 stays `PRO → PRO` (no change). Customer 3 goes `FREE → BASIC` (an *upgrade*, so excluded). Only customer 1 qualifies.

---

## Q5 — Likes, comments & shares per post

**Question:** Find the number of likes, comments, and shares for each post.

### 🗄️ Dataset

```sql
CREATE TABLE posts (
    post_id INT PRIMARY KEY,
    user_id INT,
    post_date DATE
);

CREATE TABLE engagements (
    engagement_id INT PRIMARY KEY,
    post_id INT,
    engagement_type VARCHAR(10)  -- LIKE, COMMENT, SHARE
);

INSERT INTO posts VALUES
(1,101,'2024-06-01'),
(2,102,'2024-06-02'),
(3,101,'2024-06-03');

INSERT INTO engagements VALUES
(1,1,'LIKE'),(2,1,'LIKE'),(3,1,'COMMENT'),(4,1,'SHARE'),
(5,2,'LIKE'),(6,2,'COMMENT'),(7,2,'COMMENT'),
(8,3,'LIKE');
```

### ✅ Expected Output

| post_id | likes | comments | shares |
|---------|-------|----------|--------|
| 1 | 2 | 1 | 1 |
| 2 | 1 | 2 | 0 |
| 3 | 1 | 0 | 0 |

### 💡 Solution

```sql
SELECT
    p.post_id,
    SUM(CASE WHEN e.engagement_type = 'LIKE'    THEN 1 ELSE 0 END) AS likes,
    SUM(CASE WHEN e.engagement_type = 'COMMENT' THEN 1 ELSE 0 END) AS comments,
    SUM(CASE WHEN e.engagement_type = 'SHARE'   THEN 1 ELSE 0 END) AS shares
FROM posts p
LEFT JOIN engagements e
    ON p.post_id = e.post_id
GROUP BY p.post_id;
```

### 🧠 Explanation

- **Pivoting rows into columns:** the engagements table stores one row per interaction with a `type`. **Conditional aggregation** — `SUM(CASE WHEN type = 'X' THEN 1 ELSE 0 END)` — turns those row-values into separate columns, counting each type in a single pass.
- **Why `LEFT JOIN` matters:** a post with zero engagements (none here, but the pattern must handle it) would vanish under an `INNER JOIN`. `LEFT JOIN` keeps every post; its `CASE` expressions all evaluate to `0`, so it correctly reports `0` likes/comments/shares instead of disappearing.
- `GROUP BY p.post_id` collapses each post's engagement rows into one summary row.
- 💡 On MySQL you can shorten each measure to `SUM(e.engagement_type = 'LIKE')` — a boolean condition evaluates to `1`/`0` — but the explicit `CASE` form is clearer and portable across databases.

---

<div align="center">

**More problem sets coming soon.** ⭐ the repo to follow along.

[⬅ Back to Course Home](../README.md) · [Problem Set 1](problem-set-01.md)

</div>
