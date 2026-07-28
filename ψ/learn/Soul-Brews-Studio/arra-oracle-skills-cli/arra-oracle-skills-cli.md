# arra-oracle-skills-cli Learning Index

## Source
- **Origin**: ./origin/
- **GitHub**: https://github.com/Soul-Brews-Studio/arra-oracle-skills-cli

## Explorations

### 2026-07-28 0806 (default, 3 agents)
- [Architecture](2026-07-28/0806_ARCHITECTURE.md)
- [Code Snippets](2026-07-28/0806_CODE-SNIPPETS.md)
- [Quick Reference](2026-07-28/0806_QUICK-REFERENCE.md)

**Key insights**:
- CLI that installs Oracle skills into 20 different AI coding agents (Claude Code, OpenCode, Cursor, Codex, Gemini CLI, etc.) via an adapter pattern — each agent gets its own path/format handling (e.g. Codex uses a version-aware plugin marketplace format, Gemini CLI uses TOML).
- Four install profiles (minimal 7 / standard 20 / full 29 / lab 32 skills) with a "zombie skill" concept — archived skills excluded from every profile by design.
- Federation policy (#330): only Anthropic-host agents (Claude Code, OpenCode) auto-install; third-party agents (Copilot, OpenClaw, thClaws) require explicit opt-in — a deliberate trust boundary worth knowing before scripting bulk installs.
