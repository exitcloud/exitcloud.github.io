---
title: "Ship Faster on a $10 VPS: Self‑Hosted AI Dev Workflows for Indie Hackers"
date: 2026-03-21T14:06:37.909Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# Ship Faster on a $10 VPS: Self‑Hosted AI Dev Workflows for Indie Hackers

“My entire ‘platform’ is a 43-line Makefile, a $7 VPS, and an AI agent running in tmux. It deploys faster than most teams’ Kubernetes clusters. Here’s every command.”

Here’s the thing — writing code isn’t the hard part anymore. The CNCF piece on API‑first infrastructure flat‑out says that AI‑assisted development has changed how we create and commit code, and that writing code is no longer the bottleneck. The bottleneck is everything that happens after git push — provisioning, policy, day‑two ops, drift, all the messy stuff ([source](https://www.cncf.io/blog/2026/03/20/crossplane-and-ai-the-case-for-api-first-infrastructure/)). So let’s fix that for a one‑box setup.

We’ll stand up a terminal‑first environment that treats AI as a co‑pilot not just for writing code, but for operating your stack. On one small VPS. No Kubernetes. Ship it.

- Shell as HQ with [Atuin v18.13](https://blog.atuin.sh/atuin-v18-13/) (better search, a PTY proxy, and AI for your shell), plus terminal‑native side tools like [fortransky](https://github.com/FormerLab/fortransky) (a terminal‑only Bluesky / AT Proto client written in Fortran — yes, that Fortran) and a project titled [Ghostling](https://github.com/ghostty-org/ghostling).
- An AI coding agent you can self‑host: [OpenCode](https://opencode.ai/) is described as an “Open source AI coding agent.” Its Hacker News submission drew 940 points and 447 comments, which tells you where everyone’s attention is ([discussion](https://news.ycombinator.com/item?id=47460525)).
- Minimal, pragmatic observability wired to modern guidance: [OpenTelemetry](https://opentelemetry.io/blog/2026/alternative-approaches-to-contributing/) provides the tools and standards to collect metrics, logs, and traces, and its team states that it’s deprecating the Span Event API — new code should write events as logs correlated with the current span ([details](https://opentelemetry.io/blog/2026/deprecating-span-events/)). We’ll follow that direction.

Let me show you what actually works.

## What We’re Actually Building on This Tiny Box

We’re aiming at a single small VPS (think 1–2 vCPU, 1–2GB RAM). Terminal‑first UX. A self‑hosted AI coding agent reachable from your shell. One or two containerized services. Logs that tell you what’s happening without needing a control plane.

Why now? Because, again, writing code is easy. The after‑push part is where solo devs drown. That CNCF article is blunt: the bottleneck is everything after git push — infrastructure, policies, day‑two ops, drift ([source](https://www.cncf.io/blog/2026/03/20/crossplane-and-ai-the-case-for-api-first-infrastructure/)). We’ll focus there.

Workflow sketch:

- SSH into your VPS.
- Live in the terminal. Use [Atuin v18.13](https://blog.atuin.sh/atuin-v18-13/) for better search/PTY proxy/AI in your shell. Keep social/notifications in the terminal too with [fortransky](https://github.com/FormerLab/fortransky).
- Run [OpenCode](https://opencode.ai/) locally as your coding agent. No wandering off to SaaS APIs.
- `git push`, then a Makefile target takes you from source to running container.
- Emit logs that fit the [OpenTelemetry](https://opentelemetry.io/blog/2026/deprecating-span-events/) direction: events as logs, correlated with traces. It scales up later; it works now.

Most writeups skip the copy‑paste‑able bits. We won’t.

## Step 1 – Bootstrap a Lean, Terminal‑First VPS

You don’t need Kubernetes. You need a VPS and a Makefile.

Start with a mainstream Linux distro like [Ubuntu](https://ubuntu.com/) or [Debian](https://www.debian.org/). Make your first login count.

Create a user and hand them sudo:

```bash
# As root on a fresh box
adduser dev
usermod -aG sudo dev
```

Install your SSH key and lock it down:

```bash
su - dev
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys   # paste your public key
chmod 600 ~/.ssh/authorized_keys
```

Harden SSH (disable password logins):

```bash
sudo nano /etc/ssh/sshd_config
# Change or ensure:
# PasswordAuthentication no
# PermitRootLogin no
sudo systemctl restart ssh
```

Enable a simple firewall with [UFW](https://ubuntu.com/server/docs/security-firewall):

```bash
sudo apt update
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

Install your terminal basics. I keep [tmux](https://github.com/tmux/tmux) on every box:

```bash
sudo apt install -y tmux git curl jq
```

Directory layout that won’t trip you later:

```bash
sudo mkdir -p /srv/{apps,configs,logs}
sudo chown -R dev:dev /srv
```

Persist logs with [systemd](https://systemd.io/)’s journal so reboots don’t wipe history:

```bash
sudo nano /etc/systemd/journald.conf
# Set or ensure:
# Storage=persistent
sudo systemctl restart systemd-journald
```

Resource: want to understand what your OS is actually doing with files, sockets, and processes? “Linux Applications Programming by Example: The Fundamental APIs (2nd Edition)” lives on GitHub ([repo](https://github.com/arnoldrobbins/LinuxByExample-2e)). Good backdrop while we keep moving.

Contrarian take: API‑first infrastructure for indie devs doesn’t require Crossplane or clusters. For a solo dev on 1–3 VPSs, “Makefile + SSH + systemd + a tiny HTTP API” is already API‑first. You can call it from anything, document it in 15 lines, and keep deploys under a minute. I learned this the hard way.

Optional: explore modern terminal tools like the project titled [Ghostling](https://github.com/ghostty-org/ghostling). Keeping everything in your terminal cuts context‑switching. You’ll feel it.

## Step 2 – Turn Your Shell into an AI Co‑Pilot (Atuin + fortransky)

Atuin’s latest post highlights “better search, a PTY proxy, and AI for your shell” ([announcement](https://blog.atuin.sh/atuin-v18-13/)). That’s exactly the shape you want in a terminal‑first workflow: fast command recall, smart help, and process wrangling.

- Install your usual shell stack (bash/zsh), then add Atuin to improve history search and get terminal‑native AI assistance.
- Configure Atuin per your needs; its v18.13 note calls out search improvements, a PTY proxy, and AI in the shell. Dial in what you’ll actually use.

Keep even your social notifications in the terminal so you don’t bounce out of flow. [fortransky](https://github.com/FormerLab/fortransky) is a terminal‑only Bluesky / AT Proto client written in Fortran. Yes, that Fortran. Fun example of how terminal tooling spans languages. Its Hacker News thread drew 105 points and 51 comments ([discussion](https://news.ycombinator.com/item?id=47461321)), which tells me plenty of folks prefer to stay in the terminal too.

Workflow tip:

- Use enhanced search to resurface deploy commands.
- Ask your shell AI to rewrite one safely (e.g., add `--no-cache`, change a tag).
- Paste, run, ship.

One more data point: open, self‑hosted coding agents are clearly where attention is going. [OpenCode](https://opencode.ai/) landed 940 points and 447 comments on Hacker News ([discussion](https://news.ycombinator.com/item?id=47460525)). And yet, the CNCF post reminds us the problem these days is everything after git push ([source](https://www.cncf.io/blog/2026/03/20/crossplane-and-ai-the-case-for-api-first-infrastructure/)). So let’s wire your AI into the post‑push flow too.

## Step 3 – Self‑Host an AI Coding Agent with OpenCode

OpenCode is described as an “Open source AI coding agent” ([site](https://opencode.ai/)). We’ll run an AI agent on the VPS and only expose it locally. Your terminal or editor can call it through an SSH tunnel. Simple. Own your stack.

Architecture:

- The agent listens on localhost at a port you choose.
- You connect via SSH with a local port forward.
- Optionally put a tiny reverse proxy in front, also bound to localhost.

SSH tunnel from your laptop:

```bash
# Local machine
ssh -N -L 9000:127.0.0.1:9000 dev@your-vps.example.com
# Now http://127.0.0.1:9000 on your laptop forwards to the agent on the VPS
```

A minimal reverse proxy with [Caddy](https://caddyserver.com/) bound to localhost:

```bash
# /srv/configs/Caddyfile
http://127.0.0.1:9000 {
  reverse_proxy 127.0.0.1:9001
}
```

Start it with systemd so it stays up:

```bash
# /etc/systemd/system/caddy-local.service
[Unit]
Description=Caddy (local reverse proxy)
After=network.target

[Service]
ExecStart=/usr/bin/caddy run --config /srv/configs/Caddyfile
User=dev
Group=dev
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now caddy-local
```

Test the agent with a curl (once it’s running):

```bash
curl -s http://127.0.0.1:9000/health
```

Point your shell tools at http://127.0.0.1:9000 via the SSH tunnel. Done. No public exposure.

Why log/metrics here? Because [OpenTelemetry](https://opentelemetry.io/blog/2026/alternative-approaches-to-contributing/) provides tools and standards for metrics, logs, and traces across services. And the OTel team explicitly says it’s deprecating the Span Event API, recommending that new code write events as logs correlated with the current span ([post](https://opentelemetry.io/blog/2026/deprecating-span-events/)). So we’ll structure our logs that way in the next step — and you’ll get meaningful, queryable output without pulling in a huge stack.

## Step 4 – From git push to Running Container with Pragmatic Observability

The “after git push” pipeline for indie devs should be boring:

- Build an image
- Restart the service
- Tail logs
- Roll back if needed

A Makefile you can run by muscle memory:

```makefile
# /srv/apps/Makefile
APP_NAME=helloapp
IMAGE=$(APP_NAME):latest
DOCKER=docker
DC=docker compose

.PHONY: build up logs deploy rollback

build:
\t$(DOCKER) build -t $(IMAGE) .

up:
\t$(DC) up -d

logs:
\t$(DC) logs -f --tail=100

deploy: build up logs

rollback:
\t$(DC) pull $(APP_NAME)@$$ROLLBACK_DIGEST
\t$(DC) up -d
```

A docker‑compose file that keeps things tight:

```yaml
# /srv/apps/compose.yml
services:
  helloapp:
    build: .
    image: helloapp:latest
    restart: unless-stopped
    environment:
      - NODE_ENV=production
    ports:
      - "8080:8080"
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

A tiny HTTP server that emits JSON logs with correlation fields. Tie this to the [OpenTelemetry](https://opentelemetry.io/blog/2026/deprecating-span-events/) guidance: events as logs that are correlated with the current span.

Example Node.js app:

```javascript
// /srv/apps/index.js
const http = require('http');

function log(obj) {
  process.stdout.write(JSON.stringify(obj) + '\n');
}

const server = http.createServer((req, res) => {
  const trace_id = (Math.random().toString(16).slice(2) + Date.now().toString(16));
  const start = Date.now();

  log({ level: "info", msg: "request_start", trace_id, method: req.method, path: req.url });

  if (req.url === "/health") {
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ ok: true }));
  } else {
    res.writeHead(200, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ hello: "world" }));
  }

  const dur_ms = Date.now() - start;
  log({ level: "info", msg: "request_end", trace_id, dur_ms });
});

server.listen(8080, () => {
  log({ level: "info", msg: "server_listen", port: 8080 });
});
```

Dockerfile:

```dockerfile
# /srv/apps/Dockerfile
FROM node:20-alpine
WORKDIR /app
COPY index.js package*.json ./
RUN npm pkg set name=helloapp && npm pkg set version=1.0.0
EXPOSE 8080
CMD ["node", "index.js"]
```

Deploy:

```bash
cd /srv/apps
make deploy
```

Hit it and watch correlated logs:

```bash
curl -s http://127.0.0.1:8080/health | jq
docker compose logs -f --tail=50
# Look for request_start and request_end with the same trace_id
```

This lines up with the OTel team’s direction: they state “OpenTelemetry is deprecating the Span Event API,” and “New code should write events as logs that are correlated with the current span” ([source](https://opentelemetry.io/blog/2026/deprecating-span-events/)). We aren’t pulling in an SDK here — we’re shaping logs so that if you add tracing later, the correlation fields are already present. The older “span events” style will be phased out over time, but existing data and views that show events on spans will keep working ([post](https://opentelemetry.io/blog/2026/deprecating-span-events/)).

Why bother with metrics/logs on one node? The CNCF piece on Kubernetes metrics says “Kubernetes metrics show cluster activity,” that you need them to manage clusters, nodes, and applications, and without them it’s harder to find problems and improve performance ([source](https://www.cncf.io/blog/2026/03/18/understanding-kubernetes-metrics-best-practices-for-effective-monitoring/)). The same principle applies at small scale. Different substrate, same need to see what’s happening.

Debugging workflow:

- Trigger a deploy with `make deploy`
- `curl` your endpoints
- `docker compose logs -f` and `journalctl -u caddy-local -f`
- Look for a single `trace_id` across events. Pinpoint slow paths. Roll back fast if needed.

Minimal. Repeatable. No ceremony.

## Step 5 – Stretch Goals: WASM, Media Pipelines, and Language Trade‑Offs

Language debates get loud. Reality check: your tiny box has constraints. There’s a story where a team said, “We rewrote our Rust WASM parser in TypeScript and it got faster” ([post](https://www.openui.com/blog/rust-wasm-parser)). That submission drew 242 points and 152 comments on Hacker News ([discussion](https://news.ycombinator.com/item?id=47461094)). The point isn’t that one language is universally “better.” It’s that sometimes a different toolchain fits the workload and your operational shape better — especially when CPU and RAM are tight.

Want to experiment with media? The “FFmpeg 101 (2024)” post is a solid primer ([link](https://blogs.igalia.com/llepage/ffmpeg-101/)). You can add a small job runner around ffmpeg and reuse the same log pattern.

Script it:

```bash
# /srv/apps/transcode.sh
#!/usr/bin/env bash
set -euo pipefail
IN="$1"
OUT="$2"

TRACE_ID="$(openssl rand -hex 8)-$(date +%s)"
echo "{\"level\":\"info\",\"msg\":\"transcode_start\",\"trace_id\":\"$TRACE_ID\",\"in\":\"$IN\",\"out\":\"$OUT\"}"

ffmpeg -y -i "$IN" -c:v libx264 -c:a aac "$OUT"

echo "{\"level\":\"info\",\"msg\":\"transcode_done\",\"trace_id\":\"$TRACE_ID\"}"
```

Run with the same Makefile/log tailing workflow. You’ll see start/end with a shared trace_id. Same mental model.

Linux fundamentals help you understand what’s happening under the hood — processes spawning ffmpeg, file descriptors, exit codes. That’s where “Linux Applications Programming by Example: The Fundamental APIs (2nd Edition)” comes in as a reference ([repo](https://github.com/arnoldrobbins/LinuxByExample-2e)).

Mixing languages? Totally fine. We already mentioned [fortransky](https://github.com/FormerLab/fortransky) (terminal‑only Bluesky client written in Fortran). Want to tinker with model‑adjacent ideas later? There’s a repo titled “Attention Residuals” ([link](https://github.com/MoonshotAI/Attention-Residuals); HN [discussion](https://news.ycombinator.com/item?id=47458595)). The key is to standardize your logs and basic process control so the stack doesn’t care which language is underneath. Consistency beats purity.

Community angle: if you want to grow beyond a single node, there’s plenty happening. “KubeCon + CloudNativeCon Europe 2026 is coming to Amsterdam,” with Merge Forward bringing community, conversation, and a lineup of sessions and events ([post](https://www.cncf.io/blog/2026/03/20/your-merge-forward-guide-to-kubecon-cloudnativecon-amsterdam/)). Another CNCF note says the event will bring the cloud native community together with projects, vendors and end users and describes it as one of the largest open source events ([post](https://www.cncf.io/blog/2026/03/20/kubecon-cloudnativecon-eu-2026-end-user-tab-recommendations-and-tips)). If you’re into networking, there’s even a deep‑dive on CiliumCon returning to Amsterdam, the seventh time hosting it, coming on the heels of Cilium’s tenth anniversary ([post](https://www.cncf.io/blog/2026/03/18/kubecon-cloudnativecon-europe-2026-co-located-event-deep-dive-ciliumcon)). Big tent. Start small, join when you’re ready.

Side note for folks who care about the web’s memory: the EFF’s “Blocking Internet Archive Won’t Stop AI, but Will Erase Web’s Historical Record” is worth a read ([link](https://www.eff.org/deeplinks/2026/03/blocking-internet-archive-wont-stop-ai-it-will-erase-webs-historical-record)). Owning your stack matters. So does the web’s history. Different layers; connected values.

## Quick Reference

- [OpenCode](https://opencode.ai/) — Open source AI coding agent; HN: 940 points / 447 comments ([discussion](https://news.ycombinator.com/item?id=47460525)).
- [Atuin v18.13](https://blog.atuin.sh/atuin-v18-13/) — Shell history with better search, a PTY proxy, and AI for your shell; HN: 56 points / 38 comments ([discussion](https://news.ycombinator.com/item?id=47465824)).
- [fortransky](https://github.com/FormerLab/fortransky) — Terminal‑only Bluesky / AT Proto client written in Fortran; HN: 105 points / 51 comments ([discussion](https://news.ycombinator.com/item?id=47461321)).
- [Ghostling](https://github.com/ghostty-org/ghostling) — Project titled “Ghostling”; HN: 245 points / 46 comments ([discussion](https://news.ycombinator.com/item?id=47461378)).
- [OpenTelemetry: tools and standards](https://opentelemetry.io/blog/2026/alternative-approaches-to-contributing/) — Collect metrics, logs, and traces; notes on contribution context.
- [OTel: Deprecating Span Events API](https://opentelemetry.io/blog/2026/deprecating-span-events/) — Deprecating Span Event API; new code should write events as logs correlated with the current span; older style phased out over time.
- [CNCF: API‑first infra and AI](https://www.cncf.io/blog/2026/03/20/crossplane-and-ai-the-case-for-api-first-infrastructure/) — AI‑assisted dev changed how code is created; writing code isn’t the bottleneck; bottleneck is after git push.
- [CNCF: Kubernetes metrics](https://www.cncf.io/blog/2026/03/18/understanding-kubernetes-metrics-best-practices-for-effective-monitoring/) — Kubernetes metrics show cluster activity; you need them to manage clusters, nodes, and apps.
- [LinuxByExample‑2e](https://github.com/arnoldrobbins/LinuxByExample-2e) — Linux APIs reference; HN: 122 points / 15 comments ([discussion](https://news.ycombinator.com/item?id=47462483)).
- [FFmpeg 101 (2024)](https://blogs.igalia.com/llepage/ffmpeg-101/) — FFmpeg fundamentals; HN: 129 points / 3 comments ([discussion](https://news.ycombinator.com/item?id=47463547)).
- [OpenUI WASM parser post](https://www.openui.com/blog/rust-wasm-parser) — “We rewrote our Rust WASM parser in TypeScript and it got faster”; HN: 242 points / 152 comments ([discussion](https://news.ycombinator.com/item?id=47461094)).
- [EFF: Internet Archive and AI](https://www.eff.org/deeplinks/2026/03/blocking-internet-archive-wont-stop-ai-it-will-erase-webs-historical-record) — On preserving the web’s historical record.
- [KubeCon + CloudNativeCon EU 2026 guide](https://www.cncf.io/blog/2026/03/20/your-merge-forward-guide-to-kubecon-cloudnativecon-amsterdam/) — Merge Forward guide for Amsterdam.
- [KubeCon EU 2026 recommendations and tips](https://www.cncf.io/blog/2026/03/20/kubecon-cloudnativecon-eu-2026-end-user-tab-recommendations-and-tips/) — Event overview and tips.
- [CiliumCon deep dive](https://www.cncf.io/blog/2026/03/18/kubecon-cloudnativecon-europe-2026-co-located-event-deep-dive-ciliumcon) — CiliumCon returns to Amsterdam; seventh time hosting; near Cilium’s tenth anniversary.

## Wrap

Picture the final loop. You SSH into a cheap VPS. Land in a terminal‑driven environment. [Atuin v18.13](https://blog.atuin.sh/atuin-v18-13/) helps you recall that deploy incantation, its AI gives you a safer variant. Your self‑hosted agent — [OpenCode](https://opencode.ai/) — is a local hop away via SSH. You push code, run a Makefile target, and watch JSON logs show request_start and request_end with the same trace_id.

No Rube Goldberg machines. No ceremony. And the observability lines up with where [OpenTelemetry](https://opentelemetry.io/blog/2026/deprecating-span-events/) is heading: events as logs, correlated with spans. The same mental model that folks bring to Kubernetes — the CNCF post reminds us those metrics show cluster activity and are needed to manage clusters and apps ([source](https://www.cncf.io/blog/2026/03/18/understanding-kubernetes-metrics-best-practices-for-effective-monitoring/)) — just scaled down to a single node.

From here, keep iterating. If you want a bigger tent, there’s a world of community at events like KubeCon + CloudNativeCon Europe 2026 in Amsterdam ([guide](https://www.cncf.io/blog/2026/03/20/your-merge-forward-guide-to-kubecon-cloudnativecon-amsterdam/), [tips](https://www.cncf.io/blog/2026/03/20/kubecon-cloudnativecon-eu-2026-end-user-tab-recommendations-and-tips/)). But you don’t need permission to ship. A VPS, a Makefile, and an AI sidekick. That’s enough to run something real. Ship it.
