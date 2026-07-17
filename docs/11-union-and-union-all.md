# Part 11: UNION & UNION ALL in MySQL

`UNION` and `UNION ALL` **stack the rows of two or more `SELECT` queries on top of each other** into a single result set. Where a `JOIN` combines tables **side by side** (adding columns), a `UNION` combines them **head to tail** (adding rows).

Reach for them whenever the data you need lives in **separate places but has the same shape** — e.g. employee IDs that appear in a bonuses table *and* a leaves table, or names coming from both an `employees` and a `clients` table.

---

## What is `UNION`?

`UNION` combines the results of two or more `SELECT` queries and **removes duplicate rows** — the final output contains only distinct rows.

## What is `UNION ALL`?

`UNION ALL` combines the results of two or more `SELECT` queries and **keeps all rows, including duplicates** — nothing is filtered out.

---

## `UNION` vs `UNION ALL`

| `UNION` | `UNION ALL` |
| ------- | ----------- |
| Removes duplicate rows. | Keeps duplicate rows. |
| Slightly **slower** — must sort/hash to detect duplicates. | **Faster** — no duplicate check. |
| Returns unique records. | Returns all records. |

### See the difference

Given two queries that each return these `id` values:

```
Query A → 9, 10, 11
Query B → 10, 11, 12
```

| `UNION` (distinct) | `UNION ALL` (everything) |
|--------------------|--------------------------|
| 9, 10, 11, 12 | 9, 10, 11, 10, 11, 12 |

`UNION` collapsed the repeated `10` and `11` into one each; `UNION ALL` kept both copies.

> 💡 **Rule of thumb:** default to `UNION ALL`. Only use `UNION` when you actually need duplicates removed — otherwise you pay for a de-duplication step (a hidden sort) you didn't need.

---

## The Rules Every `UNION` Must Follow

For MySQL to stack two result sets, they must line up:

1. **Same number of columns** in every `SELECT`.
2. **Compatible data types**, column by column, in the same order (MySQL matches by *position*, not by name).
3. **Column names come from the first `SELECT`** — aliases in later queries are ignored for the final headers.
4. **`ORDER BY` goes once, at the very end** — it sorts the combined result, not the individual queries.

---

## Practice Questions

### Q1. Display all employee IDs who have received either a bonus or taken a leave.

An employee ID can appear in **both** the `bonuses` and `leaves_data` tables. We want the set of IDs that show up in *either* one.

**Solution — `UNION` (each ID once):**

```sql
SELECT employee_id AS id
FROM bonuses

UNION

SELECT employee_id
FROM leaves_data;
```

**Compare — `UNION ALL` (every appearance):**

```sql
SELECT employee_id AS id
FROM bonuses

UNION ALL

SELECT employee_id
FROM leaves_data;
```

**🧠 Explanation**

- Both tables share the same shape for this query — a single `employee_id` column — so they stack cleanly.
- **`UNION`** returns the **distinct** list: an employee who both got a bonus *and* took a leave (or had two bonuses) appears **once**. This is the correct answer to "who received *either* a bonus or a leave" — it's a set of employees.
- **`UNION ALL`** returns **one row per source row**: the same employee can appear multiple times (once per bonus, once per leave). Its row count equals `rows in bonuses + rows in leaves_data`, always ≥ the `UNION` count.
- The final column is named **`id`** because that alias came from the **first** `SELECT`; the alias-less second query just contributes values.

> 💡 To also *tag* where each ID came from, add a literal column: `SELECT employee_id, 'Bonus' AS source FROM bonuses UNION ALL SELECT employee_id, 'Leave' FROM leaves_data`. (Note: once you add the `source` column, `UNION` would no longer collapse an ID that appears in both, because the rows are no longer identical.)

---

### Q2. Display the names of all employees and all clients in a single result set.

Employees and clients live in **different tables** with **different column names** (`employee_name` vs `client_name`), but conceptually they're all "people/entities with a name." We want one combined list, tagged by type.

**Solution:**

```sql
SELECT employee_name AS name, 'Employee' AS type
FROM employees

UNION ALL

SELECT client_name AS name, 'Client' AS type
FROM clients;
```

**🧠 Explanation**

- **Different column names are fine** — MySQL matches columns by **position**, not name. `employee_name` and `client_name` are both the *first* column, so they merge into a single `name` column.
- **The literal `type` column** (`'Employee'` / `'Client'`) is a classic `UNION` trick: it labels each row with its origin, so you can tell employees from clients in the merged output.
- **`UNION ALL` is the right choice here.** These are distinct real-world entities — even if an employee and a client happened to share a name, you'd want to keep both rows. `UNION` would wrongly merge two different people who share a name. (It would also do needless de-duplication work.)
- The output headers are **`name`** and **`type`**, taken from the first `SELECT`. The total row count is `employees + clients`.

---

## Key Points

- ✔ Every `SELECT` must have the **same number of columns**.
- ✔ Columns must have **compatible data types**, matched **by position**.
- ✔ The **final column names come from the first `SELECT`**.
- ✔ Write **`ORDER BY` only once**, at the end (it sorts the whole combined result).
- ✔ Use **`UNION`** when you need **unique** rows.
- ✔ Use **`UNION ALL`** to keep **all** rows (including duplicates) — and as the faster default.

---

## Cheatsheet

```sql
-- UNION: distinct rows (removes duplicates)
SELECT col1, col2 FROM table_a
UNION
SELECT col1, col2 FROM table_b;

-- UNION ALL: all rows (keeps duplicates) — faster
SELECT col1, col2 FROM table_a
UNION ALL
SELECT col1, col2 FROM table_b;

-- Tag the source of each row
SELECT id, 'A' AS source FROM table_a
UNION ALL
SELECT id, 'B'          FROM table_b;

-- ORDER BY applies to the COMBINED result — write it once, at the end
SELECT col1 FROM table_a
UNION ALL
SELECT col1 FROM table_b
ORDER BY col1;
```
