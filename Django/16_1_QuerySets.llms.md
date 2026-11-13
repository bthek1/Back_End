# QuerySet

QuerySets in Django

Author

Benedict Thekkel

### 1. **The Role of QuerySets in Django**

A **QuerySet** represents a collection of database queries that return a set of results. QuerySets are lazily evaluated, meaning that the database query is only executed when the QuerySet is evaluated (e.g., when iterating over it or converting it to a list).

#### a. **Basic QuerySet Operations**

QuerySets allow you to filter, order, and retrieve data from the database.

``` python
# Retrieving all objects
books = Book.objects.all()

# Filtering objects
filtered_books = Book.objects.filter(author="George Orwell")

# Chaining filters (AND logic)
recent_books = Book.objects.filter(published_date__year=2020).filter(author="George Orwell")

# Retrieving a single object
book = Book.objects.get(id=1)

# Exclude objects
non_orwell_books = Book.objects.exclude(author="George Orwell")

# Ordering results
ordered_books = Book.objects.order_by('published_date')

# Limit number of results (slicing)
first_ten_books = Book.objects.all()[:10]
```

#### b. **QuerySet Evaluation**

A QuerySet is **lazily evaluated**, meaning the query is not actually executed in the database until the QuerySet is iterated over or explicitly evaluated.

Examples of when QuerySets are evaluated: - **Iterating** over the QuerySet. - **Slicing** the QuerySet. - **Serializing** the QuerySet (e.g., converting it to a list or calling `len()`).

``` python
# Query is not executed yet
books = Book.objects.filter(author="George Orwell")

# Query is executed when you evaluate the QuerySet
for book in books:
    print(book.title)
```

### 2. **Custom QuerySets**

Instead of overriding the manager, you can create a custom QuerySet and use it directly in your model manager. This method allows you to chain custom QuerySet methods.

#### a. **Creating a Custom QuerySet**

``` python
class BookQuerySet(models.QuerySet):
    def published(self):
        return self.filter(published_date__isnull=False)

    def by_author(self, author_name):
        return self.filter(author=author_name)

# Use the custom QuerySet in a manager
class BookManager(models.Manager):
    def get_queryset(self):
        return BookQuerySet(self.model, using=self._db)

# Use the manager in the model
class Book(models.Model):
    title = models.CharField(max_length=100)
    author = models.CharField(max_length=100)
    published_date = models.DateField()

    objects = BookManager()
```

Now you can chain custom QuerySet methods together:

``` python
books = Book.objects.published().by_author("George Orwell")
```

#### b. **Chaining Custom QuerySet Methods**

Custom QuerySet methods allow you to chain operations and write more readable and reusable query logic:

``` python
books = Book.objects.published().by_author("J.K. Rowling").order_by('published_date')
```

#### c. **Combining Managers and QuerySets**

A common pattern is to define both custom QuerySets and managers, allowing you to use manager-level logic while preserving the ability to chain QuerySet methods.

``` python
class BookQuerySet(models.QuerySet):
    def published(self):
        return self.filter(published_date__isnull=False)

class BookManager(models.Manager):
    def get_queryset(self):
        return BookQuerySet(self.model, using=self._db)

    def published_last_year(self):
        last_year = timezone.now().year - 1
        return self.get_queryset().published().filter(published_date__year=last_year)
```

### 3. **QuerySet Methods**

Django’s QuerySet API provides numerous built-in methods to filter, manipulate, and aggregate data. Some of the most commonly used methods are:

#### 1. **Retrieval Methods**

- **`all()`**: Returns a new `QuerySet` containing all objects in the database.
- **`filter(**kwargs)`**: Returns a new `QuerySet` containing objects that match the given lookup parameters.
- **`exclude(**kwargs)`**: Returns a new `QuerySet` excluding objects that match the given lookup parameters.
- **`get(**kwargs)`**: Returns a single object matching the given lookup parameters. Raises `DoesNotExist` if no object is found and `MultipleObjectsReturned` if more than one object matches.
- **`first()`**: Returns the first object in the `QuerySet`, or `None` if the `QuerySet` is empty.
- **`last()`**: Returns the last object in the `QuerySet`, or `None` if the `QuerySet` is empty.
- **`earliest(field_name=None)`**: Returns the earliest object according to a given field or `Meta` ordering.
- **`latest(field_name=None)`**: Returns the latest object according to a given field or `Meta` ordering.
- **`count()`**: Returns the number of objects in the `QuerySet`.
- **`exists()`**: Returns `True` if the `QuerySet` contains any results, and `False` if it is empty.

#### 2. **Aggregation and Annotation**

- **`aggregate(**kwargs)`**: Returns a dictionary with the results of aggregating over the `QuerySet`.
- **`annotate(**kwargs)`**: Adds annotations to each object in the `QuerySet` based on the provided aggregate functions.

#### 3. **Ordering Methods**

- **`order_by(*field_names)`**: Returns a new `QuerySet` with the objects ordered by the given fields.
- **`reverse()`**: Reverses the order of the `QuerySet`.

#### 4. **Limiting Methods**

- **`distinct()`**: Returns a new `QuerySet` with distinct results.
- **`values(*fields)`**: Returns a `QuerySet` that returns dictionaries when iterated over, each representing an object with the specified fields as keys.
- **`values_list(*fields, flat=False)`**: Returns a `QuerySet` that yields tuples when iterated over. If `flat=True`, it returns single values.
- **`only(*fields)`**: Limits the fields that are loaded to the ones specified.
- **`defer(*fields)`**: Defers the loading of the specified fields until they are accessed.

#### 5. **Modification Methods**

- **`update(**kwargs)`**: Updates fields in the database for all objects in the `QuerySet` (returns the number of rows affected).
- **`delete()`**: Deletes all objects in the `QuerySet` (returns the number of objects deleted and a dictionary with details).
- **`create(**kwargs)`**: Creates a new object, saves it, and returns it.
- **`bulk_create(objs, batch_size=None, ignore_conflicts=False)`**: Inserts multiple objects into the database in a single query.
- **`bulk_update(objs, fields, batch_size=None)`**: Updates multiple objects with new field values in a single query.

#### 6. **Chaining and Combining Methods**

- **`union(*other_qs, all=False)`**: Combines `QuerySet` objects by performing an SQL `UNION`.
- **`intersection(*other_qs)`**: Combines `QuerySet` objects by performing an SQL `INTERSECT`.
- **`difference(*other_qs)`**: Combines `QuerySet` objects by performing an SQL `EXCEPT`.

#### 7. **Boolean and Existence Methods**

- **`exists()`**: Returns `True` if the `QuerySet` contains any results, `False` if not.
- **`none()`**: Returns an empty `QuerySet`.

#### 8. **Caching and Evaluation**

- **`iterator()`**: Uses an iterator to loop over the `QuerySet` without caching results.
- **`select_related(*fields)`**: Creates an SQL join and includes related objects in the `QuerySet`.
- **`prefetch_related(*lookups)`**: Prefetches related objects to reduce the number of database queries.

#### 9. **Date and Time Methods**

- **`dates(field_name, kind, order='ASC')`**: Returns a `QuerySet` of all available dates.
- **`datetimes(field_name, kind, order='ASC', tzinfo=None)`**: Returns a `QuerySet` of all available datetimes.

#### 10. **Advanced Querying**

- **`extra(select=None, where=None, params=None, tables=None, order_by=None, select_params=None)`**: Adds extra SQL fragments to the query.
- **`raw(raw_query, params=None, translations=None)`**: Executes a raw SQL query.

These methods make it easy to create powerful, flexible queries in Django.

Back to top
