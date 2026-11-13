# Django Basics

[Python Backend](https://docs.djangoproject.com/en/5.0/)

Author

Benedict Thekkel

## Features

- Admin site
- Object-relational mapper
- Authentication
- Caching

`HTTP` : Hypertext Transfer Protocol

## Installation

``` sh
pip install django
```

``` python
!pip list | grep django
```

### Help

``` sh
django-admin
```

Code

``` python
!django-admin
```

    Type 'django-admin help <subcommand>' for help on a specific subcommand.

    Available subcommands:

    [django]
        check
        compilemessages
        createcachetable
        dbshell
        diffsettings
        dumpdata
        flush
        inspectdb
        loaddata
        makemessages
        makemigrations
        migrate
        optimizemigration
        runserver
        sendtestemail
        shell
        showmigrations
        sqlflush
        sqlmigrate
        sqlsequencereset
        squashmigrations
        startapp
        startproject
        test
        testserver
    Note that only Django core commands are listed as settings are not properly configured (error: Requested setting INSTALLED_APPS, but settings are not configured. You must either define the environment variable DJANGO_SETTINGS_MODULE or call settings.configure() before accessing settings.).

## Create a new project

``` sh
django-admin startproject <name> <location>
```

Example

``` sh
django-admin startproject test_django
```

### 1. Project Structure

    myproject/
    ├── manage.py
    └── myproject/
        ├── __init__.py
        ├── settings.py
        ├── urls.py
        └── wsgi.py

- `manage.py`: This small but mighty script serves as the gateway to various Django management commands.

- `my_project/settings.py`: As the name suggests, this file houses the settings that configure your Django project.

- `my_project/urls.py`: The URL dispatcher — encoded within urls.py—maps URLs to views. This file determines which view is displayed when a specific URL is accessed.

- `my_project/wsgi.py`: Short for Web Server Gateway Interface, wsgi.py serves as the entry point for your application when deployed on a production server. It’s the bridge connecting your application to the web server, enabling it to handle incoming requests.

- `my_project/asgi.py`: Similar to wsgi.py, asgi.py is the entry point for asynchronous web servers. It stands for Asynchronous Server Gateway Interface and facilitates the handling of asynchronous HTTP requests.

- `my_project/__init__.py`: This seemingly unassuming file holds the magic that transforms a directory into a Python package. It’s essential for organizing and importing modules across your project

### 2. Start the Development Server

``` python
!cd ../../test_django/ && python manage.py -h
```

    Type 'manage.py help <subcommand>' for help on a specific subcommand.

    Available subcommands:

    [auth]
        changepassword
        createsuperuser

    [contenttypes]
        remove_stale_contenttypes

    [django]
        check
        compilemessages
        createcachetable
        dbshell
        diffsettings
        dumpdata
        flush
        inspectdb
        loaddata
        makemessages
        makemigrations
        migrate
        optimizemigration
        sendtestemail
        shell
        showmigrations
        sqlflush
        sqlmigrate
        sqlsequencereset
        squashmigrations
        startapp
        startproject
        test
        testserver

    [sessions]
        clearsessions

    [staticfiles]
        collectstatic
        findstatic
        runserver

``` sh
python manage.py runserver
```

on http://127.0.0.1:8000/

## Creating new app

``` sh
python manage.py startapp <appname>
```

Example

``` sh
python manage.py startapp myapp
```

    my_project/
    ├── manage.py
    ├── my_project/
    │   ├── __init__.py
    │   ├── asgi.py
    │   ├─ settings.py
    │   ├─ urls.py
    │   ├── wsgi.py
    ├── my_app/
        ├── migrations/
        │   └── __init__.py
        ├── __init__.py
        ├── admin.py
        ├── apps.py
        ├── models.py
        ├── tests.py
        ├── urls.py
        └── views.py

- `models.py`: At the heart of every application lies the models.py file. This is where you define the data structures using Django’s ORM (Object-Relational Mapping). Each model class represents a table in the database. This file forms the foundation of your application’s data management.

- `views.py`: The views.py file encapsulates the logic that defines how your application interacts with users’ requests. Views handle data processing, rendering templates, and responding to actions. This file transforms user interactions into tangible responses.

- `tests.py`: Test-driven development gains momentum through the tests.py file. Here, you write unit tests to ensure your application’s components function as expected. These tests bolster the reliability and stability of your codebase.

- `admin.py`: The admin.py file isn’t just for administrators—it configures how your application’s models are presented in Django’s admin interface. This file allows administrators to manage data seamlessly.

- migrations: This directory is a blueprint of all changes in your application models.

- Other Files: Additional files might surface based on your application’s needs. For instance, forms.py houses form classes for data input, urls.py maps URLs to views, and apps.py manages application-specific configurations.

### 1. Define Model

> Edit the models.py file in your app directory (myapp/models.py) to define your database models. For example:

myapp/models.py

``` python
from django.db import models

class MyModel(models.Model):
    name = models.CharField(max_length=100)
    description = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name
```

### 2. Create Database Tables

``` bash
python manage.py makemigrations
```

``` bash
python manage.py migrate
```

### 3. Create Views

> Create views to handle HTTP requests. Edit the views.py file in your app directory (myapp/views.py). For example:

myapp/views.py

``` python
from django.shortcuts import render
from django.http import HttpResponse

def index(request):
    return HttpResponse("Hello, world!")
```

### 4. Map URLs to Views

> Edit the urls.py file in your app directory (myapp/urls.py) to map URLs to your views. For example:

myapp/urls.py

``` python
from django.urls import path
from . import views

urlpatterns = [
    path('', views.index, name='index'),
]
```

### 5. Include App URLs in Project URLs

test_django/urls.py

``` python
from django.contrib import admin
from django.urls import include, path

urlpatterns = [
    path('admin/', admin.site.urls),
    path('myapp/', include('myapp.urls')),
]
```

**Run the Development Server**

## Directory Hierarchy

> A well-organized project structure typically follows this hierarchy:

------------------------------------------------------------------------

    project_name/
    ├── manage.py
    ├── project_name/
    │   ├── __init__.py
    │   ├── asgi.py
    │   ├─ settings.py
    │   ├─ urls.py
    │   ├── wsgi.py
    ├── app1/
    ├── app2/
      ...
    ├── static/
    ├── media/
    ├── templates/

- `project_name`: The root directory of your project.

- `project_name/project_name`: This inner directory holds core project settings and configuration.

- `app1, app2`: These are the individual apps you create within the project.

- `static`: Houses static files like CSS, JavaScript, and images.

- `media`: Stores user-uploaded files.

- `templates`: Contains HTML templates.

## URL dispatcher

The path() function is passed four arguments, two required: route and view, and two optional: kwargs, and name. At this point, it’s worth reviewing what these arguments are for.

### path() argument: route

route is a string that contains a URL pattern. When processing a request, Django starts at the first pattern in urlpatterns and makes its way down the list, comparing the requested URL against each pattern until it finds one that matches.

Patterns don’t search GET and POST parameters, or the domain name. For example, in a request to https://www.example.com/myapp/, the URLconf will look for myapp/. In a request to https://www.example.com/myapp/?page=3, the URLconf will also look for myapp/.

### path() argument: view

When Django finds a matching pattern, it calls the specified view function with an HttpRequest object as the first argument and any “captured” values from the route as keyword arguments. We’ll give an example of this in a bit.

### path() argument: kwargs

Arbitrary keyword arguments can be passed in a dictionary to the target view. We aren’t going to use this feature of Django in the tutorial

### path() argument: name

Naming your URL lets you refer to it unambiguously from elsewhere in Django, especially from within templates. This powerful feature allows you to make global changes to the URL patterns of your project while only touching a single file.

When you’re comfortable with the basic request and response flow, read part 2 of this tutorial to start working with the database.

## Database settings

Now, open up mysite/settings.py. It’s a normal Python module with module-level variables representing Django settings.

By default, the configuration uses SQLite. If you’re new to databases, or you’re just interested in trying Django, this is the easiest choice. SQLite is included in Python, so you won’t need to install anything else to support your database. When starting your first real project, however, you may want to use a more scalable database like PostgreSQL, to avoid database-switching headaches down the road.

If you wish to use another database, install the appropriate database bindings and change the following keys in the DATABASES ‘default’ item to match your database connection settings:

- ENGINE – Either ‘django.db.backends.sqlite3’, ‘django.db.backends.postgresql’, ‘django.db.backends.mysql’, or ‘django.db.backends.oracle’. Other backends are also available.
- NAME – The name of your database. If you’re using SQLite, the database will be a file on your computer; in that case, NAME should be the full absolute path, including filename, of that file. The default value, BASE_DIR / ‘db.sqlite3’, will store the file in your project directory.

While you’re editing mysite/settings.py, set TIME_ZONE to your time zone.

Also, note the INSTALLED_APPS setting at the top of the file. That holds the names of all Django applications that are activated in this Django instance. Apps can be used in multiple projects, and you can package and distribute them for use by others in their projects.

By default, INSTALLED_APPS contains the following apps, all of which come with Django:

    django.contrib.admin – The admin site. You’ll use it shortly.
    django.contrib.auth – An authentication system.
    django.contrib.contenttypes – A framework for content types.
    django.contrib.sessions – A session framework.
    django.contrib.messages – A messaging framework.
    django.contrib.staticfiles – A framework for managing static files.

These applications are included by default as a convenience for the common case.

Some of these applications make use of at least one database table, though, so we need to create the tables in the database before we can use them. To do that, run the following command:

``` bash
python manage.py migrate
```

The migrate command looks at the INSTALLED_APPS setting and creates any necessary database tables according to the database settings in your mysite/settings.py file and the database migrations shipped with the app (we’ll cover those later). You’ll see a message for each migration it applies. If you’re interested, run the command-line client for your database and type (PostgreSQL), SHOW TABLES; (MariaDB, MySQL), .tables (SQLite), or SELECT TABLE_NAME FROM USER_TABLES; (Oracle) to display the tables Django created.

## Creating models

> Now we’ll define your models – essentially, your database layout, with additional metadata.

------------------------------------------------------------------------

In our poll app, we’ll create two models: Question and Choice. A Question has a question and a publication date. A Choice has two fields: the text of the choice and a vote tally. Each Choice is associated with a Question.

These concepts are represented by Python classes. Edit the polls/models.py file so it looks like this:

polls/models.py

``` python
from django.db import models


class Question(models.Model):
    question_text = models.CharField(max_length=200)
    pub_date = models.DateTimeField("date published")


class Choice(models.Model):
    question = models.ForeignKey(Question, on_delete=models.CASCADE)
    choice_text = models.CharField(max_length=200)
    votes = models.IntegerField(default=0)
```

Here, each model is represented by a class that subclasses django.db.models.Model. Each model has a number of class variables, each of which represents a database field in the model.

Each field is represented by an instance of a Field class – e.g., CharField for character fields and DateTimeField for datetimes. This tells Django what type of data each field holds.

The name of each Field instance (e.g. question_text or pub_date) is the field’s name, in machine-friendly format. You’ll use this value in your Python code, and your database will use it as the column name.

You can use an optional first positional argument to a Field to designate a human-readable name. That’s used in a couple of introspective parts of Django, and it doubles as documentation. If this field isn’t provided, Django will use the machine-readable name. In this example, we’ve only defined a human-readable name for Question.pub_date. For all other fields in this model, the field’s machine-readable name will suffice as its human-readable name.

Some Field classes have required arguments. CharField, for example, requires that you give it a max_length. That’s used not only in the database schema, but in validation, as we’ll soon see.

A Field can also have various optional arguments; in this case, we’ve set the default value of votes to 0.

Finally, note a relationship is defined, using ForeignKey. That tells Django each Choice is related to a single Question. Django supports all the common database relationships: many-to-one, many-to-many, and one-to-one.

## Relationships between entities

Dealing with relationships between entities is one of the more challenging aspects of Web API design. There are a number of different ways that we might choose to represent a relationship:

    Using primary keys.
    Using hyperlinking between entities.
    Using a unique identifying slug field on the related entity.
    Using the default string representation of the related entity.
    Nesting the related entity inside the parent representation.
    Some other custom representation.

REST framework supports all of these styles, and can apply them across forward or reverse relationships, or apply them across custom managers such as generic foreign keys.

In this case we’d like to use a hyperlinked style between entities. In order to do so, we’ll modify our serializers to extend HyperlinkedModelSerializer instead of the existing ModelSerializer.

The HyperlinkedModelSerializer has the following differences from ModelSerializer:

    It does not include the id field by default.
    It includes a url field, using HyperlinkedIdentityField.
    Relationships use HyperlinkedRelatedField, instead of PrimaryKeyRelatedField.

## Activating models

That small bit of model code gives Django a lot of information. With it, Django is able to:

- Create a database schema (CREATE TABLE statements) for this app.
- Create a Python database-access API for accessing Question and Choice objects.

But first we need to tell our project that the polls app is installed.

To include the app in our project, we need to add a reference to its configuration class in the INSTALLED_APPS setting. The PollsConfig class is in the polls/apps.py file, so its dotted path is ‘polls.apps.PollsConfig’. Edit the mysite/settings.py file and add that dotted path to the INSTALLED_APPS setting. It’ll look like this:

test_django/settings.py

``` python
INSTALLED_APPS = [
    "myapp.aps.MyappConfig"
    "polls.apps.PollsConfig",
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
]
```

Now Django knows to include the polls app. Let’s run another command:

``` python
python manage.py makemigrations polls
```

By running makemigrations, you’re telling Django that you’ve made some changes to your models (in this case, you’ve made new ones) and that you’d like the changes to be stored as a migration.

Migrations are how Django stores changes to your models (and thus your database schema) - they’re files on disk. You can read the migration for your new model if you like; it’s the file polls/migrations/0001_initial.py. Don’t worry, you’re not expected to read them every time Django makes one, but they’re designed to be human-editable in case you want to manually tweak how Django changes things.

### SQL

There’s a command that will run the migrations for you and manage your database schema automatically - that’s called migrate, and we’ll come to it in a moment - but first, let’s see what SQL that migration would run. The sqlmigrate command takes migration names and returns their SQL:

``` sh
python manage.py sqlmigrate polls 0001
```

``` sql
BEGIN;
--
-- Create model Question
--
CREATE TABLE "polls_question" (
    "id" bigint NOT NULL PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
    "question_text" varchar(200) NOT NULL,
    "pub_date" timestamp with time zone NOT NULL
);
--
-- Create model Choice
--
CREATE TABLE "polls_choice" (
    "id" bigint NOT NULL PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
    "choice_text" varchar(200) NOT NULL,
    "votes" integer NOT NULL,
    "question_id" bigint NOT NULL
);
ALTER TABLE "polls_choice"
  ADD CONSTRAINT "polls_choice_question_id_c5b4b260_fk_polls_question_id"
    FOREIGN KEY ("question_id")
    REFERENCES "polls_question" ("id")
    DEFERRABLE INITIALLY DEFERRED;
CREATE INDEX "polls_choice_question_id_c5b4b260" ON "polls_choice" ("question_id");

COMMIT;
```

Note the following:

- The exact output will vary depending on the database you are using. The example above is generated for PostgreSQL.
- Table names are automatically generated by combining the name of the app (polls) and the lowercase name of the model – question and choice. (You can override this behavior.)
- Primary keys (IDs) are added automatically. (You can override this, too.)
- By convention, Django appends “\_id” to the foreign key field name. (Yes, you can override this, as well.)
- The foreign key relationship is made explicit by a FOREIGN KEY constraint. Don’t worry about the DEFERRABLE parts; it’s telling PostgreSQL to not enforce the foreign key until the end of the transaction.
- It’s tailored to the database you’re using, so database-specific field types such as auto_increment (MySQL), bigint PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY (PostgreSQL), or integer primary key autoincrement (SQLite) are handled for you automatically. Same goes for the quoting of field names – e.g., using double quotes or single quotes.
- The sqlmigrate command doesn’t actually run the migration on your database - instead, it prints it to the screen so that you can see what SQL Django thinks is required. It’s useful for checking what Django is going to do or if you have database administrators who require SQL scripts for changes.

If you’re interested, you can also run python manage.py check; this checks for any problems in your project without making migrations or touching the database.

## Migrate

Now, run migrate again to create those model tables in your database:

``` bash
python manage.py migrate
```

The migrate command takes all the migrations that haven’t been applied (Django tracks which ones are applied using a special table in your database called django_migrations) and runs them against your database - essentially, synchronizing the changes you made to your models with the schema in the database.

Migrations are very powerful and let you change your models over time, as you develop your project, without the need to delete your database or tables and make new ones - it specializes in upgrading your database live, without losing data. We’ll cover them in more depth in a later part of the tutorial, but for now, remember the three-step guide to making model changes:

- Change your models (in models.py).
- Run python manage.py makemigrations to create migrations for those changes
- Run python manage.py migrate to apply those changes to the database.

The reason that there are separate commands to make and apply migrations is because you’ll commit migrations to your version control system and ship them with your app; they not only make your development easier, they’re also usable by other developers and in production.

## Playing with the API

``` bash
python manage.py shell
```

``` python
>>> from polls.models import Choice, Question  # Import the model classes we just wrote.

# No questions are in the system yet.
>>> Question.objects.all()
<QuerySet []>

# Create a new Question.
# Support for time zones is enabled in the default settings file, so
# Django expects a datetime with tzinfo for pub_date. Use timezone.now()
# instead of datetime.datetime.now() and it will do the right thing.
>>> from django.utils import timezone
>>> q = Question(question_text="What's new?", pub_date=timezone.now())

# Save the object into the database. You have to call save() explicitly.
>>> q.save()

# Now it has an ID.
>>> q.id
1

# Access model field values via Python attributes.
>>> q.question_text
"What's new?"
>>> q.pub_date
datetime.datetime(2012, 2, 26, 13, 0, 0, 775217, tzinfo=datetime.timezone.utc)

# Change values by changing the attributes, then calling save().
>>> q.question_text = "What's up?"
>>> q.save()

# objects.all() displays all the questions in the database.
>>> Question.objects.all()
<QuerySet [<Question: Question object (1)>]>
```

Wait a minute. \<Question: Question object (1)\> isn’t a helpful representation of this object. Let’s fix that by editing the Question model (in the polls/models.py file) and adding a **str**() method to both Question and Choice:

> polls/models.py

``` python
from django.db import models


class Question(models.Model):
    # ...
    def __str__(self):
        return self.question_text


class Choice(models.Model):
    # ...
    def __str__(self):
        return self.choice_text
```

It’s important to add **str**() methods to your models, not only for your own convenience when dealing with the interactive prompt, but also because objects’ representations are used throughout Django’s automatically-generated admin.

Let’s also add a custom method to this model:

> polls/models.py

``` python
import datetime

from django.db import models
from django.utils import timezone


class Question(models.Model):
    # ...
    def was_published_recently(self):
        return self.pub_date >= timezone.now() - datetime.timedelta(days=1)
```

Note the addition of import datetime and from django.utils import timezone, to reference Python’s standard datetime module and Django’s time-zone-related utilities in django.utils.timezone, respectively. If you aren’t familiar with time zone handling in Python, you can learn more in the time zone support docs.

## Introducing the Django Admin

Philosophy

Generating admin sites for your staff or clients to add, change, and delete content is tedious work that doesn’t require much creativity. For that reason, Django entirely automates creation of admin interfaces for models.

Django was written in a newsroom environment, with a very clear separation between “content publishers” and the “public” site. Site managers use the system to add news stories, events, sports scores, etc., and that content is displayed on the public site. Django solves the problem of creating a unified interface for site administrators to edit content.

The admin isn’t intended to be used by site visitors. It’s for site managers.

### Create a superuser

``` bash
python manage.py createsuperuser
```

Enter username and password

``` bash
Username: <name>
Email address: <email>
Password: **********
Password (again): *********
Superuser created successfully.
```

### Make the poll app modifiable in the admin

> polls/admin.py

``` python
from django.contrib import admin

from .models import Question

admin.site.register(Question)
```

> myapp/admin.py

``` python
from django.contrib import admin

from .models import MyModel

admin.site.register(MyModel)
```

## Writing more views

> Now let’s add a few more views to polls/views.py. These views are slightly different, because they take an argument:

------------------------------------------------------------------------

> polls/views.py

``` python
def detail(request, question_id):
    return HttpResponse("You're looking at question %s." % question_id)


def results(request, question_id):
    response = "You're looking at the results of question %s."
    return HttpResponse(response % question_id)


def vote(request, question_id):
    return HttpResponse("You're voting on question %s." % question_id)
```

> polls/urls.py

``` python

from django.urls import path

from . import views

urlpatterns = [
    # ex: /polls/
    path("", views.index, name="index"),
    # ex: /polls/5/
    path("<int:question_id>/", views.detail, name="detail"),
    # ex: /polls/5/results/
    path("<int:question_id>/results/", views.results, name="results"),
    # ex: /polls/5/vote/
    path("<int:question_id>/vote/", views.vote, name="vote"),
]
```

Each view is responsible for doing one of two things: returning an HttpResponse object containing the content for the requested page, or raising an exception such as Http404. The rest is up to you.

> polls/views.py¶

``` python
from django.http import HttpResponse

from .models import Question


def index(request):
    latest_question_list = Question.objects.order_by("-pub_date")[:5]
    output = ", ".join([q.question_text for q in latest_question_list])
    return HttpResponse(output)
```

## Template Folder

There’s a problem here, though: the page’s design is hard-coded in the view. If you want to change the way the page looks, you’ll have to edit this Python code. So let’s use Django’s template system to separate the design from Python by creating a template that the view can use.

First, create a directory called templates in your polls directory. Django will look for templates in there.

Your project’s TEMPLATES setting describes how Django will load and render templates. The default settings file configures a DjangoTemplates backend whose APP_DIRS option is set to True. By convention DjangoTemplates looks for a “templates” subdirectory in each of the INSTALLED_APPS.

Within the templates directory you have just created, create another directory called polls, and within that create a file called index.html. In other words, your template should be at polls/templates/polls/index.html. Because of how the app_directories template loader works as described above, you can refer to this template within Django as polls/index.html.

> polls/templates/polls/index.html¶

``` python
{% if latest_question_list %}
    <ul>
    {% for question in latest_question_list %}
        <li><a href="/polls/{{ question.id }}/">{{ question.question_text }}</a></li>
    {% endfor %}
    </ul>
{% else %}
    <p>No polls are available.</p>
{% endif %}
```

> polls/views.py

``` python
from django.http import HttpResponse
from django.template import loader

from .models import Question


def index(request):
    latest_question_list = Question.objects.order_by("-pub_date")[:5]
    template = loader.get_template("polls/index.html")
    context = {
        "latest_question_list": latest_question_list,
    }
    return HttpResponse(template.render(context, request))
```

### A shortcut: render()

It’s a very common idiom to load a template, fill a context and return an HttpResponse object with the result of the rendered template. Django provides a shortcut. Here’s the full index() view, rewritten:

> polls/views.py

``` python
from django.shortcuts import render

from .models import Question


def index(request):
    latest_question_list = Question.objects.order_by("-pub_date")[:5]
    context = {"latest_question_list": latest_question_list}
    return render(request, "polls/index.html", context)
```

### Raising a 404 error

> Now, let’s tackle the question detail view – the page that displays the question text for a given poll. Here’s the view:

> polls/views.py¶

``` python
from django.http import Http404
from django.shortcuts import render

from .models import Question


# ...
def detail(request, question_id):
    try:
        question = Question.objects.get(pk=question_id)
    except Question.DoesNotExist:
        raise Http404("Question does not exist")
    return render(request, "polls/detail.html", {"question": question})
```

### A shortcut: get_object_or_404()

> polls/views.py

``` python
from django.shortcuts import get_object_or_404, render

from .models import Question


# ...
def detail(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    return render(request, "polls/detail.html", {"question": question})
```

The get_object_or_404() function takes a Django model as its first argument and an arbitrary number of keyword arguments, which it passes to the get() function of the model’s manager. It raises Http404 if the object doesn’t exist.

Philosophy

Why do we use a helper function get_object_or_404() instead of automatically catching the ObjectDoesNotExist exceptions at a higher level, or having the model API raise Http404 instead of ObjectDoesNotExist?

Because that would couple the model layer to the view layer. One of the foremost design goals of Django is to maintain loose coupling. Some controlled coupling is introduced in the django.shortcuts module.

> polls/templates/polls/detail.html¶

``` html
<h1>{{ question.question_text }}</h1>
<ul>
{% for choice in question.choice_set.all %}
    <li>{{ choice.choice_text }}</li>
{% endfor %}
</ul>
```

### Removing hardcoded URLs in templates

Remember, when we wrote the link to a question in the polls/index.html template, the link was partially hardcoded like this:

``` html
<li><a href="/polls/{{ question.id }}/">{{ question.question_text }}</a></li>
```

The problem with this hardcoded, tightly-coupled approach is that it becomes challenging to change URLs on projects with a lot of templates. However, since you defined the name argument in the path() functions in the polls.urls module, you can remove a reliance on specific URL paths defined in your url configurations by using the {% url %} template tag:

``` html
<li><a href="{% url 'detail' question.id %}">{{ question.question_text }}</a></li>
```

The way this works is by looking up the URL definition as specified in the polls.urls module. You can see exactly where the URL name of ‘detail’ is defined below:

``` html
...
# the 'name' value as called by the {% url %} template tag
path("<int:question_id>/", views.detail, name="detail"),
...
```

## Namespacing URL names

The tutorial project has just one app, polls. In real Django projects, there might be five, ten, twenty apps or more. How does Django differentiate the URL names between them? For example, the polls app has a detail view, and so might an app on the same project that is for a blog. How does one make it so that Django knows which app view to create for a url when using the {% url %} template tag?

The answer is to add namespaces to your URLconf. In the polls/urls.py file, go ahead and add an app_name to set the application namespace:

> polls/urls.py

``` python
from django.urls import path

from . import views

app_name = "polls"
urlpatterns = [
    path("", views.index, name="index"),
    path("<int:question_id>/", views.detail, name="detail"),
    path("<int:question_id>/results/", views.results, name="results"),
    path("<int:question_id>/vote/", views.vote, name="vote"),
]
```

## Write a minimal form

> polls/templates/polls/detail.html¶

``` html
<form action="{% url 'polls:vote' question.id %}" method="post">
{% csrf_token %}
<fieldset>
    <legend><h1>{{ question.question_text }}</h1></legend>
    {% if error_message %}<p><strong>{{ error_message }}</strong></p>{% endif %}
    {% for choice in question.choice_set.all %}
        <input type="radio" name="choice" id="choice{{ forloop.counter }}" value="{{ choice.id }}">
        <label for="choice{{ forloop.counter }}">{{ choice.choice_text }}</label><br>
    {% endfor %}
</fieldset>
<input type="submit" value="Vote">
</form>
```

A quick rundown:

- The above template displays a radio button for each question choice. The value of each radio button is the associated question choice’s ID. The name of each radio button is “choice”. That means, when somebody selects one of the radio buttons and submits the form, it’ll send the POST data choice=# where \# is the ID of the selected choice. This is the basic concept of HTML forms.
- We set the form’s action to {% url ‘polls:vote’ question.id %}, and we set method=“post”. Using method=“post” (as opposed to method=“get”) is very important, because the act of submitting this form will alter data server-side. Whenever you create a form that alters data server-side, use method=“post”. This tip isn’t specific to Django; it’s good web development practice in general.
- forloop.counter indicates how many times the for tag has gone through its loop
- Since we’re creating a POST form (which can have the effect of modifying data), we need to worry about Cross Site Request Forgeries. Thankfully, you don’t have to worry too hard, because Django comes with a helpful system for protecting against it. In short, all POST forms that are targeted at internal URLs should use the {% csrf_token %} template tag.

We also created a dummy implementation of the vote() function. Let’s create a real version. Add the following to polls/views.py:

> polls/views.py

``` python
from django.db.models import F
from django.http import HttpResponse, HttpResponseRedirect
from django.shortcuts import get_object_or_404, render
from django.urls import reverse

from .models import Choice, Question


# ...
def vote(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    try:
        selected_choice = question.choice_set.get(pk=request.POST["choice"])
    except (KeyError, Choice.DoesNotExist):
        # Redisplay the question voting form.
        return render(
            request,
            "polls/detail.html",
            {
                "question": question,
                "error_message": "You didn't select a choice.",
            },
        )
    else:
        selected_choice.votes = F("votes") + 1
        selected_choice.save()
        # Always return an HttpResponseRedirect after successfully dealing
        # with POST data. This prevents data from being posted twice if a
        # user hits the Back button.
        return HttpResponseRedirect(reverse("polls:results", args=(question.id,)))
```

This code includes a few things we haven’t covered yet in this tutorial:

- request.POST is a dictionary-like object that lets you access submitted data by key name. In this case, request.POST\[‘choice’\] returns the ID of the selected choice, as a string. request.POST values are always strings.

- Note that Django also provides request.GET for accessing GET data in the same way – but we’re explicitly using request.POST in our code, to ensure that data is only altered via a POST call.

- request.POST\[‘choice’\] will raise KeyError if choice wasn’t provided in POST data. The above code checks for KeyError and redisplays the question form with an error message if choice isn’t given.

- F(“votes”) + 1 instructs the database to increase the vote count by 1.

- After incrementing the choice count, the code returns an HttpResponseRedirect rather than a normal HttpResponse. HttpResponseRedirect takes a single argument: the URL to which the user will be redirected (see the following point for how we construct the URL in this case).

- As the Python comment above points out, you should always return an HttpResponseRedirect after successfully dealing with POST data. This tip isn’t specific to Django; it’s good web development practice in general.

- We are using the reverse() function in the HttpResponseRedirect constructor in this example. This function helps avoid having to hardcode a URL in the view function. It is given the name of the view that we want to pass control to and the variable portion of the URL pattern that points to that view. In this case, using the URLconf we set up in Tutorial 3, this reverse() call will return a string like

> polls/views.py

``` python
from django.shortcuts import get_object_or_404, render


def results(request, question_id):
    question = get_object_or_404(Question, pk=question_id)
    return render(request, "polls/results.html", {"question": question})
```

> polls/templates/polls/results.html¶

``` html
<h1>{{ question.question_text }}</h1>

<ul>
{% for choice in question.choice_set.all %}
    <li>{{ choice.choice_text }} -- {{ choice.votes }} vote{{ choice.votes|pluralize }}</li>
{% endfor %}
</ul>

<a href="{% url 'polls:detail' question.id %}">Vote again?</a>
```

## Use generic views: Less code is better

### Amend URLconf

> polls/urls.py

``` python
from django.urls import path

from . import views

app_name = "polls"
urlpatterns = [
    path("", views.IndexView.as_view(), name="index"),
    path("<int:pk>/", views.DetailView.as_view(), name="detail"),
    path("<int:pk>/results/", views.ResultsView.as_view(), name="results"),
    path("<int:question_id>/vote/", views.vote, name="vote"),
]
```

Note that the name of the matched pattern in the path strings of the second and third patterns has changed from to . This is necessary because we’ll use the DetailView generic view to replace our detail() and results() views, and it expects the primary key value captured from the URL to be called “pk”.

### Amend views

> polls/views.py

``` python
from django.db.models import F
from django.http import HttpResponseRedirect
from django.shortcuts import get_object_or_404, render
from django.urls import reverse
from django.views import generic

from .models import Choice, Question


class IndexView(generic.ListView):
    template_name = "polls/index.html"
    context_object_name = "latest_question_list"

    def get_queryset(self):
        """Return the last five published questions."""
        return Question.objects.order_by("-pub_date")[:5]


class DetailView(generic.DetailView):
    model = Question
    template_name = "polls/detail.html"


class ResultsView(generic.DetailView):
    model = Question
    template_name = "polls/results.html"


def vote(request, question_id):
    # same as above, no changes needed.
    ...
```

Each generic view needs to know what model it will be acting upon. This is provided using either the model attribute (in this example, model = Question for DetailView and ResultsView) or by defining the get_queryset() method (as shown in IndexView).

By default, the DetailView generic view uses a template called /\_detail.html. In our case, it would use the template “polls/question_detail.html”. The template_name attribute is used to tell Django to use a specific template name instead of the autogenerated default template name. We also specify the template_name for the results list view – this ensures that the results view and the detail view have a different appearance when rendered, even though they’re both a DetailView behind the scenes.

Similarly, the ListView generic view uses a default template called /\_list.html; we use template_name to tell ListView to use our existing “polls/index.html” template.

In previous parts of the tutorial, the templates have been provided with a context that contains the question and latest_question_list context variables. For DetailView the question variable is provided automatically – since we’re using a Django model (Question), Django is able to determine an appropriate name for the context variable. However, for ListView, the automatically generated context variable is question_list. To override this we provide the context_object_name attribute, specifying that we want to use latest_question_list instead. As an alternative approach, you could change your templates to match the new default context variables – but it’s a lot easier to tell Django to use the variable you want.

Run the server, and use your new polling app based on generic views.

For full details on generic views, see the generic views documentation.

When you’re comfortable with forms and generic views, read part 5 of this tutorial to learn about testing our polls app.

## Writing our first test

### Create a test to expose the bug

> polls/tests.py

``` python
import datetime

from django.test import TestCase
from django.utils import timezone

from .models import Question


class QuestionModelTests(TestCase):
    def test_was_published_recently_with_future_question(self):
        """
        was_published_recently() returns False for questions whose pub_date
        is in the future.
        """
        time = timezone.now() + datetime.timedelta(days=30)
        future_question = Question(pub_date=time)
        self.assertIs(future_question.was_published_recently(), False)
```

Here we have created a django.test.TestCase subclass with a method that creates a Question instance with a pub_date in the future. We then check the output of was_published_recently() - which ought to be False.

### Running tests

In the terminal, we can run our test:

``` bash
python manage.py test polls
```

and you’ll see something like:

### Fixing the bug

``` python

def was_published_recently(self):
    now = timezone.now()
    return now - datetime.timedelta(days=1) <= self.pub_date <= now
```

### More comprehensive tests

> polls/tests.py

``` python
def test_was_published_recently_with_old_question(self):
    """
    was_published_recently() returns False for questions whose pub_date
    is older than 1 day.
    """
    time = timezone.now() - datetime.timedelta(days=1, seconds=1)
    old_question = Question(pub_date=time)
    self.assertIs(old_question.was_published_recently(), False)


def test_was_published_recently_with_recent_question(self):
    """
    was_published_recently() returns True for questions whose pub_date
    is within the last day.
    """
    time = timezone.now() - datetime.timedelta(hours=23, minutes=59, seconds=59)
    recent_question = Question(pub_date=time)
    self.assertIs(recent_question.was_published_recently(), True)
```

### Improving our view

> polls/views.py

``` python
class IndexView(generic.ListView):
    template_name = "polls/index.html"
    context_object_name = "latest_question_list"

    def get_queryset(self):
        """Return the last five published questions."""
        return Question.objects.order_by("-pub_date")[:5]
```

## Static folder

First, create a directory called static in your polls directory. Django will look for static files there, similarly to how Django finds templates inside polls/templates/.

Django’s STATICFILES_FINDERS setting contains a list of finders that know how to discover static files from various sources. One of the defaults is AppDirectoriesFinder which looks for a “static” subdirectory in each of the INSTALLED_APPS, like the one in polls we just created. The admin site uses the same directory structure for its static files.

Within the static directory you have just created, create another directory called polls and within that create a file called style.css. In other words, your stylesheet should be at polls/static/polls/style.css. Because of how the AppDirectoriesFinder staticfile finder works, you can refer to this static file in Django as polls/style.css, similar to how you reference the path for templates.

> polls/static/polls/style.css¶

``` css
li a {
    color: green;
}
```

### Adding a background-image

Next, we’ll create a subdirectory for images. Create an images subdirectory in the polls/static/polls/ directory. Inside this directory, add any image file that you’d like to use as a background. For the purposes of this tutorial, we’re using a file named background.png, which will have the full path polls/static/polls/images/background.png.

Then, add a reference to your image in your stylesheet (polls/static/polls/style.css):

``` css
body {
    background: white url("images/background.png") no-repeat;
}
```

> polls/templates/polls/index.html¶

``` html
{% load static %}

<link rel="stylesheet" href="{% static 'polls/style.css' %}">
```

The {% static %} template tag generates the absolute URL of static files.

That’s all you need to do for development.

Start the server (or restart it if it’s already running):

## Customize the admin form

By registering the Question model with admin.site.register(Question), Django was able to construct a default form representation. Often, you’ll want to customize how the admin form looks and works. You’ll do this by telling Django the options you want when you register the object.

Let’s see how this works by reordering the fields on the edit form. Replace the admin.site.register(Question) line with:

> polls/admin.py¶

``` python
from django.contrib import admin

from .models import Choice, Question


class ChoiceInline(admin.TabularInline):
    model = Choice
    extra = 3


class QuestionAdmin(admin.ModelAdmin):
    fieldsets = [
        (None, {"fields": ["question_text"]}),
        ("Date information", {"fields": ["pub_date"], "classes": ["collapse"]}),
    ]
    inlines = [ChoiceInline]
    list_display = ["question_text", "pub_date", "was_published_recently"]
    list_filter = ["pub_date"]


admin.site.register(Question, QuestionAdmin)
```

You can click on the column headers to sort by those values – except in the case of the was_published_recently header, because sorting by the output of an arbitrary method is not supported. Also note that the column header for was_published_recently is, by default, the name of the method (with underscores replaced with spaces), and that each line contains the string representation of the output.

You can improve that by using the display() decorator on that method (in polls/models.py), as follows:

> polls/models.py¶

``` python
from django.contrib import admin


class Question(models.Model):
    # ...
    @admin.display(
        boolean=True,
        ordering="pub_date",
        description="Published recently?",
    )
    def was_published_recently(self):
        now = timezone.now()
        return now - datetime.timedelta(days=1) <= self.pub_date <= now
```

## Installing Django Debug Toolbar

``` bash
python -m pip install django-debug-toolbar
```

Back to top
