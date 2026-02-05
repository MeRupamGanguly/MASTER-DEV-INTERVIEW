
# Queries

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
