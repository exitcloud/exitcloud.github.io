---
title: "From Dead SaaS to Your Own Stack: Self‑Hosted AI Dev Workflows for Indie Teams"
date: 2026-03-17T14:06:56.107Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# From Dead SaaS to Your Own Stack: Self‑Hosted AI Dev Workflows for Indie Teams

The night Booklore vanished — Discord gone, GitHub 404 — I realized my entire reading workflow was running on someone else’s laptop. One user noticed the Booklore Discord vanished and the repo 404’d. Another asked what happened after seeing the same. Poof. No export, no fork, no plan B.

Here’s the thing — if your tools matter, run them yourself. This guide shows how to ship a tiny, opinionated stack that keeps your AI dev workflow local, fast, and boring: an agent that respects your repo, a UI both you and the model can drive, backups that don’t require wizard robes, and monitoring that won’t eat your weekend.

Not theory. Real configs and commands. Ship it.

## 1. Why your AI dev workflow should live on your own metal

Vendor drift is a tax. That Booklore blip wasn’t the first, and it won’t be the last. A Redditor’s line stuck with me: “Remember, love AI‑made apps… they disappear faster than they launch.” Source. That’s not cynicism — it’s a checklist. Ask: can I rebuild this tool from git in an afternoon? If the answer is no, you’re renting a dependency you can’t control.

And you don’t need fancy hardware to take control. Someone repurposed an old laptop with a dead screen and keyboard into an Ubuntu server, planning to host a personal site on it. In the process, a 1TB HDD on a SATA–USB adapter sparked, smoked, died. They asked what caused it and how to keep the server from overheating. Story here and the follow‑up with the failure details here. Lesson: you can run real stuff on junk hardware — just budget time for basic safety and cooling. Small stacks, small blast radius.

AI tools also love to increase what one post calls “attention cost.” The subtitle of “The Attention Debt of AI Tooling” says AI tools can increase attention cost. [Read it](https://www.wespiser.com/posts/2026-03-15-Attention-Debt-Of-AI-Tooling.html). Jumping between four SaaS dashboards, two chats, and a browser plugin? That’s attention debt. Fixable: pull AI into your editor, your CLI, and a couple of tight internal tools instead.

Zoom out and you’ll notice the homelab community talking sovereignty and resilience again. A post highlighted Reticulum as a “physical layer agnostic mesh networking protocol,” alongside projects like Meshtastic and Meshcore. Even if you never touch mesh networking, the vibe matters — don’t build critical workflows on sand.

Let me show you what actually works.

## 2. Step 1 – Spin up a self‑hosted AI dev agent that respects your repo

“Your persona: you are a senior engineer.” We’ve all tried that. It flops in real projects. The [Oh‑my‑agent](https://github.com/first-fluke/oh-my-agent) project spells out why: hallucinations, drifting off task, repeating mistakes, missing critical library versions, vague personas, and prompts that burn tokens without improving outcomes. They propose a structured protocol to keep agents aligned with real constraints, including a Clarification protocol. Translation: stop whispering vibes to the model; give it a harness.

Start with specs your agent and your future self can read. [Specifica](https://specifica.org/) is an open format for writing software specs in Markdown. Drop two files in your repo:

- ARCHITECTURE.md — components, data flow, constraints. Short and blunt.
- WORKFLOW.md — how to build, test, release. Also short.

Then wire your “agent” to those artifacts. You don’t need a sprawling framework. A Makefile is enough.

Example make targets:

```makefile
# Makefile (keep it boring)
SHELL := /bin/bash
.DEFAULT_GOAL := help

help:
	@echo "targets: spec-check, plan, apply, test, fmt, health"

# Validate that required docs exist
spec-check:
	@test -f ARCHITECTURE.md
	@test -f WORKFLOW.md

# Use your agent to plan a change. Route context from your repo + specs.
plan: spec-check
	./scripts/agent_task plan --context ARCHITECTURE.md WORKFLOW.md src/ > .agent/plan.txt
	@echo "Plan saved to .agent/plan.txt"

# Apply the plan (explicitly)
apply:
	./scripts/agent_task apply --plan .agent/plan.txt

# Fast feedback loops
test:
	./scripts/run_tests

fmt:
	./scripts/format

health:
	curl -fsS http://127.0.0.1:8080/health || (echo "unhealthy" && exit 1)
```

Those scripts are yours. Keep them small. The agent task can call out to whatever toolchain you like — the point is to turn AI into a subprocess that reads specs, manipulates code, and proposes diffs you can review.

Here’s where it gets interesting. There’s a persistent rumor that MCP is “dead.” A discussion argues that people missed a major feature in Claude Code’s January 2026 release: Dynamic Tool Registration. It lets an MCP server add/remove tools in real time during a stateful session, instead of one static list like a glorified Swagger document. [HN thread](https://news.ycombinator.com/item?id=47412133). That means your agent can “load” abilities only when needed. Smaller context. Less noise.

So: pair your harness with an MCP server that supports dynamic tool registration. Start simple — one MCP tool, one job:

- [Clawdown](https://clawdown.app/) is a “PDF exporter with MCP and a judgy pixel cat.” Great for turning your spec or a release note into a PDF on demand.

Hook your agent process to the MCP server endpoint. Don’t ship 20 tools. Load on demand, then unload. The tool list stops being a sprawling kitchen sink and becomes a set of swappable modules.

Give the agent a memory that survives the weekend. [Mengram](https://mengram.io) bills itself as an “open‑source memory layer for AI agents.” Self‑host it. Wire your agent subprocess to read/write key decisions there: “We pinned library X to avoid bug Y,” “API quota reset at midnight,” “Feature flag Z is off in staging.” Now the model doesn’t need to re‑infer shop lore every run.

Grab only the files you actually need from template repos or examples. ghgrab is a tiny CLI that lets you browse a GitHub repo and download selected files or folders without cloning the whole repo. It supports fast search, batch downloading, and Git LFS. Install it however you like:

- cargo install ghgrab
- npm i -g ghgrab
- pipx install ghgrab

Repo lives at https://github.com/abhixdd/ghgrab. I use it to pull only the scaffolding I want (e.g., a sample ARCHITECTURE.md) instead of dragging 200MB of toy code into my tree. Less yak shaving.

Looking around corners? Two projects to keep an eye on:

- [Fabro](https://github.com/fabro-sh/fabro) calls itself an open‑source “dark software factory.” Opinionated automation vibes. Worth watching as an AI‑centric build pipeline story.
- [X07](https://x07lang.org/) is a compiled language pitched as enabling agents to write correct code on the first try. Big claim, interesting direction.

All three — [Fabro](https://news.ycombinator.com/item?id=47411909), [Oh‑my‑agent](https://news.ycombinator.com/item?id=47411909), and [MUP](https://news.ycombinator.com/item?id=47411909) — showed up as Show HN posts, which tells you the community’s hungry for open, glueable pieces. Not walled gardens.

Contrarian take? You don’t need an agent framework. You need backups and a Makefile. The [Oh‑my‑agent](https://github.com/first-fluke/oh-my-agent) critique (hallucinations, drift, vague personas) plus the “Attention Debt” post suggest that adding more orchestration layers often adds cognitive load. An MCP server with dynamic tools, a memory layer, and a handful of make targets puts you back in control.

## 3. Step 2 – Build a shared UI where you and your agent push the same buttons

Chat‑only UX hides affordances and burns attention. A better pattern is a UI the model and the human both operate — same actions, same state, visible to both.

Enter [MUP (Model UI Protocol)](https://github.com/Ricky610329/mup). Each MUP is a single .html file, and the same functions can be triggered either by the user (button clicks) or by the LLM (function calls). Both sides see each other’s actions in real time. The repo ships a proof‑of‑concept host and nine example MUPs. There’s a demo mode that works without an API key, and adding an OpenAI key enables full LLM–UI collaboration. Demo videos show playful cases like drawing pixel art.

Here’s how to turn that into your dev control panel:

- Fetch the repo and run the proof‑of‑concept host in demo mode to see the shape. No key needed to explore.
- Create a tiny MUP that wraps a subset of your workflow: run tests, format code, kick off an “agent plan.” Because a MUP is a single HTML file, keep it tight — two or three buttons that call your Makefile targets via a local endpoint. The model can press the same buttons via function calls. You’ll both see what happened.
- When you’re ready, add your OpenAI key so the LLM can participate. Now “run tests” is both a click and a function call.
- Study one or two of the nine examples (pixel art, a drum machine, etc.). The patterns transfer: single‑file UI, explicit state, deterministic actions.

You can tuck an MCP tool behind a button too. Example: “Generate PDF” → calls your MCP server for [Clawdown](https://clawdown.app/). The agent might press it after producing release notes. You might press it before a client call. Same result, same history.

This pattern generalizes. [Meddle](https://www.meddleconnect.com/en) describes itself as an AI‑powered industrial IoT platform for small manufacturers. Different domain, same idea: domain‑specific UIs fronting agent actions. It’s a good mental model for internal tooling.

Unique insight: the missing layer is structure, not smarter prompts. [Oh‑my‑agent](https://github.com/first-fluke/oh-my-agent) calls out vague personas and bloated prompts. The “Attention Debt” post says tools can increase attention cost. [Specifica](https://specifica.org/) and [Mengram](https://mengram.io) add structure — formal specs and durable memory. Put constraints and context in versioned artifacts and storage the agent consumes. Prompts get smaller. Mistakes become data. Better architecture.

## 4. Step 3 – Make it durable: backups, storage layout, and resilience for small stacks

Backups should be boring. Central place to manage jobs. Tiny agents near your databases. No exposing DBs to the public internet. Portabase aims for exactly that: it’s an open‑source, self‑hosted backup/restore tool with a central server and lightweight agents running close to your databases (for example via Portainer). They’ve added backup support for Redis and Valkey (backup only at the time of the post), and they’re working on Microsoft SQL Server support, asking for input from folks with MSSQL experience in production.

A sane layout for a 1–3 node indie stack:

- Keep latency‑sensitive DB volumes on the same node as the DB. A Redditor running multi‑node Docker with centralized NFS for all volumes reported latency and I/O issues for PostgreSQL and MariaDB. Their post. They said it works well for stateless services, but DBs suffered. Not surprising.
- Centralize configs, media, and other slow‑changing assets if you like. DBs get local disks. Your backups pull from the local agents back to a central store via Portabase’s approach — central server, lightweight agents, no public exposure.
- Container rescheduling is the tradeoff. If a DB container moves to another node, so must its storage, or you accept downtime while restoring from backup. That’s okay for tiny stacks. Make the failure modes explicit.

Cron is still my favorite monitoring sometimes. Add a cron job that curls your health endpoint and pages you via whatever messenger you already use. Keep it dumb:

```bash
# /etc/cron.d/healthchecks
* * * * * root curl -fsS http://127.0.0.1:8080/health || logger -p daemon.err "agent health failed"
```

Hardware safety? People run real servers on old laptops — we saw one. If you’re reusing gear, give it airflow, use quality power and storage adapters, and keep an eye on temps. That Redditor’s SATA–USB mishap (sparks, burning smell, drive destroyed) is a reminder that cheap adapters can be a footgun. Their follow‑up. Don’t ignore it.

Resilience experiments: if this stuff runs a small company or needs to survive your ISP hiccups, take a weekend to read about Reticulum and its cousins (Meshtastic, Meshcore). Mesh‑friendly protocols aren’t mainstream dev tooling, but they embody the “own your stack” ethos. Interesting for out‑of‑band control channels later.

Tie back to Booklore: your stack lives in your git repos, your Markdown specs, and backup jobs you control. If a tool dies, you redeploy. Nothing important is trapped in a closed service.

## 5. Step 4 – Add production‑ish monitoring and K8s skills on a single host

A homelab user runs media on Proxmox and uses Beszel today, but is considering [Zabbix](https://www.zabbix.com/), [checkmk](https://checkmk.com/), or a [Grafana](https://grafana.com/) + [Prometheus](https://prometheus.io/) stack to aim at DevOps/SRE roles; they’ve heard checkmk enterprise is relatively easy to install and learn. Thread. Reasonable plan.

My move on a single server: pick one resume‑grade stack and ship it. Prometheus scraping a few HTTP health endpoints (your agent process, the [MUP](https://github.com/Ricky610329/mup) host, the Portabase server), then Grafana dashboards for CPU, RAM, disk. Keep it simple. Your future self will thank you on call.

If you want to get your hands slightly dirty with Kubernetes (I know, I know — you don’t need it for a blog), deploy a single service into a tiny cluster. Then read the CNCF production internals guide, “[When Kubernetes restarts your pod — And when it doesn’t](https://www.cncf.io/blog/2026/03/17/when-kubernetes-restarts-your-pod-and-when-it-doesnt/).” It says engineers often say “the pod restarted” to refer to four different situations, which leads to flawed runbooks and bad on‑call decisions. It’s verified against Kubernetes 1.35 GA and has a companion repo at github.com/opscart/k8s-pod-restart-mechanics. Build the mental model now; it pays off later.

On the observability side, the [OpenTelemetry](https://opentelemetry.io/blog/2026/k8s-semconv-rc/) team reports that Kubernetes attributes used by Collector processors like `k8sattributes` and `resourcedetection` have been promoted to release candidate status. Users can try the new Kubernetes semantic conventions schema via feature gates and provide feedback before the final stable release. If you’re playing with traces/metrics, flip the feature gates and see how it changes your labels.

As you scale (or if you just want to learn), policy becomes the next lever. [KyvernoCon](https://www.cncf.io/blog/2026/03/17/kubecon-cloudnativecon-europe-2026-co-located-event-deep-dive-kyvernocon/) launched in 2025 as the first event dedicated to the Kyverno community. That post says Kubernetes adoption is growing across enterprise platforms and AI‑driven workloads, and teams increasingly need automated guardrails for security, governance, and compliance. Even a tiny indie cluster benefits from a couple of policies (e.g., enforce labels). Guardrails aren’t just for banks.

The education angle matters. The creator of [D8A Academy](https://d8a.academy/) reports nine years of freelance data analyst experience and runs a Discord community of 700+ members. They built D8A Academy as a roadmap of projects to help aspiring data analysts focus on day‑to‑day work and build portfolios. Same energy here: use your stack as a lab. The skills are portable.

Wild stuff. But it works.

## Quick Reference

- Booklore Discord/repo disappearance — risk of relying on closed AI SaaS; second report.
- Ubuntu server repurpose + HDD failure story — old laptop reused; follow‑up failure details.
- [Attention Debt of AI Tooling](https://www.wespiser.com/posts/2026-03-15-Attention-Debt-Of-AI-Tooling.html) — claim: AI tools can increase attention cost.
- [Oh‑my‑agent](https://github.com/first-fluke/oh-my-agent) — structured protocol to reduce hallucinations, drift, repeated mistakes; includes a Clarification protocol.
- [Specifica](https://specifica.org/) — open format for software specs in Markdown (use for ARCHITECTURE.md/WORKFLOW.md).
- [MCP dynamic tool registration](https://news.ycombinator.com/item?id=47412133) — Claude Code Jan 2026 feature: add/remove tools in real time during stateful sessions.
- [Clawdown](https://clawdown.app/) — PDF exporter with MCP (and a judgy pixel cat).
- [Mengram](https://mengram.io) — open‑source memory layer for AI agents.
- [MUP (Model UI Protocol)](https://github.com/Ricky610329/mup) — single‑file HTML UIs inside LLM chat; POC host, nine examples, demo mode; API‑key mode enables LLM–UI collaboration.
- ghgrab — browse/download selected files/folders from any GitHub repo without cloning; install via cargo/npm/pipx; repo: https://github.com/abhixdd/ghgrab.
- Portabase — self‑hosted DB backup/restore; central server + lightweight agents; backup support added for Redis and Valkey; working on MSSQL.
- NFS latency story — centralized NFS fine for stateless; caused latency/I/O issues for PostgreSQL/MariaDB.
- Reticulum mesh networking — physical‑layer‑agnostic protocol tied to sovereignty/resilience; related projects: Meshtastic, Meshcore.
- Monitoring learning thread — Beszel today; considering Zabbix, checkmk, or Grafana+Prometheus for SRE skills; checkmk enterprise said to be easy to install/learn.
- [K8s pod restart internals](https://www.cncf.io/blog/2026/03/17/when-kubernetes-restarts-your-pod-and-when-it-doesnt/) — four meanings of “pod restarted”; verified against 1.35 GA; companion repo: github.com/opscart/k8s-pod-restart-mechanics.
- [OpenTelemetry K8s semconv RC](https://opentelemetry.io/blog/2026/k8s-semconv-rc/) — K8s attributes for `k8sattributes`/`resourcedetection` promoted to RC; try via feature gates.
- [KyvernoCon](https://www.cncf.io/blog/2026/03/17/kubecon-cloudnativecon-europe-2026-co-located-event-deep-dive-kyvernocon/) — first Kyverno community event launched in 2025; guardrails for security/governance/compliance as K8s and AI workloads grow.
- [Fabro](https://github.com/fabro-sh/fabro) — open‑source “dark software factory.”
- [X07](https://x07lang.org/) — compiled language aiming for agent‑written correct code.

## Conclusion

You just turned an old laptop and a few spare drives into something suspiciously like a tiny platform team: an AI dev agent with a real harness and memory, a shared UI that you and the model both drive, backups and storage that won’t crumble under a bad NFS choice, and monitoring that teaches you the same mental models SREs use on call. None of it tied to a single vendor’s goodwill. Your specs live in Markdown. Your agents talk open protocols like [MCP](https://news.ycombinator.com/item?id=47412133) and [MUP](https://github.com/Ricky610329/mup). Your backups are on your own disks.

Treat this as a living lab. Next weekend, add another MCP tool. The week after, port one service into a tiny cluster and read the pod restart guide. Maybe explore mesh or policy later. If another Booklore disappears tomorrow, your workflow doesn’t — because it’s running in your rack, in your repos, under your control.

I learned this the hard way. You don’t need Kubernetes. You need a VPS and a Makefile. Ship it.
