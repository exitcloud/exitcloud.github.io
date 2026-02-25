---
title: "The $5 VPS Challenge: Running a Production App on the Cheapest Box"
date: 2025-10-10T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "VPS"
---

Here's the challenge: take the cheapest VPS you can find — we're talking $5/month or less — and run a real production app on it. Not a demo. Not a "hello world." A real app with users, a database, TLS, and the whole deal.

Can you actually do it? Yeah. You can. Let me show you.

## The Box

I grabbed a **Hetzner CAX11** — their smallest ARM box. Here's what $3.90/month gets you:

- 2 ARM cores (Ampere Altra)
- 4 GB RAM
- 40 GB NVMe disk
- 20 TB transfer

If you prefer x86 or a US-based provider, Vultr's cheapest is $5/month for 1 vCPU, 1 GB RAM, 25 GB SSD. The Hetzner ARM box is objectively better spec-for-spec, but either works.

For this walkthrough I'll use the Hetzner box running Ubuntu 24.04.

## Initial Setup (10 minutes)

First things first. SSH in and lock it down:

```bash
# Update everything
apt update && apt upgrade -y

# Create a non-root user
adduser deploy
usermod -aG sudo deploy

# Set up SSH key auth (from your local machine)
ssh-copy-id deploy@your-server-ip

# Disable password auth
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart sshd

# Basic firewall
ufw allow OpenSSH
ufw allow 80
ufw allow 443
ufw enable

# Auto security updates
apt install unattended-upgrades -y
dpkg-reconfigure -plow unattended-upgrades
```

That's your base. Took maybe 10 minutes. Now let's put an app on it.

## The App: A Node.js API with SQLite

I'm going to deploy a small SaaS API — a link shortener with user accounts, rate limiting, and analytics. It's a real app that I actually run. Here's the stack:

- **Node.js 20** (could be Bun, could be Rails, whatever you ship with)
- **SQLite** with WAL mode (more on this in a future post)
- **Caddy** for reverse proxy + auto TLS
- **systemd** to keep everything running

Why not Docker? On a box this small, Docker's overhead actually matters. The daemon eats ~100MB of RAM just sitting there. When you've only got 4GB, that's 2.5% of your memory doing nothing. Systemd is already there and costs zero.

## Installing the Runtime

```bash
# Node.js via NodeSource
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
apt install nodejs -y

# Verify
node --version  # v20.x.x
```

## Deploying the App

I keep it stupid simple. Git clone, npm install, systemd service.

```bash
# As the deploy user
cd /home/deploy
git clone https://github.com/youruser/linkshort.git app
cd app
npm ci --production
cp .env.example .env
# Edit .env with your actual values
```

Now create a systemd service so it starts on boot and restarts on crash:

```ini
# /etc/systemd/system/linkshort.service
[Unit]
Description=LinkShort API
After=network.target

[Service]
Type=simple
User=deploy
WorkingDirectory=/home/deploy/app
ExecStart=/usr/bin/node src/index.js
Restart=always
RestartSec=5
Environment=NODE_ENV=production
EnvironmentFile=/home/deploy/app/.env

# Hardening
NoNewPrivileges=true
ProtectSystem=strict
ReadWritePaths=/home/deploy/app/data

[Install]
WantedBy=multi-user.target
```

```bash
systemctl enable linkshort
systemctl start linkshort

# Check it's running
systemctl status linkshort
curl http://localhost:3000/health
```

That `Restart=always` with `RestartSec=5` means if your app crashes, systemd brings it back up in 5 seconds. No process manager needed. No pm2. Systemd already does this.

## Caddy for TLS

Caddy is a single binary that gives you automatic HTTPS. No certbot cron jobs. No nginx config files that look like they were written by a Perl programmer having a bad day.

```bash
# Install Caddy
apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update
apt install caddy -y
```

The entire Caddyfile:

```
links.yourdomain.com {
    reverse_proxy localhost:3000
}
```

That's it. Three lines. Caddy handles getting a Let's Encrypt certificate, renewing it, redirecting HTTP to HTTPS, and proxying requests to your app. Reload with `systemctl reload caddy` and you're live.

## Real Resource Usage

Here's what this box actually looks like under normal load (~50 requests/minute, which is plenty for a small SaaS):

```
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       412Mi       2.1Gi       1.0Mi       1.3Gi       3.4Gi

$ htop (summary)
  CPU: 3-5% average
  Tasks: 42
  Load average: 0.08, 0.05, 0.02
```

The Node.js process uses about **80MB RSS**. Caddy uses **15MB**. SQLite uses whatever Node allocates for it — basically nothing extra. The OS and everything else accounts for maybe 300MB. We've got **3.4GB free**.

Disk usage:

```
$ df -h /
Filesystem      Size  Used Avail Use%
/dev/sda1        39G  3.2G   34G   9%
```

3.2GB used. That's the OS, Node, Caddy, the app code, and the SQLite database. 34GB free.

This box is *bored*. It could handle 10x this traffic without breaking a sweat.

## Backups

SQLite makes backups dead simple. The database is one file. But you can't just `cp` it while the app is running — you might catch it mid-write. Two options:

**Option 1: sqlite3 .backup command (cron job)**

```bash
# /etc/cron.d/backup-db
0 */6 * * * deploy sqlite3 /home/deploy/app/data/app.db ".backup /home/deploy/backups/app-$(date +\%Y\%m\%d-\%H\%M).db" && rclone copy /home/deploy/backups/ b2:my-backups/linkshort/ --max-age 1d
```

**Option 2: Litestream (continuous replication)**

```yaml
# /etc/litestream.yml
dbs:
  - path: /home/deploy/app/data/app.db
    replicas:
      - url: s3://my-bucket/linkshort
        endpoint: https://s3.us-west-000.backblazeb2.com
```

Litestream continuously streams WAL changes to S3-compatible storage. Your RPO (recovery point objective) drops to seconds instead of hours. It uses barely any CPU or bandwidth. I use it for everything now.

## Deploy Script

Here's my entire deploy process:

```bash
#!/bin/bash
# deploy.sh - run from your local machine
set -e

HOST="deploy@links.yourdomain.com"

ssh $HOST "cd /home/deploy/app && git pull origin main && npm ci --production"
ssh $HOST "sudo systemctl restart linkshort"

echo "Deployed. Checking health..."
sleep 2
curl -sf https://links.yourdomain.com/health && echo " OK" || echo " FAIL"
```

`git pull`, `npm install`, restart the service. That's the whole deploy. Takes about 15 seconds. No CI pipeline. No Docker build. No container registry. Just files on a server.

(Yes, I know, "but what about rollbacks?" Git revert. Or `git checkout HEAD~1`. Done.)

## When $5 Stops Being Enough

Real talk: when do you need to upgrade?

- **RAM is the first bottleneck.** If your app starts swapping, bump to the next tier. On Hetzner that's the CAX21 at $8.50 for 8GB. Still absurdly cheap.
- **CPU usually isn't the problem** for web apps. Most time is spent waiting on I/O.
- **Disk fills up** if you're storing uploads or have a growing database. Add a volume or move uploads to object storage (Backblaze B2 or Hetzner's object storage).
- **If you need multiple servers** — for redundancy or because you're actually getting serious traffic — that's when you graduate to Docker Compose on a bigger box, or maybe two boxes with a load balancer. But we're talking tens of thousands of requests per minute before you're anywhere near that.

Most indie apps will never outgrow a $5-10 box. That's just the reality. We've been sold a narrative that you need complex infrastructure from day one. You don't. Ship on the small box. Upgrade when the metrics tell you to, not when your anxiety tells you to.

## Total Monthly Cost

| Item | Cost |
|------|------|
| Hetzner CAX11 | $3.90 |
| Domain name (amortized) | ~$1.00 |
| Backblaze B2 backups | ~$0.10 |
| **Total** | **~$5.00** |

Five bucks. A production app with TLS, backups, auto-restarts, and room to grow. No excuse not to ship that side project now.

## Cheat Sheet

| Tool / Technique | What It Does | Link / Command |
|-----------------|-------------|----------------|
| **Hetzner CAX11** | Cheapest ARM VPS, 2 cores / 4GB / 40GB for $3.90/mo | [hetzner.com/cloud](https://www.hetzner.com/cloud) |
| **Vultr $5 tier** | x86 alternative, 1 vCPU / 1GB / 25GB | [vultr.com](https://www.vultr.com/) |
| **ufw** | Simple firewall, allow only SSH + HTTP + HTTPS | `ufw allow OpenSSH && ufw allow 80 && ufw allow 443 && ufw enable` |
| **unattended-upgrades** | Auto-install security patches | `apt install unattended-upgrades` |
| **systemd service** | Run your app as a daemon with auto-restart | Create `/etc/systemd/system/yourapp.service` |
| **Restart=always** | Systemd restarts your app on crash, no pm2 needed | Add to `[Service]` section |
| **Caddy** | Reverse proxy + automatic HTTPS in 3 lines of config | [caddyserver.com](https://caddyserver.com/) |
| **SQLite + WAL** | Embedded database, no separate process | `PRAGMA journal_mode=WAL;` |
| **Litestream** | Continuous SQLite replication to S3 | [litestream.io](https://litestream.io/) |
| **rclone** | Sync backups to any cloud storage provider | [rclone.org](https://rclone.org/) |
| **Backblaze B2** | Cheapest S3-compatible storage for backups | [backblaze.com/b2](https://www.backblaze.com/b2/cloud-storage.html) |
| **Deploy script** | `ssh box "git pull && npm ci && sudo systemctl restart app"` | — |
| **Rollback** | `ssh box "cd /app && git checkout HEAD~1 && sudo systemctl restart app"` | — |
