# maw-core-plugins: Code Snippets

**Repository:** maw-core-plugins (Soul-Brews-Studio)  
**Source:** `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-core-plugins`  
**Purpose:** Essential core plugins that ship with maw-js — agent lifecycle primitives that cannot be removed

---

## Overview

The maw-core-plugins repository contains 4 core plugins that form the agent lifecycle primitives:

| Plugin | File | Purpose |
|--------|------|---------|
| **wake** | `packages/00-wake/index.ts` | Spawn or attach to Oracle agent sessions |
| **sleep** | `packages/00-sleep/index.ts` | Gracefully stop a single Oracle agent's tmux window |
| **stop** | `packages/00-stop/index.ts` | Stop ALL oracle fleet sessions |
| **done** | `packages/00-done/index.ts` | Clean up and finish a worktree window (rrr, git save, kill, remove) |

All plugins follow the same handler pattern: they accept either CLI or API invocation via `InvokeContext`, capture console output, and return `InvokeResult`.

Weight `00` = always loads first.

---

## Plugin 1: wake

**File:** `/packages/00-wake/index.ts`  
**Command Name:** `wake`  
**Description:** Spawn or attach to an oracle session

### Handler Implementation

```typescript
import type { InvokeContext, InvokeResult } from "../../../plugin/types";
import { parseFlags } from "../../../cli/parse-args";

export const command = {
  name: "wake",
  description: "Spawn or attach to an oracle session",
};

export default async function handler(ctx: InvokeContext): Promise<InvokeResult> {
  // Dynamic imports — clean, one await, mockable
  const { cmdWake } = await import("../../wake");
  const { cmdWakeAll } = await import("../../fleet");
  const { parseWakeTarget, ensureCloned } = await import("../../wake-target");
  const { fetchGitHubPrompt } = await import("../../wake-resolve");

  const logs: string[] = [];
  const origLog = console.log;
  const origError = console.error;
  console.log = (...a: any[]) => logs.push(a.map(String).join(" "));
  console.error = (...a: any[]) => logs.push(a.map(String).join(" "));

  try {
    if (ctx.source === "cli") {
      const args = ctx.args as string[];

      if (!args[0]) {
        return {
          ok: false,
          error: "usage: maw wake <oracle|org/repo|URL> [task] [--task \"<prompt>\"] [--new <name>] [--fresh] [--no-attach] [--issue N] [--pr N] [--repo org/name] [--list]\n       maw wake all [--kill]",
        };
      }

      if (args[0].toLowerCase() === "all") {
        const flags = parseFlags(args, { "--kill": Boolean, "--all": Boolean, "--resume": Boolean }, 1);
        await cmdWakeAll({ kill: flags["--kill"], all: flags["--all"], resume: flags["--resume"] });
        return { ok: true, output: logs.join("\n") || undefined };
      }

      const flags = parseFlags(args, {
        "--new": String, "--incubate": String, "--issue": Number,
        "--pr": Number, "--repo": String, "--task": String,
        "--fresh": Boolean, "--no-attach": Boolean, "--list": Boolean, "--ls": "--list",
      }, 1);

      const wakeOpts: {
        task?: string; newWt?: string; prompt?: string;
        incubate?: string; fresh?: boolean; noAttach?: boolean; listWt?: boolean;
      } = {};
      let issueNum: number | null = flags["--issue"] ?? null;
      let repo: string | undefined = flags["--repo"];

      const parsed = parseWakeTarget(args[0]);
      const oracleName = parsed ? parsed.oracle : args[0];
      if (parsed) {
        await ensureCloned(parsed.slug);
        if (parsed.issueNum) { issueNum = parsed.issueNum; repo = parsed.slug; }
      }

      if (flags["--new"]) wakeOpts.newWt = flags["--new"];
      if (flags["--incubate"]) wakeOpts.incubate = flags["--incubate"];
      if (flags["--fresh"]) wakeOpts.fresh = true;
      if (flags["--no-attach"]) wakeOpts.noAttach = true;
      if (flags["--list"]) wakeOpts.listWt = true;
      if (flags["--task"]) wakeOpts.noAttach = true;

      const positionals = flags._;
      if (positionals.length > 0) wakeOpts.task = positionals[0];
      if (positionals.length > 1) wakeOpts.prompt = positionals.slice(1).join(" ");

      if (wakeOpts.incubate && !repo) { repo = wakeOpts.incubate; }
      const prNum: number | null = flags["--pr"] ?? null;
      if (issueNum) {
        console.log(`\x1b[36m⚡\x1b[0m fetching issue #${issueNum}...`);
        wakeOpts.prompt = await fetchGitHubPrompt("issue", issueNum, repo);
        if (!wakeOpts.task) wakeOpts.task = `issue-${issueNum}`;
      } else if (prNum) {
        console.log(`\x1b[36m⚡\x1b[0m fetching PR #${prNum}...`);
        wakeOpts.prompt = await fetchGitHubPrompt("pr", prNum, repo);
        if (!wakeOpts.task) wakeOpts.task = `pr-${prNum}`;
      } else if (flags["--task"]) {
        wakeOpts.prompt = flags["--task"];
      }

      await cmdWake(oracleName, wakeOpts);
      return { ok: true, output: logs.join("\n") || undefined };
    }

    // API source
    const body = ctx.args as Record<string, unknown>;
    const oracle = body.oracle as string | undefined;
    if (!oracle) return { ok: false, error: "missing oracle name" };

    const wakeOpts: { task?: string; prompt?: string; fresh?: boolean; noAttach?: boolean } = {};
    if (body.task) wakeOpts.task = body.task as string;
    if (body.issue) {
      const issueNum = body.issue as number;
      wakeOpts.prompt = await fetchGitHubPrompt("issue", issueNum, body.repo as string | undefined);
      if (!wakeOpts.task) wakeOpts.task = `issue-${issueNum}`;
    }
    if (body.fresh) wakeOpts.fresh = true;
    if (body.noAttach) wakeOpts.noAttach = true;

    await cmdWake(oracle, wakeOpts);
    return { ok: true, output: logs.join("\n") || undefined };
  } catch (e: any) {
    return { ok: false, error: e.message };
  } finally {
    console.log = origLog;
    console.error = origError;
  }
}
```

### Key Features

- **Dual-source handling:** Supports both CLI and API invocation via `ctx.source`
- **Dynamic imports:** Lazy-loads dependencies for better mockability and performance
- **Flag parsing:** Complex argument parsing with support for flags like `--new`, `--fresh`, `--issue`, `--pr`, etc.
- **GitHub integration:** Can fetch issue and PR prompts via `fetchGitHubPrompt()`
- **Wake target parsing:** Supports multiple target formats (oracle name, org/repo, URL)
- **Output capture:** Redirects console.log/error to return to caller
- **Error handling:** Returns structured result with `{ ok, error }` or `{ ok, output }`

---

## Plugin 2: sleep

**File:** `/packages/00-sleep/index.ts`  
**Command Name:** `sleep`  
**Description:** Gracefully stop a single Oracle agent's tmux window

### Handler Implementation

```typescript
import type { InvokeContext, InvokeResult } from "../../../plugin/types";
import { cmdSleepOne } from "../../sleep";

export const command = {
  name: ["sleep"],
  description: "Gracefully stop a single Oracle agent's tmux window.",
};

export default async function handler(ctx: InvokeContext): Promise<InvokeResult> {
  const logs: string[] = [];
  const origLog = console.log;
  const origError = console.error;
  console.log = (...a: any[]) => logs.push(a.map(String).join(" "));
  console.error = (...a: any[]) => logs.push(a.map(String).join(" "));
  try {
    let oracle: string;
    let window: string | undefined;

    if (ctx.source === "cli") {
      const args = ctx.args as string[];
      if (!args[0]) {
        return { ok: false, error: "usage: maw sleep <oracle> [window]" };
      }
      if (args[0] === "--all-done") {
        logs.push("(placeholder) maw sleep --all-done — sleep ALL agents. Not yet implemented.");
        return { ok: true, output: logs.join("\n") };
      }
      oracle = args[0];
      window = args[1];
    } else {
      const args = ctx.args as Record<string, unknown>;
      if (!args.oracle) {
        return { ok: false, error: "oracle is required" };
      }
      oracle = args.oracle as string;
      window = args.window as string | undefined;
    }

    await cmdSleepOne(oracle, window);
    return { ok: true, output: logs.join("\n") || undefined };
  } catch (e: any) {
    return { ok: false, error: e.message };
  } finally {
    console.log = origLog;
    console.error = origError;
  }
}
```

### Key Features

- **Single agent sleep:** Gracefully stops one Oracle's tmux window
- **Optional window parameter:** Can target a specific window within the oracle
- **Simple argument handling:** Takes oracle name and optional window name
- **Placeholder for future:** Comment notes that `--all-done` is not yet implemented
- **Consistent output capture:** Same pattern as wake plugin for logging

---

## Plugin 3: stop

**File:** `/packages/00-stop/index.ts`  
**Command Name:** `stop` (alias: `rest`)  
**Description:** Stop ALL oracle fleet sessions

### Handler Implementation

```typescript
import type { InvokeContext, InvokeResult } from "../../../plugin/types";
import { cmdSleep } from "../../fleet";

export const command = {
  name: ["stop", "rest"],
  description: "Stop ALL oracle fleet sessions.",
};

export default async function handler(ctx: InvokeContext): Promise<InvokeResult> {
  const logs: string[] = [];
  const origLog = console.log;
  const origError = console.error;
  console.log = (...a: any[]) => logs.push(a.map(String).join(" "));
  console.error = (...a: any[]) => logs.push(a.map(String).join(" "));
  try {
    await cmdSleep();
    return { ok: true, output: logs.join("\n") || undefined };
  } catch (e: any) {
    return { ok: false, error: e.message };
  } finally {
    console.log = origLog;
    console.error = origError;
  }
}
```

### Key Features

- **Fleet-wide operation:** Stops all oracle agents in the fleet at once
- **Minimal interface:** No parameters needed—operates on entire fleet
- **Command aliases:** Responds to both `stop` and `rest`
- **Simplest plugin:** Most straightforward implementation, delegates to `cmdSleep()` from fleet module

---

## Plugin 4: done

**File:** `/packages/00-done/index.ts`  
**Command Name:** `done` (alias: `finish`)  
**Description:** Clean up a finished worktree window (rrr, git save, kill, remove worktree)

### Handler Implementation

```typescript
import type { InvokeContext, InvokeResult } from "../../../plugin/types";
import { cmdDone } from "../../done";

export const command = {
  name: ["done", "finish"],
  description: "Clean up a finished worktree window: rrr, git save, kill, remove worktree.",
};

export default async function handler(ctx: InvokeContext): Promise<InvokeResult> {
  const logs: string[] = [];
  const origLog = console.log;
  const origError = console.error;
  console.log = (...a: any[]) => logs.push(a.map(String).join(" "));
  console.error = (...a: any[]) => logs.push(a.map(String).join(" "));
  try {
    let name: string;
    let force: boolean | undefined;
    let dryRun: boolean | undefined;

    if (ctx.source === "cli") {
      const args = ctx.args as string[];
      // Find first non-flag arg
      const positional = args.filter(a => !a.startsWith("--"));
      if (!positional[0]) {
        return { ok: false, error: "usage: maw done <window-name> [--force] [--dry-run]" };
      }
      name = positional[0];
      force = args.includes("--force");
      dryRun = args.includes("--dry-run");
    } else {
      const args = ctx.args as Record<string, unknown>;
      if (!args.name) {
        return { ok: false, error: "name is required" };
      }
      name = args.name as string;
      force = args.force as boolean | undefined;
      dryRun = args.dryRun as boolean | undefined;
    }

    await cmdDone(name, { force, dryRun });
    return { ok: true, output: logs.join("\n") || undefined };
  } catch (e: any) {
    return { ok: false, error: e.message };
  } finally {
    console.log = origLog;
    console.error = origError;
  }
}
```

### Key Features

- **Worktree cleanup:** Orchestrates complete cleanup workflow: retrospective (rrr), git save, kill tmux window, remove worktree
- **Optional flags:** Supports `--force` and `--dry-run` for flexibility
- **Requires window name:** Takes window name as positional argument
- **Command aliases:** Responds to both `done` and `finish`
- **Consistent dual-source handling:** Supports both CLI and API invocation

---

## Common Plugin Patterns

All four plugins share these consistent patterns:

### 1. Output Capture Strategy
```typescript
const logs: string[] = [];
const origLog = console.log;
const origError = console.error;
console.log = (...a: any[]) => logs.push(a.map(String).join(" "));
console.error = (...a: any[]) => logs.push(a.map(String).join(" "));
// ... do work ...
return { ok: true, output: logs.join("\n") || undefined };
```

### 2. Source-Aware Argument Parsing
```typescript
if (ctx.source === "cli") {
  const args = ctx.args as string[];
  // Parse CLI arguments
} else {
  const args = ctx.args as Record<string, unknown>;
  // Parse API object
}
```

### 3. Error Handling Structure
```typescript
try {
  // ... operation ...
  return { ok: true, output: logs.join("\n") || undefined };
} catch (e: any) {
  return { ok: false, error: e.message };
} finally {
  console.log = origLog;
  console.error = origError;
}
```

### 4. Command Metadata Export
```typescript
export const command = {
  name: "plugin-name" | ["alias1", "alias2"],
  description: "Human-readable description",
};

export default async function handler(ctx: InvokeContext): Promise<InvokeResult> {
  // implementation
}
```

---

## Type Signatures

Plugins rely on types from the core maw system:

- **`InvokeContext`**: Input context containing:
  - `source: "cli" | "api"` — invocation source
  - `args: string[] | Record<string, unknown>` — arguments (varies by source)

- **`InvokeResult`**: Output structure:
  - `ok: boolean` — success flag
  - `error?: string` — error message (when ok = false)
  - `output?: string` — captured output (when ok = true)

---

## Summary

The maw-core-plugins repository implements the four essential agent lifecycle commands:

| Plugin | Scope | Users | Implementation Lines |
|--------|-------|-------|----------------------|
| **wake** | Spawn/attach individual oracle | Frequent | 112 |
| **sleep** | Stop individual oracle | Frequent | 47 |
| **stop** | Stop all oracles | Rare | 24 |
| **done** | Cleanup finished worktree | Frequent | 48 |

Each plugin is designed as a thin handler layer that captures CLI/API input, normalizes it, and delegates to a core command module. The consistent patterns enable reliable composition within the maw-js ecosystem.
