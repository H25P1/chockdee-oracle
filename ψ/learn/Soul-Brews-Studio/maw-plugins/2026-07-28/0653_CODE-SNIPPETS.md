# maw-plugins Codebase Documentation

> **maw-plugins**: Installable CLI, API, and hook plugins for maw-js (a terminal multiplexing/fleet-management tool for AI agent oracles). Ship-tier WASM artifacts with sha256 pins.

Repository: https://github.com/Soul-Brews-Studio/maw-plugins

---

## 1. Project Overview

**Structure:**
- Monorepo with yarn workspaces (`packages/*`)
- Weight-prefixed directories (e.g., `20-contacts`, `50-artifact-manager`) determine execution order
- Three plugin tiers:
  - **Ship tier**: WASM (AssemblyScript, `wasm32-unknown-unknown`)
  - **Bun-dev tier**: TypeScript/Bun (no WASM)
  - **Native**: Swift helpers (e.g., `maw-menubar`)

**Package.json (Root):**

```json
{
  "name": "maw-plugins",
  "private": true,
  "version": "0.1.0",
  "description": "maw-js plugin packages — installable CLI, API, and hook plugins",
  "workspaces": ["packages/*"],
  "license": "BUSL-1.1",
  "scripts": {
    "setup": "bash scripts/setup.sh",
    "test": "bash scripts/test-per-package.sh",
    "registry": "bun run scripts/gen-registry.ts",
    "registry:check": "bun run scripts/gen-registry.ts --check"
  }
}
```

---

## 2. Plugin Manifest & Entry Points

### Manifest Structure (`plugin.json`)

All plugins declare a **plugin.json** manifest with:

```json
{
  "name": "contacts",
  "version": "1.0.0",
  "sdk": "^1.0.0",
  "description": "Manage oracle contacts — add, remove, list.",
  "author": "Soul-Brews-Studio",
  "license": "BUSL-1.1",
  "weight": 20,
  "schemaVersion": 1,
  "target": "wasm",
  "entry": {
    "export": "handle",
    "kind": "wasm",
    "path": "plugin.wasm"
  },
  "artifact": {
    "path": "./plugin.wasm",
    "sha256": "sha256:2d1ffdf7c7fa053cb4e51d1c3d7db3b9b01f4339c9eab76f167d60414fa7cb11"
  },
  "capabilities": [
    "fs:read:psi",
    "fs:write:psi"
  ],
  "cli": {
    "command": "contacts",
    "aliases": ["contact"],
    "help": "maw contacts [ls|add|rm] [name] [--maw V] [--thread V] [--inbox V] [--repo V] [--notes V]"
  }
}
```

**Key fields:**
- `weight`: Execution order (lower fires first). Common values: 20 (tools), 50 (features)
- `artifact.sha256`: Immutable pin for committed WASM artifact
- `capabilities`: Declared permissions (e.g., `fs:read:psi`, `tmux:send`, `proc:exec:git`)
- `cli`: Command-line surface (verb, aliases, help text)
- `entry`: Points to handler export (e.g., `handle` function in WASM or TypeScript)

---

## 3. Plugin Types & Implementations

### Type A: CLI/API Plugin (Bun-dev, TypeScript)

**Example: `20-contacts`**

**File structure:**
```
packages/20-contacts/
├── plugin.json
├── package.json
├── index.ts        (entry point / handler)
├── impl.ts         (implementation logic)
└── contacts.test.ts
```

**Entry point** (`index.ts`):

```typescript
import type { InvokeContext, InvokeResult } from "maw-js/sdk";
import { cmdContactsLs, cmdContactsAdd, cmdContactsRm } from "./impl";

export const command = {
  name: ["contacts", "contact"],
  description: "Manage oracle contacts — add, remove, list",
};

export default async function handler(ctx: InvokeContext): Promise<InvokeResult> {
  const logs: string[] = [];
  const origLog = console.log;
  const origError = console.error;
  console.log = (...a: any[]) => logs.push(a.map(String).join(" "));
  console.error = (...a: any[]) => logs.push(a.map(String).join(" "));
  try {
    if (ctx.source === "cli") {
      const args = ctx.args as string[];
      const sub = args[0]?.toLowerCase();
      if (sub === "add" && args[1]) {
        await cmdContactsAdd(args[1], args.slice(2));
      } else if (sub === "rm" || sub === "remove") {
        if (!args[1]) { logs.push("usage: maw contacts rm <name>"); return { ok: false, error: "name required" }; }
        await cmdContactsRm(args[1]);
      } else {
        await cmdContactsLs();
      }
    } else if (ctx.source === "api") {
      const body = ctx.args as Record<string, unknown>;
      const method = body.method as string | undefined;
      if (!method || method === "GET") {
        await cmdContactsLs();
      } else if (method === "POST") {
        const action = body.action as string;
        const name = body.name as string;
        if (!name) return { ok: false, error: "name required" };
        if (action === "add") {
          const transport = body.transport as string | undefined;
          await cmdContactsAdd(name, transport ? ["--maw", transport] : []);
        } else if (action === "rm") {
          await cmdContactsRm(name);
        } else {
          return { ok: false, error: `unknown action: ${action}` };
        }
      }
    } else {
      await cmdContactsLs();
    }

    return { ok: true, output: logs.join("\n") || undefined };
  } catch (e: any) {
    return { ok: false, error: e.message };
  } finally {
    console.log = origLog;
    console.error = origError;
  }
}
```

**Key pattern:**
- Handler intercepts console output into `logs` array
- Dispatches on `ctx.source` (CLI vs. API)
- Returns `InvokeResult` (typed contract)
- Error handling: try/catch + finally to restore console

**Implementation** (`impl.ts`):

```typescript
import { readFileSync, writeFileSync, existsSync, mkdirSync } from "fs";
import { join } from "path";
import { loadConfig, parseFlags } from "maw-js/sdk";

interface Contact {
  maw?: string;
  thread?: string;
  inbox?: string | null;
  repo?: string | null;
  notes?: string;
  retired?: boolean;
}

interface ContactsFile {
  contacts: Record<string, Contact>;
  updated: string;
}

function resolvePsiPath(): string {
  const config = loadConfig();
  if (config.psiPath) return config.psiPath;
  const cwd = process.cwd();
  if (existsSync(join(cwd, "ψ"))) return join(cwd, "ψ");
  return join(cwd, "psi");
}

function loadContacts(): ContactsFile {
  const path = join(resolvePsiPath(), "contacts.json");
  if (!existsSync(path)) return { contacts: {}, updated: new Date().toISOString() };
  return JSON.parse(readFileSync(path, "utf-8"));
}

function saveContacts(data: ContactsFile) {
  const psi = resolvePsiPath();
  mkdirSync(psi, { recursive: true });
  data.updated = new Date().toISOString();
  writeFileSync(join(psi, "contacts.json"), JSON.stringify(data, null, 2) + "\n");
}

export async function cmdContactsLs() {
  const { contacts } = loadContacts();
  const active = Object.entries(contacts).filter(([, c]) => !c.retired);
  if (!active.length) { console.log("\x1b[90mno contacts\x1b[0m"); return; }
  console.log(`\n\x1b[36mCONTACTS\x1b[0m (${active.length}):\n`);
  for (const [name, c] of active) {
    const maw = c.maw ? `maw: \x1b[33m${c.maw}\x1b[0m` : "";
    const thread = c.thread ? `thread: \x1b[90m${c.thread}\x1b[0m` : "";
    const inbox = c.inbox ? `inbox: \x1b[90m${c.inbox}\x1b[0m` : "";
    const repo = c.repo ? `repo: \x1b[90m${c.repo}\x1b[0m` : "";
    const notes = c.notes ? `\x1b[90m"${c.notes}"\x1b[0m` : "";
    const parts = [maw, thread, inbox, repo, notes].filter(Boolean).join("    ");
    console.log(`  \x1b[32m${name.padEnd(12)}\x1b[0m  ${parts}`);
  }
  console.log();
}

export async function cmdContactsAdd(name: string, args: string[]) {
  const data = loadContacts();
  const c: Contact = data.contacts[name] || {};
  const flags = parseFlags(args, {
    "--maw": String,
    "--thread": String,
    "--inbox": String,
    "--repo": String,
    "--notes": String,
  }, 0);
  if (flags["--maw"]) c.maw = flags["--maw"];
  if (flags["--thread"]) c.thread = flags["--thread"];
  if (flags["--inbox"]) c.inbox = flags["--inbox"];
  if (flags["--repo"]) c.repo = flags["--repo"];
  if (flags["--notes"]) c.notes = flags["--notes"];
  if (c.retired) delete c.retired;
  data.contacts[name] = c;
  saveContacts(data);
  console.log(`\x1b[32m✓\x1b[0m contact \x1b[33m${name}\x1b[0m saved`);
}

export async function cmdContactsRm(name: string) {
  const data = loadContacts();
  if (!data.contacts[name]) { console.error(`\x1b[31merror\x1b[0m: contact '${name}' not found`); return; }
  data.contacts[name].retired = true;
  saveContacts(data);
  console.log(`\x1b[32m✓\x1b[0m contact \x1b[33m${name}\x1b[0m retired`);
}
```

**Key patterns:**
- Uses SDK utilities: `loadConfig()`, `parseFlags()`
- Soft-deletes via `retired` flag (immutable append model)
- ANSI color codes for terminal output
- Plain JSON for state (no binary serialization)

---

### Type B: WASM Plugin (AssemblyScript)

**Example: `cross-team-queue`**

**Manifest** (`plugin.source.json`):

```json
{
  "name": "cross-team-queue",
  "version": "0.1.0",
  "sdk": "^1.0.0",
  "description": "Unified inbox queue scanner. Ship-tier WASM reads the configured vault via fs:read:vault and returns the queue contract.",
  "author": "Soul-Brews-Studio",
  "license": "BUSL-1.1",
  "weight": 50,
  "schemaVersion": 1,
  "entry": "src/plugin.ts",
  "capabilities": [
    "fs:read:vault"
  ],
  "cli": {
    "command": "cross-team-queue",
    "help": "maw cross-team-queue [--json] [--recipient <name>]"
  }
}
```

**Implementation** (`src/plugin.ts` - AssemblyScript):

```typescript
import { Host, Memory } from "@extism/as-pdk";
import { length } from "@extism/as-pdk/lib/env";
import { fsRead, fsList } from "@maw-rs/wasm-sdk";

@external("extism:host/user", "maw.paths.get") declare function mawPathsGet(input: u64): u64;
export function myAbort(message: string | null, fileName: string | null, lineNumber: u32, columnNumber: u32): void {}

class Item { 
  recipient: string = ""; 
  sender: string = "unknown"; 
  team: string = ""; 
  type: string = "message"; 
  subject: string = ""; 
  body: string = ""; 
  path: string = ""; 
}

export function handle(): i32 {
  const args = extractArgs(Host.inputString());
  const recipient = flagValue(args, "--recipient").toLowerCase();
  const vault = pathGet("vault");
  return vault.length == 0 
    ? finish(false, "", "vault root unavailable") 
    : finish(true, response(scan(vault, recipient)), null);
}

function scan(vault: string, recipient: string): Item[] {
  const out = new Array<Item>();
  const oracles = listPaths(vault, "dir");
  for (let i = 0; i < oracles.length; i++) {
    const files = listPaths(oracles[i] + "/inbox", "file");
    for (let j = 0; j < files.length; j++) {
      const path = files[j]; 
      if (!path.endsWith(".md")) continue;
      const body = readContent(path); 
      if (body.length == 0) continue;
      const item = parseItem(path, baseName(oracles[i]), body);
      if (recipient.length == 0 || item.recipient.toLowerCase() == recipient) out.push(item);
    }
  }
  return out;
}

function parseItem(path: string, oracle: string, content: string): Item {
  const item = new Item();
  item.path = path; 
  item.recipient = pick(front(content, "recipient"), pick(front(content, "to"), oracle));
  item.sender = pick(front(content, "sender"), pick(front(content, "from"), "unknown"));
  item.team = front(content, "team"); 
  item.type = pick(front(content, "type"), "message"); 
  item.body = markdownBody(content);
  item.subject = pick(front(content, "subject"), pick(firstLine(item.body), baseName(path)));
  return item;
}

function response(items: Item[]): string {
  return "{\"items\":[" + itemsJson(items) + "],\"stats\":{\"totalItems\":" + items.length.toString() + ",...},\"schemaVersion\":1}";
}

function pathGet(name: string): string { 
  const input = Memory.allocateString("{\"name\":" + quote(name) + "}"); 
  const output = mawPathsGet(input.offset); 
  const out = new Memory(output, length(output)).toString(); 
  return out.indexOf("\"ok\":true") < 0 ? "" : jsonStringField(out, "path"); 
}

function readContent(path: string): string { 
  const out = fsRead("{\"path\":" + quote(path) + ",\"encoding\":\"utf8\"}"); 
  return out.indexOf("\"ok\":true") < 0 ? "" : jsonStringField(out, "content"); 
}

// ... (utility functions for JSON parsing, frontmatter extraction, etc.)
```

**Key patterns:**
- Compiles to `plugin.wasm` (Extism runtime)
- Uses `@external` decorators to declare host functions
- Manual JSON parsing (no serde in WASM to keep size small)
- Host I/O via `fsRead`, `fsList`, `mawPathsGet` (capability-gated)
- Result format: `{ok: true|false, output?: string, error?: string}`
- Frontmatter parsing for metadata (YAML front matter)

---

### Type C: Fleet Plugin - AssemblyScript (Ship Tier)

**Example: `squad` (team management)**

**File structure:**
```
packages/squad/
├── plugin.source.json
├── plugin.json
├── plugin.wasm (committed)
├── impl.ts     (bun-dev reference, ported to AssemblyScript)
└── src/plugin.ts (AssemblyScript source)
```

**Ship-tier implementation** (`src/plugin.ts` - excerpt):

```typescript
import { Host, Memory } from "@extism/as-pdk";
import { length } from "@extism/as-pdk/lib/env";
import { fsRead, fsWrite, fsList, listSessions, sendKeys, hostExec } from "@maw-rs/wasm-sdk";

@external("extism:host/user", "maw.paths.get") declare function mawPathsGet(input: u64): u64;
export function myAbort(message: string | null, fileName: string | null, lineNumber: u32, columnNumber: u32): void {}

const COLORS: string[] = ["red", "green", "yellow", "blue", "purple", "cyan", "magenta", "white"];

export function handle(): i32 {
  const input = Host.inputString();
  const args = extractArgs(input);
  const cwd = pathGet("cwd");
  const home = pathGet("home");
  const sub = args.length > 0 ? args[0] : "";

  if (sub == "join") return cmdJoin(deriveTeam(cwd), args);
  if (sub != "start" && sub != "say" && sub != "ls") return usage();

  if (home == "") return finish(false, null, "no home directory in invoke context");
  const team = deriveTeam(cwd);
  if (team == "") return finish(false, null, "can't derive a team name from this directory");
  const teams = home + "/.claude/teams";

  if (sub == "start") return cmdStart(teams, team, cwd);
  if (sub == "say") return cmdSay(teams, team, args);
  return cmdLs(teams, team);
}

function deriveTeam(cwd: string): string {
  let base = cwd;
  const slash = cwd.lastIndexOf("/");
  if (slash >= 0) base = cwd.slice(slash + 1);
  if (base.endsWith("-oracle")) base = base.slice(0, base.length - 7);
  return base;
}

function cmdStart(teams: string, team: string, cwd: string): i32 {
  const existing = readFile(cfgOf(teams, team));
  const existed = existing.length > 0;
  let content: string;
  let leadSessionId: string;
  if (existed) {
    const priorSid = jsonStringField(existing, "leadSessionId");
    leadSessionId = priorSid == "" ? nowMillis() : priorSid;
    let updated = ensureStringField(existing, "leadSessionId", leadSessionId);
    updated = setStringField(updated, "leadRepo", cwd);
    content = updated;
  } else {
    const cfg = newConfig(team);
    cfg.leadSessionId = nowMillis();
    cfg.leadRepo = cwd;
    leadSessionId = cfg.leadSessionId;
    content = serializeConfig(cfg);
  }
  const wrote = writeFile(cfgOf(teams, team), content);
  if (wrote != "") return finish(false, null, wrote);

  const leadIbx = inboxOf(teams, team, "team-lead");
  if (readFile(leadIbx) == "") {
    const w = writeFile(leadIbx, "[]\n");
    if (w != "") return finish(false, null, w);
  }

  let out = "⚡ squad '" + team + "' " + (existed ? "adopted (already existed)" : "started") + " → " + dirOf(teams, team) + "\n";
  out += "   lead: " + baseName(cwd) + " (this repo)   lead session: " + leadSessionId + "\n";
  out += "   replies arrive in: inboxes/team-lead.json\n";
  out += "   next: maw squad join digger   ·   maw squad say digger \"<text>\"   ·   maw squad ls";
  return finish(true, out, null);
}

function cmdSay(teams: string, team: string, args: string[]): i32 {
  const member = args.length > 1 ? args[1] : "";
  const text = args.length > 2 ? args.slice(2).join(" ") : "";
  if (member == "" || text == "") return finish(false, null, "usage: maw squad say <member> <text>");
  if (!nameOk(member)) return finish(false, null, `invalid member name '${member}' (letters/digits/-/_ only)`);

  const cfgContent = readFile(cfgOf(teams, team));
  if (cfgContent == "")
    return finish(false, null, `squad '${team}' not started — run: maw squad start (from the lead repo)`);

  const members = parseMembers(cfgContent);
  if (!hasMember(members, member)) {
    const names = memberNames(members);
    return finish(false, null,
      `'${member}' is not in squad '${team}' — members: ${(names == "" ? "(none)" : names)}. join first: maw squad join ${member}`);
  }

  const path = inboxOf(teams, team, member);
  const existing = readFile(path);
  const msg = messageJson("team-lead", text, isoNow());
  const w = writeFile(path, appendToArray(existing, msg));
  if (w != "") return finish(false, null, w);
  let out = "✓ said to " + member + "@" + team + ": " + text;
  const nudge = nudgeMember(member);
  if (nudge != "") out += "\n  ⚠ nudge skipped: " + nudge;
  return finish(true, out, null);
}

// Guard: only valid colors accepted (prevents silent tmux spawn failure)
function colorOk(color: string): bool {
  for (let i = 0; i < COLORS.length; i++) if (COLORS[i] == color) return true;
  return false;
}

// Guard: name validation blocks path traversal (.., /, .)
function nameOk(s: string): bool {
  if (s.length == 0) return false;
  if (!isAlnum(s.charCodeAt(0))) return false;
  for (let i = 1; i < s.length; i++) {
    const c = s.charCodeAt(i);
    if (!isAlnum(c) && c != 45 && c != 95) return false; // '-' '_'
  }
  return true;
}

// Surgical JSON field updates (adopt path): edit only leadSessionId + leadRepo in place
// Other fields/keys survive byte-for-byte (unknown fields preserved)
function setStringField(json: string, key: string, value: string): string {
  const span = findTopLevelKey(json, key);
  if (span.found) return json.slice(0, span.valStart) + quote(value) + json.slice(span.valEnd);
  return appendTopLevelField(json, key, quote(value));
}

function ensureStringField(json: string, key: string, value: string): string {
  const span = findTopLevelKey(json, key);
  if (!span.found) return appendTopLevelField(json, key, quote(value));
  const cur = json.slice(span.valStart, span.valEnd);
  if (cur == "\"\"" || cur == "null") return json.slice(0, span.valStart) + quote(value) + json.slice(span.valEnd);
  return json;
}
```

**Key patterns:**
- Derives team from `cwd` (basename minus "-oracle" suffix)
- Adopts existing team folder without clobbering
- Surgical JSON field updates preserve unknown schema fields and key order
- Guards: name validation (no path traversal), color validation
- Host I/O: `fsRead`, `fsWrite`, `fsList`, `listSessions`, `sendKeys`, `hostExec`
- Wall clock via `hostExec` (WASM has no native clock): `date +%s`
- Inbox append never overwrites: reads existing, appends message, writes back

---

### Type D: Bun-dev Fleet Plugin

**Example: `share` (sshx terminal-share bridge)**

**Implementation** (`src/plugin.ts` - excerpt):

```typescript
import {
  chmodSync,
  closeSync,
  existsSync,
  mkdirSync,
  mkdtempSync,
  openSync,
  readFileSync,
  readdirSync,
  rmSync,
  unlinkSync,
  writeFileSync,
} from "fs";
import { basename, join } from "path";
import { homedir, tmpdir } from "os";

declare const Bun: any;
declare const process: any;

type Log = (s?: string) => void;
type ShareState = {
  name: string;
  url: string;
  pid: number;
  startedAt: string;
};

const DEFAULT_SERVER = "https://ssh.clubsxai.com";
const DEFAULT_BIN = "sshx";
const LABEL_RE = /^[A-Za-z0-9][A-Za-z0-9_.-]{0,127}$/;

function shareServer(): string {
  return (process.env.MAW_SHARE_SERVER || DEFAULT_SERVER).trim();
}

function sshxBin(): string {
  return (process.env.MAW_SHARE_SSHX_BIN || DEFAULT_BIN).trim();
}

function stateDir(): string {
  const home = process.env.HOME || homedir();
  return join(home, ".maw", "share");
}

function ensureStateDir(): string {
  const dir = stateDir();
  mkdirSync(dir, { recursive: true });
  chmodSync(dir, 0o700);
  return dir;
}

function statePath(label: string): string {
  return join(ensureStateDir(), `${label}.json`);
}

function isAlive(pid: unknown): boolean {
  if (!Number.isInteger(pid) || Number(pid) <= 0) return false;
  try {
    process.kill(Number(pid), 0);
    return true;
  } catch {
    return false;
  }
}

function readState(label: string): ShareState | undefined {
  const path = statePath(label);
  if (!existsSync(path)) return undefined;
  return JSON.parse(readFileSync(path, "utf-8")) as ShareState;
}

function readRequiredState(label: string): ShareState {
  const state = readState(label);
  if (!state) throw new Error(`no share state for '${label}'`);
  return state;
}

function writeState(state: ShareState): void {
  const path = statePath(state.name);
  writeFileSync(path, JSON.stringify(state, null, 2) + "\n", { mode: 0o600 });
  chmodSync(path, 0o600);
}

function currentTmuxSession(): string | undefined {
  if (!process.env.TMUX) return undefined;
  const p = Bun.spawnSync(["tmux", "display-message", "-p", "#S"], {
    stdout: "pipe",
    stderr: "ignore",
  });
  const name = p.stdout.toString().trim();
  return name.length > 0 ? name : undefined;
}
```

**Key patterns:**
- Uses Node/Bun APIs: `fs`, `process`, `Bun.spawnSync`
- State stored in `~/.maw/share` with restricted permissions (`0o600`)
- PID-based liveness check via `process.kill(pid, 0)`
- Environment variable overrides (`MAW_SHARE_SERVER`, `MAW_SHARE_SSHX_BIN`)
- Regex validation for labels (`/^[A-Za-z0-9][A-Za-z0-9_.-]{0,127}$/`)
- Integration with tmux session detection

---

## 4. SDK Contract Types

**From `maw-js/sdk`:**

```typescript
// Handler invocation context
export interface InvokeContext {
  source: "cli" | "api" | "hook";  // Dispatch source
  args: string[] | Record<string, unknown>;  // CLI args array or API body
  cwd?: string;  // Current working directory (ship tier)
  home?: string;  // User home directory (ship tier)
}

// Handler response contract
export interface InvokeResult {
  ok: boolean;
  output?: string;  // Captured stdout (optional)
  error?: string;   // Error message (if ok=false)
}

// SDK utilities
export function loadConfig(): { psiPath?: string; host?: string; port?: number; };
export function parseFlags(args: string[], spec: Record<string, Function>, skipCount: number): Record<string, string | number>;
```

**Entry point signature:**

```typescript
// Bun-dev (TypeScript)
export default async function handler(ctx: InvokeContext): Promise<InvokeResult>

// Ship-tier (AssemblyScript/WASM)
export function handle(): i32  // returns 0 on success, 1 on error
```

---

## 5. Error Handling & Safety Patterns

### Pattern 1: Guard Early, Fail Loud

```typescript
// ✓ Guard at entry: validate before state mutation
if (!NAME_RE.test(role)) throw new Error(`invalid oracle name '${role}' (letters/digits/-/_ only)`);
if (!COLORS.includes(color)) throw new Error(`invalid color '${color}' — spawn would fail SILENTLY. valid: ${COLORS.join(" ")}`);

// ✓ Check roster membership before inbox write
const members = ((cfg.members || []) as any[]).map((m) => m.name);
if (!names.includes(member))
  throw new Error(`'${member}' is not in squad '${team}' — members: ${names.join(", ")}...`);
```

### Pattern 2: Immutable Append, Soft Deletes

```typescript
// Bun-dev: soft delete (retire flag, never truncate)
export async function cmdContactsRm(name: string) {
  const data = loadContacts();
  if (!data.contacts[name]) { console.error(`error: contact '${name}' not found`); return; }
  data.contacts[name].retired = true;  // ✓ Mark, don't erase
  saveContacts(data);
  console.log(`✓ contact ${name} retired`);
}

// Ship-tier: append to array, preserve existing bytes
function appendToArray(existing: string, element: string): string {
  const trimmed = rtrimWs(existing);
  if (trimmed == "" || trimmed == "[]" || trimmed == "[\n]") return "[\n  " + element + "\n]\n";
  const close = trimmed.lastIndexOf("]");
  if (close < 0) return "[\n  " + element + "\n]\n";
  let body = rtrimWs(trimmed.slice(0, close));
  if (body.endsWith("[")) return body + "\n  " + element + "\n]\n";
  return body + ",\n  " + element + "\n]\n";  // ✓ Never clobber existing
}
```

### Pattern 3: Surgical JSON Updates (Preserve Unknown Fields)

```typescript
// Ship-tier: adopt config without lossy round-trip (unknown fields survive)
function cmdStart(teams: string, team: string, cwd: string): i32 {
  const existing = readFile(cfgOf(teams, team));
  const existed = existing.length > 0;
  let content: string;
  if (existed) {
    // ✓ Raw text edits only: leadSessionId + leadRepo
    // ✓ Unknown schema fields + key order preserved
    let updated = ensureStringField(existing, "leadSessionId", leadSessionId);
    updated = setStringField(updated, "leadRepo", cwd);
    content = updated;
  } else {
    // Create path: typed serialize (no existing unknown fields to preserve)
    const cfg = newConfig(team);
    cfg.leadSessionId = nowMillis();
    cfg.leadRepo = cwd;
    content = serializeConfig(cfg);
  }
  // ...
}
```

### Pattern 4: Console Output Capture

```typescript
export default async function handler(ctx: InvokeContext): Promise<InvokeResult> {
  const logs: string[] = [];
  const origLog = console.log;
  const origError = console.error;
  
  // ✓ Intercept all output
  console.log = (...a: any[]) => logs.push(a.map(String).join(" "));
  console.error = (...a: any[]) => logs.push(a.map(String).join(" "));
  
  try {
    await cmdContactsLs();
    return { ok: true, output: logs.join("\n") || undefined };
  } catch (e: any) {
    return { ok: false, error: e.message };
  } finally {
    // ✓ Always restore
    console.log = origLog;
    console.error = origError;
  }
}
```

### Pattern 5: Host I/O Error Handling (WASM)

```typescript
function pathGet(name: string): string {
  const input = Memory.allocateString("{\"name\":" + quote(name) + "}");
  const output = mawPathsGet(input.offset);
  const out = new Memory(output, length(output)).toString();
  return out.indexOf("\"ok\":true") < 0 ? "" : jsonStringField(out, "path");  // ✓ Fail gracefully
}

function readFile(path: string): string {
  const out = fsRead("{\"path\":" + quote(path) + ",\"encoding\":\"utf8\"}");
  if (out.indexOf("\"ok\":true") < 0) return "";  // ✓ Return empty string on error
  return jsonStringField(out, "content");
}

function writeFile(path: string, content: string): string {
  const out = fsWrite("{\"path\":" + quote(path) + ",\"content\":" + quote(content) + ",\"mode\":\"overwrite\",\"mkdirp\":true}");
  if (out.indexOf("\"ok\":true") >= 0) return "";  // ✓ Return empty string on success
  const err = jsonStringField(out, "error");
  return err == "" ? "write failed: " + path : err;  // ✓ Return error message on failure
}
```

---

## 6. Testing Patterns

**Example test** (`20-contacts/contacts.test.ts`):

```typescript
import { describe, it, expect, mock, beforeEach } from "bun:test";
import type { InvokeContext } from "maw-js/sdk";

// Mock the implementation module
mock.module("./impl", () => ({
  cmdContactsLs: async () => {
    console.log("CONTACTS (2):");
    console.log("  neo           maw: neo@white");
  },
  cmdContactsAdd: async (name: string, args: string[]) => {
    console.log(`✓ contact ${name} saved`);
  },
  cmdContactsRm: async (name: string) => {
    console.log(`✓ contact ${name} retired`);
  },
}));

describe("contacts plugin", () => {
  let handler: (ctx: InvokeContext) => Promise<any>;

  beforeEach(async () => {
    const mod = await import("./index");
    handler = mod.default;
  });

  it("cli: ls subcommand lists contacts", async () => {
    const result = await handler({ source: "cli", args: ["ls"] });
    expect(result.ok).toBe(true);
    expect(result.output).toContain("CONTACTS");
  });

  it("cli: add subcommand adds a contact", async () => {
    const result = await handler({ source: "cli", args: ["add", "neo", "--maw", "neo@white"] });
    expect(result.ok).toBe(true);
    expect(result.output).toContain("neo saved");
  });

  it("api GET: list contacts", async () => {
    const result = await handler({ source: "api", args: { method: "GET" } });
    expect(result.ok).toBe(true);
    expect(result.output).toContain("CONTACTS");
  });

  it("api POST: add contact", async () => {
    const result = await handler({
      source: "api",
      args: { method: "POST", action: "add", name: "neo", transport: "neo@white" },
    });
    expect(result.ok).toBe(true);
    expect(result.output).toContain("neo saved");
  });
});
```

**Key patterns:**
- Mock `impl.ts` module to isolate handler logic
- Test both CLI and API invocation paths
- Assert on `result.ok` and `result.output`
- Run with `bun test`

---

## 7. Build & Deployment

### Registry Generation

The `registry.json` is regenerated deterministically from package manifests:

```typescript
// scripts/gen-registry.ts (excerpt)

const repoRoot = resolve(dirname(new URL(import.meta.url).pathname), "..");
const packagesDir = join(repoRoot, "packages");
const registryPath = join(repoRoot, "registry.json");

interface Entry {
  commit: string;          // Last commit touching this package
  sha256: string;          // Artifact sha256 pin
  path: string;            // packages/<dir>
  version: string;
  capabilities: string[];
}

/**
 * Pin sourcing mirrors the CI pin-integrity gate:
 * 1. plugin.json first
 * 2. dev-tier-active package's plugin.source.json fallback
 */
function resolvePin(dir: string, manifest: Record<string, unknown>): string | null {
  const fromJson = (manifest.artifact as { sha256?: string } | undefined)?.sha256;
  if (fromJson) return fromJson;
  const source = readManifest(join(dir, "plugin.source.json"));
  const fromSource = (source?.artifact as { sha256?: string } | undefined)?.sha256;
  return fromSource ?? null;
}

/**
 * Commit is the LAST commit touching the package directory, not repo HEAD.
 * That commit is guaranteed to contain the package's current committed bytes,
 * so raw.githubusercontent.com/<o>/<r>/<commit>/packages/<dir>/plugin.wasm is
 * immutable and hashes to sha256.
 */
function lastCommitTouching(relPath: string): string {
  const out = execFileSync("git", ["-C", repoRoot, "log", "-1", "--format=%H", "--", relPath], {
    encoding: "utf8",
  }).trim();
  return out;
}
```

**Registry entry example:**

```json
{
  "contacts": {
    "commit": "ea6a3ffa00d58870c60e7d193e92d90d4504156c",
    "sha256": "sha256:2d1ffdf7c7fa053cb4e51d1c3d7db3b9b01f4339c9eab76f167d60414fa7cb11",
    "path": "packages/20-contacts",
    "version": "1.0.0",
    "capabilities": [
      "fs:read:psi",
      "fs:write:psi"
    ]
  }
}
```

### Capabilities & Permissions

Plugins declare capabilities in `plugin.json`:

| Capability | Scope | Use |
|------------|-------|-----|
| `fs:read:psi` | `~/$PSI_PATH/` | Read oracle state (contacts, config) |
| `fs:write:psi` | `~/$PSI_PATH/` | Write oracle state |
| `fs:read:vault` | Vault root | Read cross-team inbox queue |
| `fs:read:teams` | `~/.claude/teams` | Read squad rosters |
| `fs:write:teams` | `~/.claude/teams` | Write squad config |
| `tmux:read` | Read sessions | Query tmux state |
| `tmux:send` | Send input | Post messages to tmux |
| `proc:exec:*` | Execute tool | Run commands (`date`, `git`, etc.) |
| `net:fetch:*` | Network | Fetch from remote (Discord, etc.) |
| `secret:use:*` | Secrets | Use stored credentials |

---

## 8. Key Idioms & Conventions

### 1. Weight-Based Ordering

```
packages/
├── 20-costs/       ← weight 20: first-pass tools
├── 20-contacts/
├── 50-artifact-manager/  ← weight 50: second-pass features
└── 50-incubate/
```

Plugins are sorted by weight and loaded in order.

### 2. Command Naming

- **Verb-based**: `contacts`, `costs`, `squad`, `team`
- **Aliases**: `contact` (singular), `t` (short)
- **Subcommands**: `maw contacts ls|add|rm`, `maw squad start|join|say|ls`

### 3. Result Format

All plugins return:

```json
{
  "ok": true,
  "output": "rendered output or JSON"
}
```

or

```json
{
  "ok": false,
  "error": "error message"
}
```

### 4. Dispatch Patterns

**CLI:**
```bash
maw contacts ls                        # subcommand
maw contacts add neo --maw neo@white   # with flags
```

**API:**
```json
POST /api/plugin/contacts
{
  "method": "GET"  // or "POST" with action, name, etc.
}
```

### 5. TypeScript Build Setup

**tsconfig.json:**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noEmit": true,
    "skipLibCheck": true
  },
  "include": ["src/plugin.ts", "src/plugin.test.ts"]
}
```

**package.json:**

```json
{
  "name": "@maw-plugins/contacts",
  "type": "module",
  "main": "./index.ts",
  "peerDependencies": {
    "maw-js": "*"
  }
}
```

---

## 9. Summary

**maw-plugins** is a plugin system for maw-js featuring:

- **Three tiers**: Ship-tier WASM (AssemblyScript), Bun-dev TypeScript, native Swift
- **Capability-gated I/O**: Plugins declare required permissions; host enforces sandbox
- **Deterministic registry**: SHA256-pinned artifacts indexed by git commit
- **Immutable audit trail**: Soft deletes, append-only inboxes, surgical JSON updates
- **Dual dispatch**: CLI and HTTP API with unified handler contract
- **Type safety**: TypeScript for Bun plugins, AssemblyScript for WASM
- **Testing**: Bun test with module mocking and context simulation

**Key files:**
- `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-plugins/package.json`
- `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-plugins/registry.json`
- `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-plugins/README.md`
- `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-plugins/scripts/gen-registry.ts`
