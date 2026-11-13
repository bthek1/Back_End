# SQLAlchemy and Alembic

SQLAlchemy 2.0 as the ORM and Alembic as the migration tool: the mental model, async use under FastAPI, the loading trap, and a migration workflow that does not lose data.

Author

Benedict Thekkel

In Django the ORM and migrations are one product. Outside Django they are two: **SQLAlchemy** maps Python classes to tables and runs queries, and **Alembic** changes the schema over time. Both come from the same author, and Alembic works by reading SQLAlchemy’s description of your tables.

SQLAlchemy is more explicit than the Django ORM. You open sessions, you choose when to commit, and you decide how related rows load. That is more typing, but nothing happens behind your back, and that matters under async, where an implicit query is an error rather than a slow page. This page uses the 2.0 API throughout, the async engine, and PostgreSQL through psycopg 3. It is the database layer of the [FastAPI Stack](../FastAPI/fastapi_stack.llms.md).

------------------------------------------------------------------------

## 1. The Mental Model

SQLAlchemy has two layers. **Core** is a Python way to write SQL: `Table`, `select()`, `insert()`. The **ORM** sits on top and maps classes to tables. In 2.0 both layers use the same `select()`, so there is one query language, not two.

| Term | What it is | Django nearest |
|----|----|----|
| Engine | The connection pool plus the dialect for one database. One per process | `DATABASES` setting plus the connection handler |
| Session | A unit of work: it tracks objects you added or changed and writes them in one transaction | No direct equal. Django saves each object immediately |
| Identity map | Inside one session, one row is one Python object. Loading `User` 1 twice returns the same object | None |
| Flush | Sends pending `INSERT` and `UPDATE` statements inside the open transaction. Nothing is committed yet | `obj.save()` inside `atomic()` |
| Commit | Ends the transaction, making the flushed changes permanent | Leaving an `atomic()` block |
| `MetaData` | The collection of all `Table` definitions. Alembic diffs against it | The migration state Django builds from your apps |

The session **autoflushes** before each query. So if you add a `User` and then query for it in the same session, the query finds it, even before commit.

------------------------------------------------------------------------

## 2. The Base Class and Naming Convention

Every model inherits from one `Base`. Give its `MetaData` a naming convention **before the first migration**.

------------------------------------------------------------------------

``` python
# app/core/models.py
from sqlalchemy import MetaData
from sqlalchemy.orm import DeclarativeBase

NAMING_CONVENTION = {
    "ix": "ix_%(column_0_label)s",
    "uq": "uq_%(table_name)s_%(column_0_name)s",
    "ck": "ck_%(table_name)s_%(constraint_name)s",
    "fk": "fk_%(table_name)s_%(column_0_name)s_%(referred_table_name)s",
    "pk": "pk_%(table_name)s",
}


class Base(DeclarativeBase):
    metadata = MetaData(naming_convention=NAMING_CONVENTION)
```

Why it matters: without it, PostgreSQL invents a name for each unique, foreign key and check constraint. Alembic does not know those names, so a later migration that drops or changes the constraint cannot refer to it. With the convention, every constraint gets a predictable name that both sides agree on. Adding it later means a migration that renames every existing constraint, so do it on day one.

The `ck` pattern uses `%(constraint_name)s`, so each `CheckConstraint` needs a `name=`.

------------------------------------------------------------------------

## 3. Models

In 2.0 the type annotation does most of the work. `Mapped[str]` is `NOT NULL`. `Mapped[str | None]` is nullable. The Python type picks the SQL type unless you pass one to `mapped_column`.

------------------------------------------------------------------------

``` python
# app/users/models.py
from datetime import datetime

from sqlalchemy import DateTime, ForeignKey, String, Text, func
from sqlalchemy.orm import Mapped, mapped_column, relationship

from app.core.models import Base


class User(Base):
    __tablename__ = "user_account"  # "user" is a reserved word in PostgreSQL

    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(String(320), unique=True)
    full_name: Mapped[str | None]
    is_active: Mapped[bool] = mapped_column(default=True)
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())

    posts: Mapped[list["Post"]] = relationship(back_populates="author", lazy="raise")


class Post(Base):
    __tablename__ = "post"

    id: Mapped[int] = mapped_column(primary_key=True)
    author_id: Mapped[int] = mapped_column(ForeignKey("user_account.id", ondelete="CASCADE"), index=True)
    title: Mapped[str] = mapped_column(String(200))
    body: Mapped[str] = mapped_column(Text)

    author: Mapped[User] = relationship(back_populates="posts", lazy="raise")
```

Four things that differ from Django:

1.  **Foreign keys are not indexed for you.** Django adds an index to every `ForeignKey`. SQLAlchemy and PostgreSQL do not. Add `index=True` to almost every foreign key column, or joins and cascading deletes scan the whole table.
2.  **`DateTime(timezone=True)` maps to `timestamptz`.** Plain `DateTime` is `timestamp without time zone`, which loses the offset. Always pass `timezone=True`.
3.  **`default` vs `server_default`.** `default=True` is applied by Python when SQLAlchemy inserts. `server_default=func.now()` becomes part of the column definition, so it also applies to rows inserted by raw SQL or another service.
4.  **`relationship()` is not a column.** It is a Python-side link built from the foreign key. `back_populates` keeps both sides in step in memory.

`lazy="raise"` is explained in section 6. It is the most important setting on this page for async code.

------------------------------------------------------------------------

## 4. Engine and Session (Async)

``` python
from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

engine = create_async_engine(
    "postgresql+psycopg://app:app@localhost:5432/app",
    pool_size=5,
    max_overflow=10,
    pool_pre_ping=True,  # test each connection before use, so a restarted database does not fail the next request
    echo=False,          # True logs every SQL statement, useful while learning
)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)
```

The URL picks the driver: `postgresql+psycopg://` is psycopg 3, which works for both sync and async engines. `postgresql+asyncpg://` is the async-only alternative and is somewhat faster, at the cost of a second driver for anything sync.

**`expire_on_commit=False` is required under async.** By default, a commit marks every loaded object as stale. The next attribute access then fires a query to reload it. In async code that hidden query cannot run (see section 6), so reading `item.name` after a commit would fail. With `False`, objects keep their loaded values after commit.

------------------------------------------------------------------------

## 5. Querying

Every query is a `select()` statement, run through the session. Pick the method by what you want back.

| You want                  | Method                                |
|---------------------------|---------------------------------------|
| One object by primary key | `await session.get(User, 1)`          |
| A list of ORM objects     | `(await session.scalars(stmt)).all()` |
| One object or `None`      | `await session.scalar(stmt)`          |
| Rows of several columns   | `(await session.execute(stmt)).all()` |

------------------------------------------------------------------------

``` python
from sqlalchemy import func, select

# filter and order
stmt = select(User).where(User.email.ilike("%@example.com"), User.is_active).order_by(User.id)
users = (await session.scalars(stmt)).all()

# one or None
user = await session.scalar(select(User).where(User.email == "a@example.com"))

# join across a relationship
stmt = select(Post).join(Post.author).where(User.email == "a@example.com")
posts = (await session.scalars(stmt)).all()

# aggregate: post count per user, including users with none
stmt = (
    select(User.email, func.count(Post.id).label("n_posts"))
    .outerjoin(User.posts)
    .group_by(User.id)
    .order_by(func.count(Post.id).desc())
)
for email, n_posts in (await session.execute(stmt)).all():
    print(email, n_posts)
```

`where()` with several arguments joins them with `AND`. For `OR`, use `or_(a, b)`. Comparisons such as `User.email == "x"` do not return a boolean; they build a SQL expression. That is why `==` works inside `where()`.

------------------------------------------------------------------------

## 6. Loading Relationships: The Async Trap

By default a relationship is *lazy*: `user.posts` runs a `SELECT` the first time you touch it. In sync code that is the N+1 problem. **In async code it is an error.** The access is a plain attribute read, which cannot `await`, so SQLAlchemy raises `MissingGreenlet` (a greenlet is the mechanism it would need to hide the IO).

The fix is to say up front what you will need, as part of the query.

------------------------------------------------------------------------

``` python
from sqlalchemy.orm import joinedload, selectinload

# collection: one extra query, SELECT ... WHERE author_id IN (...)
stmt = select(User).options(selectinload(User.posts))
users = (await session.scalars(stmt)).all()

# many-to-one: same query, LEFT OUTER JOIN
stmt = select(Post).options(joinedload(Post.author))
posts = (await session.scalars(stmt)).all()
```

For deeper paths, chain one loader per hop, for example `selectinload(User.posts).selectinload(Post.comments)` if posts had comments.

| Strategy | SQL it runs | Use for | Django nearest |
|----|----|----|----|
| `selectinload` | A second `SELECT ... WHERE id IN (...)` | Collections (one-to-many, many-to-many) | `prefetch_related` |
| `joinedload` | A `LEFT OUTER JOIN` in the same query | Single related objects (many-to-one) | `select_related` |
| `lazy="raise"` | None. Access raises at once | The default on every relationship | None |
| `lazy="select"` | A query on first access | Sync scripts only | Django’s default |

**Set `lazy="raise"` on every relationship.** Then a missing loader fails loudly in your tests with a clear message, instead of failing as `MissingGreenlet` in production or silently running N+1 queries in sync code. You opt in to loading per query with `options(...)`.

`joinedload` on a collection repeats the parent row for each child, so the result needs `.unique()` before `.all()`. This is one more reason to use `selectinload` for collections.

------------------------------------------------------------------------

## 7. Writes and Transactions

``` python
from sqlalchemy import delete, update
from sqlalchemy.dialects.postgresql import insert

# insert: add objects, then commit once
async with SessionLocal() as session:
    async with session.begin():  # commits at the end, rolls back on exception
        user = User(email="a@example.com")
        session.add(user)
        session.add(Post(author=user, title="Hello", body="First post"))
        await session.flush()  # user.id is set now, still inside the transaction

# update one loaded object: change the attribute, the session sees it
user.full_name = "Ada"
await session.commit()

# bulk update and delete: one statement, no objects loaded
await session.execute(update(Post).where(Post.author_id == 1).values(title="Archived"))
await session.execute(delete(Post).where(Post.author_id == 1))
await session.commit()

# upsert (PostgreSQL): insert, or update the existing row on conflict
stmt = insert(User).values(email="a@example.com", full_name="Ada")
stmt = stmt.on_conflict_do_update(index_elements=[User.email], set_={"full_name": stmt.excluded.full_name})
await session.execute(stmt)
await session.commit()
```

`session.begin()` as a context manager is the equivalent of Django’s `transaction.atomic()`. Under FastAPI the [FastAPI Stack](../FastAPI/fastapi_stack.llms.md) page commits explicitly in the endpoint instead, so each request is one transaction with a visible end.

Bulk `update()` and `delete()` skip the identity map. Objects already loaded in the session keep their old values until you refresh them. The same caveat applies to Django’s `QuerySet.update()`.

------------------------------------------------------------------------

## 8. Django ORM to SQLAlchemy

| Django | SQLAlchemy 2.0 |
|----|----|
| `User.objects.get(pk=1)` | `await session.get(User, 1)` |
| `User.objects.filter(is_active=True)` | `select(User).where(User.is_active)` |
| `.exclude(email="x")` | `.where(User.email != "x")` |
| `Q(a) \| Q(b)` | `or_(a, b)` |
| `.order_by("-id")` | `.order_by(User.id.desc())` |
| `[:20]` | `.limit(20)` |
| `.values("email")` | `select(User.email)` |
| `.annotate(n=Count("posts"))` | `select(User, func.count(Post.id)).outerjoin(User.posts).group_by(User.id)` |
| `F("price") * 2` | `Item.price_cents * 2` |
| `select_related("author")` | `options(joinedload(Post.author))` |
| `prefetch_related("posts")` | `options(selectinload(User.posts))` |
| `obj.save()` | `session.add(obj)` then commit |
| `QuerySet.update(...)` | `update(Model).where(...).values(...)` |
| `transaction.atomic()` | `async with session.begin():` |
| `Model.objects.raw(sql)` | `await session.execute(text(sql))` |

------------------------------------------------------------------------

## 9. Alembic Setup

``` bash
uv add alembic
uv run alembic init -t async migrations
```

The `async` template creates `alembic.ini`, `migrations/env.py`, `migrations/script.py.mako` and an empty `migrations/versions/`. `uv run alembic list_templates` shows the other templates your installed version ships, including one that keeps the config in `pyproject.toml` on recent releases.

Three edits to `migrations/env.py`:

------------------------------------------------------------------------

``` python
# migrations/env.py (the parts you change)
from app.core.config import get_settings
from app.core.models import Base
import app.users.models  # noqa: F401  every models module must be imported, or its tables look deleted
import app.items.models  # noqa: F401

config = context.config
# configparser treats % as interpolation, so a password with % in it must be escaped
config.set_main_option("sqlalchemy.url", get_settings().database_url.replace("%", "%%"))

target_metadata = Base.metadata


def do_run_migrations(connection):
    context.configure(
        connection=connection,
        target_metadata=target_metadata,
        compare_server_default=True,  # off by default; without it, server_default changes are ignored
    )
    with context.begin_transaction():
        context.run_migrations()
```

1.  **Take the URL from your settings,** not from `alembic.ini`. Then the app and the migrations can never point at different databases, and no password sits in a tracked file.
2.  **Point `target_metadata` at `Base.metadata`,** after importing every module that defines models. A model module that is not imported is invisible, and autogenerate will propose dropping its table.
3.  **Turn on `compare_server_default`.** Type changes are compared by default; server defaults are not.

Optional but worth it, in `alembic.ini`: a dated `file_template` so revision files sort by time in the editor.

``` ini
file_template = %%(year)d_%%(month).2d_%%(day).2d_%%(hour).2d%%(minute).2d-%%(rev)s_%%(slug)s
```

------------------------------------------------------------------------

## 10. The Daily Workflow

![](alembic_workflow.svg)

Alembic workflow

Autogenerate compares two things: the tables your models describe (`target_metadata`) and the tables the live database actually has. The difference becomes the `upgrade()` function of a new revision file. So the database you run it against must already be at the latest revision, or the diff includes changes that earlier revisions already make.

| Command | Does | Django nearest |
|----|----|----|
| `alembic revision --autogenerate -m "add post"` | Writes a new revision from the diff | `makemigrations` |
| `alembic revision -m "backfill slugs"` | Writes an empty revision for hand-written work | `makemigrations --empty` |
| `alembic upgrade head` | Applies all pending revisions | `migrate` |
| `alembic downgrade -1` | Reverts the last revision | `migrate app 0003` |
| `alembic current` | Shows the revision the database is at | `showmigrations` |
| `alembic history --verbose` | Lists the revision chain | `showmigrations --plan` |
| `alembic check` | Fails if the models have changes with no revision | `makemigrations --check` |
| `alembic upgrade head --sql` | Prints the SQL instead of running it | `sqlmigrate` |
| `alembic merge heads -m "merge"` | Joins two branches of revisions | `makemigrations --merge` |
| `alembic stamp head` | Marks the database as current without running anything | `migrate --fake` |

Put `alembic check` in CI. It is the cheapest guard against a model change merged without its migration.

The database records its position in a one-row table, `alembic_version`. Each revision file names its parent in `down_revision`, which makes the files a linked list. Two developers branching from the same revision produce two heads, and `upgrade head` refuses until you merge them.

------------------------------------------------------------------------

## 11. A Generated Revision

Always read it before you apply it. This is what autogenerate writes for the `Post` model above:

------------------------------------------------------------------------

``` python
"""add post table

Revision ID: 3f2a9c1d7b10
Revises: 9b1e4f0a2c33
"""
import sqlalchemy as sa
from alembic import op

revision = "3f2a9c1d7b10"
down_revision = "9b1e4f0a2c33"


def upgrade() -> None:
    op.create_table(
        "post",
        sa.Column("id", sa.Integer(), nullable=False),
        sa.Column("author_id", sa.Integer(), nullable=False),
        sa.Column("title", sa.String(length=200), nullable=False),
        sa.Column("body", sa.Text(), nullable=False),
        sa.ForeignKeyConstraint(
            ["author_id"], ["user_account.id"], name=op.f("fk_post_author_id_user_account"), ondelete="CASCADE"
        ),
        sa.PrimaryKeyConstraint("id", name=op.f("pk_post")),
    )
    op.create_index(op.f("ix_post_author_id"), "post", ["author_id"])


def downgrade() -> None:
    op.drop_index(op.f("ix_post_author_id"), table_name="post")
    op.drop_table("post")
```

The constraint names come from the naming convention in section 2. `op.f()` marks a name as already final, so the convention is not applied a second time.

------------------------------------------------------------------------

## 12. What Autogenerate Misses

Autogenerate is a draft, not a migration. These are the cases where the draft is wrong or empty.

| Change | What autogenerate does | What to write instead |
|----|----|----|
| Rename a column or table | Drops the old one and adds a new one. **The data is gone** | `op.alter_column("post", "title", new_column_name="headline")` or `op.rename_table(...)` |
| Move or transform data | Nothing | `op.execute(...)` with SQL, in its own revision |
| Add a value to a PostgreSQL `ENUM` | Nothing | `op.execute("ALTER TYPE status ADD VALUE 'archived'")` |
| Change a `server_default` | Nothing, unless `compare_server_default=True` | Turn the option on (section 9) |
| Enable an extension, such as pgvector | Nothing | `op.execute("CREATE EXTENSION IF NOT EXISTS vector")` in the first revision |
| Views, triggers, SQL functions | Nothing | `op.execute(...)`, with the reverse in `downgrade()` |
| Constraint with no name | Cannot drop or alter it later | The naming convention, from day one |

**Never import your ORM models in a data migration.** The migration must keep working after the models change again. Describe only the columns it needs, inline:

------------------------------------------------------------------------

``` python
def upgrade() -> None:
    op.add_column("post", sa.Column("slug", sa.String(220), nullable=True))

    post = sa.table("post", sa.column("id", sa.Integer), sa.column("title", sa.String), sa.column("slug", sa.String))
    op.execute(post.update().values(slug=sa.func.lower(sa.func.replace(post.c.title, " ", "-"))))

    op.alter_column("post", "slug", nullable=False)
```

That example is also the safe shape for adding a required column to a table that already has rows: add it nullable, fill it, then make it `NOT NULL`. Adding it as `NOT NULL` in one step fails on any non-empty table.

------------------------------------------------------------------------

## 13. Migrations Without Downtime

During a rolling deploy, old and new code run against the same database for a while. So each migration must work with **both** versions of the code. The pattern is called *expand and contract*:

1.  **Expand.** Add the new column or table. Old code ignores it. Deploy code that writes to both old and new.
2.  **Migrate data.** Backfill the new column, in batches on a large table.
3.  **Contract.** Once no running code reads the old column, drop it in a later release.

A rename is therefore three releases, not one. That is the price of no downtime.

Index creation locks writes on the table by default. On a large table, build it concurrently. PostgreSQL forbids that inside a transaction, so step outside Alembic’s transaction:

------------------------------------------------------------------------

``` python
def upgrade() -> None:
    with op.get_context().autocommit_block():
        op.create_index("ix_post_title", "post", ["title"], postgresql_concurrently=True)
```

Run migrations as their own deploy step, before new app containers start. The [FastAPI Stack](../FastAPI/fastapi_stack.llms.md) page explains why not at app startup.

------------------------------------------------------------------------

## 14. Testing Against a Real Database

Test against PostgreSQL, not SQLite. Types, `ON CONFLICT`, `JSONB` and constraint behaviour all differ, and the bugs live in those differences.

Build the schema once per test run with `alembic upgrade head`, so the migrations are tested too. Then give each test a transaction that is rolled back at the end, so tests never see each other’s rows.

------------------------------------------------------------------------

``` python
# tests/conftest.py
import pytest
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine

TEST_DB_URL = "postgresql+psycopg://app:app@localhost:5432/app_test"


@pytest.fixture(scope="session")
def engine():
    # schema comes from `alembic upgrade head`, run against app_test before pytest
    return create_async_engine(TEST_DB_URL)


@pytest.fixture
async def session(engine):
    async with engine.connect() as conn:
        outer = await conn.begin()
        async with AsyncSession(
            bind=conn,
            join_transaction_mode="create_savepoint",
            expire_on_commit=False,
        ) as s:
            yield s
        await outer.rollback()
```

`join_transaction_mode="create_savepoint"` is the key line. When the code under test calls `session.commit()`, SQLAlchemy only releases a savepoint inside the outer transaction. The outer transaction is then rolled back, which removes everything the test wrote. This is the same trick `pytest-django` uses for its `db` fixture.

One more test is worth having: a *stairway* test that runs `upgrade` and `downgrade` for every revision in turn. It catches a broken `downgrade()` before the day you need it.

------------------------------------------------------------------------

## 15. Pitfalls

- **Lazy loading under async.** `MissingGreenlet`. Use `lazy="raise"` and explicit loaders (section 6).
- **`expire_on_commit=True` under async.** Reading an attribute after commit fails. Set it to `False`.
- **Forgetting to index foreign keys.** Django did it for you. SQLAlchemy does not.
- **Naive datetimes.** Use `DateTime(timezone=True)` and `datetime.now(UTC)`.
- **A model module not imported in `env.py`.** Autogenerate proposes dropping its table.
- **Applying an autogenerated rename.** It is a drop plus an add. Read every revision before you apply it.
- **ORM models imported in a migration.** It breaks when the model changes later. Use `sa.table()`.
- **Sharing one `AsyncSession` across concurrent tasks.** A session is not safe for concurrent use. One session per request or per task.

------------------------------------------------------------------------

Back to top
