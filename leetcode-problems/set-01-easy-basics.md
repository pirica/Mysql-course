# 🟩 LeetCode SQL 50 — Set 1: Select & Basic Joins

> **Problems 1–10 of the [LeetCode SQL 50](https://leetcode.com/studyplan/top-sql-50/) study plan.**
> Every problem follows the same flow: **Problem → Schema → Sample Input → Expected Output → Approach → Solution → Explanation.**

The **Approach** section is the important one. On LeetCode it's easy to memorise an answer and learn nothing — so before each solution we work out *how to read the problem* and *which tool the wording points to*. That's the transferable skill in an interview.

All queries target **MySQL 8.0+** and were executed against LeetCode's own sample data — the Expected Output blocks below are real query output, not copied from the problem page.

---

## 📋 Contents

| # | Problem | Difficulty | Core Concept |
|---|---------|:---:|--------------|
| 1 | [Recyclable and Low Fat Products](#q1--recyclable-and-low-fat-products) | 🟩 Easy | `WHERE` + `AND` |
| 2 | [Find Customer Referee](#q2--find-customer-referee) | 🟩 Easy | `NULL` is not "not equal" |
| 3 | [Big Countries](#q3--big-countries) | 🟩 Easy | `WHERE` + `OR` |
| 4 | [Article Views I](#q4--article-views-i) | 🟩 Easy | Self-comparison + `DISTINCT` |
| 5 | [Invalid Tweets](#q5--invalid-tweets) | 🟩 Easy | `LENGTH` vs `CHAR_LENGTH` |
| 6 | [Replace Employee ID With The Unique Identifier](#q6--replace-employee-id-with-the-unique-identifier) | 🟩 Easy | `LEFT JOIN` (keep unmatched) |
| 7 | [Product Sales Analysis I](#q7--product-sales-analysis-i) | 🟩 Easy | `INNER JOIN` on a foreign key |
| 8 | [Customer Who Visited but Did Not Make Any Transactions](#q8--customer-who-visited-but-did-not-make-any-transactions) | 🟩 Easy | Anti-join + `GROUP BY` |
| 9 | [Rising Temperature](#q9--rising-temperature) | 🟩 Easy | `LAG()` + `DATEDIFF` |
| 10 | [Average Time of Process per Machine](#q10--average-time-of-process-per-machine) | 🟩 Easy | Self-join to pair rows |

---

## Q1 — Recyclable and Low Fat Products

🔗 **[leetcode.com/problems/recyclable-and-low-fat-products](https://leetcode.com/problems/recyclable-and-low-fat-products/description/)**

**Problem:** Find the IDs of products that are both **low fat** and **recyclable**.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Products;
CREATE TABLE Products (
    product_id INT PRIMARY KEY,
    low_fats   ENUM('Y','N'),
    recyclable ENUM('Y','N')
);

INSERT INTO Products VALUES
(0,'Y','N'),(1,'Y','Y'),(2,'N','Y'),(3,'Y','Y'),(4,'N','N');
```

### ✅ Expected Output

| product_id |
|------------|
| 1 |
| 3 |

### 🎯 How to approach it

The word **"both"** in the problem statement is the whole hint. Two conditions that must *simultaneously* hold → `AND`. Read the wording as a checklist:

| Wording | Operator |
|---|---|
| "both A **and** B" | `AND` — every condition must be true |
| "A **or** B" / "either" | `OR` — at least one must be true |

The columns are `ENUM('Y','N')`, so they store the literal strings `'Y'`/`'N'` — compare against `'Y'` with quotes, not the boolean `TRUE`.

### 💡 Solution

```sql
SELECT product_id
FROM Products
WHERE low_fats = 'Y' AND recyclable = 'Y';
```

### 🧠 Explanation

- **`AND` requires both sides to be true.** Product 0 is low-fat but not recyclable; product 2 is recyclable but not low-fat — both fail. Only products 1 and 3 satisfy the pair.
- **Only `product_id` is selected.** The problem asks for the IDs, so don't return `SELECT *` — on LeetCode, extra columns fail the checker outright. Match the requested output columns exactly, every time.
- 💡 An `ENUM` is stored internally as an integer but compares as a string, so `low_fats = 'Y'` is the correct and readable form. Don't write `low_fats = 1` — that relies on the enum's internal ordinal and breaks the moment someone reorders the enum definition.

---

## Q2 — Find Customer Referee

🔗 **[leetcode.com/problems/find-customer-referee](https://leetcode.com/problems/find-customer-referee/description/)**

**Problem:** Find the names of customers who were **not** referred by the customer with `id = 2`.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Customer;
CREATE TABLE Customer (
    id         INT PRIMARY KEY,
    name       VARCHAR(25),
    referee_id INT
);

INSERT INTO Customer VALUES
(1,'Will',NULL),(2,'Jane',NULL),(3,'Alex',2),
(4,'Bill',NULL),(5,'Zack',1),(6,'Mark',2);
```

### ✅ Expected Output

| name |
|------|
| Will |
| Jane |
| Bill |
| Zack |

### 🎯 How to approach it

**This is the single most important `NULL` lesson in SQL, disguised as an easy problem.**

The obvious answer is `WHERE referee_id != 2`. Run it and you get only **Zack** — Will, Jane, and Bill vanish. Why?

Because `referee_id` is `NULL` for them, and **`NULL != 2` does not evaluate to `TRUE`. It evaluates to `NULL`** — which `WHERE` treats as "not true," so the row is dropped.

`NULL` means *"unknown."* Asking "is unknown different from 2?" honestly returns "I don't know" — not "yes." Every comparison operator (`=`, `!=`, `<`, `>`) behaves this way with `NULL`.

So whenever the question says **"not X"** on a **nullable** column, you need **two** conditions: the not-equal test *and* an explicit `IS NULL`.

### 💡 Solution

```sql
SELECT name
FROM Customer
WHERE referee_id != 2 OR referee_id IS NULL;
```

### 🧠 Explanation

- **`referee_id != 2`** catches Zack (referred by 1 — genuinely not 2).
- **`referee_id IS NULL`** catches Will, Jane, and Bill (never referred by anyone, so certainly not by customer 2). `IS NULL` is the *only* way to test for `NULL` — `= NULL` silently matches nothing.
- **`OR` between them** because a row qualifies if *either* is true. A customer can't be both, so there's no double-counting.
- 💡 An equivalent one-liner is `WHERE IFNULL(referee_id, 0) != 2` — it substitutes a non-2 placeholder for `NULL` so a single comparison covers both cases. It's shorter, but the `OR ... IS NULL` version states the intent plainly and doesn't depend on picking a sentinel value that can never legitimately appear. **Prefer the explicit version.**
- 💡 **Interview framing:** the reason this trips people up is that SQL uses *three-valued logic* — `TRUE`, `FALSE`, and `UNKNOWN`. `WHERE` keeps only `TRUE`. Say that out loud and you've answered the real question behind the problem.

---

## Q3 — Big Countries

🔗 **[leetcode.com/problems/big-countries](https://leetcode.com/problems/big-countries/description/)**

**Problem:** A country is **big** if it has an area of at least 3,000,000 km² **or** a population of at least 25,000,000. Return the name, population, and area of every big country.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS World;
CREATE TABLE World (
    name       VARCHAR(255) PRIMARY KEY,
    continent  VARCHAR(255),
    area       INT,
    population INT,
    gdp        BIGINT
);

INSERT INTO World VALUES
('Afghanistan','Asia',   652230, 25500100,  20343000000),
('Albania','Europe',       28748,  2831741,  12960000000),
('Algeria','Africa',    2381741, 37100000, 188681000000),
('Andorra','Europe',         468,    78115,   3712000000),
('Angola','Africa',     1246700, 20609294, 100990000000);
```

### ✅ Expected Output

| name | population | area |
|------|-----------|------|
| Afghanistan | 25500100 | 652230 |
| Algeria | 37100000 | 2381741 |

### 🎯 How to approach it

Q1 was `AND`; this is its mirror image. The definition says a country is big if it satisfies **either** condition — so `OR`. One qualifying attribute is enough.

Watch the boundary wording: **"at least"** means **inclusive** → `>=`, not `>`. Off-by-one on a boundary is the most common silent failure on these problems. Translate the phrasing deliberately:

| Wording | Operator |
|---|---|
| "at least" / "no less than" / "or more" | `>=` |
| "more than" / "greater than" / "exceeds" | `>` |
| "at most" / "no more than" | `<=` |
| "less than" / "under" | `<` |

Note the **column order in the output** — `name, population, area`, which is *not* the table's own order. Return columns in the order the problem asks for.

### 💡 Solution

```sql
SELECT name, population, area
FROM World
WHERE area >= 3000000 OR population >= 25000000;
```

### 🧠 Explanation

- **`OR` keeps a row when either side is true.** Both winners here qualify on **population**, not area: Afghanistan (25,500,100 ≥ 25,000,000, despite only 652,230 km²) and Algeria (37,100,000, with 2,381,741 km² — just *under* the area threshold). Angola fails both tests (1,246,700 km², 20,609,294 people), which is exactly why `OR` isn't the same as "everything passes."
- 💡 **Trace the near-misses, not just the winners.** Algeria's area is 2.38M against a 3M cutoff and Angola's population is 20.6M against a 25M cutoff — both deliberately close to the boundary. Test data is usually built this way on purpose: if you had written `>` instead of `>=`, or swapped `OR` for `AND`, these rows are the ones that would expose it.
- **No `NULL` handling needed here** — unlike Q2, `area` and `population` are always populated, so plain comparisons are safe. Always ask "can this column be `NULL`?" before deciding.
- 💡 On a real table this size you'd want an index, but note that `OR` across **two different columns** can't use a single composite index efficiently — MySQL may fall back to a full scan or an index merge. That's a legitimate follow-up answer if an interviewer asks "how would you optimise this?"

---

## Q4 — Article Views I

🔗 **[leetcode.com/problems/article-views-i](https://leetcode.com/problems/article-views-i/description/)**

**Problem:** Find all the authors who viewed at least one of their **own** articles. Return the result sorted by `id` ascending.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Views;
CREATE TABLE Views (
    article_id INT,
    author_id  INT,
    viewer_id  INT,
    view_date  DATE
);

INSERT INTO Views VALUES
(1,3,5,'2019-08-01'),(1,3,6,'2019-08-02'),(2,7,7,'2019-08-01'),
(2,7,6,'2019-08-02'),(4,7,1,'2019-07-22'),(3,4,4,'2019-07-21'),
(3,4,4,'2019-07-21');
```

### ✅ Expected Output

| id |
|----|
| 4 |
| 7 |

### 🎯 How to approach it

Three separate requirements hide in one sentence — handle them one at a time:

1. **"viewed their own article"** → the author *is* the viewer on that row → compare two columns **in the same row**: `WHERE author_id = viewer_id`. No join needed. Beginners reach for a self-join here; you don't need one, because both values already sit side by side.
2. **"authors"** (not view events) → author 4 appears twice in the data, so you must collapse duplicates → `DISTINCT`.
3. **"sorted by id"** → the output column must be renamed to `id`, and sorted → `AS id` plus `ORDER BY`.

That last point matters: LeetCode checks column **names**, so `AS id` isn't cosmetic — omit it and the submission fails even with correct rows.

### 💡 Solution

```sql
SELECT DISTINCT author_id AS id
FROM Views
WHERE author_id = viewer_id
ORDER BY author_id;
```

### 🧠 Explanation

- **`WHERE author_id = viewer_id` compares two columns of the same row** — a perfectly normal `WHERE` predicate. Row `(3,4,4,...)` matches (author 4 viewed article 3, which they wrote); row `(1,3,5,...)` doesn't (author 3, viewer 5).
- **`DISTINCT` is required, not optional.** The sample data contains `(3,4,4,'2019-07-21')` **twice**, so without `DISTINCT` author 4 appears twice in the output and the answer is wrong. Whenever the question asks for *who* rather than *how many times*, expect to deduplicate.
- **`ORDER BY author_id` works even though the column is aliased to `id`.** `ORDER BY` runs *after* `SELECT`, so both `ORDER BY id` and `ORDER BY author_id` are valid in MySQL — the alias and the underlying column both resolve.
- 💡 `GROUP BY author_id` would also deduplicate and even sort implicitly on some versions — but `DISTINCT` states the intent better when you're not aggregating anything. Reach for `GROUP BY` when you need an aggregate, `DISTINCT` when you just need unique values.

---

## Q5 — Invalid Tweets

🔗 **[leetcode.com/problems/invalid-tweets](https://leetcode.com/problems/invalid-tweets/description/)**

**Problem:** A tweet is **invalid** if the number of characters in its content is **strictly greater than 15**. Return the IDs of all invalid tweets.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Tweets;
CREATE TABLE Tweets (
    tweet_id INT PRIMARY KEY,
    content  VARCHAR(50)
);

INSERT INTO Tweets VALUES
(1,'Vote for Biden'),
(2,'Let us make America great again!');

-- 'Vote for Biden'                   → 14 characters → valid
-- 'Let us make America great again!'  → 32 characters → invalid
```

### ✅ Expected Output

| tweet_id |
|----------|
| 2 |

### 🎯 How to approach it

Two things to get right.

**First, the boundary.** "Strictly greater than 15" → `> 15`, **not** `>= 15`. A 15-character tweet is *valid*. Re-read boundary wording twice; it's where most wrong submissions come from.

**Second — and this is the real lesson — `LENGTH()` does not count characters.** It counts **bytes**. For plain ASCII text one character is one byte, so they agree and `LENGTH()` passes LeetCode's tests. But the moment the text contains a non-ASCII character, they diverge:

```sql
SELECT LENGTH('Vote for 🇺🇸 Biden')      AS length_bytes,   -- 23
       CHAR_LENGTH('Vote for 🇺🇸 Biden') AS char_length;    -- 17
```

The problem says **characters**, so `CHAR_LENGTH()` is the *semantically correct* function. `LENGTH()` happens to work here only because the test data is ASCII.

### 💡 Solution

```sql
-- Passes LeetCode (test data is ASCII)
SELECT tweet_id
FROM Tweets
WHERE LENGTH(content) > 15;

-- ✅ Correct by the problem's own definition ("number of characters")
SELECT tweet_id
FROM Tweets
WHERE CHAR_LENGTH(content) > 15;
```

### 🧠 Explanation

- **`CHAR_LENGTH(content) > 15`** counts characters regardless of encoding, which is what "number of characters" means. Use this one.
- **`LENGTH(content)`** counts bytes. Under a multi-byte charset like `utf8mb4`, an emoji is 4 bytes and many accented or non-Latin characters are 2–3 bytes, so `LENGTH()` **over-counts** and would flag a short tweet as invalid.
- 💡 **Say this in an interview and you stand out.** Both queries get accepted, but explaining *why* you chose `CHAR_LENGTH` shows you think about encoding — a real production concern the moment your app has users outside plain English. (`CHARACTER_LENGTH()` is a synonym for `CHAR_LENGTH()`; both are standard SQL.)
- 💡 Same trap applies to `SUBSTRING`/`LEFT`/`RIGHT`, which are character-based, versus `LENGTH`, which is byte-based — mixing them on multi-byte data produces off-by-N bugs that are painful to track down.

---

## Q6 — Replace Employee ID With The Unique Identifier

🔗 **[leetcode.com/problems/replace-employee-id-with-the-unique-identifier](https://leetcode.com/problems/replace-employee-id-with-the-unique-identifier/description/)**

**Problem:** Show each employee's `unique_id` and `name`. If an employee has **no** unique ID, show `null` for it.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Employees;
CREATE TABLE Employees (
    id   INT PRIMARY KEY,
    name VARCHAR(20)
);

INSERT INTO Employees VALUES
(1,'Alice'),(7,'Bob'),(11,'Meir'),(90,'Winston'),(3,'Jonathan');

DROP TABLE IF EXISTS EmployeeUNI;
CREATE TABLE EmployeeUNI (
    id        INT,
    unique_id INT,
    PRIMARY KEY (id, unique_id)
);

INSERT INTO EmployeeUNI VALUES (3,1),(11,2),(90,3);
```

### ✅ Expected Output

| unique_id | name |
|-----------|------|
| NULL | Alice |
| 1 | Jonathan |
| NULL | Bob |
| 2 | Meir |
| 3 | Winston |

### 🎯 How to approach it

**The phrase "If an employee does not have a unique ID, show null" is the entire problem.** It tells you two things:

1. Employees **without** a match must still appear → you cannot use `INNER JOIN`, which silently drops them (Alice and Bob would disappear).
2. The unmatched rows should show `NULL` → that's exactly what an outer join produces for the non-matching side, for free. No `IFNULL` needed.

So: `LEFT JOIN`, with **`Employees` on the left**, because employees are the rows you must keep all of.

**The general decision rule — memorise this:**

| The question says… | Use |
|---|---|
| "only where both exist" | `INNER JOIN` |
| "all X, even if no Y" / "show null when missing" | `LEFT JOIN` with X on the left |
| "X that have **no** Y at all" | `LEFT JOIN` + `WHERE y.key IS NULL` (see Q8) |

### 💡 Solution

```sql
SELECT e1.unique_id AS unique_id,
       e.name       AS name
FROM Employees e
LEFT JOIN EmployeeUNI e1
    ON e.id = e1.id;
```

### 🧠 Explanation

- **`LEFT JOIN` keeps every row of the left table** (`Employees`) whether or not a partner exists in `EmployeeUNI`. Jonathan, Meir, and Winston find matches; Alice (id 1) and Bob (id 7) don't, so `e1.unique_id` comes back as `NULL` — precisely the required output.
- **Swap it to `INNER JOIN` and you lose Alice and Bob** — the classic wrong answer here. Swap the table order to `EmployeeUNI LEFT JOIN Employees` and you also lose them, because then you're preserving the *wrong* side. **In a `LEFT JOIN`, the table you must not lose rows from goes first.**
- **The `NULL`s are generated by the join itself**, not by the data — `EmployeeUNI` simply has no row for ids 1 and 7. This is what an outer join *is*: it manufactures `NULL`-filled placeholder columns for the missing side.
- 💡 If the problem had instead asked for a default like `0` or `'N/A'` instead of `null`, you'd wrap it: `COALESCE(e1.unique_id, 0)`. The `LEFT JOIN` still does the heavy lifting — `COALESCE` just relabels the gap. (See [Problem Set 6 Q7](../interview-questions/problem-set-06.md#q7--ifnull-vs-coalesce) for `IFNULL` vs `COALESCE`.)

---

## Q7 — Product Sales Analysis I

🔗 **[leetcode.com/problems/product-sales-analysis-i](https://leetcode.com/problems/product-sales-analysis-i/description/)**

**Problem:** For each sale, report the `product_name`, `year`, and `price`.

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

INSERT INTO Sales VALUES
(1,100,2008,10,5000),(2,100,2009,12,5000),(7,200,2011,15,9000);

DROP TABLE IF EXISTS Product;
CREATE TABLE Product (
    product_id   INT PRIMARY KEY,
    product_name VARCHAR(30)
);

INSERT INTO Product VALUES (100,'Nokia'),(200,'Apple'),(300,'Samsung');
```

### ✅ Expected Output

| product_name | year | price |
|--------------|------|-------|
| Nokia | 2008 | 5000 |
| Nokia | 2009 | 5000 |
| Apple | 2011 | 9000 |

### 🎯 How to approach it

The needed columns live in **two tables**: `year` and `price` in `Sales`, `product_name` in `Product`. When your output pulls columns from more than one table, you join them — and the link is the shared **foreign key**, `product_id`.

Which join? The question says **"for each sale"** — the `Sales` table is the grain (one output row per sale row), and every sale has a valid `product_id`. So `INNER JOIN` is correct and sufficient.

**Notice what's *not* in the output:** Samsung (`product_id` 300) never appears, because no one ever sold it. That's the `INNER JOIN` doing its job — it keeps only rows with a match on both sides. If the problem had asked for "all products, including those never sold," you'd flip to `LEFT JOIN` with `Product` on the left.

**Also notice:** Nokia appears **twice**. That's correct, not a duplicate bug — Nokia was sold in two different years, and the grain is one row per sale. Don't reach for `DISTINCT` here.

### 💡 Solution

```sql
SELECT p.product_name AS product_name,
       s.year         AS year,
       s.price        AS price
FROM Sales s
JOIN Product p
    ON s.product_id = p.product_id;
```

### 🧠 Explanation

- **`JOIN` is shorthand for `INNER JOIN`** — identical behaviour. Writing `INNER JOIN` explicitly is the better habit because it makes the choice visible to whoever reads the query next.
- **The join key is `product_id`** — a foreign key in `Sales` pointing at the primary key of `Product`. This is the standard "enrich the fact table with a name from the dimension table" pattern, and it's roughly half of all real-world reporting SQL.
- **Table aliases (`s`, `p`) are doing real work.** Both tables have a `product_id` column, so an unqualified `product_id` in the `SELECT` would be an ambiguous-column error. Alias everything in a join, always.
- 💡 `year` doubles as a MySQL data type name, so it's the kind of column name that can collide with the parser. It works unquoted here, but qualifying it as `s.year` — as above — sidesteps the question entirely. If you ever do hit a syntax error on a column name, wrap it in backticks.

---

## Q8 — Customer Who Visited but Did Not Make Any Transactions

🔗 **[leetcode.com/problems/customer-who-visited-but-did-not-make-any-transactions](https://leetcode.com/problems/customer-who-visited-but-did-not-make-any-transactions/description/)**

**Problem:** For each customer, count how many times they visited without making a single transaction.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Visits;
CREATE TABLE Visits (
    visit_id    INT PRIMARY KEY,
    customer_id INT
);

INSERT INTO Visits VALUES (1,23),(2,9),(4,30),(5,54),(6,96),(7,54),(8,54);

DROP TABLE IF EXISTS Transactions;
CREATE TABLE Transactions (
    transaction_id INT PRIMARY KEY,
    visit_id       INT,
    amount         INT
);

INSERT INTO Transactions VALUES (2,5,310),(3,5,300),(9,5,200),(12,1,910),(13,2,970);
```

### ✅ Expected Output

| customer_id | count_no_trans |
|-------------|----------------|
| 30 | 1 |
| 96 | 1 |
| 54 | 2 |

### 🎯 How to approach it

This is the first genuinely multi-step problem in the set. Break it into two questions and solve them in order:

**Step 1 — which visits had no transaction at all?**
"No matching row exists in the other table" is the **anti-join** pattern, and it has a fixed three-part shape:

```
LEFT JOIN  the other table        →  keep every visit, matched or not
WHERE      other.key IS NULL      →  keep only the ones that found no partner
```

The `IS NULL` check is what converts a `LEFT JOIN` into "show me the *non*-matches." Test it on a column that can **never** legitimately be `NULL` in the right table — here `transaction_id`, its primary key. If you test a nullable column like `amount`, you'd also catch real transactions that happen to have a `NULL` amount, and the answer would be wrong.

**Step 2 — count those visits per customer.**
"For each customer" → `GROUP BY customer_id`, and `COUNT(*)` counts the surviving rows in each group.

The order matters: filter to the empty visits **first** (`WHERE`, which runs before grouping), *then* group and count. Customer 54 visited three times (visits 5, 7, 8) but visit 5 had transactions — so only visits 7 and 8 survive, giving a count of **2**.

### 💡 Solution

```sql
SELECT v.customer_id       AS customer_id,
       COUNT(*)            AS count_no_trans
FROM Visits v
LEFT JOIN Transactions t
    ON v.visit_id = t.visit_id
WHERE t.transaction_id IS NULL
GROUP BY v.customer_id;
```

### 🧠 Explanation

- **`LEFT JOIN` + `WHERE t.transaction_id IS NULL` is the anti-join.** After the join, visits 1, 2, and 5 carry real transaction data; visits 4, 6, 7, and 8 carry `NULL`s. The `WHERE` keeps only the `NULL` ones — the visits that produced nothing.
- **`COUNT(*)` is safe here** *because* of the `WHERE`. Every surviving row is a genuine empty visit, so counting rows counts empty visits. (Note the contrast with a plain `LEFT JOIN` and no filter, where `COUNT(*)` would wrongly return `1` for a customer with zero matches — the row still exists, it's just full of `NULL`s. **`COUNT(*)` counts rows; `COUNT(column)` skips `NULL`s.** That distinction is the difference between a right and wrong answer in a lot of join-plus-aggregate problems.)
- **`GROUP BY v.customer_id`** collapses the surviving visits into one row per customer. Customer 54 contributes two rows → `2`; customers 30 and 96 contribute one each → `1`. Qualifying it as `v.customer_id` rather than bare `customer_id` avoids ambiguity if `Transactions` ever gained a column of that name.
- 💡 **The `NOT EXISTS` alternative** reads more directly as "no transaction exists for this visit," and is `NULL`-safe by construction:

  ```sql
  SELECT v.customer_id, COUNT(*) AS count_no_trans
  FROM Visits v
  WHERE NOT EXISTS (SELECT 1 FROM Transactions t WHERE t.visit_id = v.visit_id)
  GROUP BY v.customer_id;
  ```

  Know both. `LEFT JOIN ... IS NULL` is the more common interview answer; `NOT EXISTS` is often the better plan and never bites you on `NULL`s.

---

## Q9 — Rising Temperature

🔗 **[leetcode.com/problems/rising-temperature](https://leetcode.com/problems/rising-temperature/description/)**

**Problem:** Find the IDs of all dates where the temperature was **higher than the previous day** — where "previous day" means the *immediately preceding calendar date*.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Weather;
CREATE TABLE Weather (
    id          INT PRIMARY KEY,
    recordDate  DATE,
    temperature INT
);

INSERT INTO Weather VALUES
(1,'2015-01-01',10),(2,'2015-01-02',25),(3,'2015-01-03',20),(4,'2015-01-04',30);
```

### ✅ Expected Output

| id |
|----|
| 2 |
| 4 |

### 🎯 How to approach it

**Two conditions, and skipping the second is the classic wrong answer.**

**Condition A — compare against the previous row.** Any time a problem says "compared to the previous / next record," reach for `LAG()` / `LEAD()`. `LAG(temperature) OVER (ORDER BY recordDate)` pulls the prior row's temperature onto the current row, so you can compare two rows with a simple `WHERE`.

**Condition B — the previous row must actually be *yesterday*.** `LAG` gives you the previous row **in the sorted order**, which is not necessarily the previous *calendar day* — real weather data has gaps. If readings jump from Jan 3 to Jan 7, `LAG` happily pairs them, and a naive solution reports a "rise" across a four-day hole. So you must *also* verify the dates are adjacent: `DATEDIFF(recordDate, previous_recordDate) = 1`.

**Then — why the subquery?** Because **you cannot filter on a window function in the same `SELECT` that defines it.** `WHERE` is evaluated *before* window functions, so `WHERE temperature > LAG(temperature) OVER (...)` is a syntax error. Wrap the window calculation in a derived table (or CTE) and filter on the outer level. This is the single most important structural rule for window-function problems.

### 💡 Solution

```sql
SELECT id
FROM (
    SELECT id, temperature, recordDate,
        LAG(temperature) OVER (ORDER BY recordDate) AS previous_temperature,
        LAG(recordDate)  OVER (ORDER BY recordDate) AS previous_recordDate
    FROM Weather
) t
WHERE DATEDIFF(recordDate, previous_recordDate) = 1
  AND temperature > previous_temperature;
```

### 🧠 Explanation

- **Two `LAG()` calls, one for each thing being compared** — the previous temperature *and* the previous date. Both share the same window (`ORDER BY recordDate`), so they read from the same prior row.
- **`DATEDIFF(recordDate, previous_recordDate) = 1`** enforces true adjacency. Argument order matters: `DATEDIFF(later, earlier)` is positive, so this reads "the current date is exactly 1 day after the previous one."
- **`temperature > previous_temperature`** is the actual rise test. Walking the data: id 2 (25 > 10, dates adjacent) ✅ · id 3 (20 < 25) ❌ · id 4 (30 > 20, adjacent) ✅.
- **Row 1 is excluded automatically.** It has no previous row, so both `LAG`s return `NULL`, and `NULL = 1` / `NULL > x` are never `TRUE` — no explicit `IS NOT NULL` guard needed. This is `NULL`'s three-valued logic (Q2) working *in your favour* for once.
- 💡 **The self-join alternative** is the pre-MySQL-8 answer and still worth knowing, because it encodes the adjacency requirement directly in the `ON` clause:

  ```sql
  SELECT w2.id
  FROM Weather w1
  JOIN Weather w2 ON DATEDIFF(w2.recordDate, w1.recordDate) = 1
  WHERE w2.temperature > w1.temperature;
  ```

  Same answer. The window version scales better (one pass instead of a join), but the self-join makes the "exactly one day apart" logic impossible to forget.

---

## Q10 — Average Time of Process per Machine

🔗 **[leetcode.com/problems/average-time-of-process-per-machine](https://leetcode.com/problems/average-time-of-process-per-machine/description/)**

**Problem:** Each process has a `'start'` row and an `'end'` row. For each machine, compute the average processing time across its processes, rounded to **3 decimal places**.

### 🗄️ Schema

```sql
DROP TABLE IF EXISTS Activity;
CREATE TABLE Activity (
    machine_id    INT,
    process_id    INT,
    activity_type ENUM('start','end'),
    timestamp     FLOAT
);

INSERT INTO Activity VALUES
(0,0,'start',0.712),(0,0,'end',1.520),(0,1,'start',3.140),(0,1,'end',4.120),
(1,0,'start',0.550),(1,0,'end',1.550),(1,1,'start',0.430),(1,1,'end',1.420),
(2,0,'start',4.100),(2,0,'end',4.512),(2,1,'start',2.500),(2,1,'end',5.000);
```

### ✅ Expected Output

| machine_id | processing_time |
|------------|-----------------|
| 0 | 0.894 |
| 1 | 0.995 |
| 2 | 1.456 |

### 🎯 How to approach it

**The key observation: the start and end times of one process live on two *different rows*.** To subtract them you need them side by side in a single row — and that means joining the table **to itself**.

Reach for a self-join whenever a problem requires **comparing or combining two rows of the same table**: start/end pairs, employee/manager, this-month/last-month, before/after.

Build the `ON` clause by listing what makes two rows "the same process, opposite ends":

| Requirement | Condition |
|---|---|
| Same machine | `a.machine_id = a1.machine_id` |
| Same process | `a.process_id = a1.process_id` |
| Left side is the start | `a.activity_type = 'start'` |
| Right side is the end | `a1.activity_type = 'end'` |

With those four, each process collapses to exactly one joined row holding both timestamps. Then it's ordinary aggregation: subtract, `AVG` per machine, `ROUND` to 3.

**Why not `GROUP BY process_id` too?** Because the question asks for the average **per machine**, across its processes — machine is the grain, process is what you're averaging over. Read the "per X" in the problem statement; that's your `GROUP BY`.

### 💡 Solution

```sql
SELECT a.machine_id,
       ROUND(AVG(a1.timestamp - a.timestamp), 3) AS processing_time
FROM Activity a
JOIN Activity a1
    ON a.machine_id    = a1.machine_id
   AND a.process_id    = a1.process_id
   AND a.activity_type = 'start'
   AND a1.activity_type = 'end'
GROUP BY a.machine_id;
```

### 🧠 Explanation

- **`Activity` is aliased twice — `a` for starts, `a1` for ends.** The aliases aren't cosmetic; they're what let MySQL treat one physical table as two logical row sources. Without them the query is unwritable.
- **All four conditions belong in the `ON` clause.** You *could* move `a.activity_type = 'start'` into a `WHERE`, and for an `INNER JOIN` the result is identical. Keeping them in `ON` states "this is what defines a matched pair" and — importantly — is the version that stays correct if you ever change it to a `LEFT JOIN`, where a filter in `WHERE` would silently undo the outer join.
- **`a1.timestamp - a.timestamp`** is the duration of one process. For machine 0: process 0 takes `1.520 - 0.712 = 0.808`, process 1 takes `4.120 - 3.140 = 0.980`. `AVG` of those two → `0.894`. ✅
- **`ROUND(..., 3)`** matches the required precision. LeetCode compares to 3 decimals, so an unrounded `0.8939999...` fails. Whenever a problem states a precision, round explicitly — never rely on display formatting.
- **`GROUP BY a.machine_id`, qualified.** Both aliases expose a `machine_id`, so bare `GROUP BY machine_id` is asking MySQL to disambiguate for you. It happens to resolve it against the `SELECT` list here and works — but qualify it and the query is unambiguous by construction.
- 💡 `timestamp` is a `FLOAT`, so the subtraction is floating-point and can carry tiny representation error — another reason the explicit `ROUND` matters. In production you'd store durations as `DECIMAL` or as integer milliseconds to avoid the issue entirely.

---

## 🧠 Set 1 — Patterns Worth Memorising

| Pattern | Trigger phrase in the problem | Tool |
|---|---|---|
| Multiple conditions, all required | "both", "and" | `WHERE a AND b` |
| Multiple conditions, any sufficient | "either", "or" | `WHERE a OR b` |
| Excluding on a nullable column | "not X" where X can be `NULL` | `!= x OR col IS NULL` |
| Unique list of values | "which authors/users/customers" | `DISTINCT` |
| Keep unmatched rows, show `NULL` | "if none, show null" | `LEFT JOIN` (must-keep table on the left) |
| Only matched rows | "for each sale/order" | `INNER JOIN` |
| Rows with **no** match at all | "did not", "never", "without any" | `LEFT JOIN` + `WHERE right.pk IS NULL` |
| Compare with the previous/next row | "than the previous day" | `LAG()` / `LEAD()` in a subquery |
| Pair two rows of the same table | start/end, employee/manager | self-join with aliases |
| A count/average "per X" | "per machine", "for each customer" | `GROUP BY x` |

### Four mistakes that cost the most submissions

1. **Boundary operators.** "At least" is `>=`; "more than" is `>`. Re-read it before submitting.
2. **`NULL` in a `!=` filter.** `NULL != 2` is `NULL`, not `TRUE`. Rows silently vanish. (Q2)
3. **Column names and order.** LeetCode checks headers — alias to exactly what's asked (`AS id`, `AS count_no_trans`) and return columns in the stated order.
4. **Filtering on a window function in the same `SELECT`.** Impossible — `WHERE` runs first. Wrap it in a subquery or CTE. (Q9)

---

<div align="center">

**[Continue to Set 2 — Joins & Aggregation ➡](set-02-joins-and-aggregation.md)**

[⬅ Back to Course Home](../README.md) · [Set 3 — Window Functions & First-Row Logic](set-03-window-functions-and-first-rows.md) · [Interview Problem Sets](../interview-questions/) · [LeetCode SQL 50 study plan ↗](https://leetcode.com/studyplan/top-sql-50/)

</div>
