# 🟩 LeetCode SQL 50 — Set 2: Joins & Aggregation

> **Problems 11–20 of the [LeetCode SQL 50](https://leetcode.com/studyplan/top-sql-50/) study plan.**
> Every problem follows the same flow: **Problem → Schema → Sample Input → Expected Output → Approach → Solution → Explanation.**

Set 1 was mostly single-table filtering. **This set is where joins meet `GROUP BY`** — and where the real interview questions live: what does `COUNT` actually count after a `LEFT JOIN`, why do you group by an ID instead of a name, and how do you make a rate come out as `0` instead of `NULL` when the denominator is empty.

All queries target **MySQL 8.0+** and were executed against LeetCode's own sample data — the Expected Output blocks below are real query output. Where a solution passes the sample data but fails a hidden test case, it's flagged.

---

## 📋 Contents

| # | Problem | Difficulty | Core Concept |
|---|---------|:---:|--------------|
| 1 | [Employee Bonus](#q1--employee-bonus) | 🟩 Easy | `LEFT JOIN` + `NULL` in a filter |
| 2 | [Students and Examinations](#q2--students-and-examinations) | 🟩 Easy | `CROSS JOIN` to build every combination |
| 3 | [Managers with at Least 5 Direct Reports](#q3--managers-with-at-least-5-direct-reports) | 🟨 Medium | Self-join + `HAVING`, group by **ID** |
| 4 | [Confirmation Rate](#q4--confirmation-rate) | 🟨 Medium | Conditional rate, `NULL`-safe denominator |
| 5 | [Not Boring Movies](#q5--not-boring-movies) | 🟩 Easy | Modulo for odd/even |
| 6 | [Average Selling Price](#q6--average-selling-price) | 🟩 Easy | Date-range join, weighted average |
| 7 | [Project Employees I](#q7--project-employees-i) | 🟩 Easy | `AVG` + `ROUND` per group |
| 8 | [Percentage of Users Attended a Contest](#q8--percentage-of-users-attended-a-contest) | 🟩 Easy | Scalar subquery as denominator |
| 9 | [Queries Quality and Percentage](#q9--queries-quality-and-percentage) | 🟩 Easy | Two aggregates, one pass ⚠️ hidden `NULL` case |
| 10 | [Monthly Transactions I](#q10--monthly-transactions-i) | 🟨 Medium | Group by a derived month + conditional sums |

---

## Q1 — Employee Bonus

🔗 **[leetcode.com/problems/employee-bonus](https://leetcode.com/problems/employee-bonus/description/)**

**Problem:** Report the name and bonus of every employee whose bonus is **less than 1000**. Employees with **no bonus record** must also appear, with `null` as their bonus.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Employee;
CREATE TABLE Employee (
    empId      INT PRIMARY KEY,
    name       VARCHAR(30),
    supervisor INT,
    salary     INT
);

INSERT INTO Employee VALUES
(3,'Brad',NULL,4000),(1,'John',3,1000),(2,'Dan',3,2000),(4,'Thomas',3,4000);

DROP TABLE IF EXISTS Bonus;
CREATE TABLE Bonus (
    empId INT PRIMARY KEY,
    bonus INT
);

INSERT INTO Bonus VALUES (2,500),(4,2000);
```

### ✅ Expected Output

| name | bonus |
|------|-------|
| John | NULL |
| Dan | 500 |
| Brad | NULL |

### 🎯 How to approach it

Two requirements, and they pull in opposite directions:

1. **"employees with no bonus must appear"** → `LEFT JOIN`, `Employee` on the left. An `INNER JOIN` would drop John and Brad entirely.
2. **"bonus less than 1000"** → but the employees from step 1 have `bonus = NULL`, and **`NULL < 1000` is not `TRUE`** — it's `NULL`, which `WHERE` discards. So a bare `WHERE b.bonus < 1000` silently undoes the `LEFT JOIN` and you're back to only Dan.

This is the **`LEFT JOIN` + `WHERE` trap**, and it's the most common way people accidentally turn an outer join back into an inner one. The fix is to make the `NULL` case explicit:

```
WHERE b.bonus < 1000 OR b.bonus IS NULL
```

Same three-valued-logic lesson as [Set 1 Q2](set-01-easy-basics.md#q2--find-customer-referee), now compounded by a join.

### 💡 Solution

```sql
SELECT e.name  AS name,
       b.bonus AS bonus
FROM Employee e
LEFT JOIN Bonus b
    ON e.empId = b.empId
WHERE b.bonus IS NULL OR b.bonus < 1000;
```

### 🧠 Explanation

- **`LEFT JOIN` keeps all four employees.** Dan (500) and Thomas (2000) find bonus rows; John and Brad don't, so `b.bonus` is `NULL` for them.
- **`b.bonus IS NULL`** rescues John and Brad — exactly the rows the problem wants. **`b.bonus < 1000`** keeps Dan. Thomas is the only one filtered out, because 2000 ≥ 1000.
- **Drop the `IS NULL` half and the output shrinks to just Dan.** Try it — that failure is worth seeing once, because it's what "the `LEFT JOIN` didn't work" almost always turns out to mean.
- 💡 **The alternative fix: move the condition into the `ON` clause.** `ON e.empId = b.empId AND b.bonus < 1000` filters *during* the join, so unmatched employees still survive as `NULL` rows and no `WHERE` is needed. Knowing that **`ON` filters before the join completes while `WHERE` filters after** is a genuine interview differentiator — and it's the reason a filter on the right-hand table of a `LEFT JOIN` belongs in `ON`, not `WHERE`.

---

## Q2 — Students and Examinations

🔗 **[leetcode.com/problems/students-and-examinations](https://leetcode.com/problems/students-and-examinations/description/)**

**Problem:** For **every student and every subject**, report how many times that student attended that subject's exam — including `0` when they never sat it. Order by `student_id`, then `subject_name`.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Students;
CREATE TABLE Students (
    student_id   INT PRIMARY KEY,
    student_name VARCHAR(20)
);

INSERT INTO Students VALUES (1,'Alice'),(2,'Bob'),(13,'John'),(6,'Alex');

DROP TABLE IF EXISTS Subjects;
CREATE TABLE Subjects (
    subject_name VARCHAR(20) PRIMARY KEY
);

INSERT INTO Subjects VALUES ('Math'),('Physics'),('Programming');

DROP TABLE IF EXISTS Examinations;
CREATE TABLE Examinations (
    student_id   INT,
    subject_name VARCHAR(20)
);

INSERT INTO Examinations VALUES
(1,'Math'),(1,'Physics'),(1,'Programming'),(2,'Programming'),(1,'Physics'),
(1,'Math'),(13,'Math'),(13,'Programming'),(13,'Physics'),(2,'Math'),(1,'Math');
```

### ✅ Expected Output

| student_id | student_name | subject_name | attended_exams |
|------------|--------------|--------------|----------------|
| 1 | Alice | Math | 3 |
| 1 | Alice | Physics | 2 |
| 1 | Alice | Programming | 1 |
| 2 | Bob | Math | 1 |
| 2 | Bob | Physics | 0 |
| 2 | Bob | Programming | 1 |
| 6 | Alex | Math | 0 |
| 6 | Alex | Physics | 0 |
| 6 | Alex | Programming | 0 |
| 13 | John | Math | 1 |
| 13 | John | Physics | 1 |
| 13 | John | Programming | 1 |

### 🎯 How to approach it

**The insight: the rows you need to output don't exist in any table.** 4 students × 3 subjects = **12 required rows**, but `Examinations` only has 11 rows covering a subset of those pairs — and Alex appears in none of them.

So the pattern is *generate first, then count*:

1. **Build all 12 pairs.** A `CROSS JOIN` between `Students` and `Subjects` produces every combination — this is the rare case where a Cartesian product is exactly what you want, not a bug.
2. **`LEFT JOIN` the exam records onto those pairs.** The join key is **both** columns: `student_id` *and* `subject_name`. Match on only one and you'll count a student's Math exams under Physics too.
3. **Count the matches per pair** with `GROUP BY student_id, subject_name`.

**The critical detail is step 3's counting function.** Use `COUNT(e.subject_name)`, **not `COUNT(*)`**. After a `LEFT JOIN`, a pair with no exam still produces one row (full of `NULL`s), so `COUNT(*)` would return **1** where the answer must be **0**. `COUNT(column)` skips `NULL`s and correctly returns 0. This single choice is the difference between passing and failing.

### 💡 Solution

```sql
SELECT s.student_id           AS student_id,
       s.student_name         AS student_name,
       sub.subject_name       AS subject_name,
       COUNT(e.subject_name)  AS attended_exams
FROM Students s
CROSS JOIN Subjects sub
LEFT JOIN Examinations e
    ON  s.student_id   = e.student_id
    AND sub.subject_name = e.subject_name
GROUP BY s.student_id, sub.subject_name
ORDER BY s.student_id, sub.subject_name;
```

### 🧠 Explanation

- **`CROSS JOIN` builds the scaffold.** Every student is paired with every subject, unconditionally — no `ON` clause, because there's no relationship to match on. That's what guarantees Alex gets three rows even though he never took an exam.
- **The `LEFT JOIN` needs both conditions in `ON`.** `AND sub.subject_name = e.subject_name` is what makes each exam row attach to the *right* pair. Alice sat Math three times, so three exam rows attach to the (1, Math) pair → count 3.
- **`COUNT(e.subject_name)` vs `COUNT(*)` is the whole problem.** For (2, Physics) the `LEFT JOIN` yields one row with `e.subject_name = NULL`. `COUNT(*)` counts that row → wrong answer `1`. `COUNT(e.subject_name)` skips the `NULL` → correct answer `0`. **`COUNT(*)` counts rows; `COUNT(col)` counts non-`NULL` values.** Internalise that and half of the join-plus-aggregate problems become mechanical.
- **`GROUP BY s.student_id, sub.subject_name` while selecting `s.student_name`** is legal under `ONLY_FULL_GROUP_BY` because `student_name` is functionally dependent on `student_id` (its primary key) — MySQL 8 detects this. Grouping by `student_name` instead would be both wrong (two students could share a name) and fragile.
- 💡 Notice the output is sorted `1, 2, 6, 13` — numeric order on `student_id`, not the table's insertion order. `ORDER BY` is explicitly required by the problem, so never rely on the order rows happen to come back in.

---

## Q3 — Managers with at Least 5 Direct Reports

🔗 **[leetcode.com/problems/managers-with-at-least-5-direct-reports](https://leetcode.com/problems/managers-with-at-least-5-direct-reports/description/)**

**Problem:** Find the names of managers who have **at least five** direct reports.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Employee;
CREATE TABLE Employee (
    id         INT PRIMARY KEY,
    name       VARCHAR(20),
    department VARCHAR(5),
    managerId  INT
);

INSERT INTO Employee VALUES
(101,'John','A',NULL),(102,'Dan','A',101),(103,'James','A',101),
(104,'Amy','A',101),(105,'Anne','A',101),(106,'Ron','B',101);
```

### ✅ Expected Output

| name |
|------|
| John |

### 🎯 How to approach it

Managers and employees live in the **same table**, linked by `managerId → id`. Comparing two rows of one table means a **self-join** with two aliases: `e` for the report, `m` for the manager.

**Then the counting question: what do you `GROUP BY`?**

The instinct is `GROUP BY m.name` — you're returning names, after all. **That's a bug.** Two different managers can share a name, and grouping by name silently merges their report counts. I tested it: add two distinct managers both called *Sam*, each with 3 reports, and `GROUP BY m.name` merges them into 6 and wrongly reports Sam as having 5+ reports. `GROUP BY m.id` — the primary key — keeps them separate and correctly excludes both.

**Rule: group by the identifier, select the label.** Never group by a display name when an ID exists.

Finally, "at least five" is a condition on a **count**, which doesn't exist until after grouping → `HAVING`, not `WHERE`.

### 💡 Solution

```sql
SELECT m.name AS name
FROM Employee e
JOIN Employee m
    ON e.managerId = m.id
GROUP BY m.id
HAVING COUNT(*) >= 5;
```

### 🧠 Explanation

- **Self-join with `ON e.managerId = m.id`** puts each report next to their manager. John has five rows pointing at him (Dan, James, Amy, Anne, Ron); everyone else has none.
- **`INNER JOIN` is correct here.** Employees with `managerId = NULL` (John himself) produce no match and are excluded — which is right, since the question asks about *being* a manager, not *having* one. Rows only survive if someone actually reports to them.
- **`COUNT(*)` is safe** because this is an inner join — every surviving row is a genuine report. (Contrast with Q2, where the `LEFT JOIN` made `COUNT(*)` wrong.)
- **`HAVING COUNT(*) >= 5`, not `WHERE`.** `WHERE` runs before grouping, when no count exists yet. And "at least five" is inclusive → `>=`, not `>`.
- **`GROUP BY m.id` while selecting `m.name`** passes `ONLY_FULL_GROUP_BY` via functional dependency on the primary key — and, as above, is the *correct* grouping regardless.
- 💡 The problem says **"direct** reports," which is why a plain join suffices. If it asked for *all* reports down the chain (reports of reports), you'd need a **recursive CTE** to walk the hierarchy — see [Part 12](../docs/12-recursive-cte.md).

---

## Q4 — Confirmation Rate

🔗 **[leetcode.com/problems/confirmation-rate](https://leetcode.com/problems/confirmation-rate/description/)**

**Problem:** The confirmation rate of a user is the number of `'confirmed'` messages divided by the total number of requested confirmation messages. A user who requested **no** confirmations has a rate of **0**. Report the rate for every signed-up user, rounded to 2 decimals.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Signups;
CREATE TABLE Signups (
    user_id    INT PRIMARY KEY,
    time_stamp DATETIME
);

INSERT INTO Signups VALUES
(3,'2020-03-21 10:16:13'),(7,'2020-01-04 13:57:59'),
(2,'2020-07-29 23:09:44'),(6,'2020-12-09 10:39:37');

DROP TABLE IF EXISTS Confirmations;
CREATE TABLE Confirmations (
    user_id    INT,
    time_stamp DATETIME,
    action     ENUM('confirmed','timeout'),
    PRIMARY KEY (user_id, time_stamp)
);

INSERT INTO Confirmations VALUES
(3,'2021-01-06 03:30:46','timeout'),(3,'2021-07-14 14:00:00','timeout'),
(7,'2021-06-12 11:57:29','confirmed'),(7,'2021-06-13 12:58:28','confirmed'),
(7,'2021-06-14 13:59:27','confirmed'),(2,'2021-01-22 00:00:00','confirmed'),
(2,'2021-02-28 23:59:59','timeout');
```

### ✅ Expected Output

| user_id | confirmation_rate |
|---------|-------------------|
| 2 | 0.50 |
| 3 | 0.00 |
| 6 | 0.00 |
| 7 | 1.00 |

### 🎯 How to approach it

Three layered requirements — solve them in this order:

1. **"every signed-up user"** → `LEFT JOIN` from `Signups`. User 6 never requested a confirmation and must still appear.
2. **"confirmed ÷ total"** → a *conditional* count over a *total* count, in one pass. In MySQL a boolean expression evaluates to `1`/`0`, so **`SUM(action = 'confirmed')`** is the count of confirmed rows. Divide by the total row count for that user.
3. **"no confirmations → 0"** → this is where it gets subtle. For user 6, `SUM(action = 'confirmed')` is `SUM(NULL)` = **`NULL`**, not 0, because there are no rows to sum. `NULL / anything` is `NULL`. So the raw expression yields `NULL` and you must convert it → wrap the whole thing in `IFNULL(..., 0)`.

**Where to put `ROUND` and `IFNULL`:** round the division first, then null-coalesce the result — `IFNULL(ROUND(x, 2), 0)`. Reversing them also works here, but rounding innermost keeps the precision rule attached to the arithmetic it applies to.

### 💡 Solution

```sql
SELECT s.user_id AS user_id,
       IFNULL(
           ROUND(SUM(action = 'confirmed') / COUNT(s.user_id), 2),
       0) AS confirmation_rate
FROM Signups s
LEFT JOIN Confirmations c
    ON s.user_id = c.user_id
GROUP BY s.user_id;
```

### 🧠 Explanation

- **`SUM(action = 'confirmed')` is conditional aggregation.** The comparison returns `1` for a match and `0` otherwise, so summing it counts matches. User 7 → 3, user 2 → 1, user 3 → 0.
- **The denominator `COUNT(s.user_id)`** counts the joined rows per user. `s.user_id` can never be `NULL` (it's the left table's primary key), so this equals the number of confirmation requests — 2 for user 3, 3 for user 7. User 2: 1 ÷ 2 = **0.50**. ✅
- **`IFNULL(..., 0)` handles user 6**, whose `SUM` is `NULL` because the `LEFT JOIN` produced a single all-`NULL` row. Without it, user 6 returns `NULL` and the submission fails.
- **`COUNT(c.action)` and `AVG(action = 'confirmed')` are equally valid.** I verified all three forms return identical results on this data. `AVG(action = 'confirmed')` is the tersest — an average of 1s and 0s *is* the rate — and it's worth naming in an interview as the idiomatic version.
- 💡 **The reason this problem is rated Medium** is entirely the `NULL` in step 3. `SUM` over zero rows returns `NULL`, not `0` — unlike `COUNT`, which returns `0`. That asymmetry between `SUM` and `COUNT` on an empty group is a favourite interview probe.

---

## Q5 — Not Boring Movies

🔗 **[leetcode.com/problems/not-boring-movies](https://leetcode.com/problems/not-boring-movies/description/)**

**Problem:** Report the movies with an **odd** `id` whose description is **not** "boring", ordered by rating descending.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Cinema;
CREATE TABLE Cinema (
    id          INT PRIMARY KEY,
    movie       VARCHAR(30),
    description VARCHAR(50),
    rating      FLOAT(2,1)
);

INSERT INTO Cinema VALUES
(1,'War','great 3D',8.9),(2,'Science','fiction',8.5),(3,'irish','boring',6.2),
(4,'Ice song','Fantacy',8.6),(5,'House card','Interesting',9.1);
```

### ✅ Expected Output

| id | movie | description | rating |
|----|-------|-------------|--------|
| 5 | House card | Interesting | 9.1 |
| 1 | War | great 3D | 8.9 |

### 🎯 How to approach it

Straightforward, with one new tool and one portability note.

**Odd numbers → the modulo operator.** `id % 2` gives the remainder after dividing by 2: `1` for odd, `0` for even. So `WHERE id % 2 = 1`. (MySQL also spells this `MOD(id, 2) = 1`.)

**"Not boring" → `<>` (or `!=`).** Note that `description` is `NOT NULL` in this problem, so no `IS NULL` guard is needed — unlike [Set 1 Q2](set-01-easy-basics.md#q2--find-customer-referee). Always check nullability before deciding whether you can get away with a bare `<>`.

**`SELECT *` is acceptable here** because the problem asks for all columns. That's rare — usually you should name them.

### 💡 Solution

```sql
SELECT *
FROM Cinema
WHERE id % 2 = 1
  AND description <> 'boring'
ORDER BY rating DESC;
```

### 🧠 Explanation

- **`id % 2 = 1`** keeps ids 1, 3, 5. **`description <> 'boring'`** then drops id 3 (*irish*). Two survivors: 1 and 5.
- **`ORDER BY rating DESC`** puts 9.1 above 8.9. Descending must be explicit — `ASC` is the default.
- ⚠️ **Use single quotes for string literals, not double.** `description <> "boring"` works on LeetCode and on a default MySQL install, but MySQL has a `sql_mode` flag called **`ANSI_QUOTES`** that makes `"..."` mean *identifier* instead of *string*. I verified it: with `ANSI_QUOTES` enabled, `<> "boring"` fails with `Unknown column 'boring' in 'where clause'`. Single quotes are unambiguous in every mode and portable to every other database. Make `'boring'` the habit.
- 💡 For **even** ids you'd write `id % 2 = 0`. And on a large table, note that `id % 2 = 1` is **not sargable** — wrapping a column in an expression prevents MySQL from using an index on it, so this forces a full scan. That's a solid answer to "how would this behave at scale?"

---

## Q6 — Average Selling Price

🔗 **[leetcode.com/problems/average-selling-price](https://leetcode.com/problems/average-selling-price/description/)**

**Problem:** The average selling price of a product is `total revenue ÷ total units sold`, where each sale is priced by whichever price period its `purchase_date` falls into. Report it per product, rounded to 2 decimals. A product with **no sales** has an average price of **0**.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Prices;
CREATE TABLE Prices (
    product_id INT,
    start_date DATE,
    end_date   DATE,
    price      INT,
    PRIMARY KEY (product_id, start_date, end_date)
);

INSERT INTO Prices VALUES
(1,'2019-02-17','2019-02-28',5),(1,'2019-03-01','2019-03-22',20),
(2,'2019-02-01','2019-02-20',15),(2,'2019-02-21','2019-03-31',30);

DROP TABLE IF EXISTS UnitsSold;
CREATE TABLE UnitsSold (
    product_id    INT,
    purchase_date DATE,
    units         INT
);

INSERT INTO UnitsSold VALUES
(1,'2019-02-25',100),(1,'2019-03-01',15),(2,'2019-02-10',200),(2,'2019-03-22',30);
```

### ✅ Expected Output

| product_id | average_price |
|------------|---------------|
| 1 | 6.96 |
| 2 | 16.96 |

### 🎯 How to approach it

**The join key isn't just an ID — it's an ID plus a date range.** A product has several price periods, and each sale belongs to exactly one of them. So the `ON` clause carries two conditions:

```
ON p.product_id = u.product_id
AND u.purchase_date BETWEEN p.start_date AND p.end_date
```

That second line is a **range join** — match when a value falls *inside* an interval rather than equalling something. It's the pattern for price history, subscription tiers, effective-dated rates, anything versioned by time. `BETWEEN` is inclusive on both ends, which is what you want when periods are defined as closed intervals.

**Then: this is a weighted average, not `AVG()`.** You cannot write `AVG(p.price)` — that would treat a 100-unit sale and a 15-unit sale as equally important. The definition is total revenue over total units:

```
SUM(price × units) / SUM(units)
```

**Finally, "no sales → 0".** `LEFT JOIN` keeps such a product, but then both `SUM`s are `NULL` (summing zero rows), so the division is `NULL`. Wrap it in `COALESCE(..., 0)`.

### 💡 Solution

```sql
SELECT p.product_id,
       COALESCE(ROUND(SUM(p.price * u.units) / SUM(u.units), 2), 0) AS average_price
FROM Prices p
LEFT JOIN UnitsSold u
    ON  p.product_id = u.product_id
    AND u.purchase_date BETWEEN p.start_date AND p.end_date
GROUP BY p.product_id;
```

### 🧠 Explanation

- **The range join assigns each sale its correct price.** Product 1's sale on `2019-02-25` falls in the first period (price 5), and its `2019-03-01` sale falls in the second (price 20) — note `2019-03-01` is the period's `start_date`, and `BETWEEN` includes it.
- **Working product 1 by hand:** revenue = (5 × 100) + (20 × 15) = 800; units = 100 + 15 = 115; 800 ÷ 115 = 6.956… → **6.96**. ✅ A plain `AVG(price)` would have given 12.5 — badly wrong, and the reason weighted averages get asked about.
- **`COALESCE` is load-bearing, and LeetCode tests it.** I added a product 3 that exists in `Prices` with no matching sale: it correctly returns **0.00**. Remove the `COALESCE` and it returns `NULL` → Wrong Answer.
- **`GROUP BY p.product_id`** — grouped on the left table's column so products with no sales still form a group. Grouping on `u.product_id` would produce a `NULL` group instead.
- 💡 I qualified `SUM(u.units)` — your original had a bare `SUM(units)`, which works only because `Prices` has no `units` column. Qualify aggregate arguments in a join; the day someone adds a same-named column, the unqualified version becomes an ambiguity error (or worse, silently reads the wrong column).

---

## Q7 — Project Employees I

🔗 **[leetcode.com/problems/project-employees-i](https://leetcode.com/problems/project-employees-i/description/)**

**Problem:** Report the average experience years of all employees on each project, rounded to 2 decimals.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Project;
CREATE TABLE Project (
    project_id  INT,
    employee_id INT,
    PRIMARY KEY (project_id, employee_id)
);

INSERT INTO Project VALUES (1,1),(1,2),(1,3),(2,1),(2,4);

DROP TABLE IF EXISTS Employee;
CREATE TABLE Employee (
    employee_id      INT PRIMARY KEY,
    name             VARCHAR(20),
    experience_years INT
);

INSERT INTO Employee VALUES (1,'Khaled',3),(2,'Ali',2),(3,'John',1),(4,'Doe',2);
```

### ✅ Expected Output

| project_id | average_years |
|------------|---------------|
| 1 | 2.00 |
| 2 | 2.50 |

### 🎯 How to approach it

After Q6 this one is a relief — and that contrast *is* the lesson.

`Project` is a **junction table** (many-to-many between projects and employees). It holds the `employee_id` but not the experience, so join to `Employee` to fetch it, then aggregate per project.

**Why `INNER JOIN` is fine here:** `Project.employee_id` is a foreign key, so every assignment references a real employee. There are no unmatched rows to preserve, and nothing about the question asks for projects with zero employees.

**Why plain `AVG()` is correct here but was wrong in Q6:** the question asks for the average *per employee*, and each employee contributes exactly one row per project. There's no quantity to weight by. Read the definition, not the word "average."

The only trap left is precision: `ROUND(..., 2)` is required, so `2` must render as `2.00`.

### 💡 Solution

```sql
SELECT p.project_id,
       ROUND(AVG(e.experience_years), 2) AS average_years
FROM Project p
JOIN Employee e
    ON p.employee_id = e.employee_id
GROUP BY p.project_id;
```

### 🧠 Explanation

- **Project 1** has employees 1, 2, 3 → (3 + 2 + 1) ÷ 3 = **2.00**. **Project 2** has employees 1, 4 → (3 + 2) ÷ 2 = **2.50**.
- **`AVG` ignores `NULL`s** in its input. `experience_years` is always populated here, but be aware: if some employees had `NULL` experience, `AVG` would divide by the count of *non-null* values, not the total employee count. Whether that's right depends on the question — sometimes you want `AVG(COALESCE(experience_years, 0))` instead.
- **`ROUND(..., 2)`** matters even when the value looks exact — LeetCode compares the string `2.00`, so returning `2` fails.
- 💡 **Project Employees II** (the follow-up problem) asks for the projects with the *most* employees, which needs `RANK()` or a `HAVING COUNT(*) = (SELECT MAX(...))` subquery. Same tables, meaningfully harder — worth doing right after this one while the schema is fresh.

---

## Q8 — Percentage of Users Attended a Contest

🔗 **[leetcode.com/problems/percentage-of-users-attended-a-contest](https://leetcode.com/problems/percentage-of-users-attended-a-contest/description/)**

**Problem:** For each contest, report the percentage of all users who registered for it, rounded to 2 decimals. Order by percentage descending, then `contest_id` ascending.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Users;
CREATE TABLE Users (
    user_id   INT PRIMARY KEY,
    user_name VARCHAR(20)
);

INSERT INTO Users VALUES (6,'Alice'),(2,'Bob'),(7,'Alex');

DROP TABLE IF EXISTS Register;
CREATE TABLE Register (
    contest_id INT,
    user_id    INT,
    PRIMARY KEY (contest_id, user_id)
);

INSERT INTO Register VALUES (215,6),(209,2),(208,2),(210,6),(208,6),(209,7),
(209,6),(215,7),(208,7),(210,2),(207,2),(210,7);
```

### ✅ Expected Output

| contest_id | percentage |
|------------|------------|
| 208 | 100.00 |
| 209 | 100.00 |
| 210 | 100.00 |
| 215 | 66.67 |
| 207 | 33.33 |

### 🎯 How to approach it

**The two halves of the fraction have different grains, and that's the whole design decision.**

- The **numerator** is per-contest → an aggregate inside `GROUP BY contest_id`.
- The **denominator** is the total user count — one fixed number for the entire query, unrelated to any contest.

A `JOIN` is the wrong tool for a denominator like that; it would multiply rows. What you want is a **scalar subquery** — a subquery returning exactly one value, usable anywhere a single value is allowed:

```sql
(SELECT COUNT(*) FROM Users)
```

MySQL evaluates it **once**, not per row, so there's no performance concern.

**Two details that decide correctness:**

- **`Users`, not `Register`, is the denominator.** The question is "what share of *all users*," including users who registered for nothing. Counting `DISTINCT user_id` from `Register` would give the wrong base.
- **Multiply by 100 *before* dividing**, or at least before rounding. `ROUND(count/total, 2) * 100` rounds to 2 decimals *first* and destroys precision — `0.6667` becomes `0.67` → `67.00` instead of `66.67`. Order of operations is the silent killer here.

### 💡 Solution

```sql
SELECT contest_id,
       ROUND((COUNT(*) * 100) / (SELECT COUNT(*) FROM Users), 2) AS percentage
FROM Register
GROUP BY contest_id
ORDER BY percentage DESC, contest_id ASC;
```

### 🧠 Explanation

- **`COUNT(*)` per contest is safe** because `Register`'s primary key is `(contest_id, user_id)` — the same user can't register twice for one contest, so rows *are* distinct users. Without that guarantee you'd need `COUNT(DISTINCT user_id)`.
- **The scalar subquery returns 3** (three users). Contest 208 has 3 registrations → 3 × 100 ÷ 3 = **100.00**; contest 215 has 2 → 200 ÷ 3 = 66.666… → **66.67**; contest 207 has 1 → **33.33**.
- **`ORDER BY percentage DESC, contest_id ASC`** — the two-key sort is required by the problem, and the tie among 208/209/210 (all 100.00) is exactly why. Without the `contest_id` tiebreaker their order is undefined and the submission may fail non-deterministically. **Whenever a problem specifies a tiebreaker, it's because the data has ties.**
- 💡 Sorting by the **alias** `percentage` works because `ORDER BY` is evaluated after `SELECT`. You couldn't use that alias in `WHERE` or `GROUP BY`, which run earlier — a clause-evaluation-order question interviewers love.

---

## Q9 — Queries Quality and Percentage

🔗 **[leetcode.com/problems/queries-quality-and-percentage](https://leetcode.com/problems/queries-quality-and-percentage/description/)**

**Problem:** For each `query_name`, report **quality** (the average of `rating / position`) and **poor_query_percentage** (the percentage of queries with `rating < 3`), both rounded to 2 decimals.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Queries;
CREATE TABLE Queries (
    query_name VARCHAR(30),
    result     VARCHAR(50),
    position   INT,
    rating     INT
);

INSERT INTO Queries VALUES
('Dog','Golden Retriever',1,5),('Dog','German Shepherd',2,5),('Dog','Mule',200,1),
('Cat','Shirazi',5,2),('Cat','Siamese',3,3),('Cat','Sphynx',7,4);
```

### ✅ Expected Output

| query_name | quality | poor_query_percentage |
|------------|---------|-----------------------|
| Dog | 2.50 | 33.33 |
| Cat | 0.66 | 33.33 |

### 🎯 How to approach it

Both metrics come from the same group in a single pass — no subquery needed. Translate each definition literally:

**Quality — "the average of `rating / position`":** `AVG(rating / position)`. Read the wording carefully: it's the average **of the ratios**, not the ratio of the averages. `AVG(rating) / AVG(position)` is a different number and a wrong answer. This distinction (average-of-ratios vs ratio-of-averages) shows up constantly in analytics work.

**Poor query percentage — "percentage where `rating < 3`":** `SUM(rating < 3)` counts the matching rows via boolean-to-integer, divide by `COUNT(*)`, multiply by 100. Same conditional-aggregation idiom as Q4.

### ⚠️ The hidden test case that fails this solution

**LeetCode's test data includes a row where `query_name` is `NULL`.** `GROUP BY query_name` dutifully creates a `NULL` group, producing a spurious output row:

```
| query_name | quality | poor_query_percentage |
| Dog        |    2.50 |                 33.33 |
| Cat        |    0.66 |                 33.33 |
| NULL       |    NULL |                  NULL |   ← extra row, expected output has no such row
```

I reproduced this by inserting `(NULL, NULL, NULL, NULL)`. The expected output has no `NULL` row, so the submission fails. The fix is one line — **`WHERE query_name IS NOT NULL`**:

### 💡 Solution

```sql
SELECT query_name,
       ROUND(AVG(rating / position), 2)                  AS quality,
       ROUND((SUM(rating < 3) / COUNT(*)) * 100, 2)      AS poor_query_percentage
FROM Queries
WHERE query_name IS NOT NULL          -- ← required: a NULL query_name would emit a bogus group
GROUP BY query_name;
```

### 🧠 Explanation

- **`AVG(rating / position)`** for Dog: (5/1 + 5/2 + 1/200) ÷ 3 = (5 + 2.5 + 0.005) ÷ 3 = 2.5016… → **2.50**. For Cat: (2/5 + 3/3 + 4/7) ÷ 3 = (0.4 + 1 + 0.5714) ÷ 3 = 0.657… → **0.66**.
- **`SUM(rating < 3)`** counts poor results — one per group here (Dog's *Mule* at rating 1, Cat's *Shirazi* at rating 2). 1 ÷ 3 × 100 = 33.333… → **33.33**.
- **`WHERE query_name IS NOT NULL` is not defensive padding — it's required.** `GROUP BY` treats all `NULL`s as one group (unlike `=`, which never matches `NULL`), so a single null-named row produces a whole extra output row. This is one of the most common "correct logic, Wrong Answer" traps in the SQL 50.
- **`ROUND` placement:** for the percentage, the `* 100` sits *inside* `ROUND`, so you round the final percentage to 2 decimals rather than rounding a fraction and scaling the error up 100×. Same trap as Q8.
- 💡 **Integer division check:** `rating / position` with two `INT`s returns a decimal in MySQL, not a truncated integer — so `5/2` is `2.5`, as needed. Don't assume; in some databases (and with the `DIV` operator in MySQL) you'd get `2` and a silently wrong answer.

---

## Q10 — Monthly Transactions I

🔗 **[leetcode.com/problems/monthly-transactions-i](https://leetcode.com/problems/monthly-transactions-i/description/)**

**Problem:** For each **month and country**, report the total transaction count, the approved count, the total amount, and the total approved amount.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Transactions;
CREATE TABLE Transactions (
    id         INT PRIMARY KEY,
    country    VARCHAR(4),
    state      ENUM('approved','declined'),
    amount     INT,
    trans_date DATE
);

INSERT INTO Transactions VALUES
(121,'US','approved',1000,'2018-12-18'),
(122,'US','declined',2000,'2018-12-19'),
(123,'US','approved',2000,'2019-01-01'),
(124,'DE','approved',2000,'2019-01-07');
```

### ✅ Expected Output

| month | country | trans_count | approved_count | trans_total_amount | approved_total_amount |
|-------|---------|-------------|----------------|--------------------|-----------------------|
| 2018-12 | US | 2 | 1 | 3000 | 1000 |
| 2019-01 | US | 1 | 1 | 2000 | 2000 |
| 2019-01 | DE | 1 | 1 | 2000 | 2000 |

### 🎯 How to approach it

This is the **conditional-aggregation summary report** — the single most reusable pattern in analytical SQL. Four metrics, one table scan, no subqueries.

**Step 1 — the month doesn't exist as a column.** You have a `DATE`, but you need to group by `YYYY-MM`. `DATE_FORMAT(trans_date, '%Y-%m')` derives it, and you group by **that same expression**. Grouping by the raw `trans_date` would split each day into its own group — wrong grain.

**Step 2 — pair each unconditional metric with its conditional twin:**

| Metric | Expression | Why |
|---|---|---|
| All transactions | `COUNT(*)` | every row in the group |
| Approved only | `SUM(state = 'approved')` | boolean → `1`/`0`, so summing counts matches |
| All amount | `SUM(amount)` | straight total |
| Approved amount | `SUM(CASE WHEN state = 'approved' THEN amount ELSE 0 END)` | sum the amount *only* on approved rows |

The last one is the important shape. `SUM(state = 'approved')` counts **rows**; to sum a *different column* conditionally you need the `CASE` form — the boolean shortcut only produces 1s and 0s, so it can't carry an amount.

**Step 3 — no filtering.** Every transaction counts toward `trans_count`, so there's no `WHERE`. All the selectivity lives inside the aggregates. That's the point of conditional aggregation: you'd otherwise need two queries and a join to combine "all" with "approved only."

### 💡 Solution

```sql
SELECT DATE_FORMAT(trans_date, '%Y-%m')  AS month,
       country,
       COUNT(*)                          AS trans_count,
       SUM(state = 'approved')           AS approved_count,
       SUM(amount)                       AS trans_total_amount,
       SUM(CASE WHEN state = 'approved' THEN amount ELSE 0 END) AS approved_total_amount
FROM Transactions
GROUP BY country, DATE_FORMAT(trans_date, '%Y-%m');
```

### 🧠 Explanation

- **The `2018-12 / US` group** contains transactions 121 (approved, 1000) and 122 (declined, 2000): `trans_count` 2, `approved_count` 1, `trans_total_amount` 3000, `approved_total_amount` **1000** — only the approved amount, which is exactly what the `CASE` isolates.
- **`SUM(state = 'approved')` vs `SUM(CASE WHEN ... THEN amount ...)`** is the key contrast. The first counts qualifying rows; the second totals a column across qualifying rows. Mixing them up is the standard mistake — `SUM(state = 'approved') * amount` is meaningless.
- **`GROUP BY` repeats the `DATE_FORMAT` expression.** MySQL also lets you `GROUP BY month` (the alias) as a non-standard convenience, but repeating the expression is portable and unambiguous. Note that `country` is grouped too — the grain is (month, country), which is why `2019-01` appears twice.
- **`ELSE 0` rather than omitting it.** Without `ELSE`, non-matching rows yield `NULL`, and while `SUM` ignores `NULL`s so the total is the same, a group with *zero* approved rows would return `NULL` instead of `0`. `ELSE 0` guarantees a number.
- 💡 **This is a pivot.** Swap `country` for a set of fixed columns and you're building the classic rows-to-columns report (see [Problem Set 5 Q3](../interview-questions/problem-set-05.md#q3--monthly-sales-pivot)). Learn this shape once and it covers monthly dashboards, funnel conversion, cohort tables, and approval-rate reporting.

---

## 🧠 Set 2 — Patterns Worth Memorising

| Pattern | Trigger phrase | Tool |
|---|---|---|
| Keep unmatched rows **and** filter the right table | "include those with none, where X < n" | `LEFT JOIN` + `WHERE col IS NULL OR ...` (or move the filter into `ON`) |
| Every combination must appear | "for each student **and each** subject" | `CROSS JOIN` to scaffold, then `LEFT JOIN` |
| Count matches after an outer join | "0 when they never did it" | `COUNT(col)` — **never** `COUNT(*)` |
| Count a group, filter on the count | "at least N", "more than N" | `GROUP BY id` + `HAVING COUNT(*) >= n` |
| Conditional count | "how many were approved" | `SUM(condition)` |
| Conditional total of another column | "total approved **amount**" | `SUM(CASE WHEN cond THEN col ELSE 0 END)` |
| Rate / percentage per group | "the percentage of…" | `SUM(cond)/COUNT(*)`, wrapped in `IFNULL`/`COALESCE` |
| A whole-table denominator | "percentage of **all** users" | scalar subquery `(SELECT COUNT(*) FROM t)` |
| Match a value into a period | price history, effective dates | `ON id = id AND date BETWEEN start AND end` |
| Weighted average | "total revenue ÷ total units" | `SUM(a*b)/SUM(b)` — **not** `AVG(a)` |
| Group by month | "for each month" | `GROUP BY DATE_FORMAT(d, '%Y-%m')` |
| Odd / even | "odd id" | `id % 2 = 1` |

### Five mistakes that cost the most submissions

1. **`WHERE` on the right table of a `LEFT JOIN`.** It silently converts the outer join to an inner one. Add `OR col IS NULL`, or move the condition into `ON`. (Q1)
2. **`COUNT(*)` after a `LEFT JOIN`.** Unmatched rows still exist, so you get `1` where the answer is `0`. Use `COUNT(column)`. (Q2)
3. **`SUM` over an empty group returns `NULL`, not `0`.** Every rate/average that must show `0` needs `IFNULL`/`COALESCE`. (Q4, Q6)
4. **`NULL` group keys.** `GROUP BY` collapses all `NULL`s into one real group and emits an extra row. Add `WHERE key IS NOT NULL` when the column is nullable. (Q9)
5. **Rounding before scaling.** `ROUND(x, 2) * 100` throws away the precision you needed. Put the `* 100` inside the `ROUND`. (Q8, Q9)

---

<div align="center">

**Set 3 coming soon.** ⭐ the repo to follow along.

[⬅ Back to Course Home](../README.md) · [Set 1 — Select & Basic Joins](set-01-easy-basics.md) · [Interview Problem Sets](../interview-questions/) · [LeetCode SQL 50 study plan ↗](https://leetcode.com/studyplan/top-sql-50/)

</div>
