# SQL

```bash
rupx@dev:~$ docker run -d \
   --name postgres \
   -e POSTGRES_USER=myuser \
   -e POSTGRES_PASSWORD=mypassword \
   -e POSTGRES_DB=mydatabase \
   -p 5432:5432 \
   postgres:latest

rupx@dev:~$ docker exec -it postgres /bin/bash
root@a447994125b9:/# psql -U myuser -d mydatabase
mydatabase=# \c mydatabase
You are now connected to database "mydatabase" as user "myuser".
```


## JOINS
1. INNER JOIN : Returns only matching rows from both tables.

2. OUTER JOIN:

    LEFT JOIN : Returns all rows from the left table, and matching rows from the right. If no match, NULLs for the right side.

    RIGHT JOIN : Returns all rows from the right table, and matching rows from the left. NULLs for the left side if no match.

    FULL JOIN : Returns all rows from both tables. If there's no match on either side, fills with NULLs.

3. CROSS JOIN : Returns the Cartesian product – all combinations.

SELF JOIN : Joins a table to itself.

Users Table:

| UserID | UserName | ReferrerID |
|--------|----------|------------|
| 1      | Alice    | NULL       |
| 2      | Bob      | 1          |
| 3      | Charlie  | 1          |
| 4      | Diana    | 2          |

Orders Table:

| OrderID | UserID | ProductID |
|---------|--------|-----------|
| 101     | 1      | 1001      |
| 102     | 2      | 1002      |
| 103     | 2      | 1003      |
| 104     | 5      | 1001      | 

Products Table:

| ProductID | ProductName |
|-----------|-------------|
| 1001      | Laptop      |
| 1002      | Phone       |
| 1003      | Tablet      |
| 1004      | Monitor     |


### Q1: Get users who placed orders
```bash
mydatabase=# SELECT UserName, Orders.OrderID
FROM Orders
INNER JOIN Users ON Orders.UserID = Users.UserID;
```
| UserName | OrderID |
|----------|---------|
| Alice    | 101     |
| Bob      | 102     |
| Bob      | 103     |
(3 rows)

### Q2: Get all users and any orders they made
```bash
mydatabase=# SELECT Users.UserName, Orders.OrderID
FROM Users
LEFT JOIN Orders ON Users.UserID = Orders.UserID;
```
| UserName | OrderID |
|----------|---------|
| Alice    | 101     |
| Bob      | 102     |
| Bob      | 103     |
| Diana    |         |
| Charlie  |         |
(5 rows)


### Q3: Get all orders and the users who made them
```bash
mydatabase=# INSERT INTO Orders (OrderID, UserID, ProductID) VALUES
(105, NULL, 1003);
```
INSERT 0 1
| OrderID | UserID | ProductID |
|---------|--------|-----------|
| 101     | 1      | 1001      |
| 102     | 2      | 1002      |
| 103     | 2      | 1003      |
| 105     |        | 1003      |

```bash
mydatabase=# SELECT Users.UserName, Orders.OrderID
FROM Users
RIGHT JOIN Orders ON Users.UserID = Orders.UserID;
```
| UserName | OrderID |
|----------|---------|
| Alice    | 101     |
| Bob      | 102     |
| Bob      | 103     |
|          | 105     |

### Q4: All users and all orders, matching where possible
```bash
mydatabase=# SELECT Users.UserName, Orders.OrderID
FROM Users
FULL OUTER JOIN Orders ON Users.UserID = Orders.UserID;
```
| UserName | OrderID |
|----------|---------|
| Alice    | 101     |
| Bob      | 102     |
| Bob      | 103     |
|          | 105     |
| Diana    |         |
| Charlie  |         |

### Q5: All possible User-Product combinations
```bash
mydatabase=# SELECT Users.UserName, Products.ProductName
FROM Users
CROSS JOIN Products;
```
| UserName | ProductName |
|----------|-------------|
| Alice    | Laptop      |
| Bob      | Laptop      |
| Charlie  | Laptop      |
| Diana    | Laptop      |
| Alice    | Phone       |
| Bob      | Phone       |
| Charlie  | Phone       |
| Diana    | Phone       |
| Alice    | Tablet      |
| Bob      | Tablet      |
| Charlie  | Tablet      |
| Diana    | Tablet      |
| Alice    | Monitor     |
| Bob      | Monitor     |
| Charlie  | Monitor     |
| Diana    | Monitor     |


### Q6:  who referred whom
```bash
mydatabase=# SELECT u1.UserName AS User , u2.UserName AS Referrer
FROM Users u1
LEFT JOIN Users u2 ON u1.ReferrerID = u2.UserID;
```
| User     | Referrer |
|----------|----------|
| Alice    |          |
| Bob      | Alice    |
| Charlie  | Alice    |
| Diana    | Bob      |


### Q7: Get user's name, order ID, and product name
```bash
mydatabase=# SELECT u.UserName, o.OrderID, p.ProductName
FROM Users u
INNER JOIN Orders o ON u.UserID = o.UserID
INNER JOIN Products p ON o.ProductID = p.ProductID;
```
| UserName | OrderID | ProductName |
|----------|---------|-------------|
| Alice    | 101     | Laptop      |
| Bob      | 102     | Phone       |
| Bob      | 103     | Tablet      |


## ACID Properties in Database Transactions

- Concurrency control makes sure that when many people use the database at the same time, the data doesn’t get messed up.  It manages the execution of transactions to maintain the ACID properties—Atomicity, Consistency, Isolation, and Durability.

- Atomicity : Eiher every operation of transaction will complete or All failed. If one operation failed then every operation in transaction rolled back.

- Consistency : After and before any Transaction, Database should maintain the desired valid state.

- Isolation : Transactions can happen simultaneously but each transaction would processed independly. Two people ordering the last item then only one order will sucess and other will failed.

- Durability : Once a transaction is complete, even after any kind of crash, in database the transaction will show completed.  

## Relationship
a relationship refers to the way tables are connected to each other. 

- One-to-One: Each record in one table is connected to only one record in another table. A person can have only one passport, and each passport is assigned to only one person.

- One-to-Many: A single record in one table can be linked to multiple records in another table, but those multiple records connect back to only one record. A single customer can place multiple orders, but each order is placed by only one customer.

- Many-to-Many: Records in one table can be linked to multiple records in another table, and vice versa. Both sides can have many connections. Students can enroll in many courses, and each course can have many students.


## DELETE vs TRUNCATE vs DROP 
- The DELETE statement is used to remove one or more rows from a table based on a condition using the WHERE clause. It is a Data Manipulation Language (DML) command. Each row deletion is logged in the transaction log, making it slower for large datasets. Rollback is possible if used within a transaction. The table structure and its schema remain unchanged after deletion. `DELETE FROM Employees WHERE Department = 'HR';`

- The TRUNCATE command is used to remove all rows from a table without logging individual row deletions. It is a Data Definition Language (DDL) command. Truncation is faster than DELETE, especially on large tables. Rollback is not possible in most databases once the truncate operation is committed. The table structure, constraints, and schema remain intact. `TRUNCATE TABLE Employees;`

- The DROP statement is used to completely remove a table, view, or database. It is also a DDL command. It not only deletes all the rows in the table but also removes the table structure, constraints, indexes, and permissions. It cannot be rolled back in most systems. `DROP TABLE Employees;`


## Normalization
Normalization is the process of organizing data to reduce redundancy, Prevent update, insert, and delete anomalies. Updating a single piece of data in multiple places leads to inconsistency. When new data cannot be added to the database without including unwanted or irrelevant information. Deleting a record accidentally removes important data.

- 1NF – Eliminate repeating groups : Ensures that each column contains atomic values, and each row is unique. A table of students with multiple phone numbers in a single column is split into multiple rows where each phone number is stored separately.

- 2NF – Remove partial dependencies : If a table has both OrderID and ProductID as the primary key, and ProductName only depends on ProductID, then ProductName should be moved to a separate table. This way, every piece of data depends on the full primary key, not just one part.

- 3NF – Remove transitive dependencies : In a Student table, if the student's department name depends on the department ID, we move department info to a separate table to avoid redundancy.

- BCNF – Stronger version of 3NF

## Primary Key 
A Primary Key is a column (or a set of columns) that uniquely identifies each record in a table. It cannot be NULL, and it must be unique. 

# Foreign Key 
A Foreign Key is a field (or collection of fields) in one table that refers to the Primary Key in another table.
```bash
CREATE TABLE Orders (
  OrderID INT PRIMARY KEY,
  CustomerID INT,
  FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID)
);
```

## Triggers
A Trigger is a set of SQL statements that automatically executes in response to a specific event on a table or view.

We want to automatically update a LastModified timestamp field any time a row is updated
```bash
CREATE TRIGGER trg_UpdateTimestamp # creates a new trigger named trg_UpdateTimestamp.
AFTER UPDATE ON Employees
FOR EACH ROW # he trigger will run once per row that is updated, not just once per statement. If our UPDATE statement affects 5 rows, this trigger will run 5 times — once for each row.
BEGIN # trigger body
   UPDATE Employees 
   SET LastModified = NOW() # This updates the same row that was just updated, setting the LastModified column to the current timestamp using NOW()
   WHERE EmployeeID = OLD.EmployeeID; # OLD.EmployeeID refers to the value of EmployeeID before the update.
END;
```

## CAP :
The CAP Theorem states that a distributed database system can provide only two out of the following three guarantees:
- Consistency: Every read receives the most recent write.
- Availability: Every request (read/write) receives a response, even if some nodes are unavailable.
- Partition Tolerance: The system continues to operate despite network partitions.

# Indexing
Indexing is a technique used in databases to improve the speed of data retrieval operations on a table at the cost of additional storage and write performance. Indexes are created on columns that are frequently used in WHERE, JOIN, ORDER BY, or GROUP BY clauses.

- Without an index, the DBMS performs a full table scan. 
- With an index, it can quickly look up the location of matching rows, improving query performance significantly.

### Indexing Strategies :-

      - B-tree (default) Good for =, <, >, BETWEEN, ORDER BY.

      - GIN (Generalized Inverted Index) Stores multiple index entries per row. Used for: full-text search, arrays, JSONB, tsvector.

      - GiST (Generalized Search Tree) Good for nearest-neighbor, ranges.

      - Hash Optimized for = only.

- Single-column index – Fast, simple.
```bash
CREATE INDEX idx_email ON users(email);
SELECT * FROM users WHERE email = 'x@example.com';
```
- Multicolumn index – Useful when queries filter on multiple fields.
PostgreSQL uses the index efficiently only if the WHERE clause includes the leftmost column(s).
```bash
CREATE INDEX idx_name_dob ON employees(last_name, date_of_birth);
#-- Uses index (filters on both)
SELECT * FROM employees WHERE last_name = 'Smith' AND date_of_birth = '1990-01-01';
#-- Uses index (filters on first column only)
SELECT * FROM employees WHERE last_name = 'Smith';
#-- ❌ Will NOT use index (skips first column)
SELECT * FROM employees WHERE date_of_birth = '1990-01-01';
```
- Partial index – Index only a subset of rows.
```bash
CREATE INDEX idx_active_email ON users(email) WHERE is_active = true;
#-- Query that uses the index
SELECT * FROM users WHERE email = 'john@example.com' AND is_active = true;
#-- ❌ Query that won't use the index (missing the WHERE clause match)
SELECT * FROM users WHERE email = 'john@example.com';
```

## POINTS 
- Transaction : A transaction is a sequence of operations performed as a single logical unit of work. It must satisfy ACID properties.

- Deadlock : A deadlock occurs when two or more transactions are waiting for each other’s resources, creating a cycle where no transaction can proceed.

- Cascading : Cascading actions define what happens when a referenced row in the parent table is updated or deleted.

      ON DELETE CASCADE – Deletes related rows in child table
      ON UPDATE CASCADE – Updates foreign keys in child table
      SET NULL / SET DEFAULT – Sets foreign key to NULL or default

    ```bash
    FOREIGN KEY (DeptID) REFERENCES Departments(DeptID)
    ON DELETE CASCADE
    ON UPDATE CASCADE;
    ```
- A surrogate key is a system-generated unique identifier, often an auto-incremented number. It's used instead of natural keys to uniquely identify a record.
- A composite key is a primary key that consists of two or more columns. It is used when a single column cannot uniquely identify a record.
- A schema is the structure of a database, defined by a collection of tables, views, indexes, procedures, and other database objects. It acts as a blueprint for how the data is organized.
- Referential Integrity ensures that foreign key values in a table match primary key values in the related table, maintaining valid references between tables.
- The WHERE clause is used to filter individual rows before any grouping or aggregation happens. It operates on raw data in the table.
- The HAVING clause is used to filter aggregated results after the GROUP BY has grouped the data. It works on grouped records, often with aggregate functions like COUNT, SUM, AVG, etc.
    ```bash
    SELECT department, COUNT(*) AS emp_count
    FROM employees
    WHERE status = 'active'         -- Filters individual rows
    GROUP BY department
    HAVING COUNT(*) > 10;   
    ```


- Clustered index : 
A clustered index means the actual table data is stored in the same order as the index.
    ```bash
    CREATE INDEX idx_emp_id ON employees(employee_id);
    CLUSTER employees USING idx_emp_id;
    ```
  Future inserts won't follow this order.It's not maintained automatically. You’d need to manually recluster again later.

  All indexes in PostgreSQL are non-clustered by default. When we: `CREATE INDEX idx_email ON users(email);` You're building a separate structure that helps look up users by email quickly, but it doesn’t change how the rows are stored in the table.

- CTID : Every index in PostgreSQL uses CTID to point to the actual row.
It allows fast lookups, but changes if the row is updated/moved.
`"Alice" → (0,1)` 

      SELECT * FROM employees WHERE name = 'Alice';

      Looks up "Alice" in the B-tree index.

      Finds the CTID: (0,1)

      Uses that CTID to jump directly to the row in the table heap.

- View : A view is a virtual table based on the result of a SQL query. It does not store data physically. Views can be used to present data in a specific way without altering the underlying tables.
  ```bash
  CREATE VIEW ActiveEmployees AS
  SELECT Name, Department FROM Employees WHERE Status = 'Active';
  ```

- Window functions perform calculations across a set of table rows that are somehow related to the current row. Unlike aggregate functions, they don’t collapse rows.

```bash
<window_function>() OVER (
    PARTITION BY column
    ORDER BY column
    ROWS BETWEEN ... -- optional
)
```

## ROW_NUMBER() vs RANK() vs DENSE_RANK()
All three are window functions used to assign a number to each row based on a specified order.

ROW_NUMBER() gives a unique sequential number, even if values are tied.

RANK() gives the same number to tied rows, but skips the next rank(s) (e.g., 1, 2, 2, 4).

DENSE_RANK() also assigns the same number to tied rows, but does not skip ranks (e.g., 1, 2, 2, 3).

# GROUP BY and HAVING
GROUP BY is used to group rows that have the same values in specified columns for aggregate functions.

It works with functions like SUM(), COUNT(), AVG(), etc., to give a single result per group.

HAVING is like a WHERE clause, but it filters after grouping, based on the result of aggregate functions.

You must use GROUP BY before using HAVING in a query.

Example: 
```bash 
SELECT department, AVG(salary) FROM employees GROUP BY department HAVING AVG(salary) > 50000;
```

## WITH Clause (Common Table Expression - CTE)
The WITH clause lets we define a temporary result set that we can reference like a table within our query.

It improves readability, especially for complex queries with subqueries or repeated logic.

A CTE only exists during the execution of the main query and doesn't store data permanently.

You can even nest CTEs or chain multiple CTEs together using commas.
```bash
WITH HighEarners AS (
    SELECT * FROM employees WHERE salary > 50000
)
SELECT name FROM HighEarners;
```
## CASE Statement
The CASE expression works like an IF-ELSE logic block inside SQL queries.

It lets we create new columns or conditions based on custom logic.

You can use it inside SELECT, WHERE, ORDER BY, and even GROUP BY clauses.
```bash
SELECT name,
       CASE WHEN salary > 50000 THEN 'High' ELSE 'Low' END AS salary_group
FROM employees;
```

## Aggregating and Joining Data from Multiple Tables
```bash
SELECT
    d.department_name,
    COUNT(e.employee_id) AS employee_count,
    AVG(e.salary) AS average_salary
FROM employees e
JOIN departments d ON e.department_id = d.department_id
GROUP BY d.department_name;
```

## Subquery with EXISTS to Find Employees Who Have Managed Teams
```bash
SELECT
    e.employee_id,
    e.name
FROM employees e
WHERE EXISTS (
    SELECT 1
    FROM teams t
    WHERE t.manager_id = e.employee_id
);
```

## SQL Advanced Question: Join 3 Tables and Find 5th Highest Salary
You are given the following three tables:
### `employees`

| employee_id | name     | department_id | salary |
|-------------|----------|----------------|--------|
| 1           | Alice    | 10             | 90000  |
| 2           | Bob      | 20             | 85000  |
| 3           | Charlie  | 10             | 95000  |
| ...         | ...      | ...            | ...    |
### `departments`
| department_id | department_name |
|---------------|-----------------|
| 10            | HR              |
| 20            | IT              |
| ...           | ...             |
### `locations`
| department_id | location        |
|---------------|-----------------|
| 10            | New York        |
| 20            | San Francisco   |
| ...           | ...             |
---

## Write a SQL query to find the **employee with the 5th highest salary**, along with their name, salary, department name, and location :-

- We're creating a Common Table Expression (CTE) — a temporary result set to hold all employees, their details, and a rank based on their salary.

- We want to rank all employees by their salary, from highest to lowest. `DENSE_RANK()` is perfect because it gives the same rank to tied salaries (e.g., if two employees make the same amount, both are rank 1). Unlike `ROW_NUMBER()`, it won’t skip ranks for ties — so we can safely find the 5th actual salary tier.

- The employees table only has the department_id, not the department name or location. We JOIN with departments to get the readable department name. We JOIN with locations to get the physical location of the department.

- Select all the columns we want to display. Use `WHERE salary_rank = 5` to filter only the employee(s) with the 5th highest salary. It’s possible that more than one employee has the same salary — we want all of them if they tie for 5th place.

```bash
WITH RankedSalaries AS (
    SELECT 
        e.employee_id,
        e.name,
        e.salary,
        d.department_name,
        l.location,
        DENSE_RANK() OVER (ORDER BY e.salary DESC) AS salary_rank
    FROM employees e
    JOIN departments d ON e.department_id = d.department_id
    JOIN locations l ON e.department_id = l.department_id
)
SELECT 
    employee_id,
    name,
    salary,
    department_name,
    location
FROM RankedSalaries
WHERE salary_rank = 5;
```

- Ranking Employees by Salary, Flag Top 3
```bash
SELECT
    employee_id,
    department,
    salary,
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
    CASE 
        WHEN RANK() OVER (PARTITION BY department ORDER BY salary DESC) <= 3 THEN 'Top 3'
        ELSE 'Others'
    END AS salary_tier
FROM employees;
```
- This query retrieves the top N records (in this case, top 3 highest-paid employees) from each department. The ROW_NUMBER() window function helps in ranking the employees within each department, and then the WHERE clause filters for the top 3.
```bash
WITH RankedEmployees AS (
    SELECT
        employee_id,
        department,
        salary,
        ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num
    FROM employees
)
SELECT
    employee_id,
    department,
    salary
FROM RankedEmployees
WHERE row_num <= 3;
```


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

Employee – Contains employee details with department reference

Department – List of departments

Salary – Salary details per employee

Project – Project roles like TechLead, Architect, Developers (multi-role possible per employee)

```bash
CREATE TABLE Department (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(50)
);

INSERT INTO Department (department_id, department_name) VALUES
(1, 'Engineering'),
(2, 'Marketing'),
(3, 'HR'),
(4, 'Finance');


SELECT * FROM Department;


CREATE TABLE Employee (
    employee_id INT PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,
    join_date DATE,
    FOREIGN KEY (department_id) REFERENCES Department(department_id)
);


INSERT INTO Employee (employee_id, name, department_id, join_date) VALUES
(101, 'Alice', 1, '2020-01-10'),
(102, 'Bob', 1, '2021-06-15'),
(103, 'Charlie', 2, '2019-03-22'),
(104, 'David', 3, '2022-07-01'),
(105, 'Eve', 4, '2018-11-30'),
(106, 'Frank', 1, '2023-02-14');


SELECT * FROM Employee;

CREATE TABLE Salary (
    salary_id INT PRIMARY KEY,
    employee_id INT,
    base_salary DECIMAL(10, 2),
    bonus DECIMAL(10, 2),
    effective_from DATE,
    FOREIGN KEY (employee_id) REFERENCES Employee(employee_id)
);


INSERT INTO Salary (salary_id, employee_id, base_salary, bonus, effective_from) VALUES
(1, 101, 90000, 5000, '2023-01-01'),
(2, 102, 85000, 4000, '2023-01-01'),
(3, 103, 75000, 3000, '2023-01-01'),
(4, 104, 65000, 2000, '2023-01-01'),
(5, 105, 80000, 4500, '2023-01-01'),
(6, 106, 92000, 5500, '2023-01-01');


SELECT * FROM Salary;

CREATE TABLE Project (
    project_id INT,
    employee_id INT,
    role VARCHAR(50), -- e.g., TechLead, Architect, Developer
    project_name VARCHAR(100),
    PRIMARY KEY (project_id, employee_id),
    FOREIGN KEY (employee_id) REFERENCES Employee(employee_id)
);

INSERT INTO Project (project_id, employee_id, role, project_name) VALUES
(1, 101, 'TechLead', 'AI Chatbot'),
(1, 102, 'Developer', 'AI Chatbot'),
(1, 106, 'Developer', 'AI Chatbot'),
(2, 103, 'Architect', 'Marketing Analytics'),
(2, 104, 'Developer', 'Marketing Analytics'),
(3, 105, 'TechLead', 'Finance Automation'),
(3, 106, 'Architect', 'Finance Automation');

SELECT * FROM Project;


-- --------------  ALL SQL QUESTIONS  ------------------
-- 1. List all departments.

SELECT * FROM Department;

-- 2. Find all employees who joined after January 1, 2023.

SELECT * FROM Employee
WHERE join_date > '2023-01-01';

-- 3. Find the average salary across all employees.

SELECT AVG(base_salary + bonus) AS avg_salary
FROM Salary;

-- 4. Retrieve names and salaries of employees earning more than ₹80,000.

SELECT e.name, s.base_salary + s.bonus as total_salary
FROM Employee e
LEFT JOIN Salary s
ON e.employee_id = s.employee_id
WHERE s.base_salary+s.bonus > 80000

-- 5. Get the department names and the number of employees in each department.

SELECT d.department_name, count(e.employee_id) AS no_of_employee 
FROM Department d
LEFT JOIN Employee e
ON e.department_id = d.department_id
GROUP BY d.department_id
-- The purpose of using a LEFT JOIN instead of an INNER JOIN is to ensure that all departments are included in the result, even if they have no employees assigned.


-- 6. List employees who have not been assigned any projects.

INSERT INTO Employee (employee_id, name, department_id, join_date) VALUES
(108, 'Neice', 1, '2020-01-10');


SELECT * 
FROM Employee e
LEFT JOIN Project p
ON e.employee_id = p.employee_id


SELECT e.name
FROM Employee e
LEFT JOIN Project p
ON e.employee_id = p.employee_id
WHERE p.project_id IS NULL;


-- 7. Show the highest salary in each department.

SELECT e.employee_id, e.name, d.department_name, s.base_salary , s.bonus , s.base_salary + s.bonus AS total_salary
FROM Employee e
LEFT JOIN Salary s
ON e.employee_id = s.employee_id
LEFT JOIN Department d
ON e.department_id = d.department_id
ORDER BY s.base_salary DESC


SELECT d.department_name, MAX(s.base_salary + s.bonus) AS highest_salary
FROM Employee e
LEFT JOIN Salary s
ON e.employee_id = s.employee_id
LEFT JOIN Department d
ON e.department_id = d.department_id
GROUP BY d.department_id

-- 8. Retrieve the names of employees who work on the 'AI Chatbot' project.

SELECT e.name , p.project_name
FROM Employee e
LEFT JOIN Project p
ON e.employee_id = p.employee_id
WHERE p.project_name = 'AI Chatbot'

-- 9. Show the total salary (base + bonus) for each employee.

SELECT e.name, s.base_salary+s.bonus as total_salary
FROM Employee e
LEFT JOIN Salary s
ON e.employee_id = s.employee_id




-- 10. Find the second-highest salary in the company.

SELECT s.base_salary , s.bonus , s.base_salary + s.bonus AS total_salary
FROM Salary s

SELECT MAX(base_salary+bonus) AS second_highest_salary
FROM Salary

SELECT MAX(base_salary+bonus) AS second_highest_salary
FROM Salary
WHERE base_salary+bonus < (
    SELECT MAX(base_salary+bonus) FROM Salary
)

-- 11. Find the average salary per department.

SELECT d.department_name, ROUND (AVG(s.base_salary + s.bonus),2) AS average_salary
FROM Employee e
LEFT JOIN Salary s
ON e.employee_id = s.employee_id
LEFT JOIN Department d
ON e.department_id = d.department_id
GROUP BY d.department_id

-- 12. List employees who earn more than their department's average salary.

--  A correlated subquery is functionally like a nested FOR loop, because it re-executes the subquery once for each row of the outer query, using data from that outer row.

-- You want to compare each employee's salary with the average salary in their own department — not the company-wide average.

-- For each employee, check if their salary is greater than the average salary of their department.

SELECT e.employee_id,e.name,  s.base_salary+s.bonus
FROM Employee e
LEFT JOIN Salary s
ON e.employee_id = s.employee_id
WHERE s.base_salary+s.bonus > (
    SELECT AVG(s2.base_salary + s2.bonus) AS average_salary
    FROM Employee e2
    LEFT JOIN Salary s2
    ON e2.employee_id = s2.employee_id
    WHERE e2.department_id = e.department_id -- it's the condition that links the inner (subquery) to the outer query. Without this condition, the subquery would just return one value for the entire table, not a customized value per row.
)

-- 13. Show the number of employees in each department who have a bonus greater than ₹4,000.

SELECT COUNT(e.employee_id)
FROM Employee e
LEFT JOIN Salary s
ON e.employee_id = s.employee_id
WHERE s.bonus>4000


SELECT  e.name, s.bonus, d.department_name
FROM Employee e
LEFT JOIN Salary s
ON e.employee_id = s.employee_id
LEFT JOIN Department d
ON e.department_id = d.department_id


SELECT COUNT(e.employee_id) , d.department_name
FROM Employee e
LEFT JOIN Salary s
ON e.employee_id = s.employee_id
LEFT JOIN Department d
ON e.department_id = d.department_id
WHERE s.bonus>4000
GROUP BY d.department_id

-- 14. Find the employee with the highest total salary (base + bonus).

SELECT e.name, s.base_salary + s.bonus AS total_salary
FROM Employee e
JOIN Salary s ON e.employee_id = s.employee_id
ORDER BY total_salary DESC;

SELECT e.name, s.base_salary + s.bonus AS total_salary
FROM Employee e
JOIN Salary s ON e.employee_id = s.employee_id
ORDER BY total_salary DESC
LIMIT 1;

-- 15. List all projects along with the number of employees assigned to each.

SELECT * FROM Project

SELECT project_name, COUNT(employee_id)
FROM Project
GROUP BY project_name

-- 16. Find employees who are both 'TechLead' and 'Architect'.

SELECT e.name, p.role
FROM Employee e
LEFT JOIN Project p
ON e.employee_id = p.employee_id

INSERT INTO Employee (employee_id, name, department_id, join_date)
VALUES (110, 'Hannah', 1, '2025-05-21');

INSERT INTO Project (project_id, employee_id, role, project_name)
VALUES 
(4, 110, 'TechLead', 'Cloud Infra'),
(5, 110, 'Architect', 'Cloud Infra');

SELECT e.name, p.role
FROM Employee e
LEFT JOIN Project p
ON e.employee_id = p.employee_id
WHERE p.role IN ('TechLead', 'Architect')

-- Now we have only employees who is either TechLead or Architect

SELECT e.name
FROM Employee e
LEFT JOIN Project p
ON e.employee_id = p.employee_id
WHERE p.role IN ('TechLead', 'Architect')
GROUP BY e.NAME
HAVING COUNT(DISTINCT p.role)=2


-- 17. Find employees who are assigned to more than one project.

-- WHERE COUNT(p.project_id) > 1; Error: aggregate functions are not allowed in WHERE


SELECT e.name, COUNT(p.project_id) AS num_projects
FROM Employee e
LEFT JOIN Project p
ON e.employee_id = p.employee_id
GROUP BY e.name
HAVING COUNT(p.project_id)>1

-- 18. Show the total number of projects each employee is working on.

SELECT e.name, COUNT(p.project_id) AS num_projects
FROM Employee e
LEFT JOIN Project p ON e.employee_id = p.employee_id
GROUP BY e.name;

-- 19. Retrieve the department names and the total salary expense per department.

SELECT d.department_name, SUM(s.base_salary + s.bonus) AS total_salary_expense
FROM Department d
JOIN Employee e ON d.department_id = e.department_id
JOIN Salary s ON e.employee_id = s.employee_id
GROUP BY d.department_name;

--20. Rank employees within each department based on their total salary.

-- RANK() : Gaps in ranking are possible if there are ties. If two rows tie for a rank, the next rank(s) will be skipped.

SELECT e.name, d.department_name, s.base_salary + s.bonus AS total_salary,
       RANK() OVER (PARTITION BY d.department_name ORDER BY s.base_salary + s.bonus DESC) AS salary_rank
FROM Employee e
JOIN Department d ON e.department_id = d.department_id
JOIN Salary s ON e.employee_id = s.employee_id;

-- 21. 3rd highest salary in each department, you'll need to use a window function with DENSE_RANK() or ROW_NUMBER() to rank salaries within each department.

-- DENSE_RANK() : No gaps in ranking, even if there are ties. If two rows tie, the next rank continues sequentially.

SELECT department_name, name, total_salary
FROM (
    SELECT  d.department_name, e.name, s.base_salary + s.bonus AS total_salary,
    DENSE_RANK() OVER (PARTITION BY d.department_name  ORDER BY s.base_salary + s.bonus DESC) AS salary_rank
    FROM Employee e
    JOIN Department d ON e.department_id = d.department_id
    JOIN Salary s ON e.employee_id = s.employee_id
) 
WHERE salary_rank = 3;

-- 22. Find employees whose salary is above the average salary of their department but who are not assigned to any project.

-- SELECT 1 : It literally just selects the number 1. The important part is not what is selected, but whether any rows are returned.In a subquery used with EXISTS or NOT EXISTS, SQL only cares about the existence of rows — not their contents. So instead of doing SELECT * (which is wasteful), developers often use SELECT 1 for clarity and performance.

-- Here, SELECT 1 just checks: "Is there at least one row in Project where p.employee_id = e.employee_id?"

SELECT e.name, s.base_salary + s.bonus AS total_salary
FROM Employee e
JOIN Salary s ON e.employee_id = s.employee_id
WHERE s.base_salary + s.bonus > (
    SELECT AVG(s2.base_salary + s2.bonus)
    FROM Employee e2
    JOIN Salary s2 ON e2.employee_id = s2.employee_id
    WHERE e2.department_id = e.department_id
)
AND NOT EXISTS (
    SELECT 1 FROM Project p WHERE p.employee_id = e.employee_id
);


-- 23. Assign a dense rank to employees within each department based on their join date.

SELECT e.name, d.department_name, e.join_date,
DENSE_RANK() OVER (PARTITION BY d.department_name ORDER BY e.join_date ASC) AS join_rank
FROM Employee e
JOIN Department d ON e.department_id = d.department_id;


-- 24. Assign a percentile rank to employees within each department based on salary.

-- If you want to assign a percentile rank to employees within each department based on salary, you can use PERCENT_RANK(): Computes each employee’s percentile rank within their department. PERCENT_RANK() returns a value from 0 to 1, showing relative position by salary.

SELECT 
    e.name,
    d.department_name,
    s.base_salary + s.bonus AS total_salary,
    PERCENT_RANK() OVER (
        PARTITION BY d.department_name 
        ORDER BY s.base_salary + s.bonus
    ) AS salary_percentile
FROM Employee e
JOIN Salary s ON e.employee_id = s.employee_id
JOIN Department d ON e.department_id = d.department_id;

-- 25. Splits the ordered salary list into 4 buckets

-- Explanation: NTILE(4) splits the ordered salary list into 4 buckets (quartiles). Quartile 1 = lowest 25%, Quartile 4 = highest 25%. PARTITION BY d.department_name makes sure this is done within each department.

SELECT 
    e.name,
    d.department_name,
    s.base_salary + s.bonus AS total_salary,
    NTILE(2) OVER (
        PARTITION BY d.department_name 
        ORDER BY s.base_salary + s.bonus
    ) AS salary_quartile
FROM Employee e
JOIN Salary s ON e.employee_id = s.employee_id
JOIN Department d ON e.department_id = d.department_id;
```
