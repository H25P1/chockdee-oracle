# maw-learn Plugin Overview

## What is this project?

**maw-learn** is a maw-js plugin that explores codebases with parallel Haiku agents and generates organized documentation to `$MAW_VAULT_ROOT/learn/<owner>/<repo>/`. It's a direct migration from the Oracle `/learn` skill, preserving the same CLI surface, depth modes, and on-disk layout.

## Comparison to Oracle /learn skill

This is a **1:1 migration** of the Oracle `/learn` skill as a maw plugin. Key alignment points:
- **Same CLI**: `maw learn <url-or-slug> [--fast|--deep]` matches the skill's interface exactly
- **Same depth modes**: fast (1 agent), default (3 agents), deep (5 agents)
- **Same output layout**: docs organized under `$MAW_VAULT_ROOT/learn/<owner>/<repo>/YYYY-MM-DD/HHMM_*.md`
- **Different form**: plugin-based distribution rather than inlined skill; can be swapped without losing existing docs

**Note**: Current version is **scaffold/stub** — CLI parsing and response types are wired; the real parallel-agent implementation lands in follow-up PRs (tracked in [maw-js#520](https://github.com/Soul-Brews-Studio/maw-js/issues/520)).

## Key Files

| File | Purpose |
|------|---------|
| `plugin.json` | Plugin manifest: name, entry point, CLI config, API route |
| `src/index.ts` | Main handler: CLI arg parsing, request dispatch, stub response |
| `src/types.ts` | Contract definitions: `LearnRequest`, `LearnResult`, `LearnMode` |

**plugin.json highlights**:
- Entry: `./src/index.ts`
- CLI command: `learn` with flags `--fast`, `--deep`, `--init`, `--json`
- API route: `POST /learn`
- Weight: 50 (plugin priority)

## How to Use

### Install

```bash
maw plugin install @maw/learn
# or, from this directory:
maw plugin link .
```

### CLI Invocation

```bash
maw learn https://github.com/owner/repo       # full URL
maw learn owner/repo                           # shorthand
maw learn my-slug                              # slug from ψ/memory/slugs.yaml
maw learn /path/to/local/repo                 # local path

maw learn owner/repo --fast                   # 1 agent, ~2 min
maw learn owner/repo --deep                   # 5 agents, ~10 min
maw learn --init                              # restore origin symlinks
```

### API Surface

```http
POST /learn
Content-Type: application/json

{ "target": "owner/repo", "mode": "fast" | "default" | "deep", "init": false }
```

### Output by Mode

| Mode | Agents | Files Generated | Use Case |
|------|--------|-----------------|----------|
| `--fast` | 1 | `OVERVIEW.md` | Quick "what is this?" scan |
| (default) | 3 | `ARCHITECTURE.md`, `CODE-SNIPPETS.md`, `QUICK-REFERENCE.md` | Normal exploration |
| `--deep` | 5 | Above + `TESTING.md`, `API-SURFACE.md` | Complex codebases, deep dive |

All docs land in `$MAW_VAULT_ROOT/learn/<owner>/<repo>/YYYY-MM-DD/HHMM_*.md` (timestamped to avoid overwrites within same day).

## Notable Patterns

1. **Multi-mode flexibility**: `agentsForMode(mode: LearnMode)` returns agent count; scaffold encodes hard limits (1, 3, or 5).

2. **Dual-source handler**: Same handler dispatches from CLI (`ctx.source === "cli"`), API (`"api"`), and peer (`"peer"`) sources; CLI args parsed explicitly, API/peer payloads parsed from body.

3. **Request contract clarity**: `LearnRequest` decouples target resolution (URL, slug, or path) from agent spawn—simplifies future PR logic for cloning and symlink setup.

4. **Stub discipline**: Current version returns `stubResult()` with explicit "not-yet-implemented" message and tracking issue link rather than throwing or silently no-opping.

5. **Distributed skill**: Plugin uses `@maw/sdk` peer dependency and `definePlugin()` for maw.js integration; no monolithic skill bundling required.

## License

BUSL-1.1 (converts to Apache 2.0 on 2040-04-18) — Non-commercial/personal/educational/internal-business use.
