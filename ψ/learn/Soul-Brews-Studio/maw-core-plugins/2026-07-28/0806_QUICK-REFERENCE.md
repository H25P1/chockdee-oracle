# maw-core-plugins Quick Reference

Core plugins that ship with **maw-js** — essential agent lifecycle commands that cannot be removed. These are always present and load first (weight 00).

---

## Overview

**maw-core-plugins** contains four foundational plugins for managing oracle agents and work sessions:

| Plugin | Purpose | Status |
|--------|---------|--------|
| `wake` | Spawn or attach to oracle sessions | Core lifecycle primitive |
| `sleep` | Stop a single oracle's tmux window | Core lifecycle primitive |
| `stop` | Stop ALL oracle fleet sessions | Core lifecycle primitive |
| `done` | Finalize and cleanup a worktree | Worktree cleanup |

Each plugin:
- Runs with **weight 00** (loads first)
- Supports both **CLI** and **API** invocation
- Exposes an HTTP endpoint for programmatic control
- Captures stdout/stderr to return structured logs

---

## Plugins

### 1. wake — Spawn or Attach to Oracle Sessions

**Description:** Spawn a new oracle session or attach to an existing one. Supports launching specific tasks, fetching GitHub issues/PRs as prompts, and listing existing worktrees.

**CLI Usage:**

```bash
# Basic: spawn or attach named oracle
maw wake neo

# Create a new worktree (don't attach to existing)
maw wake neo --new workspace-name

# Spawn with a specific task
maw wake neo review-pr --task "Review PR #123"

# Fetch GitHub issue and use as prompt
maw wake neo --issue 42 --repo org/project

# Fetch GitHub PR and use as prompt
maw wake neo --pr 15 --repo org/project

# Fresh clone + new worktree
maw wake org/repo --fresh --new my-task

# List available worktrees
maw wake neo --list

# Don't auto-attach to session
maw wake neo --no-attach

# Wake all fleet sessions
maw wake all

# Wake all and kill zombie sessions
maw wake all --kill

# Resume paused sessions
maw wake all --resume

# Incubate mode (experimental)
maw wake org/project --incubate org/project
```

**CLI Flags:**

```
--new <name>       Create a new named worktree (don't reuse existing)
--incubate <repo>  Experimental incubation mode
--issue <N>        Fetch GitHub issue #N as prompt (requires --repo)
--pr <N>           Fetch GitHub PR #N as prompt (requires --repo)
--repo <org/name>  Repository context for issue/PR fetching
--task <prompt>    Custom task prompt (sets --no-attach)
--fresh            Fresh clone of repository
--no-attach        Spawn but don't attach to session
--list             List available worktrees for oracle
```

**API Usage:**

```bash
curl -X POST http://localhost:3000/api/wake \
  -H "Content-Type: application/json" \
  -d '{
    "oracle": "neo",
    "task": "review-code",
    "prompt": "Review the authentication module",
    "fresh": false,
    "noAttach": false
  }'
```

**API Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `oracle` | string | Yes | Name of the oracle session |
| `task` | string | No | Task identifier |
| `prompt` | string | No | Custom task prompt |
| `fresh` | boolean | No | Fresh clone (default: false) |
| `noAttach` | boolean | No | Spawn without attaching (default: false) |

**Configuration:** None — inherits maw-js session config.

**Exit Codes:** Returns `ok: true` on success, `ok: false` with error message on failure.

---

### 2. sleep — Gracefully Stop Single Oracle Window

**Description:** Gracefully shut down a single oracle agent's tmux session/window without affecting others in the fleet.

**CLI Usage:**

```bash
# Stop specific oracle session
maw sleep neo

# Stop specific window in oracle's session
maw sleep neo workspace-1

# Stop all agents (not yet implemented)
maw sleep --all-done
```

**CLI Arguments:**

| Arg | Required | Description |
|-----|----------|-------------|
| `oracle` | Yes | Name of oracle to sleep |
| `window` | No | Specific tmux window (optional) |

**API Usage:**

```bash
curl -X POST http://localhost:3000/api/sleep \
  -H "Content-Type: application/json" \
  -d '{
    "oracle": "neo",
    "window": "workspace-1"
  }'
```

**API Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `oracle` | string | Yes | Name of oracle to sleep |
| `window` | string | No | Specific tmux window (optional) |

**Configuration:** None — uses tmux environment.

**Notes:**
- `--all-done` flag is a placeholder and not yet implemented
- Graceful shutdown — allows running tasks to complete
- Does NOT affect other agents in the fleet

---

### 3. stop — Stop ALL Fleet Sessions

**Description:** Immediately stop all running oracle agent sessions across the entire fleet. Opposite of `wake all`.

**CLI Usage:**

```bash
# Stop all oracles
maw stop

# Alias: rest
maw rest
```

**Aliases:** `stop`, `rest`

**API Usage:**

```bash
curl -X POST http://localhost:3000/api/stop
```

**Configuration:** None — operates on entire fleet.

**Notes:**
- Stops ALL agents, not just one
- Use `sleep <oracle>` to stop a single agent
- No arguments or flags required

---

### 4. done — Cleanup Finished Worktree

**Description:** Complete cleanup of a finished worktree: run retrospective, git save, kill tmux window, remove worktree. Designed for completed tasks.

**CLI Usage:**

```bash
# Basic cleanup
maw done my-feature

# Force cleanup (skip confirmations)
maw done my-feature --force

# Dry run (preview cleanup, don't execute)
maw done my-feature --dry-run

# Alias: finish
maw finish my-feature
```

**CLI Arguments:**

| Arg | Required | Description |
|-----|----------|-------------|
| `window-name` | Yes | Name of worktree to cleanup |

**CLI Flags:**

```
--force    Skip confirmation prompts
--dry-run  Preview cleanup without executing
```

**API Usage:**

```bash
curl -X POST http://localhost:3000/api/done \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-feature",
    "force": false,
    "dryRun": false
  }'
```

**API Parameters:**

| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | Yes | Worktree name to cleanup |
| `force` | boolean | No | Skip confirmations (default: false) |
| `dryRun` | boolean | No | Preview only (default: false) |

**Cleanup Steps:**

1. **rrr** — Run retrospective/session summary
2. **git save** — Commit pending changes (with Co-Authored-By trailer)
3. **kill** — Terminate tmux window/session
4. **remove worktree** — Delete git worktree

**Configuration:** None — uses tmux and git environment.

**Aliases:** `done`, `finish`

---

## Installation & Bundling

### How They Ship

These 4 plugins are **bundled directly with maw-js**. They cannot be installed separately or removed.

```bash
maw-js/
├── plugins/
│   ├── 00-wake/       ← Automatic
│   ├── 00-sleep/      ← Automatic
│   ├── 00-stop/       ← Automatic
│   └── 00-done/       ← Automatic
└── node_modules/      ← maw-core-plugins distributed here
```

### Automatic Loading

When **maw-js** starts:

1. Loads all plugins from `plugins/` and `node_modules/**/maw-plugins/`
2. Weight **00** plugins load **first** (core lifecycle primitives)
3. Remaining plugins load by weight order
4. Plugins become available as CLI commands and API endpoints

**No additional setup required** — they're always present.

### vs. Optional Plugins

| Category | Repo | Install | Removal |
|----------|------|---------|---------|
| **Core** (this repo) | maw-core-plugins | Automatic | Cannot remove |
| **Optional** | [maw-plugins](https://github.com/Soul-Brews-Studio/maw-plugins) | `maw plugin install <name>` | `maw plugin uninstall <name>` |

---

## Configuration

**maw-core-plugins** read configuration from:

1. **Environment Variables** (if any)
   - None currently required for core plugins

2. **maw-js Global Config**
   - `~/.maw/config.json` (session preferences)
   - Plugins inherit session, fleet, and worktree context from maw-js

3. **Per-Plugin Flags** (CLI/API)
   - Each plugin accepts flags/parameters documented above

4. **Inherited from maw-js**
   - `~/.tmux.conf` — tmux session management
   - `.git/config` — git repository settings
   - System PATH — for executable discovery

**No configuration files** are present in the core-plugins repo itself — configuration is fully delegated to maw-js and the runtime environment.

---

## Plugin Metadata

Each plugin declares itself via `plugin.json`:

```json
{
  "name": "wake",
  "version": "1.0.0",
  "entry": "./index.ts",
  "sdk": "^1.0.0",
  "description": "Spawn or attach to an oracle session",
  "cli": {
    "command": "wake",
    "help": "maw wake <oracle|org/repo|URL> [task] ...",
    "flags": { /* ... */ }
  },
  "api": {
    "path": "/api/wake",
    "methods": ["POST"]
  }
}
```

**SDK Version:** All core plugins target SDK `^1.0.0`

---

## Plugin Architecture

### Handler Pattern

Each plugin exports a default async handler with signature:

```typescript
async function handler(ctx: InvokeContext): Promise<InvokeResult> {
  // ctx.source: "cli" or "api"
  // ctx.args: string[] (CLI) or Record<string, unknown> (API)
  // Returns: { ok: boolean; output?: string; error?: string }
}
```

### Invocation Context

**CLI Invocation:**
```typescript
{
  source: "cli",
  args: ["neo", "--new", "workspace"]  // parsed argv
}
```

**API Invocation:**
```typescript
{
  source: "api",
  args: {
    oracle: "neo",
    newWt: "workspace"
  }
}
```

### Return Format

**Success:**
```typescript
{
  ok: true,
  output: "Captured stdout/stderr logs"
}
```

**Failure:**
```typescript
{
  ok: false,
  error: "Error message with usage hint"
}
```

---

## Quick Workflow Examples

### Start a new task session

```bash
# 1. Wake oracle "my-agent" with fresh repo
maw wake my-org/my-repo --fresh --new feature-xyz

# 2. Later: specify a task inline
maw wake my-agent --task "Fix login bug"

# 3. Finish task
maw done feature-xyz --force
```

### Manage fleet

```bash
# Check all agents online
maw wake all --list

# Stop everything
maw stop

# Resume all
maw wake all --resume
```

### Fetch GitHub context

```bash
# Wake oracle with issue as prompt
maw wake neo --issue 123 --repo org/project

# Wake oracle with PR as prompt
maw wake neo --pr 45 --repo org/project
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `usage: maw wake <oracle> ...` | Missing oracle name. Provide: `maw wake neo` |
| `oracle is required` (API) | Missing `oracle` field in JSON body |
| `name is required` (API done) | Missing `name` field in JSON body for cleanup |
| Session won't start | Check tmux installed and `~/.tmux.conf` valid |
| Git issue/PR fetch fails | Verify `--repo org/name` provided and GitHub token configured |

---

## Related Documentation

- **maw-js** — Main orchestration engine
- **maw-plugins** — Optional plugins registry
- **Oracle Sessions** — Agent lifecycle and tmux integration
- **Worktrees** — Git worktree management strategy

---

**Version:** maw-core-plugins 1.0.0  
**License:** MIT  
**Author:** Soul-Brews-Studio  
**Updated:** 2026-07-28
