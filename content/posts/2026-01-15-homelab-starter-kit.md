---
title: "Homelab Starter Kit: What to Buy and What to Skip"
date: 2026-01-15T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "homelab"
  - "self-hosted"
---

I spent way too long researching my first homelab setup. Read every Reddit thread, watched every YouTube video, compared spreadsheets of mini PCs. Classic yak shaving. Here's what I wish someone had told me: just buy a mini PC and start deploying stuff. You can optimize later.

This is the guide I needed two years ago.

## Picking your hardware

You have three real options. Let me save you some time.

### Option 1: Mini PC ($150-300) — the right answer for most people

A used or refurbished mini PC is the sweet spot. Something like a [Beelink](https://www.bee-link.com/) SER5 or a Lenovo ThinkCentre Tiny. You get a real x86 CPU, 16-32GB of RAM, an NVMe slot, and it sips power — usually 10-25 watts under load.

What I run: a Beelink SER5 with a Ryzen 5 5560U, 32GB RAM, 500GB NVMe. Paid $180 refurbished. It sits on a shelf in my closet, pulls about 15 watts, and is dead silent. It runs everything I need.

### Option 2: Raspberry Pi ($50-100) — good for dedicated single-purpose boxes

A [Pi 5](https://www.raspberrypi.com/) with 8GB is genuinely capable now. Good for [Pi-hole](https://pi-hole.net/), [Home Assistant](https://www.home-assistant.io/), or a dedicated [WireGuard](https://www.wireguard.com/) endpoint. Bad for anything that needs real CPU or more than 8GB of RAM. Also bad if you want to run a bunch of [Docker](https://docs.docker.com/compose/) containers — the SD card I/O will make you miserable (use a USB SSD if you go this route).

I have a Pi 4 running Pi-hole. That's it. Everything else runs on the mini PC.

### Option 3: Old laptop or desktop ($0-50) — the "what's in your closet" option

That ThinkPad from 2018 gathering dust? Wipe it, install Ubuntu Server, done. Free hardware is free hardware. Downsides: laptops are awkward to rack or shelf-mount, fans can be loud, and power draw is usually higher than a mini PC. But if you're starting from zero and don't want to spend money, this works.

### What to skip

Skip used enterprise rack servers. That Dell PowerEdge R720 for $80 on eBay will eat 200+ watts at idle, sound like a jet engine, and your partner/roommates will hate you. Homelabs are fun until your electricity bill doubles and you can hear the fans from two rooms away.

Skip NAS-specific hardware unless you actually need a NAS. A [Synology](https://www.synology.com/) is great for storage, but it's a mediocre Docker host. Get a mini PC and plug in an external drive if you need storage.

## Software: Proxmox vs bare Docker

Two schools of thought here.

**[Proxmox](https://www.proxmox.com/)** is a hypervisor — it lets you spin up virtual machines and containers from a web UI. It's great if you want to experiment with different OSes, snapshot before you break things, or isolate workloads properly. It's also heavier and adds complexity.

**Bare Docker on Ubuntu/Debian** is simpler. Install Docker, write some compose files, done. Less overhead, fewer layers to debug. If something breaks, you're debugging one Linux box, not a VM inside a hypervisor inside a box.

My recommendation: **start with bare Docker**. You can always migrate to Proxmox later when you have a reason to. Most homelab use cases don't need VM isolation. Docker compose files are portable — you can move them to a VPS, another machine, wherever.

```bash
# Fresh Ubuntu box? Here's your starting point.
sudo apt update && sudo apt install -y docker.io docker-compose-v2
sudo usermod -aG docker $USER
# Log out and back in, then:
docker compose version
```

## Networking: Tailscale changes everything

The single best homelab decision I made was installing [Tailscale](https://tailscale.com/) on every machine. It creates a WireGuard mesh network across all your devices — your homelab, your laptop, your phone. No port forwarding. No dynamic DNS. No exposing anything to the public internet.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

That's it. Now your homelab box has a stable IP on your Tailscale network (like `100.x.y.z`) and you can reach it from anywhere. Your phone on cellular? Works. Coffee shop WiFi? Works. It's magic and it's free for personal use (up to 100 devices).

For services you want to expose with nice URLs, run [Caddy](https://caddyserver.com/) as a reverse proxy on the homelab box:

```
pihole.home.ts.net {
    reverse_proxy localhost:8080
}
nextcloud.home.ts.net {
    reverse_proxy localhost:8443
}
```

Tailscale can even provision HTTPS certs for `.ts.net` domains. No more self-signed cert warnings.

## What to self-host first

In order of "most immediately useful":

**1. Pi-hole** — Network-wide ad blocking. Install it, point your router's DNS at it, enjoy an ad-free home network. This is the gateway drug of self-hosting.

```yaml
# docker-compose.yml
services:
  pihole:
    image: pihole/pihole:latest
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:80/tcp"
    environment:
      TZ: 'America/New_York'
      WEBPASSWORD: 'changeme'
    volumes:
      - './etc-pihole:/etc/pihole'
      - './etc-dnsmasq.d:/etc/dnsmasq.d'
    restart: unless-stopped
```

**2. [Uptime Kuma](https://github.com/louislam/uptime-kuma)** — A beautiful uptime monitoring dashboard. Monitor your own services, external sites, anything with a URL. Alerts via email, Slack, Discord.

**3. [Nextcloud](https://nextcloud.com/)** — Replace Google Drive/Dropbox. Calendar, contacts, file sync. It's heavy and sometimes annoying to maintain, but the privacy payoff is real.

**4. Media stack** — [Jellyfin](https://jellyfin.org/) for streaming your media library. Way better than Plex's recent direction. Pair it with *arr apps if you're into automated media management.

**5. [Vaultwarden](https://github.com/dani-garcia/vaultwarden)** — Self-hosted Bitwarden. Your passwords, your server. Compatible with all Bitwarden apps.

## Power and noise — the stuff nobody talks about

This matters more than you think, especially if the box lives in your apartment.

My mini PC: ~15W idle, ~25W under load. That's about $2-3/month in electricity. Completely silent.

A used enterprise server: 150-300W idle. That's $20-40/month. And it sounds like a server room because it is a server room.

Measure with a Kill-A-Watt meter ($20 on Amazon). Know what you're spending before you commit to running something 24/7.

## Budget breakdown

Here's what a solid starter homelab actually costs:

| Item | Cost |
|------|------|
| Beelink SER5 mini PC (refurb) | $180 |
| Extra 500GB NVMe (optional) | $40 |
| Raspberry Pi 5 for Pi-hole (optional) | $80 |
| Kill-A-Watt meter | $20 |
| Tailscale | Free |
| Docker + all the software | Free |
| **Total (minimum)** | **$180** |
| **Total (comfortable)** | **$320** |

You don't need to spend $500+ to start. A single $180 mini PC running Docker can host a dozen services without breaking a sweat. Start small, add hardware when you actually need it.

## The honest take

A homelab is a hobby that pretends to be infrastructure. You'll tell yourself it's for "learning" and "privacy" — and it is — but mostly it's because tinkering with servers is fun. Own that. Just don't let the tinkering become the point. Deploy things you actually use, not things that look cool on a dashboard.

Ship services, not dashboards.

## Cheat Sheet

| Item | What / Why | Link |
|------|-----------|------|
| Beelink SER5 mini PC | Best bang-for-buck homelab hardware, ~15W, silent | [Beelink](https://www.bee-link.com/) |
| Raspberry Pi 5 | Good for single-purpose boxes (Pi-hole, WireGuard) | [raspberrypi.com](https://www.raspberrypi.com/) |
| Docker + Compose | Run everything in containers, skip Proxmox to start | [docs.docker.com](https://docs.docker.com/) |
| Tailscale | WireGuard mesh VPN, free tier, no port forwarding needed | [tailscale.com](https://tailscale.com/) |
| Caddy | Reverse proxy with automatic HTTPS, simple config | [caddyserver.com](https://caddyserver.com/) |
| Pi-hole | Network-wide ad blocking, first thing to self-host | [pi-hole.net](https://pi-hole.net/) |
| Uptime Kuma | Self-hosted uptime monitoring with pretty dashboards | [GitHub](https://github.com/louislam/uptime-kuma) |
| Nextcloud | Self-hosted file sync, calendar, contacts | [nextcloud.com](https://nextcloud.com/) |
| Jellyfin | Free media server, better direction than Plex | [jellyfin.org](https://jellyfin.org/) |
| Vaultwarden | Self-hosted Bitwarden-compatible password manager | [GitHub](https://github.com/dani-garcia/vaultwarden) |
| Kill-A-Watt meter | Measure actual power draw before committing to 24/7 | ~$20 on Amazon |
| **Rule of thumb** | Start with one mini PC + Docker. Add hardware when you hit a real limit, not before. | — |
