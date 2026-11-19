# Pytesting

Testing

Author

Benedict Thekkel

## Setup

### Installation

``` bash
pip install pytest pytest-django
```

#### Configuration (pytest.ini)

``` ini
[pytest]
DJANGO_SETTINGS_MODULE = your_project.settings
python_files = tests.py test_*.py *_tests.py
filterwarnings =
    ignore::DeprecationWarning
    ignore::django.utils.deprecation.RemovedInDjango50Warning
```

or \#### pyprojecrt.toml file

``` toml
[tool.pytest.ini_options]
django_settings_module = "myproject.settings"  # Replace 'myproject' with your actual project name
python_files = ["tests.py", "test_*.py", "*_tests.py"]
addopts = "--reuse-db --tb=short -p no:warnings"
```

------------------------------------------------------------------------

## Basic Concepts

### Test File Structure

``` python
# tests/
#   └── test_views.py
#   └── test_models.py
#   └── test_forms.py
#   └── test_api.py
#   └── conftest.py  # shared fixtures
```

### Simple Test Example

``` python
def test_homepage_status(client):
    response = client.get('/')
    assert response.status_code == 200
```

------------------------------------------------------------------------

## Fixtures

### Basic Fixtures

``` python
import pytest
from django.contrib.auth.models import User

@pytest.fixture
def user_data():
    return {
        'username': 'testuser',
        'password': 'testpass123',
        'email': 'test@example.com'
    }

@pytest.fixture
def user(db, user_data):
    return User.objects.create_user(**user_data)

@pytest.fixture
def admin_user(db):
    return User.objects.create_superuser(
        username='admin',
        password='admin123',
        email='admin@example.com'
    )
```

### Factory Boy Integration

``` python
import factory
from myapp.models import Profile

class UserFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = User
    
    username = factory.Sequence(lambda n: f'user{n}')
    email = factory.LazyAttribute(lambda o: f'{o.username}@example.com')

class ProfileFactory(factory.django.DjangoModelFactory):
    class Meta:
        model = Profile
    
    user = factory.SubFactory(UserFactory)
    bio = factory.Faker('text')

@pytest.fixture
def user_with_profile():
    return ProfileFactory()
```

------------------------------------------------------------------------

## Database Testing

### Basic Model Testing

``` python
@pytest.mark.django_db
def test_create_user(user_data):
    user = User.objects.create_user(**user_data)
    assert User.objects.count() == 1
    assert user.username == user_data['username']

@pytest.mark.django_db
def test_profile_creation():
    user = UserFactory()
    profile = ProfileFactory(user=user)
    assert profile.user == user
```

### Query Testing

``` python
@pytest.mark.django_db
class TestUserQueries:
    def test_user_filter(self):
        UserFactory.create_batch(3)
        assert User.objects.count() == 3
        assert User.objects.filter(username__startswith='user').count() == 3

    def test_user_order(self):
        users = UserFactory.create_batch(3)
        ordered = User.objects.order_by('-date_joined')
        assert list(ordered) == sorted(users, key=lambda x: x.date_joined, reverse=True)
```

------------------------------------------------------------------------

## Client Testing

### URL Testing

``` python
def test_homepage(client):
    response = client.get('/')
    assert response.status_code == 200
    assert 'Welcome' in str(response.content)

def test_protected_view(client, user):
    client.force_login(user)
    response = client.get('/protected/')
    assert response.status_code == 200

def test_post_request(client):
    response = client.post('/submit/', {
        'title': 'Test',
        'content': 'Content'
    })
    assert response.status_code == 302  # redirect after success
```

### Template Testing

``` python
def test_template_rendering(client):
    response = client.get('/profile/')
    assert 'profile.html' in [t.name for t in response.templates]
    assert 'Profile Page' in str(response.content)
```

------------------------------------------------------------------------

## Authentication Testing

### Login Testing

``` python
@pytest.mark.django_db
class TestAuth:
    def test_login(self, client, user, user_data):
        response = client.post('/login/', {
            'username': user_data['username'],
            'password': user_data['password']
        })
        assert response.status_code == 302
        assert '_auth_user_id' in client.session

    def test_logout(self, client, user):
        client.force_login(user)
        response = client.get('/logout/')
        assert '_auth_user_id' not in client.session
```

### Permission Testing

``` python
from django.contrib.auth.models import Permission

@pytest.mark.django_db
def test_user_permissions(user):
    permission = Permission.objects.get(codename='add_user')
    user.user_permissions.add(permission)
    assert user.has_perm('auth.add_user')
```

------------------------------------------------------------------------

## Form Testing

### Form Validation

``` python
from myapp.forms import UserProfileForm

def test_valid_form():
    form = UserProfileForm(data={
        'name': 'John Doe',
        'email': 'john@example.com',
        'bio': 'Test bio'
    })
    assert form.is_valid()

def test_invalid_form():
    form = UserProfileForm(data={})
    assert not form.is_valid()
    assert 'name' in form.errors
```

### File Upload Testing

``` python
import tempfile
from django.core.files.uploadedfile import SimpleUploadedFile

def test_file_upload(client, user):
    client.force_login(user)
    with tempfile.NamedTemporaryFile() as tmp:
        tmp.write(b'test content')
        tmp.seek(0)
        response = client.post('/upload/', {
            'file': SimpleUploadedFile(tmp.name, tmp.read())
        })
    assert response.status_code == 302
```

------------------------------------------------------------------------

## API Testing

### REST Framework Testing

``` python
from rest_framework.test import APIClient
import pytest

@pytest.fixture
def api_client():
    return APIClient()

@pytest.mark.django_db
class TestUserAPI:
    def test_list_users(self, api_client, admin_user):
        api_client.force_authenticate(admin_user)
        response = api_client.get('/api/users/')
        assert response.status_code == 200
        assert len(response.json()) > 0

    def test_create_user(self, api_client, admin_user):
        api_client.force_authenticate(admin_user)
        response = api_client.post('/api/users/', {
            'username': 'newuser',
            'email': 'new@example.com',
            'password': 'secret123'
        })
        assert response.status_code == 201
```

------------------------------------------------------------------------

## Mocking

### Basic Mocking

``` python
from unittest.mock import patch

def test_external_api_call():
    with patch('requests.get') as mock_get:
        mock_get.return_value.status_code = 200
        mock_get.return_value.json.return_value = {'data': 'test'}
        # Test your function that uses requests.get
        assert your_function() == expected_result

@patch('myapp.services.external_api.make_request')
def test_service(mock_request):
    mock_request.return_value = {'status': 'success'}
    # Test your service
```

### Email Mocking

``` python
from django.core import mail

def test_send_email(client):
    response = client.post('/send-email/')
    assert len(mail.outbox) == 1
    assert mail.outbox[0].subject == 'Expected Subject'
```

------------------------------------------------------------------------

## Best Practices

### 1. Use Fixtures Effectively

- Keep fixtures focused and small
- Use factory boy for complex object creation
- Share fixtures in conftest.py

### 2. Test Organization

``` python
@pytest.mark.django_db
class TestUser:
    """Group related tests in classes"""
    
    def test_create(self):
        # test user creation
        pass
    
    def test_update(self):
        # test user update
        pass
```

### 3. Parametrize Tests

``` python
@pytest.mark.parametrize('username,expected', [
    ('valid_user', True),
    ('inv@lid', False),
    ('', False),
])
def test_username_validation(username, expected):
    form = UserForm(data={'username': username})
    assert form.is_valid() == expected
```

### 4. Use Markers

``` python
@pytest.mark.slow
def test_expensive_operation():
    # long running test
    pass

# Run with: pytest -m "not slow"
```

### 5. Debug Tips

``` python
def test_with_debug(client):
    response = client.get('/view/')
    import pdb; pdb.set_trace()  # Debug point
    # or use pytest --pdb
```

### 6. Coverage

``` bash
pytest --cov=myapp
pytest --cov=myapp --cov-report=html
```

### 7. Configuration Best Practices

``` python
# conftest.py
import pytest
from django.conf import settings

@pytest.fixture(autouse=True)
def media_storage(settings, tmpdir):
    settings.MEDIA_ROOT = tmpdir.strpath

@pytest.fixture
def enable_debug(settings):
    settings.DEBUG = True
```

Remember to: - Write tests first (TDD when possible) - Keep tests simple and focused - Use meaningful test names - Test edge cases and error conditions - Use appropriate assertions - Keep test data minimal - Clean up after tests - Use continuous integration

------------------------------------------------------------------------

## Common Testing Scenarios

### 1. Testing Signals

``` python
@pytest.mark.django_db
def test_profile_signal():
    user = User.objects.create_user(username='test')
    assert hasattr(user, 'profile')
    assert user.profile is not None
```

### 2. Testing Management Commands

``` python
from django.core.management import call_command
from io import StringIO

def test_command_output():
    out = StringIO()
    call_command('my_command', stdout=out)
    assert 'Expected output' in out.getvalue()
```

### 3. Testing Custom Template Tags

``` python
from django.template import Template, Context

def test_custom_tag():
    template = Template('{% load custom_tags %}{{ "test"|my_filter }}')
    context = Context({})
    assert template.render(context) == 'expected result'
```

### 4. Testing Middlewares

``` python
def test_middleware(client):
    response = client.get('/')
    assert response['Custom-Header'] == 'Expected Value'
```

### 5. Testing Admin

``` python
from django.contrib.admin.sites import AdminSite
from myapp.admin import UserAdmin
from myapp.models import User

@pytest.mark.django_db
def test_admin_view(admin_client):
    response = admin_client.get('/admin/myapp/user/')
    assert response.status_code == 200

def test_admin_action():
    site = AdminSite()
    user_admin = UserAdmin(User, site)
    # Test admin actions
```

------------------------------------------------------------------------

Back to top
