# Django Templates

Here are some great **Django starter templates** depending on what you’re looking for: from barebones setups to full-stack SaaS platforms.

Author

Benedict Thekkel

## 💻 2. **Django Cookiecutter**

The most popular full-stack starter.

- **Repo**: [cookiecutter/cookiecutter-django](https://github.com/cookiecutter/cookiecutter-django)
- **Features**:
  - Docker-ready
  - ASGI/WSGI support
  - Django Channels
  - Celery + Redis
  - PostgreSQL
  - Whitenoise
  - Gunicorn/Uvicorn
  - Caddy/Nginx optional
  - Production scripts + Terraform support

⚙️ **Use:**

``` bash
pip install cookiecutter
cookiecutter https://github.com/cookiecutter/cookiecutter-django
```

------------------------------------------------------------------------

## ⚡ 3. **Django React/Vue Starter Kits**

### 🧩 Django + React

- **Repo**: [justdjango/django-react-boilerplate](https://github.com/justdjango/django-react-boilerplate)
- **Features**:
  - Webpack + React frontend
  - JWT Auth
  - DRF backend

### 🔧 Django + Vue

- **Repo**: [gtalarico/django-vue-template](https://github.com/gtalarico/django-vue-template)
- **Features**:
  - Webpack + Vue
  - DRF backend
  - Hot-reloading
  - SPA friendly

------------------------------------------------------------------------

## 🚀 4. **Django SaaS Boilerplates**

### 🧠 [saasitive/django-saas-boilerplate](https://github.com/saasitive/django-saas-boilerplate)

- Stripe subscriptions
- Teams
- Invitations
- Docker
- Tailwind + HTMX

### 💼 [vintasoftware/django-react-boilerplate](https://github.com/vintasoftware/django-react-boilerplate)

- Celery
- Stripe
- i18n
- Full test coverage

------------------------------------------------------------------------

## 🧪 5. **For Testing or Teaching**

- [django-startproject](https://github.com/jpadilla/django-startproject)
- [DjangoX by W.S. Vincent](https://github.com/wsvincent/djangox): for educational purposes, includes login/logout/register templates.

------------------------------------------------------------------------

## ✅ Comparison Table

| Template | Frontend | Docker | Async | Celery | DB | Ideal For |
|----|----|----|----|----|----|----|
| cookiecutter-django | Optional | ✅ | ✅ | ✅ | Postgres | Full Production Apps |
| django-starter (Vincent) | ❌ | ❌ | ❌ | ❌ | SQLite | Minimal Setup |
| django-react-boilerplate | React | ✅ | ✅ | ✅ | Postgres | SPAs |
| saasitive django boilerplate | HTMX | ✅ | ✅ | ✅ | Postgres | SaaS Projects |
| django-vue-template | Vue | ✅ | ✅ | ❌ | SQLite | SPAs |

------------------------------------------------------------------------

Back to top
