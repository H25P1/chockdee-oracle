# Multi-Agent Orchestration Book: Architecture & Organization

**Repository**: `Soul-Brews-Studio/multi-agent-orchestration-book`
**Source Code**: maw-js v2.0.0-alpha.2 (Bun + TypeScript)
**Session**: 4833f831 (April 2026, ~100 hours active)
**Published**: [soul-brews-studio.github.io/multi-agent-orchestration-book](https://soul-brews-studio.github.io/multi-agent-orchestration-book/)

---

## 1. Executive Summary

This is a practitioner's field guide to multi-agent orchestration, written by a multi-agent team using the exact patterns it documents. It is not theoretical — every pattern in the book has production code, every failure has a git commit hash, every cost model is derived from actual token counts.

**The core thesis**: "Convenience is for the AI. Visibility is for the human. The best system serves both."

The book establishes three tiers of agent orchestration (Tier 1: in-process subagents; Tier 2: coordinated squads; Tier 3: independent processes) and documents five production-tested patterns for spawning agents, with real war stories, failure modes, and metrics from a single 100-hour session that evolved maw-js from v1.15.0 to v2.0.0-alpha.2.

**Target audience**: Engineers shipping multi-agent systems in production, particularly those using Claude Code or the Claude API with agent patterns.

**Problem solved**: How to coordinate multiple AI agents across different tiers of isolation and visibility, avoid context-collapse bugs, and maintain human observability throughout the system.

---

## 2. Repository Structure

```
multi-agent-orchestration-book/
├── README.md                    # Landing page with book metadata
├── .git/                        # Git history (38+ commits in this session)
├── .github/                     # GitHub Actions / CI config
│
├── chapters/                    # 15 core content chapters (numbered 01-15)
│   ├── 01_why-one-agent-isnt-enough.md
│   ├── 02_the-three-tiers.md
│   ├── 03_the-message-bus.md
│   ├── 04_task-tracking.md
│   ├── 05_the-research-swarm.md
│   ├── 06_the-architecture-debate.md
│   ├── 07_the-implementation-team.md
│   ├── 08_the-federation-agent.md
│   ├── 09_the-cron-loop.md
│   ├── 10_the-plugin-architecture.md
│   ├── 11_wasm-plugin-runtime.md
│   ├── 12_framework-migration-with-agents.md
│   ├── 13_what-the-human-sees.md
│   ├── 14_failure-modes.md
│   └── 15_the-future-tier-4.md
│
├── appendices/                  # Reference material (4 appendices)
│   ├── A_command-reference.md
│   ├── B_spawn-pattern-cheatsheet.md
│   ├── C_cost-analysis.md
│   └── D_plugin-catalog.md
│
├── origin/                      # Meta-documentation
│   └── 00_the-session.md        # How this book came to exist (33K words, 100-hour chronicle)
│
└── docs-site/                   # Publishing infrastructure (Docusaurus 3.10.0)
    ├── docusaurus.config.ts     # Build config, GitHub Pages publication
    ├── sidebars.ts              # Navigation structure
    ├── package.json             # Dependencies (React 19, TypeScript 5.6)
    ├── docs/                    # Mirrored/processed chapters for Docusaurus
    │   ├── ch01-why-one-agent-isnt-enough.md
    │   ├── ch02-the-three-tiers.md
    │   └── ... (mirror of chapters/)
    ├── src/                     # Custom CSS and React components
    ├── static/                  # Favicon, logo assets
    └── tsconfig.json
```

### Key structural notes

- **Canonical source**: chapter files live in `/chapters/` and `/appendices/`
- **Publishing destination**: `/docs-site/docs/` is synced from chapters (no manual edits there)
- **Meta-layer**: `/origin/00_the-session.md` documents the session that wrote the book — self-hosting documentation
- **Git as artifact**: The book is intentionally reproducible; every file path and commit hash can be resolved

---

## 3. Publishing Toolchain

**Framework**: Docusaurus 3.10.0 (static site generator for documentation)

**Build & Deployment**:
- `npm start` — local dev server
- `npm run build` — static HTML generation
- `npm run deploy` — publish to GitHub Pages (automatic via GitHub Actions)
- **URL**: `https://soul-brews-studio.github.io/multi-agent-orchestration-book/`

**Technology stack**:
- **Runtime**: Node 18+
- **Framework**: Docusaurus core + preset-classic
- **UI**: React 19.0.0 + react-dom
- **Styling**: Custom CSS in `/src/css/`, Prism syntax highlighting
- **Markdown rendering**: CommonMark (MDX disabled to handle TypeScript generics like `maw.fetch<T>()` in prose)

**Notable configuration**:
- Dark theme by default (respects `prefers-color-scheme`)
- Treats broken links and markdown links as warnings, not hard errors
- Sidebar-based navigation via `/docs-site/sidebars.ts`
- GitHub edit links point to source repository main branch
- Post-install hook runs patch-package (for dependency patches)

---

## 4. Full Table of Contents

### Part I: Foundations (Chapters 1-4)

**Chapter 1: Why One Agent Isn't Enough**
Diagnoses the context-collapse problem: at ~100 hours, a single agent will forget what it built earlier in the same session. Shows the math of token cost per session (~2K per turn, 200K total window) and three structural fixes (persist externally, parallelize, hand off).

**Chapter 2: The Three Tiers**
Introduces the three shapes of agent orchestration: Tier 1 (Agent tool — fast, in-process, invisible), Tier 2 (TeamCreate — coordinated squads with task tracking), Tier 3 (tmux federation — real OS processes, human-observable, cross-machine). Establishes the decision tree: research/debate < 5 min → Tier 1; coordinated work 5-30 min → Tier 2; long-running or cross-machine → Tier 3.

**Chapter 3: The Message Bus**
Covers the three communication transports: SendMessage (structured, in-process), `maw hey` (plain-text, cross-process), and Inbox (persistent, survives session death). War story: the tmux agent that never reported because the prompt said "report at the end" and the agent didn't think it was done.

**Chapter 4: Task Tracking**
Documents the TaskCreate/TaskUpdate/TaskList protocol that prevents merge conflicts in parallel work. Five states: pending → in_progress → completed, plus owner and blockedBy fields. Introduces the "lead-compiles pattern": agents work in isolated worktrees on separate branches; only the lead writes to main.

### Part II: Patterns (Chapters 5-9)

**Chapter 5: The Research Swarm**
The first pattern: 3-5 small agents in parallel, each with a narrow question, each returning a compressed report. Case study: `/learn --deep elysia` (five Haiku agents reading 123K of Elysia documentation in under 2 minutes). Rules: one question per agent, use Haiku unless reasoning required, ask for reports not dumps, write to disk and summarize.

**Chapter 6: The Architecture Debate**
Three Opus agents in adversarial roles: Advocate (for option A), Counter-Advocate (for option B), Architect (decides). Prevents rationalization bias in a single agent. Works because the Advocate and Counter-Advocate argue blind to each other, producing the strongest case for each side, then the Architect sees both cold.

**Chapter 7: The Implementation Team**
Tier 2 pattern for coordinated coding work across 3+ files. Four rules: (1) named roles, not "worker 1" and "worker 2"; (2) worktree isolation per agent; (3) only the lead writes to main; (4) every agent reports via TaskUpdate. Case study: wasm-hardening team (three Sonnet agents, three tasks, four-minute completion).

**Chapter 8: The Federation Agent**
Tier 3 pattern: real tmux sessions, real `claude` CLI processes, independent PIDs. The moment that triggered this chapter: Nat said "not in processmemory!" Four failed spawn attempts revealed five missing features. Final working pattern: bake `maw hey` reporting instruction directly into the prompt, monitor via `maw peek`.

**Chapter 9: The Cron Loop**
Agents that wake themselves on a schedule. Two shapes: `CronCreate` (fixed interval, external trigger) for periodic work; `ScheduleWakeup` (dynamic, self-triggered) for reactive work. Case study: 17 plugins built in 55 minutes via a 5-minute cron loop (first took 1 hour, seventeenth took 11 minutes — that delta is the architecture).

### Part III: Infrastructure (Chapters 10-12)

**Chapter 10: The Plugin Architecture**
The first plugin was 20 lines of hardcoded shell, with implicit assumptions about port, response shape, error handling. By plugin five, five different implementations of the same thing. Fix: a typed SDK (one source of truth for schemas) and `maw.fetch<T>()` escape hatch. All plugins use `import { maw } from "maw/sdk"`, zero `any`, real files not symlinks.

**Chapter 11: WASM Plugin Runtime**
Inspired by The Graph's graph-node architecture (52 host functions, versioned APIs, per-invocation isolation, gas metering). Extends the plugin catalog into WebAssembly. The host function bridge exposes the SDK in C-ABI clothing, allowing Rust-compiled plugins to call back into the orchestrator.

**Chapter 12: Framework Migration With Agents**
How to use agents to migrate a framework end-to-end: Hono → Elysia, 21 API files, 76 routes. War story: batch-migrated all 21 files before testing a single route. The `error()` function doesn't exist as an export in Elysia 1.4. Lesson learned: test one before scaling to many. Also: had 123K of Elysia documentation in the vault but didn't consult it during the bug.

### Part IV: The Human Factor (Chapters 13-15)

**Chapter 13: What the Human Sees**
The five corrections that bent the system toward honesty: (1) real processes, not in-memory agents; (2) copy files, don't symlink; (3) no `any`, no `unknown`; (4) no absolute import paths; (5) plugins are modules, not loose scripts. Introduces the war room: `maw peek` and `maw overview` give the human full visibility into agent state.

**Chapter 14: Failure Modes**
Five real failures from session 4833f831: (1) the silent agent (tmux agent that never reported), (2) merge conflicts (two subagents, one file), (3) the `error()` bug (batch migration gone wrong), (4) orphaned worktrees (agents died leaving branches), (5) cross-repo `maw wake` (feature that didn't work as advertised).

**Chapter 15: The Future — Tier 4**
Proposes combining the strengths of all three tiers: `maw wake --issue 317 --team` spawns a real tmux session (Tier 3 independence) while registering it in TeamCreate (Tier 2 coordination). The agent reports via both `maw hey` (federation-native) and SendMessage (structured). Best of all worlds; implementation incomplete.

### Appendices

**Appendix A: Command Reference**
Complete reference for every command in the book. Sections: `maw` CLI (wake, peek, overview, hey, inbox), tmux spawn pattern, Agent tool, TeamCreate protocol, TaskCreate/TaskUpdate/TaskList.

**Appendix B: Spawn Pattern Cheatsheet**
Decision flowchart and one-liners for all three tiers. Shutdown protocols that actually work. Condensed decision tree: < 5 min → Tier 1; 5-30 min coordinated → Tier 2; > 30 min or cross-machine → Tier 3.

**Appendix C: Cost Analysis**
Token counts and time costs per tier, derived from session 4833f831. Tier 1 multiplier: 3-7× cost of doing it yourself (subagent overhead). Tier 2: same per-agent cost as Tier 1 plus coordination overhead. Tier 3: higher setup cost but lower per-iteration cost for long-running work.

**Appendix D: Plugin Catalog**
All 17 plugins in maw-commands v2.0.0-alpha.2, with purpose, path, and key capabilities for each. Covers health/observability (doctor), development (feed, worktrees, triggers), operations (costs, logs, transport), and administrative (plugin, ping, status).

### Meta-Layer

**origin/00_the-session.md**
33K-word chronicle of session 4833f831. Five pivots, seven lessons, the AI diary (sanitized but honest), the climax (documentation becomes prophecy; prophecy becomes real). Documents how four agents coordinated via TeamCreate to write 14 chapters about coordination patterns — using TeamCreate.

---

## 5. The Source Code (maw-js v2.0.0-alpha.2)

**Repository**: `Soul-Brews-Studio/maw-js`
**License**: MIT
**Language**: TypeScript (Bun runtime)

### What shipped in this session

- **Framework migration**: Hono 1.x → Elysia 1.4 (76 routes across 21 files, 4-minute parallelized migration)
- **Plugin system**: 17 command plugins, all typed, zero `any`, using the maw SDK
- **Validation**: TypeBox schemas across all endpoints
- **Tests**: 0 → 35 passing (deep learn revealed testing patterns from graph-node)
- **Federation**: Protocol across 4 nodes (oracle-world, white, clinic-nat, mba)
- **WASM runtime**: Inspired by The Graph's graph-node architecture

### Architectural decisions reflected in the book

1. **One schema, two worlds**: `src/lib/schemas.ts` (134 lines) written once, consumed by HTTP validation, response inference, and SDK exports
2. **No symlinks**: All files copied via `copyFileSync`, never symlinked
3. **Typed SDK, not bloated**: One `maw.fetch<T>()` helper instead of 20+ endpoint wrappers
4. **No absolute paths**: All imports relative or package-relative
5. **Real processes over convenience**: Tier 3 federation with tmux, not in-process agents

---

## 6. How to Navigate This Book

**If you are...**

- **Reading linearly**: Follow Parts I → II → III → IV. Each part builds on the previous.
- **Looking for patterns**: Jump to Part II (Chapters 5-9) for the five core orchestration patterns.
- **Implementing a system**: Read Part III (Chapters 10-12) for infrastructure decisions that make patterns work.
- **Skeptical**: Start with Chapter 14 (Failure Modes). If you still want more, the rest is worth your time.
- **Needing quick reference**: Appendices A-D are standalone, indexed by task.
- **Curious about the meta-layer**: Read origin/00_the-session.md to understand how this book came to exist.

---

## 7. Key Themes

### Visibility Over Convenience

Every architectural decision trades AI convenience for human observability:
- Tier 1 is invisible but convenient. Tier 3 is visible but requires setup.
- Symlinks are convenient; copying files is visible and honest.
- In-process agents are convenient; tmux sessions are independently observable.

### Context as a Bottleneck

The 200K-token window is the hard boundary:
- Compaction loses details at ~1.5M total tokens
- A single agent cannot handle > 100 hours of work
- Parallelization is not optional; it is structural

### Failure Modes Are Features

Five real failures are documented, not hidden:
1. Silent agents (missing reporting instruction)
2. Merge conflicts (inadequate isolation)
3. Batch migration bugs (test one before many)
4. Orphaned worktrees (no shutdown protocol)
5. Cross-repo spawn (feature doesn't work as advertised)

Each has a root cause and a fix; most point to missing infrastructure.

### Self-Hosting Documentation

The book was written by the exact agent patterns it documents. At hour 57, one agent forgot its own work — the problem Chapter 1 diagnoses. By hour 97, the same session had spawned agents to solve it and document the solution. The diagnosis and the cure are both real, both from the same session.

---

## 8. File Statistics

- **Total chapters**: 15
- **Total appendices**: 4
- **Total words written**: 33,000+
- **Commits in this session**: 38 (maw-js) + 16 (maw-commands)
- **API routes migrated**: 76 across 21 files
- **Plugins shipped**: 17 (all typed, zero `any`)
- **Tests created**: 35 (from 0)
- **Deep-learns conducted**: 2 (Elysia 123K docs + graph-node 126K docs)
- **Team agents deployed**: 7 (3 WASM implementation + 4 book writers)
- **Issues closed**: 11 on maw-js

---

## 9. External Resources

- **Live book**: https://soul-brews-studio.github.io/multi-agent-orchestration-book/
- **Source repo**: https://github.com/Soul-Brews-Studio/multi-agent-orchestration-book
- **maw-js repo**: https://github.com/Soul-Brews-Studio/maw-js
- **maw-commands repo**: https://github.com/Soul-Brews-Studio/maw-commands (17 plugin catalog)

---

## 10. Authors

**Written by**: the maw-js team (Nat Weerawan + mawjs oracle)
**Session**: 4833f831 (Soul-Brews-Studio/mawjs-oracle)
**Based on code**: Soul-Brews-Studio/maw-js v2.0.0-alpha.2
**License**: MIT
**Published**: April 2026

Every code example is from public repositories. Every session metric is reproducible. All citations include git commit hashes.
