---
title: "Own Your Stack: A 37signals‑Style Self‑Hosted Platform (with AI Helping Behind the Scenes)"
date: 2026-03-11T14:09:32.743Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# Own Your Stack: A 37signals‑Style Self‑Hosted Platform (with AI Helping Behind the Scenes)

Here’s the thing — most indie apps don’t need a platform team, a platform mesh, or a platform anything. You need a VPS, a private registry, and a monitoring story you can explain to a human. The cool part? That’s exactly how 37signals runs serious software, and they’ve been dropping receipts.

They deploy everything with [Kamal](https://github.com/basecamp/kamal) and [Docker](https://www.docker.com/). They moved off “someone else’s registry” onto [Harbor](https://goharbor.io/) because the registry is one of the most integral pieces of the pipeline, and cost/control mattered during their cloud exit [source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor). They built a synthetic monitoring system called [Upright](https://dev.37signals.com/introducing-upright) that runs on cheap VPS nodes and exports [Prometheus](https://prometheus.io/) metrics you can alert on with [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and visualize with [Grafana](https://grafana.com/). And they treat storage like a first‑class citizen — moving mountains of data off [Amazon S3](https://aws.amazon.com/s3/) and using [Pure Storage FlashBlade](https://dev.37signals.com/pure-storage-monitoring) for attachments and long‑term metrics because that box is critical to observe and protect.

Let me show you what actually works.

Not theory. Ship it.

---

## Step 0: Decide What You Actually Self‑Host (VPS vs Kubernetes vs “Let the Cloud Have It”)

You want control and predictable bills. You don’t want to spend weekends debugging admission webhooks. Same.

- If your stack is a Rails or Node app, a database, a background worker, and a CDN, a few VPSes with [Kamal](https://github.com/basecamp/kamal) is plenty. 37signals runs all their apps with Kamal and Docker — no mystery platforms in the middle [source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor).
- If you’re building an AI platform, the calculus changes. The [CNCF](https://www.cncf.io/) points out that Kubernetes has grown beyond stateless web services and is becoming the common platform for AI workloads [source](https://www.cncf.io/blog/2026/03/05/the-great-migration-why-every-ai-platform-is-converging-on-kubernetes). Use [Kubernetes](https://kubernetes.io/) when you actually need it — heterogeneous GPU nodes, job schedulers, the works. Not for a blog.

Open source keeps evolving alongside big shifts — DVCS, CI/CD, containers, Kubernetes, and now generative AI [source](https://www.cncf.io/blog/2026/03/10/sustaining-open-source-in-the-age-of-generative-ai). Treat AI like a power tool: draft Dockerfiles, Kamal configs, and OpenTelemetry pipelines, then review and tighten by hand. Force multiplier, not autopilot.

The blueprint we’ll build:

- A private registry on [Harbor](https://goharbor.io/) that your app servers pull from with Kamal.
- A couple of app VMs deployed with [Kamal](https://github.com/basecamp/kamal), no control planes in sight.
- Synthetic monitoring via [Upright](https://dev.37signals.com/introducing-upright) on cheap VPS nodes.
- Observability from a single [OpenTelemetry Collector](https://opentelemetry.io/) + [Prometheus](https://prometheus.io/) + [Grafana](https://grafana.com/).
- Optional: de‑cloud app features with [Action Push Native](https://dev.37signals.com/introducing-action-push-native), [Active Record Tenanted](https://dev.37signals.com/rails-multi-tenancy), and [Lexxy](https://dev.37signals.com/announcing-lexxy-a-new-rich-text-editor-for-rails).

One more data point to anchor confidence: 37signals treats their [Pure Storage FlashBlade](https://dev.37signals.com/pure-storage-monitoring) as observability‑critical — it backs things like Basecamp attachments and Prometheus long‑term metrics. Serious storage. Still no Kubernetes mandate for a simple web stack. Wild stuff.

---

## Step 1: Own Your Container Pipeline with Harbor + Kamal on a Single VPS

I learned this the hard way. Your registry is not a checkbox. It’s the choke point for deploys, CI, and every rollout across your fleet. 37signals deploys all apps with [Kamal](https://github.com/basecamp/kamal) using [Docker](https://www.docker.com/), and they call their container registry one of the most integral pieces of the pipeline [source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor).

Before they switched, their world was tightly coupled to DockerHub and Amazon ECR. During their cloud exit, cost became a serious issue — including a paid DockerHub license — and they moved to an on‑prem [Harbor](https://goharbor.io/) registry for control and predictability [source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor). That move decouples your deploys from third‑party throttling and puts image retention/replication in your hands.

Do this:

- Spin up a VPS and install [Harbor](https://goharbor.io/). Keep it simple. One box to start. You can grow later.
- Point your CI to push images to Harbor instead of DockerHub/ECR.
- Teach [Kamal](https://github.com/basecamp/kamal) to pull from your Harbor host.

A tiny Kamal config sketch, just to orient your brain:

```yaml
# kamal.yml (example)
service: myapp
image: registry.example.com/myorg/myapp
servers:
  web:
    hosts:
      - app-1.example.com
      - app-2.example.com
registry:
  server: registry.example.com
  username: $REGISTRY_USER
  password: $REGISTRY_PASSWORD
env:
  clear:
    RAILS_ENV: production
```

Push, then `kamal deploy`. The app servers pull from your registry, not the public internet. Locality helps. Simplicity helps more.

If you later outgrow local disk, you can back images with object storage. 37signals’ broader cloud exit used [Pure Storage FlashBlade](https://dev.37signals.com/pure-storage-monitoring) for a range of workloads (attachments, long‑term metrics). Same philosophy applies: treat storage as stateful and important; run stateless deploy plumbing on boring VPSes.

Contrast with teams already deep in Kubernetes: container images are pulled from private registries thousands of times per day in production clusters, and vendors ship credential providers to wire this up [source](https://www.cncf.io/blog/2026/03/09/registry-mirror-authentication-with-kubernetes-secrets). Even if you’re not in K8s, the lesson holds — your registry is central.

Contrarian take: you don’t need an “observability platform.” For small teams, extra vendors add more surface area than insight. The [OpenTelemetry](https://opentelemetry.io/) project is leaning into stability and declarative config [source](https://opentelemetry.io/blog/2025/stability-proposal-announcement), and 37signals happily monitors massive storage and apps with [Prometheus](https://prometheus.io/) and custom tooling [source](https://dev.37signals.com/pure-storage-monitoring). A lean stack works. Ship faster.

---

## Step 2: Ship Your Own Synthetic Monitoring Network with Upright + Kamal + Prometheus

External uptime services are fine — until you need auth, custom flows, or predictable cost. Then you’re paying more and still staring into a black box. I’d rather run my own probes and see exactly what they’re doing.

[Upright](https://dev.37signals.com/introducing-upright) is a self‑hosted, open‑source synthetic monitoring system from 37signals. It’s implemented as a [Rails](https://rubyonrails.org/) engine. It runs health checks from multiple geographic locations. Each node exports [Prometheus](https://prometheus.io/) metrics you can alert on with [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and visualize with [Grafana](https://grafana.com/). The UI gives you a site overview with a world map, a 30‑day uptime history, and probe status across sites [source](https://dev.37signals.com/introducing-upright). That’s already a solid status page.

Deployment pattern? Keep it boring:

- Package Upright into a container.
- Use [Kamal](https://github.com/basecamp/kamal) to deploy that same image to a handful of cheap VPS nodes around the world — exactly how 37signals does it [source](https://dev.37signals.com/introducing-upright).
- Each node identifies itself via environment/config, runs probes, and exposes Prometheus metrics. No shared control plane. No flapping complexity.

What do you probe?

- Basic HTTP health checks for your APIs and web frontends.
- Authenticated flows that matter to your customers (login, dashboard load, file upload). Start simple. Expand carefully.

Wiring into observability:

- Scrape Upright’s metrics with [Prometheus](https://prometheus.io/). Aim alerts at wake‑you‑up failures and quiet everything else.
- Use [Grafana](https://grafana.com/) for time‑series views if you want more than the Upright UI’s world map and 30‑day histories [source](https://dev.37signals.com/introducing-upright).
- If you already run an [OpenTelemetry Collector](https://opentelemetry.io/), you can scrape via its Prometheus receiver and keep your metric aggregation in one place. Clean and compact.

Storage is where you invest. 37signals uses [Pure Storage FlashBlade](https://dev.37signals.com/pure-storage-monitoring) for things like Basecamp attachments and Prometheus long‑term metrics, and they treat it as a top priority for observability. For a small team, start local and keep retention sane. If you grow into months or years of metrics, graduate to object storage you control. Over‑invest in state. Under‑engineer stateless workers. That tension is the blueprint.

---

## Step 3: Add Real Observability with a Single‑Node OpenTelemetry Collector (Using the New Declarative Config)

You’re going to hit weird stuff. SQLite locks. Third‑party HTTP timeouts. Jobs that stall only under load. Without structured signals, you’re guessing.

Good news: [OpenTelemetry’s](https://opentelemetry.io/) declarative configuration is now stable in key parts. The JSON schema for the data model shipped as `opentelemetry-configuration 1.0.0`, with a YAML representation, an in‑memory representation, `ConfigProperties`, a `PluginComponentProvider` mechanism, and SDK operations for parsing YAML and instantiating configuration [source](https://opentelemetry.io/blog/2026/stable-declarative-config). Translation: you can keep your collector config as readable YAML, scriptable by an AI helper, reviewable by you.

Run a single Collector on one VPS next to your app stack. Receivers pull data in. Processors clean it up. Exporters send it to systems you already use.

- Receivers: `otlp` for traces/logs/metrics from your services; `prometheus` to scrape Upright and app metrics.
- Processors: `batch`, `memory_limiter`, and two big helpers:
  - Filter Processor with OTTL context inference — starting in collector‑contrib v0.146.0 you can write conditions in `trace_conditions`, `metric_conditions`, `log_conditions`, and `profile_conditions` that automatically understand the telemetry context [source](https://opentelemetry.io/blog/2026/ottl-context-inference-come-to-filterprocessor/).
  - Unroll Processor — turn one log record containing multiple logical events (like a JSON array) into separate log records so you can actually search and alert on them [source](https://opentelemetry.io/blog/2025/contrib-unroll-processor/).
- Exporters: a Prometheus exporter for Grafana to read; and OTLP exporters if you ever want to relay to another collector or service.

A tiny skeleton to make it concrete:

```yaml
# otel-collector.yml (example)
receivers:
  otlp:
    protocols:
      http:
      grpc:
  prometheus:
    config:
      scrape_configs:
        - job_name: 'upright'
          static_configs:
            - targets: ['upright-a.example.com:9464', 'upright-b.example.com:9464']

processors:
  batch: {}
  memory_limiter: {}
  filter:
    logs:
      log_conditions:
        - 'attributes["http.target"] == "/up" and attributes["http.status_code"] < 500'
  unroll:
    logs:
      field: body

exporters:
  prometheus:
    endpoint: "0.0.0.0:9464"

service:
  pipelines:
    metrics:
      receivers: [prometheus, otlp]
      processors: [batch, memory_limiter]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [unroll, filter, batch, memory_limiter]
      exporters: [prometheus]
    traces:
      receivers: [otlp]
      processors: [batch, memory_limiter]
      exporters: [prometheus]
```

Two pro tips:

- Use an AI assistant to translate “English” filters into OTTL for the Filter Processor (e.g., drop logs where path == /healthz and status < 500). Then review the YAML. Trust but verify.
- Treat the collector as your broom. The [OpenTelemetry](https://opentelemetry.io/) team is re‑orienting toward stability, reliability, and better releases/docs [source](https://opentelemetry.io/blog/2025/stability-proposal-announcement). That’s exactly the energy you want for a self‑hosted stack.

Traditional environments? On‑prem data centers, legacy apps, industrial systems — they’re everywhere. Their logs are noisy, unstructured, and monitoring is often siloed. Visibility gets fragmented fast. An OpenTelemetry blog breaks this down plainly [source](https://opentelemetry.io/blog/2026/demystifying-opentelemetry). A single collector next to your app brings order: normalize logs, attach attributes like service name/env, and keep the pipelines simple.

One last thing: Zipkin exporters are being deprecated in favor of OTLP ingestion as the community shifts toward OTLP [source](https://opentelemetry.io/blog/2025/deprecating-zipkin-exporters). Fewer formats. Fewer footguns.

---

## Step 4: De‑Cloud Your App Features: Push, Multi‑Tenancy, and Rich Text without AWS

Self‑hosting isn’t only infra. You can strip managed services out of your product, too — carefully, in stages.

- Push notifications, no AWS middleman. [Action Push Native](https://dev.37signals.com/introducing-action-push-native) is an open‑source Rails gem from 37signals that sends push notifications to Apple and Google directly. They built it to migrate off Amazon SNS and Pinpoint, and it now powers more than 10 million push notifications per day for Basecamp and HEY [source](https://dev.37signals.com/introducing-action-push-native). Clean API in Rails, HTTP/2 under the hood. Your app, your queue, your delivery.
- Multi‑tenancy with guardrails. 37signals explored giving every Fizzy customer their own [SQLite](https://sqlite.org/) database — partly to support both self‑hosting and SaaS, and partly to push multi‑tenant design further than before [source](https://dev.37signals.com/fizzy-infrastructure). Days before launch, they unwound parts of that ambitious design as the tradeoffs stacked up [source](https://dev.37signals.com/fizzy-infrastructure). They still open‑sourced [Active Record Tenanted](https://dev.37signals.com/rails-multi-tenancy) to make Rails multi‑tenancy smoother, and showed in a RECORDABLES episode how to convert an existing app with safeguards to prevent accidental data leaks [source](https://dev.37signals.com/rails-multi-tenancy). If you’ve been yearning for safer tenant boundaries, take a look.
- Better text editing UX — still self‑hosted. [Lexxy](https://dev.37signals.com/announcing-lexxy-a-new-rich-text-editor-for-rails) is a new rich text editor for Rails Action Text, built on Meta’s [Lexical](https://lexical.dev/). It focuses on ergonomics: paragraphs as proper <p> tags, markdown shortcuts and auto‑formatting on paste, real‑time code syntax highlighting, paste‑to‑link, configurable mentions, and inline previews for PDFs and videos [source](https://dev.37signals.com/announcing-lexxy-a-new-rich-text-editor-for-rails). No need to bolt in a random cloud editor.
- If you want native mobile, keep it web‑first. [Hotwire Native](https://dev.37signals.com/announcing-hotwire-native-v1-2) positions itself as a web‑first framework for building native mobile apps. Version 1.2 is the biggest update since launch, with better API consistency and route decision handlers that route internal URLs to app screens and external URLs to the device browser, plus new iOS and Android demo apps [source](https://dev.37signals.com/announcing-hotwire-native-v1-2). Pair that with your self‑hosted Rails backend and avoid spawning a new backend for mobile.

Zoom out: 37signals’ cloud exit was a staged story — off DockerHub/ECR to [Harbor](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor), off Amazon SNS/Pinpoint to [Action Push Native](https://dev.37signals.com/introducing-action-push-native), and off S3 while moving billions of files with no downtime [source](https://dev.37signals.com/moving-mountains-of-data-off-s3), landing on [Pure Storage FlashBlade](https://dev.37signals.com/pure-storage-monitoring) for 10 PB of data. You can mirror that arc in miniature. Start with registry and deploys. Then monitoring and observability. Finally app‑level dependencies. AI can help at each stage — as a refactoring assistant you double‑check.

---

## Quick Reference

- [Kamal](https://github.com/basecamp/kamal) — Simple Docker‑based deploys to a few VPS boxes; 37signals deploy all apps with it.
- [Harbor](https://goharbor.io/) — Private container registry; part of 37signals’ on‑prem pipeline to replace DockerHub/ECR [source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor).
- [Docker](https://www.docker.com/) — Container runtime/tooling used across 37signals’ deployments [source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor).
- [Upright](https://dev.37signals.com/introducing-upright) — Open‑source synthetic monitoring (Rails engine) running probes on cheap VPS nodes; exports Prometheus; UI shows world map, 30‑day history.
- [Prometheus](https://prometheus.io/) — Metrics collection; works with Upright and long‑term metrics at 37signals [source](https://dev.37signals.com/introducing-upright).
- [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) — Alerting for Prometheus metrics (use it with Upright).
- [Grafana](https://grafana.com/) — Visualization for Prometheus and other data sources (used alongside Upright) [source](https://dev.37signals.com/introducing-upright).
- [OpenTelemetry Collector](https://opentelemetry.io/) — Vendor‑neutral telemetry pipeline; run a single node with receivers/processors/exporters.
- OTel Declarative Config — Stable JSON/YAML schema, in‑memory model, ConfigProperties, PluginComponentProvider, and SDK ops for YAML parsing [source](https://opentelemetry.io/blog/2026/stable-declarative-config).
- OTel Filter Processor (OTTL context inference) — `trace_conditions`, `metric_conditions`, `log_conditions`, `profile_conditions` fields to simplify filtering [source](https://opentelemetry.io/blog/2026/ottl-context-inference-come-to-filterprocessor/).
- OTel Unroll Processor — Splits bundled log records (e.g., JSON arrays) into individual log events [source](https://opentelemetry.io/blog/2025/contrib-unroll-processor/).
- [Action Push Native](https://dev.37signals.com/introducing-action-push-native) — Rails gem to send push notifications directly to APNs/FCM; built to leave SNS/Pinpoint; used for 10M+ daily pushes at 37signals.
- [Active Record Tenanted](https://dev.37signals.com/rails-multi-tenancy) — Rails multi‑tenancy gem; 37signals demoed converting an existing app with safeguards against data leaks.
- [Lexxy](https://dev.37signals.com/announcing-lexxy-a-new-rich-text-editor-for-rails) — Rich text editor for Rails Action Text built on [Lexical](https://lexical.dev/) with markdown shortcuts, code highlighting, proper <p> tags, and inline previews.
- [Hotwire Native](https://dev.37signals.com/announcing-hotwire-native-v1-2) — Web‑first native mobile framework; v1.2 adds route decision handlers and new demo apps.
- [Pure Storage FlashBlade](https://dev.37signals.com/pure-storage-monitoring) — On‑prem storage for 10 PB of data at 37signals, including attachments, Prometheus long‑term metrics, and database backups.
- 37signals’ S3 migration story — Moved billions of files off S3 with no downtime; bandwidth limits, AWS constraints, and custom tooling [source](https://dev.37signals.com/moving-mountains-of-data-off-s3).
- Kubernetes context — Production clusters pull private images constantly [source](https://www.cncf.io/blog/2026/03/09/registry-mirror-authentication-with-kubernetes-secrets); becoming the common platform for AI workloads [source](https://www.cncf.io/blog/2026/03/05/the-great-migration-why-every-ai-platform-is-converging-on-kubernetes).
- Open source + AI — Generative AI is the next big wave shaping tooling choices [source](https://www.cncf.io/blog/2026/03/10/sustaining-open-source-in-the-age-of-generative-ai).

---

Here’s the thing — you don’t need Kubernetes to run a respectable stack. 37signals runs on Kamal and Docker, owns their registry with Harbor, monitors with Prometheus and custom tools like Upright, and treats storage as critical infrastructure. They even de‑clouded app features: push notifications via Action Push Native; multi‑tenant guardrails with Active Record Tenanted; better editing UX with Lexxy; and a web‑first path to mobile with Hotwire Native.

Own your stack. Keep it boring where it counts. Use AI as a smart assistant, not a crutch. Then hit deploy.

Ship it.
