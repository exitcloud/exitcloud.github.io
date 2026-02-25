---
title: "Kamal 2: Zero-Downtime Deploys Without the Complexity"
date: 2026-02-17T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "deployment"
---

# Kamal 2: Zero-Downtime Deploys Without the Complexity

I spent three years at a FAANG company watching a team of six SREs manage a Kubernetes cluster so we could deploy a Rails app. Six people. Full-time. Just to keep the deploy pipeline alive.

Then I discovered Kamal, and I felt like an idiot for not leaving sooner.

[Kamal](https://kamal-deploy.org/) is the deploy tool built by 37signals — the folks behind Basecamp and HEY. It's what they use to ship their own apps. Version 2 landed with some serious improvements, and it's become the backbone of how I deploy everything. Zero-downtime deploys to bare metal or cheap VPSes, no orchestrator required.

Let me walk you through the whole thing.

## What Kamal Actually Is

Kamal is a deploy tool that uses Docker and SSH. That's it. No daemon running on your server. No agent to install. No cluster to manage. You define your deploy config in a single `deploy.yml`, run `kamal deploy`, and it SSHes into your boxes, pulls your container image, and swaps traffic over with zero downtime.

Under the hood, it uses [Traefik](https://traefik.io/) as a reverse proxy to handle the blue-green switchover. When you deploy, Kamal boots your new container, waits for it to pass health checks, then tells Traefik to route traffic to the new container. Only after the new container is healthy does it tear down the old one. No dropped requests.

Compare that to Capistrano, which was the old standard. Cap was fine for deploying Ruby apps directly onto servers, but it couldn't do zero-downtime without a bunch of extra gymnastics. And Kubernetes? K8s can absolutely do zero-downtime deploys, but the operational overhead is absurd for a small team. You're paying the complexity tax on every single thing you do.

Kamal sits right in the sweet spot: container-based deploys with zero-downtime, but without the yak shaving.

## Installing Kamal

Kamal is a Ruby gem. You don't need a Rails app to use it — it deploys any Docker container — but you do need Ruby installed.

```bash
gem install kamal
kamal version
```

That's it. If you're in a Rails project, add it to your Gemfile instead:

```ruby
gem "kamal", "~> 2.0"
```

Then run `kamal init` to generate the config scaffold:

```bash
kamal init
```

This creates `config/deploy.yml` and a `.kamal/` directory with your secrets setup.

## The deploy.yml

Here's a real `deploy.yml` from one of my production apps. I've changed the domain and IPs, but everything else is exactly what I run:

```yaml
service: myapp
image: registry.example.com/myapp

servers:
  web:
    hosts:
      - 49.12.45.101
      - 49.12.45.102
    labels:
      traefik.http.routers.myapp.rule: Host(`myapp.example.com`)
      traefik.http.routers.myapp.tls.certresolver: letsencrypt
  worker:
    hosts:
      - 49.12.45.103
    cmd: bundle exec sidekiq

registry:
  server: registry.example.com
  username: deploy
  password:
    - KAMAL_REGISTRY_PASSWORD

env:
  clear:
    RAILS_ENV: production
    RAILS_LOG_TO_STDOUT: "true"
    DB_HOST: 49.12.45.110
  secret:
    - RAILS_MASTER_KEY
    - DATABASE_URL
    - REDIS_URL

proxy:
  ssl: true
  host: myapp.example.com
  app_port: 3000
  healthcheck:
    path: /up
    interval: 3

accessories:
  db:
    image: postgres:16
    host: 49.12.45.110
    port: 5432
    env:
      clear:
        POSTGRES_DB: myapp_production
      secret:
        - POSTGRES_PASSWORD
    directories:
      - data:/var/lib/postgresql/data

  redis:
    image: redis:7
    host: 49.12.45.110
    port: 6379
    directories:
      - data:/data
```

A few things to notice. The `servers` section separates web and worker roles. Kamal only puts Traefik in front of web servers — your workers just run their command directly. The `accessories` section manages your database and Redis. These aren't redeployed every time you ship code; they're long-running services that Kamal can set up and manage for you.

The `proxy` section is new in Kamal 2. It replaced the old `traefik` config block with something cleaner. You define your health check path, and Kamal handles the rest.

## Secrets

Kamal 2 uses `.kamal/secrets` for secret management. It's a simple `.env`-style file that never gets committed:

```bash
# .kamal/secrets
KAMAL_REGISTRY_PASSWORD=ghp_xxxxxxxxxxxx
RAILS_MASTER_KEY=abc123def456
DATABASE_URL=postgres://deploy:secretpass@49.12.45.110/myapp_production
REDIS_URL=redis://49.12.45.110:6379/0
POSTGRES_PASSWORD=secretpass
```

You can also pull secrets from 1Password, LastPass, or AWS SSM. But honestly? A `.env` file on your local machine works fine for most indie setups. Just make sure it's in your `.gitignore`.

## The Deploy Cycle

Here's what a deploy actually looks like:

```bash
$ kamal deploy
  INFO [f1a5c2d3] Running docker login ...
  INFO [f1a5c2d3] Finished in 1.2s
  INFO Building image...
  INFO Pushing image to registry.example.com/myapp:abc123f...
  INFO [49.12.45.101] Pulling image...
  INFO [49.12.45.102] Pulling image...
  INFO [49.12.45.101] Starting container with labels...
  INFO [49.12.45.101] Waiting for health check /up...
  INFO [49.12.45.101] Container is healthy
  INFO [49.12.45.101] Swapping proxy target...
  INFO [49.12.45.101] Stopping old container...
  INFO [49.12.45.102] Starting container with labels...
  INFO [49.12.45.102] Waiting for health check /up...
  INFO [49.12.45.102] Container is healthy
  INFO [49.12.45.102] Swapping proxy target...
  INFO [49.12.45.102] Stopping old container...
  INFO [49.12.45.103] Pulling image (worker)...
  INFO [49.12.45.103] Starting worker container...
  Finished deploy in 47.3s
```

Roughly 47 seconds from `kamal deploy` to fully live on two web servers and a worker. During that entire time, your users never see a blip. The old containers keep serving traffic until the new ones are healthy.

Other commands you'll use constantly:

```bash
kamal app logs             # Tail logs from all servers
kamal app logs -f          # Follow mode
kamal app exec 'bin/rails console'  # Open a Rails console
kamal lock status          # Check if a deploy is in progress
kamal rollback abc123      # Roll back to a specific version
kamal accessory reboot db  # Restart your database accessory
```

## Blue-Green Under the Hood

The zero-downtime magic works like this. Kamal runs the Traefik proxy on each web server as a Docker container (Kamal 2 actually ships its own lightweight proxy called `kamal-proxy` now, replacing the Traefik dependency). When you deploy:

1. New container starts on a random port
2. Kamal hits the health check endpoint repeatedly
3. Once healthy, Kamal tells the proxy to route traffic to the new container
4. Old container gets a graceful shutdown signal (SIGTERM)
5. After a drain period, the old container is removed

This is effectively blue-green deployment per host. At no point are both containers receiving traffic — the switchover is atomic from the proxy's perspective.

If the health check fails, Kamal aborts. The old container keeps running. Nothing changes. Your app stays up.

## Rolling Restarts

If you have multiple web servers, Kamal deploys to them sequentially by default. Server 1 gets the new version, health check passes, traffic switches. Then server 2. This means you always have at least one server running the old version while the next one is being updated.

You can control the parallelism with the `drain_timeout` and boot settings if you want, but the default sequential behavior is what you want for most setups. It's the safest approach.

## When Kamal Doesn't Fit

Be honest about the limitations. If you need auto-scaling based on load, Kamal won't do that. If you need to manage 50+ microservices with complex service mesh routing, yeah, maybe Kubernetes makes sense. If you need canary deploys with traffic splitting — 10% to the new version, 90% to old — Kamal doesn't support that natively.

But here's the thing: most indie apps don't need any of that. You've got one app, maybe a worker, a database, and Redis. Kamal handles that perfectly. You're not Netflix. Ship the feature.

## Cheat Sheet

| Tool / Concept | What It Does | Link |
|---|---|---|
| **Kamal** | Docker + SSH deploy tool from 37signals. Zero-downtime deploys without K8s | [kamal-deploy.org](https://kamal-deploy.org/) |
| `kamal init` | Scaffold `deploy.yml` and secrets directory in your project | - |
| `kamal deploy` | Build, push, pull, and swap containers with zero downtime | - |
| `kamal rollback <sha>` | Instantly roll back to a previous image version | - |
| `kamal app logs -f` | Tail production logs across all servers | - |
| `kamal app exec` | Run one-off commands (Rails console, migrations) on a server | - |
| `kamal accessory reboot` | Restart accessories (DB, Redis) without touching your app | - |
| `deploy.yml` | Single config file defining servers, registry, env, accessories | - |
| `.kamal/secrets` | Local `.env`-style file for deploy secrets (never committed) | - |
| **kamal-proxy** | Lightweight proxy replacing Traefik in Kamal 2 for blue-green swaps | - |
| **Blue-green deploy** | New container starts, passes health check, then proxy swaps atomically | - |
| **Rolling restart** | Sequential deploy across hosts — always one healthy server running | - |
| **Accessories** | Long-running services (Postgres, Redis) managed by Kamal but not redeployed on every push | - |
| **Health checks** | Define a `/up` endpoint; Kamal won't swap traffic until it passes | - |
| **Traefik** | Reverse proxy Kamal 1 used; Kamal 2 ships its own `kamal-proxy` instead | [traefik.io](https://traefik.io/) |
