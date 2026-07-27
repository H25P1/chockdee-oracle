# maw-rs Learning Index

## Source
- **Origin**: ./origin/
- **GitHub**: https://github.com/Soul-Brews-Studio/maw-rs

## Explorations

### 2026-07-28 0640 (default, 3 agents)
- [Architecture](2026-07-28/0640_ARCHITECTURE.md)
- [Code Snippets](2026-07-28/0640_CODE-SNIPPETS.md)
- [Quick Reference](2026-07-28/0640_QUICK-REFERENCE.md)

**Key insights**:
- Rust rewrite (Cargo workspace, 13-14 member crates) of "maw" — distributed terminal multiplexing (tmux-based) + fleet management for AI agent oracles, porting an earlier `maw-js` implementation tracked via a parity matrix (81 native ✅ / 29 WASM ✅ / 13 stub ⚠️ / 10 not-ported ❌).
- Clear pure/effectful split — fixture-driven crates (e.g. target matcher) are locked to `maw-js` JSON contracts so behavior stays identical across the JS→Rust port; `#![forbid(unsafe_code)]` at the workspace level.
- Covers session/fleet orchestration (spawn, attach, target resolution with fuzzy/ambiguous matching), peer federation with TOFU (trust-on-first-use) pubkey validation, WASM plugin support (Extism), and scheduling (including macOS launchd integration) — this is the tool used to run and coordinate fleets of Oracle-style agents, distinct from arra-oracle-v3 (memory/search) and opensource-nat-brain-oracle (philosophy template).
