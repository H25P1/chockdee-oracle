# maw-plugins Quick Reference

**maw-plugins** is the installable plugin repository for **maw-js** — a terminal multiplexing and fleet-management tool for AI agent oracles. Each plugin ships as a `wasm32-unknown-unknown` compiled artifact, pinned by sha256 in a manifest.

---

## What This Project Does

maw-plugins extracted the verb plugins from maw-rs core (lean-core extraction) into individually installable packages. Each plugin is a self-contained `Cargo.toml` + source + committed WASM artifact.

### Plugin Directory

**24 plugins** in total, organized by execution weight and tier:

#### Weight 20 (Tools & CLI Verbs)

| Plugin | Purpose |
|--------|---------|
| **avengers** | Configuration reader for oracle agent teams |
| **broadcast** | Send commands across all tmux panes in a fleet |
| **contacts** | Read/write agent contact roster (`psi` storage) |
| **costs** | Show token usage and cost breakdown per agent (`maw costs [--daily] [--json]`) |
| **demo** | Demo plugin with tmux pane manipulation |
| **dream** | Repository awareness and git state queries |
| **follow** | Track active tmux panes (read-only) |
| **hub** | Central config reader/writer |
| **layout** | Manage tmux window layouts |
| **mega** | Fleet orchestration and session state |
| **pulse** | Health monitoring with git, gh, date, and maw state |
| **soul-sync** | Synchronize oracle identity across the fleet |
| **stream** | Create and manage tmux sessions/windows |
| **tile** | Advanced tmux pane tiling and layout |

#### Weight 50 (Features & Advanced)

| Plugin | Purpose |
|--------|---------|
| **artifact-manager** | Manage build artifacts and cache |
| **incubate** | Launch and orchestrate plugins via `bud` CLI |

#### Fleet Plugins (Imported from maw-rs, Unprefixed)

Ship-tier WASM or bun-dev tier; referenced verbatim by maw-rs's install hints:

| Plugin | Type | Purpose |
|--------|------|---------|
| **atlas** | wasm | Discord fleet state core — read-only guild/channel/message inventory |
| **cross-team-queue** | wasm | Multi-team queue and message routing |
| **hermes** | wasm | Discord REST read-only — channels, messages, threads |
| **maw-menubar** | bun-dev + native | Native Swift macOS menubar integration |
| **p2p-share** | bun-dev | WebRTC peer-to-peer terminal sharing (requires werift) |
| **share** | bun-dev | sshx-fork terminal sharing bridge |
| **squad** | wasm | Multi-oracle team sessions and chat (`maw squad start`, `join`, `say`, `ls`) |
| **team** | wasm | Team inventory and member discovery |

---

## Installation

### From Local Clone (Recommended)

Currently, the git remote install path is [blocked](https://github.com/Soul-Brews-Studio/maw-rs/issues/521). Install from a local checkout instead:

```bash
# Clone the repo once
git clone https://github.com/Soul-Brews-Studio/maw-plugins
cd maw-plugins

# Install any plugin by path
maw plugin install --path packages/20-costs
maw plugin install --path packages/squad
```

### From Remote (When Available)

Once the git install route unblocks:

```bash
maw plugin install Soul-Brews-Studio/maw-plugins/packages/20-costs --sha256 <pin>
```

The sha256 pin is available in [`registry.json`](../registry.json) under the plugin name (e.g., `registry.json` → `costs.sha256`).

### Setup (TypeScript/Bun Plugins Only)

Run once to link maw-js into node_modules:

```bash
npm run setup
# or
bun run setup
```

This creates a symlink so plugins can resolve `import ... from "maw-js/sdk"`. The setup script looks for maw-js in this order:
1. `$MAW_JS_PATH` environment variable
2. Sibling checkout: `../maw-js` (same parent as maw-plugins)
3. Bun global: `~/.bun/install/global/node_modules/maw-js`

---

## Key Features & Examples

### Running a Plugin

```bash
# Simple command
maw costs --json

# With options
maw costs --daily 7 --json

# Alias (atlas example)
maw at ls    # "at" is an alias for "atlas"

# Squad (multi-oracle team chat)
maw squad start
maw squad join <oracle-name> [color]
maw squad say <member-name> "hello world"
```

### Plugin Manifest Structure

Each plugin declares its surface in **`plugin.json`** (or `plugin.source.json` for dev-tier fleet plugins):

```json
{
  "name": "costs",
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
    "sha256": "sha256:1947d3ed3f1a1bc5b192d918c6cc02eddd46196cf4b3455b2cbfabd51f1d2f58"
  },
  "cli": {
    "command": "costs",
    "aliases": ["c"],
    "help": "maw costs [--daily [N]|--days N] [--json]"
  },
  "capabilities": [
    "fs:read:claude-projects"
  ],
  "weight": 20,
  "author": "Soul-Brews-Studio",
  "description": "Show token usage and estimated cost breakdown per agent.",
  "sdk": "^1.0.0"
}
```

**Key manifest fields:**

- **`target`** — `"wasm"` (compiled) or `"js"` (bun-dev tier)
- **`entry`** — How to load: `{ kind: "wasm", path, export: "handle" }` or `{ entry: "src/plugin.ts" }`
- **`artifact.sha256`** — Pinned hash of committed `plugin.wasm` (enforced at load time and CI)
- **`cli.command`** — The maw verb (`maw costs`)
- **`cli.aliases`** — Short aliases (optional; e.g., `maw c` for `maw costs`)
- **`cli.help`** — Usage string shown in `maw <cmd> --help`
- **`capabilities`** — Host capabilities the plugin needs (fs read/write scopes, tmux, net, secrets, proc)
- **`weight`** — Execution order; lower fires first (weight 20 before weight 50)
- **`sdk`** — SDK version constraint

---

## Writing a New Plugin

### Rust Plugin (WASM)

The standard pattern for new verbs. Uses direct **extism-pdk**.

#### 1. Create the Cargo Project

```bash
mkdir packages/20-myverb
cd packages/20-myverb
```

**Cargo.toml:**

```toml
[package]
name = "maw_myverb_plugin"
version = "0.1.0"
edition = "2021"

[lib]
crate-type = ["cdylib"]

[dependencies]
extism-pdk = "1.4"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
# Add any other deps as needed
```

#### 2. Implement the Plugin Entry

**src/lib.rs:**

```rust
use extism_pdk::*;
use serde::{Deserialize, Serialize};
use serde_json::json;

#[derive(Deserialize)]
struct Context {
    args: Vec<String>,
    // Add other runtime context as needed
}

#[derive(Serialize)]
struct Output {
    ok: bool,
    output: String,
    error: String,
}

#[plugin_fn]
pub unsafe fn handle(input: String) -> FnResult<String> {
    let ctx: Context = serde_json::from_str(&input)
        .unwrap_or(Context { args: Vec::new() });

    // Your plugin logic here
    let result = Output {
        ok: true,
        output: "Hello from myverb!".to_string(),
        error: String::new(),
    };

    Ok(serde_json::to_string(&result).unwrap())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_handle() {
        let input = r#"{"args":[]}"#;
        let result = unsafe { handle(input.to_string()) };
        assert!(result.is_ok());
    }
}
```

#### 3. Create the Manifest

**plugin.json:**

```json
{
  "name": "myverb",
  "version": "0.1.0",
  "schemaVersion": 1,
  "target": "wasm",
  "entry": {
    "kind": "wasm",
    "path": "plugin.wasm",
    "export": "handle"
  },
  "artifact": {
    "path": "./plugin.wasm",
    "sha256": "sha256:PLACEHOLDER"
  },
  "cli": {
    "command": "myverb",
    "help": "maw myverb [args]"
  },
  "capabilities": [],
  "weight": 20,
  "author": "Your Name",
  "description": "A brief description of what this plugin does.",
  "sdk": "^1.0.0"
}
```

#### 4. Build & Pin

```bash
# Add WASM target (one-time setup)
rustup target add wasm32-unknown-unknown

# Build the plugin
cargo build --release --target wasm32-unknown-unknown

# Copy artifact
cp target/wasm32-unknown-unknown/release/maw_myverb_plugin.wasm plugin.wasm

# Compute and update sha256 in plugin.json
shasum -a 256 plugin.wasm
# Update artifact.sha256 in plugin.json with the output
```

#### 5. Test

```bash
cd /path/to/maw-plugins
npm run test
# or for a specific package:
bun run tests/per-package.sh packages/20-myverb
```

### TypeScript/AssemblyScript Plugin (WASM — Fleet Plugins)

For fleet plugins using AssemblyScript (a WebAssembly-compatible TypeScript subset):

```typescript
// src/plugin.ts
declare const Bun: any;

export function handle(input: string): string {
  // Parse input JSON
  const ctx = JSON.parse(input);
  
  // Your logic here
  return JSON.stringify({
    ok: true,
    output: "Hello from AssemblyScript"
  });
}
```

**plugin.json** declares `"target": "wasm"` and `"entry": { "kind": "wasm", "path": "plugin.wasm", "export": "handle" }`.

Build via:
```bash
maw plugin build fleet-plugins/<name>
```

This requires the AssemblyScript toolchain. See [`docs/fleet-plugins.md`](../docs/fleet-plugins.md) for full details.

### Bun-Dev Tier (Development/TypeScript)

For plugins that remain on the **bun-dev** tier (no WASM compilation), use `plugin.json` with:

```json
{
  "name": "mydevplugin",
  "target": "js",
  "runtime": "bun-dev",
  "entry": "src/plugin.ts",
  "weight": 0
}
```

The plugin is executed directly as TypeScript without compilation. Use this for plugins that need:
- Native subprocess spawning or signals
- Long-lived processes
- Local state management (fs access)
- Conditional host capabilities

Example: **share** and **p2p-share** use bun-dev tier because they spawn and manage long-running child processes.

---

## Configuration Options

### Manifest (`plugin.json` / `plugin.source.json`)

All manifest fields:

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | ✓ | Plugin identifier (used in registry, install paths) |
| `version` | string | ✓ | Semantic version |
| `schemaVersion` | number | ✓ | Always `1` |
| `target` | string | ✓ | `"wasm"` or `"js"` |
| `entry` | object | ✓ | `{ kind, path, export }` or `{ entry: "file.ts" }` |
| `artifact` | object | ✓ | `{ path, sha256 }` — for wasm tier only |
| `cli.command` | string | ✓ | User-facing maw verb |
| `cli.aliases` | string[] | | Short aliases (e.g., `["c"]` → `maw c`) |
| `cli.help` | string | ✓ | Usage string |
| `capabilities` | string[] | | Host capabilities required (see table below) |
| `weight` | number | | Execution order; lower fires first (default 0) |
| `author` | string | | Author/team name |
| `description` | string | | One-line description |
| `sdk` | string | | SDK version constraint (e.g., `"^1.0.0"`) |
| `wasm` | string | | Alternate path to WASM artifact (legacy, avoid) |
| `license` | string | | License identifier |
| `runtime` | string | | Tier: `"bun-dev"` or omit (defaults to wasm) |
| `endpoints` | object | | Host endpoint policies (net:fetch only) |
| `secrets` | object | | Secret mappings (`env` → `pass` vault path) |
| `bundledArtifacts` | object | | For native binaries (e.g., maw-menubar) |

### Capabilities (Host Permissions)

Plugins declare required **capabilities** in the manifest; the runtime enforces them at load time.

#### File System (fs:*)

- `fs:read:<scope>` — Read-only access to a scope
  - `fs:read:repos` — Repository directories
  - `fs:read:cwd` — Current working directory
  - `fs:read:config` — Maw config root
  - `fs:read:teams` — Team roster storage
  - `fs:read:psi` — Agent contact data
  - `fs:read:vault` — Encrypted secrets vault
  - `fs:read:fleet-config` — Fleet configuration
  - `fs:read:fleet-legacy` — Fleet legacy state
  - `fs:read:fleet-state` — Fleet current state
  - `fs:read:claude-projects` — Claude projects directory
  - `fs:read:maw-cache` — Build cache
  - `fs:read:maw-legacy` — Legacy maw data

- `fs:write:<scope>` — Write access to a scope (same scopes as read)

#### Terminal Multiplexer (tmux:*)

- `tmux:read` — Read pane/window/session state
- `tmux:send` — Send commands/text to panes
- `tmux:raw:<cmd>` — Raw tmux command (e.g., `tmux:raw:kill-pane`, `tmux:raw:split-window`, `tmux:raw:send-keys`)

#### Network (net:*)

- `net:fetch:<endpoint>` — HTTP fetch to a named endpoint (requires `endpoints` definition)
  - `net:fetch:discord-rest` — Discord REST API
  - `net:fetch:avengers` — Custom endpoint

#### Secrets (secret:*)

- `secret:use:<name>` — Use a named secret from the vault
  - `secret:use:atlas-bot-token` — Discord bot token for Atlas

#### Process (proc:*)

- `proc:exec:<cmd>` — Run a subprocess
  - `proc:exec:git` — Git commands
  - `proc:exec:gh` — GitHub CLI
  - `proc:exec:date` — Date command
  - `proc:exec:maw` — Maw CLI
  - `proc:exec:base64` — Base64 encoding

#### CLI (cli:*)

- `cli:run:<cmd>` — Invoke another CLI tool
  - `cli:run:bud` — Run the `bud` plugin scaffolder

#### Execution (exec:*)

- `exec:home` — Change directory to home during execution

---

## Build & Deployment

### Local Build

```bash
# Build all plugins
cargo build --release --target wasm32-unknown-unknown

# Or build a single plugin
cd packages/20-costs
cargo build --release --target wasm32-unknown-unknown
cp target/wasm32-unknown-unknown/release/maw_costs_plugin.wasm plugin.wasm
shasum -a 256 plugin.wasm  # Update artifact.sha256 in plugin.json
```

### CI Checks

The repo runs these checks on every push/PR:

1. **Artifact Build** — Compiles each `packages/*/` crate for `wasm32-unknown-unknown`
2. **Pin Integrity** — Verifies that the committed `plugin.wasm` hash matches `artifact.sha256` in the manifest
3. **Test Coverage** — Runs Rust tests in each crate; for TypeScript fleet plugins, runs `bun test` on `*.test.ts` files
4. **Registry Sync** — Ensures `registry.json` stays in sync with actual package metadata

**Pin check fails if:**
- A committed `plugin.wasm` exists but has no `artifact.sha256` pin in either `plugin.json` or `plugin.source.json`
- The file's actual sha256 doesn't match the pin

### Repository Structure

```
maw-plugins/
├── README.md                    # Main documentation
├── package.json                 # Workspaces config
├── registry.json                # Default mawx resolution root (commit/sha256/path per plugin)
├── LICENSE                      # BUSL-1.1 → Apache 2.0
├── docs/
│   ├── fleet-plugins.md         # Detailed fleet plugin lifecycle
│   └── registry.md              # Registry generation rules
├── scripts/
│   ├── setup.sh                 # Link maw-js into node_modules
│   ├── test-per-package.sh      # Run tests per package
│   └── gen-registry.ts          # Generate registry.json
└── packages/
    ├── 20-avengers/             # Rust WASM plugin
    ├── 20-costs/
    ├── 50-artifact-manager/
    ├── atlas/                   # Fleet plugin (WASM)
    ├── share/                   # Fleet plugin (bun-dev tier)
    └── ... (24 total)
```

Each plugin package:
```
packages/NN-name/
├── Cargo.toml               # Rust only; optional for TS plugins
├── package.json             # Bun-dev TS only
├── plugin.json              # Active manifest
├── plugin.wasm              # Committed artifact (WASM only)
├── plugin.source.json       # SOURCE manifest with pin (fleet plugins in dev-active state)
└── src/
    ├── lib.rs               # Rust implementation
    ├── plugin.ts            # TypeScript/AssemblyScript implementation
    └── *.test.ts            # Optional tests
```

---

## Troubleshooting

### "Cannot find module 'maw-js/sdk'"

Run setup to link maw-js:

```bash
npm run setup
# Ensure ../maw-js exists or set $MAW_JS_PATH
```

### Plugin fails to load (artifact hash mismatch)

The `plugin.wasm` hash doesn't match `artifact.sha256` in the manifest:

1. Rebuild the WASM:
   ```bash
   cd packages/20-myverb
   cargo build --release --target wasm32-unknown-unknown
   cp target/wasm32-unknown-unknown/release/maw_myverb_plugin.wasm plugin.wasm
   ```

2. Recompute and update the pin:
   ```bash
   shasum -a 256 plugin.wasm
   # Copy output into artifact.sha256 in plugin.json
   ```

### CI Pin Check Fails

**Cause:** A committed `plugin.wasm` exists but has no pin, or the pin is stale.

**Fix:**

```bash
cd packages/20-myverb
cargo build --release --target wasm32-unknown-unknown
cp target/wasm32-unknown-unknown/release/maw_myverb_plugin.wasm plugin.wasm
shasum -a 256 plugin.wasm
# Update artifact.sha256 in plugin.json
git add plugin.wasm plugin.json
git commit -m "Update myverb plugin: rebuild wasm and pin"
```

### Registry.json Out of Sync

The CI check `registry:check` ensures registry stays current:

```bash
npm run registry:check
# If out of sync, regenerate:
npm run registry
git add registry.json
git commit -m "Update registry"
```

---

## Reference Links

- **Main README** — `README.md`
- **Fleet plugins lifecycle** — `docs/fleet-plugins.md`
- **Registry structure** — `docs/registry.md`
- **Extism PDK** — [github.com/extism/extism](https://github.com/extism/extism)
- **maw-js** — [github.com/Soul-Brews-Studio/maw-js](https://github.com/Soul-Brews-Studio/maw-js)
- **maw-rs** — [github.com/Soul-Brews-Studio/maw-rs](https://github.com/Soul-Brews-Studio/maw-rs)

---

## License

**BUSL-1.1** (Business Source License 1.1), converting to Apache 2.0 on the Change Date.
See `LICENSE` file for details and the Additional Use Grant.
