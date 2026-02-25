---
title: "AI Coding Tools I Actually Use (And the Ones I Dropped)"
date: 2026-01-05T09:00:00Z
draft: false
tags:
  - "opinion"
  - "AI coding"
  - "indie-dev"
---

I've been using AI coding tools daily for over a year now. Not in a "let me try this for a weekend" way — in a "this is part of how I ship software" way. I've paid for most of them out of pocket. Some stuck. Some got uninstalled within a week. Here's the honest breakdown.

## The ones that stuck

### Claude Code

This is my daily driver. I run it in the terminal, point it at a codebase, and have a conversation about what I'm building. It reads files, writes files, runs commands. It's the closest thing to actual pair programming I've found.

Where it shines: refactoring across multiple files, writing tests for existing code, and explaining gnarly legacy code I inherited from past-me. I'll drop into a repo I haven't touched in three months and say "read the auth middleware and write integration tests for the token refresh flow" and get something 80% correct on the first pass.

Real example of a good interaction:

```
me: read src/billing/ and add stripe webhook handling for
    subscription.updated events. follow the pattern in
    invoice.paid handler.

claude: *reads 6 files, writes handler, adds route, updates types*
```

That saved me maybe 45 minutes. The code followed existing patterns because it could see them.

Real example of a bad interaction:

```
me: design the data model for a multi-tenant SaaS app
    with usage-based billing

claude: *produces something plausible but with 3 subtle
    footguns around tenant isolation*
```

I almost shipped a schema where soft deletes could leak data across tenants. Caught it in review, but still. Architecture is your job.

Cost: ~$20/month with the Max plan, or use the API and pay per token. For how much I use it, it's a bargain.

### GitHub Copilot

The autocomplete. The thing that finishes your lines. I've had this since the beta and it's become muscle memory. Tab, tab, tab.

It's best for the boring stuff. Writing a new Express route handler that looks like the last five? Copilot has it. Filling out a switch statement? Copilot. Writing the fifteenth test case that follows the same pattern? Copilot.

Where it falls apart: anything novel. If you're doing something the training data hasn't seen a thousand times, the suggestions get weird fast. I've seen it confidently autocomplete API calls to endpoints that don't exist.

Cost: $10/month. Worth it just for the autocomplete in test files alone.

### Cursor

I go back and forth on this one. It's VS Code with AI baked in deeper than Copilot goes. The Cmd+K inline editing is genuinely good — highlight some code, describe what you want changed, and it does a targeted edit.

What keeps me using it: the codebase-aware chat. It indexes your project and can answer questions like "where is the rate limiter configured?" without me grep-ing through files. For large codebases this is a real time saver.

What annoys me: it occasionally fights with extensions I already have. And the pricing tiers keep shifting. But the core editing experience is solid.

Cost: $20/month for Pro. I keep subscribing, canceling, resubscribing. That probably tells you something.

## The ones I dropped

### Aider

I really wanted to like Aider. It's open source, runs in the terminal, works with multiple LLM backends. The git integration is clever — it commits changes automatically so you can review diffs.

Why I stopped: the workflow felt clunky for my use case. It wants to own the conversation and the git history, and I'm particular about both. The auto-commits cluttered my log. And when it made mistakes across multiple files, unwinding them was more work than just writing the code myself.

That said — if you're on a budget and want to bring your own API key, Aider is a legit option. It's actively maintained and the community is good. Just wasn't for me.

### ChatGPT (for coding)

I mean the web UI, not the API. Copy-pasting code back and forth from a browser window is painful. It loses context constantly. The code it generates often doesn't match your actual project structure because it can't see your project.

I still use ChatGPT for explaining concepts, rubber-ducking architecture decisions, and writing docs. But for writing code that goes into a repo? No.

### Various "AI-first IDEs"

I tried a couple of the newer ones that launched in 2025 — the ones that want to replace your entire editor. Most felt like demos, not tools. Slow, buggy, missing basic editor features I rely on. I'll check back in a year.

## Where AI actually helps (and where it doesn't)

After a year of this, here's my mental model:

**AI is great for:**
- Boilerplate and repetitive code
- Writing tests (especially when the implementation already exists)
- Translating between formats (JSON to TypeScript types, SQL to ORM models)
- Quick scripts and one-off tools
- Code review and catching bugs
- Explaining unfamiliar codebases

**AI is still your job:**
- System architecture and data modeling
- Security decisions (auth flows, access control, encryption)
- Performance-critical code paths
- Choosing the right tool/library/pattern
- Knowing when NOT to build something

The pattern I've settled into: I design the system, sketch the interfaces, make the hard decisions. Then I use AI to fill in the implementation. It's like having a fast junior dev who knows every API but has zero taste.

## Cost comparison

Here's what I actually spend per month:

| Tool | Cost | Worth it? |
|------|------|-----------|
| Claude Code (Max plan) | $20 | Yes, daily driver |
| GitHub Copilot | $10 | Yes, autocomplete is muscle memory |
| Cursor Pro | $20 | Maybe, I keep flip-flopping |
| **Total** | **$50** | |

$50/month for tools that save me hours per week. As an indie dev billing clients or shipping my own products, that's an easy yes.

Could I get by with just Copilot and the free tier of Claude? Probably. But the paid tiers are where the real productivity gains are, and $50/month is less than one hour of consulting work.

## The honest take

AI coding tools made me faster at the things I was already decent at. They didn't make me a better architect or help me make better technical decisions. They didn't replace the need to understand what I'm building.

The biggest risk isn't that AI writes bad code. It's that you stop reading the code it writes. Review everything. Run the tests. Understand the diff before you commit it.

Ship faster, but ship intentionally.

## Cheat Sheet

| Tool | What it's good for | Cost | Verdict |
|------|-------------------|------|---------|
| [Claude Code](https://claude.ai) | Multi-file refactors, test writing, codebase Q&A | $20/mo | Daily driver, best for complex tasks |
| [GitHub Copilot](https://github.com/features/copilot) | Line-level autocomplete, repetitive patterns | $10/mo | Essential, pure muscle memory |
| [Cursor](https://cursor.com) | Inline editing (Cmd+K), codebase-aware chat | $20/mo | Good but not irreplaceable |
| [Aider](https://aider.chat) | Open source, BYO API key, git-integrated | Free + API costs | Solid option if you're on a budget |
| ChatGPT web UI | Concept explanation, rubber ducking, docs | $20/mo | Not for writing code in repos |
| **Rule of thumb** | AI for implementation, you for architecture | — | Always review the diff before committing |
| **Best combo on a budget** | Copilot + Claude free tier | $10/mo | Covers 80% of the value |
