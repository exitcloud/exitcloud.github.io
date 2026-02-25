---
title: "Coolify vs CapRover vs Dokku: Self-Hosted PaaS Showdown"
date: 2025-12-08T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "self-hosted"
---

At some point every indie dev hits the same wall. You've got 3-4 apps running on a VPS. You're SSH-ing in to deploy. You've got a pile of nginx configs, a docker-compose file that's getting unwieldy, and SSL certs that you forgot to renew last month. You think: "There has to be something between raw Docker and paying Vercel $20/month per app."

There is. Three somethings, actually: Coolify, CapRover, and Dokku. They're all self-hosted PaaS platforms. They all run on a single VPS. And they all handle the boring stuff -- builds, deploys, SSL, reverse proxy -- so you can focus on shipping.

I've run all three in production. Here's the honest breakdown.

## Dokku: The OG

Dokku has been around since 2013. It's basically "Heroku on a VPS" and it delivers exactly that promise. You `git push dokku main` and your app deploys. It uses Heroku buildpacks (or Dockerfiles), handles SSL via Let's Encrypt, and manages nginx routing.

### Install

```bash
# On a fresh Ubuntu 22.04+ box
wget -NP . https://dokku.com/install/v0.34.8/bootstrap.sh
sudo DOKKU_TAG=v0.34.8 bash bootstrap.sh
```

Takes about 5-10 minutes. Then:

```bash
# Set your domain
dokku domains:set-global yourdomain.com

# Create an app
dokku apps:create myapp

# Add a Postgres database
sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git
dokku postgres:create myapp-db
dokku postgres:link myapp-db myapp

# Deploy via git
git remote add dokku dokku@yourserver:myapp
git push dokku main
```

That's it. Your app is live with SSL. If you've ever used Heroku, this feels like home.

### The Good

- Battle-tested. 10+ years of production use. Bugs are rare.
- Plugin ecosystem is solid: Postgres, MySQL, Redis, Let's Encrypt, cron jobs.
- Tiny footprint. Dokku itself barely uses resources.
- `git push` deploys are satisfying every single time.

### The Not So Good

- No web UI. Everything is CLI. Fine for me, but some people want buttons.
- Single-server only. No built-in clustering.
- Heroku buildpack builds can be slow. Dockerfile builds are faster.
- Debugging build failures with buildpacks can be a yak-shaving session.

## CapRover: The Reliable Middle Ground

CapRover has been around since 2017 (originally CaptainDuckDuck -- yes, really). It gives you a web UI, one-click app deployments from a marketplace, and Docker-native workflows.

### Install

```bash
# Need Docker installed first
docker run -p 80:80 -p 443:443 -p 3000:3000 \
  -e ACCEPTED_TERMS=true \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v captain-data:/captain \
  caprover/caprover
```

Or the recommended way:

```bash
mkdir /captain && docker run -e ACCEPTED_TERMS=true \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /captain:/captain \
  caprover/caprover
```

Then access the web UI at `http://your-ip:3000`, set up your root domain, and enable HTTPS. CapRover handles Let's Encrypt automatically.

### Deploy Options

CapRover gives you multiple deploy paths:

```bash
# Install the CLI
npm install -g caprover

# Deploy from Dockerfile
caprover deploy

# Or use the web UI to deploy from:
# - Git repo URL
# - Uploaded tar file
# - Docker image
# - One-click apps (marketplace)
```

The one-click apps marketplace is nice. Need Postgres? Ghost? n8n? Plausible Analytics? Click, fill in a few env vars, deploy. Feels like a self-hosted app store.

### The Good

- Clean web UI for managing everything.
- One-click app marketplace with 100+ apps.
- Docker-native. If it runs in Docker, it runs on CapRover.
- Built-in cluster support (multiple servers).
- Persistent app logs in the UI.

### The Not So Good

- Uses more RAM than Dokku (~300-500MB for the CapRover stack itself).
- The web UI can feel sluggish on a $5 VPS.
- Documentation is decent but has gaps.
- Updating CapRover itself can occasionally be finicky.

## Coolify: The New Hotness

Coolify (v4) is what happens when someone looks at Vercel's UI and says "I want that, but self-hosted." It's the newest of the three (v4 released in 2024), and it's been gaining steam fast. The developer, Andras Bacsai, has been shipping features at a ridiculous pace.

### Install

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

One command. Seriously. It sets up Docker, the Coolify stack, and a reverse proxy (Traefik). You get a web UI at `http://your-ip:8000`.

### What Makes It Different

Coolify feels more like a modern cloud platform than a traditional PaaS. It supports:

- **Git-based deploys** with auto-deploy on push (GitHub/GitLab/Bitbucket webhooks)
- **Docker Compose deploys** -- paste your compose file, done
- **Nixpacks** for buildpack-style builds (faster than Heroku buildpacks)
- **Databases as first-class resources** -- spin up Postgres, MySQL, Redis, MongoDB, etc. with a click
- **Server management** -- manage multiple remote servers from one Coolify instance
- **Built-in backups** for databases
- **Preview deployments** for pull requests

That last one is the Vercel killer feature. PR preview deploys on your own infra.

### Deploy Workflow

```
1. Connect your GitHub repo in the UI
2. Pick the branch
3. Coolify detects your framework (Next.js, Rails, etc.) or Dockerfile
4. Set env vars
5. Click deploy
6. Get a URL with SSL
```

Every subsequent push to that branch triggers a new deploy. You can watch the build logs in real-time. It's genuinely nice.

### The Good

- Best UI of the three. Modern, fast, well-designed.
- Multi-server support out of the box.
- PR preview deployments.
- Active development. New features landing monthly.
- Database management with built-in backups.
- Traefik-based proxy (more flexible than nginx for some setups).

### The Not So Good

- Youngest project. You'll hit rough edges. Bugs exist.
- Heavier resource usage (~500-800MB RAM for the Coolify stack).
- Traefik config can be confusing if you need to customize it.
- Some features feel half-baked (improving rapidly though).
- Breaking changes between updates have happened.

## The Comparison Table

| Feature | Dokku | CapRover | Coolify |
|---|---|---|---|
| **First release** | 2013 | 2017 | 2022 (v4: 2024) |
| **Web UI** | No (CLI only) | Yes | Yes (best of the three) |
| **Deploy method** | `git push`, Docker | CLI, UI, Docker, marketplace | Git webhook, Docker, Compose, UI |
| **Build system** | Buildpacks, Dockerfile | Dockerfile | Nixpacks, Dockerfile, Compose |
| **SSL** | Let's Encrypt plugin | Let's Encrypt built-in | Let's Encrypt built-in |
| **Reverse proxy** | nginx | nginx | Traefik |
| **Multi-server** | No | Yes | Yes |
| **PR previews** | No | No | Yes |
| **One-click apps** | No (plugins) | Yes (100+ marketplace) | Yes (growing library) |
| **Database mgmt** | Plugins | One-click apps | Built-in with backups |
| **RAM overhead** | ~50-100MB | ~300-500MB | ~500-800MB |
| **Min VPS** | 1GB RAM | 2GB RAM | 2GB RAM (4GB preferred) |
| **Learning curve** | Medium (CLI comfort needed) | Low | Low |
| **Stability** | Rock solid | Solid | Good but younger |

## Who Should Use What

**Pick Dokku if:**
- You're comfortable with the terminal and prefer CLI over GUIs
- You're running on a tiny VPS (1GB RAM)
- You want the most battle-tested, least-surprises option
- You like the `git push` deploy workflow
- You don't need multi-server or PR previews

**Pick CapRover if:**
- You want a web UI but also want something stable
- You like the one-click app marketplace concept
- You need cluster support
- You're managing apps for non-technical teammates who need a dashboard

**Pick Coolify if:**
- You want the most modern, Vercel-like experience
- PR preview deploys matter to you
- You're managing multiple servers
- You want integrated database management with backups
- You don't mind being on the bleeding edge
- You have at least 2GB RAM (4GB is better)

## My Current Setup

I run Coolify on a 4GB RAM VPS ($12/month on Hetzner). It manages three servers -- itself plus two app servers. The PR preview deploys sold me. Being able to push a branch and get a live preview URL without any extra config is something I didn't know I needed until I had it.

That said, I kept a Dokku box around for my simplest apps. Some things don't need a UI. `git push dokku main` and move on. There's something pure about it.

If I were starting fresh with a single 2GB VPS and wanted zero fuss, I'd go CapRover. It hits the sweet spot of features-to-complexity. But Coolify is where the momentum is, and it's only getting better.

## Cheat Sheet

| Tool | One-Liner | Link |
|---|---|---|
| **Dokku** | Heroku-on-a-VPS. CLI-only, tiny footprint, rock solid | [dokku.com](https://dokku.com/) |
| **CapRover** | Docker-native PaaS with web UI and app marketplace | [caprover.com](https://caprover.com/) |
| **Coolify** | Modern self-hosted Vercel alternative with PR previews | [coolify.io](https://coolify.io/) |
| `dokku apps:create myapp` | Create a new Dokku app | -- |
| `git push dokku main` | Deploy to Dokku via git | -- |
| `caprover deploy` | Deploy to CapRover from CLI | -- |
| `curl -fsSL https://cdn.coollabs.io/coolify/install.sh \| bash` | Install Coolify in one command | -- |
| **Nixpacks** | Coolify's fast build system (alternative to Heroku buildpacks) | [nixpacks.com](https://nixpacks.com/) |
| **Traefik** | Reverse proxy used by Coolify (auto-configures routes) | [traefik.io](https://traefik.io/) |
| **Hetzner** | Cheap European VPS provider, great for self-hosted PaaS | [hetzner.com](https://www.hetzner.com/) |
| **Min RAM guideline** | Dokku: 1GB, CapRover: 2GB, Coolify: 2-4GB | -- |
