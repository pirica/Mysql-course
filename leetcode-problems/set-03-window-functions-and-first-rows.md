# 🟦 LeetCode SQL 50 — Set 3: Window Functions & First-Row Logic

> **Problems 21–30 of the [LeetCode SQL 50](https://leetcode.com/studyplan/top-sql-50/) study plan.**
> Every problem follows the same flow: **Problem → Schema → Sample Input → Expected Output → Approach → Solution → Explanation.**

Set 2 was joins meeting `GROUP BY`. **This set is dominated by one question: "which row is the first one?"** — the first order per customer, the first login per player, the first sales year per product. That single shape (`ROW_NUMBER()`/`DENSE_RANK()` inside a subquery, filter on rank outside) shows up in interviews more than any other window-function pattern, because it's the honest way to answer *"what did each user do first?"*

The other half of the set is **counting with intent**: `COUNT(DISTINCT col)` when duplicates are noise, `COUNT(*)` when they're signal, and `HAVING COUNT(...) = (SELECT COUNT(*) ...)` for relational division — the "bought all products" shape.

All queries target **MySQL 8.0+** and were executed against LeetCode's own sample data on a local MySQL 9.6 server — the Expected Output blocks below are real query output, not hand-written. Where two solutions look equivalent but diverge on data LeetCode's samples don't contain, the divergence is shown with real output too.

---

## 📋 Contents

| # | Problem | Difficulty | Core Concept |
|---|---------|:---:|--------------|
| 1 | [Immediate Food Delivery II](#q1--immediate-food-delivery-ii) | 🟨 Medium | First row per group, then a rate |
| 2 | [Game Play Analysis IV](#q2--game-play-analysis-iv) | 🟨 Medium | `ROW_NUMBER()` + `LEAD()` in one pass |
| 3 | [Number of Unique Subjects Taught by Each Teacher](#q3--number-of-unique-subjects-taught-by-each-teacher) | 🟩 Easy | `COUNT(DISTINCT col)` |
| 4 | [User Activity for the Past 30 Days I](#q4--user-activity-for-the-past-30-days-i) | 🟩 Easy | Inclusive date windows — the off-by-one |
| 5 | [Product Sales Analysis III](#q5--product-sales-analysis-iii) | 🟨 Medium | `DENSE_RANK()` vs `ROW_NUMBER()` on ties |
| 6 | [Classes With at Least 5 Students](#q6--classes-with-at-least-5-students) | 🟩 Easy | `HAVING COUNT(*) >= n` |
| 7 | [Find Followers Count](#q7--find-followers-count) | 🟩 Easy | The baseline `GROUP BY` + `ORDER BY` |
| 8 | [Biggest Single Number](#q8--biggest-single-number) | 🟩 Easy | `MAX` over a `HAVING`-filtered set, `NULL` when empty |
| 9 | [Customers Who Bought All Products](#q9--customers-who-bought-all-products) | 🟨 Medium | Relational division |
| 10 | [The Number of Employees Which Report to Each Employee](#q10--the-number-of-employees-which-report-to-each-employee) | 🟩 Easy | Self-join + `ROUND(AVG())` |

---

## Q1 — Immediate Food Delivery II

🔗 **[leetcode.com/problems/immediate-food-delivery-ii](https://leetcode.com/problems/immediate-food-delivery-ii/description/)**

**Problem:** An order is **immediate** if `order_date = customer_pref_delivery_date`, otherwise **scheduled**. Considering **only each customer's first order**, report the percentage of immediate orders, rounded to 2 decimals.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Delivery;
CREATE TABLE Delivery (
    delivery_id                 INT PRIMARY KEY,
    customer_id                 INT,
    order_date                  DATE,
    customer_pref_delivery_date DATE
);

INSERT INTO Delivery VALUES
(1,1,'2019-08-01','2019-08-02'),
(2,2,'2019-08-02','2019-08-02'),
(3,1,'2019-08-11','2019-08-12'),
(4,3,'2019-08-24','2019-08-24'),
(5,3,'2019-08-21','2019-08-22'),
(6,2,'2019-08-11','2019-08-13'),
(7,4,'2019-08-09','2019-08-09');
```

### ✅ Expected Output

| immediate_percentage |
|----------------------|
| 50.00 |

### 🎯 How to approach it

**Two independent steps — resist the urge to fuse them.**

1. **Reduce 7 rows to 4** — one first order per customer. Customer 1's first is 2019-08-01 (scheduled), customer 2's is 2019-08-02 (immediate), customer 3's is 2019-08-21 (scheduled), customer 4's is 2019-08-09 (immediate).
2. **Compute the rate over those 4 rows.** 2 immediate ÷ 4 = 50%.

The reason step 1 needs a subquery is the rule from [Set 1 Q9](set-01-easy-basics.md#q9--rising-temperature): **you cannot filter on a window function in the same `SELECT` that defines it.** `WHERE` executes before the window functions do, so `WHERE ROW_NUMBER() OVER (...) = 1` is a syntax error. Rank in an inner query, filter in the outer one.

For step 2, `SUM(order_date = customer_pref_delivery_date)` is the conditional-count trick from [Set 2 Q4](set-02-joins-and-aggregation.md#q4--confirmation-rate): the comparison yields `1`/`0`, so `SUM` counts matches. **Multiply by 100 *inside* the `ROUND`** — `ROUND(x, 2) * 100` would throw away the precision you're rounding to.

### 💡 Solution

```sql
SELECT ROUND(SUM(order_date = customer_pref_delivery_date) * 100 / COUNT(*), 2) AS immediate_percentage
FROM (
    SELECT customer_id,
           order_date,
           customer_pref_delivery_date,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date) AS ranking
    FROM Delivery
) t
WHERE ranking = 1;
```

### 🧠 Explanation

- **`PARTITION BY customer_id ORDER BY order_date`** restarts the numbering for every customer and orders each partition chronologically, so `ranking = 1` is *that customer's* earliest order. `PARTITION BY` is `GROUP BY` that doesn't collapse rows — you keep all 7 rows and gain a label, then throw away the 3 you don't want.
- **`COUNT(*)` here counts customers, not orders**, because after `WHERE ranking = 1` there is exactly one row per customer. Getting the denominator right is the whole problem: run the same aggregate over the raw table and you get 3/7 = 42.86%, which is the answer to a different question.
- **`SUM(condition)` relies on MySQL's boolean-to-integer coercion.** The portable form is `SUM(CASE WHEN order_date = customer_pref_delivery_date THEN 1 ELSE 0 END)` — identical result, and what you'd write in PostgreSQL (`COUNT(*) FILTER (WHERE ...)`) or SQL Server.

### 🔄 Alternative — tuple `IN` against the per-customer minimum

```sql
SELECT ROUND(SUM(order_date = customer_pref_delivery_date) * 100 / COUNT(*), 2) AS immediate_percentage
FROM Delivery
WHERE (order_date, customer_id) IN (
    SELECT MIN(order_date), customer_id
    FROM Delivery
    GROUP BY customer_id
);
```

Same 50.00 on this data, and it's a genuinely nice trick — **MySQL supports row constructors in `IN`**, so `(a, b) IN (SELECT x, y ...)` matches on the pair rather than requiring a join. It's shorter than the window version and works on MySQL 5.7, where `ROW_NUMBER()` doesn't exist.

⚠️ **But the two are not equivalent.** `MIN(order_date)` identifies a **date**, not a **row**. If a customer places two orders on their first day, both match the tuple and both land in the denominator. Adding one such row (delivery 8: customer 4, ordered *and* preferred differently on 2019-08-09) to the sample data:

| Query | Result |
|---|---|
| `ROW_NUMBER()` version | **50.00** ✅ (still 4 first orders) |
| Tuple `IN` version | **40.00** ❌ (5 rows in the denominator) |

LeetCode's hidden tests don't appear to contain that case, so both get accepted — but **"first row" and "minimum value of the ordering column" are different things**, and a tie-breaker (`ORDER BY order_date, delivery_id`) is what makes the window version deterministic. Say that out loud in an interview and you've answered the follow-up before it's asked.

---

## Q2 — Game Play Analysis IV

🔗 **[leetcode.com/problems/game-play-analysis-iv](https://leetcode.com/problems/game-play-analysis-iv/description/)**

**Problem:** Report the **fraction of players that logged in again on the day immediately after their first login**, rounded to 2 decimals. The denominator is *all* players.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Activity;
CREATE TABLE Activity (
    player_id    INT,
    device_id    INT,
    event_date   DATE,
    games_played INT,
    PRIMARY KEY (player_id, event_date)
);

INSERT INTO Activity VALUES
(1,2,'2016-03-01',5),
(1,2,'2016-03-02',6),
(2,3,'2017-06-25',1),
(3,1,'2016-03-02',0),
(3,4,'2018-07-03',5);
```

### ✅ Expected Output

| fraction |
|----------|
| 0.33 |

### 🎯 How to approach it

This is **Day-1 retention** — one of the most-asked metrics in product-analytics interviews, and the reason this problem is worth more than its 5 rows suggest.

Player 1 logged in on 03-01 and again on 03-02 → retained. Player 2 logged in once → not retained. Player 3 logged in on 2016-03-02 and next on 2018-07-03 → over two years later, not retained. **1 of 3 players = 0.33.**

The naive route is a self-join: join `Activity` to itself on `player_id` where the right date is the left date + 1, restricted to first logins. That works, but it means scanning the table twice. **The window-function route computes both facts in a single pass:**

- `ROW_NUMBER() OVER (PARTITION BY player_id ORDER BY event_date)` marks the first login.
- `LEAD(event_date) OVER (PARTITION BY player_id ORDER BY event_date)` pulls the *next* login date onto that same row.

Both use the identical `OVER` clause, so MySQL sorts each partition once and evaluates both functions against it. Filter to `ranking = 1` and every surviving row carries "first login date" and "second login date" side by side — the comparison becomes trivial.

**The denominator is the subtle part.** After `WHERE ranking = 1` there's exactly one row per player, so `COUNT(*)` = total distinct players — including players who never came back. That's what the problem asks for. Using `COUNT(next_event_date)` instead would count only players *with* a second login, and the fraction would come out as 1/2, not 1/3.

### 💡 Solution

```sql
SELECT IFNULL(ROUND(SUM(DATEDIFF(next_event_date, event_date) = 1) / COUNT(*), 2), 0) AS fraction
FROM (
    SELECT player_id,
           event_date,
           ROW_NUMBER() OVER (PARTITION BY player_id ORDER BY event_date) AS ranking,
           LEAD(event_date)  OVER (PARTITION BY player_id ORDER BY event_date) AS next_event_date
    FROM Activity
) t
WHERE ranking = 1;
```

### 🧠 Explanation

- **`LEAD()` looks forward, `LAG()` looks back.** Here the first-login row needs the *following* row's date, so it's `LEAD`. (In [Set 1 Q9](set-01-easy-basics.md#q9--rising-temperature), comparing today to yesterday, it was `LAG`.)
- **Players with one login get `next_event_date = NULL`** — `LEAD` has nothing to fetch. `DATEDIFF(NULL, date)` is `NULL`, the comparison `NULL = 1` is `NULL`, and **`SUM` skips `NULL`s**. So player 2 contributes 0 to the numerator while still counting in `COUNT(*)`. Three-valued logic doing exactly the right thing for once, instead of silently eating rows.
- **`DATEDIFF(a, b) = 1` means "a is one day after b"** — argument order is *later, earlier*, and the result is signed. `DATEDIFF(next, first)` for player 3 is 854, not 1. The equivalent `next_event_date = DATE_ADD(event_date, INTERVAL 1 DAY)` is arguably clearer and, on an indexed column, more optimiser-friendly.
- **`IFNULL(..., 0)` guards the empty table.** With no rows, `COUNT(*)` is 0, the division is by zero, and MySQL returns `NULL` rather than raising an error. Verified on an empty copy of the table: without the wrapper the query returns `NULL`; with it, `0`. Not needed for LeetCode's tests, but it's the difference between a metric that reports "0% retention" and a dashboard tile that renders blank.
- 💡 **The `WHERE ranking = 1` filter belongs outside.** Both window functions must see the *whole* partition to compute correctly — filter to the first row first and `LEAD` would have nothing left to look ahead to.

### 🔄 Alternative — no window functions (MySQL 5.7-compatible)

```sql
SELECT ROUND(COUNT(DISTINCT a.player_id) / (SELECT COUNT(DISTINCT player_id) FROM Activity), 2) AS fraction
FROM Activity a
WHERE (a.player_id, DATE_SUB(a.event_date, INTERVAL 1 DAY)) IN (
    SELECT player_id, MIN(event_date) FROM Activity GROUP BY player_id
);
```

Reads as: "count players who have a login whose *previous* day is their first login." The tuple-`IN` risk from Q1 doesn't apply here — `(player_id, event_date)` is the primary key, so a player's minimum date identifies exactly one row.

---

## Q3 — Number of Unique Subjects Taught by Each Teacher

🔗 **[leetcode.com/problems/number-of-unique-subjects-taught-by-each-teacher](https://leetcode.com/problems/number-of-unique-subjects-taught-by-each-teacher/description/)**

**Problem:** Report how many **unique subjects** each teacher teaches.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Teacher;
CREATE TABLE Teacher (
    teacher_id INT,
    subject_id INT,
    dept_id    INT,
    PRIMARY KEY (subject_id, dept_id)
);

INSERT INTO Teacher VALUES (1,2,3),(1,2,4),(1,3,3),(2,1,1),(2,2,1),(2,3,1),(2,4,1);
```

### ✅ Expected Output

| teacher_id | cnt |
|------------|-----|
| 1 | 2 |
| 2 | 4 |

### 🎯 How to approach it

**The grain of the table is (subject, department), not (teacher, subject)** — and that one sentence is the whole problem. Teacher 1 has three rows, but two of them are subject 2 taught in two different departments. Teaching the same subject to two departments is still *one* subject, so the answer is 2, not 3.

`COUNT(DISTINCT subject_id)` deduplicates within each group. `COUNT(*)` would return 3 and 4.

### 💡 Solution

```sql
SELECT teacher_id,
       COUNT(DISTINCT subject_id) AS cnt
FROM Teacher
GROUP BY teacher_id;
```

### 🧠 Explanation

- **`COUNT(DISTINCT col)` deduplicates *inside* each group**, not across the whole result. Teacher 1's group sees `{2, 2, 3}` → 2; teacher 2's sees `{1, 2, 3, 4}` → 4.
- **Three counting functions, three meanings** — keep them straight and most `GROUP BY` questions answer themselves:

  | Expression | Counts | Teacher 1 |
  |---|---|:---:|
  | `COUNT(*)` | rows, `NULL`s included | 3 |
  | `COUNT(subject_id)` | non-`NULL` values | 3 |
  | `COUNT(DISTINCT subject_id)` | distinct non-`NULL` values | **2** |

- **Cost isn't free.** `DISTINCT` inside an aggregate forces MySQL to materialise the distinct values per group (a temp table or in-memory set). At LeetCode scale it's irrelevant; on a hundred-million-row events table it's the difference between a fast scan and a spill to disk. If a covering index on `(teacher_id, subject_id)` exists, MySQL can walk it in order and dedupe for free — worth knowing when the interviewer asks "and how would this perform?"

---

## Q4 — User Activity for the Past 30 Days I

🔗 **[leetcode.com/problems/user-activity-for-the-past-30-days-i](https://leetcode.com/problems/user-activity-for-the-past-30-days-i/description/)**

**Problem:** Report the **daily active user count** for each day in a 30-day period **ending 2019-07-27 inclusive**. A user is active on a day if they made any activity that day. Days with zero users are omitted.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Activity;
CREATE TABLE Activity (
    user_id       INT,
    session_id    INT,
    activity_date DATE,
    activity_type ENUM('open_session','end_session','scroll_down','send_message')
);

INSERT INTO Activity VALUES
(1,1,'2019-07-20','open_session'),
(1,1,'2019-07-20','scroll_down'),
(1,1,'2019-07-20','end_session'),
(2,4,'2019-07-20','open_session'),
(2,4,'2019-07-21','send_message'),
(2,4,'2019-07-21','end_session'),
(3,2,'2019-07-21','open_session'),
(3,2,'2019-07-21','send_message'),
(3,2,'2019-07-21','end_session'),
(4,3,'2019-06-25','open_session'),
(4,3,'2019-06-25','end_session');
```

### ✅ Expected Output

| day | active_users |
|-----|--------------|
| 2019-07-20 | 2 |
| 2019-07-21 | 2 |

### 🎯 How to approach it

Two traps, both about **counting things once**.

**Trap 1 — the window is 29 days wide, not 30.** "The 30 days ending 2019-07-27, inclusive" spans 2019-06-28 → 2019-07-27. Both endpoints count, so the arithmetic is `INTERVAL 29 DAY`, not 30. Off-by-one on an inclusive range is the single most common date-filter bug in production analytics, and it's a favourite interview trip-wire.

**Trap 2 — one user, many rows.** User 1 has three rows on 2019-07-20 (open, scroll, end) but is **one** active user. `COUNT(*)` would report 3 for that day. `COUNT(DISTINCT user_id)` reports 2 — user 1 and user 2. Same lesson as Q3, different costume.

User 4's session on 2019-06-25 is outside the window (32 days before 2019-07-27) and drops out, which is exactly why the sample data includes it.

### 💡 Solution

```sql
SELECT activity_date          AS day,
       COUNT(DISTINCT user_id) AS active_users
FROM Activity
WHERE activity_date BETWEEN DATE_SUB('2019-07-27', INTERVAL 29 DAY) AND '2019-07-27'
GROUP BY activity_date;
```

### 🧠 Explanation

- **`BETWEEN` is inclusive on both ends**, so pairing it with `INTERVAL 29 DAY` gives exactly 30 calendar days. Verified: `DATE_SUB('2019-07-27', INTERVAL 29 DAY)` = `2019-06-28`; with `INTERVAL 30 DAY` you'd get `2019-06-27` and a 31-day window.
- **`WHERE` filters before `GROUP BY`**, so out-of-window rows never reach the aggregate. This is the right order — it's a row-level predicate, not a group-level one, so `HAVING` would be both wrong stylistically and slower.
- **`DATEDIFF('2019-07-27', activity_date) < 30` is the popular alternative** and produces the same set. It's more readable, but it **wraps the column in a function**, which makes the predicate non-sargable: MySQL must compute `DATEDIFF` for every row instead of seeking an index range on `activity_date`. Keeping the column bare on one side of a comparison is a habit that pays for itself the first time the table has 500 million rows.
- **No `ORDER BY`** — the problem accepts any order. But note the result *came back* sorted, because `GROUP BY` in MySQL 8 often produces grouped order as a side effect of its execution plan. **Never rely on that.** Ordering that isn't requested can vanish when the plan changes.
- 💡 **Days with zero activity are missing entirely**, and the problem explicitly allows that. In a real dashboard you'd need a calendar table `LEFT JOIN`ed against activity — the same `CROSS JOIN`-then-`LEFT JOIN` scaffold as [Set 2 Q2](set-02-joins-and-aggregation.md#q2--students-and-examinations), because a gap in a time series is a `0`, not an absence.

---

## Q5 — Product Sales Analysis III

🔗 **[leetcode.com/problems/product-sales-analysis-iii](https://leetcode.com/problems/product-sales-analysis-iii/description/)**

**Problem:** For each product, report the **quantity and price from the first year it was sold**. Return every sale from that first year.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Sales;
CREATE TABLE Sales (
    sale_id    INT,
    product_id INT,
    year       INT,
    quantity   INT,
    price      INT,
    PRIMARY KEY (sale_id, year)
);

INSERT INTO Sales VALUES (1,100,2008,10,5000),(2,100,2009,12,5000),(7,200,2011,15,9000);

DROP TABLE IF EXISTS Product;
CREATE TABLE Product (
    product_id   INT PRIMARY KEY,
    product_name VARCHAR(20)
);

INSERT INTO Product VALUES (100,'Nokia'),(200,'Apple'),(300,'Samsung');
```

### ✅ Expected Output

| product_id | first_year | quantity | price |
|------------|------------|----------|-------|
| 100 | 2008 | 10 | 5000 |
| 200 | 2011 | 15 | 9000 |

### 🎯 How to approach it

Structurally identical to Q1 — rank rows within each product, keep rank 1 — with **one critical difference in the ranking function.**

Read the requirement precisely: *"return all sales from that first year"*, not *"return one sale."* If a product sold twice in its first year, **both rows belong in the output.**

- `ROW_NUMBER()` assigns 1, 2, 3… with no ties — it would arbitrarily keep one of them and drop the other.
- `DENSE_RANK()` (and `RANK()`) assigns the **same** number to rows tied on the `ORDER BY` key — every sale in the first year gets rank 1, and all of them survive the filter.

The sample data has no such tie, so both pass LeetCode. Adding a second 2008 sale for product 100 (sale 8: 4 units @ 3000) makes them diverge — verified output:

| Function | Rows returned for product 100 |
|---|---|
| `DENSE_RANK()` | **2008/10/5000 and 2008/4/3000** ✅ |
| `ROW_NUMBER()` | 2008/10/5000 only ❌ |

**Pick the ranking function from the requirement's plurality, not from habit.** "The first year's sales" (plural, tie-inclusive) → `DENSE_RANK`/`RANK`. "The first order" (singular, exactly one) → `ROW_NUMBER`.

Note also that `Product` is never joined — the problem asks only for `product_id`. Product 300 has no sales and correctly appears nowhere.

### 💡 Solution

```sql
SELECT product_id,
       year AS first_year,
       quantity,
       price
FROM (
    SELECT product_id,
           year,
           quantity,
           price,
           DENSE_RANK() OVER (PARTITION BY product_id ORDER BY year) AS ranking
    FROM Sales
) t
WHERE ranking = 1;
```

### 🧠 Explanation

- **`RANK()` and `DENSE_RANK()` are interchangeable here.** They differ only in what happens *after* a tie — `RANK` skips (1,1,3), `DENSE_RANK` doesn't (1,1,2) — and you're filtering on rank 1, which both assign identically.
- **`year` needs the `AS first_year` alias.** LeetCode checks column headers; more importantly `year` is a MySQL keyword-adjacent name (`YEAR()` is a function), so aliasing it is good hygiene regardless.
- **The `IN` alternative** — `WHERE (product_id, year) IN (SELECT product_id, MIN(year) FROM Sales GROUP BY product_id)` — is correct here *and* handles the tie case, because it matches on the year rather than picking a row. This is the mirror image of Q1: there, matching on the minimum value was the bug; here it's the fix. **The difference is whether you want "one row" or "all rows at the minimum."**
- 💡 **Same-year duplicates are the interesting case in the real world too.** Products relaunch, prices change mid-year, and a "first year" row set of size >1 is normal — which is why `SELECT DISTINCT` or `ROW_NUMBER` on this shape quietly under-reports revenue.

---

## Q6 — Classes With at Least 5 Students

🔗 **[leetcode.com/problems/classes-with-at-least-5-students](https://leetcode.com/problems/classes-with-at-least-5-students/description/)**

**Problem:** Find all classes that have **at least five students**.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Courses;
CREATE TABLE Courses (
    student VARCHAR(20),
    class   VARCHAR(20),
    PRIMARY KEY (student, class)
);

INSERT INTO Courses VALUES
('A','Math'),('B','English'),('C','Math'),('D','Biology'),('E','Math'),
('F','Computer'),('G','Math'),('H','Math'),('I','Math');
```

### ✅ Expected Output

| class |
|-------|
| Math |

### 🎯 How to approach it

The textbook `HAVING` problem, and worth doing precisely because it's the one people answer *almost* right.

**`WHERE` filters rows; `HAVING` filters groups.** The count doesn't exist until the groups are formed, so the filter has to run after — `WHERE COUNT(*) >= 5` is a hard error, not a style preference.

**"At least five" is `>= 5`.** Not `> 5`. Math has exactly 6 students here so both happen to pass, but hidden tests are built precisely to punish that reading. `>` is "more than"; `>=` is "at least."

### 💡 Solution

```sql
SELECT class
FROM Courses
GROUP BY class
HAVING COUNT(*) >= 5;
```

### 🧠 Explanation

- **`COUNT(*)` is safe here only because `(student, class)` is the primary key** — a student can't be enrolled in the same class twice, so rows-per-class equals students-per-class. `COUNT(DISTINCT student)` is the defensive version and costs nothing to write. An earlier revision of this problem *did* contain duplicate rows, and `COUNT(*)` was the classic wrong answer.
- **Order of execution is the whole lesson:** `FROM` → `WHERE` → `GROUP BY` → `HAVING` → `SELECT` → `ORDER BY`. That sequence explains why `HAVING` can reference aggregates (they exist by then) and why `WHERE` can't, and why `SELECT` aliases are visible to `ORDER BY` but not to `WHERE`. If you can recite it, an entire category of "why doesn't this work" questions collapses.
- 💡 **You can `HAVING` without `SELECT`ing the aggregate.** `COUNT(*)` never appears in the output, and it doesn't need to — a common misconception is that you must select what you filter on.

---

## Q7 — Find Followers Count

🔗 **[leetcode.com/problems/find-followers-count](https://leetcode.com/problems/find-followers-count/description/)**

**Problem:** For each user, report the **number of followers** they have, ordered by `user_id`.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Followers;
CREATE TABLE Followers (
    user_id     INT,
    follower_id INT,
    PRIMARY KEY (user_id, follower_id)
);

INSERT INTO Followers VALUES (0,1),(1,0),(2,0),(2,1);
```

### ✅ Expected Output

| user_id | followers_count |
|---------|-----------------|
| 0 | 1 |
| 1 | 1 |
| 2 | 2 |

### 🎯 How to approach it

The baseline `GROUP BY` — included in the 50 as a breather, but there are two things worth noticing.

**Read the direction of the relationship.** `(0, 1)` means user 1 follows user 0. Grouping by `user_id` counts **followers**; grouping by `follower_id` would count **followees**. Half the wrong answers to graph-ish problems are a reversed edge, and the column names won't always save you.

**`ORDER BY` is explicitly required**, so write it — even though the output happens to come back sorted because `user_id` leads the primary key and MySQL scans the index in order. That's an accident of the plan, not a guarantee.

### 💡 Solution

```sql
SELECT user_id,
       COUNT(*) AS followers_count
FROM Followers
GROUP BY user_id
ORDER BY user_id;
```

### 🧠 Explanation

- **`COUNT(*)` is correct here** — the primary key on `(user_id, follower_id)` makes duplicate edges impossible, so each row is one distinct follower. Without that key, `COUNT(DISTINCT follower_id)` would be the honest choice.
- **Users with zero followers don't appear**, because they have no rows to group. If the problem wanted them at `0`, you'd need a `Users` table and a `LEFT JOIN` — the `COUNT(col)`-not-`COUNT(*)` situation from [Set 2 Q2](set-02-joins-and-aggregation.md#q2--students-and-examinations). Knowing *why* they're absent matters more than the query.
- **`ORDER BY user_id` is free here.** MySQL is already reading the index in `user_id` order to group, so the sort is eliminated by the optimiser — `EXPLAIN` shows no `Using filesort`.

---

## Q8 — Biggest Single Number

🔗 **[leetcode.com/problems/biggest-single-number](https://leetcode.com/problems/biggest-single-number/description/)**

**Problem:** A **single number** appears exactly once in the table. Report the **largest** single number, or `null` if there isn't one.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS MyNumbers;
CREATE TABLE MyNumbers (num INT);

INSERT INTO MyNumbers VALUES (8),(8),(3),(3),(1),(4),(5),(6);
```

### ✅ Expected Output

| num |
|-----|
| 6 |

### 🎯 How to approach it

Two steps, and the second one does more work than it looks like.

1. **Find the numbers appearing exactly once.** `GROUP BY num HAVING COUNT(*) = 1` → `{1, 4, 5, 6}`. 8 and 3 appear twice, so they're out.
2. **Take the maximum of that set** — 6.

Both steps are aggregates, but at *different grains*: step 1 aggregates per number, step 2 aggregates across numbers. **Two grains means two query levels** — a subquery (or CTE). You cannot nest aggregates directly; `MAX(COUNT(*))` is an error in MySQL.

**The `null` requirement is handled for free.** If every number is duplicated, the subquery returns zero rows, and `MAX()` over an empty set is `NULL` — not an error, not 0. Verified against `(8,8,7,7,3,3,3)`: the query returns `NULL`, exactly as the problem requires. **This only works because `MAX` is in the outer query.** Add a `LIMIT 1`-and-`ORDER BY` approach instead and the empty case returns *no row at all*, which LeetCode marks wrong — an empty result set is not the same as a row containing `NULL`.

### 💡 Solution

```sql
SELECT MAX(num) AS num
FROM (
    SELECT num
    FROM MyNumbers
    GROUP BY num
    HAVING COUNT(*) = 1
) t;
```

### 🧠 Explanation

- **An aggregate with no `GROUP BY` always returns exactly one row**, even over an empty input. That property is the entire reason this solution satisfies the `null` case without an `IFNULL` — and it's worth internalising, because it's also why `SELECT COUNT(*) FROM empty_table` gives `0` rather than nothing.
- **The derived table needs an alias** (`t`). MySQL rejects an unaliased subquery in `FROM` with "Every derived table must have its own alias" — a two-second fix that reliably costs people a minute.
- **The CTE form reads better** and compiles identically:

  ```sql
  WITH singles AS (
      SELECT num FROM MyNumbers GROUP BY num HAVING COUNT(*) = 1
  )
  SELECT MAX(num) AS num FROM singles;
  ```

- 💡 **`HAVING COUNT(*) = 1` is the deduplication primitive.** Flip it to `> 1` and the same shape finds duplicate emails, duplicate orders, or broken uniqueness constraints — see [Problem Set 3](../interview-questions/problem-set-03.md) for the find-and-delete-duplicates variant.

---

## Q9 — Customers Who Bought All Products

🔗 **[leetcode.com/problems/customers-who-bought-all-products](https://leetcode.com/problems/customers-who-bought-all-products/description/)**

**Problem:** Report the customers who bought **all** the products in the `Product` table.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Customer;
CREATE TABLE Customer (
    customer_id INT,
    product_key INT
);

INSERT INTO Customer VALUES (1,5),(2,6),(3,5),(3,6),(1,6);

DROP TABLE IF EXISTS Product;
CREATE TABLE Product (
    product_key INT PRIMARY KEY
);

INSERT INTO Product VALUES (5),(6);
```

### ✅ Expected Output

| customer_id |
|-------------|
| 1 |
| 3 |

### 🎯 How to approach it

This is **relational division** — "find the entities related to *every* member of a set" — and it has a name because it comes up constantly: users who completed all onboarding steps, students who passed all exams, servers with every required patch.

The counting solution is the one to reach for first because it's the one you can write from scratch under pressure:

1. **How many products exist?** `SELECT COUNT(*) FROM Product` → 2. A scalar subquery, evaluated once.
2. **How many distinct products did each customer buy?** `COUNT(DISTINCT product_key)` per customer.
3. **Keep the customers where those two numbers match.** Customer 1 bought {5,6} → 2 = 2 ✅. Customer 2 bought {6} → 1 ≠ 2 ❌. Customer 3 bought {5,6} → 2 ✅.

**`DISTINCT` is load-bearing, not decoration.** `Customer` has no primary key — the problem explicitly says it may contain duplicates. Add one repeat purchase (customer 2 buys product 6 twice) and the two versions split:

| Aggregate | Result |
|---|---|
| `COUNT(DISTINCT product_key)` | **1, 3** ✅ |
| `COUNT(*)` | 1, **2**, 3 ❌ — customer 2 counted twice for one product |

That's verified output, and it's the single mistake this problem is designed to catch: **a customer who buys one product twice looks identical to a customer who bought two products** unless you deduplicate.

### 💡 Solution

```sql
SELECT customer_id
FROM Customer
GROUP BY customer_id
HAVING COUNT(DISTINCT product_key) = (
    SELECT COUNT(*)
    FROM Product
);
```

### 🧠 Explanation

- **The scalar subquery in `HAVING` is uncorrelated** — it doesn't reference the outer query, so MySQL evaluates it once and reuses the constant `2` for every group. No per-row cost.
- **This assumes every `Customer.product_key` exists in `Product`.** If a customer bought a discontinued product not in the catalogue, `COUNT(DISTINCT product_key)` could exceed the catalogue size and the `=` would fail. A `JOIN Product USING (product_key)` before grouping makes the query robust — and `>=` instead of `=` is the lazier hedge.
- **The `NOT EXISTS` alternative** expresses the logic literally — *"no product exists that this customer didn't buy"* — as a double negation:

  ```sql
  SELECT DISTINCT c.customer_id
  FROM Customer c
  WHERE NOT EXISTS (
      SELECT 1 FROM Product p
      WHERE NOT EXISTS (
          SELECT 1 FROM Customer c2
          WHERE c2.customer_id = c.customer_id AND c2.product_key = p.product_key
      )
  );
  ```

  It's the canonical textbook answer and it's immune to the duplicate problem entirely. It's also harder to read and usually slower in MySQL. **Write the counting version; mention this one.** Interviewers asking about relational division are usually checking whether you know the name and the shape, not testing your nested-`NOT EXISTS` stamina.

- 💡 The same problem appears in [Problem Set 6 Q2](../interview-questions/problem-set-06.md#q2--customers-who-purchased-all-products) with a different dataset — worth solving both ways to see which one you reach for naturally.

---

## Q10 — The Number of Employees Which Report to Each Employee

🔗 **[leetcode.com/problems/the-number-of-employees-which-report-to-each-employee](https://leetcode.com/problems/the-number-of-employees-which-report-to-each-employee/description/)**

**Problem:** For each employee who **has at least one direct report**, report their id, name, number of direct reports, and the **rounded average age** of those reports.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Employees;
CREATE TABLE Employees (
    employee_id INT PRIMARY KEY,
    name        VARCHAR(20),
    reports_to  INT,
    age         INT
);

INSERT INTO Employees VALUES
(1,'Michael',NULL,45),
(2,'Alice',1,38),
(3,'Bob',1,42),
(4,'Charlie',2,34),
(5,'David',2,40),
(6,'Eve',3,37),
(7,'Frank',1,42),
(8,'Grace',3,38);
```

### ✅ Expected Output

| employee_id | name | reports_count | average_age |
|-------------|------|---------------|-------------|
| 1 | Michael | 3 | 41 |
| 2 | Alice | 2 | 37 |
| 3 | Bob | 2 | 38 |

### 🎯 How to approach it

A **self-join** — one table, two roles. Alias it twice: `e` for the employee doing the reporting, `m` for the manager being reported to. The join condition `e.reports_to = m.employee_id` is the org chart.

Then aggregate **by the manager**: `COUNT` the reports, `AVG` their ages.

**Use an `INNER JOIN`, deliberately.** The problem wants only employees who *have* reports. An inner join drops non-managers automatically — no `HAVING COUNT(*) > 0` needed. Switching to `LEFT JOIN Employees m ... e` adds all five leaf employees with `reports_count = 0` and `average_age = NULL`, which is a different (and here, wrong) answer. Verified — the `LEFT JOIN` version returns 8 rows instead of 3.

**`GROUP BY m.employee_id, m.name`, never by name alone.** Two managers named "Alice" would be merged into one row. Grouping by the primary key is always safe; grouping by a display label is a correctness bug waiting for its second data row.

### 💡 Solution

```sql
SELECT m.employee_id        AS employee_id,
       m.name               AS name,
       COUNT(e.employee_id) AS reports_count,
       ROUND(AVG(e.age))    AS average_age
FROM Employees e
JOIN Employees m
    ON e.reports_to = m.employee_id
GROUP BY m.employee_id, m.name
ORDER BY m.employee_id;
```

### 🧠 Explanation

- **Trace one group.** Michael (id 1) is matched by Alice (38), Bob (42), and Frank (42) → count 3, average 40.667 → **41**. Bob (id 3) is matched by Eve (37) and Grace (38) → average **37.5**, which rounds to **38**.
- **`ROUND()` in MySQL rounds half away from zero**, so 37.5 → 38 and 38.5 → 39. This differs from Python's `round()` and from many BI tools, which use banker's rounding (round-half-to-even: 38.5 → 38). If an interviewer asks why the same report disagrees between SQL and the Python notebook, this is usually the answer.
- **`ROUND(AVG(age))` ≠ `AVG(ROUND(age))`.** Round after averaging — the problem asks for the rounded average, not the average of rounded values. Ages are integers here so they agree, but on prices or rates they won't.
- **`COUNT(e.employee_id)` vs `COUNT(*)`** is identical under an `INNER JOIN`, since every row has a matched employee. The habit of counting the joined column instead of rows is still the right one — it's what keeps you correct the day someone converts the join to a `LEFT JOIN`.
- **The hierarchy is only traversed one level.** Michael's count of 3 covers his *direct* reports, not Charlie, David, Eve, and Grace beneath them. Total headcount under a manager needs a `WITH RECURSIVE` walk — see [Recursive CTEs](../docs/12-recursive-cte.md), the natural follow-up question to this exact problem.

---

## 🧠 Set 3 — Patterns Worth Memorising

| Pattern | Trigger phrase | Tool |
|---|---|---|
| The **one** first row per group | "their first order", "the earliest login" | `ROW_NUMBER() OVER (PARTITION BY id ORDER BY d)`, filter `= 1` outside |
| **All** rows tied at the extreme | "all sales in the first year" | `DENSE_RANK()`/`RANK()`, or tuple `IN (SELECT id, MIN(d) …)` |
| Next/previous row's value inline | "logged in again the day after" | `LEAD()` / `LAG()` with the same `OVER` clause |
| Rate over first rows only | "% of customers whose first order was…" | Rank in a subquery → `SUM(cond)*100/COUNT(*)` outside |
| Deduplicate inside a group | "unique subjects", "active users" | `COUNT(DISTINCT col)` |
| Filter on a group's size | "at least 5", "exactly once" | `HAVING COUNT(*) >= n` / `= 1` |
| Relational division | "bought **all** products" | `HAVING COUNT(DISTINCT k) = (SELECT COUNT(*) FROM set)` |
| Max over a filtered group set | "largest number appearing once" | `MAX()` over a `HAVING`-filtered derived table |
| Inclusive N-day window | "the past 30 days ending on D" | `BETWEEN DATE_SUB(D, INTERVAL 29 DAY) AND D` |
| Org chart / one-level hierarchy | "employees who report to…" | Self-join `e.reports_to = m.employee_id`, group by manager **id** |
| Guard a divide-by-zero metric | "return 0 if there are none" | `IFNULL(expr, 0)` around the whole rate |

### Five mistakes that cost the most submissions

1. **Filtering on a window function in the same `SELECT`.** `WHERE` runs before window functions exist. Rank in a subquery/CTE, filter outside. (Q1, Q2, Q5)
2. **`ROW_NUMBER()` where ties must survive.** "All sales from the first year" needs `DENSE_RANK`; `ROW_NUMBER` silently drops rows. Read the requirement's plurality. (Q5)
3. **`COUNT(*)` where the table has duplicates.** One user with three events isn't three users; one product bought twice isn't two products. `COUNT(DISTINCT col)`. (Q3, Q4, Q9)
4. **Off-by-one on inclusive date ranges.** "30 days ending D" is `INTERVAL 29 DAY` with `BETWEEN`. (Q4)
5. **Matching on `MIN(value)` when you meant "the first row".** They differ the moment two rows tie on the ordering column — 50.00 becomes 40.00. (Q1)

---

<div align="center">

**[Continue to Set 4 — Advanced Windows & Conditional Logic ➡](set-04-advanced-windows-and-conditional-logic.md)**

[⬅ Back to Course Home](../README.md) · [Set 1](set-01-easy-basics.md) · [Set 2](set-02-joins-and-aggregation.md) · [Set 4](set-04-advanced-windows-and-conditional-logic.md) · [Set 5](set-05-strings-regex-and-set-operations.md) · [Interview Problem Sets](../interview-questions/) · [LeetCode SQL 50 study plan ↗](https://leetcode.com/studyplan/top-sql-50/)

</div>
