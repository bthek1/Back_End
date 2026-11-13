# Back End

Django and the data underneath it: models, the ORM, REST APIs, authentication, background work, and the databases that store the result.

Author

Benedict Thekkel

This is the largest site in the collection and the most opinionated, because it is written from actually shipping Django applications rather than from surveying the field. Django and Django REST Framework are assumed throughout.

The shape is roughly the order you meet the problems. Model your data, query it without generating a thousand queries, expose it as an API, decide who is allowed to call it, move the slow parts into the background, and store it somewhere that survives a restart. The database folder covers that last part independently, because the choice outlives any one application.

------------------------------------------------------------------------

## Getting Started with Django

| Page | Covers |
|----|----|
| [Django Basics](Django/django_basics.llms.md) | The foundation, and by far the longest page here at 150 cells |
| [Django Stack](Django/django_stack.llms.md) | The pieces and how they fit |
| [DjangoX](Django/django_x.llms.md) | A preconfigured starting point |
| [Django Templates](Django/django_templates.llms.md) | Starter templates, from barebones to full SaaS platforms |

------------------------------------------------------------------------

## Models and Data

| Page | Covers |
|----|----|
| [Model Design](Django/0_models_design.llms.md) | Getting the schema right, which is the decision everything else inherits |
| [Saving Objects](Django/1_save_objects.llms.md) | How writes actually go through |
| [JSONField](Django/2_jsonfield.llms.md) | Storing and querying JSON in the database when a rigid schema does not fit |
| [Migrations](Django/migrations.llms.md) | Changing the schema without losing data |
| [Validators](Django/validators.llms.md) | Enforcing conditions before a value reaches the database, across models, forms, and serializers |

------------------------------------------------------------------------

## Querying the ORM

Five pages on the same subject, because this is where Django applications get slow.

| Page | Covers |
|----|----|
| [Query Filters](Django/0_query_filters.llms.md) | How the QuerySet API turns Pythonic queries into SQL |
| [QuerySets](Django/1_querysets.llms.md) | The object you are actually manipulating |
| [Query Math](Django/2_query_maths.llms.md) | Aggregations, arithmetic, and window functions, all without dropping to raw SQL |
| [Managers and QuerySets](Django/managers_and_querysets.llms.md) | Encapsulating query logic where it belongs instead of scattering it through views |
| [django-filter](Django/filters.llms.md) | Filtering a queryset from form input, wired into class-based views and DRF |
| [Indexing and Caching](Django/indexing_and_caching.llms.md) | The two levers for query optimisation |

------------------------------------------------------------------------

## REST APIs

| Page | Covers |
|----|----|
| [REST API Standard](rest_api_standard.llms.md) | The architectural style itself, before any framework |
| [DRF Basics](Django/1_drf_basic.llms.md) | What the toolkit gives you |
| [DRF Quickstart](Django/2_drf_quickstart.llms.md) | The fastest path to a working API |
| [DRF Tutorial](Django/3_drf_tutorial.llms.md) | The longer worked version |
| [Serialization](Django/serialization.llms.md) | Turning models into JSON and back |
| [Views](Django/views.llms.md) | The request handlers in between |
| [Nested Routers](Django/nested_routers.llms.md) | Routing resources that belong to other resources |
| [DRF Spectacular](Django/drf_spectacular.llms.md) | Generating OpenAPI 3.0 schemas, which is what feeds Swagger UI, ReDoc, and Postman |
| [DataTables Server-Side](Django/datatables.llms.md) | Delegating filtering, sorting, and pagination from the browser table to DRF |

------------------------------------------------------------------------

## Authentication, Permissions, and Secrets

| Page | Covers |
|----|----|
| [Authentication](Django/0_authentication.llms.md) | DRF’s authentication system |
| [User Models](Django/0_users.llms.md) | Modelling users, which is worth getting right before the first migration |
| [Allauth](Django/1_allauth_users.llms.md) | Third-party accounts and social login |
| [Permissions](Django/0_permissions.llms.md) | Controlling who can reach which endpoint |
| [Permission Rules](Django/1_permission_rules.llms.md) | Object-level permissions expressed as plain Python functions, via django-rules |
| [Permission Types](Django/2_permission_types.llms.md) | Access control models for SaaS specifically, and how to choose between them |
| [Security](Django/1_security.llms.md) | Secure coding practice, audits, and the tooling that finds problems |
| [Secrets](Django/secrets.llms.md) | Keeping credentials out of the repository |

------------------------------------------------------------------------

## Admin and Forms

| Page | Covers |
|----|----|
| [Django Admin](Django/0_admin.llms.md) | Customising the admin from the basics to the advanced techniques |
| [Admin Unfold](Django/1_admin_unfold.llms.md) | Modernising the admin interface without rebuilding it |
| [Forms](Django/0_forms.llms.md) | The built-in form layer |
| [Crispy Forms](Django/1_crispy_forms.llms.md) | Rendering them without hand-writing the markup |
| [npm in Django](Django/npm_in_django.llms.md) | Managing front-end dependencies such as Bootstrap from inside a Django project |

------------------------------------------------------------------------

## Background Work and Real Time

| Page | Covers |
|----|----|
| [django-q](Django/django_q.llms.md) | A task queue and scheduler built for Django, simpler than the alternative |
| [Celery](Django/celery.llms.md) | The distributed alternative, for when django-q is not enough |
| [Channels](Django/channels.llms.md) | WebSockets and async protocols, which is what makes chat, notifications, and live updates possible |
| [MQTT](mqtt.llms.md) | The publish/subscribe broker, for IoT devices and sensors feeding the backend |

------------------------------------------------------------------------

## Operations

| Page | Covers |
|----|----|
| [Custom Commands](Django/0_custom_commands.llms.md) | Writing your own `manage.py` subcommands |
| [Django Commands](Django/1_django_commands.llms.md) | The built-in ones |
| [Testing](Django/testing.llms.md) | pytest against a Django project |
| [Backups](Django/backups.llms.md) | django-dbbackup for database and media, to local or cloud storage |
| [Email](Django/django_mail.llms.md) | Configuring and sending mail |
| [Import and Export](Django/import_export.llms.md) | Moving data in and out as CSV, Excel, or JSON, from the admin or programmatically |
| [PDF Generation](Django/pdf.llms.md) | WeasyPrint rendering HTML and CSS into invoices, reports, and receipts |
| [Constance](Django/django_constance.llms.md) | Admin-editable settings stored in the database, which gives you feature flags and thresholds without a redeploy |
| [Recovery Metrics Setup](Django/rm_django_setup.llms.md) | The project-specific setup notes |

------------------------------------------------------------------------

## FastAPI

The async-first alternative, for API-only services, IO-heavy work and model serving. Both pages compare against the Django way at the end.

| Page | Covers |
|----|----|
| [FastAPI Stack](FastAPI/fastapi_stack.llms.md) | The standard stack, one choice per layer, and a small working service from settings to tests and deployment |
| [SQLAlchemy and Alembic](FastAPI/sqlalchemy_alembic.llms.md) | The ORM and migrations outside Django: async sessions, the lazy-loading trap, and migrations that do not lose data |

------------------------------------------------------------------------

## Databases

Independent of Django, because the choice outlives the application.

| Page | Covers |
|----|----|
| [SQL](Databases/sql.llms.md) | The language itself: querying, inserting, updating, deleting |
| [Database Scaling](Databases/database_scaling.llms.md) | Growing past one machine: pooling, caching, read replicas and replication lag, partitioning, sharding, and distributed SQL |
| [PostgreSQL](Databases/PostgreSQL/postgresql.llms.md) | Database structure and practice, for the default choice |
| [PostgreSQL Setup](Databases/PostgreSQL/postgresql_setup.llms.md) | Getting it running |
| [PostgreSQL Extensions](Databases/PostgreSQL/pqsl_extensions.llms.md) | What extensions are, how they work, and a catalogue grouped by use case |
| [TimescaleDB](Databases/PostgreSQL/timescaledb.llms.md) | Time-series inside Postgres, driven from Django |
| [MySQL and MariaDB](Databases/mysql_and_mariadb.llms.md) | The other major open-source relational option |
| [SQLite](Databases/sqlite.llms.md) | Serverless and zero-configuration, for embedded use and development |
| [DuckDB](Databases/duckdb.llms.md) | The analytical counterpart, aimed at data and ML work rather than transactions |
| [Redis](Databases/redis.llms.md) | In-memory store used as cache, broker, and database, with the data structures that make it flexible |
| [InfluxDB](influx.llms.md) | The time-series backend, covered at length |

------------------------------------------------------------------------

## Not Covered Yet

- **No Flask, and FastAPI is only two pages.** Auth beyond JWT, WebSockets and FastAPI-specific deployment are touched on, not covered in depth.
- **Nothing on deployment.** Getting the application onto a server is covered in [Web Development](https://bthek1.github.io/Web_development/) and [IaaS](https://bthek1.github.io/IaaS_docs/), not here, so this site stops at the code.
- **No API versioning or deprecation strategy**, despite the depth of DRF coverage.
- **Nothing on rate limiting or throttling** as a subject of its own.
- **[QuerySets](Django/1_querysets.llms.md), [Indexing and Caching](Django/indexing_and_caching.llms.md), and [Email](Django/django_mail.llms.md) are thin** at four to five cells, and indexing in particular deserves more given how often it is the answer.
- `Django/03_2_Permission_Types.ipynb` has a stray space in its filename, which is why its URL looks wrong.

------------------------------------------------------------------------

Back to top
