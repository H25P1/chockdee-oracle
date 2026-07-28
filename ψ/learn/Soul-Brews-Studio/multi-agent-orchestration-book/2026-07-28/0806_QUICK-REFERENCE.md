# Multi-Agent Orchestration Book — Quick Reference

**Source**: `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/multi-agent-orchestration-book`  
**Published**: https://soul-brews-studio.github.io/multi-agent-orchestration-book/

---

## How to Read This Book

### Online (Recommended)

The book is **published and fully accessible** at:
- **URL**: https://soul-brews-studio.github.io/multi-agent-orchestration-book/
- **Platform**: Docusaurus-hosted static site on GitHub Pages
- **Features**: Searchable, cross-linked, syntax-highlighted code examples, dark mode

### Locally (Offline)

Raw Markdown files are available at:
- **Chapter source**: `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/multi-agent-orchestration-book/chapters/`
- **Appendices**: `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/multi-agent-orchestration-book/appendices/`
- **Format**: Standard CommonMark—no special tools needed

### Local Development Server

To run the Docusaurus dev server locally:

```bash
cd /Users/h_wa/ghq/github.com/Soul-Brews-Studio/multi-agent-orchestration-book/docs-site
npm install
npm start
```

This starts a live-reload development server (typically on http://localhost:3000).

---

## What This Book Argues (One-Paragraph Summary)

This is a practitioner's field guide, not theory. Written from 100+ hours of real building (April 2026, session 4833f831), it documents three production-tested tiers of agent orchestration—in-process subagents, coordinated teams, and independent processes—each presented in order of increasing human visibility and operational complexity. The central thesis is: *"Convenience is for the AI. Visibility is for the human. The best system serves both."* Most multi-agent tooling optimizes for the AI at the expense of human maintainability; this book teaches when to use each tier, how to make the right tradeoffs, and how to keep humans in control of agent decisions. Every pattern includes real code that shipped, every failure has a git commit, and every cost model is derived from actual token counts.

---

## Who Should Read This and Why

### The Problem

Multi-agent systems are hard to ship because current tooling creates a visibility crisis. Agents are cheap and easy to spawn—but invisible to the humans who must review, debug, extend, and eventually kill them. This produces impressive demos and unshippable systems.

### Who Needs This

- **Multi-agent system builders** working in production or scaling to production
- **Teams shipping complex orchestration** across multiple agents with dependencies
- **Product engineers** who must understand agent decisions, failures, and costs
- **Architects** designing orchestration infrastructure (message buses, task tracking, federation)
- **Skeptics** who've seen hype-driven multi-agent projects fail (see Chapter 14: Failure Modes)

### What Problem It Solves

1. **How do I know which tier to use?** (Subagent vs. coordinated team vs. independent process)
2. **How do I keep orchestration visible and debuggable?**
3. **What are real failure modes, and how do I avoid them?**
4. **What does this actually cost?** (Cost analysis based on real token counts)
5. **How do I migrate frameworks with agents helping?** (Real example: Hono → Elysia migration with 21 API files)
6. **What plugin architecture actually works for production?** (WASM plugin runtime, graph-node inspired)

---

## Book Structure at a Glance

| Part | Chapters | Purpose |
|------|----------|---------|
| **I: Foundations** | 1–4 | Core concepts: why multi-agent, the three tiers, message bus, task tracking |
| **II: Patterns** | 5–9 | Five production patterns: Research Swarm, Architecture Debate, Implementation Team, Federation Agent, Cron Loop |
| **III: Infrastructure** | 10–12 | How to build it: plugin architecture, WASM runtime, framework migration with agents |
| **IV: The Human Factor** | 13–15 | What humans see, failure modes, and Tier 4 (the future) |

### Reading Paths

- **Linear** → Chapters 1–15 in order (full context)
- **Pattern-focused** → Jump to Part II (Chapters 5–9) for core orchestration patterns
- **Implementation-focused** → Start with Part III (Chapters 10–12) for infrastructure decisions
- **Skeptical** → Read Chapter 14 (Failure Modes) first; decide if the rest is worth your time
- **Reference-seeking** → Use Appendices A–D (Command Reference, Spawn Pattern Cheatsheet, Cost Analysis, Plugin Catalog)

---

## Key Chapters

### Part I: Foundations

| Chapter | Title | Key Insight |
|---------|-------|------------|
| 1 | Why One Agent Isn't Enough | Single-agent systems miss parallelism and specialization |
| 2 | The Three Tiers | In-process, coordinated, independent—pick the right tool |
| 3 | The Message Bus | How agents communicate reliably at scale |
| 4 | Task Tracking | Visibility into what agents are doing and why they failed |

### Part II: Patterns (The Core)

| Chapter | Pattern | When to Use |
|---------|---------|------------|
| 5 | The Research Swarm | Parallel exploration, gathering options before a decision |
| 6 | The Architecture Debate | Multiple agents proposing solutions; leader picks the best |
| 7 | The Implementation Team | Coordinated team where task completion depends on other tasks |
| 8 | The Federation Agent | Distributed agents across multiple processes or nodes |
| 9 | The Cron Loop | Recurring agent work on a schedule (monitoring, maintenance) |

### Part III: Infrastructure

| Chapter | Topic | Real Example |
|---------|-------|-------------|
| 10 | The Plugin Architecture | How maw-js v2.0 uses plugins to compose behavior |
| 11 | WASM Plugin Runtime | Sandboxed plugins (inspired by The Graph's graph-node) |
| 12 | Framework Migration With Agents | Hono → Elysia (21 API files, 76 routes, agents helping) |

### Part IV: The Human Factor

| Chapter | Topic | Core Message |
|---------|-------|-------------|
| 13 | What the Human Sees | Human-visible logging, tracing, and peek-ability |
| 14 | Failure Modes | Real war stories: what went wrong and how to fix it |
| 15 | The Future — Tier 4 | What comes next in agent orchestration |

---

## The Source Code

This book is based on **maw-js**, an open-source multi-agent workflow framework in Bun + TypeScript. During the April 2026 session that produced this book:

- **Version evolution**: v1.15.0 → v2.0.0-alpha.2
- **Scope**: 21 API files, 76 routes migrated, 17-plugin command catalog
- **Architecture**: WASM plugin runtime, federation protocol across 4 nodes
- **Testing**: 35 new tests, 3 deep-learn explorations (Elysia 123K docs, graph-node 126K docs)

**Every file path and commit hash in the book is real and reproducible.**

---

## Metadata

- **Written by**: Nat Weerawan + mawjs oracle (the maw-js team)
- **Session ID**: 4833f831 (Soul-Brews-Studio/mawjs-oracle)
- **Base code**: Soul-Brews-Studio/maw-js v2.0.0-alpha.2
- **License**: MIT
- **Repository**: https://github.com/Soul-Brews-Studio/multi-agent-orchestration-book

---

## Quick Links

| What | Where |
|------|-------|
| **Live book** | https://soul-brews-studio.github.io/multi-agent-orchestration-book/ |
| **GitHub repo** | https://github.com/Soul-Brews-Studio/multi-agent-orchestration-book |
| **maw-js framework** | https://github.com/Soul-Brews-Studio/maw-js |
| **Local source** | `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/multi-agent-orchestration-book/` |
| **Raw chapters** | `chapters/*.md` in the repo above |
| **Command reference** | `appendices/A_command-reference.md` |
| **Spawn patterns** | `appendices/B_spawn-pattern-cheatsheet.md` |
| **Cost analysis** | `appendices/C_cost-analysis.md` |
| **Plugin catalog** | `appendices/D_plugin-catalog.md` |
