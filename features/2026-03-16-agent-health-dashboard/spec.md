# Agent Health Dashboard — Design Spec

## Overview

Standalone одностраничное веб-приложение для визуального мониторинга агентов Империи в реальном времени. Терминал-стиль, тёмная тема, polling каждые 10 секунд. Два файла: `server.js` (Bun) + `index.html` (vanilla).

**Связь с другими brainstorm-ами:**
- **#1 Agent Health Monitor** — поставляет данные (JSON-файлы: agent-health.json, agent-health-extended.json, agent-incidents.json)
- **#2 Agent Control Panel** — поставляет agent-modes.json (текущие режимы)
- **Этот spec** — визуальный фронтенд, read-only, не управляет агентами

**Агенты:**
- **main** (Сократ) — gateway `main` (port 18789), 🔒 always active
- **archimedes** (Архимед) — gateway `main`
- **aristotle** (Аристотель) — gateway `main`
- **herodotus** (Геродот) — gateway `main`
- **platon** (Платон) — Docker gateway (port 18795)

---

## 1. Архитектура

```
┌──────────────────────────────────────────────────────┐
│  Cloudflare Tunnel                                    │
│    → localhost:3456 (Bun server)                      │
│    → Bearer token auth                                │
│                                                        │
│  server.js (Bun, ~80 lines):                          │
│    GET /           → index.html (static)              │
│    GET /api/health → merged JSON response             │
│      ├─ reads agent-health.json       (watchdog data) │
│      ├─ reads agent-health-extended.json (skill data) │
│      ├─ reads agent-incidents.json    (incident log)  │
│      ├─ reads agent-modes.json        (control panel) │
│      ├─ collects host metrics         (os module)     │
│      └─ collects docker stats         (Платон)        │
│                                                        │
│  index.html (vanilla HTML/CSS/JS, ~300 lines):        │
│    fetch('/api/health') every 10s                     │
│    render: host bar + agent cards + incidents          │
└──────────────────────────────────────────────────────┘
```

**Принципы:**
- Read-only dashboard — не управляет агентами
- Один API endpoint — фронт не знает про файлы, получает merged JSON
- Graceful degradation — если файл отсутствует, секция пустая
- Один экран — вся информация видна без скролла
- Zero dependencies на фронте — vanilla JS, CSS, HTML

---

## 2. Server (`server.js`)

### Endpoints

| Route | Method | Auth | Response |
|-------|--------|------|----------|
| `/` | GET | No | `index.html` |
| `/api/health` | GET | Bearer token | Merged health JSON |

### Auth
- Bearer token из env `HEALTH_TOKEN`
- `Authorization: Bearer <token>` header
- `/` (HTML) не требует auth — auth prompt на фронте
- `/api/health` без валидного токена → `401 { "error": "unauthorized" }`

### Data Collection

```javascript
// Pseudocode
async function collectHealth() {
  const health = readJSON("~/.openclaw/agent-health/data/agent-health.json") ?? {};
  const extended = readJSON("~/.openclaw/agent-health/data/agent-health-extended.json") ?? {};
  const incidents = readJSON("~/.openclaw/agent-health/data/agent-incidents.json") ?? [];
  const modes = readJSON("~/.openclaw/agent-control/agent-modes.json") ?? {};

  const host = {
    hostname: os.hostname(),
    load: os.loadavg(),           // [1min, 5min, 15min]
    mem_total_mb: Math.round(os.totalmem() / 1048576),
    mem_used_mb: Math.round((os.totalmem() - os.freemem()) / 1048576),
    mem_pct: Math.round((1 - os.freemem() / os.totalmem()) * 100),
    uptime_hours: Math.round(os.uptime() / 3600)
  };

  const docker = await getDockerStats("platon-gateway"); // { cpu_pct, mem_mb, status }

  return merge(health, extended, incidents, modes, host, docker);
}
```

### Docker Stats
```bash
docker stats platon-gateway --no-stream --format '{"cpu":"{{.CPUPerc}}","mem":"{{.MemUsage}}"}'
```
- Timeout: 3s — если не ответил, `{ status: "unknown" }`
- Parse CPU% и MEM из output

### JSON File Paths (configurable via env)

| Env var | Default | Source |
|---------|---------|--------|
| `HEALTH_JSON` | `~/.openclaw/agent-health/data/agent-health.json` | Watchdog |
| `EXTENDED_JSON` | `~/.openclaw/agent-health/data/agent-health-extended.json` | Skill |
| `INCIDENTS_JSON` | `~/.openclaw/agent-health/data/agent-incidents.json` | Watchdog |
| `MODES_JSON` | `~/.openclaw/agent-control/agent-modes.json` | Control Panel |
| `HEALTH_TOKEN` | (required) | — |
| `HEALTH_PORT` | `3456` | — |
| `DOCKER_CONTAINER` | `platon-gateway` | — |

---

## 3. API Response Format

`GET /api/health`:
```json
{
  "ts": "2026-03-16T09:30:00Z",
  "host": {
    "hostname": "empire-vps",
    "load": [1.2, 0.8, 0.6],
    "mem_total_mb": 8192,
    "mem_used_mb": 5100,
    "mem_pct": 62,
    "uptime_hours": 342
  },
  "docker": {
    "platon": {
      "cpu_pct": 3.2,
      "mem_mb": 180,
      "status": "running"
    }
  },
  "agents": {
    "main": {
      "label": "Сократ",
      "status": "healthy",
      "mode": "active",
      "locked": true,
      "last_heartbeat": "2026-03-16T09:25:00Z",
      "last_response": "2026-03-16T09:28:00Z",
      "consecutive_failures": 0,
      "tokens_24h": 14200,
      "errors_1h": 0
    },
    "archimedes": {
      "label": "Архимед",
      "status": "healthy",
      "mode": "active",
      "locked": false,
      "last_heartbeat": "2026-03-16T09:22:00Z",
      "last_response": "2026-03-16T09:20:00Z",
      "consecutive_failures": 0,
      "tokens_24h": 8100,
      "errors_1h": 0
    },
    "aristotle": {
      "label": "Аристотель",
      "status": "sleep",
      "mode": "sleep",
      "locked": false,
      "last_heartbeat": null,
      "last_response": null,
      "consecutive_failures": 0,
      "tokens_24h": 0,
      "errors_1h": 0
    },
    "herodotus": {
      "label": "Геродот",
      "status": "healthy",
      "mode": "active",
      "locked": false,
      "last_heartbeat": "2026-03-16T09:25:00Z",
      "last_response": "2026-03-16T09:23:00Z",
      "consecutive_failures": 0,
      "tokens_24h": 6000,
      "errors_1h": 0
    },
    "platon": {
      "label": "Платон",
      "status": "healthy",
      "mode": "active",
      "locked": false,
      "last_heartbeat": "2026-03-16T09:29:00Z",
      "last_response": "2026-03-16T09:28:00Z",
      "consecutive_failures": 0,
      "tokens_24h": 11700,
      "errors_1h": 0
    }
  },
  "incidents": [
    {
      "ts": "2026-03-16T07:30:00Z",
      "agent": "herodotus",
      "event": "heartbeat_missed",
      "action_taken": "ping_sent",
      "resolved": true,
      "resolved_at": "2026-03-16T07:31:12Z"
    }
  ]
}
```

**Merge logic:**
- `agents[id].status` — from `agent-health.json`
- `agents[id].mode` / `agents[id].locked` — from `agent-modes.json`
- `agents[id].tokens_24h` — from `agent-health-extended.json`
- `agents[id].label` — from `agent-modes.json`
- `incidents` — last 5 from `agent-incidents.json`, sorted by ts desc

---

## 4. Frontend (`index.html`)

### Layout

Один экран, три зоны:

```
┌─────────────────────────────────────────────────────────┐
│  ⚡ EMPIRE HEALTH              ● live  10s    09:30:15  │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  HOST: empire-vps  load: 1.2  mem: 62% █████████░░░░░   │
│  docker/platon: cpu 3.2%  mem 180MB  running              │
│                                                           │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐              │
│  │ 🟢 MAIN   │ │ 🟢 ARCHI  │ │ 💤 ARIST  │              │
│  │ Сократ    │ │ Архимед   │ │Аристотель │              │
│  │ 🔒 active │ │ active    │ │ sleep     │              │
│  │ hb: 2m    │ │ hb: 8m    │ │ hb: --    │              │
│  │ tok: 14.2k│ │ tok: 8.1k │ │ tok: 0    │              │
│  │ err: 0    │ │ err: 0    │ │ err: 0    │              │
│  └───────────┘ └───────────┘ └───────────┘              │
│  ┌───────────┐ ┌───────────┐                             │
│  │ 🟢 HEROD  │ │ 🟢 PLAT   │                             │
│  │ Геродот   │ │ Платон    │                             │
│  │ active    │ │ active    │                             │
│  │ hb: 5m    │ │ hb: 1m    │                             │
│  │ tok: 6.0k │ │ tok: 11.7k│                             │
│  │ err: 0    │ │ err: 0    │                             │
│  └───────────┘ └───────────┘                             │
│                                                           │
├─────────────────────────────────────────────────────────┤
│  INCIDENTS (last 5)                                       │
│  07:30  herodotus  heartbeat_missed   ✅ resolved 07:31  │
│  ---                                                      │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

### Visual Style

**Colors:**
- Background: `#0a0a0a`
- Primary text: `#00ff41` (matrix green)
- Dimmed text: `#4a4a4a`
- Card background: `#111111`
- Card border: `#1a3a1a`
- Status healthy: `#00ff41`
- Status sleep: `#666666`
- Status critical: `#ff4141`
- Status warning: `#ffaa00`
- Memory bar filled: `#00ff41`
- Memory bar empty: `#1a1a1a`

**Typography:**
- Font: `'JetBrains Mono', 'Fira Code', 'Cascadia Code', monospace`
- Header: 16px bold
- Card title: 14px bold
- Card body: 12px
- Incidents: 11px

**Card styling:**
- `border: 1px solid #1a3a1a`
- `border-radius: 4px`
- `padding: 12px`
- Hover: `box-shadow: 0 0 8px rgba(0, 255, 65, 0.15)`
- Status-dependent left border: 3px solid (color by status)

### Agent Card Content

```
┌──────────────┐
│ 🟢 AGENT_ID  │  ← status emoji + agentId uppercase
│ Label        │  ← human name (Сократ, Архимед...)
│ mode         │  ← active / sleep / 🔒 active (if locked)
│ hb: Nm       │  ← relative time since last heartbeat, or "--" if sleep
│ tok: N.Nk    │  ← tokens_24h formatted (14200 → 14.2k)
│ err: N       │  ← errors_1h count
└──────────────┘
```

**Status → emoji mapping:**
- `healthy` + `active` → 🟢
- `sleep` → 💤
- `warning` → ⚠️
- `critical` → 🔴
- `unknown` → ❓

### JS Logic

```javascript
const TOKEN_KEY = 'empire_health_token';

async function fetchHealth() {
  const token = localStorage.getItem(TOKEN_KEY);
  if (!token) { promptToken(); return; }

  try {
    const res = await fetch('/api/health', {
      headers: { 'Authorization': `Bearer ${token}` }
    });
    if (res.status === 401) { promptToken(); return; }
    const data = await res.json();
    renderHost(data.host, data.docker);
    renderAgents(data.agents);
    renderIncidents(data.incidents);
    setConnectionStatus('live');
    setLastUpdate(data.ts);
  } catch (e) {
    setConnectionStatus('disconnected');
  }
}

// Polling
setInterval(fetchHealth, 10000);
fetchHealth(); // initial

function promptToken() {
  const token = prompt('Enter access token:');
  if (token) {
    localStorage.setItem(TOKEN_KEY, token);
    fetchHealth();
  }
}
```

**Relative time helper:**
```javascript
function relativeTime(isoString) {
  if (!isoString) return '--';
  const diff = Math.round((Date.now() - new Date(isoString)) / 60000);
  if (diff < 1) return '<1m';
  if (diff < 60) return diff + 'm';
  if (diff < 1440) return Math.round(diff / 60) + 'h';
  return Math.round(diff / 1440) + 'd';
}
```

**Token formatter:**
```javascript
function fmtTokens(n) {
  if (n == null) return '--';
  if (n >= 1000) return (n / 1000).toFixed(1) + 'k';
  return String(n);
}
```

### Connection Indicator

Header right corner:
- `● live  10s` — зелёная точка, polling active
- `● disconnected` — красная точка, fetch failed (shows last data)
- Timestamp обновляется при каждом успешном fetch

### Responsive

- Desktop (>900px): 5 карточек в ряд (3 + 2), flex-wrap
- Tablet (600-900px): 3 + 2
- Mobile (<600px): 2 + 2 + 1, font-size уменьшен

---

## 5. Error Handling

| Сценарий | Server | Frontend |
|----------|--------|----------|
| JSON file missing | Отдаёт `{}` для секции | Карточка показывает "no data" |
| JSON file corrupted | try/catch, `{}` | То же |
| docker stats timeout (3s) | `docker.platon.status = "unknown"` | Показывает ❓ |
| Docker not running | `docker.platon.status = "stopped"` | Показывает 🔴 |
| Invalid Bearer token | 401 response | Prompt для нового токена |
| Network error (fetch fail) | — | 🔴 disconnected, сохраняет последние данные |
| All agents healthy | Normal response | Incidents: "No incidents — all clear ✅" |
| Server crash | — | 🔴 disconnected |
| Bun port conflict | Fail to start, stderr | — |

---

## 6. Deployment

### Systemd unit (`agent-health-dashboard.service`)
```ini
[Unit]
Description=Agent Health Dashboard
After=network.target

[Service]
Type=simple
User=node
WorkingDirectory=/home/node/.openclaw/agent-health-dashboard
ExecStart=/home/node/.bun/bin/bun server.js
EnvironmentFile=/home/node/.openclaw/agent-health-dashboard/.env
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

### Cloudflare Tunnel
- Добавить в существующий `cloudflared` конфиг (как vaultwarden)
- Route: `health.empire.example.com → localhost:3456`
- Tunnel auth + Bearer token = двойная защита

### `.env`
```
HEALTH_TOKEN=<random-32-char-string>
HEALTH_PORT=3456
DOCKER_CONTAINER=platon-gateway
```

---

## 7. Testing Strategy

| Тест | Метод |
|------|-------|
| Server starts | `bun server.js` → `curl localhost:3456` → HTML returned |
| API with auth | `curl -H "Authorization: Bearer $TOKEN" localhost:3456/api/health` → 200 + JSON |
| API without auth | `curl localhost:3456/api/health` → 401 |
| API with wrong token | `curl -H "Authorization: Bearer wrong" ...` → 401 |
| Missing JSON files | Удалить agent-health.json → API returns partial, не крашится |
| Corrupted JSON | Записать `{broken` → API returns `{}` для секции |
| Docker down | `docker stop platon-gateway` → `docker.platon.status = "stopped"` |
| Frontend render | Открыть в браузере → все карточки видны, host bar, incidents |
| Polling works | Изменить agent-health.json → через 10s dashboard обновился |
| Token prompt | Очистить localStorage → открыть → prompt появляется |
| Connection loss | Остановить server → dashboard показывает 🔴 disconnected |
| Responsive | Открыть на 400px → карточки 2 в ряд |

---

## 8. Scope Boundaries

### ✅ В скоупе (v1)
- Bun server (`server.js`, ~80 lines)
- Single HTML file (`index.html`, ~300 lines)
- Bearer token auth
- `/api/health` — merged JSON endpoint
- Host metrics: load average, RAM usage, uptime
- Docker stats: CPU, RAM, status (Платон)
- 5 agent cards: status, mode, heartbeat, tokens, errors
- Last 5 incidents
- Polling 10s
- Dark terminal theme (#0a0a0a + #00ff41)
- Connection indicator (live/disconnected)
- Responsive layout
- Cloudflare tunnel deployment
- Systemd unit

### ❌ НЕ в скоупе (YAGNI)
- WebSocket / SSE (polling достаточно)
- Charts / graphs (числа и статусы)
- Historical data browsing
- Agent management from dashboard (→ Control Panel spec)
- Log viewer
- Multi-VPS / multi-gateway dashboard
- User management / login page
- Dark/light theme toggle (only dark)
- Notifications from dashboard
- Mobile app / PWA

---

## 9. File Structure

```
~/.openclaw/agent-health-dashboard/
├── server.js              # Bun HTTP server (~80 lines)
├── index.html             # Vanilla SPA (~300 lines)
├── .env                   # HEALTH_TOKEN, HEALTH_PORT, DOCKER_CONTAINER
├── agent-health-dashboard.service   # Systemd unit file
└── README.md              # Setup & usage docs
```

---

## Changelog
- 2026-03-16: Initial design. Brainstorm with Sokrat in topic:4.
