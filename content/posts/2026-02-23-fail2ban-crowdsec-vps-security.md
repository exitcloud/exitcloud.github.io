---
title: "Fail2Ban + CrowdSec: Securing Your VPS Without a WAF Bill"
date: 2026-02-23T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "security"
  - "devops"
---

# Fail2Ban + CrowdSec: Securing Your VPS Without a WAF Bill

The first thing that happens when you spin up a new VPS is somebody tries to break into it. I'm not exaggerating. Within minutes of a fresh [Hetzner](https://www.hetzner.com/) or [DigitalOcean](https://www.digitalocean.com/) box getting an IP, you'll see brute-force SSH attempts in your auth log. Bots scan the entire IPv4 space constantly.

I once spun up a box and forgot to check on it for a day. When I looked at the logs, there'd been over 40,000 failed SSH login attempts from 200+ different IPs. In 24 hours. On a server that didn't even have a domain pointed at it yet.

You need to lock this down before you deploy a single line of application code. Here's how I do it, and it doesn't cost a dime.

## Step 1: SSH Hardening

Before we get to the fancy tools, let's do the basics. These take five minutes and block 90% of automated attacks.

### Key-Only Authentication

Disable password auth entirely. If you're still SSHing in with a password, stop reading this and fix it right now.

```bash
# Generate a key pair on your local machine (if you don't have one)
ssh-keygen -t ed25519 -C "your@email.com"

# Copy it to your server
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@your-server-ip
```

Then lock down the SSH config on the server:

```bash
# /etc/ssh/sshd_config
PermitRootLogin prohibit-password
PasswordAuthentication no
PubkeyAuthentication yes
AuthenticationMethods publickey
MaxAuthTries 3
LoginGraceTime 30
X11Forwarding no
AllowAgentForwarding no
```

Restart SSH:

```bash
sudo systemctl restart sshd
```

**Important:** Before you restart, make sure you can actually log in with your key in a separate terminal. Don't lock yourself out. I've done it. It's embarrassing. You'll be filing a support ticket asking for console access.

### Change the Default Port

This is security through obscurity, and yes, it won't stop a determined attacker. But it kills 99% of the bot traffic, which means cleaner logs and fewer Fail2Ban triggers.

```bash
# /etc/ssh/sshd_config
Port 2222
```

Pick any high port. Update your SSH config locally so you don't have to remember it:

```bash
# ~/.ssh/config
Host myserver
    HostName 49.12.45.101
    Port 2222
    User deploy
    IdentityFile ~/.ssh/id_ed25519
```

Now you just type `ssh myserver`. Done.

## Step 2: UFW — The Firewall You'll Actually Use

[UFW](https://help.ubuntu.com/community/UFW) (Uncomplicated Firewall) is a frontend for iptables. It lives up to its name.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH on your custom port
sudo ufw allow 2222/tcp comment 'SSH'

# Allow HTTP and HTTPS
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'

# If you use Tailscale, allow its port
sudo ufw allow 41641/udp comment 'Tailscale'

sudo ufw enable
sudo ufw status verbose
```

That's your baseline. Everything is denied by default except the ports you explicitly opened. If you add a new service later, you add a rule. If you remove a service, you remove the rule.

One footgun to watch out for: if you're running [Docker](https://docs.docker.com/compose/), it manipulates iptables directly and can bypass UFW rules. This is a well-known issue. The fix is to configure Docker to use `iptables: false` in `/etc/docker/daemon.json` and manage port exposure through UFW instead, or use the `DOCKER-USER` chain. Look into it if you're running public-facing containers.

## Step 3: Fail2Ban — The Battle-Tested Bouncer

[Fail2Ban](https://www.fail2ban.org/) has been around forever and it does one thing well: it watches log files for patterns (failed logins, 404 spam, brute force) and adds firewall rules to ban the offending IPs.

Install it:

```bash
sudo apt install fail2ban
```

Create a local config (never edit the main config directly — it gets overwritten on updates):

```bash
# /etc/fail2ban/jail.local
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 3
banaction = ufw

[sshd]
enabled = true
port = 2222
logpath = /var/log/auth.log
maxretry = 3
bantime = 24h

[nginx-http-auth]
enabled = true
logpath = /var/log/nginx/error.log
maxretry = 3

[nginx-botsearch]
enabled = true
logpath = /var/log/nginx/access.log
maxretry = 3
bantime = 1w
```

The key settings:
- **`bantime`**: How long an IP stays banned. I use 24 hours for SSH and 1 week for bot scanners.
- **`findtime`**: The window in which `maxretry` failures must occur.
- **`maxretry`**: How many failures before the ban hammer drops.
- **`banaction = ufw`**: Use UFW for banning instead of raw iptables. Cleaner.

Start it up:

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

Check what's happening:

```bash
sudo fail2ban-client status sshd
```

You'll see something like:

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 2
|  |- Total failed: 847
|  `- File list: /var/log/auth.log
`- Actions
   |- Currently banned: 14
   |- Total banned: 203
   `- Banned IP list: 103.x.x.x 45.x.x.x ...
```

203 banned IPs. And that's on a relatively quiet server. Fail2Ban is doing its job.

## Step 4: CrowdSec — The Modern Alternative

[Fail2Ban](https://www.fail2ban.org/) works great, but it's purely reactive and local. It only knows about attacks against *your* server. [CrowdSec](https://www.crowdsec.net/) takes a different approach: it's a community-driven security engine. When one CrowdSec user detects an attacker, that IP gets shared with the entire network. You benefit from everyone else's detection.

Think of it as a neighborhood watch for the internet.

### Architecture

CrowdSec has two main components:

- **The Engine (Agent):** Parses your logs, detects attack patterns, makes decisions about IPs.
- **Bouncers:** Enforcement points that actually block the traffic. There are bouncers for iptables/nftables, [Nginx](https://nginx.org/), [Traefik](https://traefik.io/), [Caddy](https://caddyserver.com/), [Cloudflare](https://www.cloudflare.com/), and more.

### Install

```bash
curl -s https://install.crowdsec.net | sudo sh
sudo apt install crowdsec
sudo apt install crowdsec-firewall-bouncer-iptables
```

CrowdSec comes with built-in "collections" — sets of parsers and detection scenarios for common services. Install the ones you need:

```bash
# SSH brute force detection
sudo cscli collections install crowdsecurity/sshd

# Nginx/HTTP attack detection
sudo cscli collections install crowdsecurity/nginx

# Generic Linux detection
sudo cscli collections install crowdsecurity/linux
```

### Community Blocklists

This is where CrowdSec shines. When you enroll your instance (free), you automatically start receiving community blocklists. These are IPs that have been flagged by other CrowdSec users worldwide. Your server starts blocking known attackers *before* they even hit your logs.

```bash
# Enroll your instance (get your key from app.crowdsec.net)
sudo cscli console enroll <your-enrollment-key>
```

After enrollment, check what's being blocked:

```bash
sudo cscli decisions list
```

You'll likely see hundreds of IPs being preemptively blocked based on community intelligence. It's a beautiful thing.

### CrowdSec vs. Fail2Ban

Should you run both? I do. Here's why:

Fail2Ban is simpler and more predictable. I know exactly what it's doing because I wrote the jail configs. It's my first line of defense and it's been running without issues for years.

CrowdSec adds the community layer. It catches things Fail2Ban misses because it has a global view of attack patterns. The bouncer architecture also means you can block traffic at the Nginx level before it even hits your app, which is more efficient than firewall rules for HTTP attacks.

They don't conflict. Fail2Ban watches logs and adds UFW rules. CrowdSec watches logs and adds iptables rules via its bouncer. Different chains, no collisions.

## Step 5: The Security Checklist

Here's the checklist I run through every time I set up a new box. Print it out. Tape it to your monitor. Don't skip steps.

- [ ] SSH key-only auth, passwords disabled
- [ ] SSH on non-default port
- [ ] Root login disabled (or key-only)
- [ ] UFW enabled, default deny incoming
- [ ] Only required ports open (SSH, 80, 443)
- [ ] Fail2Ban installed and running for SSH + web
- [ ] CrowdSec installed with community blocklists
- [ ] Automatic security updates enabled (`unattended-upgrades`)
- [ ] Docker configured to not bypass UFW
- [ ] [Tailscale](https://tailscale.com/) for internal access instead of exposing admin ports
- [ ] Backups running and tested (you *do* have backups, right?)
- [ ] Log rotation configured so logs don't fill the disk

```bash
# Enable automatic security updates on Ubuntu/Debian
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

## The Reality Check

None of this makes you invulnerable. A determined attacker with a zero-day isn't going to be stopped by Fail2Ban. But that's not the threat model for most indie devs. Your threat model is:

1. Automated bots brute-forcing SSH — **solved** by key-only auth + Fail2Ban
2. Script kiddies scanning for known exploits — **solved** by keeping packages updated + CrowdSec blocklists
3. Credential stuffing on your app's login page — **solved** by Fail2Ban jails on your app logs + rate limiting
4. DDoS — honestly, if you're getting DDoSed, put [Cloudflare](https://www.cloudflare.com/) in front. Their free tier handles it fine.

This stack costs $0/mo, takes about 30 minutes to set up, and will stop the vast majority of attacks against your VPS. You don't need a $200/mo WAF subscription. You don't need Cloudflare Pro. You just need some basic hygiene and a couple of battle-tested tools.

Lock it down, then ship your app.

## Cheat Sheet

| Tool / Technique | What It Does | Link |
|---|---|---|
| **Key-only SSH** | Disable passwords, use ed25519 keys. Blocks all password brute force | - |
| **Non-default SSH port** | Move SSH off port 22 to kill 99% of bot traffic | - |
| **UFW** | Simple iptables frontend. Default deny, whitelist only needed ports | [help.ubuntu.com/community/UFW](https://help.ubuntu.com/community/UFW) |
| **Fail2Ban** | Watches logs for attack patterns, bans IPs via firewall rules | [fail2ban.org](https://www.fail2ban.org/) |
| `jail.local` | Local Fail2Ban config — define ban times, retry limits, log paths per service | - |
| `banaction = ufw` | Tell Fail2Ban to use UFW instead of raw iptables for cleaner rule management | - |
| **CrowdSec** | Community-driven security engine with shared blocklists | [crowdsec.net](https://www.crowdsec.net/) |
| **CrowdSec Bouncers** | Enforcement plugins for iptables, Nginx, Caddy, Traefik, Cloudflare | - |
| `cscli collections install` | Add detection scenarios for SSH, Nginx, Linux, etc. | - |
| `cscli console enroll` | Connect to community blocklists — block known-bad IPs preemptively | - |
| **unattended-upgrades** | Automatic security patches on Debian/Ubuntu. Set it and forget it | - |
| **Tailscale** | WireGuard VPN mesh — access admin ports without exposing them publicly | [tailscale.com](https://tailscale.com/) |
| **Docker + UFW gotcha** | Docker bypasses UFW by default. Set `iptables: false` or use `DOCKER-USER` chain | - |
| **Cloudflare (free tier)** | DDoS protection when you actually need it. Proxy your DNS records | [cloudflare.com](https://www.cloudflare.com/) |
| `ssh-keygen -t ed25519` | Generate a modern, secure SSH key pair | - |
| `fail2ban-client status` | Check how many IPs are currently banned per jail | - |
