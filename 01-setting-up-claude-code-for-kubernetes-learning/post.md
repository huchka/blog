# Before Writing a Single Line of Code, I Spent a Session Configuring My AI Pair Programmer

*I'm learning Kubernetes by building a real application on GKE — and writing about it as I go. This series is part learning notes, part progress report, part "things I wish someone had told me." If you're on a similar journey, I hope these posts save you some time (or at least make you feel less alone in the confusion).*

*This is the first entry. I haven't written any application code yet. Instead, I spent my first session setting up the AI assistant that'll be my pair programmer throughout the project. Here's how that went.*

---

## The Project: FeedForge + This Blog

I have two parallel initiatives under one umbrella:

1. **FeedForge** — An RSS feed aggregator with AI-powered summarization, deployed on GKE. The app itself is useful, but the real goal is hands-on Kubernetes learning: node management, scheduling, Terraform IaC, CI/CD with Cloud Build — the whole stack.

2. **This blog** — Technical posts about what I actually learn building FeedForge. Not tutorials rewritten from docs, but real experience with real tradeoffs.

The workflow is straightforward: build something in FeedForge, learn something about Kubernetes, write about it. Claude Code assists with both — coding and drafting.

---

## Why Configure Before Coding?

Claude Code reads a file called `CLAUDE.md` at the root of your project. It acts as persistent instructions — think of it as a system prompt that lives in your repo. Every session picks it up automatically.

The default temptation is to dump everything into this file: project descriptions, directory trees, coding standards, deployment steps. I almost did that. My first draft for FeedForge's `CLAUDE.md` was **359 lines**.

Then I read two articles that changed my approach.

---

## What I Learned from the Community

### Source 1: CLAUDE.md as a Design Document

A Qiita article by [@nogataka](https://qiita.com/nogataka/items/1ad4e4ccaf47816c63e0) reframed `CLAUDE.md` entirely. The key insight: **CLAUDE.md is not a project manual. It's a design document for how you work with AI.**

The practical takeaways:

- **Three-tier hierarchy**: Global config (`~/.claude/CLAUDE.md`) for cross-cutting concerns like communication style. Project-level `CLAUDE.md` for tool-specific instructions. Conditional rules (`.claude/rules/*.md`) for file-pattern-scoped behavior.
- **Keep it under 200 lines**. Claude reliably follows about 150 instructions total, and the system prompt already consumes ~50 of that budget. Overloading `CLAUDE.md` dilutes the important signals.
- **Separate concerns ruthlessly**. Project descriptions belong in `README.md`. Development plans belong in `docs/`. Only actionable instructions for Claude belong in `CLAUDE.md`.

The article also covered a structured memory system and sub-agent architectures, but those are more relevant for complex multi-agent workflows. I took the configuration hierarchy and the "instructions only" philosophy.

### Source 2: The Karpathy Guidelines

A second Qiita article by [@kotai2003](https://qiita.com/kotai2003/items/e88e7c247c6cb70c57dc) packaged Andrej Karpathy's observations about LLM coding failures into four principles:

1. **Think before coding** — Surface assumptions before implementing. If multiple valid approaches exist, list options with tradeoffs. Don't silently pick one.
2. **Simplicity first** — Deliver only what was requested. No speculative features, no abstractions for single-use code, no error handling for impossible scenarios.
3. **Surgical changes** — Touch only what's needed. Don't "improve" adjacent code. Flag issues separately.
4. **Goal-driven execution** — Every task should have a clear verification step. Transform vague requests into testable goals.

These resonated because they address exactly the failure modes I'd already seen: Claude adding unrequested features, over-engineering simple functions, and silently making assumptions about requirements.

---

## The Configuration I Landed On

### Root CLAUDE.md (~55 lines)

This sits at `~/Projects/personal/CLAUDE.md` and applies to everything — FeedForge, blog drafts, any future projects.

It covers:
- **Who I am**: environment, tools available, GitHub username
- **Communication style**: peer-level senior engineer talk, conclusion first, no hedging
- **Core principles**: the four Karpathy-derived principles, adapted to my workflow
- **Blog conventions**: writing style, target audience, Medium publishing flow

Nothing project-specific lives here. Just "how to work with me" instructions.

### Project CLAUDE.md (~45 lines)

FeedForge's `CLAUDE.md` contains only what Claude can't figure out on its own:
- **Key infrastructure decisions** as a table (GKE Standard over Autopilot, zonal cluster, e2-medium nodes — and *why* each choice was made)
- **Safety rails** for dangerous commands: `NEVER terraform destroy without confirming`, `NEVER kubectl delete namespace on production`, `ALWAYS terraform plan before apply`
- **Conventions**: branch naming, commit message format, Docker best practices

What I removed from the original 359-line version:
- Directory structure (Claude reads the filesystem)
- Application description (moved to `README.md`)
- Cost budget and learning tracker (moved to `docs/plan.md`)
- Communication style (moved to root — it's not FeedForge-specific)

### Memory System

Claude Code has a built-in memory system that persists across sessions. I set up a minimal index with a user profile — enough for Claude to remember who I am and what I'm working on between sessions.

The memory system supports several types: user context, feedback (corrections and confirmations), project state, and references to external systems. I'm starting lean and will build it up as the project progresses.

---

## Key Design Decisions (and Why)

**Decision: Instructions only, not documentation.**

My first CLAUDE.md tried to be everything — project description, architecture diagram, cost tables, learning tracker. But Claude doesn't need a project description to write code. It reads the codebase. What it does need is: "use conventional commits" and "never run terraform destroy without asking."

**Decision: Global communication style.**

"Peer-level senior engineer communication. No hedging, no over-explaining basics." This goes in the root config, not per-project. I want the same interaction style whether I'm writing Terraform or drafting a blog post.

**Decision: Emphasis markers for safety rails.**

`NEVER` and `ALWAYS` in caps aren't just for readability. Community research suggests that emphasis markers measurably improve adherence for critical instructions. When the instruction is "don't destroy my production namespace," I want maximum compliance.

**Decision: Start minimal, iterate.**

I could have set up sub-agent architectures, skill files, elaborate memory taxonomies. But this is a learning project. The configuration should match the complexity of the work. I can add layers as I need them.

---

## What's Next

The configuration is done. Next session, I start building FeedForge — beginning with Phase 0: GCP foundation and GKE cluster setup via Terraform.

That will be the next blog post: standing up a GKE cluster from scratch, the Terraform modules involved, and what I learn about Kubernetes along the way.

If you're considering using Claude Code (or any AI assistant) for a serious project, my main takeaway: **spend the first session on configuration, not code.** It's like writing tests before features — it feels slow at first, but it prevents a class of problems that are expensive to fix later.

---

*This is the first post in a series about building FeedForge — an RSS feed aggregator on GKE — while learning Kubernetes from scratch. Follow along for practical, experience-based posts about Kubernetes, Terraform, GCP, and working with AI coding assistants.*

---

**References:**

- [CLAUDE.md Design Patterns (Japanese)](https://qiita.com/nogataka/items/1ad4e4ccaf47816c63e0) — Three-tier hierarchy, memory architecture, configuration philosophy
- [Karpathy Guidelines for AI-Assisted Coding (Japanese)](https://qiita.com/kotai2003/items/e88e7c247c6cb70c57dc) — Four core principles for controlling LLM coding behavior
- [Claude Code Documentation](https://docs.anthropic.com/en/docs/claude-code) — Official docs on CLAUDE.md, memory, and best practices
