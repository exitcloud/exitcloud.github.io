---
title: "SQLite in Production: Yes, Really"
date: 2025-11-08T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "databases"
---

Every time I tell someone I run [SQLite](https://sqlite.org/) in production, I get the same look. That slightly concerned head-tilt, like I just told them I drive without a seatbelt. "But what about concurrency? What about backups? What about... scale?"

Yeah. What about them. Let me tell you what actually happens when you run SQLite in production. Spoiler: it's boring. In the best way.

## The App

One of my apps is a URL analytics platform. Users create short links, I track clicks, they see dashboards. At the time of writing, it handles about 2,000 requests per minute at peak. The database is 4.3 GB. The read-to-write ratio is roughly 15:1.

It runs on SQLite. On a $9 [Hetzner](https://www.hetzner.com/) box. The P99 response time is 12ms.

This is not a toy. This is a real app that people pay for. And SQLite is not just "good enough" — it's actually *faster* than the [PostgreSQL](https://www.postgresql.org/) setup I was running before, because there's no network hop between the app and the database. The database is just... a file. On the same disk. Right there.

## Why SQLite Works (When It Works)

SQLite is an embedded database. There's no server process. No TCP connections. No connection pooling headaches. Your app opens a file and reads/writes to it directly. This means:

- **Zero latency for queries.** No network round-trip. A simple SELECT takes microseconds.
- **Zero operational overhead.** No `pg_hba.conf`. No `max_connections`. No vacuum scheduling (well, almost). No separate process to monitor and restart.
- **Atomic backups.** The database is one file. Back it up and you've got everything.
- **Dead simple deploys.** No database migrations that need to run against a remote server. Just `sqlite3 app.db < migration.sql` on the same box.

The trade-off? SQLite is single-writer. Only one process can write to the database at a time. Reads can happen concurrently (with WAL mode), but writes are serialized. For most web apps, this is totally fine. For some, it's a deal-breaker. Know the difference.

## The Essential Config: WAL Mode

If you're running SQLite in production and you haven't enabled WAL (Write-Ahead Logging) mode, stop reading and go do that right now.

```sql
PRAGMA journal_mode=WAL;
```

WAL mode changes everything. In the default "delete" journal mode, readers block writers and writers block readers. It's terrible for web apps. With WAL:

- **Readers never block writers.** Multiple connections can read while one connection writes.
- **Writers never block readers.** An active write doesn't pause your SELECT queries.
- **Writes are faster.** WAL is sequential I/O (appending to a log) rather than random I/O.

There are more pragmas you want set. Here's my full initialization:

```sql
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 5000;
PRAGMA synchronous = NORMAL;
PRAGMA cache_size = -64000;  -- 64MB
PRAGMA foreign_keys = ON;
PRAGMA temp_store = MEMORY;
PRAGMA mmap_size = 268435456;  -- 256MB
```

Let me break these down:

- **`busy_timeout = 5000`** — If a write is in progress, wait up to 5 seconds instead of immediately returning SQLITE_BUSY. Critical for web apps where multiple requests might write concurrently.
- **`synchronous = NORMAL`** — In WAL mode, NORMAL is safe and faster than FULL. You won't lose committed transactions (you might lose the last few WAL frames in a power failure, but the database won't corrupt).
- **`cache_size = -64000`** — Negative number means kilobytes. 64MB of page cache in RAM. Adjust based on your available memory.
- **`mmap_size`** — Memory-maps the database file up to 256MB. Reads become memory accesses instead of `read()` syscalls. Huge performance win for read-heavy workloads.

In [Node.js](https://nodejs.org/) with `better-sqlite3`:

```javascript
const db = require('better-sqlite3')('app.db');
db.pragma('journal_mode = WAL');
db.pragma('busy_timeout = 5000');
db.pragma('synchronous = NORMAL');
db.pragma('cache_size = -64000');
db.pragma('foreign_keys = ON');
db.pragma('temp_store = MEMORY');
db.pragma('mmap_size = 268435456');
```

## Benchmarks (Real Numbers)

I benchmarked my actual app (not a synthetic test) with `wrk`:

```
$ wrk -t4 -c50 -d30s https://myapp.example.com/api/stats/abc123

Running 30s test
  4 threads and 50 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     4.28ms    2.14ms   42.91ms   88.73%
    Req/Sec     2.96k   211.84     3.42k    72.83%
  354,291 requests in 30.01s
```

That's ~11,800 requests per second on a read endpoint. From a $9 server. With SQLite.

Write benchmarks are more modest but still plenty fast:

```
INSERT (single row):           0.03ms avg
INSERT (100 rows, transaction): 0.8ms avg
INSERT (1000 rows, transaction): 6.2ms avg
UPDATE (indexed column):       0.04ms avg
```

The key insight: batch your writes into transactions. A thousand individual INSERTs take ~30ms each (because each one is its own transaction with a disk sync). Wrap them in a single transaction and they take 6ms total. Always batch.

## Litestream: Continuous Backups

This is the piece that makes SQLite genuinely production-ready for me. [Litestream](https://litestream.io/) continuously replicates your SQLite database to S3-compatible storage by streaming WAL changes.

Install it:

```bash
wget https://github.com/benbjohnson/litestream/releases/download/v0.3.13/litestream-v0.3.13-linux-arm64.deb
dpkg -i litestream-v0.3.13-linux-arm64.deb
```

Configure it:

```yaml
# /etc/litestream.yml
dbs:
  - path: /home/deploy/app/data/app.db
    replicas:
      - type: s3
        bucket: my-backups
        path: myapp
        endpoint: https://s3.us-west-000.backblazeb2.com
        access-key-id: ${LITESTREAM_ACCESS_KEY_ID}
        secret-access-key: ${LITESTREAM_SECRET_ACCESS_KEY}
        retention: 72h
        sync-interval: 1s
```

Start it:

```bash
systemctl enable litestream
systemctl start litestream
```

Now every WAL change is streamed to Backblaze B2 within about a second. If your server catches fire, you can restore:

```bash
litestream restore -o restored.db /home/deploy/app/data/app.db
```

You get back a database that's at most 1 second behind. The cost? Litestream uses almost no CPU, and B2 storage is $0.005/GB/month. For my 4.3 GB database, that's about $0.02/month for continuous replication. Twenty cents a year.

## When SQLite Actually Works

Be honest with yourself about whether SQLite fits your use case:

**Good fit:**
- Single-server apps (the vast majority of indie projects)
- Read-heavy workloads (blogs, analytics, dashboards, content sites)
- Apps where a single writer isn't a bottleneck (write transactions complete in milliseconds)
- Embedded applications, CLI tools, desktop apps
- Any app where operational simplicity matters more than write throughput

**Bad fit:**
- Multiple application servers writing to the same database (SQLite is local to one machine)
- Very write-heavy workloads (thousands of concurrent writers)
- Apps that need replication for high availability (Litestream helps, but it's not the same as Postgres streaming replication with automatic failover)
- When you need features SQLite doesn't have (full-text search is available, but JSONB operations, CTEs, and window functions differ from Postgres)

## 37signals Showed the Way

When the team behind Basecamp and HEY started talking publicly about running SQLite in production for their [Rails](https://rubyonrails.org/) apps, it was a turning point. These aren't hobbyists. They run apps serving millions of users. And they said, yeah, SQLite works.

Rails 8 shipped with built-in SQLite support for production, including a solid adapter and `solid_cache`/`solid_queue` gems that use SQLite as a backend for caching and background jobs. No [Redis](https://redis.io/) required. The entire [37signals](https://dev.37signals.com/) stack runs on SQLite now for apps like Campfire.

If it's good enough for DHH's apps, it's probably good enough for your link shortener.

## When to Switch to Postgres

There's no shame in outgrowing SQLite. Here are the actual signals that it's time:

1. **You need multiple app servers.** Maybe you're getting enough traffic that one box can't handle it, or you need redundancy. SQLite can't be shared across machines (NFS-mounted SQLite is a footgun — don't do it).

2. **Write contention is real.** If your `busy_timeout` is regularly firing and users are seeing slow responses on writes, you've hit the single-writer limit.

3. **You need advanced Postgres features.** Full `JSONB` operators, `LISTEN/NOTIFY` for real-time features, `pg_trgm` for fuzzy search, `PostGIS` for geospatial — Postgres has an incredible extension ecosystem that SQLite can't match.

4. **Your team expects Postgres.** If you hire developers and they're all experienced with Postgres, fighting against that has a cost.

The migration isn't bad. `pgloader` can move a SQLite database to Postgres in minutes. Most ORMs abstract the SQL differences. Plan a weekend, test thoroughly, ship it.

But for most indie apps? SQLite is plenty. I've been running it in production for over a year now. Zero data loss. Zero corruption. Zero regrets. Just a file on a disk, doing its job quietly.

That's the best kind of infrastructure: the kind you forget is there.

## Cheat Sheet

| Tool / Technique | What It Does | Link / Command |
|-----------------|-------------|----------------|
| **SQLite** | Embedded database, zero-config, single-file | [sqlite.org](https://sqlite.org/) |
| **WAL mode** | Enable concurrent reads + faster writes | `PRAGMA journal_mode=WAL;` |
| **busy_timeout** | Wait instead of erroring on write contention | `PRAGMA busy_timeout=5000;` |
| **synchronous=NORMAL** | Safe + fast in WAL mode | `PRAGMA synchronous=NORMAL;` |
| **cache_size** | In-memory page cache (negative = KB) | `PRAGMA cache_size=-64000;` (64MB) |
| **mmap_size** | Memory-map the DB file for faster reads | `PRAGMA mmap_size=268435456;` (256MB) |
| **better-sqlite3** | Best SQLite driver for Node.js (synchronous, fast) | [npm: better-sqlite3](https://www.npmjs.com/package/better-sqlite3) |
| **Litestream** | Continuous WAL replication to S3/B2 | [litestream.io](https://litestream.io/) |
| **Batch writes** | Wrap multiple INSERTs in one transaction for 100x speedup | `BEGIN; ... COMMIT;` |
| **Backblaze B2** | S3-compatible storage, $0.005/GB for backups | [backblaze.com/b2](https://www.backblaze.com/b2/cloud-storage.html) |
| **litestream restore** | Recover database from S3 replica | `litestream restore -o restored.db /path/to/app.db` |
| **pgloader** | Migrate SQLite to Postgres when you outgrow it | [pgloader.io](https://pgloader.io/) |
| **Rails 8 + SQLite** | First-class SQLite production support in Rails | [rubyonrails.org](https://rubyonrails.org/) |
| **solid_cache / solid_queue** | SQLite-backed cache and job queue for Rails | No Redis needed |
| **Don't NFS-mount SQLite** | Seriously, don't. Corruption awaits. | Just don't. |
