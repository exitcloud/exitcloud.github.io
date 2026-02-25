---
title: "PostgreSQL Tuning for a Single-Server App"
date: 2026-02-02T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "databases"
  - "devops"
---

The default PostgreSQL config is designed to run on a toaster. Seriously — it ships with `shared_buffers = 128MB` and `work_mem = 4MB` because the Postgres team wants it to work on any hardware, including a Raspberry Pi from 2015. If you're running a real app on a real server, you need to change about five settings. That's it. Five settings and you're 80% of the way to a well-tuned Postgres.

I've been running Postgres on single-server setups — VPSes, mini PCs, small dedicated boxes — for years. No replicas, no clustering, no managed database service. Just one box, one Postgres instance, one app. Here's what I've learned.

## Start with PgTune

Before I tell you the settings, let me tell you the tool. Go to [PgTune](https://pgtune.leopard.in.ua/), enter your server specs, and it spits out a config. It's a starting point, not gospel, but it's a much better starting point than the defaults.

For a typical VPS — say 4 cores, 8GB RAM, SSD storage, running a web application — PgTune gives you something reasonable. Then you tweak from there based on actual workload.

## The five settings that matter

### 1. shared_buffers

This is Postgres's dedicated memory cache. The rule of thumb: **25% of total RAM**. On an 8GB server, set it to 2GB.

```
shared_buffers = 2GB
```

Why 25% and not more? Because the OS also caches disk pages, and Postgres relies heavily on that OS cache. Going above 25% gives diminishing returns and can actually hurt because you're stealing memory from the OS cache.

Going below 25% means Postgres is constantly hitting disk for data it could have kept in memory. You'll see this as higher I/O wait in your monitoring.

### 2. effective_cache_size

This isn't a memory allocation — it's a hint to the query planner about how much memory is available for caching (Postgres buffer + OS cache combined). Set it to **~75% of total RAM**.

```
effective_cache_size = 6GB
```

If this is too low, the planner will avoid index scans and prefer sequential scans because it assumes the index won't be in memory. Wrong setting, wrong query plans, slow queries. Setting it higher tells the planner "yeah, that index is probably cached, go ahead and use it."

### 3. work_mem

Memory available for each sort or hash operation within a query. This one is tricky because it's **per-operation, per-connection**. A complex query with 5 sort steps and 10 concurrent connections could use `5 * 10 * work_mem` of RAM.

Start conservative:

```
work_mem = 16MB
```

For an app with moderate concurrency (20-50 connections), 16MB is usually fine. If you're running heavy analytical queries, bump it up. If you're running hundreds of connections, keep it lower.

The footgun here: setting `work_mem = 256MB` on a server with 100 connections. One complex report query and you're swapping. I've seen this take down production boxes.

### 4. maintenance_work_mem

Memory for maintenance operations — VACUUM, CREATE INDEX, ALTER TABLE. These run less frequently but benefit from more memory.

```
maintenance_work_mem = 512MB
```

Set this to ~5-10% of RAM. It only matters during maintenance operations, so it won't compete with normal queries most of the time. A higher value means faster vacuums and faster index creation.

### 5. max_connections

The default is 100, and honestly? For a single-server app with a connection pooler, you might need even fewer Postgres connections than that.

```
max_connections = 50
```

Wait, fewer? Yes. Each Postgres connection is a full OS process with its own memory allocation. 200 connections means 200 processes, each using `work_mem` for sorts, plus per-connection overhead. More connections doesn't mean more throughput — past a certain point, you're just adding context switching overhead.

The real answer is connection pooling. Which brings us to...

## PgBouncer: stop opening direct connections

If your app opens a connection, runs a query, and closes the connection — or worse, holds connections open while waiting for user input — you're wasting Postgres resources. PgBouncer sits between your app and Postgres and pools connections.

```ini
# /etc/pgbouncer/pgbouncer.ini
[databases]
myapp = host=127.0.0.1 port=5432 dbname=myapp

[pgbouncer]
listen_addr = 127.0.0.1
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
max_client_conn = 200
default_pool_size = 20
```

The key setting: `pool_mode = transaction`. This means a Postgres connection is only held for the duration of a transaction, then returned to the pool. Your app can have 200 connections to PgBouncer, but PgBouncer only keeps 20 connections to Postgres. Massive reduction in Postgres overhead.

Point your app at PgBouncer (port 6432) instead of Postgres (port 5432). Done.

One gotcha: `pool_mode = transaction` doesn't work well with prepared statements or session-level features (like `SET` commands or advisory locks). If your ORM uses prepared statements heavily, you might need `pool_mode = session` or configure the ORM to disable prepared statements through PgBouncer.

## Vacuum tuning: the silent killer

Postgres uses MVCC, which means old row versions pile up until VACUUM cleans them. If autovacuum falls behind, your tables bloat, your indexes bloat, and queries slow down. Eventually you hit transaction ID wraparound and Postgres shuts down to protect itself. Not fun.

The defaults are conservative. For a write-heavy app, tune autovacuum to be more aggressive:

```
autovacuum_vacuum_scale_factor = 0.05      # default 0.2
autovacuum_analyze_scale_factor = 0.025    # default 0.1
autovacuum_vacuum_cost_delay = 2ms         # default 2ms (fine)
autovacuum_max_workers = 4                 # default 3
```

What this does: autovacuum triggers earlier (when 5% of rows are dead instead of 20%) and runs with more workers. On a busy database, this keeps table bloat in check.

Monitor it:

```sql
SELECT relname, n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric / greatest(n_live_tup, 1) * 100, 1) as dead_pct,
       last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 10;
```

If `dead_pct` is consistently above 10-20% on your busiest tables, autovacuum isn't keeping up.

## Real before/after

Here's a real example from a SaaS app I run. Server: 4 vCPUs, 8GB RAM, NVMe SSD. ~500K rows in the main table, mixed read/write workload.

**Before (default config):**

```
shared_buffers = 128MB
work_mem = 4MB
effective_cache_size = 4GB
max_connections = 100
No connection pooler
```

A dashboard query joining 3 tables with aggregation: **~2.4 seconds**.

**After tuning:**

```
shared_buffers = 2GB
work_mem = 32MB
effective_cache_size = 6GB
max_connections = 50
PgBouncer in transaction mode
```

Same query: **~380ms**. That's a 6x improvement from five config changes and a connection pooler. No schema changes. No query rewrites. No new indexes. Just giving Postgres enough memory to do its job.

## The full config snippet

Here's my go-to `postgresql.conf` additions for an 8GB single-server setup:

```
# Memory
shared_buffers = 2GB
effective_cache_size = 6GB
work_mem = 32MB
maintenance_work_mem = 512MB

# Connections (use PgBouncer in front)
max_connections = 50

# WAL
wal_buffers = 64MB
checkpoint_completion_target = 0.9
max_wal_size = 2GB

# Planner
random_page_cost = 1.1          # SSD, not spinning disk
effective_io_concurrency = 200  # SSD

# Autovacuum
autovacuum_vacuum_scale_factor = 0.05
autovacuum_analyze_scale_factor = 0.025
autovacuum_max_workers = 4
```

After changing, reload:

```bash
sudo systemctl reload postgresql
```

Most settings take effect on reload. A few (like `shared_buffers` and `max_connections`) require a full restart.

## Cheat Sheet

| Setting / Tool | What it does | Recommended value (8GB RAM) |
|---------------|-------------|---------------------------|
| `shared_buffers` | Postgres dedicated cache | 25% of RAM = `2GB` |
| `effective_cache_size` | Hint to planner about total cache available | 75% of RAM = `6GB` |
| `work_mem` | Memory per sort/hash operation, per connection | `16-32MB` (careful — it multiplies) |
| `maintenance_work_mem` | Memory for VACUUM, CREATE INDEX | `512MB` |
| `max_connections` | Max direct Postgres connections | `50` (use a pooler) |
| `random_page_cost` | Planner cost for random I/O — lower for SSDs | `1.1` for SSD, `4.0` for HDD |
| `effective_io_concurrency` | Concurrent I/O ops the OS can handle | `200` for SSD |
| `autovacuum_vacuum_scale_factor` | Dead row % that triggers vacuum | `0.05` (more aggressive than default `0.2`) |
| [PgTune](https://pgtune.leopard.in.ua/) | Generate a starting config from your server specs | Use as starting point, then tweak |
| [PgBouncer](https://www.pgbouncer.org/) | Connection pooler — `pool_mode = transaction` | 20 pool size behind 200 client conns |
| `pg_stat_user_tables` | Monitor dead tuples and autovacuum timing | Query it weekly or alert on bloat |
| **Rule of thumb** | Tune 5 settings + add PgBouncer. That covers 80% of single-server performance. | — |
