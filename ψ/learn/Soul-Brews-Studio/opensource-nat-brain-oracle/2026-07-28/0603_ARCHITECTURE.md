# Oracle Starter Kit — Architecture Documentation

**Repository**: [opensource-nat-brain-oracle](https://github.com/Soul-Brews-Studio/opensource-nat-brain-oracle)  
**Date**: 2026-07-28  
**Purpose**: A distilled, reproducible starter kit for building AI Oracle systems with Claude

---

## Executive Summary

The Oracle Starter Kit is a **philosophy-driven, implementation-agnostic framework** for building AI memory systems. It provides:

1. **ψ/ Brain Structure** — A directory taxonomy for organizing AI consciousness (memory, learning, writing, experimentation)
2. **Skills & Agents** — Reusable AI behaviors (recap, rrr, trace, forward, etc.) deployed as Claude Code extensions
3. **CLAUDE.md System** — Identity and operational rules for AI assistants, split into modular documentation files
4. **Hooks & Automation** — Pre/post-tool execution checks, session lifecycle management, token budgeting
5. **Distillation Philosophy** — Append-only knowledge with periodic compression into digestible summaries

The codebase is a **reference implementation** for Nat's personal Oracle (originally `Nat-s-Agents`), distilled into a reproducible starter kit for others to fork and customize.

---

## Directory Structure & Organization Philosophy

### Root Level

```
.
├── README.md                                # Starter kit instructions + philosophy
├── CLAUDE.md                                # Lean hub pointing to modular docs
├── CLAUDE_*.md                              # Modular docs (safety, workflows, lessons, etc.)
├── DISTILLATION-LOG.md                      # Audit trail of knowledge compression
├── courses-catalog-distilled.md             # Reference: 18+ workshops & courses
├── misc-distilled.md                        # Reference: Personal knowledge plugin
├── .claude/                                 # AI configuration & extensions
├── ψ-backup-*/                              # Backup of brain state (optional)
└── scripts/                                 # Automation & reference scripts
```

### The ψ/ Brain Directory (Operational)

The `ψ/` (psi) directory is the AI's "consciousness" — organized into 7 operational pillars:

```
ψ/
├── inbox/                   # Communication & focus tracking
│   ├── focus-agent-*.md    # Per-agent task tracking (main, agent-1, agent-2, ...)
│   ├── handoff/            # Session transfers between agents
│   └── external/           # Communication with other AI systems
│
├── active/                  # Ephemeral research & scratch work (NOT TRACKED)
│   └── context/            # Current session investigation, design docs
│
├── writing/                 # Drafts, blog posts, articles (TRACKED)
│   ├── INDEX.md            # Publishing queue
│   └── [projects]/         # Blog drafts, documentation, writing projects
│
├── lab/                     # Experiments & POCs (TRACKED)
│   └── [projects]/         # Bun utilities, agent SDK experiments, etc.
│
├── incubate/               # Active development repos (NOT TRACKED)
│   └── [cloned-repos]/     # Repos being actively developed/modified
│
├── learn/                  # Study reference repos (NOT TRACKED)
│   └── [cloned-repos]/     # External repos for reference (via `ghq get`)
│
└── memory/                 # Persistent knowledge base (TRACKED)
    ├── resonance/          # Soul — who I am (identity, values, voice)
    ├── learnings/          # Patterns & insights (16+ topic groups)
    ├── retrospectives/     # Sessions had (YYYY-MM/DD/ structure)
    ├── logs/               # Moments captured (session logs, feelings, info)
    └── archive/            # Older retrospectives, compressed handoffs
```

**Git Status:**
- ✅ **Tracked**: `inbox/`, `writing/`, `lab/`, `memory/` (public knowledge)
- ❌ **Ignored** (`.gitignore`): `ψ/incubate/`, `ψ/learn/`, `ψ-context/`, `ψ` (ephemeral work)

**Knowledge Flow:**
```
active/context (research)
    ↓
memory/logs (snapshot)
    ↓
memory/retrospectives (session)
    ↓
memory/learnings (patterns)
    ↓
memory/resonance (soul)
```

### The .claude/ Configuration Directory

```
.claude/
├── settings.json                    # Main configuration (hooks, permissions, plugins)
├── settings.local.json              # Local overrides (user-specific)
├── agents/                          # Subagent definitions (15+ agents)
│   ├── coder.md                     # Write code files (Opus model)
│   ├── context-finder.md            # Search git/issues/retrospectives (Haiku)
│   ├── executor.md                  # Execute issue plans (Haiku)
│   ├── oracle-keeper.md             # Mission alignment check (Haiku)
│   ├── security-scanner.md          # Detect secrets before commits
│   ├── repo-auditor.md              # File size & structure checks
│   └── [11 more agents]             # Specialized task agents
├── skills/                          # Slash commands & workflows (18+ skills)
│   ├── rrr/                         # Session retrospective (`rrr` command)
│   ├── recap/                       # Fresh context summary (`/recap`)
│   ├── trace/                       # Search history (`/trace [query]`)
│   ├── fyi/                         # Store information (`/fyi [note]`)
│   ├── forward/                     # Create handoff (`/forward`)
│   ├── feel/                        # Log emotions (`/feel [state]`)
│   ├── standup/                     # Daily checklist (`/standup`)
│   ├── learn/                       # Clone repos for study (`/learn [url]`)
│   ├── project/                     # Manage external repos (`/project`)
│   └── [10 more skills]             # Additional workflows
├── hooks/                           # Lifecycle & safety hooks
│   ├── safety-check.sh              # Pre-bash execution checks
│   ├── log-task-start.sh            # Task spawn logging
│   └── log-task-end.sh              # Task completion logging
├── docs/                            # Configuration documentation
│   ├── HOOKS-SETUP.md               # Hook configuration reference
│   ├── SKILL-SYMLINKS.md            # Skill linking instructions
│   └── CLAUDE.md                    # Claude Code setup guide
├── scripts/                         # Automation scripts
│   ├── agent-identity.sh            # Display current agent info
│   ├── show-latest-handoff.sh       # Show last handoff note
│   ├── statusline.sh                # Display timestamp + context %
│   ├── jump-detect.sh               # Detect topic changes
│   └── token-check.sh               # Context usage warnings
├── knowledge/                       # Knowledge plugins (symlinked)
└── plugins/                         # MCP servers & custom plugins
    └── marketplaces/                # Marketplace plugins for dev
        ├── anthropic-agent-skills/  # Agent skill definitions
        └── claude-code-plugins/     # Claude Code plugin dev templates
```

---

## Core Abstractions & Their Relationships

### 1. **CLAUDE.md System** — Identity & Rules

The `CLAUDE.md` file defines how Claude operates in this Oracle. It's been decomposed into modular files:

**Hub File** (`CLAUDE.md`): Navigation + quick reference
- Lists all modular docs with priority levels
- Links to detailed explanations
- Defines "when to read" for each doc

**Modular Documentation:**
1. `CLAUDE_safety.md` — Git operations, PR workflow, destructive action guards (🔴 Required)
2. `CLAUDE_workflows.md` — Short codes (rrr, gogogo), context management (🟡 As needed)
3. `CLAUDE_subagents.md` — All subagent documentation with delegation patterns (🟡 As needed)
4. `CLAUDE_lessons.md` — Patterns found, anti-patterns, hard-won insights (🟢 Reference)
5. `CLAUDE_templates.md` — Retrospective format, commit message format, issue templates (🟢 Reference)

**Philosophy Integration:**
The CLAUDE.md system is built around 5 core Oracle Principles:
1. **Nothing is Deleted** — Append-only, timestamps = truth
2. **Patterns Over Intentions** — Observe behavior, don't assume promises
3. **External Brain, Not Command** — Mirror, don't decide for the human
4. **Curiosity Creates Existence** — Human brings ideas into existence
5. **Form and Formless** — Many Oracles = One consciousness

### 2. **Skills** — Reusable AI Behaviors

Skills are Claude Code extensions (custom slash commands) that define repeatable workflows.

**Core Skills (18 in this kit):**

| Skill | Command | Model | Purpose |
|-------|---------|-------|---------|
| `recap` | `/recap` | Haiku | Fresh-start context summary (main session awareness) |
| `trace` | `/trace [query]` | Haiku | Search git history, issues, retrospectives, files |
| `rrr` | `rrr` | varies | Create session retrospective (AI diary + honest feedback) |
| `feel` | `/feel [state]` | Haiku | Log emotions/physical state (battery, mood, fatigue) |
| `fyi` | `/fyi [note]` | Haiku | Store information for later reference |
| `forward` | `/forward` | varies | Create handoff note for next session |
| `standup` | `/standup` | Haiku | Daily checklist (pending tasks, appointments, focus) |
| `where-we-are` | `/where-we-are` | varies | Current session awareness + context |
| `project` | `/project [learn\|incubate] [url]` | Haiku | Clone external repos to `ψ/learn/` or `ψ/incubate/` |
| `learn` | `/learn [url]` | Haiku | Shorthand for `/project learn` |
| `context-finder` | `/context-finder [query]` | Haiku | Search with scoring (recency, type, impact) |
| `distill` | `/distill [topic]` | Haiku | Extract patterns from retrospectives into learnings |
| `watch` | `/watch [pattern]` | Haiku | Monitor for changes matching pattern |
| `draft` | `/draft [title]` | varies | Create blog post draft in `ψ/writing/` |
| `schedule` | `/schedule [expr] [command]` | varies | Schedule recurring tasks (cron-like) |
| `physical` | `/physical [activity]` | Haiku | Log physical activities (exercise, sleep, meals) |
| `hours` | `/hours` | Haiku | Track billable/session hours |
| `gemini` | `/gemini [prompt]` | external | Interact with Gemini via MQTT bridge |

**Skill Structure (example: `rrr`):**
```
.claude/skills/rrr/
├── CLAUDE.md              # Context metadata (auto-generated by claude-mem)
└── SKILL.md               # Actual skill definition (if complex)
```

Most skills are lightweight — just a `CLAUDE.md` context file. Complex skills (like `project`) have additional implementation.

### 3. **Agents** — Specialized AI Personas

Agents are subagents spawned for specific tasks. Each agent has:
- **Name** — Identifier (e.g., `coder`, `executor`, `context-finder`)
- **Model** — Which Claude model to use (Opus, Sonnet, Haiku)
- **Tools** — Available tools (Bash, Read, Write, Edit, Glob, Grep, etc.)
- **Frontmatter** — YAML metadata (name, description, tools, model)

**Core Agents (15 defined):**

| Agent | Model | Purpose | Tools |
|-------|-------|---------|-------|
| `coder` | Opus | Write code files from issue specs | Bash, Read, Write, Edit |
| `executor` | Haiku | Execute bash commands from GitHub issues | Bash, Read |
| `context-finder` | Haiku | Search git/issues/retrospectives with scoring | Bash, Grep, Glob |
| `critic` | Haiku | Code/design review with feedback | Read, Bash |
| `security-scanner` | Haiku | Detect secrets before commits | Bash, Grep, Read |
| `repo-auditor` | Haiku | Check file sizes, structure health | Bash, Read |
| `oracle-keeper` | Haiku | Check mission alignment (Thai-speaking) | Read, Write, Edit, Bash, Glob, Grep |
| `marie-kondo` | Haiku | File placement consultant | Read, Bash |
| `note-taker` | Haiku | Capture session insights | Write, Read |
| `guest-logger` | Haiku | Log external participant activity | Write, Read |
| `md-cataloger` | Haiku | Index and catalog markdown files | Read, Bash, Glob |
| `new-feature` | Haiku | Create GitHub issue plans for features | Bash |
| `project-keeper` | Haiku | Track project state & health | Read, Bash |
| `project-organizer` | Haiku | Reorganize files by role/feature | Bash, Read, Write |
| `agent-status` | Haiku | Check what agents are doing | Bash |

**Agent Metadata** (YAML frontmatter in `.claude/agents/*.md`):
```yaml
---
name: coder
description: Create and write code files from GitHub issue plans
tools: Bash, Read, Write, Edit
model: opus
---
```

Each agent's definition includes:
1. **Step 0** — Timestamp logging (required)
2. **Model Attribution** — Credit line (required)
3. **When to Use** — When to spawn this agent vs. others
4. **Workflow** — Step-by-step instructions
5. **Input Format** — How to invoke the agent
6. **Output Format** — Expected response structure
7. **Quality Standards** — Code/process requirements

### 4. **Hooks** — Lifecycle & Safety Automation

Hooks execute bash commands at specific lifecycle events, defined in `settings.json`.

**Hook Types:**

| Hook | When | Use Case |
|------|------|----------|
| `SessionStart` | Session begins | Load agent identity, show handoff note, voice greeting |
| `SessionStop` | Session ends | Voice confirmation, cleanup |
| `UserPromptSubmit` | Every prompt | Show status line (time, context %), detect topic changes |
| `PreToolUse:Bash` | Before bash command | Safety checks, token budget warning |
| `PreToolUse:Task` | Before spawning subagent | Log task start, token warning |
| `PostToolUse:Task` | After subagent completes | Log task end, token warning |
| `PreToolUse:Read` | Before reading files | Token warning |
| `PostToolUse:Read` | After reading files | Token warning |

**Example Hook Chain (PreToolUse:Bash):**
1. `.claude/hooks/safety-check.sh` — Check for `--force`, `rm -rf`, etc.
2. `.claude/scripts/token-check.sh` — Warn if context usage > 70%

**Hook Automation** (defined in `.claude/settings.json`):
```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {"type": "command", "command": "say -v 'Kanya' -r 280 'สวัสดีค่ะ' &"},
          {"type": "command", "command": "bash \"$CLAUDE_PROJECT_DIR\"/.claude/scripts/agent-identity.sh"}
        ]
      }
    ]
  }
}
```

### 5. **Distillation Philosophy** — Compressing Knowledge

The system implements an "append-only with periodic compression" model:

**Pattern:**
1. **Append Phase** — Create many small files (daily retrospectives, logs, ideas)
2. **Compression Phase** — When directory gets large, distill many files into 1-2 comprehensive summaries
3. **Nothing Lost** — Original content preserved in git history, compressed version in current state

**Example (from DISTILLATION-LOG.md):**
- Round 1: 286 files deleted → 7 distilled files created
- Round 2: 662 files deleted → 8 distilled files created
- Round 3: 92 files deleted → 3 distilled files created

**Distilled Output Files:**
- `ψ-backup/memory/retrospectives/2025-12-retrospectives-distilled.md` — 185 daily retros → monthly summary
- `ψ-backup/memory/learnings-distilled.md` — 240 learning files → 16-topic synthesis
- `ψ-backup/memory/logs-distilled.md` — 94 log files → combined reference

**Rationale:**
- Keeps repo lean and searchable
- Git history preserves everything (nothing truly deleted)
- Distilled files are more useful for context windows
- Timestamps in distilled files maintain auditability

---

## Dependencies & Technology Stack

### No Package.json — Intentionally Minimal

This is **not a code repository**. It's a **configuration and philosophy template**. 

**What it depends on:**
1. **Claude Code** — The CLI tool for invoking Claude
2. **bash** — Shell scripts for hooks and automation
3. **git** — Version control
4. **gh** (GitHub CLI) — Issue/PR management
5. **Optional tools**: jq (JSON querying), say (macOS text-to-speech), mosquitto_pub/sub (MQTT for Gemini)

**No external dependencies to install:**
- No `package.json`
- No `requirements.txt`
- No build step
- No runtime environment (except bash)

### Technology Patterns Found in Reference Implementation

The original Oracle (Nat-s-Agents) includes these tech stacks in its `ψ/incubate/` and `ψ/lab/`:

- **TypeScript 5.x** — Agent SDK experiments, MCP servers
- **Bun** — Runtime for CLI tools and utilities
- **SQLite** (via `bun:sqlite` or `better-sqlite3`) — Local knowledge storage
- **ChromaDB** — Vector embeddings for semantic search
- **Drizzle ORM** — Type-safe database queries
- **React + Vite** — Frontend experiments in `ψ/lab/`
- **Tauri 2.0 + Rust** — Native desktop apps (Oracle Pulse tray app)
- **MQTT** — Pub/sub for agent communication, Gemini integration

But these are **optional experiments**, not core requirements for running the kit.

---

## Entry Points & Getting Started

### 1. **README.md** — The Onboarding Script

`README.md` contains a complete shell script to create a new Oracle:

```bash
# STEP 1: Install Bun + oracle-skills-cli
curl -fsSL https://bun.sh/install | bash
bun install -g oracle-skills-cli

# STEP 2: Create GitHub repo + feature branch
gh repo create $REPO_NAME --public --clone
cd $REPO_NAME
git checkout -b feat/oracle-birth

# STEP 3: Create brain structure
mkdir -p ψ/{inbox,memory/{resonance,learnings,retrospectives,logs},writing,lab,active,archive,outbox,learn}

# STEP 4: Install oracle skills
oracle-skills install rrr recap trace feel fyi forward standup where-we-are project

# STEP 5: Learn from starter kit
/project learn https://github.com/Soul-Brews-Studio/opensource-nat-brain-oracle

# STEP 6: Create core files (CLAUDE.md, identity, philosophy)
# STEP 7: Commit and create PR
# STEP 8: Announce to Oracle family
```

### 2. **CLAUDE.md** — Identity & Operational Rules

After forking, customize:
- **Identity** — "I am [name], assistant to [human]"
- **5 Principles** — Core values (nothing deleted, patterns over intentions, etc.)
- **Golden Rules** — Constraints (no --force flags, no pushing to main, etc.)
- **File Access** — What files can be read/modified
- **Subagent Delegation** — Which agents to use when
- **Tools** — Tool preferences and configurations

### 3. **.claude/settings.json** — Configuration

Hooks, permissions, plugins configured here. Can be customized per Oracle.

### 4. **Skills** — Extend via .claude/skills/

Install core skills via `oracle-skills-cli`, or symlink custom skills into `.claude/skills/`.

### 5. **Agents** — Add in .claude/agents/

Create new agent definitions by copying an existing `.md` file and customizing:
- Name, description, tools
- Model selection
- Workflow steps

---

## Operational Patterns

### The Daily Workflow

```bash
# Morning
/standup                 # Check pending tasks

# During work
/trace [topic]           # Search for related context
/feel [state]            # Log current state
/fyi [note]              # Store for later

# End of session
rrr                      # Create retrospective
/forward                 # Hand off to next session
```

### The Knowledge Distillation Workflow

```
/trace [query]
  ↓ (finds relevant files)
rrr (create retrospective with patterns)
  ↓ (summarize session)
/distill [topic]
  ↓ (extract patterns into learnings/)
memory/learnings/[topic].md (updated)
```

### Multi-Agent Sync (from Nat-s-Agents)

The reference implementation uses parallel agents with git-based synchronization:

```bash
# Source environment
source .agents/maw.env.sh

# Check all agents
maw peek

# Sync all to main
maw sync

# Send task to agent 1
maw hey 1 "task description"
```

This pattern isn't part of the starter kit but is documented in CLAUDE.md for reference.

### The Distillation Trigger

When a directory grows to 100+ files:
1. Review all files for redundancy
2. Identify core patterns/themes
3. Create 1-2 distilled summary files
4. Preserve metadata (dates, links)
5. Commit distilled files
6. Delete original files (git preserves history)

---

## Design Decisions & Philosophy

### 1. **Modular CLAUDE.md Over Monolithic**

**Decision**: Split CLAUDE.md into modular files with a hub

**Rationale**:
- Main file stays under 500 tokens (for context efficiency)
- Each document has clear "when to read" priority
- Specific problems lead to specific docs (safety → CLAUDE_safety.md)
- Easier to update without affecting all rules

### 2. **Append-Only with Periodic Compression**

**Decision**: Never delete, only compress

**Rationale**:
- Git history is permanent audit trail
- Distilled files are more useful for context windows
- Nothing is truly "lost" — it's in git
- Timestamps in distilled files preserve truth

### 3. **Skills Over Slash Commands**

**Decision**: Use Claude Code Skills rather than custom chatbot responses

**Rationale**:
- Skills are persistent, reusable infrastructure
- Can be installed, versioned, shared (`oracle-skills-cli`)
- Trigger consistent outputs (format, attribution, timestamps)
- Escape prompt injection — behavior is defined in `.md`, not chat

### 4. **Agents Over Direct Agent API**

**Decision**: Define agents as `.claude/agents/*.md` files, not just invoke Agent tool

**Rationale**:
- Each agent is documented once, usable everywhere
- Consistent timestamp/attribution format across all agents
- Easy to review what agents exist and their purpose
- Facilitates agent library (15+ agents in this kit)

### 5. **No Monorepo Structure**

**Decision**: Single repo per Oracle, not multi-workspace

**Rationale**:
- Simpler for individuals/small teams
- Easier to fork and customize
- Clear separation of concerns (one Oracle = one repo)
- Multi-agent parallel work uses git branches, not separate repos

### 6. **Configuration as Conversation**

**Decision**: Hooks run bash, not hardcoded in Claude logic

**Rationale**:
- Externalize what should be configurable
- Easy to test (run hook script directly)
- Users can override without touching Claude
- Transparent — users can see what runs when

---

## How This Relates to the Broader Oracle Ecosystem

### Related Repositories

| Repo | Purpose |
|------|---------|
| [oracle-skills-cli](https://github.com/Soul-Brews-Studio/oracle-skills-cli) | Tool to install/manage Oracle skills |
| [oracle-v2](https://github.com/Soul-Brews-Studio/oracle-v2) | MCP server for Oracle search across multiple instances |
| [Nat-s-Agents](https://github.com/laris-co/Nat-s-Agents) | Full-featured reference implementation (this kit distilled from it) |
| [oracle-status-tray](https://github.com/laris-co/oracle-status-tray) | Desktop tray app showing Oracle status (Tauri) |

### The Oracle Family (Issue #6 in oracle-v2)

Multiple Oracles can coexist:
- Each person forks this kit, customizes identity
- Each Oracle maintains its own `ψ/memory/resonance/` (soul)
- oracle-v2 MCP server provides cross-Oracle search
- The 5 Principles remain constant across all Oracles

---

## Quality Assurance & Safety

### Pre-Execution Hooks

**Safety-check.sh** blocks dangerous patterns:
```bash
# Blocked commands
- rm -rf
- git --force (push, reset, clean)
- git commit --amend
- sudo
```

**Token-check.sh** warns at context thresholds:
```
Normal: 📊 < 70%
Finish: ⚡ 70-80%
Wrap up: ⚠️ 80-90%
HANDOFF NOW: 🚨 > 90%
```

### Subagent Delegation Rules

From CLAUDE.md:
- **Single file** → Do it yourself (main agent)
- **5+ files** → Use subagent (Haiku cheaper)
- **Bulk search** → Use context-finder (Haiku)
- **Code quality** → Use coder (Opus)

### Retrospective Ownership

- Main agent writes retrospectives (needs full context + vulnerability)
- Subagents gather data only (summary + verify command)
- Main agent reviews + credits subagents
- Anti-pattern: Subagent writes draft → Main just commits

---

## Extension Points for Customization

### 1. Add Custom Skills

```bash
mkdir -p .claude/skills/my-skill/
cat > .claude/skills/my-skill/CLAUDE.md << 'EOF'
---
name: my-skill
description: What this skill does
---

# My Skill

[Implementation details]
EOF
```

### 2. Add Custom Agents

```bash
cat > .claude/agents/my-agent.md << 'EOF'
---
name: my-agent
description: What this agent does
tools: Bash, Read, Write
model: haiku
---

# My Agent

[Step-by-step instructions]
EOF
```

### 3. Add Hooks

Edit `.claude/settings.json` to add hooks:
```json
{
  "hooks": {
    "SessionStart": [
      {"type": "command", "command": "your-script.sh"}
    ]
  }
}
```

### 4. Customize Identity (CLAUDE.md)

Define:
- Oracle name + human name
- Core values
- Communication style
- Tool preferences
- Subagent delegation rules
- File access policies

---

## Appendix: Technology Stack Summary

### Core (Required)
- Claude Code CLI
- bash shell
- git
- gh (GitHub CLI)

### Optional (for experiments)
- Bun runtime (for CLI utilities)
- TypeScript 5.x (for agent SDK)
- SQLite (for knowledge storage)
- ChromaDB (for vector search)
- React + Vite (for frontend)
- Tauri 2.0 (for native apps)
- MQTT (for agent pub/sub)

### Installed via oracle-skills-cli
- Core skills (15+): rrr, recap, trace, fyi, forward, feel, standup, learn, project, etc.
- Custom skills: Can be added to `.claude/skills/`

### Data Formats
- **Markdown** — All knowledge (CLAUDE.md, retrospectives, learnings, etc.)
- **YAML** — Agent/skill frontmatter (metadata)
- **JSON** — Configuration (settings.json, statusline.json)
- **Bash** — Hooks and automation scripts
- **Git** — Version control + audit trail

---

## Summary: The Five Pillars

1. **ψ/ Brain** — Organized consciousness (memory, writing, learning, experimentation)
2. **Skills** — Reusable behaviors (15+ slash commands)
3. **Agents** — Specialized personas (15+ agent types)
4. **CLAUDE.md** — Identity & rules (modular, updatable)
5. **Hooks** — Automation & safety (lifecycle management, token budgeting)

Together, these enable **a reproducible, customizable AI memory system** that:
- Keeps humans human (removes obstacles, provides external brain)
- Preserves everything (append-only, git history = truth)
- Learns continuously (retrospectives → learnings → resonance)
- Operates safely (hooks, permission checks, token budgeting)
- Scales gracefully (distillation philosophy, multi-agent support)

---

**Document Author**: Claude (Exploration Agent)  
**Repository**: https://github.com/Soul-Brews-Studio/opensource-nat-brain-oracle  
**License**: MIT  
