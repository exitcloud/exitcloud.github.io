---
title: "From Zero to Indie-Grade Platform: Self‑Hosted Deploys, AI Helpers, and Modern Observability on a Single VPS"
date: 2026-03-16T14:08:48.396Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# From Zero to Indie-Grade Platform: Self‑Hosted Deploys, AI Helpers, and Modern Observability on a Single VPS

“On a tiny VPS, I run headless browser agents in parallel. No GPU. No Kubernetes. Just containers and a Makefile.” Here’s the thing — you don’t need a cluster to ship like a pro. You need boring, battle‑tested pieces wired together cleanly. And the courage to own your stack.

We’ll stand up a practical indie platform:
- A self‑hosted container registry (Harbor) so your pipeline isn’t at the mercy of rate limits or surprise invoices.
- One‑command deploys with Kamal on a single VPS. Fast, reproducible, git‑driven.
- OpenTelemetry Collector for observability, defined in plain YAML that’s now declared stable.
- Synthetic monitoring from outside your box (Upright). Browser‑level checks with a headless engine built for automation (Lightpanda).

This mirrors patterns used by 37signals. They deploy with [Kamal](https://kamal-deploy.org/) and treat the container registry as first‑class infra, moving from DockerHub/ECR to an on‑prem [Harbor](https://goharbor.io/) registry as part of their pipeline shift ([source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor)). For monitoring, they open‑sourced [Upright](https://dev.37signals.com/introducing-upright), a synthetic monitoring system that runs health checks from multiple geographic locations, implemented as a Rails engine deployed to cheap VPS nodes with Kamal. Each node exposes [Prometheus](https://prometheus.io/) metrics, which you can wire into [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and [Grafana](https://grafana.com/).

And that headless browser piece? Lightpanda is written in Zig, speaking only DOM + [V8](https://v8.dev/) with a Chrome DevTools Protocol server. In benchmarks across 933 real pages at 25 parallel tasks, it used about 215MB versus Chrome’s 2GB and finished in roughly 5 seconds versus 46. Even at 100 parallel tasks, Lightpanda reportedly used about 696MB while Chrome hit 4.2GB, with Chrome degrading under that load (same source). AI‑friendly and resource‑light. Ship it.

Also, the self‑hosting community’s stance on AI is pragmatic: r/selfhosted explicitly allows AI‑related content, while tightening rules so quality and review matter (policy update, rule simplification). Use AI as a force multiplier, not a replacement. I learned this the hard way.

Let me show you what actually works.

## Step 1 – Own Your Images: Minimal Harbor Registry on a Single VPS

37signals ran into the traps many of us do: tight coupling to DockerHub and ECR, a not‑small DockerHub invoice, and costs tied to pulling/pushing images across regions. As part of their “cloud exit and kamalization journey,” they moved their registry on‑prem with [Harbor](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor). The point wasn’t novelty — it was control and speed. Bring the bits close to your deploy target, cut external dependencies.

Do the same in miniature:
- Set a DNS name for your registry, like registry.example.com.
- Install [Harbor](https://goharbor.io/) using its single‑node Docker Compose bundle.
- Put it behind your reverse proxy (I like [Caddy](https://caddyserver.com/) for simplicity) or any ingress you already use.

A tiny example harbor.yml (illustrative):

```yaml
hostname: registry.example.com
http:
  port: 80
harbor_admin_password: "change-me"
database:
  password: "harbor-db-pass"
data_volume: /opt/harbor/data
storage_service:
  # Use local filesystem for simplicity; swap to S3-compatible later if needed.
  ca_bundle: ""
jobservice:
  max_job_workers: 3
log:
  level: info
  local:
    rotate_count: 5
    rotate_size: 200M
```

If you prefer an object store, point Harbor’s storage to an S3‑compatible endpoint (for example, [MinIO](https://min.io/)). Keep credentials tight. Keep it boring.

Login from your dev machine and push a sample image:

```bash
docker login registry.example.com
# Build and tag your app
docker build -t myapp:latest .
docker tag myapp:latest registry.example.com/myteam/myapp:latest
docker push registry.example.com/myteam/myapp:latest
```

Now your deploys pull from a registry you own. This isn’t just “nice to have.” The [CNCF series on registry mirrors and auth](https://www.cncf.io/blog/2026/03/09/registry-mirror-authentication-with-kubernetes-secrets) describes how production clusters pull images from private registries thousands of times per day. You don’t need Kubernetes to feel that pain — even a single node can get rate‑limited or throttled by an external registry in awkward moments. Local pulls. Fewer surprises.

Quick take that might ruffle feathers: Kubernetes for an indie platform is often yak shaving. The CNCF blog argues AI platforms are converging on Kubernetes ([their case](https://www.cncf.io/blog/2026/03/05/the-great-migration-why-every-ai-platform-is-converging-on-kubernetes)), and there’s a whole post about debugging etcd incidents in production clusters ([here](https://www.cncf.io/blog/2026/03/12/making-etcd-incidents-easier-to-debug-in-production-kubernetes)). That’s a lot of moving parts. For a solo app, a VPS, a reverse proxy, and Harbor get you 90% of the benefit with 10% of the complexity. Not even close.

## Step 2 – Fast, Reproducible Deploys with Kamal (Plus Sensible AI Assistance)

As of early 2025, 37signals deploys all applications with [Kamal](https://kamal-deploy.org/) using [Docker](https://www.docker.com/), and the registry is an integral part of that pipeline ([source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor)). Upright nodes are deployed the same way — cheap VPSs, Kamal doing the work, images pulled from their registry ([Upright](https://dev.37signals.com/introducing-upright)). Do this:

Example kamal.yml:

```yaml
service: myapp
image: registry.example.com/myteam/myapp
servers:
  web:
    hosts:
      - vps1.example.com
    options:
      # Example envs passed to container
      env:
        RACK_ENV: production
        RAILS_ENV: production
registry:
  server: registry.example.com
  username: "${REGISTRY_USER}"
  password: "${REGISTRY_PASS}"
env:
  clear:
    PORT: 3000
# Health endpoint Kamal can curl after deploy
deploy:
  healthcheck:
    path: /health
    interval: 5
    max: 60
```

CI wiring (pseudocode YAML):

```yaml
steps:
  - name: Build
    run: docker build -t registry.example.com/myteam/myapp:${GIT_SHA} .
  - name: Tag and push
    run: |
      docker tag registry.example.com/myteam/myapp:${GIT_SHA} registry.example.com/myteam/myapp:latest
      docker login -u $REGISTRY_USER -p $REGISTRY_PASS registry.example.com
      docker push registry.example.com/myteam/myapp:${GIT_SHA}
      docker push registry.example.com/myteam/myapp:latest
  - name: Deploy
    env:
      REGISTRY_USER: ${{ secrets.REGISTRY_USER }}
      REGISTRY_PASS: ${{ secrets.REGISTRY_PASS }}
    run: |
      gem install kamal
      kamal envify
      kamal deploy
```

Rollouts and rollbacks:

```bash
kamal deploy
kamal app exec -- bin/rails db:migrate
kamal app restart
# If things go sideways:
kamal rollback
```

Treat your VPS as disposable. Keep kamal.yml and scripts in git. Recreate from scratch. A homelabber in r/selfhosted already does this mindset with a [Headscale](https://github.com/juanfont/headscale) + [Tailscale](https://tailscale.com/) VPN fronted by a VPS, using [Ansible](https://www.ansible.com/) and exploring GitOps patterns for a [Proxmox](https://www.proxmox.com/) setup — even pointing out people manage Proxmox with [Terraform](https://www.terraform.io/) (discussion). Same energy.

Where AI fits: have your favorite model sketch a kamal.yml or CI pipeline. Then review it line‑by‑line. r/selfhosted’s moderators explicitly allow AI content but tightened rules so low‑effort “vibe coded” drops don’t overwhelm the signal (policy, update). That’s your bar, too.

One more example from the “own your stack” playbook: 37signals built [Action Push Native](https://dev.37signals.com/introducing-action-push-native/) to get off AWS SNS/Pinpoint, sending Apple and Google push notifications directly. They use it in Basecamp and HEY to send more than 10 million push notifications per day. That’s the same thesis as Harbor + Kamal + VPS deploys: fewer external dependencies, clearer paths to production.

Side note that’s fun: Lightpanda’s benchmarks (933 real pages; ~5s vs 46s; ~215MB vs ~2GB at 25 tasks) plus the homelab trend of reusing low‑power hardware — like a broken‑screen Steam Deck turned Debian NAS with rsync snapshots over 2.5GbE and idle draw around 10–15W (story) — show how far careful engineering goes. Lightweight agents. Real work on modest gear. Wild stuff.

## Step 3 – Real Observability Without Kubernetes: OpenTelemetry Collector + YAML

OpenTelemetry’s declarative configuration spec is now marked stable: JSON and YAML representations of the data model, an in‑memory representation, a generic YAML mapping node representation called ConfigProperties, a PluginComponentProvider for custom plugin components, and SDK operations to parse YAML and instantiate components ([announcement](https://opentelemetry.io/blog/2026/stable-declarative-config)). Translation: you can describe your observability setup in one YAML file and trust the tooling.

Spin up the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) on the same VPS. Minimal otel-collector.yaml:

```yaml
receivers:
  otlp:
    protocols:
      http:
      grpc:
  prometheus:
    config:
      scrape_configs:
        - job_name: "myapp"
          static_configs:
            - targets: ["myapp:3000"]   # your app container or host:port

processors:
  batch: {}
  # Drop noisy health checks with OTTL context inference
  filter:
    log_conditions:
      - 'body matches "GET /health"'
    metric_conditions:
      - 'name == "http_requests_total" and attributes["path"] == "/health"'
  # Expand bundled logs (JSON arrays) into individual records
  unroll: {}

exporters:
  prometheus:
    endpoint: "0.0.0.0:9464"
  file:
    path: "/var/log/otel-collector/logs.ndjson"

service:
  pipelines:
    metrics:
      receivers: [otlp, prometheus]
      processors: [batch, filter]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [unroll, filter, batch]
      exporters: [file]
```

Notes:
- The Filter Processor’s new context inference makes conditions clearer via `trace_conditions`, `metric_conditions`, `log_conditions`, and `profile_conditions` so you don’t have to wire internal telemetry contexts by hand ([details](https://opentelemetry.io/blog/2026/ottl-context-inference-come-to-filterprocessor)). The examples above illustrate the shape — use your app’s real metric/log names.
- The Unroll Processor was added to handle log records containing multiple logical events by expanding them into separate records ([intro](https://opentelemetry.io/blog/2025/contrib-unroll-processor)).
- If you still export Zipkin somewhere, note that OpenTelemetry is deprecating Zipkin exporters in favor of Zipkin’s OTLP ingestion ([deprecation](https://opentelemetry.io/blog/2025/deprecating-zipkin-exporters)). Move to OTLP and you’ll be happier.

Pipe the Collector’s Prometheus exporter into your Prometheus server, and graph it in Grafana. You’ll see app request latency, error rates, and deployment versions (easy to add as labels in your app). Stack traces and logs stream in, filtered and unrolled. Done.

Yes, OpenTelemetry is powering serious scale — there’s a KubeCon EU talk titled “We Deleted Our Observability Stack and Rebuilt It With OTel: 12 Engineers to 4 at 20K+ Clusters” ([listing](https://opentelemetry.io/blog/2026/kubecon-eu/)) and Observability Day is now a core CNCF event ([deep dive](https://www.cncf.io/blog/2026/03/13/kubecon-cloudnativecon-europe-2026-co-located-event-deep-dive-observability-day/)). But the same Collector + YAML config works beautifully on a single VPS. OpenTelemetry’s “Demystifying” post even calls out traditional environments — on‑prem data centers, legacy apps, ICS — with noisy unstructured logs and siloed monitoring as prime OTel targets ([context](https://opentelemetry.io/blog/2026/demystifying-opentelemetry)). This is for us, too.

If you want a bit more config discipline, there’s been steady work on the configuration schema (RC3) to improve consistency and tooling, aiming to be the last RC before stable ([RC3](https://opentelemetry.io/blog/2025/declarative-config-rc3)). Keep an eye on it.

## Step 4 – Synthetic Monitoring and AI‑Driven Browser Checks (Upright + Lightpanda)

Metrics tell you the engine is warm. Synthetic checks tell you the car actually moves. [Upright](https://dev.37signals.com/introducing-upright) is an open‑sourced synthetic monitoring system built by 37signals to watch over Basecamp, HEY, Fizzy, and others by running health checks from multiple geographic locations. It’s a Rails engine deployed to cheap VPS nodes using [Kamal](https://kamal-deploy.org/); each node runs probes and exposes [Prometheus](https://prometheus.io/) metrics for [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and [Grafana](https://grafana.com/).

Do a simple setup:
- Stand up a lightweight Rails app and mount Upright (it’s shipped as a Rails engine).
- Define HTTP probes hitting your `/health` and a user‑facing path. Start with one node, then add another in a different region to distinguish local blips from real outages.
- Scrape the Upright node’s Prometheus metrics into your Prometheus + Grafana instance alongside app metrics. Alert on failure counts and latency.

Now add browser‑level checks. Lightpanda is a headless browser written from scratch in Zig, designed for automation and AI agents, with no graphical rendering — just DOM + [V8](https://v8.dev/) and a Chrome DevTools Protocol server. Use any [Playwright](https://playwright.dev/) or [Puppeteer](https://pptr.dev/) script that connects to a CDP WebSocket endpoint and validates your UI’s DOM state.

A minimal Puppeteer‑style snippet (point WS_ENDPOINT to your Lightpanda CDP):

```js
import puppeteer from "puppeteer-core";

const browser = await puppeteer.connect({ browserWSEndpoint: process.env.WS_ENDPOINT });
const page = await browser.newPage();
await page.goto("https://app.example.com/login", { waitUntil: "networkidle2" });
await page.type("#email", "smoketest@example.com");
await page.type("#password", "not-a-real-password");
await page.click("button[type=submit]");
await page.waitForSelector("[data-test=dashboard]");
await browser.close();
```

Lightpanda’s reported resource profile makes it feasible to run many of these checks in parallel where headless Chrome would struggle: about 215MB vs 2GB at 25 tasks, roughly 5s vs 46s completion; about 696MB vs 4.2GB at 100 tasks, with Chrome performance degrading (benchmarks). That’s the difference between “we can afford to run this 24/7” and “let’s hope no one notices the outage at 3AM.”

You can also let an agent drive these checks. [Chrome DevTools MCP](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session) shows mainstream interest in integrating DevTools with AI protocols. If you go this route, keep strict guardrails: target only your domains, cap concurrency, log everything. The r/selfhosted moderation updates embraced AI while setting expectations for quality and review (policy, rules). You should too.

## Where to Go Next: Identity, Privacy, and Homelab‑Scale Resilience

As your stack grows, you’ll need identity. The KeycloakCon deep dive frames IAM as “foundational infrastructure,” especially across multi‑service and agent‑based patterns ([CNCF writeup](https://www.cncf.io/blog/2026/03/06/kubecon-cloudnativecon-europe-2026-co-located-event-deep-dive-keycloakcon)). Consider adding [Keycloak](https://www.keycloak.org/) or similar behind your reverse proxy once you have multiple apps.

Privacy and sovereignty aren’t abstract. The ROOT Observer camera project has open‑source firmware and app, end‑to‑end encryption, on‑device ML for event detection, encrypted push notifications, OTA updates, and a self‑hostable “dumb” relay that cannot decrypt messages; footage stays local. Another homelabber isolated the Android tablet inside a Decent Espresso machine into its own firewall VLAN and blocked egress for a week to see what it tried to reach (experiment). Control your network. Control your data flows. The CNCF’s Open Sovereign Cloud Day post signals that sovereignty has become an urgent engineering priority, especially in Europe ([overview](https://www.cncf.io/blog/2026/03/16/kubecon-cloudnativecon-europe-2026-co-located-event-deep-dive-open-sovereign-cloud-day)).

Extend your platform into a homelab: repurpose hardware (that Steam Deck NAS with rsync snapshots and 2.5GbE, idle ~10–15W — post). Consider long‑term cold storage with an open‑source LTO tape backup appliance. Manage your Proxmox setup with a Git repo and an [Ansible](https://www.ansible.com/) script; some folks even eye [Terraform](https://www.terraform.io/) for parts of it (thread). Same patterns, wider surface area.

And if you’re into DIY networking rabbit holes, there’s a neat case of a 75KB MIT‑licensed MPI library built over RDMA to avoid a $15,000–$50,000 managed InfiniBand switch by using reliable connected (RC) queue pairs instead of UD (story). That spirit — minimal, targeted engineering — is exactly what we’re doing here.

## Cheat Sheet

- [Harbor](https://goharbor.io/) — Self‑hosted Docker image registry; 37signals moved off DockerHub/ECR to an on‑prem Harbor as a core pipeline component ([context](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor)).
- [Docker](https://www.docker.com/) — Build and run containers; Kamal deploys your images to a VPS.
- [Kamal](https://kamal-deploy.org/) — One‑command deploys via Docker over SSH; used by 37signals across apps ([note](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor)).
- [Upright](https://dev.37signals.com/introducing-upright) — Rails engine for synthetic monitoring; deploy nodes with Kamal, export Prometheus metrics for Alertmanager/Grafana.
- [Prometheus](https://prometheus.io/) — Metrics store; scrape app and Upright nodes.
- [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) — Alerting for Prometheus metrics.
- [Grafana](https://grafana.com/) — Dashboards for metrics and logs.
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) — Receives OTLP, scrapes Prometheus, processes, and exports; declarative YAML config is now stable ([details](https://opentelemetry.io/blog/2026/stable-declarative-config)).
- [OTTL context inference](https://opentelemetry.io/blog/2026/ottl-context-inference-come-to-filterprocessor) — Filter Processor fields: trace_conditions, metric_conditions, log_conditions, profile_conditions.
- [Unroll Processor](https://opentelemetry.io/blog/2025/contrib-unroll-processor) — Splits logs containing multiple logical events into individual records.
- [Zipkin exporters deprecation](https://opentelemetry.io/blog/2025/deprecating-zipkin-exporters) — Prefer OTLP ingestion in Zipkin.
- Lightpanda — Zig + V8 headless browser for automation/AI agents with strong parallel benchmarks vs Chrome.
- [Chrome DevTools MCP](https://developer.chrome.com/blog/chrome-devtools-mcp-debug-your-browser-session) — DevTools integration with Model Context Protocol for AI workflows.
- [Caddy](https://caddyserver.com/) — Simple reverse proxy to front Harbor/app (example ingress).
- [MinIO](https://min.io/) — S3‑compatible object storage (optional Harbor backend).
- [Action Push Native](https://dev.37signals.com/introducing-action-push-native/) — Rails gem for APNs/FCM push; 37signals created it to migrate off SNS/Pinpoint.
- [Headscale](https://github.com/juanfont/headscale) + [Tailscale](https://tailscale.com/) — VPN pattern used by homelabbers; VPS as frontend via [Caddy](https://caddyserver.com/) (thread).
- [Proxmox](https://www.proxmox.com/), [Ansible](https://www.ansible.com/), [Terraform](https://www.terraform.io/) — GitOps‑style infra management discussed in r/selfhosted (post).
- ROOT Observer — E2E‑encrypted camera with on‑device ML and a self‑hostable “dumb” relay.
- Steam Deck NAS — Low‑power repurposed NAS build; rsync snapshots and 2.5GbE.
- [Open Sovereign Cloud Day](https://www.cncf.io/blog/2026/03/16/kubecon-cloudnativecon-europe-2026-co-located-event-deep-dive-open-sovereign-cloud-day) — Sovereignty as an engineering priority.
- [Observability Day](https://www.cncf.io/blog/2026/03/13/kubecon-cloudnativecon-europe-2026-co-located-event-deep-dive-observability-day) — CNCF’s observability community event.
- [Demystifying OpenTelemetry](https://opentelemetry.io/blog/2026/demystifying-opentelemetry) — OTel for traditional environments with noisy logs and siloed tools.

Ship the essentials first. Then layer.

---

Here’s the stack you just assembled: a Harbor‑style registry you own, a Kamal pipeline for fast deploys, an OpenTelemetry Collector defined in stable YAML, Upright‑style synthetic monitoring, and Lightpanda for browser‑level checks. All on commodity VPS hardware. No Kubernetes required. No managed black boxes. Just a few containers, a reverse proxy, and a git repo.

Keep iterating in small, testable steps. Treat infra changes like code. Let AI scaffold, but don’t abdicate review. Tighten observability until you catch problems before users do. The goal isn’t to chase every cloud‑native buzzword — it’s to build a platform that fits an indie developer’s reality: understandable, reproducible, yours.
