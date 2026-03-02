---
title: "Self-Hosted Stack from Cloud Exit FOMO for Indie Devs"
date: 2026-03-02T14:06:13.836Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# Self-Hosted Stack from Cloud Exit FOMO for Indie Devs

Here’s the thing — you don’t need Kubernetes to ship a serious product as an indie dev. You need a self-hosted VPS setup and a Makefile. Maybe two VPS if you’re feeling fancy. The big kids have shown a path you can borrow without inheriting their complexity: 37signals moved billions of files off [Amazon S3](https://aws.amazon.com/s3/) with no downtime, then began moving 10 petabytes of data out of S3 to a [Pure Storage FlashBlade](https://www.purestorage.com/products/file-and-object/flashblade.html) system that’s now a top observability priority ([source](https://dev.37signals.com)). They deploy everything with [Kamal](https://kamal-deploy.org/) and [Docker](https://www.docker.com/) as of early 2025 ([source](https://dev.37signals.com)). Those patterns scale down beautifully.

Let me show you what actually works.

We’ll stand up synthetic monitoring on cheap VPS nodes with [Upright](https://dev.37signals.com), publish images to a self-hosted [Harbor](https://goharbor.io/) registry wired into Kamal, assemble a modern [Rails](https://rubyonrails.org/) stack with multi-tenancy and a first-class editor, swap cloud push for an open gem, and keep a small K8s corner for AI inference if you truly need it. And we’ll use AI tools where they shine: drafting configs and guardrails, not running your infra.

Ship it.

## What We’re Building: A Practical, Self-Hosted Stack You Can Copy

- Monitoring: A lightweight grid of [Upright](https://dev.37signals.com) nodes (a Rails engine) on cheap VPS boxes or homelab hardware, doing probes from multiple locations and exposing [Prometheus](https://prometheus.io/) metrics for [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and [Grafana](https://grafana.com/). Upright’s UI shows a world map, a 30-day uptime history, and probe status across sites ([source](https://dev.37signals.com)).
- Registry: On-prem [Harbor](https://goharbor.io/), tied into Kamal deploys with [Docker](https://www.docker.com/). Harbor brings policies, RBAC, vulnerability scanning, and signing ([source](https://www.cncf.io)).
- Storage: Use object storage you control. 37signals are moving 10 petabytes from S3 to [Pure Storage FlashBlade](https://www.purestorage.com/products/file-and-object/flashblade.html), which they’ll also use for database backups and treat as a top observability priority ([source](https://dev.37signals.com)).
- App: Rails with multi-tenancy using [Active Record Tenanted](https://dev.37signals.com), a better editor via [Lexxy](https://dev.37signals.com), and a mobile path with [Hotwire Native](https://dev.37signals.com).
- Optional AI corner: A small [Kubernetes](https://kubernetes.io/) cluster focused on inference and batch jobs, using [Gateway API](https://gateway-api.sigs.k8s.io/) instead of vendor-specific ingress. CNCF frames v1.35 (“Timbernetes”) as AI’s operating system ([source](https://www.cncf.io)), and WG Serving has wrapped after advancing AI inference support ([source](https://www.cncf.io)). Agentic systems are moving into production; they need reliable hooks into your real tools and data ([source](https://www.cncf.io)).

The pattern from 37signals is simple: replace one dependency at a time, keep deploys boring (Kamal + containers), and instrument everything. You can do the same on a couple boxes with far less devops overhead.

---

## Step 1 — Self-Hosted Synthetic Monitoring on Cheap VPS Nodes (Upright + Prometheus + OTel)

Start with monitoring. It’s the seatbelt for the rest of the ride.

- What Upright is: an open-source synthetic monitoring system built by 37signals, watching over Basecamp, HEY, Fizzy, and others. It runs health checks from multiple geographic locations, tells you when something breaks, and it’s a Rails engine you deploy to cheap VPS nodes using Kamal ([source](https://dev.37signals.com)). Each node runs probes and exposes Prometheus metrics so you can alert with Alertmanager and visualize in Grafana ([source](https://dev.37signals.com)).

- Why it matters: you get external checks from more than one spot on Earth, plus a UI with a world map, a 30-day uptime history, and per-site probe status ([source](https://dev.37signals.com)). Real signal. Low footprint.

Here’s a minimal approach:

### 1) Create an Upright app (Rails engine)

- Install Upright into a Rails app, add your probes, and run locally. Keep it simple at first: basic HTTP checks against your app’s health endpoints across regions.

### 2) Deploy Upright nodes with Kamal

- Package a container and point [Kamal](https://kamal-deploy.org/) at two or more cheap VPS hosts. They don’t need to be beefy — Upright’s whole schtick is running on low-cost nodes or homelab boxes ([source](https://dev.37signals.com)). Rolling updates, zero drama.

### 3) Scrape metrics and alert

- Each Upright node exposes Prometheus metrics ([source](https://dev.37signals.com)). Add a scrape job per node. Route alerts through Alertmanager so you page only when multiple regions agree something’s down. Use [Grafana](https://grafana.com/) for dashboards.

### 4) Tame logs with OpenTelemetry

Traditional environments often drown in noisy, unstructured logs and siloed monitoring, which fractures visibility ([source](https://opentelemetry.io)). This is where the [OpenTelemetry](https://opentelemetry.io) Collector shines.

- OTTL context inference: Write filters without sweating internal telemetry contexts. Starting with collector-contrib v0.146.0, the Filter Processor exposes top-level fields — trace_conditions, metric_conditions, log_conditions, and profile_conditions — so conditions are declared where they belong ([source](https://opentelemetry.io)). Cleaner configs. Fewer footguns.

- Unroll processor: If a log line bundles multiple events (say, a JSON array), the Unroll Processor expands it to one record per event ([source](https://opentelemetry.io)). Now your queries speak human.

Where AI helps: Ask your AI coding assistant to draft Alertmanager rules and OTTL filters in plain English (“Alert when 3 of 4 probes fail for 2 minutes”). Then edit by hand. You still own the logic.

Contrarian take: yes, [Kubernetes v1.35](https://www.cncf.io) is pitched as AI’s operating system, and the [WG Serving](https://www.cncf.io) work shows inference on K8s is very real. But you don’t need it for monitoring or for most indie apps. Two Dockerized services deployed with Kamal will get you 99% of the way with a fraction of the cognitive load. Fewer knobs, fewer surprises. Ship faster.

---

## Step 2 — Run Your Own Docker Registry with Harbor (and Wire It into Kamal)

I learned this the hard way. External registries add surprises: rate limits, vendor lock-in, and bills for features you barely use. 37signals had tight coupling to DockerHub and Amazon ECR and cited the considerable cost of a paid DockerHub license among the issues they hit during their “kamalization” journey ([source](https://dev.37signals.com)).

Bring the registry in-house with a self-hosted setup:

- Harbor overview: [Harbor](https://goharbor.io/) is an open-source container registry. It lets you secure artifacts with policies and role-based access control, scan images for vulnerabilities, and sign them as trusted ([source](https://www.cncf.io)). That bundle covers what most indie teams otherwise bolt on later.

- On-prem with Kamal: As of early 2025, 37signals deploys all apps with [Kamal](https://kamal-deploy.org/) using [Docker](https://www.docker.com/) containers ([source](https://dev.37signals.com)). You can do the same for your app images, targeting Harbor as your single source of truth.

How to fit it together:

### 1) Stand up Harbor

- Install [Harbor](https://goharbor.io/) on a host you control. Configure projects, set RBAC roles for your team, and enable vulnerability scanning and signing features ([source](https://www.cncf.io)).

### 2) Point Kamal at Harbor

- Update your Kamal deployment config to tag and push images to your Harbor registry instead of DockerHub/ECR. Keep image names consistent across environments, and you’ve just cut out a flaky dependency without touching your app.

### 3) Use Harbor’s security features

- Turn on scanning and enforce policies for what can be deployed ([source](https://www.cncf.io)). Sign images so your deploys only pull trusted artifacts. Peace of mind without a new micro-service.

Where AI helps: have your assistant spit out a skeleton Kamal config for image building and pushing, then generate Harbor RBAC entries and policy checklists. You’ll still review it, but the scaffolding work is done in minutes.

Unique insight: 37signals’ cloud exit pattern is rinse-and-repeat. They replaced S3 with on-prem storage, SNS/Pinpoint with an open gem, and DockerHub/ECR with Harbor — each time sticking to Kamal + containers ([source](https://dev.37signals.com); [source](https://dev.37signals.com); [source](https://dev.37signals.com)). That template works at small scale: pick a single dependency, swap in a focused tool, keep deployment boring, and only build custom bits where the off-the-shelf stuff doesn’t fit.

---

## Step 3 — Cut More Cloud Cords: Push, Storage, and Backups

This is where your app earns its independence.

### 1) Push notifications without SNS/Pinpoint

- Use [Action Push Native](https://dev.37signals.com), an open-sourced Rails gem for sending push notifications to mobile platforms. It supports both Apple and Google push services and was built as part of 37signals’ migration off Amazon SNS and Pinpoint ([source](https://dev.37signals.com)). It’s used in Basecamp and HEY to send more than 10 million push notifications per day ([source](https://dev.37signals.com)). Serious capacity.

- Wire this into your Rails app: configure credentials for Apple and Google, enqueue notifications, and deliver. Keep it observable with your existing metrics pipeline — the same Prometheus/Alertmanager combo you stood up earlier.

### 2) Plan storage migrations like an adult

- 37signals moved billions of files out of S3 with no downtime ([source](https://dev.37signals.com)). They had to juggle bandwidth limits and AWS constraints and even built custom tooling when off-the-shelf options fell short ([source](https://dev.37signals.com)). The human side was real, too: verification, anxiety, and the moment you finally hit delete on the cloud data ([source](https://dev.37signals.com)).

- For small teams: copy the mindset. Design for verification at each step, do dry runs, build checks you trust, and only flip the switch when you’re ready. The scale might be smaller, but the practice is the same.

### 3) Treat storage as a first-class system

- As part of their final move out of the cloud, 37signals are moving 10 petabytes from S3 to a [Pure Storage FlashBlade](https://www.purestorage.com/products/file-and-object/flashblade.html) solution ([source](https://dev.37signals.com)). The data includes Basecamp attachments and long-term Prometheus metrics; FlashBlade’s filesystem capabilities also fit database backup storage ([source](https://dev.37signals.com)). Because it holds core data and backups, they describe it as a top observability priority ([source](https://dev.37signals.com)).

- Your action: whatever storage you use — on-prem, colocation, or another provider — put it under the same observability umbrella. Make it visible in Prometheus, build alerts for availability and capacity trends, and treat it like your app depends on it. Because it does.

AI uses here: ask for checklists and scaffolding. “Draft a storage migration runbook with verification steps.” “Propose Prometheus alert rules for storage latency and free space.” Then trim the bloat and keep only what you’ll run at 3 a.m.

---

## Step 4 — Build a Modern Rails App Stack on Top (Lexxy, Multi-Tenancy, Hotwire Native)

Infra is only useful if your app is great. Let’s make the app great.

### 1) Multi-tenancy without tears

- 37signals explored moving Fizzy from a commingled database to separate [SQLite](https://www.sqlite.org/) databases for each customer — a per-customer model intended to support both self-hosted and SaaS, and it turned into a performance experiment ([source](https://dev.37signals.com); [source](https://dev.37signals.com)). Days before release, they unwound the multi-tenant experiment because the tradeoffs got too heavy ([source](https://dev.37signals.com)). Candid call.

- They open-sourced [Active Record Tenanted](https://dev.37signals.com) to make Rails multi-tenancy more seamless, with safeguards to prevent accidental data leaks ([source](https://dev.37signals.com)). Use that for multi-tenant boundaries you can trust. Start simple: tenant scoping, per-request tenant selection, and strong protections.

### 2) Better editing with Lexxy

- [Lexxy](https://dev.37signals.com) is a new rich text editor for Rails’ Action Text, based on Meta’s Lexical framework. It renders paragraphs as real <p> tags, supports Markdown shortcuts and auto-formatting on paste, offers real-time code syntax highlighting, lets users create links by pasting URLs onto selected text, and includes configurable prompts with mentions and multiple loading/filtering strategies ([source](https://dev.37signals.com)). It can preview attachments like PDFs and videos and works with Action Text ([source](https://dev.37signals.com)). A real upgrade.

### 3) Mobile without a separate API app

- [Hotwire Native](https://dev.37signals.com) is a web-first framework for building native mobile apps. Apps route internal URLs to screens in-app and external URLs to the device browser; v1.2 improves route decision handlers and brings key fixes plus greater API consistency between platforms, along with new iOS and Android demo apps ([source](https://dev.37signals.com)). Keep Rails as your source of truth.

Tie it back to your infra:

- Probe your Rails app with Upright.
- Build images, publish to Harbor, and deploy with Kamal.
- Send mobile push through Action Push Native.

Where AI fits: use it as your Rails co-pilot to draft model/migration boilerplate for tenancy or to scaffold editor integrations. Review every line that touches security boundaries. No autopilot on that.

If you like a Heroku-style layer on top of this, tools like Coolify give you a PaaS experience you can still run on your own servers.

---

## Step 5 — A Lightweight K8s Corner for AI Inference (Gateway API + OTel) with AI-Assisted Dev Loops

You don’t need K8s for your blog. You might want a small cluster for AI inference or batch jobs. Keep it in a corner. Don’t let it run the house.

Why now?

- The Kubernetes Working Group Serving was created to support the AI inference stack and aimed to make Kubernetes the orchestration platform of choice for AI inference; it has concluded after successfully advancing this support ([source](https://www.cncf.io)). And the v1.35 “Timbernetes” release leans into Kubernetes as AI’s operating system where teams coordinate services, batch jobs, data pipelines, and ML training ([source](https://www.cncf.io)). This isn’t vapor.

How to keep it small:

- Expose services with [Gateway API](https://gateway-api.sigs.k8s.io/), which is more than an “Ingress v2.” It avoids encoding routing into vendor-specific, unstructured annotations, as shown in the SpinKube + GatewayAPI example ([source](https://www.cncf.io)). Cleaner interfaces, less YAML archaeology.

- Agentic services: design agents that call your self-hosted tools (Rails endpoints, Upright metrics, Harbor API) instead of random SaaS. That aligns with the Agentics Day theme: connecting models to real tools, data, and workflows in reliable, secure ways without brittle one-off integrations ([source](https://www.cncf.io)). Practical. Maintainable.

Observability without sprawl:

- Use the [OpenTelemetry](https://opentelemetry.io) Collector’s context inference so filter configs drop into place with trace_conditions, metric_conditions, log_conditions, and profile_conditions in the Filter Processor ([source](https://opentelemetry.io)). Less config glue.

- Follow the project’s direction: the Zipkin exporter spec is being deprecated in favor of Zipkin’s OTLP ingestion support ([source](https://opentelemetry.io)). Prefer OTLP so you aren’t painting yourself into a protocol corner.

- The latest OpenTelemetry Configuration Schema release candidate is pulling the schema toward stability after years of iteration, with changes aimed at consistency and fewer new inconsistencies ([source](https://opentelemetry.io)). That helps small teams keep configs standard and slim.

Let AI earn its keep: ask it for a first pass at a Deployment + Service + Gateway API route for a tiny inference service, then you cut it down to essentials. Same for OTTL filters: “Drop logs with level=debug from namespace X” — now you get a starting point that uses context inference correctly. Don’t outsource judgment.

---

## Cheat Sheet

- [Upright](https://dev.37signals.com) — 37signals’ open-source synthetic monitoring (Rails engine) that runs probes from multiple locations, exposes Prometheus metrics, and shows a world map, 30-day uptime, and probe status (deployed via Kamal on cheap VPS nodes).
- [Prometheus](https://prometheus.io/) — Metrics collection for your Upright nodes and infra; pair with [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) for paging/notifications.
- [Grafana](https://grafana.com/) — Dashboards to visualize Prometheus metrics collected from Upright and more.
- [Kamal](https://kamal-deploy.org/) — Container-based deploys 37signals use for all apps as of early 2025.
- [Docker](https://www.docker.com/) — Containerization platform used with Kamal; replace external registries with Harbor.
- [Harbor](https://goharbor.io/) — Open-source container registry with policies, RBAC, vulnerability scanning, and image signing ([CNCF overview](https://www.cncf.io)).
- [Amazon S3](https://aws.amazon.com/s3/) — 37signals moved billions of files out with no downtime and are exiting S3 for 10 PB to on-prem storage ([source](https://dev.37signals.com); [source](https://dev.37signals.com)).
- [Pure Storage FlashBlade](https://www.purestorage.com/products/file-and-object/flashblade.html) — Destination for 10 PB of data (attachments and long-term Prometheus metrics); filesystem capability for DB backups; top observability priority ([source](https://dev.37signals.com)).
- [Action Push Native](https://dev.37signals.com) — Rails gem for mobile push (Apple + Google), built to migrate off SNS/Pinpoint; used to send 10M+ daily push notifications for Basecamp and HEY.
- [Active Record Tenanted](https://dev.37signals.com) — Rails gem to make multi-tenancy more seamless, with safeguards to prevent accidental data leaks; grounded in 37signals’ Fizzy work.
- [Lexxy](https://dev.37signals.com) — Action Text editor based on Lexical: semantic HTML (<p> tags), Markdown shortcuts, auto-format on paste, real-time code highlighting, paste-to-link, configurable prompts with mentions, and attachment previews.
- [Hotwire Native](https://dev.37signals.com) — Web-first native apps; v1.2 adds improvements, more API consistency, better route decision handlers, plus new iOS/Android demo apps.
- [Kubernetes v1.35](https://www.cncf.io) — Release framed as AI’s operating system; coordinate services, batch jobs, data pipelines, and ML training.
- [Kubernetes WG Serving](https://www.cncf.io) — Concluded after advancing AI inference support on Kubernetes, aiming to make it the platform of choice for inference.
- [Gateway API](https://gateway-api.sigs.k8s.io/) — Modern service exposure for Kubernetes; avoids vendor-specific, unstructured annotations ([context](https://www.cncf.io)).
- [OpenTelemetry Collector](https://opentelemetry.io) — Use OTTL context inference in Filter Processor (trace_conditions, metric_conditions, log_conditions, profile_conditions) for simpler configs; Unroll Processor expands bundled log records; Zipkin exporter spec is deprecated in favor of Zipkin’s OTLP ingestion.
- [OpenTelemetry Configuration Schema](https://opentelemetry.io) — Latest release candidate pushes toward stability with consistency improvements.
- Observability in traditional environments — Noisy logs and siloed data cause fragmented visibility; OTel helps address this ([source](https://opentelemetry.io)).
- [Coolify](https://coolify.io) — A self-hosted PaaS you can drop onto your own servers when you want a Heroku-like DX on top of this stack.

---

## FAQ: Self-Hosted Indie Stack

### Do I need Kubernetes for a self-hosted indie app?

Usually not. For most indie dev projects, a couple of Docker services on a VPS with Kamal are enough. Keep Kubernetes in a small corner only if you truly need AI inference or heavy batch workloads.

### Can I run this self-hosted stack on a homelab?

Yes. Upright, Harbor, and Rails all run fine on modest hardware, so a homelab or small VPS fleet works. Just treat power, connectivity, and backups as seriously as you would in the cloud.

### How does this compare to using a PaaS like Coolify?

Coolify smooths over a lot of deployment details with a self-hosted PaaS layer. This guide shows you the lower-level building blocks — Kamal, Docker, Harbor — so you can understand and control the stack even if you later add a PaaS.

---

Here’s the thing — by now you’ve got more than diagrams. You’ve got a path to:

- a running synthetic monitoring grid on cheap VPS nodes,
- a self-hosted Harbor registry wired into Kamal,
- a Rails app with multi-tenancy and an editor your users won’t fight,
- a push pipeline that isn’t tied to SNS or Pinpoint,
- and a small, focused K8s corner for AI inference if you need it,

all observable with Prometheus and OpenTelemetry. 37signals’ stories back up the pattern: boring deploys, tight control over core data, and open tooling that you understand. Not even close to a Rube Goldberg machine.

Start with one piece — Upright or Harbor — get it solid, then layer in the rest. Own your self-hosted stack. Ship it.

