# Part 10: Triggers in MySQL

A **Trigger** is a database object that **automatically executes** a set of SQL statements when a specific event — `INSERT`, `UPDATE`, or `DELETE` — happens on a table.

Where a **stored procedure** runs only when you explicitly `CALL` it, a trigger runs **on its own**, the moment the event fires. You attach it to a table once, and from then on the database silently does the work for you — logging a change, validating a value, or updating a related table — with no application code and no one having to remember to run anything.

Think of it as a **rule the database enforces automatically, every single time.**

---

## Why Are Triggers Needed?

Some rules have to run **every time** the data changes — no exceptions. If you rely on application code to log every salary change, one forgotten line or one script that writes directly to the table and the log is incomplete. A trigger closes that gap: it lives *inside* the database, so it fires no matter who or what makes the change.

They're also the natural home for **automatic side effects** — keeping an audit trail, stamping a timestamp, syncing a summary table — work that should happen alongside the change without the caller having to ask for it.

---

## When to Use Triggers?

| ✅ Use a trigger to… | Why it helps |
|---|---|
| **Maintain audit logs** | Record every change automatically, no matter the source. |
| **Track data changes** | Capture old vs new values as they happen. |
| **Enforce business rules** | Guarantee a rule runs on every insert/update/delete. |
| **Synchronize related tables** | Keep a summary or history table in step. |
| **Validate or modify data** | Clean or check a value *before* it's stored. |

---

## Trigger vs Stored Procedure

| Trigger | Stored Procedure |
|---|---|
| Executes **automatically** when an event occurs | Executes only when explicitly called with `CALL` |
| **Cannot** be called manually | Can be called whenever needed |
| Used for **automatic actions** (logging, syncing) | Used for **reusable business logic** |
| Tied to a specific table and event | Independent — called from anywhere |

---

## Types of Triggers

A trigger fires either **`BEFORE`** or **`AFTER`** an event, for each of the three data-changing events. That gives six combinations:

| Timing | `INSERT` | `UPDATE` | `DELETE` |
|---|---|---|---|
| **`BEFORE`** | `BEFORE INSERT` | `BEFORE UPDATE` | `BEFORE DELETE` |
| **`AFTER`** | `AFTER INSERT` | `AFTER UPDATE` | `AFTER DELETE` |

- **`BEFORE`** runs *before* the row change is written — use it to **validate or modify** the incoming data.
- **`AFTER`** runs *after* the change is committed — use it to **react** to the change, e.g. write to a log or history table.

---

## `OLD` and `NEW`

Inside a trigger, MySQL gives you two special row references:

| Keyword | Meaning | Available in |
|---|---|---|
| `OLD` | The row **before** the change | `UPDATE`, `DELETE` |
| `NEW` | The row **after** the change | `INSERT`, `UPDATE` |

- On **`INSERT`** there's no previous row → only `NEW` exists.
- On **`DELETE`** there's no resulting row → only `OLD` exists.
- On **`UPDATE`** both exist → `OLD.salary` is the value before, `NEW.salary` is the value after.

---

## Syntax

```sql
DELIMITER $$

CREATE TRIGGER trigger_name
{ BEFORE | AFTER } { INSERT | UPDATE | DELETE }
ON table_name
FOR EACH ROW
BEGIN
    -- SQL statements;
END $$

DELIMITER ;
```

> 💡 **Note — why `DELIMITER` and what is `FOR EACH ROW`?**
> As with stored procedures, the trigger body contains its own `;` after each inner statement, so we temporarily switch the terminator to `$$` (or `//`) with `DELIMITER $$`, then switch it back with `DELIMITER ;` afterwards. `FOR EACH ROW` means the trigger runs **once per affected row** — if an `UPDATE` touches 10 rows, the trigger fires 10 times, once for each row, with `OLD`/`NEW` pointing at that specific row.

---

## Practice Question

### Q. Whenever an employee's salary is updated, automatically store the employee ID, old salary, new salary, increment date, and increment percentage in a `salary_history` table.

First, a table to hold the history:

```sql
CREATE TABLE salary_history (
    id            INT AUTO_INCREMENT PRIMARY KEY,
    employee_id   INT,
    old_salary    DECIMAL(10, 2),
    new_salary    DECIMAL(10, 2),
    increment_date DATE,
    increment_pct DECIMAL(5, 2)
);
```

Now the trigger — it fires **after** an employee row is updated and logs the change:

```sql
DELIMITER //

CREATE TRIGGER updateSalaryAndInsert
AFTER UPDATE ON employees
FOR EACH ROW
BEGIN
    IF OLD.salary <> NEW.salary THEN
        INSERT INTO salary_history (
            employee_id,
            old_salary,
            new_salary,
            increment_date,
            increment_pct
        )
        VALUES (
            OLD.employee_id,
            OLD.salary,
            NEW.salary,
            CURDATE(),
            ROUND(((NEW.salary - OLD.salary) / OLD.salary) * 100, 2)
        );
    END IF;
END //

DELIMITER ;
```

Test it:

```sql
UPDATE employees
SET salary = salary + 5000
WHERE employee_id = 45;

SELECT * FROM salary_history;
```

> 💡 **Note:** The `IF OLD.salary <> NEW.salary THEN ... END IF;` guard is the key detail. `AFTER UPDATE` fires on **every** update to the `employees` table — even one that only changes a name or address. Without the guard, every such edit would insert a "salary change" row where the old and new salary are identical (a 0% increment). The `IF` makes sure we only log a row when the salary **actually changed**.

---

### What happens without the salary check?

If we drop the `IF` guard, the trigger logs a row on *every* update — including ones that don't touch the salary:

```sql
DELIMITER //

CREATE TRIGGER withoutSalaryCheck
AFTER UPDATE ON employees
FOR EACH ROW
BEGIN
    INSERT INTO salary_history (
        employee_id,
        old_salary,
        new_salary,
        increment_date,
        increment_pct
    )
    VALUES (
        OLD.employee_id,
        OLD.salary,
        NEW.salary,
        CURDATE(),
        ROUND(((NEW.salary - OLD.salary) / OLD.salary) * 100, 2)
    );
END //

DELIMITER ;
```

```sql
-- This update doesn't change the salary at all...
UPDATE employees
SET employee_name = 'Vivek Pandey'
WHERE employee_id = 45;

SELECT * FROM salary_history;   -- ...yet a row is still logged (old = new, 0% increment)
```

> 💡 **Note:** This is exactly the noise the `IF` guard prevents. A change to `employee_name` has nothing to do with salary, but this version still writes a `salary_history` row with `old_salary = new_salary` and a `0.00` increment. Always guard `AFTER UPDATE` triggers so they only act on the columns you actually care about.

---

## Managing Triggers

### Show all triggers

```sql
SHOW TRIGGERS;
```

### Show triggers for a specific table (current database)

```sql
SHOW TRIGGERS
WHERE `Table` = 'employees';
```

> 💡 **Note:** `Table` is wrapped in backticks because it's a reserved word in MySQL — the backticks tell MySQL to treat it as a column name, not a keyword.

### View the SQL used to create a trigger

```sql
SHOW CREATE TRIGGER trigger_name;
```

### Delete a trigger

```sql
DROP TRIGGER trigger_name;
```

> 💡 **Note:** MySQL has no `ALTER TRIGGER` for the body — to change a trigger's logic you **drop and recreate** it, just like a stored procedure. Use `DROP TRIGGER IF EXISTS trigger_name;` to avoid an error when it might not exist yet.

---

## When to Use a Trigger

1. **To maintain audit logs** — record who/what changed automatically, from any source.
2. **To track data changes** — capture `OLD` vs `NEW` values as they happen.
3. **To enforce business rules** — guarantee a rule runs on every insert/update/delete.
4. **To synchronize related tables** — keep history or summary tables in step.
5. **To validate or modify data** — clean or check a value with a `BEFORE` trigger before it's stored.

---

## Cheatsheet

```sql
-- Create a trigger
DELIMITER $$
CREATE TRIGGER trigger_name
{ BEFORE | AFTER } { INSERT | UPDATE | DELETE }
ON table_name
FOR EACH ROW
BEGIN
    -- use OLD.column / NEW.column here
    -- SQL statements;
END $$
DELIMITER ;

-- OLD  → row value BEFORE the change  (UPDATE, DELETE)
-- NEW  → row value AFTER  the change  (INSERT, UPDATE)

-- Inspect
SHOW TRIGGERS;
SHOW TRIGGERS WHERE `Table` = 'employees';
SHOW CREATE TRIGGER trigger_name;

-- Change logic (no ALTER for the body) → drop and recreate
DROP TRIGGER IF EXISTS trigger_name;

-- Delete
DROP TRIGGER trigger_name;
```
