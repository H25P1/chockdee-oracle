# maw-js Learning Index

## Source
- **Origin**: ./origin/
- **GitHub**: https://github.com/Soul-Brews-Studio/maw-js

## Explorations

### 2026-07-28 0749 (default, 3 agents)
- [Architecture](2026-07-28/0749_ARCHITECTURE.md)
- [Code Snippets](2026-07-28/0749_CODE-SNIPPETS.md)
- [Quick Reference](2026-07-28/0749_QUICK-REFERENCE.md)

**Key insights**:
- The Bun/TypeScript original that maw-rs is porting — CLI for orchestrating multiple AI agents (Claude Code, Codex, Aider, OpenCode) across machines via tmux. This is the tool H uses day-to-day.
- Command resolution is a 7-step dispatch ladder (routeComm → routeTools → aliases → plugin registry → agent shorthand, with prefix auto-complete guarded by word boundaries) — over 100 vendor plugins live in `src/vendor/mpr-plugins/` using the same manifest system as user plugins.
- Config/paths follow the XDG Base Directory spec with legacy-path migration; federation (cross-node communication) uses named peers + HMAC tokens. Directly answers the earlier parity-matrix question: this is the live behavior maw-rs's `native ✅` / `WASM ✅` / `stub ⚠️` rows are being measured against.
