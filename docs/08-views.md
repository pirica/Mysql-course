# Part 8: Views in MySQL

A **View** is a virtual table created from the result of a SQL query. You give a `SELECT` statement a name, and from then on you can query that name exactly like a real table.

The key word is **virtual**. A view does **not** store any data of its own. What it stores is the **query definition**. Every time you run `SELECT * FROM view_name`, MySQL re-runs the underlying query behind the scenes and hands you the **latest data** from the base tables.

Think of a view as a **saved query with a name** — write the complex logic once, then reuse it everywhere with a simple `SELECT`.

---

## Why Are Views Needed?

Without views, any complex query has to be **rewritten (or copy-pasted) every time** you need it — in every report, every dashboard, every script. That's error-prone and hard to maintain: fix a bug in the logic and you have to fix it in ten places.

A view solves this by giving that logic **one home**. Write it once, reference it by name everywhere, and if the logic changes you update it in a single place.

Views also give you **security and abstraction**. You can expose a view that shows only a few safe columns (say, name and department) while keeping the underlying table — with salaries, personal data, etc. — hidden from the user.

---

## When to Use Views?

| ✅ Use a view to… | Why it helps |
|---|---|
| **Hide complex queries** | Turn a 30-line join into `SELECT * FROM view_name`. |
| **Restrict column access** | Expose only safe columns; hide sensitive ones like `salary`. |
| **Reuse frequent queries** | Define the logic once, use it in many places. |
| **Simplify reporting** | Analysts query a clean, ready-made view instead of raw tables. |
| **Improve readability** | Business-friendly names instead of tangled SQL. |

---

## Syntax

```sql
CREATE VIEW view_name AS
SELECT column1, column2, ...
FROM table_name
WHERE condition;
```

Once created, you query it just like a table:

```sql
SELECT *
FROM view_name;
```

> 💡 **Note:** The `SELECT` after `AS` is the **definition** — it is what MySQL stores. It does *not* run when you create the view; it runs each time you query the view. That's why a view always reflects the current data in its base table: it's re-executed on every read.

---

## Practice Questions

### Q1. Create a view to display all Active employees.

```sql
CREATE VIEW employee_active_status AS
SELECT *
FROM employees
WHERE employment_status = 'Active';
```

Now query it like any table:

```sql
SELECT *
FROM employee_active_status;
```

> 💡 **Note:** Anyone who needs active employees can now just `SELECT * FROM employee_active_status` — they never have to remember the `WHERE employment_status = 'Active'` filter. And if a new employee is set to Active tomorrow, they'll appear automatically, because the view re-runs its query on every read.

---

### Q2. Create a view to display employee name, department, and salary of employees earning more than ₹70,000.

```sql
CREATE VIEW employee_record AS
SELECT employee_name, department, salary
FROM employees
WHERE salary > 70000;
```

```sql
SELECT *
FROM employee_record;
```

> 💡 **Note:** Notice this view exposes **only three columns**. This is the classic use of views for **security**: a reporting user can be given access to `employee_record` without ever seeing other columns in the `employees` table. The view acts as a controlled window onto the data.

---

### Q3. Create a view to display each employee along with their manager's name.

```sql
CREATE VIEW employee_with_manager AS
SELECT e.employee_name AS employee_name,
       m.employee_name AS manager_name
FROM employees e
INNER JOIN employees m
    ON e.manager_id = m.employee_id;
```

```sql
SELECT *
FROM employee_with_manager;
```

> 💡 **Note:** This is a **self-join** (the `employees` table joined to itself) — genuinely tricky logic that's easy to get wrong. Wrapping it in a view means you solve it **once**. From now on, "employee and their manager" is just `SELECT * FROM employee_with_manager` — the join complexity is hidden behind a friendly name.

---

### Q4. Create a view to display the total number of employees in each department.

```sql
CREATE VIEW department_count AS
SELECT department, COUNT(*) AS total_count
FROM employees
GROUP BY department;
```

```sql
SELECT *
FROM department_count;
```

> 💡 **Note:** This view uses an aggregate (`COUNT(*)`) and `GROUP BY`. That makes it perfect for **reporting** — but keep it in mind for later: a view built on `GROUP BY` and aggregates is **read-only** (you can't `INSERT`/`UPDATE` through it). See [When Is a View Updatable?](#when-is-a-view-updatable) below.

---

### Q5. Create a view to display the highest-paid employee from each department.

```sql
CREATE VIEW highest_paid_employee AS
SELECT *
FROM (
    SELECT *,
           RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS ranking
    FROM employees
    WHERE department IS NOT NULL
) t
WHERE ranking = 1;
```

```sql
SELECT *
FROM highest_paid_employee;
```

> 💡 **Note:** This is the payoff. A window function *inside* a subquery, then filtered on `ranking = 1` — the kind of query that's hard to write and even harder to remember. Save it as a view once, and "top earner per department" becomes a single, trivial `SELECT`. This is exactly the scenario views were built for: **hide hard queries behind simple names.**

---

## Managing Views

### See all views in the database

```sql
SHOW FULL TABLES
WHERE Table_type = 'VIEW';
```

### See the SQL used to create a view

```sql
SHOW CREATE VIEW view_name;
```

> 💡 **Note:** Handy when you inherit a database and need to understand what a view actually does under the hood — it prints back the full stored `SELECT` definition.

### Modify a view

```sql
ALTER VIEW view_name AS
SELECT ...
FROM ...;
```

Example — redefine `department_count` to count by city instead:

```sql
ALTER VIEW department_count AS
SELECT city, COUNT(*) AS total_city_count
FROM employees
GROUP BY city;
```

> 💡 **Note:** `ALTER VIEW` replaces the stored definition. Any query that reads the view immediately starts using the new logic — this is the single-point-of-change benefit in action. (`CREATE OR REPLACE VIEW ...` does the same job in one step and also works if the view doesn't exist yet.)

### Delete a view

```sql
DROP VIEW view_name;
```

Deleting a view only removes the **saved query** — it never touches the data in the underlying tables.

---

## When Is a View Updatable?

Because a view is just a stored query, in some cases you can `INSERT`, `UPDATE`, or `DELETE` through it and the change flows down to the base table. But this only works when MySQL can **unambiguously map each view row back to exactly one base-table row**.

A view is generally **updatable** when it:

- Is based on a **single table**
- Does **not** use `GROUP BY`
- Does **not** use aggregate functions (`COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, …)
- Does **not** use `DISTINCT`
- Does **not** use `UNION`
- Does **not** use window functions (`RANK`, `ROW_NUMBER`, …)

> 💡 **Note:** The rule behind all of these is the same: **one view row must trace back to one real row.** The moment a view *summarizes* or *combines* rows — a `COUNT` collapses many rows into one, a `UNION` merges two sources, a window function ranks across a set — MySQL can no longer tell which base row your update should hit, so the view becomes **read-only**. From the examples above: `employee_active_status` (Q1) and `employee_record` (Q2) are updatable; `department_count` (Q4) and `highest_paid_employee` (Q5) are read-only.

---

## View vs Table — Quick Reference

| View | Table |
|---|---|
| Stores a **query**, not data | Stores actual **data** |
| Virtual — computed on each read | Physical — persisted on disk |
| Always reflects the latest base data | Holds its own rows until modified |
| Great for hiding complexity & restricting access | The source of truth the view reads from |
| Sometimes updatable (see rules above) | Always writable |

---

## When to Use a View

1. **To hide complex queries** — turn joins, subqueries, and window functions into a single simple `SELECT`.
2. **To restrict access** — expose only safe columns and hide sensitive ones like `salary`.
3. **To reuse frequent queries** — define logic once, reference it everywhere.
4. **To simplify reporting** — give analysts clean, ready-made virtual tables.
5. **To improve readability & maintainability** — fix the logic in one place instead of in every copy.

---

## Cheatsheet

```sql
-- Create a view
CREATE VIEW view_name AS
SELECT column1, column2
FROM table_name
WHERE condition;

-- Query a view (just like a table)
SELECT * FROM view_name;

-- See all views
SHOW FULL TABLES WHERE Table_type = 'VIEW';

-- See a view's definition
SHOW CREATE VIEW view_name;

-- Modify a view
ALTER VIEW view_name AS
SELECT ...;
-- (or) CREATE OR REPLACE VIEW view_name AS SELECT ...;

-- Delete a view (data in base tables is untouched)
DROP VIEW view_name;
```
