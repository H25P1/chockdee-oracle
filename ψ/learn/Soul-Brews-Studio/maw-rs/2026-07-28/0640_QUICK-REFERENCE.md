# maw-rs Quick Reference Guide

**maw-rs** is a distributed terminal multiplexing and fleet management system for AI agent oracles—a Rust port of [maw-js](https://github.com/Soul-Brews-Studio/maw-js). It provides robust session management, tmux integration, plugin architecture, and federation capabilities for orchestrating distributed compute fleets.

---

## Project Overview

### What It Does

maw-rs enables:

- **Terminal multiplexing**: Manage multiple terminal sessions and panes with enhanced tmux integration
- **Fleet orchestration**: Control and coordinate multiple compute nodes (agents, oracles, teams)
- **Distributed messaging**: Cross-node communication, peer federation, and inbox workflows
- **Plugin system**: Extensible runtime via WASM plugins and native Rust verbs
- **Team & workspace management**: Create teams, assign tasks, coordinate work across agents
- **Authentication & consent**: Built-in auth flows, trust management, and cross-scope security gates

### Architecture

The project uses a Rust **workspace** with modular crates:

```
crates/
├── maw-cli               (main binary: maw-rs)
├── maw-matcher           (target resolution logic)
├── maw-tmux              (tmux integration)
├── maw-transport         (routing & federation)
├── maw-worktree          (session/workspace tracking)
├── maw-peer              (peer discovery & comm)
├── maw-auth              (authentication primitives)
├── maw-plugin-manifest   (plugin descriptor parsing & WASM host)
├── maw-schedule          (scheduling core)
├── maw-schedule-launchd  (macOS launchd integration)
├── maw-schedule-runner   (scheduler runtime)
├── maw-xdg               (XDG paths & config resolution)
├── maw-discord           (Discord integration)
└── (others)              (support & utilities)
```

**Philosophy**: Leaf crates are deterministic and side-effect-free with fixture-based testing; mid-layer crates compose logic; `maw-cli` is the top-level binary and integration surface.

---

## Installation

### Prerequisites

- **macOS Apple Silicon** or **Linux x86_64** (prebuilt binaries available)
- curl or wget (for installer)
- tmux (for terminal multiplexing features)

### Option 1: Homebrew (macOS Apple Silicon Only)

```bash
brew install soul-brews-studio/maw/maw
maw --version
maw ls
```

**Update** to the next stable release:

```bash
brew upgrade maw
```

**Hold** a release:

```bash
brew pin maw                  # freeze current version
brew unpin maw                # allow updates
```

### Option 2: Release Installer (Prebuilt Binary)

Stable release (latest):

```bash
curl -fsSL https://github.com/Soul-Brews-Studio/maw-rs/releases/latest/download/install.sh | sh
```

Pin a specific CalVer release (e.g., `v26.7.16`):

```bash
curl -fsSL https://github.com/Soul-Brews-Studio/maw-rs/releases/download/v26.7.16/install.sh | MAW_VERSION=v26.7.16 sh
```

Or using environment variables:

```bash
MAW_VERSION=v26.7.16 INSTALL_DIR="$HOME/bin" sh install.sh
```

Bleeding-edge from `alpha` branch:

```bash
curl -fsSL https://raw.githubusercontent.com/Soul-Brews-Studio/maw-rs/alpha/install.sh | sh
```

**Supported platforms**:
- `maw-rs-macos-arm64` — macOS Apple Silicon
- `maw-rs-linux-x86_64-musl` — Linux x86_64 (static, musl libc)

**Post-install verification**:

```bash
maw --version
```

**If PATH is not set**, add to your shell profile (`.bashrc`, `.zshrc`, etc.):

```bash
export PATH="$HOME/.local/bin:$PATH"
```

**macOS Gatekeeper (if binary is blocked)**:

```bash
xattr -d com.apple.quarantine ~/.local/bin/maw
```

### Option 3: Build from Source

Requires Rust 1.97.0+ toolchain:

```bash
cargo install --path crates/maw-cli --features wasm-host
ln -sf "$(command -v maw-rs)" "$HOME/.local/bin/maw"
```

**Features**:
- `--features wasm-host` (default for ship-tier use): Includes Extism WASM runtime for plugin execution
- Without `--features wasm-host`: Lean build (~44% fewer dependencies); omits WASM plugins with graceful fallback hints

**Development build** (iterating):

```bash
cargo build --release -p maw-cli
```

**Run tests**:

```bash
cargo test --workspace
```

---

## Quick Start: Common Commands

### Session & Workspace Management

```bash
# List active sessions
maw ls
maw ls --all                    # include inactive sessions
maw ls --json                   # JSON output
maw ls --federation             # federated peer view

# Attach to a session
maw attach <session-id>         # attach and split if in tmux
maw a <session-id>              # short alias
maw a --dry-run <session-id>    # preview without attaching
maw a --print <session-id>      # print target info
maw a --readonly <session-id>   # read-only attach

# View current session/node info
maw session                      # display current session details
maw whoami                       # display current identity
```

### Fleet & Team Management

```bash
# Team operations
maw team spawn --repo <url>     # create a new team
maw team list                   # list all teams
maw team enter <team-id>        # switch to team
maw team status                 # show team status
maw team members                # list team members
maw team assign <agent>         # assign work to agent

# Swarm (spawn multiple parallel agents)
maw swarm --count 5             # spawn 5 parallel agents
maw swarm --split --tiled       # tiled layout

# Overall fleet health
maw fleet status                # check fleet status
maw fleet doctor                # diagnose & repair issues
maw activity                    # monitor agent activity
```

### Plugin Management

```bash
# List plugins
maw plugins ls                  # list installed plugins
maw plugins ls -v               # verbose with details
maw plugins ls --json           # JSON output

# Plugin creation (Rust → WASM workflow)
maw plugin create --rust my-plugin
cd my-plugin
maw plugin build                # compile wasm32-unknown-unknown
maw plugin dev                  # test in dev environment

# Plugin info
maw plugin-manifest parse <path>  # validate plugin.json
maw plugin-manifest discover      # scan for plugins
```

### Messaging & Communication

```bash
# Send messages to peer
maw send <target> "hello"       # send message
maw hey <target> "status?"      # synchronous message with reply

# Read inbox
maw inbox status                # check pending messages
maw inbox read                  # show messages
maw inbox approve               # approve/trust sender
maw inbox reject                # reject message

# Broadcast
maw broadcast --fleet "update"  # announce to fleet
```

### Peer & Federation

```bash
# Peer discovery
maw peers ls                    # list discovered peers
maw peers add <addr>            # add peer manually
maw peers probe <peer>          # check peer reachability
maw peers remove <peer>         # forget peer

# Federation sync
maw federation status           # check federation state
maw federation sync             # sync state across peers
```

### Authentication & Trust

```bash
# Consent/approval workflows
maw consent approve <request>   # approve a request
maw consent reject <request>    # reject a request
maw consent list                # show pending consents

# Trust management
maw trust add <peer>            # trust a peer
maw trust remove <peer>         # untrust a peer
maw trust ls                    # list trusted peers

# Scope (ACL boundaries)
maw scope create --lead <peer>  # create access scope
maw scope ls                    # list scopes
```

### Workspace & Project Management

```bash
# Work on a repository
maw work <url>                  # scaffold and enter workspace
maw awaken --repo <url>         # create oracle from repo
maw wake --pr <number>          # focus on PR

# Project discovery
maw project list                # list projects
maw project find <pattern>      # search projects

# Session snapshots
maw snapshots                   # list saved snapshots
```

### Debugging & Admin

```bash
# Diagnostics
maw about                       # system info
maw check tools                 # verify tool availability
maw doctor --fix                # auto-repair issues

# Activity monitoring
maw activity --watch            # stream activity
maw activity --json             # structured output

# Background jobs
maw bg ls                       # list background tasks
maw bg follow <job>             # tail job output
```

### Config & Setup

```bash
# User setup
maw user-setup                  # initial setup wizard
maw config set <key> <value>    # set config value
maw config get <key>            # read config value

# Generate shell completions
maw completions zsh > ~/.zshrc.d/maw-completions
maw completions bash > ~/.bashrc.d/maw-completions
```

---

## Configuration

### Config File Locations

maw-rs follows **XDG Base Directory Specification**:

- **Linux**: `~/.config/maw/`
- **macOS**: `~/Library/Application Support/maw/`

### Weighted Configuration Layers

If any weighted layer (`maw.config.<N>.json` or `maw.config.<N>.local.json`) exists in the active config directory, **legacy `maw.config.json` is ignored**.

**Weighted layer example**:

```
~/.config/maw/
├── maw.config.0.json           # layer 0 (lowest priority)
├── maw.config.1.json           # layer 1
├── maw.config.5.json           # layer 5
├── maw.config.99.json          # layer 99 (highest priority)
├── maw.config.99.local.json    # local overrides (highest)
└── (legacy maw.config.json is ignored if any .N. layer exists)
```

**Priority order** (highest to lowest):
1. `maw.config.<N>.local.json` (local machine overrides)
2. `maw.config.<N>.json` (layers by number, high to low)
3. Defaults

### Config File Format (JSON Example)

```json
{
  "node": "agent-01",
  "port": 3000,
  "hub": {
    "url": "https://hub.example.com",
    "token": "..."
  },
  "peers": [
    {
      "name": "peer-02",
      "address": "192.168.1.10:3001"
    }
  ],
  "tmux": {
    "default_shell": "zsh",
    "layout": "tiled"
  },
  "plugins": {
    "enabled": ["stream", "hub", "layout"],
    "scan_dirs": ["/opt/maw-plugins"]
  }
}
```

**Common settings**:
- `node` — unique identifier for this instance
- `port` — HTTP server port for communication
- `hub` — federation hub URL and authentication
- `peers` — static peer list
- `tmux` — tmux-specific defaults
- `plugins` — plugin enable/disable and scan paths

---

## Key Features with Examples

### 1. Target Resolution

maw-rs resolves targets flexibly:

```bash
maw a session-id                # by session ID
maw a team/agent-01             # by team/agent path
maw a @peer-name                # by peer nickname
maw a active                    # current active session
maw a .                         # current session
```

Targets support wildcards and fuzzy matching.

### 2. Tmux Integration

Raw tmux commands should **never** be used directly; use maw verbs instead:

```bash
# Instead of:                   Use maw:
tmux split-window               → maw split --vertical
tmux send-keys "cmd" Enter      → maw send-text "cmd"
tmux kill-window                → maw kill <target>
tmux select-pane -T "title"     → maw rename-pane "title"
tmux select-pane                → maw focus <pane>
tmux resize-pane                → maw resize <size>
```

### 3. Plugin Architecture

**Native verbs** (fast path, ~81 of 133 total):
- Built into the maw-rs binary
- No WASM overhead
- Examples: `ls`, `hey`, `team`, `attach`, `split`

**WASM plugins** (ship-tier, ~29 of 133 total):
- Compiled to `wasm32-unknown-unknown` via Extism
- Extracted to external [`maw-plugins`](https://github.com/Soul-Brews-Studio/maw-plugins) repo
- Examples: `stream`, `hub`, `layout`, `tile`, `dream`, `costs`
- Require `--features wasm-host` build

**Dev-tier plugins** (Bun/TypeScript):
- Not compiled into ship builds
- Remain first-class for local development
- Load via plugin scaffold

**Example: Create a Rust WASM plugin**:

```bash
maw plugin create --rust metrics-monitor
cd metrics-monitor
# Edit src/lib.rs with Extism SDK
maw plugin build
maw plugin dev --watch
```

### 4. Peer Federation & Trust

**Federation** connects independent maw instances across a network:

```bash
# Bootstrap trust
maw peers tofu-bootstrap <addr>      # Trust-On-First-Use
maw trust add <peer>                 # explicit trust
maw scope create --lead <peer>       # create ACL boundary

# Verify federation
maw federation status                # check peer connections
maw federation sync --dry-run        # preview sync
maw federation sync --apply          # sync state
```

### 5. Session Snapshots

Preserve and restore fleet state:

```bash
maw snapshots                        # list saved sessions
maw wake --from-snapshot <id>        # restore from snapshot
maw wake --snapshot <path>           # load custom snapshot
```

### 6. Consent & Approval Workflows

Cross-scope requests require explicit approval:

```bash
# Peer sends request (blocks until approval)
maw send untrusted-peer "approve-me"

# Lead receives in inbox
maw inbox pending
maw inbox approve <request-id>       # grant access
maw inbox reject <request-id>        # deny access
```

---

## Development

### Build Gates

Before committing, run:

```bash
# Quick check (iterating)
scripts/gate.sh quick              # fmt + clippy(stable) + affected tests

# Full gate (before merge)
scripts/gate.sh full               # all 4 CI dimensions + wasm-host

# Batch check multiple branches
scripts/gate.sh batch branch1 branch2 ...
```

### Cargo Isolation Rule

**Never wait for other cargo processes**—isolate your target dir:

```bash
CARGO_TARGET_DIR=/tmp/maw-rs-target-<unique-id> cargo test ...
CARGO_TARGET_DIR=/tmp/maw-rs-target-<unique-id> cargo clippy ...
```

### Workspace & Style

- **Rust edition**: 2021
- **unsafe_code**: Forbidden (workspace lint)
- **Clippy**: Pedantic warnings treated as errors in CI
- **Fixtures**: All observable behavior validated against maw-js JSON fixtures
- **Linting**: `cargo fmt --all` before commit

### Adding a Command

Auto-registration via `core_impl/part*.rs`:

```bash
# Create new file: crates/maw-cli/src/core_impl/partNN_mycommand.rs
#
# impl DispatcherEntry for MyCommand {
#     const DISPATCH_NN: u16 = 42;  // unique number
#     const COMMAND: &'static str = "mycommand";
#     // ... implementation
# }
#
# build.rs auto-discovers DispatcherEntry registrations
```

No manual dispatcher registration needed.

### Testing

All crates have fixture-based unit tests:

```bash
cargo test -p maw-cli          # CLI tests
cargo test --workspace         # all crates
cargo test -p maw-cli --test <fixture_name>  # specific fixture
```

---

## Parity & Command Status

maw-rs is a **Phase 1** Rust port of maw-js with the following completion status (as of 2026-07-28):

- **Native ✅**: 81 commands (built-in Rust verbs)
- **WASM ✅**: 29 commands (ship-tier plugins, require `--features wasm-host`)
- **Stub ⚠️**: 13 commands (partial implementation)
- **NOT-PORTED ❌**: 10 commands (intentional no-code or deferred)

**Total**: 133 verbs mapped from maw-js.

See `docs/parity/parity-matrix.md` in the repo for detailed coverage by category (messaging, tmux, teams, plugins, auth, etc.).

### High-Priority Verbs

Native implementations:
- `ls` — list sessions
- `hey` — peer messaging
- `attach` — join session
- `team` — team management
- `swarm` — parallel spawning
- `awaken` — oracle creation
- `plugin` — plugin lifecycle
- `consent` — trust workflows

WASM plugins (ship-tier, `--features wasm-host`):
- `stream` — stream capture
- `hub` — federation hub
- `layout` — tmux layouts
- `tile` — advanced splits
- `dream` — activity logs
- `costs` — resource tracking

---

## Troubleshooting

### Command Not Found

Ensure maw is on PATH:

```bash
which maw
echo $PATH
```

Add to shell profile if needed:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

### WASM Plugin Error: `--features wasm-host`

Some plugins (`stream`, `hub`, `layout`) require the WASM host:

```bash
# Error on default build:
$ maw stream --into output
plugin 'stream' is a ship-tier WASM plugin. 
Rebuild with: cargo install --path crates/maw-cli --features wasm-host

# Solution:
cargo install --path crates/maw-cli --features wasm-host
```

### Tmux Issues

Verify tmux is running and accessible:

```bash
tmux list-sessions
maw tmux ls                     # maw-native tmux listing
```

If attach fails, check `TMUX` environment variable:

```bash
echo $TMUX
maw attach --dry-run <target>  # preview attach flow
```

### Config Not Loading

Verify config file exists and is valid JSON:

```bash
# Linux/BSD:
~/.config/maw/maw.config.json

# macOS:
~/Library/Application\ Support/maw/maw.config.json

# Validate:
jq empty maw.config.json
```

If using weighted layers, ensure filename pattern matches:

```bash
ls -la ~/.config/maw/maw.config.*.json
```

### Peer Connection Issues

Test peer reachability:

```bash
maw peers probe <peer>          # check connectivity
maw federation status --json    # detailed status
maw discovery --peers           # list reachable peers
```

---

## Resources

- **GitHub**: https://github.com/Soul-Brews-Studio/maw-rs
- **Releases**: https://github.com/Soul-Brews-Studio/maw-rs/releases
- **Homebrew Formula**: https://github.com/Soul-Brews-Studio/homebrew-maw
- **Plugins**: https://github.com/Soul-Brews-Studio/maw-plugins
- **Fixtures**: https://github.com/Soul-Brews-Studio/maw-fixtures

### Documentation Files

- `docs/install.md` — Detailed install methods
- `docs/guides/` — How-tos for development
- `docs/reference/` — Technical specs (wire protocol, CLI dispatch, plugin invoke)
- `docs/design/` — Architecture decisions (WASM migration, plugin design, etc.)
- `docs/parity/` — maw-js parity matrix and scorecard

---

## Version & CalVer

maw-rs uses **Calendar Versioning (CalVer)**:

- Format: `v26.7.16` (year.month.day)
- Example: `v26.7.16` = July 16, 2026
- Stable releases published weekly
- Alpha branch: `alpha` branch, daily builds

Check version:

```bash
maw --version
maw about
```

---

## License

Licensed under **BUSL-1.1** (Business Source License).

---

**Last updated**: 2026-07-28  
**Scope**: Installation, quick start, configuration, development
