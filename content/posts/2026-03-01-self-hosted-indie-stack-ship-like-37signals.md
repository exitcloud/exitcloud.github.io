---
title: "Self-Hosted Indie Stack: Ship Like 37signals"
date: 2026-03-01T14:05:20.436Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# Self-Hosted Indie Stack: Ship Like 37signals

Here’s the thing — you don’t need Kubernetes to ship a self-hosted SaaS. You need a private registry, a sane deploy tool, checks that call your app from outside the box, and visibility that isn’t a black box. Then you iterate. That’s your deployment story.

This is a small-team clone of a proven pattern. It’s inspired by how [37signals](https://dev.37signals.com) runs their stuff: images in a private registry, apps shipped with [Kamal](https://github.com/basecamp/kamal), synthetic monitoring from multiple regions with [Upright](https://dev.37signals.com), and graphs/alerts powered by [Prometheus](https://prometheus.io/), [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/), and [Grafana](https://grafana.com/). They’ve even made storage an observability priority as they migrate data from S3 to [Pure Storage FlashBlade](https://dev.37signals.com), and they’re all‑in on [OpenTelemetry](https://opentelemetry.io) style, vendor‑neutral signals.

If you’re an indie dev running a tiny studio, a scrappy homelab, or a modest devops crew, this is the shape of stack you can actually keep in your head.

Let me show you what actually works.

- Your private registry: [Harbor](https://goharbor.io/) (because running images through third‑party registries can get messy and expensive — [37signals called out DockerHub’s paid license and pull/push usage leading to a considerable invoice](https://dev.37signals.com)).
- Your deploys: [Kamal](https://github.com/basecamp/kamal) with [Docker](https://www.docker.com/), the same model [37signals uses across all apps](https://dev.37signals.com).
- Your uptime checks: [Upright](https://dev.37signals.com), deployed to cheap VPS nodes across regions, exporting [Prometheus](https://prometheus.io/) metrics into [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and [Grafana](https://grafana.com/) dashboards (world map, 30‑day history, probe status views).
- Your signals pipeline: [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) as the glue, fed by your app and infra.

Yes, AI coding tools can draft boilerplate. They’re a force multiplier. You still own the stack.

One caveat before we start: the sources here are architectural and experience‑driven. They don’t include copy‑paste configs or full CLI steps. So I’ll map integrations, choices, and where to put each piece — and link you straight to the docs. No yak shaving.

Wild stuff.

## What We’re Building: A Small-Team Clone of 37signals’ Self-Hosted Stack

### The core pieces

Picture the flow:

- Build and store images in [Harbor](https://goharbor.io/).
- Deploy your app with [Kamal](https://github.com/basecamp/kamal) using [Docker](https://www.docker.com/). This mirrors how [37signals deploys everything](https://dev.37signals.com).
- Run [Upright](https://dev.37signals.com) on a couple of cheap VPS nodes in different regions. It’s a [Rails](https://rubyonrails.org/) engine deployed via Kamal that runs probes and exports [Prometheus](https://prometheus.io/) metrics. Those flow to [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and [Grafana](https://grafana.com/).
- Use the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) as a vendor‑neutral hub for metrics and traces. The OpenTelemetry community is focusing on stability and better‑organized releases and artifacts like docs and examples — a safe default for small teams who don’t want churn ([source](https://opentelemetry.io)).

Why this pattern? [37signals ran into registry costs with external providers](https://dev.37signals.com), moved to [Harbor](https://goharbor.io/), and deploy all apps with [Kamal](https://github.com/basecamp/kamal) using [Docker](https://www.docker.com/). They open‑sourced [Upright](https://dev.37signals.com) to replace third‑party uptime tools — it runs on cheap VPS nodes and drives [Prometheus](https://prometheus.io/) + [Grafana](https://grafana.com/) + [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) from day one. On storage, they’re migrating 10 petabytes out of S3 to [Pure Storage FlashBlade](https://dev.37signals.com) and monitoring it as a top observability priority with Prometheus. No black boxes.

And yes, the brief is intentionally high‑level — no full configs included. So we’ll keep it concrete without pretending we can paste a one‑size‑fits‑all file here. You’ll wire it from the docs. Then ship.

## Step 1 – Stand Up Harbor as Your Self-Hosted Docker Registry

A private registry isn’t optional once deploys become frequent. [37signals previously depended on DockerHub and Amazon ECR](https://dev.37signals.com), and they called out DockerHub’s paid license and pull/push usage generating a considerable invoice. That was a nudge to run [Harbor](https://goharbor.io/).

If you just want a PaaS, tools like Coolify exist. Here we’re deliberately owning the plumbing instead of outsourcing it.

### Why Harbor for a self-hosted registry?

Why Harbor?

- It’s open source and purpose‑built to be an internal registry.
- It secures artifacts with policies and role‑based access control, scans images for vulnerabilities, and signs images as trusted ([Harbor features](https://www.cncf.io)).
- It’s the right fit if your pipeline is already building docker images and pushing them with kamal.

### How to run Harbor in your stack

How to put it in your stack:

- Install [Harbor](https://goharbor.io/) following its official docs. You’ll configure a hostname, enable HTTPS, and choose where to keep layers.
- If you already have object storage, Harbor supports S3‑compatible backends. [37signals’ registry move was part of their broader cloud exit](https://dev.37signals.com), and they tied it into the rest of their on‑prem story.
- Enable the metrics endpoint so [Prometheus](https://prometheus.io/) can scrape Harbor. That keeps your registry from being a blind spot. Harbor’s production considerations cover policy/RBAC, vulnerability scanning, and signing flows ([CNCF overview](https://www.cncf.io)).

Contrarian take. There’s a lot of noise about Kubernetes being AI’s operating system — and the [Kubernetes v1.35 (Timbernetes) release](https://www.cncf.io) leans into AI infrastructure and mixed workloads. [WG Serving](https://www.cncf.io) wrapped after advancing AI inference support. That’s great. But you don’t need Kubernetes to push images to a registry and roll containers on two or three hosts. For an indie team, Harbor + Kamal + OpenTelemetry on a few VPSes gets you most of the value without turning your deploy into a 47‑service Rube Goldberg machine.

Make one change at a time. Registry first. Then deploys.

## Step 2 – Wire Kamal & Your App to Harbor (With AI as a YAML Sidekick)

[37signals deploys all of its applications with Kamal using Docker](https://dev.37signals.com). So should you. It’s a straight line from “I built an image” to “it’s running on the box.”

### Kamal + Harbor deployment flow

Integration checklist:

- Point [Kamal](https://github.com/basecamp/kamal) at your [Harbor](https://goharbor.io/) registry. You’ll set the registry URL, credentials, image name, and tags in your Kamal config.
- Keep your build push pull flow tight: build the image, push to Harbor, then deploy to your app nodes. That’s the whole deployment pipeline.
- Store minimal secrets in Kamal (only what’s necessary to log in to Harbor and your hosts). Everything else should come from your runtime config.

Use AI where it shines: drafting boilerplate. Have an LLM spit out a Kamal config skeleton or a CI snippet that logs into Harbor and runs Kamal. Then validate against the [Kamal docs](https://github.com/basecamp/kamal) and [Harbor docs](https://goharbor.io/). Trust but verify. I learned this the hard way.

### Keep the stack boring

This is also where the “small, simple, inspectable” stack pays off. If a deploy takes more than 60 seconds, something’s wrong. You’ll know immediately because the moving parts are few: build, push, roll. That’s the point.

One more thing that pays off later: requests that flow through your app should produce signals your stack understands. We’ll get to [OpenTelemetry](https://opentelemetry.io) in a minute, but keep that in your head while you wire Kamal. You’re setting up plumbing.

Ship it.

## Step 3 – Roll Your Own Uptime Monitoring with Upright on Cheap VPS Nodes

External checks are non‑negotiable. Curling your own health endpoint from inside the same network doesn’t tell you if DNS is broken in Sydney.

Enter [Upright](https://dev.37signals.com). It’s a [Rails](https://rubyonrails.org/) engine that you deploy to cheap VPS nodes via [Kamal](https://github.com/basecamp/kamal). Each node runs probes against your services from multiple geographic locations and exports [Prometheus](https://prometheus.io/) metrics that drive [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) alerts and [Grafana](https://grafana.com/) dashboards. The dashboards include a world map overview, a 30‑day uptime history, and per‑probe status views.

### Why this pattern works

Why this pattern works:

- You control the probe code. No surprises. No vendor lock‑in.
- It’s trivial to add a new region: spin up a small VPS, deploy the same Rails app via [Kamal](https://github.com/basecamp/kamal), and it’s part of your fleet.
- The outputs are plain [Prometheus](https://prometheus.io/) metrics, so alerting is first‑class with [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/), and visualization is first‑class with [Grafana](https://grafana.com/).

### Wiring Upright into your monitoring

How to make it real in your stack:

- Bootstrap an Upright instance and configure your first probes to hit your app. (Upright is a Rails engine; once mounted, you deploy it like any other Rails service with [Kamal](https://github.com/basecamp/kamal).)
- Set up two or three low‑cost VPS nodes in different regions. Deploy Upright to each node with Kamal. That gives you multi‑region checks by default.
- Expose the metrics endpoints so your monitoring box can scrape them with [Prometheus](https://prometheus.io/). Wire [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) routes so you hear about failures immediately. Import the world map and 30‑day history dashboards in [Grafana](https://grafana.com/).

The nicest part? It’s the same toolchain you’ll use for everything else. 37signals explicitly calls out that Upright fits into a [Prometheus + Alertmanager + Grafana](https://dev.37signals.com) flow. And they treat storage the same way — [monitoring 10 petabytes on Pure Storage with Prometheus](https://dev.37signals.com). One mental model. Fewer footguns.

Keep your probes short and clear. Start with simple HTTP checks. Expand later.

## Step 4 – Add Vendor-Neutral Observability with OpenTelemetry on a Single VPS

Logs. Metrics. Traces. If they don’t agree, you end up guessing. This is where [OpenTelemetry](https://opentelemetry.io) fits — a single schema and pipeline, lots of destinations you can swap out later. It’s the observability backbone of this self-hosted stack.

### Use the Collector as your hub

Use the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) as your hub:

- Collect [Prometheus](https://prometheus.io/) metrics from Upright and Harbor. Forward them to your metrics store and feed [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/).
- Accept [OTLP](https://opentelemetry.io/docs/specs/otlp/) traces from your app. Keep your code vendor‑neutral from day one.
- Normalize weird logs. The [Unroll Processor](https://opentelemetry.io) turns a single log record that contains multiple logical events (like a JSON array) into separate log records per event. That lets you alert on individual failures, not the bundle.
- Auto‑instrument where you can’t touch code yet. The [eBPF Instrumentation (OBI)](https://opentelemetry.io) project — originally Grafana Beyla — shipped its first alpha under the OTel umbrella with many new protocols and quality improvements. It’s designed to capture common HTTP/gRPC flows without code changes.

### Why OTLP in your code and pipeline?

Why OTLP in your code and pipeline? The OpenTelemetry project is sunsetting the Zipkin exporter specification in favor of Zipkin’s OTLP ingestion support. Rationale: the community strongly prefers OTLP, Zipkin exporters see limited usage (even less than already‑deprecated Jaeger exporters in some ecosystems), and user engagement on Zipkin‑exporter issues has been minimal ([deprecation rationale](https://opentelemetry.io)). Translation: emit OTLP, keep your options open, and you can still talk to Zipkin if needed.

Worried your environment is “too traditional” for this? On‑prem data centers, legacy apps, industrial systems — they’re battle‑tested, but observability is hard there. Noisy, unstructured logs. Siloed monitoring tools that fragment visibility. The OTel community wrote directly to that audience and showed incremental adoption works in exactly those setups ([traditional environments piece](https://opentelemetry.io)). Start with the Collector. Move one signal at a time.

And the community? It’s big and maturing. There’s a push on stability, reliability, and better organization of releases/docs ([stabilization focus](https://opentelemetry.io)). Localization became a core pillar at OpenTelemetry.io in 2025, which led to more contributors, translations, supported languages, and visibility for localized content ([localization recap](https://opentelemetry.io)). Humans matter here — community managers invited folks to connect via the CNCF Slack’s #opentelemetry channel and talked about the growth and transition of responsibilities as contributors expanded from just over 5,000 to almost 20,000 across a few years ([community updates](https://opentelemetry.io), [community growth](https://opentelemetry.io)). You’re not alone.

Keep it boring. One Collector on one small box. Signals from a few services. Expand as you need.

## Step 5 – Future‑Proof for AI & Agents on Kubernetes (When You’re Ready)

You don’t need Kubernetes for a blog. Not even close. But if your app grows into heavier AI/agentic workflows, K8s becomes attractive as an orchestration layer.

Why? [Kubernetes v1.35 (“Timbernetes”)](https://www.cncf.io) is framed as an AI‑infrastructure release — teams coordinate mixed production workloads: services, batch, data pipelines, ML training. [Kubernetes WG Serving](https://www.cncf.io) was created to make Kubernetes an orchestration platform of choice for AI inference and concluded after successfully advancing inference support. Meanwhile, agentic systems are moving from experiments to real production workloads, and teams are being asked to connect models to real tools, data, and workflows reliably — not brittle one‑off integrations ([Agentics Day](https://www.cncf.io)).

If you reach that point, don’t throw away your indie stack. Instead:

- Keep [Harbor](https://goharbor.io/) as your internal registry backbone. It slots nicely in front of clusters too.
- Expose services with [Gateway API](https://www.cncf.io). It’s more than “Ingress v2” — it’s a cleaner way to expose services without vendor‑specific, unstructured annotations.
- Use the same [OpenTelemetry](https://opentelemetry.io) pipeline for AI services. No black‑box inference endpoints. Signals or it didn’t happen.

Where to learn? The ecosystem is broad and welcoming. [BackstageCon](https://www.cncf.io) brings together adopters of [Backstage](https://backstage.io/), so you can build an internal developer portal that links Harbor, Upright, OTel dashboards, and your AI services. [Open Source SecurityCon](https://www.cncf.io) focuses on open source software and cloud security. The CNCF marked its [10th anniversary](https://www.cncf.io), and programs like [Kubernetes Community Days](https://www.cncf.io) and [Google Summer of Code 2026](https://www.cncf.io) keep the ladder down for new contributors and indie devs leveling up.

Grow into it. Don’t start there.

## A few Rails extras you can slot in (if relevant)

Not everyone needs this, but if you’re building product:

- Multi‑tenancy: a RECORDABLES episode walks through moving Fizzy from a commingled database to separate SQLite databases per customer and introduces [Active Record Tenanted](https://dev.37signals.com) to make multi‑tenancy more seamless in Rails, with safeguards to prevent accidental data leaks ([multi‑tenancy episode](https://dev.37signals.com)). 37signals even explored giving every Fizzy customer their own SQLite database, then unwound that days before launch due to tradeoffs ([behind Fizzy infrastructure](https://dev.37signals.com)). Real talk.
- Rich text: [Lexxy](https://dev.37signals.com) is a new editor for Action Text built on Meta’s Lexical framework, bringing proper HTML semantics, Markdown shortcuts and auto‑formatting, real‑time code syntax highlighting, link creation by pasting URLs on selected text, configurable prompts with mentions, and attachment previews for PDFs and videos.
- Push notifications: [Action Push Native](https://dev.37signals.com) is an open‑sourced Rails gem for sending push notifications to Apple and Google platforms. It was created to migrate off Amazon SNS and Pinpoint as part of their broader cloud exit and is used in Basecamp and HEY to send more than 10 million push notifications per day.
- Mobile: [Hotwire Native 1.2](https://dev.37signals.com) is a big update, improving API consistency and shipping new iOS and Android demo apps. It’s a web‑first framework for building native apps, including route decision handlers to send internal URLs to app screens and external URLs to the device browser.

Use what fits. Skip what doesn’t.

## Quick Reference

- [Harbor](https://goharbor.io/) — Open‑source container registry with policies, RBAC, vulnerability scanning, and image signing ([CNCF overview](https://www.cncf.io)).
- [Running our Docker registry on‑prem with Harbor](https://dev.37signals.com) — 37signals’ move from DockerHub/ECR to on‑prem Harbor, citing cost issues with external registries and deploying all apps with Kamal using Docker.
- [Kamal](https://github.com/basecamp/kamal) — Minimal deploy tool used by 37signals; build Docker image, push to registry, roll out to hosts.
- [Upright](https://dev.37signals.com) — Open‑sourced synthetic monitoring system; Rails engine deployed to cheap VPS nodes via Kamal; runs probes from multiple geographies; exports Prometheus metrics; Grafana dashboards include world map and 30‑day history.
- [Prometheus](https://prometheus.io/) — Metrics collection; scrape Harbor and Upright; feed [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/).
- [Grafana](https://grafana.com/) — Dashboards for Upright and infrastructure metrics.
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) — Vendor‑neutral signals hub for metrics/logs/traces.
- [OpenTelemetry eBPF Instrumentation (OBI)](https://opentelemetry.io) — First alpha under OTel (originally Grafana Beyla); adds protocols and quality improvements for large‑scale deployments.
- [Unroll Processor](https://opentelemetry.io) — Collector contrib processor that splits bundled logs (e.g., JSON arrays) into per‑event records.
- [Deprecating Zipkin Exporter](https://opentelemetry.io) — OTel is sunsetting Zipkin exporter spec in favor of Zipkin’s OTLP ingestion support.
- [Evolving OTel’s Stabilization and Release Practices](https://opentelemetry.io) — Community push on stability, reliability, and better‑organized releases/docs.
- [Demystifying OTel in Traditional Environments](https://opentelemetry.io) — Applying OTel incrementally to on‑prem/legacy/ICS with noisy logs and siloed tools.
- [OpenTelemetry.io 2025 Review](https://opentelemetry.io) — Localization became a core pillar; more contributors, languages, and visibility for localized docs.
- [What’s Up, OTel?](https://opentelemetry.io) and [Welcoming New Community Managers](https://opentelemetry.io) — Community growth and how to connect via CNCF Slack #opentelemetry.
- [Monitoring 10 Petabytes of data in Pure Storage](https://dev.37signals.com) — 37signals migration off S3 to Pure Storage FlashBlade and why it’s a top observability priority.
- [Moving Mountains of Data off S3](https://dev.37signals.com) — RECORDABLES episode on migrating billions of files with zero downtime; handling bandwidth limits, AWS constraints, custom tools, and human aspects.
- [Behind the Fizzy Infrastructure](https://dev.37signals.com) — Exploration (and unwinding) of per‑customer SQLite multi‑tenant design days before launch; tradeoffs and Kamal proxy/load‑balancing improvements.
- [Rails Multi‑Tenancy](https://dev.37signals.com) — RECORDABLES episode introducing Active Record Tenanted; converting an existing app and guarding against leaks.
- [Lexxy: A new rich text editor for Rails](https://dev.37signals.com) — Action Text editor features and prompts/mentions support.
- [Introducing Action Push Native](https://dev.37signals.com) — Self‑hosted push notifications for Apple/Google; used in Basecamp and HEY.
- [Announcing Hotwire Native 1.2](https://dev.37signals.com) — Biggest update since launch; API consistency and demo apps; route decision handlers.
- [Kubernetes as AI’s operating system: 1.35 release signals](https://www.cncf.io) — Timbernetes framing and mixed workload coordination.
- [Kubernetes WG Serving concludes](https://www.cncf.io) — Advancing AI inference support on Kubernetes.
- [Gateway API](https://www.cncf.io) — More than Ingress v2; structured routing without vendor‑specific annotations.
- [BackstageCon](https://www.cncf.io) — Co‑located event for [Backstage](https://backstage.io/) developer portals.
- [Open Source SecurityCon](https://www.cncf.io) — Co‑located event on open source and cloud security.
- [State of cloud native 2026](https://www.cncf.io) — CNCF’s 10th anniversary reflections and CTO insights.
- [Kubernetes Community Days 2026](https://www.cncf.io) — Community‑organized local events.
- [CNCF joins Google Summer of Code 2026](https://www.cncf.io) — Mentoring organization participation.
- [The Rails Delegated Type Pattern](https://dev.37signals.com) — RECORDABLES episode with conversational walkthroughs and screen shares.

## FAQ: Your Self-Hosted Indie Stack

### Does this self-hosted stack work on a single VPS?

Yes. You can run Harbor elsewhere and start with a single VPS that hosts your app, Upright, and the OpenTelemetry Collector. As you grow, split those roles onto separate nodes without changing the core pattern.

### How is this different from using a platform like Coolify or Fly.io?

Platforms are great when you don’t want to touch infra. Here you’re explicitly owning the registry, deployment flow, and observability. It’s closer to how 37signals runs — opinionated, self-hosted, and portable.

### Is this overkill for a small indie dev project?

Not if you keep it boring. One registry, one deployment tool, a couple of VPS nodes, and basic signals. That’s enough to keep an ambitious indie dev project upright without hiring a full devops team.

## Ship it

What you’ve got now: a private [Harbor](https://goharbor.io/) registry replacing DockerHub/ECR, [Kamal](https://github.com/basecamp/kamal) pointed at that registry, [Upright](https://dev.37signals.com) nodes calling your app from multiple regions, and an [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) feeding [Prometheus](https://prometheus.io/)/[Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and [Grafana](https://grafana.com/) so nothing hides.

This mirrors pieces of what [37signals runs](https://dev.37signals.com) — scaled to indie size. It’s all open source. It’s portable. It’s understandable. And it’s fully self-hosted.

Next moves: add more Upright probes for critical flows, expand OTel coverage across services, and, if you grow into AI/agent workloads, carry this stack forward into Kubernetes using [Gateway API](https://www.cncf.io) and the same observability pipeline. No rewrites.

And don’t go it alone. Hang out in CNCF Slack’s #opentelemetry ([community note](https://opentelemetry.io)), watch talks like the ones highlighted for KubeCon EU 2026 ([schedule callout](https://opentelemetry.io)), and binge 37signals’ [RECORDABLES](https://dev.37signals.com) for the real stories. Then get back to building.

