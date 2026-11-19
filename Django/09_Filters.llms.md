# Filters

**Django Filters** provide a way to filter queryset data based on form input. It is especially useful for enabling dynamic filtering in Django views or API endpoints. The **django-filter** library simplifies this process by providing reusable filter classes for Django models, which can be integrated seamlessly with Django’s class-based views, forms, and **Django REST Framework (DRF)**.

Author

Benedict Thekkel

### 1. **Basic Setup: Install and Configure Django Filter**

#### Installation:

To use **django-filter**, install it via `pip`:

``` bash
pip install django-filter
```

Add `'django_filters'` to your `INSTALLED_APPS` in the Django `settings.py`:

``` python
INSTALLED_APPS = [
    # Other apps...
    'django_filters',
]
```

### 2. **Defining a `FilterSet` Class**

A **FilterSet** is the core class of **django-filter**. It defines which model fields should be available for filtering.

#### Example: A `Book` Model

``` python
from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=255)
    author = models.CharField(max_length=100)
    published_date = models.DateField()
    price = models.DecimalField(max_digits=5, decimal_places=2)
```

#### Defining a FilterSet:

``` python
import django_filters
from .models import Book

class BookFilter(django_filters.FilterSet):
    class Meta:
        model = Book
        fields = ['author', 'published_date', 'price']
```

### 3. **Using Filters with Django’s Generic Views**

Django’s **ListView** or **TemplateView** can easily be enhanced with filtering capabilities.

#### Example: Using Filters in a View

``` python
from django.shortcuts import render
from django_filters.views import FilterView
from .models import Book
from .filters import BookFilter

class BookListView(FilterView):
    model = Book
    filterset_class = BookFilter
    template_name = 'books/book_list.html'
```

#### Displaying the Filter in Templates:

``` html
<form method="get">
    {{ filter.form.as_p }}
    <button type="submit">Filter</button>
</form>

<ul>
  {% for book in filter.qs %}
    <li>{{ book.title }} by {{ book.author }}</li>
  {% endfor %}
</ul>
```

### 4. **Using Filters with Django REST Framework (`ModelViewSet`)**

In Django REST Framework (DRF), you can easily integrate filters with your API endpoints using `ModelViewSet` and the **DjangoFilterBackend**.

#### Steps to Enable Filtering in DRF:

1.  **Add `django_filters` to DRF Configuration**: In your `settings.py` file, add `DjangoFilterBackend` to `DEFAULT_FILTER_BACKENDS`:

``` python
REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': ['django_filters.rest_framework.DjangoFilterBackend']
}
```

2.  **Create a FilterSet**: Using the earlier `BookFilter` example:

``` python
import django_filters
from .models import Book

class BookFilter(django_filters.FilterSet):
    min_price = django_filters.NumberFilter(field_name="price", lookup_expr='gte')
    max_price = django_filters.NumberFilter(field_name="price", lookup_expr='lte')

    class Meta:
        model = Book
        fields = ['author', 'published_date', 'min_price', 'max_price']
```

3.  **Integrate FilterSet with `ModelViewSet`**:

``` python
from rest_framework import viewsets
from django_filters.rest_framework import DjangoFilterBackend
from .models import Book
from .serializers import BookSerializer
from .filters import BookFilter

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [DjangoFilterBackend]
    filterset_class = BookFilter
```

#### Example Query Parameters in API:

You can now filter books by passing query parameters like:

    /api/books/?author=John&min_price=10&max_price=50

This URL filters books written by “John” with a price between 10 and 50.

### 5. **Advanced Filters**

#### Custom Filters:

For more advanced use cases, you can define custom filters. For example, to filter by a date range:

``` python
class BookFilter(django_filters.FilterSet):
    date_range = django_filters.DateFromToRangeFilter(field_name="published_date")

    class Meta:
        model = Book
        fields = ['author', 'date_range']
```

#### Using Different Lookups:

You can apply different lookup expressions, such as: - **exact**: Matches an exact value. - **iexact**: Case-insensitive exact match. - **contains**: Checks if a field contains a substring. - **gte**: Greater than or equal. - **lte**: Less than or equal.

``` python
class BookFilter(django_filters.FilterSet):
    title_contains = django_filters.CharFilter(field_name="title", lookup_expr='icontains')
```

### 6. **Pagination with Filters**

Django Filter works well with Django’s built-in **pagination**. You can paginate the filtered results in both class-based and function-based views.

#### Example with Pagination in Function-Based View:

``` python
from django.core.paginator import Paginator

def book_list(request):
    books = Book.objects.all()
    filter = BookFilter(request.GET, queryset=books)
    paginator = Paginator(filter.qs, 10)  # Show 10 books per page
    page = request.GET.get('page')
    books = paginator.get_page(page)
    return render(request, 'books/book_list.html', {'filter': filter, 'books': books})
```

### 7. **Performance Considerations**

- **Large Querysets**: Filtering large datasets can be computationally expensive. If you expect heavy use of filters on large datasets, you may want to optimize the queries by adding **database indexes** on fields being filtered.
- **Complex Queries**: If your filters are complex (e.g., involving joins across multiple tables), consider writing custom queryset methods or optimizing the queries to improve performance.

### 8. **Best Practices for Django Filters**

- **Use Filters Sparingly**: Avoid overcomplicating views by adding too many filters. Provide users with the most relevant filter options.
- **Cache Filter Results**: If filtering is used frequently, caching the filtered querysets can greatly improve performance.
- **Paginate Large Querysets**: Always paginate filtered querysets to avoid slow rendering times on large datasets.
- **Combine Filters and Search**: For a more flexible user experience, consider combining filters with a search functionality using Django’s `Q` objects or a full-text search tool.

Back to top
