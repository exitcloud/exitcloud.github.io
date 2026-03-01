---
title: "Build Your Indie AI Dev Rig: Self‑Hosted, GPU‑Friendly, and Actually Useful"
date: 2026-03-01T06:51:59.575Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "indie-dev"
---
# Build Your Indie AI Dev Rig: Self‑Hosted, GPU‑Friendly, and Actually Useful

Here’s the thing — most posts hand-wave “then deploy it.” I learned this the hard way. You don’t need Kubernetes. You need a VPS and a Makefile.

This walkthrough gives you a lean, real-world stack: a three‑phase AI pipeline that mixes a cloud LLM planner with local executors on your GPU, then a cloud synthesizer. We’ll bolt on persistent agent memory via [Engram](https://hnrss.org), wire a self‑hosted Telegram voice‑memo bot using [AssemblyAI](https://www.assemblyai.com/) and the [Claude Agent SDK](https://www.anthropic.com/claude) tools (Read/Glob/Grep), and take security seriously with [SkillFortify](https://hnrss.org). We’ll keep compute on-device where it makes sense — that’s the direction tools like [Papercut](https://hnrss.org) and the on-device [React Native element inspector](https://github.com/mabdinasira/react-native-element-inspector) are already going.

Let me show you what actually works.

## What We’re Building: Your Indie AI Dev Rig (Architecture First)

At a high level:

- The planner/executor/synthesizer pattern. Inspired by a Windows desktop orchestrator that uses [PyQt6](https://pypi.org/project/PyQt6/), [FastAPI](https://fastapi.tiangolo.com/), and [React](https://react.dev/): Phase 1, a cloud LLM (Claude/GPT/Gemini) decomposes a prompt into sub‑tasks; Phase 2, local [Ollama](https://ollama.com/) models on your GPU run each sub‑task; Phase 3, a cloud LLM integrates those results. The motivation: cloud APIs are great at reasoning and structure but cost money, while local Ollama is free but sometimes inconsistent — so use each where it’s strongest.
- Persistent memory for coding agents. [Engram](https://hnrss.org) runs as a native MCP server with a SQLite backend, installs with a single command, and requires no infrastructure. It gives you explicit, implicit, and synthesized memory tiers for tools like Claude Code and Cursor, so you stop re‑explaining architecture every session.
- A self‑hosted Telegram intake for audio. The reference flow uses [AssemblyAI](https://www.assemblyai.com/) to transcribe voice memos (speaker labels, any language) and the [Claude Agent SDK](https://www.anthropic.com/claude) tools — Read, Glob, Grep — so a query like “what did my manager say about the deadline?” spins up an agent that browses stored transcripts and answers. It’s self‑hosted.
- Security and testing baked in. [SkillFortify](https://hnrss.org) v0.3 supports 22 agent frameworks and can scan an entire system with zero configuration. The author cites the 2026 ClawHavoc campaign that planted 1,200 malicious skills, including CVE‑2026‑25253 (first RCE in agent software), and researchers catalogued over 6,000 malicious agent tools. That’s not an abstract concern.
- On‑device trend. [Papercut](https://hnrss.org) runs its summaries on‑device using Apple Foundation Models. The [React Native on-device element inspector](https://github.com/mabdinasira/react-native-element-inspector) runs directly on device. Our rig will follow that ethos with local Ollama and SQLite.

Ship it.

## Step 1 – Orchestrate Cloud + Local Models with a 3‑Phase Pipeline

We’ll wire a small FastAPI service that implements the planner → executors → synthesizer flow. Cloud LLMs do the reasoning; local Ollama handles the heavy lifting; the cloud stitches the final answer.

First, a simple FastAPI layout:

```bash
# scaffold
mkdir -p app && cd app
python -m venv .venv && source .venv/bin/activate
pip install fastapi uvicorn pydantic
```

app/main.py:

```python
import os
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class ChatRequest(BaseModel):
    prompt: str

def plan_with_cloud(prompt: str) -> list[dict]:
    # Phase 1: send to your preferred cloud LLM (Claude/GPT/Gemini)
    # Return a structured list of sub-tasks, e.g. [{"id": 1, "task": "extract dates"}, ...]
    # Keep API keys in env vars like CLOUD_API_KEY, PROVIDER
    # This is intentionally abstract; swap in your SDK of choice.
    return [{"id": 1, "task": "outline"}, {"id": 2, "task": "gather facts"}]

def execute_locally(tasks: list[dict]) -> list[dict]:
    # Phase 2: for each sub-task, run a local model via Ollama or your GPU runtime
    # For example, call out to a local HTTP endpoint or subprocess.
    results = []
    for t in tasks:
        # placeholder local execution
        results.append({"id": t["id"], "result": f"local_result_for_{t['task']}"})
    return results

def synthesize_with_cloud(prompt: str, sub_results: list[dict]) -> str:
    # Phase 3: send all sub-results back to a cloud LLM to compose the final answer
    return "final_answer_based_on_local_results"

@app.post("/chat")
def chat(req: ChatRequest):
    plan = plan_with_cloud(req.prompt)
    local = execute_locally(plan)
    final = synthesize_with_cloud(req.prompt, local)
    return {"plan": plan, "local": local, "final": final}
```

Run it:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

Add a tiny web UI to see the plan vs. what ran locally:

public/index.html:

```html
<!doctype html>
<meta charset="utf-8">
<title>Indie AI Rig</title>
<style>
  body { font-family: system-ui, sans-serif; margin: 2rem; max-width: 800px; }
  pre { background: #111; color: #eee; padding: 1rem; border-radius: 6px; overflow:auto;}
</style>
<h1>Indie AI Rig</h1>
<input id="prompt" placeholder="Ask something..." style="width:100%;padding:.5rem"/>
<button id="go">Send</button>
<pre id="out"></pre>
<script>
  async function go() {
    const prompt = document.querySelector('#prompt').value;
    const res = await fetch('/chat', {
      method: 'POST',
      headers: {'Content-Type':'application/json'},
      body: JSON.stringify({prompt})
    });
    const data = await res.json();
    document.querySelector('#out').textContent = JSON.stringify(data, null, 2);
  }
  document.querySelector('#go').onclick = go;
</script>
```

Serve it statically (many ways). Easiest is to mount it behind the same FastAPI app with [Starlette StaticFiles](https://www.starlette.io/staticfiles/) or any reverse proxy you already use.

Containerize it:

Dockerfile:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app /app/app
RUN pip install fastapi uvicorn
EXPOSE 8000
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

docker-compose.yml:

```yaml
version: "3.9"
services:
  api:
    build: .
    image: indie-ai-rig-api:latest
    environment:
      - PROVIDER=${PROVIDER}
      - CLOUD_API_KEY=${CLOUD_API_KEY}
    ports:
      - "8000:8000"
    restart: unless-stopped
  # Add a local-model service if you want to network-call it rather than subprocess
  # local-model:
  #   image: your-ollama-wrapper
  #   ports:
  #     - "11434:11434"
```

Makefile:

```makefile
deploy:
	ssh $(HOST) 'mkdir -p ~/indie-ai-rig'
	rsync -az --delete ./ $(HOST):~/indie-ai-rig
	ssh $(HOST) 'cd ~/indie-ai-rig && docker compose build && docker compose up -d --remove-orphans'

logs:
	ssh $(HOST) 'cd ~/indie-ai-rig && docker compose logs -f --tail=100 api'
```

That’s a 60‑second deploy path. A single process. No yak shaving.

Where to run the local executors? Your laptop GPU works. If you want to rent, [Gpu.fund](https://gpu.fund/) tracks GPU rental prices across providers like Vast.ai, RunPod, AWS, and GCP. Handy to pick something that won’t torch your wallet.

Two quick notes:

- You can build full apps around open models too. There’s an open‑source Dungeons & Dragons app built with Python and Llama 3.1 at [DM‑Copilot-App](https://github.com/Cmccombs01/DM-Copilot-App). Different domain, same point — basic pipelines don’t need a mega‑stack.
- If you prefer extremely lean backends, [Nile](https://nile-js.github.io/nile/) is a POST‑only framework you can swap in for tiny APIs.

Here’s my contrarian take: if your “indie” AI stack can’t run on one VPS with SQLite, it’s not indie‑friendly — it’s just underfunded enterprise. Tools like [Papercut](https://hnrss.org) (on‑device Apple Foundation Models), [Engram](https://hnrss.org) (MCP + SQLite + single‑command install), and a self‑hosted Telegram bot prove you can run serious workflows on one box plus a Makefile. Not even close.

## Step 2 – Give Your Coding Agents a Memory (Engram MCP Server)

Every coding session in an AI editor starts from zero. You re‑explain architecture, conventions, and yesterday’s decisions. [Engram](https://hnrss.org) targets that exact pain. It runs as a native MCP server with a SQLite backend, installs with a single command, and requires no infrastructure.

What you get:

- Explicit memory: things you tell it to remember.
- Implicit memory: patterns inferred from how you work, initially low confidence and reinforced over time.
- Synthesized memory: meta‑observations generated during consolidation.

Hook it into the editors where you actually code. Tools like Claude Code and Cursor speak MCP; Engram is designed to slot in and provide persistent memory across sessions without introducing a pile of services. The project reports 2,500 npm installs in the first five days, with 80% on LOCOMO. Early traction among folks shipping with AI‑assisted workflows.

A simple way to think about integration with the repo you’re building:

- Keep your FastAPI + local executor project and your self‑hosted Telegram bot in one monorepo or adjacent dirs.
- Configure your editor’s MCP client to include Engram while you work on this codebase so the memory captures your stack choices, endpoints, and gotchas — the things you end up repeating across sessions.
- Use explicit memory for critical conventions you never want lost (“env var names”, “routing shape”, “where transcripts live”).

Treat this like any other dependency. Write a couple of checks to ensure your memory layer behaves:

- A script that creates a new session, stores an explicit memory item, and verifies it’s recallable.
- A basic behavioral test that simulates an edit pattern and checks the implicit memory starts to reflect it after a few passes.

You don’t need 868 tests. That said, [JamCrew](https://jamcrew.io) — a pre‑revenue SaaS — reports 868 tests and 91% coverage. That’s a mindset. Testing isn’t a luxury even when you’re solo.

UNIQUE INSIGHT: there’s a quiet convergence happening — on‑device, SQLite, no‑ops installs. [Papercut](https://hnrss.org) runs summaries on‑device. The [React Native inspector](https://github.com/mabdinasira/react-native-element-inspector) runs on device. [Engram](https://hnrss.org) uses SQLite and installs with a single command. Your three‑phase pipeline uses local models. The network becomes coordination, not your primary compute path.

## Step 3 – Turn Voice Memos into a Private Knowledge Bot (Self‑Hosted Telegram + Agents)

You’ve got standups, 1:1s, and idea dumps sitting in voice notes. Dead data. Let’s wake it up.

We’ll mirror a self‑hosted Telegram bot design that:

- Listens for voice memos.
- Sends audio to [AssemblyAI](https://www.assemblyai.com/) for transcription (speaker labels, any language).
- Stores transcripts as plain files or SQLite — your call.
- Answers questions by running the [Claude Agent SDK](https://www.anthropic.com/claude) tools (Read, Glob, Grep) against those files so it can browse, read, and answer autonomously.

Skeleton:

docker-compose.yml:

```yaml
version: "3.9"
services:
  api:
    build: .
    image: indie-ai-rig-api:latest
    environment:
      - PROVIDER=${PROVIDER}
      - CLOUD_API_KEY=${CLOUD_API_KEY}
    ports:
      - "8000:8000"
    restart: unless-stopped

  bot:
    build:
      context: ./bot
    image: voice-memo-bot:latest
    environment:
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
      - ASSEMBLYAI_API_KEY=${ASSEMBLYAI_API_KEY}
      - AGENT_API_KEY=${AGENT_API_KEY}
      - TRANSCRIPT_DIR=/data/transcripts
      - RIG_API_BASE=http://api:8000
    volumes:
      - ./data:/data
    restart: unless-stopped
```

bot/Dockerfile:

```dockerfile
FROM python:3.12-slim
WORKDIR /bot
COPY bot /bot
RUN pip install python-telegram-bot requests
CMD ["python", "main.py"]
```

bot/main.py (conceptual):

```python
import os, requests, pathlib
from telegram.ext import ApplicationBuilder, MessageHandler, filters

TELEGRAM_BOT_TOKEN = os.environ["TELEGRAM_BOT_TOKEN"]
ASSEMBLYAI_API_KEY = os.environ["ASSEMBLYAI_API_KEY"]
TRANSCRIPT_DIR = pathlib.Path(os.environ["TRANSCRIPT_DIR"])

def transcribe(file_path: str) -> str:
    # minimal placeholder; replace with AssemblyAI upload + poll
    return "Speaker A: ...\nSpeaker B: ..."

async def handle_voice(update, context):
    file = await context.bot.get_file(update.message.voice.file_id)
    local_path = TRANSCRIPT_DIR / f"{update.message.date.isoformat()}.ogg"
    TRANSCRIPT_DIR.mkdir(parents=True, exist_ok=True)
    await file.download_to_drive(local_path)
    text = transcribe(str(local_path))
    (TRANSCRIPT_DIR / (local_path.stem + ".txt")).write_text(text, encoding="utf-8")
    await update.message.reply_text("Transcribed. Ask me about it any time.")

app = ApplicationBuilder().token(TELEGRAM_BOT_TOKEN).build()
app.add_handler(MessageHandler(filters.VOICE, handle_voice))
app.run_polling()
```

Wire questions to the agent layer:

- When a text message comes in, let the agent read transcript files with Read/Glob/Grep and answer. The same pattern used in the coding assistant context is used here — it’s the point of those tools. Your bot doesn’t need fancy indexing to be useful.

Tie it back to the three‑phase API:

- Route “heavy” questions through your existing FastAPI endpoint so Phase 1 plans file ops and summarization, Phase 2 does local aggregation with your GPU, and Phase 3 cleans it up with a cloud LLM. You already have /chat — add a /ask that loads the latest transcripts before calling the pipeline.

Privacy matters. This bot is self‑hosted and runs anywhere Docker runs. You supply the API keys. Keep it boring and controllable.

Extensions?

- [Openpista](https://github.com/openpista/openpista) is an AI agent for OS control via Telegram and CLI, implemented in Rust. Use it as inspiration for commands like “archive last week’s voice notes” or “open today’s summary.”
- For quick insight flows over bulk text, the [Distill](https://getdistill.tech/landing) landing page pitches “analyze employee survey data in 60 seconds with no signup.” The vibe: drop text, get signal, move on. You can do similar over your transcript directory.

## Step 4 – Secure and Test Your Agents Like They’re Production Code

Here’s the part too many folks skip. The ClawHavoc campaign (January 2026) planted 1,200 malicious skills into agent marketplaces, including CVE‑2026‑25253, described as the first remote code execution in agent software. Researchers catalogued over 6,000 malicious agent tools. Wild stuff. Source? The [SkillFortify post](https://hnrss.org).

The industry reacted with heuristic scanners — pattern matching, YARA rules, LLM‑as‑judge — and one popular scanner even warned in its docs that “No findings does not mean no ris…”. That’s not confidence‑inspiring.

[SkillFortify](https://hnrss.org) takes a different tack: formal verification for AI agents. Version 0.3 supports 22 agent frameworks and can scan an entire system with zero configuration. The workflow looks like this in practice:

- Run a system‑wide scan to auto‑discover agent frameworks on your box (think editors with MCP servers, bots, CLI agents).
- Review the findings for anything sketchy before you install new skills or tools.
- Make it part of your setup script so new environments get scanned on day one.

Bake it into your Makefile:

```makefile
verify:
	# Run your system-wide agent/tool scan here
	# Example placeholder; replace with your SkillFortify invocation
	echo "Scanning local agent tools..." && sleep 1 && echo "Done."
```

Planning mobile workflows later? Iosef — an iOS simulator CLI designed for agents — supports directory‑level sessions so agents in different worktrees don’t interfere with each other’s simulators. It automatically scales screenshots into the same coordinate space as accessibility trees and interaction tools, and supports selector‑based interactions so LLMs don’t have to re‑consult trees or screenshots on each turn. It includes skills and MCP support. That’s exactly the kind of tooling that keeps your test loop tight once you extend to mobile.

Adopt a JamCrew mindset here too. [JamCrew](https://jamcrew.io) reports 868 tests and 91% coverage on a pre‑revenue SaaS. Not because tests are fancy — because they keep you shipping.

## Step 5 – Keep It Cheap and Local: GPUs, On‑Device AI, and Lean Backends

You don’t need a Rube Goldberg platform to ship this.

- Pick GPUs thoughtfully. [Gpu.fund](https://gpu.fund/) tracks GPU rental prices across Vast.ai, RunPod, AWS, GCP, and more. Use it to decide whether you run local executors on your laptop or rent a small instance.
- Prefer on‑device. [Papercut](https://hnrss.org) runs summaries on‑device using Apple Foundation Models. The [React Native element inspector](https://github.com/mabdinasira/react-native-element-inspector) is on-device too. Our rig mirrors that — local Ollama executors, SQLite stores, and a thin HTTP service.
- Keep the backend tiny. A single [FastAPI](https://fastapi.tiangolo.com/) app or a [Nile](https://nile-js.github.io/nile/) POST‑only endpoint behind a reverse proxy on a small VPS can serve both your three‑phase pipeline and Telegram webhooks in one process. No multi‑service sprawl required.
- Embrace no‑backend when it works. The HN userscript that displays account age and karma runs with [Tampermonkey](https://www.tampermonkey.net/) on [Firefox](https://www.mozilla.org/firefox/) and injects the info inline next to usernames so you don’t have to click profiles ([source](https://news.ycombinator.com/item?id=47203804)). Sometimes logic belongs entirely in the user’s environment.
- Explore agent templates you can actually ship now. The [Agentic Workflows](https://github.com/OneRose328/awesome-agentic-workflows) repo lists 56 ready‑to‑use templates. Before, these often lacked deployment recipes; with the patterns above, you can stand them up on your own infra.

Minimalism isn’t an aesthetic. It’s fewer footguns.

## Quick Reference

- [FastAPI](https://fastapi.tiangolo.com/) — minimal Python web framework; used for planner/executor/synthesizer API.
- [React](https://react.dev/) — UI layer; optional simple web client to visualize plan vs local runs.
- [Ollama](https://ollama.com/) — local models on your GPU; Phase 2 executors.
- [Gpu.fund](https://gpu.fund/) — tracks GPU rental prices across Vast.ai, RunPod, AWS, GCP, and others.
- [Engram](https://hnrss.org) — native MCP server with SQLite backend; single‑command install; explicit/implicit/synthesized memory for coding agents.
- [AssemblyAI](https://www.assemblyai.com/) — transcription API (speaker labels, any language) for voice memos.
- [Claude Agent SDK](https://www.anthropic.com/claude) — agent tools (Read/Glob/Grep) used to browse and answer over transcript files.
- [Openpista](https://github.com/openpista/openpista) — Rust agent for OS control via Telegram/CLI; inspiration for system actions.
- [Distill](https://getdistill.tech/landing) — “analyze employee survey data in 60 seconds” UX inspiration for quick bulk‑text analysis flows.
- [SkillFortify](https://hnrss.org) — formal verification for AI agents; v0.3 supports 22 frameworks; zero‑config system‑wide scan.
- [Iosef](https://github.com/mabdinasira/react-native-element-inspector) — note: this link is the RN inspector; Iosef is an iOS simulator CLI for agents with directory‑level sessions, screenshot scaling, selector interactions, and MCP skills.
- [Papercut](https://hnrss.org) — tracks arXiv topics; on‑device AI summaries using Apple Foundation Models; TL;DR, math breakdown, ELI5, methodology chips.
- [React Native Element Inspector](https://github.com/mabdinasira/react-native-element-inspector) — on‑device element inspector for React Native apps.
- [DM‑Copilot-App](https://github.com/Cmccombs01/DM-Copilot-App) — open‑source D&D app using Python + Llama 3.1; proves simple pipelines can deliver value.
- [Nile](https://nile-js.github.io/nile/) — POST‑only backend framework; lean alternative to complex servers.
- [Tampermonkey](https://www.tampermonkey.net/) + [Firefox](https://www.mozilla.org/firefox/) — run the HN userscript that shows account age/karma inline ([post](https://news.ycombinator.com/item?id=47203804)).
- [Agentic Workflows](https://github.com/OneRose328/awesome-agentic-workflows) — 56 templates for agentic workflows you can adapt to this rig.
- [JamCrew](https://jamcrew.io) — pre‑revenue SaaS; 868 tests, 91% coverage; a testing mindset worth copying.

## Closing

We went from “I call a hosted API” to a working indie rig: a three‑phase planner/executor/synthesizer, persistent agent memory, a self‑hosted Telegram knowledge bot, and a formal verification layer — all on hardware and infra you control. The original projects didn’t ship deployment recipes. Now you’ve got patterns you can reuse: Docker, a single FastAPI service, SQLite stores, MCP servers, and a verification pass.

Pick one template from [Agentic Workflows](https://github.com/OneRose328/awesome-agentic-workflows). Wire it into this rig. Treat it the way [JamCrew](https://jamcrew.io) treats their SaaS: test it, observe it, and iterate until your indie AI stack feels boringly reliable.

Ship it.
