# TimescaleDB

Here’s a comprehensive, **forward-thinking guide to using TimescaleDB in Django**, including **what it is**, **why it matters**, and **how to use it for time-series data analysis, dashboards, and efficient data storage**.

Author

Benedict Thekkel

------------------------------------------------------------------------

## 🔍 What is TimescaleDB?

**TimescaleDB** is a PostgreSQL extension built for **time-series data**, offering:

| Feature | Benefit |
|----|----|
| ⏱️ Hypertables | Automatic time-based partitioning for massive datasets |
| 📊 Continuous Aggregates | Precomputed rollups for fast queries |
| 🧊 Compression | Reduces storage, improves read performance |
| 🔁 Time Bucketing | Easy `GROUP BY time_interval` for summaries |
| 💥 PostgreSQL Compatibility | Works with Django ORM via PostgreSQL backend |

------------------------------------------------------------------------

## 🔧 1. Install and Enable TimescaleDB

### 🐳 Docker Example:

``` yaml
services:
  db:
    image: timescale/timescaledb:latest-pg15
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: password
```

### Manual Install (Linux):

``` bash
sudo apt install timescaledb-postgresql-15
```

Update `postgresql.conf`:

``` ini
shared_preload_libraries = 'timescaledb'
```

Then restart PostgreSQL and enable the extension:

``` sql
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

------------------------------------------------------------------------

## ⚙️ 2. Django Project Setup

### Install dependencies:

``` bash
pip install psycopg2 django
```

### `settings.py`:

``` python
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'yourdb',
        'USER': 'youruser',
        'PASSWORD': 'yourpass',
        'HOST': 'localhost',
        'PORT': '5432',
    }
}
```

------------------------------------------------------------------------

## 📦 3. Django Model for Time-Series Data

``` python
# models.py
from django.db import models

class SensorReading(models.Model):
    timestamp = models.DateTimeField()
    sensor_id = models.CharField(max_length=50)
    temperature = models.FloatField()

    class Meta:
        indexes = [models.Index(fields=["timestamp"])]
```

### Migrate:

``` bash
python manage.py makemigrations
python manage.py migrate
```

------------------------------------------------------------------------

## 🔁 4. Convert to Hypertable

Run this via `psql` or Django `connection.cursor()`:

``` sql
SELECT create_hypertable('yourapp_sensorreading', 'timestamp', if_not_exists => TRUE);
```

Or dynamically in Django:

``` python
from django.db import connection
with connection.cursor() as cursor:
    cursor.execute("SELECT create_hypertable('yourapp_sensorreading', 'timestamp', if_not_exists => TRUE);")
```

------------------------------------------------------------------------

## ⏱️ 5. Time Bucket + Aggregation (Rollups)

### Basic hourly average query:

``` sql
SELECT time_bucket('1 hour', timestamp) AS hour,
       AVG(temperature)
FROM yourapp_sensorreading
GROUP BY hour
ORDER BY hour;
```

Use this in Django with raw SQL and convert to Pandas for dashboards.

------------------------------------------------------------------------

## 📊 6. Continuous Aggregates

``` sql
CREATE MATERIALIZED VIEW hourly_temp
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', timestamp) AS bucket,
       sensor_id,
       AVG(temperature) AS avg_temp
FROM yourapp_sensorreading
GROUP BY bucket, sensor_id;
```

### Automate refresh:

``` sql
SELECT add_continuous_aggregate_policy('hourly_temp',
  start_offset => INTERVAL '7 days',
  end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '30 minutes');
```

------------------------------------------------------------------------

## 🧊 7. Data Compression

``` sql
ALTER TABLE yourapp_sensorreading
SET (timescaledb.compress, timescaledb.compress_segmentby = 'sensor_id');

SELECT compress_chunk(i) FROM show_chunks('yourapp_sensorreading') i;
```

### Policy:

``` sql
SELECT add_compression_policy('yourapp_sensorreading', INTERVAL '30 days');
```

------------------------------------------------------------------------

## 🧠 8. Query via Django ORM or Raw SQL

### Django ORM (limited for rollups):

``` python
from yourapp.models import SensorReading

SensorReading.objects.filter(
    timestamp__gte=timezone.now() - timedelta(days=7)
).aggregate(Avg('temperature'))
```

### Raw SQL with Pandas:

``` python
from django.db import connection
import pandas as pd

query = """
SELECT time_bucket('1 hour', timestamp) AS hour,
       AVG(temperature) as avg_temp
FROM yourapp_sensorreading
WHERE timestamp > now() - interval '7 days'
GROUP BY hour ORDER BY hour;
"""

df = pd.read_sql(query, connection)
```

------------------------------------------------------------------------

## 🧮 9. Visualize with Jupyter

Perfect for exploring time-series:

``` python
import matplotlib.pyplot as plt

df.plot(x='hour', y='avg_temp', title='Hourly Avg Temp')
plt.grid(True)
plt.show()
```

------------------------------------------------------------------------

## 📘 Summary Table

| Feature               | TimescaleDB Usage                                |
|-----------------------|--------------------------------------------------|
| Store time-series     | Regular Django model + `create_hypertable()`     |
| Partitioning          | Automatic via hypertables                        |
| Aggregations          | `time_bucket()`, `GROUP BY`                      |
| Continuous aggregates | Fast rollups with materialized views             |
| Compression           | Save space, improve historical query performance |
| Compatibility         | Full Django & PostgreSQL support                 |

------------------------------------------------------------------------

## 🛠️ Tools & Add-ons

- 📊 [Grafana](https://grafana.com/) for live dashboards
- 📦 [pgAdmin](https://www.pgadmin.org/) for TimescaleDB queries
- 🔁 [pg_cron](https://github.com/citusdata/pg_cron) for DB-side scheduling
- 🧪 [tsbs](https://github.com/timescale/tsbs) for benchmarking

------------------------------------------------------------------------

## 🧪 Starter Ideas

- IoT metrics dashboard (sensor data)
- Health tracker (heart rate over time)
- Financial tick data rollups
- Website traffic analysis (time-bucketed)

------------------------------------------------------------------------

Back to top
