# Django For Recovery Metrics

Django For Recovery Metrics

Author

Benedict Thekkel

## [Install Poetry](https://bthek1.github.io/Web_development/Environments/poetry.html)

> Add to bash

``` sh
export PATH="$HOME/.local/bin:$PATH"
```

> Reload bash

``` sh
source ~/.bashrc 
```

## [Install direnv](https://bthek1.github.io/Web_development/Environments/direnv.html)

## Add libraries

``` sh
poetry add django
poetry add djangorestframework
poetry add pygments  
poetry add django-filter
poetry add --dev pytest
```

## Create Default Project

``` sh
django-admin startproject RM_django
cd RM_django
python manage.py startapp Users
```

> add model code

``` sh
poetry run python manage.py makemigrations
poetry run python manage.py migrate
```

> run server

``` sh
python manage.py runserver
```

### Workflows

``` yml
name: Django CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: '3.12'


    - name: Install Poetry
      run: |
        curl -sSL https://install.python-poetry.org | python3 -
        export PATH="$HOME/.local/bin:$PATH"
    - name: Install dependencies
      run: poetry install
    - name: List files in the repository
      run: |
        ls ${{ github.workspace }}
    - name: Run migrations
      run: poetry run python RM_Django/manage.py migrate
    - name: Run tests
      run: poetry run python RM_Django/manage.py test
```

## DjangoX - postgresql

1.  Use djanogx template to create repo
2.  Setup poetry
3.  add .envrc file for direnv
4.  add .env

### Setup Poetry

> add required libraries into poetry env

``` sh
poetry add $(cat requirements.txt | awk '{print $1}')
```

### [Setup postgresql using .env](https://bthek1.github.io/Web_development/Back_end/Django/secrets.html)

``` sql
sudo -u postgres createuser --interactive
sudo -u postgres createdb rm_db
sudo -u postgres psql
ALTER USER x_user WITH PASSWORD 'x_password';
GRANT ALL PRIVILEGES ON DATABASE x_db TO x_user;
\q
```

#### db check:

``` bash
# load env vars with export if req'd
psql -d $DB_NAME -U $DB_USER -W
```

``` sh
python manage.py startapp base
```

``` sh
python manage.py startapp questionnaire
```

> Add to settings -\> Installed Apps

``` python
INSTALLED_APPS = [
    "rest_framework",
    # Local
    "accounts",
    "pages",
    "base",
    "questionnaire",
]
```

``` python
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',  # Use session authentication
        'rest_framework.authentication.TokenAuthentication',    # Use token authentication
        # Add other authentication classes as needed
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',  # Ensure authenticated users have access
    ],
}
```

> add model code

``` sh
poetry run python manage.py makemigrations base
poetry run python manage.py makemigrations questionniare
poetry run python manage.py migrate
```

> run server

``` sh
python manage.py runserver
```

### Full Prompt

Back to top
