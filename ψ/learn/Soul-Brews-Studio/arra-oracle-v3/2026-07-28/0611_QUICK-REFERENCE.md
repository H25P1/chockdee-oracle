# Arra Oracle V3 Quick Reference

**Arra Oracle V3** is the Oracle family's memory and semantic search layer — a Docker-first MCP server that stores notes in SQLite, searches them with FTS (Full Text Search) and vector embeddings, and exposes memory through HTTP, MCP, CLI, plugins, and web UI.

**Current version:** v26.7.26-alpha.227 | **License:** BUSL-1.1 | **Runtime:** Bun 1.2+

---

## Table of Contents

1. [What This Project Does](#what-this-project-does)
2. [Quick Start: Docker Path (Recommended)](#quick-start-docker-path-recommended)
3. [Installation Methods](#installation-methods)
4. [Key Features & Surfaces](#key-features--surfaces)
5. [Configuration](#configuration)
6. [Development Setup](#development-setup)
7. [Project Structure](#project-structure)
8. [Contributing & Workflow](#contributing--workflow)
9. [Troubleshooting](#troubleshooting)
10. [Resources](#resources)

---

## What This Project Does

Arra Oracle V3 is a **memory and search layer** for the Oracle ecosystem:

- **Central Hub**: Registry and controller for 80+ Oracle AI agent instances across the family
- **Semantic Search**: Search notes via full-text search (FTS5) or vector embeddings (bge-m3, nomic, qwen3)
- **Multiple Interfaces**: Access via HTTP API, MCP (28 tools), CLI (`arra` command), or web UI
- **Local-First**: Runs locally with Docker, SQLite database, and optional cloud vector backends
- **Extensible**: Plugin system for custom CLI commands, MCP tools, API routes, and integrations
- **Federation-Ready**: Optional mesh networking to federate memory across multiple Oracle instances

### Not Included
- Doesn't edit menus/plugins/settings via API (read-only UI)
- Doesn't provide auth flows (bearer token support available)
- Doesn't include legacy web marketing site

---

## Quick Start: Docker Path (Recommended)

This is the primary path — **no local Bun install, API keys, or vector configuration required**.

### 1. Run the Server

```bash
# Set environment variables
export ARRA_PORT="${ARRA_PORT:-47778}"
export ARRA_URL="http://127.0.0.1:${ARRA_PORT}"
export ARRA_CONTAINER="${ARRA_CONTAINER:-arra-oracle}"
export ARRA_VOLUME="${ARRA_VOLUME:-arra-oracle-data}"
export ARRA_NOTES_DIR="${ARRA_NOTES_DIR:-$HOME/notes}"

# Create folders and Docker volume
mkdir -p "$ARRA_NOTES_DIR"
docker volume create "$ARRA_VOLUME"

# Run HTTP server
docker run --rm -d --name "$ARRA_CONTAINER" \
  -p "${ARRA_PORT}:47778" \
  -v "${ARRA_VOLUME}:/data" \
  -v "${ARRA_NOTES_DIR}:${ARRA_NOTES_DIR}:ro" \
  ghcr.io/soul-brews-studio/arra-oracle-v3:http

# Wait for health check
until curl -sf "${ARRA_URL}/api/health" >/dev/null; do sleep 1; done
echo "Arra Oracle is ready: ${ARRA_URL}"
```

**Ports:**
- Default: `47778` (change via `ARRA_PORT`)
- If busy, set `ARRA_PORT=47878` before `docker run`

**Volumes:**
- `${ARRA_VOLUME}` → `/data` (persists SQLite DB inside container)
- `${ARRA_NOTES_DIR}` → mounted read-only (your notes folder)

### 2. Mine Your Notes

```bash
# Create a shell function for CLI access (writes to the same Docker volume)
arra() {
  docker exec "$ARRA_CONTAINER" bun dist-cli/index.js "$@"
}

# Index your notes (Markdown, MDX, text files)
arra mine ~/notes

# Unchanged files are skipped (deterministic IDs)
# Re-run safely anytime
```

### 3. Open the UI

```bash
# Simple Mode (minimal entry point)
echo "${ARRA_URL}/simple"
# Open: http://127.0.0.1:47778/simple

# Full Studio (React dashboard)
# Available at same URL when frontend is enabled
```

### 4. Search Your Memory

**Via UI:**
- Open `/simple` in browser, use search box

**Via CLI:**
```bash
arra search "runbook" --limit 5
```

**Via HTTP API:**
```bash
# Full-text search
curl -sfS "${ARRA_URL}/api/v1/search?q=runbook&mode=fts&limit=5"

# Grounded ask (extractive, no LLM)
curl -sfS "${ARRA_URL}/api/v1/ask" \
  -H 'content-type: application/json' \
  -d '{"q":"What did I write about runbooks?","limit":5,"llm":false}'
```

### 5. Stop & Restart

```bash
# View logs
docker logs "$ARRA_CONTAINER"

# Stop (data persists in Docker volume)
docker stop "$ARRA_CONTAINER"

# Restart (re-run the docker run command)
docker run --rm -d --name "$ARRA_CONTAINER" \
  -p "${ARRA_PORT}:47778" \
  -v "${ARRA_VOLUME}:/data" \
  -v "${ARRA_NOTES_DIR}:${ARRA_NOTES_DIR}:ro" \
  ghcr.io/soul-brews-studio/arra-oracle-v3:http
```

---

## Installation Methods

### Docker (Recommended for Users)

**Pros:** No local setup, no Bun install, single command
**Cons:** Requires Docker daemon

```bash
docker run --rm -d \
  -p 47778:47778 \
  -v arra-oracle-data:/data \
  ghcr.io/soul-brews-studio/arra-oracle-v3:http
```

Images available:
- `:http` — Long-running HTTP server (port 47778)
- `:stdio` — Stdio MCP server for Claude/agents
- `:cli` — CLI-only image

### Global Bun Package (For Users & Developers)

```bash
# Install a specific release
bun add -g github:Soul-Brews-Studio/arra-oracle-v3#vX.Y.Z-alpha.N

# Or install latest alpha (working trunk)
bun add -g github:Soul-Brews-Studio/arra-oracle-v3#alpha

# Commands available after install
arra-oracle-v3 --help
arra --version
```

Commands:
- `arra-oracle-v3 serve --port 47778` — Start HTTP server
- `arra mine ~/notes` — Index note folder
- `arra search <query>` — Search memory
- `arra-oracle-v3 mcp` — Start MCP server (for Claude)

### Local Source Checkout (For Development)

```bash
git clone https://github.com/Soul-Brews-Studio/arra-oracle-v3.git
cd arra-oracle-v3
bun install
bunx tsc --noEmit  # Type check

# Run dev server
bun run server  # HTTP at http://localhost:47778

# Or run MCP
bun bin/mcp.ts [--read-only]

# Or run CLI
arra mine ~/notes
```

### Cloudflare Worker Deploy

One-click deploy or manual setup:

```bash
# Deploy MCP to Cloudflare Workers
npm run cloudflare:mcp:deploy

# Deploy Studio frontend proxy
npm run cloudflare:studio:deploy

# Deploy federation service
npm run cloudflare:federation:deploy
```

Requires: `wrangler` CLI and Cloudflare account

---

## Key Features & Surfaces

### 1. Web UIs

| Surface | URL | Use Case |
|---------|-----|----------|
| **Simple Mode** | `/simple` | Health check, basic search (minimal page) |
| **Studio** | `/` | Full React dashboard (menu, plugins, MCP tools, vector search, settings) |
| **Swagger API Docs** | `/api/v1/docs` | API endpoint reference |

### 2. HTTP API

**Base URL:** `http://localhost:47778/api/v1/`

Common endpoints:

```bash
# Health
curl ${ARRA_URL}/api/health

# Search (FTS or vector)
curl "${ARRA_URL}/api/v1/search?q=query&mode=fts&limit=10"

# Ask (grounded retrieval + optional LLM)
curl -X POST "${ARRA_URL}/api/v1/ask" \
  -H 'content-type: application/json' \
  -d '{"q":"Question?","limit":5,"llm":false}'

# Learn (add a memory)
curl -X POST "${ARRA_URL}/api/v1/learn" \
  -H 'content-type: application/json' \
  -d '{"text":"Memory text","source":"cli","tags":["tag1"]}'

# List documents
curl "${ARRA_URL}/api/v1/list"

# Vector search status
curl "${ARRA_URL}/api/v1/vector/status"

# Plugin list
curl "${ARRA_URL}/api/v1/plugins"

# MCP tools
curl "${ARRA_URL}/api/v1/mcp/tools"
```

Full reference: See `docs/API-REFERENCE-INDEX.md` and `/api/v1/docs`

### 3. MCP (Model Context Protocol)

**28 Core Tools** advertised to Claude, agents, and other MCP clients:

**Read tools:**
- `oracle_search` — Hybrid FTS/vector search
- `oracle_read` — Read one document by ID
- `oracle_list` — Browse documents
- `oracle_concepts` — List concept tags
- `oracle_recap` — Identity + top memory recap
- `oracle_stats` — Knowledge base health
- `oracle_profile` — Read Oracle profiles
- `oracle_threads` — List discussion threads
- `oracle_trace_list` — List research traces

**Write tools:**
- `oracle_learn` — Add/index a learning
- `oracle_supersede` — Mark doc superseded (not deleted)
- `oracle_research_note` — Store research note
- `oracle_handoff` — Write session handoff
- `oracle_thread` — Create/continue discussion
- `oracle_trace` — Log trace session
- `oracle_verify` — Verify disk ↔ DB integrity

**Advanced:**
- `oracle_mcp_call` — Call external MCP tools
- `oracle_mcp_list_tools` — List tools from other servers

### 4. CLI (`arra` command)

```bash
# Configuration
arra config add local http://localhost:47778
arra config use local
arra config list

# Search
arra search "confidence ranking" --limit 5

# Learn (add memory)
arra learn "New fact about X" --source cli-guide --tags tag1,tag2

# Health check
arra health

# Vector config
arra vector-config list --json

# Export
arra export --collection oracle_documents --format markdown --output oracle.md

# Threads
arra threads list
arra thread read thread-id

# Traces
arra trace create "Session about X" --findings "Found: ..."
arra trace list
```

### 5. Plugins

**Unified plugin manifest** (`plugin.json`) can contribute:
- CLI commands
- MCP tools
- Menu items (UI)
- API routes
- Sidecars (background processes)
- Lifecycle hooks

Example plugin structure:
```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "type": "command",
  "cli": {"command": "my-cmd"},
  "mcp": {"tools": [{"name": "my_tool"}]},
  "menu": [{"label": "My Item", "href": "/api/my-plugin"}]
}
```

Install plugins:
```bash
arra plugin install my-plugin
arra plugin list
arra plugin uninstall my-plugin
```

---

## Configuration

### Environment Variables

**Core:**
- `ARRA_PORT` → HTTP server port (default: 47778)
- `ARRA_URL` → Oracle base URL for clients
- `ARRA_CONTAINER` → Docker container name (default: arra-oracle)
- `ARRA_VOLUME` → Docker volume name (default: arra-oracle-data)
- `ARRA_NOTES_DIR` → Notes folder to index (default: ~/notes)

**Vector & Storage:**
- `ORACLE_VECTOR_BACKEND` → `lancedb`, `qdrant`, `cloudflare-vectorize` (default: lancedb)
- `ORACLE_EMBEDDING_MODEL` → `bge-m3` (CF), `nomic`, `qwen3` (default: bge-m3)
- `ORACLE_STORAGE_BACKEND` → `sqlite`, `d1` (Cloudflare) (default: sqlite)
- `ORACLE_DATA_DIR` → SQLite database folder (default: ~/.oracle)
- `ORACLE_VECTOR_READONLY` → Set to read-only vector mode
- `ORACLE_VECTOR_DB` → For vector proxy (`lancedb`, etc.)

**Server & Auth:**
- `ARRA_API_TOKEN` → Bearer token for protected writes (random 64-char hex string)
- `ORACLE_HTTP_URL` → Proxy mode: URL of running HTTP backend
- `ORACLE_LOG_TARGET` → `stdout` or `stderr` (MCP logging)
- `ORACLE_MCP_PATH` → Remote MCP mount path (default: /mcp)

**Plugins & Features:**
- `ORACLE_MENU_DIR` → Auto-load menus from folder
- `ORACLE_PLUGINS_DIR` → Plugin directory
- `ORACLE_READ_ONLY` → Set `1` for read-only mode (hides write tools)
- `HUGINN_CAPTURE_ENABLED` → Enable auto-capture hook

**Cloudflare Workers:**
- `ORACLE_ORIGIN_URL` → Secret origin URL (Tunnel)
- `ORACLE_STORAGE_BACKEND=d1` → Use D1 database
- `ORACLE_VECTOR_BACKEND=cloudflare-vectorize` → Use Vectorize
- `ARRA_API_TOKEN` → Bearer token for auth

### Configuration Files

**CLAUDE.md** (project conventions)
- Versioning: Always `v{YY}.{M}.{D}-alpha.{HMM}`
- Branching: Push to `alpha` (working trunk), never directly to `main`
- File size: ≤250 lines per file (source, tests, docs)
- Runtime: Bun ≥1.2

**.claude/hooks/block-push-main.sh**
- Prevents accidental pushes to `main` (stable release only)

**bunfig.toml**
- Test roots and path ignore patterns (ignores `agents/` worktree copies)

---

## Development Setup

### Prerequisites

```bash
# Install Bun
curl -fsSL https://bun.sh/install | bash

# Verify
bun --version  # Should be 1.2.0+
```

### Local Development

```bash
# Clone repo
git clone https://github.com/Soul-Brews-Studio/arra-oracle-v3.git
cd arra-oracle-v3

# Install dependencies
bun install

# Type check (build gate)
bunx tsc --noEmit

# Run HTTP server
bun run server
# Opens http://localhost:47778/simple

# Run MCP server
bun bin/mcp.ts [--read-only]

# Run CLI
bun cli/src/cli.ts search "query"

# React frontend dev (Vite proxy to :47778)
cd frontend
bun install
bun run dev
# Opens http://localhost:3000
```

### Testing

```bash
# Unit tests
bun run test:unit

# Integration tests
bun run test:integration

# Full test suite
bun test

# Scoped test (avoid agent worktrees)
bun test --isolate tests/http/health/

# Coverage
bun test --coverage

# Specific test file
bun test tests/http/search/search.test.ts
```

**Important:** Always use `--isolate` when scoping tests (prevents mock leakage).

### Build & Merge Workflow

1. **Branch from `alpha`** (never `main`)
   ```bash
   git checkout alpha
   git pull origin alpha
   git checkout -b feat/my-feature
   ```

2. **Implement & commit**
   ```bash
   # Keep commits clean
   git add src/my-file.ts
   git commit -m "feat: add new search mode"
   ```

3. **Pass build gate**
   ```bash
   bunx tsc --noEmit  # Type check
   bun test --isolate tests/http/my-cluster/
   ```

4. **Push & create PR**
   ```bash
   git push -u origin feat/my-feature
   gh pr create --title "Add new search mode" --body "..."
   ```

   **Always target `alpha`**, never `main`

5. **Report done (don't merge)**
   - Lead reviews and merges
   - Coders never self-merge

### Database Migrations

Use Drizzle ORM:

```bash
# Define schema in src/db/schema.ts

# Generate migration
bun run db:generate

# Apply migration
bun run db:push

# View in studio
bun run db:studio
```

**Never** use raw `CREATE TABLE` / `ALTER TABLE` / `CREATE INDEX` inline in code.

---

## Project Structure

```
arra-oracle-v3/
├── src/
│   ├── routes/          # 21 Elysia route clusters (HTTP endpoints)
│   │   ├── health/      # Reference module for new clusters
│   │   ├── search/
│   │   ├── vector/
│   │   ├── plugins/
│   │   ├── mcp/
│   │   └── ...
│   ├── tools/           # MCP tool definitions
│   ├── vector/          # Embedding & vector search (batch, multiple backends)
│   ├── indexer/         # Document ingestion pipeline
│   ├── db/              # Drizzle schema & migrations
│   ├── vault/           # Knowledge management
│   ├── learn/           # Memory consolidation
│   ├── federation/      # Mesh networking
│   ├── server.ts        # Elysia app composition
│   └── index.ts         # Entry point
├── cli/
│   └── src/cli.ts       # `arra` CLI commands
├── frontend/            # React + Vite Studio UI
│   ├── src/
│   │   ├── components/
│   │   ├── routes/
│   │   └── styles.css
│   └── index.html
├── workers/             # Cloudflare Workers
│   ├── mcp/            # MCP remote transport
│   ├── studio/         # Studio proxy
│   └── federation/     # Federation mesh
├── docs/               # Comprehensive guides
├── tests/
│   ├── http/           # HTTP contract tests (21 clusters)
│   ├── storage/        # Storage tests
│   └── integration/    # End-to-end tests
├── AGENTS.md           # Team model & workflow
├── CLAUDE.md           # Project conventions
├── package.json        # Bun workspace config
├── bunfig.toml         # Bun settings
├── tsconfig.json       # TypeScript config
└── Dockerfile          # Docker image

Key Sizes:
├── docs/               # 60+ markdown guides
├── src/routes/         # 21 clusters (health, search, plugins, etc.)
└── src/                # Total ~25k lines, modular by concern
```

### Route Clusters (21 total)

Each cluster is a self-contained Elysia sub-app under `src/routes/<cluster>/`:

- **health** — Server status, readiness
- **search** — FTS & vector search
- **vector** — Vector config, embeddings
- **plugins** — Plugin discovery
- **mcp** — MCP tool catalog
- **menu** — Navigation menu
- **auth** — Bearer token validation
- **dashboard** — UI config & stats
- **feed** — Activity feed
- **files** — Document management
- **forum** — Discussion threads
- **indexer** — Ingestion control
- **knowledge** — Knowledge base
- **oraclenet** — Family networking
- **peer** — Federation peering
- **schedule** — Job scheduling
- **sessions** — Session tracking
- **settings** — User preferences
- **supersede** — Version history
- **traces** — Research logs
- **vault** — Encrypted storage

Reference: See `src/routes/health/` for clean example.

---

## Contributing & Workflow

### Issue → PR Flow

1. **Read contracts first**
   - `AGENTS.md` — Team model, branch rules, reporting
   - `CLAUDE.md` — Conventions, file sizes, testing
   - `.claude/hooks/` — Git safety hooks

2. **Branch from `alpha`** (never `main`)
   ```bash
   git fetch origin
   git checkout origin/alpha -b my-feature
   ```

3. **Implement following patterns**
   - **File size:** ≤250 lines (source, tests, docs)
   - **Routes:** Use `src/routes/health/` as reference
   - **Tests:** Nested under `tests/http/<cluster>/`
   - **Type check:** `tsc --noEmit` must pass
   - **No destructive ops:** No `git push -f`, `rm -rf`, etc.

4. **Self-check before push**
   ```bash
   # Type check
   bunx tsc --noEmit
   
   # Scoped tests
   bun test --isolate tests/http/my-cluster/
   
   # File sizes
   wc -l src/my-file.ts  # Must be ≤250
   
   # Self review
   git diff  # No console.log, dead code, or exports removed
   ```

5. **Push & create PR**
   ```bash
   git push -u origin my-feature
   gh pr create --base alpha  # Always target alpha
   ```

6. **Report (three reports, no intermediate noise)**
   - 🟢 `starting <task> — plan: ...` (on receipt)
   - ❌ `blocked: <reason>` (if stuck, try alternative)
   - ✅ `done <task> — commit <sha>, build pass, PR <url>`

### Team Model

- **Lead** (`claude`): Architecture, reference modules, PR review
- **Coders** (`codex-N`, engine `omx`): Mechanical fan-out from reference modules
- **Worktrees:** Each coder gets `agents/1-codex-N/` (gitignored)

Pattern: Lead ships ONE clean reference module → coders copy that shape → consistent work.

---

## Troubleshooting

### Docker Issues

**Container won't start:**
```bash
# Check image exists
docker images | grep arra-oracle

# Pull latest
docker pull ghcr.io/soul-brews-studio/arra-oracle-v3:http

# See logs
docker logs $ARRA_CONTAINER

# Start with verbose output
docker run -it ghcr.io/soul-brews-studio/arra-oracle-v3:http
```

**Port already in use:**
```bash
export ARRA_PORT=47879
# Then run docker run with updated port mapping
```

**Volume issues:**
```bash
# List volumes
docker volume ls

# Inspect volume
docker volume inspect arra-oracle-data

# Backup before deletion
docker run --rm -v arra-oracle-data:/data -v $(pwd):/backup alpine tar czf /backup/data.tar.gz -C /data .

# Delete
docker volume rm arra-oracle-data
```

### MCP Connection Issues

**Claude can't connect:**
```bash
# Test stdio MCP directly
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{}}' | \
  docker exec -i $ARRA_CONTAINER bun dist-cli/index.js mcp

# Check token format
echo $ARRA_API_TOKEN | wc -c  # Should be 64+ chars
```

**Read-only mode:**
```bash
# Start in read-only (write tools hidden)
docker run ... \
  -e ORACLE_READ_ONLY=1 \
  ghcr.io/soul-brews-studio/arra-oracle-v3:stdio
```

### Search Issues

**No results after `arra mine`:**
```bash
# Check notes were indexed
arra list

# Verify vector status
arra vector-config list

# Re-index a folder
arra mine ~/notes --force

# Check logs
docker logs $ARRA_CONTAINER | grep -i "index"
```

**Vector backend not available:**
```bash
# Check backend
curl ${ARRA_URL}/api/v1/vector/status

# If lancedb: ensure write permissions to /data
docker exec $ARRA_CONTAINER ls -la /data

# Switch backend
export ORACLE_VECTOR_BACKEND=qdrant
# Or: cloudflare-vectorize (for Workers)
```

### Performance

**Slow search:**
- Try FTS first: `curl ".../api/v1/search?q=term&mode=fts"`
- Vector search is async; check indexing status

**Memory usage:**
- Check logs for large embeddings batches
- Reduce `ORACLE_BATCH_SIZE` if needed

### Type Errors in Development

**`tsc --noEmit` fails:**
```bash
# Clear build cache
rm -rf dist/

# Reinstall
bun install

# Type check
bunx tsc --noEmit
```

**MCP tool type errors:**
- Regenerate from OpenAPI: `bun run openapi:export`
- Check `src/tools/mcp-manifest.ts`

---

## Resources

### Official Docs

- **[docs/INSTALL.md](../docs/INSTALL.md)** — Install methods (Bun, Docker, plugins)
- **[docs/CLI-GUIDE.md](../docs/CLI-GUIDE.md)** — Full CLI, MCP, HTTP reference
- **[docs/QUICKSTART-10MIN.md](../docs/QUICKSTART-10MIN.md)** — Docker 10-min walk
- **[docs/API-REFERENCE-INDEX.md](../docs/API-REFERENCE-INDEX.md)** — API endpoint map
- **[docs/architecture.md](../docs/architecture.md)** — System architecture
- **[docs/PLUGIN-GUIDE.md](../docs/PLUGIN-GUIDE.md)** — Write plugins
- **[docs/README.md](../docs/README.md)** — Docs index (60+ guides)

### Deployment Guides

- **[docs/deploy-cloudflare.md](../docs/deploy-cloudflare.md)** — Cloudflare Workers
- **[docs/deploy-vercel.md](../docs/deploy-vercel.md)** — Vercel Studio proxy
- **[docs/DEPLOY-DIGITALOCEAN.md](../docs/DEPLOY-DIGITALOCEAN.md)** — DigitalOcean VPS
- **[docs/deploy-production.md](../docs/deploy-production.md)** — Production checklist

### Key Documentation

- **[docs/mcp-tools.md](../docs/mcp-tools.md)** — MCP tool contracts (28 tools)
- **[docs/UNIFIED-PLUGIN.md](../docs/UNIFIED-PLUGIN.md)** — Plugin manifest spec
- **[docs/LOCAL-DEV.md](../docs/LOCAL-DEV.md)** — Development workflow
- **[docs/TROUBLESHOOTING.md](../docs/TROUBLESHOOTING.md)** — Common issues

### GitHub

- **Repository:** https://github.com/Soul-Brews-Studio/arra-oracle-v3
- **Issues:** https://github.com/Soul-Brews-Studio/arra-oracle-v3/issues
- **Releases:** https://github.com/Soul-Brews-Studio/arra-oracle-v3/releases
- **Actions:** CI/CD pipeline, calver release automation

### Community & Oracle Family

- **Family registry:** 80+ Oracle instances indexed in this repo
- **Issues used for:** Family introductions, registry updates
- **Federation:** Optional mesh networking for multi-Oracle setups

---

## Quick Command Reference

```bash
# Docker
docker run -d -p 47778:47778 -v arra-oracle-data:/data \
  ghcr.io/soul-brews-studio/arra-oracle-v3:http
docker exec arra-oracle bun dist-cli/index.js mine ~/notes

# Bun package
bun add -g github:Soul-Brews-Studio/arra-oracle-v3#alpha
arra mine ~/notes
arra search "query" --limit 10

# Development
bun install && bunx tsc --noEmit && bun run server
bun test --isolate tests/http/
gh pr create --base alpha

# Vector & search
curl "http://localhost:47778/api/v1/search?q=term&mode=fts"
curl -X POST http://localhost:47778/api/v1/ask \
  -H 'content-type: application/json' \
  -d '{"q":"Question?","llm":false}'

# MCP (Claude)
claude mcp add arra-oracle -- arra-oracle-v3 mcp
claude mcp list

# Config
arra config add local http://localhost:47778
arra config use local
arra health
arra vector-config list
```

---

## Version & License

- **Version:** 26.7.26-alpha.227 (CalVer with wall-clock HMM suffix)
- **Always alpha** by default (stable releases rare and intentional)
- **License:** BUSL-1.1 (Business Source Use License)
- **Repository:** `github:Soul-Brews-Studio/arra-oracle-v3`

---

**Last Updated:** 2026-07-28 | **Source:** arra-oracle-v3 origin repository
