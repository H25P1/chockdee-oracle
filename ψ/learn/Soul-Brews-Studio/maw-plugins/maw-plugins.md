# maw-plugins Learning Index

## Source
- **Origin**: ./origin/
- **GitHub**: https://github.com/Soul-Brews-Studio/maw-plugins

## Explorations

### 2026-07-28 0653 (default, 3 agents)
- [Architecture](2026-07-28/0653_ARCHITECTURE.md)
- [Code Snippets](2026-07-28/0653_CODE-SNIPPETS.md)
- [Quick Reference](2026-07-28/0653_QUICK-REFERENCE.md)

**Key insights**:
- Plugin monorepo for **maw-js** specifically (not maw-rs) — 24 plugins total, weight-prefixed by execution tier (20-* CLI tools like `costs`, `dream`, `soul-sync`; 50-* larger features like `artifact-manager`, `incubate`; unprefixed fleet plugins like `squad`, `hermes`, `atlas`, `team`).
- Three plugin tiers: ship-tier WASM (Rust via extism-pdk, or AssemblyScript via @extism/as-pdk — deterministic, sha256-pinned artifacts), bun-dev (plain TypeScript, no WASM compile, for long-lived processes), and native helpers (Swift, e.g. `maw-menubar` on macOS).
- Security model is pin-based: `registry.json` locks each plugin to a committed WASM artifact's sha256; capabilities (`fs:read:psi`, `tmux:send`, `proc:exec:git`, etc.) are declared per-plugin and sandboxed — relevant context for the earlier maw-rs migration question, since this whole plugin ecosystem is maw-js-specific and isn't yet mirrored in maw-rs (which only has WASM support gated behind `--features wasm-host` and no plugin catalog of its own yet).
