# Part 12: Recursive CTE in MySQL

A **Recursive CTE** is a CTE that **references itself repeatedly** until a termination condition is met.

"Recursive" simply means: **repeat the same task again and again until a condition is met.**

It's the tool you reach for whenever data is **self-referencing** — rows that point to other rows in the *same* table. Classic examples: an employee whose `manager_id` points to another employee, a category whose `parent_id` points to another category, or a folder nested inside another folder. A normal `JOIN` can only walk **one level** of that chain; a recursive CTE walks **as many levels as exist**.

---

## Syntax

```sql
WITH RECURSIVE cte_name AS
(
    -- Anchor Query

    UNION ALL

    -- Recursive Query
)
SELECT *
FROM cte_name;
```

| Part | Role |
| ---- | ---- |
| **Anchor Query** | The **starting point** — runs once and seeds the result. |
| **Recursive Query** | Runs **repeatedly**, each pass feeding on the *previous* pass's rows. |
| **`UNION ALL`** | **Combines** the previous rows with the newly produced rows. |
| **Termination Condition** | **Stops** the recursion when the recursive query returns **no new rows**. |

> ⚠️ **The `RECURSIVE` keyword is required.** Unlike a normal CTE, you must write `WITH RECURSIVE`. And the recursive part must reference the CTE **by name** — that self-reference is what makes it loop.

---

## How the loop actually runs

Reading `WITH RECURSIVE` top-to-bottom hides the loop. Here's what MySQL does under the hood:

1. Run the **anchor query** once → these are the first rows.
2. Run the **recursive query**, but only against the rows produced by the **previous step** (not the whole accumulated set).
3. `UNION ALL` those new rows into the result.
4. Repeat step 2–3, each time using only the *latest* batch of rows.
5. **Stop** when a pass produces **zero rows** — the `WHERE` in the recursive query is what eventually makes that happen.

> ⚠️ **A missing or wrong termination condition = infinite loop.** MySQL protects you with `cte_max_recursion_depth` (default **1000**) — hit that ceiling and the query errors out instead of hanging forever. You can raise it with `SET SESSION cte_max_recursion_depth = 5000;` when a legitimately deep hierarchy needs it.

---

## Dataset for This Part

The examples below use a self-referencing `organization` table — each employee's `manager_id` points to another employee's `employee_id` in the **same table** (the CEO has `NULL`, since they report to no one).

```sql
CREATE TABLE organization (
    employee_id   INT PRIMARY KEY,
    employee_name VARCHAR(1000),
    designation   VARCHAR(50),
    manager_id    INT
);

INSERT INTO organization VALUES
(1, 'Raj Sharma',    'CEO',                 NULL),
(2, 'Sneha Kapoor',  'CTO',                 1),
(3, 'Amit Patel',    'HR Head',             1),
(4, 'Aarav Sharma',  'Engineering Manager', 2),
(5, 'Priya Mehta',   'Senior Developer',    4),
(6, 'Rohit Jain',    'Developer',           5),
(7, 'Anjali Verma',  'HR Executive',        3);
```

The reporting chain this encodes:

```
Raj Sharma (CEO)
├── Sneha Kapoor (CTO)
│   └── Aarav Sharma (Engineering Manager)
│       └── Priya Mehta (Senior Developer)
│           └── Rohit Jain (Developer)
└── Amit Patel (HR Head)
    └── Anjali Verma (HR Executive)
```

---

## Practice Questions

### Q1. Generate the numbers 1 to 10 using a Recursive CTE.

```sql
WITH RECURSIVE numbers AS
(
    SELECT 1 AS num              -- Anchor: start at 1

    UNION ALL

    SELECT num + 1               -- Recursive: add 1 to the previous number
    FROM numbers
    WHERE num < 10               -- Termination: stop once we reach 10
)
SELECT *
FROM numbers;
```

**🧠 Explanation**

- The **anchor** produces a single row: `num = 1`.
- The **recursive query** takes the previous value and adds 1 → `2`, then `3`, then `4`… each pass working off the row before it.
- The **`WHERE num < 10`** is the termination condition. When `num` becomes `10`, the recursive query's `WHERE` filters it out, the next pass produces **no rows**, and the loop stops.
- Final output: `1, 2, 3, 4, 5, 6, 7, 8, 9, 10`.

> 💡 **Teaching Tip:** This "number series" pattern is the *hello world* of recursion — but it's genuinely useful. Generate a run of dates, fill gaps in a report, or produce a row per day/hour for a calendar table, all without a physical numbers table. Change `WHERE num < 10` to `WHERE num < 100` and you instantly have 1–100.

---

### Q2. Display the complete employee hierarchy.

Walk the reporting chain from the top (`manager_id IS NULL`) all the way down, tracking each person's **level** in the org and the full **path** from the CEO to them.

```sql
WITH RECURSIVE organization_hierarchy AS
(
    -- Anchor: the top of the tree (the person with no manager)
    SELECT
        employee_id,
        employee_name,
        designation,
        1 AS level,
        employee_name AS hierarchy_path
    FROM organization
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: attach each employee to the manager already in the result
    SELECT
        o.employee_id,
        o.employee_name,
        o.designation,
        oh.level + 1,
        CONCAT(oh.hierarchy_path, ' -> ', o.employee_name)
    FROM organization o
    JOIN organization_hierarchy oh
        ON o.manager_id = oh.employee_id
)
SELECT *
FROM organization_hierarchy;
```

**Result:**

| employee_id | employee_name | designation | level | hierarchy_path |
|---|---|---|---|---|
| 1 | Raj Sharma | CEO | 1 | Raj Sharma |
| 2 | Sneha Kapoor | CTO | 2 | Raj Sharma -> Sneha Kapoor |
| 3 | Amit Patel | HR Head | 2 | Raj Sharma -> Amit Patel |
| 4 | Aarav Sharma | Engineering Manager | 3 | Raj Sharma -> Sneha Kapoor -> Aarav Sharma |
| 7 | Anjali Verma | HR Executive | 3 | Raj Sharma -> Amit Patel -> Anjali Verma |
| 5 | Priya Mehta | Senior Developer | 4 | Raj Sharma -> Sneha Kapoor -> Aarav Sharma -> Priya Mehta |
| 6 | Rohit Jain | Developer | 5 | Raj Sharma -> Sneha Kapoor -> Aarav Sharma -> Priya Mehta -> Rohit Jain |

**🧠 Explanation**

- The **anchor** grabs the root — `WHERE manager_id IS NULL` finds Raj Sharma (the CEO). He's assigned `level = 1` and his `hierarchy_path` starts as just his own name.
- The **recursive query** joins the `organization` table back onto the CTE: `o.manager_id = oh.employee_id` means *"find every employee whose manager is already in our result."* On the first pass that's everyone reporting to the CEO (Sneha, Amit); on the next pass, everyone reporting to *them*; and so on down the tree.
- **`oh.level + 1`** increments the depth counter each level down — 1 (CEO), 2 (their reports), 3, 4, 5.
- **`CONCAT(oh.hierarchy_path, ' -> ', o.employee_name)`** builds a breadcrumb trail by appending each new name to the parent's path — so you can read the full chain of command in one column.
- The loop **terminates naturally**: eventually it reaches employees who have no reports (Rohit, Anjali), the join finds nothing new, and recursion stops.

> 💡 **Teaching Tip:** The direction of the join is the whole trick. `ON o.manager_id = oh.employee_id` walks **top-down** (manager → reports). Flip it to `ON o.employee_id = oh.manager_id` and start the anchor from a *leaf* employee, and you'd walk **bottom-up** instead (report → all their managers). Same table, same recursion — just reverse the link you follow.

---

## When to Use a Recursive CTE

1. **Hierarchies** — org charts, employee → manager chains, category → subcategory trees.
2. **Graph / tree traversal** — folder structures, bill-of-materials (part → sub-parts), threaded comments.
3. **Sequence generation** — number series, date/calendar ranges, filling gaps in reports.
4. **Path building** — producing a "breadcrumb" from a root node to any descendant.

---

## Key Points

- ✔ Use **`WITH RECURSIVE`** — the keyword is mandatory.
- ✔ Every recursive CTE needs an **anchor** (seed) and a **recursive** part joined by **`UNION ALL`**.
- ✔ The recursive query must **reference the CTE by name** — that self-reference is the loop.
- ✔ Always include a **termination condition** (a `WHERE` that eventually returns no rows).
- ✔ MySQL caps recursion at **`cte_max_recursion_depth`** (default 1000) to stop runaway loops — raise it only when a genuinely deep tree needs it.
- ✔ The **join direction** decides whether you walk top-down or bottom-up.

---

## Cheatsheet

```sql
-- Basic recursive CTE
WITH RECURSIVE cte_name AS
(
    SELECT ...              -- Anchor (runs once)
    UNION ALL
    SELECT ...              -- Recursive (runs until no new rows)
    FROM cte_name
    WHERE termination_condition
)
SELECT * FROM cte_name;

-- Number series 1..N
WITH RECURSIVE numbers AS (
    SELECT 1 AS num
    UNION ALL
    SELECT num + 1 FROM numbers WHERE num < 10
)
SELECT * FROM numbers;

-- Top-down hierarchy walk (manager -> reports)
WITH RECURSIVE tree AS (
    SELECT id, name, 1 AS level, name AS path
    FROM t WHERE parent_id IS NULL          -- anchor at the root
    UNION ALL
    SELECT c.id, c.name, p.level + 1, CONCAT(p.path, ' -> ', c.name)
    FROM t c JOIN tree p ON c.parent_id = p.id
)
SELECT * FROM tree;

-- Raise the recursion ceiling for a very deep tree
SET SESSION cte_max_recursion_depth = 5000;
```
