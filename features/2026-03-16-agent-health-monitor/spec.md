# Agent Health Monitor — Design Spec

## Overview

Система мониторинга здоровья агентов Империи. Отслеживает liveness, heartbeat, token usage и аномалии поведения для 5 агентов на 2 gateway-ах.

**Агенты:**
- **main** (Сократ) — главный агент, gateway `main` (port 18789)
- **archimedes** (Архимед) — dev, gateway `main`
- **aristotle** (Аристотель) — research, gateway `main`
- **herodotus** (Геродот) — news, gateway `main`
- **platon** (Платон) — координатор, отдельный Docker gateway (port 18795)

**Группа:** Империя (Telegram), каждый агент в своём топике.

---

## 1. Архитектура

Два слоя, разделённые по принципу "наблюдатель не зависит от наблюдаемого":

### Слой 1 — Shell Watchdog (cron)
- Независимый bash-скрипт, работает через cron/systemd timer
- Проверяет gateway liveness и per-agent heartbeat
- Эскалация: ping → restart → алерт
- Пишет `agent-health.json` (текущий snapshot) и `agent-incidents.json` (лог событий)
- Алерты через Telegram Bot API напрямую (curl)

### Слой 2 — OpenClaw Skill `agent-health`
- Запускается по heartbeat cron (каждые 30 мин) или по команде
- Собирает расширенные метрики через Gateway API (session_status)
- Детектит аномалии: token usage spikes, зацикливание
- Пишет `agent-health-extended.json`
- Отправляет сводки в топик группы

### Dashboard (Brain page)
- Новая секция "Agent Health" на существующем дашборде (localhost:3333)
- Читает все JSON-файлы состояния
- Auto-refresh

---

## 2. Конфигурация

`watchdog-config.json`:
```json
{
  "gateways": {
    "main":   { "port": 18789 },
    "platon": { "port": 18795 }
  },
  "agents": {
    "main":       { "gateway": "main",   "agentId": "main",       "interval_min": 5,  "alert_after_failures": 2 },
    "archimedes": { "gateway": "main",   "agentId": "archimedes", "interval_min": 15, "alert_after_failures": 2 },
    "aristotle":  { "gateway": "main",   "agentId": "aristotle",  "interval_min": 15, "alert_after_failures": 2 },
    "herodotus":  { "gateway": "main",   "agentId": "herodotus",  "interval_min": 15, "alert_after_failures": 2 },
    "platon":     { "gateway": "platon", "agentId": "default",    "interval_min": 5,  "alert_after_failures": 2 }
  },
  "telegram": {
    "bot_token": "${HEALTH_BOT_TOKEN}",
    "alert_topic_chat": "-100XXXXXXXXXX",
    "alert_topic_id": "NN",
    "personal_chat_id": "XXXXXXXXXX"
  },
  "escalation": {
    "ping_timeout_sec": 30,
    "restart_commands": {
      "main": "systemctl --user restart openclaw-gateway",
      "platon": "docker restart platon-gateway"
    },
    "max_auto_restarts_per_hour": 3
  }
}
```

**Параметры per-agent:**
- `interval_min` — частота проверки (cron)
- `alert_after_failures` — сколько подряд провалов до эскалации

---

## 3. Компоненты

### 3.1 `agent-health-watchdog.sh`

**Ответственность:** базовый liveness мониторинг, эскалация, алертинг.

**Логика работы:**
1. Читает `watchdog-config.json`
2. Для каждого gateway: `curl -s -m 5 localhost:{port}/api/status`
   - Если gateway мёртв → critical alert + restart command
3. Для каждого агента: проверяет `heartbeat-state.json`
   - Читает timestamp последнего heartbeat
   - Если `now - last_heartbeat > interval × alert_after` → начинает эскалацию
4. Эскалация (per-agent):
   - **Step 1:** Ping — отправляет тестовое сообщение через Gateway API
   - **Step 2:** Ждёт `ping_timeout_sec` (30s), проверяет ответ
   - **Step 3:** Нет ответа → restart gateway (если не превышен лимит restarts/hour)
   - **Step 4:** После restart ждёт 60s, повторяет ping
   - **Step 5:** Всё ещё нет ответа → critical alert в личку
5. Обновляет `agent-health.json`
6. При смене статуса агента → аппендит в `agent-incidents.json`

**Cron setup:**
```bash
# Минимальный интервал = 5 мин (для main и platon)
*/5 * * * * /path/to/agent-health-watchdog.sh
```
Скрипт сам определяет, каких агентов проверять в этом цикле (по `interval_min`).

### 3.2 OpenClaw Skill `agent-health`

**Ответственность:** расширенная аналитика, anomaly detection, отчёты.

**Триггер:** heartbeat cron (каждые 30 мин) или ручная команда "health report".

**Логика:**
1. Для каждого агента вызывает Gateway API `session_status`:
   - `curl localhost:{port}/api/agents/{agentId}/status`
   - Получает: token usage (input/output), session count, model, uptime
2. Записывает метрики с timestamp в `agent-health-extended.json`
3. Anomaly detection:
   - **Token spike:** usage за последний час > 2× среднего за 24h → 🟡 warning
   - **Token spike critical:** usage > 5× среднего → 🔴 alert
   - **Loop detection:** последние 5 ответов агента содержат >80% дублирующегося текста → 🟡 warning
4. Отправляет сводку в топик группы (ежедневно в 09:00 или по запросу)

**Daily digest формат:**
```
📊 Agent Health — 2026-03-16

🟢 main (Сократ) — healthy, 14.2k tokens/24h
🟢 archimedes — healthy, 8.1k tokens/24h
🟡 aristotle — high usage, 45.3k tokens/24h ⚠️
🟢 herodotus — healthy, 6.0k tokens/24h
🟢 platon — healthy, 11.7k tokens/24h

Incidents (24h): 1 — aristotle token spike at 03:14
```

### 3.3 Data Files

**`agent-health.json`** — текущий snapshot (пишет watchdog):
```json
{
  "updated_at": "2026-03-16T08:50:00Z",
  "gateways": {
    "main":   { "status": "up", "checked_at": "2026-03-16T08:50:00Z", "response_ms": 42 },
    "platon": { "status": "up", "checked_at": "2026-03-16T08:50:00Z", "response_ms": 38 }
  },
  "agents": {
    "main": {
      "status": "healthy",
      "last_heartbeat": "2026-03-16T08:45:00Z",
      "last_response": "2026-03-16T08:42:00Z",
      "consecutive_failures": 0,
      "last_restart": null
    },
    "archimedes": { "..." : "..." },
    "aristotle":  { "..." : "..." },
    "herodotus":  { "..." : "..." },
    "platon":     { "..." : "..." }
  }
}
```

**`agent-health-extended.json`** — расширенные метрики (пишет skill):
```json
{
  "updated_at": "2026-03-16T09:00:00Z",
  "agents": {
    "main": {
      "tokens_24h": { "input": 8200, "output": 6000 },
      "tokens_7d":  { "input": 52000, "output": 41000 },
      "avg_tokens_per_hour": 592,
      "sessions_active": 2,
      "model": "anthropic/claude-sonnet-4-6",
      "anomalies": []
    }
  }
}
```

**`agent-incidents.json`** — rolling log (7 дней):
```json
[
  {
    "ts": "2026-03-16T07:30:00Z",
    "agent": "herodotus",
    "event": "heartbeat_missed",
    "action_taken": "ping_sent",
    "resolved": true,
    "resolved_at": "2026-03-16T07:31:12Z",
    "details": "Responded to ping after 12s"
  },
  {
    "ts": "2026-03-15T03:14:00Z",
    "agent": "aristotle",
    "event": "token_spike",
    "action_taken": "alert_sent",
    "resolved": true,
    "resolved_at": "2026-03-15T04:00:00Z",
    "details": "Usage 5.2x average, normalized after 46min"
  }
]
```

---

## 4. Data Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                        cron (*/5 * * * *)                       │
│                              │                                   │
│                    agent-health-watchdog.sh                       │
│                         │          │                             │
│              curl :18789/api/status  curl :18795/api/status      │
│                    │                    │                         │
│              ┌─────▼────┐        ┌─────▼────┐                   │
│              │ Gateway   │        │ Gateway   │                  │
│              │ main      │        │ platon    │                  │
│              │ (4 agents)│        │ (1 agent) │                  │
│              └─────┬─────┘       └─────┬─────┘                  │
│                    │                    │                         │
│              heartbeat-state.json  heartbeat-state.json          │
│                    │                    │                         │
│                    ▼                    ▼                         │
│              ┌──────────────────────────┐                        │
│              │   agent-health.json       │  ← current snapshot   │
│              │   agent-incidents.json    │  ← event log          │
│              └────────────┬─────────────┘                        │
│                           │                                      │
│    ┌──────────────────────┼──────────────────────┐              │
│    │                      │                       │              │
│    ▼                      ▼                       ▼              │
│  Dashboard          OC Skill                TG Alerts            │
│  (Brain page)       agent-health            (Bot API)            │
│  reads JSONs        extends with            ├─ topic: warnings   │
│  auto-refresh       token/anomaly data      └─ personal: critical│
│                     writes extended.json                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Error Handling

| Сценарий | Детекция | Реакция |
|----------|----------|---------|
| Gateway не отвечает | curl timeout >5s | Restart gateway → alert |
| Agent heartbeat stale | timestamp > interval × alert_after | Ping → restart → alert |
| Agent не отвечает на ping | Нет ответа за 30s | Restart → повторный ping → critical alert |
| Token usage spike | >2× avg (warning), >5× avg (critical) | Alert в топик / личку |
| Зацикливание агента | >80% дублей в последних 5 ответах | Alert в топик |
| Watchdog сам упал | Нет обновления agent-health.json >15 мин | Отдельный cron-canary: `check-watchdog.sh` |
| Слишком много restarts | >3 restarts/hour per gateway | Прекращает авто-restart, only alerts |
| TG Bot API недоступен | curl к api.telegram.org fail | Лог в stderr, retry при следующем цикле |

**Watchdog self-monitoring:**
Отдельный минимальный cron (раз в 15 мин) проверяет `mtime` файла `agent-health.json`. Если старше 15 мин → алерт через fallback (можно email или отдельный TG бот).

---

## 6. Dashboard — Brain Page секция "Agent Health"

### Layout

```
┌─────────────────────────────────────────────────┐
│  🧠 Agent Health                    🔄 60s auto │
├─────────────────────────────────────────────────┤
│                                                   │
│  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐   │
│  │ 🟢     │ │ 🟢     │ │ 🟡     │ │ 🟢     │   │
│  │ Сократ │ │Архимед │ │Аристот.│ │Геродот │   │
│  │ main   │ │archimed│ │aristotl│ │herodotu│   │
│  │ 2m ago │ │ 8m ago │ │ ⚠️ hi  │ │ 5m ago │   │
│  └────────┘ └────────┘ └────────┘ └────────┘   │
│                                                   │
│  ┌────────┐                                      │
│  │ 🟢     │                                      │
│  │ Платон │                                      │
│  │ platon │                                      │
│  │ 1m ago │                                      │
│  └────────┘                                      │
│                                                   │
│  ── Incidents (24h) ──────────────────────────── │
│  08:30  herodotus  heartbeat_missed  ✅ resolved │
│  03:14  aristotle  token_spike       ✅ resolved │
│                                                   │
│  ── Token Usage (7d) ────────────────────────── │
│  main       ▁▂▃▂▂▃▂  avg 590/h                  │
│  archimedes ▁▁▂▁▁▁▁  avg 340/h                  │
│  aristotle  ▁▂▃▅█▃▂  avg 420/h  ⚠️ spike 03:14 │
│  herodotus  ▁▁▁▁▂▁▁  avg 250/h                  │
│  platon     ▁▂▂▃▂▂▁  avg 490/h                  │
│                                                   │
└─────────────────────────────────────────────────┘
```

### Статусы
- 🟢 **healthy** — heartbeat свежий, метрики в норме
- 🟡 **warning** — heartbeat stale ИЛИ anomaly detected
- 🔴 **critical** — не отвечает, gateway down, или после неудачного restart

### Технические детали
- Dashboard делает `fetch('/api/agent-health')` каждые 60s
- Backend endpoint читает JSON-файлы и отдаёт merged view
- Sparkline — CSS/SVG, без внешних библиотек
- Инциденты — последние 20, сортировка по времени

---

## 7. Alerting

### Каналы

| Уровень | Куда | Пример |
|---------|------|--------|
| 🟡 Warning | Топик в группе Империя | "⚠️ aristotle: token usage 2.3× avg" |
| 🔴 Critical | Топик + личка | "🔴 main gateway DOWN, restart failed" |
| ℹ️ Info | Только топик | "ℹ️ herodotus: auto-restarted, back online" |

### Формат алертов

```
🔴 CRITICAL — main gateway
Gateway localhost:18789 не отвечает.
Auto-restart: failed (attempt 2/3)
Agents affected: main, archimedes, aristotle, herodotus
Action required: manual intervention

Last healthy: 2026-03-16T08:45:00Z (7 min ago)
```

```
⚠️ WARNING — aristotle
Token usage spike: 1,840 tokens/h (avg: 420/h, 4.4×)
Duration: 46 min
No action taken — monitoring.
```

### Дедупликация
- Не слать повторный алерт того же уровня для того же агента чаще раз в 30 мин
- При повышении уровня (warning → critical) — слать сразу
- При разрешении — одно сообщение "✅ resolved"

---

## 8. Testing Strategy

| Что тестируем | Как |
|--------------|-----|
| Watchdog: gateway down | Остановить gateway, проверить alert + restart |
| Watchdog: heartbeat stale | Подменить timestamp в heartbeat-state.json на old |
| Watchdog: escalation flow | Мокнуть curl чтобы ping не проходил |
| Skill: token spike detection | Подменить extended.json с аномальными значениями |
| Skill: loop detection | Создать mock ответы с дублями |
| Dashboard: render | Подменить JSON-файлы с разными статусами |
| Alerting: dedup | Запустить watchdog дважды подряд, проверить 1 алерт |
| E2E: full cycle | Kill agent → watchdog detects → restart → verify recovery |

**Smoke test script** (`test-health-monitor.sh`):
1. Подменяет heartbeat timestamp на 1 час назад
2. Запускает watchdog
3. Проверяет: alert отправлен, incident записан, статус = warning
4. Восстанавливает timestamp
5. Запускает watchdog
6. Проверяет: resolved сообщение, статус = healthy

---

## 9. Scope Boundaries

### В скоупе (v1)
- Gateway liveness check
- Per-agent heartbeat monitoring
- Эскалация: ping → restart → alert
- Token usage tracking (24h/7d rolling)
- Basic anomaly detection (spike + loop)
- Dashboard секция в Brain page
- TG алерты (топик + личка)
- Дедупликация алертов
- Watchdog self-monitoring

### НЕ в скоупе (YAGNI)
- Response quality analysis (семантический анализ ответов)
- Cross-agent dependency tracking
- Автоматическое масштабирование агентов
- Метрики latency per-message
- Интеграция с Prometheus/Grafana
- Mobile app для мониторинга
- Log aggregation (logstash/elastic)
- Метрики стоимости ($/agent) — может быть v2

---

## 10. File Structure

```
~/.openclaw/
├── agent-health/
│   ├── watchdog-config.json        # Конфигурация
│   ├── agent-health-watchdog.sh    # Cron скрипт (слой 1)
│   ├── check-watchdog.sh           # Self-monitoring canary
│   ├── test-health-monitor.sh      # Smoke tests
│   ├── data/
│   │   ├── agent-health.json       # Current snapshot
│   │   ├── agent-health-extended.json  # Extended metrics
│   │   └── agent-incidents.json    # Rolling incident log (7d)
│   └── README.md
├── skills/
│   └── agent-health/
│       └── SKILL.md                # OpenClaw skill definition
└── workspaces/
    └── brain-dashboard/
        └── src/
            └── pages/
                └── agent-health/   # Dashboard page component
```

---

## Changelog
- 2026-03-16: Initial design. Brainstorm with Sokrat in topic:4.
