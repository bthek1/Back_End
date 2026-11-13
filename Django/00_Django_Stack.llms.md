# Django Stack

Django Stack

Author

Benedict Thekkel

From the project instructions:

**Python 3.14, Django 5.1+, Django REST Framework**

| Layer | Technology |
|----|----|
| Language | Python 3.13 |
| Framework | Django 5.1+ + Django REST Framework |
| Database | PostgreSQL 16 (pgvector extension) |
| DB driver | psycopg3 (`psycopg[binary]`) |
| Auth | JWT via `djangorestframework-simplejwt` |
| Async tasks | Celery + Redis |
| Task results | `django-celery-results` |
| Config | `django-environ` |
| Package manager | `uv` |
| Linting/formatting | `ruff` |
| Type checking | `mypy` |
| Testing | `pytest` + `pytest-django`, `factory-boy`, `faker`, `freezegun` |
| Embeddings | `sentence-transformers` (`BAAI/bge-large-en-v1.5`, local) |
| RAG generation | Anthropic Claude (via `ANTHROPIC_API_KEY`) |
| Web search | `ddgs` (DuckDuckGo) |
| HTTP client | `httpx` |
| LLM chat | Ollama (local, via `httpx`) |

Settings are split into `core/settings/base.py`, `dev.py`, `prod.py`, and `test.py`.

Back to top
