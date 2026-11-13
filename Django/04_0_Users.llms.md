# User Models

User Models

Author

Benedict Thekkel

## Default User Model

Django comes with a built-in user model, `User`, which is located in the `django.contrib.auth.models` module. This model provides several fields and methods to manage users, such as:

- `username`
- `first_name`
- `last_name`
- `email`
- `password`
- `is_staff`
- `is_active`
- `is_superuser`
- `last_login`
- `date_joined`

### Creating a Superuser

To create an admin/superuser, use the `createsuperuser` management command: \> This command prompts you to enter a username, email, and password.

``` bash
python manage.py createsuperuser

---

## Authenticating Users

> Django provides an `authenticate()` function in `django.contrib.auth` to verify user credentials:

---

```python
from django.contrib.auth import authenticate

user = authenticate(username='john', password='secret')
if user is not None:
    # A backend authenticated the credentials
else:
    # No backend authenticated the credentials
```

### Logging In Users

> To log in a user, use the `login()` function from `django.contrib.auth`

``` python
from django.contrib.auth import login

login(request, user)
```

### Logging Out Users

> To log out a user, use the `logout()` function

``` python
from django.contrib.auth import logout

logout(request)
```

## Custom User Model

> Django allows you to customize the user model by extending the base model or by creating a completely new one. This is especially useful if you need to add extra fields or change the behavior of the user model.

------------------------------------------------------------------------

### Extending the Default User Model

> If you only need to add a few extra fields to the existing user model, you can extend `AbstractUser`

``` python
from django.contrib.auth.models import AbstractUser

class CustomUser(AbstractUser):
    bio = models.TextField(max_length=500, blank=True)
```

### Creating a Custom User Model

> If you need more control over the user model, you can create a custom user model by extending `AbstractBaseUser` and `PermissionsMixin`:

- `UserManager`: djangos default user manager

- `BaseUserManager`: custom

- `AbstractUser`: djangos default user model

- `AbstractBaseUser`: custom

- `PermissionsMixin`: all the methods for django permission model

``` python
from django.contrib.auth.models import AbstractBaseUser, BaseUserManager, PermissionsMixin
from django.db import models

class CustomUserManager(BaseUserManager):
    def create_user(self, email, password=None, **extra_fields):
        if not email:
            raise ValueError('The Email field must be set')
        if not password:
            raise ValueError('The Password field must be set')
        email = self.normalize_email(email)
        user = self.model(email=email, **extra_fields)
        user.set_password(password)
        user.save(using=self._db)
        return user

    def create_superuser(self, email, password=None, **extra_fields):
        extra_fields.setdefault('is_staff', True)
        extra_fields.setdefault('is_superuser', True)

        if extra_fields.get('is_staff') is not True:
            raise ValueError('Superuser must have is_staff=True.')
        if extra_fields.get('is_superuser') is not True:
            raise ValueError('Superuser must have is_superuser=True.')

        return self.create_user(email, password, **extra_fields)

class CustomUser(AbstractBaseUser, PermissionsMixin):
    email = models.EmailField(unique=True)
    first_name = models.CharField(max_length=30, blank=True)
    last_name = models.CharField(max_length=30, blank=True)
    date_joined = models.DateTimeField(auto_now_add=True)
    is_active = models.BooleanField(default=True)
    is_staff = models.BooleanField(default=False)

    objects = CustomUserManager()

    USERNAME_FIELD = 'email'
    REQUIRED_FIELDS = ['first_name', 'last_name']

    def __str__(self):
        return self.email
```

### Using the Custom User Model

> To use the custom user model, update the `AUTH_USER_MODEL` setting in your `settings.py`

``` python
AUTH_USER_MODEL = 'yourapp.CustomUser'
```

### Migrating to a Custom User Model

Switching to a custom user model after running migrate can be complex. It’s generally recommended to set up the custom user model at the start of your project. If you need to switch to a custom user model later, you may need to: - Create a new model. - Write custom migration scripts to handle the transition. - Update all references to the old user model.

### User Managers

Custom user models often require a custom manager. The manager is responsible for creating users and superusers.

- `create_user()`: Method to create a regular user.
- `create_superuser()`: Method to create a superuser.

The custom manager should inherit from `BaseUserManager`.

### Example of Custom User Manager

``` python
class CustomUserManager(BaseUserManager):
    def create_user(self, email, password=None, **extra_fields):
        # Implementation here...

    def create_superuser(self, email, password=None, **extra_fields):
        # Implementation here...
```

## User Authentication Backends

Django uses authentication backends to verify credentials. The default backend is `ModelBackend`, which authenticates against `username` and `password`.

You can create custom authentication backends by subclassing `BaseBackend`

``` python
from django.contrib.auth.backends import BaseBackend

class EmailBackend(BaseBackend):
    def authenticate(self, request, username=None, password=None, **kwargs):
        try:
            user = CustomUser.objects.get(email=username)
        except CustomUser.DoesNotExist:
            return None

        if user.check_password(password):
            return user
        return None

    def get_user(self, user_id):
        try:
            return CustomUser.objects.get(pk=user_id)
        except CustomUser.DoesNotExist:
            return None
```

> To use the custom backend, add it to `AUTHENTICATION_BACKENDS` in `settings.py`:

``` python
AUTHENTICATION_BACKENDS = ['yourapp.backends.EmailBackend']
```

## Signals

> Django signals can be used to perform actions based on user-related events, such as user creation, login, or logout.

------------------------------------------------------------------------

### Example of Using Signals

``` python
from django.contrib.auth.models import User
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=User)
def create_user_profile(sender, instance, created, **kwargs):
    if created:
        Profile.objects.create(user=instance)
```

Back to top
