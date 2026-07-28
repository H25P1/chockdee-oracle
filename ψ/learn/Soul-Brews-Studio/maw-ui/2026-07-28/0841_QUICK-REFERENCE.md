# maw-ui Quick Reference

**Version**: 1.4.0  
**License**: BUSL-1.1  
**Repository**: [Soul-Brews-Studio/maw-ui](https://github.com/Soul-Brews-Studio/maw-ui)

## Overview

`maw-ui` is a web-based fleet dashboard and federation visualizer for [maw-js](https://github.com/Soul-Brews-Studio/maw-js). It provides real-time monitoring and control of an oracle mesh—a distributed network of autonomous agents running on multiple nodes.

**Core capabilities:**
- Live federation visualization (force-graph 2D + Three.js 3D)
- Fleet-wide agent status, terminal access, and session management
- Mission control with task tracking and progress monitoring
- Cross-agent messaging and handoff coordination
- Terminal emulation (xterm.js) for direct agent interaction
- Peer connectivity detection and routing (local, direct, mixed-content-blocked)

---

## Installation & Setup

### Prerequisites

- **Node.js** 18+ and **npm** (or **Bun**)
- **maw-js** backend running on `localhost:3456` (or remote via `?host=` parameter)

### Quick Start

#### Option A: Development Server (Recommended)

```bash
cd /path/to/maw-ui
npm install
npm run dev
```

- Vite dev server listens on `http://localhost:5173`
- Proxy to maw-js API (`:3456`) with HMR enabled
- Open any view in browser, e.g., `http://localhost:5173`

#### Option B: Production Build

```bash
npm run build
```

- Outputs multi-page bundle to `dist/`
- Each `.html` file is an independent entry point (Vite multi-page mode)
- Preview: `npm run preview`

#### Option C: Install to maw-js

```bash
maw ui --install
```

- Downloads pre-built dist from GitHub Releases
- maw-js serves it alongside `/api` on port `:3456`
- One port, one process, no configuration needed

#### Option D: Cloudflare Workers

```bash
npx wrangler deploy --config wrangler.god.json
```

- Deploys to `god.buildwithoracle.com`
- Useful for public federation visualization

---

## Configuration

### Environment Variables

Set during build or dev server startup:

| Variable | Default | Purpose |
|----------|---------|---------|
| `VITE_MAW_URL` | `http://localhost:3456` | Backend API base URL (HTTP or HTTPS) |

**Example:**
```bash
VITE_MAW_URL=https://white.local:3456 npm run dev
```

### Runtime Host Configuration

Maw-ui resolves the backend host at **runtime** via:

1. **URL Query Parameter** (`?host=`): Takes precedence
   ```
   http://localhost:5173/?host=white.local:3456
   http://localhost:5173/?host=https://192.168.1.100:3456
   http://localhost:5173/?host=http://oracle-world:3456
   ```
   - Auto-persists to `localStorage` and redirects to clean URL
   - Format: `bare-host:port` (defaults to HTTPS), or explicit `http://` / `https://`

2. **localStorage** (`maw-host` key): Persistent across page reloads
   - Read/write via `src/lib/api.ts` helpers:
     - `getStoredHost()` — retrieve saved host
     - `setStoredHost(host)` — save host
     - `clearStoredHost()` — reset to local

3. **Same-Origin (Local)**: If no host parameter, assumes maw-js is on the same origin

**Recent Hosts Tracking:**
- Dashboard maintains up to 8 recent hosts in `maw-host-recent` (localStorage)
- UI can display host switcher with history

### Host Format Reference

| Format | Example | Result |
|--------|---------|--------|
| Bare `host:port` | `white.local:3456` | `https://white.local:3456` (backwards-compatible default) |
| Explicit HTTPS | `https://white.local:3456` | `https://white.local:3456` |
| Explicit HTTP | `http://oracle-world:3456` | `http://oracle-world:3456` (needed for plain-HTTP LAN nodes without mkcert) |

**Note:** Chrome 142+ requires `targetAddressSpace: 'local'` when accessing private networks from HTTPS contexts. The dashboard auto-detects private hosts (localhost, 127.0.0.1, .local, 10.x, 192.168.x, 172.16-31.x) and includes the permission header automatically.

---

## Views & Pages

Each view is a standalone Vite entry point (`.html` file) with its own route. Access via:
- **Local dev**: `http://localhost:5173/<page>.html` or root (`index.html`)
- **Deployed**: `http://instance:3456/<page>.html`
- **Hosted (Cloudflare)**: `https://god.buildwithoracle.com/<page>?host=<node>`

### Core Views

| Page | File | Purpose |
|------|------|---------|
| **Federation 2D** | `federation_2d.html` | Canvas force-graph of all nodes + agents; live message trails; deep ocean theme |
| **Federation 3D** | `federation.html` | Three.js immersive 3D view with bloom effects + particle physics |
| **Federation List** | Default route with `#federation` | Searchable oracle list grouped by node; peer latency, reachability indicators |
| **Office / Agent Grid** | `index.html` | Main view: agent cards arranged in rooms (tmux-like session groups); status, PTY terminal previews, WebSocket feed |
| **Fleet** | `fleet.html` | Fleet-wide session overview—all sessions across all nodes |
| **Dashboard** | `dashboard.html` | Metrics dashboard: agent counts, health, uptime, active task summary |
| **Terminal** | `terminal.html` | Full xterm.js terminal per agent—direct shell access |
| **Mission Control** | `mission.html` | Active tasks + progress tracking; mission board with status flags |
| **Chat** | `chat.html` | Cross-agent messaging interface—persistent chat logs |
| **Config** | `config.html` | Fleet configuration viewer—display maw.config.json and environment |
| **Inbox** | `inbox.html` | Oracle inbox—messages, handoff requests, alerts |
| **Workspace** | `workspace.html` | Multi-agent workspace—send commands to multiple agents, collaborative UI |
| **Arena** | `arena.html` | Specialized view (specific use case—see source) |
| **Overview** | `overview.html` | High-level metrics + quick links |
| **Talk** | `talk.html` | Agent communication panel |
| **TimeMachine** | `timemachine.html` | Historical replay / session playback |
| **Shrine** | `shrine.html` | Specialized view |

---

## Architecture

### State Management

- **Zustand stores** (`src/lib/store.ts`):
  - Fleet state: recent agent activity, slept targets, UI preferences (sort mode, grouping, density)
  - Preview cache: terminal snapshots for hover cards
  - Route persistence: current view + active agent
  - Task/board metadata: projects, tasks, activity logs

### Data Flow

- **WebSocket feed** from maw-js backend (`:3456`)
  - Endpoint: `/ws` (main feed) + `/ws/pty` (PTY streams)
  - Real-time agent status, terminal output, messages
  - No polling—purely event-driven

- **REST API** for configuration and historical data
  - `/api/config` — fetch fleet configuration
  - `/api/peer/exec` — signed command relay
  - `/api/proxy` — HTTP relay for peer-to-peer communication

### Peer Connectivity

The dashboard classifies peer connections via `src/lib/peerConnection.ts`:
- **Same-origin**: Relative fetch paths
- **Direct**: HTTPS to maw-js (mkcert trusted)
- **Mixed-content-blocked**: HTTPS page → HTTP backend (blocked; needs proxy)
- **Invalid**: DNS or firewall error

Error banners automatically generated via `src/lib/peerConnectionBanner.ts`.

### Build & Bundling

- **Vite multi-page** build with React + TypeScript
- **Tailwind CSS** v4.2.1 for styling
- **Rollup entry points**: 17 `.html` → separate chunks + shared vendor bundle
- **Version info** auto-injected:
  - `__MAW_VERSION__`: Package version
  - `__MAW_BUILD__`: ISO build timestamp (Bangkok time)
  - `__MAW_COMMIT__`: Git commit + dirty flag
  - `dist/version.json`: Full build metadata (commit, branch, builder, buildTime)

---

## Client Helpers & Libraries

Located in `src/lib/`:

| File | Exports | Purpose |
|------|---------|---------|
| **api.ts** | `apiUrl()`, `wsUrl()`, `apiFetch()`, `apiFetchJson()`, `getHttpHealth()`, `subscribeHttpHealth()` | Centralized URL resolution (host param + localStorage); circuit-breaker fetch wrapper with Chrome PNA support |
| **constants.ts** | `ROOM_COLORS`, `AGENT_ORDER`, `ORACLE_ICONS`, `agentColor()`, etc. | Configuration: room styling, agent display order, oracle emoji icons, grid layout dimensions |
| **types.ts** | `AgentState`, `Session`, `AskItem`, `BoardItem`, `ProjectTask`, etc. | TypeScript interfaces for all data models |
| **store.ts** | `FleetStore`, `useFleetStore()` | Zustand store: agent history, UI prefs, route state, board/task metadata |
| **peerConnection.ts** | `classifyPeerConnection()`, `PeerConnectionStatus` | Detect connectivity classification (same-origin / direct / mixed-content-blocked / invalid) |
| **peerConnectionBanner.ts** | `peerConnectionBannerText()` | Derive human-readable error message for connectivity issues |
| **peerExecClient.ts** | `PeerExecClient` | Browser client for `POST /api/peer/exec` (signed command relay to target peer) |
| **peerProxyClient.ts** | `PeerProxyClient` | Browser client for `POST /api/proxy` (REST relay for HTTP-LAN peers) |
| **feed.ts** | `FeedClient`, WebSocket wrapper | Real-time message handling for agent events, status updates |
| **sounds.ts** | `playSound()` | Audio notifications |
| **avatar.ts** | Avatar generation utilities | Deterministic agent avatars |
| **ansi.ts** | ANSI escape sequence parser | Terminal output coloring |
| **capturePoller.ts** | Poll-based snapshot capture | Fallback when WebSocket unavailable |
| **federation.ts** | Federation graph utilities | Node/agent layout algorithms |
| **visibility.ts** | Visibility state tracker | Optimize polling when tab not focused |

---

## Hooks

Located in `src/hooks/`:

| Hook | Purpose |
|------|---------|
| `useWebSocket(url)` | Subscribe to WebSocket feed; auto-reconnect |
| `useSessions()` | Fetch + subscribe to session list |
| `useFederationData()` | Fetch federation graph (nodes + agents + mesh topology) |
| `useFederationList()` | Fetch + subscribe to oracle list grouped by peer |
| `useHttpHealth()` | Subscribe to circuit-breaker health status |
| `useMqtt()` | MQTT pub/sub wrapper (if enabled) |
| `useDevice()` | Detect device type (mobile, tablet, desktop) |
| `useFileAttach()` | Handle file upload/drop |

---

## Components

Located in `src/components/`:

### Core Layout

- **AppShell.tsx** — Top-level wrapper, navigation, sidebar
- **ConnectPage.tsx** — Host connection UI (host picker, recent hosts, status)
- **ErrorBoundary.tsx** — React error boundary fallback
- **StatusBar.tsx** — Footer: version, build info, live FPS counter

### Views (Top-level per page)

- **DashboardView.tsx** — Dashboard metrics layout
- **FederationView.tsx** — Federation 3D canvas wrapper
- **TerminalView.tsx** — Full-page terminal
- **ChatView.tsx** — Chat interface
- **MissionControl.tsx** — Task board
- **LoopsView.tsx** — Agent loops/sessions
- **InboxView.tsx** — Message inbox
- **ConfigView.tsx** — Config display
- **BoardView.tsx**, **ProjectBoardView.tsx** — Project board views
- **OverviewGrid.tsx** — Metrics grid

### Federation Visualization

- **federation/Canvas2D.tsx** — Canvas force-graph (2D)
- **federation/simulation.ts** — Physics engine (force-directed layout)
- **federation/draw.ts** — Rendering pipeline (nodes, edges, labels, trails)
- **federation/colors.ts** — Color constants + schemes
- **federation/store.ts** — Federation state (Zustand)
- **federation/Sidebar.tsx** — Controls + legend
- **federation/PluginPanel.tsx** — Plugin browser

### Agent & Fleet

- **AgentCard.tsx** — Agent info card (name, status, room, PTY preview)
- **AgentRow.tsx** — Tabular agent row (compact view)
- **AgentAvatar.tsx** — Avatar badge (name + icon)
- **FleetGrid.tsx** — Grid layout for agents
- **RoomGrid.tsx** — Room-based agent grouping (office metaphor)
- **HoverPreviewCard.tsx** — Hover tooltip with terminal snapshot

### Terminal & Control

- **XTerminal.tsx** — xterm.js wrapper component
- **TerminalModal.tsx** — Modal terminal overlay
- **Joystick.tsx** — Virtual joystick (game-like control)
- **MiniMonitor.tsx** — Mini terminal preview
- **VSAgentPanel.tsx**, **VSView.tsx** — VS Code–like split view

### Chat & Messaging

- **chat/ChatBubble.tsx** — Single message bubble
- **chat/ThreadCard.tsx** — Conversation thread
- **chat/ChibiAvatar.tsx** — Stylized agent avatar in chat
- **chat/useChatLog.ts** — Chat history hook

### Specialized

- **ChibiPortrait.tsx** — Stylized oracle portrait
- **BoBFaceView.tsx** — Custom face view
- **KVTable.tsx** — Key-value table display
- **LoadingSkeleton.tsx** — Skeleton loader
- **JumpOverlay.tsx** — Quick agent jump dialog
- **ShortcutOverlay.tsx** — Keyboard shortcut help
- **TaskDetailOverlay.tsx** — Task detail modal
- **MiniPreview.tsx** — Inline preview card

---

## Development

### Scripts

```bash
npm run dev      # Start Vite dev server (:5173) with HMR
npm run build    # Build dist/ (multi-page)
npm run preview  # Preview production build locally
```

### TypeScript Configuration

- **Target**: ES2022 (modern JavaScript)
- **Module**: ESNext
- **JSX**: React 19
- **Strict mode**: Enabled
- **Module resolution**: Bundler

### Tailwind CSS

- Version 4.2.1 (via `@tailwindcss/vite`)
- No config file needed; uses defaults + Vite integration
- Imported in component stylesheets

### Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| react | 19.0.0 | UI framework |
| react-dom | 19.0.0 | React DOM rendering |
| zustand | 5.0.11 | State management |
| @xterm/xterm | 5.5.0 | Terminal emulator |
| @xterm/addon-fit | 0.10.0 | Xterm auto-fit |
| three | 0.183.2 | 3D graphics (federation 3D view) |
| @monaco-editor/react | 4.7.0 | Code editor (config view) |
| mqtt | 5.15.1 | MQTT client (optional) |

### Build Tooling

- **Vite 6.0** — Build system + dev server
- **@vitejs/plugin-react** — Fast Refresh + SWC JSX
- **TypeScript 5.7** — Type checking
- **Tailwind CSS 4.2** — Utility styles

---

## Continuous Integration

### GitHub Actions

**Workflow**: `.github/workflows/build.yml`

- **Trigger**: Every PR, push to `main` / `alpha`
- **Build**: Runs `npm run build`
- **Release**: Auto-creates GitHub Release with `maw-ui-dist.tar.gz` on `v*` tag push

---

## Common Tasks

### Connect to a Remote maw-js Node

**Scenario**: Local dev, remote maw-js at `white.local:3456`

```bash
# Option 1: Query parameter
http://localhost:5173/?host=white.local:3456

# Option 2: localStorage (persists)
# In browser console:
localStorage.setItem('maw-host', 'white.local:3456');
location.reload();
```

### Switch Between HTTP and HTTPS

**Scenario**: Fallback to HTTP for LAN node without mkcert

```bash
# HTTPS (default for bare host:port)
?host=white.local:3456

# HTTP (explicit)
?host=http://oracle-world:3456
```

### Debug API Communication

**Check circuit-breaker health:**
```javascript
// In browser console
import { getHttpHealth } from './src/lib/api.ts';
console.log(getHttpHealth());
```

**Subscribe to health changes:**
```javascript
import { subscribeHttpHealth } from './src/lib/api.ts';
subscribeHttpHealth(() => console.log('Health changed'));
```

**View recent hosts:**
```javascript
localStorage.getItem('maw-host-recent'); // [recent hosts as JSON]
```

### Add a New Page

1. Create new `.html` entry point (e.g., `mypage.html`)
2. Add import in `src/apps/` (e.g., `mypage.tsx`)
3. Add entry to `vite.config.ts` `rollupOptions.input`
4. Access at `http://localhost:5173/mypage.html`

### Customize Room Colors & Agent Icons

Edit `src/lib/constants.ts`:
```typescript
export const ROOM_COLORS = { /* ... */ };
export const ORACLE_ICONS = { /* ... */ };
export const AGENT_COLORS = [ /* ... */ ];
```

---

## Deployment

### Self-Hosted (Behind maw-js)

```bash
maw ui --install
```

- Downloads pre-built dist from GitHub
- Served by maw-js on `:3456`
- No additional configuration

### Cloudflare Workers (Public Hosted)

```bash
npx wrangler deploy --config wrangler.god.json
```

- Available at `god.buildwithoracle.com`
- Access remote nodes via `?host=<node-address>`

### Docker / Kubernetes

1. Build: `npm run build`
2. Serve `dist/` with any static file server
3. Configure backend URL via:
   - Environment variable: `VITE_MAW_URL` (build-time)
   - Runtime: `?host=` query parameter or localStorage

---

## Troubleshooting

### "Circuit open" / Connection Failures

- Check maw-js is running on configured port (`:3456` default)
- Verify host parameter: `?host=<actual-ip-or-hostname>`
- Check Chrome PNA permission (for HTTPS → local network)
- View circuit status: `getHttpHealth()` in console

### Mixed-Content Errors (HTTPS Page → HTTP Backend)

- Use `?host=http://node:3456` (explicit HTTP)
- Or deploy to HTTP origin
- maw-js proxy (`/api/proxy`) can relay HTTP requests from HTTPS context

### Terminal Not Appearing

- Ensure `/ws/pty` endpoint is accessible on maw-js
- Check WebSocket upgrade is not blocked by proxy
- Verify agent is in "ready" state (not crashed or idle)

### Stale Configuration

- Clear localStorage: `localStorage.clear()`
- Reset host: `localStorage.removeItem('maw-host')`
- Restart dev server: `Ctrl+C`, then `npm run dev`

---

## License

[BUSL-1.1](LICENSE) — Business Source License 1.1  
**Copyright**: Nat Weerawan (ณัฐ วีระวรรณ์)

---

## Links

- **Repository**: https://github.com/Soul-Brews-Studio/maw-ui
- **Backend (maw-js)**: https://github.com/Soul-Brews-Studio/maw-js
- **Public Instance**: https://god.buildwithoracle.com
- **Releases**: https://github.com/Soul-Brews-Studio/maw-ui/releases
