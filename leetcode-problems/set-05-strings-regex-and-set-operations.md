# 🟥 LeetCode SQL 50 — Set 5: Strings, Regex & Set Operations

> **Problems 41–50 of the [LeetCode SQL 50](https://leetcode.com/studyplan/top-sql-50/) study plan — the final set.**
> Every problem follows the same flow: **Problem → Schema → Sample Input → Expected Output → Approach → Solution → Explanation.**

The last ten problems are the ones that look easiest and fail most often. **Half of them are string work** — capitalisation, prefix matching, regex, list aggregation — and strings are where SQL's defaults quietly disagree with what you meant. `LIKE '%DIAB1%'` matches `SDIAB100`. `REGEXP '@leetcode\.com'` matches `@LEETCODE.COM`. `GROUP_CONCAT` truncates at 1024 characters and tells nobody.

The other half is **set operations and mutation**: flipping a bidirectional relationship with `UNION ALL`, ranking with the *right* rank function, and the two ways to `DELETE` from a table you're simultaneously reading — a statement MySQL refuses outright if you write it the obvious way.

All queries target **MySQL 8.0+** and were executed against LeetCode's own sample data on a local MySQL 9.6 server — the Expected Output blocks below are real query output, not hand-written. Where two solutions look equivalent but diverge on data LeetCode's samples don't contain, the divergence is shown with real output too.

---

## 📋 Contents

| # | Problem | Difficulty | Core Concept |
|---|---------|:---:|--------------|
| 1 | [Friend Requests II: Who Has the Most Friends](#q1--friend-requests-ii-who-has-the-most-friends) | 🟨 Medium | `UNION ALL` to flatten a two-column relationship |
| 2 | [Investments in 2016](#q2--investments-in-2016) | 🟨 Medium | Two `COUNT() OVER` at different grains, one pass |
| 3 | [Department Top Three Salaries](#q3--department-top-three-salaries) | 🟥 Hard | `DENSE_RANK()` — and why only it works |
| 4 | [Fix Names in a Table](#q4--fix-names-in-a-table) | 🟩 Easy | `CONCAT` + `UPPER`/`LOWER` + `SUBSTRING` |
| 5 | [Patients With a Condition](#q5--patients-with-a-condition) | 🟩 Easy | Word-boundary matching with `LIKE` |
| 6 | [Delete Duplicate Emails](#q6--delete-duplicate-emails) | 🟩 Easy | `DELETE` from a table you're reading |
| 7 | [Second Highest Salary](#q7--second-highest-salary) | 🟨 Medium | Returning `NULL` when there's no answer |
| 8 | [Group Sold Products By The Date](#q8--group-sold-products-by-the-date) | 🟩 Easy | `GROUP_CONCAT` with `DISTINCT` + `ORDER BY` |
| 9 | [List the Products Ordered in a Period](#q9--list-the-products-ordered-in-a-period) | 🟩 Easy | Half-open date range + `HAVING` on an alias |
| 10 | [Find Users With Valid E-Mails](#q10--find-users-with-valid-e-mails) | 🟩 Easy | `REGEXP` and the case-sensitivity trap |

---

## Q1 — Friend Requests II: Who Has the Most Friends

🔗 **[leetcode.com/problems/friend-requests-ii-who-has-the-most-friends](https://leetcode.com/problems/friend-requests-ii-who-has-the-most-friends/)**

**Problem:** Find the person with the **most friends**, and how many they have. Friendship is mutual: an accepted request makes both people friends.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS RequestAccepted;
CREATE TABLE RequestAccepted (
    requester_id INT,
    accepter_id  INT,
    accept_date  DATE,
    PRIMARY KEY (requester_id, accepter_id)
);

INSERT INTO RequestAccepted VALUES
(1,2,'2016-06-03'),(1,3,'2016-06-08'),(2,3,'2016-06-08'),(3,4,'2016-06-09');
```

### ✅ Expected Output

| id | num |
|----|-----|
| 3 | 3 |

### 🎯 How to approach it

**The relationship is symmetric but the storage is not.** Person 3 appears as an *accepter* twice (rows 2 and 3) and as a *requester* once (row 4) — three friends total. Counting either column alone gets you a third of the answer.

The fix is to **stack the two columns into one**, so every friendship contributes one row per participant:

```
requester_id: 1, 1, 2, 3
accepter_id:  2, 3, 3, 4
             ↓ stacked ↓
         1, 1, 2, 3, 2, 3, 3, 4
```

Now person 3 appears three times, person 1 twice, persons 2 and 4 twice and once. `GROUP BY id`, count, take the top. **Eight rows from four friendships** — each friendship correctly counted from both ends.

### 💡 Solution

```sql
WITH all_freinds AS (
    SELECT requester_id AS id
    FROM RequestAccepted

    UNION ALL

    SELECT accepter_id AS id
    FROM RequestAccepted
)
SELECT id,
       COUNT(*) AS num
FROM all_freinds
GROUP BY id
ORDER BY COUNT(*) DESC
LIMIT 1;
```

### 🧠 Explanation

- ⚠️ **`UNION ALL` is mandatory, and `UNION` gives a wrong answer rather than a slow one.** `UNION` deduplicates, collapsing the eight-row stack into the four distinct ids `{1,2,3,4}` — each appearing exactly once. Every count becomes 1 and the query returns an arbitrary person:

  | Operator | Result |
  |---|---|
  | `UNION ALL` | **id 3, num 3** ✅ |
  | `UNION` | id 1, num 1 ❌ |

  That's verified output. **Deduplication destroys the counts you are trying to compute** — the same class of mistake as [Set 4 Q9](set-04-advanced-windows-and-conditional-logic.md#q9--movie-rating), where `UNION` would have collapsed two legitimate answer rows into one.
- **`ORDER BY COUNT(*) DESC LIMIT 1`** doesn't require the count in the `SELECT` list, though here it's wanted anyway. `ORDER BY` runs after `GROUP BY`, so aggregates are always available to it.
- **Ties are undefined.** With two people on equal counts, `LIMIT 1` returns whichever the plan reaches first — verified on a table where all four people tie at 1. LeetCode guarantees no tie in the test data. **If it didn't, you'd need `RANK() OVER (ORDER BY COUNT(*) DESC)` and keep rank 1**, which returns everyone tied at the top. Worth saying out loud in an interview; "the problem guarantees uniqueness" is a better answer than not noticing.
- 💡 **This unpivot-by-`UNION ALL` pattern is how you handle any symmetric relationship** stored as two columns: mutual follows, co-authorship, trades between accounts, matches between teams. The moment you see "A and B are both X", stack the columns first.

---

## Q2 — Investments in 2016

🔗 **[leetcode.com/problems/investments-in-2016](https://leetcode.com/problems/investments-in-2016/description)**

**Problem:** Sum `tiv_2016` for all policyholders who satisfy **both**: they have the **same `tiv_2015` value as at least one other** policyholder, and their **location `(lat, lon)` is unique** in the table. Round to 2 decimals.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Insurance;
CREATE TABLE Insurance (
    pid      INT PRIMARY KEY,
    tiv_2015 FLOAT,
    tiv_2016 FLOAT,
    lat      FLOAT,
    lon      FLOAT
);

INSERT INTO Insurance VALUES (1,10,5,10,10),(2,20,20,20,20),(3,10,30,20,20),(4,10,40,40,40);
```

### ✅ Expected Output

| tiv_2016 |
|----------|
| 45 |

### 🎯 How to approach it

**Two conditions, two different groupings, one table.** That's the shape to recognise — and window functions handle it in a single pass where correlated subqueries would need two.

Tag every row with both counts, then filter:

| pid | tiv_2015 | count by tiv_2015 | (lat, lon) | count by location | keep? |
|---|---|:---:|---|:---:|:---:|
| 1 | 10 | 3 | (10,10) | 1 | ✅ |
| 2 | 20 | 1 | (20,20) | 2 | ❌ both fail |
| 3 | 10 | 3 | (20,20) | 2 | ❌ location shared |
| 4 | 10 | 3 | (40,40) | 1 | ✅ |

Policies 1 and 4 survive: `5 + 40 = 45`.

**`PARTITION BY lat, lon` must list both columns together.** Two separate windows — one on `lat`, one on `lon` — would ask "is this latitude unique?" and "is this longitude unique?", which is a different and wrong question. The city is the *pair*.

### 💡 Solution

```sql
WITH policy_count AS (
    SELECT *,
           COUNT(*) OVER (PARTITION BY tiv_2015) AS tiv_2015_count,
           COUNT(*) OVER (PARTITION BY lat, lon) AS lat_lon_count
    FROM Insurance
)
SELECT ROUND(SUM(tiv_2016), 2) AS tiv_2016
FROM policy_count
WHERE tiv_2015_count > 1
  AND lat_lon_count = 1;
```

### 🧠 Explanation

- **Two window functions over different partitions in the same `SELECT` is completely legal** and is the reason this solution reads so cleanly. Each `OVER` clause is independent; MySQL evaluates them against the same input rows. The equivalent correlated-subquery version scans `Insurance` twice more per row and is markedly harder to read:

  ```sql
  WHERE (SELECT COUNT(*) FROM Insurance i2 WHERE i2.tiv_2015 = i.tiv_2015) > 1
    AND (SELECT COUNT(*) FROM Insurance i3 WHERE i3.lat = i.lat AND i3.lon = i.lon) = 1
  ```

- **Neither `COUNT` has an `ORDER BY`**, so both count the whole partition rather than running a total. Same deliberate omission as [Set 4 Q1](set-04-advanced-windows-and-conditional-logic.md#q1--primary-department-for-each-employee).
- **`COUNT(*) OVER (PARTITION BY x) > 1` is "this value is shared"; `= 1` is "this value is unique".** Two of the most reusable predicates in SQL — duplicate detection, uniqueness auditing, and deduplication all reduce to one of these.
- **The filter runs outside the CTE** because window functions aren't visible to `WHERE` in the query that defines them. That constraint has now shaped the solution in all five sets.
- 💡 **`ROUND(SUM(...), 2)` is a formatting step, not a correctness one** here — but it matters that it wraps the `SUM`, not each row. Rounding per row and then summing accumulates error.

---

## Q3 — Department Top Three Salaries

🔗 **[leetcode.com/problems/department-top-three-salaries](https://leetcode.com/problems/department-top-three-salaries)**

**Problem:** A person is a **high earner** if they're in the **top three distinct salaries** for their department. Report all high earners.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Employee;
DROP TABLE IF EXISTS Department;

CREATE TABLE Department (
    id   INT PRIMARY KEY,
    name VARCHAR(30)
);
INSERT INTO Department VALUES (1,'IT'),(2,'Sales');

CREATE TABLE Employee (
    id           INT PRIMARY KEY,
    name         VARCHAR(30),
    salary       INT,
    departmentId INT
);
INSERT INTO Employee VALUES
(1,'Joe',85000,1),(2,'Henry',80000,2),(3,'Sam',60000,2),(4,'Max',90000,1),
(5,'Janet',69000,1),(6,'Randy',85000,1),(7,'Will',70000,1);
```

### ✅ Expected Output

| Department | Employee | Salary |
|------------|----------|--------|
| IT | Max | 90000 |
| IT | Joe | 85000 |
| IT | Randy | 85000 |
| IT | Will | 70000 |
| Sales | Henry | 80000 |
| Sales | Sam | 60000 |

### 🎯 How to approach it

**Read "top three salaries" as *three distinct salary values*, not *three people*.** IT returns **four** employees, and that number is the whole test:

| name | salary | `DENSE_RANK` | `RANK` | `ROW_NUMBER` |
|---|---|:---:|:---:|:---:|
| Max | 90000 | 1 | 1 | 1 |
| Joe | 85000 | 2 | 2 | 2 |
| Randy | 85000 | **2** | 2 | **3** |
| Will | 70000 | **3** | **4** | **4** |
| Janet | 69000 | 4 | 5 | 5 |

That's verified output, and it shows why the function choice is the answer:

- **`DENSE_RANK`** — Joe and Randy tie at 2, Will gets 3. Filtering `<= 3` keeps all four. ✅
- **`RANK`** — the tie *consumes* rank 3, pushing Will to 4. He's excluded. ❌
- **`ROW_NUMBER`** — no ties at all, so Randy takes rank 3 and Will is 4. Excluded, and Randy's inclusion is arbitrary. ❌

**`DENSE_RANK` is the only function that counts distinct values**, which is exactly what the problem asks for. Same reasoning as [Set 3 Q5](set-03-window-functions-and-first-rows.md#q5--product-sales-analysis-iii), where the requirement's plurality picked the function.

### 💡 Solution

```sql
WITH department_ranking AS (
    SELECT e.*,
           d.name AS department,
           DENSE_RANK() OVER (PARTITION BY e.departmentId ORDER BY e.salary DESC) AS ranking
    FROM Employee e
    JOIN Department d
        ON e.departmentId = d.id
)
SELECT department AS Department,
       name       AS Employee,
       salary     AS Salary
FROM department_ranking
WHERE ranking <= 3;
```

### 🧠 Explanation

- **`PARTITION BY e.departmentId` restarts the ranking per department**, which is why Sales' 80000 ranks 1 even though IT has three salaries above it. Top-N *per group* always means `PARTITION BY` the group.
- **The join happens inside the CTE, before ranking.** It could equally happen after; ranking on `departmentId` and joining `Department` in the outer query gives the same result. Joining first keeps the outer query to a plain projection.
- **`ranking <= 3` is the generalisation of `= 1`.** Every first-row problem in Sets 3 and 4 was this pattern with N = 1 — top-N per group is the same query with a different constant, which is why it's worth seeing them together.
- **Column aliases are capitalised to match LeetCode's expected headers** (`Department`, `Employee`, `Salary`). Header matching has cost more submissions across these 50 problems than any single piece of logic.
- 💡 **This is the canonical "top N per group" problem** and the most likely of all 50 to appear verbatim in an interview. The follow-up is almost always "now do it without window functions" — the answer is a correlated subquery counting distinct higher salaries:

  ```sql
  WHERE (SELECT COUNT(DISTINCT e2.salary) FROM Employee e2
         WHERE e2.salary > e.salary AND e2.departmentId = e.departmentId) < 3
  ```

  Note `COUNT(DISTINCT ...)` — that's what reproduces `DENSE_RANK` semantics rather than `RANK`.

---

## Q4 — Fix Names in a Table

🔗 **[leetcode.com/problems/fix-names-in-a-table](https://leetcode.com/problems/fix-names-in-a-table/description)**

**Problem:** Fix the names so that only the **first character is uppercase** and the rest are lowercase. Order by `user_id`.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Users;
CREATE TABLE Users (
    user_id INT PRIMARY KEY,
    name    VARCHAR(40)
);

INSERT INTO Users VALUES (1,'aLice'),(2,'bOB');
```

### ✅ Expected Output

| user_id | name |
|---------|------|
| 1 | Alice |
| 2 | Bob |

### 🎯 How to approach it

**Split, transform each part, rejoin** — the standard shape for any string reformatting.

1. **First character:** `SUBSTRING(name, 1, 1)` → `'a'`, then `UPPER()` → `'A'`.
2. **The rest:** `SUBSTRING(name, 2)` → `'Lice'`, then `LOWER()` → `'lice'`.
3. **Rejoin:** `CONCAT('A', 'lice')` → `'Alice'`.

Two details that trip people up: **SQL strings are 1-indexed**, so the first character is position 1, not 0. And **`SUBSTRING(name, 2)` with no length argument runs to the end of the string** — no need to compute `LENGTH(name) - 1`.

### 💡 Solution

```sql
SELECT user_id,
       CONCAT(UPPER(SUBSTRING(name, 1, 1)), LOWER(SUBSTRING(name, 2))) AS name
FROM Users
ORDER BY user_id;
```

### 🧠 Explanation

- **`LOWER` on the tail is not optional.** `'bOB'` needs its `'OB'` flattened to `'ob'`; without it the output is `'BOB'`. The problem says *only* the first character is uppercase, so both halves need explicit handling.
- **`LEFT(name, 1)` and `SUBSTRING(name, 2)`** are an equally valid pair, and `LEFT`/`RIGHT` read more clearly when you're taking from an end. `SUBSTRING` is the more general tool and the one that ports everywhere (`SUBSTR` is the alias; PostgreSQL and Oracle use the same name).
- **A single-character name still works.** `SUBSTRING('a', 2)` returns an empty string, not `NULL`, so `CONCAT` yields `'A'`. Worth knowing, because `CONCAT` with a genuine `NULL` argument returns `NULL` for the whole expression — one `NULL` poisons the concatenation. If any input column were nullable you'd want `CONCAT_WS` or `COALESCE`.
- 💡 **This is an `UPDATE` in disguise.** The problem asks for a `SELECT`, but the real-world version is `UPDATE Users SET name = CONCAT(...)` — the same expression, applied in place. Data cleanup is where these string functions actually earn their keep.

---

## Q5 — Patients With a Condition

🔗 **[leetcode.com/problems/patients-with-a-condition](https://leetcode.com/problems/patients-with-a-condition/description/)**

**Problem:** Find patients with **Type I Diabetes**, whose condition code starts with the prefix **`DIAB1`**. The `conditions` column holds space-separated codes.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Patients;
CREATE TABLE Patients (
    patient_id   INT PRIMARY KEY,
    patient_name VARCHAR(30),
    conditions   VARCHAR(100)
);

INSERT INTO Patients VALUES
(1,'Daniel','YFEV COUGH'),(2,'Alice',''),(3,'Bob','DIAB100 MYOP'),
(4,'George','ACNE DIAB100'),(5,'Alain','DIAB201');
```

### ✅ Expected Output

| patient_id | patient_name | conditions |
|------------|--------------|------------|
| 3 | Bob | DIAB100 MYOP |
| 4 | George | ACNE DIAB100 |

### 🎯 How to approach it

**This is a word-boundary problem wearing a `LIKE` costume**, and the obvious answer is wrong.

`LIKE '%DIAB1%'` matches the substring *anywhere*, including in the middle of another code — `SDIAB100` or `XDIAB1` would pass, and those are different diagnoses. The prefix must start a **word**, which means one of exactly two things:

1. **It's the first code in the string** → `LIKE 'DIAB1%'`
2. **It follows a space** → `LIKE '% DIAB1%'`

Note the space inside the second pattern. That's the entire mechanism, and it's why the two patterns can't be collapsed into one.

Bob matches pattern 1, George matches pattern 2. **`DIAB201` correctly fails both** — it's Type II, and the prefix `DIAB1` simply isn't there. Verified against a table containing `SDIAB100`, which scores 0 as required.

### 💡 Solution

```sql
SELECT *
FROM Patients
WHERE conditions LIKE "DIAB1%"
   OR conditions LIKE "% DIAB1%";
```

### 🧠 Explanation

- **`%` matches any sequence including empty; `_` matches exactly one character.** `'DIAB1%'` anchors to the start because there's no leading `%`.
- ⚠️ **`LIKE` is case-insensitive under MySQL's default collation.** With `utf8mb4_0900_ai_ci`, `'diab100 myop'` matches `'DIAB1%'` — verified. LeetCode's data is uppercase so it passes, but a real EHR with mixed-case entry would return false positives. `LIKE BINARY 'DIAB1%'` forces a case-sensitive comparison, which is the same guard your [Q10](#q10--find-users-with-valid-e-mails) solution applies to email domains.
- **The `REGEXP` alternative expresses the boundary directly** instead of enumerating two cases:

  ```sql
  WHERE conditions REGEXP '(^|[[:space:]])DIAB1'
  ```

  `(^|[[:space:]])` means "start of string, or a whitespace character" — one pattern covering both positions. More expressive, and it scales when the boundary rule gets more complex. `LIKE` is faster on an indexed prefix, though neither uses an index here since both start with a wildcard branch.
- 💡 **The real lesson is about the schema.** Space-separated codes in a `VARCHAR` violate first normal form, which is why matching them needs these gymnastics. In production this is a `patient_conditions` join table with one code per row, and the whole problem becomes `WHERE code LIKE 'DIAB1%'` with an index behind it. Being able to say that is worth more than the query.

---

## Q6 — Delete Duplicate Emails

🔗 **[leetcode.com/problems/delete-duplicate-emails](https://leetcode.com/problems/delete-duplicate-emails/description/)**

**Problem:** **Delete** all duplicate emails, keeping only the one with the **smallest `id`**. This problem modifies the table rather than returning a result set.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Person;
CREATE TABLE Person (
    id    INT PRIMARY KEY,
    email VARCHAR(60)
);

INSERT INTO Person VALUES (1,'john@example.com'),(2,'bob@example.com'),(3,'john@example.com');
```

### ✅ Expected Output — the table after deletion

| id | email |
|----|-------|
| 1 | john@example.com |
| 2 | bob@example.com |

### 🎯 How to approach it

**Start with the obvious answer, because MySQL rejects it outright.** "Delete rows that aren't the minimum id for their email" translates to:

```sql
DELETE FROM Person WHERE id NOT IN (SELECT MIN(id) FROM Person GROUP BY email);
```

which fails with:

```
ERROR 1093 (HY000): You can't specify target table 'Person' for update in FROM clause
```

That's verified, not remembered. **MySQL refuses to read a table in a subquery while deleting from it** in the same statement, because the result would depend on evaluation order. Every working solution is a way around this restriction, so knowing *why* it exists explains both solutions below.

**Route 1 — self-join.** Join `Person` to itself on matching emails where the left id is larger, then delete the left side. A row is only matched if some *smaller* id shares its email, which is precisely the definition of "not the one to keep".

**Route 2 — a CTE.** MySQL 8 materialises the CTE first, so it's no longer "reading the target table in the `FROM` clause" as far as the restriction is concerned. `ROW_NUMBER()` numbers each email's rows by id and everything above 1 goes.

### 💡 Solution — self-join

```sql
DELETE P1
FROM Person P1
JOIN Person P2
    ON P1.id > P2.id
   AND P1.email = P2.email;
```

### 💡 Solution — CTE + `ROW_NUMBER()`

```sql
WITH person_email_ranking AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY email ORDER BY id) AS ranking
    FROM Person
)
DELETE FROM Person
WHERE id IN (
    SELECT id
    FROM person_email_ranking
    WHERE ranking > 1
);
```

Both verified to leave exactly ids 1 and 2.

### 🧠 Explanation

- **`DELETE P1 FROM Person P1 JOIN ...` — the alias after `DELETE` names which side to remove.** This is easy to miss and it is the entire statement: swap it to `DELETE P2` and you'd keep the *largest* id instead. In a multi-table delete MySQL needs to be told explicitly, since a join produces rows from both tables.
- **`P1.id > P2.id` does double duty** — it identifies duplicates *and* encodes which one survives. Flip to `<` and you keep the maximum id, which is the "keep the newest record" variant you'll meet more often in practice.
- **The CTE version works in MySQL 8.0+ only.** `WITH` before `DELETE` was added in 8.0, and window functions in 8.0 as well — on 5.7 the self-join is the only option of the two.
- **`ROW_NUMBER()`, not `RANK()` or `DENSE_RANK()`.** Duplicates share an email but you must keep exactly one, so the ranking has to break ties. `RANK()` would give every duplicate rank 1 and delete nothing. This is the mirror image of [Q3](#q3--department-top-three-salaries), where ties had to be preserved.
- ⚠️ **Run the `SELECT` before the `DELETE`.** Swap `DELETE FROM Person WHERE id IN (...)` for `SELECT * FROM Person WHERE id IN (...)` and confirm the row set is what you expect. On a real table there is no undo, and a transaction (`START TRANSACTION` … `ROLLBACK`) is the safety net when you can't preview.
- 💡 **The durable fix isn't a `DELETE` at all** — it's `ALTER TABLE Person ADD UNIQUE (email)` once the duplicates are cleared, so they can't come back. Deduplication scripts that run on a schedule are usually a missing constraint in disguise.

---

## Q7 — Second Highest Salary

🔗 **[leetcode.com/problems/second-highest-salary](https://leetcode.com/problems/second-highest-salary/)**

**Problem:** Report the **second highest distinct** salary. If it doesn't exist, return `null`.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Employee;
CREATE TABLE Employee (
    id     INT PRIMARY KEY,
    salary INT
);

INSERT INTO Employee VALUES (1,100),(2,200),(3,300);
```

### ✅ Expected Output

| SecondHighestSalary |
|---------------------|
| 200 |

### 🎯 How to approach it

**The `null` requirement is the problem.** Finding the second highest salary is trivial; returning a row containing `null` when there *is* no second highest is what the hidden tests check, and it's where the natural answer fails.

With a single employee, `ORDER BY salary DESC LIMIT 1 OFFSET 1` returns **no rows at all** — verified. An empty result set is not the same as a row containing `null`, and LeetCode marks it wrong. **Wrapping the query in an outer `SELECT` fixes it**: a scalar subquery that matches nothing evaluates to `NULL`, and the outer `SELECT` still produces its one row.

The `MAX` formulation gets there differently: "the largest salary below the largest salary". Over an empty filtered set, `MAX` returns `NULL` by definition — no wrapper needed, the same property that carried [Set 3 Q8](set-03-window-functions-and-first-rows.md#q8--biggest-single-number).

### 💡 Solution — `MAX` below the max

```sql
SELECT MAX(salary) AS SecondHighestSalary
FROM Employee
WHERE salary < (SELECT MAX(salary) FROM Employee);
```

### 💡 Solution — wrapped `LIMIT`/`OFFSET`

```sql
SELECT (SELECT DISTINCT salary
        FROM Employee
        ORDER BY salary DESC
        LIMIT 1 OFFSET 1) AS SecondHighestSalary;
```

### 🧠 Explanation

- ⚠️ **The `DISTINCT` in the second solution is load-bearing.** The problem says *second highest **distinct** salary*. Without it, two employees tied at the top push the offset onto the second copy of the same value and the query returns the highest salary as though it were the second:

  | Data | `LIMIT 1 OFFSET 1` without `DISTINCT` | with `DISTINCT` | `MAX` form |
  |---|:---:|:---:|:---:|
  | 300, 300, 200 | **300** ❌ | **200** ✅ | **200** ✅ |

  That's verified output. LeetCode's tests don't appear to include tied top salaries, so the version without `DISTINCT` is accepted — but it answers the wrong question, and a tie at the top of a salary table is not exotic. **The `MAX` form is immune**, because `salary < (SELECT MAX(salary))` excludes *every* copy of the top value in one stroke.
- **Both forms return `NULL` on a single-row table** — verified. The mechanisms differ: the `MAX` form aggregates over zero rows, while the wrapped form relies on a scalar subquery with no result evaluating to `NULL`.
- **`LIMIT 1 OFFSET 1` is the pattern that generalises.** For the Nth highest, change the offset to N-1 — which is exactly how the follow-up problem (*Nth Highest Salary*) is solved, since it takes a parameter. The `MAX`-below-`MAX` trick doesn't extend past 2 without nesting.
- 💡 **The window-function answer** is the one to lead with in an interview, because it generalises and it's a single pass: `SELECT MAX(salary) FROM (SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) r FROM Employee) t WHERE r = 2`. `DENSE_RANK` handles the distinctness for free, and swapping `2` for any N is the whole change.

---

## Q8 — Group Sold Products By The Date

🔗 **[leetcode.com/problems/group-sold-products-by-the-date](https://leetcode.com/problems/group-sold-products-by-the-date/description)**

**Problem:** For each sell date, report **how many distinct products** were sold and a **comma-separated, alphabetically sorted list** of them. Order by date.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Activities;
CREATE TABLE Activities (
    sell_date DATE,
    product   VARCHAR(30)
);

INSERT INTO Activities VALUES
('2020-05-30','Headphone'),('2020-06-01','Pencil'),('2020-06-02','Mask'),
('2020-05-30','Basketball'),('2020-06-01','Bible'),('2020-06-02','Mask'),
('2020-05-30','T-Shirt');
```

### ✅ Expected Output

| sell_date | num_sold | products |
|-----------|----------|----------|
| 2020-05-30 | 3 | Basketball,Headphone,T-Shirt |
| 2020-06-01 | 2 | Bible,Pencil |
| 2020-06-02 | 1 | Mask |

### 🎯 How to approach it

**Collapsing rows into a delimited string is what `GROUP_CONCAT` exists for**, and this problem is really a checklist of its three modifiers:

1. **`DISTINCT`** — 2020-06-02 has `Mask` twice. Without it the output reads `Mask,Mask` and `num_sold` disagrees with the list.
2. **`ORDER BY product`** — the sort goes *inside* the function call, not in the query's `ORDER BY`. Alphabetical order within each group is required, and it's independent of how rows are ordered overall.
3. **The separator** — defaults to a comma, which is exactly what's wanted, so `SEPARATOR` can be omitted. Any other delimiter needs `SEPARATOR ' | '` explicitly.

`COUNT(DISTINCT product)` supplies `num_sold` from the same grouping.

### 💡 Solution

```sql
SELECT sell_date,
       COUNT(DISTINCT product) AS num_sold,
       GROUP_CONCAT(DISTINCT product ORDER BY product) AS products
FROM Activities
GROUP BY sell_date
ORDER BY sell_date;
```

### 🧠 Explanation

- **`ORDER BY` inside an aggregate is unusual and specific to `GROUP_CONCAT`.** The syntax is `GROUP_CONCAT(DISTINCT expr ORDER BY expr SEPARATOR ',')` — no comma between the expression and `ORDER BY`. It's the one place in SQL where an aggregate takes a sort.
- **`COUNT(DISTINCT product)` and the `DISTINCT` inside `GROUP_CONCAT` are separate deduplications.** Drop either one and the two columns tell different stories about the same day.
- ⚠️ **`GROUP_CONCAT` truncates silently at `group_concat_max_len`, which defaults to 1024 bytes.** No error, no warning in the result — the string just stops. Verified by lowering the limit to 20: the 2020-05-30 list came back as `Basketball,Headphone` with `T-Shirt` simply gone, while `num_sold` still said 3. **On real data this is a genuine reporting bug**, and the fix is `SET SESSION group_concat_max_len = 1000000;` before the query. It's the single most useful thing to know about this function.
- **The outer `ORDER BY sell_date` is separate from the inner sort.** One orders the result rows, the other orders items within each row's string.
- 💡 **This is a one-dimensional pivot.** `GROUP_CONCAT` is how you flatten a one-to-many into a single readable cell — order line items, user roles, tags on an article. PostgreSQL spells it `STRING_AGG`, SQL Server `STRING_AGG`, Oracle `LISTAGG`; the concept is identical and interviewers use the names interchangeably.

---

## Q9 — List the Products Ordered in a Period

🔗 **[leetcode.com/problems/list-the-products-ordered-in-a-period](https://leetcode.com/problems/list-the-products-ordered-in-a-period/description/)**

**Problem:** Report products with **at least 100 units ordered in February 2020**, with their total units.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Orders;
DROP TABLE IF EXISTS Products;

CREATE TABLE Products (
    product_id       INT PRIMARY KEY,
    product_name     VARCHAR(40),
    product_category VARCHAR(40)
);
INSERT INTO Products VALUES
(1,'Leetcode Solutions','Book'),(2,'Jewels of Stringology','Book'),
(3,'HP','Laptop'),(4,'Lenovo','Laptop'),(5,'Leetcode Kit','T-shirt');

CREATE TABLE Orders (
    product_id INT,
    order_date DATE,
    unit       INT
);
INSERT INTO Orders VALUES
(1,'2020-02-05',60),(1,'2020-02-10',70),(2,'2020-01-18',30),(2,'2020-02-11',80),
(3,'2020-02-17',2),(3,'2020-02-24',3),(4,'2020-03-01',20),(4,'2020-03-04',30),
(4,'2020-03-04',60),(5,'2020-02-25',50),(5,'2020-02-27',50),(5,'2020-03-01',50);
```

### ✅ Expected Output

| product_name | unit |
|--------------|------|
| Leetcode Solutions | 130 |
| Leetcode Kit | 100 |

### 🎯 How to approach it

**Three steps in the right order, and the order is what makes it correct:**

1. **Filter to February** with `WHERE` — a row-level condition, so it runs before grouping. Product 5's 2020-03-01 order of 50 is excluded here, which is why its total is 100 and not 150.
2. **Sum per product** with `GROUP BY`.
3. **Filter on the sum** with `HAVING` — a group-level condition that can only be evaluated after the totals exist.

Putting the date filter in `HAVING` would still work but scans and groups rows it then throws away. Putting the unit filter in `WHERE` is simply impossible, since `SUM` doesn't exist yet. **`WHERE` before, `HAVING` after** is the rule, and this problem is the clean illustration of it.

`>= 100` for "at least 100" — Leetcode Kit lands on exactly 100 and must be included.

### 💡 Solution

```sql
SELECT p.product_name,
       SUM(o.unit) AS unit
FROM Products p
JOIN Orders o
    ON p.product_id = o.product_id
WHERE o.order_date >= "2020-02-01"
  AND o.order_date <  "2020-03-01"
GROUP BY p.product_id, p.product_name
HAVING unit >= 100;
```

### 🧠 Explanation

- **`HAVING unit >= 100` refers to the `SELECT` alias, not the raw `Orders.unit` column** — MySQL resolves aliases in `HAVING`, and it picks the alias over the base column here. Verified identical to writing `HAVING SUM(o.unit) >= 100`. **Standard SQL does not guarantee this**, and the shadowing is genuinely ambiguous to a reader, so `HAVING SUM(o.unit) >= 100` is the version that both ports and explains itself.
- **The half-open date range `>= '2020-02-01' AND < '2020-03-01'`** is the right way to express a month, for the reasons covered in [Set 4 Q9](set-04-advanced-windows-and-conditional-logic.md#q9--movie-rating): the column stays bare so an index can be used, and it survives the column becoming a `DATETIME`. `BETWEEN '2020-02-01' AND '2020-02-29'` silently drops anything timestamped after midnight on the 29th.
- **`GROUP BY p.product_id, p.product_name` groups by the key first.** Grouping by name alone would merge two distinct products that happen to share a name.
- **`INNER JOIN` is correct** because a product with no February orders shouldn't appear at all — and it wouldn't clear 100 units anyway. HP and Lenovo drop out on the `HAVING`, Jewels of Stringology on its 80-unit total.
- 💡 **This is the shape of nearly every "top customers/products this period" report** you'll write professionally: filter the window, aggregate per entity, threshold the aggregate. Recognising it means you write it once and reuse it forever.

---

## Q10 — Find Users With Valid E-Mails

🔗 **[leetcode.com/problems/find-users-with-valid-e-mails](https://leetcode.com/problems/find-users-with-valid-e-mails/description/)**

**Problem:** Find users with a **valid email**: a prefix that **starts with a letter**, contains only **letters, digits, underscore `_`, period `.`, and dash `-`**, followed by the domain **`@leetcode.com`**.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Users;
CREATE TABLE Users (
    user_id INT PRIMARY KEY,
    name    VARCHAR(30),
    mail    VARCHAR(60)
);

INSERT INTO Users VALUES
(1,'Winston','winston@leetcode.com'),(2,'Jonathan','jonathanisgreat'),
(3,'Annabelle','bella-@leetcode.com'),(4,'Sally','sally.come@leetcode.com'),
(5,'Marwan','quarz#2@leetcode.com'),(6,'David','david69@gmail.com'),
(7,'Shapiro','.shapo@leetcode.com');
```

### ✅ Expected Output

| user_id | name | mail |
|---------|------|------|
| 1 | Winston | winston@leetcode.com |
| 3 | Annabelle | bella-@leetcode.com |
| 4 | Sally | sally.come@leetcode.com |

### 🎯 How to approach it

**Translate the specification into the pattern one clause at a time**, then check what the engine does that the pattern didn't say.

| Fragment | Meaning | Rejects |
|---|---|---|
| `^` | anchor to the start | — |
| `[a-zA-Z]` | first character must be a letter | `.shapo@` (Shapiro) |
| `[a-zA-Z0-9_.-]*` | any number of allowed characters | `quarz#2@` (Marwan) — `#` isn't listed |
| `@leetcode\.com` | the literal domain, dot escaped | `@gmail.com` (David) |
| `$` | anchor to the end | trailing junk after `.com` |

Two details inside the character class: **`.` is literal inside `[...]`** so it needs no escape there, while **outside a class it means "any character"** and must be escaped as `\.` — an unescaped `@leetcode.com` would match `@leetcodeXcom`. And **`-` is placed last** so it reads as a literal dash rather than a range.

`*` and not `+`, because a one-letter prefix like `a@leetcode.com` is valid — the first letter is already consumed by `[a-zA-Z]`.

### 💡 Solution

```sql
SELECT *
FROM Users
WHERE mail REGEXP "^[a-zA-Z][a-zA-Z0-9_.-]*@leetcode\\.com$"
  AND mail LIKE BINARY "%@leetcode.com";
```

### 🧠 Explanation

- ⚠️ **The `LIKE BINARY` guard is doing real work, and without it the regex alone is wrong.** MySQL's default collation `utf8mb4_0900_ai_ci` is case-insensitive, and `REGEXP` inherits that — so `@leetcode\.com$` happily matches uppercase domains. Verified on three spellings of the same address:

  | Filter | Rows passed |
  |---|---|
  | `REGEXP` only | `winston@leetcode.com`, `winston@LeetCode.com`, `winston@LEETCODE.COM` ❌ |
  | `REGEXP` + `LIKE BINARY` | `winston@leetcode.com` ✅ |

  `BINARY` forces a byte-for-byte comparison, so only the exact lowercase domain survives. **The domain is specified as literally `@leetcode.com`**, and hidden tests do probe case variants — this guard is why the solution is robust rather than lucky.
- **The alternative is to make the regex itself case-sensitive** rather than bolting on a second predicate: `WHERE mail COLLATE utf8mb4_bin REGEXP '...'`. One predicate instead of two, verified to pass only the lowercase domain. ⚠️ **`REGEXP BINARY` — the form you'll see in older answers — no longer works.** MySQL 8 rebuilt `REGEXP` on ICU, and it now raises `ERROR 3995: Character set 'utf8mb4_0900_ai_ci' cannot be used in conjunction with 'binary' in call to regexp_like`. Use `COLLATE`, or the `LIKE BINARY` guard above.
- **`\\.` in a SQL string literal becomes `\.` at the regex engine.** MySQL processes backslash escapes in string literals first, so the doubling is required. Using single-quoted `'...\\.'` behaves the same way. This double-escaping catches everyone once.
- **`[a-zA-Z]` rather than relying on case-insensitivity.** Since the collation is already case-insensitive, `[a-z]` would work here — but writing both ranges states the intent and survives a change of collation.
- 💡 **Don't validate real email addresses this way.** The genuine RFC 5322 grammar permits quoted strings, plus-addressing, internationalised domains, and comments; every "email regex" is an approximation. For a known single domain like this it's fine. For user signups, validate by sending a confirmation link — that's the only check that proves an address exists.

---

## 🧠 Set 5 — Patterns Worth Memorising

| Pattern | Trigger phrase | Tool |
|---|---|---|
| Flatten a symmetric two-column relationship | "A and B are both friends" | `UNION ALL` of both columns, then `GROUP BY` |
| Two conditions at two different grains | "same X as someone else, but unique Y" | Two `COUNT(*) OVER (PARTITION BY …)` in one pass |
| "Is this value shared / unique?" | duplicate detection, uniqueness audit | `COUNT(*) OVER (PARTITION BY v) > 1` / `= 1` |
| Top N **distinct values** per group | "top three salaries" | `DENSE_RANK()`, filter `<= N` |
| Capitalise / reformat a string | "only the first letter uppercase" | `CONCAT(UPPER(SUBSTRING(s,1,1)), LOWER(SUBSTRING(s,2)))` |
| Match a whole word inside a delimited list | "condition code starting with X" | `LIKE 'X%' OR LIKE '% X%'`, or `REGEXP '(^\|[[:space:]])X'` |
| Delete duplicates keeping the smallest id | "remove duplicate emails" | `DELETE t1 FROM t t1 JOIN t t2 ON t1.id > t2.id AND t1.k = t2.k` |
| Read the target table while deleting from it | error 1093 | Self-join, or a CTE (MySQL 8+) |
| Return `NULL` rather than no row | "return null if it doesn't exist" | Wrap in an outer `SELECT (…)`, or use `MAX()` |
| Nth highest distinct value | "second highest salary" | `DENSE_RANK() = N`, or `DISTINCT … LIMIT 1 OFFSET N-1` |
| Rows into one delimited cell | "comma-separated list of products" | `GROUP_CONCAT(DISTINCT x ORDER BY x)` |
| A calendar month, index-friendly | "ordered in February 2020" | `>= '2020-02-01' AND < '2020-03-01'` |
| Threshold an aggregate | "at least 100 units" | `WHERE` to filter rows, `HAVING` to filter groups |
| Pattern-validate a string, case-sensitively | "valid email" | `REGEXP` + `LIKE BINARY`, or `COLLATE utf8mb4_bin REGEXP` |

### Five mistakes that cost the most submissions

1. **`UNION` instead of `UNION ALL` when counting.** Deduplication destroys the counts — the answer became "id 1, num 1" instead of "id 3, num 3". (Q1)
2. **`RANK()` or `ROW_NUMBER()` for "top N distinct".** Both drop a legitimate high earner when two people tie. Only `DENSE_RANK` counts distinct values. (Q3)
3. **`LIKE '%X%'` for a word prefix.** It matches mid-word, so `SDIAB100` passes. Anchor with `'X%' OR '% X%'`. (Q5)
4. **Assuming `REGEXP` and `LIKE` are case-sensitive.** They aren't under the default collation, so `@LEETCODE.COM` passes a lowercase pattern. Add `BINARY`. (Q10, and latent in Q5)
5. **Returning an empty result set where `NULL` was required.** `LIMIT 1 OFFSET 1` returns nothing on a one-row table; the outer `SELECT` wrapper is what turns that into a `NULL` row. (Q7)

---

## 🏁 That's all 50

Across the five sets you've now covered the complete [LeetCode SQL 50](https://leetcode.com/studyplan/top-sql-50/):

| Set | Problems | What it established |
|---|---|---|
| [1 — Select & Basic Joins](set-01-easy-basics.md) | 1–10 | `WHERE` logic, `NULL` three-valued traps, `INNER` vs `LEFT JOIN`, anti-joins |
| [2 — Joins & Aggregation](set-02-joins-and-aggregation.md) | 11–20 | `CROSS JOIN` scaffolding, `COUNT(*)` vs `COUNT(col)`, conditional aggregation, rates |
| [3 — Window Functions & First-Row Logic](set-03-window-functions-and-first-rows.md) | 21–30 | First-row-per-group, `DENSE_RANK` on ties, `COUNT(DISTINCT)`, relational division |
| [4 — Advanced Windows & Conditional Logic](set-04-advanced-windows-and-conditional-logic.md) | 31–40 | Window **frames**, running totals, moving averages, `UNION ALL` for empty buckets |
| **5 — Strings, Regex & Set Operations** | 41–50 | String functions, word-boundary matching, `GROUP_CONCAT`, `DELETE`, top-N per group |

**If you remember only three things from all fifty:**

1. **`WHERE` runs before window functions exist.** Rank in a subquery or CTE, filter outside. It shaped the solution in Sets 3, 4, and 5.
2. **`COUNT(*)` counts rows; `COUNT(col)` counts non-`NULL` values; `COUNT(DISTINCT col)` counts distinct ones.** Choosing wrong is the most common silent wrong answer in the whole set.
3. **A filter on the right-hand table of a `LEFT JOIN` belongs in `ON`, not `WHERE`.** Otherwise the outer join quietly becomes an inner one.

---

<div align="center">

**🎉 SQL 50 complete.** ⭐ the repo if it helped, and check the [Interview Problem Sets](../interview-questions/) for patterns the study plan doesn't cover.

[⬅ Back to Course Home](../README.md) · [Set 1](set-01-easy-basics.md) · [Set 2](set-02-joins-and-aggregation.md) · [Set 3](set-03-window-functions-and-first-rows.md) · [Set 4](set-04-advanced-windows-and-conditional-logic.md) · [LeetCode SQL 50 study plan ↗](https://leetcode.com/studyplan/top-sql-50/)

</div>
