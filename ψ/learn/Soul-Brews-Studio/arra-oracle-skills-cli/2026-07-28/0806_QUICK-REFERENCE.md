# arra-oracle-skills-cli — Quick Reference

**Oracle Skills Installer** — Install Oracle skills to Claude Code, OpenCode, Cursor, and 19 other AI coding agents.

---

## Overview

arra-oracle-skills-cli is a command-line tool that distributes Oracle skills (persistent memory, session awareness, collaborative tools) to 20+ AI coding agents. Supports flexible installation: pick a profile (minimal/standard/full/lab) or cherry-pick individual skills. Works globally (user-wide) or locally (project-scoped).

**Skills included:** 21 production skills + 40 archived (zombie) skills available by name.  
**Version:** v26.7.27-alpha.947 (as of 2026-07-28)  
**License:** MIT

---

## Installation

### Method 1: Claude Code Plugin (easiest)
For Claude Code users only — no Bun or git required:

```bash
/plugin marketplace add Soul-Brews-Studio/arra-oracle-skills-cli
/plugin install oracle-skills@oracle-skills
```

### Method 2: Terminal — Bun/bunx (recommended)
Works for any agent. Requires Bun runtime:

```bash
# Install Bun (if not already installed)
curl -fsSL https://bun.sh/install | bash

# Install skills globally
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -p full
```

### Method 3: Compiled Binary
Pre-compiled binary for macOS (x64, arm64) and Linux (x64, arm64) — no Bun required:

```bash
curl -fsSL https://raw.githubusercontent.com/Soul-Brews-Studio/arra-oracle-skills-cli/main/install.sh | bash
```

**Environment variables** (optional):
- `ORACLE_SKILLS_VERSION=v1.6.6` — Pin to a specific version
- `ORACLE_SKILLS_USE_BUNX=1` — Skip binary download, use bunx instead

### Method 4: npm (lagging mirror)
Published to npm, but updates lag behind GitHub. Works only with Bun:

```bash
bunx npm:arra-oracle-skills@latest install -g -y
```

> **⚠️ Note:** `npx arra-oracle-skills` does NOT work under Node.js — the CLI requires Bun runtime. If you see "arra-oracle-skills requires Bun", use bunx instead.

### Quick Reference: Installation Examples

| Goal | Command |
|------|---------|
| **Minimal set** (7 skills) | `bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -p minimal` |
| **Standard set** (20 skills) | `bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -p standard` |
| **All production skills** (29 skills) | `bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -p full` |
| **All + experimental** (32 skills) | `bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -p lab` |
| **Pick specific skills** | `bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -s about-oracle recap rrr` |
| **Project-local install** (instead of global) | Drop `-g` → installs to `./.claude/skills/` |
| **Update existing install** | Re-run the same command (always fetch from GitHub, not via arra binary) |

---

## Supported Agents (20 total)

### Auto-Detected Agents (install by default)
These agents are detected automatically and installed to by default:

| Agent | Installation Path | Command Format |
|-------|-------------------|-----------------|
| **Claude Code** | `~/.claude/skills/` | Markdown |
| **Codex** | `~/.codex/skills/` | Markdown (prompts/) |

### Optional Agents (require explicit flag)
Detected on your system but require `-a <name>` to install:

| Agent | Installation Path | Command Format |
|-------|-------------------|-----------------|
| Cursor | `~/.cursor/skills/` | Markdown |
| Amp | `~/.config/agents/skills/` | Markdown |
| Kilo Code | `~/.kilocode/skills/` | Markdown |
| Roo Code | `~/.roo/skills/` | Markdown |
| Goose | `~/.config/goose/skills/` | Markdown |
| Gemini CLI | `~/.gemini/skills/` | TOML (flat) |
| Antigravity | `~/.gemini/antigravity/skills/` | Markdown |
| Droid | `~/.factory/skills/` | Markdown |
| Windsurf | `~/.codeium/windsurf/skills/` | Markdown |
| Cline | `~/.cline/skills/` | Markdown |
| Aider | `~/.aider/skills/` | Markdown |
| Continue | `~/.continue/skills/` | Markdown |
| Zed | `~/.zed/skills/` | Markdown |
| Grok CLI | `~/.grok/skills/` | Markdown |

### Federated/Third-Party Agents (explicit opt-in)
These agents require explicit opt-in flags and are NOT auto-detected:

| Agent | Flag | Installation Path | Notes |
|-------|------|-------------------|-------|
| **OpenCode** | `-a opencode` | `~/.config/opencode/skills/` | Commands: flat `.md` files |
| **GitHub Copilot** | `-a copilot` | `~/.copilot/skills/` | Federated agent |
| **OpenClaw** | `-a openclaw` | `~/.openclaw/skills/` | Federated agent |
| **thClaws** | `--with-thclaws` or `-a thclaws` | `~/.config/thclaws/skills/` | Federated agent; detect by binary |

**Examples:**
```bash
# Install to Claude Code + OpenCode
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -a claude-code opencode

# Install to thClaws only
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y --thclaws-only

# Install to all detected agents, including federated
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y --all-detected
```

---

## Skill Profiles

Profiles are predefined collections of skills bundled by use case. Profiles are cumulative — upgrading always adds or keeps, never removes.

| Profile | Count | Contents | Best For |
|---------|-------|----------|----------|
| **minimal** | 7 | Essential lifecycle + identity + discovery | Newcomers, minimal footprint |
| **standard** | 20 | Daily-driver skills (usage-driven curation) | Most users |
| **full** | 29 | All production skills (excludes experimental) | Power users |
| **lab** | 32 | Everything including experimental/bleeding-edge | Developers, testing |

**Minimal includes:**
about-oracle, forward, go, recap, rrr, trace, who-are-you

**Standard includes (superset of minimal):**
about-oracle, awaken, bampenpien, bud, create-shortcut, dig, forward, go, incubate, learn, oracle-cheatsheet, oracle-family-scan, oracle-prism, oracle-write-complete-book, recap, resonance, rrr, trace, where-we-are, who-are-you

**Lab-only additions (not in standard or full):**
dream, fyi, watch

**Zombie/archived skills** (40 total):
Excluded from all profiles. Install by name only with `-s` flag. Examples: alpha-feature, birth, deep-research, gemini, handover, my, workon, etc.

**Switch profiles anytime:**
```bash
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -p standard
```

---

## CLI Commands & Usage

### Primary Commands

#### `install` — Install skills (default command)
Install skills to detected agents or specific targets.

```bash
arra-oracle-skills install [options]
```

**Common options:**
```bash
-g, --global              Install to user directory (default: project-scoped)
-l, --local               Explicitly target project .claude/skills/ (default behavior)
-p, --profile <name>      Choose profile: minimal|standard|full|lab (default: minimal)
-a, --agent <agents...>   Target specific agents (e.g., -a claude-code opencode cursor)
-s, --skill <skills...>   Add specific skills by name (additive to profile)
-y, --yes                 Skip confirmation prompts (non-interactive)
--list                    List all available skills without installing
--with-commands           Also install command stubs to ~/.claude/commands/ (OpenCode, Codex, Gemini need this)
--force-global            Override local-skill precedence (#230)
--with-thclaws            Include thClaws if detected (federated opt-in)
--thclaws-only            Install ONLY to thClaws paths (testing escape hatch)
--all-detected            Install to ALL detected agents including federated (CI escape hatch)
--shell                   Force Bun shell commands (Windows testing)
--no-shell                Force Node.js fs operations (Unix compatibility)
```

**Examples:**
```bash
# First time: interactive, ask which agents
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g

# Silent install: minimal to detected agents
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y

# Install full + 2 extra skills
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -p full -s fyi

# Install to specific agents (don't auto-detect)
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -a claude-code cursor

# Project-local install
cd myproject && bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -y

# Install just a few skills
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -s recap rrr trace
```

#### `uninstall` — Remove skills
Remove installed skills from agents.

```bash
arra-oracle-skills uninstall [options]
```

**Options:**
```bash
-g, --global              Remove from user directory
-l, --local               Remove from project .claude/skills/
-a, --agent <agents...>   Target specific agents
-s, --skill <skills...>   Remove specific skills only (all if omitted)
-y, --yes                 Skip confirmation
--thclaws-only            Remove ONLY from thClaws
--shell, --no-shell       Shell mode override
```

**Examples:**
```bash
# Remove all skills globally
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha uninstall -g -y

# Remove specific skill from project
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha uninstall -l -y -s fyi

# Remove from specific agents
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha uninstall -g -y -a claude-code codex
```

#### `list` — Show installed skills
Display all installed skills and their versions.

```bash
arra-oracle-skills list [options]
```

**Options:**
```bash
-g, --global              Show global skills (default: project-scoped)
-a, --agent <agents...>   Show skills for specific agents only
```

**Example output:**
```
Installed Oracle skills:

  Claude Code (global): 7 skills
    - about-oracle (v26.7.26)
    - forward (v26.7.26)
    - go (v26.7.26)
    - recap (v26.7.26) [hidden]
    - rrr (v26.7.26)
    - trace (v26.7.26)
    - who-are-you (v26.7.26)

  Codex (global): 7 skills
    - about-oracle (v26.7.26)
    ...

Total: 14 skills across 2 agent(s)
```

#### `profiles` — List available skill profiles
Show all profiles and their contents.

```bash
arra-oracle-skills profiles [name]
```

**Examples:**
```bash
# List all profiles with counts
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha profiles

# Show details of a specific profile
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha profiles standard
```

#### `agents` — List supported agents
Show all 20 agents and detect which are installed.

```bash
arra-oracle-skills agents
```

**Output shows:**
- Agent name and display name
- ✓ if detected on this system
- [ ] if not detected

#### `select` — Interactive skill picker
Interactively choose skills to install.

```bash
arra-oracle-skills select [options]
```

Useful for GUI-like interaction instead of flag-heavy CLI.

#### `init` — First-time setup
Install minimal profile globally and validate prerequisites.

```bash
arra-oracle-skills init [options]
```

Checks for Bun, Claude Code, and sets up the minimal profile.

#### `about` — Version & system status
Show version, prerequisites check, and system information.

```bash
arra-oracle-skills about
```

#### `inspect` — Deep skill inspection
Show where a skill lives, which profiles include it, and metadata.

```bash
arra-oracle-skills inspect [skill]
```

**Example:**
```bash
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha inspect about-oracle
```

#### `xray` — System deep-scan
Scan Oracle memory, auto-memory, installations, and hooks.

```bash
arra-oracle-skills xray [target] [project]
```

**Targets:** memory, auto-memory, installations, hooks

#### `shortcut` — Manage local command shortcuts
Create, list, or delete local skill shortcuts.

```bash
arra-oracle-skills shortcut [action] [name] [command]
```

#### `contacts` — Manage Oracle federation contacts
Add, list, or remove Oracle agents for cross-oracle communication.

```bash
arra-oracle-skills contacts [action] [name]
```

#### `awaken` — Oracle birth ritual
Start guided Oracle setup for a fresh repository.

```bash
arra-oracle-skills awaken
```

---

## Configuration & Advanced Options

### Installation Locations

Skills install to agent-specific directories:

**Global (user-level):**
```
~/.claude/skills/                  # Claude Code
~/.codex/skills/                   # Codex
~/.cursor/skills/                  # Cursor
~/.config/opencode/skills/         # OpenCode
~/.config/goose/skills/            # Goose
~/.gemini/skills/                  # Gemini CLI
~/.cline/skills/                   # Cline
... and more (see Supported Agents table)
```

**Project-local:**
```
./.claude/skills/                  # Claude Code
./.codex/skills/                   # Codex
./.opencode/skills/                # OpenCode
... (mirrors global structure)
```

### Skill Metadata (SKILL.md)

Each skill is a directory with a `SKILL.md` file:

```
skills/
├── about-oracle/
│   ├── SKILL.md              # Skill definition + description
│   ├── index.ts              # Implementation
│   └── ...
├── forward/
│   ├── SKILL.md
│   └── ...
```

**SKILL.md frontmatter markers:**
```markdown
---
hidden: true              # Hide from command autocomplete (install but don't expose)
secret: true              # Exclude from all profiles (install by name only)
zombie: true              # Archived skill (in .archive/, install by name only)
v26.7.27-alpha.947        # Version (displayed in `list` output)
---
```

### Manifest Files

Installation tracked via `.arra-oracle-skills.json` manifest in each agent's skills directory:

```json
{
  "version": "26.7.27-alpha.947",
  "installed": ["about-oracle", "forward", "go", ...],
  "timestamp": "2026-07-28T08:06:00Z"
}
```

Used to:
- Track what's installed (for updates via `/go`)
- Detect already-installed agents (with `-y` flag)
- Prevent duplicates

### Environment Variables

**Installation options (set before running):**

| Variable | Effect | Example |
|----------|--------|---------|
| `ORACLE_SKILLS_VERSION` | Pin installer to specific release | `ORACLE_SKILLS_VERSION=v26.7.0 bunx ...` |
| `ORACLE_SKILLS_USE_BUNX` | Skip binary, always use bunx | `ORACLE_SKILLS_USE_BUNX=1 bash install.sh` |

### Shell Compatibility

Most systems auto-detect (Bun or Node.js). Override if needed:

```bash
--shell                   # Force Bun.$ shell (Windows testing)
--no-shell                # Force Node.js fs operations (Unix compatibility)
```

---

## Full Skill List (22 production skills)

| Skill | Type | Description |
|-------|------|-------------|
| about-oracle | Skill + Subagent | What is Oracle — orientation |
| awaken | Skill | Guided Oracle birth and awakening ritual |
| bampenpien | Skill | บำเพ็ญเพียร (Thai spiritual practice) |
| bud | Skill | Create a new oracle via maw bud |
| create-shortcut | Skill | Create local skills as shortcuts |
| dig | Skill | Mine Claude Code sessions for knowledge |
| forward | Skill + Subagent | Hand off work to the next oracle |
| go | Skill | Manage Oracle skills (update, switch profiles) |
| incubate | Skill | Clone or create repos for active development |
| learn | Skill + Subagent | Explore codebase with parallel Haiku agents |
| oracle-cheatsheet | Skill | Generate copy-paste cheat sheet from docs |
| oracle-family-scan | Skill + Code | Oracle Family Registry — detect other oracles |
| oracle-prism | Skill | Multi-perspective analysis |
| oracle-write-complete-book | Skill | Generate a complete book/guide from context |
| philosophy | Skill | Display Oracle philosophy (5 Principles + Rule 6) |
| project | Skill + Code | Clone and track external repos |
| recap | Skill + Code | Session orientation and awareness |
| resonance | Skill | Capture a resonance moment |
| rrr | Skill + Subagent | Create session retrospective with AI diary |
| trace | Skill | Find projects, code, and knowledge across repos |
| where-we-are | Skill | Session awareness |
| who-are-you | Skill | Know ourselves — identity check |

**Archived (zombie) skills** — 40 available by name via `-s` flag:  
alpha-feature, birth, deep-research, gemini, handover, mine, new-issue, oracle-manage, speak, what-we-done, whats-next, workon, i-believed, work-with, morpheus, retrospective, skills-list, fleet, machines, warp, release, wormhole, harden, vault, dream-original, oracle-soul-sync-update, forward-lite, recap-lite, rrr-lite, oracle-up, schedule, worktree, standup, xray, feel, hey, contacts, mailbox, inbox, and others.

---

## Common Workflows

### First-time installation
```bash
# Interactive setup (asks which agents)
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g

# Or silent with defaults
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y
```

### Upgrade to a higher profile
```bash
# From minimal to standard
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -p standard

# From standard to full
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -p full
```

### Add experimental skills
```bash
# Install full profile + lab-only skills
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -p lab
```

### Pick just a few skills
```bash
# Essential skills for session management
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -s recap rrr forward trace
```

### Install to multiple agents
```bash
# Claude Code + OpenCode + Cursor
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y -a claude-code opencode cursor -p standard
```

### Project-local skills (for this repo only)
```bash
# Install to ./.claude/skills/ instead of ~/.claude/skills/
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -y -p standard
```

### Inspect what's installed
```bash
# Show all global skills
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha list -g

# Show skills for a specific agent
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha list -g -a claude-code

# Deep system scan
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha xray memory
```

### Manage skills via slash commands (after install)
Once installed, use these slash commands in your agent:

```bash
/go minimal              # Switch to minimal profile
/go standard             # Switch to standard profile
/go full                 # Switch to full profile
/go lab                  # Switch to lab profile
/go list                 # List installed skills
```

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "arra-oracle-skills requires Bun" | Install Bun: `curl -fsSL https://bun.sh/install \| bash` |
| `npx arra-oracle-skills` doesn't work | Use `bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha` instead |
| Skills not showing up after install | Restart your agent. Check installation: `arra-oracle-skills list -g` |
| Agent not detected | Use explicit `-a <agent>` flag or check if the agent is installed |
| Installation hung on permissions | Add `-y` flag to skip prompts, or run with `--no-shell` |
| "local skill precedence" warning | Use `--force-global` to override and install anyway |
| Old skills still showing | Skills in older installations may not auto-update; reinstall with fresh profile |

---

## Notes & Known Issues

- **install.sh URL:** The compiled binary installer references `/oracle-skills-cli` in its download URLs, but this repo is `arra-oracle-skills-cli`. The `.sh` script handles this via GitHub API calls; direct URLs may not work.
- **npm lag:** npm publishes lag behind GitHub releases. For cutting-edge skills, use `bunx --bun github:...#alpha`.
- **Federated agents:** OpenCode, GitHub Copilot, OpenClaw, and thClaws are third-party agents and require explicit opt-in (`--with-thclaws`, `-a opencode`, etc.) to avoid cross-contamination.
- **Node.js not supported:** This CLI runs on Bun runtime only. The compiled binary bundles Bun; bunx users need Bun installed.
- **Profile invariant:** Profiles are always additive — upgrading from minimal to standard always keeps all minimal skills and adds more.

---

## Further Reading

- **GitHub Repository:** https://github.com/Soul-Brews-Studio/arra-oracle-skills-cli
- **Package:** https://www.npmjs.com/package/arra-oracle-skills
- **Author:** Nat Weerawan (Soul Brews Studio)
- **License:** MIT

---

**Last updated:** 2026-07-28 | **CLI version:** v26.7.27-alpha.947
