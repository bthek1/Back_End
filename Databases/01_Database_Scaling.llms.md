# Database Scaling

How a database grows past one machine: query tuning, connection pooling, caching, read replicas, partitioning, sharding, and the distributed systems that do it for you.

Author

Benedict Thekkel

Scaling a database means keeping latency and throughput acceptable as data volume, request rate, and concurrency grow. There are two directions:

- **Vertical (scale up):** a bigger machine. Simple, no code changes, but it has a ceiling and stays a single point of failure.
- **Horizontal (scale out):** more machines. No hard ceiling, but every step adds consistency and operational problems.

The techniques form a ladder. Each rung is cheaper and simpler than the one above it, so climb only when the rung below is exhausted.

| Rung | Technique | Scales | Cost in complexity |
|----|----|----|----|
| 1 | Query and index optimisation | Everything | Very low |
| 2 | Connection pooling | Concurrency | Low |
| 3 | Vertical scaling | Everything, up to a ceiling | Low (money only) |
| 4 | Caching | Reads | Medium (invalidation) |
| 5 | Read replicas | Reads | Medium (replication lag) |
| 6 | Partitioning | Large tables | Medium |
| 7 | Functional split | Writes, by domain | Medium to high |
| 8 | Sharding | Writes and storage | Very high |
| 9 | Distributed SQL or NoSQL | Writes and storage | High (new system to learn) |

The most common mistake is jumping to rung 8 when the problem is on rung 1.

------------------------------------------------------------------------

## Measure Before Scaling

Find out which resource is actually running out. Each symptom points at a different rung.

| Symptom | Likely limit | Rung |
|----|----|----|
| High CPU, a few slow queries dominate | Missing indexes, bad query plans | 1 |
| `FATAL: sorry, too many clients already` | Connection count, not data | 2 |
| Low buffer cache hit ratio, high disk reads | Working set no longer fits in RAM | 3, 6 |
| Same rows read over and over | Repeated reads | 4, 5 |
| Primary CPU saturated by reads | Read volume | 5 |
| Lock waits, deadlocks on a few rows | Write contention | Schema or app design |
| Primary saturated by writes, replicas lagging | Write throughput | 7, 8, 9 |
| One table is hundreds of GB, vacuum never finishes | Table size | 6 |

Useful tools in PostgreSQL:

``` sql
-- Which queries consume the most total time (needs the pg_stat_statements extension)
SELECT query, calls, round(total_exec_time) AS total_ms, round(mean_exec_time, 2) AS mean_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Buffer cache hit ratio; below about 0.99 on an OLTP database means RAM pressure
SELECT sum(blks_hit)::float / nullif(sum(blks_hit) + sum(blks_read), 0) AS hit_ratio
FROM pg_stat_database;

-- Current connections by state
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
```

Look at p95 and p99 latency, not the average. An average hides the slow tail that users actually notice.

------------------------------------------------------------------------

## Query and Index Optimisation

Often a 10x to 100x improvement for no infrastructure change at all.

- **Read the plan.** `EXPLAIN (ANALYZE, BUFFERS) SELECT ...` shows whether the database is doing a sequential scan over millions of rows to return ten.
- **Index for the real query patterns.**
  - Composite indexes in the order of the filter: equality columns first, then range columns.
  - Covering indexes (`INCLUDE (...)`) so the query is answered from the index alone.
  - Partial indexes (`WHERE status = 'active'`) when only a small subset is queried.
  - Every index slows writes and uses RAM, so drop unused ones (`pg_stat_user_indexes.idx_scan = 0`).
- **Kill N+1 queries.** In Django that means `select_related` and `prefetch_related`.
- **Paginate by key, not by offset.** `OFFSET 100000` still reads and throws away 100000 rows.

``` sql
-- Slow: cost grows with the page number
SELECT * FROM orders ORDER BY id LIMIT 50 OFFSET 100000;

-- Fast: keyset pagination, constant cost
SELECT * FROM orders WHERE id > :last_seen_id ORDER BY id LIMIT 50;
```

- **Batch writes.** One `INSERT` of 1000 rows (or `COPY`) instead of 1000 round trips and 1000 commits.
- **Select only the columns you need**, especially when wide `JSONB` or `TEXT` columns are involved.

------------------------------------------------------------------------

## Connection Pooling

PostgreSQL forks one OS process per connection, and each costs several MB of RAM plus scheduling overhead. Throughput peaks at a fairly small number of active connections (roughly 2 to 4 per CPU core) and then falls as they fight over CPU and locks. A web tier with 20 instances of 50 threads each wants 1000 connections, which a database handles badly.

A pooler sits between the application and the database and multiplexes many client connections onto a few server connections.

      app instances (1000 client connections)
            |  |  |  |  |
            v  v  v  v  v
        +----------------+
        |   PgBouncer    |   pool_mode = transaction
        +----------------+
            |  |  |
            v  v  v        (40 server connections)
        +----------------+
        |   PostgreSQL   |
        +----------------+

Pool modes in PgBouncer:

| Mode | A server connection is held for | Notes |
|----|----|----|
| `session` | The whole client session | Safe for everything, but little multiplexing |
| `transaction` | One transaction | The usual choice. Session state (`SET`, advisory locks, `LISTEN`) does not survive between transactions |
| `statement` | One statement | Multi-statement transactions are not allowed |

``` ini
; pgbouncer.ini
[databases]
app = host=10.0.0.10 port=5432 dbname=app

[pgbouncer]
listen_port = 6432
pool_mode = transaction
max_client_conn = 2000
default_pool_size = 40
```

Managed alternatives: RDS Proxy, Supabase Supavisor, ProxySQL for MySQL. In Django, behind a transaction pooler, set `CONN_MAX_AGE = 0` (or use the pooler and Django’s own persistent connections carefully) and disable server-side cursors with `DISABLE_SERVER_SIDE_CURSORS = True`, because those cursors span transactions.

------------------------------------------------------------------------

## Vertical Scaling

Move to a bigger machine: more RAM so the working set fits in memory, faster NVMe for I/O, more cores for concurrency.

It is underrated. A single modern server with hundreds of GB of RAM and 64 or more cores runs workloads far beyond what most applications will ever reach, and it costs no engineering time. The limits:

- Cost rises steeply at the top end.
- There is a hard ceiling.
- It is still one machine, so it does nothing for availability (that is what replicas are for).
- Resizing often needs a restart or failover.

Check the configuration before buying hardware. Defaults are conservative: in PostgreSQL, `shared_buffers` (about 25% of RAM), `effective_cache_size` (about 75%), `work_mem`, and `max_wal_size` are the usual first tuning targets.

------------------------------------------------------------------------

## Caching

The fastest query is the one that never reaches the database. A cache (Redis or Memcached) in front of read-heavy data absorbs repeated reads.

| Pattern | How it works | Trade-off |
|----|----|----|
| Cache-aside (lazy loading) | Read the cache; on a miss read the DB and fill the cache | Most common. First read after expiry is slow |
| Read-through | The cache layer itself loads from the DB on a miss | Same as above, logic lives in the cache layer |
| Write-through | Every write goes to the cache and the DB together | Cache always fresh, writes slower |
| Write-behind | Write to the cache now, flush to the DB later | Fast writes, but data is lost if the cache dies |

    read:   app --> cache --hit--> return
                      |
                     miss
                      v
                      DB --> store in cache with TTL --> return

    write:  app --> DB --> delete cache key

The hard parts:

- **Invalidation.** Use a TTL as a safety net and delete the key on write. Deleting is safer than updating, because two concurrent updates can leave the cache holding the older value.
- **Cache stampede.** A hot key expires and a thousand requests miss at once and all hit the database. Fixes: a lock so only one request rebuilds the value, request coalescing, or refreshing slightly before expiry with some randomness.
- **Stale reads** are a feature you are choosing. Decide per piece of data how stale is acceptable.

Other cache layers worth knowing: materialised views inside the database, HTTP caching and CDNs for API responses, and per-request memoisation in the application.

------------------------------------------------------------------------

``` python
# Cache-aside with Django's cache framework (backed by Redis)
from django.core.cache import cache

def get_product(product_id: int):
    key = f"product:{product_id}"
    product = cache.get(key)
    if product is None:
        product = Product.objects.get(pk=product_id)
        cache.set(key, product, timeout=300)  # TTL as a safety net
    return product

def update_product(product_id: int, **fields):
    Product.objects.filter(pk=product_id).update(**fields)
    cache.delete(f"product:{product_id}")  # invalidate, do not overwrite
```

## Read Replicas

Most applications read far more than they write, often 80 to 95 percent reads. A read replica is a copy of the primary that receives every change and serves read-only queries, so read load spreads across several machines while all writes still go to one primary.

                        writes + reads that must be fresh
           app  ------------------------------------------>  PRIMARY
            |                                                   |
            |                                         WAL stream (changes)
            |                                       /           |           \
            |   reads                              v            v            v
            +---------------------------------> REPLICA 1    REPLICA 2    REPLICA 3

### How replication works

The primary records every change in a log before applying it: the **write-ahead log (WAL)** in PostgreSQL, the **binlog** in MySQL. Replicas receive that log and replay it.

| Kind | What is shipped | Examples | Use |
|----|----|----|----|
| Physical (streaming) | Raw WAL bytes, byte-identical copy | PostgreSQL streaming replication | Read replicas, standby for failover |
| Logical | Row changes (insert, update, delete) | PostgreSQL logical replication, MySQL row-based binlog | Replicating some tables, cross-version upgrades, feeding other systems |

### Synchronous versus asynchronous

| Mode | Primary commits when | Pros | Cons |
|----|----|----|----|
| Asynchronous | Its own WAL is on disk | Fast writes, replica outage does not affect primary | Replicas lag; on failover the last few transactions can be lost |
| Synchronous | At least one replica confirms | No data loss on failover | Every write waits a network round trip; a stalled replica stalls writes |
| Semi-sync / quorum | N of M replicas confirm | Balance of the two | Still adds latency |

PostgreSQL controls this with `synchronous_commit` and `synchronous_standby_names`. The common production shape is one synchronous standby in the same region for durability plus asynchronous replicas for read scaling.

### Setting up a PostgreSQL streaming replica

``` bash
# On the primary (postgresql.conf): wal_level = replica, max_wal_senders = 10
# and allow the replication user in pg_hba.conf:
#   host replication replicator 10.0.0.0/24 scram-sha-256

# On the new replica: clone the primary and write the standby config in one step
pg_basebackup -h 10.0.0.10 -U replicator -D /var/lib/postgresql/17/main -R -X stream -P
# -R creates standby.signal and primary_conninfo, then start PostgreSQL as usual
```

Managed databases (RDS, Cloud SQL, Azure) reduce this to one button, and Aurora replicas share the same storage, so they lag by milliseconds rather than seconds.

### What replicas do not do

- They do **not** scale writes. Every replica replays every write.
- They do **not** reduce storage per node. Every replica holds the full dataset.
- They are **not** backups. A `DROP TABLE` replicates instantly.

------------------------------------------------------------------------

### Routing reads in Django

Django supports several databases and a **database router** that decides which alias a query uses.

``` python
# settings.py
DATABASES = {
    "default": {"ENGINE": "django.db.backends.postgresql", "HOST": "primary.db.internal", "NAME": "app"},
    "replica1": {"ENGINE": "django.db.backends.postgresql", "HOST": "replica1.db.internal", "NAME": "app"},
    "replica2": {"ENGINE": "django.db.backends.postgresql", "HOST": "replica2.db.internal", "NAME": "app"},
}
DATABASE_ROUTERS = ["core.routers.PrimaryReplicaRouter"]

# For tests, mark replicas as mirrors so they share the test database:
# DATABASES["replica1"]["TEST"] = {"MIRROR": "default"}
```

``` python
# core/routers.py
import random

class PrimaryReplicaRouter:
    replicas = ["replica1", "replica2"]

    def db_for_read(self, model, **hints):
        return random.choice(self.replicas)

    def db_for_write(self, model, **hints):
        return "default"

    def allow_relation(self, obj1, obj2, **hints):
        return True  # all aliases hold the same data

    def allow_migrate(self, db, app_label, model_name=None, **hints):
        return db == "default"  # replicas get schema changes through replication
```

Force a specific query to the primary with `.using("default")`, for example `Order.objects.using("default").get(pk=pk)`. Reads inside `transaction.atomic()` are worth sending to the primary too, because a transaction that mixes primary writes with replica reads sees an inconsistent picture.

------------------------------------------------------------------------

## Replication Lag

With asynchronous replication a replica is always slightly behind, usually milliseconds, but seconds or minutes under heavy write load, long-running queries on the replica, or network trouble. The classic bug:

    1. User updates their profile      -> write goes to PRIMARY
    2. Page redirects and reloads      -> read goes to REPLICA (still 200 ms behind)
    3. User sees the OLD profile       -> "my save did not work", saves again

Anomalies caused by lag, and their fixes:

| Anomaly | Example | Fix |
|----|----|----|
| Read-your-writes violated | Edit, reload, see the old value | Route that user’s reads to the primary for a few seconds after a write |
| Monotonic reads violated | Refresh twice, a comment appears then disappears (two replicas with different lag) | Pin a user or session to one replica |
| Causal order violated | A reply is visible before the question it answers | Read related data from the same node, or from the primary |

Strategies, from simplest to most precise:

1.  **Primary for critical paths.** Anything the user just changed, or anything involving money, reads the primary.
2.  **Sticky primary after write.** After a write, set a short flag (cookie, session, or cache key) and send that user’s reads to the primary until it expires.
3.  **Wait for position.** Record the primary’s WAL position (LSN) after the write, and only read from a replica that has replayed past it.
4.  **Lag-aware routing.** Take a replica out of the read pool when its lag exceeds a threshold.

``` sql
-- On a replica: how far behind is it, in time?
SELECT now() - pg_last_xact_replay_timestamp() AS replication_delay;

-- On the primary: per-replica lag in bytes and time
SELECT client_addr, state, sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS bytes_behind,
       replay_lag
FROM pg_stat_replication;
```

Note that `pg_last_xact_replay_timestamp()` stops advancing when the primary is idle, so a quiet system can look lagged. Compare LSNs when precision matters.

------------------------------------------------------------------------

``` python
# A sticky-after-write router: reads go to the primary for a few seconds after this
# request (or user) wrote. A middleware sets the flag in a contextvar from a cookie.
import contextvars, random

recently_wrote = contextvars.ContextVar("recently_wrote", default=False)

class StickyPrimaryRouter:
    replicas = ["replica1", "replica2"]

    def db_for_read(self, model, **hints):
        if recently_wrote.get():
            return "default"
        return random.choice(self.replicas)

    def db_for_write(self, model, **hints):
        recently_wrote.set(True)  # middleware also sets a short-lived cookie on the response
        return "default"
```

------------------------------------------------------------------------

## High Availability and Failover

A replica is also the standby that takes over when the primary dies. Failover means: detect the failure, pick the most up-to-date replica, promote it to primary, repoint the other replicas, and move clients to it.

- **Automate it.** Patroni (with etcd or Consul) is the standard for self-hosted PostgreSQL; managed services (RDS Multi-AZ, Cloud SQL HA) do it for you, typically in 30 to 120 seconds.
- **Split brain** is the danger: two nodes both believe they are primary and both accept writes, and the histories diverge. Prevent it with a consensus store that grants leadership to one node at a time, and by fencing the old primary (shutting it off) before promoting.
- **Clients need a stable address**: a virtual IP, a DNS name the failover tool updates, or a proxy (HAProxy, PgBouncer) that follows the leader.
- **RPO and RTO.** Recovery point objective is how much data you can lose (zero needs synchronous replication). Recovery time objective is how long you can be down.
- **Practise it.** A failover that has never been triggered on purpose will surprise you when it happens for real.

------------------------------------------------------------------------

## Partitioning

Partitioning splits one large table into smaller physical tables **on the same server**, while queries still see one logical table.

| Type | Split by | Typical use |
|----|----|----|
| Range | Value ranges, usually time | Events, logs, metrics, orders by month |
| List | Explicit values | Region, tenant tier |
| Hash | Hash of a key | Even spread when there is no natural range |

Benefits:

- **Partition pruning:** a query with `WHERE created_at >= '2026-10-01'` only touches the matching partitions.
- **Smaller indexes** that fit in memory.
- **Cheap data retention:** dropping a month is `DROP TABLE events_2025_01`, instant, instead of a huge `DELETE` that bloats the table and hammers vacuum.
- Maintenance (vacuum, reindex) runs per partition.

``` sql
CREATE TABLE events (
    id         bigint GENERATED ALWAYS AS IDENTITY,
    created_at timestamptz NOT NULL,
    user_id    bigint NOT NULL,
    payload    jsonb,
    PRIMARY KEY (id, created_at)          -- the partition key must be in the primary key
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_10 PARTITION OF events
    FOR VALUES FROM ('2026-10-01') TO ('2026-11-01');
CREATE TABLE events_2026_11 PARTITION OF events
    FOR VALUES FROM ('2026-11-01') TO ('2026-12-01');

-- Retention: detach and drop old months
ALTER TABLE events DETACH PARTITION events_2025_10;
DROP TABLE events_2025_10;
```

Use `pg_partman` to create future partitions automatically, or TimescaleDB, which does time partitioning (hypertables) and compression for you. Queries that do not filter on the partition key scan every partition, so choose the key from the dominant access pattern.

Partitioning splits a table; sharding (below) splits the **data across servers**. Partitioning is sometimes called horizontal partitioning, and sharding is horizontal partitioning across machines.

------------------------------------------------------------------------

## Sharding

Sharding splits the data across several independent database servers. Each shard holds a subset of the rows and handles the reads **and writes** for that subset. It is the only way to scale write throughput and storage past a single primary, and the most expensive step in complexity.

                           app + shard router
                     (shard = f(tenant_id))
                 /            |              \
                v             v               v
           SHARD A         SHARD B          SHARD C
         tenants 1-..    tenants ..-..    tenants ..-..
         primary+replicas primary+replicas primary+replicas

Each shard is usually itself a primary with replicas, so sharding and replication combine.

### Choosing the shard key

The most important decision, and very hard to change later. A good key:

- Has **high cardinality** and spreads load evenly.
- **Matches the access pattern**, so most queries hit exactly one shard. `tenant_id` for multi-tenant SaaS and `user_id` for consumer apps are the classic choices.
- Keeps data that is used together on the same shard (a tenant’s users, orders, and invoices).

Bad keys: a timestamp (all current writes land on the newest shard), a low-cardinality field such as country, or anything where one value is enormous (a single giant customer becomes a hot shard).

### Strategies

| Strategy | Routing | Pros | Cons |
|----|----|----|----|
| Range | Key ranges map to shards | Range scans stay on one shard | Hot spots; uneven growth |
| Hash modulo | `hash(key) % N` | Even spread | Changing N moves almost every key |
| Consistent hashing | Hash ring with virtual nodes | Adding a node moves only about 1/N of the keys | No efficient range queries |
| Directory | A lookup table maps key to shard | Total flexibility, can move one tenant at a time | Lookup service is a dependency to keep fast and available |

### What sharding breaks

- **Cross-shard queries** become scatter-gather: send to every shard, merge the results. Latency is set by the slowest shard.
- **Cross-shard joins** move into the application, or are avoided by co-locating related data and duplicating small reference tables on every shard.
- **Cross-shard transactions** need two-phase commit (slow, blocks on failures) or sagas (a sequence of local transactions with compensating actions, eventually consistent).
- **Auto-increment IDs** collide. Use globally unique, time-ordered IDs: UUIDv7, ULID, or Snowflake-style 64-bit IDs (timestamp + machine + sequence), which stay friendly to B-tree indexes unlike random UUIDv4.
- **Unique constraints** only hold within a shard unless the unique column is the shard key.
- **Rebalancing** live data needs double writes, backfill, verification, and a cutover.
- **Operations multiply**: N sets of backups, migrations, monitoring, and failovers.

Tools that take on much of this: **Vitess** for MySQL (YouTube, Slack, GitHub), **Citus** for PostgreSQL (distributed tables plus reference tables), and the managed versions of each.

------------------------------------------------------------------------

### Why consistent hashing

With `hash(key) % N`, going from 4 to 5 shards changes the shard for roughly 80 percent of keys, so almost all data has to move. Consistent hashing places shards and keys on the same ring and gives each key to the next shard clockwise, so adding a shard only takes keys from its neighbours. The cell below measures both.

``` python
import bisect, hashlib

def h(s: str) -> int:
    return int(hashlib.md5(s.encode()).hexdigest(), 16)

class Ring:
    def __init__(self, nodes, vnodes=100):
        self.ring = sorted((h(f"{n}#{i}"), n) for n in nodes for i in range(vnodes))
        self.keys = [k for k, _ in self.ring]

    def node(self, key: str) -> str:
        i = bisect.bisect(self.keys, h(key)) % len(self.ring)
        return self.ring[i][1]

keys = [f"user:{i}" for i in range(100_000)]

mod_moved = sum(h(k) % 4 != h(k) % 5 for k in keys) / len(keys)

before = Ring(["a", "b", "c", "d"])
after = Ring(["a", "b", "c", "d", "e"])
ring_moved = sum(before.node(k) != after.node(k) for k in keys) / len(keys)

print(f"hash % N, 4 -> 5 shards: {mod_moved:.0%} of keys move")
print(f"consistent hashing     : {ring_moved:.0%} of keys move (ideal is 1/5 = 20%)")
```

------------------------------------------------------------------------

## Consistency and CAP

Once data lives on more than one machine, you trade consistency against availability and latency.

- **CAP:** during a network partition, a system either refuses some requests (consistent, CP) or answers with possibly stale data (available, AP). Partitions are not optional, so the real choice is what to do when one happens.
- **PACELC** is the more useful everyday version: if there is a **P**artition choose **A** or **C**, **E**lse choose **L**atency or **C**onsistency. Even with a healthy network, strong consistency costs a round trip to other nodes on every write.

Consistency models, strongest first:

| Model | Guarantee |
|----|----|
| Linearizable | Every read sees the latest committed write, as if there were one copy |
| Sequential | All nodes see operations in the same order |
| Causal | Operations that depend on each other are seen in order |
| Read-your-writes | You always see your own writes |
| Eventual | If writes stop, all replicas eventually agree |

How distributed systems get there:

- **Leader-based replication** (PostgreSQL, MySQL): one node orders all writes.
- **Consensus** (Raft, Paxos): a majority of replicas must agree before a write commits. This is how CockroachDB, YugabyteDB, TiDB, etcd, and Spanner stay strongly consistent and survive a minority of nodes failing.
- **Leaderless quorums** (Cassandra, DynamoDB-style): with N replicas, write to W and read from R. When `R + W > N`, every read overlaps a node holding the latest write. Tunable per query.

------------------------------------------------------------------------

## Distributed SQL and NoSQL

Instead of sharding by hand, use a database that shards and replicates internally.

| Family | Examples | How it scales | Good for | Watch out for |
|----|----|----|----|----|
| Distributed SQL (NewSQL) | CockroachDB, YugabyteDB, TiDB, Google Spanner | Automatic range sharding, Raft per range, distributed transactions | Strong consistency plus horizontal writes, multi-region | Higher per-query latency than a single Postgres; cross-range transactions cost round trips |
| Cloud-native relational | Aurora, AlloyDB, Neon | Compute separated from shared distributed storage | Fast replicas, fast failover, storage that grows on its own | Still one writer |
| Key-value / wide-column | DynamoDB, Cassandra, ScyllaDB | Hash partitioning, leaderless replication | Massive write volume, predictable access patterns | No joins; schema designed around queries |
| Document | MongoDB | Built-in sharding on a shard key | Document-shaped data | Same shard key rules as manual sharding |
| Columnar / OLAP | ClickHouse, BigQuery, Snowflake, DuckDB | Columnar storage, massively parallel scans | Analytics and reporting | Not for transactional row updates |
| Search | Elasticsearch, OpenSearch | Sharded inverted indexes | Full-text search, faceting | Not a system of record |

Rule of thumb for NoSQL stores like DynamoDB and Cassandra: **you design the tables around the queries, not around the entities.** If the access patterns are not known in advance, a relational database will serve you better.

Moving analytics off the transactional database is one of the highest-value scaling moves. Heavy reporting queries on the OLTP primary compete with user traffic; ship the data to a columnar store instead.

------------------------------------------------------------------------

## Architectural Patterns

- **Functional partitioning.** Split by domain: users, orders, and analytics in separate databases, usually along service boundaries. Each scales on its own, and joins across them move to the application.
- **CQRS (command query responsibility segregation).** Separate the write model from the read models, and build read-optimised projections (denormalised tables, search indexes, caches) from the writes.
- **Change data capture (CDC).** Tools such as Debezium read the WAL or binlog and publish every row change to Kafka. One stream then feeds the search index, cache invalidation, the data warehouse, or a new database during a migration, without the application writing to all of them.
- **Transactional outbox.** Writing to the database and then publishing to a queue is a dual write: if the second step fails, they disagree. Instead, insert the event into an `outbox` table in the **same transaction** as the change, and have a relay (or CDC) publish from that table.
- **Queues for write smoothing.** Put bursty, non-urgent writes on a queue (Celery, SQS, Kafka) so the database sees a steady rate instead of spikes.
- **Hot and cold tiering.** Keep recent data in the primary and move old data to cheap storage (Parquet on S3, queried with DuckDB or Athena). A smaller hot set means indexes and the working set stay in memory.
- **Polyglot persistence.** Each workload in the store suited to it: PostgreSQL for transactions, Redis for sessions and counters, ClickHouse for analytics, OpenSearch for search, all kept in sync through CDC.

------------------------------------------------------------------------

## Multi-Region

Serving users on several continents adds the speed of light to every design decision: a round trip between continents is roughly 70 to 250 ms.

| Topology | Writes | Reads | Trade-off |
|----|----|----|----|
| Single primary, remote replicas | Go to one region | Local | Simple; remote users pay latency on writes and see lag |
| Multi-primary (active-active) | Local in every region | Local | Concurrent writes to the same row conflict; resolution by last-write-wins (loses data) or CRDTs (complex) |
| Geo-partitioned | Local for the row’s home region | Local | Each row lives in its user’s region (CockroachDB, Spanner). Also helps with data residency law |
| Global consensus | Majority across regions | Local, possibly stale, or global for fresh | Strongly consistent, but every write pays a cross-region round trip |

Most applications do well with a single primary region, a synchronous standby in another zone of the same region, and asynchronous replicas or a CDN in other regions.

------------------------------------------------------------------------

## Operations at Scale

- **Schema migrations.** `ALTER TABLE` on a large table can take a lock that blocks traffic. Use online tools (gh-ost or pt-online-schema-change for MySQL, `pg_repack` and `CREATE INDEX CONCURRENTLY` for PostgreSQL) and the **expand and contract** pattern: add the new column, write to both, backfill in batches, switch reads, then drop the old column in a later release. Set `lock_timeout` so a migration gives up rather than queueing behind a long transaction.
- **Backups.** Replicas are not backups. Take base backups plus continuous WAL archiving (pgBackRest, WAL-G) for point-in-time recovery, and test the restore regularly.
- **Vacuum and bloat (PostgreSQL).** Updates and deletes leave dead rows that autovacuum must clean. High-churn tables need per-table autovacuum tuning, and transaction ID wraparound is a real outage risk that monitoring should watch.
- **Long transactions** on a replica can either delay replay (raising lag) or be cancelled (`hot_standby_feedback` and `max_standby_streaming_delay` choose which). On the primary they block vacuum.
- **Monitoring.** Track p95 and p99 query latency, connections, replication lag, cache hit ratio, disk growth, lock waits, and dead tuples.
- **Capacity planning.** Know the growth rate and plan the next rung at about 50 percent of the current ceiling, not at 95 percent.
- **Load testing** with production-sized data and real query mixes; a table of 1000 rows tells you nothing about one of 100 million.

------------------------------------------------------------------------

## Decision Guide

Work down the list and stop at the first step that solves the problem.

1.  **Slow queries?** `EXPLAIN ANALYZE`, add or fix indexes, remove N+1 queries, use keyset pagination.
2.  **Too many connections?** Put PgBouncer (or RDS Proxy) in front.
3.  **Generally overloaded?** Tune the configuration, then scale up. Hardware is cheaper than engineering time.
4.  **Same data read repeatedly?** Cache it with a TTL and invalidate on write.
5.  **Read-heavy and still overloaded?** Add read replicas, route reads to them, and handle replication lag for read-your-writes.
6.  **A few very large tables, often time-based?** Partition them and drop old partitions for retention.
7.  **Analytics slowing down user traffic?** Move them to a columnar store fed by CDC.
8.  **Writes or storage beyond one machine?** Split by domain first; then shard on a carefully chosen key, or adopt distributed SQL, DynamoDB, or Cassandra.

A single well-tuned PostgreSQL primary with a pooler, a cache, and a couple of replicas serves the large majority of applications. Distributing too early swaps a performance problem for a correctness and operations problem that never goes away.

------------------------------------------------------------------------

## Further Reading

- Martin Kleppmann, *Designing Data-Intensive Applications*: the standard reference for replication, partitioning, and consistency.
- Alex Petrov, *Database Internals*: storage engines and distributed systems from the inside.
- [Use The Index, Luke](https://use-the-index-luke.com/): indexing and query performance.
- [PostgreSQL documentation: High Availability, Load Balancing, and Replication](https://www.postgresql.org/docs/current/high-availability.html)
- [Django documentation: Multiple databases](https://docs.djangoproject.com/en/stable/topics/db/multi-db/): database routers and `using()`.
- [Vitess](https://vitess.io/docs/) and [Citus](https://docs.citusdata.com/) documentation: sharding MySQL and PostgreSQL.
- [Jepsen](https://jepsen.io/analyses): how distributed databases actually behave under failure.

------------------------------------------------------------------------

Back to top
