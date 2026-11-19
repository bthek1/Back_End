# FastAPI Stack

The standard FastAPI stack: what each layer is, why it is the default, and how the pieces connect in one small working service.

Author

Benedict Thekkel

FastAPI is a thin layer over two other libraries. **Starlette** does the HTTP work: routing, middleware, requests and responses, WebSockets. **Pydantic** does the data work: it validates what comes in and serialises what goes out. FastAPI adds the glue between them, which is dependency injection and an OpenAPI schema generated from your type hints.

That thinness is the point and the cost. Django gives you an ORM, migrations, auth, an admin and a settings system in one box. FastAPI gives you none of those, so a “FastAPI project” is really a stack you assemble. This page fixes that stack to one sensible default per layer, so you do not re-decide it for every new service. The database layer has its own page: [SQLAlchemy and Alembic](../FastAPI/sqlalchemy_alembic.llms.md).

------------------------------------------------------------------------

## 1. The Stack

One choice per layer. The third column says why it is the default rather than the alternatives.

| Layer | Choice | Why this one |
|----|----|----|
| Language | Python 3.13+ | Current typing syntax (`X \| None`, `list[int]`) that FastAPI reads directly |
| Package manager | `uv` | Lockfile, fast installs, and `uv run` replaces activating a venv |
| Framework | FastAPI, installed as `fastapi[standard]` | The `standard` extra brings Uvicorn, the `fastapi` CLI, `httpx` and `email-validator` |
| ASGI server | Uvicorn | ASGI (Asynchronous Server Gateway Interface) is the async successor to WSGI. `fastapi run` is Uvicorn underneath |
| Validation and schemas | Pydantic v2 | Built in. The core is written in Rust, so validation is cheap |
| Settings | `pydantic-settings` | Reads environment variables and `.env` into a typed, validated object |
| ORM | SQLAlchemy 2.0, async | The mature Python ORM. The 2.0 API is fully typed |
| Database driver | psycopg 3 (`psycopg[binary]`) | One driver for sync and async, and the same one the Django stack uses |
| Migrations | Alembic | Written by the SQLAlchemy author. Reads your models to generate migrations |
| Database | PostgreSQL (+ pgvector) | Same as the Django stack |
| Auth | OAuth2 password flow + JWT, with `PyJWT` and `pwdlib[argon2]` | What the FastAPI docs now use. `python-jose` and `passlib` are no longer maintained well |
| Background work | `BackgroundTasks` for small jobs, Celery + Redis for real ones | `BackgroundTasks` runs in the same process after the response, with no retries |
| Outbound HTTP | `httpx.AsyncClient` | Async, and the same library the tests use |
| Testing | `pytest`, `httpx.AsyncClient` with `ASGITransport`, `anyio` | Calls the app in process, with no server and no network |
| Lint and format | `ruff` | One tool for both |
| Type checking | `mypy` | FastAPI and SQLAlchemy 2.0 are both fully typed, so it pays off |
| Metrics and logs | `prometheus-fastapi-instrumentator`, stdlib `logging` to Loki | Plugs into the existing Grafana, Prometheus and Loki setup |
| Container | Docker, one Uvicorn process per container | Scale by adding replicas, not workers inside one container |

**Not on the list: SQLModel.** It merges the SQLAlchemy table class and the Pydantic schema into one class. That saves typing for a tutorial. In a real service the table and the API shape drift apart (hidden columns, computed fields, different create and read shapes), and then one class is doing two jobs badly. Keep them as two classes.

------------------------------------------------------------------------

## 2. How a Request Moves Through It

![](fastapi_request_flow.svg)

FastAPI request flow

1.  Uvicorn accepts the connection and turns it into an ASGI call.
2.  Middleware runs first, for every request: CORS, request logging, timing.
3.  The router matches the path and method to one function, the *path operation*.
4.  FastAPI resolves the function’s **dependencies**: settings, a database session, the current user. Each dependency can have dependencies of its own.
5.  Pydantic validates the path, query and body against the type hints. A failure returns `422` before your code runs.
6.  Your function runs. It talks to the database through the session, and hands slow work to Celery.
7.  FastAPI filters the return value through `response_model` and serialises it to JSON.

Steps 4 and 7 are where FastAPI earns its place. Dependencies replace Django’s middleware-and-`request.user` pattern with plain function parameters. `response_model` is the guard that stops a database column such as `password_hash` from leaking into a response.

------------------------------------------------------------------------

## 3. Project Layout

Group by domain, the same way Django groups by app. Each domain package holds its own models, schemas, router and service.

    app/
      main.py            # create the app, lifespan, include routers
      core/
        config.py        # Settings
        db.py            # engine, sessionmaker, get_session
        models.py        # Base, with the constraint naming convention
        security.py      # password hashing, JWT
        deps.py          # SessionDep, SettingsDep, CurrentUser
      items/
        models.py        # SQLAlchemy tables
        schemas.py       # Pydantic in and out shapes
        router.py        # path operations, thin
        service.py       # business logic, no FastAPI imports
      users/
        ...
    migrations/          # Alembic
    tests/
    pyproject.toml

Keep `service.py` free of FastAPI imports. Then the same logic runs from a Celery task, a CLI script or a test without faking an HTTP request.

Install the stack:

``` bash
uv init --app
uv add "fastapi[standard]" "sqlalchemy[asyncio]" "psycopg[binary]" alembic pydantic-settings pyjwt "pwdlib[argon2]"
uv add --dev pytest anyio ruff mypy
```

------------------------------------------------------------------------

## 4. Settings

`pydantic-settings` reads each field from an environment variable of the same name (case-insensitive), then from `.env`. A field with no default is required, so the app fails at startup if `JWT_SECRET` is missing, not on the first login.

------------------------------------------------------------------------

``` python
# app/core/config.py
from functools import lru_cache

from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    database_url: str = "postgresql+psycopg://app:app@localhost:5432/app"
    jwt_secret: str
    jwt_expire_minutes: int = 30
    cors_origins: list[str] = ["http://localhost:5173"]


@lru_cache
def get_settings() -> Settings:
    return Settings()
```

`get_settings()` with `lru_cache` builds the object once. Endpoints receive it as a dependency rather than importing a module-level global, so a test can swap it with `app.dependency_overrides[get_settings]`.

------------------------------------------------------------------------

## 5. Database Session

One engine per process, created at import. One session per request, handed out by a dependency. The full explanation of engines, sessions and `expire_on_commit` is in [SQLAlchemy and Alembic](../FastAPI/sqlalchemy_alembic.llms.md).

------------------------------------------------------------------------

``` python
# app/core/db.py
from collections.abc import AsyncIterator

from sqlalchemy.ext.asyncio import AsyncSession, async_sessionmaker, create_async_engine

from app.core.config import get_settings

engine = create_async_engine(
    get_settings().database_url,
    pool_size=5,
    max_overflow=10,
    pool_pre_ping=True,
)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)


async def get_session() -> AsyncIterator[AsyncSession]:
    async with SessionLocal() as session:
        yield session
```

A dependency that uses `yield` is a setup and teardown pair. Code before `yield` runs before the endpoint. Code after it (here, closing the session through `async with`) runs after. It is the FastAPI version of a context manager around each request.

Pool size is per process. With 4 replicas, `pool_size=5, max_overflow=10` allows up to 60 connections. Check that against PostgreSQL’s `max_connections` (default 100) before you scale out.

------------------------------------------------------------------------

## 6. Dependencies as Type Aliases

`Annotated` attaches the `Depends(...)` to a type, so the alias reads like a plain type everywhere it is used.

------------------------------------------------------------------------

``` python
# app/core/deps.py
from typing import Annotated

from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

from app.core.config import Settings, get_settings
from app.core.db import get_session

SessionDep = Annotated[AsyncSession, Depends(get_session)]
SettingsDep = Annotated[Settings, Depends(get_settings)]
```

FastAPI caches each dependency **per request**. If the endpoint and `get_current_user` both ask for `SessionDep`, they get the same session, so the user lookup and the endpoint’s queries share one transaction.

------------------------------------------------------------------------

## 7. Model, Schemas, Router

The model is the table. The schemas are the API shapes. They are separate classes on purpose (see section 1).

------------------------------------------------------------------------

``` python
# app/items/models.py
from datetime import datetime

from sqlalchemy import DateTime, String, func
from sqlalchemy.orm import Mapped, mapped_column

from app.core.models import Base


class Item(Base):
    __tablename__ = "item"

    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(200))
    price_cents: Mapped[int]
    created_at: Mapped[datetime] = mapped_column(DateTime(timezone=True), server_default=func.now())
```

``` python
# app/items/schemas.py
from datetime import datetime

from pydantic import BaseModel, ConfigDict, Field


class ItemCreate(BaseModel):
    name: str = Field(min_length=1, max_length=200)
    price_cents: int = Field(ge=0)


class ItemRead(ItemCreate):
    model_config = ConfigDict(from_attributes=True)

    id: int
    created_at: datetime
```

`from_attributes=True` lets Pydantic read a SQLAlchemy object by attribute. It is the equivalent of a DRF `ModelSerializer` reading a model instance.

``` python
# app/items/router.py
from collections.abc import Sequence
from typing import Annotated

from fastapi import APIRouter, HTTPException, Query, status
from sqlalchemy import select

from app.core.deps import SessionDep
from app.items.models import Item
from app.items.schemas import ItemCreate, ItemRead

router = APIRouter(prefix="/items", tags=["items"])


@router.post("", response_model=ItemRead, status_code=status.HTTP_201_CREATED)
async def create_item(data: ItemCreate, session: SessionDep) -> Item:
    item = Item(**data.model_dump())
    session.add(item)
    await session.commit()
    await session.refresh(item)  # load created_at, which the database filled in
    return item


@router.get("/{item_id}", response_model=ItemRead)
async def get_item(item_id: int, session: SessionDep) -> Item:
    item = await session.get(Item, item_id)
    if item is None:
        raise HTTPException(status.HTTP_404_NOT_FOUND, "Item not found")
    return item


@router.get("", response_model=list[ItemRead])
async def list_items(
    session: SessionDep,
    limit: Annotated[int, Query(ge=1, le=100)] = 20,
    offset: Annotated[int, Query(ge=0)] = 0,
) -> Sequence[Item]:
    result = await session.scalars(select(Item).order_by(Item.id).limit(limit).offset(offset))
    return result.all()
```

The return annotation says what the function returns (an ORM object). `response_model` says what the client receives. When both are present, `response_model` wins, and that is what makes the filtering reliable.

The commit is explicit, in the endpoint. One request is one transaction, and you can see where it ends. For logic bigger than a few lines, move it into `service.py` and keep the commit at the edge.

------------------------------------------------------------------------

## 8. The App and Its Lifespan

`lifespan` is an async context manager that wraps the whole life of the process. Startup work goes before `yield`, shutdown work after. It replaces the old `@app.on_event("startup")` hooks, which are deprecated.

------------------------------------------------------------------------

``` python
# app/main.py
from collections.abc import AsyncIterator
from contextlib import asynccontextmanager

from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

from app.core.config import get_settings
from app.core.db import engine
from app.items.router import router as items_router


@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncIterator[None]:
    yield
    await engine.dispose()


app = FastAPI(title="Items API", lifespan=lifespan)

app.add_middleware(
    CORSMiddleware,
    allow_origins=get_settings().cors_origins,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(items_router, prefix="/api")


@app.get("/health")
async def health() -> dict[str, str]:
    return {"status": "ok"}
```

The lifespan is also where you load anything expensive exactly once per process. An ONNX Runtime session or a `transformers` model loaded here and stored on `app.state` turns this same skeleton into a model-serving API, which is one of the places FastAPI is clearly the better tool than Django.

Run it:

``` bash
uv run fastapi dev app/main.py     # auto-reload, http://127.0.0.1:8000/docs
```

`/docs` is Swagger UI and `/redoc` is ReDoc, both generated from the type hints. `/openapi.json` is the raw schema, which is what you feed to a TypeScript client generator for the React side. This is what `drf-spectacular` does for Django, built in.

------------------------------------------------------------------------

## 9. Authentication

The FastAPI docs pattern: a `/token` endpoint checks the password and returns a signed JWT (JSON Web Token). Every protected endpoint declares a `CurrentUser` dependency that verifies the token and loads the user.

------------------------------------------------------------------------

``` python
# app/core/security.py
from datetime import UTC, datetime, timedelta

import jwt
from pwdlib import PasswordHash

password_hash = PasswordHash.recommended()  # Argon2id


def hash_password(password: str) -> str:
    return password_hash.hash(password)


def verify_password(password: str, hashed: str) -> bool:
    return password_hash.verify(password, hashed)


def create_access_token(subject: str, secret: str, expire_minutes: int) -> str:
    expire = datetime.now(UTC) + timedelta(minutes=expire_minutes)
    return jwt.encode({"sub": subject, "exp": expire}, secret, algorithm="HS256")
```

``` python
# app/core/deps.py (continued)
import jwt
from fastapi import HTTPException, status
from fastapi.security import OAuth2PasswordBearer

from app.users.models import User

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/auth/token")


async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    session: SessionDep,
    settings: SettingsDep,
) -> User:
    unauthorized = HTTPException(
        status.HTTP_401_UNAUTHORIZED, "Invalid credentials", headers={"WWW-Authenticate": "Bearer"}
    )
    try:
        payload = jwt.decode(token, settings.jwt_secret, algorithms=["HS256"])
    except jwt.InvalidTokenError:
        raise unauthorized
    user = await session.get(User, int(payload["sub"]))
    if user is None or not user.is_active:
        raise unauthorized
    return user


CurrentUser = Annotated[User, Depends(get_current_user)]
```

An endpoint is now protected by adding one parameter:

``` python
@router.get("/me", response_model=UserRead)
async def me(user: CurrentUser) -> User:
    return user
```

Permissions follow the same pattern. A dependency such as `require_role("admin")` returns a function that depends on `CurrentUser` and raises `403`. That replaces DRF’s `permission_classes`.

For a browser SPA, consider the token in an `HttpOnly` cookie instead of `localStorage`, so page scripts cannot read it. That needs CSRF (cross-site request forgery) protection in return, which FastAPI does not provide out of the box.

------------------------------------------------------------------------

## 10. `async def` or `def`

This is the one rule that FastAPI punishes you for breaking.

| You write | FastAPI runs it | Use it when |
|----|----|----|
| `async def` | On the event loop, in the main thread | Everything slow inside is `await`ed: async SQLAlchemy, `httpx.AsyncClient`, Redis async client |
| `def` | In a thread pool (40 threads by default) | You call blocking code: `requests`, a sync driver, file IO, a CPU-heavy function |

A blocking call inside `async def` (for example `requests.get`, `time.sleep`, or a sync database query) stalls the event loop. While it waits, **every other request in that process waits too**. The symptom is latency that rises with load for no visible reason.

CPU-heavy work, such as model inference, blocks either way. Run it with `await anyio.to_thread.run_sync(fn, ...)`, or move it to Celery.

------------------------------------------------------------------------

## 11. Testing

`httpx.AsyncClient` with `ASGITransport` calls the app directly in process. There is no server and no port. `app.dependency_overrides` swaps any dependency, which is how tests get a test database session or a fake user.

------------------------------------------------------------------------

``` python
# tests/conftest.py
import pytest
from httpx import ASGITransport, AsyncClient

from app.core.db import get_session
from app.main import app


@pytest.fixture
def anyio_backend() -> str:
    return "asyncio"


@pytest.fixture
async def client(session):  # `session` is the rollback fixture from the SQLAlchemy page
    async def override_session():
        yield session

    app.dependency_overrides[get_session] = override_session
    async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as c:
        yield c
    app.dependency_overrides.clear()
```

``` python
# tests/test_items.py
import pytest


@pytest.mark.anyio
async def test_create_then_get_item(client):
    created = await client.post("/api/items", json={"name": "bolt", "price_cents": 50})
    assert created.status_code == 201

    fetched = await client.get(f"/api/items/{created.json()['id']}")
    assert fetched.json()["name"] == "bolt"


@pytest.mark.anyio
async def test_negative_price_is_rejected(client):
    r = await client.post("/api/items", json={"name": "bolt", "price_cents": -1})
    assert r.status_code == 422
```

`ASGITransport` does not run the lifespan. That is usually what you want in unit tests. If a test needs startup work, wrap the app in `asgi-lifespan`’s `LifespanManager`.

The rollback fixture that gives each test a clean database is in [SQLAlchemy and Alembic](../FastAPI/sqlalchemy_alembic.llms.md#testing-against-a-real-database).

------------------------------------------------------------------------

## 12. Running in Production

``` dockerfile
FROM python:3.13-slim
COPY --from=ghcr.io/astral-sh/uv:latest /uv /bin/uv
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project
COPY . .
RUN uv sync --frozen --no-dev
CMD ["/app/.venv/bin/fastapi", "run", "app/main.py", "--port", "8000"]
```

Pin the `uv` image tag in real use; `latest` is here only to keep the example short. Copying `pyproject.toml` and `uv.lock` before the source means the dependency layer is cached until the lockfile changes.

Four rules for deployment:

1.  **One process per container.** Let Docker, Compose or the orchestrator run replicas. Use `--workers N` only on a plain VM with no orchestrator.
2.  **Migrations are a separate step.** Run `alembic upgrade head` as a one-off job before the new containers start. Never in `lifespan`: several replicas starting at once race each other on the same migration.
3.  **Behind a reverse proxy, trust its headers.** Nginx or a load balancer terminates TLS, so pass Uvicorn `--proxy-headers` and `--forwarded-allow-ips` set to the proxy’s address. Otherwise generated URLs say `http` and client IPs are the proxy’s.
4.  **No state in process memory.** Each replica has its own memory. Sessions, rate-limit counters and caches go in Redis.

------------------------------------------------------------------------

## 13. Coming From Django

| Django / DRF | FastAPI stack |
|----|----|
| `settings.py` + `django-environ` | `pydantic-settings` `Settings` |
| `models.Model` | SQLAlchemy `DeclarativeBase` subclass |
| `makemigrations` / `migrate` | `alembic revision --autogenerate` / `alembic upgrade head` |
| `Serializer` / `ModelSerializer` | Pydantic models, one per shape (`ItemCreate`, `ItemRead`) |
| `APIView` / `ViewSet` | Path operation functions on an `APIRouter` |
| `urls.py` + `DefaultRouter` | `app.include_router(...)` |
| `authentication_classes` + `request.user` | `CurrentUser` dependency |
| `permission_classes` | Dependencies that raise `403` |
| Middleware | Starlette middleware (`app.add_middleware`) |
| `drf-spectacular` | Built in: `/docs`, `/redoc`, `/openapi.json` |
| Django admin | Nothing built in. `sqladmin` is the closest add-on |
| `pytest-django` `db` fixture | Your own rollback fixture |
| Celery | Celery, unchanged |
| Channels | Built in: `@app.websocket(...)` |

**When to choose which.** Pick Django when the product is many CRUD models, needs an admin for staff, and is one deployable unit. That describes most business applications. Pick FastAPI when the service is API-only and small, is dominated by waiting on IO (streaming LLM responses, proxying other APIs, WebSockets), or serves an ML model. Both can sit behind the same React SPA and share the same PostgreSQL and Celery.

------------------------------------------------------------------------

## 14. Pitfalls

- **Blocking call inside `async def`.** Stalls the whole process. See section 10.
- **No `response_model`.** Returning an ORM object without one can expose every column, including `password_hash`.
- **Lazy loading a relationship in async code.** Raises `MissingGreenlet`. Load relationships explicitly. See [SQLAlchemy and Alembic](../FastAPI/sqlalchemy_alembic.llms.md).
- **An engine per request.** Each engine owns a connection pool. Create one per process, at import.
- **Migrations in `lifespan`.** Replicas race. Run them as a deploy step.
- **Reading settings at import in many places.** Tests then cannot override them. Go through `get_settings()` or `SettingsDep`.
- **`BackgroundTasks` for important work.** It has no retry and dies with the process. Use Celery for anything you would miss.

------------------------------------------------------------------------

Back to top
