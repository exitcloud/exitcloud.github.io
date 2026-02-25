---
title: "Docker Compose Is All You Need"
date: 2025-10-20T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "docker"
---

Every few months someone on Hacker News posts about their Kubernetes setup for their side project. And every time, a little part of me dies. They've got Helm charts. They've got Argo CD. They've got a GitOps workflow with Flux. For an app that has 200 users.

Stop it.

[Docker Compose](https://docs.docker.com/compose/) is all you need. I run three profitable SaaS apps on it. Let me show you the actual setup.

## The Anti-Kubernetes Manifesto

Kubernetes solves real problems. If you're running 50 microservices across multiple regions with teams of developers who need independent deploy cycles and you've got a dedicated platform team — yeah, K8s makes sense. That's what it was built for.

But you're not doing that. You're one person (or maybe two) running a web app with a database and maybe a background worker. You've got one server. Maybe two if you're fancy.

For this, Kubernetes is like hiring a 747 to go get groceries. It technically works. But the fuel costs, the runway requirements, the pre-flight checklist — all of it is absurd overkill.

Docker Compose gives you:
- Declarative service definitions
- Networking between containers
- Volume management for persistent data
- Health checks and restart policies
- One-command deploys

That's the whole list. That's all you need.

## A Real docker-compose.yml

Here's the actual Compose file I use for one of my apps. It's a [Rails](https://rubyonrails.org/) API with [PostgreSQL](https://www.postgresql.org/), [Redis](https://redis.io/), Sidekiq for background jobs, and [Caddy](https://caddyserver.com/) as the reverse proxy. Nothing is redacted except domain names.

```yaml
version: "3.8"

services:
  app:
    build: .
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://app:${DB_PASSWORD}@db:5432/app_production
      REDIS_URL: redis://redis:6379/0
      RAILS_ENV: production
      SECRET_KEY_BASE: ${SECRET_KEY_BASE}
    volumes:
      - uploads:/app/public/uploads
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s
    deploy:
      resources:
        limits:
          memory: 512M

  worker:
    build: .
    command: bundle exec sidekiq -C config/sidekiq.yml
    restart: unless-stopped
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://app:${DB_PASSWORD}@db:5432/app_production
      REDIS_URL: redis://redis:6379/0
      RAILS_ENV: production
      SECRET_KEY_BASE: ${SECRET_KEY_BASE}
    deploy:
      resources:
        limits:
          memory: 512M

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: app_production
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 256M

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 192M

  caddy:
    image: caddy:2-alpine
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      app:
        condition: service_healthy

volumes:
  pgdata:
  redisdata:
  uploads:
  caddy_data:
  caddy_config:
```

And the `.env` file (not committed to git, obviously):

```bash
DB_PASSWORD=some-long-random-string
SECRET_KEY_BASE=another-long-random-string
```

That's the whole infrastructure. Five services, one file. Let me walk through the important bits.

## Health Checks Are Non-Negotiable

See those `healthcheck` blocks? They're doing real work. Without health checks, `depends_on` only waits for the container to *start* — not for the service inside to be *ready*. Your app container will boot, try to connect to Postgres, and crash because Postgres is still initializing.

With `condition: service_healthy`, Compose waits until the dependency passes its health check before starting dependents. This eliminates 90% of the "it works on second try" problems.

The `start_period` on the app container gives it 30 seconds to boot before health checks start counting failures. Rails apps need time to warm up. [Node.js](https://nodejs.org/) apps too, if you're precompiling assets.

## Restart Policies

`restart: unless-stopped` means: always restart the container if it crashes, unless I explicitly stopped it with `docker compose stop`. This is the right default for production. If your app segfaults at 3 AM, Docker brings it back up. If you stop it for maintenance, it stays stopped.

The alternative `restart: always` will restart even after a manual stop, which gets annoying when you're debugging.

## Volume Management (Don't Lose Your Data)

Named volumes (`pgdata`, `redisdata`, etc.) persist across container restarts and rebuilds. This is critical. If you use bind mounts (`./data:/var/lib/postgresql/data`), you risk permission issues and it's harder to back up.

Back up your volumes:

```bash
# Dump Postgres
docker compose exec db pg_dump -U app app_production > backup.sql

# Or copy the volume directly
docker run --rm -v myapp_pgdata:/data -v $(pwd):/backup alpine \
  tar czf /backup/pgdata.tar.gz -C /data .
```

I run Postgres dumps via cron every 6 hours and ship them to Backblaze B2 with rclone:

```bash
# /etc/cron.d/backup
0 */6 * * * root docker compose -f /home/deploy/app/docker-compose.yml exec -T db pg_dump -U app app_production | gzip > /home/deploy/backups/db-$(date +\%Y\%m\%d-\%H\%M).sql.gz && rclone copy /home/deploy/backups/ b2:my-backups/ --max-age 1d
```

## Resource Limits

Those `deploy.resources.limits.memory` blocks? They prevent a single service from eating all your RAM and taking down everything else. On a $20 box with 8GB RAM, I budget roughly:

- App: 512MB
- Worker: 512MB
- Postgres: 256MB
- Redis: 192MB (set both in Compose AND in redis.conf)
- Caddy: 64MB (it's extremely lightweight)
- OS + overhead: ~1GB

That's about 2.5GB committed, leaving 5.5GB for disk cache and headroom. Postgres *loves* disk cache — the OS will use that free RAM to cache frequently accessed database pages. Don't over-allocate to containers.

## The Deploy Script

Here's how I deploy:

```bash
#!/bin/bash
set -e

HOST="deploy@myapp.example.com"
APP_DIR="/home/deploy/app"

echo "Pulling latest code..."
ssh $HOST "cd $APP_DIR && git pull origin main"

echo "Building and restarting..."
ssh $HOST "cd $APP_DIR && docker compose build app && docker compose up -d"

echo "Running migrations..."
ssh $HOST "cd $APP_DIR && docker compose exec -T app bin/rails db:migrate"

echo "Checking health..."
sleep 5
curl -sf https://myapp.example.com/health && echo " OK" || echo " FAIL"
```

`docker compose up -d` is smart enough to only recreate containers whose configuration or image changed. If you only rebuilt the `app` image, Postgres and Redis keep running. Zero downtime for the database.

Want actual zero-downtime deploys? You can do a rolling restart:

```bash
docker compose up -d --no-deps --build app
```

This rebuilds and restarts just the app container. There's a brief moment during the restart where requests might fail, but Caddy will retry, and for most indie apps a 1-2 second blip is totally fine.

## Docker Compose Watch (The New Hotness)

Since Compose v2.22, there's `docker compose watch` which auto-rebuilds and restarts when files change. Great for development:

```yaml
# In your service definition
develop:
  watch:
    - action: rebuild
      path: .
      ignore:
        - node_modules/
        - .git/
```

Run `docker compose watch` and it's like having a file watcher that triggers rebuilds. Not for production, but it makes local dev feel snappy.

## When You Actually DO Need Kubernetes

Okay, I promised I'd be honest. Here's when Compose stops being enough:

1. **Multiple servers.** Compose runs on one host. If you need your app on three boxes behind a load balancer, Compose can't orchestrate that. (Docker Swarm can, sort of, but it's basically abandoned.)

2. **Dozens of services.** If you've got 20+ services with complex dependency graphs, Compose gets unwieldy. But also, ask yourself if you really need 20 services.

3. **Team needs independent deploys.** If multiple teams need to deploy different services on different schedules with different rollback windows, Compose's "everything in one file" model breaks down.

4. **Compliance requirements.** Some industries require specific orchestration features — rolling updates with health gates, automatic rollback, audit trails for deployments.

For 95% of indie dev and small-team scenarios? None of that applies. Ship with Compose.

## The One-Command Setup

New server? Here's the whole setup:

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh
usermod -aG docker deploy

# Clone your app
su - deploy
git clone https://github.com/youruser/yourapp.git app
cd app
cp .env.example .env
# Edit .env

# Ship it
docker compose up -d
```

Your entire infrastructure is a git repo with a `docker-compose.yml` and a `Caddyfile`. You can tear down the server, spin up a new one, clone the repo, restore the database backup, and be running again in under 15 minutes.

Try doing that with a Kubernetes cluster.

## Cheat Sheet

| Tool / Technique | What It Does | Link / Command |
|-----------------|-------------|----------------|
| **Docker Compose** | Declarative multi-container orchestration | [docs.docker.com/compose](https://docs.docker.com/compose/) |
| **Health checks** | Ensure services are actually ready, not just started | `healthcheck:` block in compose YAML |
| **condition: service_healthy** | Wait for real readiness before starting dependents | `depends_on: {db: {condition: service_healthy}}` |
| **restart: unless-stopped** | Auto-restart on crash, stay stopped on manual stop | Best default restart policy for prod |
| **Named volumes** | Persistent data that survives rebuilds | `volumes:` top-level key in compose |
| **Resource limits** | Prevent one container from eating all RAM | `deploy.resources.limits.memory` |
| **docker compose up -d** | Start/update all services (only recreates changed ones) | Primary deploy command |
| **docker compose watch** | Auto-rebuild on file changes (dev only) | `docker compose watch` (Compose v2.22+) |
| **pg_dump via cron** | Scheduled Postgres backups | `docker compose exec -T db pg_dump -U user dbname` |
| **rclone** | Ship backups to B2/S3/GCS | [rclone.org](https://rclone.org/) |
| **.env file** | Keep secrets out of compose YAML and git | `cp .env.example .env`, add to `.gitignore` |
| **docker compose logs -f** | Tail all service logs in one stream | Debugging lifesaver |
| **--no-deps --build** | Rebuild and restart one service without touching deps | `docker compose up -d --no-deps --build app` |
