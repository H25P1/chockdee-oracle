# maw-rs Architecture Documentation

**Distributed terminal multiplexing & fleet management for AI agent oracles**

Date: 2026-07-28  
Project: maw-rs (Rust port of maw-js)  
Repository: https://github.com/Soul-Brews-Studio/maw-rs

---

## Table of Contents

1. [Overview](#overview)
2. [Directory Structure](#directory-structure)
3. [Cargo Workspace Organization](#cargo-workspace-organization)
4. [Core Abstractions](#core-abstractions)
5. [Module Dependencies](#module-dependencies)
6. [Key Crates](#key-crates)
7. [Entry Points](#entry-points)
8. [Design Patterns](#design-patterns)

---

## Overview

**maw-rs** is a Rust port of the maw-js TypeScript project. It provides distributed terminal multiplexing and fleet management for AI agent oracles — enabling orchestration of agents across multiple nodes, sessions, and windows within a tmux-based environment.

### Project Philosophy

The project follows a **deterministic, side-effect-free** architecture where logic is ported incrementally:

- Pure crates (domain logic without IO) are prioritized
- Each pure crate is locked to JSON fixture contracts from maw-js
- Runtime IO and transports are added as secondary layers after core logic is proven
- The maw-rs CLI is introduced in Phase 3, keeping maw-js and maw-rs side-by-side during validation

### Phases

| Phase | Status | Focus |
|-------|--------|-------|
| Phase 1 | Complete | Pure logic crates with fixture contracts |
| Phase 2 | In Progress | Side-effecting transports (tmux, HTTP, Zenoh) |
| Phase 3 | Planned | CLI with clap; port high-value commands (ls, hey, peek) |

---

## Directory Structure

```
maw-rs/
├── Cargo.toml                      # Workspace root (13 member crates)
├── Cargo.lock                      # Locked dependency versions
├── README.md                        # Installation and phase status
├── AGENTS.md                        # Agent-related documentation
├── CLAUDE.md                        # Oracle identity and configuration
├── crates/                          # 13 workspace member crates
│   ├── maw-auth/                   # Federation auth & consent
│   ├── maw-cli/                    # Main CLI application (extensive core_impl)
│   ├── maw-discord/                # Discord integration & runtime
│   ├── maw-matcher/                # Target name resolution & matching
│   ├── maw-peer/                   # Peer source discovery
│   ├── maw-plugin-manifest/        # Plugin validation & WASM management
│   ├── maw-schedule/               # Schedule config & lifecycle
│   ├── maw-schedule-launchd/       # macOS launchd integration
│   ├── maw-schedule-runner/        # Scheduled job execution
│   ├── maw-tmux/                   # tmux client & parser
│   ├── maw-transport/              # Transport routing & HTTP federation
│   ├── maw-worktree/               # Worktree-to-window matching
│   └── maw-xdg/                    # XDG paths & config discovery
├── docs/                            # Installation guides
├── scripts/                         # Build & utility scripts
└── packages/                        # (TBD) Further packaging
```

---

## Cargo Workspace Organization

### Workspace Configuration

The workspace uses **Cargo 2 resolver** with unified:

- **Edition**: 2021 (across all crates)
- **License**: BUSL-1.1 (Business Source Use License)
- **Repository**: https://github.com/Soul-Brews-Studio/maw-rs

### Workspace Dependencies

**Core async runtime:**
```toml
tokio = { version = "1", features = ["rt-multi-thread", "macros", "sync", "time", "signal"] }
```

All crates inherit this and share it via workspace dependencies.

### Linting

- **Clippy**: `pedantic` warnings enabled at workspace level
- **Safety**: `unsafe_code = "forbid"` — no unsafe code permitted

---

## Core Abstractions

### 1. **Target Resolution & Matching** (maw-matcher)

**Purpose**: Portable name-based resolution for sessions, windows, and worktrees.

**Key Types:**

```rust
pub enum ResolveResult<T> {
    None { hints: Option<Vec<T>> },
    Exact { matched: T },
    Fuzzy { matched: T },
    Ambiguous { candidates: Vec<T> },
}

pub trait Named {
    fn name(&self) -> &str;
}
```

**Resolution Hierarchy:**

1. Case-insensitive exact match
2. Suffix segment match (`*-target`, preferred)
3. Prefix/middle segment match (`target-*` or `*-target-*`)
4. Substring hints only

**Fixtures:** Locked to maw-js `test/spec/matcher-resolve-target.fixtures.json`, `normalize-target.fixtures.json`

---

### 2. **Fleet & Session Model** (maw-matcher, maw-worktree, maw-peer)

**Key Abstractions:**

- **Window**: Index, name, active flag (maw-worktree)
- **Session**: Named container with ordered windows (maw-worktree)
- **FleetWindow/FleetWindowSessionLike**: Generic fleet-scoped window metadata

**Worktree-to-Window Resolution** (maw-worktree):

```rust
pub enum WorktreeWindowResolution {
    Bound { window: String },
    Ambiguous { query: String, candidates: Vec<String> },
    None,
}
```

Resolves git worktree names to tmux windows within parent oracle sessions, with smart numeric prefix stripping (e.g., `123-feature` → `feature`).

---

### 3. **Transport & Routing** (maw-transport)

**Purpose**: Abstract transport layer for federated multi-node communication.

**Core Routing Concerns:**

- **Transport Router**: Routes commands to tmux, HTTP federation, or Zenoh based on target
- **HTTP Federation**: Cross-network node discovery and command relay
- **Tmux HTTP Contract**: Native tmux client interactions via HTTP adapter

**Design Pattern**: Policy-driven routing without concrete implementation in Phase 1.

---

### 4. **Authentication & Authorization** (maw-auth)

**Responsibilities:**

- Federation authentication (request/verify/sign flows)
- Pair consent workflows (device pairing, PIN exchange)
- Federation health status and edge cases

**Key Implementation Files:**

- `auth_contract.rs`: Auth flow types and policies
- `pair_consent.rs`: Device pairing lifecycle
- `request_verify.rs`: Request verification & signing

**Fixtures:** Locked to maw-js federation auth tests and O6 decision tests.

---

### 5. **Scheduling & Execution** (maw-schedule, maw-schedule-launchd, maw-schedule-runner)

**Pure Schedule Model** (maw-schedule):

```rust
pub struct Schedule {
    pub id: String,
    pub command: String,
    pub cadence: String,
    pub max_fires_per_day: u32,
    pub exec: ExecMode,
    pub expected_output: Option<String>,
    pub token_name: String,
    // ...
}

pub enum RunStatus {
    Reserved, Spawned, Succeeded, Failed,
    CompletedWithoutDeliverable, Abandoned, CapHit,
}
```

**Lifecycle:**

1. **Reserve**: Check quota before spawning
2. **Mark Spawned**: Transition to active state
3. **Finalize**: Record exit code, deliverable presence, commit quota
4. **Abandon**: Mark stale runs (age > 2x cadence) as abandoned

**Execution Modes:**

- `ClaudeHeadless`: Default (AI-driven execution)
- `Shell`: Direct shell execution

**macOS Integration** (maw-schedule-launchd):

- plist generation and sync
- launchctl bootstrap/bootout management
- Atomic plist file writes
- Health checks (plist current + loaded)

---

### 6. **Plugin System** (maw-plugin-manifest)

**Purpose**: WASM plugin loading, validation, and execution.

**Key Concerns:**

- Plugin manifest validation (CLI & API)
- WASM artifact path resolution
- Extism WASM runtime integration (Phase 2+)

**Feature Flags:**

- `wasm-host`: Enables full WASM host support (maw-cli optional feature)

**Rust Plugin Authoring:**

```bash
maw plugin create --rust my-plugin
cd my-plugin
maw plugin build  # Builds wasm32-unknown-unknown + plugin.json
```

**Limitation** (Ship Tier): No JS/TS-to-WASM compilation; Rust + prebuilt JS artifacts only.

---

### 7. **Paths & Configuration** (maw-xdg)

**Purpose**: XDG-compatible path resolution and config discovery (port of maw-js `src/core/xdg.ts`, `src/core/paths.ts`).

**Key Functions:**

```rust
pub fn maw_config_dir() -> Result<PathBuf, String>
pub fn maw_data_dir() -> Result<PathBuf, String>
pub fn maw_cache_dir() -> Result<PathBuf, String>
pub fn maw_state_dir() -> Result<PathBuf, String>
pub fn discover_config_layers(...) -> Result<Vec<MawConfigLayerSource>, String>
pub fn load_merged_config(...) -> Result<MergedMawConfig, String>
pub fn deep_merge_config(...) -> MergedMawConfig
```

**Configuration Discovery:**

Layers (bottom-up):
1. XDG_CONFIG_HOME/maw/config.toml
2. XDG_DATA_HOME/maw/config.toml (legacy)
3. Instance-specific config overrides

**Instance Naming**: Validates alphanumeric + underscore patterns

---

## Module Dependencies

### Dependency Graph (Crates)

```
maw-cli (main application)
  ├─ maw-matcher         (target resolution)
  ├─ maw-transport       (routing)
  ├─ maw-tmux            (tmux runtime)
  ├─ maw-worktree        (worktree→window mapping)
  ├─ maw-auth            (auth workflows)
  ├─ maw-peer            (peer discovery)
  ├─ maw-schedule        (scheduling logic)
  ├─ maw-schedule-launchd (macOS integration)
  ├─ maw-schedule-runner (execution)
  ├─ maw-plugin-manifest (WASM plugins)
  ├─ maw-xdg             (paths & config)
  ├─ maw-discord         (Discord runtime)
  └─ [external crates]
     ├─ axum 0.7        (web framework)
     ├─ tokio           (async runtime)
     ├─ reqwest         (HTTP client)
     ├─ tokio-tungstenite (WebSocket)
     ├─ twilight-gateway (Discord gateway)
     ├─ twilight-model  (Discord types)
     ├─ portable-pty    (PTY abstraction)
     ├─ serde/serde_json (serialization)
     └─ others

Pure crates (no external I/O):
  ├─ maw-matcher
  ├─ maw-worktree
  ├─ maw-auth
  ├─ maw-schedule
  ├─ maw-xdg
  └─ maw-plugin-manifest
```

### Key External Dependencies

| Crate | Version | Purpose |
|-------|---------|---------|
| tokio | 1.x | Async runtime (multi-threaded, signals, timers) |
| axum | 0.7 | Web framework for API & WebSocket server |
| reqwest | 0.12 | HTTP client (rustls-tls backend) |
| serde/serde_json | 1.x | Serialization (JSON, TOML) |
| portable-pty | 0.9 | Cross-platform PTY abstraction |
| tokio-tungstenite | 0.24 | WebSocket support (rustls) |
| twilight-gateway | 0.17.1 | Discord gateway protocol |
| twilight-model | 0.17.1 | Discord API types |

**TLS Backend**: rustls exclusively (no OpenSSL/native-tls)

---

## Key Crates

### maw-cli (Main Application)

**Structure:**

```
src/
  ├─ lib.rs                 # Exports core_impl and serve_core
  ├─ core_impl/            # 188 modules (extensive command implementations)
  │  ├─ about.rs
  │  ├─ absorb.rs
  │  ├─ activity_core.rs
  │  ├─ agents.rs
  │  ├─ archive.rs
  │  ├─ artifacts.rs
  │  ├─ assign.rs
  │  ├─ atlas.rs
  │  ├─ atlas_render.rs
  │  ├─ attach.rs
  │  ├─ attach_ssh.rs
  │  ├─ attach_private_tests.rs
  │  ├─ audit.rs
  │  ├─ auth_plan_parse_request.rs
  │  ├─ auth_plan_parse_sign.rs
  │  ├─ auth_plan_run.rs
  │  ├─ auth.rs
  │  ├─ awaken.rs
  │  ├─ background_jobs.rs
  │  ├─ bind_feed_fuzzy_plan.rs
  │  ├─ buddy_workspace.rs
  │  ├─ capture.rs
  │  ├─ census.rs
  │  ├─ channel.rs          # Large (79KB) — core channel/window logic
  │  ├─ check_tools.rs
  │  ├─ cli_help.rs
  │  └─ [170+ more modules...]
  ├─ serve_core/           # API & WebSocket server
  │  └─ mod.rs (105KB, core server logic)
  ├─ tests/                # Integration tests (~30+ test suites)
  └─ build.rs              # Build script
```

**Notable Size:** maw-cli/src/core_impl is the largest module directory (~6MB), containing command implementations ported from maw-js.

**Compile Target:**

```toml
[[bin]]
name = "maw-rs"
path = "src/main.rs"     # (Does not currently exist in repo)
```

**Dependencies:**

- All local workspace crates (auth, transport, tmux, etc.)
- maw-auto-wake, maw-bring, maw-bind, maw-fuzzy, maw-feed (external git)
- maw-calver, maw-identity, maw-hub (external git)
- axum, reqwest, tokio, portable-pty, tokio-tungstenite
- Feature: `wasm-host` (optional, defaults to empty)

---

### maw-matcher (Pure Logic)

**Responsibility**: Portable target-name matching against session/window/peer names.

**Public API:**

```rust
// Normalize user input
pub fn normalize_target(raw: &str) -> String

// Resolve by exact, fuzzy, or ambiguous match
pub fn resolve_by_name<T: Named>(target: &str, items: &[T], opts: ResolveOptions) -> ResolveResult<T>
pub fn resolve_session_target(target: &str, sessions: &[S]) -> ResolveResult<S>
pub fn resolve_worktree_target(target: &str, windows: &[W]) -> ResolveResult<W>

// Numeric fleet prefix handling
pub fn resolve_numeric_fleet_stem_exact(stem: &str, items: &[T]) -> Option<T>
pub fn resolve_numeric_fleet_stem_prefix(stem: &str, items: &[T]) -> Option<T>

// Fleet-scoped resolution
pub fn resolve_fleet_window_session_target(target: &str, fleet: &FleetWindow) -> FleetWindowSessionLike
pub fn resolve_typed_target(target: &str, candidates: &[ResolveTypedCandidate]) -> ResolveTypedResult
```

**Fixtures:** Locked to maw-js `test/spec/*.fixtures.json`.

---

### maw-tmux (Tmux Client & Parser)

**Responsibility**: Tmux-specific runtime abstraction and parsing.

**Large Submodules (core_impl):**

- `client_session_window_parts/`: Session/window creation logic
- `client_pane_send_parts/`: Pane send-keys operations
- `live_state_parts/`: Live tmux state discovery
- `parsers_resolution_parts/`: Tmux output parsing
- `action_resolution_parts/`: Action resolution strategy

**Core Capabilities:**

- List/create/target sessions and windows
- Send keystrokes to panes
- Discover live tmux server state
- Parse tmux `list-sessions`, `list-windows`, `list-panes` output

---

### maw-transport (Abstract Transport)

**Responsibility**: Route commands to appropriate backend (tmux, HTTP, Zenoh).

**Core Abstractions:**

- `transport_router_parts/`: Policy routing
- `tmux_http_contract_parts/`: HTTP-to-tmux adapter contract
- `peer_http_parts/`: Peer (node-to-node) HTTP federation
- `host_http_parts/`: Host-side HTTP server logic

**Fixtures:** Locked to maw-js `transport-router.fixtures.json`.

---

### maw-discord (Discord Integration)

**Structure:**

```
src/
  ├─ access_core.rs            # Core access control
  ├─ access_read.rs            # Read-side Discord queries
  ├─ access_write.rs           # Write-side Discord commands
  ├─ bind.rs                   # Channel/role binding
  ├─ command_dispatch.rs       # Command router
  ├─ command_dispatch_tests.rs # Tests
  ├─ discord_runtime.rs        # Twilight gateway runtime
  └─ [additional modules]
```

**Dependencies:**

- twilight-gateway, twilight-model (Discord protocol)
- tokio, reqwest
- serde/serde_json

**Features:**

- Gateway event handling (Twilight v0.17.1)
- Command dispatch routing
- Channel/role binding for oracle access control

---

### maw-auth (Federation Auth & Consent)

**Core Implementations (core_impl):**

```rust
// Authentication contract
pub struct AuthRequest { /* protocol fields */ }
pub struct AuthSign { /* signature fields */ }
pub struct AuthVerify { /* verification fields */ }

// Pair consent workflow
pub struct PairConsentRequest { pub pin: String, /* ... */ }
pub struct PairConsentResponse { pub approved: bool, /* ... */ }

// Federation health
pub enum FederationHealth {
    Healthy,
    Degraded(String),
    Unreachable(String),
}
```

**Fixtures:** Locked to maw-js federation auth tests and O6 decision coverage.

---

### maw-schedule & maw-schedule-launchd (Scheduling)

**maw-schedule (Pure):**

- TOML config parsing
- Quota/cap management
- Run lifecycle state machine (Reserve → Spawned → Succeeded/Failed/Abandoned)

**maw-schedule-launchd (macOS-specific):**

- Plist generation from schedule config
- launchctl integration (bootstrap/bootout)
- Atomic file writes
- Health check (plist current + loaded)

**Supported Execution Modes:**

- ClaudeHeadless: AI-driven (default)
- Shell: Direct shell invocation

---

## Entry Points

### Primary CLI Entry Point

**Status**: Declared in Cargo.toml but `src/main.rs` does not yet exist in the codebase.

```toml
[[bin]]
name = "maw-rs"
path = "src/main.rs"
```

**Expected (Phase 3):**

The CLI will be built using `clap` and prioritize high-value commands:

- `maw ls` — List sessions/windows
- `maw hey` — Send commands
- `maw peek` — Inspect state
- Target resolution helpers

### Library Entry Points

Each crate exports its public API via `src/lib.rs`:

```rust
// maw-cli/src/lib.rs
mod core_impl;
pub mod serve_core;
pub use core_impl::*;

// maw-matcher/src/lib.rs
pub use fleet::*;
pub use normalize::*;
pub use numeric::*;
pub use resolver::*;
pub use typed_resolver::*;

// maw-xdg/src/lib.rs
pub use config::*;
pub use paths::*;
pub use types::*;
```

### Integration Tests

Located in `crates/maw-cli/tests/`:

- ~30 test modules covering CLI edge cases
- Test names indicate scope: `discover_route_worktree_cli_edges`, `native_ls_flags`, `transport_cli`, `calver_cli`, etc.
- Each validates fixture contracts or maw-js compatibility

---

## Design Patterns

### 1. **Fixture-Driven Development**

Every pure crate is locked to JSON fixtures from maw-js:

```
maw-matcher        → matcher-resolve-target.fixtures.json
maw-worktree       → worktree-window-match.fixtures.json
maw-transport      → transport-router.fixtures.json
maw-auth           → federation auth tests
maw-schedule       → schedule parsing & quota tests
```

**Workflow:**

1. Copy fixture from maw-js `test/spec/`
2. Port logic until all fixtures pass
3. Lock behavior before runtime IO is added

### 2. **core_impl + Facade Pattern**

Crates follow a consistent structure:

```
src/
  ├─ lib.rs              # Public facade (re-exports)
  └─ core_impl/          # Implementation detail (mod.rs + submodules)
```

**Example: maw-cli**

```rust
// src/lib.rs
mod core_impl;
pub mod serve_core;
pub use core_impl::*;

// src/core_impl/mod.rs
pub mod about;
pub mod absorb;
pub mod activity;
// ...
pub use about::*;
pub use absorb::*;
// ...
```

**Benefit**: Internal refactoring (submodule organization) doesn't break public API.

### 3. **Async-First with Tokio**

All async work uses:

- **Tokio multi-threaded runtime** (default feature: `rt-multi-thread`)
- **Structured concurrency**: Spawned tasks properly awaited or canceled
- **Signal handling**: SIGTERM/SIGINT via tokio::signal

### 4. **Pure vs. Effectful Boundary**

**Pure Crates** (no external IO):

- maw-matcher
- maw-worktree
- maw-schedule
- maw-auth (logic only; IO via HTTP adapter)
- maw-xdg (path resolution only; no filesystem access except config load)
- maw-plugin-manifest

**Effectful Crates** (IO, runtime):

- maw-cli (commands, server, Discord)
- maw-tmux (process spawning, PTY)
- maw-transport (HTTP, networking)
- maw-schedule-launchd (launchctl process, plist write)
- maw-schedule-runner (job execution)
- maw-discord (network, gateway)

### 5. **Serialization-First Design**

Heavy use of serde for:

- Configuration (TOML, JSON)
- API contracts (HTTP, WebSocket)
- Fixture testing
- WASM plugin manifests

### 6. **Traits for Extensibility**

Key generic traits allow custom implementations:

```rust
// Named — anything with a .name() can be resolved
pub trait Named { fn name(&self) -> &str; }

// LaunchctlRunner — mock-friendly launchctl abstraction
pub trait LaunchctlRunner { fn run(&mut self, args: &[String]) -> Result<LaunchctlOutput, String>; }

// (More in effectful layers: transport handlers, auth providers, etc.)
```

### 7. **Error Handling Strategy**

- Pure crates: Return `Result<T, ParseError>` or `Result<T, String>`
- Effectful crates: Detailed error types with context
- No panics in public APIs (library-safe)

### 8. **Feature Flags for Optional Functionality**

```toml
[features]
default = []
wasm-host = ["maw-plugin-manifest/wasm-host"]
```

Allows builds to omit heavy dependencies (e.g., Extism) in constrained environments.

---

## Build & Test Gates

### Local Validation

```bash
cargo test --workspace          # Unit + integration tests
cargo clippy --workspace --all-targets -- -D warnings   # Linting
```

### Workspace Lints

- **Clippy pedantic**: Warnings enabled
- **Unsafe code**: Forbid (zero unsafe blocks)

### CI Pipeline

GitHub Actions workflow (`.github/workflows/ci.yml`):

- Runs on push to main, PRs, and releases
- Publishes prebuilt binaries for macOS ARM64 and Linux x86_64

---

## Deployment & Distribution

### Prebuilt Binaries

**Platforms:**

- `maw-rs-macos-arm64` — macOS Apple Silicon
- `maw-rs-linux-x86_64-musl` — Linux x86_64 (static)

**Installation:**

```bash
# Stable (via Homebrew)
brew install soul-brews-studio/maw/maw

# Bleeding-edge installer
curl -fsSL https://raw.githubusercontent.com/Soul-Brews-Studio/maw-rs/alpha/install.sh | sh

# Pinned version (CalVer tag, e.g., v26.7.16)
curl -fsSL https://github.com/Soul-Brews-Studio/maw-rs/releases/download/v26.7.16/install.sh | MAW_VERSION=v26.7.16 sh
```

**Installer Features:**

- SHA-256 signature verification
- Backup of existing binary
- Default install to `~/.local/bin/maw`
- Gatekeeper bypass (`xattr -d com.apple.quarantine`) on macOS

---

## Phase Roadmap

### Phase 1 (Complete)

- Cargo workspace scaffolded
- 13 pure crates with fixture contracts
- Tests passing against maw-js JSON fixtures

### Phase 2 (In Progress)

- Side-effecting transports:
  - tmux CLI via process spawning
  - HTTP federation via reqwest
  - Zenoh via stylos/themion ecosystem
- Runtime adapters around pure crates
- Parallel maw-js/maw-rs operation

### Phase 3 (Planned)

- maw-rs CLI with clap
- Fast-path commands: `ls`, `hey`, `peek`, target resolution
- Fixture validation and golden output comparison
- Replace maw-js entrypoint once parity proven

---

## Notable Implementation Details

### Target Name Normalization

Handles Unicode, case-insensitive matching, segment-based resolution:

```rust
pub fn normalize_target(raw: &str) -> String {
    raw.trim().to_lowercase()
}

// Resolution precedence:
// 1. Exact match (case-insensitive)
// 2. Suffix: "-target" (preferred)
// 3. Prefix/middle: "target-*" or "*-target-*"
// 4. Substring hints only
```

### Numeric Fleet Session Handling

Fleet sessions are named `NN-*` (numeric prefix). Resolution can exclude these:

```rust
pub fn resolve_by_name<T>(
    target: &str,
    items: &[T],
    options: ResolveOptions,
) -> ResolveResult<T>
where
    T: Named + Clone,
{
    // If ResolveOptions.fleet_sessions = true, skip "NN-*" prefixed items
}
```

### Worktree Prefix Stripping

Worktree names may have numeric task prefixes (`123-feature`), which are stripped:

```rust
fn strip_numeric_prefix(name: &str) -> &str {
    let Some((prefix, rest)) = name.split_once('-') else {
        return name;
    };
    if !prefix.is_empty() && prefix.bytes().all(|byte| byte.is_ascii_digit()) {
        rest
    } else {
        name
    }
}
```

### Schedule Quota Model

Avoids burst execution with daily caps:

```rust
pub fn reserve(request: ReserveRequest) -> OutcomeRecord {
    let cap_hit = !request.forced
        && request.committed.saturating_add(request.active_reservations)
            >= request.cap;
    // Return CapHit or Reserved
}
```

### macOS Launchd Plist Sync

Atomic file writes + idempotent launchctl management:

```rust
pub fn sync_job<R: LaunchctlRunner>(
    job: &DesiredJob,
    domain: &str,
    mode: SyncMode,
    runner: &mut R,
) -> Result<SyncResult, String> {
    // Idempotent: Check → Write (if needed) → Bootstrap (if needed)
    // No double-bootstrap on healthy state
}
```

---

## Security Considerations

### No Unsafe Code

- Rust safe-code guarantee upheld
- `unsafe_code = "forbid"` enforced at workspace level
- All external crates reviewed for safe Rust

### TLS Backend

- **rustls exclusively** (no OpenSSL dependency)
- Applied to all HTTP transports (reqwest, tokio-tungstenite, twilight-gateway)

### License

- **BUSL-1.1** (Business Source Use License)
- Commercial use requires subscription; perpetual conversion to open-source after 4 years

---

## Summary

**maw-rs** is a carefully staged Rust port of maw-js, prioritizing:

1. **Pure, testable logic** first (fixture-locked)
2. **Incremental IO integration** (tmux, HTTP, scheduling)
3. **Federated agent coordination** across nodes and terminals
4. **Plugin extensibility** via WASM
5. **Production readiness** via Cargo CI, prebuilt binaries, and signature verification

The architecture separates concerns cleanly: pure domain logic lives in isolated crates, effectful layers wrap them, and the CLI acts as the user-facing facade. By Phases 2–3, maw-rs will be feature-complete and ready to replace maw-js as the primary agent fleet orchestration tool.

---

**Document generated**: 2026-07-28  
**Explored repository**: /Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-rs
