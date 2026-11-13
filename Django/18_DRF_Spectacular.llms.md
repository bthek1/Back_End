# DRF Spectacular

**DRF Spectacular** is a powerful tool for generating **OpenAPI 3.0-compliant API schemas** for Django REST Framework (DRF) projects. It automates the generation of the API documentation, making it easy to integrate DRF projects with tools like **Swagger UI**, **ReDoc**, and **Postman**. It focuses on compliance with OpenAPI standards and allows for significant customization and extensibility.

Author

Benedict Thekkel

### 1. **What is DRF Spectacular?**

**DRF Spectacular** is a schema generation tool for Django REST Framework (DRF) that creates OpenAPI 3.0-compliant API documentation. It provides an out-of-the-box solution for generating and maintaining API documentation with minimal configuration, and it integrates well with DRF’s serializers, views, and routers.

#### Key Features:

- **OpenAPI 3.0 Support**: Full OpenAPI 3.0-compliant schema generation.
- **Schema Generation**: Automatically generates schema documentation for all your API endpoints.
- **Customization**: Fine-grained customization options for views, endpoints, and fields.
- **Swagger & ReDoc Integration**: Easily integrates with Swagger UI and ReDoc for interactive documentation.
- **Component Reuse**: Reuses schema components (e.g., serializers, response formats) to reduce duplication in the schema.

### 2. **Installation and Setup**

To use DRF Spectacular, you’ll need to install the package and configure it in your Django project.

#### a. **Install DRF Spectacular**

You can install DRF Spectacular using `pip`:

``` bash
pip install drf-spectacular
```

#### b. **Add Spectacular to Installed Apps**

In your `settings.py`, add `drf_spectacular` to the `INSTALLED_APPS`:

``` python
INSTALLED_APPS = [
    # Other apps
    'drf_spectacular',
]
```

#### c. **Configure DRF Settings**

Add DRF Spectacular to the `DEFAULT_SCHEMA_CLASS` setting in Django REST Framework’s settings:

``` python
REST_FRAMEWORK = {
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}
```

This tells DRF to use DRF Spectacular’s schema generator.

#### d. **Add URLs for Schema and UI**

You’ll need to add URLs for the OpenAPI schema and the documentation UI in your `urls.py`:

``` python
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView, SpectacularRedocView

urlpatterns = [
    # API schema and documentation views
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    path('api/schema/swagger-ui/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    path('api/schema/redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),
]
```

Now, the following URLs will be available in your Django app: - `/api/schema/` – OpenAPI schema in JSON format. - `/api/schema/swagger-ui/` – Interactive API documentation using Swagger UI. - `/api/schema/redoc/` – Interactive API documentation using ReDoc.

### 3. **Basic Usage of DRF Spectacular**

Once you have DRF Spectacular set up, it will automatically generate an OpenAPI 3.0-compliant schema for your API endpoints based on your DRF views, serializers, and routers.

#### a. **Default Schema Generation**

DRF Spectacular will introspect your DRF views and serializers to automatically generate the OpenAPI schema.

For example, if you have a DRF `ViewSet`:

``` python
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

And the following router setup in `urls.py`:

``` python
from rest_framework.routers import DefaultRouter

router = DefaultRouter()
router.register(r'books', BookViewSet)

urlpatterns = [
    path('api/', include(router.urls)),
]
```

The generated schema will automatically include the `BookViewSet` with all its CRUD operations.

#### b. **Schema Viewing**

- **Swagger UI**: Visit `/api/schema/swagger-ui/` to view the Swagger UI for the API.
- **ReDoc**: Visit `/api/schema/redoc/` to view the ReDoc documentation.

These interactive docs allow you to visualize and test your API endpoints.

### 4. **Customizing the Schema**

DRF Spectacular offers a wide range of customization options to fine-tune the generated schema. You can customize global settings, individual views, serializers, and fields.

#### a. **Global Settings**

In your `settings.py`, you can customize DRF Spectacular’s behavior with the `SPECTACULAR_SETTINGS` dictionary.

Example:

``` python
SPECTACULAR_SETTINGS = {
    'TITLE': 'My API',
    'DESCRIPTION': 'This is the API documentation for my Django project.',
    'VERSION': '1.0.0',
    'SERVE_INCLUDE_SCHEMA': False,
    'COMPONENT_SPLIT_REQUEST': True,  # Separate request and response serializers
    'SCHEMA_PATH_PREFIX': '/api/v1',
    'SERVERS': [
        {'url': 'https://api.example.com/v1', 'description': 'Production Server'},
        {'url': 'https://staging.example.com/v1', 'description': 'Staging Server'},
    ],
}
```

- **`TITLE`**: The title of the API (used in the documentation UI).
- **`DESCRIPTION`**: A description of the API (used in the documentation UI).
- **`VERSION`**: The version of the API (used in the schema).
- **`SERVERS`**: Allows you to define multiple server URLs (e.g., production, staging).
- **`COMPONENT_SPLIT_REQUEST`**: Separates request and response serializers in the schema.
- **`SCHEMA_PATH_PREFIX`**: Specifies the API path prefix to exclude common URL patterns.

#### b. **Per-View Customization**

You can customize the schema for individual views using DRF Spectacular’s `@extend_schema` decorator. This allows you to override or specify certain aspects of the schema for a particular view.

Example:

``` python
from drf_spectacular.utils import extend_schema
from rest_framework import generics
from .models import Book
from .serializers import BookSerializer

class BookListView(generics.ListCreateAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer

    @extend_schema(
        description='List all books or create a new one',
        summary='Books List',
        responses={200: BookSerializer(many=True)},
    )
    def get(self, request, *args, **kwargs):
        return super().get(request, *args, **kwargs)
```

This allows you to: - **Set custom descriptions**: Provide detailed descriptions for each endpoint. - **Define response types**: Specify the response format, such as lists of serializers or custom response messages. - **Add summary**: Provide short summaries for specific views.

#### c. **Custom Serializer Components**

DRF Spectacular can automatically generate schema components from your serializers. However, you can customize how serializers are represented using the `@extend_schema_serializer` decorator.

Example:

``` python
from drf_spectacular.utils import extend_schema_serializer

@extend_schema_serializer(
    exclude_fields=['internal_field'],  # Exclude certain fields from the schema
)
class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = '__all__'
```

This allows you to: - **Exclude specific fields**: Hide sensitive fields from the schema. - **Add custom examples**: Provide examples of how the data looks in requests and responses.

#### d. **Field-Level Customization**

At the field level, you can provide custom OpenAPI parameters using `@extend_schema_field`.

Example:

``` python
from drf_spectacular.utils import extend_schema_field

class CustomSerializer(serializers.Serializer):
    field = serializers.CharField()

    @extend_schema_field(OpenApiTypes.STR)
    def get_custom_field(self, obj):
        return "custom value"
```

### 5. **Using `@extend_schema` for Fine-Grained Control**

The `@extend_schema` decorator provides fine-grained control over each view’s schema.

#### a. **Custom Parameters**

You can add or override parameters in the schema, such as query parameters or headers:

``` python
@extend_schema(
    parameters=[
        OpenApiParameter("search", OpenApiTypes.STR, OpenApiParameter.QUERY, description="Search for books")
    ],
)
class BookListView(generics.ListAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

#### b. **Custom Responses**

You can specify custom response types, including different status codes:

``` python
@extend_schema(
    responses={200: BookSerializer, 400: OpenApiResponse(description="Invalid request")}
)
class BookDetailView(generics.RetrieveAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

#### c. **Custom Request Bodies**

If you want to specify a custom request body that differs from the default serializer representation, you can do so:

``` python
@extend_schema(request=BookCreateSerializer)
class BookCreateView(generics.CreateAPIView):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

### 6. **OpenAPI Security Schemes**

DRF Spectacular allows you

to define security schemes, such as JWT authentication or OAuth2, for your API.

#### a. **Basic Security Example**

In `SPECTACULAR_SETTINGS`, you can define security schemes like so:

``` python
SPECTACULAR_SETTINGS = {
    'TITLE': 'My API',
    'VERSION': '1.0.0',
    'AUTHENTICATION_WHITELIST': [
        'rest_framework.authentication.BasicAuthentication',
    ],
    'SECURITY': [{'basicAuth': []}],
}
```

This adds HTTP Basic Authentication to your schema.

#### b. **JWT Authentication Example**

If you’re using JWT authentication, you can configure the security scheme for JWT tokens like this:

``` python
SPECTACULAR_SETTINGS = {
    'TITLE': 'My API',
    'VERSION': '1.0.0',
    'SECURITY': [{'bearerAuth': []}],
    'SWAGGER_UI_SETTINGS': {
        'supportedSubmitMethods': ['get', 'post', 'put', 'patch', 'delete'],
    },
}
```

In this case, `bearerAuth` will be used in your OpenAPI schema to handle JWT tokens in requests.

### 7. **Component Reuse**

DRF Spectacular reuses components wherever possible to reduce schema size and increase consistency.

#### a. **Shared Serializers**

If multiple views use the same serializer, DRF Spectacular will generate a shared component for that serializer, making the schema cleaner.

#### b. **Component Naming**

You can customize component names by using `@extend_schema_serializer` and `@extend_schema`.

Example:

``` python
@extend_schema_serializer(component_name='BookDetailSerializer')
class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = '__all__'
```

### 8. **Debugging and Troubleshooting**

If you run into issues with your schema, DRF Spectacular provides a **debugging view** to help troubleshoot.

You can enable it by visiting `/api/schema/?format=openapi` or `/api/schema/?format=debug`.

Back to top
