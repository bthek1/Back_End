# Query Filters

Django **query filters** allow you to **retrieve, filter, and manipulate** database records using Django’s ORM (`QuerySet API`). Filters **convert Pythonic queries into SQL queries** automatically.

Author

Benedict Thekkel

## **1. Basic Query Filtering**

Filters are applied using **`.filter()`**, **`.exclude()`**, and **`.get()`** on Django QuerySets.

| Method | Description |
|----|----|
| `filter(**conditions)` | Returns records **matching** the conditions. |
| `exclude(**conditions)` | Returns records **not matching** the conditions. |
| `get(**conditions)` | Returns **a single object** matching the condition (raises `MultipleObjectsReturned` or `DoesNotExist`). |

### **Example Usage**

``` python
from myapp.models import Employee

# Get all employees with age 30
employees = Employee.objects.filter(age=30)

# Get all employees EXCEPT those with age 30
employees_exclude = Employee.objects.exclude(age=30)

# Get a single employee with a unique field (e.g., email)
employee = Employee.objects.get(email="john@example.com")
```

------------------------------------------------------------------------

## **2. Field Lookups (Operators)**

Django provides **lookup expressions** to perform SQL-like queries.

### **2.1 Exact Match (`exact`, `iexact`)**

| Lookup | Example | SQL Equivalent |
|----|----|----|
| `exact` | `.filter(name__exact="John")` | `WHERE name = 'John'` |
| `iexact` (case-insensitive) | `.filter(name__iexact="john")` | `WHERE LOWER(name) = 'john'` |

``` python
Employee.objects.filter(name__exact="John")
Employee.objects.filter(name__iexact="john")  # Case-insensitive
```

------------------------------------------------------------------------

### **2.2 Partial Match (`contains`, `icontains`, `startswith`, `endswith`)**

| Lookup | Example | SQL Equivalent |
|----|----|----|
| `contains` | `.filter(name__contains="Jo")` | `WHERE name LIKE '%Jo%'` |
| `icontains` (case-insensitive) | `.filter(name__icontains="jo")` | `WHERE LOWER(name) LIKE '%jo%'` |
| `startswith` | `.filter(name__startswith="Jo")` | `WHERE name LIKE 'Jo%'` |
| `endswith` | `.filter(name__endswith="son")` | `WHERE name LIKE '%son'` |

``` python
Employee.objects.filter(name__icontains="jo")  # Matches "John", "Jordan"
```

------------------------------------------------------------------------

### **2.3 Comparison Operators (`gt`, `gte`, `lt`, `lte`)**

| Lookup                        | Example                | SQL Equivalent    |
|-------------------------------|------------------------|-------------------|
| `gt` (greater than)           | `.filter(age__gt=30)`  | `WHERE age > 30`  |
| `gte` (greater than or equal) | `.filter(age__gte=30)` | `WHERE age >= 30` |
| `lt` (less than)              | `.filter(age__lt=30)`  | `WHERE age < 30`  |
| `lte` (less than or equal)    | `.filter(age__lte=30)` | `WHERE age <= 30` |

``` python
Employee.objects.filter(salary__gte=50000)
```

------------------------------------------------------------------------

### **2.4 Multiple Values (`in`)**

| Lookup | Example | SQL Equivalent |
|----|----|----|
| `in` | `.filter(department__in=["HR", "IT"])` | `WHERE department IN ('HR', 'IT')` |

``` python
Employee.objects.filter(department__in=["HR", "IT"])
```

------------------------------------------------------------------------

### **2.5 Null and Empty Values (`isnull`, `exact=""`)**

| Lookup        | Example                         | SQL Equivalent          |
|---------------|---------------------------------|-------------------------|
| `isnull=True` | `.filter(manager__isnull=True)` | `WHERE manager IS NULL` |
| `exact=""`    | `.filter(name="")`              | `WHERE name = ''`       |

``` python
Employee.objects.filter(manager__isnull=True)
```

------------------------------------------------------------------------

### **2.6 Date and Time Filters (`year`, `month`, `day`, `week_day`, `hour`, `minute`, `second`)**

| Lookup | Example | SQL Equivalent |
|----|----|----|
| `year` | `.filter(join_date__year=2024)` | `WHERE YEAR(join_date) = 2024` |
| `month` | `.filter(join_date__month=6)` | `WHERE MONTH(join_date) = 6` |
| `day` | `.filter(join_date__day=15)` | `WHERE DAY(join_date) = 15` |
| `week_day` | `.filter(join_date__week_day=2)` | `WHERE DAYOFWEEK(join_date) = 2` |

``` python
Employee.objects.filter(join_date__year=2024, join_date__month=1)
```

------------------------------------------------------------------------

## **3. Combining Filters**

### **3.1 AND Condition (`filter()`)**

``` python
# Employees in HR department with age > 30
Employee.objects.filter(department="HR", age__gt=30)
```

------------------------------------------------------------------------

### **3.2 OR Condition (`Q` objects)**

``` python
from django.db.models import Q

# Employees in HR OR IT department
Employee.objects.filter(Q(department="HR") | Q(department="IT"))
```

------------------------------------------------------------------------

### **3.3 NOT Condition (`~Q`)**

``` python
# Employees NOT in HR department
Employee.objects.filter(~Q(department="HR"))
```

------------------------------------------------------------------------

## **4. Ordering and Limiting Results**

| Method | Description | Example |
|----|----|----|
| `order_by()` | Orders by field | `.order_by("age")` |
| `order_by("-field")` | Orders in descending | `.order_by("-salary")` |
| `reverse()` | Reverses existing order | `.order_by("age").reverse()` |
| `distinct()` | Removes duplicate records | `.distinct()` |
| `count()` | Counts total rows | `.count()` |
| `first()` | First object | `.first()` |
| `last()` | Last object | `.last()` |
| `values()` | Returns a dictionary | `.values("name", "age")` |

``` python
# Get first 10 employees ordered by salary (descending)
Employee.objects.order_by("-salary")[:10]
```

------------------------------------------------------------------------

## **5. Query Aggregation (`Sum`, `Avg`, `Min`, `Max`, `Count`)**

``` python
from django.db.models import Avg, Sum, Count, Min, Max

# Average salary
Employee.objects.aggregate(Avg("salary"))

# Total salary
Employee.objects.aggregate(Sum("salary"))

# Count employees in IT department
Employee.objects.filter(department="IT").count()
```

------------------------------------------------------------------------

## **6. Prefetching and Optimizing Queries**

### **6.1 Select Related (One-to-One, ForeignKey)**

``` python
# Optimized query (joins related tables)
Employee.objects.select_related("department").all()
```

### **6.2 Prefetch Related (Many-to-Many, Reverse Relations)**

``` python
# Optimized query for many-to-many fields
Employee.objects.prefetch_related("projects").all()
```

------------------------------------------------------------------------

## **7. Raw SQL Queries (When Filters Are Not Enough)**

``` python
from django.db import connection

query = "SELECT * FROM myapp_employee WHERE age > %s"
with connection.cursor() as cursor:
    cursor.execute(query, [30])
    results = cursor.fetchall()
```

------------------------------------------------------------------------

## **8. Common Mistakes and Best Practices**

- Avoid fetching all records unnecessarily

``` python
# BAD: Loads entire table into memory
employees = Employee.objects.all()
for e in employees:
    print(e.name)
```

- Use efficient filtering

``` python
# GOOD: Filters data in the database
employees = Employee.objects.filter(age__gt=30)
```

- Avoid multiple queries inside loops

``` python
for emp in Employee.objects.all():
    print(emp.department.name)  # Triggers multiple DB queries
```

- Use `select_related` to optimize joins

``` python
for emp in Employee.objects.select_related("department").all():
    print(emp.department.name)  # Reduces queries
```

------------------------------------------------------------------------

## **Final Thoughts**

- **Use `filter()` for precise queries.**
- **Use `Q` for `OR` conditions.**
- **Use `select_related()` and `prefetch_related()` for optimized queries.**
- **Use `values()` and `only()` to fetch only required fields.**
- **Use `.count()` instead of `len(queryset)` to avoid unnecessary loading.**

Mastering Django filters **enhances database efficiency and performance!** 🚀

------------------------------------------------------------------------

Back to top
