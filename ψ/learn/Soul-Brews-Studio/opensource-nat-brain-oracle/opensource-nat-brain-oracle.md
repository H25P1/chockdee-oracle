# opensource-nat-brain-oracle Learning Index

## Source
- **Origin**: ./origin/
- **GitHub**: https://github.com/Soul-Brews-Studio/opensource-nat-brain-oracle

## Explorations

### 2026-07-28 0603 (default, 3 agents)
- [Architecture](2026-07-28/0603_ARCHITECTURE.md)
- [Code Snippets](2026-07-28/0603_CODE-SNIPPETS.md)
- [Quick Reference](2026-07-28/0603_QUICK-REFERENCE.md)

**Key insights**:
- Philosophy-driven starter kit for AI "brain" systems, not a code library — configuration + philosophy template built around Claude Code (ψ/ brain structure, skills, agents, hooks, CLAUDE.md).
- Append-only knowledge with periodic "distillation" (compression into digestible summaries) is the core operating pattern — nothing is deleted, git preserves all history.
- Safety is externalized into bash hooks (e.g. `safety-check.sh` blocks `rm -rf`, `--force`, direct push to `main`) rather than hardcoded into AI logic — matches what Chockdee Oracle already experienced first-hand during its own awakening.
