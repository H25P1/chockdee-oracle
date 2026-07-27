# Arra Oracle V3 Architecture Documentation

**Repository:** `Soul-Brews-Studio/arra-oracle-v3`  
**Analysis Date:** 2026-07-28  
**Version:** 26.7.26-alpha.227  
**Status:** MCP Memory Layer with semantic search, philosophy, and knowledge management

---

## 1. Project Identity

**Arra Oracle V3** is the **MCP memory and search layer** for the Oracle family—a large, production-grade ecosystem of 80+ Oracle instances. It is the semantic search engine, knowledge registry, and federation hub for the family, featuring:

- Semantic search over 20,000+ documents (bge-m3, nomic, qwen3 embeddings + FTS5)
- Indexing pipeline with batch processing and deterministic ingest
- Federation plumbing for cross-Oracle queries
- MCP (Model Context Protocol) surface for Claude and agent integration
- Local SQLite data store with optional vector backends (Qdrant, LanceDB, Cloudflare Vectorize)
- Docker-first deployment: HTTP server, stdio MCP, CLI tools, Studio UI
- Unified plugin system with lifecycle hooks and runtime reload

---

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Entry Points                                │
├─────────────────────────────────────────────────────────────┤
│  CLI (arra mine/search/learn)  │  HTTP (:47778)  │  MCP     │
│  Docker + local Bun            │  Elysia server   │  stdio   │
│  Desktop + browser UI          │  Studio/Simple   │  agents  │
└─────────────────────────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
┌───────▼──────┐  ┌──────▼──────┐  ┌─────▼──────────┐
│  Plugin      │  │  MCP Routes │  │  HTTP Routes   │
│  System      │  │ (32 tools)  │  │ (32 clusters)  │
└───────┬──────┘  └──────┬──────┘  └─────┬──────────┘
        │                │               │
        └────────────────┼───────────────┘
                   ┌─────▼────────┐
                   │  Core Layer  │
                   ├──────────────┤
                   │ • Search     │
                   │ • Vector/FTS │
                   │ • Indexer    │
                   │ • Vault      │
                   │ • Federation │
                   │ • Knowledge  │
                   └─────┬────────┘
                         │
        ┌────────────────┼───────────────┐
        │                │               │
┌───────▼──────┐  ┌──────▼──────┐  ┌────▼───────────┐
│  SQLite DB   │  │ Vector DBs  │  │ Vault (local   │
│ (Drizzle)    │  │ (Qdrant,    │  │  file storage) │
│              │  │ LanceDB, CF) │  │                │
└──────────────┘  └─────────────┘  └────────────────┘
```

### One Memory Core, Thin Adapters Philosophy

Arra Oracle implements a **single memory core** with **thin surface adapters**:
- **CLI, HTTP, MCP, Plugins, Canvas, Web/Desktop UIs** all call the same business logic
- **No duplication:** shared contracts (response envelopes, error handling, auth)
- **Scalable:** adding a new surface means composing existing routes/tools, not rewriting logic

---

## 3. Directory Structure & Organization Philosophy

### Top-Level Layout

```
arra-oracle-v3/
├── .claude/                    # Claude Code project config
│   ├── settings.json           # Hooks (blocks push to main)
│   ├── agents/                 # Specialized agent personas
│   ├── commands/               # Slash commands (/rrr, /recap, etc.)
│   ├── knowledge/              # Philosophy, writing style
│   ├── hooks/                  # Git + lifecycle hooks
│   └── skills/                 # Reusable task patterns
│
├── .github/                    # GitHub workflows (CI/CD, calver-release)
├── .rtk/                       # RTK configuration (deprecated, legacy)
├── .mcp.json                   # Claude Code MCP server config
├── .env.example                # Environment variables for Docker/workers
│
├── bin/                        # Entry points
│   └── arra.ts                 # Main CLI binary
│
├── src/                        # Source code (58 directories)
│   ├── server.ts              # Elysia app + middleware composition
│   ├── index.ts               # Module export entry
│   ├── config.ts              # Runtime config parser
│   ├── const.ts               # Constants (MCP_SERVER_NAME, etc.)
│   │
│   ├── routes/                # 32+ HTTP route clusters (Elysia sub-apps)
│   │   ├── health/            # Reference module: /api/health
│   │   ├── search/            # FTS + vector search
│   │   ├── vector/            # Vector model discovery, config, search
│   │   ├── ask/               # LLM-grounded Q&A
│   │   ├── menu/              # Menu item registry + seeding
│   │   ├── plugins/           # Plugin registry, manifest, discovery
│   │   ├── mcp/               # MCP server + streamable routes
│   │   ├── auth/              # API token validation
│   │   ├── vault/             # Local file storage CLI
│   │   ├── memory/            # Memory contracts + provenance
│   │   ├── supersede/         # Reversible history & document updates
│   │   ├── indexer/           # Batch indexing + daemon lifecycle
│   │   ├── dashboard/         # Metrics, stats, health overview
│   │   ├── settings/          # User + tenant configuration
│   │   ├── files/             # File metadata + upload routes
│   │   ├── forum/             # Discussion threads (optional)
│   │   ├── traces/            # Call tracing + observability
│   │   ├── export/            # Document export (JSON, CSV, etc.)
│   │   ├── knowledge/         # Knowledge graph + concepts
│   │   ├── canvas/            # Canvas rendering + collaboration
│   │   ├── schedule/          # Job scheduling + cron
│   │   └── [20 more...]       # Other specialized endpoints
│   │
│   ├── tools/                 # MCP tools (27 exported functions)
│   │   ├── search.ts          # muninn_search main tool
│   │   ├── learn.ts           # Learning + documentation ingestion
│   │   ├── read.ts            # Read document with provenance
│   │   ├── ask.ts             # Ask questions
│   │   ├── recap.ts           # Session summaries
│   │   ├── trace.ts           # Call tracing
│   │   ├── forum.ts           # Discussion management
│   │   ├── schedule.ts        # Job scheduling
│   │   ├── supersede.ts       # Document updates
│   │   ├── verify.ts          # Citation verification
│   │   ├── concepts.ts        # Concept extraction
│   │   └── [17 more...]       # Other MCP tools
│   │
│   ├── db/                    # Data layer (Drizzle ORM)
│   │   ├── schema.ts          # Core schema definition
│   │   ├── oracle-dig-schema.ts  # Dig/trace schema
│   │   ├── forum-schema.ts    # Forum discussion schema
│   │   ├── fleet-log-schema.ts   # Fleet logging
│   │   ├── vector-schema.ts   # Vector embeddings metadata
│   │   ├── plugin-schema.ts   # Plugin registry
│   │   ├── index.ts           # DB factory, connection pool
│   │   ├── factory.ts         # Storage backend selection
│   │   ├── migrations/        # 45+ Drizzle migrations
│   │   ├── seeders/           # Database seeders (menu, defaults)
│   │   └── [atomic-ops, drizzle-schema, etc.]
│   │
│   ├── storage/               # Storage backend abstractions
│   │   ├── drizzle-sqlite.ts  # SQLite adapter (primary local)
│   │   ├── registry.ts        # Multi-backend dispatcher
│   │   ├── config.ts          # Storage config (D1, Workers)
│   │   ├── audit-log.ts       # Write audit trail
│   │   ├── soft-delete.ts     # Logical delete tracking
│   │   └── migration-repair.ts   # Schema healing
│   │
│   ├── vector/                # Embeddings & vector search
│   │   ├── factory.ts         # Vector store selector (Qdrant, LanceDB, CF)
│   │   ├── provider-detection.ts # Detect available models
│   │   ├── preflight.ts       # Warm up embedding model
│   │   ├── [adapter-*.ts]     # Backend-specific drivers
│   │   └── [search, index, etc.]
│   │
│   ├── search/                # Full-text + semantic search core
│   │   ├── fts.ts             # FTS5 query compilation
│   │   ├── hybrid.ts          # BM25 + vector fusion
│   │   └── [ranking, dedup, etc.]
│   │
│   ├── indexer/               # Batch ingestion pipeline
│   │   ├── cli.ts             # CLI entry (arra mine)
│   │   ├── pipeline.ts        # Pipeline orchestration
│   │   ├── [parsers/]         # Markdown, MDX, text parsers
│   │   └── [strategies/]      # Deterministic ID, chunking
│   │
│   ├── vault/                 # Local file storage management
│   │   ├── cli.ts             # vault init/sync/pull
│   │   ├── handler.ts         # File watch + ingest
│   │   └── [sync, pull, index]
│   │
│   ├── gateway/               # Federation & peer routing
│   │   ├── index.ts           # Gateway plugin
│   │   └── [discovery, routing]
│   │
│   ├── federation/            # Cross-Oracle federation
│   │   └── [peer discovery, query forwarding]
│   │
│   ├── plugins/               # Unified plugin system
│   │   ├── unified-loader.ts  # Load all plugin types (wasm, server, cli)
│   │   ├── runtime-routes.ts  # Plugin route mounting
│   │   ├── runtime-reload.ts  # Lifecycle hooks + hot reload
│   │   ├── unified-server.ts  # Start plugin servers
│   │   ├── watcher.ts         # Manifest change detection
│   │   └── [registry, model, etc.]
│   │
│   ├── middleware/            # 20+ HTTP middlewares
│   │   ├── auth.ts            # API token auth
│   │   ├── cors.ts            # CORS + private network
│   │   ├── rate-limiter.ts    # Request throttling
│   │   ├── compress.ts        # gzip compression
│   │   ├── etag.ts            # ETag caching
│   │   ├── timeout.ts         # Request timeout enforcement
│   │   ├── request-logger.ts  # Structured logging
│   │   ├── tenant.ts          # Multi-tenant routing
│   │   ├── response-format.ts # Unified response envelope
│   │   └── [errors, body-limit, spa, etc.]
│   │
│   ├── server/                # Server lifecycle & init
│   │   ├── api-token-auth.ts  # Token validation logic
│   │   └── [request-context, etc.]
│   │
│   ├── lifecycle/             # Startup/shutdown orchestration
│   │   ├── startup-context.ts # Bootstrap phase
│   │   ├── shutdown.ts        # Graceful drain + exit
│   │   ├── self-test.ts       # Health check on startup
│   │   ├── banner.ts          # ASCII art startup message
│   │   └── [metrics, telemetry]
│   │
│   ├── config/                # Configuration modules
│   │   ├── validate.ts        # Environment validation
│   │   └── [storage, vector, auth config]
│   │
│   ├── mcp/                   # MCP protocol implementation
│   │   ├── index.ts           # MCP server factory
│   │   └── [resources, prompts, sampling]
│   │
│   ├── cli/                   # CLI commands
│   │   ├── index.ts           # Command dispatcher
│   │   └── commands/          # Individual command handlers
│   │
│   ├── knowledge/             # Knowledge graph + concepts
│   │   └── [concept extraction, entity linking]
│   │
│   ├── menu/                  # Menu registry + navigation
│   │   ├── model.ts           # Menu item type
│   │   └── [rendering, seeding]
│   │
│   ├── huginn/                # Huginn event capture (optional)
│   │   └── [capture hooks, memory sweeps]
│   │
│   ├── process-manager/       # Child process orchestration
│   │   └── [spawn, restart, signal handling]
│   │
│   ├── lib/                   # Shared utilities
│   │   └── [parsers, formatters, validators]
│   │
│   ├── simple.html            # Minimal static UI
│   ├── simple-mode.ts         # Simple mode handler
│   ├── dashboard.html         # Dashboard stub
│   ├── arthur.html            # CLI documentation (HTML)
│   └── [chroma-mcp, constants, etc.]
│
├── frontend/                  # React UI (Vite + Tailwind)
│   ├── src/
│   │   ├── components/        # React components
│   │   ├── pages/             # React Router pages
│   │   ├── styles/            # Tailwind + custom CSS
│   │   ├── App.tsx            # Router setup
│   │   └── main.tsx
│   ├── index.html
│   ├── vite.config.ts
│   └── package.json
│
├── cli/                       # CLI implementation (arra CLI)
│   └── src/
│       ├── cli.ts             # Entry point
│       └── commands/          # Command handlers
│
├── workers/                   # Cloudflare Workers deployments
│   ├── mcp/                   # Worker MCP server
│   ├── studio/                # Worker Studio UI proxy
│   └── federation/            # Worker federation router
│
├── packages/                  # Monorepo packages (if any)
│
├── tests/                     # Integration + contract tests
│   ├── http/                  # HTTP endpoint tests (nested by cluster)
│   │   ├── health/
│   │   ├── search/
│   │   ├── vector/
│   │   ├── [32+ clusters]
│   │   └── core.test.ts       # Opt-in live server tests
│   ├── storage/               # Storage backend tests
│   ├── integration/           # End-to-end tests
│   └── docs/                  # Documentation validation
│
├── benchmarks/                # Performance testing
│   └── [vector search, indexing, etc.]
│
├── scripts/                   # Build + utility scripts
│   ├── calver.ts              # CalVer versioning
│   ├── seed-test-data.ts      # Test data seeding
│   ├── export-openapi.ts      # OpenAPI export
│   ├── huginn-*.ts            # Huginn integrations
│   └── [vault-rsync, fleet-ingest, etc.]
│
├── docs/                      # User & developer documentation
│   ├── README.md              # Docs index
│   ├── API-REFERENCE-INDEX.md # HTTP API guide
│   ├── architecture.md        # Architecture overview
│   ├── architecture/          # Detailed architecture docs
│   ├── LOCAL-DEV.md           # Local development setup
│   ├── CLI-GUIDE.md           # CLI reference
│   ├── QUICKSTART-10MIN.md    # Docker quickstart
│   ├── MCP-FROM-OPENAPI.md    # MCP tool generation
│   ├── DOCKER-MCP-TOOLKIT.md  # Docker MCP setup
│   ├── deploy-*.md            # Deployment guides (Cloudflare, DO, Vercel)
│   ├── decisions/             # Architecture decision records
│   ├── design/                # Design docs (UI, interaction)
│   ├── examples/              # Example code + recipes
│   └── issues/                # Tracking known issues
│
├── catalog/                   # Oracle family registry
│   └── [Oracle instance metadata]
│
├── specs/                     # Specification documents
│   └── [Technical specs for features]
│
├── e2e/                       # End-to-end / playwright tests
│   └── [UI automation]
│
├── api/                       # API documentation / OpenAPI specs
│   └── [auto-generated or manual specs]
│
├── sidecar/                   # Sidecar services (optional)
│   └── [Standalone utilities]
│
├── services/                  # Long-running services
│   └── file-watcher.ts        # Watch local files for changes
│
├── CLAUDE.md                  # Project conventions + AI guidelines
├── AGENTS.md                  # Operating contract (team model, PR flow)
├── DESIGN.md                  # Frontend design principles
├── README.md                  # Main project README
├── CHANGELOG.md               # Release history
├── CONTRIBUTING.md            # Contribution guidelines
├── LICENSE                    # BUSL-1.1
├── package.json               # Root workspace + dependencies
├── bun.lock                   # Bun lockfile
├── bunfig.toml                # Bun runtime config
├── tsconfig.json              # TypeScript config
├── playwright.config.ts       # E2E test config
├── drizzle.config.ts          # Drizzle migration config
├── ecosystem.config.cjs       # PM2 process manager config
├── ecosystem.oracle-stack.config.js  # Multi-instance config
├── docker-compose.yml         # Local docker compose
├── docker-compose.prod.yml    # Production compose
├── docker-compose.ci.yml      # CI compose
├── Dockerfile                 # Multi-stage Docker build
├── .gitignore
├── .env.example
└── .env.production.example
```

### Philosophy Behind Directory Organization

1. **Routes are first-class abstractions:** Each cluster (`src/routes/<cluster>/`) is a self-contained Elysia sub-app. No shared route logic; each cluster is responsible for its own contracts.

2. **Test mirror:** Tests live alongside routes (`tests/http/<cluster>/<endpoint>.test.ts`). This makes test discovery and scoping straightforward.

3. **Core layers (search, indexer, vector, vault, db) are shared:** Business logic shared by CLI, HTTP, and MCP lives in `src/` toplevel, not duplicated in each route.

4. **Plugins are separate runtime:** Plugin system (`src/plugins/`) is orthogonal; plugins can export CLI commands, HTTP routes, MCP tools, and sidecars simultaneously.

5. **Configuration is centralized:** All config parsing in `src/config.ts` + `src/config/` subdirs. Environment validation happens at startup, not per-request.

---

## 4. Core Abstractions & Dependencies

### Core Layers (Business Logic)

| Layer | Purpose | Key Files |
|-------|---------|-----------|
| **Search** | Hybrid BM25 + vector search, deduplication, ranking | `src/search/fts.ts`, `hybrid.ts`, `dedup.ts` |
| **Vector** | Embeddings pipeline, model detection, backend adapters | `src/vector/factory.ts`, `provider-detection.ts`, `preflight.ts` |
| **Indexer** | Batch document ingestion, deterministic IDs, chunking | `src/indexer/cli.ts`, `pipeline.ts`, `src/indexer/strategies/` |
| **Database** | Schema (Drizzle), migrations, query builders | `src/db/schema.ts`, `src/db/migrations/`, `src/storage/` |
| **Vault** | Local file storage, sync, pull, watch | `src/vault/cli.ts`, `handler.ts` |
| **Federation** | Cross-Oracle queries, peer discovery | `src/federation/`, `src/gateway/` |
| **Knowledge** | Concept extraction, entity linking, graph | `src/knowledge/` |
| **Plugins** | Unified plugin loading, lifecycle hooks | `src/plugins/unified-loader.ts`, `runtime-reload.ts` |
| **MCP** | MCP server implementation, tool registry | `src/mcp/index.ts`, `src/tools/` |

### Route Clusters (HTTP/MCP Surfaces)

**32+ route clusters**, each a self-contained Elysia sub-app under `src/routes/<cluster>/`:

- **Core surfaces:** `health/`, `search/`, `vector/`, `ask/`, `menu/`, `plugins/`, `mcp/`
- **Knowledge management:** `memory/`, `knowledge/`, `concepts/`, `research/`, `learn/`
- **Data management:** `vault/`, `indexer/`, `supersede/`, `files/`, `export/`
- **Observability:** `dashboard/`, `metrics/`, `traces/`, `sessions/`
- **Admin:** `auth/`, `settings/`, `tenants/`, `profile/`
- **Collaboration:** `forum/`, `feed/`, `canvas/`
- **Scheduling:** `schedule/`, `indexer-daemon/`
- **Compatibility:** `compat.ts` (legacy routes for older clients)

### Middleware Stack (20+)

Applied in layered composition (`src/server.ts`, lines 122-147):

1. **Request pipeline:** logging, correlation IDs, tenant routing
2. **Security:** CORS, private network, API token auth, rate limiting
3. **Headers:** security headers, API version, content negotiation
4. **Size/Format:** body limit, compression, response envelope
5. **Cache:** ETag, 304 Not Modified
6. **Error handling:** catch-all error handler

### Dependencies (Direct)

From `package.json`:

| Package | Role |
|---------|------|
| `elysia@^1.4.28` | Web framework (Bun-native, TypeBox schemas) |
| `drizzle-orm@^0.45.2` | ORM for SQLite/D1/PlanetScale |
| `@modelcontextprotocol/sdk@^1.29.0` | MCP server/tool definitions |
| `better-sqlite3@^12.9.0` | SQLite driver (local dev) |
| `sqlite-vec@^0.1.9` | SQLite vector extension (local development) |
| `@lancedb/lancedb@^0.27.2` | LanceDB vector store client |
| `@qdrant/js-client-rest@^1.17.0` | Qdrant vector store client |
| `commander@^14.0.3` | CLI argument parsing |
| `zod@^3.25.76` | Schema validation |
| `@elysiajs/swagger@^1.3.1` | OpenAPI Swagger plugin |
| `@elysiajs/cors@^1.4.1` | CORS middleware |
| `hook-menu@github:Soul-Brews-Studio/hook-menu` | Plugin hook system |
| `agents@^0.16.1` | Agent orchestration (optional) |

### Dev Dependencies

| Package | Role |
|---------|------|
| `typescript@^5.7.2` | Type checking (`tsc --noEmit`) |
| `bun-types@^1.3.12` | Bun API types |
| `drizzle-kit@^0.31.10` | Schema migration tools |
| `wrangler@^4.101.0` | Cloudflare Workers tooling |
| `@playwright/test@^1.61.1` | E2E testing |

### Transitive Dependencies (Patterns)

- **TypeBox** (via Elysia): Runtime schema validation
- **Zod** (explicit): Alternative schema validation (coexists with TypeBox)
- **Drizzle ORM**: Type-safe query building (PostgreSQL, SQLite, D1)
- **Better-SQLite3** (local), Drizzle adapters (prod): Storage abstraction
- **MCP SDK**: Tool definitions, transport, resource types

---

## 5. Data Layer Architecture

### Schema Layers

```
┌─ Drizzle Schema (src/db/schema.ts)
│  └─ Core tables: oracle_documents, oracle_fts, indexing_status
│
├─ Specialized Schemas
│  ├─ oracle-dig-schema.ts       (tracing, call history)
│  ├─ forum-schema.ts            (discussions)
│  ├─ fleet-log-schema.ts        (fleet events)
│  ├─ vector-schema.ts           (embeddings metadata)
│  └─ plugin-schema.ts           (plugin registry)
│
├─ Storage Adapters (src/storage/)
│  ├─ drizzle-sqlite.ts          (primary: SQLite local)
│  ├─ [cf-d1, workers-kv]        (Cloudflare backends)
│  └─ registry.ts                (multi-backend dispatcher)
│
└─ Migrations (45+ in src/db/migrations/)
   └─ Drizzle generates SQL; bun db:push applies
```

### Database Architecture

- **Primary:** SQLite with FTS5 extension (local dev + Docker)
- **Alternate backends:**
  - **Cloudflare D1** (serverless SQL)
  - **Qdrant** (standalone vector DB)
  - **LanceDB** (embedded vector DB)
  - **Vectorize** (Cloudflare vector store)
- **Storage selection:** `ORACLE_STORAGE_BACKEND` env var (sqlite/d1/kv) + factory pattern

### Key Tables

| Table | Purpose |
|-------|---------|
| `oracle_documents` | Main document index (ID, title, body, metadata) |
| `oracle_fts` | FTS5 index for full-text search |
| `oracle_vectors` | Vector embeddings + metadata |
| `oracle_menu_items` | Menu registry |
| `oracle_plugins` | Plugin metadata + manifest |
| `indexing_status` | Current indexing state |
| `oracle_dig_*` | Tracing / observability tables |
| `forum_*` | Discussion threads |
| `fleet_log_*` | Fleet event log |

---

## 6. API Surface

### HTTP API (Elysia Routes)

**Base:** `http://localhost:47778/api/`

#### OpenAPI/Swagger
- **Endpoint:** `/api/docs` (Swagger UI), `/api/docs/json` (OpenAPI spec)
- **Auto-generated** from TypeBox schemas in each route cluster

#### Main Surfaces

| Cluster | Endpoints | Purpose |
|---------|-----------|---------|
| **health** | `GET /api/health` | Liveness + readiness |
| **search** | `POST /api/v1/search`, `POST /api/v1/ask` | FTS + semantic search |
| **vector** | `GET /api/v1/vector/status`, `POST /api/v1/vector/search` | Vector DB status + search |
| **menu** | `GET /api/v1/menu`, `POST /api/v1/menu/items` | Navigation registry |
| **plugins** | `GET /api/v1/plugins`, `POST /api/v1/plugins/reload` | Plugin discovery + reload |
| **memory** | `POST /api/v1/memory/save`, `GET /api/v1/memory/<id>` | Document persistence |
| **supersede** | `POST /api/v1/supersede` | Reversible updates |
| **indexer** | `POST /api/v1/indexer/mine`, `GET /api/v1/indexer/status` | Ingest pipeline |
| **vault** | `POST /api/v1/vault/sync` | Vault operations |
| **export** | `POST /api/v1/export`, formats: JSON/CSV/Markdown | Data export |
| **knowledge** | `GET /api/v1/concepts`, `POST /api/v1/knowledge/link` | Knowledge graph |
| **traces** | `GET /api/v1/traces`, `POST /api/v1/traces/record` | Call tracing |
| **schedule** | `POST /api/v1/schedule/job`, `GET /api/v1/schedule/jobs` | Job scheduling |
| **dashboard** | `GET /api/v1/dashboard/stats` | System metrics |
| **settings** | `GET /api/v1/settings`, `POST /api/v1/settings/update` | Configuration |
| **auth** | `POST /api/v1/auth/token` | Token validation (internal) |
| **mcp** | `GET /api/v1/mcp/tools`, `/api/mcp` (streaming) | MCP tool discovery |
| **[+13 more]** | | Federation, forum, sessions, etc. |

### Response Envelope

All responses wrapped in `ResponseEnvelope` (from `src/routes/response-envelope.ts`):

```json
{
  "data": { /* route-specific data */ },
  "meta": {
    "timestamp": "2026-07-28T06:11:00Z",
    "version": "26.7.26-alpha.227",
    "requestId": "req-uuid",
    "status": "success"
  },
  "error": null
}
```

### MCP Tools (27 Exported Functions)

Defined in `src/tools/` and registered via `src/routes/mcp/`:

| Tool | Purpose |
|------|---------|
| `muninn_search` | Primary search tool (hybrid FTS + vector) |
| `muninn_learn` | Ingest documents into memory |
| `muninn_read` | Read document with provenance |
| `muninn_ask` | Ask questions with LLM grounding |
| `muninn_recap` | Generate session summaries |
| `muninn_trace` | Record function calls / traces |
| `muninn_forum_*` | Discussion management (create thread, post, etc.) |
| `muninn_schedule` | Job scheduling |
| `muninn_supersede` | Update documents reversibly |
| `muninn_verify` | Verify citations |
| `muninn_concepts` | Extract concepts from text |
| `muninn_oracle_info` | Oracle metadata + version |
| `muninn_list_*` | List documents, concepts, sessions |
| `[+13 more]` | Handoff, inbox, profile, etc. |

---

## 7. Deployment & Runtime Models

### Docker (Primary)

#### HTTP Server Image
```bash
docker run --rm -d \
  -p 47778:47778 \
  -v /data:/data \
  -v $HOME/notes:$HOME/notes:ro \
  ghcr.io/soul-brews-studio/arra-oracle-v3:http
```

- Single-stage build: `bun build src/server.ts`
- Runs `bun dist/server.js` on port `47778`
- Persists data to `/data` (SQLite, vector stores, logs)
- Reads notes from mounted volume (read-only)

#### MCP Stdio Image
```bash
docker run --rm -i \
  -v arra-oracle-data:/data \
  ghcr.io/soul-brews-studio/arra-oracle-v3:stdio
```

- Stdio transport for Claude/agents
- Shares same data volume as HTTP server
- Exports MCP tools directly

### Local Development

```bash
# Install
bun install

# Type-check
bunx tsc --noEmit

# Run HTTP server
bun run server

# Or run vector server
bun run vector

# CLI
bun dist-cli/index.js mine ~/notes
```

### Cloudflare Workers

#### MCP Worker
```bash
bun run cloudflare:mcp:dev
# or
bun run cloudflare:mcp:deploy --config workers/mcp/wrangler.jsonc
```

Bindings:
- `ORACLE_DB` (D1 database)
- `ORACLE_VECTORIZE` (Vectorize index)
- `ORACLE_STATE` (KV namespace)

#### Studio UI Worker
```bash
bun run cloudflare:studio:dev
bun run cloudflare:studio:deploy --config workers/studio/wrangler.jsonc
```

Serves built frontend + proxies to backend.

#### Federation Worker
```bash
bun run cloudflare:federation:dev
bun run cloudflare:federation:deploy --config workers/federation/wrangler.jsonc
```

Routes cross-Oracle queries.

### Environment Variables

Core:
- `PORT` (default 47778)
- `ORACLE_DATA_DIR` (SQLite + vector data)
- `ORACLE_STORAGE_BACKEND` (sqlite/d1/kv)
- `ORACLE_VECTOR_BACKEND` (sqlite-vec/qdrant/lancedb/cloudflare-vectorize)
- `ORACLE_VECTOR_DB` (lancedb/qdrant connection string, optional)
- `ORACLE_EMBEDDING_MODEL` (bge-m3/nomic/qwen3)
- `ARRA_API_TOKEN` (optional bearer auth)

Cloudflare:
- `ORACLE_DB` (D1 binding)
- `ORACLE_VECTORIZE` (Vectorize binding)
- `ORACLE_STATE` (KV binding)

---

## 8. Development Workflow

### Branch Strategy

- **`alpha`:** Working trunk (PRE-RELEASE). All feature work targets here.
- **`main`:** Stable trunk (STABLE release, tagged vX.Y.Z marked latest). **Never push directly; gated by `.claude/hooks/block-push-main.sh`.**

### Versioning

CalVer with wall-clock suffix: `vYY.M.D-alpha.HMM`
- Example: `26.7.26-alpha.227` (26 years, July, 26th, 02:27 UTC)
- Always alpha by default; stable releases only on explicit user direction
- `scripts/calver.ts` generates version; `calver-release.yml` CI workflow tags + releases

### Test Execution

```bash
# Type-check (required before push)
bunx tsc --noEmit

# Unit tests
bun run test:unit

# Integration tests
bun run test:integration

# Scoped by cluster
bun test --isolate tests/http/search/

# Live server tests (opt-in)
ORACLE_LIVE_CONTRACT=1 bun test --isolate tests/http/core.test.ts
```

**Important:** Always use `bun test --isolate` to avoid module pollution across test suites.

### File Size Constraint

- **≤ 250 lines per file** (source, tests, docs)
- If exceeding, split by concern (no padding with helpers)
- Enforced in PR review

### Commit Message Style

```
<type>: <subject>

<body (optional)>
```

Types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`

---

## 9. Key Conventions & Constraints

### 1. Web Framework: Elysia (Bun-native)

- **No Hono.** Hono → Elysia migration is **complete**.
- **TypeBox schemas** for request/response validation (type-safe at runtime).
- **Sub-app composition:** each route cluster is a `new Elysia()` sub-app, `.use()`-composed in `src/server.ts`.
- **Reference module:** `src/routes/health/` is the cleanest example of a route cluster.

### 2. Database: Drizzle ORM

- **No raw SQL.** Schema changes go through `src/db/schema.ts` + Drizzle migrations.
- **`bun db:push`** applies migrations to SQLite; `bun db:generate` creates new migration files.
- **Drizzle Kit studio:** `bun db:studio` opens a browser UI for schema inspection.

### 3. Runtime: Bun ≥ 1.2

- **No Node-specific APIs.** Use `bun-types`, `node:*` imports from Bun.
- **Bun test:** `bun test --isolate` (isolation flag is required for clean module state).

### 4. Plugin System

- **Unified plugin loader** (`src/plugins/unified-loader.ts`): loads WASM, server, CLI plugins from a manifest.
- **Lifecycle hooks:** plugins can hook into server start, shutdown, reload, index completion, etc.
- **Runtime reload:** hot-reload plugins via `POST /api/v1/plugins/reload` without server restart.

### 5. MCP Surface

- **Tool definitions** in `src/tools/` (one function per tool file).
- **MCP server** via `src/mcp/index.ts` and Elysia route `/api/mcp` (stdio transport).
- **Streamable routes:** `/api/mcp` returns streaming JSON for long-running operations.
- **Tool discovery:** OpenAPI spec auto-exported, MCP manifest exposes all tools.

### 6. Multi-Tenant Isolation

- **Tenant middleware** (`src/middleware/tenant.ts`) routes requests to tenant-scoped data.
- **Tenant ID** from request header or subdomain; falls back to default tenant.
- **DB partition** by tenant; searches only return tenant-scoped documents.

### 7. API Auth

- **API tokens:** optional bearer token (`ARRA_API_TOKEN` env var) + `Authorization: Bearer <token>` header.
- **Protected paths:** most `/api/*` routes are guarded; `/health`, `/peer/*`, `/identity` are public (federation).
- **Token validation:** `src/server/api-token-auth.ts` + middleware hook at `src/server.ts:139-141`.

### 8. Error Handling

- **Unified error middleware** (`src/middleware/errors.ts`) catches all exceptions + returns envelope with error details.
- **Status codes:** 400 (validation), 401 (auth), 403 (forbidden), 404 (not found), 500 (server error).
- **Error envelope:** always includes requestId, timestamp, error message/code.

### 9. Graceful Shutdown

- **Shutdown handler** (`src/lifecycle/shutdown.ts`) drains in-flight requests + closes DB connections.
- **Signal handlers:** SIGTERM, SIGINT registered at startup.
- **Request tracking:** middleware tracks all active requests; shutdown waits for completion before exit.

### 10. Self-Test on Startup

- **Preflight checks** (`src/lifecycle/self-test.ts`): database connectivity, vector model availability, plugin loading.
- **Startup banner** (`src/lifecycle/banner.ts`): ASCII art with version, port, config summary.

---

## 10. Special Features

### Vault System

Local file storage + sync mechanism (`src/vault/`):
- `vault init`: Initialize a local vault directory
- `vault sync`: Bi-directional sync (local ↔ Oracle)
- `vault pull`: Pull documents from Oracle into local files
- File watcher: automatically ingest changes

### Indexing Pipeline

Deterministic, resumable ingestion (`src/indexer/`):
- Parse `.md`, `.mdx`, `.txt` files
- Generate deterministic document IDs (stable across re-runs)
- Skip unchanged files (fingerprint comparison)
- Batch embeddings via Ollama or cloud provider
- Populate FTS + vector indices

### Search: Hybrid BM25 + Vector

Fusion algorithm (`src/search/hybrid.ts`):
1. Run FTS query (BM25 ranking)
2. Run vector query (cosine similarity)
3. Combine scores, deduplicate, rank
4. Return top-K results with citations

### Federation

Cross-Oracle queries (`src/federation/`, `src/gateway/`):
- Discover peer Oracle instances (DHT or registry)
- Forward search/read queries to peers
- Aggregate results, rank, return to client

### Knowledge Graph

Concept extraction + entity linking (`src/knowledge/`):
- Extract entities/concepts from documents
- Link concepts across documents
- Build navigable graph (optional)

---

## 11. Important Files by Use Case

### For Newcomers
1. `README.md` — quick start, overview
2. `CLAUDE.md` — project conventions (versioning, file size, testing)
3. `AGENTS.md` — operating contract (branch rules, PR flow, team model)
4. `docs/LOCAL-DEV.md` — local development setup
5. `src/routes/health/` — reference route cluster

### For Backend Developers
1. `src/server.ts` — middleware + route composition
2. `src/db/schema.ts` — database schema
3. `src/search/` — search algorithms
4. `src/vector/factory.ts` — vector backend selection
5. `src/indexer/cli.ts` — ingestion entry point

### For API Users
1. `docs/API-REFERENCE-INDEX.md` — HTTP endpoint catalog
2. `http://localhost:47778/api/docs` — interactive Swagger UI
3. `docs/http-api-reference.md` — detailed API docs

### For MCP Developers
1. `src/tools/` — tool implementations
2. `src/mcp/index.ts` — MCP server setup
3. `docs/mcp-tools.md` — tool reference
4. `.mcp.json` — stdio transport config

### For DevOps/Deployment
1. `Dockerfile` — container image
2. `docker-compose.yml`, `.prod.yml`, `.ci.yml` — compose configs
3. `docs/deploy-*.md` — platform-specific deploy guides
4. `wrangler.jsonc` files — Cloudflare Workers config

### For Plugin Authors
1. `docs/plugin-quickstart.md` — plugin development
2. `src/plugins/unified-loader.ts` — plugin loading mechanism
3. `src/plugins/runtime-reload.ts` — lifecycle hooks

---

## 12. Architecture Decisions & Trade-offs

### One Memory Core, Multiple Surfaces
- **Decision:** Centralize business logic in `src/` layers; adapters (`routes/`, `tools/`, `cli/`) consume it.
- **Tradeoff:** Slightly more boilerplate per surface; massive consistency + testability gain.
- **Validated by:** no logic duplication, shared test helpers, unified error handling.

### Elysia over Hono
- **Decision:** Bun-native, TypeBox schemas, faster compilation, cleaner plugin system.
- **Tradeoff:** Smaller ecosystem than Express/Hono; some edge cases require raw Bun APIs.
- **Evidence:** Complete Hono → Elysia migration successful; CI builds consistently.

### Drizzle ORM over Raw SQL
- **Decision:** Type-safe query builders, migration history, multi-backend support (SQLite/D1/PlanetScale).
- **Tradeoff:** Slight performance overhead (negligible for SQLite); learning curve.
- **Evidence:** 45+ migrations managed safely; schema evolution is auditable.

### CalVer Versioning (Always Alpha)
- **Decision:** Date-based versions + wall-clock suffix; default to pre-release; stable only on explicit user direction.
- **Tradeoff:** No semantic versioning signal (major.minor.patch); requires discipline to avoid stable releases by accident.
- **Validated by:** `.claude/hooks/block-push-main.sh` prevents accidental stable releases.

### 250-Line File Size Limit
- **Decision:** Keep files small, easy to reason about, fast to test.
- **Tradeoff:** More files, more imports; slightly more cognitive load scanning directory.
- **Evidence:** PR review time is faster; file-level tests are quicker to write.

### Multi-Tenant at HTTP Layer (Not DB)
- **Decision:** Tenant routing via middleware + query filter; not separate databases.
- **Tradeoff:** Requires explicit filtering on every query; single DB schema for all tenants.
- **Evidence:** Simpler backups, migrations, operational model; lower cost than per-tenant DBs.

---

## 13. Known Limitations & Future Directions

### Current Limitations
1. **Test scope:** `bun test tests/http/` crashes intermittently on bun 1.3.14 (run by cluster instead).
2. **Vector providers:** Local embedding via Ollama requires separate Docker container.
3. **Federation:** peer discovery is manual or via DNS; no DHT yet.
4. **Plugin hot-reload:** may leave stale sockets if plugin server crashes; manual restart sometimes needed.

### Future Directions
1. Persistent caching layer (Redis optional) for vector queries
2. Streaming document ingestion (current: batched)
3. Distributed indexing (federation-wide batch jobs)
4. Native WebAssembly plugins (beyond current HTTP server plugins)
5. Real-time collaboration (WebSocket + CRDT for concurrent edits)

---

## 14. Key Entry Points Summary

| Entry Point | Location | Purpose |
|-------------|----------|---------|
| **HTTP Server** | `src/server.ts` + `src/index.ts` | Main Elysia app |
| **MCP Server** | `bin/mcp.ts` | Stdio MCP transport for Claude |
| **CLI** | `bin/arra.ts` | `arra` command (mine, search, learn, export) |
| **Docker** | `Dockerfile` | Multi-stage build → `/dist/server.js` |
| **Frontend** | `frontend/src/main.tsx` | React + React Router UI |
| **Tests** | `bun test tests/http/<cluster>/` | Scoped test execution |
| **Plugins** | `src/plugins/unified-loader.ts` | Load WASM/server/CLI plugins |
| **Configuration** | `src/config.ts` | Env var parsing + validation |

---

## 15. Monitoring & Observability

### Health Endpoints
- `GET /api/health` — liveness + readiness + document/indexing counts
- `GET /api/v1/dashboard/stats` — detailed system metrics

### Tracing
- `POST /api/v1/traces/record` — record function calls
- `GET /api/v1/traces` — retrieve traces by correlation ID
- `muninn_trace` MCP tool — record from Claude

### Logging
- Structured JSON logs via `src/middleware/request-logger.ts`
- Request correlation IDs tracked across layers
- Log level configurable via `LOG_LEVEL` env var

### Metrics
- Request latency, count, status distribution
- Document count, FTS indexed, vector indexed
- Indexing progress (documents per second)
- Cache hit rate (ETag, if enabled)

---

## Conclusion

**Arra Oracle V3** is a production-grade, modular memory layer designed for semantic search and knowledge federation in the Oracle family. Its architecture prioritizes:

1. **Single core, multiple surfaces:** business logic in `src/`, adapters in `routes/tools/cli/`
2. **Type safety:** TypeScript throughout, Drizzle ORM, TypeBox schemas, Zod validation
3. **Composability:** Elysia sub-app routes, Drizzle migrations, unified plugin system
4. **Observability:** structured logging, tracing, health checks, metrics
5. **Deployability:** Docker (primary), Cloudflare Workers, local Bun dev
6. **Extensibility:** plugin lifecycle hooks, MCP tools, custom routes

The codebase is organized for **team collaboration** (CLAUDE.md, AGENTS.md), with strict conventions (250-line files, CalVer, alpha-first versioning) and comprehensive testing (scoped by cluster, isolated test execution).

For questions or contributions, see `docs/README.md`, `docs/LOCAL-DEV.md`, and `CONTRIBUTING.md`.
