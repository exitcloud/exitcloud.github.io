---
title: "Setting Up Caddy: Goodbye Nginx Config Hell"
date: 2025-10-30T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "self-hosted"
---

I used [Nginx](https://nginx.org/) for ten years. I wrote more `location` blocks than I can count. I debugged `proxy_set_header` issues at 2 AM. I ran certbot cron jobs that silently broke and left me with expired certificates on a Friday afternoon.

Then I switched to [Caddy](https://caddyserver.com/), and I'm never going back.

## The Nginx Config That Made Me Snap

Here's the nginx config for a typical reverse proxy setup — HTTPS, redirect HTTP, proxy to an app, WebSocket support, some basic headers. This is *minimal*:

```nginx
server {
    listen 80;
    server_name myapp.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name myapp.example.com;

    ssl_certificate /etc/letsencrypt/live/myapp.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_stapling on;
    ssl_stapling_verify on;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_buffering off;
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
}
```

Plus you need certbot installed, a cron job or [systemd](https://systemd.io/) timer for renewal, and a post-renewal hook to reload nginx. And you need to have run `certbot certonly` before nginx can even start, because nginx will refuse to boot if the certificate files don't exist yet. Chicken-and-egg problem.

Now here's the equivalent Caddyfile:

```
myapp.example.com {
    reverse_proxy localhost:3000
}
```

Three lines. That's it.

Caddy handles: obtaining the TLS certificate from [Let's Encrypt](https://letsencrypt.org/), renewing it automatically, HTTP-to-HTTPS redirect, setting the right proxy headers, HTTP/2, OCSP stapling — all of it. Out of the box. Zero config.

I stared at this for about five minutes the first time, waiting for the catch. There isn't one.

## Installing Caddy

On [Ubuntu](https://ubuntu.com/)/Debian:

```bash
apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | \
  gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | \
  tee /etc/apt/sources.list.d/caddy-stable.list
apt update && apt install caddy -y
```

Or in [Docker](https://docs.docker.com/compose/):

```yaml
caddy:
  image: caddy:2-alpine
  ports:
    - "80:80"
    - "443:443"
  volumes:
    - ./Caddyfile:/etc/caddy/Caddyfile:ro
    - caddy_data:/data
    - caddy_config:/config
```

The `caddy_data` volume is critical — that's where it stores your TLS certificates. Lose that volume and Caddy has to re-issue certs (which it'll do automatically, but Let's Encrypt has rate limits).

## Real Caddyfile Configs

Let me show you configs for actual use cases I run in production.

### Multiple apps on one box

```
app1.example.com {
    reverse_proxy localhost:3000
}

app2.example.com {
    reverse_proxy localhost:3001
}

status.example.com {
    reverse_proxy localhost:3002
}
```

Each domain gets its own certificate. Caddy manages all of them. I've run up to eight apps behind one Caddy instance without any issues.

### Static site + API

```
example.com {
    handle /api/* {
        reverse_proxy localhost:8080
    }

    handle {
        root * /var/www/site
        file_server
    }
}
```

The `handle` directive is order-dependent — first match wins. API requests go to your backend, everything else gets served from the filesystem. Caddy will set the right `Content-Type` headers, handle `gzip` compression (on by default), and serve `index.html` for directory requests.

### With rate limiting and basic auth

```
admin.example.com {
    basicauth {
        admin $2a$14$Zkx19XLiW6VYouLkR1BIwOSfB7AtXoWMqR9KVIt33GRQHQ8JKWvS6
    }
    reverse_proxy localhost:8080
}

api.example.com {
    rate_limit {
        zone api {
            key {remote_host}
            events 100
            window 1m
        }
    }
    reverse_proxy localhost:3000
}
```

(The rate limiting requires building Caddy with the `caddy-ratelimit` plugin, or using `xcaddy`. More on that in a sec.)

Generate a bcrypt hash for basicauth with: `caddy hash-password`

### WebSocket support

Here's the thing: **you don't need to configure anything.** Caddy's `reverse_proxy` handles WebSocket upgrades automatically. No `proxy_set_header Upgrade` dance. No `Connection "upgrade"` header. It just works.

If you've spent any time debugging nginx WebSocket configs, you know what a relief this is.

## Caddy in Docker Compose

Here's how Caddy fits into a full [Docker Compose](https://docs.docker.com/compose/) setup. The gotcha most people hit is Docker networking — your app isn't on `localhost` from Caddy's perspective; it's on the Docker network.

```yaml
services:
  app:
    build: .
    # No ports exposed to host! Caddy handles that.
    expose:
      - "3000"

  caddy:
    image: caddy:2-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    depends_on:
      - app

volumes:
  caddy_data:
  caddy_config:
```

And the Caddyfile uses the Docker service name instead of localhost:

```
myapp.example.com {
    reverse_proxy app:3000
}
```

Notice `app:3000` not `localhost:3000`. Docker's internal DNS resolves the service name `app` to the container's IP. This trips up everyone the first time. If your Caddyfile says `localhost:3000` inside Docker, it's looking for port 3000 on the Caddy container itself — which obviously isn't where your app is running.

Also note: the `app` service uses `expose` instead of `ports`. This makes port 3000 accessible to other containers on the Docker network but NOT to the outside world. Only Caddy's ports 80 and 443 are exposed. Clean separation.

## Custom Caddy Builds with xcaddy

Need plugins? Caddy has a module system. Want DNS challenge for wildcard certs? Rate limiting? IP geolocation? Build a custom Caddy binary:

```bash
# Install xcaddy
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest

# Build with plugins
xcaddy build \
  --with github.com/caddy-dns/cloudflare \
  --with github.com/mholt/caddy-ratelimit
```

Or use a Dockerfile:

```dockerfile
FROM caddy:2-builder AS builder
RUN xcaddy build \
  --with github.com/caddy-dns/cloudflare \
  --with github.com/mholt/caddy-ratelimit

FROM caddy:2-alpine
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

## Wildcard Certificates

This is the one place Caddy requires a bit more config. Wildcard certs (`*.example.com`) can't use the standard HTTP challenge — you need the DNS challenge. That means Caddy needs API access to your DNS provider.

With [Cloudflare](https://www.cloudflare.com/) (and the `caddy-dns/cloudflare` plugin):

```
*.example.com {
    tls {
        dns cloudflare {env.CF_API_TOKEN}
    }

    @app1 host app1.example.com
    handle @app1 {
        reverse_proxy localhost:3000
    }

    @app2 host app2.example.com
    handle @app2 {
        reverse_proxy localhost:3001
    }
}
```

Set the `CF_API_TOKEN` environment variable to a Cloudflare API token with Zone:DNS:Edit permissions. Caddy will create a TXT record, Let's Encrypt will verify it, and you get a wildcard cert. Renewal is automatic.

Most of the time you don't need wildcards, though. Individual certs per domain work fine and don't require DNS plugins.

## Real-World Gotchas

A few things I've learned the hard way:

**1. Caddy reloads are graceful, restarts aren't.** Use `caddy reload` (or `systemctl reload caddy`) when you change the Caddyfile. A full restart (`systemctl restart caddy`) briefly drops connections. In Docker, `docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile` is your friend.

**2. Let's Encrypt rate limits.** You get 50 certificates per registered domain per week. If you're testing and keep tearing down / recreating your Caddy setup, you'll hit this. Use Let's Encrypt's staging environment while testing:

```
{
    acme_ca https://acme-staging-v02.api.letsencrypt.org/directory
}
```

Remove that block when you go to production.

**3. Docker health check ordering.** If Caddy starts before your app is ready, it'll try to proxy to a dead backend. Use health checks and `depends_on: condition: service_healthy` in your Compose file. Caddy itself will return 502 for a backend that's down, which is fine — it'll start working once the app boots. But it's cleaner to wait.

**4. The admin API.** Caddy exposes an admin API on `localhost:2019` by default. In production, make sure this isn't exposed to the internet. In Docker it's fine (no port mapping). On a bare-metal install, the default binding to localhost is safe. But double-check.

**5. Logs.** Caddy logs to stderr by default in a JSON format. To get structured access logs:

```
myapp.example.com {
    log {
        output file /var/log/caddy/access.log
        format json
    }
    reverse_proxy localhost:3000
}
```

## The Migration

Switching from nginx to Caddy took me about 20 minutes per app. The Caddyfile is so much shorter that you're not really "converting" — you're rewriting from scratch in a fraction of the lines.

My nginx config directory had 847 lines across 6 files. The equivalent Caddyfile: 34 lines. One file. No includes, no `sites-available`/`sites-enabled` symlink dance, no `nginx -t` finger-crossing.

Caddy's a single static binary with zero dependencies. It just does the right thing by default. After a decade of nginx, that still feels like magic.

## Cheat Sheet

| Tool / Technique | What It Does | Link / Command |
|-----------------|-------------|----------------|
| **Caddy** | Web server + reverse proxy + auto HTTPS | [caddyserver.com](https://caddyserver.com/) |
| **Caddyfile** | Human-readable config format, 3 lines for a reverse proxy | [Caddyfile docs](https://caddyserver.com/docs/caddyfile) |
| **Auto TLS** | Caddy auto-obtains and renews Let's Encrypt certificates | Zero config, just use a domain name |
| **reverse_proxy** | Proxy requests to your app (handles WebSocket automatically) | `reverse_proxy localhost:3000` |
| **handle directive** | Route requests to different backends by path/host | `handle /api/* { reverse_proxy ... }` |
| **xcaddy** | Build custom Caddy with plugins (DNS challenge, rate limit) | `go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest` |
| **caddy-dns/cloudflare** | DNS challenge plugin for wildcard certs via Cloudflare | [github.com/caddy-dns/cloudflare](https://github.com/caddy-dns/cloudflare) |
| **caddy reload** | Graceful config reload (no dropped connections) | `systemctl reload caddy` or `caddy reload` |
| **expose vs ports** | In Docker: `expose` = internal only, `ports` = host-accessible | Use `expose` for app, `ports` for Caddy |
| **Docker service DNS** | Inside Compose, use service name not localhost | `reverse_proxy app:3000` (not `localhost:3000`) |
| **caddy_data volume** | Stores TLS certs — never delete this | Mount as named volume in Docker |
| **Staging ACME** | Avoid Let's Encrypt rate limits while testing | `acme_ca https://acme-staging-v02.api.letsencrypt.org/directory` |
| **caddy hash-password** | Generate bcrypt hash for basicauth | `caddy hash-password` |
| **Admin API** | Caddy's control API on localhost:2019 | Don't expose to internet |
