---
title: "From Cloud-Dependent to Indie-Self-Hosted: Deploys, Observability, and AI-Boosted Dev on Your Own Metal"
date: 2026-03-13T14:06:00.294Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# From Cloud-Dependent to Indie-Self-Hosted: Deploys, Observability, and AI-Boosted Dev on Your Own Metal

I deleted the yak shaving. Kept the parts that ship.

Here’s the thing — you can run a serious app on a couple of VPS boxes with clear, boring pieces: a container registry you control, a dead-simple deploy tool, cheap synthetic monitoring, and an observability pipeline that doesn’t fight you. Then you wire AI into your editor and scripts like a power tool, not a mystery box.

Let me show you what actually works.

Step 0: The Indie Stack We’re Building (and Why)

- Deploys: [Kamal](https://kamal-deploy.org/) + [Docker](https://www.docker.com/). This is the playbook [37signals uses for all their apps](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/).
- Registry: self-hosted [Harbor](https://goharbor.io/), instead of DockerHub/ECR — a move 37signals made during their cloud exit to gain control over cost, transfer, and storage for image pulls and pushes [source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/).
- Monitoring: [Upright](https://dev.37signals.com/introducing-upright/) nodes poking your app from the outside, emitting [Prometheus](https://prometheus.io/) metrics, alerts via [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/), and dashboards in [Grafana](https://grafana.com/).
- Observability: [OpenTelemetry](https://opentelemetry.io/) Collector with its now-stable declarative configuration model [source](https://opentelemetry.io/blog/2026/stable-declarative-config/). Logs, metrics, traces. One pipe.
- Dev workflow: [Neovim](https://neovim.io/) with [chat.nvim](https://github.com/wsdjeg/chat.nvim/releases/tag/v1.4.0) as your AI hub, or try the “no learning curve” terminal editor [PNANA](https://github.com/Cyxuan0311/PNANA) if Vim isn’t your thing.

Hardware footprint? Small. Think: 1–2 VPS nodes for your app and registry, 1 small VPS for Upright probes, and your laptop.

Why not stick with cloud registries and hosted monitoring? [37signals cites cost (licenses, transfer, storage)](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/) and control as reasons to bring the registry home. And [Upright is designed to run as a Rails engine on cheap VPS nodes using Kamal](https://dev.37signals.com/introducing-upright/) — no mystery service required. You own the loop.

Preview of the AI workflow: [chat.nvim](https://github.com/wsdjeg/chat.nvim/releases/tag/v1.4.0) turns Neovim into an AI hub that can chat with multiple providers (OpenAI, [Gemini](https://ai.google/), [Anthropic](https://www.anthropic.com/), [Ollama](https://ollama.com/)), run tools (web search, file search, git diff), keep long-term memory, and even bridge Discord/Telegram/Lark/WeCom/DingTalk. And if you want a small agent, the [build-your-own-openclaw](https://github.com/czl9707/build-your-own-openclaw) steps plus a guardrail like [DashClaw](https://github.com/ucsandman/DashClaw) get you something controllable. Ship it.

Step 1: Self-Host Your Container Registry with Harbor and Wire It into Kamal

You don’t need a cloud registry. [37signals runs their Docker registry on-prem using Harbor](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/) and deploys with [Kamal](https://kamal-deploy.org/) as a universal tool. That’s the path: a single Harbor instance you control, Kamal pushing/pulling images there, and your app nodes pulling locally.

High-level plan:

- Stand up [Harbor](https://goharbor.io/) on a VPS. Keep it simple to start — single-node is fine for indies. 
- Enable auth and serve it behind TLS. You can run Harbor on a hostname you control and lock it down.
- Point your Kamal deploy config at your Harbor registry. Now your build and deploy pipeline doesn’t depend on DockerHub/ECR.

Why this beats Kubernetes for a small team

Yes, the [CNCF blog says “every AI platform is converging on Kubernetes”](https://www.cncf.io/blog/2026/03/05/the-great-migration-why-every-ai-platform-is-converging-on-kubernetes/). True at a certain scale. But here’s the catch for 1–5 devs: production Kubernetes clusters do a staggering amount of pull/push/auth churn with private registries — [thousands of pulls per day](https://www.cncf.io/blog/2026/03/09/registry-mirror-authentication-with-kubernetes-secrets/) — and ship credential providers for different vendors. That’s before you even debug your app.

A Kamal + Harbor pipeline is boring: a few Docker hosts, one registry you own, and a basic deploy command. And [37signals runs Basecamp/HEY on Kamal with Docker](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/). If the team behind those products can do it without Kubernetes, your indie app can too.

Reality check: If you do grow into heavy AI workloads later, you can revisit orchestration. Until then — please don’t turn your weekend app into a cluster science project.

Step 2: Add Synthetic Monitoring with Upright on Cheap VPS Nodes

The best monitoring I’ve ever used? A curl in a cron job. Next best: a minimal synthetic monitor that runs checks where your users are and tells you when the real thing breaks.

[Upright](https://dev.37signals.com/introducing-upright/) is built for exactly this. It’s a [Rails](https://rubyonrails.org/) engine that you deploy to cheap VPS nodes with [Kamal](https://kamal-deploy.org/). Each node runs probes against your services and exposes [Prometheus](https://prometheus.io/) metrics, which you can alert on via [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and visualize in [Grafana](https://grafana.com/). Lightweight. Repeatable.

What to do:

- Deploy an Upright node (or two) with [Kamal](https://kamal-deploy.org/). That’s the reference flow described by 37signals — Upright on VPS nodes, deployed the same way you deploy your app [source](https://dev.37signals.com/introducing-upright/).
- Define a few HTTP probes for your app endpoints. Think: homepage, login, health checks. Keep it honest and fast.
- Scrape those metrics with [Prometheus](https://prometheus.io/) and pipe alerts through [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/). One place to look. One alert path to trust.

Tie this into your existing Grafana boards for vibes and sanity. Your app has outside eyes now.

Here’s the strategic bit

The coolest AI agent stuff right now isn’t a new “mind.” It’s operational glue. [VibeTrade](https://github.com/vibetrade-ai/vibe-trade) calls out the real blockers: no persistent operating memory across sessions, no trustworthy record of actions and reasoning, no hard approval boundary before money moves, no cheap always-on monitoring. [DashClaw](https://github.com/ucsandman/DashClaw) sits between agents and tools so it can evaluate decisions, apply policy rules, require approval, and log reasoning. [Tarvos](https://github.com/Photon48/tarvos) experiments with a relay architecture to avoid context rot. [Amux](https://news.ycombinator.com/item?id=47363707) keeps coding sessions live in [tmux](https://github.com/tmux/tmux/wiki) with watchdog logic, auto-compacting context and restarting after crashes.

All glue. And it matters because production systems need state, durability, and auditability. Not vibes. Not even close.

Step 3: Build a Self-Hosted Observability Pipeline with OpenTelemetry

You want everything flowing into one pipe: logs, metrics, traces. You also want configs that won’t break on a minor release. [OpenTelemetry](https://opentelemetry.io/) has been pushing hard on this, and key parts of their declarative config are now marked stable — including a [1.0.0 JSON schema for the data model](https://opentelemetry.io/blog/2026/stable-declarative-config/), with a YAML representation, an in-memory model, [ConfigProperties](https://opentelemetry.io/blog/2026/stable-declarative-config/) for generic YAML mappings, a [PluginComponentProvider](https://opentelemetry.io/blog/2026/stable-declarative-config/) for custom components, and SDK operations to parse YAML and instantiate components. Translation: you can build config once and reuse it across services.

Collector layout that works:

- Install the OpenTelemetry Collector (the contrib build). Configure via the stable declarative model.
- Set up receivers for your app (stdout or file logs; OTLP for traces/metrics), a processors stage, and exporters to your backends.
- Use the Unroll Processor if your app emits bundled logs. The [Unroll Processor](https://opentelemetry.io/blog/2025/contrib-unroll-processor/) was created specifically to split a JSON array of many events into separate log records for each logical event.
- Cut noise with the Filter Processor and OTTL’s context inference so you don’t have to micromanage internal telemetry contexts. Starting with [Collector Contrib v0.146.0](https://opentelemetry.io/blog/2026/ottl-context-inference-come-to-filterprocessor/), you can set top-level config fields like trace_conditions, metric_conditions, log_conditions, and profile_conditions.

You can keep it tiny. Example Collector snippet to illustrate the moving parts (adjust to your stack):

```yaml
receivers:
  filelog:
    include: [/var/log/myapp.log]
    operators:
      - type: json_parser
exporters:
  otlp:
    endpoint: http://otel-backend:4317
processors:
  filter/logs:
    log_conditions:
      - 'body matches "DEBUG"'
  unroll:
    logs:
      field: body
      mode: json_array
service:
  pipelines:
    logs:
      receivers: [filelog]
      processors: [unroll, filter/logs]
      exporters: [otlp]
```

One more change worth making: align on OTLP for tracing. [OpenTelemetry is deprecating its Zipkin exporters in favor of Zipkin’s OTLP ingestion support](https://opentelemetry.io/blog/2025/deprecating-zipkin-exporters/) because communities have gravitated to OTLP and exporter adoption was limited in several languages (in some cases even lower than the Jaeger exporter). If you’re still on a Zipkin exporter, switch to OTLP and call it a day.

The OpenTelemetry project has also been explicit about prioritizing stability, reliability, and better organization of releases and artifacts like docs and examples [source](https://opentelemetry.io/blog/2025/stability-proposal-announcement/). That’s good news if you’re betting on self-hosted observability. And if you’re dragging legacy workloads or on-prem boxes into this world, OTel is talking directly about those pain points — noisy logs and siloed data that lead to fragmented visibility [source](https://opentelemetry.io/blog/2026/demystifying-opentelemetry/).

Put Upright into this same pipe so your synthetic checks show up next to app traces and logs. One pane. Less guessing.

Step 4: Turn Your Terminal into an AI-Assisted Dev Cockpit

I learned this the hard way. AI is a force multiplier when it’s inside your editor, not in a separate tab. It should read code, run tools, remember context, and hand you diffs right where you work.

Option A: Neovim

- Install [Neovim](https://neovim.io/) and add [chat.nvim v1.4.0](https://github.com/wsdjeg/chat.nvim/releases/tag/v1.4.0). It runs entirely inside Neovim and is designed to be lightweight and hackable (it’s [Lua](https://www.lua.org/) under the hood).
- Hook up providers: OpenAI, [Gemini](https://ai.google/), [Anthropic](https://www.anthropic.com/), [Ollama](https://ollama.com/). Start with one.
- Turn on the tool system: web search, file search, git diff. Give the model eyes on your repo.
- Enable long-term memory and multiple sessions so you can keep a per-repo thread. It streams responses. It can even bridge messages from external chat platforms like [Discord](https://discord.com/), [Telegram](https://telegram.org/), [Lark](https://www.larksuite.com/), [WeCom](https://work.weixin.qq.com/), and [DingTalk](https://www.dingtalk.com/).

Option B: PNANA

If Vim-mode isn’t your style, try [PNANA](https://github.com/Cyxuan0311/PNANA). It’s a modern terminal editor built with C++17 and [FTXUI](https://github.com/ArthurSonzogni/FTXUI), with GUI-like shortcuts (Ctrl+S, Ctrl+Z), themes (Monokai, Dracula, Nord), a three-column layout, smart status bars, and built-in LSP for completion/diagnostics/jump-to-definition. Zero setup to look good. Feels at home over SSH.

A tiny self-hosted agent — with guardrails

If you want more than chat, scaffold a small coding/ops agent using [build-your-own-openclaw](https://github.com/czl9707/build-your-own-openclaw). It’s an 18-step path from a simple chat loop to a lightweight OpenClaw-style agent, and each step ships runnable code and a README explaining the design. Start small. Add a tool or two.

But don’t give it raw access to prod. Put [DashClaw](https://github.com/ucsandman/DashClaw) in front as a policy gateway. DashClaw sits between agents and tools so actions are requested, evaluated, optionally approved, and logged. That directly addresses the gaps [VibeTrade](https://github.com/vibetrade-ai/vibe-trade) flagged for autonomous systems: no persistent operating memory across sessions, no trustworthy record of actions and reasoning, no hard approval boundary before money moves, no cheap always-on monitoring. You need the same boundaries for deploys and migrations. Obvious in hindsight.

Stability tricks from the field

Borrow reliability techniques from [Tarvos](https://github.com/Photon48/tarvos) and [Amux](https://news.ycombinator.com/item?id=47363707):

- Tarvos proposes a “relay architecture” — cascade a fresh coding agent before context gets stale, aiming to avoid context rot.
- Amux keeps [Claude Code](https://www.anthropic.com/) sessions alive by parsing ANSI-stripped [tmux](https://github.com/tmux/tmux/wiki) output to see if it’s working, stuck, needs input, or if context is running low; when context drops below 20%, it sends /compact. When a session crashes from a redacted_thinking error, it restarts and replays the last message. In a YOLO session that hits a safety prompt, it auto-answers. There’s a watchdog to keep it all alive.

You can adapt the ideas: run your agent in tmux, add a watchdog process to detect hangs or low context, and restart while replaying the last safe message. Small scripts. Big difference.

Step 5: Optional Backend Simplification with Oxyde (for Your Indie API)

There’s a nice trick if you’re building a control plane or small API for your app: make a single model do double duty.

[Oxyde](https://github.com/mr-fatalyst/oxyde) is a Pydantic-native async ORM with a Rust core. The design goal is simple: your [Pydantic](https://docs.pydantic.dev/) model is also your database model, so you avoid duplicating models. One class gives you full validation on input and output and native type hints. The query API is consciously similar to Django’s while relying on Pydantic models rather than a separate ORM model system [source](https://github.com/mr-fatalyst/oxyde). It keeps things explicit. Good for small teams who want fewer moving parts and easier observability.

Put this API next to your main app, deploy it with [Kamal](https://kamal-deploy.org/), and pipe its logs/metrics through the same OpenTelemetry Collector and Upright probes as everything else. Same shape. Less mental overhead.

Real-world signals that this path scales beyond weekend projects

- [37signals cut their dependency on external registries like DockerHub and ECR](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/) and now run an on-prem Harbor registry as a central part of their deployment pipeline. They deploy with [Kamal](https://kamal-deploy.org/) and [Docker](https://www.docker.com/).
- [Upright](https://dev.37signals.com/introducing-upright/) watches over Basecamp, HEY, Fizzy and others. It runs checks from multiple geographic locations and reports when something breaks, with each node exposing [Prometheus](https://prometheus.io/) metrics for [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and [Grafana](https://grafana.com/).
- They’ve been migrating massive data sets off AWS — moving billions of files with no downtime as part of their cloud exit, building custom tooling for bandwidth and constraints, and choosing [Pure Storage FlashBlade](https://dev.37signals.com/pure-storage-monitoring/) for 10 PB of data [source](https://dev.37signals.com/moving-mountains-of-data-off-s3/). That post also speaks to the human side — verification, anxiety, the moment they hit delete.
- They even open-sourced critical building blocks like [Action Push Native](https://dev.37signals.com/introducing-action-push-native/), a Rails gem for sending push notifications to Apple and Google that they use in Basecamp and HEY to send more than 10 million push notifications per day.

Different scale. Same ethos: own the stack, keep it understandable, monitor it cheaply.

A quick word on where the CNCF ecosystem is headed

If you do outgrow a single VPS or need identity and trust wired across many systems, the CNCF ecosystem is plenty busy: Kubernetes pulls private images constantly and vendors ship credential providers for their registries [source](https://www.cncf.io/blog/2026/03/09/registry-mirror-authentication-with-kubernetes-secrets/). Identity and access management has become foundational infrastructure as architectures span more trust domains [source](https://www.cncf.io/blog/2026/03/06/kubecon-cloudnativecon-europe-2026-co-located-event-deep-dive-keycloakcon/). Infrastructure-as-code has its own inflection point — [OpenTofu Day debuted at KubeCon EU 2024 and has run at each KubeCon since](https://www.cncf.io/blog/2026/03/09/kubecon-cloudnativecon-europe-2026-co-located-event-deep-dive-opentofu-day/). And OpenTelemetry adoption keeps accelerating — you’ll even see “we deleted our observability stack and rebuilt it with OTel” talks at KubeCon EU 2026 [source](https://opentelemetry.io/blog/2026/kubecon-eu/). Translation: you’re not painting yourself into a corner by starting simple.

## Quick Reference

- [Kamal](https://kamal-deploy.org/) — Simple deploy tool used by 37signals across apps [source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/).
- [Docker](https://www.docker.com/) — Container runtime used in Kamal deploys at 37signals [source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/).
- [Harbor](https://goharbor.io/) — Self-hosted container registry; 37signals runs their registry on-prem with Harbor [source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/).
- [Upright](https://dev.37signals.com/introducing-upright/) — Synthetic monitoring (Rails engine) deployed to cheap VPS nodes with Kamal; exports Prometheus metrics; alert with Alertmanager; dashboard in Grafana.
- [Prometheus](https://prometheus.io/) — Metrics system scraping Upright nodes; use with [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/).
- [Grafana](https://grafana.com/) — Visualize Prometheus metrics from Upright and your apps.
- [OpenTelemetry](https://opentelemetry.io/) — Observability standard; declarative configuration spec marked stable for key parts [source](https://opentelemetry.io/blog/2026/stable-declarative-config/).
- [OTLP](https://opentelemetry.io/docs/specs/otlp/) — Preferred protocol; Zipkin exporters deprecated in favor of Zipkin’s OTLP ingestion [source](https://opentelemetry.io/blog/2025/deprecating-zipkin-exporters/).
- [Unroll Processor](https://opentelemetry.io/blog/2025/contrib-unroll-processor/) — Expands bundled logs (e.g., JSON arrays) into separate log records.
- [Filter Processor + OTTL context inference](https://opentelemetry.io/blog/2026/ottl-context-inference-come-to-filterprocessor/) — Use trace_conditions, metric_conditions, log_conditions, profile_conditions to filter without manual context wiring.
- [chat.nvim v1.4.0](https://github.com/wsdjeg/chat.nvim/releases/tag/v1.4.0) — Neovim AI hub; multiple providers (OpenAI, Gemini, Anthropic, Ollama); tools (web search, file search, git diff); long-term memory; multiple sessions; bridges Discord/Telegram/Lark/WeCom/DingTalk.
- [Neovim](https://neovim.io/) — Editor that hosts chat.nvim.
- [PNANA](https://github.com/Cyxuan0311/PNANA) — Terminal editor with GUI-style shortcuts, themes, 3-column layout, and built-in LSP.
- [build-your-own-openclaw](https://github.com/czl9707/build-your-own-openclaw) — 18-step tutorial to build a minimal OpenClaw-style agent; each step has a README and runnable code.
- [DashClaw](https://github.com/ucsandman/DashClaw) — Policy gateway between AI agents and tools; evaluate decisions, enforce approval, log reasoning.
- [VibeTrade](https://github.com/vibetrade-ai/vibe-trade) — Notes the real blockers for autonomous action: persistent memory, action/reason logs, approval boundaries, cheap monitoring.
- [Tarvos](https://github.com/Photon48/tarvos) — Relay architecture for coding agents to avoid context rot.
- [Amux](https://news.ycombinator.com/item?id=47363707) — Self-healing multiplexer for Claude Code; tmux-based watchdog; auto-compact at 20% context; restart on crashes.
- [Oxyde](https://github.com/mr-fatalyst/oxyde) — Pydantic-native async ORM; one model for API and DB; Django-style query API built on Pydantic.
- [Action Push Native](https://dev.37signals.com/introducing-action-push-native/) — Rails gem for Apple/Google push; used by Basecamp/HEY to send 10M+ pushes per day.
- [Cloud-exit data migration](https://dev.37signals.com/moving-mountains-of-data-off-s3/) — 37signals’ move of billions of files off S3; [Pure Storage FlashBlade choice](https://dev.37signals.com/pure-storage-monitoring/) for 10 PB.

Here’s the finish line

At this point you’ve got an indie-grade platform without renting a platform team. Your app ships via [Kamal](https://kamal-deploy.org/) to boxes you control, with images living in your own [Harbor](https://goharbor.io/) instance [source](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/). [Upright](https://dev.37signals.com/introducing-upright/) pokes it from the outside, [OpenTelemetry](https://opentelemetry.io/) streams what it’s doing on the inside, and [Prometheus](https://prometheus.io/)/[Grafana](https://grafana.com/) give you a single place to see it. On the dev side, you’re not just prompting a chatbot — you’re embedding AI into your editor with [chat.nvim](https://github.com/wsdjeg/chat.nvim/releases/tag/v1.4.0) or working comfortably in [PNANA](https://github.com/Cyxuan0311/PNANA), and, if you want, running a tiny agent built from [build-your-own-openclaw](https://github.com/czl9707/build-your-own-openclaw) with [DashClaw](https://github.com/ucsandman/DashClaw) as the safety harness. For extra resilience, borrow the watchdog ideas from [Amux](https://news.ycombinator.com/item?id=47363707) and the context tricks from [Tarvos](https://github.com/Photon48/tarvos).

You can stop here and be in a better place than the average default-cloud stack. Or keep going: scale Harbor, add Kubernetes later for heavier AI loads if you truly need it — the [CNCF](https://www.cncf.io/blog/2026/03/05/the-great-migration-why-every-ai-platform-is-converging-on-kubernetes/) world is ready for you when that day comes. The key is that every piece here is something you can run, inspect, and swap. A modern stack that still feels like yours.

Wild stuff. Now go ship it.
