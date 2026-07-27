# Arra Oracle V3 — Code Patterns & Implementation Guide

**Project:** arra-oracle-v3 (formerly oracle-v2) — MCP Memory + Search Layer for the Oracle family
**Version:** 26.7.26-alpha  
**Generated:** 2026-07-28

---

## Table of Contents

1. [Quick Start & Entry Points](#quick-start--entry-points)
2. [Architecture Overview](#architecture-overview)
3. [Route Composition Pattern (Elysia)](#route-composition-pattern-elysia)
4. [Plugin System](#plugin-system)
5. [Database & Schema](#database--schema)
6. [Vector Search & Embeddings](#vector-search--embeddings)
7. [MCP Tools & Search Implementation](#mcp-tools--search-implementation)
8. [Configuration & Environment](#configuration--environment)
9. [Error Handling & Middleware](#error-handling--middleware)
10. [Lifecycle & Graceful Shutdown](#lifecycle--graceful-shutdown)
11. [Safety Guards & Hooks](#safety-guards--hooks)
12. [Interesting Patterns & Idioms](#interesting-patterns--idioms)

---

## Quick Start & Entry Points

### Docker-First Entry Point

The primary deployment path uses Docker. From `README.md`:

```bash
export ARRA_PORT="${ARRA_PORT:-47778}"
export ARRA_URL="http://127.0.0.1:${ARRA_PORT}"
export ARRA_CONTAINER="${ARRA_CONTAINER:-arra-oracle}"
export ARRA_VOLUME="${ARRA_VOLUME:-arra-oracle-data}"
export ARRA_NOTES_DIR="${ARRA_NOTES_DIR:-$HOME/notes}"

mkdir -p "$ARRA_NOTES_DIR"
docker volume create "$ARRA_VOLUME" >/dev/null

docker run --rm -d --name "$ARRA_CONTAINER" \
  -p "${ARRA_PORT}:47778" \
  -v "${ARRA_VOLUME}:/data" \
  -v "${ARRA_NOTES_DIR}:${ARRA_NOTES_DIR}:ro" \
  ghcr.io/soul-brews-studio/arra-oracle-v3:http

# Wait for health
until curl -sf "${ARRA_URL}/api/health" >/dev/null; do sleep 1; done

# Mine notes using the bundled CLI
arra() {
  docker exec "$ARRA_CONTAINER" bun dist-cli/index.js "$@"
}
arra mine ~/notes

# Search
curl -sfS "${ARRA_URL}/api/v1/search?q=runbook&mode=fts&limit=5"
```

### Local Development Entry Points

From `package.json`:

```bash
# HTTP server on :47778
bun run server

# Type-check only (build is type-check)
bunx tsc --noEmit

# Run tests (with isolation to prevent cross-pollution)
bun test --isolate tests/http/<cluster>/
bun run test:unit
bun run test:integration

# Vector sidecar server
ORACLE_VECTOR_READONLY=1 bun src/vector-server.ts

# MCP server via Claude Code
claude mcp add arra-oracle -- docker run --rm -i \
  -v "${ARRA_VOLUME:-arra-oracle-data}:/data" \
  ghcr.io/soul-brews-studio/arra-oracle-v3:stdio
```

### CLI Entry Point

From `cli/src/cli.ts` (handles plugin discovery, command routing):

```typescript
#!/usr/bin/env bun

// Plugin discovery & registration flow
async function loadAll() {
  const { plugins, bundled, user } = await discoverPlugins();
  registerPlugins(plugins);
  const total = bundled + user;
  console.log(`loaded ${total} plugin${total !== 1 ? "s" : ""}`);
}

async function main() {
  const args = process.argv.slice(2);
  
  // Handle --at <oracle-name> prefix (multi-oracle mode)
  const atIndex = args.indexOf("--at");
  if (atIndex >= 0) {
    const target = args[atIndex + 1];
    if (!target) {
      console.error("usage: arra --at <name> <command>");
      process.exit(1);
    }
    process.env.ARRA_AT = target;
    args.splice(atIndex, 2);
  }
  
  const cmd = args[0]?.toLowerCase();
  
  // Routes: version, help, config, serve, search, plugins, etc.
  if (cmd === "--version" || cmd === "-v") {
    console.log(`arra-cli v${CLI_VERSION}`);
    return;
  }
  
  if (!cmd || cmd === "--help") {
    await loadAll();
    const commands = listCommands().map(c => ({ command: c.command, help: c.help }));
    console.log(renderRootHelp(commands));
    return;
  }
  
  // Dispatch to command handlers
  if (cmd === "config") return configCommand(args.slice(1));
  if (cmd === "serve") return serveCommand(args.slice(1));
  if (cmd === "search") return searchCommand(args.slice(1));
  // ... more commands
}
```

---

## Architecture Overview

### System Topology

```
Clients (Notes, Agents, Browsers, MCP Clients)
    │
    ├── Docker HTTP Server (ghcr.io/.../arra-oracle-v3:http)
    ├── Docker stdio MCP (ghcr.io/.../arra-oracle-v3:stdio)
    ├── CLI: arra mine/search/learn/export
    └── Studio UI (Vite frontend + React)
            │
    Elysia HTTP Routes + MCP Tools + Plugin Registry
            │
    SQLite + FTS5 + Optional Vector Stores
            │
    Local vault files (ψ/memory/*), plugin servers
```

### Directory Structure

```
src/
  ├── server.ts                 # Main Elysia app (21 route clusters)
  ├── config.ts                 # Config resolution (env + paths)
  ├── const.ts                  # Constants (collection names, dirs)
  ├── index.ts                  # Export barrel
  │
  ├── routes/                   # 21 Elysia route clusters
  │   ├── health/               # /api/health, /api/stats (reference module)
  │   ├── search/               # /api/search, /api/reflect, /api/list
  │   ├── ask/                  # /api/ask (grounded answers w/ citations)
  │   ├── vector/               # /api/vector, /api/vector/config
  │   ├── mcp/                  # /api/mcp/tools (tool discovery)
  │   ├── plugins/              # /api/plugins (plugin management)
  │   ├── menu/                 # /api/menu (navigation items)
  │   ├── auth/, vault/, learn/ # Other domains
  │   └── ... (17 more)
  │
  ├── tools/                    # MCP tool definitions & handlers
  │   ├── index.ts              # Barrel export
  │   ├── search.ts             # Search tool
  │   ├── learn.ts              # Learn (note ingestion)
  │   ├── recap.ts              # Recap (session summary)
  │   ├── read.ts               # Read document
  │   ├── forum.ts              # Forum threads
  │   └── ... (18 more tools)
  │
  ├── db/                       # Database layer (Drizzle + SQLite)
  │   ├── index.ts              # Module-level connection (lazy proxy)
  │   ├── schema.ts             # Drizzle schema (tenants, documents, FTS, etc.)
  │   └── factory.ts            # Connection factory
  │
  ├── vector/                   # Vector search & embeddings
  │   ├── factory.ts            # Vector store creation (8 adapter types)
  │   ├── adapter.ts            # VectorStoreAdapter interface
  │   ├── adapters/
  │   │   ├── lancedb.ts        # LanceDB (default local)
  │   │   ├── sqlite-vec.ts     # SQLite with sqlite-vec extension
  │   │   ├── qdrant.ts         # Qdrant remote
  │   │   ├── cloudflare.ts     # Cloudflare Vectorize
  │   │   ├── chroma.ts         # Chroma MCP
  │   │   └── ... (3 more)
  │   ├── embeddings.ts         # Embedding provider abstraction
  │   ├── providers/            # ollama, gemini, cloudflare-ai
  │   └── health.ts, config.ts
  │
  ├── plugins/                  # Unified plugin system
  │   ├── unified-loader.ts     # Plugin discovery & loading
  │   ├── unified-manifest.ts   # Plugin manifest schema
  │   ├── registry.ts           # Plugin registry
  │   ├── error-containment.ts  # Error isolation per plugin
  │   ├── unified-server.ts     # Plugin server processes
  │   └── watcher.ts            # Plugin manifest hot-reload
  │
  ├── middleware/               # Request/response middleware (Elysia)
  │   ├── auth.ts               # API key auth
  │   ├── errors.ts             # Error formatting
  │   ├── cors.ts               # CORS + private network preflight
  │   ├── rate-limiter.ts       # Rate limiting
  │   ├── compress.ts           # Compression (brotli/gzip)
  │   ├── response-format.ts    # Unified response shape
  │   ├── tenant.ts             # Multi-tenant routing
  │   └── ... (10 more)
  │
  ├── process-manager/          # PID, health, graceful shutdown (from claude-mem)
  ├── lifecycle/                # Startup, shutdown, self-test
  ├── indexer/                  # Note ingestion & FTS5 indexing
  ├── vault/                    # Knowledge vault (ψ/memory) sync
  ├── workers/                  # Background workers (sleep consolidation, entity backfill)
  └── ... (10+ more domains)

cli/
  ├── src/cli.ts               # CLI entry point
  ├── src/commands/            # Command handlers (serve, search, mine, etc.)
  └── src/plugin/              # CLI plugin loading

.claude/
  ├── agents/                  # Agent role definitions
  ├── commands/                # Slash commands
  ├── hooks/                   # Pre-tool-use guards (e.g., block-push-main.sh)
  ├── knowledge/               # Project philosophy, style guide
  └── settings.json

.github/workflows/
  ├── ci.yml                   # Type-check + tests (Bun 1.2+)
  └── calver-release.yml       # CalVer tagging on version bump
```

---

## Route Composition Pattern (Elysia)

### Reference Route: Health (`src/routes/health/index.ts`)

The health cluster is the documented reference module for new routes:

```typescript
// src/routes/health/index.ts — 21 lines
import { Elysia } from 'elysia';
import { createHealthEndpoint, type HealthEndpointOptions } from './health.ts';
import { createDeepHealthEndpoint } from './deep.ts';
import { createStatsEndpoint } from './stats.ts';
import { createOraclesEndpoint } from './oracles.ts';
import { createOracleProfilesEndpoint } from './oracle-profiles.ts';
import { createThorOracleEndpoint } from './thor.ts';

export function createHealthRoutes(options: HealthEndpointOptions = {}) {
  return new Elysia({ prefix: '/api' })
    .use(createHealthEndpoint(options))
    .use(createDeepHealthEndpoint(options))
    .use(createStatsEndpoint())
    .use(createOraclesEndpoint())
    .use(createOracleProfilesEndpoint())
    .use(createThorOracleEndpoint());
}
```

Each sub-endpoint is a function returning an Elysia sub-app. Compose them with `.use()`.

### Server Composition (`src/server.ts`)

All route clusters are composed in the main server:

```typescript
// Simplified excerpt from src/server.ts (250+ lines of composition)
export function createApp({ unifiedPlugins, runtimeRef = createUnifiedRuntimeRef(unifiedPlugins), dataDir = ORACLE_DATA_DIR, vectorUrl = VECTOR_URL }: CreateAppOptions) {
  const app = new Elysia()
    // Middleware stack (12 middlewares)
    .use(createRequestLoggingMiddleware())
    .use(createCorrelationMiddleware())
    .use(createTenantMiddleware())
    .use(createPrivateNetworkPreflightMiddleware())
    .use(createCorsMiddleware())
    .use(createApiVersionHeaderMiddleware())
    .use(createSecurityHeadersMiddleware())
    .use(createContentTypeMiddleware())
    .use(createBodyLimitMiddleware())
    .use(createApiKeyAuthMiddleware())
    .use(createRateLimiterMiddleware())
    .use(createMetricsLifecycle())
    
    // Swagger + response format
    .use(swagger(createOpenApiSwaggerConfig(pkg.version)))
    .use(createResponseFormatMiddleware())
    .use(createCompressMiddleware())
    .use(createSpaMiddleware())
    .use(createEtagMiddleware())
    
    // Auth guard: checks isApiAuthorized before every protected route
    .onBeforeHandle(({ request, set }) => {
      const pathname = new URL(request.url).pathname;
      if (isApiPathProtected(pathname) && !isApiAuthorized(request)) {
        set.status = 401;
        return unauthorizedApiResponse();
      }
    })
    
    // Response headers
    .onAfterHandle(({ set }) => {
      set.headers['Cache-Control'] = 'no-cache, no-store, must-revalidate';
      set.headers['Referrer-Policy'] = 'strict-origin-when-cross-origin';
    })
    
    .use(createErrorMiddleware())
    .use(gatewayPlugin(dataDir, vectorUrl || undefined))
    
    // Redirects
    .get('/swagger', () => Response.redirect('/api/docs', 308), { detail: { hide: true } })
    
    // Route clusters (21 total, all using the same pattern)
    .use(createHealthRoutes(options))
    .use(searchRoutes)
    .use(askRoutes)
    .use(vectorRoutes)
    .use(vectorConfigApiRoutes)
    .use(knowledgeRoutes)
    .use(researchRoutes)
    .use(verifyRoutes)
    .use(supersedeRoutes)
    .use(forumApi)
    .use(tracesApi)
    .use(scheduleApi)
    // ... 9 more clusters
    .use(createMcpRoutes(mcpToolContext))
    .use(createMcpStreamableRoutes(mcpToolContext));
  
  return app;
}
```

**Key pattern:** Each route cluster is a `new Elysia()` sub-app with its own `prefix`, composed into the main app via `.use()`. All middleware is stacked once; all routes reuse the same stack.

### Search Route Example (`src/routes/search/index.ts`)

```typescript
// src/routes/search/index.ts — 4 lines
import { Elysia } from 'elysia';
import { searchEndpoint } from './search.ts';
import { reflectEndpoint } from './reflect.ts';
import { listEndpoint } from './list.ts';
import { chainSearchEndpoint } from './chain.ts';

export const searchRoutes = new Elysia({ prefix: '/api' })
  .use(searchEndpoint)
  .use(reflectEndpoint)
  .use(listEndpoint)
  .use(chainSearchEndpoint);
```

Each endpoint file defines a handler and schema using Elysia's TypeBox integration.

---

## Plugin System

### Unified Plugin Manifest Format

From `src/plugins/unified-manifest.ts`, a plugin declares capabilities:

```typescript
// plugin.json example in a plugin directory
{
  "name": "my-custom-plugin",
  "enabled": true,
  "config": { "timeout": 5000 },
  
  // Lifecycle: init runs once at startup, destroy on shutdown
  "init": "initHandler",
  "destroy": "cleanupHandler",
  
  // API routes: mounted at /api/plugins/my-custom-plugin/path
  "routes": [
    {
      "path": "/status",
      "method": "get",
      "handler": "getStatus"
    },
    {
      "path": "/action",
      "method": "post",
      "handler": "doAction"
    }
  ],
  
  // CLI subcommands
  "subcommands": [
    {
      "command": "my-command",
      "help": "Do something",
      "handler": "handleMyCommand"
    }
  ],
  
  // MCP tools
  "mcpTools": [
    {
      "name": "my_tool",
      "description": "Tool description",
      "handler": "myToolHandler",
      "inputSchema": { ... TypeBox schema ... }
    }
  ],
  
  // Menu items (navigation)
  "menu": [
    {
      "label": "My Plugin",
      "href": "/my-plugin",
      "icon": "icon-name"
    }
  ],
  
  // Server process (sidecar)
  "server": {
    "entry": "server.ts",
    "port": 8765
  }
}
```

### Plugin Loading & Error Containment (`src/plugins/unified-loader.ts`)

```typescript
// Key excerpt: plugins are discovered, dependency-sorted, then invoked with timeout + error containment

export async function discoverUnifiedPluginManifests(
  options: UnifiedLoaderOptions = {},
): Promise<LoadedUnifiedPlugin[]> {
  const found: LoadedUnifiedPlugin[] = [];
  const seen = new Set<string>();
  
  for (const baseDir of options.dirs ?? defaultUnifiedPluginDirs()) {
    if (!existsSync(baseDir)) continue;
    
    const entries = readdirSync(baseDir, { withFileTypes: true });
    for (const entry of entries) {
      if (!entry.isDirectory() && !entry.isSymbolicLink()) continue;
      
      const pluginDir = join(baseDir, entry.name);
      if (!isContainedPluginPath(baseDir, pluginDir)) {
        warn(options, `skipped ${pluginDir}: plugin directory symlink escapes plugin root`);
        continue;
      }
      
      const loaded = await readPluginDir(pluginDir, options);
      if (!loaded || seen.has(loaded.manifest.name)) continue;
      
      seen.add(loaded.manifest.name);
      found.push(loaded);
    }
  }
  return found;
}

// Each plugin handler gets timeout + error containment
async function invoke(plugin: LoadedUnifiedPlugin, handler: string | undefined, ctx: InvokeContext, timeoutMs: number) {
  if (!handler) return { ok: true, plugin: plugin.manifest.name, source: ctx.source };
  
  const result = await runPluginWithErrorContainment({
    plugin: plugin.manifest.name,
    phase: ctx.source === 'init' || ctx.source === 'destroy' ? ctx.source : 'runtime',
  }, async () => {
    const mod = await import(pathToFileURL(plugin.entryPath).href);
    const fn = handler === 'default' ? mod.default : (mod[handler] ?? mod.default);
    if (typeof fn !== 'function') throw new Error(`handler not found: ${handler}`);
    
    // Timeout wrapper: default 5000ms
    return await withPluginTimeout(() => fn({ ...ctx, config: plugin.manifest.config ?? {} }), timeoutMs);
  });
  
  return result.ok ? result.value : { ok: false, error: result.error };
}
```

**Safety features:**
- Plugins run with **error containment**: one plugin failure doesn't crash the server
- **Timeout enforcement**: default 5000ms, configurable per plugin
- **Dependency sorting**: plugin order respects declared dependencies
- **Symlink containment**: plugins can't escape their directories
- **Enable/disable flag**: `"enabled": false` skips a plugin without uninstalling it

---

## Database & Schema

### Drizzle Schema (`src/db/schema.ts`)

Using Drizzle ORM with SQLite, the schema handles:

```typescript
// Core tables for documents and memory

export const oracleDocuments = sqliteTable('oracle_documents', {
  id: text('id').primaryKey(),
  tenantId: text('tenant_id').default('default').notNull().references(() => tenants.id),
  type: text('type').notNull(),                    // 'note', 'session', 'reflection', etc.
  sourceFile: text('source_file').notNull(),       // Path to source file
  concepts: text('concepts').notNull(),             // JSON array of concept strings
  createdAt: integer('created_at').notNull(),
  updatedAt: integer('updated_at').notNull(),
  indexedAt: integer('indexed_at').notNull(),      // When vector indexed
  validTime: integer('valid_time'),                // Time-validity for stale filtering
  supersededBy: text('superseded_by'),             // Reversible supersession (audit trail)
  supersededAt: integer('superseded_at'),
  supersededReason: text('superseded_reason'),
  origin: text('origin'),                          // 'psi/memory', 'vault', 'api', etc.
  project: text('project'),                        // Oracle instance name
  createdBy: text('created_by'),                   // User or agent
  usageCount: integer('usage_count').default(0),   // Heat metric
  lastAccessedAt: integer('last_accessed_at'),
}, (table) => [
  // Strategic indexes for common queries
  index('idx_source').on(table.sourceFile),
  index('idx_type').on(table.type),
  index('idx_superseded').on(table.supersededBy),
  index('idx_documents_tenant_superseded').on(table.tenantId, table.supersededBy),
  index('idx_origin').on(table.origin),
  index('idx_project').on(table.project),
  index('idx_documents_tenant').on(table.tenantId),
  index('idx_documents_usage_heat').on(table.usageCount, table.lastAccessedAt),
  index('idx_documents_tenant_type_active_updated').on(table.tenantId, table.type, table.supersededAt, table.updatedAt),
  index('idx_documents_tenant_valid_time').on(table.tenantId, table.validTime),
]);

// FTS5 index for full-text search
export const oracleFts = sqliteTable('oracle_fts', {
  id: text('id'),
  content: text('content'),
  concepts: text('concepts'),
});

// Entity linking (concept co-occurrence)
export const oracleEntityLinks = sqliteTable('oracle_entity_links', {
  id: text('id').primaryKey(),
  tenantId: text('tenant_id').default('default').notNull().references(() => tenants.id),
  documentId: text('document_id').notNull().references(() => oracleDocuments.id, { onDelete: 'cascade' }),
  entity: text('entity').notNull(),
  entityKey: text('entity_key').notNull(),
  weight: integer('weight').default(1).notNull(),
  createdAt: integer('created_at').notNull(),
  updatedAt: integer('updated_at').notNull(),
}, (table) => [
  index('idx_entity_links_tenant_key').on(table.tenantId, table.entityKey),
  index('idx_entity_links_tenant_doc').on(table.tenantId, table.documentId),
]);

// Pointer index for efficient concept→documents lookup
export const oraclePointerIndex = sqliteTable('oracle_pointer_index', {
  id: text('id').primaryKey(),
  tenantId: text('tenant_id').default('default').notNull().references(() => tenants.id),
  kind: text('kind').notNull(),        // 'concept', 'entity', 'project'
  key: text('key').notNull(),
  docIds: text('doc_ids').default('[]').notNull(),  // JSON array
  updatedAt: integer('updated_at').notNull(),
}, (table) => [
  index('idx_pointer_tenant_kind_key').on(table.tenantId, table.kind, table.key),
]);

// Indexing jobs queue
export const indexingJobs = sqliteTable('indexing_jobs', {
  id: text('id').primaryKey(),
  docId: text('doc_id').notNull(),
  modelKey: text('model_key').notNull(),            // e.g., 'bge-m3', 'nomic', 'qwen3'
  collection: text('collection').notNull(),
  status: text('status').default('pending').notNull(), // 'pending', 'claimed', 'done', 'error'
  attempts: integer('attempts').default(0).notNull(),
  createdAt: integer('created_at').default(sql`(strftime('%s','now')*1000)`).notNull(),
  claimedAt: integer('claimed_at'),
  finishedAt: integer('finished_at'),
  error: text('error'),
});
```

### Module-Level Connection (Lazy Proxy)

From `src/db/index.ts`:

```typescript
// Lazy proxy pattern: db/sqlite/storage resolve on first access, not import time
function lazyProxy<T extends object>(resolve: () => T): T {
  return new Proxy({} as T, {
    get(_target, prop) {
      const target = resolve() as Record<PropertyKey, unknown>;
      const value = target[prop];
      return typeof value === 'function' ? value.bind(target) : value;
    },
    set(_target, prop, value) {
      (resolve() as Record<PropertyKey, unknown>)[prop] = value;
      return true;
    },
    has(_target, prop) {
      return prop in resolve();
    },
  });
}

let defaultStorage: StorageBackend | null = null;

function openDefaultStorage(): StorageBackend {
  if (!defaultStorage) {
    const readonly = process.env.ORACLE_VECTOR_READONLY === '1';
    defaultStorage = createStorageBackend({ dbPath: defaultDbPath(), readonly });
    if (readonly) console.log('[DB] Opened in READONLY mode (vector sidecar)');
  }
  return defaultStorage;
}

// Exports used everywhere: db, sqlite, storage
export const storage = lazyProxy<StorageBackend>(() => openDefaultStorage());
export const sqlite = lazyProxy<Database>(() => openDefaultStorage().sqlite);
export const db = lazyProxy<BunSQLiteDatabase<typeof schema>>(() => openDefaultStorage().db);

// For tests that need to swap DB after import
export function resetDefaultDatabaseForTests(dbPath?: string): void {
  try { defaultStorage?.close(); } catch {}
  defaultStorage = null;
  defaultStorage = createStorageBackend({ dbPath: resolveDatabasePath(dbPath, defaultDbPathForReset()) });
}
```

**Why lazy proxies?** Tests redirect `ORACLE_DATA_DIR` after other files have imported the DB module. Lazy resolution lets tests reset the connection without needing process-level isolation.

---

## Vector Search & Embeddings

### Factory Pattern: 8 Pluggable Vector Adapters

From `src/vector/factory.ts`:

```typescript
export function createVectorStore(config: VectorStoreConfig = {}): VectorStoreAdapter {
  const type = (clean(config.type) || clean(process.env.ORACLE_VECTOR_DB) || 'lancedb').toLowerCase() as VectorDBType;
  const collectionName = clean(config.collectionName) || COLLECTION_NAME;
  
  switch (type) {
    case 'sqlite-vec': {
      const dbPath = tenantDataPath(clean(config.dataPath) || clean(process.env.ORACLE_VECTOR_DB_PATH) || VECTORS_DB_PATH);
      const embedder = createConfiguredEmbedder(config);
      return guardVectorStore(new SqliteVecAdapter(collectionName, dbPath, embedder), type, collectionName, embedder, dbPath, config.embeddingModel);
    }
    case 'lancedb': {
      const dbPath = tenantDataPath(clean(config.dataPath) || clean(process.env.ORACLE_VECTOR_DB_PATH) || LANCEDB_DIR);
      const embedder = createConfiguredEmbedder(config);
      return guardVectorStore(new LanceDBAdapter(collectionName, dbPath, embedder), type, collectionName, embedder, dbPath, config.embeddingModel);
    }
    case 'qdrant': {
      const embedder = createConfiguredEmbedder(config);
      return guardVectorStore(new QdrantAdapter(collectionName, embedder, {
        url: clean(config.qdrantUrl) || clean(process.env.QDRANT_URL),
        apiKey: clean(config.qdrantApiKey) || clean(process.env.QDRANT_API_KEY),
      }), type, collectionName, embedder, ...);
    }
    case 'cloudflare-vectorize': {
      return createCloudflareVectorStore(collectionName, config);
    }
    case 'proxy': {
      const contract = requireVectorProxyContract({
        endpoint: config.proxyEndpoint,
        collectionName,
        backend: config.type,
      });
      return new ProxyVectorAdapter(collectionName, contract.baseUrl, contract.timeoutMs);
    }
    case 'turbovec': {
      return new TurboVecAdapter(collectionName, clean(config.proxyEndpoint));
    }
    case 'chroma':
    default: {
      const dataPath = tenantDataPath(clean(config.dataPath) || CHROMADB_DIR);
      const pythonVersion = clean(config.pythonVersion) || '3.12';
      return new ChromaMcpAdapter(collectionName, dataPath, pythonVersion);
    }
  }
}

// All adapters wrapped with embedder identity guard (prevents silent model swaps)
function guardVectorStore<T extends VectorStoreAdapter>(
  store: T, adapterName: VectorDBType, collectionName: string, embedder: EmbeddingProvider,
  storagePath?: string, modelName?: string,
): T {
  return withEmbedderIdentityGuard(store, {
    adapterName,
    collectionName,
    embedder,
    modelName: resolveEmbeddingModel(modelName),
    storagePath,
  });
}
```

**Supported backends:**
- **Local:** LanceDB (default), SQLite-vec, Chroma (MCP)
- **Remote:** Qdrant, Cloudflare Vectorize, TurboVec proxy
- **Custom:** Proxy adapter for any HTTP vector service

### Embedding Provider Chain

```typescript
// Fallback chain: try primary, fall back to alternatives
function createConfiguredEmbedder(config: VectorStoreConfig) {
  const selection = resolveEmbeddingProviderSelection(config.embeddingProvider ?? (config.embeddingModel ? 'ollama' : undefined));
  const provider = selection.provider;      // 'ollama', 'gemini', 'cloudflare-ai', 'none'
  const model = resolveEmbeddingModel(config.embeddingModel);
  const fallbackChain = resolveEmbeddingFallbackChain(config.embeddingFallbackChain);
  
  // Example: 'ollama' → 'gemini' → 'none' (FTS5-only on all failures)
  const chain = [provider, ...fallbackChain].filter((item, index, all) =>
    item !== 'none' && all.indexOf(item) === index  // Deduplicate
  );
  
  if (chain.length > 1 && chain.includes('gemini')) {
    // Multi-provider with Gemini: needs special handling
    return observe(new FallbackEmbeddings(chain.map((item) =>
      item === 'gemini'
        ? new GeminiEmbeddings({ model })
        : createEmbeddingProvider(item, model, options)
    )));
  }
  
  // Single provider or Gemini-only
  if (provider === 'gemini' && fallbackChain.length === 0) return observe(new GeminiEmbeddings({ model }));
  return observe(createEmbeddingProvider(provider, model, options));
}
```

---

## MCP Tools & Search Implementation

### Tool Manifest Registry

From `src/tools/mcp-manifest.ts`:

```typescript
// Unified runtime manifest: all MCP tools in one place
export const mcpTools = [
  // Search family
  {
    name: 'search',
    description: 'Search memory by FTS5 or vector similarity',
    inputSchema: T.Object({ q: T.String(), limit: T.Optional(T.Number()), mode: T.Optional(T.Union([T.Literal('fts'), T.Literal('vector')])) }),
    handler: handleSearch,
  },
  {
    name: 'chain_search',
    description: 'Search with reasoning hops (concept → entity → related)',
    inputSchema: T.Object({ q: T.String(), hops: T.Optional(T.Number()) }),
    handler: handleChainSearch,
  },
  
  // Learn / note ingestion
  {
    name: 'learn',
    description: 'Ingest and index a note',
    inputSchema: T.Object({
      source: T.String(),
      content: T.String(),
      title: T.Optional(T.String()),
      concepts: T.Optional(T.Array(T.String())),
    }),
    handler: handleLearn,
  },
  
  // Recap (session summary)
  {
    name: 'recap',
    description: 'Summarize session activity and store recap',
    inputSchema: T.Object({
      title: T.String(),
      summary: T.String(),
      duration_minutes: T.Optional(T.Number()),
    }),
    handler: handleRecap,
  },
  
  // Forum (threaded conversation)
  {
    name: 'thread_create',
    inputSchema: T.Object({ title: T.String(), initial_post: T.String() }),
    handler: handleThread,
  },
  
  // ... 18 more tools
];

export function mcpToolByName(name: string): RuntimeMcpToolManifest | undefined {
  return mcpTools.find(t => t.name === name);
}

export function toMcpToolDefinition(tool: RuntimeMcpToolManifest): ToolDefinition {
  return {
    name: tool.name,
    description: tool.description,
    inputSchema: toJsonSchema(tool.inputSchema),
  };
}
```

### Search Tool Implementation

Search has its own module structure (`src/tools/search/`):

```typescript
// Search dispatches to FTS5 or vector backend
export async function handleSearch(input: OracleSearchInput, ctx: ToolContext): Promise<ToolResponse> {
  const { q, limit = 10, mode = 'auto', concepts, rank_by } = input;
  
  if (mode === 'fts' || mode === 'auto') {
    // Full-text search always available
    const ftsResults = await querySqliteFts(q, { limit });
    
    // Combine with vector if available
    if (mode === 'auto' && vectorAvailable()) {
      const vectorResults = await vectorSearch(q, { limit });
      return combineResults(ftsResults, vectorResults, { rankBy: rank_by });
    }
    
    return combineResults(ftsResults, [], { rankBy: rank_by });
  }
  
  if (mode === 'vector') {
    if (!vectorAvailable()) {
      return {
        status: 'degraded',
        results: [],
        message: 'Vector search unavailable, using FTS5 fallback',
      };
    }
    return vectorSearch(q, { limit });
  }
}

// Vector search with confidence scoring
export async function vectorSearch(query: string, options: VectorSearchOptions = {}): Promise<CombinedSearchResult[]> {
  const embedder = await getEmbedder();
  const embedding = await embedder.embed(query);
  
  const vectorStore = getVectorStore();
  const hits = await vectorStore.search(embedding, { limit: options.limit ?? 10 });
  
  return hits.map(hit => ({
    id: hit.id,
    score: normalizeFtsScore(hit.similarity, 'vector'),
    confidence: confidenceForResult({ vectorScore: hit.similarity }),
    provenance: provenanceForResult({ vectorModel: embedder.model }),
    text: hit.text,
    metadata: hit.metadata,
  }));
}

// Combine FTS5 + vector results with smart ranking
export function combineResults(
  ftsResults: FtsResult[],
  vectorResults: VectorResult[],
  options: { rankBy?: 'relevance' | 'recency' } = {},
): CombinedSearchResult[] {
  const combined = new Map<string, CombinedSearchResult>();
  
  // Add FTS results
  ftsResults.forEach(r => {
    combined.set(r.id, {
      id: r.id,
      ftsScore: r.score,
      vectorScore: undefined,
      combined: r.score,  // Will recompute if vector added
      rank: 0,
      text: r.text,
    });
  });
  
  // Merge vector results
  vectorResults.forEach(r => {
    const existing = combined.get(r.id);
    if (existing) {
      existing.vectorScore = r.score;
      // Weighted average: 0.4 * fts + 0.6 * vector
      existing.combined = 0.4 * (existing.ftsScore ?? 0) + 0.6 * r.score;
    } else {
      combined.set(r.id, {
        id: r.id,
        ftsScore: undefined,
        vectorScore: r.score,
        combined: r.score,
        rank: 0,
        text: r.text,
      });
    }
  });
  
  // Sort by ranking strategy
  const results = Array.from(combined.values());
  if (options.rankBy === 'recency') {
    results.sort((a, b) => (b.timestamp ?? 0) - (a.timestamp ?? 0));
  } else {
    results.sort((a, b) => b.combined - a.combined);
  }
  
  return results;
}
```

---

## Configuration & Environment

### Config Resolution (`src/config.ts`)

Configuration follows a priority chain:

```typescript
// ES Module compatibility for __dirname
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
const PROJECT_ROOT = path.resolve(__dirname, '..');

// HOME resolution (fail fast if not set)
const envText = (key: string): string => process.env[key]?.trim() || '';
const home = envText('HOME') || envText('USERPROFILE');
if (!home) throw new Error('HOME environment variable not set — cannot resolve paths');
export const HOME_DIR = home;

// Core paths with env overrides
export const PORT = parseInt(String(envText('ORACLE_PORT') || envText('PORT') || C.ORACLE_DEFAULT_PORT), 10);

// Data directory: env override OR ~/.arra-oracle-v2/
export const DEFAULT_ORACLE_DATA_DIR = path.join(HOME_DIR, C.ORACLE_DATA_DIR_NAME);
export const ORACLE_DATA_DIR = envText('ORACLE_DATA_DIR') || path.join(HOME_DIR, C.ORACLE_DATA_DIR_NAME);

// Database path: env OR DATABASE_URL OR data-dir/oracle.db
function pathFromDatabaseUrl(value?: string): string {
  value = value?.trim();
  if (!value) return '';
  try {
    const url = new URL(value);
    if (url.protocol === 'file:') return fileURLToPath(url);
    if (url.protocol === 'sqlite:' || url.protocol === 'sqlite3:') {
      return decodeURIComponent(url.pathname || url.host);
    }
  } catch {
    return value;
  }
  return value;
}

export const DB_PATH = envText('ORACLE_DB_PATH') || pathFromDatabaseUrl(envText('DATABASE_URL')) || path.join(ORACLE_DATA_DIR, C.ORACLE_DB_FILE);

// REPO_ROOT: priority 1) env override, 2) ORACLE_DATA_DIR if has ψ/, 3) PROJECT_ROOT if has ψ/, 4) ORACLE_DATA_DIR
// (Data dir wins over project root so accidental ψ/ in checkout doesn't override real indexed data)
export const REPO_ROOT = envText('ORACLE_REPO_ROOT') ||
  (fs.existsSync(path.join(ORACLE_DATA_DIR, 'ψ')) ? ORACLE_DATA_DIR :
   fs.existsSync(path.join(PROJECT_ROOT, 'ψ')) ? PROJECT_ROOT : ORACLE_DATA_DIR);

// Derived paths
export const FEED_LOG = path.join(ORACLE_DATA_DIR, C.FEED_LOG_FILE);
export const PLUGINS_DIR = path.join(ORACLE_DATA_DIR, C.PLUGINS_DIR_NAME);
export const VECTORS_DB_PATH = path.join(ORACLE_DATA_DIR, C.VECTORS_DB_FILE);
export const LANCEDB_DIR = path.join(ORACLE_DATA_DIR, C.LANCEDB_DIR_NAME);
export const CHROMADB_DIR = path.join(HOME_DIR, C.CHROMADB_DIR_NAME);

// Ensure data directory exists on import
if (!fs.existsSync(ORACLE_DATA_DIR)) {
  fs.mkdirSync(ORACLE_DATA_DIR, { recursive: true });
}

// Vector proxy routing (phase 1.2 of vector architecture separation)
// VECTOR_URL: if set, proxy vector calls to this base URL
// VECTOR_FALLBACK: what to do when proxy unreachable ('fts5' = serve FTS5-only)
// ORACLE_EMBEDDER: unset auto-selects local Ollama; 'none' = FTS5-only; remote = needs ORACLE_EMBEDDER_URL
export function resolveVectorUrl(
  env: Record<string, string | undefined> = process.env,
  argv: string[] = process.argv,
): string {
  if (env.ORACLE_VECTOR_SERVER === '1' || isVectorServerEntrypoint(argv[1])) return '';
  if (env.VECTOR_URL?.trim()) return env.VECTOR_URL.trim();

  try {
    const dataDir = env.ORACLE_DATA_DIR?.trim() || envText('ORACLE_DATA_DIR') || ORACLE_DATA_DIR;
    const raw = fs.readFileSync(path.join(dataDir, 'vector-server.json'), 'utf-8');
    const config = JSON.parse(raw) as { vectorProxyUrl?: unknown; vectorUrl?: unknown };
    const fromConfig = typeof config.vectorProxyUrl === 'string'
      ? config.vectorProxyUrl
      : typeof config.vectorUrl === 'string' ? config.vectorUrl : '';
    const trimmed = fromConfig.trim();
    return isHttpUrl(trimmed) ? trimmed : '';
  } catch {
    return '';
  }
}

export const VECTOR_URL = resolveVectorUrl();
export const VECTOR_FALLBACK = envText('VECTOR_FALLBACK') || 'fts5';
```

**Key patterns:**
- All paths computed once at module load time (no lazy resolution)
- Env overrides checked first, then sensible defaults
- Guard against unsafe paths (prevents production data loss in tests)
- Auto-create data dir on import (fresh installs via `bunx` just work)

---

## Error Handling & Middleware

### Structured Error Response

From `src/middleware/errors.ts`:

```typescript
export type ApiErrorResponse = {
  success: false;
  error: string;
  code: number;
  details?: unknown;
};

export type StructuredErrorResponse = {
  success: false;
  error: string;
  message: string;
  statusCode: number;
  correlationId: string;
};

export function apiErrorResponse<const T extends string, const C extends number, D>(
  error: T,
  code: C,
  details: D,
): { success: false; error: T; code: C; details: D } {
  return { success: false, error, code, details };
}

// Typed HTTP errors
export class HttpError extends Error {
  constructor(
    message: string,
    readonly statusCode = 500,
    readonly error = statusLabel(statusCode),
  ) {
    super(message);
    this.name = new.target.name;
  }
}

export class BadRequestError extends HttpError {
  constructor(message = 'Bad request') {
    super(message, 400, 'Bad Request');
  }
}

export class NotFoundError extends HttpError {
  constructor(message = 'Not found') {
    super(message, 404, 'Not Found');
  }
}

export class UnprocessableEntityError extends HttpError {
  constructor(message = 'Unprocessable entity') {
    super(message, 422, 'Unprocessable Entity');
  }
}

const STATUS_LABELS: Record<number, string> = {
  400: 'Bad Request',
  401: 'Unauthorized',
  403: 'Forbidden',
  404: 'Not Found',
  405: 'Method Not Allowed',
  409: 'Conflict',
  413: 'Payload Too Large',
  415: 'Unsupported Media Type',
  422: 'Unprocessable Entity',
  429: 'Too Many Requests',
  500: 'Internal Server Error',
  502: 'Bad Gateway',
  503: 'Service Unavailable',
  504: 'Gateway Timeout',
};

function statusLabel(statusCode: number): string {
  return STATUS_LABELS[statusCode] ?? (statusCode >= 500 ? 'Internal Server Error' : 'Request Error');
}
```

### Error Middleware Composition

Error middleware wraps all routes:

```typescript
export function createErrorMiddleware() {
  return new Elysia()
    .onError(({ code, error, set }) => {
      // Handle HttpError subclasses
      if (error instanceof HttpError) {
        set.status = error.statusCode;
        return apiErrorResponse(error.error, error.statusCode, { message: error.message });
      }
      
      // Handle other errors
      const statusCode = numericStatus(code) ?? 500;
      set.status = statusCode;
      return apiErrorResponse(
        statusLabel(statusCode),
        statusCode,
        { message: errorMessage(error), name: errorName(error) },
      );
    });
}
```

---

## Lifecycle & Graceful Shutdown

### Graceful Shutdown Pattern

From `src/lifecycle/shutdown.ts`:

```typescript
type ShutdownStep = { name: string; run: () => void | Promise<void> };

export interface ShutdownOptions {
  timeoutMs?: number;        // Total shutdown timeout
  minDrainMs?: number;       // Minimum drain time before checking active requests
  close: () => Promise<void>; // Server close function
  log?: (message: string) => void;
  exit?: (code: number) => never;
}

let draining = false;
let shuttingDown = false;
let activeRequests = 0;

// Middleware integration: track active requests
export async function trackRequest<T>(handler: () => T | Promise<T>): Promise<T> {
  activeRequests += 1;
  try {
    return await handler();
  } finally {
    activeRequests = Math.max(0, activeRequests - 1);
  }
}

// Wait for in-flight requests to complete
export async function waitForActiveRequests(timeoutMs = 10_000, minDrainMs = 250): Promise<boolean> {
  timeoutMs = safeMs(timeoutMs);
  minDrainMs = safeMs(minDrainMs);
  const deadline = Date.now() + timeoutMs;
  
  if (minDrainMs > 0) await sleep(minDrainMs);
  while (activeRequests > 0) {
    if (Date.now() >= deadline) return false;  // Timeout: force exit
    await sleep(25);
  }
  return true;
}

// Draining response: return 503 while server is shutting down
export function drainingResponseFor(
  request: Request,
  options: DrainingResponseOptions = {},
): Response | null {
  if (!(options.draining ?? draining)) return null;
  
  const healthPaths = options.healthPaths ?? ['/api/health', '/api/v1/health'];
  const pathname = new URL(request.url).pathname;
  if (healthPaths.includes(pathname)) return null;  // Health check always OK
  
  return Response.json(
    { error: 'server is draining', status: 'draining', draining: true },
    { status: 503, headers: { 'Retry-After': String(safeRetryAfterSeconds(options.retryAfterSeconds)) } },
  );
}

// Run shutdown steps sequentially, collecting errors
export async function runShutdownSteps(
  steps: readonly ShutdownStep[],
  log: (message: string) => void = console.warn,
): Promise<void> {
  const errors: string[] = [];
  for (const step of steps) {
    try {
      await step.run();
    } catch (error) {
      const message = error instanceof Error ? error.message : String(error);
      errors.push(`${step.name}: ${message}`);
      log(`[Shutdown] ${step.name} cleanup failed: ${message}`);
    }
  }
  if (errors.length) throw new Error(`shutdown cleanup failed (${errors.join('; ')})`);
}

// Register graceful shutdown on SIGTERM/SIGINT
export function registerGracefulShutdown(options: ShutdownOptions): void {
  const timeoutMs = options.timeoutMs ?? envMs('ARRA_SHUTDOWN_TIMEOUT_MS', 10_000);
  const minDrainMs = options.minDrainMs ?? envMs('ARRA_SHUTDOWN_DRAIN_MS', 250);
  
  const handler = async (signal: SignalName) => {
    if (shuttingDown) return;
    shuttingDown = true;
    draining = true;
    
    (options.log ?? console.warn)(`[Shutdown] Received ${signal}, draining in-flight requests…`);
    
    const allOk = await waitForActiveRequests(timeoutMs, minDrainMs);
    (options.log ?? console.warn)(`[Shutdown] In-flight requests: ${allOk ? 'drained' : 'timeout (forcing close)'}`);
    
    try {
      await options.close();
    } catch (error) {
      (options.log ?? console.warn)(`[Shutdown] Close error: ${error instanceof Error ? error.message : String(error)}`);
    }
    
    (options.exit ?? process.exit)(0);
  };
  
  process.on('SIGTERM', () => handler('SIGTERM'));
  process.on('SIGINT', () => handler('SIGINT'));
}
```

---

## Safety Guards & Hooks

### Pre-Tool-Use Guard: Block Push to Main

From `.claude/hooks/block-push-main.sh`:

This hook (PreToolUse) intercepts Bash commands before the AI executes them:

```bash
#!/usr/bin/env bash
# PreToolUse(Bash) guard — block `git push` to `main` branch.
# Rationale: pushing/merging to `main` triggers a STABLE CalVer release.
# Routine work must go to `alpha` (pre-release). `main` is gated to explicit user direction.
# Exit 2 = block the tool call; stderr is shown to the model as the reason.

input=$(cat)

# Extract command from JSON tool input
cmd=$(printf '%s' "$input" | python3 -c 'import sys,json;
try:
    print(json.load(sys.stdin).get("tool_input",{}).get("command",""))
except Exception:
    print("")' 2>/dev/null)

# Only care about git push invocations.
printf '%s' "$cmd" | grep -qE '(^|[;&|[:space:]])git[[:space:]]+push' || exit 0

reason="BLOCKED: this repo's release policy forbids pushing to 'main' (it triggers a STABLE release via calver-release.yml). Push feature work to the 'alpha' branch instead. If a stable release is genuinely intended, the user must do it explicitly."

# 1) Explicit main target: `git push origin main`, `HEAD:main`, etc.
if printf '%s' "$cmd" | grep -qE '(^|[[:space:]]|:/)main([[:space:]]|$|:)'; then
  echo "$reason" >&2
  exit 2
fi

# 2) Bare `git push` while current branch is `main`
cur=$(git -C "${CLAUDE_PROJECT_DIR:-.}" branch --show-current 2>/dev/null)
if [ "$cur" = "main" ]; then
  echo "$reason (current branch is 'main')" >&2
  exit 2
fi

exit 0
```

**How it works:**
1. Hook intercepts the Bash tool call before execution
2. Parses JSON input to extract the git command
3. Checks for two push-to-main scenarios:
   - Explicit `main` in refspec: `git push origin main`
   - Bare push while current branch is `main`
4. Exit code 2 blocks execution; stderr message is shown to the AI
5. The constraint is enforced in the tool's pre-execution guard, not in the code

### Production Data Guard

From `src/db/production-db-guard.ts`:

```typescript
// Prevent tests from accidentally using production data
export function assertNotProductionDb(dbPath: string, productionDataDir: string): string {
  const resolved = path.resolve(dbPath);
  const realPath = fs.realpathSync(resolved, { encoding: 'utf-8' });
  const prodPath = fs.realpathSync(productionDataDir, { encoding: 'utf-8' });
  
  if (realPath.startsWith(prodPath)) {
    throw new Error(`ABORT: Attempted to open ${realPath}, but it is within the production data directory ${prodPath}`);
  }
  
  return dbPath;
}
```

**Why:** Tests that redirect `ORACLE_DATA_DIR` after other modules import must not accidentally open the real user data directory. This guard prevents that.

---

## Interesting Patterns & Idioms

### 1. **CalVer Version on Wall-Clock Suffix**

From `CLAUDE.md`:

```typescript
// scripts/calver.ts — version format
// v{YY}.{M}.{D}-alpha.{HMM}
// where HMM = wall-clock suffix = HOUR * 100 + MINUTE

// Example: 2026-07-28 14:32 → v26.7.28-alpha.1432
// (The -alpha.227 in README means 02:27, computed as 2*100+27)

// This ensures:
// - Two commits on the same day get different versions
// - The suffix is deterministic (no random hash)
// - Order is stable: v26.7.28-alpha.800 < v26.7.28-alpha.900
```

### 2. **Lazy Proxy for Test-Friendly DB**

Explained above in Database section. The pattern:

```typescript
// Instead of:
export const db = new Database(path);  // Connects at import time

// Use:
export const db = lazyProxy(() => openDefaultStorage().db);  // Connects on first access

// Benefit: tests can reset `ORACLE_DATA_DIR` after import, and the proxy re-resolves
```

### 3. **Embedder Identity Guard**

From `src/vector/embedder-identity.ts`:

```typescript
// Prevent silent model swaps (e.g., user switches embedder, but index wasn't rebuilt)
export function withEmbedderIdentityGuard(
  store: VectorStoreAdapter,
  identity: EmbedderIdentity,
): VectorStoreAdapter {
  const { modelName, adapterName, storagePath } = identity;
  
  // On first use, check if stored model matches current model
  // If mismatch, warn and either: rebuild index, fall back to FTS5, or fail
  
  return {
    async search(query, options) {
      const stored = await store.getMetadata('embedder_model');
      if (stored && stored !== modelName) {
        throw new Error(`Embedder mismatch: index was built with ${stored}, but active embedder is ${modelName}. Rebuild with 'arra reindex' or switch back.`);
      }
      return store.search(query, options);
    },
    // ... other methods
  };
}
```

### 4. **Plugin Error Containment**

From `src/plugins/error-containment.ts`:

```typescript
// Each plugin runs in an error boundary; one plugin crash doesn't cascade
export async function runPluginWithErrorContainment(
  context: { plugin: string; phase: 'init' | 'destroy' | 'runtime' },
  run: () => Promise<unknown>,
): Promise<{ ok: true; value: unknown } | { ok: false; error: string }> {
  try {
    const value = await run();
    return { ok: true, value };
  } catch (error) {
    const message = error instanceof Error ? error.message : String(error);
    console.error(`[Plugin ${context.plugin}] ${context.phase} error: ${message}`);
    return { ok: false, error: message };
  }
}

// If one plugin's init fails, server still starts; status endpoint reports it degraded
```

### 5. **Vector Adapter Polymorphism**

```typescript
// All vector adapters implement VectorStoreAdapter:
export interface VectorStoreAdapter {
  search(embedding: number[], options?: SearchOptions): Promise<VectorResult[]>;
  insert(id: string, embedding: number[], metadata?: Record<string, unknown>): Promise<void>;
  delete(id: string): Promise<void>;
  getMetadata(key: string): Promise<unknown>;
  setMetadata(key: string, value: unknown): Promise<void>;
}

// Switching backends is config-only:
ORACLE_VECTOR_DB=lancedb    # LanceDB (default)
ORACLE_VECTOR_DB=sqlite-vec # SQLite-vec
ORACLE_VECTOR_DB=qdrant      # Qdrant (remote)
ORACLE_VECTOR_DB=cloudflare-vectorize  # Cloudflare Workers
# No code changes required; factory.ts creates the right adapter
```

### 6. **Tenant Isolation via Middleware**

From `src/middleware/tenant.ts`:

```typescript
// Multi-tenant routing without explicit URL paths
export function createTenantMiddleware() {
  return new Elysia()
    .derive(({ headers, request }) => {
      // Read tenant from header or query param (defaults to 'default')
      const tenant = 
        headers['x-oracle-tenant']?.toString() ||
        new URL(request.url).searchParams.get('tenant') ||
        'default';
      
      // Store in context for all downstream handlers
      return { tenant };
    });
}

// In handlers:
export async function search(input: SearchInput, { tenant }: { tenant: string }) {
  // Query includes tenant filter implicitly
  return db.query()
    .from(oracleDocuments)
    .where(eq(oracleDocuments.tenantId, tenant))
    .all();
}
```

### 7. **Correlation IDs for Request Tracing**

From `src/middleware/correlation.ts`:

```typescript
// Every request gets a unique ID for logging
export const REQUEST_ID_HEADER = 'x-request-id';
export const RESPONSE_TIME_HEADER = 'x-response-time-ms';

export function createCorrelationMiddleware() {
  return new Elysia()
    .derive(({ request, set }) => {
      const requestId = request.headers.get(REQUEST_ID_HEADER) || crypto.randomUUID();
      const start = Date.now();
      
      return {
        requestId,
        logContext: { requestId },
        recordResponseTime: () => {
          const elapsed = Date.now() - start;
          set.headers[RESPONSE_TIME_HEADER] = String(elapsed);
        },
      };
    })
    .onAfterHandle(({ recordResponseTime }) => recordResponseTime());
}
```

### 8. **File Size Constraint for Maintainability**

From `AGENTS.md` and `CLAUDE.md`:

```
File size limit: ≤ 250 lines per file
Applies to: source code, tests, and docs

Rationale:
- Easier to reason about in review
- Enforces single responsibility
- Encourages thoughtful refactoring over padding
- Test files stay focused: one behavior per file

When a file would exceed: split by concern, don't create helpers
```

### 9. **Test Isolation with Bun**

From `CLAUDE.md`:

```bash
# Use --isolate flag to prevent test pollution
# (Mock setup in one file was leaking into unrelated tests)
bun test --isolate tests/http/<cluster>/

# Never run bare `bun test` in CI; CI also runs sibling worktree copies
# under agents/, inflating runtime and potentially causing failures

# pathIgnorePatterns = ["**/agents/**"] prevents that
```

### 10. **OpenAPI Export for API Discovery**

From `package.json`:

```bash
bun run openapi:export
# Exports Elysia routes to OpenAPI 3.0 spec
# Used by Swagger UI and MCP tool discovery
```

---

## Final Observations

### Design Coherence

1. **Single memory core, thin adapters:** All surfaces (CLI, HTTP, MCP, plugins, UI) share contracts from `src/tools/` and `src/db/schema.ts`.
2. **Middleware-driven:** Cross-cutting concerns (auth, logging, tenant, compression) are Elysia middleware, not scattered logic.
3. **Pluggable backends:** 8 vector adapters, fallback embedding chains, tenant isolation—all via config, not code changes.
4. **Safety-first:** Production data guards, error containment, graceful shutdown, draining during deploy.

### Philosophy (from `.claude/knowledge/oracle-philosophy.md`)

The repo treats memory and search as infrastructure for the Oracle family. The architecture prioritizes:
- **Confidence ranking** over false precision
- **Reversible operations** (audit trails, supersession history)
- **Observability** (health endpoints, metrics, draining state)
- **Local-first** (Docker + SQLite default) with remote options

### Release Model

- **`alpha` branch is the working trunk** → pre-release tags `vX.Y.Z-alpha.N`
- **`main` branch is gated** → stable releases, requires explicit user direction
- **CalVer wall-clock suffix** ensures unique, deterministic versions on same-day commits
- **Zero force-push discipline** enforces clean history

