# Nested Routers

Nested Routers

Author

Benedict Thekkel

## 1. Introduction to Routers in DRF

**Routers** in Django REST Framework are classes that automatically generate URL configurations for your API views. They simplify the process of mapping HTTP methods to view actions, reducing boilerplate code and ensuring consistency across your API.

### **Key Components:**

- **ViewSets**: Classes that combine the logic for multiple related views (e.g., list, create, retrieve, update, delete).
- **Router Classes**: Handle the URL routing by registering ViewSets and generating the appropriate URL patterns.

### **Basic Router Example:**

``` python
# urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from .views import AuthorViewSet, BookViewSet

router = DefaultRouter()
router.register(r'authors', AuthorViewSet)
router.register(r'books', BookViewSet)

urlpatterns = [
    path('', include(router.urls)),
]
```

This setup automatically creates routes like:

- `/authors/` (GET, POST)
- `/authors/{id}/` (GET, PUT, DELETE)
- `/books/` (GET, POST)
- `/books/{id}/` (GET, PUT, DELETE)

------------------------------------------------------------------------

## 2. What Are Nested Routers?

**Nested Routers** extend the functionality of standard routers by allowing you to create nested URL patterns that reflect the relationships between different resources (models) in your application. This is particularly useful when you have related models, such as a `Client` having multiple `Treatments`.

### **Benefits of Nested Routers:**

- **Hierarchical URL Structure**: Reflects the relationships between resources, making the API more intuitive.
- **Scoped Access**: Ensures that nested resources are accessed within the context of their parent resource.
- **Organized Codebase**: Keeps URL configurations clean and manageable, especially in large projects.

### **Example of Nested Routes:**

Given models `Client` and `Treatment`, where a `Client` has many `Treatments`:

- `/clients/` (List/Create Clients)
- `/clients/{client_id}/` (Retrieve/Update/Delete a Client)
- `/clients/{client_id}/treatments/` (List/Create Treatments for a Client)
- `/clients/{client_id}/treatments/{treatment_id}/` (Retrieve/Update/Delete a Treatment for a Client)

------------------------------------------------------------------------

## 3. When to Use Nested Routers

**Use Nested Routers When:**

- **Hierarchical Relationships**: Models have parent-child relationships (e.g., Authors and Books, Users and Posts).
- **Scoped Operations**: Operations on child resources should be scoped to a specific parent (e.g., adding a comment to a specific post).
- **Improved Readability**: You want your API endpoints to clearly represent the relationships between resources.

**Avoid Nested Routers When:**

- **Deep Nesting**: Excessive nesting can lead to complex and unwieldy URLs. Limit nesting to 1-2 levels.
- **No Clear Hierarchy**: Models are not directly related or the relationship does not require nested access.
- **Alternative Structures Suffice**: Flat structures with query parameters or filters can achieve similar results without nesting.

------------------------------------------------------------------------

## 4. Installing and Setting Up Nested Routers

To implement nested routers in DRF, you typically use the [`drf-nested-routers`](https://github.com/alanjds/drf-nested-routers) package, which extends DRF’s router classes to support nested URL patterns.

### **Installation:**

Use `pip` to install `drf-nested-routers`:

``` bash
pip install drf-nested-routers
```

### **Verify Installation:**

Ensure that `drf-nested-routers` is installed correctly by checking the package list:

``` bash
pip list | grep drf-nested-routers
```

------------------------------------------------------------------------

## 5. Basic Implementation with `drf-nested-routers`

Let’s walk through a basic implementation of nested routers using `drf-nested-routers`.

### **Scenario:**

You have two models: `Author` and `Book`. Each `Author` can have multiple `Books`.

### **Models:**

``` python
# models.py
from django.db import models

class Author(models.Model):
    name = models.CharField(max_length=100)
    bio = models.TextField(blank=True)

    def __str__(self):
        return self.name

class Book(models.Model):
    author = models.ForeignKey(Author, related_name='books', on_delete=models.CASCADE)
    title = models.CharField(max_length=200)
    published_date = models.DateField()

    def __str__(self):
        return self.title
```

### **Serializers:**

``` python
# serializers.py
from rest_framework import serializers
from .models import Author, Book

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'published_date', 'author']

class AuthorSerializer(serializers.ModelSerializer):
    books = BookSerializer(many=True, read_only=True)  # Nested representation

    class Meta:
        model = Author
        fields = ['id', 'name', 'bio', 'books']
```

### **ViewSets:**

``` python
# views.py
from rest_framework import viewsets
from .models import Author, Book
from .serializers import AuthorSerializer, BookSerializer

class AuthorViewSet(viewsets.ModelViewSet):
    """
    A viewset for viewing and editing author instances.
    """
    queryset = Author.objects.all()
    serializer_class = AuthorSerializer

class BookViewSet(viewsets.ModelViewSet):
    """
    A viewset for viewing and editing book instances.
    """
    serializer_class = BookSerializer

    def get_queryset(self):
        """
        Optionally restricts the returned books to a given author,
        by filtering against a `author_pk` query parameter in the URL.
        """
        queryset = Book.objects.all()
        author_pk = self.kwargs.get('author_pk')
        if author_pk is not None:
            queryset = queryset.filter(author_id=author_pk)
        return queryset

    def perform_create(self, serializer):
        """
        Associates the book with the given author.
        """
        author_pk = self.kwargs.get('author_pk')
        serializer.save(author_id=author_pk)
```

### **URL Configuration with Nested Routers:**

``` python
# urls.py
from django.urls import path, include
from rest_framework_nested import routers
from .views import AuthorViewSet, BookViewSet

# Create the main router and register the AuthorViewSet
router = routers.DefaultRouter()
router.register(r'authors', AuthorViewSet, basename='authors')

# Create a nested router for books under authors
authors_router = routers.NestedDefaultRouter(router, r'authors', lookup='author')
authors_router.register(r'books', BookViewSet, basename='author-books')

# Include both routers in the URL patterns
urlpatterns = [
    path('', include(router.urls)),
    path('', include(authors_router.urls)),
]
```

### **Generated URL Patterns:**

- `/authors/` - List/Create Authors
- `/authors/{author_id}/` - Retrieve/Update/Delete a Specific Author
- `/authors/{author_id}/books/` - List/Create Books for a Specific Author
- `/authors/{author_id}/books/{book_id}/` - Retrieve/Update/Delete a Specific Book for an Author

### **Testing the Endpoints:**

Using tools like **Postman** or **cURL**, you can interact with the API:

- **List Authors:**

  ``` http
  GET /authors/
  ```

- **Retrieve a Specific Author:**

  ``` http
  GET /authors/1/
  ```

- **List Books for an Author:**

  ``` http
  GET /authors/1/books/
  ```

- **Create a Book for an Author:**

  ``` http
  POST /authors/1/books/
  Content-Type: application/json

  {
      "title": "New Book Title",
      "published_date": "2024-01-01"
  }
  ```

- **Retrieve a Specific Book for an Author:**

  ``` http
  GET /authors/1/books/2/
  ```

------------------------------------------------------------------------

## 6. Advanced Usage and Customization

While basic nested routing is straightforward, you might encounter scenarios that require more advanced configurations.

### **a. Multiple Levels of Nesting**

You can nest routers to multiple levels, but it’s generally recommended to keep nesting to a reasonable depth (typically 2 levels) to avoid overly complex URLs.

**Example:**

Models: `Author`, `Book`, `Chapter`

``` python
# urls.py
from rest_framework_nested import routers
from .views import AuthorViewSet, BookViewSet, ChapterViewSet

router = routers.DefaultRouter()
router.register(r'authors', AuthorViewSet, basename='authors')

authors_router = routers.NestedDefaultRouter(router, r'authors', lookup='author')
authors_router.register(r'books', BookViewSet, basename='author-books')

books_router = routers.NestedDefaultRouter(authors_router, r'books', lookup='book')
books_router.register(r'chapters', ChapterViewSet, basename='book-chapters')

urlpatterns = [
    path('', include(router.urls)),
    path('', include(authors_router.urls)),
    path('', include(books_router.urls)),
]
```

Generated URLs:

- `/authors/{author_id}/books/{book_id}/chapters/` - List/Create Chapters for a Book
- `/authors/{author_id}/books/{book_id}/chapters/{chapter_id}/` - Retrieve/Update/Delete a Specific Chapter

### **b. Using `trailing_slash=False`**

By default, DRF routers append a trailing slash to URLs. If you prefer URLs without a trailing slash, you can configure it globally or per router.

**Example:**

``` python
router = routers.DefaultRouter(trailing_slash=False)
router.register(r'authors', AuthorViewSet, basename='authors')

authors_router = routers.NestedDefaultRouter(router, r'authors', lookup='author', trailing_slash=False)
authors_router.register(r'books', BookViewSet, basename='author-books')
```

### **c. Custom Lookup Fields**

By default, DRF uses the `pk` (primary key) for URL lookups. You can customize this to use other fields, such as `slug`.

**Example:**

``` python
# views.py
class AuthorViewSet(viewsets.ModelViewSet):
    queryset = Author.objects.all()
    serializer_class = AuthorSerializer
    lookup_field = 'slug'

# urls.py
router.register(r'authors', AuthorViewSet, basename='authors')
authors_router = routers.NestedDefaultRouter(router, r'authors', lookup='author', lookup_field='slug')
```

### **d. Handling Permissions and Access Control**

Nested routers don’t inherently handle permissions differently. However, you can leverage the nested context in your ViewSets to enforce permissions based on the parent resource.

**Example:**

``` python
# views.py
from rest_framework import permissions

class BookViewSet(viewsets.ModelViewSet):
    serializer_class = BookSerializer
    permission_classes = [permissions.IsAuthenticated]

    def get_queryset(self):
        author_pk = self.kwargs.get('author_pk')
        return Book.objects.filter(author_id=author_pk)

    def perform_create(self, serializer):
        author_pk = self.kwargs.get('author_pk')
        author = get_object_or_404(Author, pk=author_pk)
        serializer.save(author=author)
```

### **e. Custom Actions in Nested ViewSets**

You can define custom actions (e.g., additional endpoints) within nested ViewSets to handle specific operations.

**Example: Adding a custom action to retrieve books published after a certain date:**

``` python
# views.py
from rest_framework.decorators import action
from rest_framework.response import Response
from django.utils.dateparse import parse_date

class BookViewSet(viewsets.ModelViewSet):
    # ... existing methods ...

    @action(detail=False, methods=['get'])
    def published_after(self, request, author_pk=None):
        date_str = request.query_params.get('date')
        if not date_str:
            return Response({"error": "Date parameter is required."}, status=400)
        date = parse_date(date_str)
        if not date:
            return Response({"error": "Invalid date format."}, status=400)
        books = self.get_queryset().filter(published_date__gt=date)
        serializer = self.get_serializer(books, many=True)
        return Response(serializer.data)
```

**Accessing the Custom Action:**

    GET /authors/{author_id}/books/published_after/?date=2024-01-01

------------------------------------------------------------------------

## 7. Common Use Cases and Examples

### **a. E-commerce Application**

**Models:**

- `Category`
- `Product`
- `Review`

**URL Structure:**

- `/categories/`
- `/categories/{category_id}/products/`
- `/categories/{category_id}/products/{product_id}/reviews/`

**Implementation:**

``` python
# urls.py
router = routers.DefaultRouter()
router.register(r'categories', CategoryViewSet, basename='categories')

categories_router = routers.NestedDefaultRouter(router, r'categories', lookup='category')
categories_router.register(r'products', ProductViewSet, basename='category-products')

products_router = routers.NestedDefaultRouter(categories_router, r'products', lookup='product')
products_router.register(r'reviews', ReviewViewSet, basename='product-reviews')

urlpatterns = [
    path('', include(router.urls)),
    path('', include(categories_router.urls)),
    path('', include(products_router.urls)),
]
```

### **b. Blogging Platform**

**Models:**

- `Author`
- `Post`
- `Comment`

**URL Structure:**

- `/authors/`
- `/authors/{author_id}/posts/`
- `/authors/{author_id}/posts/{post_id}/comments/`

**Implementation:**

``` python
# urls.py
router = routers.DefaultRouter()
router.register(r'authors', AuthorViewSet, basename='authors')

authors_router = routers.NestedDefaultRouter(router, r'authors', lookup='author')
authors_router.register(r'posts', PostViewSet, basename='author-posts')

posts_router = routers.NestedDefaultRouter(authors_router, r'posts', lookup='post')
posts_router.register(r'comments', CommentViewSet, basename='post-comments')

urlpatterns = [
    path('', include(router.urls)),
    path('', include(authors_router.urls)),
    path('', include(posts_router.urls)),
]
```

------------------------------------------------------------------------

## 8. Potential Pitfalls and How to Avoid Them

While nested routers are powerful, they can introduce complexity if not used judiciously. Here are common pitfalls and strategies to avoid them:

### **a. Over-Nesting**

**Issue:** Excessive nesting leads to deeply hierarchical URLs, making them hard to manage and understand.

**Solution:** Limit nesting to 1-2 levels. For deeper relationships, consider alternative approaches like query parameters or flattening the URL structure.

### **b. Complex URL Patterns**

**Issue:** Managing multiple nested routers can result in complex URL configurations.

**Solution:** Organize your `urls.py` logically, possibly splitting into multiple modules if necessary. Use consistent naming conventions and documentation.

### **c. Handling Lookup Fields**

**Issue:** Incorrect or inconsistent lookup fields can lead to broken URLs and failed lookups.

**Solution:** Ensure that `lookup_field` is consistently defined across ViewSets and routers. Use meaningful and unique fields like `slug` if necessary.

### **d. Performance Concerns**

**Issue:** Nested queries can lead to increased database hits, affecting performance.

**Solution:** Optimize your ViewSets by using `select_related` and `prefetch_related` to minimize database queries. Implement pagination where appropriate.

### **e. Permissions and Access Control**

**Issue:** Inconsistent permissions across nested resources can lead to security vulnerabilities.

**Solution:** Define and enforce permissions within each ViewSet, considering the context provided by nested relationships. Use DRF’s permission classes effectively.

### **f. Cache Invalidation**

**Issue:** When using caching mechanisms, nested resources might require careful cache invalidation strategies.

**Solution:** Implement cache invalidation rules that account for changes in parent resources affecting child resources.

------------------------------------------------------------------------

## 9. Best Practices

Adhering to best practices ensures that your use of nested routers is effective and maintainable.

### **a. Keep Nesting Shallow**

Limit nesting to avoid overly complex URLs. A common guideline is to nest no more than 1-2 levels deep.

### **b. Use Meaningful Lookup Fields**

Prefer using unique and meaningful fields like `slug` over numeric IDs when appropriate, enhancing the readability of URLs.

### **c. Optimize Querysets**

Leverage Django’s ORM capabilities (`select_related`, `prefetch_related`) to optimize database access and reduce query counts.

### **d. Consistent Naming Conventions**

Use consistent naming for routers, ViewSets, and URL patterns to improve code readability and maintainability.

### **e. Implement Proper Permissions**

Ensure that each ViewSet has appropriate permission classes to secure access to resources, especially in nested contexts.

### **f. Document Your API**

Use tools like **Swagger** or **DRF-YASG** to generate API documentation, making it easier for developers to understand and use your nested endpoints.

### **g. Handle Errors Gracefully**

Implement robust error handling in your ViewSets to manage scenarios where parent resources do not exist or access is unauthorized.

### **h. Consider Alternative Approaches When Necessary**

If nesting becomes too complex, explore alternative structures such as:

- **Query Parameters**: Filter resources based on query parameters without nesting.
- **Separate Endpoints**: Provide separate endpoints for related resources without hierarchical URLs.

------------------------------------------------------------------------

## 10. Alternatives to Nested Routers

While nested routers are useful, they aren’t always the best solution. Here are some alternatives:

### **a. Flat Routers with Query Parameters**

Instead of nesting, use flat URLs and filter child resources based on query parameters.

**Example:**

- `/books/?author_id=1/`

**Pros:**

- Simpler URL structure.
- Easier to manage and understand.

**Cons:**

- Less intuitive in reflecting relationships.
- May require more complex filtering logic.

### **b. Hyperlinked Relationships**

Use hyperlinked serializers to include links to related resources without nesting URLs.

**Pros:**

- Decouples URL structure from resource relationships.
- Flexible and RESTful.

**Cons:**

- Less intuitive browsing through related resources via URL.

### **c. Custom URL Patterns**

Define custom URL patterns tailored to specific needs rather than relying solely on nested routers.

**Pros:**

- Greater control over URL structure.
- Can cater to complex or unique routing requirements.

**Cons:**

- More manual configuration.
- Potential for inconsistency.

### **d. Use DRF’s `ViewSet` Without Routers**

Manually map URLs to ViewSet actions without using routers.

**Pros:**

- Full control over URL patterns.
- Avoids potential complexities of nested routers.

**Cons:**

- Increased boilerplate code.
- Potentially less scalable for large APIs.

------------------------------------------------------------------------

Back to top
