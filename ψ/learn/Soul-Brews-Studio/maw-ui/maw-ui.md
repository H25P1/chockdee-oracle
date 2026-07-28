# maw-ui Learning Index

## Source
- **Origin**: ./origin/
- **GitHub**: https://github.com/Soul-Brews-Studio/maw-ui

## Explorations

### 2026-07-28 0841 (default, 3 agents)
- [Architecture](2026-07-28/0841_ARCHITECTURE.md)
- [Code Snippets](2026-07-28/0841_CODE-SNIPPETS.md)
- [Quick Reference](2026-07-28/0841_QUICK-REFERENCE.md)

**Key insights**:
- Bigger than expected for a "dashboard" — React 19 + TypeScript + Tailwind + Three.js multi-page SPA with **17 standalone views** (Office grid, Fleet, Dashboard, Terminal, Mission Control, Federation 2D/3D, Chat, Config, Inbox…) and 50+ components, built with Vite as 17 separate HTML bundles.
- Talks to maw-js over WebSocket for live session/agent state (no polling) — status decays busy → ready → idle client-side between feed events; a circuit breaker (`apiFetch`) trips after 5 failed calls and stays open 30s to avoid hammering a down server.
- Connects to any maw-js instance at runtime via a `?host=` query param (persisted to localStorage) rather than a build-time config — same dashboard binary can point at different fleets/nodes without rebuilding. Dev server on :5173 proxies to a local maw-js on :3456.
