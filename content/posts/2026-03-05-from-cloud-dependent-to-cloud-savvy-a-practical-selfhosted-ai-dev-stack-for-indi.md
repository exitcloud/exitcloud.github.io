---
title: "From Cloud-Dependent to Cloud-Savvy: A Practical Self‑Hosted + AI Dev Stack for Indie Teams"
date: 2026-03-05T16:55:12.229Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# From Cloud-Dependent to Cloud-Savvy: A Practical Self‑Hosted + AI Dev Stack for Indie Teams

Here’s the thing — you don’t need a platform team to own your stack. The [CNCF](https://cncf.io) keeps publishing big-kid stories about everyone converging on [Kubernetes](https://kubernetes.io/) for AI workloads and platform engineering. True for them. But I’ve found a simpler stack works beautifully for indie teams: containers, a private registry, synthetic monitoring, and a couple of dedicated VMs for agents. Ship it.

I learned this the hard way. Simple deploys balloon when you outsource ownership. So I started from what the pragmatists actually do. [37signals deploys with Kamal using Docker](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/). They run their own [Harbor](https://goharbor.io/) registry on‑prem to avoid external coupling. They built Upright for synthetic checks and push its [Prometheus](https://prometheus.io/) metrics into [Grafana](https://grafana.com/) with [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/). They even moved billions of files off S3 with no downtime by building custom tooling when nothing off the shelf quite fit ([story here](https://dev.37signals.com/moving-mountains-of-data-off-s3/)). That mindset scales down nicely.

What follows is the stack I’d hand to a small team that wants to own their deploys, keep an eye on the app, and let a couple of AI agents run 24/7 doing the boring work.

No Kubernetes required. Not even close.


## 1. Map the Stack: What We’re Building on Two Cheap VPSs

- App deployment via [Kamal](https://github.com/basecamp/kamal) + [Docker](https://www.docker.com/), pushing images to your own [Harbor](https://goharbor.io/). This mirrors how [37signals moved from Docker Hub and ECR to Harbor](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/) to reduce license cost exposure and avoid tight coupling to external registries.
- Synthetic monitoring via Upright — a Rails engine deployed to cheap nodes. It runs probes from multiple geographies and exports [Prometheus](https://prometheus.io/) metrics for [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and [Grafana](https://grafana.com/). Clean, understandable, and battle‑tested on services like Basecamp and HEY.
- 24/7 AI agents using the [LiberClaw](https://hnrss.org) pattern: one VM per agent, each with its own filesystem, database, and HTTPS endpoint, with inference on open models like Qwen3 Coder and GLM‑4.7 so you don’t need external API keys. They report 61 agents across 578 conversations with 99.7% uptime — a nice proof that the VM‑per‑agent pattern is viable at modest scale ([source](https://hnrss.org)).
- Storage choices are yours. If you’re doing anything heavy, treat storage as a first‑class citizen. 37signals is moving 10 petabytes to Pure Storage FlashBlade, spanning customer attachments, long‑term metrics, and database backups, and they consider it top‑priority for observability. At indie scale: same mindset, smaller box.

This borrows from their cloud‑exit playbook without the budget or staffing assumptions. Start small. One app box, one monitoring box. Add a third later if you feel cramped.


## 2. Ditch Docker Hub: Run Harbor as Your Self‑Hosted Registry for Kamal

If you deploy often and rely on external registries, you’re tying your ship rhythm to someone else’s platform. That’s a footgun. [37signals moved to Harbor](https://dev.37signals.com/running-our-docker-registry-on-prem-with-harbor/) because Docker Hub licenses and coupling were getting in the way of their “kamalization” effort. You can do the same on a single host.

What Harbor gives you:
- Your own image registry, local to where you build and deploy.
- A path to independence from Docker Hub/ECR-style vendor gravity.
- A straightforward target for Kamal builds and pushes.

A lightweight deployment pattern:
- Single‑server [Harbor](https://goharbor.io/) with object storage. 37signals backed Harbor with S3 storage on Pure Storage FlashBlade. At indie scale, any S3‑compatible backend works. You keep the same Harbor knobs, different storage endpoint.
- Point [Kamal](https://github.com/basecamp/kamal) to your Harbor registry and push images there. This lets you ship containers straight from your CI or laptop to your own registry, then pull to your app box.

Why this instead of Kubernetes for a tiny team? Look at the scale where Kubernetes really shines. The CNCF writes about every AI platform converging on Kubernetes, with a dedicated WG Serving to push AI inference. Meanwhile, at KubeCon Europe 2026, there’s a talk titled “We Deleted Our Observability Stack and Rebuilt It With OTel: 12 Engineers to 4 at 20K+ Clusters” (source). That’s the scale where Kubernetes has gravity. For a handful of services? Kamal + Docker + Harbor is simpler to run, simpler to reason about, and avoids babysitting a control plane you don’t need.

Here’s the practical loop:
- Build images locally or in CI.
- Push to your [Harbor](https://goharbor.io/) host.
- Pull on the target box and start containers through [Kamal](https://github.com/basecamp/kamal).

Keep backups simple. Primary Harbor + periodic off‑site sync of your object storage bucket and database. If you want extra safety, a second Harbor instance on another VPS with read‑only replication. Not perfect, but it’ll survive most day‑to‑day hiccups.


## 3. Ship with Confidence: Upright + Prometheus + Grafana on a Single VPS

You don’t know you’re up until something outside your box says you’re up. Upright is purpose‑built for that. It’s a [Rails](https://rubyonrails.org/) engine you deploy to cheap nodes, and it runs probes from multiple locations. It’s used by 37signals to watch Basecamp, HEY, and Fizzy.

What I like:
- Real HTTP probes that hit your endpoints.
- [Playwright](https://playwright.dev/) probes for browser flows, with video on failure.
- SMTP and traceroute/MTR probes for extra coverage.
- Every node exports [Prometheus](https://prometheus.io/) metrics; set up [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) to page you, and use [Grafana](https://grafana.com/) for dashboards. Simple and powerful.

Pattern to copy at small scale:
- Run Upright on your monitoring VPS.
- Scrape its metrics locally with Prometheus. Keep alerting rules tiny and loud.
- Add a second cheap node later if you want geographic diversity.

Want to clean up noisy telemetry as you grow? Drop in an [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) sidecar for processing. Two features worth using right away:
- The Filter Processor’s new OTTL context inference (since Collector Contrib v0.146.0) adds `trace_conditions`, `metric_conditions`, `log_conditions`, and `profile_conditions` so you can write filter conditions without juggling internal contexts (details).
- The Unroll Processor converts bundled events in a single log record (e.g., a JSON array) into individual log records so downstream processing stays sane (background). Early discussions attempted this via transformations; the dedicated processor won out.

And for inspiration: there’s that real‑world rebuild where a team managed observability at 20K+ clusters with fewer engineers after moving to OTel (talk reference). We can borrow the pattern without the headcount — one collector, a couple of processors, and done.

One more network trick: routing mysteries. When a region looks wrong in traceroute land, use [Tracemap](https://hnrss.org) and [Globalping](https://hnrss.org) together. The workflow is simple — run traceroutes or MTR from probes worldwide, visualize hops on a map, and spot broken IP geolocation when you see 1–2 ms latency between points that are allegedly far apart ([source](https://hnrss.org)). It’s a great sanity check before you blame your ISP.


## 4. Run Your Own 24/7 AI Agent on a Small VPS (LiberClaw Pattern)

Background agents shouldn’t be glued to your chat tab. [LiberClaw](https://hnrss.org) shows a crisp model: give each agent its own VM, filesystem, database, and HTTPS endpoint, and run it around the clock. Define behavior in a Markdown “skills” file. Use open models like Qwen3 Coder and GLM‑4.7 so you aren’t blocked on external API keys. They report 61 agents running across 578 conversations with 99.7% uptime. Solid.

What an indie setup looks like:
- Provision one small VM per agent. Keep it boring: system service, heartbeat, a local database.
- Package tools the agent can call: shell, file ops, HTTP fetch, maybe web search. That toolkit is part of the LiberClaw pattern ([source](https://hnrss.org)).
- Give it a single responsibility. For example: “watch my synthetic monitoring data and open an issue if error rate spikes.” Or “scrape docs and draft a digest.” Clear. Bounded.

Sensitive operations? Copy the isolation patterns:

- For purchases, [Selva](https://hnrss.org) is a CLI/API “skill” that lets agents search/comparison shop and complete Amazon orders. It uses [Stripe](https://stripe.com/) for card storage so card details never touch Selva’s servers and are never exposed to the agent; you can set a dollar threshold that requires email approval above that amount ([source](https://hnrss.org)). Clean boundary.
- For messaging/file sync without servers, [Ouroboros](https://hnrss.org) is a Rust‑based P2P system designed to avoid infrastructure seizure. It supports live sessions using a shared passphrase that deterministically sets network parameters, plus asynchronous EtherSync spaces that sync via gossip. Like self‑hosting on hard mode, but the design goal is strong isolation and independence.

Could this all sit on Kubernetes? Sure — the ecosystem is heading there for AI (CNCF analysis; WG Serving outcome). But for one or two agents, a VM is faster to stand up and easier to reason about. You can always containerize later and hang them behind [Kamal](https://github.com/basecamp/kamal) with images in [Harbor](https://goharbor.io/). Start small; avoid yak shaving.


## 5. Day‑to‑Day: An LLM‑First Dev Workflow That Doesn’t Overwhelm You

The trick is to use AI as a force multiplier without turning your week into prompt archaeology.

- Plan with the [Beads planner plugin for Claude Code](https://hnrss.org). It borrows Steve Yegge’s Beads idea: only the active work lives in Beads; the backlog stays in GitHub issues. The workflow splits into a planning loop (pull issues into Claude Code) and an execution loop ([source](https://hnrss.org)). Lightweight. Focused.
- Execute with [Todoglow](https://hnrss.org): a keyboard‑first macOS todo app with MCP support, built to sit next to Notion/Jira. You break big tasks into atomic todos and track how long each takes. It intentionally omits categories, tags, and priorities — if the todos are small enough, you don’t need them ([source](https://hnrss.org)). Sharp.
- Keep living docs in Markdown. Use [Attn (attnmd)](https://hnrss.org) — a Rust binary under 20MB that runs on an OS webview instead of Electron — so you can open docs straight from the terminal and keep the loop snappy. Command is simple:
  
  ```
  npx attnmd README.md
  ```

  It’s designed to pair with Claude Code for AI‑generated plans and docs.

- Need collaboration fast? The [Velt](https://hnrss.org) plugin for Claude Code can add CRDT‑based live document sync, contextual comments, live presence/cursors, notifications, and reactions to your app in under 10 minutes via one command. That encapsulates infrastructure honed over three years and used by Pendo, HeyGen, and LambdaTest ([source](https://hnrss.org)). You get real‑time features without a multi‑week CRDT buildout.

- Prompting and experiments: [OXPT](https://hnrss.org) gives you a branching canvas so you can fork AI responses, compare side‑by‑side, and treat prompts as modular building blocks with version control. A massive upgrade from linear chat interfaces.

- Media work? [Montage](https://hnrss.org) is a fork of [Remotion](https://www.remotion.dev/)'s Next.js template configured so coding agents can assemble product launch videos from reusable animation primitives. The idea came after the author paid about $2000 and 4–5 full days of attention for a prior launch video; they hypothesized that encoding reusable primitives in code would enable faster, AI‑driven production ([source](https://hnrss.org)). Interesting direction.

- Small helpers:
  - [CodeConvert](https://www.codeconvert.dev/) for JSON→TypeScript and YAML↔JSON conversions. Saves you from silly formatting bugs.
  - [Sift](https://hnrss.org) if you write Java: a fluent DSL that enforces valid regex syntax at compile time through a strict type‑state machine. It provides immutable builders, modular composition, and lazy validation for backreferences across disconnected blocks ([source](https://hnrss.org)). Regexes you can read later. Imagine that.
  - Lexxy for Rails apps: a rich text editor for Action Text based on Meta’s [Lexical](https://lexical.dev/), with better semantics (real <p> tags), Markdown shortcuts and auto‑format on paste, real‑time code highlighting, creating links by pasting URLs onto selected text, configurable prompts/mentions, and inline previews for PDFs and video. Solid default for AI‑assisted writing in web apps.

- Supervise agents, don’t babysit them. The author of [OmoiOS](https://hnrss.org) built a sizable Apache 2.0 Python codebase (~190K lines) to reduce human babysitting of AI agents, after struggling with four approaches: direct LLM chat (good for small tasks but you keep re‑explaining context), Copilot/Cursor (autocomplete is great, but you’re still doing most of the thinking), continuous agents (no errors doesn’t mean the feature works), and parallel agents (faster, but heavy manual review) ([source](https://hnrss.org)). The lesson for us: make the phases explicit — spec, requirements, design, tasks, validation — even if you implement it in a lightweight way.


## 6. Keep It Observable (and Simple) as You Grow

Two rules:
- Treat your storage and registry as critical. 37signals treats their Pure Storage FlashBlade as top‑priority for observability because of the volume and importance of data there. Scale that idea down — even if you’re using a modest S3‑compatible box, give it alerts for capacity and latency. Don’t wait.
- Prefer [OTLP](https://opentelemetry.io/docs/specs/otlp/) as your telemetry lingua franca. OpenTelemetry is deprecating the Zipkin exporter spec because community adoption strongly favors OTLP (Zipkin has OTLP ingestion). Future‑proofs your pipeline.

A minimal toolkit that punches above its weight:
- [Prometheus](https://prometheus.io/) scrapes from Upright.
- [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) pings you on hard failures.
- An [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) with the Filter Processor context inference fields (`trace_conditions`, `metric_conditions`, `log_conditions`, `profile_conditions`) and the Unroll Processor to handle bundled log events (feature; unroll).
- Keep an eye on the OpenTelemetry Configuration Schema — there’s a new release candidate after about three years of work, with better tooling to prevent inconsistencies. The ecosystem is mature and growing fast; contributors went from just over 5,000 in October 2022 to almost 20,000 in a later update (community note). Good signal.

Kubernetes‑wise, keep your options open. The Gateway API is a rethought way to expose services in Kubernetes, avoiding vendor‑specific annotations, and it’s being used in contexts like SpinKube. And the CNCF calls out how cloud‑native management has matured with stronger supply‑chain expectations and increasing regulation. If you outgrow your simple stack, you can migrate intentionally. Not before.

Also — if you’re local to a community, Kubernetes Community Days are worthwhile for learning. Even if you’re not running a cluster yet, you’ll meet folks who’ve solved the ugly problems.


## Quick Reference

- [Kamal](https://github.com/basecamp/kamal) — Simple app deployment with Docker containers (used by 37signals across all apps by early 2025).
- [Docker](https://www.docker.com/) — Container platform for builds and deploys.
- [Harbor](https://goharbor.io/) — Self‑hosted Docker registry; 37signals moved off Docker Hub/ECR to Harbor to reduce costs and coupling.
- Upright — Rails engine for synthetic monitoring; probes from multiple geographies, exports Prometheus metrics for Alertmanager/Grafana.
- [Prometheus](https://prometheus.io/) — Metrics scraping and time‑series storage.
- [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) — Alert routing and notifications.
- [Grafana](https://grafana.com/) — Dashboards for metrics.
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) — Telemetry pipeline; use Filter Processor context inference and Unroll Processor.
- OTTL context inference — Filter Processor fields: trace_conditions, metric_conditions, log_conditions, profile_conditions.
- Unroll Processor — Split bundled log events into individual records.
- Zipkin exporter deprecation — Prefer OTLP; Zipkin ingests OTLP.
- [LiberClaw](https://hnrss.org) — Open‑source platform: one VM per agent, open models, Markdown “skills”; reports 61 agents, 578 conversations, 99.7% uptime.
- [Selva](https://hnrss.org) — CLI/API skill for Amazon shopping; uses Stripe for card storage, never exposes card data to agents, supports email approval thresholds.
- [Ouroboros](https://hnrss.org) — Post‑quantum P2P messenger/file sync in Rust; zero servers; live sessions via shared passphrase; EtherSync spaces via gossip.
- [Tracemap + Globalping](https://hnrss.org) — Run traceroutes/MTR from global probes and visualize; useful to spot wrong IP geolocation.
- [Beads planner for Claude Code](https://hnrss.org) — Active‑only work items (Beads) with planning loop separated from execution; backlog in GitHub issues.
- [Todoglow](https://hnrss.org) — Keyboard‑first macOS todos with MCP; atomic tasks; no categories/tags/priorities; tracks time per todo.
- [Attn (attnmd)](https://hnrss.org) — <20MB Rust Markdown viewer/editor using OS webview; launch from terminal (npx attnmd README.md).
- [Velt plugin for Claude Code](https://hnrss.org) — Adds CRDT collaboration (live cursors, comments, notifications, reactions) in under 10 minutes; used by Pendo, HeyGen, LambdaTest.
- [OXPT](https://hnrss.org) — Visual branching canvas for prompt/versioning; side‑by‑side branches; modular prompts.
- [Montage](https://hnrss.org) — Remotion Next.js template for AI‑assisted product videos using reusable animation primitives.
- [CodeConvert](https://www.codeconvert.dev/) — JSON→TypeScript and YAML↔JSON developer conversions.
- [Sift](https://hnrss.org) — Java DSL for readable, type‑safe regexes; compile‑time guarantees via type‑state machine; immutable builders; lazy validation.
- Lexxy — Rich text editor for Rails Action Text based on Lexical; semantic HTML, Markdown shortcuts, code highlighting, link‑by‑paste, prompts/mentions, attachment previews.
- Pure Storage FlashBlade — Used by 37signals for 10 PB of data; top‑priority observability.
- [37signals S3 migration](https://dev.37signals.com/moving-mountains-of-data-off-s3/) — Migrated billions of files with no downtime; built custom tooling; human verification mattered.
- CNCF: AI on Kubernetes — AI platforms converging on K8s; WG Serving advanced inference support.
- Gateway API — Rethought service exposure on Kubernetes; avoids vendor‑specific annotations.
- CNCF notes on platform engineering — Mature cloud‑native management, supply chain expectations, increasing regulation.
- Kubernetes Community Days — Local community events sharing cloud‑native knowledge.
- OpenTelemetry configuration schema RC — New RC after ~3 years, with tooling to improve consistency.
- OTel docs on traditional environments — Noisy logs and siloed tools lead to fragmented visibility; OTel helps consolidate.
- OpenTelemetry community growth — From just over 5,000 contributors (Oct 2022) to almost 20,000 later; maturity signal.


## The mindset that scales down

You don’t need a 20‑person platform group or 10PB on Pure Storage to take control of your stack. By pairing [Kamal](https://github.com/basecamp/kamal) with a self‑hosted [Harbor](https://goharbor.io/) registry, wiring up Upright + [Prometheus](https://prometheus.io/) + a bit of [OpenTelemetry](https://opentelemetry.io/), and layering in a sane AI workflow — 24/7 agents on tiny VMs, Claude Code plugins, and [OmoiOS](https://hnrss.org)‑inspired supervision — you get most of the benefits of a big‑company platform without the overhead.

Start with one piece. Maybe Harbor + Kamal for your next deploy. Or Upright on a small VPS. Or a single LiberClaw‑style agent that watches your logs and taps you when something drifts. As those pieces harden, expand horizontally: more probes, more agents, more automation. The through‑line matches what 37signals and the best AI tool builders keep demonstrating: own the critical parts, keep the architecture understandable, and let automation and observability do the boring work so you can ship faster.

Partial to a cron that curls /health every minute? Same energy.
