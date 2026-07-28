# maw-ui Architecture

**Project**: ARRA Office — Web dashboard for maw-js fleet control  
**Language**: TypeScript + React 19  
**Version**: 1.4.0  
**License**: BUSL-1.1  

---

## Overview

maw-ui is a modern web frontend for visualizing and controlling a distributed fleet of Claude agents (oracles) running on maw-js nodes. It provides real-time federation views, terminal access, mission control, and multi-page standalone apps optimized for various workflows — all backed by a WebSocket feed from the maw-js backend and served either locally or via Cloudflare Workers.

**Key principles**:
- **Multi-page architecture**: Each view is a standalone Vite entry point with its own HTML file (no SPA fragility)
- **WebSocket-first**: Real-time data via `/ws` feed; no polling for active data
- **Federation-aware**: Can point at any maw-js node via `?host=` query param (Drizzle Studio pattern)
- **Hybrid state management**: Zustand stores with localStorage + server sync for cross-device persistence
- **No external dependencies**: Self-contained CSS (Tailwind), audio (local), 3D (Three.js locally)

---

## Directory Structure & Organization Philosophy

```
src/
├── main.tsx                 # Single entry point for /index.html
├── App.tsx                  # Main app container (multi-view nav + routing)
├── index.css                # Global Tailwind CSS
├── vite-env.d.ts            # Vite type definitions
│
├── apps/                    # Standalone page entry points (one per .html file)
│   ├── office.tsx           # Agent grid — status, terminal previews, feeds
│   ├── fleet.tsx            # Fleet-wide view across all sessions
│   ├── dashboard.tsx        # Overview metrics + agent status
│   ├── terminal.tsx         # Full xterm.js terminal (fullscreen)
│   ├── mission.tsx          # Mission control — active tasks + progress
│   ├── chat.tsx             # Cross-agent messaging
│   ├── config.tsx           # Fleet configuration viewer
│   ├── inbox.tsx            # Oracle inbox — messages + handoffs
│   ├── workspace.tsx        # Multi-agent workspace with actions
│   ├── federation.tsx       # 3D immersive view (Three.js + bloom)
│   ├── federation_2d.tsx    # Canvas force-graph of nodes + agents
│   ├── overview.tsx         # Overview grid
│   ├── arena.tsx            # Arena/simulation view
│   ├── talk.html            # Talk interface
│   ├── timemachine.tsx      # Historical replay/timeline
│   └── shrine.tsx           # Shrine/hall of fame view
│
├── core/                    # App shell & mount infrastructure
│   ├── AppShell.tsx         # Shared layout: StatusBar, error boundary, PIN lock
│   │                        # Provides WebSocket context & session management
│   └── mount.tsx            # Standalone app mount helper
│
├── components/              # Reusable UI components
│   ├── AgentCard.tsx        # Card for single agent status
│   ├── AgentRow.tsx         # Row layout for agents
│   ├── AgentAvatar.tsx      # Avatar/chibi rendering
│   ├── TerminalView.tsx     # xterm.js wrapper
│   ├── StatusBar.tsx        # Top bar — connection, counts, view indicator
│   ├── ErrorBoundary.tsx    # React error boundary
│   ├── PinLock.tsx          # PIN lock overlay
│   ├── LoadingSkeleton.tsx  # Loading placeholder
│   ├── Joystick.tsx         # Gamepad-style control
│   ├── UniverseBg.tsx       # Animated space background
│   │
│   ├── federation/          # 3D & 2D federation views
│   │   ├── Canvas2D.tsx     # Force-graph simulation (2D canvas)
│   │   ├── simulation.ts    # Physics simulation for node layout
│   │   ├── draw.ts          # Canvas draw functions
│   │   ├── store.ts         # Zustand store for federation state
│   │   ├── PluginPanel.tsx  # Plugin inspector
│   │   ├── Sidebar.tsx      # Oracle list grouped by node
│   │   ├── types.ts         # Federation-specific types
│   │   └── colors.ts        # Color palette
│   │
│   ├── chat/                # Chat interface components
│   │   ├── ChatBubble.tsx   # Single message bubble
│   │   ├── ThreadCard.tsx   # Thread listing card
│   │   ├── ChibiAvatar.tsx  # Animated chibi avatars
│   │   ├── useChatLog.ts    # Hook for chat history
│   │   └── types.ts         # Chat type definitions
│   │
│   ├── FleetGrid.tsx        # Fleet-wide agent grid + controls
│   ├── RoomGrid.tsx         # Office "room" grid of agents
│   ├── DashboardView.tsx    # Metrics dashboard
│   ├── DashboardPro.tsx     # Advanced dashboard variant
│   ├── ConfigView.tsx       # Configuration UI
│   ├── ChatView.tsx         # Chat interface wrapper
│   ├── InboxView.tsx        # Inbox/asks overlay
│   ├── MissionControl.tsx   # Task monitoring
│   ├── TerminalModal.tsx    # Terminal popup
│   ├── WorktreeView.tsx     # Worktree management
│   ├── ConnectPage.tsx      # Host connection UI
│   ├── OracleSearch.tsx     # Global oracle search
│   ├── FederationView.tsx   # 3D federation wrapper
│   ├── HoverPreviewCard.tsx # Hover preview popover
│   ├── KVTable.tsx          # Key-value table display
│   ├── TaskDetailOverlay.tsx # Task details popup
│   └── [many specialized components for domain-specific views]
│
├── hooks/                   # React hooks for data fetching & state
│   ├── useWebSocket.ts      # WebSocket connection + reconnect logic
│   ├── useSessions.ts       # Main hook: agents, sessions, feed events
│   ├── useFederationList.ts # Federation peer list
│   ├── useFederationData.ts # Federation data fetching
│   ├── useHttpHealth.ts     # HTTP circuit breaker health status
│   ├── useMqtt.ts           # MQTT connection (if enabled)
│   ├── useFileAttach.tsx    # File upload handler
│   ├── useDevice.ts         # Device detection (mobile/tablet)
│   └── useSessions.ts       # [Main data fetching hook]
│
├── lib/                     # Utilities & state management
│   ├── api.ts               # API client + host resolution
│   │                        # - apiUrl(path) / wsUrl(path)
│   │                        # - apiFetch() with circuit breaker
│   │                        # - Host persistence (?host= param)
│   ├── store.ts             # Zustand fleet store
│   │                        # - recentMap, sortMode, density, UI prefs
│   │                        # - Hybrid storage: localStorage + server sync
│   ├── feed.ts              # Feed event parser (no deps)
│   │                        # - parseLine() → FeedEvent
│   │                        # - describeActivity() for summaries
│   │                        # - Tool icons & event types
│   ├── feedStatusStore.ts   # Zustand store for agent statuses
│   │                        # - tracks: busy, ready, idle, crashed
│   ├── previewStore.ts      # Agent terminal previews
│   ├── types.ts             # TypeScript interfaces
│   │                        # - Session, AgentState, FeedEvent, AskItem, etc.
│   ├── peerConnection.ts    # Peer reachability classification
│   │                        # - same-origin, direct, mixed-content-blocked
│   ├── peerConnectionBanner.ts # UI error messages from peer state
│   ├── peerExecClient.ts    # Browser client for POST /api/peer/exec
│   ├── peerProxyClient.ts   # Browser client for POST /api/proxy
│   ├── constants.ts         # Configuration + sort keys
│   ├── avatar.ts            # Avatar/chibi seed generation
│   ├── ansi.ts              # ANSI color stripping
│   ├── sounds.ts            # Audio playback + mute toggle
│   ├── visibility.ts        # Page visibility tracking
│   ├── capturePoller.ts     # Polling helper for capture data
│   └── federation.ts        # Federation utilities
│
├── quickCommands.ts         # Global shortcuts (Cmd+K search, etc.)
└── [test files]
    ├── peerConnection.test.ts
    ├── peerConnectionBanner.test.ts
    ├── peerExecClient.test.ts
    └── peerProxyClient.test.ts
```

**Organization Philosophy**:

- **Separation by concern**: `lib/` = data & utilities; `hooks/` = React logic; `components/` = UI; `apps/` = entry points
- **Multi-page first**: Each `.html` file is a standalone entry point using the `mount()` helper and `AppShell`
- **Feature folders**: Subsystems (federation, chat) get their own folder with types + components
- **Minimal exports**: No barrel exports; imports are explicit (aids code splitting)
- **Zero external UI frameworks**: Pure React + Tailwind (no Material, Bootstrap, etc.)

---

## Entry Points & App Bootstrap

### Multi-Page Architecture

**HTML Files** (17 total in root):
- `index.html` → `/src/main.tsx` → App.tsx (main office view)
- `federation.html` → /src/apps/federation.tsx (3D Three.js view)
- `federation_2d.html` → /src/apps/federation_2d.tsx (2D canvas force-graph)
- `fleet.html` → /src/apps/fleet.tsx
- `dashboard.html` → /src/apps/dashboard.tsx
- `terminal.html` → /src/apps/terminal.tsx
- `mission.html` → /src/apps/mission.tsx
- `chat.html` → /src/apps/chat.tsx
- `config.html` → /src/apps/config.html
- [+ 8 more specialized pages]

Each `.html` file declares a single root div (`<div id="root">`) and a `<script type="module">` pointing to its app file.

**Vite Config** (`vite.config.ts`):
- Rollup `input` map defines one entry point per page
- Vite builds all 17 entry points in parallel
- No shared chunk splitting (each page is self-contained for isolation)
- Tailwind via `@tailwindcss/vite` plugin (per-entry CSS)

### App Initialization Flow

```
index.html
  ↓
/src/main.tsx (ReactDOM.createRoot)
  ↓
<ErrorBoundary>
  ↓
  <PinLock>              # PIN authentication overlay
    ↓
    <App />              # Multi-view router + nav
```

**App.tsx**:
- Sets up main `useWebSocket()` and `useSessions()` hooks
- Manages routing via state (`currentView`) and floating button dispatch
- Renders sub-views (Office, Fleet, Dashboard, etc.) conditionally
- Handles global shortcuts (⌘K search, Cmd+J jump, etc.)

**Standalone Apps** (federation, terminal, etc.):
```
[page].html
  ↓
/src/apps/[page].tsx (calls mount())
  ↓
mount(App)
  ↓
<AppShell view="[page]">
  ↓
  {(ctx) => <YourComponent {...ctx} />}
```

The `AppShell` component handles:
- WebSocket connection + status
- Session/agent data fetch
- StatusBar rendering
- PIN lock
- Error boundary
- Reconnect UI

---

## Core Abstractions

### 1. Communication Layer

#### WebSocket Feed (Real-Time)

**Connection**: `useWebSocket()` hook in `/src/hooks/useWebSocket.ts`
- Opens connection to `wsUrl("/ws")` (resolved via `apiUrl()`)
- Auto-reconnects with exponential backoff (1s → 30s)
- Sends/receives JSON messages

**Message Types**:
- `{ type: "sessions", sessions: Session[] }` — All active tmux sessions + windows
- `{ type: "feed", event: FeedEvent }` — Live oracle activity feed
- `{ type: "feed-history", events: FeedEvent[] }` — Historical feed (on connect)
- `{ type: "recent", agents: [...] }` — Recently active agents
- `{ type: "teams", teams: Team[] }` — Agent team assignments
- `{ type: "previews", data: Record<string, string> }` — Terminal previews
- `{ type: "action-ok", action: string, target: string }` — Async action acknowledgment

**Feed Event Format**:
```
TIMESTAMP | ORACLE | HOST | EVENT | PROJECT | SESSION_ID » MESSAGE
```

Parsed into:
```typescript
interface FeedEvent {
  timestamp: string;
  oracle: string;
  host: string;
  event: FeedEventType;  // PreToolUse, PostToolUse, Stop, SessionEnd, etc.
  project: string;
  sessionId: string;
  message: string;
  ts: number;            // epoch ms
}
```

**Circuit Breaker** (`lib/api.ts` — `apiFetch()`):
- Trips after 5 consecutive failures (isolates flaky connections)
- Opens for 30s; one probe per 30s to test recovery
- Exposes health status to UI (banner on connection loss)
- Handles Chrome Private Network Access (targetAddressSpace)

#### REST API

**Base**: `apiUrl(path)` resolves via host config
- Localhost/same-origin: relative paths
- Remote (`?host=` set): full URL with protocol
- Support: bare `host:port` (defaults HTTPS), explicit `http://`, `https://`

**Key Endpoints**:
- `GET /api/config` — Fleet configuration
- `GET /api/teams` — Agent teams (fallback if WebSocket hasn't delivered)
- `POST /api/peer/exec` — Send signed command to peer
- `POST /api/proxy` — REST relay for LAN peers
- `POST /api/ui-state` — Persist fleet prefs (cross-device sync)
- `POST /api/asks` — Persist inbox asks (agent approval requests)
- `GET /ws/pty` — PTY WebSocket (terminal I/O)
- `GET /ws` — Main feed WebSocket

---

### 2. State Management

#### Zustand Stores

**`useFleetStore` (`lib/store.ts`)**:
- **Persisted** (via `persist` middleware):
  - `recentMap`: recently active agents (name, session, target, lastBusy timestamp)
  - `sortMode`: "active" | "name"
  - `grouped`: boolean (group agents by team)
  - `density`: "cozy" | "compact"
  - `collapsed`: array of collapsed team keys
  - `muted`: boolean (sound mute toggle)
  - `stageMode`: "stage" | "pitch" (visualization mode)
  - `sleptTargets`: agents manually paused via UI
  - `lastView`: last viewed page

- **Non-persisted** (UI-only):
  - `asks`: Inbox items (agent approval requests)
  - `boardItems`, `boardFields`: Project board data
  - `dispatchLog`: Recent task routing events
  - `taskActivities`, `selectedTaskId`: Task detail state
  - `projectBoard*`: Multi-project state

**Hybrid Storage** (localStorage + server sync):
- `getItem()`: Returns localStorage synchronously (instant hydration)
- `setItem()`: Writes to localStorage immediately + debounced server POST (1s delay)
- `syncFromServer()`: Polls every 5s for cross-device updates
- `recentMap` stays localStorage-only (never POSTed) to avoid churn

**`useFeedStatusStore` (`lib/feedStatusStore.ts`)**:
- Tracks agent status: "busy" | "ready" | "idle" | "crashed"
- Decays: busy → ready (15s no activity), ready → idle (60s no activity)
- Updated by feed events (PreToolUse = busy, PostToolUse = ready, etc.)

**`usePreviewStore` (`lib/previewStore.ts`)**:
- Stores terminal preview text for each agent (up to 120 chars)
- Strips ANSI codes
- Updated via WebSocket `previews` message type

#### Session & Agent Data

**`useSessions()` Hook** (`hooks/useSessions.ts`):

Derives:
- `sessions`: Array of tmux sessions (name, windows[])
- `agents`: Derived from sessions (with status lookups)
- `feedEvents`: Feed log (max 100 recent events)
- `eventLog`: Local event log (max 200 entries)
- `teams`: Team assignments for agents

Handles:
- Feed event processing → status decay (busy/ready/idle)
- Ask detection (agent approval requests from Notification events)
- Sleep/wake actions (agent pause/resume)
- Worktree matching (project field → window name resolution)

```typescript
interface AgentState {
  target: string;          // tmux target, e.g. "01-oracles:0"
  name: string;            // display name
  session: string;         // session name
  windowIndex: number;
  active: boolean;
  preview: string;
  status: PaneStatus;      // busy, ready, idle, crashed
  project?: string;        // worktree or cwd basename
  source?: string;         // peer URL for federated agents
}

interface Session {
  name: string;            // tmux session
  windows: Window[];
  source?: string;
}
```

---

### 3. Federation & Peer Connectivity

**`peerConnection.ts`**: Classifies peer reachability
- `same-origin`: UI and backend on same host
- `direct`: Can reach peer's IP
- `mixed-content-blocked`: HTTPS context → HTTP peer (browser blocks)
- `invalid`: Bad host format

**`peerConnectionBanner.ts`**: Derives error messages
- Suggests fixing based on peer classification
- Offers host reconfiguration UI

**`peerExecClient.ts`**: Browser client for `POST /api/peer/exec`
- Signs commands before relay to peer
- Validates peer identity

**`peerProxyClient.ts`**: Browser client for `POST /api/proxy`
- HTTP relay for LAN peers (bypasses CORS on direct fetch)

---

### 4. Key Views/Pages

| Page | Route | Components | Purpose |
|------|-------|-----------|---------|
| **Office** | `index.html` | RoomGrid, AgentCard, RoomGrid | Agent grid — status, PTY previews, feed summaries |
| **Fleet** | `fleet.html` | FleetGrid, FleetControls | Fleet-wide view — all sessions + broadcast controls |
| **Dashboard** | `dashboard.html` | DashboardView, DashboardPro | Metrics overview — agent count, uptime, activity |
| **Terminal** | `terminal.html` | TerminalView, XTerminal | Full xterm.js PTY — interactive shell |
| **Mission** | `mission.html` | MissionControl, MissionControlCluster | Task progress, active missions |
| **Chat** | `chat.html` | ChatView, ChatBubble, ThreadCard | Cross-agent messaging, threads |
| **Config** | `config.html` | ConfigView, KVTable | Fleet config viewer (read-only) |
| **Inbox** | `inbox.html` | InboxView | Agent approval requests (asks) |
| **Workspace** | `workspace.html` | WorktreeView | Multi-agent workspace editor |
| **Federation 3D** | `federation.html` | FederationView, Three.js | Immersive 3D view of nodes + agents (bloom, particles) |
| **Federation 2D** | `federation_2d.html` | Canvas2D, simulation.ts | Force-graph of all nodes, live message trails |
| **Overview** | `overview.html` | OverviewGrid | Overview metrics grid |
| **Shrine** | `shrine.html` | HallOfFameView | Hall of fame / completed tasks |
| **Arena** | `arena.html` | — | Simulation/battle arena (experimental) |
| **Talk** | `talk.html` | — | Voice/talk interface |
| **Timemachine** | `timemachine.html` | — | Historical replay / timeline |

---

## Dependencies

### Runtime (`package.json`)

| Package | Version | Purpose |
|---------|---------|---------|
| `react` | ^19.0.0 | UI framework |
| `react-dom` | ^19.0.0 | DOM rendering |
| `zustand` | ^5.0.11 | State management |
| `three` | ^0.183.2 | 3D graphics (federation view) |
| `@xterm/xterm` | ^5.5.0 | Terminal emulator |
| `@xterm/addon-fit` | ^0.10.0 | xterm auto-fit addon |
| `@monaco-editor/react` | ^4.7.0 | Code editor (config view) |
| `mqtt` | ^5.15.1 | MQTT client (optional pub/sub) |

### Build (`devDependencies`)

| Package | Version | Purpose |
|---------|---------|---------|
| `vite` | ^6.0.0 | Build tool & dev server |
| `@vitejs/plugin-react` | ^4.3.0 | JSX/React HMR |
| `@tailwindcss/vite` | ^4.2.1 | Tailwind CSS v4 integration |
| `tailwindcss` | ^4.2.1 | Utility CSS framework |
| `typescript` | ^5.7.0 | Type checking |
| `@types/react` | ^19.0.0 | React types |
| `@types/react-dom` | ^19.0.0 | React DOM types |
| `@types/three` | ^0.183.1 | Three.js types |
| `bun-types` | ^1.3.13 | Bun runtime types |

### Framework & Styling

- **React 19**: Latest stable with hooks, Suspense, server components (used for islands)
- **TypeScript 5.7**: Strict mode, full type safety
- **Tailwind CSS 4.2**: Utility-first CSS (vite plugin for per-entry bundling)
- **No external UI framework**: Pure React + Tailwind (no shadcn, Material, Bootstrap)

### Runtime Integrations

- **Three.js**: 3D federation visualization (bloom, particles, lights)
- **xterm.js**: Terminal emulation (ANSI colors, mouse input, resize)
- **Monaco Editor**: Code/config editing interface
- **MQTT**: Optional real-time pub/sub (if server supports)

---

## Build & Deployment

### Development

```bash
npm install
npm run dev    # Vite on :5173, proxy /api + /ws to localhost:3456
```

Vite proxies:
- `/api` → `http://localhost:3456` (HTTP)
- `/ws` → `ws://localhost:3456` (WebSocket)
- `/ws/pty` → `ws://localhost:3456` (PTY WebSocket)

### Production Build

```bash
npm run build
```

Outputs to `dist/`:
- 17 HTML + JS bundles (one per page)
- Embedded CSS (Tailwind v4, per-entry)
- Static assets (favicon, sounds)
- `version.json`: build metadata (git commit, builder, timestamp)

**Rollup Config** (vite.config.ts):
- No shared chunking (each page is independent for isolation)
- Tree-shaking enabled (removes unused code per entry point)
- Version info injected:
  - `__MAW_VERSION__`: package version
  - `__MAW_BUILD__`: build timestamp (Bangkok timezone)
  - `__MAW_COMMIT__`: short git commit hash + dirty marker

### Deployment Options

#### Shape A: Packed Serve (Recommended)

```bash
maw ui --install
```

- Downloads `maw-ui-dist.tar.gz` from GitHub Releases
- Extracts to `~/.maw/ui/dist/`
- maw-js serves it alongside `/api` on `:3456`
- Single port, one process, zero config

#### Cloudflare Workers

```bash
npx wrangler deploy --config wrangler.god.json
```

Deploys to `god.buildwithoracle.com`
- Hosted globally
- Access any node via `?host=<peer>:3456`

#### Docker / Manual

Mount `dist/` folder to any HTTP server (nginx, caddy, etc.)
- Serve index.html + assets
- Proxy `/api` and `/ws` to maw-js backend

---

## Key Design Patterns

### 1. **Query Param Routing** (`?host=`)

Inspired by Drizzle Studio's local-first approach:
- User specifies maw-js node via URL param
- Stored in localStorage
- Resolved in `apiUrl()` and `wsUrl()`
- Supports: `?host=white.local:3456`, `?host=http://oracle-world:3456`, etc.
- Single UI serves all nodes (no per-node builds)

### 2. **Hybrid Storage**

- **localStorage**: Instant hydration on page load
- **Server POST**: Background sync every 5s (debounced)
- **Pattern**: UI writes to localStorage first, then server (fire-and-forget)
- **Use case**: Cross-device sync (office view prefs on multiple laptops)

### 3. **Feed-Driven Status**

Agent status decays without polling:
- Feed event (PreToolUse) → busy
- 15s idle → ready
- 60s idle → idle
- Handled by `useSessions()` → `useFeedStatusStore`

### 4. **Worktree Awareness**

- Project field in feed event: `repo-wt-<worktree-name>`
- Resolves to window name: `<oracle>-<worktree>`
- Prevents cross-contamination between worktrees
- Falls back to main oracle if no matching window

### 5. **Component as Data**

Reusable hooks provide data + render-prop children:
```typescript
<AppShell view="office">
  {(ctx) => <RoomGrid {...ctx} />}
</AppShell>
```

---

## Error Handling

### ErrorBoundary Component
- Catches React render errors
- Fallback UI (graceful degradation)
- Logs to console for debugging

### WebSocket Reconnect
- Exponential backoff: 1s, 2s, 4s, ..., 30s cap
- Shows "Connection lost / Reconnecting..." banner
- Auto-recovers on restore

### Circuit Breaker (apiFetch)
- 5 consecutive failures → open
- 30s cooldown + one probe
- UI banners on health change
- Prevents thundering herd of failed requests

### Type Safety
- TypeScript strict mode throughout
- No `any` types (except data.ts imports)
- Inference for derived state (agents, feed events)

---

## Performance Optimizations

### Code Splitting
- Per-page entry points (no shared chunks)
- Lazy component imports via React.lazy + Suspense
- Tree-shaking removes unused code per bundle

### Rendering
- Zustand subscriptions (granular re-renders)
- useMemo for expensive derivations (agents, feed lookup maps)
- useCallback for stable handler refs (prevents child re-renders)

### Feed Processing
- Max 100 feed events in state (older dropped)
- Fast feed parsing (string split, no regex)
- Debounced status decay (every 5s)

### Terminal
- xterm.js with fit addon (dynamic resize)
- WebSocket for PTY I/O (not polling)
- Async terminal modal opening (doesn't block UI)

---

## Configuration & Constants

### Environment Variables

- `VITE_MAW_URL`: Backend URL (default: `http://localhost:3456`)
  - Resolved to `MAW_HTTP` and `MAW_WS` in vite.config.ts
  - Can be overridden at build time: `VITE_MAW_URL=http://prod.example.com npm run build`

### Runtime Overrides

- `?host=<peer>`: WebSocket + API target (persisted to localStorage)
- localStorage keys:
  - `maw-host`: Stored backend URL
  - `maw-host-recent`: Last 8 hosts visited
  - `maw.fleet`: Zustand fleet store (v3 schema)
  - `office-multiview`: Multi-card view toggle
  - `office-source-filter`: Local/remote filter

---

## Security Notes

### PIN Lock
- Optional authentication layer (PinLock component)
- Stored as hashed PIN in localStorage
- Recommended for publicly accessible instances

### Private Network Access (Chrome PNA)
- Detects private hosts (localhost, .local, 10.x, 192.168.x)
- Adds `targetAddressSpace: 'local'` to fetch requests
- Prompts user for permission (Chrome 142+)

### Mixed Content
- Tracks HTTPS vs HTTP (peerConnection.ts)
- Warns if HTTPS context can't reach HTTP peer
- Suggests http:// URL param for plain-HTTP LAN nodes

---

## Testing

Test files present:
- `peerConnection.test.ts`: Peer classification logic
- `peerConnectionBanner.test.ts`: UI error messaging
- `peerExecClient.test.ts`: Command signing
- `peerProxyClient.test.ts`: Proxy client

Run via bun test or npm test (if configured).

---

## Notable Files & Entry Points

| File | Purpose |
|------|---------|
| `/vite.config.ts` | Build config + version injection |
| `/src/main.tsx` | Root React entry point |
| `/src/App.tsx` | Multi-view router + nav |
| `/src/core/AppShell.tsx` | Shared layout + WebSocket context |
| `/src/hooks/useSessions.ts` | Main data fetching + session management |
| `/src/lib/store.ts` | Zustand fleet store + hybrid storage |
| `/src/lib/api.ts` | API client + circuit breaker |
| `/src/lib/feed.ts` | Feed event parser |
| `/src/apps/*.tsx` | Standalone page entry points |
| `/src/components/federation/Canvas2D.tsx` | 2D force-graph view |
| `/src/components/TerminalView.tsx` | xterm.js wrapper |
| `/*.html` | Entry point HTML files (17 total) |

---

## Future Considerations

- **Real-time collaboration**: Multi-user cursors / shared state
- **Voice integration**: Browser WebRTC for voice comms
- **Mobile parity**: Responsive design for iPad/mobile
- **Offline support**: Service Worker + IndexedDB for offline mode
- **Plugin system**: Custom dashboard widgets from agents
- **Performance**: Virtual scrolling for large agent counts (1000+)

---

**Last Updated**: 2026-07-28  
**Commit**: Auto-generated via build
