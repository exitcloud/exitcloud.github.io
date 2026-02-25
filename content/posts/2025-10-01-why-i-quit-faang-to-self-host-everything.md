---
title: "Why I Quit FAANG to Self-Host Everything"
date: 2025-10-01T09:00:00Z
draft: false
tags:
  - "opinion"
  - "indie-dev"
  - "self-hosted"
---

I spent six years at a company you've definitely heard of. Good pay. Good people, mostly. Free snacks. The whole deal. And I quit to run my apps on $20 servers I rent from a German hosting company.

People think I'm crazy. Let me explain why I think *they* are.

## The Moment I Knew

It was a Tuesday. I needed to deploy a config change. One line. A feature flag. In a sane world, this is a `git push` and a 30-second deploy.

Here's what actually happened: I opened a PR. Waited for three CI pipelines to pass (42 minutes). Got a code review from someone in a different timezone (next day). Merged to staging. Waited for the staging deploy (another pipeline, 18 minutes). Filed a deploy ticket for production. Waited for the deploy window. Got a rollback plan approved. Then — finally — my one-line change hit prod. Through a 47-service mesh. With a canary deployment. Monitored by four different observability platforms.

Total elapsed time: about 36 hours.

For a feature flag.

I sat there staring at my monitor thinking, "I could have SSH'd into a box and changed an environment variable in 10 seconds." And that thought never left.

## The Cloud Bill That Broke Me

Around the same time, I started doing napkin math on what our team's infrastructure actually cost. I'm not talking about the whole company — just our corner of it. One mid-sized service with a [PostgreSQL](https://www.postgresql.org/) database, a [Redis](https://redis.io/) cache, some background workers, and a CDN.

The monthly AWS bill for this setup? Roughly **$4,200/month**. And that was *after* someone on the team had done a cost optimization pass.

RDS instance: $800. ElastiCache: $400. A couple of ECS Fargate tasks: $600. Load balancer: $200. CloudWatch logs and metrics: $350. Data transfer: who even knows, AWS pricing is designed by sadists. The rest was a grab bag of S3, SNS, SQS, and services I'm not sure anyone was still using.

You know what that same workload costs me now? **$20/month.** A [Hetzner](https://www.hetzner.com/) CAX21 (4 ARM cores, 8GB RAM, 80GB disk). Running everything on one box. Postgres, Redis, the app, [Caddy](https://caddyserver.com/) for TLS. All in [Docker Compose](https://docs.docker.com/compose/). And it handles more traffic than that old service ever did.

I'm not exaggerating. I'm not running a toy. I'm running real apps with real users who pay real money, and my entire infrastructure bill is less than what my old team spent on CloudWatch *alone*.

## What "Self-Hosting" Actually Means

Let me clear something up: self-hosting doesn't mean you're running servers in your closet. (Though some people do that. Respect.) For me, it means renting cheap VPS boxes from providers like Hetzner, [Vultr](https://www.vultr.com/), or OVH, and managing everything myself.

The stack is simple:

- **A VPS** — usually Hetzner. Their ARM boxes are absurd value.
- **Docker Compose** — one `docker-compose.yml` to rule them all.
- **Caddy** — reverse proxy with automatic TLS. No more [Let's Encrypt](https://letsencrypt.org/) cron jobs.
- **[SQLite](https://sqlite.org/) or Postgres** — depends on the app. SQLite for anything single-server.
- **[Litestream](https://litestream.io/) or pg_dump** — backups to S3-compatible storage (Backblaze B2, $0.005/GB).
- **A simple deploy script** — `ssh box "cd /app && git pull && docker compose up -d"`. Done.

No Kubernetes. No [Terraform](https://www.terraform.io/) (usually). No service mesh. No config management tool that takes three days to learn. Just files on a server.

## The Freedom

Here's what nobody tells you about FAANG: you don't own anything. Not your code, not your infrastructure, not your deploy process. Everything is mediated through internal tools that work great until they don't, and when they don't, you file a ticket and wait.

When I own the box, I own the problem. Sounds scary, right? It's actually the opposite. When something breaks at 2 AM, I can SSH in and fix it. I don't need to page someone on the platform team. I don't need to wait for an internal tool to come back online. I `ssh` in, read the logs, fix it, move on.

Last month my app went down because I forgot to set up log rotation and `/var/log` filled the disk. Took me 4 minutes to fix. At my old job, that would've been a P2 incident with a post-mortem document and three follow-up action items tracked in Jira.

## The Trade-offs (Being Honest)

Look, I'm not going to pretend this is all upside. There are real trade-offs:

**You're the on-call team.** If the server goes down at 3 AM, that's your problem. I use uptime monitoring (Uptime Kuma, self-hosted obviously) and keep my phone on loud. It hasn't happened often — maybe twice in a year — but it's on you.

**Scaling is manual.** If you suddenly get a flood of traffic, you can't just auto-scale. You need to either over-provision a bit or have a plan to spin up a bigger box. For most indie apps, this isn't an issue. If you're getting traffic floods, congratulations, you have a good problem.

**Security is your job.** You need to keep your OS patched, your firewall configured, your SSH keys rotated. It's not hard — `unattended-upgrades` handles most of it — but you need to actually set it up.

**Some things are genuinely better managed.** Transactional email? Use [Postmark](https://postmarkapp.com/) or [Resend](https://resend.com/). Don't run your own mail server. DNS? [Cloudflare](https://www.cloudflare.com/)'s free tier. There's a line between self-hosting and masochism.

## The Math That Matters

Here's my actual monthly spend running three profitable apps:

| Item | Cost |
|------|------|
| Hetzner CAX21 (main server) | $8.50 |
| Hetzner CAX11 (staging/side projects) | $3.90 |
| Backblaze B2 (backups, ~50GB) | $0.25 |
| Cloudflare (DNS, CDN) | $0.00 |
| Postmark (transactional email) | $15.00 |
| **Total** | **$27.65** |

These three apps gross about $4,800/month combined. My infrastructure cost is 0.6% of revenue. At my old job, infrastructure was a line item that made finance people cry.

## Why I'm Writing This

I'm not trying to convince you to quit your job. FAANG pays well. The problems can be interesting. If you're happy, stay.

But if you've got a side project, or an app idea, or you're just tired of over-engineering everything — consider that a $20 server might be all you need. The indie dev path is real. The tools have never been better. A single developer with a VPS and some battle-tested open source software can ship things that would've required a whole team ten years ago.

I'll be writing more about the specific tools and techniques I use. How to set up a VPS from scratch. Why Docker Compose is the only orchestrator you need. Why Caddy replaced [Nginx](https://nginx.org/) for me. Why I'm running SQLite in production (yes, really).

This is the stack I wish someone had shown me when I was still over-engineering everything at BigCo. Simple. Cheap. Yours.

Let's ship some stuff.

## Cheat Sheet

| Item | What / Why | Link |
|------|-----------|------|
| **Hetzner Cloud** | ARM VPS boxes, best price-to-performance for indie devs | [hetzner.com/cloud](https://www.hetzner.com/cloud) |
| **Vultr** | Good alternative VPS provider, more US/Asia locations | [vultr.com](https://www.vultr.com/) |
| **Docker Compose** | Single-file orchestration for app + db + cache + proxy | [docs.docker.com/compose](https://docs.docker.com/compose/) |
| **Caddy** | Reverse proxy with automatic HTTPS, zero config TLS | [caddyserver.com](https://caddyserver.com/) |
| **SQLite** | Embedded database, no separate process, great for single-server | [sqlite.org](https://sqlite.org/) |
| **Litestream** | Continuous SQLite replication to S3-compatible storage | [litestream.io](https://litestream.io/) |
| **Backblaze B2** | Cheap S3-compatible object storage ($0.005/GB) | [backblaze.com/b2](https://www.backblaze.com/b2/cloud-storage.html) |
| **Cloudflare** | Free DNS + CDN + DDoS protection | [cloudflare.com](https://www.cloudflare.com/) |
| **Postmark** | Transactional email that actually lands in inboxes | [postmarkapp.com](https://postmarkapp.com/) |
| **Uptime Kuma** | Self-hosted uptime monitoring with alerts | [github.com/louislam/uptime-kuma](https://github.com/louislam/uptime-kuma) |
| **unattended-upgrades** | Automatic security patches for Debian/Ubuntu | `apt install unattended-upgrades` |
| **Simple deploy script** | `ssh box "cd /app && git pull && docker compose up -d"` | — |
