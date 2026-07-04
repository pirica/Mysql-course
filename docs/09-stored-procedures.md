# Part 9: Stored Procedures in MySQL

A **Stored Procedure** is a named block of one or more SQL statements that is saved *inside the database* and run on demand by calling its name.

Where a **view** is a saved `SELECT`, a stored procedure is a saved **program**. It can run several statements in sequence, accept input parameters, return values, hold variables, and use procedural logic like `IF` and `LOOP`. You write the logic once, store it in the database, and every application, script, or user runs it the same way: `CALL procedure_name(...)`.

Think of it as a **reusable function that lives in your database.**

---

## Why Are Stored Procedures Needed?

Without procedures, the same business logic gets rewritten in every app and every script that talks to the database. Change the rule and you have to hunt down every copy. A stored procedure gives that logic **one home inside the database** — update it once and every caller instantly uses the new version.

They also let you bundle **multiple steps into a single call**. Instead of an application sending an `UPDATE` and then a `SELECT` over the network as two round-trips, it makes one `CALL` and the database runs both together — less network chatter and consistent behavior for everyone.

---

## When to Use Stored Procedures?

| ✅ Use a stored procedure to… | Why it helps |
|---|---|
| **Reuse frequent SQL logic** | Write it once, `CALL` it everywhere. |
| **Run multiple statements in one call** | Bundle `UPDATE` + `SELECT` (etc.) into a single trip. |
| **Reduce duplicate SQL** | One definition instead of copies scattered across apps. |
| **Improve maintainability** | Fix the logic in one place. |
| **Accept parameters** | Pass values in (`IN`) for dynamic, reusable queries. |

---

## View vs Stored Procedure

| View | Stored Procedure |
|---|---|
| Stores a `SELECT` query | Stores one or more SQL statements |
| Returns a virtual table | Performs operations and returns results if needed |
| Mainly used for data **retrieval** | Used for **business logic and automation** |
| Cannot contain procedural logic (`IF`, `LOOP`) | Can contain `IF`, `LOOP`, variables, etc. |
| Invoked using `SELECT` | Invoked using `CALL` |

---

## Syntax

```sql
DELIMITER $$

CREATE PROCEDURE procedure_name()
BEGIN
    -- SQL statements;
END $$

DELIMITER ;
```

> 💡 **Note — what is `DELIMITER` and why do we need it?**
> Normally MySQL treats `;` as "end of statement." But a procedure *body* contains its own `;` after each inner statement — so MySQL would think the procedure ends at the first inner `;`. To avoid this, we temporarily change the statement terminator to something else (`$$` or `//`) with `DELIMITER $$`. Now the inner `;` are just part of the body, and the whole `CREATE PROCEDURE ... END $$` is treated as one statement. Afterwards we run `DELIMITER ;` to switch the terminator back to normal. The `$$` vs `//` choice is purely stylistic — both work.

---

## Practice Questions

### Q1. Create a Stored Procedure to display all Active employees.

```sql
DELIMITER $$

CREATE PROCEDURE activeEmployees()
BEGIN
    SELECT *
    FROM employees
    WHERE employment_status = 'Active';
END $$

DELIMITER ;
```

Run it:

```sql
CALL activeEmployees();
```

View its definition:

```sql
SHOW CREATE PROCEDURE activeEmployees;
```

> 💡 **Note:** This procedure takes no parameters — `activeEmployees()` — so it's essentially a saved query, similar to a view. The difference shows up next: a procedure can accept inputs and do far more than a `SELECT`.

---

### Q2. Create a Stored Procedure that accepts a department name and displays employees of that department.

```sql
DELIMITER $$

CREATE PROCEDURE displayDepartment(IN dept_name VARCHAR(100))
BEGIN
    SELECT *
    FROM employees
    WHERE department = dept_name;
END $$

DELIMITER ;
```

Run it with an argument:

```sql
CALL displayDepartment('IT');
```

> 💡 **Note:** `IN dept_name VARCHAR(100)` declares an **input parameter** — a value the caller passes in. Inside the body, `dept_name` behaves like a variable holding whatever you sent (`'IT'`). This is what makes a procedure *dynamic*: one definition serves every department instead of hard-coding one. Note the parameter type (`VARCHAR(100)`) should be compatible with the column you compare it against.

---

### Q3. Create a Stored Procedure to add ₹1,000 to the salary of all employees in a department, then display the updated rows.

```sql
DELIMITER //

CREATE PROCEDURE displayAndIncrementSalary(IN dept_name VARCHAR(100))
BEGIN
    UPDATE employees
    SET salary = salary + 1000
    WHERE department = dept_name;

    SELECT *
    FROM employees
    WHERE department = dept_name;
END //

DELIMITER ;
```

```sql
CALL displayAndIncrementSalary('IT');
```

> 💡 **Note:** This is the real strength of procedures — **multiple statements in one call.** First the `UPDATE` changes the data, then the `SELECT` returns the updated rows, all in a single `CALL`. An application would otherwise need two separate database trips. Because it writes data, run it deliberately: each call adds another ₹1,000, so calling it twice adds ₹2,000.

---

### Q4. Create a Stored Procedure to return the total number of employees.

```sql
DELIMITER //

CREATE PROCEDURE numberOfEmployees(OUT totalCount INT)
BEGIN
    SELECT COUNT(*) INTO totalCount
    FROM employees;
END //

DELIMITER ;
```

Call it and read the result:

```sql
CALL numberOfEmployees(@totalCount);
SELECT @totalCount;
```

> 💡 **Note:** Here we use an **`OUT` parameter** to *return a value*. `SELECT COUNT(*) INTO totalCount` stores the count into the `totalCount` variable instead of displaying it. We pass a **session variable** `@totalCount` into the call — MySQL fills it with the result — and then we read it with `SELECT @totalCount`. This is how a procedure hands a single value back to the caller.

---

## IN vs OUT Parameters

| Keyword | Direction | Purpose |
|---|---|---|
| `IN` | Caller → Procedure | Pass a value **into** the procedure (e.g. a department name). Default if not specified. |
| `OUT` | Procedure → Caller | Return a value **out** of the procedure (e.g. a computed count). |

> 💡 **Note:** There's also `INOUT`, which does both — the caller passes a value in, the procedure modifies it, and the updated value comes back out through the same parameter.

---

## Managing Stored Procedures

### See all stored procedures in the current database

```sql
SHOW PROCEDURE STATUS
WHERE Db = DATABASE();
```

### View a procedure's code

```sql
SHOW CREATE PROCEDURE procedure_name;
```

### Modify a stored procedure

MySQL does **not** let you change a procedure's body with `ALTER PROCEDURE`. To change the logic, drop it and recreate it:

```sql
DROP PROCEDURE procedure_name;

-- then CREATE PROCEDURE ... again with the new body
```

> 💡 **Note:** `ALTER PROCEDURE` exists but only edits *characteristics* (like comments or security settings) — never the actual SQL inside. So the standard workflow to update logic is **drop → recreate**. Use `DROP PROCEDURE IF EXISTS procedure_name;` to avoid an error when the procedure might not exist yet.

### Delete a stored procedure

```sql
DROP PROCEDURE procedure_name;
```

---

## When to Use a Stored Procedure

1. **To reuse frequent SQL logic** — save it once, `CALL` it from anywhere.
2. **To run multiple statements in one call** — bundle related steps into a single database trip.
3. **To reduce duplicate SQL** — one definition instead of copies across many apps.
4. **To accept parameters** — make queries dynamic with `IN`, and return results with `OUT`.
5. **To centralize business logic** — keep the rules in the database so every caller behaves consistently.

---

## Cheatsheet

```sql
-- Create a procedure (no parameters)
DELIMITER $$
CREATE PROCEDURE procedure_name()
BEGIN
    SELECT ...;
END $$
DELIMITER ;

-- With an IN parameter
DELIMITER $$
CREATE PROCEDURE procedure_name(IN param_name VARCHAR(100))
BEGIN
    SELECT ... WHERE column = param_name;
END $$
DELIMITER ;

-- With an OUT parameter
DELIMITER $$
CREATE PROCEDURE procedure_name(OUT result INT)
BEGIN
    SELECT COUNT(*) INTO result FROM table_name;
END $$
DELIMITER ;

-- Run a procedure
CALL procedure_name();
CALL procedure_name('IT');
CALL procedure_name(@out_var);
SELECT @out_var;

-- Inspect
SHOW PROCEDURE STATUS WHERE Db = DATABASE();
SHOW CREATE PROCEDURE procedure_name;

-- Change logic (no ALTER for the body) → drop and recreate
DROP PROCEDURE IF EXISTS procedure_name;

-- Delete
DROP PROCEDURE procedure_name;
```
