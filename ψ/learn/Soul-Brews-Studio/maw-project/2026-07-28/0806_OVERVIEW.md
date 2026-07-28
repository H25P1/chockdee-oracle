# maw-project: Overview

## What is this project?

`@maw/project` is a thin dispatcher plugin for the maw-js ecosystem that orchestrates `learn` (read-only study) and `incubate` (active development) workflows for external GitHub repositories. It wraps sibling plugins (`@maw/learn` and `@maw/incubate`) and provides cross-cutting utilities (`find`, `list`) for managing the ψ vault directory structure.

## Relationship to sibling plugins

The plugin follows a **delegation pattern**:

- **`maw project learn <url-or-slug>`** → delegates to `@maw/learn` (creates symlinks in `ψ/learn/owner/repo`)
- **`maw project incubate <url-or-slug>`** → delegates to `@maw/incubate` (creates symlinks in `ψ/incubate/owner/repo` with mode flags)
- **`maw project find <query>`** and **`maw project list`** → remain in this plugin (cross-cutting utilities)

Both `@maw/learn` and `@maw/incubate` are **optional peerDependencies**. The plugin detects and calls them when installed; if absent, those subcommands return "not yet implemented" stubs (v0.1.0 status).

### The ghq/symlink principle

The plugin enforces:
> **"ghq owns the clone → ψ/ owns the symlink. Never copy. Always symlink. One source of truth."**

Actual clones live in `~/ghq/github.com/owner/repo` (or equivalent ghq root); the ψ vault contains only symlinks to a single canonical source.

## Key files

| File | Purpose |
|------|---------|
| `plugin.json` | maw manifest: subcommand definitions, CLI flags, API route (`POST /project`) |
| `package.json` | npm metadata; lists `@maw/learn` and `@maw/incubate` as optional peerDependencies |
| `src/index.ts` | Dispatcher handler: parses subcommand, flags (via `arg`), and delegates to subcommands.ts |
| `src/subcommands.ts` | Stub implementations for each subcommand (learn, incubate, find, list) |
| `src/types.ts` | TypeScript interfaces mirroring maw-js SDK types (InvokeContext, InvokeResult, SubcommandOpts variants) |
| `test/smoke.test.ts` | Routing tests (bun); verifies dispatcher correctly routes subcommands and rejects bad input |

## How to use it

### Install

```bash
maw plugin install https://github.com/Soul-Brews-Studio/maw-project
```

Requires `@maw/learn` and/or `@maw/incubate` to be installed (optional peerDependencies).

### CLI invocation

```bash
# Study a repo (read-only reference)
maw project learn https://github.com/owner/repo
maw project learn owner/repo                 # slug form, auto-resolved by ghq

# Active development modes
maw project incubate https://github.com/owner/repo
maw project incubate owner/repo --flash "fix typo"     # fast: issue → branch → PR → offload
maw project incubate owner/repo --contribute           # multi-feature mode
maw project incubate owner/repo --offload              # remove symlink, keep clone
maw project incubate owner/repo --status               # show state

# Utilities
maw project find oracle                      # search ghq + vault trees
maw project list                             # show all tracked projects
maw project list --filter learn              # learn-only projects
maw project list --filter incubate           # incubate-only projects

# Output formatting
maw project learn owner/repo --json          # JSON payload
maw project list --dry-run                   # dry-run flag
```

### API invocation

```bash
POST /project
{
  "source": "api",
  "args": ["learn", "owner/repo", "--json"]
}
```

## Notable patterns

1. **Dispatcher-first design**: `index.ts` is a pure arg parser and router; all logic is subcommand-shaped (`learn`, `incubate`, `find`, `list`).

2. **Flag parsing via `arg` (Vercel)**: Same pattern used in `50-bud` and `@maw/incubate`. Supports both positional args and flags (e.g., `--flash`, `--contribute`, `--offload`, `--status`, `--filter`, `--dry-run`, `--json`).

3. **v0.1.0 stubs return not-yet-implemented**: All four subcommands currently return `ok: false` with a descriptive error message showing the parsed intent. This allows testing the dispatcher without the sibling plugins.

4. **Type mirroring**: `types.ts` replicates maw-js SDK types locally, letting the plugin compile standalone. No runtime dependency on the SDK's type definitions.

5. **Optional peerDependencies pattern**: `@maw/learn` and `@maw/incubate` marked optional in package.json. The plugin loads gracefully if they're absent; v2 will use dynamic imports or peer-discovery to invoke them.

6. **Result formatting**: `formatResult()` unifies JSON and plain-text outputs — errors always fill the `error` field; success fills `message` (plain) or wraps in JSON.

---

**Status**: v0.1.0 — scaffolded stubs, wired dispatcher, pending v2 delegation to sibling plugins (tracked in [maw-js#520](https://github.com/Soul-Brews-Studio/maw-js/issues/520)).
