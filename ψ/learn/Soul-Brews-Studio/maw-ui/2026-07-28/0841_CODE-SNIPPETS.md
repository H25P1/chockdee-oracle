# maw-ui: Code Snippets Documentation

**Project**: maw-ui — a web dashboard for maw-js fleet control (TypeScript + React + Vite)

**Date**: 2026-07-28

This document collects representative code samples showing fleet/session state management, API client architecture, and real-time update mechanisms in maw-ui.

---

## 1. Core Data Types

### Session and Agent State

`src/lib/types.ts` defines the core domain models:

```typescript
export interface Window {
  index: number;
  name: string;
  active: boolean;
  cwd?: string;
}

export interface Session {
  name: string;
  windows: Window[];
  source?: string;  // peer URL or "local"
}

export type PaneStatus = "ready" | "busy" | "idle" | "crashed";

export interface AgentState {
  target: string;
  name: string;
  session: string;
  windowIndex: number;
  active: boolean;
  preview: string;
  status: PaneStatus;
  project?: string;
  cwd?: string;
  source?: string;  // peer URL for federated agents, undefined = local
}

export interface AgentEvent {
  time: number;
  target: string;
  type: "status" | "command";
  detail: string;
}

export type AskType = "input" | "attention" | "plan" | "report" | "meeting" | "handoff";

export interface AskItem {
  id: string;
  oracle: string;
  target: string;      // tmux target e.g. "01-oracles:0"
  type: AskType;
  message: string;
  ts: number;
  dismissed?: boolean;
}
```

---

## 2. API Client: Circuit Breaker Pattern

`src/lib/api.ts` implements centralized API host resolution with a circuit-breaker wrapper to prevent cascading failures when the maw-js host becomes unreachable.

### Host Resolution

Supports three URL forms:
- `?host=white.local:3456` → https://white.local:3456 (bare host:port, defaults to https)
- `?host=https://white.local:3456` → https://white.local:3456 (explicit https)
- `?host=http://oracle-world:3456` → http://oracle-world:3456 (plain HTTP for LAN-only nodes)

```typescript
const STORAGE_KEY = "maw-host";
const RECENT_KEY = "maw-host-recent";

const params = new URLSearchParams(window.location.search);
const urlHost = params.get("host");

// Auto-persist: ?host= in URL → save to localStorage → redirect clean
if (urlHost) {
  localStorage.setItem(STORAGE_KEY, urlHost);
  addRecentHost(urlHost);
  const url = new URL(window.location.href);
  url.searchParams.delete("host");
  window.location.replace(url.toString());
}

export const isRemote = !!localStorage.getItem(STORAGE_KEY);
export const activeHost: string | null = localStorage.getItem(STORAGE_KEY);

export function apiUrl(path: string): string {
  const r = resolveHost();
  if (!r) return path;
  return `${r.httpProto}//${r.host}${path}`;
}

export function wsUrl(path: string): string {
  const r = resolveHost();
  if (!r) {
    const proto = location.protocol === "https:" ? "wss:" : "ws:";
    return `${proto}//${location.host}${path}`;
  }
  return `${r.wsProto}//${r.host}${path}`;
}
```

### Circuit Breaker

Trips after 5 consecutive failures, holding circuit open for 30 seconds with health observable:

```typescript
const FAIL_THRESHOLD = 5;
const OPEN_MS = 30_000;

type Health = {
  healthy: boolean;
  consecutiveFails: number;
  openUntil: number;         // timestamp; 0 when closed
  lastError: string | null;
};

let healthSnapshot: Health = { healthy: true, consecutiveFails: 0, openUntil: 0, lastError: null };
const listeners = new Set<() => void>();

export function getHttpHealth(): Health {
  return healthSnapshot;
}

export function subscribeHttpHealth(cb: () => void): () => void {
  listeners.add(cb);
  return () => { listeners.delete(cb); };
}

export async function apiFetch(path: string, init?: RequestInit): Promise<Response> {
  const url = path.startsWith("http") ? path : apiUrl(path);
  const now = Date.now();

  // Circuit open: allow exactly one probe per OPEN_MS; reject the rest.
  if (!healthSnapshot.healthy && now < healthSnapshot.openUntil) {
    throw new Error("circuit_open");
  }

  // Chrome PNA: opt the request into the local-network address space
  const finalInit: RequestInit & { targetAddressSpace?: "loopback" | "local" | "private" } = { ...init };
  if (isPrivateHost()) {
    const r = resolveHost();
    const h = r?.host.split(":")[0].toLowerCase() ?? "";
    finalInit.targetAddressSpace =
      h === "localhost" || h === "127.0.0.1" ? "loopback" : "local";
  }

  try {
    const res = await fetch(url, finalInit);
    if (res.status >= 500) throw new Error(`http_${res.status}`);
    if (!healthSnapshot.healthy || healthSnapshot.consecutiveFails > 0) {
      commit({ healthy: true, consecutiveFails: 0, openUntil: 0, lastError: null });
    }
    return res;
  } catch (err) {
    const fails = healthSnapshot.consecutiveFails + 1;
    const msg = err instanceof Error ? err.message : String(err);
    if (fails >= FAIL_THRESHOLD) {
      commit({ healthy: false, consecutiveFails: fails, openUntil: now + OPEN_MS, lastError: msg });
    } else {
      commit({ consecutiveFails: fails, lastError: msg });
    }
    throw err;
  }
}

export async function apiFetchJson<T = any>(path: string, init?: RequestInit): Promise<T | null> {
  try {
    const r = await apiFetch(path, init);
    if (!r.ok) return null;
    return await r.json() as T;
  } catch { return null; }
}
```

---

## 3. Real-Time Updates: WebSocket

`src/hooks/useWebSocket.ts` implements auto-reconnecting WebSocket with exponential backoff.

```typescript
import { useEffect, useRef, useState, useCallback } from "react";
import { wsUrl } from "../lib/api";

type MessageHandler = (data: any) => void;

const BASE_DELAY = 1000;
const MAX_DELAY = 30000;

export function useWebSocket(onMessage: MessageHandler) {
  const wsRef = useRef<WebSocket | null>(null);
  const [connected, setConnected] = useState(false);
  const [reconnecting, setReconnecting] = useState(false);
  const onMessageRef = useRef(onMessage);
  onMessageRef.current = onMessage;

  useEffect(() => {
    let alive = true;
    let reconnectTimer: ReturnType<typeof setTimeout>;
    let attempt = 0;

    function connect() {
      if (!alive) return;
      const ws = new WebSocket(wsUrl("/ws"));
      wsRef.current = ws;

      ws.onopen = () => {
        attempt = 0;
        setConnected(true);
        setReconnecting(false);
      };
      ws.onmessage = (e) => {
        try { onMessageRef.current(JSON.parse(e.data)); } catch {}
      };
      ws.onclose = () => {
        setConnected(false);
        if (alive) {
          setReconnecting(true);
          const delay = Math.min(BASE_DELAY * 2 ** attempt, MAX_DELAY);
          attempt++;
          reconnectTimer = setTimeout(connect, delay);
        }
      };
      ws.onerror = () => ws.close();
    }

    connect();
    return () => {
      alive = false;
      clearTimeout(reconnectTimer);
      wsRef.current?.close();
    };
  }, []);

  const send = useCallback((msg: object) => {
    const ws = wsRef.current;
    if (ws && ws.readyState === WebSocket.OPEN) {
      ws.send(JSON.stringify(msg));
    }
  }, []);

  return { connected, reconnecting, send };
}
```

---

## 4. Real-Time Updates: MQTT Bridge

`src/hooks/useMqtt.ts` connects to a Cloudflare Workers MQTT bridge for peer-to-peer messaging.

```typescript
import { useEffect, useRef, useState } from "react";
import mqtt from "mqtt";

const BROKER = "wss://maw-mqtt-bridge.laris.workers.dev/ws/mqtt";
const TOPIC = "maw/v1/hey/#";

export interface MqttMessage {
  from: string;
  to: string;
  timestamp: string;
  message: string;
  topic: string;
}

type MqttMessageHandler = (msg: MqttMessage) => void;

export function useMqtt(onMessage: MqttMessageHandler) {
  const [connected, setConnected] = useState(false);
  const onMessageRef = useRef(onMessage);
  onMessageRef.current = onMessage;

  useEffect(() => {
    const client = mqtt.connect(BROKER, {
      reconnectPeriod: 5000,
      connectTimeout: 10000,
    });

    client.on("connect", () => {
      console.log("[mqtt] connected to", BROKER);
      setConnected(true);
      client.subscribe(TOPIC, (err) => {
        if (err) console.error("[mqtt] subscribe error:", err);
        else console.log("[mqtt] subscribed to", TOPIC);
      });
    });

    client.on("close", () => setConnected(false));
    client.on("error", (err) => console.error("[mqtt] error:", err));

    client.on("message", (topic: string, payload: Buffer) => {
      try {
        const msg = JSON.parse(payload.toString());
        onMessageRef.current({
          from: msg.from || "",
          to: msg.to || "",
          timestamp: msg.timestamp || "",
          message: msg.message || "",
          topic,
        });
      } catch {}
    });

    return () => { client.end(); };
  }, []);

  return { connected };
}
```

---

## 5. State Management: Zustand Store

`src/lib/store.ts` manages fleet state with hybrid persistence (localStorage + server sync) and handles recent activity tracking.

### Fleet Store Interface

```typescript
export interface DispatchStatus {
  step: "routing" | "done" | "error";
  oracle?: string;
  oracleName?: string;
  target?: string;
  message?: string;
  error?: string;
  task?: string;
  ts: number;
}

interface FleetStore {
  // Recently active: target → agent metadata + timestamp
  recentMap: Record<string, RecentEntry>;
  markBusy: (agents: { target: string; name: string; session: string }[], at?: number) => void;
  pruneRecent: () => void;

  // Slept agents (Ctrl+C'd from UI — grey + collapsed until wake/busy)
  sleptTargets: string[];
  markSlept: (target: string) => void;
  clearSlept: (target: string) => void;

  // UI preferences
  sortMode: "active" | "name";
  setSortMode: (mode: "active" | "name") => void;
  grouped: boolean;
  toggleGrouped: () => void;
  density: "cozy" | "compact";
  toggleDensity: () => void;
  collapsed: string[];
  toggleCollapsed: (key: string) => void;
  muted: boolean;
  toggleMuted: () => void;
  stageMode: "stage" | "pitch";
  toggleStageMode: () => void;

  // Route persistence
  lastView: string;
  setLastView: (view: string) => void;

  // Dispatch log (BoB task routing)
  dispatchLog: DispatchStatus[];
  addDispatchStatus: (status: Omit<DispatchStatus, "ts">) => void;

  // Inbox asks
  asks: AskItem[];
  addAsk: (ask: Omit<AskItem, "id" | "ts">) => void;
  dismissAsk: (id: string) => void;
  dismissByOracle: (oracle: string) => void;

  // Board state (non-persisted)
  boardItems: BoardItem[];
  boardFields: BoardField[];
  boardLoading: boolean;
  boardFilter: string;
  boardSubView: "board" | "timeline" | "scan" | "activity" | "pulse" | "projects";
  // ... setters for each
}
```

### Hybrid Storage: localStorage → Server Sync

Writes to localStorage immediately (instant on next refresh), then debounces server POST with smart diffing:

```typescript
// --- Hybrid storage: localStorage for instant hydration + server for cross-device sync ---

let writeTimer: ReturnType<typeof setTimeout> | null = null;
let pendingWrite: string | null = null;
let lastServerWrite: string | null = null;

function flushWrite() {
  if (pendingWrite === null) return;
  const body = pendingWrite;
  pendingWrite = null;
  apiFetch(`/api/ui-state`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body,
  }).catch(() => {}); // fire-and-forget
}

let lastSyncTime = 0;
const SYNC_INTERVAL = 5_000;

function syncFromServer(name: string) {
  // Hidden tabs don't need cross-device updates — resync on return
  if (typeof document !== "undefined" && document.hidden) return;
  const now = Date.now();
  if (now - lastSyncTime < SYNC_INTERVAL) return;
  lastSyncTime = now;
  apiFetch("/api/ui-state").then(async (res) => {
    if (!res.ok) return;
    const data = await res.json();
    if (!data || Object.keys(data).length === 0) return;
    const existing = localStorage.getItem(name);
    const parsed = existing ? (() => { try { return JSON.parse(existing); } catch { return null; } })() : null;
    // recentMap is localStorage-only (never POSTed) — compare and merge
    const { recentMap: localRecent, ...localState } = parsed?.state ?? {};
    const { recentMap: _serverRecent, ...serverState } = data;
    if (JSON.stringify(serverState) !== JSON.stringify(localState)) {
      const ver = parsed?.version ?? 3;
      localStorage.setItem(name, JSON.stringify({ state: { ...serverState, recentMap: localRecent }, version: ver }));
      useFleetStore.persist.rehydrate();
    }
  }).catch(() => {});
}

const hybridStorage: StateStorage = {
  getItem: (name) => {
    // Return localStorage synchronously → instant hydration
    // Then background-sync from server (debounced)
    if (!syncScheduled) {
      syncScheduled = true;
      setTimeout(() => { syncScheduled = false; syncFromServer(name); }, 0);
    }
    return localStorage.getItem(name);
  },
  setItem: (name, value) => {
    // Write to localStorage immediately
    localStorage.setItem(name, value);
    // Debounced write to server. recentMap churns on every feed event,
    // so we compare/send the state WITHOUT recentMap and skip the POST
    // when nothing else changed. recentMap stays localStorage-only.
    try {
      const { state } = JSON.parse(value);
      const { recentMap: _recentMap, ...serverState } = state;
      const body = JSON.stringify(serverState);
      if (body === lastServerWrite) return;
      lastServerWrite = body;
      pendingWrite = body;
      if (writeTimer) clearTimeout(writeTimer);
      writeTimer = setTimeout(flushWrite, 1000);
    } catch {}
  },
  removeItem: (name) => {
    localStorage.removeItem(name);
    apiFetch(`/api/ui-state`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: "{}",
    }).catch(() => {});
  },
};
```

---

## 6. Session & Feed Event Handling

`src/hooks/useSessions.ts` orchestrates WebSocket messages into sessions, agents, and feed events. It implements status decay (busy → ready → idle) and ask detection.

### Message Handling

Processes WebSocket messages for sessions, recent agents, feed events, and team updates:

```typescript
const handleMessage = useCallback((data: any) => {
  if (data.type === "sessions") {
    const next = (data.sessions as Session[]).filter(s => !s.name.startsWith("maw-pty-"));
    // Server pushes sessions every 2s whether or not anything changed.
    // Skip identical pushes — a fresh array re-renders every session-consuming view.
    const json = JSON.stringify(next);
    if (json !== sessionsJsonRef.current) {
      sessionsJsonRef.current = json;
      setSessions(next);
    }
  } else if (data.type === "recent") {
    const agents: { target: string; name: string; session: string }[] = data.agents || [];
    if (agents.length > 0) {
      markBusy(agents);
      // Set initial "ready" status for agents detected as running Claude.
      // Without this they'd render as "idle" for up to BUSY_TIMEOUT seconds
      // before the first feed event bumps them to "ready".
      const store = useFeedStatusStore.getState();
      for (const a of agents) {
        const current = store.statuses[a.target];
        if (current !== "busy") feedLastSeen.current[a.target] = Date.now();
        if (!current || current === "idle") {
          store.setStatus(a.target, "ready");
        }
      }
    }
  } else if (data.type === "feed") {
    const feedEvent = data.event as FeedEvent;
    setFeedEvents(prev => {
      const next = [...prev, feedEvent];
      return next.length > MAX_FEED ? next.slice(-MAX_FEED) : next;
    });
    updateStatusFromFeed(feedEvent);
    detectAsk(feedEvent);
  } else if (data.type === "feed-history") {
    const events = (data.events as FeedEvent[]).slice(-MAX_FEED);
    setFeedEvents(events);
    for (const e of events) {
      updateStatusFromFeed(e);
      if (FEED_BUSY_EVENTS.has(e.event as FeedEventType)) {
        const agent = resolveAgentFromFeed(e);
        if (agent) markBusy([{ target: agent.target, name: agent.name, session: agent.session }], e.ts);
      }
    }
  } else if (data.type === "teams") {
    setTeams(data.teams || []);
  } else if (data.type === "previews") {
    const rawPreviews: Record<string, string> = data.data;
    const cleaned: Record<string, string> = {};
    for (const [target, raw] of Object.entries(rawPreviews)) {
      const text = stripAnsi(raw);
      const lines = text.split("\n").filter((l: string) => l.trim());
      const compactingLine = lines.find((l: string) => l.toLowerCase().includes("compacting"));
      cleaned[target] = (compactingLine || lines[lines.length - 1] || "").slice(0, 120);
    }
    usePreviewStore.getState().setPreviews(cleaned);
  } else if (data.type === "action-ok") {
    if (data.action === "sleep") markSlept(data.target);
    else if (data.action === "wake") clearSlept(data.target);
  }
}, []);
```

### Status Decay

Busy → ready after 15s without feed, ready → idle after 60s:

```typescript
const BUSY_TIMEOUT = 15_000;  // 15s without feed → ready
const IDLE_TIMEOUT = 60_000;  // 60s without feed → idle

useEffect(() => {
  const interval = setInterval(() => {
    const now = Date.now();
    const { statuses, setStatus } = useFeedStatusStore.getState();
    for (const agent of agentsRef.current) {
      const lastSeen = feedLastSeen.current[agent.target] || 0;
      const status = statuses[agent.target];
      if (!status) continue;

      const lastEvt = feedLastEvent.current[agent.target];
      const inToolCall = lastEvt === "PreToolUse" || lastEvt === "SubagentStart";
      if (status === "busy" && lastSeen > 0 && now - lastSeen > BUSY_TIMEOUT && !inToolCall) {
        setStatus(agent.target, "ready");
      } else if (status === "ready" && (lastSeen === 0 || now - lastSeen > IDLE_TIMEOUT)) {
        setStatus(agent.target, "idle");
      }
    }
  }, 5000);
  return () => clearInterval(interval);
}, []);
```

### Agent Derivation

Agents are derived from sessions + statuses (memoized to prevent re-renders):

```typescript
const agents: AgentState[] = useMemo(() => {
  const list = sessions.flatMap((s) =>
    s.windows.map((w) => {
      const key = `${s.name}:${w.index}`;
      let project: string | undefined;
      if (w.cwd) {
        const base = w.cwd.split("/").pop() || "";
        const wtMatch = base.match(/[.-]wt-(?:\d+-)?(.+)$/);
        project = wtMatch ? `wt:${wtMatch[1]}` : base;
      }
      return {
        target: key,
        name: w.name,
        session: s.name,
        windowIndex: w.index,
        active: w.active,
        preview: "",
        status: statuses[key] || "idle",
        project,
        cwd: w.cwd,
        source: s.source && s.source !== "local" ? s.source : undefined,
      };
    })
  );
  list.sort((a, b) => agentSortKey(a.name) - agentSortKey(b.name));
  agentsRef.current = list;
  return list;
}, [sessions, statuses]);
```

---

## 7. Fleet Grid Component

`src/components/FleetGrid.tsx` is the primary fleet view component. It displays sessions as collapsible rooms with agent rows, tracks recently active agents, and supports inline previews and broadcasts.

### FleetGrid Props & Controls

```typescript
interface FleetGridProps {
  sessions: Session[];
  agents: AgentState[];
  connected: boolean;
  send: (msg: object) => void;
  onSelectAgent: (agent: AgentState) => void;
  eventLog: AgentEvent[];
  addEvent: (target: string, type: AgentEvent["type"], detail: string) => void;
  feedActive?: Map<string, FeedEvent>;
  agentFeedLog?: Map<string, FeedEvent[]>;
  teams?: Team[];
}

export function FleetControls({ agents, send }: { agents: AgentState[]; send: (msg: object) => void }) {
  const { sortMode, setSortMode } = useFleetStore();
  const busyCount = agents.filter(a => a.status === "busy").length;
  const readyCount = agents.filter(a => a.status === "ready").length;
  const idleCount = agents.length - busyCount - readyCount;

  const wakeAll = () => {
    for (const a of agents) {
      if (a.status === "idle") send({ type: "wake", target: a.target, command: guessCommand(a.name) });
    }
  };
  const sleepAll = () => {
    if (!confirm("Sleep all busy agents?")) return;
    for (const a of agents) {
      if (a.status === "busy") send({ type: "sleep", target: a.target });
    }
  };

  return (
    <>
      {busyCount > 0 && (
        <span className="flex items-center gap-1.5 text-xs font-mono whitespace-nowrap">
          <span className="w-1.5 h-1.5 rounded-full bg-amber-400 shadow-[0_0_6px_#ffa726] animate-pulse" />
          <span className="text-amber-400">{busyCount}</span>
        </span>
      )}
      <span className="flex items-center gap-1.5 text-xs font-mono whitespace-nowrap">
        <span className="w-1.5 h-1.5 rounded-full bg-emerald-400" />
        <span className="text-emerald-400">{readyCount}</span>
      </span>
      {idleCount > 0 && (
        <span className="flex items-center gap-1.5 text-xs font-mono whitespace-nowrap">
          <span className="w-1.5 h-1.5 rounded-full bg-white/20" />
          <span className="text-white/30">{idleCount}</span>
        </span>
      )}
      {idleCount > 0 && (
        <button className="px-2 py-1 text-[10px] font-mono font-bold rounded-md active:scale-95 transition-all whitespace-nowrap"
          style={{ background: "rgba(34,197,94,0.15)", color: "#22c55e" }}
          onClick={wakeAll} title="Wake all idle agents">Wake</button>
      )}
      {busyCount > 0 && (
        <button className="px-2 py-1 text-[10px] font-mono font-bold rounded-md active:scale-95 transition-all whitespace-nowrap"
          style={{ background: "rgba(251,191,36,0.1)", color: "#fbbf24" }}
          onClick={sleepAll} title="Sleep all busy agents">Sleep</button>
      )}
    </>
  );
}
```

### Room Grouping & Sorting

FleetGrid tracks sessions and agents by room, supports collapsible sections, and caches agent feed logs:

```typescript
function sortRooms(sessions: Session[], agentMap: Map<string, AgentState[]>, mode: "active" | "name") {
  return [...sessions].sort((a, b) => {
    if (mode === "active") {
      const aBusy = (agentMap.get(a.name) || []).filter(ag => ag.status === "busy").length;
      const bBusy = (agentMap.get(b.name) || []).filter(ag => ag.status === "busy").length;
      if (aBusy !== bBusy) return bBusy - aBusy;
      const aLen = (agentMap.get(a.name) || []).length;
      const bLen = (agentMap.get(b.name) || []).length;
      if (aLen !== bLen) return bLen - aLen;
    }
    return a.name.localeCompare(b.name);
  });
}

// Recently active: busy agents first, then recently-gone from store
// Deduplicated by agent name (same agent may have multiple tmux windows)
const recentlyActive = useMemo((): (AgentState | RecentEntry)[] => {
  const agentMap = new Map(agents.map(a => [a.target, a]));
  const busyTargets = new Set(busyAgents.map(a => a.target));

  // Dedup busy agents by name — keep first
  const seenNames = new Set<string>();
  const dedupBusy = busyAgents.filter(a => {
    if (seenNames.has(a.name)) return false;
    seenNames.add(a.name);
    return true;
  });

  // Recently-gone: in store but not currently busy, dedup by name (keep most recent)
  const recentByName = new Map<string, RecentEntry>();
  for (const e of Object.values(recentMap)) {
    if (busyTargets.has(e.target)) continue;
    const prev = recentByName.get(e.name);
    if (!prev || e.lastBusy > prev.lastBusy) recentByName.set(e.name, e);
  }
  const recentGone = [...recentByName.values()]
    .filter(e => !seenNames.has(e.name))
    .sort((a, b) => b.lastBusy - a.lastBusy)
    .slice(0, 5)
    .map(e => agentMap.get(e.target) || agents.find(a => a.name === e.name) || e);

  // Active first, then recently-gone
  return [...dedupBusy, ...recentGone];
}, [agents, busyAgents, recentMap]);
```

### Intersection Observer for Visible Targets

Optimization: only stream preview data for visible agents. Releases on tab hide, re-subscribes on return:

```typescript
function useVisibleTargets(send: (msg: object) => void) {
  const visibleRef = useRef(new Set<string>());
  const observerRef = useRef<IntersectionObserver | null>(null);
  const debounceRef = useRef<ReturnType<typeof setTimeout> | undefined>(undefined);

  const syncToServer = useCallback(() => {
    clearTimeout(debounceRef.current);
    debounceRef.current = setTimeout(() => {
      send({ type: "subscribe-previews", targets: [...visibleRef.current] });
    }, 150);
  }, [send]);

  useEffect(() => {
    observerRef.current = new IntersectionObserver(
      (entries) => {
        let changed = false;
        for (const entry of entries) {
          const target = (entry.target as HTMLElement).dataset.target;
          if (!target) continue;
          if (entry.isIntersecting) {
            if (!visibleRef.current.has(target)) { visibleRef.current.add(target); changed = true; }
          } else {
            if (visibleRef.current.has(target)) { visibleRef.current.delete(target); changed = true; }
          }
        }
        if (changed) syncToServer();
      },
      { rootMargin: "100px" }
    );
    // Hidden tabs: release streams, re-subscribe on return
    const onVis = () => {
      if (document.hidden) {
        clearTimeout(debounceRef.current);
        send({ type: "subscribe-previews", targets: [] });
      } else {
        syncToServer();
      }
    };
    document.addEventListener("visibilitychange", onVis);
    return () => {
      observerRef.current?.disconnect();
      clearTimeout(debounceRef.current);
      document.removeEventListener("visibilitychange", onVis);
    };
  }, [syncToServer, send]);

  const observe = useCallback((el: HTMLElement | null, target: string) => {
    if (!el || !observerRef.current) return;
    el.dataset.target = target;
    observerRef.current.observe(el);
  }, []);

  return observe;
}
```

---

## 8. Peer Status Panel Component

`src/components/PeerStatusPanel.tsx` displays federation peer reachability and latency for a local node.

```typescript
import type { PeerStatus } from "../lib/federation";
import { nodeColor } from "../lib/federation";

interface PeerStatusPanelProps {
  localNode: string;
  peers: PeerStatus[];
}

/** Compact panel showing federation peer reachability + latency. */
export default function PeerStatusPanel({ localNode, peers }: PeerStatusPanelProps) {
  if (peers.length === 0) return null;

  return (
    <div className="rounded-lg border border-white/[0.08] bg-white/[0.03] p-3">
      <div className="flex items-center gap-2 mb-2">
        <span className="text-xs font-medium text-white/60">Federation Peers</span>
        <span className="text-[10px] text-white/30">from {localNode}</span>
      </div>
      <div className="space-y-1.5">
        {peers.map((peer) => {
          const { accent } = nodeColor(peer.name);
          return (
            <div key={peer.name} className="flex items-center justify-between gap-2">
              <div className="flex items-center gap-2 min-w-0">
                <span
                  className="w-2 h-2 rounded-full flex-shrink-0"
                  style={{ backgroundColor: peer.reachable ? "#22c55e" : "#ef4444" }}
                />
                <span className="text-sm truncate" style={{ color: accent }}>
                  {peer.name}
                </span>
              </div>
              <span className="text-[10px] text-white/40 flex-shrink-0">
                {peer.reachable
                  ? `${peer.latencyMs ?? "?"}ms`
                  : "unreachable"}
              </span>
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

---

## 9. Mission Control Hook

`src/components/useMissionControl.ts` powers the orbital mission control view. It handles agent positioning in a circular layout, pinned preview cards, multi-card mode, and zoom/pan gestures.

### Key Features

- **Circular Layout**: Agents positioned in circles around room nodes
- **Zoom & Pan**: Manual pan with mouse or Shift+drag, auto-zoom for portrait
- **Pinned Cards**: Click agent to pin a large preview card
- **Multi-Card Mode**: Toggle between single pinned card and multi-card view
- **SVG Coordinate Mapping**: Transforms SVG positions to screen-space for card placement

```typescript
// Simplified excerpt showing layout computation
const layout = useMemo(() => {
  const sessionList = sessions.map((s) => ({
    session: { name: s.name, windows: s.windows.map((w) => w.name) },
    agents: sessionAgents.get(s.name) || [],
    style: roomStyle(s.name),
  }));

  // Optionally merge solo rooms into "Oracles" cluster
  let virtual: LayoutItem[];
  if (groupSolo) {
    const multi = sessionList.filter(s => s.agents.length > 1);
    const soloAgents = sessionList.filter(s => s.agents.length === 1).flatMap(s => s.agents);
    virtual = [];
    if (soloAgents.length > 0) {
      virtual.push({
        session: { name: "_oracles", windows: [] },
        agents: soloAgents,
        style: { accent: "#7e57c2", floor: "#1a1428", wall: "#120e1e", label: "Oracles" },
      });
    }
    virtual.push(...multi);
  } else {
    virtual = sessionList;
  }

  const cx = 640, cy = 460;
  const radius = Math.min(370, 160 + virtual.length * 28);

  return virtual.map((s, i) => {
    const angle = (i / virtual.length) * Math.PI * 2 - Math.PI / 2;
    const x = cx + Math.cos(angle) * radius;
    const y = cy + Math.sin(angle) * radius;
    return { ...s, x, y };
  });
}, [sessions, sessionAgents, groupSolo]);

// Compute agent positions within each room
const agentPositions = useMemo(() => {
  const map = new Map<string, { svgX: number; svgY: number; style: ReturnType<typeof roomStyle> }>();
  for (const s of layout) {
    const count = s.agents.length;
    s.agents.forEach((agent, ai) => {
      const angle = (ai / Math.max(1, count)) * Math.PI * 2 - Math.PI / 2;
      const r = count === 1 ? 0 : Math.min(Math.max(70, 35 + count * 18) - 35, 35 + count * 6);
      map.set(agent.target, {
        svgX: s.x + Math.cos(angle) * r,
        svgY: s.y + Math.sin(angle) * r,
        style: s.style,
      });
    });
  }
  return map;
}, [layout]);

// Compute viewBox based on zoom and pan
const isPortrait = typeof window !== "undefined" && window.innerHeight > window.innerWidth;
const baseH = isPortrait ? 650 : 1000;
const vbW = 1200 / zoom;
const vbH = baseH / zoom;
const vbX = (1200 - vbW) / 2 - pan.x;
const vbY = (baseH - vbH) / 2 - pan.y + (isPortrait ? 250 : 0);
```

---

## Summary: Data Flow Architecture

1. **WebSocket Connection** (`useWebSocket`) connects to `/ws` at the maw-js server
2. **Session Feed** pushes `{type: "sessions", sessions: Session[]}` every 2s
3. **Recent Agents** pushes `{type: "recent", agents: [{target, name, session}, ...]}` on activity
4. **Feed Events** stream `{type: "feed", event: FeedEvent}` for status updates
5. **useSessions Hook** processes all messages:
   - Aggregates sessions into agents (derived state)
   - Maps feed events to agents (worktree-aware)
   - Applies status decay (busy → ready → idle)
   - Stores state in Zustand (persisted + synced to server)
6. **Component Trees** (FleetGrid, MissionControl) consume agents + sessions
7. **Preview Streaming** via IntersectionObserver → `subscribe-previews` message → server streams latest terminal output

All state is observable and updates propagate reactively via React hooks + Zustand subscriptions.

---

## File Paths

- API client: `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-ui/src/lib/api.ts`
- Store: `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-ui/src/lib/store.ts`
- Types: `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-ui/src/lib/types.ts`
- WebSocket hook: `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-ui/src/hooks/useWebSocket.ts`
- MQTT hook: `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-ui/src/hooks/useMqtt.ts`
- Sessions hook: `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-ui/src/hooks/useSessions.ts`
- FleetGrid component: `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-ui/src/components/FleetGrid.tsx`
- PeerStatusPanel component: `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-ui/src/components/PeerStatusPanel.tsx`
- Mission Control hook: `/Users/h_wa/ghq/github.com/Soul-Brews-Studio/maw-ui/src/components/useMissionControl.ts`
