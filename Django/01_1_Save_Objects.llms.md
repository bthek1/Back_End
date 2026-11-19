# Saving objects in Django

Saving objects in Django

Author

Benedict Thekkel

Here’s a table showing which methods internally use the `save()` method and which do not:

| **Method** | **Uses `save()`?** | **Explanation** |
|----|----|----|
| `Model.save()` | ✅ Yes | Directly calls `save()` to persist changes. |
| `QuerySet.create()` | ✅ Yes | Creates an object and calls `save()` internally. |
| `QuerySet.update()` | ❌ No | Directly executes an SQL `UPDATE` query without calling `save()`. |
| `update_or_create()` | ✅ Yes (for creation) / ❌ No (for update) | Calls `save()` for new objects but uses `update()` for existing objects. |
| `get_or_create()` | ✅ Yes (for creation) | Calls `save()` if a new object is created. |
| `bulk_create()` | ❌ No | Directly inserts objects into the database without calling `save()`. |
| `bulk_update()` | ❌ No | Directly executes an SQL `UPDATE` query without calling `save()`. |
| Raw SQL Queries | ❌ No | Bypasses Django ORM and executes SQL directly. |
| Signals | ✅ Depends | `save()` is typically called by the triggering event, which may then trigger signals. |
| `F()` Expressions | ❌ No | Works directly at the database level with `update()` or queries. |
| Custom Managers | ✅ Depends | Custom logic can call `save()` depending on implementation. |
| Django Admin | ✅ Yes | Uses `save()` when saving objects through the admin interface. |
| Django Forms | ✅ Yes | Calls `save()` when a valid form instance saves an object. |
| Fixtures | ❌ No | Data is loaded directly into the database without calling `save()`. |
| Migrations | ✅ Depends | If the migration script uses ORM methods like `create()` or `save()`, then `save()` is used. |

### Key Insights

- Methods like `save()`, `create()`, and `get_or_create()` call `save()`.
- Bulk operations like `bulk_create()` and `update()` do not call `save()`, making them faster but skipping custom logic or signal triggers.
- Certain methods (e.g., `update_or_create()`) conditionally call `save()` depending on whether the object is being created or updated.

Here’s a table showing which Django signals are triggered when different object creation and update methods are used:

| **Method** | **`pre_save`** | **`post_save`** | **`pre_init`** | **`post_init`** | **`pre_delete`** | **`post_delete`** | **Explanation** |
|----|----|----|----|----|----|----|----|
| **`Model.save()`** | ✅ Yes | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No | Both `pre_save` and `post_save` are triggered. |
| **`QuerySet.create()`** | ✅ Yes | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No | Internally calls `save()` on the object, triggering `pre_save` and `post_save`. |
| **`QuerySet.update()`** | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | Direct SQL `UPDATE` query; no signals are triggered. |
| **`update_or_create()`** | ✅ Yes (if created) | ✅ Yes (if created) | ❌ No | ❌ No | ❌ No | ❌ No | Signals are triggered only if a new object is created (via `save()`). Updates bypass signals. |
| **`get_or_create()`** | ✅ Yes (if created) | ✅ Yes (if created) | ❌ No | ❌ No | ❌ No | ❌ No | Signals are triggered only if a new object is created (via `save()`). |
| **`bulk_create()`** | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | Direct SQL `INSERT` query; no signals are triggered. |
| **`bulk_update()`** | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | Direct SQL `UPDATE` query; no signals are triggered. |
| **Raw SQL** | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | No Django signals are triggered; database operations are manual. |
| **Signals (`save()`)** | ✅ Yes | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No | Signals are triggered when `save()` is explicitly called. |
| **`F()` Expressions** | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | Updates using `F()` are performed directly in the database and bypass signals. |
| **Custom Managers** | Depends on implementation | Depends on implementation | ❌ No | ❌ No | ❌ No | ❌ No | Signals are triggered only if the manager explicitly calls `save()`. |
| **Django Admin** | ✅ Yes | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No | Uses the `save()` method internally, triggering `pre_save` and `post_save`. |
| **Django Forms** | ✅ Yes | ✅ Yes | ❌ No | ❌ No | ❌ No | ❌ No | When `form.save()` is invoked, `pre_save` and `post_save` are triggered. |
| **Fixtures** | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | ❌ No | Inserts data directly into the database; no signals are triggered. |
| **Migrations** | Depends on implementation | Depends on implementation | ❌ No | ❌ No | ❌ No | ❌ No | Signals are triggered only if the migration code explicitly calls `save()`. |

------------------------------------------------------------------------

### Key Points

1.  **Triggered Signals**:
    - `pre_save` and `post_save` are triggered when `save()` is called directly or indirectly (e.g., `QuerySet.create()`, `update_or_create()` during creation).
    - `pre_delete` and `post_delete` are triggered when an object is deleted explicitly using `delete()`.
2.  **Not Triggered**:
    - Methods like `bulk_create()`, `bulk_update()`, and `QuerySet.update()` bypass Django’s ORM logic and perform raw SQL operations, so no signals are triggered.
3.  **Conditional Signal Triggering**:
    - `update_or_create()` and `get_or_create()` trigger signals only when creating a new object.
4.  **Custom Managers and Migrations**:
    - Signals depend on whether `save()` is explicitly called in custom logic.

This table helps you understand when signals are fired for various methods in Django.

Back to top
