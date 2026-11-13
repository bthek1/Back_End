# Indexing and Caching

Let’s break down **query optimization** with **indexing** and **caching** in Django (and databases in general). These are key tools to speed up data retrieval, reduce load, and scale your application.

Author

Benedict Thekkel

## 🎯 **Goal of Query Optimization**

- **Reduce query execution time** ⏱️\
- **Minimize database load** 💾\
- **Improve overall app performance** 🚀

------------------------------------------------------------------------

## 1️⃣ **Indexing in Django (and Databases)**

### 📖 **What is an Index?**

An **index** is a data structure (like a sorted list or B-tree) that makes **lookup operations fast**.

Imagine an index at the back of a textbook. Instead of reading every page, you jump to the exact location.

------------------------------------------------------------------------

### ✅ **Why Use Indexes?**

- Speed up **SELECT**, **JOIN**, **ORDER BY**, **WHERE**, and **GROUP BY** queries.
- Reduce **full table scans** (which are slow on large datasets).

------------------------------------------------------------------------

### 🛠️ **How to Create Indexes in Django**

#### **1. Automatic Indexes**

- Django automatically creates indexes for **primary keys** and **unique fields**.

#### **2. Add Indexes on Other Fields**

In `models.py`:

``` python
from django.db import models

class Player(models.Model):
    name = models.CharField(max_length=100)
    score = models.IntegerField(db_index=True)  # Create an index on score
```

#### **3. Multi-Column Indexes**

``` python
class Player(models.Model):
    name = models.CharField(max_length=100)
    country = models.CharField(max_length=100)

    class Meta:
        indexes = [
            models.Index(fields=['country', 'name']),
        ]
```

------------------------------------------------------------------------

### 🔍 **When to Use Indexes**

| Query Type           | Index Field(s)    |
|----------------------|-------------------|
| `WHERE` filters      | Filter columns    |
| `ORDER BY` sorting   | Sorted column     |
| `JOIN` conditions    | ForeignKey fields |
| `UNIQUE` constraints | Enforced fields   |

------------------------------------------------------------------------

### 🚨 **When NOT to Use Indexes**

- Columns that **frequently update** (write-heavy workloads). Indexes slow down INSERT/UPDATE/DELETE.
- Columns with **low cardinality** (few unique values, e.g., boolean flags).

------------------------------------------------------------------------

### 🔧 **Check Your Indexes**

In PostgreSQL (via `psql`):

``` sql
\d tablename
```

You’ll see **indexes**, **constraints**, etc.

------------------------------------------------------------------------

## 2️⃣ **Caching in Django**

### 📖 **What is Caching?**

Caching stores **frequently used data** in a **fast-access layer** (memory) to reduce **database load**.

------------------------------------------------------------------------

### ✅ **Why Use Caching?**

- Reduces **database queries**
- Speeds up **response time**
- Supports **high concurrency** apps (leaderboards, dashboards, etc.)

------------------------------------------------------------------------

### 🛠️ **Types of Caching in Django**

| Cache Type | Use Case | Example |
|----|----|----|
| **Per-View Cache** | Cache entire view responses | `@cache_page` decorator |
| **Template Cache** | Cache parts of templates | `{% cache 600 my_key %}` |
| **Low-Level Cache** | Cache arbitrary data (queries, objects) | `cache.set()` / `cache.get()` |
| **Database Query Cache** | Cache expensive ORM queries | Custom caching logic |

------------------------------------------------------------------------

### ⚙️ **How to Set Up Caching in Django**

#### 1. **Install Redis or Memcached**

For Redis:

``` bash
pip install django-redis
```

#### 2. **Configure in `settings.py`**

``` python
CACHES = {
    "default": {
        "BACKEND": "django_redis.cache.RedisCache",
        "LOCATION": "redis://127.0.0.1:6379/1",
        "OPTIONS": {
            "CLIENT_CLASS": "django_redis.client.DefaultClient",
        }
    }
}
```

------------------------------------------------------------------------

### 🔨 **Cache Example: Per-View**

``` python
from django.views.decorators.cache import cache_page

@cache_page(60 * 15)  # Cache for 15 minutes
def leaderboard(request):
    # expensive DB logic
    return render(request, "leaderboard.html", context)
```

------------------------------------------------------------------------

### 🔨 **Cache Example: Low-Level Data Caching**

``` python
from django.core.cache import cache

def get_top_players():
    data = cache.get('top_players')

    if data is None:
        data = Player.objects.order_by('-score')[:10]
        cache.set('top_players', data, timeout=300)  # Cache for 5 mins

    return data
```

------------------------------------------------------------------------

## 3️⃣ **Indexing vs Caching (At a Glance)**

| Aspect | Indexing | Caching |
|----|----|----|
| **Location** | Database engine (B-tree) | Memory store (Redis, Memcached) |
| **Purpose** | Speed up **data retrieval** | Reduce **query repetition** |
| **Persistence** | Persistent (on disk) | Temporary (in memory) |
| **Use When** | Querying frequently-used columns | Serving repeated data |

------------------------------------------------------------------------

## 4️⃣ **Advanced Optimizations**

| Technique | How |
|----|----|
| **Select Related / Prefetch Related** | Reduce `N+1` queries when joining related models |
| **Materialized Views** | Pre-compute and store complex queries (PostgreSQL) |
| **Database Partitioning** | Split large tables to speed up queries |
| **CDNs** | Cache static/dynamic API responses closer to users |

------------------------------------------------------------------------

## ✅ **Best Practices for Indexing & Caching**

| Best Practice | Why |
|----|----|
| Profile queries (Django Debug Toolbar) | Know which queries are slow |
| Cache only **expensive** data | Simple queries don’t need caching |
| Invalidate caches on **data changes** | Prevent stale data |
| Index **high-read** fields with filters, joins, sorting | Speed up common lookups |
| Use **Redis** for caching dynamic leaderboard dashboards | Fast, in-memory store |

------------------------------------------------------------------------

## 🏆 **Example: Leaderboard with Indexes and Caching**

### Model

``` python
class Player(models.Model):
    name = models.CharField(max_length=100)
    score = models.IntegerField(db_index=True)
```

### View (with caching)

``` python
from django.core.cache import cache
from django.shortcuts import render

def leaderboard(request):
    top_players = cache.get('leaderboard_top_10')

    if not top_players:
        top_players = Player.objects.order_by('-score')[:10]
        cache.set('leaderboard_top_10', top_players, timeout=300)

    return render(request, 'leaderboard.html', {'players': top_players})
```

------------------------------------------------------------------------

Back to top
