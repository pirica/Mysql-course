# 🧩 SQL Interview Practice — Problem Set 1

> **Part 1: Problem Set** — five real-world query problems inspired by product companies (food delivery, wallets, streaming, ride-hailing).
> Each problem follows the same flow: **Question → Dataset → Expected Output → Solution(s) → Explanation.**

Every problem is self-contained: run the *Dataset Creation* block, then run the solution to reproduce the expected output. All queries target **MySQL 8.0+**.

---

## 📋 Contents

| # | Problem | Core Concept |
|---|---------|--------------|
| 1 | [Customers who never placed an order](#q1--customers-who-never-placed-an-order) | `LEFT JOIN` / `NOT EXISTS` (anti-join) |
| 2 | [Average delivery time per restaurant](#q2--average-delivery-time-per-restaurant) | `TIMESTAMPDIFF`, `AVG`, `GROUP BY` |
| 3 | [Running wallet balance per customer](#q3--running-wallet-balance-per-customer) | Window function + `CASE` |
| 4 | [Distinct active users per month](#q4--distinct-active-users-per-month) | `DATE_FORMAT`, `COUNT(DISTINCT ...)` |
| 5 | [Trip cancellation rate per city](#q5--trip-cancellation-rate-per-city) | Conditional aggregation, rate math |

---

## Q1 — Customers who never placed an order

**Question:** Find customers who have never placed an order.

### 🗄️ Dataset

```sql
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(50),
    city VARCHAR(50),
    signup_date DATE
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE,
    order_status VARCHAR(20)
);

INSERT INTO customers VALUES
(1,'Ravi Kumar','Mumbai','2024-01-05'),
(2,'Anita Sharma','Delhi','2024-01-10'),
(3,'Vikram Singh','Pune','2024-01-15'),
(4,'Neha Joshi','Bangalore','2024-02-01'),
(5,'Suresh Rao','Chennai','2024-02-10'),
(6,'Priya Nair','Hyderabad','2024-02-15'),
(7,'Amit Verma','Mumbai','2024-03-01'),
(8,'Kavita Desai','Delhi','2024-03-05');

INSERT INTO orders VALUES
(101,1,'2024-01-20','DELIVERED'),
(102,2,'2024-01-25','DELIVERED'),
(103,1,'2024-02-05','CANCELLED'),
(104,4,'2024-02-10','DELIVERED'),
(105,5,'2024-02-20','DELIVERED');
```

### ✅ Expected Output

| customer_id | customer_name |
|-------------|---------------|
| 3 | Vikram Singh |
| 6 | Priya Nair |
| 7 | Amit Verma |
| 8 | Kavita Desai |

### 💡 Solution

**Approach 1 — `LEFT JOIN` + `IS NULL` (anti-join)**

```sql
SELECT c.customer_id, c.customer_name
FROM customers c
LEFT JOIN orders o
    ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

**Approach 2 — `NOT EXISTS`**

```sql
SELECT c.customer_id, c.customer_name
FROM customers c
WHERE NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.customer_id
);
```

### 🧠 Explanation

- A **`LEFT JOIN`** keeps every customer, attaching order rows where they exist. Customers with no match get `NULL`s in the order columns, so filtering `WHERE o.order_id IS NULL` isolates exactly the customers who never ordered. This is the classic **anti-join** pattern.
- **`NOT EXISTS`** reads more literally as the intent — *"keep the customer only if no matching order exists"* — and stops scanning as soon as it finds the first match, which is often efficient with an index on `orders.customer_id`.
- Prefer `NOT EXISTS` over `NOT IN` here: `NOT IN` behaves unexpectedly if the subquery can return `NULL`s.

---

## Q2 — Average delivery time per restaurant

**Question:** Find the average delivery time (in minutes) per restaurant.

### 🗄️ Dataset

```sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    restaurant_name VARCHAR(50),
    order_time DATETIME,
    delivery_time DATETIME  -- NULL means order is still in transit / undelivered
);

INSERT INTO orders VALUES
(1,'Spice Villa','2024-03-01 12:00:00','2024-03-01 12:45:00'),
(2,'Spice Villa','2024-03-01 13:00:00','2024-03-01 13:50:00'),
(3,'Pizza Hub','2024-03-01 12:10:00','2024-03-01 12:40:00'),
(4,'Pizza Hub','2024-03-01 13:15:00',NULL),
(5,'Pizza Hub','2024-03-01 14:00:00','2024-03-01 14:35:00'),
(6,'Curry House','2024-03-01 12:30:00','2024-03-01 13:20:00');
```

### ✅ Expected Output

| restaurant_name | avg_delivery_minutes |
|-----------------|----------------------|
| Curry House | 50.00 |
| Pizza Hub | 32.50 |
| Spice Villa | 47.50 |

### 💡 Solution

```sql
SELECT
    restaurant_name,
    ROUND(AVG(TIMESTAMPDIFF(MINUTE, order_time, delivery_time)), 2) AS avg_delivery_minutes
FROM orders
GROUP BY restaurant_name
ORDER BY avg_delivery_minutes;
```

### 🧠 Explanation

- **`TIMESTAMPDIFF(MINUTE, order_time, delivery_time)`** returns the gap between the two timestamps in whole minutes — the delivery duration for each order.
- **`AVG(...)`** collapses those per-order durations into one average per restaurant, driven by `GROUP BY restaurant_name`.
- The undelivered Pizza Hub order (`order_id 4`, `delivery_time = NULL`) produces a `NULL` duration. **`AVG` ignores `NULL`s**, so Pizza Hub's average is computed over its two *completed* orders only `(30 + 35) / 2 = 32.50` — exactly what you want.
- **`ROUND(..., 2)`** formats the result to 2 decimal places for clean, money/metric-style output.

---

## Q3 — Running wallet balance per customer

**Question:** Show each customer's running wallet balance after every transaction.

### 🗄️ Dataset

```sql
CREATE TABLE wallet_transactions (
    txn_id INT PRIMARY KEY,
    customer_id INT,
    txn_date DATE,
    txn_type VARCHAR(10),   -- CREDIT or DEBIT
    amount DECIMAL(10,2)
);

INSERT INTO wallet_transactions VALUES
(1,1,'2024-01-01','CREDIT',1000.00),
(2,1,'2024-01-03','DEBIT',200.00),
(3,1,'2024-01-05','CREDIT',500.00),
(4,1,'2024-01-07','DEBIT',300.00),
(5,2,'2024-01-01','CREDIT',2000.00),
(6,2,'2024-01-04','DEBIT',700.00);
```

### ✅ Expected Output

| customer_id | txn_date | txn_type | amount | running_balance |
|-------------|----------|----------|--------|-----------------|
| 1 | 2024-01-01 | CREDIT | 1000.00 | 1000.00 |
| 1 | 2024-01-03 | DEBIT | 200.00 | 800.00 |
| 1 | 2024-01-05 | CREDIT | 500.00 | 1300.00 |
| 1 | 2024-01-07 | DEBIT | 300.00 | 1000.00 |
| 2 | 2024-01-01 | CREDIT | 2000.00 | 2000.00 |
| 2 | 2024-01-04 | DEBIT | 700.00 | 1300.00 |

### 💡 Solution

```sql
SELECT
    customer_id,
    txn_date,
    txn_type,
    amount,
    SUM(
        CASE
            WHEN txn_type = 'CREDIT' THEN amount
            WHEN txn_type = 'DEBIT'  THEN -amount
            ELSE 0
        END
    ) OVER (PARTITION BY customer_id ORDER BY txn_date) AS running_balance
FROM wallet_transactions;
```

### 🧠 Explanation

- The **`CASE`** expression turns each transaction into a *signed* amount: credits stay positive, debits become negative. This is the trick that makes a single `SUM` handle both directions.
- **`SUM(...) OVER (...)`** is a window function — it computes a cumulative total *without collapsing rows*, so every transaction is still returned individually.
- **`PARTITION BY customer_id`** restarts the running total for each customer (customer 2's balance doesn't carry over from customer 1).
- **`ORDER BY txn_date`** defines the accumulation order. With `ORDER BY` present, the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` — i.e. "sum everything up to and including this row" — which is precisely a running balance.

---

## Q4 — Distinct active users per month

**Question:** Find the number of distinct active users per month.

### 🗄️ Dataset

```sql
CREATE TABLE watch_history (
    watch_id INT PRIMARY KEY,
    user_id INT,
    content_id INT,
    watch_date DATE
);

INSERT INTO watch_history VALUES
(1,1,101,'2024-01-05'),
(2,2,102,'2024-01-10'),
(3,1,103,'2024-01-20'),
(4,3,101,'2024-02-01'),
(5,1,104,'2024-02-15'),
(6,4,105,'2024-02-20'),
(7,2,101,'2024-03-01'),
(8,3,103,'2024-03-05');
```

### ✅ Expected Output

| month | active_users |
|---------|--------------|
| 2024-01 | 2 |
| 2024-02 | 3 |
| 2024-03 | 2 |

### 💡 Solution

```sql
SELECT
    DATE_FORMAT(watch_date, '%Y-%m') AS month,
    COUNT(DISTINCT user_id) AS active_users
FROM watch_history
GROUP BY DATE_FORMAT(watch_date, '%Y-%m')
ORDER BY month;
```

### 🧠 Explanation

- **`DATE_FORMAT(watch_date, '%Y-%m')`** buckets every row into a `YYYY-MM` month label, and `GROUP BY` on that same expression collapses each month into one row.
- **`COUNT(DISTINCT user_id)`** is the key detail. A user who watches multiple times in the same month must count **once** — e.g. in January user 1 has two rows (`watch_id 1` and `3`), but they're one active user.
- ⚠️ **Common mistake:** using plain `COUNT(user_id)` instead. That counts *watch events*, not *users* — it would report January as `3` (three rows) instead of `2` (two distinct users). Whenever a question says *"distinct / unique users,"* reach for `COUNT(DISTINCT ...)`.

---

## Q5 — Trip cancellation rate per city

**Question:** Find the trip cancellation rate for each city.

### 🗄️ Dataset

```sql
CREATE TABLE trips (
    trip_id INT PRIMARY KEY,
    city VARCHAR(30),
    rider_id INT,
    driver_id INT,
    trip_status VARCHAR(20)  -- COMPLETED, CANCELLED_BY_RIDER, CANCELLED_BY_DRIVER
);

INSERT INTO trips VALUES
(1,'Mumbai',1,10,'COMPLETED'),
(2,'Mumbai',2,11,'CANCELLED_BY_RIDER'),
(3,'Mumbai',3,10,'COMPLETED'),
(4,'Mumbai',4,12,'CANCELLED_BY_DRIVER'),
(5,'Delhi',5,13,'COMPLETED'),
(6,'Delhi',6,14,'COMPLETED'),
(7,'Delhi',7,13,'CANCELLED_BY_RIDER'),
(8,'Pune',8,15,'COMPLETED');
```

### ✅ Expected Output

| city | total_trips | cancelled_trips | cancellation_rate_pct |
|--------|-------------|-----------------|-----------------------|
| Mumbai | 4 | 2 | 50.00 |
| Delhi | 3 | 1 | 33.33 |
| Pune | 1 | 0 | 0.00 |

### 💡 Solution

```sql
SELECT
    city,
    COUNT(*) AS total_trips,
    SUM(CASE WHEN trip_status LIKE 'CANCELLED%' THEN 1 ELSE 0 END) AS cancelled_trips,
    ROUND(
        SUM(CASE WHEN trip_status LIKE 'CANCELLED%' THEN 1 ELSE 0 END) * 100.0 / COUNT(*),
        2
    ) AS cancellation_rate_pct
FROM trips
GROUP BY city
ORDER BY cancellation_rate_pct DESC;
```

### 🧠 Explanation

- **Conditional aggregation** is the pattern here: `SUM(CASE WHEN <condition> THEN 1 ELSE 0 END)` counts only the rows that match a condition, all within one pass over the data.
- **`trip_status LIKE 'CANCELLED%'`** captures *both* cancellation types (`CANCELLED_BY_RIDER` and `CANCELLED_BY_DRIVER`) with a single prefix match — cleaner than listing each value with `IN (...)`.
- The rate is `cancelled / total * 100`. Multiplying by **`100.0`** (not `100`) forces floating-point division so partial rates like Delhi's `1/3 = 33.33` don't get truncated by integer math. `ROUND(..., 2)` gives the clean percentage.
- `COUNT(*)` counts every trip in the city (the denominator); the conditional `SUM` counts just the cancelled ones (the numerator).

---

<div align="center">

**More problem sets coming soon.** ⭐ the repo to follow along.

[⬅ Back to Course Home](../README.md)

</div>
