# arra-oracle-skills-cli: Code Architecture & Implementation

**Repository**: Soul-Brews-Studio/arra-oracle-skills-cli  
**Language**: TypeScript (Bun runtime)  
**Purpose**: CLI tool that installs Oracle skills to Claude Code, OpenCode, Cursor, Codex, and 15+ AI coding agents  
**Version**: 26.7.27-alpha.947

---

## 1. Project Overview

**arra-oracle-skills-cli** is a universal skill installer that works with multiple AI coding agents by abstracting away agent-specific installation paths and file formats. It follows an **adapter pattern** where each agent has its own configuration that maps to standardized operations.

### Key Statistics
- **30+ skills** in the public library
- **16+ supported agents** (Claude Code, OpenCode, Cursor, Codex, Gemini CLI, GitHub Copilot, and more)
- **Multiple installation profiles** (minimal, standard, full, lab)
- **VFS + Binary distribution** (compiled Bun binary or runtime via bunx)

---

## 2. Main CLI Entry Point

**File**: `/src/cli/index.ts`

This is the CLI bootstrap that loads Commander and registers all subcommands.

```typescript
#!/usr/bin/env bun

// Build-time define for compiled binaries
declare const IS_COMPILED: boolean;

// Bun runtime check - skip in compiled mode (binary embeds Bun)
try {
  if (!(typeof IS_COMPILED !== 'undefined' && IS_COMPILED) && typeof Bun === 'undefined') {
    console.error(`
❌ arra-oracle-skills requires Bun runtime

You're running with Node.js, but this CLI uses Bun-specific features.

To fix:
  1. Install Bun: curl -fsSL https://bun.sh/install | bash
  2. Run with: bunx arra-oracle-skills install -g -y

Or install the compiled binary (no Bun needed):
  curl -fsSL https://raw.githubusercontent.com/Soul-Brews-Studio/arra-oracle-skills-cli/main/install.sh | bash

More info: https://bun.sh
`);
    process.exit(1);
  }
} catch {
  // IS_COMPILED not defined — running in dev mode, check passed
}

import { program } from 'commander';
import pkg from '../../package.json' with { type: 'json' };

import { registerInstall } from './commands/install.js';
import { registerInit } from './commands/init.js';
import { registerUninstall } from './commands/uninstall.js';
import { registerSelect } from './commands/select.js';
import { registerAgents } from './commands/agents.js';
import { registerList } from './commands/list.js';
import { registerProfiles } from './commands/profiles.js';
import { registerAbout } from './commands/about.js';
import { registerAwaken } from './commands/awaken.js';
import { registerXray } from './commands/xray.js';
import { registerShortcut } from './commands/shortcut.js';
import { registerContacts } from './commands/contacts.js';

const VERSION = pkg.version;

program
  .name('arra-oracle-skills')
  .description('Install Oracle skills to Claude Code, OpenCode, Cursor, and 11+ AI coding agents')
  .version(VERSION);

// Register all commands (agents first — most useful for discovery)
registerAgents(program);
registerInstall(program, VERSION);
registerInit(program, VERSION);
registerUninstall(program, VERSION);
registerSelect(program, VERSION);
registerList(program);
registerProfiles(program);
registerAbout(program, VERSION);
registerAwaken(program, VERSION);
registerXray(program, VERSION);
registerShortcut(program);
registerContacts(program);

program.parse();
```

**Entry point**: Declared in `package.json` bin field:
```json
{
  "bin": {
    "arra-oracle-skills": "./src/cli/index.ts"
  }
}
```

---

## 3. Agent Configuration & Adapter Pattern

**File**: `/src/cli/agents.ts`

The adapter pattern is implemented via an `AgentConfig` interface. Each supported agent has a configuration object that maps agent-specific directory structures to a unified installer interface.

### Agent Registry

```typescript
import { homedir } from 'os';
import { join } from 'path';
import { existsSync } from 'fs';
import type { AgentConfig, AgentType } from './types.js';

const home = homedir();

export const agents: Record<AgentType, AgentConfig> = {
  // --- Host Agents (auto-detected) ---
  
  'claude-code': {
    name: 'claude-code',
    displayName: 'Claude Code',
    skillsDir: '.claude/skills',                    // Local project skills
    globalSkillsDir: join(home, '.claude/skills'),  // User global skills
    commandsDir: '.claude/commands',                // Local slash commands
    globalCommandsDir: join(home, '.claude/commands'),
    useFlatFiles: true,
    commandsOptIn: true,  // Only install commands with --commands flag
    detectInstalled: () => existsSync(join(home, '.claude')),
  },

  codex: {
    name: 'codex',
    displayName: 'Codex',
    skillsDir: '.codex/skills',
    globalSkillsDir: join(home, '.codex/skills'),
    commandsDir: '.codex/prompts',  // Codex uses "prompts" dir instead of "commands"
    globalCommandsDir: join(home, '.codex/prompts'),
    useFlatFiles: true,
    detectInstalled: () => existsSync(join(home, '.codex')),
  },

  // --- Third-Party / Federated Agents (explicit opt-in) ---
  
  opencode: {
    name: 'opencode',
    displayName: 'OpenCode',
    skillsDir: '.opencode/skills',
    globalSkillsDir: join(home, '.config/opencode/skills'),
    commandsDir: '.opencode/commands',
    globalCommandsDir: join(home, '.config/opencode/commands'),
    useFlatFiles: true,
    federated: true,  // #330: third-party — requires explicit -a opencode or --all-detected
    detectInstalled: () => existsSync(join(home, '.config/opencode')),
  },

  cursor: {
    name: 'cursor',
    displayName: 'Cursor',
    skillsDir: '.cursor/skills',
    globalSkillsDir: join(home, '.cursor/skills'),
    detectInstalled: () => existsSync(join(home, '.cursor')),
  },

  gemini: {
    name: 'gemini',
    displayName: 'Gemini CLI',
    skillsDir: '.gemini/skills',
    globalSkillsDir: join(home, '.gemini/skills'),
    commandsDir: '.gemini/commands',
    globalCommandsDir: join(home, '.gemini/commands'),
    useFlatFiles: true,
    commandFormat: 'toml',  // Gemini uses TOML instead of Markdown
    detectInstalled: () => existsSync(join(home, '.gemini')),
  },

  thclaws: {
    name: 'thclaws',
    displayName: 'thClaws',
    skillsDir: '.thclaws/skills',
    globalSkillsDir: join(home, '.config/thclaws/skills'),
    federated: true,  // #330: explicit opt-in only
    detectInstalled: () => thClawsAvailable(),  // By binary presence, not config dir
  },

  // ... additional agents (amp, kilo, roo, goose, continue, zed, aider, etc.)
};

/**
 * Default agents to install to (unless --agent overrides).
 * Federated agents (thclaws, opencode, copilot, openclaw) are NOT included.
 * They require explicit opt-in via `-a <name>` or `--with-thclaws`.
 */
export const defaultAgentNames = ['claude-code', 'codex'];

export function detectInstalledAgents(): string[] {
  return Object.entries(agents)
    .filter(([_, config]) => config.detectInstalled())
    .map(([name]) => name);
}

export function getDefaultAgents(): string[] {
  const installed = detectInstalledAgents();
  // Exclude federated agents — they're opt-in only
  const nonFederated = installed.filter(
    (a) => !agents[a as AgentType]?.federated
  );
  const defaults = defaultAgentNames.filter((a) => nonFederated.includes(a));
  return defaults.length > 0 ? defaults : nonFederated;
}
```

### Agent Type Definition

**File**: `/src/cli/types.ts`

```typescript
export interface AgentConfig {
  name: string;
  displayName: string;                    // Human-readable name
  skillsDir: string;                      // Relative path for local installs
  globalSkillsDir: string;                // Absolute path for global installs
  commandsDir?: string;                   // Separate commands directory (optional)
  globalCommandsDir?: string;
  useFlatFiles?: boolean;                 // Use skillname.md instead of skillname/SKILL.md
  commandsOptIn?: boolean;                // Only install commands with --commands flag
  commandFormat?: 'md' | 'toml';          // Command stub format (Gemini uses TOML)
  federated?: boolean;                    // Third-party agents (opt-in only)
  detectInstalled: () => boolean;         // Function to detect if agent is installed
}

export type AgentType =
  | 'opencode'
  | 'claude-code'
  | 'codex'
  | 'cursor'
  | 'amp'
  | 'kilo'
  | 'roo'
  | 'goose'
  | 'gemini'
  | 'antigravity'
  | 'copilot'
  | 'openclaw'
  | 'droid'
  | 'windsurf'
  | 'cline'
  | 'aider'
  | 'continue'
  | 'zed'
  | 'thclaws'
  | 'grok';
```

---

## 4. Skill Data Structure & Manifest Format

**File**: `/src/cli/types.ts`

### Skill Interface

```typescript
export interface Skill {
  name: string;              // e.g., "recap", "rrr", "awaken"
  description: string;       // One-line description
  path: string;              // File system path to the skill directory
  hidden?: boolean;          // If true, install SKILL.md but skip command stub
  secret?: boolean;          // Excluded from ALL profiles — install by name only (-s)
  zombie?: boolean;          // Excluded from ALL profiles — internal development candidates
}
```

### Skill Markdown Manifest Format

**File**: `/skills/about-oracle/SKILL.md` (example)

```yaml
---
name: about-oracle
description: What is Oracle — told by the AI itself. Origin story, stats, family count, ecosystem overview.
argument-hint: "--short | --stats | --family | --th | --en/th"
---

# /about-oracle

> This is not marketing copy. This is an AI writing about the system it lives inside.

## Step 0: System Check

First, run `arra-oracle-skills about` to check prerequisites...

## Usage Examples

/about-oracle            # Full story (English)
/about-oracle --th       # Full story (Thai)
/about-oracle --en/th    # Nat Weerawan's style (Thai + English tech terms)

---

## Implementation Steps

[Full instructions follow...]
```

**Key characteristics**:
- YAML frontmatter with metadata (name, description, argument hints)
- Markdown body contains the actual skill instructions
- Each skill is a **standalone markdown file** — no framework, no runtime
- Any AI can follow the instructions; the format is human-friendly, not machine-compiled

---

## 5. Installation Profiles & Registry

**File**: `/src/profiles.ts`

Profiles organize skills into curated tiers. They form a **tier invariant**: `minimal ⊆ standard ⊆ full ⊆ lab`.

```typescript
/**
 * minimal: newcomer essentials — 7 skills (lifecycle + trace + identity)
 * standard: daily driver — 20 skills (usage-driven, re-cut 2026-07-25)
 * full: all stable skills (excludes lab-only experiments)
 * lab: everything including experimental / bleeding edge
 *
 * TIER INVARIANT: **minimal ⊆ standard**. The tiers are a ladder —
 * upgrading a profile must never REMOVE a skill you already had.
 */

/** Minimal profile — essential lifecycle + trace + identity */
export const MINIMAL_SKILLS = [
  'about-oracle', 'forward', 'go', 'recap', 'rrr', 'trace', 'who-are-you',
] as const;

/** Standard profile — daily driver skills (always installed).
 *  MUST be a superset of MINIMAL_SKILLS. Re-cut 2026-07-25.
 *  Rule: if the census says people use it, it ships in standard. */
export const STANDARD_SKILLS = [
  'about-oracle', 'awaken', 'bampenpien', 'bud', 'create-shortcut', 'dig',
  'forward', 'go', 'incubate', 'learn', 'oracle-cheatsheet',
  'oracle-family-scan', 'oracle-prism', 'oracle-write-complete-book', 'recap',
  'resonance', 'rrr', 'trace', 'where-we-are', 'who-are-you',
] as const;

/** Lab-only skills — experimental, not in standard or full */
export const LAB_SKILLS = [
  'dream', 'fyi', 'watch',
] as const;

/** Zombie skills — internal development candidates.
 *  Excluded from ALL profiles. Install by name only: `arra install -s workon`
 *  Storage: each zombie lives under `src/skills/.archive/<name>/SKILL.md` */
export const ZOMBIE_SKILLS = [
  // Original 13 (from arra-symbiosis-skills)
  'alpha-feature', 'birth', 'deep-research', 'gemini', 'handover',
  // ... more zombies ...
  // 2026-07-06 zombie round 2 — usage census over 15,895 sessions
  'schedule', 'worktree', 'standup', 'xray', 'feel', 'hey', 'contacts', 'mailbox', 'inbox',
] as const;

/** Profile definitions */
export const profiles: Record<string, { include?: string[]; exclude?: string[] }> = {
  minimal: {
    include: [...MINIMAL_SKILLS],
  },
  standard: {
    include: [...STANDARD_SKILLS],
  },
  full: {
    exclude: [...labOnly, ...minimalOnly],  // all except lab-only + minimal-only variants
  },
  lab: {
    exclude: minimalOnly,                   // everything except minimal-only variants
  },
};

/**
 * Resolve a profile to a filtered list of skill names.
 * Returns null for profiles that mean "all skills" (lab).
 * Secret and zombie skills are excluded from ALL profiles.
 */
export function resolveProfile(
  profileName: string,
  allSkillNames: string[],
  secretSkillNames?: string[],
  zombieSkillNames?: string[]
): string[] | null {
  const excluded = new Set([...(secretSkillNames || []), ...(zombieSkillNames || [])]);
  const profile = profiles[profileName];
  if (!profile) return null;

  if (profile.include && profile.include.length > 0) {
    return profile.include.filter((s) => !excluded.has(s));
  }

  if (profile.exclude && profile.exclude.length > 0) {
    return allSkillNames.filter((s) => !profile.exclude!.includes(s) && !excluded.has(s));
  }

  // Empty = all skills (lab) — but still exclude secrets + zombies
  return excluded.size > 0
    ? allSkillNames.filter((s) => !excluded.has(s))
    : null;
}

/** Helper to resolve skill directory (handles zombie archive subdirectory) */
export function skillDirFor(name: string, skillsRoot: string): string {
  const isZombie = (ZOMBIE_SKILLS as readonly string[]).includes(name);
  const sep = skillsRoot.endsWith('/') ? '' : '/';
  return isZombie
    ? `${skillsRoot}${sep}.archive/${name}`
    : `${skillsRoot}${sep}${name}`;
}
```

---

## 6. Install Command & CLI Interface

**File**: `/src/cli/commands/install.ts`

The install command orchestrates skill discovery, profile resolution, user confirmation, and delegates to the installer module for the actual file operations.

```typescript
export function registerInstall(program: Command, version: string) {
  program
    .command('install', { isDefault: true })
    .description('Install Oracle skills to agents')
    .option('-g, --global', 'Install to user directory instead of project')
    .option('-l, --local', 'Install to project .claude/skills/ (explicit form of the default)')
    .option('-a, --agent <agents...>', 'Target specific agents (e.g., claude-code, opencode)')
    .option('-s, --skill <skills...>', 'Install specific skills by name')
    .option('-p, --profile <name>', 'Install a skill profile (minimal, standard, full, lab)', 'minimal')
    .option('--list', 'List available skills without installing')
    .option('-y, --yes', 'Skip confirmation prompts')
    .option('--with-commands', 'Also install command stubs to ~/.claude/commands/')
    .option('--force-global', 'Install global skills even if a same-named local skill exists (#230)')
    .option('--with-thclaws', 'Include thClaws if detected (federated agent — explicit opt-in)')
    .option('--thclaws-only', 'Install ONLY to thClaws paths (skips Claude Code, Codex, OpenCode, etc.)')
    .option('--all-detected', 'Install to ALL detected agents incl. federated (CI escape hatch)')
    .option('--shell', 'Force Bun.$ shell commands (use on Windows to test shell compatibility)')
    .option('--no-shell', 'Force Node.js fs operations (use on Unix if Bun.$ causes issues)')
    .action(async (options, cmd) => {
      p.intro(`🔮 Oracle Skills Installer v${version}`);

      try {
        // Detect whether --profile was explicitly passed on CLI or came from default value
        const profileSource = (cmd as any).getOptionValueSource?.('profile') ?? 'default';
        const profileExplicit = profileSource === 'cli';

        // Build target agents list (with explicit opt-in logic for federated agents)
        let targetAgents: string[] = options.agent || [];
        
        if (targetAgents.length === 0) {
          let detected: string[];
          if (options.allDetected) {
            // Escape hatch: ALL detected including federated
            detected = detectInstalledAgents();
          } else {
            detected = getDefaultAgents();
            // --with-thclaws: add thclaws to the auto set if binary is present
            if (options.withThclaws && thClawsAvailable() && !detected.includes('thclaws')) {
              detected = [...detected, 'thclaws'];
            }
          }

          if (detected.length > 0) {
            p.log.info(`Detected agents: ${detected.map((a) => agents[a as keyof typeof agents]?.displayName).join(', ')}`);
            targetAgents = detected;
          }
        }

        // Invoke the installer module
        await installSkills(targetAgents, {
          global: options.global,
          skills: options.skill,
          profile: options.profile,
          profileExplicit,
          yes: options.yes,
          commands: options.withCommands,
          forceGlobal: options.forceGlobal,
          shellMode,
        });

        p.outro('✨ Oracle skills installed!');
      } catch (error) {
        p.log.error(`Error: ${error instanceof Error ? error.message : 'Unknown error'}`);
        process.exit(1);
      }
    });
}
```

---

## 7. Core Installation Logic: Adapter Pattern in Action

**File**: `/src/cli/installer.ts`

This is the largest file and contains the adapter logic for installing skills to different agents. Each agent type has different requirements:

### 7.1 Claude Code & OpenCode Installation

Both agents store skills in their home directory under `.claude/skills/` or `.opencode/skills/`. The installer:
1. Creates the target skills directory
2. Copies the full skill folder (preserving structure)
3. Injects version metadata into the SKILL.md frontmatter
4. Optionally creates command stubs in `.claude/commands/` or `.opencode/commands/`

```typescript
// All agents: copy full skill directory to skills/
// OpenCode reads from .opencode/skills/ and creates slash commands automatically
const scope = options.global ? 'Global' : 'Local';

// Track skills with hooks for separate plugin installation
const skillsWithHooks: Skill[] = [];

for (const skill of agentSkillsToInstall) {
  // Check if skill has hooks - needs plugin installation
  if (await skillHasHooks(skill.name)) {
    skillsWithHooks.push(skill);
  }

  const destPath = join(targetDir, skill.name);

  // Remove existing if present
  if (existsSync(destPath)) {
    await rmrf(destPath, shellMode);
  }

  // Copy skill folder (VFS mode writes from memory, fs mode copies from disk)
  if (isCompiled()) {
    await writeSkillToDir(skill.name, destPath);
  } else {
    await cpr(skill.path, destPath, shellMode);
  }

  // Inject version into SKILL.md frontmatter and description
  const skillMdPath = join(destPath, 'SKILL.md');
  if (existsSync(skillMdPath)) {
    let content = await Bun.file(skillMdPath).text();
    if (content.startsWith('---')) {
      // Add installer field after opening ---
      content = content.replace(
        /^---\n/,
        `---\ninstaller: arra-oracle-skills-cli v${pkg.version}\norigin: Nat Weerawan's brain, digitized — how one human works with AI, captured as code — Soul Brews Studio\n`
      );
      // Prepend version AND scope to description
      const scopeChar = scope === 'Global' ? 'G' : 'L';
      const tierTag = (STANDARD_SKILLS as readonly string[]).includes(skill.name) ? '[standard]'
        : (LAB_SKILLS as readonly string[]).includes(skill.name) ? '[lab]'
        : '[core]';
      
      content = content.replace(
        /^(description:\s*)'?(.+?)'?(\n)/m,
        (_, p1, p2, p3) => {
          const desc = `${tierTag} v${pkg.version} ${scopeChar}-SKLL | ${p2}`;
          return `${p1}${yamlQuote(desc)}${p3}`;
        }
      );
      await Bun.write(skillMdPath, content);
    }
  }
}
```

### 7.2 Codex Plugin Marketplace Installation (Adapter for Version Branching)

Codex is unique because it has **two different formats** depending on the version:
- **0.128–0.129**: TOML format with `plugin.toml` + `prompt.md`
- **0.130+**: JSON format with `.codex-plugin/plugin.json` + `skills/<name>/SKILL.md`

This is a perfect example of the adapter pattern handling version-specific requirements.

```typescript
/**
 * Install skills for Codex 0.128.0+ plugin marketplace.
 *
 * Branches the layout based on the runtime Codex version.
 */
export async function installCodexPluginMarketplace(
  skills: Skill[],
  version: string,
  shellMode: ShellMode,
  opts?: {
    marketplaceDir?: string;
    configPath?: string;
    useJson?: boolean;
    pluginCacheDir?: string;
  }
): Promise<void> {
  const marketplaceDir = opts?.marketplaceDir ?? getCodexMarketplaceDir();
  const configPath = opts?.configPath ?? join(homedir(), '.codex', 'config.toml');
  const useJson = opts?.useJson ?? codexUsesJsonFormat();
  const pluginCacheDir = opts?.pluginCacheDir ?? getCodexPluginCacheDir();

  // ── 1. Create marketplace bundle directory ─────────────────────────────────
  await mkdirp(marketplaceDir, shellMode);

  // ── 2. Marketplace manifest ───────────────────────────────────────────────
  // 0.128 TOML: <marketplaceDir>/manifest.toml
  // 0.130 JSON: <marketplaceDir>/.agents/plugins/marketplace.json
  
  if (useJson) {
    // ── Codex 0.130+ JSON format ────────────────────────────────────────
    const agentsManifestDir = join(marketplaceDir, '.agents', 'plugins');
    await mkdirp(agentsManifestDir, shellMode);
    const marketplaceManifest = {
      name: 'arra-oracle-skills',
      interface: {
        displayName: 'Arra Oracle Skills',
      },
      plugins: skills
        .filter((s) => !s.hidden)
        .map((s) => ({
          name: s.name,
          source: {
            source: 'local',
            path: `./plugins/${s.name}`,
          },
          policy: {
            installation: 'AVAILABLE',
            authentication: 'ON_INSTALL',
          },
          category: 'Productivity',
        })),
    };
    await Bun.write(
      join(agentsManifestDir, 'marketplace.json'),
      `${JSON.stringify(marketplaceManifest, null, 2)}\n`
    );
  } else {
    // ── Codex 0.128 TOML format ────────────────────────────────────────
    const manifestContent = [
      `name = "arra-oracle-skills"`,
      `version = "${version}"`,
      `description = "Oracle skills for Codex by Soul Brews Studio"`,
      ``,
    ].join('\n');
    await Bun.write(manifestPath, manifestContent);
  }

  // ── 3. Per-skill plugin directories ───────────────────────────────────────
  const pluginsDir = join(marketplaceDir, 'plugins');
  await mkdirp(pluginsDir, shellMode);

  for (const skill of skills) {
    const skillPluginDir = join(pluginsDir, skill.name);
    await mkdirp(skillPluginDir, shellMode);

    if (useJson) {
      // ── Codex 0.130+ JSON format ────────────────────────────────────────
      const codexPluginDir = join(skillPluginDir, '.codex-plugin');
      await mkdirp(codexPluginDir, shellMode);
      const skillsSubDir = join(skillPluginDir, 'skills', skill.name);
      await mkdirp(skillsSubDir, shellMode);

      const pluginJson = {
        name: skill.name,
        version,
        description: skill.description,
        skills: './skills/',
        interface: {
          displayName: skill.name,
          shortDescription:
            skill.description.length > 100
              ? `${skill.description.slice(0, 97)}...`
              : skill.description,
        },
      };
      await Bun.write(
        join(codexPluginDir, 'plugin.json'),
        `${JSON.stringify(pluginJson, null, 2)}\n`
      );

      // Skill content → skills/<name>/SKILL.md
      if (isCompiled()) {
        await writeSkillToDir(skill.name, skillsSubDir);
      } else {
        const skillMdPath = join(skill.path, 'SKILL.md');
        if (existsSync(skillMdPath)) {
          const content = await Bun.file(skillMdPath).text();
          await Bun.write(join(skillsSubDir, 'SKILL.md'), content);
        }
      }

      // ── Populate Codex 0.130+ plugin cache ─────────────────────────────
      const cachePluginRoot = join(pluginCacheDir, skill.name);
      await mkdirp(cachePluginRoot, shellMode);
      const cacheDest = join(cachePluginRoot, version);
      if (existsSync(cacheDest)) await rmrf(cacheDest, shellMode);
      await cpr(skillPluginDir, cacheDest, shellMode);
    } else {
      // ── Codex 0.128 TOML format ─────────────────────────────────────────
      const escapedDesc = skill.description.replace(/\\/g, '\\\\').replace(/"/g, '\\"');
      const pluginToml = [
        `name = "${skill.name}"`,
        `description = "${escapedDesc}"`,
        `version = "${version}"`,
        `command = "/${skill.name}"`,
        ``,
      ].join('\n');
      await Bun.write(join(skillPluginDir, 'plugin.toml'), pluginToml);

      // prompt.md — skill body for Codex to execute
      if (isCompiled()) {
        await writeSkillToDir(skill.name, skillPluginDir);
      } else {
        const skillMdPath = join(skill.path, 'SKILL.md');
        if (existsSync(skillMdPath)) {
          const content = await Bun.file(skillMdPath).text();
          await Bun.write(join(skillPluginDir, 'prompt.md'), content);
        }
      }
    }
  }

  // ── 4. Update ~/.codex/config.toml ────────────────────────────────────────
  let configContent = existsSync(configPath) ? await Bun.file(configPath).text() : '';

  // Add marketplace registration if not already present
  const marketplaceSection = `[marketplaces.arra-oracle-skills]`;
  if (!configContent.includes(marketplaceSection)) {
    configContent += [
      ``,
      `${marketplaceSection}`,
      `source_type = "local"`,
      `source = "${marketplaceDir}"`,
      ``,
    ].join('\n');
  }

  // Enable each non-hidden skill as a plugin
  for (const skill of skills) {
    if (skill.hidden) continue;
    const pluginKey = `[plugins."${skill.name}@arra-oracle-skills"]`;
    if (!configContent.includes(pluginKey)) {
      configContent += [
        `${pluginKey}`,
        `enabled = true`,
        ``,
      ].join('\n');
    }
  }

  await Bun.write(configPath, configContent);
}
```

### 7.3 Gemini CLI Command Installation (TOML Adapter)

Gemini CLI uses TOML format for commands instead of Markdown:

```typescript
if (cmdFormat === 'toml') {
  // Gemini CLI: .toml slash commands
  const desc = skill.description.replace(/"/g, '\\"');
  const tomlContent = `description = "v${pkg.version} ${scopeChar}-CMD | ${desc}"
prompt = """
You are running the /${skill.name} skill.

Read the skill file at ${skillsPath}/${skill.name}/SKILL.md and follow ALL instructions in it.

Arguments: {{args}}

---
arra-oracle-skills-cli v${pkg.version}
"""
`;
  await Bun.write(join(commandsDir, `${skill.name}.toml`), tomlContent);
} else {
  // Claude Code, OpenCode, etc.: .md slash commands
  const stubContent = `---
description: ${yamlQuote(`v${pkg.version} ${scopeChar}-CMD | ${skill.description}`)}
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - Task
  - WebFetch
---

# /${skill.name}

Execute the \`${skill.name}\` skill with args: \`$ARGUMENTS\`

**If you have a Skill tool available**: Use it directly with \`skill: "${skill.name}"\` instead of reading the file manually.

**Otherwise**: Read the skill file at \`${skillsPath}/${skill.name}/SKILL.md\` and follow ALL instructions in it.

---
*arra-oracle-skills-cli v${pkg.version}*
`;
  await Bun.write(join(commandsDir, `${skill.name}.md`), stubContent);
}
```

### 7.4 Manifest Generation

After installation, the installer writes a manifest file that serves as a receipt and a registry of installed skills:

```typescript
// Write manifest with version info
const installedOnDisk = readdirSync(targetDir, { withFileTypes: true })
  .filter((d) => d.isDirectory() && !d.name.startsWith('.'))
  .map((d) => d.name)
  .filter((name) => existsSync(join(targetDir, name, 'SKILL.md')))
  .sort();

const manifest = {
  version: pkg.version,
  installedAt: new Date().toISOString(),
  skills: installedOnDisk,
  agent: agentName,
};
await Bun.write(join(targetDir, '.arra-oracle-skills.json'), JSON.stringify(manifest, null, 2));

// Write human-readable VERSION.md for agents to report
const versionMd = `# Oracle Skills

Installed by: **arra-oracle-skills-cli v${pkg.version}**
Installed at: ${new Date().toISOString()}
Agent: ${agent.displayName}
Skills: ${agentSkillsToInstall.length}

## Report This Version

When asked about skills version, report:
\`\`\`
arra-oracle-skills-cli v${pkg.version}
\`\`\`

## Installed Skills

${agentSkillsToInstall.map((s) => `- ${s.name}`).join('\n')}

## Update Skills

\`\`\`bash
bunx --bun arra-oracle-skills@github:Soul-Brews-Studio/arra-oracle-skills-cli#v${pkg.version} install -y -g
\`\`\`
`;
await Bun.write(join(targetDir, 'VERSION.md'), versionMd);
```

---

## 8. End-User Installation Flow

**File**: `/install.sh`

This bash script provides a unified installation experience for end users. It:
1. Detects the platform (darwin-x64, linux-arm64, etc.)
2. Attempts to download a pre-compiled native binary
3. Falls back to a bunx wrapper if binary is unavailable
4. Runs the initial install command

```bash
#!/bin/bash
# Oracle Skills Installer — downloads pre-built binary or falls back to bunx

set -e

echo "🔮 Oracle Skills Installer"

# ── Platform detection ──────────────────────────────────────
detect_platform() {
  local os arch
  os=$(uname -s | tr '[:upper:]' '[:lower:]')
  arch=$(uname -m)

  case "$arch" in
    x86_64|amd64) arch="x64" ;;
    aarch64|arm64) arch="arm64" ;;
    *) echo ""; return ;;
  esac

  case "$os" in
    darwin|linux) echo "${os}-${arch}" ;;
    *) echo "" ;;
  esac
}

PLATFORM=$(detect_platform)

# ── Version detection ───────────────────────────────────────
if [ -z "$ORACLE_SKILLS_VERSION" ]; then
  echo "🔍 Fetching latest version..."
  ORACLE_SKILLS_VERSION=$(curl -s https://api.github.com/repos/Soul-Brews-Studio/oracle-skills-cli/releases/latest 2>/dev/null | grep '"tag_name"' | cut -d'"' -f4)
fi

echo "📦 Version: $ORACLE_SKILLS_VERSION"

# ── Install method: binary or bunx ─────────────────────────
# Unified UX: `oracle-skills` always works (native binary or bunx wrapper)

INSTALL_DIR="$HOME/.oracle-skills/bin"
BINARY_NAME="oracle-skills-${PLATFORM}"
BINARY_URL="https://github.com/Soul-Brews-Studio/oracle-skills-cli/releases/download/${ORACLE_SKILLS_VERSION}/${BINARY_NAME}"
PKG_SPEC="oracle-skills@github:Soul-Brews-Studio/oracle-skills-cli#${ORACLE_SKILLS_VERSION}"

try_binary_install() {
  if [ -z "$PLATFORM" ]; then
    return 1
  fi

  echo "🔧 Downloading binary for ${PLATFORM}..."
  mkdir -p "$INSTALL_DIR"

  if curl -fsSL "$BINARY_URL" -o "$INSTALL_DIR/oracle-skills" 2>/dev/null; then
    chmod +x "$INSTALL_DIR/oracle-skills"
    echo "✓ Binary installed: $INSTALL_DIR/oracle-skills"
    ensure_path
    return 0
  else
    echo "⚠️  Binary not available for ${PLATFORM}, falling back to bunx"
    return 1
  fi
}

install_bunx_wrapper() {
  ensure_bun

  echo "📦 Installing bunx wrapper..."
  mkdir -p "$INSTALL_DIR"

  # Create a wrapper script that delegates to bunx
  cat > "$INSTALL_DIR/oracle-skills" << WRAPPER
#!/bin/bash
# Oracle Skills CLI — bunx wrapper (v${ORACLE_SKILLS_VERSION#v})
# Upgrade to native binary: curl -fsSL https://raw.githubusercontent.com/Soul-Brews-Studio/oracle-skills-cli/main/install.sh | bash
exec bunx --bun ${PKG_SPEC} "\$@"
WRAPPER
  chmod +x "$INSTALL_DIR/oracle-skills"
  echo "✓ Wrapper installed: $INSTALL_DIR/oracle-skills"
  ensure_path
}

# ── Install ─────────────────────────────────────────────────
INSTALL_MODE=""

if [ "$ORACLE_SKILLS_USE_BUNX" = "1" ]; then
  install_bunx_wrapper
  INSTALL_MODE="bunx"
elif try_binary_install; then
  INSTALL_MODE="binary"
else
  install_bunx_wrapper
  INSTALL_MODE="bunx"
fi

# Run skill installation with standard profile
"$INSTALL_DIR/oracle-skills" init -y

echo ""
if [ "$INSTALL_MODE" = "binary" ]; then
  echo "✨ Done! (native binary — fast mode)"
else
  echo "✨ Done! (bunx wrapper — re-run installer to upgrade to native binary)"
fi
```

---

## 9. Key Design Patterns

### 9.1 Adapter Pattern (Agent Configuration)

Each agent is configured with:
- Paths for local vs. global installation
- Command stub format (markdown vs. TOML)
- Installation detection logic
- Optional/mandatory command installation

This allows the core installer logic to remain agent-agnostic.

### 9.2 Profile Resolution Pattern

Profiles define **sets of skills** organized in tiers. Profiles are resolved via `resolveProfile()`, which:
1. Takes a profile name (minimal, standard, full, lab)
2. Applies include/exclude filters
3. Removes secrets and zombies
4. Returns the final skill list

### 9.3 Manifest Registry Pattern

Each installation produces:
- `.arra-oracle-skills.json` — machine-readable registry (version, install time, skill list)
- `VERSION.md` — human-readable receipt for the agent to report

### 9.4 Self-Target Guard

Prevents the installer from accidentally overwriting the skill *source* when run from the repo root. The check `isSelfTarget()` compares real paths to detect this scenario.

### 9.5 VFS (Virtual File System) Mode

When compiled into a binary, skills are embedded as a VFS. The installer detects this via `isCompiled()` and reads from memory instead of disk.

---

## 10. Multi-Agent Comparison: How Each Agent Differs

| Agent | Skills Path | Commands Path | Format | Notes |
|-------|------------|---|--------|-------|
| **Claude Code** | `.claude/skills/` | `.claude/commands/` | Markdown | Commands opt-in (--with-commands); plugins support |
| **OpenCode** | `.opencode/skills/` | `.opencode/commands/` | Markdown | Federated (explicit -a); auto-creates slash commands |
| **Codex** | `.codex/skills/` | `.codex/prompts/` | TOML (0.128–0.129) or JSON (0.130+) | Version-aware installer; plugin marketplace |
| **Gemini CLI** | `.gemini/skills/` | `.gemini/commands/` | TOML | Commands always installed (no opt-in) |
| **Cursor** | `.cursor/skills/` | — | — | No separate commands dir; skills only |
| **Cline** | `.cline/skills/` | — | — | Skills only |
| **Continue** | `.continue/skills/` | — | — | Skills only |

---

## 11. Example: Full Installation Flow

**User runs:**
```bash
curl -fsSL https://raw.githubusercontent.com/Soul-Brews-Studio/arra-oracle-skills-cli/main/install.sh | bash
```

**Happens:**
1. Script detects platform (darwin-arm64)
2. Fetches latest version tag from GitHub releases
3. Attempts to download `oracle-skills-darwin-arm64` binary
4. On success: installs to `~/.oracle-skills/bin/oracle-skills` (no Bun needed)
5. On failure: installs bunx wrapper to same path
6. Runs `oracle-skills init -y` to install standard profile (11 skills)
7. Skills installed to `~/.claude/skills/`, `~/.codex/skills/`, etc. (detected agents)

**Result:**
- Manifest written to each agent's skills dir
- Version info available via `oracle-skills list`
- User can restart agent and use `/recap`, `/rrr`, `/forward`, etc.

---

## Summary

**arra-oracle-skills-cli** is a multi-agent skill installer that:
1. **Abstracts agent differences** via `AgentConfig` (adapter pattern)
2. **Organizes skills** into versioned profiles (minimal, standard, full, lab)
3. **Handles version branching** (e.g., Codex 0.128 vs. 0.130)
4. **Provides both** binary and runtime (bunx) distribution
5. **Generates manifests** as receipts and registries
6. **Supports all major AI coding agents** (16+ agents)

The codebase is well-structured for extensibility: adding a new agent requires only a new `AgentConfig` entry in `agents.ts`.
