# 🔥 SQL Interview Practice — Advanced Problem Set

> **Curated hard problems** for product-company interviews. These deliberately **skip** the fundamentals already covered in the [course docs](../README.md) and in [Problem Set 1](problem-set-01.md) / [Problem Set 2](problem-set-02.md) (basic joins, `GROUP BY`/`HAVING`, `EXISTS`, simple running totals, LEAD downgrade, etc.).
>
> Every problem is self-contained: **Problem Statement → Dataset → Expected Output.** No solutions — solve them yourself. All target **MySQL 8.0+**.

---

## 📋 Contents

| # | Problem | Pattern |
|---|---------|---------|
| 1 | [Streak ranges from active days](#q1) | Islands (collapse consecutive dates) |
| 2 | [Longest win streak per player](#q2) | Streak with reset |
| 3 | [Sessionize a clickstream](#q3) | Sessionization (30-min inactivity gap) |
| 4 | [Cohort retention grid](#q4) | Cohort × month-offset matrix |
| 5 | [Price history → validity ranges](#q5) | SCD Type 2 |
| 6 | [Median order value per city](#q6) | Median (no built-in) |
| 7 | [Salary percentile per employee](#q7) | `PERCENT_RANK` |
| 8 | [Org hierarchy depth](#q8) | Recursive CTE |
| 9 | [Missing invoice-number ranges](#q9) | Gap detection (range form) |
| 10 | [Missing daily inventory snapshots](#q10) | Matrix gap (cross join) |
| 11 | [Source-vs-target upsert plan](#q11) | MERGE / upsert classification |
| 12 | [Impossible-travel card fraud](#q12) | Anomaly (self-join + time window) |
| 13 | [Product share of category revenue](#q13) | Partitioned ratio-to-report |
| 14 | [Customer spend quartiles](#q14) | `NTILE` segmentation |
| 15 | [Year-over-year revenue growth](#q15) | `LAG` + growth math |
| 16 | [Funnel drop-off by stage](#q16) | Funnel conversion |
| 17 | [3-day consecutive logins](#q17) | Consecutive records |
| 18 | [Which duplicate rows to delete](#q18) | Dedup (keep-earliest) |
| 19 | [Payment pivot: count + amount](#q19) | Pivot, multiple aggregates |
| 20 | [2nd-highest salary per dept](#q20) | Nth-per-group with ties |
| 21 | [Pareto cumulative market share](#q21) | Cumulative distribution |
| 22 | [Merge overlapping leave periods](#q22) | Interval merge (consolidate) |
| 23 | [Rolling 3-day distinct users](#q23) | Windowed distinct count |
| 24 | [Device status change points](#q24) | Change detection (`LAG`) |
| 25 | [Monthly lifecycle transitions](#q25) | New / Retained / Churned / Resurrected |
| 26 | [Fill missing dates + carry forward](#q26) | Time-series LOCF (recursive) |
| 27 | [Customers who bought every product](#q27) | Relational division |
| 28 | [Earn more than their manager](#q28) | Self-join |
| 29 | [Max concurrent sessions](#q29) | Sweep line (events ±1) |
| 30 | [Unpivot quarterly columns](#q30) | Columns → rows |
| 31 | [Most frequent category per customer](#q31) | Mode (tie-break) |
| 32 | [Days since previous order](#q32) | `LAG` + gap flag |
| 33 | [Running account balance](#q33) | Running total (signed) |
| 34 | [Each customer's 2nd order](#q34) | Nth event per group |
| 35 | [Products never sold](#q35) | Anti-join |
| 36 | [Count all subordinates](#q36) | Recursive rollup |
| 37 | [BOM component explosion](#q37) | Recursive weighted |
| 38 | [Rank movers month-over-month](#q38) | `RANK` + `LAG` delta |
| 39 | [Quarterly sales pivot](#q39) | Conditional aggregation |
| 40 | [3+ consecutive absences](#q40) | Streak on gaps |
| 41 | [Double-booked rooms](#q41) | Interval overlap self-join |
| 42 | [Active every month (no gap)](#q42) | Contiguous presence |
| 43 | [Top 3 products per category](#q43) | `DENSE_RANK` top-N |
| 44 | [Cumulative new users per day](#q44) | First-seen running count |
| 45 | [Longest gap between logins](#q45) | `LAG` max gap |
| 46 | [Dedup keep-latest](#q46) | `ROW_NUMBER` dedup |
| 47 | [First available seat number](#q47) | Single-value gap |
| 48 | [User pairs sharing a hobby](#q48) | Self-join pair dedup |
| 49 | [First-touch vs last-touch](#q49) | First/last per group |
| 50 | [Region totals + grand total](#q50) | `WITH ROLLUP` |

---

<a id="q1"></a>
## Q1. Collapse each user's consecutive active days into streak ranges (start, end, length).

**Dataset:**
```sql
CREATE TABLE app_logins (user_id INT, login_date DATE);

INSERT INTO app_logins VALUES
(1,'2024-03-01'),(1,'2024-03-02'),(1,'2024-03-03'),
(1,'2024-03-06'),(1,'2024-03-07'),
(1,'2024-03-09'),
(2,'2024-03-01'),(2,'2024-03-02');
```

**Expected Output:**
```
user_id | streak_start | streak_end | streak_length
1       | 2024-03-01   | 2024-03-03 | 3
1       | 2024-03-06   | 2024-03-07 | 2
1       | 2024-03-09   | 2024-03-09 | 1
2       | 2024-03-01   | 2024-03-02 | 2
```

---

<a id="q2"></a>
## Q2. Find the longest streak of consecutive wins for each player (matches ordered by match number).

**Dataset:**
```sql
CREATE TABLE matches (player VARCHAR(20), match_no INT, result CHAR(1));

INSERT INTO matches VALUES
('Kohli',1,'W'),('Kohli',2,'W'),('Kohli',3,'L'),('Kohli',4,'W'),
('Kohli',5,'W'),('Kohli',6,'W'),('Kohli',7,'L'),('Kohli',8,'W'),
('Rohit',1,'L'),('Rohit',2,'W'),('Rohit',3,'W');
```

**Expected Output:**
```
player | longest_win_streak
Kohli  | 3
Rohit  | 2
```

---

<a id="q3"></a>
## Q3. Count sessions per user, where any gap of more than 30 minutes between consecutive events starts a new session.

**Dataset:**
```sql
CREATE TABLE clickstream (user_id INT, event_time DATETIME);

INSERT INTO clickstream VALUES
(1,'2024-04-01 09:00:00'),
(1,'2024-04-01 09:20:00'),
(1,'2024-04-01 09:45:00'),
(1,'2024-04-01 11:00:00'),
(2,'2024-04-01 10:00:00'),
(2,'2024-04-01 10:10:00'),
(2,'2024-04-01 10:20:00');
```

**Expected Output:**
```
user_id | session_count
1       | 2
2       | 1
```

---

<a id="q4"></a>
## Q4. Build a cohort retention grid: for each signup-month cohort, count active customers by months elapsed since signup.

**Dataset:**
```sql
CREATE TABLE customers (id INT PRIMARY KEY, signup_date DATE);
CREATE TABLE activity (customer_id INT, activity_date DATE);

INSERT INTO customers VALUES (1,'2024-01-05'),(2,'2024-01-20'),(3,'2024-02-03');
INSERT INTO activity VALUES
(1,'2024-01-10'),(1,'2024-02-02'),(1,'2024-03-01'),
(2,'2024-01-25'),(2,'2024-02-15'),
(3,'2024-02-10'),(3,'2024-04-01');
```

**Expected Output:**
```
cohort_month | month_offset | active_customers
2024-01      | 0            | 2
2024-01      | 1            | 2
2024-01      | 2            | 1
2024-02      | 0            | 1
2024-02      | 2            | 1
```

---

<a id="q5"></a>
## Q5. Convert a stream of price-change events into non-overlapping validity ranges (SCD Type 2: valid_from / valid_to; current row has NULL end).

**Dataset:**
```sql
CREATE TABLE price_changes (product VARCHAR(10), change_date DATE, price INT);

INSERT INTO price_changes VALUES
('SKU1','2024-01-01',100),
('SKU1','2024-04-01',120),
('SKU1','2024-09-01',150);
```

**Expected Output:**
```
product | price | valid_from | valid_to
SKU1    | 100   | 2024-01-01 | 2024-03-31
SKU1    | 120   | 2024-04-01 | 2024-08-31
SKU1    | 150   | 2024-09-01 | NULL
```

---

<a id="q6"></a>
## Q6. Compute the median order amount per city (handle both odd and even counts).

**Dataset:**
```sql
CREATE TABLE orders (city VARCHAR(20), amount INT);

INSERT INTO orders VALUES
('Mumbai',100),('Mumbai',200),('Mumbai',300),('Mumbai',400),
('Delhi',50),('Delhi',150),('Delhi',250);
```

**Expected Output:**
```
city   | median_amount
Delhi  | 150.0000
Mumbai | 250.0000
```

---

<a id="q7"></a>
## Q7. For each employee, compute their salary percentile using `PERCENT_RANK`.

**Dataset:**
```sql
CREATE TABLE employees (name CHAR(1), salary INT);

INSERT INTO employees VALUES
('A',30000),('B',45000),('C',50000),('D',60000),('E',80000),('F',100000);
```

**Expected Output:**
```
name | salary | percentile
A    | 30000  | 0.00
B    | 45000  | 0.20
C    | 50000  | 0.40
D    | 60000  | 0.60
E    | 80000  | 0.80
F    | 100000 | 1.00
```

---

<a id="q8"></a>
## Q8. Using a recursive CTE, print each employee's depth in the org chart (CEO = level 1).

**Dataset:**
```sql
CREATE TABLE employees (id INT PRIMARY KEY, name VARCHAR(20), mgr_id INT);

INSERT INTO employees VALUES
(1,'CEO',NULL),(2,'VP1',1),(3,'VP2',1),(4,'Mgr1',2),(5,'Eng1',4);
```

**Expected Output:**
```
name | level
CEO  | 1
VP1  | 2
VP2  | 2
Mgr1 | 3
Eng1 | 4
```

---

<a id="q9"></a>
## Q9. Given a table of used invoice numbers, report the ranges of missing numbers (start and end of each gap).

**Dataset:**
```sql
CREATE TABLE invoices (inv_no INT PRIMARY KEY);

INSERT INTO invoices VALUES (1001),(1002),(1003),(1006),(1007),(1010);
```

**Expected Output:**
```
gap_start | gap_end
1004      | 1005
1008      | 1009
```

---

<a id="q10"></a>
## Q10. Every active product needs a daily inventory snapshot. Find the (product, date) snapshots that are missing.

**Dataset:**
```sql
CREATE TABLE products (product VARCHAR(10));
CREATE TABLE calendar (snap_date DATE);
CREATE TABLE snapshots (product VARCHAR(10), snap_date DATE);

INSERT INTO products VALUES ('P1'),('P2');
INSERT INTO calendar VALUES ('2024-06-01'),('2024-06-02'),('2024-06-03');
INSERT INTO snapshots VALUES
('P1','2024-06-01'),('P1','2024-06-02'),('P1','2024-06-03'),
('P2','2024-06-01'),('P2','2024-06-03');
```

**Expected Output:**
```
product | missing_date
P2      | 2024-06-02
```

---

<a id="q11"></a>
## Q11. Compare a staging feed against the master table and classify each staging row as INSERT (new id), UPDATE (id exists but email differs), or UNCHANGED.

**Dataset:**
```sql
CREATE TABLE master (id INT PRIMARY KEY, email VARCHAR(30));
CREATE TABLE staging (id INT PRIMARY KEY, email VARCHAR(30));

INSERT INTO master VALUES (1,'a@x.com'),(2,'b@x.com'),(3,'c@x.com');
INSERT INTO staging VALUES (1,'a@x.com'),(2,'b_new@x.com'),(4,'d@x.com');
```

**Expected Output:**
```
id | action
1  | UNCHANGED
2  | UPDATE
4  | INSERT
```

---

<a id="q12"></a>
## Q12. Flag "impossible travel": a card used in two different cities within 60 minutes.

**Dataset:**
```sql
CREATE TABLE card_txns (txn_id INT PRIMARY KEY, card_id VARCHAR(10), txn_time DATETIME, city VARCHAR(20));

INSERT INTO card_txns VALUES
(1,'C1','2024-07-01 10:00:00','Mumbai'),
(2,'C1','2024-07-01 10:20:00','Delhi'),
(3,'C2','2024-07-01 09:00:00','Pune'),
(4,'C2','2024-07-01 15:00:00','Pune'),
(5,'C3','2024-07-01 12:00:00','Chennai'),
(6,'C3','2024-07-01 12:30:00','Chennai');
```

**Expected Output:**
```
card_id | time1               | city1  | time2               | city2
C1      | 2024-07-01 10:00:00 | Mumbai | 2024-07-01 10:20:00 | Delhi
```

---

<a id="q13"></a>
## Q13. Express each product's revenue as a percentage of its own category's total revenue (partitioned ratio-to-report).

**Dataset:**
```sql
CREATE TABLE sales (product VARCHAR(15), category VARCHAR(15), revenue INT);

INSERT INTO sales VALUES
('Rice','Grocery',6000),('Wheat','Grocery',4000),
('Chips','Snacks',3000),('Namkeen','Snacks',1000);
```

**Expected Output:**
```
category | product | revenue | pct_of_category
Grocery  | Rice    | 6000    | 60.00
Grocery  | Wheat   | 4000    | 40.00
Snacks   | Chips   | 3000    | 75.00
Snacks   | Namkeen | 1000    | 25.00
```

---

<a id="q14"></a>
## Q14. Segment customers into 4 quartiles by lifetime spend (quartile 1 = highest spenders).

**Dataset:**
```sql
CREATE TABLE customers (customer_id INT PRIMARY KEY, lifetime_spend INT);

INSERT INTO customers VALUES
(1,1000),(2,2000),(3,3000),(4,4000),(5,5000),(6,6000),(7,7000),(8,8000);
```

**Expected Output:**
```
customer_id | lifetime_spend | quartile
8           | 8000           | 1
7           | 7000           | 1
6           | 6000           | 2
5           | 5000           | 2
4           | 4000           | 3
3           | 3000           | 3
2           | 2000           | 4
1           | 1000           | 4
```

---

<a id="q15"></a>
## Q15. Compute year-over-year revenue growth percentage (rounded to 2 decimals; first year has no prior).

**Dataset:**
```sql
CREATE TABLE yearly_revenue (yr INT PRIMARY KEY, revenue INT);

INSERT INTO yearly_revenue VALUES
(2021,500000),(2022,650000),(2023,585000),(2024,760500);
```

**Expected Output:**
```
yr   | revenue | yoy_growth_pct
2021 | 500000  | NULL
2022 | 650000  | 30.00
2023 | 585000  | -10.00
2024 | 760500  | 30.00
```

---

<a id="q16"></a>
## Q16. Build a funnel: count distinct users reaching each stage, and the conversion rate from the previous stage. Stage order: Visit → SignUp → AddToCart → Purchase.

**Dataset:**
```sql
CREATE TABLE funnel_events (user_id INT, stage VARCHAR(15));

INSERT INTO funnel_events VALUES
(1,'Visit'),(1,'SignUp'),(1,'AddToCart'),(1,'Purchase'),
(2,'Visit'),(2,'SignUp'),(2,'AddToCart'),
(3,'Visit'),(3,'SignUp'),
(4,'Visit'),
(5,'Visit'),(5,'SignUp'),(5,'AddToCart'),(5,'Purchase');
```

**Expected Output:**
```
stage     | users | conv_from_prev_pct
Visit     | 5     | NULL
SignUp    | 4     | 80.00
AddToCart | 3     | 75.00
Purchase  | 2     | 66.67
```

---

<a id="q17"></a>
## Q17. Find users who logged in on at least 3 consecutive calendar days at least once.

**Dataset:**
```sql
CREATE TABLE logins (user_id INT, login_date DATE);

INSERT INTO logins VALUES
(1,'2024-06-01'),(1,'2024-06-02'),(1,'2024-06-03'),
(2,'2024-06-01'),(2,'2024-06-02'),(2,'2024-06-04'),(2,'2024-06-05'),
(3,'2024-06-10'),(3,'2024-06-11'),(3,'2024-06-12'),(3,'2024-06-13');
```

**Expected Output:**
```
user_id
1
3
```

---

<a id="q18"></a>
## Q18. Duplicate customers share the same email. Identify the exact rows to delete, keeping only the smallest id per email.

**Dataset:**
```sql
CREATE TABLE customers (id INT PRIMARY KEY, name VARCHAR(20), email VARCHAR(30));

INSERT INTO customers VALUES
(1,'Amit','a@x.com'),
(2,'Bob','b@x.com'),
(3,'Amit K','a@x.com'),
(4,'Carol','c@x.com'),
(5,'A Kumar','a@x.com'),
(6,'Bobby','b@x.com');
```

**Expected Output:**
```
id_to_delete | email
3            | a@x.com
5            | a@x.com
6            | b@x.com
```

---

<a id="q19"></a>
## Q19. Per store, pivot transactions by payment method into both a count and a total amount for CARD and UPI.

**Dataset:**
```sql
CREATE TABLE transactions (store CHAR(1), method VARCHAR(10), amount INT);

INSERT INTO transactions VALUES
('A','CARD',200),('A','CARD',300),('A','UPI',100),
('B','UPI',400);
```

**Expected Output:**
```
store | card_cnt | card_amt | upi_cnt | upi_amt
A     | 2        | 500      | 1       | 100
B     | 0        | 0        | 1       | 400
```

---

<a id="q20"></a>
## Q20. Find the employee(s) with the 2nd-highest **distinct** salary in each department (include all ties).

**Dataset:**
```sql
CREATE TABLE employees (name VARCHAR(20), dept VARCHAR(10), salary INT);

INSERT INTO employees VALUES
('A','Eng',120000),('B','Eng',100000),('C','Eng',100000),('D','Eng',90000),
('E','Sales',80000),('F','Sales',70000),('G','Sales',60000);
```

**Expected Output:**
```
dept  | name | salary
Eng   | B    | 100000
Eng   | C    | 100000
Sales | F    | 70000
```

---

<a id="q21"></a>
## Q21. For a Pareto (80/20) analysis, compute each brand's revenue share and the running cumulative share, ordered by revenue descending.

**Dataset:**
```sql
CREATE TABLE brand_revenue (brand CHAR(1), revenue INT);

INSERT INTO brand_revenue VALUES ('X',5000),('Y',3000),('Z',1500),('W',500);
```

**Expected Output:**
```
brand | revenue | share_pct | cumulative_pct
X     | 5000    | 50.00     | 50.00
Y     | 3000    | 30.00     | 80.00
Z     | 1500    | 15.00     | 95.00
W     | 500     | 5.00      | 100.00
```

---

<a id="q22"></a>
## Q22. An employee filed several leave requests, some overlapping or adjacent. Merge them into consolidated, non-overlapping leave periods.

**Dataset:**
```sql
CREATE TABLE leaves (emp_id INT, start_date DATE, end_date DATE);

INSERT INTO leaves VALUES
(1,'2024-05-01','2024-05-05'),
(1,'2024-05-04','2024-05-08'),
(1,'2024-05-15','2024-05-18'),
(1,'2024-05-18','2024-05-20');
```

**Expected Output:**
```
emp_id | period_start | period_end
1      | 2024-05-01   | 2024-05-08
1      | 2024-05-15   | 2024-05-20
```

---

<a id="q23"></a>
## Q23. For each date, count the distinct users active over that date plus the preceding 2 days (a trailing 3-day distinct-user window).

**Dataset:**
```sql
CREATE TABLE daily_active (activity_date DATE, user_id INT);

INSERT INTO daily_active VALUES
('2024-08-01',1),('2024-08-01',2),
('2024-08-02',2),
('2024-08-03',3),
('2024-08-04',1);
```

**Expected Output:**
```
activity_date | rolling_3day_users
2024-08-01    | 2
2024-08-02    | 2
2024-08-03    | 3
2024-08-04    | 3
```

---

<a id="q24"></a>
## Q24. From a device status log, report only the moments where the status changed from the previous reading (with old and new status).

**Dataset:**
```sql
CREATE TABLE status_log (device_id VARCHAR(10), log_time DATETIME, status VARCHAR(5));

INSERT INTO status_log VALUES
('D1','2024-09-01 08:00:00','ON'),
('D1','2024-09-01 09:00:00','ON'),
('D1','2024-09-01 10:00:00','OFF'),
('D1','2024-09-01 11:00:00','OFF'),
('D1','2024-09-01 12:00:00','ON');
```

**Expected Output:**
```
device_id | change_time         | prev_status | new_status
D1        | 2024-09-01 10:00:00 | ON          | OFF
D1        | 2024-09-01 12:00:00 | OFF         | ON
```

---

<a id="q25"></a>
## Q25. For each month a customer is active, classify them as **New** (first-ever active month), **Retained** (also active the previous month), or **Resurrected** (inactive last month but active in some earlier month).

**Dataset:**
```sql
CREATE TABLE monthly_active (customer_id INT, active_month VARCHAR(7));

INSERT INTO monthly_active VALUES
(1,'2024-01'),(1,'2024-02'),(1,'2024-03'),
(2,'2024-01'),(2,'2024-03'),
(3,'2024-02');
```

**Expected Output:**
```
customer_id | active_month | status
1           | 2024-01      | New
1           | 2024-02      | Retained
1           | 2024-03      | Retained
2           | 2024-01      | New
2           | 2024-03      | Resurrected
3           | 2024-02      | New
```

---

<a id="part2"></a>
# 🧩 Part 2 — 25 More Patterns (with worked solutions)

> Same rigor, 25 **fresh** patterns (division, sweep-line, LOCF, BOM explosion, rank-movers, interval overlap…). Unlike Part 1, each of these ships a **worked solution** you can run on MySQL 8.0+. Every query below was executed and its output verified against the "Expected Output" shown.

---

<a id="q26"></a>
## Q26. Fill in every missing calendar day per sensor and carry forward the last known reading (LOCF: last-observation-carried-forward).

**Dataset:**
```sql
CREATE TABLE readings (sensor VARCHAR(5), reading_date DATE, temp INT);

INSERT INTO readings VALUES
('S1','2024-01-01',10),('S1','2024-01-03',20),('S1','2024-01-05',15),
('S2','2024-01-02',5),('S2','2024-01-04',8);
```

**Expected Output:**
```
sensor | reading_date | temp
S1     | 2024-01-01   | 10
S1     | 2024-01-02   | 10
S1     | 2024-01-03   | 20
S1     | 2024-01-04   | 20
S1     | 2024-01-05   | 15
S2     | 2024-01-02   | 5
S2     | 2024-01-03   | 5
S2     | 2024-01-04   | 8
```

**Solution:**
```sql
WITH RECURSIVE bounds AS (
  SELECT sensor, MIN(reading_date) mn, MAX(reading_date) mx
  FROM readings GROUP BY sensor
),
cal AS (
  SELECT sensor, mn AS d, mx FROM bounds
  UNION ALL
  SELECT sensor, d + INTERVAL 1 DAY, mx FROM cal WHERE d < mx
)
SELECT c.sensor, c.d AS reading_date,
  (SELECT r.temp FROM readings r
   WHERE r.sensor = c.sensor AND r.reading_date <= c.d
   ORDER BY r.reading_date DESC LIMIT 1) AS temp
FROM cal c
ORDER BY c.sensor, c.d;
```
> Generate every date in each sensor's min→max span with a recursive CTE, then for each generated day pull the most recent reading on-or-before it (the correlated "latest ≤ d" subquery is the carry-forward).

---

<a id="q27"></a>
## Q27. Find the customers who have purchased **every** product in the catalog (relational division).

**Dataset:**
```sql
CREATE TABLE products27 (pid VARCHAR(5));
CREATE TABLE purchases27 (customer VARCHAR(5), pid VARCHAR(5));

INSERT INTO products27 VALUES ('P1'),('P2'),('P3');
INSERT INTO purchases27 VALUES
('C1','P1'),('C1','P2'),('C1','P3'),
('C2','P1'),('C2','P2'),
('C3','P1'),('C3','P2'),('C3','P3');
```

**Expected Output:**
```
customer
C1
C3
```

**Solution:**
```sql
SELECT customer
FROM purchases27
GROUP BY customer
HAVING COUNT(DISTINCT pid) = (SELECT COUNT(*) FROM products27);
```
> Classic division: a customer covers the whole catalog when their count of *distinct* products bought equals the catalog size. `DISTINCT` guards against duplicate purchases inflating the count.

---

<a id="q28"></a>
## Q28. List every employee who earns strictly more than their direct manager (self-join).

**Dataset:**
```sql
CREATE TABLE emp28 (id INT, name VARCHAR(10), salary INT, mgr_id INT);

INSERT INTO emp28 VALUES
(1,'CEO',100000,NULL),(2,'Alice',80000,1),(3,'Bob',120000,1),
(4,'Carol',60000,2),(5,'Dan',90000,2);
```

**Expected Output:**
```
emp_name | emp_salary | mgr_name | mgr_salary
Bob      | 120000     | CEO      | 100000
Dan      | 90000      | Alice    | 80000
```

**Solution:**
```sql
SELECT e.name AS emp_name, e.salary AS emp_salary,
       m.name AS mgr_name, m.salary AS mgr_salary
FROM emp28 e
JOIN emp28 m ON e.mgr_id = m.id
WHERE e.salary > m.salary;
```
> Join the table to itself: alias `e` for the employee, `m` for the manager via `e.mgr_id = m.id`. The `INNER JOIN` naturally drops the CEO (NULL manager).

---

<a id="q29"></a>
## Q29. Find the maximum number of sessions that were active at the same instant (sweep line).

**Dataset:**
```sql
CREATE TABLE sessions29 (id INT, start_time DATETIME, end_time DATETIME);

INSERT INTO sessions29 VALUES
(1,'2024-05-01 09:00:00','2024-05-01 10:00:00'),
(2,'2024-05-01 09:30:00','2024-05-01 10:30:00'),
(3,'2024-05-01 09:45:00','2024-05-01 11:00:00'),
(4,'2024-05-01 11:30:00','2024-05-01 12:00:00');
```

**Expected Output:**
```
max_concurrent
3
```

**Solution:**
```sql
WITH ev AS (
  SELECT start_time AS t,  1 AS d FROM sessions29
  UNION ALL
  SELECT end_time,        -1     FROM sessions29
)
SELECT MAX(running) AS max_concurrent
FROM (
  SELECT SUM(d) OVER (ORDER BY t, d DESC) AS running
  FROM ev
) x;
```
> Turn each session into a +1 "start" event and a −1 "end" event, order all events on the timeline, and take a running sum. Its peak is the max overlap. `d DESC` breaks ties so a start counts before an end at the same instant.

---

<a id="q30"></a>
## Q30. Unpivot a wide quarterly table (one column per quarter) into tall (product, quarter, sales) rows.

**Dataset:**
```sql
CREATE TABLE quarterly30 (product VARCHAR(5), q1 INT, q2 INT, q3 INT, q4 INT);

INSERT INTO quarterly30 VALUES ('A',10,20,30,40),('B',5,0,15,0);
```

**Expected Output:**
```
product | quarter | sales
A       | Q1      | 10
A       | Q2      | 20
A       | Q3      | 30
A       | Q4      | 40
B       | Q1      | 5
B       | Q2      | 0
B       | Q3      | 15
B       | Q4      | 0
```

**Solution:**
```sql
SELECT product, 'Q1' AS quarter, q1 AS sales FROM quarterly30
UNION ALL SELECT product, 'Q2', q2 FROM quarterly30
UNION ALL SELECT product, 'Q3', q3 FROM quarterly30
UNION ALL SELECT product, 'Q4', q4 FROM quarterly30
ORDER BY product, quarter;
```
> MySQL has no `UNPIVOT`; the idiom is one `SELECT` per source column stitched with `UNION ALL`, each emitting a literal label plus that column's value.

---

<a id="q31"></a>
## Q31. For each customer, find the category they order most often; break ties alphabetically (statistical mode).

**Dataset:**
```sql
CREATE TABLE orders31 (customer VARCHAR(5), category VARCHAR(10));

INSERT INTO orders31 VALUES
('C1','Books'),('C1','Books'),('C1','Toys'),
('C2','Food'),('C2','Food'),('C2','Toys'),('C2','Toys');
```

**Expected Output:**
```
customer | top_category | times
C1       | Books        | 2
C2       | Food         | 2
```

**Solution:**
```sql
WITH cnt AS (
  SELECT customer, category, COUNT(*) AS c,
    ROW_NUMBER() OVER (
      PARTITION BY customer
      ORDER BY COUNT(*) DESC, category
    ) AS rn
  FROM orders31
  GROUP BY customer, category
)
SELECT customer, category AS top_category, c AS times
FROM cnt
WHERE rn = 1;
```
> Count per (customer, category), then `ROW_NUMBER` ordered by count desc with `category` as the tie-breaker (C2 ties Food/Toys → alphabetical Food wins). Keep `rn = 1`.

---

<a id="q32"></a>
## Q32. For each order show days since the customer's previous order, and flag gaps longer than 30 days.

**Dataset:**
```sql
CREATE TABLE orders32 (order_id INT, customer VARCHAR(5), order_date DATE);

INSERT INTO orders32 VALUES
(1,'C1','2024-01-01'),(2,'C1','2024-01-10'),(3,'C1','2024-02-20'),
(4,'C2','2024-03-01');
```

**Expected Output:**
```
customer | order_date | days_since_prev | gap_over_30
C1       | 2024-01-01 | NULL            | N
C1       | 2024-01-10 | 9               | N
C1       | 2024-02-20 | 41              | Y
C2       | 2024-03-01 | NULL            | N
```

**Solution:**
```sql
SELECT customer, order_date,
  DATEDIFF(order_date,
           LAG(order_date) OVER (PARTITION BY customer ORDER BY order_date)) AS days_since_prev,
  CASE WHEN DATEDIFF(order_date,
           LAG(order_date) OVER (PARTITION BY customer ORDER BY order_date)) > 30
       THEN 'Y' ELSE 'N' END AS gap_over_30
FROM orders32
ORDER BY customer, order_date;
```
> `LAG` fetches the prior order date within each customer; `DATEDIFF` gives the gap. The first order per customer has no predecessor → NULL → flag `N`.

---

<a id="q33"></a>
## Q33. Show the running balance of each account after every transaction (signed amounts).

**Dataset:**
```sql
CREATE TABLE txns33 (id INT, account VARCHAR(5), txn_date DATE, amount INT);

INSERT INTO txns33 VALUES
(1,'A','2024-01-01',1000),(2,'A','2024-01-05',-200),(3,'A','2024-01-10',500);
```

**Expected Output:**
```
account | txn_date   | amount | running_balance
A       | 2024-01-01 | 1000   | 1000
A       | 2024-01-05 | -200   | 800
A       | 2024-01-10 | 500    | 1300
```

**Solution:**
```sql
SELECT account, txn_date, amount,
  SUM(amount) OVER (PARTITION BY account ORDER BY txn_date, id) AS running_balance
FROM txns33
ORDER BY account, txn_date, id;
```
> A windowed `SUM(...) OVER (ORDER BY ...)` accumulates. Add `id` to the order so same-day transactions have a deterministic sequence.

---

<a id="q34"></a>
## Q34. Return each customer's **2nd** order date (customers with fewer than 2 orders are excluded).

**Dataset:**
```sql
CREATE TABLE orders34 (customer VARCHAR(5), order_date DATE);

INSERT INTO orders34 VALUES
('C1','2024-01-01'),('C1','2024-01-05'),('C1','2024-01-09'),
('C2','2024-02-01'),
('C3','2024-03-01'),('C3','2024-03-15');
```

**Expected Output:**
```
customer | second_order_date
C1       | 2024-01-05
C3       | 2024-03-15
```

**Solution:**
```sql
WITH r AS (
  SELECT customer, order_date,
    ROW_NUMBER() OVER (PARTITION BY customer ORDER BY order_date) AS rn
  FROM orders34
)
SELECT customer, order_date AS second_order_date
FROM r
WHERE rn = 2;
```
> Number each customer's orders chronologically; `rn = 2` isolates the second. Customers with a single order never produce an `rn = 2` row, so they drop out automatically.

---

<a id="q35"></a>
## Q35. List the products that have never appeared in the sales table (anti-join).

**Dataset:**
```sql
CREATE TABLE products35 (pid VARCHAR(5), pname VARCHAR(10));
CREATE TABLE sales35 (pid VARCHAR(5));

INSERT INTO products35 VALUES ('P1','Pen'),('P2','Book'),('P3','Bag'),('P4','Cup');
INSERT INTO sales35 VALUES ('P1'),('P3'),('P1');
```

**Expected Output:**
```
pid | pname
P2  | Book
P4  | Cup
```

**Solution:**
```sql
SELECT p.pid, p.pname
FROM products35 p
WHERE NOT EXISTS (SELECT 1 FROM sales35 s WHERE s.pid = p.pid);
```
> `NOT EXISTS` is the safest anti-join (immune to NULL surprises and duplicate sales rows). `LEFT JOIN ... WHERE s.pid IS NULL` is an equivalent alternative.

---

<a id="q36"></a>
## Q36. For every employee, count **all** subordinates beneath them (direct + indirect) using a recursive CTE.

**Dataset:**
```sql
CREATE TABLE emp36 (id INT, name VARCHAR(10), mgr_id INT);

INSERT INTO emp36 VALUES
(1,'CEO',NULL),(2,'Alice',1),(3,'Bob',1),(4,'Carol',2),(5,'Dan',2),(6,'Eve',4);
```

**Expected Output:**
```
name  | total_subordinates
CEO   | 5
Alice | 3
Carol | 1
Bob   | 0
Dan   | 0
Eve   | 0
```

**Solution:**
```sql
WITH RECURSIVE sub AS (
  SELECT id AS root, id AS node FROM emp36
  UNION ALL
  SELECT s.root, e.id
  FROM sub s
  JOIN emp36 e ON e.mgr_id = s.node
)
SELECT e.name, COUNT(*) - 1 AS total_subordinates
FROM sub
JOIN emp36 e ON e.id = sub.root
GROUP BY sub.root, e.name
ORDER BY total_subordinates DESC, e.name;
```
> Seed every node as its own `root`, then walk downward. Each (root, node) pair is one reachable descendant; grouping by root and subtracting 1 (to exclude the node itself) gives the subtree size.

---

<a id="q37"></a>
## Q37. Explode a bill of materials: how many of each component are needed to build one unit of assembly **A** (quantities multiply down the tree)?

**Dataset:**
```sql
CREATE TABLE bom37 (parent VARCHAR(5), child VARCHAR(5), qty INT);

INSERT INTO bom37 VALUES
('A','B',2),('A','C',3),('B','D',4),('C','D',1);
```

**Expected Output:**
```
component | total_qty
B         | 2
C         | 3
D         | 11
```

**Solution:**
```sql
WITH RECURSIVE explode AS (
  SELECT child, qty FROM bom37 WHERE parent = 'A'
  UNION ALL
  SELECT b.child, e.qty * b.qty
  FROM explode e
  JOIN bom37 b ON b.parent = e.child
)
SELECT child AS component, SUM(qty) AS total_qty
FROM explode
GROUP BY child
ORDER BY child;
```
> Start from A's direct children, then multiply the accumulated quantity by each sub-component's per-unit qty as you descend. D is reached via B (2×4=8) and via C (3×1=3) → 11 total.

---

<a id="q38"></a>
## Q38. Rank products by sales within each month, then report how each product's rank **moved** versus the prior month.

**Dataset:**
```sql
CREATE TABLE msales38 (product VARCHAR(5), mon VARCHAR(7), sales INT);

INSERT INTO msales38 VALUES
('A','2024-01',100),('B','2024-01',80),('C','2024-01',50),
('A','2024-02',60),('B','2024-02',90),('C','2024-02',70);
```

**Expected Output:**
```
product | mon     | curr_rank | prev_rank | rank_change
A       | 2024-02 | 3         | 1         | -2
B       | 2024-02 | 1         | 2         | 1
C       | 2024-02 | 2         | 3         | 1
```

**Solution:**
```sql
WITH ranked AS (
  SELECT product, mon, sales,
    RANK() OVER (PARTITION BY mon ORDER BY sales DESC) AS rnk
  FROM msales38
),
moves AS (
  SELECT product, mon, rnk,
    LAG(rnk) OVER (PARTITION BY product ORDER BY mon) AS prev_rnk
  FROM ranked
)
SELECT product, mon, rnk AS curr_rank, prev_rnk AS prev_rank,
  CAST(prev_rnk AS SIGNED) - CAST(rnk AS SIGNED) AS rank_change
FROM moves
WHERE prev_rnk IS NOT NULL
ORDER BY product;
```
> Rank within month, then `LAG` the rank along each product's timeline. A positive `rank_change` means it climbed. ⚠️ `RANK()` is `BIGINT UNSIGNED` — `CAST(... AS SIGNED)` before subtracting or negatives overflow.

---

<a id="q39"></a>
## Q39. Pivot sales into one column per quarter (missing quarters show 0).

**Dataset:**
```sql
CREATE TABLE salesq39 (product VARCHAR(5), quarter VARCHAR(2), amount INT);

INSERT INTO salesq39 VALUES
('A','Q1',100),('A','Q2',200),('A','Q4',50),
('B','Q1',30),('B','Q3',70);
```

**Expected Output:**
```
product | q1  | q2  | q3 | q4
A       | 100 | 200 | 0  | 50
B       | 30  | 0   | 70 | 0
```

**Solution:**
```sql
SELECT product,
  SUM(CASE WHEN quarter = 'Q1' THEN amount ELSE 0 END) AS q1,
  SUM(CASE WHEN quarter = 'Q2' THEN amount ELSE 0 END) AS q2,
  SUM(CASE WHEN quarter = 'Q3' THEN amount ELSE 0 END) AS q3,
  SUM(CASE WHEN quarter = 'Q4' THEN amount ELSE 0 END) AS q4
FROM salesq39
GROUP BY product
ORDER BY product;
```
> The rows→columns idiom: one conditional `SUM` per target column. `ELSE 0` yields zero (not NULL) for quarters a product never sold in.

---

<a id="q40"></a>
## Q40. Find employees who were absent on 3 or more **consecutive** calendar days at least once.

**Dataset:**
```sql
CREATE TABLE attendance40 (emp VARCHAR(5), att_date DATE, status CHAR(1));

INSERT INTO attendance40 VALUES
('E1','2024-06-01','A'),('E1','2024-06-02','A'),('E1','2024-06-03','A'),
('E2','2024-06-01','A'),('E2','2024-06-02','P'),('E2','2024-06-03','A'),('E2','2024-06-04','A'),
('E3','2024-06-10','A'),('E3','2024-06-11','A'),('E3','2024-06-12','A'),('E3','2024-06-13','A');
```

**Expected Output:**
```
emp | max_absent_streak
E1  | 3
E3  | 4
```

**Solution:**
```sql
WITH a AS (
  SELECT emp, att_date,
    DATE_SUB(att_date,
      INTERVAL ROW_NUMBER() OVER (PARTITION BY emp ORDER BY att_date) DAY) AS grp
  FROM attendance40
  WHERE status = 'A'
),
runs AS (
  SELECT emp, grp, COUNT(*) AS cnt
  FROM a GROUP BY emp, grp
)
SELECT emp, MAX(cnt) AS max_absent_streak
FROM runs
GROUP BY emp
HAVING MAX(cnt) >= 3
ORDER BY emp;
```
> The gaps-and-islands trick on absent rows only: `date − row_number` is constant across a consecutive run, so it labels each island. Count per island, keep employees whose longest island ≥ 3.

---

<a id="q41"></a>
## Q41. Detect double-booked meeting rooms: pairs of bookings for the same room whose time ranges overlap.

**Dataset:**
```sql
CREATE TABLE bookings41 (id INT, room VARCHAR(5), start_time DATETIME, end_time DATETIME);

INSERT INTO bookings41 VALUES
(1,'R1','2024-07-01 09:00:00','2024-07-01 10:00:00'),
(2,'R1','2024-07-01 09:30:00','2024-07-01 10:30:00'),
(3,'R1','2024-07-01 10:30:00','2024-07-01 11:00:00'),
(4,'R2','2024-07-01 09:00:00','2024-07-01 10:00:00');
```

**Expected Output:**
```
room | booking1 | booking2
R1   | 1        | 2
```

**Solution:**
```sql
SELECT a.room, a.id AS booking1, b.id AS booking2
FROM bookings41 a
JOIN bookings41 b
  ON a.room = b.room
  AND a.id < b.id
  AND a.start_time < b.end_time
  AND b.start_time < a.end_time
ORDER BY a.room, a.id, b.id;
```
> Two intervals overlap iff each starts before the other ends. `a.id < b.id` reports each colliding pair once. Booking 3 (starts exactly when 2 ends) does **not** overlap — end is treated as exclusive.

---

<a id="q42"></a>
## Q42. Find customers active in **every** month between their first and last active month (no gap in the middle).

**Dataset:**
```sql
CREATE TABLE active42 (customer VARCHAR(5), mon VARCHAR(7));

INSERT INTO active42 VALUES
('C1','2024-01'),('C1','2024-02'),('C1','2024-03'),
('C2','2024-01'),('C2','2024-03'),
('C3','2024-01'),('C3','2024-02'),('C3','2024-03');
```

**Expected Output:**
```
customer
C1
C3
```

**Solution:**
```sql
SELECT customer
FROM active42
GROUP BY customer
HAVING COUNT(DISTINCT mon) =
       PERIOD_DIFF(MAX(REPLACE(mon,'-','')), MIN(REPLACE(mon,'-',''))) + 1
ORDER BY customer;
```
> Contiguity check: the count of distinct months must equal the span (`PERIOD_DIFF` of last vs first `YYYYMM`, +1). C2 spans Jan→Mar (3 months) but has only 2 → gap → excluded.

---

<a id="q43"></a>
## Q43. Return the top 3 products by revenue within each category (ties share a rank).

**Dataset:**
```sql
CREATE TABLE sales43 (category VARCHAR(10), product VARCHAR(5), revenue INT);

INSERT INTO sales43 VALUES
('Elec','P1',500),('Elec','P2',400),('Elec','P3',300),('Elec','P4',100),
('Home','H1',200),('Home','H2',150);
```

**Expected Output:**
```
category | product | revenue | rnk
Elec     | P1      | 500     | 1
Elec     | P2      | 400     | 2
Elec     | P3      | 300     | 3
Home     | H1      | 200     | 1
Home     | H2      | 150     | 2
```

**Solution:**
```sql
WITH r AS (
  SELECT category, product, revenue,
    DENSE_RANK() OVER (PARTITION BY category ORDER BY revenue DESC) AS rnk
  FROM sales43
)
SELECT category, product, revenue, rnk
FROM r
WHERE rnk <= 3
ORDER BY category, rnk;
```
> `DENSE_RANK` per category, filter `rnk <= 3`. Use `DENSE_RANK` (not `ROW_NUMBER`) so tied revenues all make the cut instead of being arbitrarily truncated.

---

<a id="q44"></a>
## Q44. For each active date, show the cumulative number of **distinct** users seen so far (each user counted once, on their first-ever day).

**Dataset:**
```sql
CREATE TABLE events44 (user_id INT, event_date DATE);

INSERT INTO events44 VALUES
(1,'2024-01-01'),(2,'2024-01-01'),
(2,'2024-01-02'),
(3,'2024-01-03'),
(1,'2024-01-04');
```

**Expected Output:**
```
event_date | cumulative_users
2024-01-01 | 2
2024-01-02 | 2
2024-01-03 | 3
2024-01-04 | 3
```

**Solution:**
```sql
WITH firsts AS (
  SELECT user_id, MIN(event_date) AS fd FROM events44 GROUP BY user_id
),
per_day AS (
  SELECT fd AS d, COUNT(*) AS new_users FROM firsts GROUP BY fd
),
alldays AS (SELECT DISTINCT event_date AS d FROM events44)
SELECT a.d AS event_date,
  SUM(COALESCE(p.new_users, 0)) OVER (ORDER BY a.d) AS cumulative_users
FROM alldays a
LEFT JOIN per_day p ON p.d = a.d
ORDER BY a.d;
```
> Collapse each user to their first-seen date, count first-appearances per day, then take a running sum. Counting only first appearances is what makes the cumulative total a distinct-user count.

---

<a id="q45"></a>
## Q45. For each user, find the longest gap (in days) between two consecutive logins.

**Dataset:**
```sql
CREATE TABLE logins45 (user_id INT, login_date DATE);

INSERT INTO logins45 VALUES
(1,'2024-01-01'),(1,'2024-01-05'),(1,'2024-01-06'),
(2,'2024-02-01'),(2,'2024-02-10');
```

**Expected Output:**
```
user_id | longest_gap_days
1       | 4
2       | 9
```

**Solution:**
```sql
WITH g AS (
  SELECT user_id,
    DATEDIFF(login_date,
             LAG(login_date) OVER (PARTITION BY user_id ORDER BY login_date)) AS gap
  FROM logins45
)
SELECT user_id, MAX(gap) AS longest_gap_days
FROM g
GROUP BY user_id;
```
> `LAG` gives the previous login; `DATEDIFF` the gap; `MAX` the biggest. The first login per user is NULL and `MAX` ignores it.

---

<a id="q46"></a>
## Q46. Keep only the latest row per entity; report the older rows that should be deleted.

**Dataset:**
```sql
CREATE TABLE records46 (id INT, entity VARCHAR(5), updated_at DATE, val VARCHAR(5));

INSERT INTO records46 VALUES
(1,'E1','2024-01-01','x'),
(2,'E1','2024-01-03','y'),
(3,'E2','2024-01-02','z'),
(4,'E1','2024-01-02','w');
```

**Expected Output:**
```
id_to_delete | entity
1            | E1
4            | E1
```

**Solution:**
```sql
WITH r AS (
  SELECT id, entity,
    ROW_NUMBER() OVER (PARTITION BY entity ORDER BY updated_at DESC, id DESC) AS rn
  FROM records46
)
SELECT id AS id_to_delete, entity
FROM r
WHERE rn > 1
ORDER BY id;
```
> Rank rows per entity newest-first; `rn = 1` is the keeper, everything else (`rn > 1`) is stale. `id DESC` is a deterministic tie-break for identical timestamps.

---

<a id="q47"></a>
## Q47. Find the first (smallest) unoccupied seat number in a mostly-contiguous block.

**Dataset:**
```sql
CREATE TABLE seats47 (seat_no INT);

INSERT INTO seats47 VALUES (1),(2),(3),(5),(6);
```

**Expected Output:**
```
first_available
4
```

**Solution:**
```sql
SELECT MIN(s.seat_no) + 1 AS first_available
FROM seats47 s
WHERE NOT EXISTS (SELECT 1 FROM seats47 t WHERE t.seat_no = s.seat_no + 1);
```
> Find every seat whose successor is missing (a gap's lower edge); the smallest such seat +1 is the first free number. Seat 3 has no 4 → answer 4.

---

<a id="q48"></a>
## Q48. List all distinct pairs of users who share at least one hobby (each pair once, no self-pairs).

**Dataset:**
```sql
CREATE TABLE uh48 (usr VARCHAR(5), hobby VARCHAR(10));

INSERT INTO uh48 VALUES
('A','Chess'),('A','Music'),('B','Chess'),('C','Music'),('C','Chess');
```

**Expected Output:**
```
user1 | user2
A     | B
A     | C
B     | C
```

**Solution:**
```sql
SELECT DISTINCT a.usr AS user1, b.usr AS user2
FROM uh48 a
JOIN uh48 b ON a.hobby = b.hobby AND a.usr < b.usr
ORDER BY user1, user2;
```
> Self-join on shared hobby. `a.usr < b.usr` kills self-pairs and the mirror duplicate (A-B vs B-A); `DISTINCT` collapses pairs that match on more than one hobby (A & C share both Chess and Music).

---

<a id="q49"></a>
## Q49. For each user, report their first-touch and last-touch marketing channel (attribution).

**Dataset:**
```sql
CREATE TABLE touches49 (usr VARCHAR(5), ts DATETIME, channel VARCHAR(10));

INSERT INTO touches49 VALUES
('U1','2024-03-01 10:00:00','Google'),
('U1','2024-03-01 11:00:00','Email'),
('U1','2024-03-01 12:00:00','Direct'),
('U2','2024-03-01 09:00:00','Facebook'),
('U2','2024-03-01 10:00:00','Google');
```

**Expected Output:**
```
usr | first_channel | last_channel
U1  | Google        | Direct
U2  | Facebook      | Google
```

**Solution:**
```sql
WITH r AS (
  SELECT usr, channel,
    ROW_NUMBER() OVER (PARTITION BY usr ORDER BY ts)      AS rn_asc,
    ROW_NUMBER() OVER (PARTITION BY usr ORDER BY ts DESC) AS rn_desc
  FROM touches49
)
SELECT usr,
  MAX(CASE WHEN rn_asc  = 1 THEN channel END) AS first_channel,
  MAX(CASE WHEN rn_desc = 1 THEN channel END) AS last_channel
FROM r
GROUP BY usr;
```
> Number each user's touches both ascending and descending by time; `rn_asc = 1` is the first touch, `rn_desc = 1` the last. Conditional `MAX` pivots both onto one row.

---

<a id="q50"></a>
## Q50. Show total sales per region plus a grand-total row, using `WITH ROLLUP`.

**Dataset:**
```sql
CREATE TABLE rsales50 (region VARCHAR(10), product VARCHAR(5), amount INT);

INSERT INTO rsales50 VALUES
('North','A',100),('North','B',200),('South','A',50);
```

**Expected Output:**
```
region | total_sales
North  | 300
South  | 50
TOTAL  | 350
```

**Solution:**
```sql
SELECT COALESCE(region, 'TOTAL') AS region, SUM(amount) AS total_sales
FROM rsales50
GROUP BY region WITH ROLLUP;
```
> `WITH ROLLUP` appends a super-aggregate row where the grouping column is NULL; `COALESCE` labels that NULL as `TOTAL`.

---

<div align="center">

**50 hard problems, 50 distinct patterns.** Part 1 (Q1–Q25): solve-yourself. Part 2 (Q26–Q50): worked & verified solutions. ⭐ the repo to follow along.

[⬅ Back to Course Home](../README.md) · [Problem Set 1](problem-set-01.md) · [Problem Set 2](problem-set-02.md)

</div>
