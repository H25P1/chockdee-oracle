# maw-core-plugins Learning Index

## Source
- **Origin**: ./origin/
- **GitHub**: https://github.com/Soul-Brews-Studio/maw-core-plugins

## Explorations

### 2026-07-28 0806 (default, 3 agents)
- [Architecture](2026-07-28/0806_ARCHITECTURE.md)
- [Code Snippets](2026-07-28/0806_CODE-SNIPPETS.md)
- [Quick Reference](2026-07-28/0806_QUICK-REFERENCE.md)

**Key insights**:
- **Correction vs. the repo's own description**: the description says "wake, sleep, stop, done, hey, ls, serve" (7 plugins) but the repo actually only contains 4: `wake`, `sleep`, `stop`, `done`. `hey`, `ls`, and `serve` live elsewhere (native in maw-js/maw-rs, or in the separate maw-plugins monorepo) — the description is stale.
- These 4 are weight-`00` (load first) and ship automatically with maw-js itself — unlike the optional maw-plugins monorepo, these can't be removed and have no per-package package.json.
- Consistent handler pattern across all 4: dual-source dispatch (CLI string[] vs API object), console-output capture, structured `InvokeResult` return — same contract documented in the maw-plugins learning session.
