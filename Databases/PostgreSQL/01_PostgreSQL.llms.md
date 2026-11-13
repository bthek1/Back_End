# PostgreSQL Overview

Here’s a comprehensive, forward-thinking overview of **PostgreSQL’s database structure**, with diagrams, examples, and best practices.

Author

Benedict Thekkel

------------------------------------------------------------------------

## 🧠 1. **PostgreSQL Architecture Overview**

    Cluster → Databases → Schemas → Tables → Columns
                            ↘      ↘
                           Views   Functions

| Layer        | Description                                              |
|--------------|----------------------------------------------------------|
| **Cluster**  | The entire PostgreSQL server instance (1+ databases)     |
| **Database** | A single logical database (separate namespaces)          |
| **Schema**   | Logical grouping inside a database (`public` by default) |
| **Table**    | Actual data storage (rows and columns)                   |
| **Column**   | Field in a table with a data type                        |
| **Row**      | A record (tuple) in a table                              |
| **Index**    | Improves lookup speed                                    |
| **View**     | Saved SQL query (virtual table)                          |
| **Function** | Procedural logic (in PL/pgSQL, SQL, Python, etc.)        |

------------------------------------------------------------------------

## 🗂 2. **Directory Hierarchy (Linux/Unix)**

| Directory    | Description                           |
|--------------|---------------------------------------|
| `pg_data/`   | Base directory                        |
| `base/`      | Actual database data files            |
| `pg_wal/`    | Write-Ahead Logs                      |
| `pg_tblspc/` | Tablespace links                      |
| `global/`    | Global metadata (roles, transactions) |

------------------------------------------------------------------------

## 🔢 3. **Data Types**

| Type Category     | Examples                            |
|-------------------|-------------------------------------|
| Numeric           | `int`, `float`, `decimal`, `serial` |
| Character         | `char(n)`, `varchar(n)`, `text`     |
| Temporal          | `date`, `timestamp`, `interval`     |
| Boolean           | `boolean`                           |
| UUID              | `uuid`                              |
| Array             | `integer[]`, `text[]`               |
| JSON/Binary       | `json`, `jsonb`, `bytea`            |
| Geometric/Network | `point`, `cidr`, `inet`             |
| Custom            | `enum`, `composite`, `domain`       |

✅ Use `jsonb` over `json` for indexing and performance.

------------------------------------------------------------------------

## 🔑 4. **Schema Objects**

| Object          | Description                                |
|-----------------|--------------------------------------------|
| **Tables**      | Store data in rows                         |
| **Indexes**     | Accelerate queries                         |
| **Views**       | Stored SQL queries                         |
| **Sequences**   | Auto-increment counters (used in `serial`) |
| **Functions**   | Reusable code blocks                       |
| **Triggers**    | Hooks before/after changes                 |
| **Constraints** | Rules (e.g. `UNIQUE`, `CHECK`)             |

------------------------------------------------------------------------

## ⚙️ 5. **System Catalogs**

All metadata is stored in system tables (prefixed with `pg_`).

Examples:

``` sql
SELECT * FROM pg_tables WHERE schemaname = 'public';
SELECT * FROM pg_indexes WHERE tablename = 'my_table';
```

Useful views:

- `information_schema.tables`
- `pg_stat_activity`
- `pg_locks`

------------------------------------------------------------------------

## 📈 6. **Index Types**

| Index Type    | Use Case                            |
|---------------|-------------------------------------|
| **B-tree**    | Default, great for equality/range   |
| **Hash**      | Fast equality checks (less used)    |
| **GIN**       | Full-text search, `jsonb`, arrays   |
| **GiST**      | Geospatial or custom indexing       |
| **BRIN**      | Very large tables with ordered data |
| **Partial**   | Only for subset of rows             |
| **Composite** | Index on multiple columns           |

``` sql
CREATE INDEX idx_name ON table(column);
CREATE INDEX idx_gin ON documents USING gin(jsonb_column);
```

------------------------------------------------------------------------

## 🧰 7. **Constraints & Relationships**

| Constraint    | Purpose                                  |
|---------------|------------------------------------------|
| `PRIMARY KEY` | Uniquely identifies a row                |
| `FOREIGN KEY` | Enforces relationships                   |
| `UNIQUE`      | Ensures values are unique                |
| `CHECK`       | Validates conditions (`CHECK (age > 0)`) |
| `NOT NULL`    | Column must have a value                 |

Relational example:

``` sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  email TEXT UNIQUE
);

CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INT REFERENCES users(id)
);
```

------------------------------------------------------------------------

## 🔄 8. **Views vs Materialized Views**

| Type              | Description              |
|-------------------|--------------------------|
| View              | Virtual table (dynamic)  |
| Materialized View | Cached, can be refreshed |

``` sql
CREATE MATERIALIZED VIEW recent_orders AS
SELECT * FROM orders WHERE created_at > NOW() - interval '7 days';
```

------------------------------------------------------------------------

## 🚀 9. **Advanced Features**

| Feature                       | Benefit                        |
|-------------------------------|--------------------------------|
| `jsonb`                       | Semi-structured data           |
| `ARRAY`                       | Native array support           |
| `CTE (WITH)`                  | Reusable subqueries            |
| `Window Functions`            | Running totals, rankings       |
| `Partitioning`                | Scale large tables efficiently |
| `Inheritance`                 | Table OOP-style extension      |
| `Foreign Data Wrappers (FDW)` | Query other databases/files    |

Example:

``` sql
SELECT name, AVG(score) OVER (PARTITION BY subject) FROM results;
```

------------------------------------------------------------------------

## 📊 10. **Monitoring and Performance**

| Tool or View         | Purpose                       |
|----------------------|-------------------------------|
| `pg_stat_activity`   | See current queries           |
| `pg_stat_statements` | Track slow queries            |
| `EXPLAIN ANALYZE`    | Query plan + execution timing |
| `VACUUM`             | Clean up dead tuples          |
| `autovacuum`         | Automatic garbage collection  |

``` sql
EXPLAIN ANALYZE SELECT * FROM large_table WHERE id = 1000;
```

------------------------------------------------------------------------

## 🔐 11. **Security Structure**

| Element       | Purpose                                   |
|---------------|-------------------------------------------|
| Roles & Users | Role-based access control (`CREATE ROLE`) |
| Privileges    | `GRANT SELECT ON table TO analyst;`       |
| Row-Level Sec | `CREATE POLICY` for per-user filtering    |
| Encryption    | TLS + `pgcrypto`                          |

------------------------------------------------------------------------

## 🌍 12. **Extension Ecosystem**

| Extension     | Use Case                      |
|---------------|-------------------------------|
| `postgis`     | Geospatial types and queries  |
| `pg_trgm`     | Text similarity, fuzzy search |
| `uuid-ossp`   | UUID generation               |
| `citext`      | Case-insensitive text         |
| `timescaledb` | Time-series data              |
| `pg_partman`  | Partition management          |

``` sql
CREATE EXTENSION IF NOT EXISTS postgis;
```

------------------------------------------------------------------------

## 🔄 13. **Backup, Restore, and Migrations**

| Tool                            | Use Case                       |
|---------------------------------|--------------------------------|
| `pg_dump`                       | Dump single DB                 |
| `pg_restore`                    | Restore from `.dump` or `.sql` |
| `pg_basebackup`                 | Entire cluster backup          |
| `psql`                          | Interactive SQL shell          |
| `alembic` / `django-migrations` | Schema evolution               |

------------------------------------------------------------------------

## 📌 Summary Table

| Category    | Key Concepts                                       |
|-------------|----------------------------------------------------|
| Structure   | Cluster → DB → Schema → Tables                     |
| Data Types  | Rich: JSONB, ARRAY, GEOMETRY                       |
| Indexes     | B-Tree, GIN, BRIN, Composite                       |
| Features    | CTEs, Views, Partitions, Extensions                |
| Performance | VACUUM, ANALYZE, EXPLAIN                           |
| Security    | Roles, RLS, TLS, Audit                             |
| Tools       | `pg_dump`, `psql`, `pgadmin`, `pg_stat_statements` |

------------------------------------------------------------------------

Back to top
