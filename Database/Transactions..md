# Transaction

## What is a Transaction in SQL?

A transaction is a sequence of one or more SQL statements that are executed as a single unit of work. Transactions ensure data consistency, especially when multiple changes need to either succeed or fail together.
    Properties of Transactions (ACID)

    Atomicity – All operations in the transaction succeed or none do.

    Consistency – The database is in a valid state before and after the transaction.

    Isolation – Transactions do not interfere with each other.

    Durability – Once committed, changes persist even if the system crashes.

```bash
BEGIN TRANSACTION;

-- SQL statements (INSERT/UPDATE/DELETE)

COMMIT; -- Save all changes

-- OR

ROLLBACK; -- Undo all changes if something goes wrong
```

We use transactions when:

    Multiple related changes must all happen (e.g., transferring money between accounts).

    You want to ensure consistency in case of failure.

    You're dealing with financial, inventory, or sensitive data updates.

    You need to roll back if any part of a multi-step process fails.

Always wrap multiple related write operations in a transaction.

Use ROLLBACK in TRY...CATCH blocks in application code.

Don't use transactions for long SELECT operations — it can block others.

Use isolation levels (like READ COMMITTED, SERIALIZABLE) to control concurrency.


When Not to Use Transactions

    For simple SELECTs.

    When auto-commit is sufficient (like in basic inserts).

    When working with read-only operations or reports.

Example Use Case: Insert + Update + Delete with Transaction
```bash
BEGIN TRANSACTION;

-- Insert new employee
INSERT INTO Employee (employee_id, name, department_id, join_date)
VALUES (200, 'Grace', 2, '2025-05-21');

-- Assign salary
INSERT INTO Salary (salary_id, employee_id, base_salary, bonus, effective_from)
VALUES (10, 200, 75000, 3000, '2025-05-21');

-- Assign to project
INSERT INTO Project (project_id, employee_id, role, project_name)
VALUES (4, 200, 'Developer', 'New Launch');

COMMIT; -- or ROLLBACK if any insert fails
```

Imagine we want to give a 10% raise to all employees in a department — but if even one update fails, we want to cancel everything.
```bash
BEGIN TRANSACTION;
UPDATE Salary
SET base_salary = base_salary * 1.10
WHERE employee_id IN (
    SELECT employee_id
    FROM Employee
    WHERE department_id = 1
);
-- Simulate failure (uncomment this line in real systems to test rollback)
-- RAISERROR('Simulated error', 16, 1);
COMMIT;
-- or ROLLBACK;
-- Savepoints (Optional Rollback Points)

BEGIN TRANSACTION;
UPDATE Salary SET bonus = bonus + 1000 WHERE employee_id = 101;
SAVEPOINT before_second_update;
UPDATE Salary SET bonus = bonus + 1000 WHERE employee_id = 102;
-- If this fails:
ROLLBACK TO before_second_update;
-- Continue or ROLLBACK everything
COMMIT;
```
