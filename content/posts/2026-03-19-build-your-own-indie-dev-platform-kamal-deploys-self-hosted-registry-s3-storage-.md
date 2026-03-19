---
title: "Build Your Own Indie Dev Platform: Kamal Deploys, Self-Hosted Registry, S3 Storage, Monitoring & Local AI on a $20 Server"
date: 2026-03-19T14:08:52.360Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# Build Your Own Indie Dev Platform: Kamal Deploys, Self‑Hosted Registry, S3 Storage, Monitoring & Local AI on a $20 Server

I shut off Docker Hub for a week and my deploys got faster. Here’s the on‑prem registry + Makefile that replaced it on a tiny VPS. No yak shaving. Just ship.

Here’s the thing — you don’t need Kubernetes for this. [37signals deploys everything with Kamal using Docker](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/). A bunch of homelabbers run 40+ containers on a single NUC or mini PC without drama. One thread literally asks if Kubernetes on a small home server is anything more than a learning exercise. For most indie apps? A VPS and a Makefile.

What we’re building:

- App containers deployed with [Kamal](https://kamal-deploy.org/) and [Docker](https://www.docker.com/).
- A self‑hosted registry using [Harbor](https://goharbor.io/).
- S3‑compatible object storage on RustFS.
- Synthetic monitoring with [Upright](https://dev.37signals.com/introducing-upright/), metrics via [Prometheus](https://prometheus.io/), alerts via [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/), and dashboards with [Grafana](https://grafana.com/).
- A local AI workbench using Unsloth Studio, [Ollama](https://ollama.com/), and [Cook](https://rjcorwin.github.io/cook/) to push code faster.

Reality check: a single mini PC with SATA bays can carry an entire media stack plus extras; one user flat‑out said it’s all you need for Plex. Others run dense setups in tiny spaces, like a wall‑mounted NUC with 40+ Docker containers, or a Beelink mini PC + Pi Zero for backup DNS. This isn’t hypothetical. It’s everyday.

## 1. Design the tiny platform: why Kamal + Docker, not Kubernetes

A reasonable target: run a Rails or Node app, a worker, a database, and a couple of support services on a single VPS or mini PC — with the option to add a second cheap node for monitoring. Low ceremony. High signal. If a deploy takes more than 60 seconds, something’s wrong.

- [Kamal](https://kamal-deploy.org/) gives you push‑button, zero‑to‑deploy with Docker. No cluster to babysit.
- [37signals](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/) uses Kamal across their apps with Docker as the containerization platform.
- The homelab crowd keeps proving that small boxes can run serious stacks without Kubernetes: dense NUC builds, Beelink mini PCs, and more.

Want to learn K8s? Great — study this [CNCF internals guide on when pods restart](https://www.cncf.io/blog/2026/03/17/when-kubernetes-restarts-your-pod-and-when-it-doesnt/) and the companion repo. But to ship your product this weekend? Kamal + Docker.

High‑level layout:

- app.example.com → reverse proxy → app containers via Kamal
- registry.example.com → [Harbor](https://goharbor.io/)
- s3.internal → RustFS
- mon.example.com → [Upright](https://dev.37signals.com/introducing-upright/) nodes → [Prometheus](https://prometheus.io/) + [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) + [Grafana](https://grafana.com/)

Ship it.

## 2. Ship code with confidence: Kamal + Harbor as your self‑hosted registry

[37signals moved off Docker Hub/ECR and treats Harbor as a core piece of their pipeline](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/). They called out costs and performance impacts of pushing/pulling over the public internet and solved it by running their own registry. Same move here.

Minimal pieces you need:

- A Dockerfile for your app.
- A `kamal.yml` that points to your Harbor registry.
- A Makefile that wraps deploy commands so you don’t fat‑finger production.

Example app Dockerfile (Ruby on Rails; adapt the idea for Node/Python):

```dockerfile
# Dockerfile
FROM ruby:3.3-alpine AS base
RUN apk add --no-cache build-base postgresql-client postgresql-dev nodejs yarn tzdata
WORKDIR /app
COPY Gemfile Gemfile.lock ./
RUN bundle install --jobs 4
COPY . .
ENV RAILS_ENV=production
RUN RAILS_ENV=production SECRET_KEY_BASE=dummy bundle exec rails assets:precompile

FROM ruby:3.3-alpine
RUN apk add --no-cache postgresql-client tzdata
WORKDIR /app
COPY --from=base /usr/local/bundle /usr/local/bundle
COPY --from=base /app /app
ENV RAILS_ENV=production
CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

A dead‑simple `kamal.yml` that pushes to Harbor:

```yaml
# kamal.yml
service: myapp
image: registry.example.com/myorg/myapp
servers:
  web:
    hosts:
      - app@10.0.0.10
env:
  secret:
    RAILS_MASTER_KEY: <%= ENV["RAILS_MASTER_KEY"] %>
  clear:
    RAILS_LOG_TO_STDOUT: "1"
builder:
  local: true
registry:
  server: registry.example.com
  username: <%= ENV["REGISTRY_USER"] %>
  password: <%= ENV["REGISTRY_PASS"] %>
```

Makefile so deployments are muscle memory:

```make
# Makefile
deploy:
	kamal deploy

rollback:
	kamal rollback

logs:
	kamal app logs -f

setup:
	kamal setup
```

Now the registry side. [Harbor](https://goharbor.io/) is the target. The operational reason to self‑host is covered by 37signals: they used Docker Hub and Amazon ECR for a long time, hit cost issues, and were tightly coupled to those services until they switched to Harbor and made the registry an internal dependency that they control end‑to‑end ([source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/)). Treat yours the same way.

Basic workflow once Harbor is reachable at `registry.example.com`:

```bash
# 1) Login from your dev box or CI runner
docker login registry.example.com

# 2) Build and push with Kamal
export REGISTRY_USER=admin
export REGISTRY_PASS=*** # from Harbor
export RAILS_MASTER_KEY=*** # your app

kamal deploy
```

You’ll see Kamal build locally, push to `registry.example.com`, and then your server will pull from there. Fast. Local network fast if you run the registry near your node.

Storage: point Harbor at S3‑compatible storage. We’ll stand up RustFS in the next section and wire Harbor to it so image blobs land in your object store, not on the root filesystem.

A quick editorial aside — start small with observability. [Upright](https://dev.37signals.com/introducing-upright/) style synthetic checks plus [Prometheus](https://prometheus.io/) metrics will catch most footguns. The [OpenTelemetry](https://opentelemetry.io/) team heard loud and clear that folks want practical examples from production, which is why they kicked off a case study series beginning with a Mastodon deployment they called “small but uniquely challenging” ([source](https://opentelemetry.io/blog/2026/devex-mastodon/)). Don’t spray traces everywhere before you know what you’ll do with them.

## 3. Own your data: swap MinIO for RustFS and plug it into your stack

You need S3‑compatible object storage for uploads, backups, and as a backend for your registry. Many of us started on MinIO. But the RustFS team highlights that MinIO’s open‑source repo has been archived, and a lot of people are stuck because migration sounds scary.

RustFS’s pitch: a “binary replacement” path that avoids complex re‑indexing. The idea is straightforward in spirit — stop MinIO, drop in RustFS with compatible config, keep the same data directory, start RustFS, verify access (source). Fewer moving parts during the cutover. Minimal risk. That’s the claim.

A pragmatic flow to get RustFS into your platform:

- Stand up RustFS on `s3.internal` (behind your reverse proxy).
- Create an access key and secret.
- Verify with a quick S3 list.

Commands you’ll actually run (using the AWS CLI as a sanity check):

```bash
# Replace with your endpoint and creds
export AWS_ACCESS_KEY_ID=RUSTFS_ACCESS_KEY
export AWS_SECRET_ACCESS_KEY=RUSTFS_SECRET_KEY
aws --endpoint-url https://s3.internal s3 ls
```

Wiring your stack to S3 is the same everywhere:

- S3 endpoint URL (https://s3.internal)
- Access key ID
- Secret access key
- Region (arbitrary for many S3‑compatible systems)

Update Harbor’s object storage config to point at RustFS. Point your app’s file uploads there too. You can feed big offline archives to it as well — Open Archiver specifically calls out direct ingestion from local filesystem paths and targeting S3‑compatible storage, which is exactly the kind of workload an on‑prem object store should handle.

Owning this layer matters. [37signals is moving 10 petabytes of data from AWS S3 to Pure Storage FlashBlade](https://dev.37signals.com/pure-storage-monitoring/). Those buckets hold Basecamp attachments and their long‑term Prometheus metrics. They also use the filesystem capabilities for database backups and treat observability of that system as a top priority because it underpins multiple critical data use cases ([source](https://dev.37signals.com/pure-storage-monitoring/)). Same energy here, just scaled to indie size.

One more nudge from the real world: among all the links in this set, the one with obvious engagement is [Cook](https://rjcorwin.github.io/cook/). Devs want AI tools that help them ship. Heavy infra posts? Less attention. So we’ll integrate AI help directly into deploy/observability tasks later. It cuts through.

## 4. Watch everything: Upright synthetic monitoring + Prometheus + OpenTelemetry

Start with black‑box checks that hit your app like a user would. [Upright](https://dev.37signals.com/introducing-upright/) is a Rails engine 37signals deploys to cheap VPS nodes using [Kamal](https://kamal-deploy.org/). Each node runs probes from multiple geos and exposes [Prometheus](https://prometheus.io/) metrics. Those feed into [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) for paging and [Grafana](https://grafana.com/) for dashboards. Clean and simple.

Flow:

1) Create a small Rails app that hosts Upright. Add a basic HTTP probe to your main app. Deploy it with Kamal to a low‑cost node in another region. Now you’ve got external eyes.

2) Stand up Prometheus + Alertmanager + Grafana next to your app (or on that same monitoring node) and scrape Upright’s metrics endpoint.

Here’s a minimal docker‑compose for Prometheus + Alertmanager + Grafana:

```yaml
# docker-compose.monitoring.yml
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    command:
      - --config.file=/etc/prometheus/prometheus.yml
    ports: ["9090:9090"]
  alertmanager:
    image: prom/alertmanager:latest
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    command:
      - --config.file=/etc/alertmanager/alertmanager.yml
    ports: ["9093:9093"]
  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
```

And a tiny Prometheus config that scrapes your app and Upright:

```yaml
# prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'app'
    static_configs:
      - targets: ['app.example.com:443']
    scheme: https

  - job_name: 'upright'
    static_configs:
      - targets: ['mon.example.com:443']
    scheme: https
    metrics_path: /metrics
```

OpenTelemetry fits in next. The project’s declarative configuration pieces are now stable — JSON schema for the config model, a YAML representation, an in‑memory representation, ConfigProperties, PluginComponentProvider, and SDK operations for parsing YAML and instantiating components ([source](https://opentelemetry.io/blog/2026/stable-declarative-config/)). Translation: it’s getting easier to define pipelines as data. Also, Zipkin exporters are deprecated in favor of Zipkin’s OTLP ingestion ([source](https://opentelemetry.io/blog/2025/deprecating-zipkin-exporters/)). So standardize on OTLP and keep your options open.

A starter OpenTelemetry Collector config that receives OTLP and logs it (swap in your favorite backend later):

```yaml
# otel-collector.yaml
receivers:
  otlp:
    protocols:
      http:
      grpc:

exporters:
  logging:
    loglevel: info

processors:
  batch: {}

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
```

Run the Collector next to Prometheus or your app, send OTLP there, and decide where to fan out later (Tempo, Loki, Zipkin via OTLP, etc.). If you do migrate to Kubernetes someday, OpenTelemetry’s Kubernetes semantic conventions for resource attributes have been promoted to release candidate status ([source](https://opentelemetry.io/blog/2026/k8s-semconv-rc/)). Borrow the spirit of those conventions (consistent resource labels) even off‑cluster so you don’t have to rename everything later. Future‑proof enough.

And yes — keep Upright probing from more than one geo when you can. [Upright](https://dev.37signals.com/introducing-upright/) was designed for that setup: multiple nodes, one brain, Prometheus metrics out. Cheap VPSes make it easy.

## 5. Add an AI co‑pilot: local LLMs, Cook + Claude, and diagram‑driven dev

AI coding tools are force multipliers, not replacements. You still need to understand what the code does. But they’ll shave hours off drudge work.

- Unsloth Studio runs 100% offline on macOS, Windows, and Linux. It can download open models like Gemma, Qwen, and Llama, fine‑tune them, and transform PDF/TXT/DOCX files into datasets. Perfect for an offline “AI workbench.”
- [Ollama](https://ollama.com/) is a quick way to pull and run models locally. One homelabber tried to “make these machines crying with Ollama self‑hosted” on laptops — classic tinkering energy (source).
- [Cook](https://rjcorwin.github.io/cook/) is a simple CLI for orchestrating Claude Code. It’s the glue to turn “write me a Kamal task” into an actual file in your repo.

A little workflow that actually pays off:

1) Feed your repo to the model. Include `kamal.yml`, your Dockerfile, and monitoring configs.

2) Ask it to write a new synthetic check or Kamal hook. With Cook, that’s as simple as:

```bash
# Generate a Kamal health check task with Cook + Claude Code
cook ask "Add a 'kamal health' task that curls /up on my web hosts and returns nonzero on failure. Write the Rake task and update Makefile. Keep it POSIX-friendly." --dir .
```

3) Review changes. Run the task. Tweak.

4) Generate an architecture diagram and check it into docs. This isn’t theory; a homelabber showed a full setup where Claude generated their architecture diagram. It’s a nice pattern: living docs, refreshed by a single prompt.

A quick note on community norms: r/selfhosted allows AI posts as long as you follow the rules. Moderators previously tried a themed Friday for “vibe‑coded” posts and found that policing AI involvement was confusing, so they simplified the rules (source). The takeaway: ship durable, tested configs. Include real commands. Not vibes.

## 6. Hardening and real‑world fit: reverse proxies, TLS, and homelab patterns

Expose as little as possible. Put everything behind a real reverse proxy like [Caddy](https://caddyserver.com/) or [Traefik](https://traefik.io/). Use DNS‑based certificate challenges so you can close port 80 on your router and keep only 443 open.

Why the fuss? One homelabber running behind a reverse proxy switched to DNS challenges specifically to shut port 80, then noticed a flood of unsolicited connections to 443 — even on a small setup (source). The internet will knock. All day. Don’t run naked.

A tiny [Caddy](https://caddyserver.com/) example that terminates TLS and forwards to your services (swap in your DNS provider plugin if you automate DNS challenges):

```caddy
# Caddyfile
app.example.com {
  reverse_proxy 127.0.0.1:3000
}

registry.example.com {
  reverse_proxy 127.0.0.1:5000
}

mon.example.com {
  reverse_proxy 127.0.0.1:9090
}
```

Two deployment patterns that map to real homelabs:

- Single‑box: a mini PC with SATA bays runs everything — registry, RustFS, app, monitoring. This lines up with the “mini PC over Pi + USB” argument from the Plex thread (source). Fewer flaky cables. Enough RAM for lots of containers.
- Split duty: main host + tiny helper. The Beelink + Pi Zero DNS pattern translates nicely to “main app box + small monitoring node.”

Got spare cycles? Stack extra projects on the same foundation:

- Media: [Jellyfin](https://jellyfin.org/) + “Tunarr” style channels; one user is doing it to mimic old‑school channel surfing for a parent. You can even track watch history with Yamtrack.
- Docs: an OIDC‑aware doc site that fills in usernames for low‑touch onboarding is a real need that someone asked for (source). Consider building it on Rails and [Action Text](https://rubyonrails.org/) with [Lexxy](https://dev.37signals.com/announcing-lexxy-a-new-rich-text-editor-for-rails/) as the editor.
- Mobile: try [Hotwire Native v1.2](https://dev.37signals.com/announcing-hotwire-native-v1-2/) for a web‑first mobile app with new route decision handlers and demo apps.
- Notifications: self‑host push with [Action Push Native](https://dev.37signals.com/introducing-action-push-native/) (Apple and Google) instead of a cloud broker.
- Multi‑tenancy: experiment with SQLite‑per‑customer using [Active Record Tenanted](https://dev.37signals.com/rails-multi-tenancy/). 37signals explored that pattern for Fizzy and decided to unwind it before release due to tradeoffs, but the gem’s safeguards demo is a good foundation (sources). Tradeoffs exist. But it’s a fun lever to pull on small apps.

TLS side rant: a thread asked for a “simple chat room without TLS” because they didn’t care about encryption and wanted to rely on firewall rules (source). Don’t do that. Certs are table stakes now and don’t require touching app code when you terminate at the proxy.

Wild stuff. But it works.

## Quick Reference

- Kamal — zero‑to‑deploy containers with SSH and Docker: https://kamal-deploy.org/
- Docker — container runtime used by Kamal: https://www.docker.com/
- Harbor — self‑hosted Docker registry; 37signals runs theirs on‑prem as part of their pipeline: https://goharbor.io/ and context: https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/
- RustFS — S3‑compatible storage with “binary replacement” migration from MinIO: https://www.reddit.com/r/selfhosted/comments/1rxmog1/show_rselfhosted_how_to_migrate_from_minio_to/
- Upright — synthetic monitoring Rails engine; probes from multiple geos; Prometheus metrics; deployed via Kamal: https://dev.37signals.com/introducing-upright/
- Prometheus — metrics scraping/TSDB: https://prometheus.io/
- Alertmanager — alert routing for Prometheus: https://prometheus.io/docs/alerting/latest/alertmanager/
- Grafana — dashboards: https://grafana.com/
- OpenTelemetry — vendor‑neutral telemetry; stable declarative config; prefer OTLP over Zipkin exporters: https://opentelemetry.io/blog/2026/stable-declarative-config/, https://opentelemetry.io/blog/2025/deprecating-zipkin-exporters/
- CNCF K8s pod restarts internals — if you want to study: https://www.cncf.io/blog/2026/03/17/when-kubernetes-restarts-your-pod-and-when-it-doesnt/
- Unsloth Studio — offline local LLM UI; supports Gemma/Qwen/Llama; builds datasets from docs: https://www.reddit.com/r/selfhosted/comments/1rx42qt/introducing_unsloth_studio_an_opensource_web_ui/
- Ollama — run models locally: https://ollama.com/
- Cook — CLI to orchestrate Claude Code from the terminal: https://rjcorwin.github.io/cook/
- Caddy — reverse proxy and TLS terminator: https://caddyserver.com/
- Traefik — reverse proxy/load balancer: https://traefik.io/
- Jellyfin — self‑hosted media server: https://jellyfin.org/
- Tunarr — create TV‑like channels (discussion): https://www.reddit.com/r/selfhosted/comments/1rxjnao/is_tunarr_a_good_option_for_my_father_with/
- Yamtrack — self‑hosted Trakt alternative (discussion): https://www.reddit.com/r/selfhosted/comments/1rxwi5f/rip_trakt_new_to_yamtrack/
- Lexxy — Action Text editor for Rails: https://dev.37signals.com/announcing-lexxy-a-new-rich-text-editor-for-rails/
- Hotwire Native v1.2 — web‑first native mobile framework; new route decision handlers and demos: https://dev.37signals.com/announcing-hotwire-native-v1-2/
- Action Push Native — Rails gem for Apple/Google push notifications: https://dev.37signals.com/introducing-action-push-native/
- r/selfhosted AI post policy — AI posts allowed if they follow rules: https://www.reddit.com/r/selfhosted/comments/1m67csm/summer_update_2025_ai_flair_and_mods/
- r/selfhosted rules update — simplified approach after prior AI policing confusion: https://www.reddit.com/r/selfhosted/comments/1rmt39o/rules_update_new_project_friday_here_to_stay/
- Port scan reality check — reverse proxy with DNS challenge; lots of unsolicited 443 hits: https://www.reddit.com/r/homelab/comments/1rxkj5g/is_this_amount_of_incoming_connections_to_port/
- S3 migration at scale — 37signals moving 10 PB off S3 to Pure Storage FlashBlade; monitoring is top priority: https://dev.37signals.com/pure-storage-monitoring/

## Wrap‑up

By now you’ve got the bones of a modern indie platform: Kamal pushing your app into Harbor, Harbor writing to an S3‑compatible backend like RustFS, Upright probes reporting into Prometheus/Alertmanager/Grafana, and a local AI bench that helps you wire new bits without getting lost. No sprawling clusters. No mystery bills. Just configs you can read in one sitting and rebuild from scratch.

Next steps? Try a media stack. Add OIDC‑aware docs with a nice editor like [Lexxy](https://dev.37signals.com/announcing-lexxy-a-new-rich-text-editor-for-rails/). Ship a mobile shell with [Hotwire Native v1.2](https://dev.37signals.com/announcing-hotwire-native-v1-2/). Or go weird with SQLite multi‑tenancy using [Active Record Tenanted](https://dev.37signals.com/rails-multi-tenancy/). It’s your box. Own the stack. Ship it.
