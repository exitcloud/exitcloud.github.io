---
title: "Monitoring Your Stack with Uptime Kuma"
date: 2025-11-28T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "monitoring"
---

Here's a thing that happened to me: my side project went down for 14 hours and I didn't notice until a user DMed me on Twitter. The app was making ~$400/month at the time. I did the math later -- that outage probably cost me a couple of sign-ups and definitely cost me some trust. All because my [Docker](https://docs.docker.com/compose/) container ran out of memory and I had zero monitoring.

You'd think the fix is "just use [Datadog](https://www.datadoghq.com/)" or "just use [Better Uptime](https://betteruptime.com/)." And yeah, those work. They also cost $20-50/month per service once you get past the free tier. For an indie dev running three or four apps on cheap boxes, that adds up fast.

Enter [Uptime Kuma](https://github.com/louislam/uptime-kuma). Self-hosted. Open source. Takes about 5 minutes to deploy. Does everything I need.

## Spin It Up

One Docker command and you're running:

```bash
docker run -d \
  --name uptime-kuma \
  --restart=unless-stopped \
  -p 3001:3001 \
  -v uptime-kuma:/app/data \
  louislam/uptime-kuma:1
```

Or if you're a [docker-compose](https://docs.docker.com/compose/) person (you should be):

```yaml
# docker-compose.yml
version: "3"
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - uptime-kuma-data:/app/data

volumes:
  uptime-kuma-data:
```

```bash
docker compose up -d
```

Hit `http://your-server:3001`, create an admin account, and you're in. The whole UI is clean and obvious. No 45-minute onboarding wizard. No "talk to sales."

## Putting It Behind a Reverse Proxy

You probably want this on a real domain with HTTPS. Here's a minimal [nginx](https://nginx.org/) config:

```nginx
server {
    listen 443 ssl http2;
    server_name status.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/status.yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/status.yourdomain.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

The WebSocket headers (`Upgrade` and `Connection`) matter here. Uptime Kuma uses them for the real-time dashboard. Skip those and you'll get a dashboard that looks frozen.

## What to Monitor

Here's where people overthink it. You don't need 47 checks per service. Start with these:

### HTTP Checks (Your Apps)

The bread and butter. Point it at your app's URL, set the interval to 60 seconds, and pick a keyword to verify. I always check for a specific string in the response body -- not just a 200 status code. An app can return 200 with an error page.

```
Type: HTTP(s)
URL: https://myapp.com/api/health
Method: GET
Expected Status: 200
Keyword: "status":"ok"
Interval: 60 seconds
Retries: 3
```

That retry count matters. Set it to at least 2-3. Networks blip. You don't want a 2am alert because one packet got dropped.

### TCP Checks (Databases, Redis)

For services that don't speak HTTP:

```
Type: TCP Port
Hostname: 127.0.0.1
Port: 5432 (PostgreSQL)
Interval: 120 seconds
```

I run these for Postgres, Redis, and any other backing services. If your database port stops responding, you want to know before your users do.

### DNS Checks

This one's underrated. Monitor that your domains are resolving correctly:

```
Type: DNS
Hostname: myapp.com
Resolver: 1.1.1.1
Expected Record Type: A
Expected Value: 203.0.113.10
```

I once had a DNS propagation issue after a migration that took 6 hours to notice. A DNS check would've caught it in 60 seconds.

### Docker Container Checks

If Uptime Kuma is on the same box as your apps (common for indie setups), you can monitor Docker containers directly. Mount the Docker socket:

```yaml
volumes:
  - uptime-kuma-data:/app/data
  - /var/run/docker.sock:/var/run/docker.sock:ro
```

Then add a monitor with type "Docker Container" and pick the container from the dropdown. It checks if the container is running. Simple and effective.

**Security note:** mounting the Docker socket gives Uptime Kuma root-level access to your Docker host. On a single-tenant indie VPS this is fine. On a shared box, think twice.

## Notification Channels

Uptime Kuma supports something like 90+ notification services. Here are the ones I actually use:

### Discord Webhook

Create a webhook in your [Discord](https://discord.com/) server (Server Settings > Integrations > Webhooks). Paste the URL into Uptime Kuma. Done. I have a `#alerts` channel in my project's Discord that gets pinged on any status change.

### Telegram Bot

1. Message `@BotFather` on [Telegram](https://telegram.org/), create a bot, get the token
2. Start a chat with your bot, get your chat ID (message `@userinfobot`)
3. Plug both into Uptime Kuma

Telegram notifications hit your phone instantly. This is my primary "wake me up" channel.

### Email (SMTP)

For the formal record. I use a free [Resend](https://resend.com/) account or just my existing SMTP setup:

```
SMTP Host: smtp.resend.com
SMTP Port: 465
Security: TLS
Username: resend
Password: re_xxxxxxxxxxxx
From: alerts@yourdomain.com
```

### My Recommendation

Use two channels minimum. I do Telegram for immediate alerts and Discord for the team-visible log. If one service is down, the other probably isn't.

## Status Pages

This is the feature that makes Uptime Kuma punch way above its weight. You can create a public status page that shows your monitors' uptime history. It looks professional, it's customizable, and it's free.

Go to "Status Pages" in the sidebar, create one, add your monitors, and publish it. You can map it to a custom domain like `status.yourdomain.com`.

Your users get a clean page showing what's up and what's down. You look like you have your act together. Win-win.

## How It Compares to Paid Alternatives

I've used most of these at some point. Here's my honest take:

| Feature | Uptime Kuma (Free) | Better Uptime ($20+/mo) | Datadog ($15+/host/mo) | UptimeRobot (Free/$7/mo) |
|---|---|---|---|---|
| HTTP/TCP/DNS checks | Yes | Yes | Yes | Yes |
| Docker container checks | Yes | No | Yes (agent) | No |
| Notification channels | 90+ | ~15 | Tons | ~10 |
| Status pages | Yes | Yes | Yes (extra $) | Yes (paid) |
| Self-hosted | Yes | No | No | No |
| Setup time | 5 min | 10 min | 30+ min | 5 min |
| Data retention | Unlimited (your disk) | Plan-dependent | Plan-dependent | Plan-dependent |
| Cost for 20 monitors | $0 | $20/mo | $$$$ | $7/mo |

[UptimeRobot](https://uptimerobot.com/)'s free tier is actually decent (50 monitors, 5-min intervals), and I used it for years. But the 5-minute check interval means up to 5 minutes before you know something's down. Uptime Kuma lets you go down to 20-second intervals if you want.

## Resource Usage

Uptime Kuma is surprisingly light. On my box with ~30 monitors running at 60-second intervals:

- **RAM:** ~150MB
- **CPU:** Basically nothing (0.1-0.5%)
- **Disk:** ~500MB after a year of data

You can comfortably run it on the same $5 VPS as your apps. I wouldn't put it on the *same* box you're monitoring if you can avoid it (if the box goes down, so does your monitoring), but for indie devs starting out, it's fine. Once you're making money, spin up a second tiny VPS just for monitoring.

## The "Monitor Your Monitor" Problem

If Uptime Kuma goes down, who monitors Uptime Kuma? This is the turtles-all-the-way-down problem. My solution: I keep a free UptimeRobot account with one check pointed at my Uptime Kuma instance. UptimeRobot monitors Uptime Kuma. Uptime Kuma monitors everything else. Two layers. Good enough.

## Quick Setup Recap

1. `docker compose up -d` with the compose file above
2. Create admin account at `:3001`
3. Add HTTP monitors for each of your apps (with keyword checks)
4. Add TCP monitors for databases
5. Set up Telegram + Discord notifications
6. Create a status page
7. Point UptimeRobot at your Uptime Kuma instance
8. Go back to building features

The whole thing takes maybe 20 minutes including the Telegram bot setup. That's less time than you'll spend debugging your next outage without monitoring.

## Cheat Sheet

| Tool/Technique | What/Why | Link |
|---|---|---|
| **Uptime Kuma** | Self-hosted monitoring with 90+ notification channels | [github.com/louislam/uptime-kuma](https://github.com/louislam/uptime-kuma) |
| `docker compose up -d` | Deploy Uptime Kuma in one command | -- |
| **HTTP keyword check** | Verify response body content, not just status codes | -- |
| **TCP check** | Monitor non-HTTP services (Postgres, Redis) | -- |
| **DNS check** | Catch DNS propagation issues or misconfigs | -- |
| **Docker container check** | Monitor container status (mount Docker socket read-only) | -- |
| **Telegram bot** | Fastest push notification channel for alerts | [core.telegram.org/bots](https://core.telegram.org/bots) |
| **Discord webhook** | Team-visible alert log in your project's server | -- |
| **Status pages** | Free public uptime dashboard for your users | -- |
| **Retry count >= 3** | Avoid false alarms from network blips | -- |
| **UptimeRobot free tier** | Use it to monitor your Uptime Kuma instance itself | [uptimerobot.com](https://uptimerobot.com/) |
| **WebSocket proxy headers** | Add `Upgrade` + `Connection` headers in nginx for real-time UI | -- |
| **Separate monitoring box** | Don't monitor a server from itself if you can avoid it | -- |
