# SQL

SQL (Structured Query Language) is the standard language for managing and manipulating relational databases. It allows you to query, insert, update, delete, and manage data, making it a cornerstone of database management. Here’s a comprehensive guide to understanding SQL, with examples and key concepts.

Author

Benedict Thekkel

### 1. **What is SQL?**

- **Language for Databases**: SQL is used to communicate with relational database management systems (RDBMS) such as MySQL, PostgreSQL, SQLite, Oracle, and Microsoft SQL Server.
- **Declarative Language**: SQL focuses on *what* data to retrieve or modify rather than *how* to do it, which the database engine handles.

### 2. **Key SQL Commands**

SQL commands are divided into categories based on their purpose:

#### Data Definition Language (DDL)

- **`CREATE`**: Creates new tables, databases, indexes, and other objects.
- **`ALTER`**: Modifies an existing database object, such as a table.
- **`DROP`**: Deletes objects from the database.
- **`TRUNCATE`**: Removes all records from a table (without deleting the table itself).

#### Data Manipulation Language (DML)

- **`SELECT`**: Retrieves data from the database.
- **`INSERT`**: Adds new records to a table.
- **`UPDATE`**: Modifies existing records in a table.
- **`DELETE`**: Removes records from a table.

#### Data Control Language (DCL)

- **`GRANT`**: Grants permissions to users.
- **`REVOKE`**: Revokes permissions from users.

#### Transaction Control Language (TCL)

- **`COMMIT`**: Saves changes to the database.
- **`ROLLBACK`**: Undoes changes since the last commit.
- **`SAVEPOINT`**: Sets a savepoint within a transaction to which you can later roll back.

### 3. **Creating and Modifying Tables (DDL)**

#### Create a Table

``` sql
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(50),
    position VARCHAR(50),
    salary DECIMAL(10, 2),
    hire_date DATE
);
```

#### Alter a Table

Add a new column:

``` sql
ALTER TABLE employees
ADD email VARCHAR(100);
```

Remove a column:

``` sql
ALTER TABLE employees
DROP COLUMN email;
```

#### Drop a Table

``` sql
DROP TABLE employees;
```

### 4. **Basic Data Operations (DML)**

#### Inserting Data

``` sql
INSERT INTO employees (id, name, position, salary, hire_date)
VALUES (1, 'John Doe', 'Software Engineer', 75000, '2022-01-15');
```

#### Selecting Data

``` sql
SELECT name, position, salary
FROM employees
WHERE salary > 50000
ORDER BY hire_date DESC;
```

#### Updating Data

``` sql
UPDATE employees
SET salary = salary * 1.1  -- Increase salary by 10%
WHERE position = 'Software Engineer';
```

#### Deleting Data

``` sql
DELETE FROM employees
WHERE hire_date < '2020-01-01';
```

### 5. **Querying Data with `SELECT`**

#### Filtering with `WHERE`

``` sql
SELECT * FROM employees
WHERE position = 'Manager' AND salary > 70000;
```

#### Sorting with `ORDER BY`

``` sql
SELECT name, salary FROM employees
ORDER BY salary DESC, name ASC;
```

#### Limiting Results with `LIMIT` (MySQL) / `FETCH FIRST` (SQL Server)

``` sql
SELECT * FROM employees
ORDER BY salary DESC
LIMIT 5;
```

#### Aggregating Data with Functions

- **`COUNT`**: Counts rows.
- **`SUM`**: Sums values.
- **`AVG`**: Calculates average.
- **`MIN`** and **`MAX`**: Find minimum and maximum values.

Example:

``` sql
SELECT AVG(salary) AS avg_salary, MAX(salary) AS max_salary
FROM employees;
```

#### Grouping Results with `GROUP BY` and `HAVING`

``` sql
SELECT position, AVG(salary) AS avg_salary
FROM employees
GROUP BY position
HAVING avg_salary > 50000;
```

### 6. **Joins**

Joins are used to retrieve data from multiple tables based on related columns.

- **INNER JOIN**: Returns rows where there is a match in both tables.
- **LEFT JOIN**: Returns all rows from the left table and matching rows from the right table.
- **RIGHT JOIN**: Returns all rows from the right table and matching rows from the left.
- **FULL OUTER JOIN**: Returns all rows from both tables, with NULLs where no match exists.

Example of INNER JOIN:

``` sql
SELECT employees.name, departments.department_name
FROM employees
INNER JOIN departments ON employees.department_id = departments.id;
```

### 7. **Subqueries**

A subquery is a query nested within another query.

#### Example: Subquery in `SELECT`

``` sql
SELECT name,
       (SELECT AVG(salary) FROM employees) AS avg_salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

### 8. **Indexes**

Indexes improve the speed of data retrieval but may slow down insertions, updates, and deletions.

``` sql
CREATE INDEX idx_position ON employees (position);
```

### 9. **Constraints**

Constraints enforce data rules on tables, such as: - **PRIMARY KEY**: Uniquely identifies each record. - **FOREIGN KEY**: Ensures referential integrity between tables. - **UNIQUE**: Ensures all values in a column are unique. - **CHECK**: Ensures values meet a specific condition. - **NOT NULL**: Ensures a column cannot contain NULL values.

Example:

``` sql
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_id INT,
    amount DECIMAL(10, 2) CHECK (amount > 0),
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

### 10. **Transactions and Atomicity**

Transactions allow multiple SQL commands to be executed as a single unit.

#### Example of Transaction with COMMIT and ROLLBACK

``` sql
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;  -- Save changes
-- Or ROLLBACK; to undo changes
```

### 11. **Views**

Views are virtual tables that provide a way to save complex queries for easy reuse.

``` sql
CREATE VIEW high_earners AS
SELECT name, salary
FROM employees
WHERE salary > 80000;
```

### 12. **Stored Procedures and Functions**

Stored procedures and functions allow you to store reusable SQL code on the server side.

#### Stored Procedure Example

``` sql
CREATE PROCEDURE increase_salary (IN emp_id INT, IN increment DECIMAL(10, 2))
BEGIN
    UPDATE employees SET salary = salary + increment WHERE id = emp_id;
END;
```

#### User-Defined Function Example

``` sql
CREATE FUNCTION get_full_name(first_name VARCHAR(50), last_name VARCHAR(50))
RETURNS VARCHAR(100)
BEGIN
    RETURN CONCAT(first_name, ' ', last_name);
END;
```

### 13. **Common SQL Functions**

- **String Functions**: `UPPER`, `LOWER`, `CONCAT`, `SUBSTRING`.
- **Numeric Functions**: `ROUND`, `FLOOR`, `CEILING`, `ABS`.
- **Date Functions**: `NOW`, `CURDATE`, `YEAR`, `MONTH`, `DATEDIFF`.
- **Conversion Functions**: `CAST`, `CONVERT`.

### 14. **Normalization**

Normalization organizes data to reduce redundancy and dependency. Key normal forms include: - **1NF (First Normal Form)**: Ensures each column contains atomic values. - **2NF (Second Normal Form)**: Builds on 1NF by removing partial dependencies. - **3NF (Third Normal Form)**: Removes transitive dependencies.

### 15. **Advanced Concepts**

- **Recursive Queries**: Using `WITH RECURSIVE` for hierarchical data.
- **Window Functions**: Perform calculations across a set of table rows related to the current row (e.g., `ROW_NUMBER()`, `RANK()`).
- **Partitioning**: Divide large tables into smaller, manageable pieces.
- **Stored JSON Data**: Many RDBMS support storing and querying JSON data in tables.

#### Example of a Recursive Query

``` sql
WITH RECURSIVE org_chart AS (
    SELECT employee_id, manager_id, name
    FROM employees
    WHERE manager_id IS NULL
    UNION ALL
    SELECT e.employee_id, e.manager_id, e.name
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.employee_id
)
SELECT * FROM org_chart;
```

### 16. **Best Practices**

- Use indexes thoughtfully, as too many can slow down data modifications.
- Always back up your database before making major changes.
- Avoid using `SELECT *`; instead, specify columns for efficiency.
- Normalize tables to reduce redundancy and improve data integrity.

## Django ORM to SQL

> In Django, the **ORM (Object-Relational Mapping)** translates SQL operations into Python code, allowing you to interact with a database using Django’s high-level `QuerySet` API. Here’s how common SQL concepts map to Django ORM concepts:

------------------------------------------------------------------------

### 1. **Models as Tables**

In Django, each database **table** is represented by a **model class**. Each field in the model corresponds to a column in the database table, and each instance of the model represents a row.

#### SQL Table Definition

``` sql
CREATE TABLE employee (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department VARCHAR(50),
    salary DECIMAL
);
```

#### Django Model

``` python
from django.db import models

class Employee(models.Model):
    name = models.CharField(max_length=100)
    department = models.CharField(max_length=50)
    salary = models.DecimalField(max_digits=10, decimal_places=2)
```

### 2. **SQL Data Types to Django Field Types**

| SQL Type  | Django Field Type                           |
|-----------|---------------------------------------------|
| VARCHAR   | `models.CharField`                          |
| INTEGER   | `models.IntegerField`                       |
| DECIMAL   | `models.DecimalField`                       |
| DATE      | `models.DateField`                          |
| TIMESTAMP | `models.DateTimeField`                      |
| BOOLEAN   | `models.BooleanField`                       |
| SERIAL    | `models.AutoField` (Primary Key by default) |

### 3. **Basic CRUD Operations**

#### CREATE: Inserting Data

In SQL:

``` sql
INSERT INTO employee (name, department, salary) VALUES ('John Doe', 'Sales', 50000);
```

In Django ORM:

``` python
Employee.objects.create(name="John Doe", department="Sales", salary=50000)
```

#### READ: Querying Data

- **SELECT All**: Retrieve all rows from a table.

  SQL: `sql SELECT * FROM employee;`

  Django ORM: `python Employee.objects.all()`

- **SELECT with WHERE Condition**:

  SQL: `sql SELECT * FROM employee WHERE department = 'Sales';`

  Django ORM: `python Employee.objects.filter(department="Sales")`

- **SELECT with Filtering and Ordering**:

  SQL: `sql SELECT * FROM employee WHERE department = 'Sales' ORDER BY salary DESC;`

  Django ORM: `python Employee.objects.filter(department="Sales").order_by("-salary")`

#### UPDATE: Modifying Data

In SQL:

``` sql
UPDATE employee SET salary = 55000 WHERE name = 'John Doe';
```

In Django ORM:

``` python
Employee.objects.filter(name="John Doe").update(salary=55000)
```

#### DELETE: Removing Data

In SQL:

``` sql
DELETE FROM employee WHERE name = 'John Doe';
```

In Django ORM:

``` python
Employee.objects.filter(name="John Doe").delete()
```

### 4. **Relationships**

Django provides fields like `ForeignKey`, `OneToOneField`, and `ManyToManyField` to define relationships between models. These fields translate to `JOIN` operations in SQL.

#### One-to-Many Relationship

SQL:

``` sql
CREATE TABLE department (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50)
);

CREATE TABLE employee (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    department_id INTEGER REFERENCES department(id)
);
```

Django Models:

``` python
class Department(models.Model):
    name = models.CharField(max_length=50)

class Employee(models.Model):
    name = models.CharField(max_length=100)
    department = models.ForeignKey(Department, on_delete=models.CASCADE)
```

**Querying with a Foreign Key**:

``` python
# Get all employees in a specific department
sales_dept = Department.objects.get(name="Sales")
employees = Employee.objects.filter(department=sales_dept)
```

#### Many-to-Many Relationship

SQL:

``` sql
CREATE TABLE project (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE employee_project (
    employee_id INTEGER REFERENCES employee(id),
    project_id INTEGER REFERENCES project(id),
    PRIMARY KEY (employee_id, project_id)
);
```

Django Models:

``` python
class Project(models.Model):
    name = models.CharField(max_length=100)

class Employee(models.Model):
    name = models.CharField(max_length=100)
    projects = models.ManyToManyField(Project)
```

**Querying Many-to-Many Relationships**:

``` python
# Get all projects for a specific employee
employee = Employee.objects.get(name="John Doe")
projects = employee.projects.all()
```

### 5. **Aggregation and Grouping**

The Django ORM provides methods like `annotate()` and `aggregate()` for aggregation and grouping operations, similar to SQL’s `GROUP BY` and aggregate functions like `COUNT()`, `SUM()`, `AVG()`.

#### Example: COUNT and GROUP BY

SQL:

``` sql
SELECT department, COUNT(*) FROM employee GROUP BY department;
```

Django ORM:

``` python
from django.db.models import Count

Employee.objects.values('department').annotate(count=Count('id'))
```

#### Example: SUM, AVG, MIN, MAX

SQL:

``` sql
SELECT AVG(salary) FROM employee WHERE department = 'Sales';
```

Django ORM:

``` python
from django.db.models import Avg

Employee.objects.filter(department="Sales").aggregate(Avg('salary'))
```

### 6. **Joins with Django ORM**

The Django ORM automatically performs joins when you query across related models.

#### Inner Join (default with ForeignKey)

SQL:

``` sql
SELECT employee.name, department.name
FROM employee
JOIN department ON employee.department_id = department.id;
```

Django ORM:

``` python
Employee.objects.select_related('department').values('name', 'department__name')
```

#### Left Outer Join (using `prefetch_related` for Many-to-Many or reverse ForeignKey relationships)

``` python
Employee.objects.prefetch_related('projects')
```

### 7. **Subqueries**

You can use Django’s `Subquery` and `OuterRef` classes to perform subqueries within queries.

#### Example: Subquery with Aggregate

SQL:

``` sql
SELECT name, salary
FROM employee
WHERE salary > (SELECT AVG(salary) FROM employee);
```

Django ORM:

``` python
from django.db.models import Subquery, OuterRef, Avg

avg_salary = Employee.objects.aggregate(Avg('salary'))['salary__avg']
high_earners = Employee.objects.filter(salary__gt=avg_salary)
```

### 8. **Transactions in Django ORM**

Django’s `transaction.atomic` function allows you to manage transactions to ensure that operations are completed as a unit, similar to SQL’s `BEGIN TRANSACTION`, `COMMIT`, and `ROLLBACK`.

``` python
from django.db import transaction

try:
    with transaction.atomic():
        Employee.objects.create(name="John Doe", salary=60000)
        Employee.objects.create(name="Jane Doe", salary=70000)
except:
    print("Transaction failed")
```

### 9. **Raw SQL in Django**

In cases where complex queries are required, Django allows you to execute raw SQL directly.

``` python
from django.db import connection

with connection.cursor() as cursor:
    cursor.execute("SELECT * FROM employee WHERE salary > %s", [50000])
    rows = cursor.fetchall()
```

### Summary

| SQL Concept | Django ORM Equivalent |
|----|----|
| Table | Model |
| Row | Model instance |
| Column | Model field |
| SELECT | `.all()`, `.filter()`, `.get()` |
| INSERT | `.create()` |
| UPDATE | `.update()` |
| DELETE | `.delete()` |
| JOIN | `.select_related()`, `.prefetch_related()` |
| GROUP BY | `.values()`, `.annotate()` |
| Aggregate Functions (SUM, AVG, etc.) | `.aggregate()`, `.annotate()` |
| Transactions | `transaction.atomic()` |
| Raw SQL | `connection.cursor().execute()` |

The Django ORM provides a high-level, Pythonic interface to interact with your database, which is both powerful and flexible. Understanding how SQL concepts translate into Django ORM methods allows you to leverage the full potential of Django’s database management capabilities.

Back to top
