# Views

Views

Author

Benedict Thekkel

## Types of Views in Django

### 1. Function-Based Views (FBVs):

- Simple functions that take a request and return a response.
- Use decorators like @api_view to specify allowed methods (GET, POST, etc.).

``` python
from rest_framework.decorators import api_view
from rest_framework.response import Response

@api_view(['GET', 'POST'])
def my_view(request):
    if request.method == 'GET':
        data = {"message": "Hello, world!"}
        return Response(data)
    elif request.method == 'POST':
        data = request.data
        return Response(data)
```

### 2. Class-Based Views (CBVs):

- Inherit from Django’s View or DRF’s APIView.
- Provide more structure and functionality.

``` python
from rest_framework.views import APIView
from rest_framework.response import Response

class MyView(APIView):
    def get(self, request):
        data = {"message": "Hello, world!"}
        return Response(data)

    def post(self, request):
        data = request.data
        return Response(data)
```

## Built-in Generic Views

Django provides a set of built-in generic views that handle common patterns. Examples include: - `ListView`: Display a list of objects. - `DetailView`: Display a single object. - `CreateView`: Display a form for creating a new object. - `UpdateView`: Display a form for updating an existing object. - `DeleteView`: Display a confirmation page for deleting an object.

### Generic Views

- Simplify common patterns (CRUD operations) by providing pre-built classes.
- Example: ListAPIView, CreateAPIView, RetrieveAPIView, UpdateAPIView, DestroyAPIView

``` python
from rest_framework.generics import ListCreateAPIView
from .models import MyModel
from .serializers import MyModelSerializer

class MyModelListCreateView(ListCreateAPIView):
    queryset = MyModel.objects.all()
    serializer_class = MyModelSerializer
```

## ViewSets

- Combine logic for multiple views in a single class.
- Automatically create URLs for CRUD operations using a Router.

------------------------------------------------------------------------

``` python
from rest_framework import viewsets
from .models import MyModel
from .serializers import MyModelSerializer

class MyModelViewSet(viewsets.ModelViewSet):
    queryset = MyModel.objects.all()
    serializer_class = MyModelSerializer
```

### Common Actions in Viewsets

Viewsets provide a variety of standard actions:

1.  **list**: Retrieves a collection of objects.

    ``` python
    GET /products/
    ```

2.  **retrieve**: Retrieves a single object based on its primary key.

    ``` python
    GET /products/{id}/
    ```

3.  **create**: Creates a new object.

    ``` python
    POST /products/
    ```

4.  **update**: Updates an existing object (entire object).

    ``` python
    PUT /products/{id}/
    ```

5.  **partial_update**: Partially updates an object (only changes specific fields).

    ``` python
    PATCH /products/{id}/
    ```

6.  **destroy**: Deletes an object.

    ``` python
    DELETE /products/{id}/
    ```

### Routing for Viewsets

Instead of manually defining URLs, DRF provides **routers** that automatically map viewsets to URL patterns.

#### Step 1: Use `DefaultRouter` to Automate URL Routing

``` python
from rest_framework.routers import DefaultRouter
from .views import ProductViewSet

router = DefaultRouter()
router.register(r'products', ProductViewSet, basename='product')

urlpatterns = router.urls
```

#### Resulting Routes:

- `GET /products/` → `list`
- `GET /products/{id}/` → `retrieve`
- `POST /products/` → `create`
- `PUT /products/{id}/` → `update`
- `PATCH /products/{id}/` → `partial_update`
- `DELETE /products/{id}/` → `destroy`

**Customizing URL Paths**: If you need to override or customize the URLs, you can use `SimpleRouter` or `register` with custom prefixes or basenames.

------------------------------------------------------------------------

### Custom Actions in Viewsets

Viewsets allow you to define **custom actions** that aren’t covered by the default CRUD operations using the `@action` decorator.

#### Example of Custom Action:

``` python
from rest_framework.decorators import action
from rest_framework.response import Response
from rest_framework import viewsets

class ProductViewSet(viewsets.ModelViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer

    # Custom action that returns products on sale
    @action(detail=False, methods=['get'])
    def on_sale(self, request):
        products_on_sale = self.queryset.filter(on_sale=True)
        serializer = self.get_serializer(products_on_sale, many=True)
        return Response(serializer.data)
```

**Key Features of `@action`**: - **`detail=False`**: Means this is a collection-level action (works on the entire queryset, e.g., `GET /products/on_sale/`). - **`detail=True`**: For object-level actions (works on a single object, e.g., `GET /products/{id}/related/`).

You can also specify the HTTP methods allowed for the action (e.g., `methods=['post']` for a POST request).

------------------------------------------------------------------------

### Mixins in Viewsets

> Mixins provide reusable behavior for viewsets, allowing you to include only the actions you need.

#### Common Mixins:

- **ListModelMixin**: Adds the `list()` action (returns a collection of objects).
- **CreateModelMixin**: Adds the `create()` action.
- **RetrieveModelMixin**: Adds the `retrieve()` action (retrieves a single object).
- **UpdateModelMixin**: Adds the `update()` action (updates a single object).
- **DestroyModelMixin**: Adds the `destroy()` action (deletes a single object).

#### Example Using Mixins:

``` python
from rest_framework import viewsets, mixins
from .models import Product
from .serializers import ProductSerializer

class ProductViewSet(mixins.ListModelMixin,
                     mixins.RetrieveModelMixin,
                     viewsets.GenericViewSet):
    queryset = Product.objects.all()
    serializer_class = ProductSerializer
```

In this example: - Only `list` and `retrieve` actions are available. - You can mix and match to implement only the behaviors needed for your viewset.

------------------------------------------------------------------------

> You can combine mixins with GenericAPIView for custom behavior:

``` python
from rest_framework import mixins, generics
from .models import MyModel
from .serializers import MyModelSerializer

class MyModelView(mixins.ListModelMixin, mixins.CreateModelMixin, generics.GenericAPIView):
    queryset = MyModel.objects.all()
    serializer_class = MyModelSerializer

    def get(self, request, *args, **kwargs):
        return self.list(request, *args, **kwargs)

    def post(self, request, *args, **kwargs):
        return self.create(request, *args, **kwargs)
```

## URL Routing

DRF’s routers automatically generate URL patterns for ViewSets. Common routers include:

- SimpleRouter: Basic router for CRUD operations.
- DefaultRouter: Extends SimpleRouter with additional functionality like a default API root view.

------------------------------------------------------------------------

``` python
from rest_framework.routers import DefaultRouter
from .views import MyModelViewSet

router = DefaultRouter()
router.register(r'mymodel', MyModelViewSet)

urlpatterns = [
    path('', include(router.urls)),
]
```

## Customizing Views

### Permissions

- Control access to views.
- Use permission_classes attribute or override get_permissions method.

``` python
from rest_framework.permissions import IsAuthenticated

class MyView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        data = {"message": "Hello, authenticated user!"}
        return Response(data)
```

### Throttling

- Limit the rate of requests.
- Use throttle_classes attribute or override get_throttles method.

``` python
from rest_framework.throttling import UserRateThrottle

class MyView(APIView):
    throttle_classes = [UserRateThrottle]

    def get(self, request):
        data = {"message": "Hello, throttled user!"}
        return Response(data)
```

### Filtering, Searching, and Ordering

- Use DjangoFilterBackend, SearchFilter, and OrderingFilter for filtering, searching, and ordering querysets.

``` python
from rest_framework import filters
from django_filters.rest_framework import DjangoFilterBackend

class MyModelListCreateView(ListCreateAPIView):
    queryset = MyModel.objects.all()
    serializer_class = MyModelSerializer
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_fields = ['field1', 'field2']
    search_fields = ['field1', 'field2']
    ordering_fields = ['field1', 'field2']
```

Back to top
