# maw-js Architecture Documentation

**Project**: maw-js v26.6.14-alpha.2110  
**Description**: Multi-Agent Workflow CLI for running multiple AI agents across machines via tmux. Built on Bun/TypeScript, engine-agnostic (Claude Code, Codex, Aider, OpenCode).

---

## 1. High-Level Overview

maw-js is a distributed agent orchestration CLI that:
- Wakes up AI agents in tmux windows on local or remote machines
- Enables inter-agent communication via structured messages
- Tracks session costs and provides terminal UI/API access
- Manages fleet configuration, federation across nodes, and plugin extensibility

**Core Value**: One command set works uniformly across single-node or multi-node deployments. Plugins extend functionality without modifying core.

---

## 2. Directory Structure & Organization Philosophy

### Top-Level Layout
```
maw-js/
├── src/                   # Main source directory (TypeScript)
├── test/                  # Test suites (800+ test files)
├── docs/                  # Documentation
├── packages/              # Workspace packages (@maw-js/sdk, etc.)
├── scripts/               # Utility scripts (build, test, deploy)
├── demo/                  # Demo agents and example configs
├── docker/                # Docker images
├── completions/           # Shell completion scripts
├── ui/                    # Web UI (separate from core)
├── bun.lock               # Bun lockfile
└── GATEWAY_CONTRACT.md    # API contract specification
```

### src/ Directory (Core Architecture)

```
src/
├── cli.ts                      # Entry point (dispatcher bootstrap)
├── cli/                        # CLI infrastructure
│   ├── dispatch.ts             # Command routing logic (25KB)
│   ├── command-registry.ts     # Scans & registers user commands
│   ├── command-registry-*.ts   # WASM + matching + execution
│   ├── dispatch-match.ts       # Plugin name matching logic
│   ├── dispatch-flag-parse.ts  # Flag parsing for plugins
│   ├── top-aliases.ts          # RFC #954 verb aliases
│   ├── route-comm.ts           # hey/send/notify shortcuts
│   ├── route-tools.ts          # Tool-specific routes
│   ├── plugin-bootstrap.ts     # Auto-install bundled plugins
│   ├── instance-preset.ts      # --as <name> handling
│   ├── usage.ts                # Help/usage text
│   ├── verbosity.ts            # --quiet / --silent flags
│   └── *.ts                    # Version, update, auto-restore, etc.
│
├── plugin/                     # Plugin system (types, registry, lifecycle)
│   ├── types.ts                # PluginManifest, InvokeContext, InvokeResult
│   ├── manifest.ts             # Load plugin.json & validate
│   ├── registry.ts             # Discover & memoize plugin packages
│   ├── registry-invoke.ts      # Invoke plugin handler
│   ├── registry-semver.ts      # SDK version gate + mismatch errors
│   ├── registry-helpers.ts     # Hashing, dev-mode detection, profiling
│   ├── dependencies.ts         # Plugin dependency resolution
│   ├── lifecycle.ts            # Plugin lifecycle hooks
│   └── 20+ lifecycle files     # Before/after-init, notify, startup, etc.
│
├── vendor/                     # Vendor plugin packages
│   └── mpr-plugins/            # MPR (maw plugin registry) — 100+ plugins
│       ├── about/              # Show oracle info
│       ├── fleet/              # Fleet management commands
│       ├── tmux/               # Tmux-specific operations
│       ├── oracle/             # Oracle lifecycle
│       ├── plugin/             # Plugin management (install, enable, etc.)
│       ├── costs/              # Cost tracking
│       ├── team/               # Team coordination
│       ├── discord/            # Discord integration
│       └── [90+ more]          # Various domain plugins
│
├── commands/                   # Built-in command implementations
│   ├── plugins/                # Plugin-related commands
│   └── shared/                 # Shared utilities for commands
│       ├── comm.ts             # cmdPeek, cmdSend (messaging)
│       ├── fleet*.ts           # Fleet operations
│       ├── wake*.ts            # Agent wake patterns
│       ├── queue-store.ts      # Message queue persistence
│       ├── scan-signals.ts     # Signal file scanning
│       └── [20+ shared modules]
│
├── core/                       # Core abstractions & engine
│   ├── fleet/                  # Fleet management (10+ files)
│   │   ├── leaf.ts             # Signal writing API
│   │   ├── oracle-registry.ts  # Registry of oracles per node
│   │   ├── worktrees.ts        # Worktree metadata
│   │   ├── snapshot.ts         # Point-in-time fleet state
│   │   ├── audit.ts            # CLI audit logging
│   │   └── [more]
│   │
│   ├── transport/              # Transport abstraction layer
│   │   ├── tmux.ts             # Barrel export (wrapper)
│   │   ├── tmux-*.ts           # Tmux abstractions (types, class, locks, tags)
│   │   ├── ssh.ts              # SSH host execution + attach
│   │   ├── peers.ts            # Federation peer discovery
│   │   ├── mqtt-publish.ts     # MQTT event publishing
│   │   ├── curl-fetch.ts       # HTTP client
│   │   └── transport.ts        # Abstract transport interface
│   │
│   ├── matcher/                # Target resolution & matching
│   │   ├── resolve-target.ts   # Resolve session/worktree targets
│   │   └── normalize-target.ts # Normalize target syntax
│   │
│   ├── routing.ts              # High-level routing logic
│   ├── gateway.ts              # API gateway setup
│   ├── server.ts               # Elysia HTTP server
│   ├── resolve.ts              # Multi-step resolution
│   ├── paths.ts                # Path resolution (home, config, etc.)
│   ├── xdg.ts                  # XDG Base Directory Specification
│   ├── types.ts                # Core type definitions
│   ├── consent/                # Consent management
│   ├── runtime/                # Runtime hooks & lifecycle
│   ├── engine/                 # Engine plugins & dispatch
│   ├── static/                 # Static assets
│   └── util/                   # Utilities (fuzzy match, terminal, user-error)
│
├── config/                     # Configuration management
│   ├── load.ts                 # Load maw.config.json
│   ├── types.ts                # MawConfig interface
│   ├── engine-registry.ts      # Engine definitions (Claude Code, Aider, etc.)
│   ├── engine-def.ts           # EngineDef shape
│   ├── ghq-root.ts             # ghq repository root detection
│   └── index.ts                # Barrel exports + defaults
│
├── sdk/                        # Public SDK (exported to plugins)
│   └── index.ts                # Plugin API surface (types, config, transport, etc.)
│
├── lib/                        # Library utilities
│   ├── schemas.ts              # Type definitions & validators
│   ├── oracle-manifest.ts      # Oracle metadata loading
│   ├── message-events.ts       # Feed event types
│   ├── profile-loader.ts       # Profile filtering
│   ├── artifacts.ts            # Artifact tracking
│   ├── feed.ts                 # Event feed interface
│   ├── sparkline.ts            # Sparkline rendering
│   └── sleep.ts                # Utility delays
│
├── api/                        # REST API definitions
├── views/                      # HTML/template views
├── bridges/                    # Bridge integrations
├── transports/                 # Additional transport implementations
├── wasm/                       # WASM module support
├── static/                     # Static assets (CSS, JS)
└── engine/                     # AI engine abstractions
```

### Organization Philosophy

1. **Layered approach**: CLI → Commands → Core Abstractions → Transport
2. **Plugin-first**: Bundled functionality lives in `vendor/mpr-plugins/` using the same plugin system as user plugins
3. **Transport abstraction**: Tmux, SSH, and federation peers are interchangeable backends
4. **XDG compliance**: Runtime paths migrate toward XDG Base Directory Specification for multi-user systems
5. **Modular exports**: `package.json` exports specific paths so consumers only import what they need

---

## 3. Entry Points & Command Resolution

### CLI Entry Point: `src/cli.ts`

Sequence of operations:

1. **Instance Preset** (--as flag)
   ```typescript
   // Apply before any state-touching import
   applyInstancePreset(); // Sets MAW_HOME if --as provided
   ```

2. **Verbosity Flags**
   ```typescript
   // Strip --quiet/-q, --silent/-s before command detection
   const verbosity = { quiet?, silent? };
   setVerbosityFlags(verbosity);
   ```

3. **Plugin Bootstrap**
   ```typescript
   // Auto-symlink bundled plugins + install from pluginSources
   await runBootstrap(pluginDir, import.meta.dir);
   ```

4. **Plugin Scanning**
   ```typescript
   // Load plugins into registry (single-invocation cache)
   await scanCommands(pluginDir, "user");
   ```

5. **Command Dispatch**
   ```typescript
   await dispatchCommand(cmd, args);
   ```

### Command Dispatch Flow: `src/cli/dispatch.ts`

The dispatcher implements a **priority-based routing ladder**:

```
Input: cmd, args[]
       ↓
[1] routeComm()           → hey, send, notify shortcuts
       ↓ (not handled)
[2] routeTools()          → Tool-specific routes (internal)
       ↓ (not handled)
[3] Top-level aliases     → RFC #954 verb rewrites (e.g., "v" → "version")
       ↓ (not handled)
[4] matchCommand()        → Beta command registry (src/cli/command-registry-*.ts)
       ↓ (not handled)
[5] dispatchPluginRegistry()
    ├─ resolvePluginMatch()  → Plugin name matching (exact, prefix, word-boundary)
    ├─ Dependency check
    ├─ Flag validation
    ├─ Parse declared flags into ctx.flags
    └─ invokePlugin()
       ↓ (not handled / no match)
[6] Fuzzy matching        → Suggest close typo matches
       ↓ (still no match)
[7] Oracle name shorthand → Check if args[0] is a tmux session name
       ↓ (if match)
    Send message or peek at session
```

**Key Design Decisions**:

- **Prefix match requires word boundary**: `n + " "` instead of `startsWith(n)` (issue #349/#351/#354) — prevents "rest" plugin matching "restart --help"
- **Lowercase ONLY for matching**: Original-case args passed to plugin so team names, paths stay case-correct (issue #393)
- **Instance isolation via --as**: Resets MAW_HOME before loading config (issue #566)
- **Prefix auto-resolve**: "v" uniquely prefixes "version", runs without ambiguity
- **Fuzzy first, tmux query second**: Only spend tmux query if arg shape looks like an oracle name

---

## 4. Core Abstractions & Relationships

### 4.1 Fleet System (`src/core/fleet/`)

**Fleet**: A distributed collection of nodes (local + remote) running oracles (tmux sessions).

**Key Types & Files**:

- **`oracle-registry.ts`** — Registry of all oracles on this node; maps `oracle_name → TmuxSession`
- **`snapshot.ts`** — Point-in-time view of fleet state (all oracles, windows, panes)
- **`worktrees.ts`** — Metadata about repositories and worktrees per oracle
- **`leaf.ts`** — Signal writing API (writes JSON to `ψ/memory/signals/<date>_<bud>_<slug>.json`)
- **`audit.ts`** — CLI command audit logging
- **`node-identity.ts`** — Hostname + SSH identity for federation
- **`parent-session.ts`** — Resolve parent oracle for child sessions
- **`validate.ts`** — Validate fleet state consistency

**Federation Support**:
- Nodes discover peers via gateway or mDNS
- Each node has a unique identity
- Cross-node targeting: `{node}:{session}:{window}:{pane}`

### 4.2 Transport Layer (`src/core/transport/`)

**Abstraction**: Pluggable backends for executing commands, capturing output, splitting windows.

**Implementations**:

1. **Tmux** (`tmux.ts`, `tmux-class.ts`, `tmux-types.ts`)
   ```typescript
   class Tmux {
     q(cmd: string): Promise<string>     // Query (read-only)
     x(cmd: string): Promise<void>       // Execute (write)
     sendKeys(target, keys, enter?)
     listSessions(): Promise<TmuxSession[]>
   }
   export const tmux = new Tmux();
   ```

2. **SSH** (`ssh.ts`, `ssh-attach.ts`)
   ```typescript
   hostExec(transport, cmd)               // Execute on remote
   listSessions()                         // List remote tmux sessions
   attachRemoteSession(opts)              // SSH + tmux attach
   ```

3. **Federation Peers** (`peers.ts`)
   ```typescript
   getPeers()                             // Discover federation peers
   findPeerForTarget(target)              // Route to appropriate node
   ```

4. **Utilities**
   - `tmux-pane-lock.ts` — Lock panes during concurrent operations
   - `tmux-pane-tags.ts` — Metadata tags on panes (oracle_id, bud_name, etc.)
   - `curl-fetch.ts` — HTTP client (wraps curl for sandbox compatibility)
   - `mqtt-publish.ts` — Publish events to MQTT broker

### 4.3 Plugin System (`src/plugin/`)

**Plugin Package Structure**:
```
~/.maw/plugins/<plugin-name>/
├── plugin.json              # Manifest
├── index.ts                 # TS entry (full access) OR
└── <name>.wasm              # WASM entry (sandboxed)
```

**Plugin Manifest** (`plugin.json`):
```json
{
  "name": "about",
  "version": "1.0.0",
  "entry": "./index.ts",
  "sdk": "^1.0.0",
  "cli": {
    "command": "about",
    "aliases": ["info"],
    "help": "maw about <oracle>",
    "flags": {
      "--verbose": "boolean",
      "--format": "string"
    }
  },
  "api": {
    "path": "/api/about",
    "methods": ["GET"]
  },
  "weight": 10,
  "dependencies": { "plugins": ["fleet"] }
}
```

**Plugin Invoke Context**:
```typescript
interface InvokeContext {
  source: "cli" | "api" | "hook";
  args: string[] | Record<string, unknown>;
  flags?: Record<string, any>;           // Parsed CLI flags
  matchedName?: string;
  writer?: (...args: any[]) => void;     // Output writer
}

interface InvokeResult {
  ok: boolean;
  output?: string;
  error?: string;
  exitCode?: number;
}
```

**Plugin Surfaces**:

1. **CLI** — Invoked via `maw <command> [args]`
   - Matched via word-boundary prefix matching
   - Longest prefix wins

2. **API** — HTTP route, e.g., `GET /api/about?oracle=foo`
   - Registered via gateway
   - JSON request/response

3. **HTTP Routes** — Custom serve routes
   - Plugin registers handlers: `route("GET", "/custom", handler)`
   - Fallback route for static content

4. **WebSocket** — Real-time bidirectional communication
   - Feed event subscriptions
   - Interactive sessions

5. **Lifecycle Hooks** — Async plugins loaded at startup
   - `before-init` — Pre-fleet setup
   - `after-init` — Post-fleet ready
   - `notify` — Event notifications
   - Custom hooks per plugin

**Discovery & Lifecycle**:

```typescript
// discoverPackages() loads all ~/.maw/plugins/*/ with plugin.json
// For each plugin:
// 1. Validate SDK version (semver gate)
// 2. Check artifact hash (if non-symlink)
// 3. Dev-mode skip for symlinks
// 4. Warn legacy manifests (no artifact field)
// Memoized per CLI invocation (reused across dispatch calls)
```

**Dependency Resolution**:
- `manifest.dependencies.plugins` lists required plugins
- Topological sort ensures load order
- Missing deps → error with remediation hint
- Disabled deps → error with enable instructions

### 4.4 Message & Signaling (`src/commands/shared/`)

**Messaging Architecture**:

1. **Peek** (`cmdPeek`) — Show last N messages from a session
2. **Send** (`cmdSend`) — Queue message for session to receive (via event feed)
3. **Queue Store** (`queue-store.ts`) — Persistent queue in `ψ/queue/<session>.jsonl`

**Signaling** (`leaf.ts`, `scan-signals.ts`):
- Oracles write signals to `ψ/memory/signals/<date>_<bud>_<slug>.json`
- Signals: `{ timestamp, bud, kind, message, context }`
- Parent can scan and act on signals (e.g., move session, escalate, notify)

---

## 5. Dependencies & Key Libraries

### package.json Summary

**Runtime Dependencies** (v26.6.14-alpha.2110):

```json
{
  "@maw-js/sdk": "workspace:*",
  "@eclipse-zenoh/zenoh-ts": "^1.9.0",      // Distributed systems messaging
  "@elysiajs/cors": "^1.4.1",                // CORS middleware
  "@elysiajs/swagger": "^1.3.1",             // OpenAPI docs
  "@monaco-editor/react": "^4.7.0",          // Code editor (web)
  "@sinclair/typebox": "^0.34.49",           // JSON Schema validation
  "@xterm/addon-fit": "^0.11.0",             // Terminal UI (web)
  "@xterm/xterm": "^5.5.0",                  // xterm.js
  "arg": "^5.0.2",                           // CLI arg parsing
  "elysia": "^1.4.28",                       // Bun web framework
  "hono": "^4.12.5",                         // Middleware framework
  "mqtt": "^5.15.1",                         // MQTT client
  "react": "^19.0.0",                        // Web UI
  "react-dom": "^19.0.0",
  "three": "^0.184.0",                       // 3D visualization
  "typescript": "^6.0.3",
  "zustand": "^5.0.11"                       // State management
}
```

**Dev Dependencies**:
- `@types/bun`, `@types/react`, `@types/react-dom`, `@types/three`
- `vite`, `@vitejs/plugin-react`, `@tailwindcss/vite`, `tailwindcss` (v4)
- `@resvg/resvg-js` (SVG rendering)

**Bun-Specific APIs Used**:

- `import.meta.dir` — Resolve plugin dir relative to src/
- `bun test` — Test runner (800+ test files)
- `process.env.MAW_*` — Custom env vars
- Bun's native `fetch`, WebSocket support
- Bun's glob expansion for file patterns

**Workspace Packages**:
- `@maw-js/sdk` — Public SDK (src/sdk/index.ts re-exported)

---

## 6. Configuration Management (`src/config/`)

**Config File**: `maw.config.json` (per-instance or global)

**Key Exports**:

```typescript
loadConfig()                              // Load maw.config.json
saveConfig(config)                        // Write config
buildCommand(template)                    // Substitute env vars
getEnvVars()                              // Collect MAW_* variables
cfg, D                                    // Config accessors + defaults
cfgTimeout(ms)                            // Timeout settings
cfgLimit(n), cfgInterval(ms)              // Limits & intervals
```

**Engine Registry** (`engine-registry.ts`):

Pre-defined AI engines:
- Claude Code (`--engine claude-code`)
- Aider (`--engine aider`)
- Codex (`--engine codex`)
- OpenCode (`--engine opencode`)

Each engine has a template command + parameter resolution.

**Paths** (`paths.ts`, `xdg.ts`):

- `mawConfigDir()` — `~/.config/maw/` or `$XDG_CONFIG_HOME/maw/`
- `mawDataDir()` — `~/.maw/` or `$XDG_DATA_HOME/maw/`
- `mawStateDir()` — `~/.maw/state/` or `$XDG_STATE_HOME/maw/`
- `mawCacheDir()` — `~/.cache/maw/` or `$XDG_CACHE_HOME/maw/`

Enable XDG mode: `export MAW_XDG=1`

---

## 7. API & Web Server (`src/core/server.ts`, `src/api/`)

**Framework**: Elysia (Bun web framework, similar to Express)

**Key Routes**:

- **WebSocket** — `/ws` — Real-time sessions, feed events
- **API** — `/api/*` — Plugin-registered endpoints
- **Gateway** — `/gateway/*` — Plugin fallback routes
- **Static** — `/` — UI, assets

**Served on**: Port 3456 by default (can override via `MAW_PORT`)

**Features**:
- CORS middleware (configurable)
- Swagger/OpenAPI documentation
- Type-safe route definitions (Typebox schemas)
- Plugin route registration via `ServeRouteRegistrar`

---

## 8. Command Registration & Execution (`src/cli/command-registry-*.ts`)

### Registration

**Scanned Sources** (in order of load):

1. `~/.oracle/commands/` — User commands (legacy, single-file)
2. `~/.maw/plugins/<name>/` — Plugin packages (new, multi-surface)
3. Built-in commands (core routes: hey, send, notify, etc.)

**File Support**:

- `.ts` — TypeScript (dynamic import + eval)
- `.js` — JavaScript (dynamic import + eval)
- `.wasm` — WebAssembly (sandboxed, WASM-bridge)

**Command Descriptor**:

```typescript
interface CommandDescriptor {
  name: string | string[];             // "fleet doctor" or ["fleet", "fd"]
  description: string;
  path: string;                        // File path
  scope: "builtin" | "user";
  handler?: (args: string[]) => Promise<void>;
}
```

### Execution

**Matching**: Longest prefix wins within registered commands

```typescript
matchCommand(["fleet", "doctor", "--dry-run"])
// Matches: "fleet doctor" (2 words)
// Remaining: ["--dry-run"]
```

**Execution** (WASM + TS):

1. Parse args from CLI
2. Invoke handler with context
3. Capture output
4. Exit with result code

---

## 9. SDK Public Surface (`src/sdk/index.ts`)

Exported for both TS and WASM plugins:

**Types**:
- `PluginManifest`, `LoadedPlugin`, `InvokeContext`, `InvokeResult`
- `EngineDef`, `EngineRegistry`, `TScope`

**Config**:
- `loadConfig()`, `saveConfig()`, `getEnvVars()`, `cfg`, `D`
- `buildCommand()` — Substitute template vars
- Engine registry functions

**Transport**:
- `tmux`, `Tmux` class, `tmuxCmd()`
- `hostExec()`, `listSessions()`, `capture()`, `sendKeys()`
- `resolveSocket()`, `withPaneLock()`, `splitWindowLocked()`
- `tagPane()`, `readPaneTags()`
- `getPeers()`, `getFederationStatus()`, `findPeerForTarget()`
- `resolveTarget()`, `normalizeTarget()`, `resolveSessionTarget()`

**Consent**:
- `listPending()`, `listTrust()`, `recordTrust()`, `removeTrust()`
- `approveConsent()`, `rejectConsent()`

**Paths & XDG**:
- `mawCacheDir()`, `mawConfigDir()`, `mawDataDir()`, `mawStateDir()`, `mawDataPath()`
- `isMawXdgEnabled()`, `legacyMawPath()`, `mawMessageLogPath()`

**Utilities**:
- `loadCommandFromDir()`, `createLogger()`, `sleep()`
- `validateSchema()`, `parseProfileFilter()`

---

## 10. Testing Infrastructure

**Test Files**: 800+ across:

- `test/spec/` — Specification tests
- `test/isolated/` — Isolated unit tests
- `test/` — Integration & mock-transport tests
- `src/commands/plugins/` — Plugin-specific tests
- `.test.ts` files throughout src/

**Test Commands**:

```bash
bun run test:default:safe       # Default safe suite
bun run test:isolated           # Isolated tests
bun run test:mock-smoke         # Mock transport smoke tests
bun run test:plugin             # Plugin tests
bun run test:all                # All suites
bun run test:coverage           # Coverage report
```

**Environment**:

- `MAW_TEST_MODE=1` — Activate test-specific behavior
- Mock transport for testing without tmux
- Memoized discovery cache reset between tests

---

## 11. Vendor Plugins (`src/vendor/mpr-plugins/`)

**MPR** (maw Plugin Registry) contains 100+ production plugins:

**Core Plugins** (tier: core):
- `about` — Oracle info
- `plugin` — Plugin management (install, enable, disable, ls)
- `plugins` — Plugin listing
- `audit` — Audit logs
- `agents` — Agent discovery
- `serve` — Start API + UI server

**Fleet Management** (tier: standard):
- `fleet` — Fleet commands (ls, sync, doctor, load)
- `tmux` — Tmux operations
- `attach` — Attach to sessions
- `awaken`, `wake` — Create new oracles
- `bring` — Move oracle to target window

**Communication**:
- `hey` — Send message to oracle
- `send` — Queue message
- `notify` — Send notification
- `broadcast` — Send to multiple oracles

**Features**:
- `costs` — Track LLM costs
- `team` — Team coordination
- `discord` — Discord integration
- `contacts` — Contact management
- `archive` — Archive sessions
- `capture` — Screenshot/record panes
- `cleanup` — Clean up old sessions
- `completions` — Shell completions
- `consent` — Consent workflows
- `demo` — Demo plugins
- `doctor` — Diagnostic tools

**Plugin Lifecycle**:

Each plugin:
1. Defines CLI command (if any) in `plugin.json`
2. Exports `default async function handler(ctx: InvokeContext): Promise<InvokeResult>`
3. Can register HTTP routes, WebSocket handlers, lifecycle hooks
4. Can depend on other plugins (resolved at load time)

---

## 12. Data Flow Example: `maw hello world`

1. **CLI Entry** (`src/cli.ts`)
   - Apply instance preset
   - Set verbosity
   - Bootstrap plugins
   - Scan plugins
   - `dispatchCommand("hello", ["world"])`

2. **Dispatch** (`src/cli/dispatch.ts`)
   - Try `routeComm("hello", ["world"])` → No match
   - Try `routeTools("hello", ["world"])` → No match
   - Try top-level aliases → No match
   - Try beta command registry → No match
   - Try plugin registry:
     - Discover packages (cached)
     - Match "hello" against plugin names
     - Find plugin "hello"
     - Resolve dependencies
     - Validate flags
     - Parse declared flags
     - Call `invokePlugin(plugin, { source: "cli", args: ["world"] })`

3. **Plugin Execution** (`src/plugin/registry-invoke.ts`)
   - Load plugin.json manifest
   - Load index.ts or hello.wasm
   - Call `handler(ctx)` with context
   - Capture output
   - Return `InvokeResult`

4. **Output** → Print to console, exit with code

---

## 13. Key Design Patterns & Principles

### 1. Instance Isolation (`--as <name>`)

Sets `MAW_HOME=~/.maw-<name>/` before loading config. Enables:
- Running multiple maw instances on same machine
- Testing without affecting production state
- Per-user/per-project configs

**Applied at module load time** (issue #566) — before imports that evaluate paths.

### 2. Plugin-First Architecture

- Core functionality lives in plugins (`vendor/mpr-plugins/`)
- Same API as user plugins (no privileged paths)
- Bundled plugins auto-bootstrap if missing
- Dependency graph ensures load order

### 3. Transport Abstraction

- Tmux, SSH, federation peers all pluggable
- Abstract `transport.ts` defines interface
- Routes dispatch based on target context
- Pane locks ensure concurrent-safe operations

### 4. XDG Base Directory Migration

- Supports legacy `~/.maw/` and `~/.config/maw/`
- Opt-in XDG: `MAW_XDG=1`
- Path resolver prioritizes modern paths
- Doctor command helps migrate

### 5. Memoized Discovery

- Plugin discovery scanned once per CLI invocation
- Cached in memory (`_discoverCache`)
- Tests can reset cache
- Profiler showed ~50ms per call, now ~0 amortized

### 6. Consent Workflow

- Explicit approval for sensitive operations
- Trust recording (approve once, remember)
- `src/core/consent/` manages state
- Exported in SDK for plugins

### 7. Signal Passing

- Inter-oracle communication via JSON files
- Written to `ψ/memory/signals/<date>_<bud>_<slug>.json`
- Parent oracle can scan and react
- Lightweight, no server dependency

### 8. Error Handling

- `UserError` class for user-facing messages
- Fuzzy matching suggests corrections
- Detailed error context (missing deps, disabled plugins, etc.)
- Top-level error handler strips stack traces in production

---

## 14. Notable Implementation Details

### Plugin Bootstrapping

```typescript
// src/cli/plugin-bootstrap.ts
// If ~/.maw/plugins/ is empty:
// 1. Symlink bundled plugins from src/vendor/mpr-plugins/
// 2. Fetch additional plugins from pluginSources (optional)
// Later: Plugins get built, sha256 recorded in artifact field
```

### Flag Parsing

```typescript
// Plugins declare flags in plugin.json:
"flags": { "--verbose": "boolean", "--format": "string" }
// CLI parses into ctx.flags, validates unknown flags
// Fuzzy suggestions for typos
```

### Prefix Auto-Resolution

```typescript
// "v" uniquely prefixes ["version"]
// "up" uniquely prefixes ["update", "upgrade"]
// Auto-resolve + re-dispatch to correct command
```

### Pane Tagging

```typescript
// Store metadata on tmux panes:
// Tags: oracle_id, bud_name, repo_path, branch
// Used by commands to map panes to semantic context
```

---

## 15. Build & Deployment

### Build

```bash
bun build src/cli.ts \
  --outfile dist/maw \
  --target=bun \
  --minify \
  --external @eclipse-zenoh/zenoh-ts
```

**External deps**: Zenoh excluded (optional, for federation).

### Entry Point

`package.json` bin:
```json
"bin": { "maw": "./src/cli.ts" }
```

Direct TypeScript execution via Bun (no pre-compilation needed).

### Distribution

- **npm**: `bun add -g github:Soul-Brews-Studio/maw-js` (latest)
- **CalVer**: v{yy}.{m}.{d}[-alpha.{HHMM}], e.g., v26.6.14-alpha.2110
- **Update cmd**: `maw update` pulls latest release

---

## Summary

**maw-js** is a sophisticated multi-agent CLI built on a modular plugin architecture. Its core strength is the **dispatch ladder** that routes commands through multiple layers (core routes → tools → aliases → plugins → fuzzy matching), combined with a **transport abstraction** that works uniformly across local tmux and remote SSH targets. The **plugin system** reuses the same manifest/invocation interface for both bundled and user plugins, enabling extensibility without privilege escalation. **Fleet management** and **federation** support enable deploying orchestration across many machines without per-node configuration changes.

Key files for quick reference:
- **Entry**: `src/cli.ts`
- **Dispatch**: `src/cli/dispatch.ts` (25KB, comprehensive routing)
- **Plugin types**: `src/plugin/types.ts`
- **Plugin registry**: `src/plugin/registry.ts`
- **Transport**: `src/core/transport/` (Tmux, SSH, peers)
- **Fleet**: `src/core/fleet/` (Oracle registry, snapshot, signals)
- **SDK**: `src/sdk/index.ts` (Public plugin API)
- **Plugins**: `src/vendor/mpr-plugins/` (100+ production plugins)
