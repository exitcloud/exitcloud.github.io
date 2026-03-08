---
title: "From $10 VPS to Serious Stack: Self‑Hosted Deploys, AI Helpers, and Observability for Indie Devs"
date: 2026-03-08T14:07:15.853Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# From a tiny VPS to a serious stack: self‑hosted deploys, AI helpers, and observability that fits in your head

Here’s the thing — most indie apps don’t need a platform team. You need a box, a few services, and a way to see smoke before fire. I learned this the hard way.

Let me show you what actually works. Real tools. Real commands. And a stack you can run without spinning up a cluster zoo.


## 1. Bootstrap your remote dev box (remote UX that doesn’t suck)

Spin up a small VPS and make it your cockpit. One shell you live in for deploys, scripts, and experiments. Not fancy. Just yours.

- Start with [Havn](https://hnrss.org). It exists because constantly typing `lsof -i :PORT` is yak shaving. Havn makes a single lsof/netstat pass to map listening processes, then scans 100+ ports in parallel via TCP connects with a 150 ms timeout. It even fingerprints HTTP by reading response headers so you can spot frameworks like Spring Boot, Express, and Django in one glance. Bonus: it peeks at package.json, pom.xml, go.mod, and Cargo.toml in each process’s working directory to infer the actual project name. And yes — it PING/PONGs Redis and performs a Postgres connection handshake to check health. One command, the whole picture.

- If you love [tmux](https://github.com/tmux/tmux) but hate memorizing chords, try [tmuxy](https://hnrss.org). It wraps tmux in a web/desktop GUI, aims to stay powerful and scriptable, and uses a Rust backend that hooks tmux control mode to stream state updates. Control mode is one of those lesser-known superpowers — tmuxy rides it so you don’t have to.

- Your CLI app feels slick on your fiber connection; your users are on hotel Wi‑Fi. Test that. [ttylag](https://hnrss.org) wraps any command in a shaped PTY and simulates RTT and jitter in both directions. It exposes a `--bits-per-byte` knob, runs entirely in userspace, and works on macOS (and probably Linux) without touching network namespaces or firewall rules. Chaos for TUIs. No kernel hacks.

Now you’ve got a friendly remote workspace. From here we ship.


## 2. Own your deploy pipeline: on‑prem registry + SQLite‑friendly app design

You don’t need Kubernetes. You need a container registry you control and a deploy tool that pushes images to a few boxes. That’s it.

- 37signals deploys everything with [Kamal](https://github.com/basecamp/kamal) using [Docker](https://www.docker.com/) as the containerization layer ([source](https://dev.37signals.com)). Their ecosystem had been tightly coupled to external registries like Docker Hub and Amazon ECR — and the paid Docker Hub license plus pull/push activity added up ([source](https://dev.37signals.com)). The move? Bring the registry in‑house.

- If you want a self‑hosted registry, look at [Harbor](https://goharbor.io/). It’s the kind of “own your artifacts” decision that keeps your deploys fast and predictable. The 37signals write‑up on running their Docker registry on‑prem with Harbor lays out the motivation and the outcome: cut external dependencies, keep images under your control, and keep Kamal happy pulling from your own endpoints ([source](https://dev.37signals.com)). Keep it simple: start with Harbor’s docs, get it running, point your deploy tool at it.

- Architecture note: SQLite isn’t a toy database. It can be a first‑class citizen if you make room for it in your design. 37signals explored giving every customer their own SQLite database for their Fizzy product to support both self‑hosted and SaaS models — a performance‑oriented multi‑tenant experiment ([source](https://dev.37signals.com)). They even open‑sourced [Active Record Tenanted](https://dev.37signals.com), a gem to make per‑tenant SQLite in Rails more seamless and add safeguards to prevent leaks between tenants. That experiment surfaced increasing tradeoffs as launch approached, and they chose to unwind months of work days before release ([source](https://dev.37signals.com)). Lessons learned, shared publicly. The takeaway for indie apps: design with SQLite in mind so you can choose it when it fits, or swap for Postgres/MySQL later without rewriting the world.

Here’s my contrarian take: do synthetic monitoring first, metrics second. Stop painting dashboards before you know if /login works. [Upright](https://dev.37signals.com) — more on this next — is a Rails engine they deploy with Kamal to cheap VPS nodes around the world, and they consider it core infrastructure. Start there. Then add metrics where they inform action. Fancy graphs? Later.


## 3. Add synthetic monitoring in a weekend with Upright + cheap VPS nodes

Monitoring that mirrors how a user experiences your app beats raw CPU graphs. Every time.

- 37signals open‑sourced [Upright](https://dev.37signals.com), the synthetic monitoring system they use to watch Basecamp, HEY, Fizzy, and other services. It runs health checks from multiple geographic locations and exposes [Prometheus](https://prometheus.io/) metrics — plug those into [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) for notifications and [Grafana](https://grafana.com/) for a dashboard when you need it ([source](https://dev.37signals.com)). It’s packaged as a Rails engine and they deploy it to cheap VPS nodes with Kamal ([source](https://dev.37signals.com)). Simple idea, big surface area covered.

- Local first. Spin up an Upright app on your laptop with [SQLite](https://www.sqlite.org/) to understand the model. Then deploy a couple of nodes so your checks run from multiple regions. Not a science project — a few VPSs and a Kamal config.

- Write a handful of health checks that reflect the happy path. Home page. Login. The common action in your app. Let Prometheus scrape Upright’s metrics and Alertmanager page you when a check flips. Grafana can wait.

- If you’re into factoring in the human side of ops, read how they moved billions of files out of S3 with no downtime and the verification anxiety that came with it ([source](https://dev.37signals.com)). It’s a good reminder: the goal of monitoring is trust. You want sufficient signal to make decisions, not graphs to admire.

End result? You’ll catch the user‑visible failures fast and early. Before a customer email. That’s the win.


## 4. Layer in AI‑assisted workflows that actually run on your machines

AI is a force multiplier. But only if it lives alongside your stack and doesn’t pull you back into vendor lock‑ins. Local‑first where it counts.

- Start with scraping. [Trawl](https://hnrss.org) flips the usual approach: instead of brittle CSS selectors, you describe fields in natural language and it uses LLMs to understand the HTML structure, with [Go](https://go.dev/) doing high‑throughput extraction ([source](https://hnrss.org)). Try this pattern in a tiny service of your own. Example straight from their demo:
  
  ```
  trawl "https://books.toscrape.com" --fields "title, price, rating, in_stock"
  ```

  Small CLI, big payoff. Minimal glue code. If a site tweaks its DOM, you’re not babysitting selectors all night. Scrape. Ship.

- Don’t dump scraped text into a void. Ingest it. [RAG‑Ready Extractor](https://github.com/CarlosManuelDiaz/rag-ready-extractor) describes itself as structure‑aware ingestion with semantic scoring ([source](https://github.com/CarlosManuelDiaz/rag-ready-extractor)). That’s exactly the kind of preprocessing you want before feeding a RAG or search layer. Keep your pipeline explicit — documents in, structured chunks out.

- Want to run agents locally? [Moruk OS](https://hnrss.org) is an autonomous AI “operating system” that runs on Linux — and it’s explicitly not a chatbot ([source](https://hnrss.org)). It decomposes tasks into subtasks, executes them, writes and runs code, browses the web, and learns from interactions ([source](https://hnrss.org)). It supports multiple models including Claude, GPT‑4, Gemini, Groq, DeepSeek, and any OpenAI‑compatible model ([source](https://hnrss.org)). It leans on persistent memory via a vector store and [SQLite](https://www.sqlite.org/) ([source](https://hnrss.org)), includes a DeepThink secondary reasoning layer to review critical actions ([source](https://hnrss.org)), and has a plugin system where dropping a .py file into plugins/ is enough to add functionality ([source](https://hnrss.org)). Local power.

- Harden the runtime. There’s a growing set of self‑hostable safety and isolation tools:
  - [BunkerVM](https://github.com/ashishgituser/bunkervm) provides a secure runtime using microVM sandboxes for AI agents ([source](https://github.com/ashishgituser/bunkervm)).
  - [AvaKill](https://github.com/log-bell/avakill) adds a deterministic safety firewall for agents that runs in under 1 ms and doesn’t use machine learning ([source](https://github.com/log-bell/avakill)).
  - [SkyClaw](https://github.com/nagisanzenin/skyclaw) is a self‑healing LLM agent runtime in Rust that includes task checkpointing ([source](https://github.com/nagisanzenin/skyclaw)).

- Prefer simple, inspectable stores. Moruk OS shows how far SQLite + a vector store can go on a single machine. The same spirit appears in [Country Cockpit](https://hnrss.org), a not‑yet‑live dashboard that shows country‑level exports as concrete objects (think dresses, yachts, liters of wine) instead of money ([source](https://hnrss.org)). It covers 37 countries and pulls from UN Comtrade, OECD/COFOG, revenue, debt, strategic dependencies, and 24 World Bank indicators ([source](https://hnrss.org)). Frontend is [Next.js](https://nextjs.org/), [React](https://react.dev/), [D3.js](https://d3js.org/), backend is [FastAPI](https://fastapi.tiangolo.com/) and SQLite ([source](https://hnrss.org)). SQLite shows up where work gets done. Again.

- Bonus thought: measuring AI drift on your own terms matters. There’s a small dataset on [Hugging Face](https://huggingface.co/datasets/louidev/glassballai) that logs stock predictions from Gemini for 38 days to study LLM drift ([source](https://huggingface.co/datasets/louidev/glassballai)). It’s a tiny but useful practice: log outputs and audit over time.

AI helpers aren’t magic. They’re just services you run next to your app. The hard part is owning the edges: isolation, safety, and data pipelines. The rest is wiring.


## 5. Wire up observability with OpenTelemetry’s declarative config

Here’s where most stacks get bloated. You don’t need ten agents and a consultant. Use [OpenTelemetry](https://opentelemetry.io/) the way it’s meant to be used: explicit configuration, portable formats, and a couple of processors that clean your signals before storage.

- The declarative configuration specification has key portions marked stable, including the JSON schema for the data model (the opentelemetry‑configuration project released v1.0.0) ([source](https://opentelemetry.io)). The spec defines a YAML representation, an in‑memory representation, a generic mapping node representation called ConfigProperties, and a PluginComponentProvider mechanism for custom plugin components ([source](https://opentelemetry.io)). The SDK now includes operations to parse YAML and instantiate components from it ([source](https://opentelemetry.io)). Translation: write a YAML file, spin up the Collector, and it builds your pipeline. No mystery.

- Logs a mess? Use the [Unroll Processor](https://opentelemetry.io) to handle log records that bundle multiple logical events — like a JSON array of ten entries in one line — and expand them into separate log records ([source](https://opentelemetry.io)). It started as a possible OTTL trick, then became its own processor ([source](https://opentelemetry.io)). Now your tools can work with individual entries the way they’re supposed to.

- Routing and filtering got easier too. Starting with opentelemetry‑collector‑contrib v0.146.0, OTTL context inference is available in the Filter Processor via four top‑level fields: trace_conditions, metric_conditions, log_conditions, and profile_conditions ([source](https://opentelemetry.io)). That simplifies configuration and changes how filter condition evaluation works so you don’t have to juggle internal telemetry contexts ([source](https://opentelemetry.io)). Cleaner configs. Fewer footguns.

- Standardize on OTLP. The project is deprecating the Zipkin exporter spec in favor of Zipkin’s OTLP ingestion ([source](https://opentelemetry.io)). Maintainers saw the community gravitate to OTLP, with Zipkin exporters showing limited adoption — sometimes even less than the already‑deprecated Jaeger exporter ([source](https://opentelemetry.io)). Minimal user engagement on Zipkin‑exporter issues plus plenty of alternatives sealed it ([source](https://opentelemetry.io)). Fewer protocols, fewer headaches.

Traditional environments aren’t going anywhere. On‑prem data centers, legacy apps, industrial systems — battle‑tested and deeply integrated, but tricky for modern observability ([source](https://opentelemetry.io)). Expect noisy, unstructured logs and siloed monitoring data scattered across tools ([source](https://opentelemetry.io)). That’s exactly where OTel’s declarative model and processors shine. Write the intent, ship the collector, iterate.

Want to see where OTel is headed? There’s an upcoming talk at KubeCon + CloudNativeCon Europe 2026 titled “We Deleted Our Observability Stack and Rebuilt It With OTel: 12 Engineers to 4 at 20K+ Clusters” ([source](https://opentelemetry.io)). Big shops are consolidating. You can too — at a different scale, same ideas.


## Bonus: 37signals’ cloud exit playbook bits you can steal

- Data gravity is real. They’re moving 10 petabytes out of AWS S3 and chose [Pure Storage FlashBlade](https://www.purestorage.com/) as the replacement ([source](https://dev.37signals.com)). It’s top‑priority for observability because it stores critical data like Basecamp customer attachments and long‑term Prometheus metrics, and its filesystem features enable uses like database backups ([source](https://dev.37signals.com)). If storage is critical, monitor it like it is.

- They open‑sourced [Action Push Native](https://dev.37signals.com), a Rails gem for sending push notifications to Apple and Google as part of moving off Amazon SNS and Pinpoint. It’s used by Basecamp and HEY to send more than 10 million push notifications per day, relying on HTTP‑based calls ([source](https://dev.37signals.com)). Self‑host the glue, not the world.

- On the UI side, [Lexxy](https://dev.37signals.com) upgrades Rails’ Action Text with a rich editor built on Meta’s [Lexical](https://lexical.dev/), better HTML semantics (actual p tags for paragraphs), Markdown shortcuts/auto‑formatting on paste, real‑time code syntax highlighting, paste‑to‑link, configurable prompts for mentions and interactive features, and inline previews for PDFs/videos ([source](https://dev.37signals.com)). Developer experience matters.

- If you’re doing native apps, [Hotwire Native](https://dev.37signals.com) is a web‑first framework for building them, and the 1.2 release adds “route decision handlers” so internal URLs go to in‑app screens and external URLs hop to the device browser ([source](https://dev.37signals.com)). Clean routing without reinventing navigation.

All very indie‑friendly. Just pick the pieces that fit your app and move.


## Quick Reference

- [Havn](https://hnrss.org) — One‑command local service map; scans ports, fingerprints HTTP, infers project names, and checks Redis/Postgres health.
- [tmuxy](https://hnrss.org) — GUI for [tmux](https://github.com/tmux/tmux) using control mode; Rust backend; web and desktop frontends.
- [ttylag](https://hnrss.org) — Userspace PTY wrapper to simulate RTT/jitter; wraps any command; `--bits-per-byte`; macOS and probably Linux.
- [Docker](https://www.docker.com/) — Containerization layer used with [Kamal](https://github.com/basecamp/kamal) by 37signals ([source](https://dev.37signals.com)).
- [Harbor](https://goharbor.io/) — Self‑hosted Docker registry; see 37signals’ registry story ([source](https://dev.37signals.com)).
- [Active Record Tenanted](https://dev.37signals.com) — Rails gem to support per‑tenant SQLite multi‑tenancy; adds safeguards.
- [Upright](https://dev.37signals.com) — Open‑sourced synthetic monitoring; Rails engine; checks from multiple locations; exposes [Prometheus](https://prometheus.io/) metrics; works with [Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) and [Grafana](https://grafana.com/).
- [Trawl](https://hnrss.org) — Natural‑language web scraping; LLMs for structure, [Go](https://go.dev/) for throughput; example: `trawl "https://books.toscrape.com" --fields "title, price, rating, in_stock"`.
- [RAG‑Ready Extractor](https://github.com/CarlosManuelDiaz/rag-ready-extractor) — Structure‑aware ingestion with semantic scoring.
- [Moruk OS](https://hnrss.org) — Local Linux autonomous agent; decomposes tasks; executes code; browses; learns; multi‑model; vector store + [SQLite](https://www.sqlite.org/); DeepThink review layer; drop‑in plugin system.
- [BunkerVM](https://github.com/ashishgituser/bunkervm) — Secure microVM sandbox runtime for agents.
- [AvaKill](https://github.com/log-bell/avakill) — Deterministic sub‑millisecond safety firewall for agents; no ML.
- [SkyClaw](https://github.com/nagisanzenin/skyclaw) — Rust LLM agent runtime; self‑healing; task checkpointing.
- [Country Cockpit](https://hnrss.org) — Country data dashboard (screenshots pre‑launch); front: [Next.js](https://nextjs.org/), [React](https://react.dev/), [D3.js](https://d3js.org/); back: [FastAPI](https://fastapi.tiangolo.com/), [SQLite](https://www.sqlite.org/).
- [OpenTelemetry Declarative Config](https://opentelemetry.io) — Stable JSON schema (v1.0.0), YAML/in‑memory models, ConfigProperties, PluginComponentProvider; SDK parses YAML and instantiates components.
- [OTTL context inference (Filter Processor)](https://opentelemetry.io) — v0.146.0 adds trace_conditions, metric_conditions, log_conditions, profile_conditions.
- [Unroll Processor](https://opentelemetry.io) — Expands bundled log events (e.g., JSON array) into individual log records.
- [Zipkin Exporter Deprecation](https://opentelemetry.io) — Zipkin exporter spec sunset; use Zipkin’s OTLP ingestion instead.
- [Pure Storage FlashBlade](https://www.purestorage.com/) — 37signals’ S3 replacement for 10 PB migration; critical observability target ([source](https://dev.37signals.com)).
- [Action Push Native](https://dev.37signals.com) — Rails gem for Apple/Google push; used for >10M pushes/day via HTTP; part of cloud exit.
- [Lexxy](https://dev.37signals.com) — Action Text editor based on [Lexical](https://lexical.dev/); Markdown, code highlighting, link on paste, inline previews.
- [Hotwire Native 1.2](https://dev.37signals.com) — Route decision handlers to send internal URLs in‑app and external to browser.
- [Hugging Face – glassballai](https://huggingface.co/datasets/louidev/glassballai) — 38‑day log of Gemini stock predictions to study LLM drift.


## Wrap‑up

By the time you wire this up, you’ve got more than a pile of tools — you’ve got a pattern. A single VPS running tmuxy and Havn becomes your cockpit. [Harbor](https://goharbor.io/) gives you a registry you control, wired into a Rails app that’s happy on [SQLite](https://www.sqlite.org/) or something bigger. [Upright](https://dev.37signals.com) and [Prometheus](https://prometheus.io/) watch your app from the outside, while [OpenTelemetry](https://opentelemetry.io/) gives you a clean, declarative way to see what’s happening on the inside. And the AI helpers — [Trawl](https://hnrss.org)‑style scraping, [RAG‑Ready Extractor](https://github.com/CarlosManuelDiaz/rag-ready-extractor) ingestion, local agents hardened by [BunkerVM](https://github.com/ashishgituser/bunkervm) and [AvaKill](https://github.com/log-bell/avakill) — all run on your boxes.

Start small: get the dev box and monitoring in place. Then pull your registry in‑house. Then layer on AI workflows and OTel processors. You’ll end up with a self‑hosted, AI‑assisted setup that’s cheap, understandable, and very hard to take away from you.

Ship it.
