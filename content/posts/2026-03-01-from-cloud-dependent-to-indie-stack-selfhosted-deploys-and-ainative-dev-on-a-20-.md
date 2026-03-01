---
title: "From Cloud-Dependent to Indie Stack: Self‑Hosted Deploys and AI‑Native Dev on a $20 VPS"
date: 2026-03-01T14:06:17.865Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# From Cloud‑Dependent to Indie Stack: Self‑Hosted Deploys and AI‑Native Dev on a $20 VPS

Here’s the thing — you don’t need Kubernetes to ship a small product. You need a VPS and a Makefile. A handful of containers. A couple of pings from other continents. And some AI helpers that actually sit next to your code instead of in a separate SaaS silo.

In this tutorial we’ll wire a lean, self‑hosted stack that’s friendly to a single low‑cost box. We’ll set up a private container registry, synthetic checks from multiple regions, simple container metrics, OTLP‑first observability, and a terminal workflow that lets AI tools work with your repo instead of fighting it. Then we’ll sketch a Rails stack that avoids cloud lock‑in with battle‑tested open source from the folks at 37signals.

Ship it.

---

## What we’re building (and why)

- Goals: Own the core pieces of your stack on one or two cheap VPSs; avoid sticky managed services; favor open source; keep deploys fast and obvious.
- Components:
  - [Harbor](https://goharbor.io/) for a self‑hosted container registry with RBAC, scanning, and signing. 37signals runs their registry on‑prem with Harbor as part of their pipeline, after external registries (Docker Hub/ECR) created cost and coupling pain during their cloud exit and “kamalization” journey ([source](https://dev.37signals.com)).
  - [Upright](https://dev.37signals.com) for synthetic monitoring — open source, runs probes from multiple geographic locations, exposes Prometheus metrics, and you can alert with [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and visualize in [Grafana](https://grafana.com/).
  - [DockWatch](https://hnrss.org) for a single‑container Docker health dashboard with anomaly detection and Telegram alerts. Built intentionally as a lightweight alternative to heavier stacks.
  - [OpenTelemetry](https://opentelemetry.io/) to standardize telemetry (metrics, traces, logs) via OTLP. The project is leaning into stability, reliability, and cleaner releases, including for traditional on‑prem setups with noisy logs and siloed tools ([stability focus](https://opentelemetry.io), [traditional envs](https://opentelemetry.io)).
  - A terminal‑first AI workflow inspired by [Frame](https://hnrss.org) that uses wrapper scripts, shared project context, and Git‑native provenance.

If you’re picking providers, tools like [CloudPriceCheck](https://cloudpricecheck.com/) help reality‑check that you’re staying in the $5–$20 VPS ballpark rather than drifting into “yak farm” territory. Good mental guardrail.

---

## Step 1 — Run your own Docker registry with Harbor on a single VPS

A private registry removes a hidden dependency from your deploy path. It also gives you image policies, scanning, and signing on your turf. [Harbor](https://goharbor.io/) is built for exactly this, with policies and role‑based access control, vulnerability scanning, and image signing/trust enforcement ([CNCF overview](https://cncf.io)). 37signals put it at the center of their deployments, on‑prem, and moved off external registries after cost and coupling bit hard ([their write‑up](https://dev.37signals.com)). As of early 2025, they deploy everything with [Kamal](https://kamal-deploy.org/) using Docker, so the registry is on the hot path ([source](https://dev.37signals.com)).

Here’s how I do it on a single host with a reverse proxy in front:

1) DNS and TLS
- Point registry.example.com at your box.
- Put [Caddy](https://caddyserver.com/) in front to terminate TLS. Caddy handles certs without drama.

Caddyfile (assume Harbor HTTP is on 8080 locally; adjust to your setup):
```
registry.example.com {
  reverse_proxy 127.0.0.1:8080
}
```

2) Push/pull from your laptop or CI
You don’t need anything fancy to wire your workflow to Harbor. Log in, tag, push. A tiny Makefile helps.

Makefile:
```
REGISTRY ?= registry.example.com
IMAGE    ?= app
TAG      ?= $(shell git rev-parse --short HEAD)

login:
	docker login $(REGISTRY)

build:
	docker build -t $(IMAGE):$(TAG) .

tag:
	docker tag $(IMAGE):$(TAG) $(REGISTRY)/$(IMAGE):$(TAG)

push: tag
	docker push $(REGISTRY)/$(IMAGE):$(TAG)

release: build push
	@echo "Pushed $(REGISTRY)/$(IMAGE):$(TAG)"
```

CI pipeline (GitHub Actions snippet):
```
- name: Login to registry
  run: echo "$REGISTRY_PASSWORD" | docker login registry.example.com -u "$REGISTRY_USERNAME" --password-stdin

- name: Build and push
  run: |
    make build
    make push
```

3) Using Kamal with your registry
If you deploy with [Kamal](https://kamal-deploy.org/), point it at your Harbor hostname and credentials. Kamal builds Docker images and ships them to your registry. Keep the registry close to the boxes that pull. Keep deploys snappy.

A quick aside. You’ll hear that “Kubernetes is AI’s operating system” — and the [v1.35 “Timbernetes” release](https://cncf.io) does read like a platform that runs services, batch, pipelines, even training. That’s great at 20,000 clusters. Not two VPSs. You can steal the playbook (artifact registry, OTLP, metrics, alerts) and glue it together with Docker, Kamal, and systemd. The DNA, minus the overhead. Works.

---

## Step 2 — Self‑hosted monitoring: Upright for uptime, DockWatch for containers

Uptime checks shouldn’t live inside the same blast radius as your app. They should call you from across the ocean when your app stops behaving. That’s Upright’s job.

- [Upright](https://dev.37signals.com) is an open‑source synthetic monitoring system used to watch Basecamp, HEY, Fizzy, and other services at 37signals. Each node runs health checks from multiple geographic locations and reports when something breaks ([source](https://dev.37signals.com)). It’s implemented as a Rails engine that you deploy to cheap VPS nodes using [Kamal](https://kamal-deploy.org/). Each node exposes Prometheus metrics, which enables alerting with [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and visualization in [Grafana](https://grafana.com/) ([source](https://dev.37signals.com)).

Minimal Prometheus scrape config to ingest Upright’s metrics (adapt host/port to your install):
```
scrape_configs:
  - job_name: 'upright'
    static_configs:
      - targets: ['upright-1.example.com:YOUR_PORT', 'upright-2.example.com:YOUR_PORT']
```

Alertmanager route that pages you when anything tagged `severity="critical"` fires:
```
route:
  receiver: 'pager'
  group_by: ['alertname']
  routes:
    - matchers:
        - severity="critical"
      receiver: 'pager'

receivers:
  - name: 'pager'
    email_configs:
      - to: you@example.com
        from: alerts@example.com
        smarthost: smtp.example.com:587
        auth_username: alerts@example.com
        auth_password: $SMTP_PASSWORD
```

Now for the host‑level view:

- [DockWatch](https://hnrss.org) was built because Prometheus+Grafana or Portainer felt like overkill for “just watching a few Docker containers.” It runs as a single container and provides a real‑time dashboard covering CPU, memory, network, disk, and temperature metrics. It includes six anomaly‑detection rules with Telegram alerts, supports self‑signed HTTPS or [Cloudflare Tunnel](https://www.cloudflare.com/products/tunnel/) for remote access, and ships a single HTML file dashboard (no build step) — all on a footprint of roughly 50MB with only four Python dependencies, built on [FastAPI](https://fastapi.tiangolo.com/), [aiodocker](https://github.com/aio-libs/aiodocker), [Chart.js](https://www.chartjs.org/), and [SQLite](https://www.sqlite.org/index.html) ([source](https://hnrss.org)).

Compose stub to run DockWatch (replace image and env variables per the project docs):
```
version: "3.8"
services:
  dockwatch:
    image: REPLACE_WITH_PROJECT_IMAGE
    restart: unless-stopped
    ports:
      - "9000:9000"
    # If the project requires the Docker socket, uncomment:
    # volumes:
    #   - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      # Set according to DockWatch docs if you want Telegram alerts and HTTPS
      # TELEGRAM_BOT_TOKEN: "..."
      # TELEGRAM_CHAT_ID: "..."
      # ENABLE_HTTPS: "true"
      # TLS_CERT_PATH: "/certs/cert.pem"
      # TLS_KEY_PATH: "/certs/key.pem"
    # volumes:
    #   - ./certs:/certs:ro
```

Why I like this pairing: Upright tells you “users in region X can’t log in.” DockWatch tells you “CPU just spiked and the disk is thrashing.” Two signals. No dashboard sprawl.

A side note on self‑hosting as a serious option: 37signals built [Action Push Native](https://dev.37signals.com) to move off Amazon SNS and Pinpoint for mobile push. It’s an open‑source Rails gem for sending push notifications to Apple and Google services, and it currently powers more than 10 million pushes per day for Basecamp and HEY ([source](https://dev.37signals.com)). That’s not a toy. It’s proof that you can own critical infra without a hyperscaler middleman.

---

## Step 3 — Observability without a vendor: OpenTelemetry Collector on Docker

I treat observability like plumbing: one standard pipe, many valves. That pipe is [OpenTelemetry](https://opentelemetry.io/). Use OTLP everywhere, avoid per‑vendor agents.

A minimal Collector config that accepts OTLP from your services, unrolls bundled logs, and prints to console (start here, add storage later):

otel-collector-config.yaml
```
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  batch: {}
  # Splits a JSON array payload (one record with multiple events) into one log record per event.
  unroll: {}

exporters:
  logging:
    loglevel: debug

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [logging]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [logging]
    logs:
      receivers: [otlp]
      processors: [unroll, batch]
      exporters: [logging]
```

Compose service:
```
version: "3.8"
services:
  otel-collector:
    image: otel/opentelemetry-collector:latest
    command: ["--config=/etc/otelcol/config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otelcol/config.yaml:ro
    ports:
      - "4317:4317" # OTLP gRPC
      - "4318:4318" # OTLP HTTP
    restart: unless-stopped
```

A few things to keep in mind:
- The OpenTelemetry project is deprecating the Zipkin exporter spec and steering users toward Zipkin’s OTLP ingestion due to low exporter usage and a strong preference for OTLP across languages ([deprecation note](https://opentelemetry.io)). So pick OTLP for new work.
- The Configuration Schema has a near‑final release candidate after years of work, with tooling to keep it consistent and less error‑prone ([schema RC](https://opentelemetry.io)). Handy if you’re hand‑editing YAML like above.
- The Unroll Processor exists for a reason: logs in traditional environments are often bundled or noisy. This processor turns one record containing a JSON array of multiple events into separate records, directly addressing multi‑event payloads ([unroll processor](https://opentelemetry.io)).
- OpenTelemetry eBPF Instrumentation (OBI) hit its first alpha release, originally donated from Grafana Beyla. Since then, it’s gained more protocol support and better behavior at scale ([OBI alpha](https://opentelemetry.io)). That makes auto‑capturing HTTP metrics/latencies without code changes feel doable even for small teams.
- If you’re worried about “will this scale with us?” — there’s a KubeCon EU talk about an org running observability for 20K+ clusters after rebuilding on OpenTelemetry. Different planet scale‑wise, but a good signal that OTel won’t be your bottleneck ([talk](https://opentelemetry.io)).

And yes, OTel is for old‑school environments too. The maintainers call out on‑prem data centers and legacy apps where logs are messy and monitoring is siloed — and argue OTel’s still a fit there ([traditional envs](https://opentelemetry.io)). Which is exactly our little box.

---

## Step 4 — Terminal‑first AI workflow: Frame‑style agents for your indie stack

AI tools work best when they live where your code lives. Windows and panes, not a separate tab farm.

- [Frame](https://hnrss.org) started as a terminal‑first lightweight IDE and is evolving into a platform for larger projects with AI agents. It gives you up to 9 terminals in a 3×3 grid, can host multiple AI tools (Claude Code, Codex CLI, Gemini CLI) in one window, and uses wrapper scripts for automatic context injection when integrating non‑native tools. It also promotes a standardized project structure with files like AGENTS.md, STRUCTURE.json, and PROJECT_NOTES.md ([source](https://hnrss.org)).

You can recreate most of that with [tmux](https://github.com/tmux/tmux/wiki) and a couple scripts.

tmux layout script (nine panes):
```
#!/usr/bin/env bash
tmux new-session -d -s dev 'nvim'
tmux split-window -h
tmux split-window -v
tmux select-pane -t 0
tmux split-window -v
tmux select-pane -t 3
tmux split-window -v
tmux select-pane -t 0
tmux select-layout tiled
tmux rename-window 'frame'
tmux attach -t dev
```

Standardize project context (drop these at repo root so every tool can read them):
- AGENTS.md — what agents exist, their roles, and boundaries.
- STRUCTURE.json — high‑level map of services and key files.
- PROJECT_NOTES.md — decisions, gotchas, runbooks.

Wrapper script for an AI CLI (inject repo context and recent changes automatically):
```
#!/usr/bin/env bash
set -euo pipefail

CONTEXT_FILE=.ai_context.txt
git status --porcelain > .ai_recent_changes.txt || true

cat > "$CONTEXT_FILE" <<'EOF'
# Project Context
- See STRUCTURE.json for the layout.
- See AGENTS.md for existing agents and constraints.
- See PROJECT_NOTES.md for decisions and gotchas.
EOF

# Combine context and user prompt, then call your AI CLI (replace with your tool)
jq -n --arg ctx "$(cat $CONTEXT_FILE)" --arg diff "$(cat .ai_recent_changes.txt)" --arg prompt "$*" \
  '{context:$ctx, recent_changes:$diff, prompt:$prompt}' > .ai_request.json

# Example: forward to a CLI like `claude-code` or `gemini` if you use them
# claude-code --json < .ai_request.json > .ai_response.json

echo "Request written to .ai_request.json; wire to your preferred CLI."
```

Git‑native provenance for AI decisions
- [RTS](https://github.com/nobutakayamauchi/RTS) proposes a Git‑native execution provenance protocol for AI decisions. Use it as a mental model: store model inputs/outputs alongside code so reviewers can audit what an agent did. Even a simple “.rts/” directory with request/response JSONs committed on a branch helps.

Turn PR comments into prompts
- The [PR Comment Prompter](https://hnrss.org) Chrome extension adds a “Copy Prompt” button to each GitHub PR comment and lets you customize the prompt template. Handy to funnel review threads into structured prompts for your AI CLI.

Agent email without giving away your inbox
- [TheAgentMail](https://hnrss.org) provides an email API for AI agents. With a single API call an agent can create a real address on the shared @theagentmail.net domain. It was designed to avoid security risks of giving agents access to personal inboxes and the overhead of buying/configuring domains, and it focuses on preventing spam via a karma‑based system with no manual review ([source](https://hnrss.org)).

Monitoring how models talk about your product
- [Geostorm.ai](https://geostorm.ai) monitors multiple AI models (for example, ChatGPT, Claude, Gemini via [OpenRouter](https://openrouter.ai/)) on a schedule to see what they say about your software or topics. You define search terms (like “best Python async framework”), it parses responses and tracks results over time ([source](https://geostorm.ai)). Nice “outside view” telemetry.

Architecting AI‑native tools
- [MCP Fusion](https://hnrss.org) frames AI‑native MCP servers with an MVA (Model‑View‑Agent) pattern and a deterministic View layer. Useful lens for anything that lets an agent touch real tools like `kamal deploy` or `docker tag`. Make the “View” deterministic and auditable.

Curious corners
- [Autolang](https://hnrss.org) is an embedded language/VM in C++ tuned for high‑frequency, short‑lived tasks like agent loops. Target ~2ms startup, arena‑restart memory model (~2x faster allocation than Lua), and can compile 100k classes in ~888ms ([source](https://hnrss.org)). If you’re experimenting with tight agent loops, it’s an interesting direction.
- [Zagora](https://hnrss.org) stitches mixed GPUs over standard 1Gbps internet into a fine‑tuning cluster using pipeline‑style parallelism (passing boundary activations, assigning layers across nodes). It avoids heavy tensor synchronization assumptions that break over heterogeneous fleets ([source](https://hnrss.org)). Not for your indie box today — but good to know it exists.

---

## Step 5 — Example indie Rails stack: multi‑tenant, mobile, and push without cloud lock‑in

Let me show you what actually works for a Rails product you can run yourself or host as SaaS.

Multi‑tenancy that isn’t a footgun
- 37signals experimented with giving every Fizzy customer their own SQLite database to support both self‑hosted and SaaS models and to push multi‑tenant design — then unwound months of work shortly before launch because the tradeoffs stacked up ([their story](https://dev.37signals.com)). The takeaway: separate databases can be great, but be intentional.
- Their [Active Record Tenanted](https://dev.37signals.com) gem was open‑sourced to make Rails multi‑tenancy with separate databases more seamless. In the demo they show converting an existing app to multi‑tenanted and discuss safeguards to prevent accidental data leaks across tenants.

Rich text that respects HTML
- [Lexxy](https://dev.37signals.com) is a new rich text editor for Rails Action Text, based on Meta’s [Lexical](https://lexical.dev/). It aims for good HTML semantics (real <p> tags for paragraphs), supports Markdown shortcuts and auto‑format on paste, real‑time code syntax highlighting, link creation by pasting URLs onto selected text, mention prompts with interactive filtering, and previewing attachments like PDFs and videos right in the editor ([source](https://dev.37signals.com)).

Mobile without a full native rewrite
- [Hotwire Native 1.2](https://dev.37signals.com) is described as a web‑first framework for building native mobile apps. Version 1.2 is the biggest update since launch, improves API consistency across platforms, ships new iOS/Android demo apps, and uses route decision handlers so internal URLs go to native screens and external URLs to the device browser ([source](https://dev.37signals.com)).

Push without SNS/Pinpoint
- [Action Push Native](https://dev.37signals.com) is an open‑source Rails gem for sending push notifications to Apple and Google platforms. It was created to migrate off Amazon SNS and Pinpoint as part of 37signals’ broader cloud exit, and they use it in Basecamp and HEY to send more than 10 million push notifications per day ([source](https://dev.37signals.com)). Own your push path, just like your registry.

Storage and observability mindset
- 37signals moved billions of files off S3 with no downtime, handling AWS bandwidth limits and constraints and building custom tooling when off‑the‑shelf options weren’t enough ([moving data post](https://dev.37signals.com)). They’re moving about 10 petabytes of data from S3 to Pure Storage FlashBlade; this data includes Basecamp attachments, long‑term Prometheus metrics, and filesystem use cases like database backups — and they treat observability of FlashBlade as a top priority because it underpins critical workloads ([Pure/observability post](https://dev.37signals.com)). Different scale, same habit: watch the things your business rides on.

Tie it all together: your Rails app ships images to Harbor, deploys with Kamal, gets synthetic checks from Upright nodes in other regions, chugs along under DockWatch on the host, reports telemetry through the OpenTelemetry Collector, and your terminal is the cockpit where AI tools see the same files and diffs you do. No black boxes.

Wild stuff.

---

## Quick Reference

- Harbor — Open‑source container registry with RBAC, scanning, and signing: https://goharbor.io/ (context: [CNCF](https://cncf.io), [37signals using Harbor](https://dev.37signals.com))
- Kamal — Deploy Dockerized apps to your own servers: https://kamal-deploy.org/ (mentioned by 37signals [here](https://dev.37signals.com))
- Upright — Open‑source synthetic monitoring (multi‑geo checks, Prometheus metrics, Alertmanager/Grafana): https://dev.37signals.com
- DockWatch — Lightweight Docker monitoring (single container, real‑time dashboard, anomaly rules, Telegram alerts): https://hnrss.org
- Prometheus — Metrics store and alerting (with Alertmanager): https://prometheus.io/
- Alertmanager — Routing/aggregation for alerts: https://prometheus.io/docs/alerting/latest/alertmanager/
- Grafana — Dashboards and visualizations: https://grafana.com/
- OpenTelemetry — Vendor‑neutral telemetry; use OTLP: https://opentelemetry.io/
- OTel eBPF (OBI) — Alpha auto‑instrumentation via eBPF (originated from Grafana Beyla): https://opentelemetry.io
- OTel Config Schema RC — Near‑final release candidate: https://opentelemetry.io
- OTel Unroll Processor — Split bundled log records into separate events: https://opentelemetry.io
- Zipkin exporter deprecation — Prefer OTLP: https://opentelemetry.io
- Traditional env guidance — OTel for on‑prem/legacy/noisy logs: https://opentelemetry.io
- KubeCon talk — OTel at 20K+ cluster scale: https://opentelemetry.io
- Frame — Terminal‑first AI dev platform (3×3 terminals, multi‑AI CLIs, wrapper scripts, standard files): https://hnrss.org
- PR Comment Prompter — Chrome extension for copy‑prompt buttons on GitHub PR comments: https://hnrss.org
- TheAgentMail — Email API for AI agents on @theagentmail.net (karma‑based spam prevention): https://hnrss.org
- Geostorm.ai — Monitor what AI chatbots say about your software over time via OpenRouter: https://geostorm.ai
- MCP Fusion — Framework for AI‑native MCP servers; MVA (Model‑View‑Agent) pattern: https://hnrss.org
- Autolang — Embedded C++ VM for high‑frequency, short‑lived tasks (target ~2ms startup): https://hnrss.org
- Zagora — Distributed fine‑tuning on mixed GPUs over standard internet (pipeline‑style parallelism): https://hnrss.org
- Active Record Tenanted — Rails multi‑tenancy with separate databases, data‑leak safeguards: https://dev.37signals.com
- Lexxy — Rich text editor for Rails Action Text, based on Lexical: https://dev.37signals.com
- Hotwire Native 1.2 — Web‑first native apps; route decision handlers; new demos: https://dev.37signals.com
- 37signals on Harbor — On‑prem registry to reduce costs/coupling from external registries: https://dev.37signals.com
- Moving off S3 — Billions of files, no downtime, custom tooling: https://dev.37signals.com
- Pure Storage monitoring — ~10 PB migration; top‑priority observability: https://dev.37signals.com
- CloudPriceCheck — Compare cloud pricing across eight providers: https://cloudpricecheck.com/

---

A few closing notes and next steps

- Start with Harbor + Caddy, one Upright node, DockWatch, and the OpenTelemetry Collector. Trivial footprint. Big payoff.
- Add a second Upright node in another region when you can. Cross‑checks save your bacon during regional blips.
- Use the Frame‑style terminal layout and wrapper scripts for a week. If you still bounce between tabs, I owe you a coffee.
- Rails‑wise, explore Active Record Tenanted if you need strong tenant isolation, use Lexxy for solid editing, and consider Hotwire Native for mobile without a rewrite. Replace SNS/Pinpoint with Action Push Native to own your push path from day one.
- When the single VPS starts to sweat, add a second. If you truly outgrow the boxes, then consider more orchestration. Not before.

I learned this the hard way. Keep it boring. Keep it yours. Then go build something users want.
