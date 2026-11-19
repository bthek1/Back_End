# DRF Quickstart

DRF - Django REST framework is a powerful and flexible toolkit for building Web APIs.

Author

Benedict Thekkel

## Project setup

``` bash
# Create the project directory
mkdir tutorial
cd tutorial

# Create a virtual environment to isolate our package dependencies locally
python3 -m venv env
source env/bin/activate  # On Windows use `env\Scripts\activate`

# Install Django and Django REST framework into the virtual environment
pip install djangorestframework

# Set up a new project with a single application
django-admin startproject tutorial .  # Note the trailing '.' character
cd tutorial
django-admin startapp quickstart
cd ..
```

> Now sync your database for the first time:

``` bash
python manage.py migrate
```

> We’ll also create an initial user named admin with a password. We’ll authenticate as that user later in our example.

``` bash
python manage.py createsuperuser --username bthek1 --email admin@gmail.com
```

## Serializers

> First up we’re going to define some serializers. Let’s create a new module named `tutorial/quickstart/serializers.py` that we’ll use for our data representations.

> Notice that we’re using hyperlinked relations in this case with `HyperlinkedModelSerializer`. You can also use primary key and various other relationships, but hyperlinking is good RESTful design.

------------------------------------------------------------------------

``` bash
from django.contrib.auth.models import Group, User
from rest_framework import serializers


class UserSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = User
        fields = ['url', 'username', 'email', 'groups']


class GroupSerializer(serializers.HyperlinkedModelSerializer):
    class Meta:
        model = Group
        fields = ['url', 'name']
```

## Views

``` bash
from django.contrib.auth.models import Group, User
from rest_framework import permissions, viewsets

from tutorial.quickstart.serializers import GroupSerializer, UserSerializer


class UserViewSet(viewsets.ModelViewSet):
    """
    API endpoint that allows users to be viewed or edited.
    """
    queryset = User.objects.all().order_by('-date_joined')
    serializer_class = UserSerializer
    permission_classes = [permissions.IsAuthenticated]


class GroupViewSet(viewsets.ModelViewSet):
    """
    API endpoint that allows groups to be viewed or edited.
    """
    queryset = Group.objects.all().order_by('name')
    serializer_class = GroupSerializer
    permission_classes = [permissions.IsAuthenticated]
```

## URLs

> Because we’re using viewsets instead of views, we can automatically generate the URL conf for our API, by simply registering the viewsets with a router class.

> Again, if we need more control over the API URLs we can simply drop down to using regular class-based views, and writing the URL conf explicitly.

> Finally, we’re including default login and logout views for use with the browsable API. That’s optional, but useful if your API requires authentication and you want to use the browsable API.

------------------------------------------------------------------------

``` bash
from django.urls import include, path
from rest_framework import routers

from tutorial.quickstart import views

router = routers.DefaultRouter()
router.register(r'users', views.UserViewSet)
router.register(r'groups', views.GroupViewSet)

# Wire up our API using automatic URL routing.
# Additionally, we include login URLs for the browsable API.
urlpatterns = [
    path('', include(router.urls)),
    path('api-auth/', include('rest_framework.urls', namespace='rest_framework'))
]
```

## Pagination

> Pagination allows you to control how many objects per page are returned. To enable it add the following lines to `tutorial/settings.py`

------------------------------------------------------------------------

``` python
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10
}
```

## Settings

Add `rest_framework` to your `INSTALLED_APPS` setting.

``` python
INSTALLED_APPS = [
    ...
    'rest_framework',
]
```

## Testing our API

``` python
python manage.py runserver
```

> We can now access our API, both from the command-line, using tools like `curl...`

``` bash
bash: curl -u admin -H 'Accept: application/json; indent=4' http://127.0.0.1:8000/users/
Enter host password for user 'admin':
{
    "count": 1,
    "next": null,
    "previous": null,
    "results": [
        {
            "url": "http://127.0.0.1:8000/users/1/",
            "username": "admin",
            "email": "admin@example.com",
            "groups": []
        }
    ]
}
```

> Or using the httpie, command line tool…

``` bash
bash: http -a admin http://127.0.0.1:8000/users/
http: password for admin@127.0.0.1:8000:: 
$HTTP/1.1 200 OK
...
{
    "count": 1,
    "next": null,
    "previous": null,
    "results": [
        {
            "email": "admin@example.com",
            "groups": [],
            "url": "http://127.0.0.1:8000/users/1/",
            "username": "admin"
        }
    ]
}
```

Back to top
