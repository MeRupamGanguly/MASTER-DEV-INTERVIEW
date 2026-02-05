
# JOINS
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
