# multi-agent-orchestration-book Learning Index

## Source
- **Origin**: ./origin/
- **GitHub**: https://github.com/Soul-Brews-Studio/multi-agent-orchestration-book
- **Published**: https://soul-brews-studio.github.io/multi-agent-orchestration-book/ (Docusaurus)

## Explorations

### 2026-07-28 0806 (default, 3 agents — roles adapted for a book, not code)
- [Architecture](2026-07-28/0806_ARCHITECTURE.md) — structure, full table of contents (15 chapters + 4 appendices)
- [Code Snippets](2026-07-28/0806_CODE-SNIPPETS.md) — the 6 most actionable patterns, with sources
- [Quick Reference](2026-07-28/0806_QUICK-REFERENCE.md) — how to read it, thesis, audience

**Key insights**:
- Central thesis: "Convenience is for the AI. Visibility is for the human." — a field guide from 100+ hours of building maw-js, documenting three production-tested orchestration tiers (in-process subagents / coordinated teams / independent tmux processes).
- **The Reporting Contract** pattern (Ch 3, 8): "let me know when done" is not a contract, it's a hope — agents must commit to an exact channel/recipient/timing, backed by a war story of four failed attempts. Directly relevant to how this session's own background-agent notifications are structured.
- **The Lead-Compiles Pattern** (Ch 4): only the lead writes to main; subordinates work their own branches — matches the branch+PR discipline this chockdee-oracle repo has been following all session (via arra-safety-hooks' enforcement, learned in the same batch).
