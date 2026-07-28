# maw-plugins Architecture

**Repository**: `Soul-Brews-Studio/maw-plugins`  
**Purpose**: Installable CLI, API, and hook plugins for maw-js (the TypeScript predecessor to maw-rs)  
**License**: BUSL-1.1 (converting to Apache-2.0 on the Change Date)  

---

## Overview

maw-plugins is a **plugin-package monorepo** that extracts verbs (features) from the maw core into independently installable, ship-tier WASM artifacts pinned by SHA256. Plugins are one of two origins:

1. **Extracted from maw-rs** (repo split phase 1, 2026-07-15): Fleet plugins that manage Discord, teams, task queues, etc.
2. **New plugins**: Verbs that extend maw-js capabilities post-extraction.

All plugins implement the **Extism Plugin Development Kit (PDK)** interface and are registered via a deterministic **registry.json** file that serves as the default resolution root for the mawx package manager.

---

## Directory Structure & Organization

```
maw-plugins/
├── packages/                    # 26 plugin packages (weights + names)
│   ├── 20-avengers/            # Weight 20 (tools tier)
│   ├── 20-broadcast/
│   ├── 20-costs/               # Rust → WASM; committed plugin.wasm
│   ├── 20-demo/
│   ├── 20-dream/
│   ├── 20-follow/
│   ├── 20-hub/
│   ├── 20-layout/
│   ├── 20-mega/
│   ├── 20-pulse/
│   ├── 20-soul-sync/
│   ├── 20-stream/
│   ├── 20-tile/
│   ├── 50-artifact-manager/    # Weight 50 (features tier)
│   ├── 50-incubate/
│   ├── atlas/                  # Fleet plugin (no prefix); AssemblyScript → WASM
│   ├── cross-team-queue/
│   ├── hermes/
│   ├── maw-menubar/            # Fleet plugin + native Swift helper (bin/maw-menubar)
│   ├── p2p-share/
│   ├── share/                  # bun-dev tier; TypeScript only, no WASM
│   ├── squad/
│   └── team/
├── docs/
│   ├── fleet-plugins.md        # Dev-to-ship WASM ladder & pin lifecycle
│   └── registry.md             # registry.json structure & mawx resolution
├── scripts/
│   ├── setup.sh                # Links node_modules/maw-js for local development
│   ├── test-per-package.sh     # Runs bun tests in isolation per package
│   └── gen-registry.ts         # Regenerates registry.json deterministically
├── .github/workflows/
│   └── ci.yml                  # Build, test, pin-integrity, registry-freshness gates
├── package.json                # Root workspace definition
├── registry.json               # Generated index of all plugins (mawx AUTO-TRUST root)
└── README.md                   # High-level overview & install instructions
```

### Weight System

Plugins are prefixed with a **numeric weight** (e.g., `20-costs`, `50-incubate`) that determines **execution order**. Lower weights fire first.

- **Weight 20**: Tools tier — utilities, read-only operations, cost analysis
- **Weight 50**: Features tier — higher-level composed features

Unprefixed plugins (fleet plugins imported from maw-rs) run at their declared `weight` in plugin.json.

---

## Workspaces & Package Management

**Root package.json**:
```json
{
  "name": "maw-plugins",
  "private": true,
  "version": "0.1.0",
  "workspaces": ["packages/*"],
  "scripts": {
    "setup": "bash scripts/setup.sh",
    "test": "bash scripts/test-per-package.sh",
    "registry": "bun run scripts/gen-registry.ts",
    "registry:check": "bun run scripts/gen-registry.ts --check"
  }
}
```

The monorepo uses **npm/bun workspaces** to manage 26 interdependent packages. Each plugin package is a standalone, versioned unit with its own `package.json`.

### Setup

The `setup.sh` script creates a symlink `node_modules/maw-js` so plugins can resolve:
```typescript
import { ... } from "maw-js/sdk"
```

This resolves in the following order (first hit wins):
1. `$MAW_JS_PATH` env var
2. Sibling checkout: `../maw-js` (same parent as maw-plugins)
3. Bun global install: `~/.bun/install/global/node_modules/maw-js`
4. `bun pm ls --global` resolved path

---

## Entry Points & Plugin Registration

### Manifest Format (`plugin.json`)

Every plugin declares a **plugin.json** manifest that defines its interface to the maw-rs host.

**Example** (Rust/WASM plugin — 20-costs):
```json
{
  "artifact": {
    "path": "./plugin.wasm",
    "sha256": "sha256:1947d3ed3f1a1bc5b192d918c6cc02eddd46196cf4b3455b2cbfabd51f1d2f58"
  },
  "author": "Soul-Brews-Studio",
  "capabilities": ["fs:read:claude-projects"],
  "cli": {
    "command": "costs",
    "help": "maw costs [--daily [N]|--days N] [--json]"
  },
  "description": "Show token usage and estimated cost breakdown per agent.",
  "entry": {
    "export": "handle",
    "kind": "wasm",
    "path": "plugin.wasm"
  },
  "name": "costs",
  "schemaVersion": 1,
  "sdk": "^1.0.0",
  "target": "wasm",
  "version": "1.0.0",
  "weight": 20
}
```

**Example** (AssemblyScript fleet plugin — atlas):
```json
{
  "artifact": {
    "path": "./plugin.wasm",
    "sha256": "sha256:8bce593d9727d5fcbb74ef7eecf06798b82e1208031b6d7d44a9c6463c7e2310"
  },
  "author": "atlas-oracle",
  "capabilities": ["net:fetch:discord-rest", "secret:use:atlas-bot-token"],
  "cli": {
    "command": "atlas",
    "aliases": ["at"],
    "help": "maw atlas <whoami|ls|read|threads>"
  },
  "description": "Discord state core: identity, guilds, channels, messages, threads.",
  "entry": {
    "export": "handle",
    "kind": "wasm",
    "path": "plugin.wasm"
  },
  "endpoints": {
    "discord-rest": {
      "baseUrl": "https://discord.com/api/v10",
      "methods": ["GET"],
      "paths": ["/users/@me", "/users/@me/guilds", "/guilds/*/channels", ...],
      "auth": {
        "kind": "discord-bot",
        "secret": "atlas-bot-token"
      }
    }
  },
  "name": "atlas",
  "schemaVersion": 1,
  "sdk": "^1.0.0",
  "secrets": {
    "atlas-bot-token": {
      "env": "DISCORD_BOT_TOKEN",
      "pass": "discord/atlas-oracle-token"
    }
  },
  "target": "wasm",
  "weight": 50
}
```

**Example** (Bun-dev tier — share):
```json
{
  "name": "share",
  "version": "0.1.0",
  "sdk": "^1.0.0",
  "runtime": "bun-dev",
  "target": "js",
  "entry": "src/plugin.ts",
  "description": "Minimal maw terminal-share bridge for sshx-fork sessions.",
  "author": "Soul Brews Studio",
  "license": "MIT",
  "weight": 0,
  "schemaVersion": 1,
  "cli": {
    "command": "share",
    "help": "maw share <start|ls|url|stop> [--name <label>]"
  }
}
```

### Manifest Fields

| Field | Purpose |
|-------|---------|
| `name` | Short identifier; must match `cli.command` in registry.json |
| `schemaVersion` | Manifest format version (currently 1) |
| `target` | `"wasm"` for WASM plugins, `"js"` for bun-dev tier |
| `runtime` | `"bun-dev"` for TypeScript plugins not shipped as WASM |
| `entry` | Entry point: `{ kind: "wasm", path, export }` or TypeScript file path |
| `artifact` | SHA256 pin of committed WASM: `{ path, sha256: "sha256:<hex>" }` |
| `cli` | CLI surface: `{ command, aliases?, help }` |
| `capabilities` | Permissions granted to the plugin (host functions it can call) |
| `endpoints` | HTTP API endpoints the plugin can call (with auth, query params) |
| `secrets` | Environment variable mappings and secret stores |
| `weight` | Execution order (lower fires first) |
| `version` | SemVer version string |
| `author`, `description`, `license` | Metadata |

### Dev vs. Ship Tiers

**Fleet plugins** (imported from maw-rs) support a **dual-manifest pattern**:

- **plugin.json**: Ship-tier manifest; points to committed WASM (`plugin.wasm`), pinned by SHA256 in `artifact.sha256`
- **plugin.source.json**: Dev-tier manifest; points to TypeScript source (`src/plugin.ts`), includes dev-only metadata

The CI pin-integrity gate checks `plugin.json` first, falling back to `plugin.source.json` for dev-tier-active plugins.

---

## Core Abstractions & Plugin Types

### 1. Rust/WASM Plugins (Direct Extism-PDK)

**Technologies**: Rust 2021 edition, `extism-pdk = "1.4"`, `wasm32-unknown-unknown` target

**Structure**:
```
20-costs/
├── Cargo.toml              # cdylib crate with extism-pdk dependency
├── Cargo.lock
├── src/lib.rs              # Rust plugin implementation
├── plugin.wasm             # Committed ship artifact (committed bytes, immutable)
└── plugin.json             # Manifest with WASM pin
```

**Cargo.toml Pattern**:
```toml
[package]
name = "maw_<verb>_plugin"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
extism-pdk = "1.4"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

**Entry Point** (`src/lib.rs`):
```rust
use extism_pdk::*;

#[link(wasm_import_module = "extism:host/user")]
extern "C" {
    #[link_name = "maw.fs.read"]
    fn maw_fs_read(input: u64) -> u64;
    #[link_name = "maw.paths.get"]
    fn maw_paths_get(input: u64) -> u64;
}

#[plugin_fn]
pub unsafe fn handle(input: String) -> FnResult<String> {
    // Parse JSON input, call host functions, return JSON output
    Ok(result(true, "output", ""))
}
```

**Host Functions Provided by maw-rs**:
- `maw.fs.read`, `maw.fs.list`: File system access (sandboxed)
- `maw.paths.get`: Resolve named paths (e.g., "claude-projects")
- `maw.net.fetch`: HTTP requests to declared endpoints
- `maw.send`, `maw.identity`: Message passing and plugin identity
- `tmux.raw:*`, `tmux.send`, `tmux.read`: Terminal multiplexing

**Build Process**:
```bash
rustup target add wasm32-unknown-unknown
cargo build --release --target wasm32-unknown-unknown
cp target/wasm32-unknown-unknown/release/maw_<verb>_plugin.wasm plugin.wasm
shasum -a 256 plugin.wasm   # Update artifact.sha256 in plugin.json
```

**Testing**: No host-target `cargo test` (WASM imports don't exist in native runtime). Behavior is tested host-side by maw-rs's parity/invoke harness against the committed, pinned plugin.wasm.

### 2. AssemblyScript/TypeScript Fleet Plugins (Extism WASM)

**Technologies**: AssemblyScript (preferred) or TypeScript compiled to WASM via `@extism/as-pdk`

**Structure**:
```
atlas/
├── src/plugin.ts           # AssemblyScript source (compiles to WASM)
├── plugin.wasm             # Committed ship artifact
├── plugin.json             # Ship-tier manifest
├── plugin.source.json      # Dev-tier manifest (entry: "src/plugin.ts")
└── NOTES.md                # Implementation notes
```

**plugin.source.json Example**:
```json
{
  "name": "atlas",
  "entry": "src/plugin.ts",
  "target": "wasm",
  "cli": { "command": "atlas", ... },
  "capabilities": ["net:fetch:discord-rest", "secret:use:atlas-bot-token"],
  "endpoints": { "discord-rest": { ... } }
}
```

**Entry Point** (src/plugin.ts):
```typescript
import { Host, Memory } from "@extism/as-pdk";

@external("extism:host/user", "maw.net.fetch")
declare function mawNetFetch(input: u64): u64;

export function handle(): i32 {
  const args = extractArgs(Host.inputString());
  const cmd = args.length > 0 ? args[0] : "";
  
  if (cmd == "whoami") return done(whoami());
  if (cmd == "ls") return done(listState());
  // ...
  
  return fail("usage: maw atlas <whoami|ls|read|threads>");
}
```

**Key Differences from Rust**:
- Compiles to smaller WASM (AssemblyScript → smaller bytecode than Rust)
- Simpler memory management via `@extism/as-pdk` helper library
- Dev tier runs directly on TypeScript (bun-dev); ship tier pre-compiled to WASM
- Used exclusively for fleet plugins extracted from maw-rs

### 3. Bun-Dev Tier (TypeScript Only, No WASM)

**Technologies**: TypeScript, bun-dev runtime

**Structure**:
```
share/
├── src/plugin.ts           # TypeScript entry (no WASM compilation)
├── src/plugin.test.ts      # Bun tests
└── plugin.json             # Manifest with runtime: "bun-dev"
```

**plugin.json**:
```json
{
  "name": "share",
  "runtime": "bun-dev",
  "target": "js",
  "entry": "src/plugin.ts",
  "cli": { "command": "share", ... }
}
```

**Use Case**: Lightweight plugins that don't need WASM (local state management, simple CLI bridges). Currently only `share` and `p2p-share` use this tier.

### 4. Native Bundles (maw-menubar)

**Technologies**: Swift (native macOS), universal binary (arm64 + x86_64)

**Structure**:
```
maw-menubar/
├── src/plugin.ts           # TypeScript wrapper
├── plugin.wasm             # AssemblyScript-compiled bridge
├── bin/maw-menubar         # Committed universal Swift binary
├── plugin.json             # Manifest with bundledArtifacts
└── scripts/build-universal.sh
```

**plugin.json** (bundledArtifacts):
```json
{
  "bundledArtifacts": [
    {
      "path": "bin/maw-menubar",
      "sha256": "sha256:<hex>"
    }
  ]
}
```

**CI Validation** (maw-menubar-native job):
- Verifies SHA256 pin of committed universal helper
- Confirms arm64 + x86_64 architecture slices present
- Validates codesignature
- Can rebuild from Swift source if needed

---

## Dependencies & External Contracts

### Root Dependencies (via package.json)

The root `package.json` declares:
- `workspaces: ["packages/*"]` — Enables npm/bun workspace hoisting
- No direct dependencies (all are workspace-internal or transitive)

### Per-Package Dependencies

**Rust/WASM Plugins**:
- `extism-pdk = "1.4"` — Provides `#[plugin_fn]` macro, memory management, host calls
- `serde`, `serde_json` — JSON serialization (manifest, host messages)
- `base64` — For encoding/decoding (e.g., binary file reads in costs plugin)

**Fleet Plugins** (TypeScript/AssemblyScript):
- `@extism/as-pdk` — AssemblyScript runtime for WASM, host call wrappers
- Dev only: `bun` — Built-in to the ecosystem (no explicit dep)

**Peer Dependency**:
- `maw-js` — Linked locally via `scripts/setup.sh`, resolved from sibling checkout or global install

### Plugin Interface Contract

All plugins implement the **Extism WASM interface**:

**Input**: JSON string via `Host.inputString()` (or host memory in Rust)
```json
{
  "args": ["subcommand", "arg1", "arg2"],
  "today": "2026-07-28",
  "home": "/Users/h_wa"
}
```

**Output**: JSON string via `Host.outputString()`
```json
{
  "ok": true,
  "output": "formatted text output"
}
```
or
```json
{
  "ok": false,
  "error": "error message"
}
```

**Host Function Calling Pattern** (Rust):
```rust
#[link(wasm_import_module = "extism:host/user")]
extern "C" {
    #[link_name = "maw.fs.read"]
    fn maw_fs_read(input: u64) -> u64;
}

fn call_host(f: unsafe extern "C" fn(u64) -> u64, input: String) -> String {
    let mem = Memory::from_bytes(input.as_bytes())?;
    let offset = mem.offset();
    let out = unsafe { f(offset) };
    mem.free();
    Memory::find(out).map_or_else(String::new, |m| {
        let bytes = m.to_vec();
        m.free();
        String::from_utf8_lossy(&bytes).into_owned()
    })
}
```

**Capability System**: Plugins declare capabilities they need; the host sandboxes access:
```
"capabilities": [
  "fs:read:claude-projects",    # Read-only access to a named path
  "tmux:send",                   # Send commands to tmux
  "net:fetch:discord-rest",      # Make HTTP calls to a declared endpoint
  "secret:use:atlas-bot-token"   # Access to a secret via key
]
```

---

## Build, Test & Release Pipeline

### Build Process

**CI Job: build-and-verify** (Ubuntu latest)

1. **Install Rust WASM target**:
   ```bash
   rustup target add wasm32-unknown-unknown
   ```

2. **Build all Rust → WASM plugins**:
   ```bash
   for dir in packages/*/; do
     [ -f "$dir/Cargo.toml" ] || continue
     cargo build --release --target wasm32-unknown-unknown --manifest-path "$dir/Cargo.toml"
   done
   ```

3. **Pin Integrity Gate** (critical):
   - For each `plugin.wasm`, verify its SHA256 matches the manifest pin
   - Checks `plugin.json` first, falls back to `plugin.source.json` for dev-tier
   - **Does NOT compare rebuilt WASM against pin** (rebuild determinism is unproven)
   - This ensures committed artifacts are immutable

### Testing

**Per-Package Isolation** (bun):
```bash
for d in packages/*/; do
  [ -f "$d"*.test.ts ] || continue
  bun test "$d"
done
```

Tests are run in isolation per package to prevent mock state leakage across files.

**Fleet Plugin Tests**:
- `src/*.test.ts` files run via `bun test`
- `bun install` is best-effort (may fail on unpublished workspace deps)

**Rust Plugin Tests**:
- No host-target `cargo test` (WASM imports unavailable natively)
- Host-side testing via maw-rs's parity/invoke harness

### Registry Generation

**gen-registry.ts**: Regenerates `registry.json` deterministically from `packages/*`

**Key Logic**:
1. For each package dir, read manifest (`plugin.json` → `plugin.source.json`)
2. Extract: `name`, `version`, `capabilities`, SHA256 pin
3. Get last commit touching the package: `git log -1 -- packages/<dir>`
4. Build flat map keyed by `cli.command` verb

**Entry Shape**:
```json
{
  "<verb>": {
    "commit": "abf2272d47e98962d252f092172d436db8fb5631",
    "sha256": "sha256:...",
    "path": "packages/<dir>",
    "version": "1.0.0",
    "capabilities": [...]
  }
}
```

**CI Gate: registry-freshness**:
- Regenerates registry.json and compares to committed version
- Fails if they differ (prevents stale pins)
- Requires full git history (`fetch-depth: 0`)

### Install Process

```bash
# From local clone
git clone https://github.com/Soul-Brews-Studio/maw-plugins
maw plugin install --path maw-plugins/packages/NN-verb

# From remote (currently blocked by maw-rs#521)
maw plugin install Soul-Brews-Studio/maw-plugins/packages/NN-verb --sha256 <pin>
```

---

## Security & Pinning Model

### SHA256 Pinning

Every WASM plugin includes a **committed artifact** (plugin.wasm) whose hash is pinned in the manifest:

```json
{
  "artifact": {
    "path": "./plugin.wasm",
    "sha256": "sha256:1947d3ed3f1a1bc5b192d918c6cc02eddd46196cf4b3455b2cbfabd51f1d2f58"
  }
}
```

**Why**:
1. Immutable audit trail: the commit ID + package path + SHA256 uniquely identifies a plugin version
2. Rebuild determinism is unproven; CI enforces committed artifact integrity, not rebuild equivalence
3. registry.json is an AUTO-TRUST resolution root (mawx WI-9): entries run without first-run prompts

**Pin Lifecycle**:
1. Developer builds WASM locally: `cargo build --release --target wasm32-unknown-unknown`
2. Copies to plugin.wasm: `cp target/.../release/maw_<verb>_plugin.wasm plugin.wasm`
3. Computes hash: `shasum -a 256 plugin.wasm`
4. Updates `artifact.sha256` in plugin.json
5. Commits both files together
6. CI verifies pin matches committed artifact

### Capability Sandboxing

Plugins request capabilities in their manifest; the host grants sandboxed access:

**fs:read:*** — File system read access to a named path:
- `fs:read:claude-projects` → ~/.claude/projects
- `fs:read:repos` → known repository directories
- `fs:read:psi` → user identity & secrets store

**fs:write:*** — File system write access (same semantics)

**tmux:*** — Terminal multiplexer control:
- `tmux:read` — Query pane/window/session state
- `tmux:send` — Send keystrokes
- `tmux:raw:kill-pane`, etc. — Direct tmux commands

**net:fetch:*** — HTTP client limited to declared endpoints:
- `net:fetch:discord-rest` → can call `/users/@me`, `/guilds/*/channels`, etc.
- Defines baseUrl, HTTP methods, path patterns
- Auth kind (e.g., Discord bot token)

**secret:use:*** — Access named secrets via the host's secret store

**proc:exec:*** — Execute system commands (e.g., `git`, `date`, `base64`)

**cli:run:*** — Invoke other CLI plugins (e.g., `incubate` can call `bud`)

---

## Git Workflow & Commit Anchoring

The **registry.json** pins each plugin entry to the **last commit that touched its directory**:

```typescript
function lastCommitTouching(relPath: string): string {
  const out = execFileSync("git", ["-C", repoRoot, "log", "-1", "--format=%H", "--", relPath]).trim();
  return out;
}
```

**Why**:
- The commit ID is guaranteed to exist on GitHub at generation time
- Immutable raw URLs: `raw.githubusercontent.com/<o>/<r>/<commit>/packages/<dir>/plugin.wasm` are stable
- Using HEAD instead would reference a commit that doesn't yet exist

**Implication**: `registry.json` must be regenerated whenever any package changes, and the commit ID is only valid after the regeneration commit itself is pushed.

---

## Workflow for Authoring a New Plugin

### Step 1: Create Package Directory

```bash
mkdir packages/NN-<verb>
cd packages/NN-<verb>
```

Choose an appropriate weight:
- **20** for tools (read-only, utilities)
- **50** for features (composed, higher-level)

### Step 2: Rust Plugin (Cargo-based)

**Cargo.toml**:
```toml
[package]
name = "maw_<verb>_plugin"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
extism-pdk = "1.4"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

**src/lib.rs**:
```rust
use extism_pdk::*;

#[plugin_fn]
pub unsafe fn handle(input: String) -> FnResult<String> {
    // Parse input JSON, call host functions, return result
    Ok(json!({"ok": true, "output": "..."}).to_string())
}
```

### Step 3: Build & Pin

```bash
cargo build --release --target wasm32-unknown-unknown
cp target/wasm32-unknown-unknown/release/maw_<verb>_plugin.wasm plugin.wasm
shasum -a 256 plugin.wasm  # → copy to plugin.json artifact.sha256
```

### Step 4: plugin.json Manifest

```json
{
  "name": "<verb>",
  "version": "1.0.0",
  "schemaVersion": 1,
  "target": "wasm",
  "entry": {
    "kind": "wasm",
    "path": "plugin.wasm",
    "export": "handle"
  },
  "artifact": {
    "path": "./plugin.wasm",
    "sha256": "sha256:<hex from step 3>"
  },
  "cli": {
    "command": "<verb>",
    "help": "maw <verb> <subcommand> [options]"
  },
  "capabilities": ["fs:read:<path>", "tmux:read"],
  "description": "...",
  "author": "Soul-Brews-Studio",
  "weight": 20
}
```

### Step 5: Tests (Optional but Recommended)

For Rust plugins, add `src/lib.rs` tests at the bottom of the file:
```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_parse_args() {
        // Test logic...
    }
}
```

### Step 6: Commit & Push

```bash
git add packages/NN-<verb>/Cargo.toml packages/NN-<verb>/Cargo.lock packages/NN-<verb>/src/ packages/NN-<verb>/plugin.wasm packages/NN-<verb>/plugin.json
git commit -m "feat: add NN-<verb> plugin"
npm run registry  # Regenerates registry.json
git add registry.json
git commit -m "chore: registry.json update"
git push
```

CI will:
1. Build your plugin
2. Verify pin integrity (committed plugin.wasm ↔ manifest SHA256)
3. Regenerate registry.json and ensure it matches
4. Run any tests

---

## Key Files Reference

| File | Purpose |
|------|---------|
| `package.json` | Root workspace definition |
| `registry.json` | Generated index; default mawx resolution root (AUTO-TRUST) |
| `scripts/setup.sh` | Links node_modules/maw-js for dev |
| `scripts/gen-registry.ts` | Deterministically regenerates registry.json |
| `scripts/test-per-package.sh` | Runs bun tests per-package in isolation |
| `.github/workflows/ci.yml` | Build, test, pin-integrity, registry-freshness gates |
| `docs/fleet-plugins.md` | Detailed dev-to-ship WASM ladder & pin lifecycle |
| `docs/registry.md` | registry.json structure & mawx resolution semantics |
| `README.md` | High-level overview & install instructions |

---

## Notable Implementation Patterns

### 1. Host Function Wrappers (Rust)

The costs plugin demonstrates a clean **host_call abstraction**:
```rust
fn host_call(f: unsafe extern "C" fn(u64) -> u64, input: String) -> String {
    let mem = Memory::from_bytes(input.as_bytes())?;
    let offset = mem.offset();
    let out = unsafe { f(offset) };
    mem.free();
    Memory::find(out).map_or_else(String::new, |m| { ... })
}
```

Encapsulates memory allocation, calling the host, and freeing in one place.

### 2. Pagination & Streaming (Rust/fs)

The costs plugin reads large JSONL files with `nextOffset` pagination:
```rust
fn read_bytes(path: &str, offset: u64) -> Option<(Vec<u8>, Option<u64>)> {
    let r = host_call(
        maw_fs_read,
        json!({"path": path, "offset": offset, "maxBytes": 10485760u64}).to_string(),
    );
    // Parse response, return (bytes, nextOffset)
}
```

Handles files larger than 10 MB in chunks.

### 3. Manual JSON Parsing (AssemblyScript)

The atlas plugin uses hand-rolled **field extraction** to minimize WASM size:
```typescript
function str(s: string, key: string): string {
  const at = s.indexOf("\"" + key + "\"");
  const q = s.indexOf("\"", s.indexOf(":", at) + 1);
  return q < 0 ? "" : read(s, q).value;
}
```

This avoids shipping a large JSON library in the WASM binary.

### 4. CLI Argument Parsing

Plugins parse `args: string[]` to dispatch subcommands:
```rust
match args.get(0).map(|s| s.as_str()) {
    Some("read") => read_messages(args),
    Some("ls") => list_state(),
    Some("--help") => Err("usage: ..."),
    _ => Err("unknown subcommand")
}
```

All error handling returns JSON with `"ok": false` and an error message.

---

## Limitations & Known Issues

### Plugin Installation (WASM git route blocked)

**maw-rs#521**: Installing plugins directly from GitHub is currently blocked. Workaround:
```bash
git clone https://github.com/Soul-Brews-Studio/maw-plugins
maw plugin install --path maw-plugins/packages/NN-verb
```

### Rebuild Determinism (Unproven)

Rust WASM builds are not guaranteed to be deterministic. The CI pin-integrity gate **verifies committed artifacts**, not rebuilt equivalence. This is intentional: pins protect against tampering, not against compiler changes.

### bun-dev Tier (Dev-only)

Plugins with `runtime: "bun-dev"` (share, p2p-share) run as TypeScript on the dev machine but cannot be shipped as standalone WASM. They require the full maw-js TypeScript runtime.

---

## Summary

**maw-plugins** is a tightly organized, security-conscious plugin monorepo that:

1. **Separates concerns**: 26 packages, each a self-contained unit with a manifest and optional WASM artifact
2. **Enforces determinism**: Registry and pins are regenerated from source; CI gates ensure freshness
3. **Sandboxes plugins**: Capabilities limit what each plugin can access (files, secrets, HTTP endpoints)
4. **Supports multiple tiers**: Rust → WASM (ship), AssemblyScript → WASM (fleet), TypeScript (bun-dev, dev-only)
5. **Pins for security**: Every WASM artifact is SHA256-pinned and verified by CI
6. **Enables offline installation**: registry.json is an AUTO-TRUST root; plugins run without first-run prompts

The architecture balances **flexibility** (multiple implementation languages), **safety** (capability-based access control, SHA256 pinning), and **efficiency** (lean WASM binaries via Extism PDK).
