# maw-project Learning Index

## Source
- **Origin**: ./origin/
- **GitHub**: https://github.com/Soul-Brews-Studio/maw-project

## Explorations

### 2026-07-28 0806 (fast, 1 agent)
- [Overview](2026-07-28/0806_OVERVIEW.md)

**Key insights**: A thin dispatcher plugin that routes `maw project learn ...` / `maw project incubate ...` to the sibling `maw-learn` / `maw-incubate` plugins (both optional peerDependencies). Like maw-learn, its subcommands are currently **v0.1.0 stubs** returning "not yet implemented" pending v2 delegation — `find` and `list` are the only cross-cutting utilities that stay in this plugin rather than delegating.
