# oracle-framework Learning Index

## Source
- **Origin**: ./origin/
- **GitHub**: https://github.com/Soul-Brews-Studio/oracle-framework

## Explorations

### 2026-07-28 0806 (default, 3 agents)
- [Architecture](2026-07-28/0806_ARCHITECTURE.md)
- [Code Snippets](2026-07-28/0806_CODE-SNIPPETS.md) — **corrected**, see note below
- [Quick Reference](2026-07-28/0806_QUICK-REFERENCE.md)

**Key insights**:
- This repo is **pure philosophy, not a reference implementation** — its entire content is a single 660-line `README.md`, nothing else. It defines the 3 core principles and the ψ/ architecture concept in the abstract; it doesn't ship any skills, agents, or `.claude/` config itself.
- The README does document how to *use* the philosophy via other repos — `laris-co/oracle-v2` for the MCP memory server (a different org/owner than Soul-Brews-Studio, worth noting), and a `trace-oracle` skill.
- **Data-quality note**: the first draft of the Code Snippets doc pulled real code from a *different* repo (`arra-oracle-skills-cli`) and mislabeled it as living in `oracle-framework` — caught by cross-checking against the Architecture agent's (correct) directory listing and fixed before commit. Kept as a reminder to verify agent output against ground truth rather than trust it blindly.
- Relationship to `opensource-nat-brain-oracle` (learned in an earlier session): that repo is the concrete reference implementation of these same principles — oracle-framework is upstream philosophy, opensource-nat-brain-oracle is downstream code.
