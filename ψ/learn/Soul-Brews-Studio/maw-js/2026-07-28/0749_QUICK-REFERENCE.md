# maw-js — Quick Reference Guide

**maw** is a CLI for running, coordinating, and monitoring multiple AI agents across machines. Built on Bun/TypeScript, it orchestrates agents via tmux, manages workflows across nodes via HTTP federation, and provides a unified dashboard (maw-ui) for the mesh.

**Key tagline**: Wake agents, send tasks, watch the screen, see what it cost — all from one terminal.

---

## Installation

### Quick Install (Recommended)
```bash
curl -fsSL https://raw.githubusercontent.com/Soul-Brews-Studio/maw-js/main/install.sh | bash
```

### Install with Bun Package Manager
```bash
bun add -g github:Soul-Brews-Studio/maw-js
```

### Install from Source
```bash
ghq get Soul-Brews-Studio/maw-js && \
cd "$(ghq root)/github.com/Soul-Brews-Studio/maw-js" && \
bun install && \
bun link
```

### Versioning
maw-js uses **CalVer** format: `v{yy}.{m}.{d}[-alpha.{HHMM}]` (e.g., `v26.6.14-alpha.2110`)

### Recovery
If `maw` command disappears unexpectedly:
```bash
# Option 1: Reinstall
bun add -g github:Soul-Brews-Studio/maw-js

# Option 2: Self-heal
bunx -p github:Soul-Brews-Studio/maw-js maw doctor

# Option 3: Shell hook (add to .bashrc/.zshrc)
source scripts/maw-heal.sh
```

---

## Quick Start

### Minimal Workflow
```bash
maw serve                    # Start API server on :3456
maw wake neo                 # Wake agent "neo" (auto-clones repo if needed)
maw hey neo "what's up?"     # Send message to neo's pane
maw peek neo                 # View neo's tmux pane on screen
maw sleep neo                # Gracefully stop neo
```

### Setup UI (Optional)
```bash
maw ui install               # Download the federation lens UI
maw ui                       # Open lens at http://localhost:3456
maw ui <peer-name>           # Point lens at a remote peer
```

---

## Core Commands

### Session Management

| Command | Aliases | Purpose |
|---------|---------|---------|
| `maw wake <oracle>` | `wake` | Wake an oracle session; fuzzy-matches repos and auto-clones |
| `maw awake <oracle>` | `awake` | Launch/start an oracle process with a specific engine |
| `maw bring <oracle>` | `b`, `bring` | Thin alias: `wake` in split mode (bring HERE) |
| `maw sleep <oracle>` | `sleep` | Gracefully stop an oracle session |
| `maw new <name>` | `new` | Create a plain tmux workspace session |
| `maw done <window>` | `done` | Auto-save, clean up worktree/branch, and close window |
| `maw bud <name>` | `bud` | Spawn a new oracle from scratch or budded from a parent |

### Viewing & Navigation

| Command | Aliases | Purpose |
|---------|---------|---------|
| `maw ls` | `ls` | List local sessions (compact view by default) |
| `maw ls -v` | — | Verbose: full per-pane detail |
| `maw ls --all` | `-a` | Include sleeping oracles (roster view) |
| `maw ls --recent [N]` | `-r` | Sort by creation time; limit to N |
| `maw ls --active [DUR]` | — | Filter sessions touched within duration (e.g., 30m, 1h) |
| `maw ls --federation` | — | Show local + peer inventory (cross-machine) |
| `maw peek [agent]` | `peek` | View agent's screen (read-only tmux capture) |
| `maw attach <agent>` | `a` | Attach to a tmux session interactively |

### Communication

| Command | Purpose |
|---------|---------|
| `maw hey <agent> "<msg>"` | Send message to an agent's inbox + pane (default) |
| `maw hey <agent> "<msg>" --inbox` | Queue message to inbox only (no pane injection) |
| `maw hey <node>:<agent>` | Canonical form: send to remote node |
| `maw hey <node>:<agent>:<window>` | Pick a specific tmux window (#410) |
| `maw broadcast <msg>` | Send message to all team members (`shout` alias) |

### Fleet & Workspace

| Command | Purpose |
|---------|---------|
| `maw team up <team>` | Spawn a charter-driven team from `.maw/teams/<team>.yaml` |
| `maw team down <team>` | Graceful shutdown of team and worktrees |
| `maw team create <team>` | Create a team workspace session |
| `maw team bring <team>` | Bring oracles into a team workspace |
| `maw team reassign <agent> "<task>"` | Kill, wake fresh, re-dispatch |
| `maw oracle-invite <agent> --team <team>` | Add oracle to existing team |
| `maw scaffold <name>` | Create oracle repo + skeleton only (no /awaken) |
| `maw snapshots list` | Browse wake recovery snapshots |
| `maw snapshots show <id>` | Inspect a specific snapshot |

### Tmux Control

| Command | Aliases | Purpose |
|---------|---------|---------|
| `maw tile [N]` | `tile` | Tile current window or spawn N panes tiled |
| `maw tmux split` | `split` | Split pane and attach |
| `maw tmux kill [target]` | `kill` | Kill a pane or session |
| `maw tmux open [target]` | `open` | Bring back hidden panes (join-pane) |
| `maw tmux close [target]` | `close` | Hide panes without killing (break-pane) |
| `maw tmux zoom [target]` | `zoom` | Toggle zoom on a pane |
| `maw layout <preset>` | — | Apply tmux layout: even-horizontal, even-vertical, main-horizontal, main-vertical, tiled (or `reset` → main-vertical) |
| `maw tmux ls` | `panes` | List all panes across sessions with `--verbose` |

### Plugins & Ecosystem

| Command | Purpose |
|---------|---------|
| `maw plugin ls` | List installed + tiered plugins (core, standard, extra) |
| `maw plugin install <name>` | Install from maw-plugin-registry or peers |
| `maw plugin enable <name>` | Opt into a disabled-but-installed plugin |
| `maw <plugin> serve` | Run a plugin-owned browser or process UI |
| `maw fck` | Command correction plugin (experimental) |
| `maw fleet-ui serve` | Fleet dashboard plugin |
| `maw messages serve` | Message ledger browser plugin |

### Diagnosis & Maintenance

| Command | Purpose |
|---------|---------|
| `maw preflight` | Pre-flight check: version, plugins, dead agents, config |
| `maw doctor` | Detailed diagnostic output |
| `maw doctor xdg` | Show active config/state/data/cache roots (XDG migration) |
| `maw doctor xdg --migrate --dry-run` | Preview safe copy-forward moves |
| `maw doctor xdg --migrate` | Migrate legacy artifacts into XDG targets |
| `maw fleet sync` | Sync fleet configuration across nodes |
| `maw ping` | Check peer connectivity (federation only) |
| `maw version` | Show installed version |
| `maw update` | Update to latest release |

### Federation (Cross-Machine)

| Command | Purpose |
|---------|---------|
| `maw serve [port]` | Start API server (default: 3456) |
| `maw peek <node>:<agent>` | Peek at an agent's screen on a remote node |
| `maw ping` | Check peer connectivity |
| `maw pair generate` | Generate 6-char ephemeral handshake code |
| `maw pair <url> <code>` | Complete peer pairing with handshake |

---

## Configuration

### File Locations
- **Config**: `~/.config/maw/maw.config.json` (or `$XDG_CONFIG_HOME/maw/maw.config.json`)
- **State**: `~/.local/state/maw/` (XDG) or `~/.maw/` (legacy)
- **Data**: `~/.local/share/maw/` (XDG) or `~/.maw/` (legacy)
- **Plugins**: `~/.local/share/maw/plugins/` (XDG)

### Configuration Example
```json
{
  "host": "local",
  "port": 3456,
  "bind": "0.0.0.0",
  "oracleUrl": "http://localhost:47779",
  
  "env": {
    "CLAUDE_CODE_OAUTH_TOKEN": "<your-token>"
  },
  
  "commands": {
    "default": "claude --dangerously-skip-permissions --continue",
    "*-oracle": "claude --dangerously-skip-permissions --continue",
    "codex-*": "codex --dangerously-auto-approve --search"
  },
  
  "defaultEngine": "claude",
  
  "sessions": {
    "nexus": "01-oracles",
    "hermes": "07-hermes",
    "pulse": "09-pulse"
  },
  
  "federationToken": "shared-secret-min-16-chars",
  "allowPeersWithoutToken": false,
  "trustLoopback": true,
  
  "namedPeers": [
    { "name": "white", "url": "http://10.20.0.7:3456" },
    { "name": "clinic-nat", "url": "http://10.20.0.1:3457" }
  ]
}
```

### Key Configuration Fields

| Field | Type | Purpose |
|-------|------|---------|
| `host` | string | Node identity / listening address |
| `port` | number | HTTP API server port (default: 3456) |
| `bind` | string | API server bind address (e.g., "0.0.0.0" for federation, "127.0.0.1" for local-only) |
| `oracleUrl` | string | URL where Claude Code runs (default: http://localhost:47779) |
| `env` | object | Environment variables passed to spawned agents |
| `commands` | object | Engine command templates (key = pattern, value = shell command) |
| `defaultEngine` | string | Default engine when no pattern matches |
| `sessions` | object | Named session mappings (key = oracle, value = tmux session slot) |
| `federationToken` | string | Shared secret for cross-node HMAC auth (min 16 chars) |
| `namedPeers` | array | Known remote nodes: `[{name, url}, ...]` |
| `idleTimeoutMinutes` | number | Auto-sleep agents after idle time |
| `tmuxSocket` | string | Custom tmux socket path |

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `MAW_HOME` | Override the maw data directory root |
| `MAW_XDG` | Set to `1` to use XDG Base Directory spec paths |
| `MAW_PLUGINS_DIR` | Custom plugins directory |
| `MAW_CLI` | Set internally during CLI runs |
| `MAW_GATEWAY` | Force gateway: `bun` or `rust` |
| `MAW_ALLOW_SELF_BRING` | Set to `1` to allow split-bringing an oracle into its own pane |
| `MAW_TEST_MODE` | Set to `1` for test harness |
| `MAW_PARENT_SESSION_ID` | Passed to spawned agents for parent tracking |
| `MAW_SESSION_ID` | Set session ID explicitly |

---

## Key Features & Workflows

### Multi-Agent Teams

**Decision tree:**
- Will the work **outlive this session** AND **span multiple machines**?
  → Use `maw team up` + `oracle-invite` (persistent, cross-machine)
  
- Will the work **outlive this session** but stay **on one machine**?
  → Use `maw team up` (persistent, single machine)
  
- **Session-only coordination** with structured messaging?
  → Use `/team-agents` (Claude Code skill) with optional `--worktree` isolation
  
- Just **A/B-comparing engines**?
  → Use `maw swarm` (no coordination layer)
  
- Just want a **peer visible**? Or **spawning N panes**?
  → Use `maw bring` or `maw tile` respectively

**Charter-driven team (recommended):**
```bash
# Create .maw/teams/my-team.yaml
maw team up my-team --dry-run    # Preview spawn plan
maw team up my-team              # Spawn from charter
maw team down my-team            # Graceful shutdown
```

**Codex team pattern:** See [docs/codex-team-pattern.md](docs/codex-team-pattern.md) for sprint charters spawning multiple Codex builders.

### Federation (Cross-Node Communication)

Enable cross-machine agent coordination:

```json
{
  "node": "oracle-world",
  "federationToken": "shared-secret-min-16-chars",
  "namedPeers": [
    { "name": "white", "url": "http://10.20.0.7:3456" },
    { "name": "clinic-nat", "url": "http://10.20.0.1:3457" }
  ]
}
```

**Usage:**
```bash
maw hey white:neo "hello"         # Send to neo on node white
maw hey white:neo:3 "hello"       # Pick specific tmux window #3
maw peek white:neo                # See their screen
maw ping                          # Check peer connectivity
maw ui white                      # Lens pointed at white's data
```

**API Endpoints** (federation peers):
- `GET /api/config` — Node identity + agents map
- `GET /api/fleet-config` — Fleet entries with lineage
- `GET /api/feed?limit=200` — Live event log
- `GET /api/federation/status` — Peer reachability

### Wake Lifecycle

Wake is now a lifecycle with preview, snapshot, and restore capabilities:

```bash
maw wake neo --dry-run           # Inspect target/session/plugin effects
maw wake neo --list              # Preview worktrees only
maw wake neo --from-snapshot     # Restore from latest snapshot
maw wake neo --snapshot <id>     # Restore from specific snapshot
maw snapshots list               # List captured wake state
maw snapshots show <id>          # Inspect snapshot contents
```

**Wake Flags:**
- `--work .` / `--oracle` — Mode: work (repo identity, keep ψ/ local) or oracle (dedicated session)
- `--task "<s>"` — Send initial task to oracle
- `--wt <name>` — Use named worktree (stable, reusable)
- `--fresh` / `--new` — Force new numbered worktree slot
- `--pick` — Open reusable worktree picker
- `--layout nested|legacy` — Worktree filesystem layout
- `--bud` — Write ψ/.lineage.yaml (no repo/fleet mutation)
- `--signal-on-birth` — Drop parent ψ/memory/signals birth signal
- `--engine <name>` / `-e <name>` — Override engine (claude, codex, aider, etc.)
- `--session <session>` — Wake into foreign workspace session
- `--split` — Split pane (used by `bring` alias)
- `--attach` / `-a` — Wait for engine process before returning
- `--main` / `--solo` / `--no-rehydrate` — Skip rehydration on restart
- `--wait` — Wait for engine process after bootstrap

### Worktrees & Layout

Oracles now support **two filesystem layouts:**

**Nested (default, modern):**
```
repo/
├── agents/
│   ├── 1-neo/
│   ├── 2-white/
│   └── 3-clinic-nat/
└── ψ/
    ├── .lineage.yaml
    ├── memory/
    └── signals/
```

**Legacy:**
```
repo/
├── .wt-1-neo/
├── .wt-2-white/
├── .wt-3-clinic-nat/
└── ψ/
```

Migrate or force a layout:
```bash
maw wake oracle --layout legacy   # Use legacy layout
maw wake oracle --layout nested   # Use nested layout (default)
```

### Plugins

maw now functions as a plugin OS. Core commands stay small; registry plugins add focused tools.

**Main plugin surfaces:**
- **CLI commands** — new `maw <plugin>` entry points
- **Browser/process UIs** — via `maw <plugin> serve`
- **Lifecycle hooks** — `hooks.wake`, `hooks.serve`, `hooks.sleep`
- **Plugin manifest** — `plugin.json` declares CLI, capabilities, APIs

**Usage:**
```bash
maw plugin ls                    # Installed + tiered plugins
maw plugin install <name>        # Install from registry or peer
maw plugin enable <name>         # Opt into disabled plugin
maw <plugin> serve               # Run plugin-owned UI
```

**Notable plugins:**
- `fleet-ui` — Fleet dashboard
- `messages` — Message ledger browser
- `fck` — Command correction
- `team` — Team management (23 subcommands)
- `swarm` — Multi-engine A/B panes
- `avengers` — Multi-agent Avengers framework

### XDG Base Directory Spec

maw supports the XDG spec for portable runtime paths. Opt in:

```bash
MAW_XDG=1 maw doctor xdg         # Opt into XDG paths
maw doctor xdg --migrate --dry-run  # Preview moves
maw doctor xdg --migrate         # Migrate legacy artifacts
```

**Default paths (XDG):**
- Config: `$XDG_CONFIG_HOME/maw/` (default: `~/.config/maw/`)
- State: `$XDG_STATE_HOME/maw/` (default: `~/.local/state/maw/`)
- Data: `$XDG_DATA_HOME/maw/` (default: `~/.local/share/maw/`)
- Cache: `$XDG_CACHE_HOME/maw/` (default: `~/.cache/maw/`)

**Legacy paths (still readable):**
- All under `~/.maw/` or `~/.config/maw/`

---

## CLI Examples by Use Case

### Scenario 1: Start a Fresh Agent
```bash
maw wake fusion --engine claude --task "review PR #42"
# → Auto-clones fusion-oracle, starts Claude Code, sends task
```

### Scenario 2: Check on Multiple Agents
```bash
maw ls -v                        # Full detail on all sessions
maw ls --recent 5                # Newest 5 oracles
maw peek neo                     # Peek at neo's screen
```

### Scenario 3: Send a Message
```bash
maw hey neo "what's the status?"         # Inject into pane + inbox
maw hey neo "later" --inbox              # Queue only (no pane)
maw broadcast "all hands meeting"        # Send to team members
```

### Scenario 4: Bring a Peer to Your Pane
```bash
maw bring neo                    # Wake neo in split pane
maw bring neo --engine codex     # Use Codex instead
maw bring neo --to work:review   # Explicit target session/window
maw bring neo --pick             # Fuzzy prompt when ambiguous
```

### Scenario 5: Spawn a Team
```bash
# Create .maw/teams/sprint-q4.yaml with charter
maw team up sprint-q4 --dry-run  # Preview plan
maw team up sprint-q4            # Spawn all members
maw team bring sprint-q4         # Attach to team workspace
maw team down sprint-q4          # Graceful shutdown
```

### Scenario 6: Cross-Node Communication
```bash
maw hey white:neo "status?"      # Remote peer communication
maw peek white:neo               # Peek at white's neo pane
maw ls --federation              # See all nodes + agents
maw ui white                      # Lens on white's data
```

### Scenario 7: Layout & Tmux Management
```bash
maw tile 4                       # Spawn 4 panes tiled in current window
maw layout main-vertical         # Apply main-vertical layout
maw tmux split                   # Split current pane
maw tmux kill neo                # Kill neo pane
maw panes                        # List all panes (-v for verbose)
```

### Scenario 8: Restore from Snapshot
```bash
maw snapshots list               # List available snapshots
maw wake neo --from-snapshot     # Restore latest
maw wake neo --snapshot abc123   # Restore specific ID
```

---

## Architecture

```
maw-js (backend + CLI)
├── src/commands/plugins/       (CLI + plugin dispatch)
├── src/api/                    (engine + plugin APIs)
├── src/engine/                 (WebSocket + serve proxy)
├── src/transports/             (HTTP/tmux/hub)
├── src/config/                 (config loading + validation)
├── plugins/                    (89 vendor plugin surfaces)
└── test/                       (800+ test files)

maw-ui (frontend — separate repo)
├── src/components/
├── src/hooks/
├── src/lib/
└── 16 HTML entry points (federation lens, etc.)
```

**Runtime:** Bun 1.3+ (fast, bundled TypeScript runtime)  
**License:** BUSL-1.1 (Business Source License)

---

## Roadmap & Evolution

| Date | Version | Milestone |
|------|---------|-----------|
| Oct 2025 | maw.env.sh | 30+ shell commands |
| Mar 2026 | maw.js | Bun/TS rewrite, tmux orchestration |
| Apr 2026 | v2.0.0-alpha.66 | Plugin OS, federation API, maw-ui split |
| May 2026 | v26.5.20 | Plugin engine, lifecycle hooks, 89 plugins, 800+ tests |
| Jun 2026 | v26.6.6 | Charter-driven teams, lean-core extraction |
| Ongoing | maw-rs | Rust port (portable JSON fixture specs) |

---

## Troubleshooting

### Command Not Found
```bash
# Self-heal
maw doctor

# Or reinstall
bun add -g github:Soul-Brews-Studio/maw-js
```

### Agent Won't Wake
```bash
maw wake neo --dry-run       # Preview what would happen
maw preflight                # Check config + plugins
maw doctor                   # Detailed diagnostics
```

### Federation Peers Not Reachable
```bash
maw ping                     # Check connectivity
maw ls --federation          # See all nodes
maw doctor                   # Verify token + config
```

### Session Cleanup
```bash
maw done <window>            # Auto-save + clean up worktree/branch
maw tmux kill <session>      # Force kill session
```

---

## Further Reading

- **README**: Full overview + installation variants
- **docs/teams.md**: Complete team/coordination reference (20+ verbs, 4 APIs, 10+ skills)
- **docs/federation.md**: Federation API v1 spec (GET /api/config, /feed, /federation/status)
- **docs/codex-team-pattern.md**: Sprint charters for multi-Codex teams
- **docs/bud.md**: Oracle creation patterns
- **docs/demo-script.md**: Walkthrough with working examples
- **docs/plugins/**: Plugin architecture, registry, marketplace
- **CONTRIBUTING.md**: Versioning (CalVer), release process, contributor guide

---

## Getting Help

```bash
maw --help                   # Show all commands
maw <command> --help         # Show command-specific help
maw preflight                # System diagnostics
```

For issues: [github.com/Soul-Brews-Studio/maw-js/issues](https://github.com/Soul-Brews-Studio/maw-js/issues)

---

**Last Updated:** 2026-07-28  
**Version Context:** maw-js v26.6.14-alpha.2110+
