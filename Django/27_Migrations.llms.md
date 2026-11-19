# Migrations

Migrations are Django’s way of handling database schema changes. They allow you to create, modify, and delete database tables and fields **without losing data**.

Author

Benedict Thekkel

## **🔹 1. Understanding Migrations**

Migrations are:

- **Auto-generated files** that store changes to your models.
- **Applied to the database** to create/update/delete tables and fields.
- **Version-controlled** so changes can be tracked over time.
- **State-aware**: Django keeps both a **model state** (from migration files) and a **DB state** (via the `django_migrations` table).

They **ensure consistency** between the database and Django models.

------------------------------------------------------------------------

## **🔹 2. Key Migration Commands**

| **Command** | **Description** |
|----|----|
| `python manage.py makemigrations` | Generates migration files based on model changes. |
| `python manage.py migrate` | Applies migrations to the database. |
| `python manage.py showmigrations` | Displays a list of applied and unapplied migrations. |
| `python manage.py sqlmigrate <app> <migration_number>` | Shows the raw SQL for a migration. |
| `python manage.py migrate <app> zero` | Rolls back all migrations for an app. |
| `python manage.py makemigrations --dry-run --check` | Check if new migrations are needed (CI safe). |
| `python manage.py migrate --plan` | Preview what will run in order. |
| `python manage.py migrate --fake-initial` | Mark initial migrations as applied without running them. |
| `python manage.py makemigrations --merge` | Merge conflicting migration histories. |

------------------------------------------------------------------------

## **🔹 3. How Migrations Work**

1.  **Detect model changes** → `makemigrations` generates migration files (operations).
2.  **Migration files form a DAG** → dependencies define order.
3.  **Apply with `migrate`** → runs SQL + records applied migrations in `django_migrations`.
4.  **State sync** → Django assumes DB matches its recorded state.

👉 Drift (manual DB edits) breaks this. Use `--fake`, `SeparateDatabaseAndState`, or fresh migrations to realign.

------------------------------------------------------------------------

## **🔹 4. Common Migration Operations**

- `CreateModel` → new table
- `AddField` → new column (with `default` or `null=True`)
- `RenameField` / `RenameModel` → keeps data
- `RemoveField` → drops column
- `AlterField` → type/option changes (may warn if risky)
- `RunPython` / `RunSQL` → data or schema logic
- `AddIndex`, `AddConstraint`, `CreateExtension` (Postgres-specific)

✅ Django usually **detects renames vs. add/remove** if done in isolation.

------------------------------------------------------------------------

## **🔹 5. Rolling Back Migrations**

- To a previous migration:

  ``` bash
  python manage.py migrate app_name 0002
  ```

- Reset an app:

  ``` bash
  python manage.py migrate app_name zero
  ```

- Fake back:

  ``` bash
  python manage.py migrate app_name 0005 --fake
  ```

------------------------------------------------------------------------

## **🔹 6. Handling Migration Issues**

| **Issue**                | **Fix**                                    |
|--------------------------|--------------------------------------------|
| Missing migrations error | Run `makemigrations` + `migrate`           |
| Table already exists     | `migrate --fake-initial`                   |
| FK constraint fails      | Ensure related app migrated first          |
| Corrupt migration        | Roll back → delete bad file → regenerate   |
| Drift (manual DB change) | Use `SeparateDatabaseAndState` or `--fake` |

------------------------------------------------------------------------

## **🔹 7. State-only vs DB-only Migrations**

Use `SeparateDatabaseAndState`:

``` python
migrations.SeparateDatabaseAndState(
    database_operations=[],
    state_operations=[
        migrations.RenameField("customer", "name", "full_name")
    ]
)
```

👉 Useful when the DB is already correct, but Django’s state is not (or vice versa).

------------------------------------------------------------------------

## **🔹 8. Large Tables & Zero-Downtime**

- **Add column**: always nullable or with default.

- **Two-step change for non-null**:

  1.  Add nullable → backfill → update app code to write both.
  2.  Make column required → drop old.

- **Indexes**:

  ``` python
  from django.contrib.postgres.operations import CreateIndexConcurrently
  class Migration(migrations.Migration):
      atomic = False
      operations = [
          CreateIndexConcurrently(
              model_name="order",
              index=models.Index(fields=["created_at"], name="order_created_idx")
          )
      ]
  ```

- **Set `atomic = False`** if long ops (Postgres concurrent indexes can’t run in a transaction).

------------------------------------------------------------------------

## **🔹 9. Migrating Models Between Apps**

1.  Keep same DB table via `db_table`.
2.  Run `makemigrations app_b`.
3.  Apply with `--fake-initial`.

👉 Ensures Django points at the new app, but DB untouched.

------------------------------------------------------------------------

## **🔹 10. Custom Data Migrations**

``` python
def forwards(apps, schema_editor):
    Product = apps.get_model("shop", "Product")
    Product.objects.filter(category="").update(category="General")

def backwards(apps, schema_editor):
    Product = apps.get_model("shop", "Product")
    Product.objects.update(category="")

class Migration(migrations.Migration):
    operations = [migrations.RunPython(forwards, backwards)]
```

⚠️ Always use `apps.get_model` (historical models), never direct imports.

------------------------------------------------------------------------

## **🔹 11. Squashing & Merging**

- **Squash long histories**:

  ``` bash
  python manage.py squashmigrations app 0001 0050
  ```

- **Merge conflicts**:

  ``` bash
  python manage.py makemigrations --merge
  ```

------------------------------------------------------------------------

## **🔹 12. Best Practices**

| ✅ Do                         | ❌ Avoid                             |
|-------------------------------|--------------------------------------|
| Commit migration files        | Ignoring migrations in VCS           |
| Use `--check --dry-run` in CI | Editing old migrations after release |
| Review SQL with `sqlmigrate`  | Manual DB edits without aligning     |
| Write reverse functions       | `RunPython` without rollback         |
| Keep them small & atomic      | Huge multi-change migrations         |

------------------------------------------------------------------------

## **🔹 13. Quick Cheat Sheet**

``` bash
# Generate migrations
python manage.py makemigrations

# Apply migrations
python manage.py migrate

# Roll back app
python manage.py migrate app_name zero

# Preview SQL
python manage.py sqlmigrate app 0005

# Preview plan
python manage.py migrate --plan
```

------------------------------------------------------------------------

Back to top
