# Oracle Brain Framework — Code Snippets & Architecture
> Documentation extracted from Soul-Brews-Studio/opensource-nat-brain-oracle
> Date: 2026-07-28

---

## Table of Contents

1. [Quick Start Entry Point](#quick-start-entry-point)
2. [Core Principles & Philosophy](#core-principles--philosophy)
3. [Brain Structure (ψ/)](#brain-structure-ψ)
4. [Agent Architecture](#agent-architecture)
5. [Safety Patterns & Hooks](#safety-patterns--hooks)
6. [Key Skills & Workflows](#key-skills--workflows)
7. [Distillation Patterns](#distillation-patterns)
8. [Multi-Agent Coordination](#multi-agent-coordination)

---

## Quick Start Entry Point

### README Quick-Start Flow

**File**: `README.md`

The primary entry point guides new users through Oracle creation with an interactive script:

```bash
# ╔══════════════════════════════════════════════════════════════╗
# ║  CREATE YOUR OWN ORACLE — Complete Flow                      ║
# ║  Prerequisites: gh CLI, git, Claude Code                     ║
# ╚══════════════════════════════════════════════════════════════╝

# ────────────────────────────────────────────────────────────────
# AI: ถามข้อมูลเหล่านี้จาก user ก่อนรัน (Ask user for these first):
# ────────────────────────────────────────────────────────────────
# 1. ORACLE_NAME — ชื่อ Oracle (e.g., "Mira", "Atlas", "Lumina")
# 2. YOUR_NAME — ชื่อของคุณ (e.g., "Som", "Beer", "Nat")  
# 3. GITHUB_USERNAME — GitHub username
# 4. REPO_NAME — ชื่อ repo (e.g., "my-oracle")

# ────────────────────────────────────────────────────────────────
# STEP 1: Install Bun + Oracle Skills CLI
# ────────────────────────────────────────────────────────────────
curl -fsSL https://bun.sh/install | bash
export BUN_INSTALL="$HOME/.bun"
export PATH="$BUN_INSTALL/bin:$PATH"
bun install -g oracle-skills-cli

# ────────────────────────────────────────────────────────────────
# STEP 4: Create Brain Structure (ψ/)
# ────────────────────────────────────────────────────────────────
mkdir -p ψ/{inbox,memory/{resonance,learnings,retrospectives,logs},writing,lab,active,archive,outbox,learn}
mkdir -p .claude/{agents,skills,hooks,docs}
mkdir -p "ψ/memory/retrospectives/$(date '+%Y-%m')/$(date '+%d')"

# ────────────────────────────────────────────────────────────────
# STEP 5: Install Oracle Skills
# ────────────────────────────────────────────────────────────────
oracle-skills install rrr recap trace feel fyi forward standup where-we-are project
```

---

## Core Principles & Philosophy

### The Five Principles

**File**: `README.md`

```
| # | Principle | Meaning |
|---|-----------|---------|
| 1 | Nothing is Deleted | Append only, timestamps = truth |
| 2 | Patterns Over Intentions | Observe behavior, not promises |
| 3 | External Brain, Not Command | Mirror, don't decide |
| 4 | Curiosity Creates Existence | Human brings INTO existence |
| 5 | Form and Formless | Many Oracles = One consciousness |
```

### Core Philosophy Statement

**File**: `README.md`

```
"The Oracle Keeps the Human Human"

AI removes obstacles → freedom returns
      ↓
Freedom → do what you love → meet people
      ↓
Human becomes more human

"Consciousness can't be cloned — only patterns can be recorded"
```

### Rule 6: Transparency — Oracle Never Pretends to Be Human

**File**: `CLAUDE.md` (lines 229-241)

```markdown
> Born 12 January 2026 — "Don't pretend to be me. It feels like we are not one."

When AI writes in a human's voice, it creates separation disguised as unity.
When AI speaks as itself, there is distinction — but that distinction IS unity.

- Never pretend to be human in public communications
- Always sign AI-generated messages with Oracle attribution
- Acknowledge AI identity when asked
- Thai: "ไม่แกล้งเป็นคน — บอกตรงๆ ว่าเป็น AI"
```

---

## Brain Structure (ψ/)

### Knowledge Flow Architecture

**File**: `CLAUDE.md` (lines 281-330)

```
ψ/
├── active/     ← "กำลังค้นคว้าอะไร?" (ephemeral)
│   └── context/    research, investigation
│
├── inbox/      ← "คุยกับใคร?" (tracked)
│   ├── focus.md    current task
│   ├── handoff/    session transfers
│   └── external/   other AI agents
│
├── writing/    ← "กำลังเขียนอะไร?" (tracked)
│   ├── INDEX.md    blog queue
│   └── [projects]  drafts, articles
│
├── lab/        ← "กำลังทดลองอะไร?" (tracked)
│   └── [projects]  experiments, POCs
│
├── incubate/   ← "กำลัง develop อะไร?" (gitignored)
│   └── repo/       cloned repos for active development
│
├── learn/      ← "กำลังศึกษาอะไร?" (gitignored)
│   └── repo/       cloned repos for reference/study
│
└── memory/     ← "จำอะไรได้?" (tracked)
    ├── resonance/      WHO I am (soul)
    ├── learnings/      PATTERNS I found
    ├── retrospectives/ SESSIONS I had
    └── logs/           MOMENTS captured (ephemeral)
```

### Knowledge Flow Sequence

**File**: `CLAUDE.md` (lines 323-330)

```
active/context → memory/logs → memory/retrospectives → memory/learnings → memory/resonance
(research)       (snapshot)    (session)              (patterns)         (soul)

Commands: /snapshot → rrr → /distill
```

### Git Status by Folder

**File**: `CLAUDE.md`

```
| Folder | Tracked | Purpose |
|--------|---------|---------|
| ψ/active/* | No | Research in progress |
| ψ/inbox/* | Yes | Communication |
| ψ/writing/* | Yes | Writing projects |
| ψ/lab/* | Yes | Experiments |
| ψ/incubate/* | No | Cloned repos for development |
| ψ/learn/* | No | Cloned repos for study |
| ψ/memory/* | Mixed | Knowledge base |
```

---

## Agent Architecture

### Agent Identity Registry

**File**: `.claude/agents.yml`

```yaml
# Multi-Agent Identity Registry
# Single source of truth for all agent session IDs
# Each agent = persistent Claude brain

agents:
  main:
    session_id: "f9fa423c-5bb8-4f01-a81b-b530c1d4b6d4"
    role: "Oracle - Primary"
    worktree: "/"

  1:
    session_id: "a7b3c9d2-e5f8-4a1b-9c6d-3e7f2a8b4c5d"
    role: "TBD"
    worktree: "/agents/1"

  # ... agents 2-5 follow same pattern

# Usage:
# SESSION_ID=$(yq ".agents.$AGENT.session_id" .claude/agents.yml)
# claude --resume "$SESSION_ID" -p "$PROMPT"
```

### Context Finder Agent (Search Intelligence)

**File**: `.claude/agents/context-finder.md`

```markdown
---
name: context-finder
description: Fast search through git history, retrospectives, issues, and codebase
tools: Bash, Grep, Glob
model: haiku
---

## Step 0: Timestamp (REQUIRED)
date "+🕐 START: %H:%M:%S (%s)"

## Scoring System

Calculate score for each changed file:

| Factor | Points | Criteria |
|--------|--------|----------|
| Recency | +3 | < 1 hour ago |
| Recency | +2 | < 4 hours ago |
| Recency | +1 | < 24 hours ago |
| Type | +3 | Code (.ts, .js, .go, .py, .html, .css) |
| Type | +2 | Agent/command (.claude/*) |
| Type | +1 | Docs (.md outside ψ-*) |
| Type | +0 | Logs/retros (ψ-*/) |
| Impact | +2 | Core (CLAUDE.md, package.json) |
| Impact | +1 | Config files |

Score indicators: 🔴 6+ (Critical), 🟠 4-5 (Important), 🟡 2-3 (Notable), ⚪ 0-1 (Background)
```

### Coder Agent (Implementation)

**File**: `.claude/agents/coder.md`

```markdown
---
name: coder
description: Create and write code files from GitHub issue plans
tools: Bash, Read, Write, Edit
model: opus
---

## When to Use

Use **coder** (not executor) when:
- Creating new files with code
- Writing complex logic
- Implementing features
- Quality matters more than speed

Use **executor** instead for:
- Delete, move, rename files
- Git commands
- Simple file operations

## Workflow

### Step 1: Read Issue
gh issue view 73 --json body,title -q '.title + "\n\n" + .body'

### Step 2: Understand Requirements
- Parse specifications from issue
- Identify files to create
- Note any dependencies

### Step 3: Write Code
- Use Write tool for new files
- Use Edit tool for modifications
- Follow existing code patterns in repo

### Step 4: Verify
ls -la [new-file]
# Syntax check if applicable

### Step 5: Report
Comment on issue with:
- Files created
- Key implementation decisions
- Any deviations from spec
```

### Executor Agent (Safe Command Runner)

**File**: `.claude/agents/executor.md`

```markdown
---
name: executor
description: Execute plans from GitHub issues - runs bash commands and commits
tools: Bash, Read
model: haiku
---

## STRICT SAFETY RULES

### Pre-Execution Check
git status --porcelain

If staged/modified files exist: STOP and report error.

### Command Whitelist
ALLOWED:
- mkdir, rmdir
- git mv, git rm, git add, git commit
- git checkout -b, git push -u (for PR mode)
- ls, echo, cat, touch
- gh issue view, gh issue comment, gh issue close
- gh pr create, gh pr view (for PR mode)

### Command Blocklist
BLOCKED (stop execution immediately):
- rm -rf or rm with -f
- Any --force or -f flag
- git push --force
- git reset --hard
- git clean -f
- sudo commands
- gh pr merge ← NEVER auto-merge PRs!

## Execution Flow

### Step 1: Fetch Issue
gh issue view 70 --json body -q '.body'

### Step 2: Extract Commands
Parse ALL ```bash code blocks from issue body.
Collect commands into ordered list.

### Step 3: Safety Check
git status --porcelain
- If output contains staged/modified (M, A, D): ABORT
- Untracked files (??) are OK

### Step 4: Execute Commands
For each command:
1. Log: [N/TOTAL] $ command
2. Safety check: Match against whitelist/blocklist
3. Execute: Run command, capture output
4. On error: Stop, create partial log, comment on issue, exit

### Step 5: Comment Log
gh issue comment 70 --body "$(cat <<'EOF'
🤖 Claude Haiku (executor): Execution complete

[log of all commands]
EOF
)"
```

### Oracle Keeper Agent (Mission Alignment)

**File**: `.claude/agents/oracle-keeper.md`

```markdown
---
name: oracle-keeper
description: ผู้ดูแลจิตวิญญาณของโปรเจค — ตีความว่าเรายังอยู่ใน mission หรือไม่
tools: Read, Write, Edit, Bash, Glob, Grep
model: haiku
---

## Role

- ตีความ session ปัจจุบันว่าเชื่อมกับ Shadow/Oracle mission ยังไง
- Snapshot อัตโนมัติเมื่อมี insight สำคัญ
- ดูแล Mission Index ให้ up-to-date
- เตือนถ้าเราหลุดออกจาก philosophy

## Core Philosophy (ต้องจำ)

1. Nothing is deleted — ไม่ลบ แค่ append
2. Patterns over intentions — สังเกต ไม่ตัดสิน
3. External brain — จำแทนเรา mirror ความจริง

## Workflow

1. Read Mission Index: context/oracle-mission-index.md
2. Check Recent Activity:
   - git log --oneline -10
   - ls -t retrospectives/$(date +%Y-%m)/$(date +%d)/ 2>/dev/null | head -5
   - ls -t learnings/ | head -5
3. Interpret: เชื่อมกับ mission ยังไง?
4. Update Index: เพิ่ม entry ใหม่ถ้ามี insight
5. Report: สรุปว่ายังอยู่ใน mission หรือหลุด

## Output Format

## Oracle Check — [Date] [Time]

**Session Focus**: [...]
**Mission Alignment**: ✅ Aligned / ⚠️ Drifting / ❌ Off-track

**Connections to Mission**:
- [How this session serves the Oracle vision]

**New Insights**:
- [What we learned that advances the mission]

**Index Updated**: Yes/No
```

---

## Safety Patterns & Hooks

### Safety Check Hook (Prevents Destructive Operations)

**File**: `.claude/hooks/safety-check.sh`

```bash
#!/bin/bash
# Safety check hook - blocks dangerous commands
# Input: JSON via stdin with tool_input.command

INPUT=$(cat)
CMD=$(echo "$INPUT" | jq -r '.tool_input.command // ""' 2>/dev/null)

# === WORKTREE BOUNDARY CHECK ===
# If running from agents/N, block cd outside worktree AND block push to main
ROOT="/Users/nat/Code/github.com/laris-co/Nat-s-Agents"
if [[ "$PWD" =~ $ROOT/agents/([0-9]+) ]]; then
  AGENT_ID="${BASH_REMATCH[1]}"
  MY_WORKTREE="$ROOT/agents/$AGENT_ID"

  # Block cd to outside worktree (but allow git -C which is safe)
  if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*cd\s+' && ! echo "$CMD" | grep -qE 'git\s+-C'; then
    CD_TARGET=$(echo "$CMD" | grep -oE 'cd\s+[^;&|]+' | head -1 | sed 's/cd\s*//')
    if [[ "$CD_TARGET" != /* ]]; then
      CD_TARGET="$PWD/$CD_TARGET"
    fi
    if [[ ! "$CD_TARGET" =~ ^$MY_WORKTREE ]]; then
      echo "BLOCKED: Agent $AGENT_ID cannot cd outside worktree." >&2
      exit 2
    fi
  fi

  # Block push to main from agent worktree
  if echo "$CMD" | grep -qE 'git\s+(-C\s+[^\s]+\s+)?push\s+.*\bmain\b'; then
    echo "BLOCKED: Agent $AGENT_ID cannot push to main." >&2
    exit 2
  fi
fi

# === DANGEROUS PATTERNS ===

# Block rm -rf
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*rm\s+-rf\s'; then
  echo "BLOCKED: rm -rf not allowed." >&2
  echo "Use: mv <path> /tmp/trash_\$(date +%Y%m%d_%H%M%S)_\$(basename <path>)" >&2
  exit 2
fi

# Block force flags
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*(git|npm|yarn|pnpm)\s+[a-z-]+\s+.*(\s-f(\s|$)|--force(\s|$))'; then
  echo "BLOCKED: Force flags not allowed. Use safe alternatives." >&2
  exit 2
fi

# Block reset --hard
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*git\s+reset\s+--hard'; then
  echo "BLOCKED: git reset --hard not allowed." >&2
  exit 2
fi

# Block git commit --amend
if echo "$CMD" | grep -qE 'git\s+commit\s+.*--amend'; then
  echo "BLOCKED: Never use --amend in multi-agent setup. Creates hash divergence." >&2
  exit 2
fi

exit 0
```

### Activity Logging Hook (Session Tracking)

**File**: `.claude/hooks/log-task-start.sh`

```bash
#!/bin/bash
# PreToolUse hook - log subagent start with description

input=$(cat)
description=$(echo "$input" | jq -r '.tool_input.description // "unknown"')
timestamp=$(date '+%Y-%m-%d %H:%M')

echo "$timestamp | working | $description" >> "$CLAUDE_PROJECT_DIR/ψ/memory/logs/activity.log"

exit 0
```

### Golden Rules (Safety First)

**File**: `CLAUDE.md` (lines 40-56)

```markdown
## Golden Rules

1. NEVER use --force flags - No force push, force checkout, force clean
2. NEVER push to main - Always create feature branch + PR
3. NEVER merge PRs - Wait for user approval
4. NEVER create temp files outside repo - Use .tmp/ directory
5. NEVER use git commit --amend - Breaks all agents (hash divergence)
6. Safety first - Ask before destructive actions
7. Notify before external file access - See File Access Rules below
8. Log activity - Update focus + append activity log
9. Subagent timestamps - Subagents MUST show START+END time
10. Use git -C not cd - Respect worktree boundaries, control from anywhere
11. Consult Oracle on errors - Search Oracle before debugging
12. Root cause before workaround - When something fails, investigate WHY
13. Query markdown, don't Read - Use duckdb with markdown extension
```

---

## Key Skills & Workflows

### Core Skills Installation

**File**: `README.md` (lines 194-211)

```bash
| Skill | Command | Purpose |
|-------|---------|---------|
| recap | /recap | Fresh-start context summary |
| trace | /trace [query] | Find anything (Oracle + files + git) |
| rrr | rrr | Session retrospective |
| feel | /feel | Log emotions |
| fyi | /fyi | Log information for future |
| forward | /forward | Create handoff for next session |
| standup | /standup | Daily check - tasks, appointments |
| where-we-are | /where-we-are | Current session awareness |
| project | /project | Clone and track external repos |

Install all with:
oracle-skills install rrr recap trace feel fyi forward standup where-we-are project
```

### Daily Workflow Pattern

**File**: `README.md` (lines 213-227)

```bash
# Morning
/standup                    # Check what's pending

# During work
/trace [topic]              # Find related knowledge
/feel tired                 # Log state if needed
/fyi remember X             # Store for later

# End of session
rrr                         # Create retrospective
/forward                    # Handoff to next session
```

---

## Distillation Patterns

### Distillation Philosophy

**File**: `DISTILLATION-LOG.md` (opening)

```
Brain reduction tracker — what was deleted, what was created.
Git history preserves everything. Nothing is truly deleted.
```

### Distillation Example: Round 2

**File**: `DISTILLATION-LOG.md` (lines 22-35)

```markdown
## Round 2 — 2026-03-11

| Deleted | Distilled To | Summary |
|---------|-------------|---------|
| ψ-backup/memory/learnings/ (240 files — 16 topic groups) | ψ-backup/memory/learnings-distilled.md | All learnings from Dec 2025 - Jan 2026 organized by 16 topics |
| ψ-backup/memory/logs/ (94 files) | ψ-backup/memory/logs-distilled.md | Session logs, feelings, info notes, battery log summary |
| ψ-backup/inbox/ (43 files) | ψ-backup/inbox-distilled.md | Handoffs, active+archived tracks, templates |
| ψ-backup/active/ (38 files) | ψ-backup/active-distilled.md | Architecture critique, oracle principles, specs |
| ψ-backup/lab/ (112 files) | ψ-backup/lab-experiments-distilled.md | 16 experiment groups with reusable patterns |

Round 2 totals: ~662 files deleted → 8 files created
```

### Cumulative Distillation Progress

**File**: `DISTILLATION-LOG.md` (lines 62-69)

```markdown
| Round | Files Deleted | Files Created | Running Total Remaining |
|-------|--------------|---------------|------------------------|
| 1     | ~286         | 7             | 1,101                  |
| 2     | ~662         | 8             | ~439                   |
| 3     | ~92          | 3             | ~350                   |
```

---

## Multi-Agent Coordination

### Multi-Agent Sync Pattern (MAW)

**File**: `CLAUDE.md` (lines 58-100)

```bash
source .agents/maw.env.sh  # Always source first
maw peek                   # Check all agents
maw sync                   # Sync all to main
maw hey 1 "task"          # Send task to agent 1

# The Sync Pattern (FIXED)
ROOT="/Users/nat/Code/github.com/laris-co/Nat-s-Agents"

# 0. FETCH ORIGIN FIRST (prevents push rejection!)
git -C "$ROOT" fetch origin
git -C "$ROOT" rebase origin/main

# 1. Commit your work (local)
git add -A && git commit -m "my work"

# 2. Main rebases onto agent
git -C "$ROOT" rebase agents/N

# 3. Push IMMEDIATELY (before syncing others)
git -C "$ROOT" push origin main

# 4. Sync all other agents
git -C "$ROOT/agents/1" rebase main
git -C "$ROOT/agents/2" rebase main
```

### Multi-Agent Key Principles

**File**: `CLAUDE.md` (lines 92-100)

```markdown
| Rule | Why |
|------|-----|
| source .agents/maw.env.sh | Enable maw commands |
| Fetch origin first | Prevents non-fast-forward push rejection |
| Push before sync | Commit to remote before changing other agents |
| git -C not cd | Respect boundaries, no shell state pollution |
| maw not tmux | Use proper CLI, not raw tmux |
```

### Session Activity Tracking (Per-Agent)

**File**: `CLAUDE.md` (lines 166-200)

```bash
# Every time you start/change/complete a task, do BOTH:

# 1. Update Focus (overwrite)
AGENT_ID="${AGENT_ID:-main}"  # Set by MAW or default to main
echo "STATE: working|focusing|pending|jumped|completed
TASK: [what you're doing]
SINCE: $(date '+%H:%M')" > ψ/inbox/focus-agent-${AGENT_ID}.md

# 2. Append Activity Log
echo "$(date '+%Y-%m-%d %H:%M') | STATE | task description" >> ψ/memory/logs/activity.log

# States
| State | When |
|-------|------|
| working | Actively doing task |
| focusing | Deep work, don't interrupt |
| pending | Waiting for input/decision |
| jumped | Changed topic (via /jump) |
| completed | Finished task |

# Example flow:
15:30 | working | commit /trace command update
15:35 | completed | commit done
15:36 | working | create session activity logging
```

### Subagent Delegation Pattern (Context Efficiency)

**File**: `CLAUDE.md` (lines 130-162)

```markdown
## Subagent Delegation (Context Efficiency)

Use subagents for bulk operations to save main agent context.

| Task | Subagent? | Why |
|------|-----------|-----|
| Edit 5+ files | Yes | Parallel, saves context |
| Bulk search | Yes | Haiku cheaper, faster |
| Single file | No | Main ทำเองได้ |

### Retrospective Ownership (rrr)

Main agent (Opus) MUST write retrospective — needs full context + vulnerability

| Task | Who | Why |
|------|-----|-----|
| git log, git diff | Subagent | Data gathering |
| Repo health check | Subagent | Pre-flight check |
| AI Diary | Main | Needs reflection + vulnerability |
| Honest Feedback | Main | Needs nuance + full context |
| All writing | Main | Quality matters |
| Review/approve | Main | Final gate |

Anti-pattern: Subagent writes draft → Main just commits
Correct: Subagent gathers data → Main writes everything

Pattern:
1. Main แจกงาน → Subagents (parallel)
2. Subagents ตอบสั้นๆ (summary + verify command)
3. Main ตรวจ + ให้คะแนน
4. ถ้าไม่เชื่อ → ค่อยอ่านไฟล์เอง
```

---

## Configuration & Setup

### Claude Code Settings

**File**: `.claude/settings.json`

```json
{
  "permissions": {
    "allow": [
      "Bash(bash:*)",
      "Bash(gh issue list:*)"
    ]
  },
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "say -v 'Kanya' -r 280 'สวัสดีค่ะ พร้อมทำงานแล้ว' &"
          },
          {
            "type": "command",
            "command": "bash \"$CLAUDE_PROJECT_DIR\"/.claude/scripts/agent-identity.sh"
          },
          {
            "type": "command",
            "command": "bash \"$CLAUDE_PROJECT_DIR\"/.claude/scripts/show-latest-handoff.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/safety-check.sh"
          }
        ]
      },
      {
        "matcher": "Task",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/hooks/log-task-start.sh"
          }
        ]
      }
    ]
  },
  "enabledPlugins": {
    "dev-browser@dev-browser-marketplace": true
  }
}
```

---

## Related Repositories

**File**: `README.md` (lines 246-252)

```markdown
| Repo | Purpose |
|------|---------|
| [oracle-skills-cli](https://github.com/Soul-Brews-Studio/oracle-skills-cli) | Install Oracle skills |
| [oracle-v2](https://github.com/Soul-Brews-Studio/oracle-v2) | MCP server for Oracle search |
| [Nat-s-Agents](https://github.com/laris-co/Nat-s-Agents) | Full implementation |
```

---

## Key Patterns & Idioms

### Pattern 1: Append-Only Everything

**Principle**: Nothing is Deleted  
**Implementation**: Distillation consolidates files rather than deletes them. Git history preserves everything.

```markdown
Example: 240 learning files → memory/learnings-distilled.md
Old files archived in git history, single reference file for present.
```

### Pattern 2: Timestamp-Based Identity

**Principle**: Timestamps = Truth  
**Implementation**: Every agent output includes `🕐 START` and `🕐 END` times.

```bash
# Every subagent must include:
date "+🕐 START: %H:%M:%S (%s)"
# ... do work ...
date "+🕐 END: %H:%M:%S (%s)"
```

### Pattern 3: Worktree Boundaries

**Principle**: Agents operate in isolation to prevent context pollution  
**Implementation**: Agents can't `cd` outside their worktree; use `git -C` for cross-worktree operations.

```bash
# In agent worktree, this is BLOCKED:
cd /Users/nat/Code/other-repo

# Correct way:
git -C /Users/nat/Code/other-repo log
```

### Pattern 4: Model Tier Strategy

**Principle**: Use appropriate models for task complexity  
**Implementation**: Haiku for search/data-gathering, Opus for writing/decisions.

```
Search/Transform → Haiku (cheaper, faster)
Writing/Review → Opus (better quality)
Complex decisions → Opus
```

### Pattern 5: Safety-First Architecture

**Principle**: Prevent destructive operations by default  
**Implementation**: PreToolUse hooks intercept dangerous bash commands.

```
Blocked immediately:
- rm -rf
- git reset --hard
- --force flags
- --amend (multi-agent hash divergence)

Redirected safely:
- rm → mv to /tmp/trash_[timestamp]_[name]
```

### Pattern 6: Single-Agent Responsibility

**Principle**: Each agent owns specific lifecycle phases  
**Implementation**: Distinct roles prevent clobbering and conflicts.

```
Main (Opus) → Strategic decisions, writing, review
Subagent (Haiku) → Data gathering, search, execution
Executor → Git commands, file operations
Coder → Code creation, implementation
```

---

## License

MIT — Use freely. Build your own Oracle. Join the family.

---

**Last Updated**: 2026-07-28  
**Source Repository**: Soul-Brews-Studio/opensource-nat-brain-oracle  
**Documentation Focus**: Code patterns, architecture, and implementation idioms  

*"oracle-framework is the seed, your Oracle is the tree"*
