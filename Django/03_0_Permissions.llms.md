# Permissions

Django REST Framework (DRF) provides a comprehensive system for handling permissions to ensure that API endpoints are accessed and modified only by users with the appropriate rights.

Author

Benedict Thekkel

## Built-in Permission Classes

> DRF comes with several built-in permission classes that can be used to restrict access to your API views:

------------------------------------------------------------------------

- AllowAny: Grants access to any user, whether authenticated or not. It is the default permission class.
- IsAuthenticated: Grants access only to authenticated users.
- IsAdminUser: Grants access only to users with admin status.
- IsAuthenticatedOrReadOnly: Grants read-only access to unauthenticated users and full access to authenticated users.

## Using Permission Classes

> You can set permission classes at the view level or globally in your settings.py file.

------------------------------------------------------------------------

### View Level

``` python
from rest_framework.permissions import IsAuthenticated
from rest_framework.views import APIView

class MyView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        return Response({"message": "Hello, world!"})
```

### Global Level

``` python
# settings.py
REST_FRAMEWORK = {
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}
```

## Custom Permission Classes

> DRF allows you to define custom permission classes by subclassing BasePermission. Custom permissions can be used to implement more complex logic.

------------------------------------------------------------------------

#### Example: Custom Permission Class

``` python
from rest_framework.permissions import BasePermission

class IsOwner(BasePermission):
    def has_object_permission(self, request, view, obj):
        return obj.owner == request.user
```

## Permission Methods

Custom permissions override two main methods: - `has_permission(self, request, view)`: Checks permissions at the view level. - `has_object_permission(self, request, view, obj)`: Checks permissions at the object level.

#### Example: Custom Permission Methods

``` python
class IsOwner(BasePermission):
    def has_permission(self, request, view):
        # Custom logic for view level permission
        return True

    def has_object_permission(self, request, view, obj):
        # Custom logic for object level permission
        return obj.owner == request.user
```

## Combining Permissions

> You can combine multiple permission classes to apply multiple layers of permission checks.

------------------------------------------------------------------------

``` python
from rest_framework.permissions import IsAuthenticated, IsAdminUser

class MyView(APIView):
    permission_classes = [IsAuthenticated, IsAdminUser]

    def get(self, request):
        return Response({"message": "Hello, world!"})
```

## Third-Party Permissions

DRF integrates with several third-party libraries to extend the default permissions system. For example:

- django-guardian: Provides per-object permissions.
- django-rules: Allows for rule-based permissions.

## Best Practices

1.  Least Privilege Principle: Only give users the minimum permissions they need to perform their tasks.
2.  Reuse Custom Permissions: Create custom permissions for reusable and consistent permission logic across views.
3.  Object-Level Permissions: Use object-level permissions to fine-tune access control, especially for models with user-specific data.

## Advanced Permissions

> For advanced use cases, you can create permissions that depend on request data, view arguments, or any other contextual information.

------------------------------------------------------------------------

> Example: Permission Based on Query Parameter

``` python
class QueryParameterPermission(BasePermission):
    def has_permission(self, request, view):
        return 'allow' in request.query_params
```

Back to top
