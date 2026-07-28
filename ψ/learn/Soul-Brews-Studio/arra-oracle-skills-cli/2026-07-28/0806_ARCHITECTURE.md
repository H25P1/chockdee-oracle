# arra-oracle-skills-cli Architecture

**Version:** 26.7.27-alpha.947  
**Purpose:** Install Oracle skills to Claude Code, OpenCode, Cursor, and 19+ AI coding agents  
**Language:** TypeScript (Bun runtime)  
**Build:** Bun bundler to ES2022 binary

---

## Directory Structure & Organization

```
arra-oracle-skills-cli/
├── src/                          # Source code (TypeScript)
│   ├── cli/                       # CLI entry point & command implementations
│   │   ├── index.ts              # Main CLI program entry; registers all commands
│   │   ├── types.ts              # TypeScript interfaces (AgentConfig, Skill, InstallOptions)
│   │   ├── agents.ts             # Agent registry & detection (19 agents, federated vs. host)
│   │   ├── installer.ts          # Core installation logic & skill discovery
│   │   ├── skill-source.ts        # Unified skill access (filesystem or VFS/compiled mode)
│   │   ├── fs-utils.ts           # Cross-platform file operations (Bun.$ or Node.js fs)
│   │   └── commands/             # CLI command implementations
│   │       ├── install.ts        # `install` command (default)
│   │       ├── uninstall.ts      # `uninstall` command
│   │       ├── select.ts         # Interactive skill picker
│   │       ├── list.ts           # List installed skills
│   │       ├── profiles.ts       # Show available profiles
│   │       ├── agents.ts         # Show supported agents
│   │       ├── about.ts          # Version + system status
│   │       ├── awaken.ts         # Oracle awakening ritual
│   │       ├── xray.ts           # Deep system inspection
│   │       ├── shortcut.ts       # Create local skill shortcuts
│   │       └── contacts.ts       # Manage Oracle contacts
│   ├── hooks/                     # Pre/post-install hooks
│   │   └── opencode/             # OpenCode-specific hooks
│   ├── skills/                    # Vault: secret & archive skills
│   │   └── .archive/             # Zombie skills (opt-in only)
│   └── profiles.ts               # Skill profiles (minimal, standard, full, lab)
│
├── skills/                        # Public shelf: curated skills (every channel serves this)
│   ├── about-oracle/             # Sample skill structure
│   │   └── SKILL.md             # Skill definition with frontmatter
│   ├── awaken/
│   ├── forward/
│   ├── recap/
│   ├── rrr/
│   ├── trace/
│   ├── who-are-you/
│   ├── ... (18+ more)
│   └── CONVENTIONS.md            # Skill authoring standards
│
├── scripts/                       # Build & utility scripts
│   ├── compile.ts                # Validate frontmatter & generate marketplace manifest
│   ├── generate-vfs.ts           # Generate virtual filesystem for compiled binary
│   ├── ensure-vfs-stub.ts        # Create VFS stub if real one doesn't exist
│   ├── calver.ts                 # Calendar versioning
│   └── update-readme-table.ts    # Auto-update skill list in README.md
│
├── hooks/                         # Pre/post-commit hooks for development
│   ├── safety-check.sh           # Verify no broken symlinks or bad configs
│   ├── session-start.sh          # Session initialization
│   ├── statusline-command.sh     # Status updates
│   └── BEST_PRACTICES.md
│
├── tests-e2e/                     # End-to-end tests (agent installation flows)
├── __tests__/                     # Unit tests
│
├── .claude/                       # Claude Code configuration (development)
├── .claude-plugin/                # Claude plugin marketplace definition
├── .opencode/                     # OpenCode configuration
├── .github/                       # GitHub workflows
│
├── package.json                   # Dependencies, scripts, bin entry
├── tsconfig.json                  # TypeScript configuration
├── bun.lock                        # Bun lock file
├── bunfig.toml                    # Bun runtime configuration
└── lefthook.yml                   # Git hooks configuration

```

### Dual-Root Skill Layout

**Public Shelf** (`skills/`): Curated skills every external channel serves
- GitHub marketplace integration
- npm package distribution
- Claude plugin marketplace

**Vault** (`src/skills/`): Internal skills repository
- Secret skills (excluded from all profiles)
- `.archive/` subdirectory: zombie skills (internal development candidates)
- `.template/`: skill authoring templates

This split allows skills to be packaged, versioned, and published independently from infrastructure-specific or experimental code.

---

## Entry Points

### CLI Binary

**Location:** `src/cli/index.ts`  
**Package.json entry:** `"bin": { "arra-oracle-skills": "./src/cli/index.ts" }`

Runtime check ensures Bun is available or the binary is compiled. Uses `commander.js` for command registration.

```typescript
// Bun shebang
#!/usr/bin/env bun

// Verify Bun or compiled binary
if (!(typeof IS_COMPILED !== 'undefined' && IS_COMPILED) && typeof Bun === 'undefined') {
  console.error('arra-oracle-skills requires Bun runtime');
  process.exit(1);
}

import { program } from 'commander';
program.name('arra-oracle-skills')
  .description('Install Oracle skills to Claude Code, OpenCode, Cursor, and 11+ AI coding agents')
  .version(VERSION);

// Register all commands
registerAgents(program);
registerInstall(program, VERSION);
registerInit(program, VERSION);
// ... etc
```

### Main Commands

**Default:** `install` (skill installation)

**Full command set:**
- `install [options]` — install skills (default profile: minimal)
- `uninstall [options]` — remove installed skills
- `select [options]` — interactive skill picker
- `list [options]` — show installed skills
- `profiles [name]` — list profiles
- `agents` — list supported agents
- `about` — version + status
- `awaken` — guided Oracle awakening ritual (new repo setup)
- `xray` — deep system inspection
- `shortcut` — create local skill shortcuts
- `init` — initialize new Oracle
- `contacts` — manage Oracle contacts

Each command is registered in `src/cli/commands/<name>.ts` as a function that takes the commander `program` object.

---

## Core Abstractions

### 1. Agent Detection & Targeting

**File:** `src/cli/agents.ts`

Abstracts how the CLI discovers and targets different AI agents. Each agent has a configuration describing:
- Where skills live (project-local vs. user-global paths)
- How to detect if the agent is installed
- File format (flat markdown `.md` or nested `SKILL.md`)
- Whether commands are opt-in or auto-installed
- Federated vs. host agent status

```typescript
export const agents: Record<AgentType, AgentConfig> = {
  'claude-code': {
    name: 'claude-code',
    displayName: 'Claude Code',
    skillsDir: '.claude/skills',           // project-local
    globalSkillsDir: join(home, '.claude/skills'),  // ~/.claude/skills
    commandsDir: '.claude/commands',
    globalCommandsDir: join(home, '.claude/commands'),
    useFlatFiles: true,
    commandsOptIn: true,                   // commands require --with-commands flag
    detectInstalled: () => existsSync(join(home, '.claude')),
  },
  'opencode': {
    name: 'opencode',
    displayName: 'OpenCode',
    skillsDir: '.opencode/skills',
    globalSkillsDir: join(home, '.config/opencode/skills'),
    commandsDir: '.opencode/commands',
    globalCommandsDir: join(home, '.config/opencode/commands'),
    useFlatFiles: true,
    federated: true,                       // third-party agent (explicit -a required)
    detectInstalled: () => existsSync(join(home, '.config/opencode')),
  },
  // ... 17 more agents
};
```

**Supported Agents (19 total):**
- **Host agents (auto-detected):** Claude Code, Codex
- **Third-party/federated (opt-in only):** OpenCode, OpenClaw, GitHub Copilot, thClaws, and others
- **IDE plugins:** Cursor, Cline, Continue, Windsurf, Zed
- **Specialized:** Gemini CLI, Amp, Kilo Code, Roo Code, Goose, Antigravity, Droid, Aider, Grok

**Federation Policy (#330):** Federated agents (third-party code editors) require explicit opt-in via `-a <agent>`, `--with-thclaws`, or `--all-detected` to prevent unwanted installations. Only host Anthropic agents (Claude Code, Codex) auto-install.

### 2. Skill Discovery & Resolution

**Files:** `src/cli/skill-source.ts`, `src/cli/installer.ts`

**Dual-mode skill access:**
- **Filesystem mode (dev):** Reads from `skills/` (public shelf) → `src/skills/` (vault) → `src/skills/.archive/` (zombies)
- **VFS mode (compiled):** Reads from generated `generated/skills-vfs.js` (virtual filesystem embedded in binary)

```typescript
// Filesystem resolution order:
function resolveSkillDir(skillName: string): string {
  const shelf = join(getPublicSkillsDir(), skillName);      // skills/<name>
  if (existsSync(shelf)) return shelf;
  
  const vault = join(getVaultSkillsDir(), skillName);       // src/skills/<name>
  if (existsSync(vault)) return vault;
  
  const archived = join(vaultRoot, '.archive', skillName);  // src/skills/.archive/<name>
  if (existsSync(archived)) return archived;
  
  return vault; // fallback for not-found
}
```

Each skill is a directory with:
- `SKILL.md` — Skill definition with frontmatter (name, description, argument hints)
- Optional hook files (subagent definitions, tool specifications)

### 3. Skill Installation Pipeline

**File:** `src/cli/installer.ts`

**Install flow:**
1. **Discovery** — collect all available skills (from public shelf or VFS)
2. **Profile resolution** — map profile name (minimal/standard/full/lab) to skill list
3. **Filtering** — exclude secrets and zombies unless explicitly requested
4. **Preview** — show user what will be installed before confirmation
5. **Agent targeting** — determine which agents to install to (default auto-detected or explicit)
6. **Local precedence check** — if local skill exists, skip global unless `--force-global`
7. **Per-agent installation** — copy/symlink skills to each agent's config directory
8. **Command stub generation** — for agents with command support (OpenCode, Codex)

```typescript
// Agent-specific install paths
claude-code:    ~/.claude/skills/<name>/SKILL.md
opencode:       ~/.config/opencode/skills/<name>.md        (flat file)
codex:          ~/.codex/skills/<name>/SKILL.md
cursor:         ~/.cursor/skills/<name>/SKILL.md
// ... and so on
```

### 4. Profile Management

**File:** `src/profiles.ts`

Four tiers of skills:

| Profile | Count | Purpose |
|---------|-------|---------|
| **minimal** | 7 | Essential for newcomers: lifecycle + trace + identity |
| **standard** | 20 | Daily driver skills based on usage audit (10+ session appearances) |
| **full** | 21+ | All stable skills (excludes lab-only experiments) |
| **lab** | 21+ | Everything including experimental/bleeding-edge |

**Tier Invariant:** `minimal ⊆ standard ⊆ full`. Upgrading a profile must never remove a skill you already had (enforced by test).

**Special categories:**
- **Zombie skills:** Internal development candidates, excluded from all profiles; install by name only (`-s <name>`)
- **Secret skills:** Same as zombies (excluded globally, opt-in only)
- **Hidden skills:** Installed silently without command stub in autocomplete

### 5. Cross-Platform File Operations

**File:** `src/cli/fs-utils.ts`

Abstracts shell vs. filesystem operations for cross-platform compatibility:

```typescript
export type ShellMode = 'auto' | 'shell' | 'no-shell';

// Shell mode (Unix): uses Bun.$ for atomic operations
await mkdirp(dir, 'auto');  // mkdir -p on Unix, mkdirSync on Windows

// No-shell mode (Windows): uses Node.js fs API directly
await rmrf(path, 'shell');  // rm -rf on Unix, rmSync on Windows
```

Defaults to shell on Unix (atomic, robust to symlinks), fs API on Windows (avoids shell escaping issues).

---

## Dependencies

### Runtime Dependencies

```json
{
  "@clack/prompts": "^0.7.0",   // Terminal UI (spinners, selections, confirmations)
  "commander": "^12.0.0",        // CLI argument parsing & command registration
  "mqtt": "^5.14.1"              // MQTT WebSocket for real-time features
}
```

### Dev Dependencies

```json
{
  "@types/bun": "^1.3.6",
  "@types/node": "^20.0.0",
  "lefthook": "^2.0.15",         // Git hooks manager (pre-commit, safety checks)
  "typescript": "^5.0.0",
  "yaml": "^2.9.0"               // YAML parsing (skill frontmatter validation)
}
```

### Runtime Environment

- **Bun:** Default runtime (JavaScript built-in, native performance)
- **Node.js:** ≥18 (for fallback, npm distribution)
- **Compilation:** Bun build → standalone binary (ES2022 target)

---

## Key Modules & Responsibilities

### CLI Core (`src/cli/index.ts`)

- Program initialization with version & description
- Command registration in priority order (agents first for discovery)
- Bun runtime verification (exit with helpful error if Node.js only)

### Types (`src/cli/types.ts`)

- `AgentConfig` — agent detection & path configuration
- `Skill` — skill metadata (name, description, path, flags)
- `InstallOptions` — parsed CLI arguments for installation
- `AgentType` — discriminated union of 19 agent names

### Agents Registry (`src/cli/agents.ts`)

- `agents` object: 19 agent configurations keyed by name
- `detectInstalledAgents()` — scan home directory for installed agents
- `getDefaultAgents()` — return non-federated agents to auto-install to
- Federation detection: `thClawsAvailable()`, `grokAvailable()` via shell commands
- Agent version detection: `getCodexVersion()` for backwards-compatible plugin format selection

### Skill Source (`src/cli/skill-source.ts`)

- Unified skill access: filesystem or VFS (compiled)
- `discoverSkills()` — scan skills/ + src/skills/ directories
- `readSkillFile()` — read SKILL.md from any mode
- `writeSkillToDir()` — copy skill to agent's config directory
- `extractDescription()` — parse YAML frontmatter (handles block scalars)
- `isCompiled()` — detect if running as binary vs. dev mode

### Installer (`src/cli/installer.ts`)

- `installSkills(targets, options)` — core installation logic
- `uninstallSkills(targets, options)` — skill removal
- `listSkills(targets)` — show currently installed skills
- Per-agent installation paths & file format handling
- Codex marketplace detection & plugin cache management (v0.128 TOML vs. v0.130+ JSON)
- Self-target guard: prevents `openclaw`'s `skillsDir: 'skills'` from clobbering skill source
- Local skill precedence: skip global if same-named skill exists locally (unless `--force-global`)

### File Operations (`src/cli/fs-utils.ts`)

- `mkdirp()`, `rmrf()`, `cpr()`, `mv()`, `rmf()`, `cp()`
- ShellMode routing: Bun.$ on Unix, Node.js fs on Windows
- Cross-platform safety: no shell escaping issues on Windows, atomic operations on Unix

### Profiles (`src/profiles.ts`)

- Single source of truth for skill tiers
- `resolveProfile(name, allSkills)` — map profile to skill list
- `skillDirFor(name, root)` — resolve zombie vs. active skill paths
- Tier invariant validation: `minimal ⊆ standard` (locked by tests)

### Commands (`src/cli/commands/`)

Each command module exports a registration function:

```typescript
export function register<Command>(program: Command, version: string) {
  program
    .command('<command>', { isDefault: true })
    .description('...')
    .option('...')
    .action(async (options, cmd) => { /* implementation */ });
}
```

**Key command patterns:**
- `install.ts` — skill selection preview (#337), profile resolution, agent detection
- `uninstall.ts` — safe removal (checks not self-target)
- `select.ts` — interactive checkboxes for skills & agents
- `about.ts` — system status, version, prerequisites check
- `xray.ts` — deep introspection (installed skills, config files, permissions)

---

## Build Pipeline

### Development

```bash
bun run dev                   # Run CLI from source (src/cli/index.ts)
bun test                      # Unit tests
bun test:e2e                  # End-to-end (real agent installations)
```

### Production Build

```bash
bun run build                 # Bun bundler: src/cli/index.ts → dist/cli.js
bun run compile              # Validate frontmatter + generate marketplace.json
```

### Pre-build Steps

1. **VFS Generation** (`scripts/ensure-vfs-stub.ts`)
   - Creates `src/cli/generated/skills-vfs.js` if missing
   - Stubs can be replaced by real VFS during native build
   - Embedded in compiled binary for zero-dependency execution

2. **Frontmatter Validation** (`scripts/compile.ts`)
   - Validates skill name (≤64 chars, lowercase/digits/hyphens)
   - Validates description (non-empty, ≤1024 chars, no XML tags)
   - Warns on large skill bodies (>500 lines)
   - Generates `.claude-plugin/marketplace.json` for plugin marketplace

3. **README Update** (`scripts/update-readme-table.ts`)
   - Auto-generates skill table in README.md
   - Ensures documentation stays in sync with source

### Installed Binary

```bash
# Via Bun
bunx --bun github:Soul-Brews-Studio/arra-oracle-skills-cli#alpha install -g -y

# Via npm (lagging mirror)
npx arra-oracle-skills@latest install -g -y

# Via shell installer (pre-compiled binary)
curl -fsSL https://raw.githubusercontent.com/.../install.sh | bash
```

---

## Hooks & Integration

### OpenCode Hooks (`src/hooks/opencode/`)

Agent-specific integration points (pre/post-install, configuration updates).

### Git Hooks (`hooks/`, configured in `lefthook.yml`)

- `safety-check.sh` — verify no broken symlinks before commit
- `session-start.sh` — initialize session state
- `statusline-command.sh` — update development status

---

## Plugin Marketplace Integration

**File:** `.claude-plugin/marketplace.json`  
**Generated by:** `scripts/compile.ts`

Manifest for:
- Claude plugin marketplace (official distribution channel)
- `npx skills add Soul-Brews-Studio/arra-oracle-skills-cli` (npm distribution)
- Plugin version tracking & compatibility metadata

---

## Testing Strategy

### Unit Tests (`__tests__/`)
- Frontmatter validation
- Profile resolution & tier invariants
- Agent detection & default selection
- Cross-platform path handling

### E2E Tests (`tests-e2e/`)
- Real agent installations (CLI against actual agent configs)
- Skill copying to global/local paths
- Command stub generation
- Marketplace plugin installation

---

## Summary

**arra-oracle-skills-cli** is a sophisticated multi-agent skill installer that:

1. **Abstracts agent diversity** — 19 agents with different config formats, paths, and capabilities via a unified `AgentConfig` registry
2. **Manages skill tiers** — profiles (minimal/standard/full/lab) with tier invariants to prevent silent downgrades
3. **Supports multiple distribution modes** — npm, GitHub, compiled binary, plugin marketplace
4. **Handles complex installation scenarios** — local precedence, federated agent opt-in, Codex plugin cache versioning
5. **Prioritizes UX** — interactive skill picker, preview before install, deep system inspection (`xray`)
6. **Maintains high code quality** — TypeScript strict mode, comprehensive testing, pre-commit hooks

The architecture separates concerns cleanly: agent detection (agents.ts), skill discovery (skill-source.ts), installation logic (installer.ts), and CLI scaffolding (commands/), allowing each layer to evolve independently.
