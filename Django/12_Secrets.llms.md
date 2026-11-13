# Secrets

Using a .env file to manage secrets in a Django project is a common practice for securely storing and accessing sensitive information. Here’s a detailed guide on how to set this up:

Author

Benedict Thekkel

## Use Environment Variables

### Create a .env File

    # .env file
    DJANGO_SECRET_KEY='your-secret-key'
    DB_NAME='your-database-name'
    DB_USER='your-database-user'
    DB_PASSWORD='your-database-password'
    DB_HOST='your-database-host'
    DB_PORT='your-database-port'

### Install django-environ

``` sh
poetry add django-environ
```

### Configure Django to Use Environment Variables

> Modify settings.py file

``` python
import environ
import os

env = environ.Env()
environ.Env.read_env(os.path.join(BASE_DIR, '.env'))

SECRET_KEY = env('DJANGO_SECRET_KEY')
DEBUG = env.bool('DEBUG', default=False)
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS', default=[])

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST', default='localhost'),
        'PORT': env('DB_PORT', default='5432'),
    }
}
```

### Add `.env` to `.gitignore`

## Use GitHub Secrets for CI/CD

### Add Secrets to GitHub

- Navigate to your repository on GitHub.
- Go to `Settings` \> `Secrets and variables` \> `Actions`.
- Add your secrets (e.g., `DJANGO_SECRET_KEY`, `DB_NAME`).

### Access Secrets in GitHub Actions

> Example workflow file

``` yaml
name: Django CI/CD

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v2

    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: '3.9'

    - name: Install dependencies
      run: |
        python -m pip install --upgrade pip
        pip install -r requirements.txt

    - name: Set environment variables
      run: echo "DJANGO_SECRET_KEY=${{ secrets.DJANGO_SECRET_KEY }}" >> $GITHUB_ENV

    - name: Run migrations
      env:
        DJANGO_SECRET_KEY: ${{ secrets.DJANGO_SECRET_KEY }}
        DB_NAME: ${{ secrets.DB_NAME }}
        DB_USER: ${{ secrets.DB_USER }}
        DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
        DB_HOST: ${{ secrets.DB_HOST }}
        DB_PORT: ${{ secrets.DB_PORT }}
      run: |
        python manage.py migrate

    - name: Run tests
      env:
        DJANGO_SECRET_KEY: ${{ secrets.DJANGO_SECRET_KEY }}
        DB_NAME: ${{ secrets.DB_NAME }}
        DB_USER: ${{ secrets.DB_USER }}
        DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
        DB_HOST: ${{ secrets.DB_HOST }}
        DB_PORT: ${{ secrets.DB_PORT }}
      run: |
        python manage.py test
```

## Use a Secret Management Service

For large projects, consider using secret management services like: - AWS Secrets Manager - Azure Key Vault - HashiCorp Vault.

## Use config.settings Structure

> For more advanced setups, you can structure your settings using the config.settings module approach to separate settings for different environments (development, production, etc.).

------------------------------------------------------------------------

### Base Settings (config/settings/base.py)

``` python
import environ
import os

env = environ.Env()
environ.Env.read_env(os.path.join(BASE_DIR, '.env'))

BASE_DIR = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))

SECRET_KEY = env('DJANGO_SECRET_KEY')
DEBUG = env.bool('DEBUG', default=False)
ALLOWED_HOSTS = env.list('ALLOWED_HOSTS', default=[])

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': env('DB_NAME'),
        'USER': env('DB_USER'),
        'PASSWORD': env('DB_PASSWORD'),
        'HOST': env('DB_HOST', default='localhost'),
        'PORT': env('DB_PORT', default='5432'),
    }
}
```

### Development Settings (config/settings/development.py)

``` python
from .base import *

DEBUG = True
ALLOWED_HOSTS = ['localhost', '127.0.0.1']
```

### Production Settings (config/settings/production.py)

``` python
from .base import *

DEBUG = False
ALLOWED_HOSTS = ['yourdomain.com']
```

### Specify Settings Module:

- Set the DJANGO_SETTINGS_MODULE environment variable to specify the settings module to use.
- For development: export DJANGO_SETTINGS_MODULE=config.settings.development
- For production: export DJANGO_SETTINGS_MODULE=config.settings.production

Back to top
