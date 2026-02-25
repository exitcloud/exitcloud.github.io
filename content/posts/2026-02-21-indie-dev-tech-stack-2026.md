---
title: "The Indie Dev Tech Stack: What I Run in 2026"
date: 2026-02-21T09:00:00Z
draft: false
tags:
  - "opinion"
  - "indie-dev"
  - "self-hosted"
---

# The Indie Dev Tech Stack: What I Run in 2026

Every January I do a full audit of my stack. What's running, what's it costing me, what's actually earning its keep. This year I realized my entire production infrastructure — serving real paying customers across three apps — costs me less than a single Datadog license would.

Here's the full rundown. No affiliate links. No "I got this for free in exchange for a review." Just what I actually run, what it costs, and why I picked it.

## The Box: Hetzner VPS — $20/mo

I run everything on a single [Hetzner](https://www.hetzner.com/) CPX31: 4 vCPUs, 8GB RAM, 160GB NVMe SSD. Located in Falkenstein, Germany. Ping time from the US East Coast is around 90ms, which is totally fine for my user base.

Why Hetzner? Because they're not trying to nickel-and-dime you on bandwidth. You get 20TB of outbound traffic included. Compare that to AWS where egress costs will make you cry. DigitalOcean and Vultr are fine alternatives, but Hetzner's price-to-performance ratio is hard to beat.

I also have a second smaller box (CX22, $5/mo) that runs monitoring and backups. More on that later.

Total compute cost: **$25/mo**.

## Container Runtime: Docker Compose

No Kubernetes. No Nomad. No Swarm. Just Docker Compose.

I have a `docker-compose.yml` that defines my entire stack. Every service, every volume, every network. One file. I can read it top to bottom in two minutes and understand exactly what's running on my box.

```yaml
services:
  app:
    image: registry.example.com/myapp:latest
    restart: unless-stopped
    depends_on: [db, redis]
    env_file: .env
    labels:
      caddy: myapp.example.com
      caddy.reverse_proxy: "{{upstreams 3000}}"

  worker:
    image: registry.example.com/myapp:latest
    command: bundle exec sidekiq
    restart: unless-stopped
    depends_on: [db, redis]
    env_file: .env

  db:
    image: postgres:16
    restart: unless-stopped
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redisdata:/data
```

People will tell you Docker Compose isn't "production-ready." Those people have never run a profitable business on it. `restart: unless-stopped` handles crashes. Health checks handle readiness. It's been battle-tested by millions of deployments at this point.

## Reverse Proxy & TLS: Caddy

[Caddy](https://caddyserver.com/) replaced Nginx for me two years ago and I haven't looked back. Automatic HTTPS with Let's Encrypt. Zero config for basic reverse proxying. The Caddyfile is so simple it almost feels like cheating:

```
myapp.example.com {
    reverse_proxy app:3000
}

api.example.com {
    reverse_proxy api:4000
}

plausible.example.com {
    reverse_proxy plausible:8000
}
```

That's the entire config. Caddy handles cert issuance, renewal, OCSP stapling, HTTP/2, HTTP/3. All automatic. I literally never think about TLS anymore.

If you want something with more routing power, Traefik is the other good option. It integrates with Docker labels so you don't even need a config file. But for my needs, Caddy is simpler and I prefer explicit configs I can read.

## Database: PostgreSQL 16

Postgres. Running in Docker. With a volume mount for data persistence.

"But isn't running your database in Docker risky?" Not if you're doing backups properly. I've been running Postgres in Docker for four years with zero data loss. The container is just a process — the data lives on a persistent volume on the host's NVMe drive.

I run `pg_dump` every 6 hours via cron. The dumps go to a local directory, then get shipped off-box by BorgBackup (more below).

```bash
# /etc/cron.d/pg-backup
0 */6 * * * root docker exec myapp-db-1 pg_dump -U postgres myapp_production | gzip > /backups/postgres/myapp-$(date +\%Y\%m\%d-\%H\%M).sql.gz
```

Old backups get pruned after 7 days. Simple. Reliable.

## Cache & Queues: Redis 7

Redis handles both caching and background job queues (via Sidekiq). It's running in Docker with an append-only file for persistence. Uses about 50MB of RAM for my workload.

Nothing fancy here. Redis just works. It's one of those tools where the best thing I can say is that I never think about it.

## Deploys: Kamal 2

[Kamal](https://kamal-deploy.org/) handles getting my app from a Git push to running in production. I wrote a whole post about it, but the short version: `kamal deploy` builds the Docker image, pushes it to my registry, SSHes into my servers, pulls the image, and does a zero-downtime swap. Takes about 45 seconds.

For CI, I use GitHub Actions. A push to `main` triggers the pipeline: run tests, build the image, then `kamal deploy`. The whole thing is maybe 30 lines of YAML.

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ruby/setup-ruby@v1
        with:
          bundler-cache: true
      - run: bundle exec rake test
      - run: bundle exec kamal deploy
        env:
          KAMAL_REGISTRY_PASSWORD: ${{ secrets.REGISTRY_PASSWORD }}
          RAILS_MASTER_KEY: ${{ secrets.RAILS_MASTER_KEY }}
```

## Monitoring: Uptime Kuma

[Uptime Kuma](https://github.com/louislam/uptime-kuma) runs on my second box and checks all my services every 60 seconds. HTTP checks, TCP port checks, certificate expiry checks. If anything goes down, I get a notification via Ntfy (self-hosted push notifications) and email.

The dashboard is clean and it was trivially easy to set up:

```bash
docker run -d --restart=unless-stopped \
  -p 3001:3001 \
  -v uptime-kuma:/app/data \
  louislam/uptime-kuma:1
```

It replaced a $30/mo Pingdom subscription and does more.

## Backups: BorgBackup + Backblaze B2

This is the part most indie devs skip. Don't be that person.

[BorgBackup](https://www.borgbackup.org/) is a deduplicating backup tool. It's fast, space-efficient, and encrypted. I run it nightly via cron, backing up:

- Postgres dumps (from the `pg_dump` cron above)
- Redis RDB snapshots
- App uploads directory
- All config files and docker-compose files

Borg repos live locally on the box first, then get synced to [Backblaze B2](https://www.backblaze.com/cloud-storage) using `rclone`. B2 costs $6/TB/month for storage and $0.01/GB for egress. My total B2 bill is about $2/mo.

```bash
# Nightly backup script
#!/bin/bash
export BORG_PASSPHRASE="$(cat /root/.borg-passphrase)"

borg create --compression zstd \
  /backups/borg::'{hostname}-{now}' \
  /backups/postgres \
  /data/uploads \
  /opt/myapp

borg prune --keep-daily=7 --keep-weekly=4 --keep-monthly=6 /backups/borg

rclone sync /backups/borg b2:myapp-backups --transfers 4
```

Test your restores. Seriously. A backup you've never tested is just a wish.

## Analytics: Plausible

[Plausible](https://plausible.io/) is a privacy-friendly, open-source analytics tool. I self-host it because I don't want to pay $9/mo for their cloud version and I like owning my data.

It's a single Docker Compose stack with a ClickHouse backend. Gives me pageviews, referrers, countries, device types. No cookies, no GDPR banner needed. The script is under 1KB.

Total replacement for Google Analytics, without the creepy tracking.

## Network Access: Tailscale

[Tailscale](https://tailscale.com/) is how I access everything that shouldn't be public. SSH, database admin, monitoring dashboards. It creates a WireGuard mesh network between my devices and servers.

I don't expose SSH to the public internet at all. Port 22 is firewalled. I SSH in over my Tailscale network. Same for Uptime Kuma's dashboard, Postgres connections for debugging, and anything else that's internal-only.

The free tier covers up to 100 devices and 3 users. For a solo dev, you'll never hit that limit.

```bash
# On the server
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --ssh

# Now SSH via Tailscale IP
ssh user@100.64.x.x
```

## The Total Bill

Let's add it up:

| Service | Monthly Cost |
|---|---|
| Hetzner CPX31 (main server) | $20 |
| Hetzner CX22 (monitoring/backup) | $5 |
| Backblaze B2 (backup storage) | ~$2 |
| Domain (amortized yearly) | ~$1 |
| GitHub (free tier) | $0 |
| Tailscale (free tier) | $0 |
| Plausible (self-hosted) | $0 |
| Uptime Kuma (self-hosted) | $0 |
| **Total** | **~$28/mo** |

Call it $30 with some rounding. That's my entire production infrastructure for three revenue-generating apps. I've seen single AWS Lambda functions cost more than that.

## What I'd Change

If I were starting from scratch today, I'd probably look at [Coolify](https://coolify.io/) as an all-in-one platform. It's basically a self-hosted Heroku and it handles deploys, SSL, databases, and monitoring in one package. But I've already got my stack dialed in and I like understanding every piece of it.

I'd also consider moving Postgres off-box to a managed database if my data got critical enough. Hetzner's managed Postgres starts at $15/mo and takes backups off my plate. Not there yet, but it's the obvious next upgrade.

## The Point

You don't need AWS. You don't need Vercel. You don't need a $500/mo cloud bill to run a profitable app. A cheap VPS, some battle-tested open source tools, and a couple hours of setup gets you a stack that's fast, cheap, and entirely under your control.

Stop paying the cloud tax. Ship on your own metal.

## Cheat Sheet

| Tool / Service | What It Does | Monthly Cost | Link |
|---|---|---|---|
| **Hetzner** | VPS provider with great price-to-performance and generous bandwidth | $5-$20 | [hetzner.com](https://www.hetzner.com/) |
| **Docker Compose** | Define and run multi-container apps in a single YAML file | Free | [docs.docker.com/compose](https://docs.docker.com/compose/) |
| **Caddy** | Reverse proxy with automatic HTTPS and Let's Encrypt certs | Free | [caddyserver.com](https://caddyserver.com/) |
| **PostgreSQL** | The database. Run it in Docker with volume mounts for persistence | Free | [postgresql.org](https://www.postgresql.org/) |
| **Redis** | In-memory cache and background job queue backend | Free | [redis.io](https://redis.io/) |
| **Kamal** | Zero-downtime Docker deploys over SSH from 37signals | Free | [kamal-deploy.org](https://kamal-deploy.org/) |
| **GitHub Actions** | CI pipeline: run tests, build images, trigger deploys on push to main | Free tier | [github.com/features/actions](https://github.com/features/actions) |
| **Uptime Kuma** | Self-hosted uptime monitoring with alerts via Ntfy, email, Slack | Free | [github.com/louislam/uptime-kuma](https://github.com/louislam/uptime-kuma) |
| **BorgBackup** | Deduplicating, encrypted backups. Run nightly via cron | Free | [borgbackup.org](https://www.borgbackup.org/) |
| **Backblaze B2** | Cheap object storage for off-site backup sync ($6/TB/mo) | ~$2 | [backblaze.com/cloud-storage](https://www.backblaze.com/cloud-storage) |
| **rclone** | CLI tool to sync local backups to B2 (or any cloud storage) | Free | [rclone.org](https://rclone.org/) |
| **Plausible** | Privacy-friendly, cookie-free web analytics. Self-host or use their cloud | Free (self-hosted) | [plausible.io](https://plausible.io/) |
| **Tailscale** | WireGuard mesh VPN for secure access to internal services | Free tier | [tailscale.com](https://tailscale.com/) |
| **Coolify** | Self-hosted Heroku alternative — deploys, SSL, DBs in one package | Free | [coolify.io](https://coolify.io/) |
| `pg_dump` via cron | Automated Postgres backups every 6 hours, gzipped and date-stamped | - | - |
| `restart: unless-stopped` | Docker restart policy that auto-recovers crashed containers | - | - |
