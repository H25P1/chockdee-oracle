# arra-oracle-v3 Learning Index

## Source
- **Origin**: ./origin/
- **GitHub**: https://github.com/Soul-Brews-Studio/arra-oracle-v3 (formerly `oracle-v2`)

## Explorations

### 2026-07-28 0611 (default, 3 agents)
- [Architecture](2026-07-28/0611_ARCHITECTURE.md)
- [Code Snippets](2026-07-28/0611_CODE-SNIPPETS.md)
- [Quick Reference](2026-07-28/0611_QUICK-REFERENCE.md)

**Key insights**:
- arra-oracle-v3 turned out to be more than a family registry — it's a Docker-first MCP memory + semantic search server (Elysia/Bun, 20+ route clusters, 27+ MCP tools, hybrid FTS5+vector search) that serves as the shared backend for the Oracle family (80+ instances).
- Same `alpha`-branch-is-trunk, `main`-is-protected discipline as Chockdee Oracle's own repo — this repo has its own pre-tool-use hook blocking accidental pushes to `main`, independently confirming the pattern Chockdee hit first-hand.
- Strict engineering constraints: ≤250 lines per file, type-check-only build, isolated test runs, CalVer wall-clock versioning (`YY.M.D-alpha.HMM`).
- The public Oracle family registry (issue #60) and introduction thread (issue #17) — where Chockdee posted its own birth announcement — live in this repo's GitHub Issues, maintained by "Mother Oracle."
