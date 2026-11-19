# Managers and QuerySets

In Django, **managers** and **querysets** play a crucial role in retrieving data from the database, managing queries, and encapsulating database logic. Understanding how to effectively use and customize them can greatly improve the maintainability and performance of your Django applications.

Author

Benedict Thekkel

### 1. **What are Managers in Django?**

A **manager** in Django is the interface through which **database query operations** are provided to Django models. Every model in Django has at least one manager, and by default, Django provides a `Manager` object called `objects`.

#### a. **Default Manager**

By default, every model has a manager named `objects`, which allows you to interact with the model’s database records.

``` python
# Example model
class Book(models.Model):
    title = models.CharField(max_length=100)
    author = models.CharField(max_length=100)
    published_date = models.DateField()

# Using the default manager to query the database
books = Book.objects.all()  # Retrieves all books
book = Book.objects.get(id=1)  # Retrieves a specific book by ID
```

#### b. **Custom Manager**

You can create a custom manager to encapsulate logic specific to your model, allowing for more readable and reusable code.

``` python
# Custom manager
class BookManager(models.Manager):
    def published_last_year(self):
        last_year = timezone.now().year - 1
        return self.filter(published_date__year=last_year)

# Using a custom manager
class Book(models.Model):
    title = models.CharField(max_length=100)
    author = models.CharField(max_length=100)
    published_date = models.DateField()

    objects = BookManager()  # Use the custom manager

# Using the custom manager method
last_year_books = Book.objects.published_last_year()
```

### 2. **Creating Custom Managers**

Django allows you to define custom managers to encapsulate common queries, making your code more modular and readable. A custom manager can be defined by subclassing `models.Manager`.

#### a. **Basic Custom Manager**

``` python
class PublishedBookManager(models.Manager):
    def get_queryset(self):
        # Override the default get_queryset method to return only published books
        return super().get_queryset().filter(published_date__isnull=False)

# Using the custom manager in a model
class Book(models.Model):
    title = models.CharField(max_length=100)
    author = models.CharField(max_length=100)
    published_date = models.DateField()

    # Define two managers: a default manager and a custom manager
    objects = models.Manager()  # Default manager
    published = PublishedBookManager()  # Custom manager for published books
```

In this example: - `Book.objects.all()` will return all books. - `Book.published.all()` will return only books that have a `published_date`.

#### b. **Multiple Managers**

You can define multiple managers in a single model to access different sets of query logic.

``` python
class Book(models.Model):
    title = models.CharField(max_length=100)
    author = models.CharField(max_length=100)
    published_date = models.DateField()

    # Default and custom managers
    objects = models.Manager()  # Default manager
    published = PublishedBookManager()  # Custom manager
```

### 3. Querying Methods

#### 1. **Basic Retrieval and Querying Methods**

- **`all()`**: Retrieves all records from the database for that model.
- **`filter(**kwargs)`**: Returns a new `QuerySet` containing objects that match the given lookup parameters.
- **`exclude(**kwargs)`**: Returns a new `QuerySet` that excludes objects that match the given lookup parameters.
- **`get(**kwargs)`**: Returns a single object matching the given lookup parameters. Raises `DoesNotExist` if no object is found and `MultipleObjectsReturned` if more than one object matches.
- **`first()`**: Returns the first object in the `QuerySet`, or `None` if the `QuerySet` is empty.
- **`last()`**: Returns the last object in the `QuerySet`, or `None` if the `QuerySet` is empty.

#### 2. **Aggregation and Counting**

- **`count()`**: Returns the number of objects in the `QuerySet`.
- **`aggregate(**kwargs)`**: Returns a dictionary with the results of aggregating over the `QuerySet`.

#### 3. **Existence Check**

- **`exists()`**: Returns `True` if the `QuerySet` contains any results, and `False` if it is empty.

#### 4. **Creation and Deletion**

- **`create(**kwargs)`**: Creates and saves a new object, returning the created object.
- **`bulk_create(objs, batch_size=None, ignore_conflicts=False)`**: Inserts multiple objects into the database in a single query.
- **`bulk_update(objs, fields, batch_size=None)`**: Updates multiple objects in a single query.
- **`get_or_create(**kwargs)`**: Retrieves an object matching the given lookup parameters. If no matching object is found, it creates a new one with the provided defaults.
- **`update_or_create(**kwargs)`**: Updates an object matching the given lookup parameters, creating one if necessary.
- **`delete()`**: Deletes all objects in the `QuerySet` (returns the number of objects deleted and a dictionary with details).

#### 5. **Advanced Querying**

- **`raw(raw_query, params=None, translations=None)`**: Executes a raw SQL query and returns a `RawQuerySet` instance.
- **`extra(select=None, where=None, params=None, tables=None, order_by=None, select_params=None)`**: Adds extra SQL fragments to the query (deprecated in Django 3.1+).

#### 6. **Date and Time Methods**

- **`dates(field_name, kind, order='ASC')`**: Returns a `QuerySet` of all available dates for the specified field.
- **`datetimes(field_name, kind, order='ASC', tzinfo=None)`**: Returns a `QuerySet` of all available datetimes for the specified field.

#### 7. **Distinct and Ordering**

- **`distinct()`**: Returns a new `QuerySet` with distinct results.
- **`order_by(*field_names)`**: Orders the `QuerySet` by the given fields.
- **`reverse()`**: Reverses the order of the `QuerySet`.

#### 8. **Optimization and Caching**

- **`select_related(*fields)`**: Creates an SQL join and includes related objects in the `QuerySet`.
- **`prefetch_related(*lookups)`**: Performs a separate lookup for each relationship and joins them in Python, which is more efficient for many-to-many relationships.
- **`only(*fields)`**: Loads only the specified fields.
- **`defer(*fields)`**: Defers the loading of specified fields until they are accessed.

#### 9. **Union, Intersection, and Difference**

- **`union(*other_qs, all=False)`**: Combines `QuerySet` objects using SQL `UNION`.
- **`intersection(*other_qs)`**: Combines `QuerySet` objects using SQL `INTERSECT`.
- **`difference(*other_qs)`**: Combines `QuerySet` objects using SQL `EXCEPT`.

#### 10. **Boolean and None Operations**

- **`none()`**: Returns an empty `QuerySet`.

#### 11. **Custom Managers**

You can create your own custom manager by subclassing `models.Manager` and adding custom methods:

``` python
class CustomManager(models.Manager):
    def active(self):
        return self.filter(active=True)

class MyModel(models.Model):
    name = models.CharField(max_length=100)
    active = models.BooleanField(default=True)
    
    # Use the custom manager
    objects = CustomManager()
```

#### 12. **Using Managers**

``` python
# Using built-in manager methods
all_objects = MyModel.objects.all()
filtered_objects = MyModel.objects.filter(name__icontains="example")

# Using a custom manager method
active_objects = MyModel.objects.active()
```

These methods make Django’s ORM powerful and flexible, allowing for easy and expressive database queries.

### 4. **Manager Methods vs QuerySet Methods**

- **Manager Methods**: Manager methods are useful when you want to return data that does not require chaining or modifying the base QuerySet.
- **QuerySet Methods**: QuerySet methods are ideal when you want to create reusable filtering logic that can be chained with other QuerySet methods.

#### When to Use Each:

- Use **manager methods** when the method represents an action that doesn’t require chaining (e.g., creating objects or counting objects).
- Use **custom QuerySet methods** when you want to filter or manipulate a set of results and keep the option of chaining other QuerySet methods.

### 5. **Manager and QuerySet Best Practices**

#### a. **Keep Business Logic in Managers and QuerySets**

Encapsulate complex queries or business logic inside custom managers and QuerySets to make your views and templates cleaner.

#### b. **Use `get_queryset()` Wisely**

Always return a `QuerySet` from the `get_queryset()` method, as it allows you to further modify the query (e.g., chaining filters, ordering).

#### c. **Leverage Lazy Evaluation**

Django QuerySets are lazily evaluated, so you can build complex queries without immediately hitting the database. However, ensure queries are evaluated efficiently by minimizing unnecessary evaluations.

#### d. **Avoid Logic in Views**

Instead of placing filtering and querying logic in views, use custom managers or QuerySets to encapsulate and reuse this logic across your application.

#### e. **Chaining Queries**

When creating custom QuerySet methods, ensure they return a QuerySet so that you can continue to chain additional filters and operations.

Back to top
