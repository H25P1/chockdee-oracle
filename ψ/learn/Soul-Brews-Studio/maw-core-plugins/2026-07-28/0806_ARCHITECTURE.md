# maw-core-plugins Architecture

## Overview

**maw-core-plugins** is a monorepo of essential lifecycle plugins for [maw-js](https://github.com/Soul-Brews-Studio/maw-js). These four core plugins are:

1. **wake** — Spawn or attach to an oracle session
2. **sleep** — Gracefully stop a single oracle's tmux window
3. **stop** — Stop ALL fleet sessions (system-wide)
4. **done** — Clean up a finished worktree (rrr, git save, kill, remove)

Unlike the larger [maw-plugins](https://github.com/Soul-Brews-Studio/maw-plugins) monorepo (optional, user-installable), maw-core-plugins ships with maw-js itself. These plugins cannot be removed — they are lifecycle primitives the system depends on.

---

## Repository Structure

```
maw-core-plugins/
├── package.json              # Root workspace (npm/yarn/bun workspaces)
├── README.md
└── packages/
    ├── 00-wake/              # Spawn/attach oracle sessions
    │   ├── plugin.json       # Manifest
    │   ├── index.ts          # Handler entry point
    │   └── wake.test.ts      # Plugin tests (Bun)
    ├── 00-sleep/             # Gracefully stop one oracle window
    │   ├── plugin.json
    │   ├── index.ts
    │   └── sleep.test.ts
    ├── 00-stop/              # Stop all fleet sessions
    │   ├── plugin.json
    │   ├── index.ts
    │   └── stop.test.ts
    └── 00-done/              # Finish a worktree window
        ├── plugin.json
        ├── index.ts
        └── done.test.ts
```

### Weight Convention

All plugins are prefixed `00-` (e.g., `00-wake`). This naming reflects the `weight` field in the manifest (value `0` in plugin.json, implicit; default is 50). Lower weight = executes first. Weight 00 ensures these **agent lifecycle primitives** load before all other plugins.

### No Per-Package package.json

Unlike the larger maw-plugins monorepo, each package directory in maw-core-plugins contains **no package.json**. Configuration is:
- **Manifest**: `plugin.json` (see below)
- **Workspace root**: `/package.json` declares workspaces

---

## Manifest Format (plugin.json)

Each plugin declares a `plugin.json` using the **PluginManifest** schema (defined in maw-js/src/plugin/types.ts). Here's the common shape:

```json
{
  "name": "wake",
  "version": "1.0.0",
  "entry": "./index.ts",
  "sdk": "^1.0.0",
  "description": "Spawn or attach to an oracle session",
  "author": "Soul-Brews-Studio",
  "cli": {
    "command": "wake",
    "aliases": ["w"],
    "help": "maw wake <oracle|org/repo|URL> [task] [--task '<prompt>'] [--new <name>] ...",
    "flags": {
      "--new": "string",
      "--incubate": "string",
      "--issue": "number",
      "--pr": "number",
      "--repo": "string",
      "--task": "string",
      "--fresh": "boolean",
      "--no-attach": "boolean",
      "--list": "boolean"
    }
  },
  "api": {
    "path": "/api/wake",
    "methods": ["POST"]
  }
}
```

### Required Fields

| Field | Type | Purpose |
|-------|------|---------|
| `name` | string | Unique plugin ID (kebab-case, /^[a-z0-9-]+$/) |
| `version` | string | Semver (N.N.N) |
| `entry` | string | Relative path to handler (./index.ts) |
| `sdk` | string | Semver range (e.g., "^1.0.0") maw-js compatibility |

### Optional Fields

| Field | Type | Purpose |
|-------|------|---------|
| `description` | string | Human-readable plugin purpose |
| `author` | string | Plugin author/team |
| `weight` | number | Execution order (0–99; lower first; default 50) |
| `tier` | "core" \| "standard" \| "extra" | Membership contract (advisory in Phase A) |
| `cli` | object | CLI surface (command, aliases, flags, help text) |
| `api` | object | REST/HTTP surface (path, methods) |
| `hooks` | object | Lifecycle hooks (gate, filter, on, late, wake, sleep, serve) |
| `dependencies` | object | Other plugins required before dispatch |

### CLI Surface

The `cli` object defines the command-line interface:

```json
"cli": {
  "command": "done",
  "aliases": ["finish"],
  "help": "maw done <window-name> [--force] [--dry-run] — clean up a finished worktree",
  "flags": {
    "--force": "boolean",
    "--dry-run": "boolean"
  }
}
```

- **command**: Primary command name
- **aliases**: Alternate names (e.g., `finish` as alias for `done`)
- **help**: Usage/docstring
- **flags**: Declared flags and their types (boolean, string, number)

### API Surface

The `api` object defines HTTP access:

```json
"api": {
  "path": "/api/done",
  "methods": ["POST"]
}
```

Enables invocation via REST (e.g., `POST /api/done` with JSON body).

---

## Handler Implementation Pattern

Each plugin exports a default async handler conforming to the **InvokeContext → InvokeResult** contract:

```typescript
import type { InvokeContext, InvokeResult } from "../../../plugin/types";

export const command = {
  name: "done",
  description: "Clean up a finished worktree window: rrr, git save, kill, remove worktree."
};

export default async function handler(ctx: InvokeContext): Promise<InvokeResult> {
  // Implementation
}
```

### InvokeContext

```typescript
interface InvokeContext {
  source: "cli" | "api" | "peer";        // Invocation source
  args: string[] | Record<string, unknown>; // CLI: string[]; API: object
  matchedName?: string;                  // Matched command/alias name
  writer?: (...args: unknown[]) => void; // Output stream (if available)
  flags?: Record<string, boolean | string | number | string[]>; // Parsed flags
}
```

**Dual-source pattern**: Handlers process both CLI and API calls. CLI sends `args` as a string array; API sends args as a JSON object.

### InvokeResult

```typescript
interface InvokeResult {
  ok: boolean;              // Success flag
  output?: string;          // Captured output (if no ctx.writer)
  error?: string;           // Error message (if ok: false)
  exitCode?: number;        // Custom exit code (optional; defaults to 1 on failure)
}
```

### Common Pattern: Log Capture

Most handlers intercept `console.log` and `console.error` to collect output:

```typescript
const logs: string[] = [];
const origLog = console.log;
const origError = console.error;

console.log = (...a: any[]) => logs.push(a.map(String).join(" "));
console.error = (...a: any[]) => logs.push(a.map(String).join(" "));

try {
  // Handle CLI or API
} catch (e: any) {
  return { ok: false, error: e.message };
} finally {
  console.log = origLog;
  console.error = origError;
}
```

This allows consistent output capture across both CLI and API invocations.

---

## Core Abstractions & Relationships

### 1. Dual-Source Dispatch

Handlers must support two invocation modes:

| Source | args | Example |
|--------|------|---------|
| **CLI** | string[] | `["neo", "--task", "review PR"]` |
| **API** | object | `{ oracle: "neo", task: "review PR" }` |

Handlers branch on `ctx.source` to parse accordingly.

### 2. Command Import Pattern

Handlers use dynamic imports to load core command implementations from maw-js:

```typescript
const { cmdWake } = await import("../../wake");
const { cmdWakeAll } = await import("../../fleet");
const { parseWakeTarget, ensureCloned } = await import("../../wake-target");
```

These commands live in **maw-js/src/commands/** and contain the actual business logic. The plugin layer is purely a dispatch wrapper.

### 3. Flag Parsing

Plugins use `parseFlags` utility (from maw-js/src/cli/parse-args) to extract typed flags:

```typescript
const flags = parseFlags(args, {
  "--new": String,
  "--incubate": String,
  "--issue": Number,
  "--fresh": Boolean,
  "--no-attach": Boolean,
}, 1);  // startIndex = 1 (skip positional)
```

### 4. Lifecycle Relationship

The four plugins form a coherent lifecycle:

- **wake** → create/attach an oracle session (start work)
- **sleep** → pause a single oracle (pause one task)
- **stop** → kill all oracles (emergency stop)
- **done** → finalize a worktree (complete work, cleanup)

---

## Differences from maw-plugins

### maw-core-plugins

- **Scope**: 4 essential lifecycle commands (wake, sleep, stop, done)
- **Distribution**: Ships with maw-js (always present)
- **Structure**: Minimal — plugin.json + index.ts + test only
- **Dependencies**: Tightly integrated with maw-js internals
- **Versioning**: Synced with maw-js release cycle

### maw-plugins (optional monorepo)

- **Scope**: Larger collection of user-installable plugins
- **Distribution**: `maw plugin install <name>` (registry)
- **Structure**: Each package typically has its own package.json
- **Dependencies**: Looser coupling; uses public plugin SDK
- **Versioning**: Independent semver per plugin

---

## Testing

Tests use **Bun's test framework** with `mock.module()` for dependency injection:

```typescript
import { describe, it, expect, mock, beforeEach } from "bun:test";
import { join } from "path";

const src = join(import.meta.dir, "../../..");

mock.module(join(src, "commands/wake"), () => ({
  cmdWake: async (oracle: string, opts: any) => {
    console.log(`woke ${oracle}`);
  },
  // ... more mocks
}));

describe("wake plugin", () => {
  it("CLI basic: wake <name> → calls cmdWake with oracle name", async () => {
    const result = await handler({ source: "cli", args: ["neo"] });
    expect(result.ok).toBe(true);
  });
});
```

### Known Test Limitation

Bun 1.3's `mock.module()` does not re-intercept dynamic imports when the module is already cached by another test file. Individual plugin tests pass in isolation (`bun test packages/00-wake/wake.test.ts`), but may flake in a combined suite (`bun test`). This is a **Bun limitation**, not a code bug; the live commands work correctly.

---

## Type System

All core types are defined in **maw-js/src/plugin/types.ts**:

- **PluginManifest** — the plugin.json schema
- **PluginTier** — "core" | "standard" | "extra"
- **PluginTarget** — "js" | "wasm" (Phase A: js only; wasm reserved for Phase C)
- **InvokeContext** — handler input contract
- **InvokeResult** — handler output contract
- **LoadedPlugin** — runtime plugin descriptor (path, manifest, kind)

Manifest parsing/validation is handled by maw-js:
- **manifest-parse.ts** — parseManifestInput()
- **manifest-validate.ts** — field validators (cli, api, hooks, etc.)
- **manifest-constants.ts** — NAME_RE, SEMVER_RE, SEMVER_RANGE_RE

---

## Entry Points & Build

### During Development

Plugins reference TypeScript sources directly:

```json
"entry": "./index.ts"
```

Bun natively compiles TS on import.

### For Distribution

When a plugin is built (`maw plugin build`), the entry point may be transpiled to JavaScript and an `artifact` field added:

```json
"artifact": {
  "path": "dist/index.js",
  "sha256": "abc123..."
}
```

The manifest parser validates that:
1. The entry/wasm file exists on disk
2. Either entry or wasm is declared (both optional; fallback to ./src/index.ts)
3. sha256 is non-null for built artifacts (null = unbuilt, loader rejects with "run `maw plugin build`")

---

## Plugin Loading & Dispatch

The maw-js runtime loads all plugins via **registry.ts**:

1. **Scan** — find all plugin.json files
2. **Parse** — validate manifest per PluginManifest schema
3. **Weight sort** — order by weight (lower first; default 50)
4. **Register** — CLI dispatch, API routes, lifecycle hooks
5. **Invoke** — call handler on dispatch (CLI command or REST route)

Core plugins (weight 00) load **before** standard plugins, ensuring lifecycle primitives are always available.

---

## Summary

**maw-core-plugins** is a minimal, tightly-coupled set of lifecycle plugins that:

- Use a **simple manifest schema** (plugin.json) shared across all plugins
- Implement a **dual-source handler pattern** (CLI + API)
- Delegate business logic to **maw-js command modules** (cmdWake, cmdSleepOne, etc.)
- Load **first** (weight 00) to ensure availability before user plugins
- Ship **automatically** with maw-js (no installation required)
- Are **tested with Bun** using mock.module for dependency injection
- Serve as **agent lifecycle primitives** (wake, sleep, stop, done)

The architecture reflects a clean separation: manifests declare surface (CLI/API), handlers bridge user input to core commands, and maw-js manages loading, dispatch, and integration.
