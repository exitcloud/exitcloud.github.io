---
title: "Own the Stack, Not the Bill: Self‑Hosted Dev, AI Agents, and Observability on a Single VPS"
date: 2026-03-14T14:08:01.557Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# Own the Stack, Not the Bill: Self‑Hosted Dev, AI Agents, and Observability on a Single VPS

Here’s the thing — you don’t need Kubernetes for a six-service indie app. You need a VPS, a Makefile, and a couple of battle‑tested tools. Pair that with an AI assistant that can actually touch your files and run commands, not just autocomplete. Ship it.

This walkthrough shows a tiny but serious setup:
- A private container registry with Harbor, feeding fast Kamal deploys.
- Synthetic monitoring nodes with Upright, plus Prometheus/Alertmanager and Grafana.
- An OpenTelemetry Collector running a clean, declarative pipeline (filters and unrolling included).
- An AI dev workflow in VS Code that’s agentic: it edits files, runs the terminal, follows stack traces.
- Optional: self‑hosted push notifications and Rails multi‑tenancy patterns that scale to both SaaS and self‑hosted customers.

No magic. No yak shaving. Just a modern stack that a small team can actually operate.

## 1. From Cloud Rent to Small Metal: What We’re Building and Why

If you want permission to run “serious” stuff on a couple of boxes, look at what [37signals](https://dev.37signals.com/) has been doing. They moved “massive amounts of data” — billions of files — off AWS S3 with no downtime, and they’re moving approximately [10 petabytes out of S3](https://dev.37signals.com/pure-storage-monitoring/) onto [Pure Storage FlashBlade](https://dev.37signals.com/pure-storage-monitoring/). Big move. They also run their own [on‑prem Docker registry using Harbor](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/) and deploy all apps with [Kamal](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/) on top of [Docker](https://www.docker.com/). Different scale than us, same pattern: own the stack.

Monitoring? They open‑sourced [Upright](https://dev.37signals.com/introducing-upright/), a synthetic monitoring system. Each Upright node runs probes from different regions and exports [Prometheus](https://prometheus.io/) metrics, so you can alert via [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and visualize in [Grafana](https://grafana.com/). Simple, effective.

For app telemetry, we’ll use [OpenTelemetry](https://opentelemetry.io/) with the new stable [declarative configuration](https://opentelemetry.io/blog/2026/stable-declarative-config/) — a proper JSON schema (opentelemetry-configuration 1.0.0), YAML representation, and in‑memory model. You get a clean YAML file that describes your whole collector pipeline. The Filter Processor now has OTTL context inference (hello, fewer footguns), and there’s a dedicated Unroll Processor for bundled logs. That directly addresses the classic “noisy unstructured logs and siloed data” mess in traditional/on‑prem environments called out in [Demystifying OpenTelemetry](https://opentelemetry.io/blog/2026/demystifying-opentelemetry/). Perfect fit for a small server.

We’ll wire all of this on a single VPS to start, then split out Upright to a second one later. Keep it boring. Keep it fast.

## 2. Turn VS Code into an Agentic Workspace (Not Just Fancy Autocomplete)

I got tired of copilots that do one thing: guess the next line. The author of [Glass Arc](https://news.ycombinator.com/item?id=47376342) had the same pain — paying for an “AI” that mainly acted as a strong autocomplete that waits for input, guesses the next block, and stops. Software engineering isn’t just syntax. It’s managing the filesystem, running terminal commands, and debugging stack traces. Glass Arc lives inside [VS Code](https://code.visualstudio.com/) and gives an assistant controlled access to your local environment. Safely. That’s the whole point.

Even better, it supports an “Agentic Execution” mode: give it an intent and it drafts the action sequence. Not just line‑by‑line code completion. You say: “Add an Upright health probe and a Prometheus scrape target; commit on a new branch.” It proposes steps. You approve. It runs. Real work gets done.

How do you keep that power under control? Treat your agent like code. [GitAgent](https://www.gitagent.sh/) defines an agent as files in a Git repo:
- agent.yaml — configuration
- SOUL.md — personality/instructions
- SKILL.md — capabilities

It’s “git‑native,” which means you version the behavior, branch it, review it, and roll it back like any other change. And it’s portable: GitAgent agents can be exported to [Claude Code](https://www.gitagent.sh/), [OpenAI Agents SDK](https://www.gitagent.sh/), [CrewAI](https://www.gitagent.sh/), [Google ADK](https://www.gitagent.sh/), and [LangChain](https://www.gitagent.sh/). No more framework lock‑in. No more rewrites every time the industry pivots.

Start tiny. Create a minimal agent repo:
- agent.yaml: repo path, allowed directories, tools it can invoke (git, shell, test runner).
- SOUL.md: “You are a cautious assistant. You read errors before retrying. You create branches for risky changes.”
- SKILL.md: “Run test suites, edit YAML, modify Dockerfiles, write Rails migrations.”

Then connect it in Glass Arc. Give it tasks one by one — the [Inscribe](https://rahuldshetty.github.io/inscribe/) playbook. Inscribe’s author built a static‑site generator over a few days with AI help, feature by feature, starting from a blank project. That incremental approach bought customization, testing, and deeper understanding — not a one‑shot code dump. Same idea here: ask your agent to add a Prometheus scrape job today; tomorrow, let it refactor a command into your Makefile. Small steps. Tight loops.

Guardrails matter. If the agent writes database code, look at patterns like [lean‑pq](https://github.com/typednotes/lean-pq). It’s a PostgreSQL connector for [Lean 4](https://github.com/leanprover/lean4) that does compile‑time SQL validation against your schema via macros. Misspell a column? Type mismatch? Build fails. The author also argues Lean’s dependent types and formal verification features make it an ideal target for AI agents that need to generate provably correct code. That mindset — push as many errors left as possible — is exactly how you keep AI from quietly wrecking prod.

And for old‑school editing? Even Vim can get a little AI assist without becoming a circus. [vim‑gramaculate](https://github.com/ahalbert/vim-gramaculate) is a grammar checker that stays out of your way. Great for commit messages and docs cleanup.

Contrarian take: Kubernetes is the wrong default at this scale. A [CNCF post](https://www.cncf.io/blog/2026/03/05/the-great-migration-why-every-ai-platform-is-converging-on-kubernetes/) argues AI platforms are converging on Kubernetes, and I get why at huge scale. But for a handful of services, it’s yak‑shaving. An agent defined in Git (GitAgent), operating locally in [VS Code](https://code.visualstudio.com/) via [Glass Arc](https://news.ycombinator.com/item?id=47376342), shipping with [Kamal](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/) and [Docker](https://www.docker.com/) to a VPS — that gives you most of the power with a fraction of the operational surface. Not even close.

## 3. Roll Your Own Docker Registry with Harbor and Deploy via Kamal

Why bother with a private registry as a team of one? Control, speed, and fewer surprises. Pulls don’t depend on a third‑party throttling you, and images live where your servers live. [37signals runs Harbor on‑prem](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/) after years of being tightly coupled to external registries like Docker Hub and Amazon ECR. They did it to own the pipeline and address costs tied to licenses and pull/push operations. Sensible.

Minimal setup you can adapt:

- Install [Harbor](https://goharbor.io/) on a small VM. Use the official installer or containerized deployment.
- Configure a hostname and storage; enable metrics for visibility with [Prometheus](https://prometheus.io/) and [Grafana](https://grafana.com/).
- Create a project for your app images.

Log in from your laptop or CI:
```
docker login harbor.yourdomain.com
# push after your build:
docker tag yourapp:latest harbor.yourdomain.com/yourproj/yourapp:latest
docker push harbor.yourdomain.com/yourproj/yourapp:latest
```

Wire it to [Kamal](https://kamal-deploy.org/) so deploys build and push to Harbor, then pull on the server. A minimal Kamal deploy config conceptually looks like this (adapt names to your app):

```
# config/deploy.yml
service: yourapp
image: harbor.yourdomain.com/yourproj/yourapp
servers:
  - your-vps-1.yourdomain.com
registry:
  server: harbor.yourdomain.com
  username: $HARBOR_USER
  password: $HARBOR_PASS
env:
  clear:
    RAILS_ENV: production
  secret:
    - DATABASE_URL
    - SECRET_KEY_BASE
```

Then ship:
```
kamal build
kamal push
kamal deploy
```

You’ll feel the difference on cold starts. Local pulls from your Harbor are simply closer and more predictable than hauling layers over the public internet. And if you ever decide to bring in Kubernetes later, you’re not painted into a corner — the [CNCF](https://www.cncf.io/blog/2026/03/09/registry-mirror-authentication-with-kubernetes-secrets/) has patterns for registry mirror authentication with Kubernetes Secrets, and vendors provide credential helpers. Harbor fits right into that world too.

Here’s the bigger pattern I keep coming back to:
- [Glass Arc](https://news.ycombinator.com/item?id=47376342): local agent with real powers (shell, filesystem).
- [GitAgent](https://www.gitagent.sh/): agent behavior versioned in Git.
- [lean‑pq](https://github.com/typednotes/lean-pq): type‑level safety for high‑blast‑radius code like SQL.

That triangle is strong. Your AI isn’t just a chat box; it’s a controlled process whose behavior you review and roll back, fenced by compile‑time checks where it matters.

## 4. DIY Synthetic Monitoring + Observability: Upright, Prometheus, Grafana, and OpenTelemetry

Monitoring that actually catches outages? Start with synthetic checks from the outside. [Upright](https://dev.37signals.com/introducing-upright/) is a Rails engine 37signals deploys to cheap VPS nodes around the world with [Kamal](https://dev.37signals.com/introducing-upright/). Each node runs probes against your services from different regions and exports [Prometheus](https://prometheus.io/) metrics. Alert with [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/). Plot in [Grafana](https://grafana.com/). It’s exactly what you want — “is the site actually up” from outside the firewall.

Practical flow:
- Stand up an Upright instance (a Rails app including the engine) and define a couple of probes that hit your app’s health endpoint and a login flow.
- Deploy that Upright node to a second VPS with [Kamal](https://kamal-deploy.org/).
- Feed Upright’s emitted metrics into Prometheus and set an “/health failed” alert.

Now layer in app telemetry with [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/). The declarative config is stable now — JSON schema, YAML files, and an in‑memory model — which means you describe your pipeline like code. Cleaner diffs, fewer “what is this component doing” moments. The new Filter Processor context inference (available since collector‑contrib v0.146.0) lets you write OTTL conditions without memorizing internal telemetry contexts; just use top‑level fields for traces, metrics, logs, and profiles. And the Unroll Processor turns “bundled” log records (like a JSON array of many events) into individual log records — perfect when some third‑party spits out noisy packets.

Here’s an example collector config you can run on the same VPS as your app (trim as needed):

```yaml
# otel-collector.yaml
receivers:
  otlp:
    protocols:
      http:
      grpc:
  prometheus:
    config:
      scrape_configs:
        - job_name: 'upright-nodes'
          static_configs:
            - targets: ['upright-node-1.yourdomain.com:PORT']
processors:
  batch: {}
  filter:
    logs:
      log_conditions:
        # Drop your own health checks from app logs
        - 'attributes["http.target"] == "/health"'
  unroll: {} # enable the Unroll Processor for bundled logs
exporters:
  prometheus:
    endpoint: "0.0.0.0:9464"
  logging:
    loglevel: info

service:
  pipelines:
    metrics:
      receivers: [prometheus, otlp]
      processors: [batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [filter, unroll, batch]
      exporters: [logging]
```

Notes:
- The YAML is declarative per [OpenTelemetry’s stable config](https://opentelemetry.io/blog/2026/stable-declarative-config/).
- The filter uses the Filter Processor’s context inference released to the Filter Processor (via `log_conditions`) as described [here](https://opentelemetry.io/blog/2026/ottl-context-inference-come-to-filterprocessor/).
- The Unroll Processor is documented [here](https://opentelemetry.io/blog/2025/contrib-unroll-processor/).

Scrape the collector’s Prometheus exporter in your Grafana datasource. Build a tiny dashboard:
- Uptime panel (from Upright metrics).
- Latency histogram (from app metrics if you export them via OTLP).
- Logs panel fed by the collector’s processed logs (or send logs to a backend of your choice).

Alerts:
- An Upright “HTTP probe failed” tied to Alertmanager.
- A Prometheus rule for error rate or tail latency.

Yes, this is the same bag of tricks folks use at scale. You’ll see it center stage at community events like [Observability Day at KubeCon + CloudNativeCon Europe 2026](https://www.cncf.io/blog/2026/03/13/kubecon-cloudnativecon-europe-2026-co-located-event-deep-dive-observability-day/). And there’s even a scheduled talk — “[We Deleted Our Observability Stack and Rebuilt It With OTel: 12 Engineers to 4 at 20K+ Clusters](https://opentelemetry.io/blog/2026/kubecon-eu/)”. We’re just shrinking the playbook down to a single box without the ceremony.

If your current stack lives in a traditional/on‑prem corner with unstructured logs and siloed dashboards, [Demystifying OpenTelemetry](https://opentelemetry.io/blog/2026/demystifying-opentelemetry/) sums up the pain. The collector, a few processors, and a consistent config file go a long way. You can actually find things again.

## 5. Self‑Hosted Push and Multi‑Tenancy: Shipping a SaaS+Self‑Host Hybrid from One Box

You’ve got a Rails app. Some customers want it hosted. Some want to run it themselves. You also need to send mobile push notifications without outsourcing it to a cloud messaging service. Totally doable.

For push, 37signals open‑sourced [Action Push Native](https://dev.37signals.com/introducing-action-push-native/), a Rails gem for sending push notifications to mobile platforms, supporting both Apple and Google push services. They built it to migrate off Amazon SNS and Pinpoint. It uses HTTP/2 persistent connections to APNs, and it’s handling more than 10 million pushes per day. That’s not hobby‑scale. That’s battle‑tested.

For multi‑tenancy, they explored giving every customer their own [SQLite](https://sqlite.org/) database for the [Fizzy](https://dev.37signals.com/fizzy-infrastructure/) product — aiming to support both self‑hosted and SaaS with stronger isolation — and later unwound that design before release when the tradeoffs piled up. They also released [Active Record Tenanted](https://dev.37signals.com/rails-multi-tenancy/), a gem to make Rails multi‑tenancy more seamless, including converting existing apps and adding safeguards against accidental data leaks. That last part is key. Leaks are the nightmare scenario. Guardrails help.

How to adopt this on a single VPS:
- Start with one tenant (your SaaS) and keep code tenant‑aware from day one using Active Record Tenanted. You can keep running SQLite for self‑hosted customers if you want local, file‑based DBs per instance, while your SaaS runs a separate database — same codebase.
- Add [Action Push Native](https://dev.37signals.com/introducing-action-push-native/) to send push notifications without relying on managed messaging. Configure Apple and Google credentials; keep secrets outside the repo.

37signals’ cloud exit work underlines a point: they replaced managed services with first‑class components they own — storage via [Pure Storage FlashBlade](https://dev.37signals.com/pure-storage-monitoring/), messaging via [Action Push Native](https://dev.37signals.com/introducing-action-push-native/), registry via [Harbor](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/). You can follow that path at tiny scale. Gradually. No need to flip the table.

## A tiny Makefile to tie it together

Let me show you what actually works. A Makefile that keeps muscle memory simple:

```
# Makefile
REGISTRY=harbor.yourdomain.com/yourproj
IMAGE=$(REGISTRY)/yourapp
TAG?=latest

build:
\tdocker build -t $(IMAGE):$(TAG) .

push:
\tdocker push $(IMAGE):$(TAG)

deploy:
\tkamal deploy

logs:
\tkamal app logs -f

health:
\tcurl -fsS https://yourapp.yourdomain.com/health || exit 1
```

One glance. One minute. Ship it.

## Quick Reference

- [Glass Arc](https://news.ycombinator.com/item?id=47376342) — Agentic workspace inside VS Code with controlled access to filesystem/terminal; supports “Agentic Execution” intents.
- [Visual Studio Code](https://code.visualstudio.com/) — Editor host for agentic workflows like Glass Arc.
- [GitAgent](https://www.gitagent.sh/) — Git‑native spec for AI agents (agent.yaml, SOUL.md, SKILL.md); export to multiple frameworks.
- [Inscribe](https://rahuldshetty.github.io/inscribe/) — Static‑site generator built incrementally with AI help; pattern for feature‑by‑feature AI development.
- [lean‑pq](https://github.com/typednotes/lean-pq) — Lean 4 PostgreSQL connector with compile‑time SQL validation; argues for provably correct AI‑generated code.
- [vim‑gramaculate](https://github.com/ahalbert/vim-gramaculate) — Vim grammar checker powered by AI; handy for commit messages/docs.
- [Harbor](https://goharbor.io/) — Private container registry; 37signals runs their registry on‑prem with Harbor.
- [Kamal](https://kamal-deploy.org/) — Container deploy tool used by 37signals across apps; works with Docker and on‑prem registries.
- [Docker](https://www.docker.com/) — Container runtime/platform underpinning Kamal deploys.
- [Upright](https://dev.37signals.com/introducing-upright/) — Synthetic monitoring Rails engine; runs probes from multiple locations; exports Prometheus metrics; alerts via Alertmanager; visualize in Grafana.
- [Prometheus](https://prometheus.io/) — Metrics collection; scrape Upright nodes and app metrics.
- [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) — Alerting for Prometheus metrics, including Upright probe results.
- [Grafana](https://grafana.com/) — Visualization for Prometheus metrics/dashboards.
- [OpenTelemetry Declarative Config](https://opentelemetry.io/blog/2026/stable-declarative-config/) — Stable JSON schema, YAML, in‑memory model; configure collectors as files.
- [OTel Filter Processor (context inference)](https://opentelemetry.io/blog/2026/ottl-context-inference-come-to-filterprocessor/) — Use trace_conditions, metric_conditions, log_conditions, profile_conditions without knowing internals.
- [OTel Unroll Processor](https://opentelemetry.io/blog/2025/contrib-unroll-processor/) — Split bundled log records (e.g., arrays) into individual log events.
- [Demystifying OpenTelemetry](https://opentelemetry.io/blog/2026/demystifying-opentelemetry/) — Challenges in traditional/on‑prem setups; how OTel helps.
- [Observability Day at KubeCon EU 2026](https://www.cncf.io/blog/2026/03/13/kubecon-cloudnativecon-europe-2026-co-located-event-deep-dive-observability-day/) — Community focus on OTel/Prometheus patterns we’re using here.
- [“We Deleted Our Observability Stack and Rebuilt It With OTel” talk](https://opentelemetry.io/blog/2026/kubecon-eu/) — Evidence of consolidation on OTel at large scale.
- [Action Push Native](https://dev.37signals.com/introducing-action-push-native/) — Rails gem for Apple/Google push; built to exit SNS/Pinpoint; >10M pushes/day in production.
- [Active Record Tenanted](https://dev.37signals.com/rails-multi-tenancy/) — Rails multi‑tenancy gem; assists conversion and adds leak safeguards.
- [Fizzy infrastructure reflection](https://dev.37signals.com/fizzy-infrastructure/) — SQLite‑per‑customer exploration and tradeoffs; unwind before launch.
- [37signals S3 cloud exit story](https://dev.37signals.com/moving-mountains-of-data-off-s3/) — Moving billions of files off S3 with no downtime; bandwidth constraints; custom tooling.
- [Pure Storage FlashBlade at 37signals](https://dev.37signals.com/pure-storage-monitoring/) — Replacement storage for ~10 PB; top observability priority.

## Wrap

Own your stack in layers. Start with Harbor + Kamal so deploys are fast and under your control. Add an Upright node and some Prometheus/Grafana to catch outages from the outside. Drop in an OpenTelemetry Collector with declarative config to tame logs and metrics. Then graduate to an AI workflow that actually operates on your code — [Glass Arc](https://news.ycombinator.com/item?id=47376342) inside [VS Code](https://code.visualstudio.com/), agent defined in Git via [GitAgent](https://www.gitagent.sh/), guarded by tests and, where possible, compile‑time guarantees like [lean‑pq](https://github.com/typednotes/lean-pq).

You don’t need a platform engineering team. You need a Makefile and a couple of boxes. The rest is discipline — version the behavior (including your agent), keep configs as code, and keep shipping.
