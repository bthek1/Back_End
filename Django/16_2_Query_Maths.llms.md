# Query Math

If you’re working with **numbers**, **aggregations**, **calculations**, and **complex query math** in Django, the ORM gives you **powerful tools** to replicate **SQL arithmetic**, **aggregates**, and **window functions**, **without raw SQL**.

Author

Benedict Thekkel

## 🔎 **Everything You Need to Know About Query Math in Django ORM**

This guide will cover: 1. **Basic Math in Queries** (F-expressions, arithmetic) 2. **Aggregations & Grouping** (`Sum`, `Avg`, `Count`, etc.) 3. **Annotations (Field-level Math)** 4. **Window Functions (Rank, DenseRank, etc.)** 5. **Subquery + OuterRef (Correlated Calculations)** 6. **Complex Conditions (Q objects, Case/When)** 7. **Performance Tips** 8. **Real-world Examples**

------------------------------------------------------------------------

## 🧠 **1. Basic Math with F-expressions**

`F()` allows **referring to other fields** in a model **without fetching objects into Python memory**.

### Math Operations with `F()`

``` python
from django.db.models import F

# Give every product a 10% price increase
Product.objects.update(price=F('price') * 1.1)

# Deduct stock after an order
Product.objects.update(stock=F('stock') - 1)

# Compare two fields in a filter
Order.objects.filter(total__gt=F('discount'))
```

✅ F-expressions **avoid extra queries**, are **database-side**, and **atomic**.

------------------------------------------------------------------------

## 📊 **2. Aggregations (Summing, Counting, Averaging)**

You can perform **aggregate math** across rows.

### Common Aggregation Functions

``` python
from django.db.models import Sum, Avg, Count, Min, Max

# Get total sales
Order.objects.aggregate(total_sales=Sum('total'))

# Get average order value
Order.objects.aggregate(avg_order=Avg('total'))

# Count of orders
Order.objects.aggregate(order_count=Count('id'))
```

### Filter + Aggregate Example:

``` python
high_value_orders = Order.objects.filter(total__gt=1000).aggregate(
    total=Sum('total'),
    count=Count('id')
)
```

------------------------------------------------------------------------

## ➕ **3. Annotations (Per-Row Math)**

`annotate()` adds **calculated fields** to each row in a queryset.

### Example: Calculate Revenue per Product

``` python
from django.db.models import Sum, F

Product.objects.annotate(
    revenue=F('price') * F('units_sold')
)

# Fetch revenue with product
for p in Product.objects.annotate(
    revenue=F('price') * F('units_sold')
):
    print(p.name, p.revenue)
```

------------------------------------------------------------------------

## 🪝 **4. Case/When for Conditional Math**

`Case` and `When` enable **conditional logic**, like SQL’s `CASE ... WHEN ... THEN ... END`.

### Example: Apply Discount Conditionally

``` python
from django.db.models import Case, When, Value, F

Product.objects.annotate(
    discounted_price=Case(
        When(is_on_sale=True, then=F('price') * 0.8),
        default=F('price')
    )
)
```

### Another Example: Calculate Employee Bonus

``` python
from django.db.models import IntegerField

Employee.objects.annotate(
    bonus=Case(
        When(sales__gte=100, then=Value(1000)),
        When(sales__gte=50, then=Value(500)),
        default=Value(0),
        output_field=IntegerField()
    )
)
```

------------------------------------------------------------------------

## 🔄 **5. Window Functions (Advanced Math by Groups)**

Window functions perform **calculations across a set of rows** related to the current row.

### Example: Ranking Orders by Amount

``` python
from django.db.models import Window
from django.db.models.functions import Rank

Order.objects.annotate(
    rank=Window(
        expression=Rank(),
        order_by=F('total').desc()
    )
)
```

### Example: Running Totals

``` python
from django.db.models.functions import Sum

Order.objects.annotate(
    running_total=Window(
        expression=Sum('total'),
        order_by='order_date'
    )
)
```

✅ Great for **leaderboards**, **cumulative sums**, **partitioned aggregates**.

------------------------------------------------------------------------

## 🔢 **6. Subquery + OuterRef (Correlated Subqueries for Math)**

Use `Subquery` + `OuterRef` for **row-by-row subquery math**.

### Example: Latest Order Amount Per Customer

``` python
from django.db.models import Subquery, OuterRef

latest_order = Order.objects.filter(
    customer=OuterRef('pk')
).order_by('-order_date')

Customer.objects.annotate(
    latest_order_total=Subquery(latest_order.values('total')[:1])
)
```

✅ Combine with `F()`, `Case`, and `Window` for **complex formulas**.

------------------------------------------------------------------------

## 🏎️ **7. Combining Aggregation, Annotation, and Filtering**

You can **chain** these functions for **multi-layered math**.

### Example: Total Spent Per Customer and Filtering High Spenders

``` python
Customer.objects.annotate(
    total_spent=Sum('order__total')
).filter(total_spent__gte=1000)
```

------------------------------------------------------------------------

## 🔨 **8. Complex Math: Expressions & Functions**

### Django Built-in Math Functions:

| Function         | Purpose                              |
|------------------|--------------------------------------|
| `Abs`            | Absolute value (`Abs(F('field'))`)   |
| `Ceil` / `Floor` | Round up / down (`Ceil(F('field'))`) |
| `Power`          | Exponents (`Power(F('field'), 2)`)   |
| `Sqrt`           | Square root (`Sqrt(F('field'))`)     |
| `Log`, `Exp`     | Logarithm, Exponentiation            |

#### Example: Calculate Compound Interest

``` python
from django.db.models.functions import Power

Account.objects.annotate(
    future_value=F('balance') * Power(1.05, F('years'))
)
```

------------------------------------------------------------------------

## 🕵️ **9. Q Objects (AND/OR/NOT Conditions)**

`Q()` objects allow **complex WHERE logic**.

### Example: Math in Filters

``` python
from django.db.models import Q

Product.objects.filter(
    Q(price__gte=100) | Q(stock__lte=10)
)
```

### Example: Sales Threshold

``` python
Employee.objects.filter(
    Q(sales__gte=100) & Q(region='East')
)
```

------------------------------------------------------------------------

## 📈 **10. Performance Tips with ORM Math**

| Tip | Why |
|----|----|
| Use `F()` expressions | Database-side math is faster |
| Combine `annotate()` + `aggregate()` | Reduce query round trips |
| Use `select_related()` / `prefetch_related()` | Reduce N+1 queries |
| Add indexes | Speed up filtering and ordering |
| Profile your queries | `django-debug-toolbar`, `EXPLAIN` |
| Avoid unnecessary fields | Use `only()` or `values()` |

------------------------------------------------------------------------

## ✅ **Summary of Django Query Math Tools**

| Tool                    | Use                                              |
|-------------------------|--------------------------------------------------|
| `F()`                   | Refer to other fields, arithmetic in DB          |
| `annotate()`            | Add calculated fields to each row                |
| `aggregate()`           | Calculate sums, averages, counts across all rows |
| `Case` / `When`         | Conditional math and logic                       |
| `Window`                | Rankings, cumulative sums, per-group aggregates  |
| `Subquery` / `OuterRef` | Correlated subqueries for row-level math         |
| Math Functions          | `Abs`, `Power`, `Ceil`, `Floor`, `Log`, `Exp`    |
| `Q()`                   | Complex query conditions (`AND`, `OR`, `NOT`)    |

------------------------------------------------------------------------

## ⚡ Real-World Scenarios

| Task                            | Solution                       |
|---------------------------------|--------------------------------|
| **Discount based on quantity**  | `Case` + `When` + `F()`        |
| **Sales leaderboard**           | `Window` + `Rank`              |
| **Latest related record field** | `OuterRef` + `Subquery`        |
| **Customer lifetime value**     | `annotate(Sum(...))`           |
| **Running totals by date**      | `Window(Sum(...))`             |
| **Tiered commissions**          | `Case` + `When` + `annotate()` |

------------------------------------------------------------------------

Back to top
