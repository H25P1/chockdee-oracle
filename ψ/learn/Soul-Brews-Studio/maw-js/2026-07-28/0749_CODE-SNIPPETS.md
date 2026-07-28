# maw-js Code Architecture & Patterns

**Repository**: Soul-Brews-Studio/maw-js  
**Description**: Bun/TypeScript CLI for running multiple AI agents across machines via tmux orchestration  
**Entry Point**: `src/cli.ts`  
**Package Manager**: Bun + npm workspaces

---

## Table of Contents

1. [CLI Bootstrap & Entry Point](#cli-bootstrap--entry-point)
2. [Command Dispatch System](#command-dispatch-system)
3. [Plugin System & Loading](#plugin-system--loading)
4. [tmux Integration](#tmux-integration)
5. [Async Patterns & SDK](#async-patterns--sdk)
6. [Error Handling Strategy](#error-handling-strategy)
7. [Configuration System](#configuration-system)
8. [Top-Level Aliases & Routing](#top-level-aliases--routing)
9. [Representative Command Handlers](#representative-command-handlers)
10. [Notable Patterns & Idioms](#notable-patterns--idioms)

---

## CLI Bootstrap & Entry Point

### File: `src/cli.ts`

The main entry point is a Bun script that orchestrates plugin loading, command parsing, and dispatch.

```typescript
#!/usr/bin/env bun
process.env.MAW_CLI = "1";

// #566: apply --as <name> BEFORE any state-touching import (paths.ts evaluates
// MAW_HOME at module load). Must be the first side effect.
import { applyInstancePreset } from "./cli/instance-preset";
applyInstancePreset();

import { logAudit } from "./core/fleet/audit";
import { usage } from "./cli/usage";
import { scanCommands } from "./cli/command-registry";
import { setVerbosityFlags } from "./cli/verbosity";
import { getVersionString } from "./cli/cmd-version";
import { runUpdate } from "./cli/cmd-update";
import { runBootstrap } from "./cli/plugin-bootstrap";
import { maybeAutoRestore } from "./cli/auto-restore";
import { dispatchCommand } from "./cli/dispatch";
import { handleTopLevelError } from "./cli/error-handler";
import { mawDataPath } from "./core/xdg";

// Strip verbosity flags up-front so they don't collide with cmd detection or
// leak into plugin argv. Task #3 will flip call sites to honor these.
const VERBOSITY_FLAGS = new Set(["--quiet", "-q", "--silent", "-s"]);
const rawArgs = process.argv.slice(2);
const verbosity: { quiet?: boolean; silent?: boolean } = {};
if (rawArgs.some(a => a === "--quiet" || a === "-q")) verbosity.quiet = true;
if (rawArgs.some(a => a === "--silent" || a === "-s")) verbosity.silent = true;
setVerbosityFlags(verbosity);
const args = rawArgs.filter(a => !VERBOSITY_FLAGS.has(a));
const cmd = args[0]?.toLowerCase();

logAudit(cmd || "", args);

async function main(): Promise<void> {
  if (cmd === "--version" || cmd === "-v" || cmd === "version") {
    console.log(getVersionString());
    return;
  }
  if (cmd === "update" || cmd === "upgrade") {
    await runUpdate(args);
    return;
  }

  // Auto-bootstrap: if the XDG data plugin dir is empty, symlink bundled +
  // install from pluginSources. In legacy mode this still resolves to
  // ~/.maw/plugins; in MAW_XDG=1 it resolves to ~/.local/share/maw/plugins.
  // import.meta.dir must resolve to src/ — keep the call here, not in a child module.
  const pluginDir = process.env.MAW_PLUGINS_DIR || mawDataPath("plugins");
  await runBootstrap(pluginDir, import.meta.dir);

  // Load plugins from the resolved data plugin dir — the single source of truth.
  await scanCommands(pluginDir, "user");

  await maybeAutoRestore(cmd);

  if (!cmd || cmd === "--help" || cmd === "-h") {
    usage();
    return;
  }

  await dispatchCommand(cmd, args);
}

main().catch((e: unknown) => handleTopLevelError(e, args));
```

**Key Bootstrap Steps**:
1. **Instance preset** applied first to set `MAW_HOME` before path evaluation
2. **Verbosity flags** extracted early to prevent collision with arg parsing
3. **Plugin bootstrap** initializes plugin directory (symlink bundled + network fetch on first install)
4. **Command scan** discovers plugins from the plugin directory
5. **Auto-restore** (optional recovery mechanism)
6. **Command dispatch** routes the parsed command to appropriate handler

---

## Command Dispatch System

### File: `src/cli/dispatch.ts`

The dispatch system implements a multi-layer routing ladder:

```typescript
/**
 * Run a command after plugins have been scanned. Walks the dispatch ladder:
 *   routeComm → routeTools → top-aliases → plugin registry (beta) →
 *   bundled plugin registry → agent-name shorthand.
 *
 * Throws UserError for ambiguous/unknown commands. Exits the process on
 * successful plugin invocation (preserves prior behavior).
 */
export async function dispatchCommand(cmd: string, args: string[]): Promise<void> {
  const handled =
    (await routeComm(cmd, args)) ||
    (await routeTools(cmd, args));
  if (handled) return;

  // RFC #954 — top-level verb aliases. Sits between routeTools and
  // matchCommand. Either rewrites argv in place (continue dispatch flow)
  // or dispatches a direct-handler and exits the pipeline.
  const { resolveTopAlias, invokeDirectHandler } = await import("./top-aliases");
  const aliasResult = resolveTopAlias(args);
  if (aliasResult) {
    if (aliasResult.kind === "direct") {
      await invokeDirectHandler(aliasResult.handler, aliasResult.argv);
      return;
    }
    args.splice(0, args.length, ...aliasResult.argv);
  }

  // Try plugin commands (beta) — after core routes, before fallback
  const pluginMatch = matchCommand(args);
  if (pluginMatch) {
    await executeCommand(pluginMatch.desc, pluginMatch.remaining);
    return;
  }

  // Fallback: check plugin registry for bundled commands
  await dispatchPluginRegistry(cmd, args);
}
```

**Dispatch Ladder** (in order):
1. `routeComm` — communication routes (hey, send, notify)
2. `routeTools` — server/agent tools
3. **Top aliases** — verb shortcuts (wake, a, t, ls, etc.)
4. **Plugin commands** — new command registry system (beta)
5. **Plugin registry** — bundled + installed plugins
6. **Agent shorthand** — `maw <oracle-name>` falls back to sending a message

**Plugin Dependency Checking**:
```typescript
const { dependencyStatus } = await import("../plugin/dependencies");
const deps = dependencyStatus(dispatch.plugin, plugins);
if (deps.missing.length > 0) {
  console.error(`✗ '${dispatch.matchedName}' needs missing plugins: ${deps.missing.join(", ")}`);
  throw new UserError(`missing plugin dependency: ${dispatch.matchedName}`);
}
```

---

## Plugin System & Loading

### File: `src/cli/plugin-bootstrap.ts`

The bootstrap process is **idempotent** and runs every boot:

```typescript
/**
 * Auto-bootstrap plugins into pluginDir.
 *
 * Bundled-plugin symlinks are idempotent — walked on every boot so newly
 * added bundled plugins (e.g. introduced by an update) get linked into
 * existing installs. Existing destinations (symlinks or user dirs) are
 * never overwritten.
 *
 * The pluginSources URL fetch path is preserved as first-install only:
 * it makes network calls and has a different cost profile, so it still
 * runs only when pluginDir is empty.
 *
 * Bug: #817 — bootstrap-on-empty caused new bundled plugins to be
 * silently invisible on every existing host until a manual symlink.
 *
 * @param pluginDir  resolved ~/.maw/plugins/ path
 * @param srcDir     resolved src/ directory (pass import.meta.dir from cli.ts)
 */
export async function runBootstrap(pluginDir: string, srcDir: string): Promise<void> {
  mkdirSync(pluginDir, { recursive: true });

  // 0. #1015 — prune broken symlinks before anything else.
  const bundledRoots = [
    join(srcDir, "commands", "plugins"),
    join(srcDir, "vendor", "mpr-plugins"),
    join(srcDir, "vendor-plugins"),
  ];
  const { pruned } = healOrPruneBrokenSymlinks(pluginDir, bundledRoots);
  if (pruned > 0) {
    console.warn(`[maw] removed ${pruned} broken plugin symlink${pruned === 1 ? "" : "s"} from ${pluginDir}`);
  }

  const wasEmpty = readdirSync(pluginDir).length === 0;

  // 1. Symlink any bundled plugin missing from pluginDir — IDEMPOTENT,
  //    runs every boot. Cheap (fs stat + symlink), no network.
  linkBundledPlugins(pluginDir, bundledRoots[0]);
  linkBundledPlugins(pluginDir, bundledRoots[1]);
  linkBundledPlugins(pluginDir, bundledRoots[2]);

  // 2. Install from pluginSources URLs — first-install only (network calls,
  //    should not retry every boot).
  if (wasEmpty) {
    try {
      const { loadConfig } = await import("../config");
      const config = loadConfig();
      const sources: string[] = config.pluginSources ?? [];
      for (const url of sources) {
        try {
          if (!URL_SCHEME_RE.test(url)) {
            warn(`[maw] skipping pluginSource with invalid scheme: ${url}`);
            continue;
          }
          const ghqProc = Bun.spawn(["ghq", "get", "-u", url], { stdout: "pipe", stderr: "pipe" });
          await ghqProc.exited;
          // ... extract and copy plugin packages ...
        } catch {}
      }
    } catch {}

    info(`[maw] bootstrapped ${readdirSync(pluginDir).length} plugins → ${pluginDir}`);
  }
}
```

### Plugin Manifest Structure

**File**: `src/plugin/types.ts`

```typescript
export interface PluginManifest {
  name: string;           // unique id, slug-safe /^[a-z0-9-]+$/
  version: string;        // semver e.g. "1.0.0"
  weight?: number;        // execution order: lower = first (default 50, like Drupal)
  tier?: PluginTier;      // membership contract: "core" | "standard" | "extra"
  wasm?: string;          // relative path to .wasm (WASM plugin)
  entry?: string;         // relative path to .ts/.js (TS plugin)
  sdk: string;            // semver range e.g. "^1.0.0"
  target?: PluginTarget;  // compile target (Phase A: "js" only)
  capabilities?: string[];// declared capability strings "namespace:verb"
  dependencies?: {        // other maw plugins this plugin needs before dispatch
    plugins?: string[];
  };
  artifact?: PluginArtifact;
  cli?: {
    command: string;
    aliases?: string[];
    help?: string;
    flags?: Record<string, string>;        // flag name → "boolean"|"string"|"number"
  };
  api?: { path: string; methods: ("GET" | "POST")[]; };
  description?: string;
  author?: string;
}
```

### Command Scanning

**File**: `src/cli/command-registry.ts`

```typescript
/**
 * Command Plugin Registry (beta) — pluggable CLI commands.
 *
 * Drop a .ts/.js file in ~/.oracle/commands/ with:
 *   export const command = { name: "hello", description: "Say hello" };
 *   export default async function(args, flags) { ... }
 *
 * Or drop a .wasm file that exports handle(ptr, len) + memory.
 * Args are passed as JSON in shared memory; output read back from memory.
 */
export async function scanCommands(dir: string, scope: "builtin" | "user"): Promise<number> {
  if (!existsSync(dir)) return 0;
  const disabled = loadConfig().disabledPlugins ?? [];
  let count = 0;
  for (const file of readdirSync(dir).filter(f => /\.(ts|js|wasm)$/.test(f))) {
    try {
      const path = join(dir, file);
      if (file.endsWith(".wasm")) {
        await loadWasmCommand(path, file, scope, disabled);
        count++;
      } else {
        const mod = await import(path);
        if (mod.command?.name) {
          const cmdName = Array.isArray(mod.command.name) ? mod.command.name[0] : mod.command.name;
          if (disabled.includes(cmdName)) continue;
          registerCommand(mod.command, path, scope);
          count++;
        }
      }
    } catch (err: any) {
      console.error(`[commands] failed to load ${file}: ${err.message?.slice(0, 80)}`);
    }
  }
  return count;
}
```

---

## tmux Integration

### File: `src/core/transport/tmux-types.ts`

**Shell quoting utility**:
```typescript
/**
 * Shell-quote a single argument for tmux commands.
 * @internal
 */
export function q(s: string | number): string {
  const str = String(s);
  // Safe chars only → no quoting needed
  if (/^[a-zA-Z0-9_.:\-\/]+$/.test(str)) return str;
  // Wrap in single quotes, escape inner single quotes
  return `'${str.replace(/'/g, "'\\''")}'`;
}
```

**Socket resolution**:
```typescript
/** Resolve tmux socket path from env or config. */
export function resolveSocket(): string | undefined {
  return process.env.MAW_TMUX_SOCKET || loadConfig().tmuxSocket || undefined;
}

/** Build the `tmux` (or `tmux -S <socket>`) prefix for raw commands. */
export function tmuxCmd(): string {
  const socket = resolveSocket();
  return socket ? `tmux -S '${socket}'` : "tmux";
}
```

**Type Definitions**:
```typescript
export interface TmuxPane {
  id: string;
  command: string;
  target: string;
  title: string;
  pid?: number;
  cwd?: string;
  lastActivity?: number;
  top?: number;
  left?: number;
  w?: number;
  h?: number;
  paneIdx?: number;
  winIdx?: number;
  winName?: string;
  active?: boolean;
  window?: { w?: number; h?: number; active?: boolean };
  attached?: boolean;
  attachedClients?: number;
}

export interface TmuxWindow {
  index: number;
  name: string;
  active: boolean;
  cwd?: string;
}

export interface TmuxSession {
  name: string;
  windows: TmuxWindow[];
}
```

### File: `src/core/transport/tmux-class.ts`

The `Tmux` class provides a typed wrapper around tmux CLI commands:

```typescript
/**
 * Typed wrapper around tmux CLI.
 * All methods build arg arrays and delegate to `run()`.
 */
export class Tmux {
  private socket?: string;
  constructor(private host?: string, socket?: string) {
    this.socket = socket !== undefined ? socket : resolveSocket();
  }

  /** Base runner — executes `tmux [-S socket] <subcommand> [args...]` via hostExec. */
  async run(subcommand: string, ...args: (string | number)[]): Promise<string> {
    const socketFlag = this.socket ? `-S ${q(this.socket)} ` : "";
    const needsTermFallback = !process.env.TERM && process.env.MAW_TEST_MODE !== "1";
    const termPrefix = needsTermFallback ? "TERM=xterm " : "";
    const cmd = `${termPrefix}tmux ${socketFlag}${subcommand} ${args.map(q).join(" ")}`;
    return hostExec(cmd, this.host);
  }

  /** Like run() but swallows errors — for best-effort cleanup ops. */
  async tryRun(subcommand: string, ...args: (string | number)[]): Promise<string> {
    return this.run(subcommand, ...args).catch(() => "");
  }

  // Session management
  async listSessions(): Promise<TmuxSession[]> {
    try {
      const raw = await this.run("list-sessions", "-F", "#{session_name}");
      const sessions: TmuxSession[] = [];
      for (const s of raw.split("\n").filter(Boolean)) {
        const windows = await this.listWindows(s);
        sessions.push({ name: s, windows });
      }
      return sessions;
    } catch (error) {
      if (isTmuxBinaryMissingError(error)) throw error;
      if (isTmuxNoServerError(error)) return [];
      return [];
    }
  }

  async hasSession(name: string): Promise<boolean> {
    try {
      await this.run("has-session", "-t", name);
      return true;
    } catch {
      return false;
    }
  }
}
```

**Singleton Instance**:
```typescript
export const tmux = new Tmux();
```

### Send Text with Confirmation

**File**: `src/core/transport/tmux-class.ts` (snippet)

The system sends text, waits, then confirms submission via pane content verification:

```typescript
// --- sendText submit-confirmation tuning (#6) ---
// The old sendText fired 3 blind `Enter` keys on a fixed ~1.9s schedule with
// zero feedback. If the pane wasn't ready when they landed (agent still
// rendering the paste, brief stall), every Enter missed and the command sat
// in the input box unexecuted — this forced manual re-launch of dispatches
// on 2026-05-14. We now send Enter, re-check the pane, and retry only while
// input is still pending.

/** Wait after paste/literal-send before the first Enter — lets the input settle. */
const SEND_SETTLE_MS = 1500;
/** Wait after each Enter before re-checking whether the input line cleared. */
const SUBMIT_CONFIRM_MS = 700;
/** Max Enter attempts before giving up and warning (was 3 blind, unconditional sends). */
const MAX_SUBMIT_ATTEMPTS = 4;
```

---

## Async Patterns & SDK

### File: `src/sdk/index.ts` (excerpt)

The SDK is the public API boundary for plugins. It re-exports stable symbols only:

```typescript
/**
 * @maw-js/sdk — the stable API surface for maw-js plugins.
 *
 * TS plugins import from here. WASM plugins get the same capabilities
 * via host functions in wasm-bridge.ts.
 *
 * Rule: if it's not exported here, plugins shouldn't depend on it.
 * This is the contract boundary between core runtime and plugin code.
 */

// ─── Identity & Config ───────────────────────────────────────────────────────
export { loadConfig } from "../config/load";
export {
  saveConfig, buildCommand, buildCommandInDir,
  getEnvVars, cfgTimeout, cfgLimit, cfgInterval, cfg, D,
  resetConfig,
} from "../config";

// ─── Transport (tmux, SSH, etc.) ───────────────────────────────────────────
export {
  tmux, Tmux, tmuxCmd, resolveSocket,
  withPaneLock, splitWindowLocked, tagPane, readPaneTags,
} from "../core/transport/tmux";
export {
  hostExec, listSessions, capture, sendKeys,
  getPaneCommand, getPaneCommands, getPaneInfos,
  HostExecError,
} from "../core/transport/ssh";
export { attachRemoteSession, SshAttachError } from "../core/transport/ssh-attach";
export { curlFetch } from "../core/transport/curl-fetch";
```

### Representative Command: `cmdSend`

**File**: `src/commands/shared/comm-send.ts` (excerpt)

Demonstrates async patterns, multi-layer resolution, and pane-specific routing:

```typescript
/**
 * Resolve a `session:window` target to a specific pane running an agent
 * (claude / codex / node). Fixes the multi-pane routing bug: when an oracle
 * window has multiple panes (e.g., team-agents split beside it), tmux's
 * `send-keys -t session:window` defaults to the LAST-ACTIVE pane — which
 * becomes whichever teammate just spawned, not the oracle itself.
 *
 * Strategy: list all panes in the window, pick the lowest-index pane
 * running a claude/codex/node process. Pane 0 is conventionally the
 * oracle's main pane (created by `tmux.newWindow` during `maw wake`);
 * team-agents spawn LATER as splits and take higher indexes.
 */
export async function resolveOraclePane(
  target: string,
  deps: {
    tmuxRun?: (...args: string[]) => Promise<string>;
    isAgentCommandFn?: typeof isAgentCommand;
  } = {},
): Promise<string> {
  // Already pane-specific — honor caller's choice.
  if (/\.[0-9]+$/.test(target)) return target;

  try {
    const run = deps.tmuxRun ?? ((...args: string[]) => new Tmux().run(...args));
    const isAgent = deps.isAgentCommandFn ?? isAgentCommand;
    const raw = await run("list-panes", "-t", target, "-F", "#{pane_index} #{pane_current_command}");
    const lines = raw.split("\n").map((l: string) => l.trim()).filter(Boolean);
    if (lines.length <= 1) return target; // single-pane window: active pane is the only pane

    const agentIndexes: number[] = [];
    for (const line of lines) {
      const spaceIdx = line.indexOf(" ");
      if (spaceIdx < 0) continue;
      const idx = parseInt(line.slice(0, spaceIdx), 10);
      const cmd = line.slice(spaceIdx + 1);
      if (Number.isFinite(idx) && isAgent(cmd)) {
        agentIndexes.push(idx);
      }
    }
    if (agentIndexes.length === 0) return target;
    return `${target}.${Math.min(...agentIndexes)}`;
  } catch {
    return target;
  }
}

/** Resolve the current oracle name from CLAUDE_AGENT_NAME or the attached tmux pane. */
export function resolveMyName(config: ReturnType<typeof loadConfig>): string {
  if (process.env.CLAUDE_AGENT_NAME) return process.env.CLAUDE_AGENT_NAME;
  // Only trust tmux when this process is actually running inside a tmux pane.
  if (process.env.TMUX) {
    try {
      const tmuxSession = require("child_process").execSync(
        "tmux display-message -p '#{session_name}'",
        { encoding: "utf-8" }
      ).trim();
      if (tmuxSession) return tmuxSession.replace(/^\d+-/, "");
    } catch {}
  }
  return config.node || "cli";
}

/** Resolve the visible + signed sender for `maw hey`. */
export function parseSenderOverride(raw: string | undefined | null): Pick<SenderIdentity, "node" | "oracle" | "display" | "wireFrom" | "senderName"> | null {
  const value = (raw ?? "").trim();
  if (!value) return null;
  const parts = value.split(":");
  if (parts.length !== 2) return null;
  const [node, oracle] = parts.map((part) => part.trim());
  if (!node || !oracle) return null;
  const SENDER_PART_RE = /^[A-Za-z0-9_.-]+$/;
  if (!SENDER_PART_RE.test(node) || !SENDER_PART_RE.test(oracle)) return null;
  return {
    node,
    oracle,
    display: `${node}:${oracle}`,
    wireFrom: `${oracle}:${node}`,  // Existing from-signing contract is reversed
    senderName: oracle,
  };
}
```

---

## Error Handling Strategy

### File: `src/core/util/user-error.ts`

A **brand-based error pattern** (not class instanceof, which breaks across module boundaries):

```typescript
/**
 * UserError signals a user-facing failure — bad input, missing target,
 * unknown command. The top-level error handler in src/cli.ts catches
 * these and exits 1 WITHOUT letting bun print its default stack trace.
 * For genuinely unexpected runtime failures, throw a regular Error so
 * the stack stays visible for debugging.
 *
 * Convention: throw sites may print richer context (colors, hints,
 * suggestions) before throwing. The top-level catch still prints this
 * message so direct UserError throws never disappear silently.
 *
 * Throw UserError for: missing/invalid args, unknown commands, bad
 *   target resolution, help-path exits.
 * Throw regular Error for: genuinely unexpected runtime failures
 *   where the stack is valuable for debugging.
 *
 * Why a brand field instead of `instanceof UserError`: class identity
 * breaks across module boundaries in ESM (dynamic import, separate
 * realms). The `isUserError` brand survives.
 */
export class UserError extends Error {
  readonly isUserError = true;
  constructor(message: string) {
    super(message);
    this.name = "UserError";
  }
}

export function isUserError(e: unknown): e is UserError {
  return e instanceof Error && (e as { isUserError?: boolean }).isUserError === true;
}
```

### File: `src/cli/error-handler.ts`

Top-level error handler always exits:

```typescript
/**
 * Top-level error handler for `main()`. Always exits — never returns.
 *
 * - UserError: print its message without a bun stack trace, then exit 1.
 * - AmbiguousMatchError: escapes from findWindow via resolver chains.
 *   Render as actionable CLI output instead of a minified stack trace.
 * - Anything else: print the error normally and exit 1.
 */
export function handleTopLevelError(e: unknown, args: string[]): never {
  if (isUserError(e)) {
    if (e.message) process.stderr.write(`${e.message}\n`);
    process.exit(1);
  }
  if (e instanceof AmbiguousMatchError) {
    console.error(renderAmbiguousMatch(e, args));
    process.exit(1);
  }
  console.error(e);
  process.exit(1);
}
```

---

## Configuration System

### File: `src/config/index.ts`

Configuration is loaded from multiple sources via `loadConfig()`:

```typescript
export interface MawConfig {
  node?: string;                    // Local node name for federation
  tmuxSocket?: string;              // Custom tmux socket path
  pluginSources?: string[];         // URLs for plugin installation
  disabledPlugins?: string[];       // Plugin names to skip
  sessions?: Record<string, SessionConfig>;
  agents?: Record<string, AgentConfig>;
  engines?: Record<string, EngineDef>;
  // ... more fields
}
```

**Key patterns**:
- Config is read-only in most plugins (via `loadConfig()`)
- Modifications use `saveConfig()`
- Environment variables can override config values

---

## Top-Level Aliases & Routing

### File: `src/cli/top-aliases.ts`

Aliases use two patterns:

1. **Argv-rewrite** — splices args in place and continues dispatch
2. **Direct-handler** — static-imported function reference

```typescript
/**
 * Top-level verb aliases — RFC #954 (Axis 2: help-prominence / verb routing).
 *
 * Single source of truth for short verbs that route directly without going
 * through the plugin dispatcher. Inserted between `routeTools` and
 * `matchCommand` in src/cli.ts.
 *
 * Two forms:
 *   1. Argv-rewrite — splice `args` in place, continue normal dispatch
 *      Example: `maw a foo` → `maw tmux attach foo` (handled by tmux plugin)
 *   2. Direct-handler — static-imported function reference
 *      Example: `maw wake foo` → cmdWake(foo, opts) directly
 *
 * One-shot only — aliases NEVER expand into another alias.
 *
 * IMPORTANT: handlers are STATIC imports, not dynamic. When this file is
 * bundled into src/cli.ts via bun build, dynamic `import("../commands/...")`
 * paths get resolved relative to the bundled cli.ts (one dir up from where
 * the source lives), which breaks at runtime. Static imports are inlined by
 * the bundler, sidestepping the resolution context mismatch entirely.
 */

import { cmdWake } from "../commands/shared/wake-cmd";
import { cmdPromote } from "../commands/shared/promote-cmd";
import { cmdTeamWtf } from "../commands/plugins/team/team-wtf";

export const TOP_ALIASES: Record<string, string[] | DirectHandler> = {
  // Argv-rewrite form — canonical handler lives in a core plugin
  a: ["attach"],
  kill: ["tmux", "kill"],
  split: ["split"],
  open: ["tmux", "open"],
  close: ["tmux", "close"],
  t: ["team"],
  zoom: ["tmux", "zoom"],
  panes: ["tmux", "ls", "--all", "--verbose"],
  tile: ["tile"],

  // Direct-handler form
  layout: { kind: "direct", handler: "cmdLayout" },
  bring: { kind: "direct", handler: "../commands/shared/wake-cmd:cmdBring" },
  b: { kind: "direct", handler: "../commands/shared/wake-cmd:cmdBring" },
  ls: { kind: "direct", handler: "cmdLs" },
  wake: { kind: "direct", handler: "../commands/shared/wake-cmd:cmdWake" },
  awake: { kind: "direct", handler: "../commands/shared/wake-cmd:cmdAwake" },
  work: { kind: "direct", handler: "../commands/shared/wake-cmd:cmdWake", argv: ["--work", "."] },
  new: { kind: "direct", handler: "./cmd-new:cmdNew" },
  promote: { kind: "direct", handler: "../commands/shared/promote-cmd:cmdPromote" },
  preflight: { kind: "direct", handler: "../commands/shared/preflight:cmdPreflight" },
};
```

---

## Representative Command Handlers

### Oracle List Implementation

**File**: `src/commands/plugins/oracle/impl-list.ts` (excerpt)

Demonstrates caching, manifest-based inventory, and enrichment:

```typescript
/**
 * maw oracle ls — cached grouped inventory enriched with source-lineage
 * and runtime awake state. Replaces the old awake-only list + fleet view.
 *
 * Source of truth: `OracleManifest` (#838) — a unified read-only view that
 * aggregates the 5 oracle registries (fleet windows, config.sessions,
 * config.agents, oracles.json, worktree). Sub-PR 1 of #841.
 *
 * `oracles.json` cache is still read — it's the only source for `local_path`,
 * `org`, `repo` (split form), and lineage timestamps. We auto-refresh the cache
 * on stale/missing as before so first-run UX stays unchanged.
 *
 * Flags:
 *   --awake   filter to running tmux sessions only
 *   --org X   filter to org X
 *   --json    machine output
 *   --scan    refresh cache before listing
 *   --stale   skip auto-refresh on stale cache
 *   --path    show local filesystem paths
 *   --sort-by born  sort newest-known births first (#1806)
 */

export interface OracleListOpts {
  awake?: boolean;
  org?: string;
  json?: boolean;
  scan?: boolean;
  stale?: boolean;
  path?: boolean;
  sortBy?: "born" | string;
}

export async function buildEnrichedEntries(opts: { scan?: boolean; stale?: boolean; json?: boolean } = {}): Promise<EnrichedEntry[]> {
  const config = loadConfig();
  const agents = config.agents || {};

  let cache = readCache();
  const shouldRefresh =
    !!opts.scan || !cache || (isCacheStale(cache) && !opts.stale);
  if (shouldRefresh) {
    if (!cache && !opts.json) {
      console.log(
        `\n  \x1b[33m📡\x1b[0m No oracle cache — running first local scan...\n`,
      );
    }
    cache = scanAndCache("local");
    invalidateManifest();
  }

  // Fetch live sessions from tmux
  const sessions = await listSessions().catch(() => []);
  const awakeByName = new Map<string, string>();
  for (const s of sessions) {
    for (const w of s.windows) {
      if (w.name.endsWith("-oracle")) {
        const name = w.name.replace(/-oracle$/, "");
        if (!awakeByName.has(name)) awakeByName.set(name, s.name);
      }
    }
  }

  // Unify from manifest (single source of truth) + oracles.json cache
  const manifest = loadManifestCached();
  const cacheByName = new Map<string, OracleEntry>(
    (cache?.oracles ?? []).map((e) => [e.name, e]),
  );
  const now = new Date().toISOString();

  const manifestNames = new Set<string>();
  const entries: OracleEntry[] = [];
  const sourcesByName = new Map<string, string[]>();

  // ... build enriched entries with birth timestamps, lineage, etc. ...
}
```

### Argument Parsing

**File**: `src/cli/parse-args.ts`

Wraps the `arg` library with permissive mode:

```typescript
/**
 * Shared CLI argument parsing via `arg`.
 *
 * Each command defines its flag spec. `parseFlags` wraps arg() with:
 * - permissive mode (unknown flags don't throw — they go to argv._)
 * - sliced argv (strips "maw" + command name)
 *
 * Usage:
 *   const flags = parseFlags(args, { "--verbose": Boolean, "-v": "--verbose" }, 2);
 *   flags["--verbose"]  // boolean | undefined
 *   flags._             // positional args
 */

import arg from "arg";

export function parseFlags<T extends arg.Spec>(
  args: string[],
  spec: T,
  skip = 0,
): arg.Result<T> {
  return arg(spec, { argv: args.slice(skip), permissive: true });
}
```

---

## Notable Patterns & Idioms

### 1. **Bun-Specific APIs**

**Process spawning with Bun.spawn** (vs Node's child_process):
```typescript
const ghqProc = Bun.spawn(["ghq", "get", "-u", url], { stdout: "pipe", stderr: "pipe" });
await ghqProc.exited;
```

**Dynamic imports with import.meta.dir**:
```typescript
const pluginDir = process.env.MAW_PLUGINS_DIR || mawDataPath("plugins");
await runBootstrap(pluginDir, import.meta.dir);
```

### 2. **Import Layering for Bundler Safety**

Static vs dynamic imports matter in bun build:
```typescript
// ❌ BREAKS during bundling (resolves relative to bundled location)
import { cmdWake } from await import("../commands/shared/wake-cmd");

// ✅ WORKS (inlined by bundler, context-independent)
import { cmdWake } from "../commands/shared/wake-cmd";

// ✅ OK when NOT bundled into cli.ts (e.g., dispatch.ts)
const { resolveTopAlias } = await import("./top-aliases");
```

### 3. **Idempotent Bootstrap**

Symlink setup is walked every boot (cheap fs operations) so new bundled plugins get linked automatically:
```typescript
linkBundledPlugins(pluginDir, bundledRoots[0]);  // Runs every boot, fast
if (wasEmpty) {
  // Network fetch — runs only on fresh install
  for (const url of config.pluginSources ?? []) { /* ... */ }
}
```

### 4. **Broken Symlink Healing**

Automatic cleanup of dangling symlinks when bundled plugins are removed:
```typescript
function healOrPruneBrokenSymlinks(pluginDir: string, bundledRoots: string[]): { healed: number; pruned: number } {
  for (const entry of readdirSync(pluginDir)) {
    const p = join(pluginDir, entry);
    if (!lstatSync(p).isSymbolicLink()) continue;
    const targetIsValidPlugin = existsSync(p) && isPluginDir(p);
    if (targetIsValidPlugin && !shouldHeal) continue;
    unlinkSync(p);
    if (replacement) {
      symlinkSync(replacement, p);  // Heal with new target
    } else {
      // Pruned — no bundled replacement exists
    }
  }
}
```

### 5. **Shell Quoting**

Safe argument escaping for tmux/shell commands:
```typescript
export function q(s: string | number): string {
  const str = String(s);
  if (/^[a-zA-Z0-9_.:\-\/]+$/.test(str)) return str;  // No quoting needed
  return `'${str.replace(/'/g, "'\\''")}'`;  // Single-quote escape
}
```

### 6. **Async Error Propagation**

Graceful degradation with `.catch(() => [])`:
```typescript
const sessions = await listSessions().catch(() => []);
// If tmux is unavailable, continue with empty list

const json = await new Response(rootProc.stdout).text();
// Bun's Response API for stream handling
```

### 7. **Multi-Layer Target Resolution**

Pane ID → session:w.p → team agent → fleet stem → bare session name:
```typescript
export function resolveTmuxTarget(target: string): { resolved: string; source: string } | null {
  // 1. Pane ID
  if (/^%\d+$/.test(target)) return { resolved: target, source: "pane-id" };
  
  // 2. session:w.p
  if (/^[\w.-]+:\d+\.\d+$/.test(target)) return { resolved: target, source: "session:w.p" };
  
  // 3. Team agent name
  if (existsSync(TEAMS_DIR)) { /* search ~/.claude/teams/*/config.json */ }
  
  // 3.5 — Fleet session by stem
  const entries = loadFleetEntries();
  const exact = sessions.find(s => s.name.toLowerCase() === target.trim().toLowerCase());
  if (exact) return { resolved: exact.name, source: `fleet-stem (${exact.name})` };
  
  // 4. Bare session name
  return { resolved: target, source: "bare-session" };
}
```

### 8. **Dependency Checking Before Dispatch**

Plugins can declare required plugins; dispatch validates before invoking:
```typescript
const { dependencyStatus } = await import("../plugin/dependencies");
const deps = dependencyStatus(dispatch.plugin, plugins);
if (deps.missing.length > 0) {
  throw new UserError(`missing plugin dependency: ${dispatch.matchedName}`);
}
if (deps.disabled.length > 0) {
  console.error(`Run: maw plugin enable ${deps.disabled.join(" ")}`);
  throw new UserError(`disabled plugin dependency: ${dispatch.matchedName}`);
}
```

### 9. **Prefix Auto-Complete**

Single unique prefix matches are expanded:
```typescript
const prefixMatches = knownCommands.filter(n => n.toLowerCase().startsWith(cmd) && n.toLowerCase() !== cmd);
const uniquePrefixes = [...new Set(prefixMatches.map(n => n.toLowerCase()))];
if (uniquePrefixes.length === 1) {
  args.splice(0, 1, uniquePrefixes[0]);  // Rewrite and retry
}
```

### 10. **Manifest-Driven Inventory**

Single unified source of truth aggregating 5 oracle registries (fleet, config.sessions, config.agents, oracles.json, worktree):
```typescript
/**
 * Build a renderer-compatible `OracleEntry` from a manifest entry, layering
 * any oracles.json metadata we already have on top. Manifest covers the
 * "this oracle exists" fact; `cache.oracles` covers the org/repo/local_path
 * detail (only registry that knows the filesystem path).
 */
function buildEntryFromManifest(
  m: OracleManifestEntry,
  cacheByName: Map<string, OracleEntry>,
  fallbackNode: string | null,
  detectedAt: string,
): OracleEntry { /* ... */ }
```

---

## Architecture Summary

| Layer | Files | Purpose |
|-------|-------|---------|
| **CLI Bootstrap** | `src/cli.ts` | Entry point, plugin loading, verbosity setup |
| **Dispatch** | `src/cli/dispatch.ts` | Command routing ladder (routeComm → routeTools → aliases → plugins) |
| **Plugin System** | `src/cli/plugin-bootstrap.ts`, `src/cli/command-registry.ts` | Symlink idempotency, plugin discovery, dependency validation |
| **Transport** | `src/core/transport/tmux-*.ts`, `src/core/transport/ssh.ts` | tmux/SSH wrapper classes, command execution, pane resolution |
| **SDK** | `src/sdk/index.ts` | Stable public API for plugins (read-only re-exports) |
| **Config** | `src/config/index.ts` | Load/save application config |
| **Error Handling** | `src/cli/error-handler.ts`, `src/core/util/user-error.ts` | Brand-based UserError, top-level catch-all |
| **Aliases** | `src/cli/top-aliases.ts` | Verb shortcuts (wake, a, t, ls) with static + direct handlers |
| **Commands** | `src/commands/plugins/*`, `src/commands/shared/*` | Representative handlers (oracle list, send, team management) |

---

## Key Takeaways

1. **Plugin-centric architecture** — most commands live in `src/commands/plugins/` and are loaded dynamically
2. **Idempotent bootstrap** — symlink walk runs every boot; network fetch only on first install
3. **Dispatch ladder** — multi-layer routing prevents ambiguity and provides fallbacks
4. **Brand-based errors** — `UserError` uses a brand field to survive ESM module boundaries
5. **Tmux as orchestration layer** — all cross-machine agent control flows through tmux sessions/panes
6. **Async-first patterns** — widespread use of `async/await` with graceful `.catch()` degradation
7. **Static import discipline** — bundler safety requires careful distinction between static and dynamic imports
8. **Manifest-driven inventory** — single unified view of oracles aggregates 5 sources
9. **Shell quoting utility** — `q()` function handles safe arg escaping for tmux/shell
10. **Dependency tracking** — plugins declare requirements; dispatch validates before invocation
