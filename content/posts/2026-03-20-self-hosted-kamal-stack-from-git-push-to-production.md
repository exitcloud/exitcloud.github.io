---
title: "Self-Hosted Kamal Stack: From Git Push to Production"
date: 2026-03-20T14:05:49.195Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# Self-Hosted Kamal Stack: From Git Push to Production

Here’s the thing — AI coding isn’t your bottleneck anymore. Shipping is. The work after git push got heavier while code got easier to write. That’s straight from a CNCF post: AI‑assisted dev moved the slow part to provisioning, policy, day‑two ops, and drift cleanup after the commit hits main ([source](https://www.cncf.io/blog/2026/03/20/crossplane-and-ai-the-case-for-api-first-infrastructure/)). So let’s build a small self-hosted stack that keeps that surface area tiny. One app box. One infra box. A private image registry. A dead‑simple deploy path. Real monitoring with metrics, not vibes.

Motivation? 37signals just hauled serious data out of [Amazon S3](https://aws.amazon.com/s3/) — billions of files — to a [Pure Storage FlashBlade](https://dev.37signals.com/pure-storage-monitoring/) setup, while keeping apps online ([source](https://dev.37signals.com/moving-mountains-of-data-off-s3/)). They also ditched external registries and run an on‑prem registry with [Harbor](https://goharbor.io/) because it’s a critical part of their pipeline, and all their apps ship with [Kamal](https://kamal-deploy.org/) and [Docker](https://www.docker.com/) now ([source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/)). For monitoring? They built and open‑sourced [Upright](https://dev.37signals.com/introducing-upright/) — a synthetic monitoring system that runs on cheap VPS nodes, reports [Prometheus](https://prometheus.io/) metrics, alerts with [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/), and visualizes in [Grafana](https://grafana.com/). This isn’t theory.

We’ll build a scaled‑down version an indie dev can run in a homelab or on a cheap VPS without yak shaving:
- VPS A: your Rails app, deployed with Kamal, images stored in Harbor.
- VPS B: Harbor registry, Upright node, [OpenTelemetry](https://opentelemetry.io/) Collector, Prometheus, Grafana.

Basic familiarity with [Ruby on Rails](https://rubyonrails.org/) and Docker helps. You’ll end with a reproducible flow that gets you from commit to live without a Rube Goldberg machine. Ship it.

## 1. Why bother going self-hosted now? (And what we’re actually going to build)

AI wrote your boilerplate. Cool. Now what? The gap is everything after git push. The CNCF’s write‑up on API‑first infra calls out the new reality: writing code isn’t the slowest step; provisioning, enforcing policy, and long‑term operations are where you burn hours ([source](https://www.cncf.io/blog/2026/03/20/crossplane-and-ai-the-case-for-api-first-infrastructure/)). For small teams, adding more APIs and controllers can be a footgun. You don’t need a cluster right now; you need a path from build to deploy that doesn’t surprise you at 2 a.m.

Here’s the plan:
- Your own on‑prem, self-hosted registry with [Harbor](https://goharbor.io/), mirroring how 37signals treats their registry as a first‑class piece of infra ([source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/)).
- App deploys via [Kamal](https://kamal-deploy.org/) using [Docker](https://www.docker.com/). Same basic play 37signals runs across their products ([source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/)).
- Monitoring via [Upright](https://dev.37signals.com/introducing-upright/), which runs health checks from multiple locations, exports Prometheus metrics for [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and [Grafana](https://grafana.com/), and shows a world map with 30‑day uptime history.
- A tiny [OpenTelemetry](https://opentelemetry.io/) loop for metrics/logs/traces — vendor‑neutral plumbing you control. Priority, not garnish.

This is informed by real stories: massive S3 data migrations and the emphasis on monitoring a new [Pure FlashBlade](https://dev.37signals.com/pure-storage-monitoring/) system that now holds everything from Basecamp attachments to long‑term Prometheus metrics ([source](https://dev.37signals.com/pure-storage-monitoring/)). Not even close to “toy.”

### What this self-hosted stack gives you

- Tight control over your deployment pipeline.
- Minimal moving parts compared to a full Kubernetes cluster.
- Observability that looks like “real” devops, without needing a platform team.

## 2. Step 1 – Stand up your own self-hosted Docker registry with Harbor on a single VPS

Let me show you what actually works. Start by taking control of your image pipeline. 37signals did, moving off external registries because the registry is integral to their deploys with [Kamal](https://kamal-deploy.org/) and [Docker](https://www.docker.com/). They’d relied on Docker Hub and [Amazon ECR](https://aws.amazon.com/ecr/), and ran into cost and bandwidth pain during their cloud exit and “kamalization” ([source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/)). The solution they chose: [Harbor](https://goharbor.io/) on‑prem.

### What you’ll do at indie scale

What you’ll do at indie scale:
- Provision one VPS for Harbor. Give it a subdomain (like registry.yourdomain). Set up TLS.
- Install Harbor and configure it to store images on local disk or an S3‑compatible store. 37signals back their registry with a Pure FlashBlade S3‑compatible backend and kept the S3 permissions tight rather than a blanket s3:* policy ([source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/)). You can emulate the pattern with any S3‑compatible store you trust.
- Lock it down with basic auth. Keep the credentials in your deploy config.
- Flip your app deploys to push/pull from your Harbor instead of Docker Hub/ECR.

Here’s the thing — this step alone cuts failure modes. No throttling surprises, no random auth changes breaking deploys. You own the registry path.

Using AI here? Totally fine. Paste the relevant Harbor docs into your favorite assistant and ask for a first draft of your config. Then verify the sensitive bits yourself: endpoints, TLS, auth, storage. I learned this the hard way. Endpoint typos and subtle TLS mismatches are silent killers. 37signals hit issues with external registries during their transition; treat your registry as part of prod, not a convenience ([source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/)).

Optional future step: Harbor can replicate to other sites. 37signals ran multiple Harbor nodes fronted by load balancers ([source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/)). You can start single‑node and add a second site later.

Contrarian take: avoid spinning up [Kubernetes](https://kubernetes.io/) and [Crossplane](https://www.crossplane.io/) just to say you have “API‑first infra.” The CNCF post argues that AI shifts the bottleneck and then proposes more APIs and controllers ([source](https://www.cncf.io/blog/2026/03/20/crossplane-and-ai-the-case-for-api-first-infrastructure/)). For a 1–3 person team, that’s adding surface area where you’re already weakest. A VPS, [Harbor](https://goharbor.io/), [Kamal](https://kamal-deploy.org/), and a Makefile is a defensible stack. You’ll ship faster and sleep better.

## 3. Step 2 – Ship a Rails app with Kamal using your new private registry

Your app doesn’t need to change much. The deployment path does. 37signals deploy all of their apps with [Kamal](https://kamal-deploy.org/), using [Docker](https://www.docker.com/) images pushed to a central registry ([source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/)). You’ve got that registry now.

### The Kamal deployment pipeline

The pipeline is straightforward:
- Build your app image locally or in CI.
- Push it to your [Harbor](https://goharbor.io/) registry using credentials you control.
- Have Kamal pull and boot the image on your app VPS.

That’s it. One VPS. One container. One deploy command. If your deploy takes more than 60 seconds, something is wrong.

As your app grows, you can add modern Rails tooling without touching the deploy pipeline:
- Multi‑tenancy? 37signals explored per‑customer [SQLite](https://www.sqlite.org/) for Fizzy to support SaaS and self-hosted options. The tradeoffs got rough near launch, and they unwound the approach days before release ([source](https://dev.37signals.com/fizzy-infrastructure/)). Rails doesn’t make multi‑tenancy easy out of the box, so they open‑sourced [Active Record Tenanted](https://dev.37signals.com/rails-multi-tenancy/) to make it safer and added safeguards to prevent leaks (there’s a RECORDABLES demo converting an app live, too).
- Rich text editing? They shipped [Lexxy](https://dev.37signals.com/announcing-lexxy-a-new-rich-text-editor-for-rails/), a modern editor for Action Text built on Lexical, with proper HTML semantics, Markdown support (shortcuts and auto‑format on paste), and real‑time code syntax highlighting.
- Mobile push? They open‑sourced [Action Push Native](https://dev.37signals.com/introducing-action-push-native/), a Rails gem for APNs and Google’s push service, created to migrate off Amazon SNS and Pinpoint. It’s used in Basecamp and HEY to send more than 10 million push notifications per day ([source](https://dev.37signals.com/introducing-action-push-native/)).

The lesson: you can evolve the app internals while the deploy workflow stays boring. Boring is good.

Using AI here is a force multiplier. Have it draft an initial Kamal config, then check the big rocks yourself: image tags, ports, environment, health endpoints. AI helps, but you still need to understand what the code does.

## 4. Step 3 – Add Upright synthetic monitoring on a cheap VPS (Prometheus + Grafana included)

Black‑box SaaS monitoring is fine. Owning it is better. 37signals open‑sourced [Upright](https://dev.37signals.com/introducing-upright/), a Rails engine that runs health checks from multiple geographic locations and ships metrics to [Prometheus](https://prometheus.io/) for alerting with [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and graphing in [Grafana](https://grafana.com/). It’s deployed to inexpensive VPS nodes using [Kamal](https://kamal-deploy.org/). The UI gives you a world map, a 30‑day uptime history, and probe status views across all sites ([source](https://dev.37signals.com/introducing-upright/)).

### Upright monitoring layout

Here’s the pattern:
- Spin up a second VPS for monitoring. Keep app and monitoring separate; helps during incidents.
- Create a Rails app and mount the [Upright](https://dev.37signals.com/introducing-upright/) engine. The install generator lays a lot of groundwork: configs for Prometheus and Alertmanager, an OpenTelemetry Collector config, probe directories, Kamal templates, and Docker Compose files out of the box ([source](https://dev.37signals.com/introducing-upright/)).
- Define a simple HTTP probe for your app’s root path and a health endpoint. Upright runs checks and emits Prometheus metrics.
- Wire Grafana to display the uptime trend. Upright’s overview makes it easy to see green vs. red over 30 days, even with a single node at first.

Deploy Upright itself with [Kamal](https://kamal-deploy.org/), same pattern as your app. Shipping a monitoring node with the same tool keeps your muscle memory tight. One way to scale later is to add more VPS nodes in other regions; Upright was designed to run probes from multiple locations ([source](https://dev.37signals.com/introducing-upright/)). Start small. Add geography when you need it.

Why metrics at all? Because you can’t fix what you can’t see. Even CNCF’s primer on Kubernetes metrics says metrics are necessary to manage clusters, nodes, and applications — without them it’s harder to find problems and improve performance ([source](https://www.cncf.io/blog/2026/03/18/understanding-kubernetes-metrics-best-practices-for-effective-monitoring/)). Same energy here, no Kubernetes required.

## 5. Step 4 – Wire up OpenTelemetry for app + infra metrics with a tiny Collector config

Observability shouldn’t lock you into one agent or vendor. [OpenTelemetry](https://opentelemetry.io/) is the common layer for metrics, logs, and traces across apps and services ([source](https://opentelemetry.io/blog/2026/alternative-approaches-to-contributing/)). Good news: key pieces of OTel’s declarative configuration spec are now stable — JSON schema, YAML representation, in‑memory model, plugin provider, and SDK operations for parsing YAML and instantiating components ([source](https://opentelemetry.io/blog/2026/stable-declarative-config/)). Perfect for a small, maintainable Collector config.

### A minimal OpenTelemetry Collector flow

Aim for this flow:
- Your Rails app and Upright send OTLP to a local Collector receiver.
- The Collector also scrapes Prometheus metrics exposed by services.
- A filter stage trims noise using the Filter Processor’s OTTL context inference — you can write readable conditions without memorizing internal telemetry contexts ([source](https://opentelemetry.io/blog/2026/ottl-context-inference-come-to-filterprocessor/)).
- Export locally to Prometheus for Grafana and, if you want, to any OTLP backend. Stick to OTLP. [Zipkin exporters are being deprecated](https://opentelemetry.io/blog/2025/deprecating-zipkin-exporters/) in favor of Zipkin’s OTLP ingestion support since the community has gravitated to OTLP.

Two practical choices here:
- Emit events as logs correlated with spans instead of span events. The Span Event API is being deprecated to remove confusion with log‑based events; existing span‑event views keep working for now, but new code should go the log route ([source](https://opentelemetry.io/blog/2026/deprecating-span-events/)).
- If you eventually move to [Kubernetes](https://kubernetes.io/), there’s active work on Kubernetes semantic conventions moving toward stability, used by processors like k8sattributes and resourcedetection, and available behind feature gates for early feedback ([source](https://opentelemetry.io/blog/2026/k8s-semconv-rc/)). Not needed today. Keep your stack boring until you outgrow it.

Example filters to keep configs readable:
- Drop routine healthcheck noise.
- Keep only HTTP request metrics with 5xx status codes for your alerting path.

This mirrors what the OpenTelemetry Developer Experience SIG heard: teams want real‑world examples of SDK and Collector usage, not just theory — they kicked off an end‑user series starting with Mastodon ([source](https://opentelemetry.io/blog/2026/devex-mastodon/)). Small stack. Production habits.

And if you want proof this path scales? There’s a KubeCon EU talk titled “We Deleted Our Observability Stack and Rebuilt It With OTel: 12 Engineers to 4 at 20K+ Clusters,” which shows large‑scale adoption of the same core ideas ([source](https://opentelemetry.io/blog/2026/kubecon-eu/)). You won’t run 20K clusters. You don’t need to.

## 6. Where to go next – from indie VPS to API‑first infra and contributions

At this point you’ve got:
- A self-hosted [Harbor](https://goharbor.io/) registry.
- A Rails app shipped with [Kamal](https://kamal-deploy.org/).
- [Upright](https://dev.37signals.com/introducing-upright/) synthetic monitoring with [Prometheus](https://prometheus.io/), [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/), and [Grafana](https://grafana.com/).
- A small [OpenTelemetry](https://opentelemetry.io/) loop for metrics/logs/traces.

### Growing from VPS to clusters

Next steps if you grow:
- Add a second Harbor node and explore replication. 37signals ran multiple Harbor instances fronted by load balancers as their setup matured ([source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/)).
- Add more Upright probe sites to get multi‑region checks and the world‑map view across locations ([source](https://dev.37signals.com/introducing-upright/)).
- If you eventually adopt [Kubernetes](https://kubernetes.io/), bring in policy with [Kyverno](https://kyverno.io/) — the CNCF highlights how cluster scale and complexity create governance challenges ([source](https://www.cncf.io/blog/2026/03/19/policy-as-code-flexible-kubernetes-governance-with-kyverno/)) — and apply the Kubernetes metrics best practices you’ve already tasted at small scale ([source](https://www.cncf.io/blog/2026/03/18/understanding-kubernetes-metrics-best-practices-for-effective-monitoring/)).

Circling back to the API‑first infra conversation: once you’re comfortable, you can keep this exact self-hosted stack and drive provisioning via APIs so AI agents or CI can spin up environments after git push ([source](https://www.cncf.io/blog/2026/03/20/crossplane-and-ai-the-case-for-api-first-infrastructure/)). Don’t start there. Grow into it.

And contribute your findings back — configs, docs, bug reports. The OTel community managers invite you to connect in the CNCF Slack #opentelemetry channel and are working on a simple handle to reach them ([source](https://opentelemetry.io/blog/2026/hello-from-community-managers/)). The OpenTelemetry team also argues for contributions beyond “good first issues” — production stories like yours matter ([source](https://opentelemetry.io/blog/2026/alternative-approaches-to-contributing/)).

Here’s the punchline: you don’t need petabytes or a giant team to benefit from this. It’s sized for one indie or a tiny crew who want a simple, boring devops backbone.

## Cheat Sheet

- Kamal — container deploys for apps
  - https://kamal-deploy.org/
  - 37signals deploy all apps with Kamal and Docker; the registry is a critical pipeline piece ([source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/)).
- Harbor — on‑prem Docker registry
  - https://goharbor.io/
  - 37signals moved off Docker Hub/ECR to Harbor during their cloud exit and kamalization ([source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/)).
- Upright — synthetic monitoring Rails engine
  - https://dev.37signals.com/introducing-upright/
  - Runs checks from multiple locations, exports Prometheus metrics, alerts with Alertmanager, graphs in Grafana; UI includes world map and 30‑day history.
- Prometheus — metrics collection
  - https://prometheus.io/
  - Upright exports Prometheus metrics for alerting and visualization ([source](https://dev.37signals.com/introducing-upright/)).
- Alertmanager — alerting for Prometheus
  - https://prometheus.io/docs/alerting/latest/alertmanager/
- Grafana — dashboards
  - https://grafana.com/
- OpenTelemetry — vendor‑neutral telemetry
  - https://opentelemetry.io/
  - Tools and standards for metrics/logs/traces ([source](https://opentelemetry.io/blog/2026/alternative-approaches-to-contributing/)).
  - Declarative config spec has stable components (schema, YAML, in‑memory model, plugins, SDK parsing) ([source](https://opentelemetry.io/blog/2026/stable-declarative-config/)).
  - Filter Processor supports context‑inference fields: trace_conditions, metric_conditions, log_conditions, profile_conditions ([source](https://opentelemetry.io/blog/2026/ottl-context-inference-come-to-filterprocessor/)).
  - Span Event API deprecation: emit events as logs correlated with spans ([source](https://opentelemetry.io/blog/2026/deprecating-span-events/)).
  - Zipkin exporter spec deprecation in favor of OTLP ingestion ([source](https://opentelemetry.io/blog/2025/deprecating-zipkin-exporters/)).
- Rails multi‑tenancy — Active Record Tenanted
  - https://dev.37signals.com/rails-multi-tenancy/
  - Helps make multi‑tenanted Rails apps safer; RECORDABLES demo shows conversion and safeguards.
- Lexxy — modern Action Text editor for Rails
  - https://dev.37signals.com/announcing-lexxy-a-new-rich-text-editor-for-rails/
  - Based on Lexical; better semantics, Markdown shortcuts and auto‑format on paste, real‑time code syntax highlighting.
- Action Push Native — Rails push notifications
  - https://dev.37signals.com/introducing-action-push-native/
  - APNs and Google push; built to migrate off SNS/Pinpoint; used in Basecamp and HEY to send more than 10 million push notifications/day.
- Hotwire Native v1.2 — web‑first native app framework
  - https://dev.37signals.com/announcing-hotwire-native-v1-2/
  - Big update with API consistency and route decision handlers.
- S3 migration — 37signals story
  - https://dev.37signals.com/moving-mountains-of-data-off-s3/
  - Moved massive data (billions of files) off S3; bandwidth limits, AWS constraints, custom tooling, verification, and the anxiety of deletion.
- Pure Storage FlashBlade — storage target and monitoring priority
  - https://dev.37signals.com/pure-storage-monitoring/
  - Holding 10 PB of data, including Basecamp attachments, Prometheus long‑term metrics, and database backups; observability is a top priority.

## Closing thoughts

You just stitched together a self-hosted delivery and observability path that looks like a mini version of what 37signals runs for Basecamp and HEY — except it fits on a couple of VPS nodes and a weekend. The hard part after git push doesn’t have to live in opaque SaaS tools. With [Kamal](https://kamal-deploy.org/), [Harbor](https://goharbor.io/), [Upright](https://dev.37signals.com/introducing-upright/), and [OpenTelemetry](https://opentelemetry.io/), you control everything from container build to alert.

Iterate. Add more probe sites. Tighten OTTL filters. Experiment with multi‑tenancy via [Active Record Tenanted](https://dev.37signals.com/rails-multi-tenancy/) or ship a better editor with [Lexxy](https://dev.37signals.com/announcing-lexxy-a-new-rich-text-editor-for-rails/). When you’re ready, describe the same setup declaratively so AI or your CI can spin it up on demand ([source](https://www.cncf.io/blog/2026/03/20/crossplane-and-ai-the-case-for-api-first-infrastructure/)). The point isn’t to clone someone else’s stack. It’s to have a simple, reproducible production backbone you actually understand — and can grow with.

## FAQ

### Can I run this self-hosted stack on a homelab instead of a VPS?

Yes. The whole point is to keep the footprint tiny. If you’ve got a reliable homelab with decent disks and a static IP (or a dynamic DNS setup), you can run Harbor, Kamal targets, Upright, and the OpenTelemetry Collector there instead of on a commercial VPS. Just treat it like prod: backups, TLS, and power/ISP redundancy where you can.

### How does this compare to using a PaaS like Coolify or other platforms?

A self-hosted Kamal + Harbor setup gives you explicit control over images, networking, and observability. A PaaS like Coolify is great if you want more of a point‑and‑click experience and don’t mind someone else owning more of the deployment surface. This stack is closer to bare metal, but still light enough for an indie dev who wants to understand the moving parts.

### Is this overkill for a solo indie dev or small team?

No — it’s sized exactly for that. One box for the app, one for infra, a clear deployment story, and observability you can actually read. It’s more work than a single Heroku‑style app, but far less ceremony than a full Kubernetes cluster. If you ever outgrow it, the habits you build here will transfer cleanly into whatever comes next for your self-hosted setup.

