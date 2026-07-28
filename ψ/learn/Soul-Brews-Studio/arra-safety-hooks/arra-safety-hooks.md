# arra-safety-hooks Learning Index

## Source
- **Origin**: ./origin/
- **GitHub**: https://github.com/Soul-Brews-Studio/arra-safety-hooks

## Explorations

### 2026-07-28 0806 (default, 3 agents)
- [Architecture](2026-07-28/0806_ARCHITECTURE.md)
- [Code Snippets](2026-07-28/0806_CODE-SNIPPETS.md)
- [Quick Reference](2026-07-28/0806_QUICK-REFERENCE.md)

**Key insights**:
- **This is the actual origin/reference implementation of the `~/.claude/hooks/safety-check.sh` PreToolUse:Bash hook that blocked us directly in this session** (twice: pushing to `main`, and a `git reset --hard`). Confirmed by matching rule list: rm -rf, git --force/-f, reset --hard, commit --amend, push origin main, checkout -- ./restore ., clean -f, branch -D, stash drop/clear, --no-verify.
- Philosophy: "documentation = suggestion, hooks with exit 2 = walls" — 12 core rules (always on) + 4 beta rules (opt-in via a `/tmp/arra-safety-beta-on` flag file), installed by patching `~/.claude/settings.json`'s `PreToolUse` array via `install.sh` + `jq`.
- Regex rules are anchored with delimiters (`^|;|&&|\|\|`) specifically to avoid false-positive blocks — worth referencing if we ever want to add our own custom rule to the same script.
