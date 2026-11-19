# Django Constance

Here’s **everything you need to know about `django-constance`**, the dynamic configuration system for Django that allows **admin-editable settings stored in the database**—perfect for **feature flags**, **thresholds**, or **customizable behavior** without code changes or redeploys.

Author

Benedict Thekkel

## 🧠 What Is `django-constance`?

`django-constance` lets you:

✅ Define **configuration variables in `settings.py`** ✅ **Edit them via Django Admin** ✅ Store them in the **database or Redis** ✅ Use them like normal Django settings (e.g. `config.FEATURE_ENABLED`)

------------------------------------------------------------------------

## 📦 Installation

``` bash
pip install django-constance
```

Then add to `settings.py`:

``` python
INSTALLED_APPS += [
    "constance",
    "constance.backends.database",  # or "constance.backends.redisd"
]
```

------------------------------------------------------------------------

## 🛠️ Configuration

### 1. Select Backend

**Database:**

``` python
CONSTANCE_BACKEND = "constance.backends.database.DatabaseBackend"
```

**Redis:**

``` python
CONSTANCE_BACKEND = "constance.backends.redisd.RedisBackend"
CONSTANCE_REDIS_CONNECTION = {
    "host": "localhost",
    "port": 6379,
    "db": 0,
}
```

------------------------------------------------------------------------

### 2. Define Your Config Variables

``` python
from datetime import timedelta

CONSTANCE_CONFIG = {
    "FEATURE_X_ENABLED": (False, "Enable the new feature X toggle"),
    "MAX_UPLOAD_SIZE_MB": (5, "Maximum upload size in MB"),
    "WELCOME_MESSAGE": ("Welcome to Recovery Metrics!", "Homepage welcome message"),
    "SESSION_TIMEOUT": (timedelta(minutes=30), "User session timeout"),
}
```

You can also define types and validators:

``` python
CONSTANCE_CONFIG_FIELDSETS = {
    "General": ["FEATURE_X_ENABLED", "WELCOME_MESSAGE"],
    "Limits": ["MAX_UPLOAD_SIZE_MB", "SESSION_TIMEOUT"],
}
```

------------------------------------------------------------------------

### 3. Run Migrations

``` bash
python manage.py migrate
```

------------------------------------------------------------------------

### 4. Use in Code

``` python
from constance import config

if config.FEATURE_X_ENABLED:
    print("Feature X is enabled!")

print("Max upload size:", config.MAX_UPLOAD_SIZE_MB)
```

You access them like normal settings: `config.<SETTING_NAME>`

------------------------------------------------------------------------

### 5. Access via Django Admin

Go to Django Admin → **“Config”** From here, all defined settings will be editable by superusers.

------------------------------------------------------------------------

## ✨ Advanced Features

### ✅ `config` Is Lazy

You don’t have to worry about startup-time queries—`config` fetches settings **lazily and caches them**.

------------------------------------------------------------------------

### 🧪 Use in Tests

To override during tests:

``` python
from constance.test import override_config

@override_config(FEATURE_X_ENABLED=True)
def test_feature_flag():
    assert config.FEATURE_X_ENABLED
```

------------------------------------------------------------------------

### 🔄 Override Backend

Want to override `config` behavior? You can subclass the backend.

------------------------------------------------------------------------

## 🔒 Security Tip

- Only **trusted superusers** should have access to the `Config` admin section.
- Avoid storing **sensitive secrets** (use `django-environ` or `os.environ` for secrets).

------------------------------------------------------------------------

## 📊 When Should You Use It?

Use `django-constance` when you want to:

| Use Case                       | Use? ✅                       |
|--------------------------------|-------------------------------|
| Feature toggles                | ✅                            |
| Custom user-facing text        | ✅                            |
| Thresholds or limits           | ✅                            |
| Secrets and credentials        | ❌                            |
| Per-user or per-client configs | ❌ (use model fields instead) |

------------------------------------------------------------------------

## 🧩 Alternative: `django-dynamic-preferences`

- Per-user or per-model settings
- More complex forms
- Slower performance

But for **global, site-wide config**, `django-constance` is **simple and production-ready**.

------------------------------------------------------------------------

## 🧪 Example Project

``` python
# settings.py
CONSTANCE_CONFIG = {
    "ENABLE_DAILY_SYNC": (True, "Toggle daily CRM sync job"),
    "DAILY_SYNC_TIME": ("02:00", "Time to run daily CRM sync"),
}

# Usage
from constance import config
if config.ENABLE_DAILY_SYNC:
    print("Starting daily sync at", config.DAILY_SYNC_TIME)
```

------------------------------------------------------------------------

Would you like help setting this up in your project with a feature flag (e.g. `"CRM_BACKFILL_ENABLED"`)?

------------------------------------------------------------------------

Back to top
