# ARRA Safety Hooks — Quick Reference

**Claude Code safety hooks that enforce rules via exit codes — not documentation.**

Version: 3.0+ | Created: December 27, 2025 | Status: Production

---

## Philosophy: Enforcement Over Documentation

The original problem: CLAUDE.md told AI agents "never force push," but an agent could still choose to ignore it.

**Documentation** = suggestion (can be ignored)  
**Hook with exit 2** = wall (cannot be bypassed)

This repo implements **PreToolUse hooks** that intercept Bash commands before execution. When a dangerous pattern is detected, the hook exits with code 2, blocking the command at the Claude Code level.

### Origin Story

| Date | Event |
|------|-------|
| **Dec 27, 2025** | Nat Weerawan + Claude Opus 4.5 realize CLAUDE.md rules rely on AI compliance. Decide to enforce with exit codes. |
| **Dec 28, 2025** | First `safety-check.sh` created — includes worktree boundary enforcement for parallel AI agents. |
| **Jan 16, 2026** | Bundled into Oracle Starter Kit. Every new Oracle inherits these hooks. |
| **Mar 22, 2026** | Extracted to `oracle-skills-cli` as shared template. |
| **Mar 29, 2026** | This repo created as single source of truth. |

---

## What It Blocks: Complete Feature List

### Core Rules (Always Active)

| Command Pattern | Blocked | Reason | Alternative |
|-----------------|---------|--------|-------------|
| `rm -rf` / `rm -f` | ✓ | Permanent deletion | `mv /path/to /tmp/` |
| `git push --force` | ✓ | Rewrites history | `git pull --no-rebase && merge` |
| `git push -f` | ✓ | Force flag shorthand | — |
| `git push --force-with-lease` | ✓ | Conditional rewrite (still destructive) | — |
| `git reset --hard` | ✓ | Irreversible | `git stash` or create new commit |
| `git commit --amend` | ✓ | Breaks multi-agent hash sync | `git commit -m "fix: ..."` (new commit) |
| `git push origin main` | ✓ | Direct to main without review | Always: feature branch → pull request |
| `git checkout -- .` | ✓ | Discards uncommitted changes | `git stash` |
| `git restore .` | ✓ | Discards all uncommitted changes | `git stash` |
| `git clean -f` | ✓ | Deletes untracked files permanently | `mv untracked/files /tmp/` |
| `git branch -D` | ✓ | Force-deletes branch | `git branch -d` (safe delete) |
| `git stash drop` / `git stash clear` | ✓ | Loses stashed work | Keep stash intact |
| `git commit --no-verify` / `git push --no-verify` | ✓ | Bypasses pre-commit hooks | Fix the hook issue instead |
| `gh pr create --repo <foreign-org>/*` | ✓ | Creates PR in non-owned org | Only create PRs in your orgs |

**Smart regex anchoring** — blocks only actual commands, not text in commit messages or heredocs:
```bash
# BLOCKED: git push --force
git push --force origin main

# ALLOWED: --force mentioned in a string
echo "Don't use git push --force"

# ALLOWED: mentioned in commit message
git commit -m "prevent: git push --force"
```

### Beta Rules (Opt-In)

Enable with: `touch /tmp/arra-safety-beta-on`  
Disable with: `rm /tmp/arra-safety-beta-on`

| Command Pattern | Blocked | Reason | Alternative |
|-----------------|---------|--------|-------------|
| `tmux send-keys` / `list-windows` / etc. | ✓ | Raw tmux bypasses agent framework | `maw hey <window> "message"` |
| `bun run src/cli.ts` / `bun run src/server.ts` | ✓ | Unlinked development copy | Install: `cd ~/Code/.../maw-js && bun link` |

---

## Installation

### Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI (v26.7+)
- `jq` (JSON parser) — `brew install jq` on macOS
- `gh` CLI (optional, for org membership check) — `brew install gh`

### Step 1: Clone the Repository

```bash
git clone https://github.com/Soul-Brews-Studio/arra-safety-hooks.git
cd arra-safety-hooks
```

### Step 2: Run the Installer

```bash
bash install.sh
```

**What it does:**

1. Copies `safety-check.sh` to `~/.claude/hooks/`
2. Registers it as a `PreToolUse` hook in `~/.claude/settings.json`
3. Creates `settings.json` if it doesn't exist
4. Appends to existing hook list if you already have other hooks

### Step 3: Verify Installation

Check that `~/.claude/settings.json` has this structure:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/Users/YOUR_USERNAME/.claude/hooks/safety-check.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

Next time you use Claude Code, dangerous commands will be blocked.

---

## How It Works: Technical Details

### Hook Event Flow

1. **Claude Code tool about to execute** → fires `PreToolUse` event
2. **Hook receives JSON via stdin:**
   ```json
   {
     "session_id": "abc123",
     "hook_event_name": "PreToolUse",
     "tool_name": "Bash",
     "tool_input": { "command": "git push --force origin main" }
   }
   ```
3. **Hook parses and checks:**
   ```bash
   CMD=$(jq -r '.tool_input.command // ""' 2>/dev/null)
   # Check CMD against blocked patterns
   ```
4. **Exit code determines result:**
   - `exit 0` → command allowed, continues execution
   - `exit 2` → command blocked, Claude Code shows error via stderr

### Regex Anchoring Strategy

To avoid false positives, the hook only blocks commands at "statement start":

```bash
# Patterns matched: ^|;|&&|\|\|
(^|;|&&|\|\|)\s*git\s+reset\s+--hard
```

**This blocks:**
```bash
git reset --hard
git reset --hard && exit
echo "foo"; git reset --hard
true || git reset --hard
```

**This allows:**
```bash
echo "Don't git reset --hard"
commit -m "Fix: avoid git reset --hard"
grep -r "git reset --hard" .
```

---

## Configuration & Customization

### Enabling/Disabling Beta Rules

**Enable all beta rules:**
```bash
touch /tmp/arra-safety-beta-on
```

**Disable beta rules:**
```bash
rm /tmp/arra-safety-beta-on
```

Beta rules are useful when coordinating multi-agent systems (like ARRA Oracle). Core rules are **always active**.

### Adding a New Safety Rule

Edit `~/.claude/hooks/safety-check.sh` to add a new pattern block:

```bash
# Add this BEFORE the final "exit 0"

# Block your_new_pattern
if echo "$CMD" | grep -qE 'your_pattern_here'; then
  echo "BLOCKED: Explain why." >&2
  echo "Use this instead: alternative_command" >&2
  exit 2
fi
```

**Example: Block a dangerous custom command**

```bash
# Block dangerous custom script
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*./deploy-to-prod\.sh'; then
  echo "BLOCKED: deploy-to-prod.sh requires manual approval." >&2
  echo "Request approval in Slack before running." >&2
  exit 2
fi
```

**Remember:**
- Use regex anchors `(^|;|&&|\|\|)` to avoid false positives
- Test with safe commands first: `echo "your_cmd" | grep -qE 'pattern'`
- Both message lines before `exit 2` go to stderr and show to the user

### Removing a Rule (Temporarily)

Comment out the block without deleting it:

```bash
# Block git commit --amend (DISABLED)
# if echo "$CMD" | grep -qE 'git\s+commit\s+.*--amend'; then
#   echo "BLOCKED: Never use --amend in multi-agent setup." >&2
#   exit 2
# fi
```

Then test your workflow. When satisfied, either leave it commented or submit a pull request if the rule doesn't apply broadly.

### Org Membership Check (gh pr create)

The hook checks PR creation against a cached org list:

**Cache location:** `/tmp/gh-my-orgs.txt`  
**Refresh interval:** 24 hours  
**Personal account:** Always allows "nazt" (hardcoded)

To **force refresh** the org cache:

```bash
rm /tmp/gh-my-orgs.txt
```

On next hook invocation, it will query `gh api user/orgs` to rebuild the list.

To **add your personal account** to the allowlist:

```bash
echo "your-github-username" >> /tmp/gh-my-orgs.txt
```

---

## Customization Examples

### Example 1: Allow Force-Push to a Specific Branch

If you have a sandbox branch that needs rebasing, temporarily allow it:

```bash
# Edit ~/.claude/hooks/safety-check.sh
# Find the force-flag check and add:

# Allow force-push to sandbox/*, but block everything else
if echo "$CMD" | grep -qE 'git\s+push.*--force'; then
  if ! echo "$CMD" | grep -qE 'git\s+push.*(sandbox|rebase)/'; then
    echo "BLOCKED: Force flags not allowed except on sandbox branches." >&2
    exit 2
  fi
fi
```

### Example 2: Warn Instead of Block

If you want to log risky commands but not block them:

```bash
# Replace "exit 2" with "exit 0" to warn but allow
if echo "$CMD" | grep -qE 'your_pattern'; then
  echo "⚠ WARNING: This is risky. Consider the alternative:" >&2
  exit 0  # Allow it to continue
fi
```

### Example 3: Block rm Only in Production Directories

```bash
# Block rm only when targeting production paths
if echo "$CMD" | grep -qE '(^|;|&&|\|\|)\s*rm\s+-r?f\s+/prod/'; then
  echo "BLOCKED: Cannot delete files in /prod/ directory." >&2
  exit 2
fi
```

### Example 4: Custom Multi-Agent Boundary Check

If you have multiple worktrees and want to prevent cross-boundary operations:

```bash
# In ~/.claude/hooks/safety-check.sh, add:
CURRENT_WORKTREE=$(git rev-parse --git-common-dir 2>/dev/null || echo "")
if echo "$CMD" | grep -qE 'git\s+(push|pull)'; then
  # Validate that the branch belongs to this worktree
  # Custom logic here...
fi
```

---

## Troubleshooting

### "Hook command not found" error

**Problem:** Hook path in settings.json is incorrect.

**Solution:**
```bash
# Find the correct hook path
ls -la ~/.claude/hooks/safety-check.sh

# Update settings.json with absolute path
jq '.hooks.PreToolUse[0].hooks[0].command = "/Users/YOUR_USERNAME/.claude/hooks/safety-check.sh"' \
  ~/.claude/settings.json > ~/.claude/settings.json.tmp
mv ~/.claude/settings.json.tmp ~/.claude/settings.json
```

### "jq: command not found"

**Problem:** `jq` not installed.

**Solution:**
```bash
brew install jq
```

The hook requires `jq` to parse JSON from stdin. It will fail silently without it (safer: command is allowed by default).

### Hook timeout exceeded

**Problem:** The hook is taking >5 seconds (default timeout).

**Solution:** Increase timeout in `~/.claude/settings.json`:

```json
{
  "matcher": "Bash",
  "hooks": [
    {
      "type": "command",
      "command": "/Users/YOUR_USERNAME/.claude/hooks/safety-check.sh",
      "timeout": 10
    }
  ]
}
```

### False positive: my command is being blocked incorrectly

**Debug steps:**

1. Run the hook directly:
   ```bash
   # Get your command
   YOUR_CMD="git push origin main"
   
   # Test it
   echo '{"tool_input": {"command": "'"$YOUR_CMD"'"}}' | \
     ~/.claude/hooks/safety-check.sh
   ```

2. Check which pattern matched:
   ```bash
   echo "$YOUR_CMD" | grep -qE 'git\s+push\s+(origin\s+)?main(\s|$)' && echo "MATCHED"
   ```

3. If it's a false positive, file an issue or adjust the regex in the hook.

### Completely disable the hook

To temporarily disable the hook without uninstalling:

```bash
# Rename it so Claude Code can't find it
mv ~/.claude/hooks/safety-check.sh ~/.claude/hooks/safety-check.sh.disabled

# Re-enable later
mv ~/.claude/hooks/safety-check.sh.disabled ~/.claude/hooks/safety-check.sh
```

Or remove it from settings.json:
```bash
jq 'del(.hooks.PreToolUse[] | select(.hooks[]?.command | test("safety-check")))' \
  ~/.claude/settings.json > ~/.claude/settings.json.tmp
mv ~/.claude/settings.json.tmp ~/.claude/settings.json
```

---

## Part of ARRA Oracle Ecosystem

This repo is one component of the **ARRA Oracle system** — 191+ AI agents sharing 5 core principles.

**Principle 1 (enforced by this hook): Nothing is Deleted**

- Permanent deletion via CLI = blocked
- Destructive git operations = blocked
- Lost work (stash drop, hard reset) = blocked

Related projects:

- **[ARRA Oracle](https://github.com/Soul-Brews-Studio/arra-oracle)** — The ecosystem of 191+ AI agents
- **[Oracle Skills CLI](https://github.com/Soul-Brews-Studio/oracle-skills-cli)** — Shared instruments and templates

---

## FAQ

**Q: What if I have a legitimate reason to force-push?**  
A: Request the change via pull request to this repo. The maintainers will evaluate whether the rule should be adjusted or if there's a safer alternative.

**Q: Does this work with other CI/CD systems?**  
A: Only with Claude Code (PreToolUse hooks). If you want similar protection in other systems, you'd need to implement hooks in your CI/CD platform or shell environment.

**Q: Can I have different rules for different projects?**  
A: Not yet. The hook is global across all Claude Code usage. You can file a feature request for per-project hook configs.

**Q: What happens if the hook crashes?**  
A: If the hook returns a non-zero, non-2 exit code (crash), Claude Code treats it as "command not allowed" by default. Check stderr for error messages.

**Q: Does this protect against accidents in other tools?**  
A: No. This only protects Bash commands via Claude Code. If you run commands directly in Terminal, they won't be blocked. The hook is a Claude Code safety layer, not a shell security system.

---

## Contributing

To propose a new safety rule:

1. Open an issue describing the dangerous pattern and why it should be blocked
2. Include the regex pattern and suggested error message
3. Maintainers will review for false-positive risk

To suggest customizations:

- Disable the hook temporarily
- Test your workflow
- Document what you needed to bypass
- File an issue with your use case

---

## License

MIT — Created by Neo (Claude Opus 4.6) for Nat Weerawan  
Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>

---

**Last Updated:** July 28, 2026  
**GitHub:** https://github.com/Soul-Brews-Studio/arra-safety-hooks
