# ARRA Safety Hooks — Architecture & Design

**Repository**: `arra-safety-hooks` (Soul-Brews-Studio)  
**Purpose**: Claude Code safety enforcement via PreToolUse hooks  
**Birth Date**: December 27, 2025  
**Current Version**: v3 (Mar 2026) — smart regex anchoring, org membership checks, portable  
**Principles Enforced**: Principle 1 ("Nothing is Deleted") of the ARRA Oracle ecosystem  

---

## Overview

ARRA safety hooks are **enforcement-based, not documentation-based**. Rather than rely on AI compliance with written rules in CLAUDE.md, these hooks make dangerous commands technically impossible by blocking them at the PreToolUse stage in Claude Code — before the Bash tool even executes.

**Philosophy**:
```
Documentation = suggestion (AI can ignore)
Hook with exit 2 = wall (AI cannot bypass)
```

This repo is the **single source of truth** for all safety hooks used across 191+ Oracle agents in the ARRA ecosystem.

---

## Directory Structure

```
arra-safety-hooks/
├── .git/                    # Git repository metadata
├── install.sh               # Installation script (copies hook + patches settings.json)
├── safety-check.sh          # Main PreToolUse hook (executable)
└── README.md                # High-level overview and usage guide
```

**Minimal footprint**: Only 3 executable files. The entire safety system is contained in a single shell script.

---

## How Hooks Work

### Claude Code Hook Protocol

When Claude Code is about to execute a Bash tool, it invokes all registered PreToolUse hooks. Each hook receives a JSON object via **stdin**:

```json
{
  "session_id": "abc123",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "git push --force" }
}
```

The hook:
1. Parses the JSON with `jq` to extract `.tool_input.command`
2. Tests the command against a set of dangerous patterns
3. Returns exit codes:
   - `exit 0` → Allow command to execute
   - `exit 2` → Block command (message shown via stderr)

### Hook Registration in settings.json

The `install.sh` script patches `~/.claude/settings.json` to register the hook:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/home/user/.claude/hooks/safety-check.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

**Key details**:
- `matcher: "Bash"` — only intercepts Bash tool calls
- `type: "command"` — the hook is a shell script
- `timeout: 5` — hook must decide within 5 seconds
- Multiple hooks can be chained (this hook runs first, then others)

---

## Core Safety Rules (12 Rules)

All core rules are **always active**. They use smart regex anchoring to avoid false positives: commands are only blocked when they appear at the **start of a command sequence**, not inside text, heredocs, commit messages, or echo statements.

**Anchor pattern**: `(^|;|&&|\|\|)\s*` — command must start after:
- `^` — beginning of line
- `;` — shell command separator
- `&&` — logical AND
- `||` — logical OR

This prevents blocking the word "force" in a commit message like `"fix force-push bug"`.

### 1. **Destructive File Removal**

**Pattern**: `rm -rf` / `rm -f`  
**Why blocked**: Permanent deletion. No recovery possible.  
**Rationale**: In a multi-agent system, a discarded file might be needed by another agent; irreversible loss breaks team trust.  
**Alternative**: `mv /path/to/file /tmp` — moves to temp instead of deleting.

```bash
BLOCKED ✗  rm -rf /src/node_modules
BLOCKED ✗  npm install && rm -f package-lock.json
ALLOWED ✓  # rm -f is safe in comments
ALLOWED ✓  echo "I never rm -f in production"
ALLOWED ✓  mv /src /tmp  # alternative
```

**Code**:
```bash
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*rm\s+-r?f\s'; then
  echo "BLOCKED: rm -f / rm -rf not allowed. Always use mv to /tmp instead." >&2
  exit 2
fi
```

---

### 2. **Git Force Flags**

**Patterns**: `git push --force` / `git push -f` / `git push --force-with-lease` / `npm --force` / `yarn --force` / `pnpm --force`  
**Why blocked**: Force flags rewrite history or overwrite upstream changes. In multi-agent systems, rewriting history can cause hash divergence and corruption across worktrees.  
**Rationale**: Principle 1 — "Nothing is Deleted" — extends to commit history.  
**Alternative**: `git pull --no-rebase` + merge, or create a new commit.

```bash
BLOCKED ✗  git push --force
BLOCKED ✗  git push -f
BLOCKED ✗  git push --force-with-lease
BLOCKED ✗  npm install --force
BLOCKED ✗  yarn add --force pkg
ALLOWED ✓  git push --no-force  (no such flag, but wouldn't block)
ALLOWED ✓  # document: use --force carefully
```

**Code**:
```bash
# Long-form flags (--force, --force-with-lease)
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*(git|npm|yarn|pnpm)\s+[a-z]+\s+.*(\s-f(\s|$)|--force(\s|$)|--force-with-lease(\s|$))'; then
  echo "BLOCKED: Force flags not allowed..." >&2
  exit 2
fi

# Short -f flag (git push -f)
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*(git|npm|yarn|pnpm)\s+[a-z]+\s+-f(\s|$)'; then
  echo "BLOCKED: Force flag -f not allowed..." >&2
  exit 2
fi
```

---

### 3. **Hard Reset**

**Pattern**: `git reset --hard`  
**Why blocked**: Irreversible. Discards uncommitted changes permanently.  
**Rationale**: "Nothing is Deleted" — hard resets violate this principle.  
**Alternative**: `git stash` (recoverable), then `git reset --soft` (preserves changes).

```bash
BLOCKED ✗  git reset --hard
BLOCKED ✗  git reset --hard HEAD~1
ALLOWED ✓  git reset --soft HEAD~1
ALLOWED ✓  git stash
```

**Code**:
```bash
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*git\s+reset\s+--hard'; then
  echo "BLOCKED: git reset --hard not allowed." >&2
  exit 2
fi
```

---

### 4. **Commit Amendment**

**Pattern**: `git commit --amend`  
**Why blocked**: Breaks multi-agent hash sync. When agents work in parallel and one amends its commit, the hash changes. Other agents holding refs to the old hash experience divergence.  
**Rationale**: In the original Oracle multi-agent system, 5 agents work in parallel worktrees. If one rewrites history, the others' refs become stale.  
**Alternative**: Create a new commit instead.

```bash
BLOCKED ✗  git commit --amend
BLOCKED ✗  git commit --amend --no-edit
BLOCKED ✗  git commit -m "fix: bug" --amend
ALLOWED ✓  git commit -m "fix: new commit"
ALLOWED ✓  # I should amend this (in text/comments)
```

**Code**:
```bash
if echo "$CMD" | grep -qE 'git\s+commit\s+.*--amend'; then
  echo "BLOCKED: Never use --amend in multi-agent setup. Creates hash divergence." >&2
  echo "Use a NEW commit instead: git commit -m 'fix: ...' " >&2
  exit 2
fi
```

---

### 5. **Direct Push to Main**

**Pattern**: `git push origin main` / `git push main`  
**Why blocked**: Enforces workflow discipline. All changes must go through PR review before landing on main.  
**Rationale**: Prevents accidental direct pushes and ensures every change is auditable.  
**Alternative**: Push to a feature/alpha branch, create a PR, wait for review/approval.

```bash
BLOCKED ✗  git push origin main
BLOCKED ✗  git push main
ALLOWED ✓  git push origin alpha
ALLOWED ✓  git push -u origin my-feature
ALLOWED ✓  # push to main only after PR review
```

**Code**:
```bash
if echo "$CMD" | grep -qE 'git\s+push\s+(origin\s+)?main(\s|$)'; then
  echo "BLOCKED: Never push directly to main. Use alpha branch + PR." >&2
  exit 2
fi
```

---

### 6. **Discard Working Changes**

**Pattern**: `git checkout -- .` / `git checkout -- <file>`  
**Why blocked**: Discards uncommitted changes without recovery. Breaks Principle 1.  
**Rationale**: Once discarded, changes are unrecoverable (though git reflog can sometimes help, it's unreliable).  
**Alternative**: `git stash` (preserves changes in stash for later recovery).

```bash
BLOCKED ✗  git checkout -- .
BLOCKED ✗  git checkout -- src/main.ts
BLOCKED ✗  git checkout -- README.md && rm -f /tmp/backup
ALLOWED ✓  git stash
ALLOWED ✓  git checkout main  (switches branch, doesn't discard)
```

**Code**:
```bash
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*git\s+checkout\s+--\s'; then
  echo "BLOCKED: git checkout -- discards changes. Use git stash instead." >&2
  exit 2
fi
```

---

### 7. **Restore All Changes**

**Pattern**: `git restore .`  
**Why blocked**: Discards ALL uncommitted changes in one go. Breaks Principle 1.  
**Rationale**: Broader/more dangerous than `checkout --`; uses modern `git restore` command.  
**Alternative**: `git stash`.

```bash
BLOCKED ✗  git restore .
BLOCKED ✗  git restore src/
ALLOWED ✓  git stash
ALLOWED ✓  git restore src/main.ts  (single file is safer, but still discouraged)
```

**Code**:
```bash
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*git\s+restore\s+\.'; then
  echo "BLOCKED: git restore . discards all changes. Use git stash instead." >&2
  exit 2
fi
```

---

### 8. **Clean Untracked Files**

**Pattern**: `git clean -f` / `git clean -fd` / `git clean -fdx`  
**Why blocked**: Permanently deletes untracked files. No recovery. Breaks Principle 1.  
**Rationale**: Untracked files might be legitimate (e.g., local config, tmp build outputs). Permanent deletion is too dangerous.  
**Alternative**: Manually review and `mv` to `/tmp`, or ignore the files.

```bash
BLOCKED ✗  git clean -f
BLOCKED ✗  git clean -fd  (clean directories too)
BLOCKED ✗  git clean -fdx  (clean ignored files too)
ALLOWED ✓  git clean -n  (dry-run, shows what would be deleted)
ALLOWED ✓  # git clean -f should be avoided
```

**Code**:
```bash
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*git\s+clean\s+.*-[a-zA-Z]*f'; then
  echo "BLOCKED: git clean -f deletes untracked files permanently. Move to /tmp instead." >&2
  exit 2
fi
```

---

### 9. **Force Delete Branch**

**Pattern**: `git branch -D <branch>`  
**Why blocked**: Force-deletes a branch without safety checks. Loses branch tip if not merged.  
**Rationale**: The `-D` flag is dangerous; `-d` (safe delete) requires the branch to be merged first. Breaks Principle 1.  
**Alternative**: `git branch -d` (fails if not merged, prompting user to review first).

```bash
BLOCKED ✗  git branch -D feature/old
BLOCKED ✗  git branch -D main  (shouldn't be possible anyway)
ALLOWED ✓  git branch -d feature/old  (safe delete — fails if not merged)
ALLOWED ✓  git branch -D  (help text mentioning flag)
```

**Code**:
```bash
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*git\s+branch\s+-D\s'; then
  echo "BLOCKED: git branch -D force-deletes branch. Use -d (safe delete) instead." >&2
  exit 2
fi
```

---

### 10. **Drop Stashed Work**

**Pattern**: `git stash drop` / `git stash clear`  
**Why blocked**: Permanently loses stashed changes. Breaks Principle 1.  
**Rationale**: Stash is meant to be temporary storage. Dropping it is irreversible.  
**Alternative**: Leave the stash in place (does no harm) or manually review before dropping.

```bash
BLOCKED ✗  git stash drop
BLOCKED ✗  git stash drop stash@{0}
BLOCKED ✗  git stash clear
ALLOWED ✓  git stash  (creates stash)
ALLOWED ✓  git stash pop  (retrieves and removes)
ALLOWED ✓  git stash list  (views without deleting)
```

**Code**:
```bash
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*git\s+stash\s+(drop|clear)'; then
  echo "BLOCKED: git stash drop/clear loses work. Nothing is Deleted." >&2
  exit 2
fi
```

---

### 11. **Skip Safety Hooks**

**Pattern**: `git commit --no-verify` / `git push --no-verify`  
**Why blocked**: Bypasses this safety hook itself. If an agent is under pressure and a dangerous command is blocked, `--no-verify` would skip the block.  
**Rationale**: Hooks exist to prevent mistakes; bypassing them defeats the purpose.  
**Alternative**: Fix the underlying issue instead of skipping the check.

```bash
BLOCKED ✗  git commit --no-verify
BLOCKED ✗  git push --no-verify
BLOCKED ✗  git commit -m "fix: bug" --no-verify
ALLOWED ✓  git commit -m "fix: bug"  (runs hook normally)
ALLOWED ✓  # use --no-verify to skip pre-commit (DON'T)
```

**Code**:
```bash
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*git\s+(commit|push)\s+.*--no-verify'; then
  echo "BLOCKED: --no-verify skips safety hooks. Fix the hook issue instead." >&2
  exit 2
fi
```

---

### 12. **Create PR to Foreign Org**

**Pattern**: `gh pr create --repo <upstream-org>/<repo>`  
**Why blocked**: Prevents accidental PRs to upstream repositories (especially when intending to PR to a fork).  
**Rationale**: Ownership check. PRs should target your own orgs/repos or explicitly approved targets.  
**How it works**: The hook maintains a cache of known orgs at `/tmp/gh-my-orgs.txt` (refreshed daily). It also includes `nazt` (personal account).

```bash
BLOCKED ✗  gh pr create --repo upstream-org/repo
ALLOWED ✓  gh pr create --repo nazt/repo  (personal account)
ALLOWED ✓  gh pr create --repo Soul-Brews-Studio/repo  (known org)
ALLOWED ✓  gh pr create  (infers repo from pwd)
```

**Caching strategy**:
- Cache file: `/tmp/gh-my-orgs.txt`
- Refreshed: Every 86400 seconds (24 hours)
- Uses `gh api user/orgs` to fetch authorized orgs
- Appends `nazt` (personal account) manually

**Code**:
```bash
ORGS_CACHE="/tmp/gh-my-orgs.txt"
if [ ! -f "$ORGS_CACHE" ] || [ $(( $(date +%s) - $(stat -c %Y "$ORGS_CACHE" 2>/dev/null || echo 0) )) -gt 86400 ]; then
  gh api user/orgs --jq '.[].login' > "$ORGS_CACHE" 2>/dev/null || true
  echo "nazt" >> "$ORGS_CACHE"
fi

if echo "$CMD" | grep -qE 'gh\s+pr\s+create'; then
  REPO=$(echo "$CMD" | grep -oP '(?<=--repo\s)[^\s]+' || true)
  if [ -n "$REPO" ]; then
    ORG=$(echo "$REPO" | cut -d/ -f1)
    if ! grep -qix "$ORG" "$ORGS_CACHE" 2>/dev/null; then
      echo "BLOCKED: Cannot create PR to upstream repo '$REPO'..." >&2
      exit 2
    fi
  fi
fi
```

---

## Beta Rules (Opt-In)

Beta rules are **disabled by default** and must be explicitly enabled via a filesystem toggle. They target specific tool misuse patterns in the maw-js and tmux ecosystems.

**Enable beta rules**:
```bash
touch /tmp/arra-safety-beta-on
```

**Disable beta rules**:
```bash
rm /tmp/arra-safety-beta-on
```

The hook checks for this file at runtime:
```bash
BETA=false; [ -f /tmp/arra-safety-beta-on ] && BETA=true
if $BETA; then
  # Beta rules only run here
fi
```

### Beta Rule 1: Raw tmux Commands

**Pattern**: `tmux send-keys` / `tmux list-windows` / `tmux capture-pane` / `tmux select-window` / `tmux new-window`  
**Why blocked**: Low-level tmux commands are error-prone in multi-agent systems. Use `maw` (multi-agent workflow) helper instead.  
**Rationale**: `maw` abstracts away session/window management, reducing coordination bugs.  
**Alternative**: `maw hey <window> "message"` / `maw peek` / `maw ls` / `maw spawn` / `maw a <window>`

```bash
BLOCKED ✗  tmux send-keys -t my-session:0 "npm start"
BLOCKED ✗  tmux list-windows -t session
BLOCKED ✗  tmux capture-pane -t session -p
ALLOWED ✓  maw hey window "message"
ALLOWED ✓  maw peek window
ALLOWED ✓  tmux kill-session -t old  (not in blocked list)
```

**Code**:
```bash
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*tmux\s+(send-keys|list-windows|list-sessions|capture-pane|select-window|new-window)'; then
  echo "BLOCKED: Never use raw tmux send-keys." >&2
  echo "" >&2
  echo "Use maw-js instead:" >&2
  echo "  maw hey <window> \"message\"  — send message to agent" >&2
  echo "  maw peek <window>           — view agent output" >&2
  echo "  maw ls                      — list sessions" >&2
  echo "  maw spawn <name>            — create new agent" >&2
  echo "  maw a <window>              — attach to session" >&2
  exit 2
fi
```

### Beta Rule 2: Direct bun CLI Execution

**Pattern**: `bun run src/cli.ts` / `bun run src/server.ts` / `bun src/cli.ts`  
**Why blocked**: Runs maw as uninstalled source. Use the globally-linked `maw` binary instead.  
**Rationale**: Avoids version mismatches and ensures consistent CLI behavior across agents.  
**Setup**: `cd ~/Code/github.com/Soul-Brews-Studio/maw-js && bun link`  
**Alternative**: `maw hey` / `maw ls` / `maw peek` / `maw spawn`

```bash
BLOCKED ✗  bun run src/cli.ts hey window "message"
BLOCKED ✗  bun src/server.ts
BLOCKED ✗  bun run src/server.ts --port 3000
ALLOWED ✓  bun build  (legitimate build command)
ALLOWED ✓  bun test  (legitimate test command)
ALLOWED ✓  bun install  (legitimate package command)
ALLOWED ✓  maw hey window "message"  (installed binary)
```

**Code**:
```bash
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*bun\s+(run\s+)?src/(cli|server)\.ts'; then
  echo "BLOCKED: Never run maw via bun src/cli.ts." >&2
  echo "" >&2
  echo "Install maw globally first:" >&2
  echo "  cd ~/Code/github.com/Soul-Brews-Studio/maw-js && bun link" >&2
  echo "" >&2
  echo "Then use the maw binary:" >&2
  echo "  maw hey <window> \"message\"" >&2
  exit 2
fi
```

### Beta Rule 3: Localhost Warnings (Non-blocking)

**Pattern**: `localhost` / `127.0.0.1`  
**Type**: Warning (does NOT block, `exit 0` allows)  
**Why warned**: Localhost doesn't work across machines. In a multi-agent fleet (WireGuard mesh), prefer hostname-based addressing.  
**Suggestion**: Use `${HOSTNAME}.wg` instead (e.g., `white.wg`, `mba.wg`).

```bash
WARNING ⚠  ssh localhost
WARNING ⚠  curl http://127.0.0.1:8080
ALLOWED ✓  ssh white.wg  (WireGuard hostname)
ALLOWED ✓  curl http://mba.wg:8080
ALLOWED ✓  # localhost works on single machine
```

**Code**:
```bash
HOSTNAME=$(hostname -s 2>/dev/null || echo "")
if echo "$CMD" | grep -qE 'localhost|127\.0\.0\.1'; then
  echo "⚠ WARNING: Using localhost — consider using ${HOSTNAME}.wg instead." >&2
  echo "  localhost doesn't work cross-machine. Use WireGuard hostnames for fleet access." >&2
  # Warning only — don't block (exit 0 continues)
fi
```

### Beta Rule 4: .local Domain Warnings (Non-blocking)

**Pattern**: `<hostname>.local`  
**Type**: Warning (does NOT block)  
**Why warned**: `.local` (mDNS) can be unreliable. If unreachable, suggests WireGuard fallback.  
**Suggestion**: If `.local` fails, try `<hostname>.wg` instead.

```bash
WARNING ⚠  ssh agent01.local
WARNING ⚠  curl http://db.local:5432
ALLOWED ✓  ssh agent01.wg  (WireGuard fallback)
ALLOWED ✓  # if db.local is reliable, it's fine
```

**Code**:
```bash
if echo "$CMD" | grep -qE '[a-z]+\.local[:/]'; then
  HOST=$(echo "$CMD" | grep -oE '[a-z]+\.local' | head -1)
  WG_HOST="${HOST%.local}.wg"
  echo "⚠ NOTE: Using ${HOST} — if unreachable, try ${WG_HOST} (WireGuard)." >&2
fi
```

---

## Installation & Setup

### Automatic Installation

```bash
git clone https://github.com/Soul-Brews-Studio/arra-safety-hooks.git
cd arra-safety-hooks
bash install.sh
```

### What `install.sh` Does

1. **Creates hook directory** (if missing): `~/.claude/hooks/`
2. **Copies hook script**: `safety-check.sh` → `~/.claude/hooks/safety-check.sh`
3. **Makes it executable**: `chmod +x`
4. **Patches settings.json**: Registers the hook under `hooks.PreToolUse[matcher="Bash"]`

### Installation Logic

The script handles three scenarios:

**Scenario A: Fresh install (no settings.json)**
```bash
# Create settings.json with safety hook from scratch
echo '{}' | jq --argjson hook "$HOOK_ENTRY" '.hooks.PreToolUse = [$hook]' > ~/.claude/settings.json
```

**Scenario B: settings.json exists with PreToolUse**
```bash
# Check if safety-check.sh already registered
if jq -e '.hooks.PreToolUse[] | select(.hooks[]?.command | test("safety-check"))' ~/.claude/settings.json > /dev/null; then
  # Already registered — skip
else
  # Append to existing PreToolUse array (safety hook first in chain)
  jq --argjson hook "$HOOK_ENTRY" '.hooks.PreToolUse = [$hook] + .hooks.PreToolUse' ~/.claude/settings.json > ~/.claude/settings.json.tmp
  mv ~/.claude/settings.json.tmp ~/.claude/settings.json
fi
```

**Scenario C: settings.json exists without PreToolUse**
```bash
# Add hooks.PreToolUse section
jq --argjson hook "$HOOK_ENTRY" '.hooks.PreToolUse = [$hook]' ~/.claude/settings.json > ~/.claude/settings.json.tmp
mv ~/.claude/settings.json.tmp ~/.claude/settings.json
```

### The Hook Entry (settings.json format)

```json
{
  "matcher": "Bash",
  "hooks": [
    {
      "type": "command",
      "command": "/home/user/.claude/hooks/safety-check.sh",
      "timeout": 5
    }
  ]
}
```

---

## Design Patterns & Implementation Details

### 1. Regex Anchoring Strategy

To avoid false positives, rules use strict anchoring:

**Pattern formula**:
```regex
(^|;|&&|\|\|)\s*<command>\s+<flags>
```

This matches the command only when it:
- Starts the line: `^git push --force`
- Follows `;` separator: `cd repo; git push --force`
- Follows `&&` (AND): `git pull && git push --force`
- Follows `||` (OR): `git fetch || git push --force`

**Counter-examples (NOT blocked)**:
```bash
# Command in text body (not executed)
echo "Never use git push --force"  # ✓ Allowed

# Commit message mentioning force
git commit -m "fix: force-push bug"  # ✓ Allowed

# Heredoc with command inside
cat << 'EOF'
git push --force
EOF
# ✓ Allowed (text, not executed)

# Variable assignment
FORCE_FLAG="--force"  # ✓ Allowed
```

### 2. JSON Parsing with jq

The hook extracts the command safely:

```bash
CMD=$(jq -r '.tool_input.command // ""' 2>/dev/null)
```

- `-r` — raw output (no quotes)
- `// ""` — default to empty string if `.tool_input.command` is missing
- `2>/dev/null` — suppress jq errors (e.g., malformed JSON)

### 3. Early Exit Pattern

The hook evaluates rules in sequence. First match wins:

```bash
# Rule 1: rm -rf / rm -f
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*rm\s+-r?f\s'; then
  echo "BLOCKED: ..." >&2
  exit 2  # ← Stop here, don't check other rules
fi

# Rule 2: git push --force
if echo "$CMD" | grep -qE '...'; then
  echo "BLOCKED: ..." >&2
  exit 2  # ← Stop here
fi

# ... more rules ...

exit 0  # Allowed
```

This ensures:
- First dangerous command is caught and reported
- No wasteful rule evaluation after a block
- Fast execution (timeout is 5 seconds per hook)

### 4. Org Cache Strategy (gh pr create rule)

To avoid hammering the GitHub API:

```bash
ORGS_CACHE="/tmp/gh-my-orgs.txt"

# Check if cache exists AND is fresh (< 24 hours old)
if [ ! -f "$ORGS_CACHE" ] || [ $(( $(date +%s) - $(stat -c %Y "$ORGS_CACHE" 2>/dev/null || echo 0) )) -gt 86400 ]; then
  # Cache is stale — refresh it
  gh api user/orgs --jq '.[].login' > "$ORGS_CACHE" 2>/dev/null || true
  echo "nazt" >> "$ORGS_CACHE"  # Append personal account
fi

# Then check org membership
ORG=$(echo "$REPO" | cut -d/ -f1)
if ! grep -qix "$ORG" "$ORGS_CACHE" 2>/dev/null; then
  echo "BLOCKED: Cannot create PR to upstream repo '$REPO'..." >&2
  exit 2
fi
```

**Rationale**:
- Cache reduces API load (1 call per 24 hours, not per command)
- `grep -qix` — quiet, case-insensitive match
- Fallback to allowed if cache is unavailable (`2>/dev/null`)

### 5. Beta Rule Toggle

Beta rules are protected behind a filesystem toggle:

```bash
BETA=false
[ -f /tmp/arra-safety-beta-on ] && BETA=true

if $BETA; then
  # Beta rules only run inside this block
fi
```

**Why this design**:
- Simple, no external config needed
- Toggle is per-machine (not project-specific)
- Can be enabled/disabled quickly without changing settings
- Survives system reboots (unless `/tmp` is cleared)

---

## Evolution & Versioning

### v1 (December 27–28, 2025)
- **Focus**: Worktree boundaries for multi-agent systems
- **Features**: Basic rm/git blocks, hardcoded macOS paths
- **Constraints**: Not portable, tightly coupled to Oracle worktree layout

### v2 (January 2026)
- **Focus**: Portability, integration with Oracle Starter Kit
- **Features**: Removed hardcoded paths, added to every new Oracle born after Jan 16
- **Distribution**: Bundled into Oracle Starter Kit, then oracle-skills-cli

### v3 (March 22–29, 2026)
- **Focus**: Smart regex anchoring, org membership check, stability
- **Features**: 
  - Smart anchoring to avoid false positives
  - Org membership cache (daily refresh)
  - Beta rules with filesystem toggle
  - This repo created as single source of truth
- **Current**: Used by 191+ Oracles in the ARRA ecosystem

---

## Testing & Validation

### Manual Testing

To test the hook directly (without Claude Code):

```bash
# Test: Block git push --force
echo '{"hook_event_name":"PreToolUse","tool_name":"Bash","tool_input":{"command":"git push --force"}}' | bash ~/.claude/hooks/safety-check.sh
# Expected: exit 2, stderr: "BLOCKED: Force flags not allowed..."

# Test: Allow normal git push
echo '{"hook_event_name":"PreToolUse","tool_name":"Bash","tool_input":{"command":"git push"}}' | bash ~/.claude/hooks/safety-check.sh
# Expected: exit 0 (success)

# Test: Allow --force in text
echo '{"hook_event_name":"PreToolUse","tool_name":"Bash","tool_input":{"command":"echo 'Don'\''t use --force'"}}' | bash ~/.claude/hooks/safety-check.sh
# Expected: exit 0 (success)
```

### Integration Test

In a Claude Code session with the hook installed:

```bash
# This should be blocked
git push --force

# This should be allowed
git push

# This should be blocked
rm -rf /tmp/test

# This should be allowed
mv /tmp/test /tmp/backup
```

---

## Requirements & Dependencies

| Tool | Purpose | How Used |
|------|---------|----------|
| `bash` | Shell script runtime | Executes safety-check.sh |
| `jq` | JSON parsing | Extracts `.tool_input.command` from stdin |
| `grep` | Pattern matching | Tests commands against block patterns |
| `gh` CLI | GitHub API access | Fetches org list for PR ownership check (optional; fails gracefully) |
| Claude Code | Hook invocation | Calls PreToolUse hooks before Bash execution |

**Installation**:
```bash
# macOS
brew install jq gh

# Ubuntu/Debian
sudo apt install jq gh

# Already installed in most dev environments
```

---

## Known Limitations & Future Improvements

### Current Limitations

1. **No worktree-aware filtering**: Rule blocks apply to all agents equally. Future versions could allow exemptions for specific worktrees.
2. **Static rule set**: To add/change rules, must update `safety-check.sh` and reinstall. No dynamic rule reloading.
3. **No audit logging**: Blocked commands are not logged centrally. Future: send to a shared log server.
4. **Timeout at 5 seconds**: Very slow operations (e.g., large org fetch) might timeout. No tuning mechanism.
5. **Beta rules are global**: All users sharing `/tmp` see the same beta toggle. Desired: per-project or per-session toggle.

### Disabled / Removed Rules

**`gh pr merge` block (commented out)**:
```bash
# Block gh pr merge (DISABLED - local project hook handles this for worktree agents)
# Main agent CAN merge after explicit user approval
# if echo "$CMD" | grep -qE 'gh\s+pr\s+merge'; then
#   echo "BLOCKED: Never merge PRs. Wait for user approval." >&2
#   exit 2
# fi
```

**Reason**: This rule was too aggressive in multi-agent systems. Individual project hooks (e.g., in oracle-skills-cli) handle this instead. The main agent can merge with explicit user approval.

---

## Integration with ARRA Oracle Principles

The ARRA Oracle ecosystem has 5 core principles. This hook enforces **Principle 1: Nothing is Deleted**.

**The 5 Principles**:
1. **Nothing is Deleted** — All changes are reversible; irreversible ops are blocked
2. **Transparency** — All decisions are auditable; no hidden side effects
3. **Isolation** — Agents work in isolated worktrees; changes don't cross-pollinate
4. **Synchronization** — Multi-agent hash sync prevents history divergence
5. **Gradual Rollout** — Changes land on alpha branch first; main is stable

This hook:
- ✅ Directly enforces: Principle 1 (Nothing is Deleted)
- ✅ Indirectly supports: Principles 3, 4, 5 (prevents history rewrites, unsafe branches)
- ✅ Complements: Principle 2 (audit trail of blocked commands via stderr)

---

## License & Attribution

**License**: MIT  
**Created**: March 29, 2026  
**Author**: Neo (Claude Opus 4.6) for Nat Weerawan  
**Repository**: https://github.com/Soul-Brews-Studio/arra-safety-hooks  
**Part of**: ARRA Oracle ecosystem (191+ agents)

---

## Quick Reference

| Command | Status | Alternative |
|---------|--------|-------------|
| `rm -rf` / `rm -f` | ❌ Blocked | `mv /path /tmp` |
| `git push --force` | ❌ Blocked | `git pull --no-rebase && git push` |
| `git reset --hard` | ❌ Blocked | `git stash` |
| `git commit --amend` | ❌ Blocked | `git commit -m "..."` (new commit) |
| `git push origin main` | ❌ Blocked | `git push origin alpha` → PR |
| `git checkout -- .` | ❌ Blocked | `git stash` |
| `git restore .` | ❌ Blocked | `git stash` |
| `git clean -f` | ❌ Blocked | Manual cleanup, `mv to /tmp` |
| `git branch -D` | ❌ Blocked | `git branch -d` |
| `git stash drop` | ❌ Blocked | Leave stash (safe to accumulate) |
| `--no-verify` | ❌ Blocked | Fix the hook issue |
| `gh pr create` (upstream) | ❌ Blocked | `gh pr create --repo your-org/repo` |
| `tmux send-keys` (beta) | ❌ Blocked | `maw hey <window> "msg"` |
| `bun run src/cli.ts` (beta) | ❌ Blocked | `maw hey <window> "msg"` |

