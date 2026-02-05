
# Advance Queries

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
