# Agent Control Panel — Design Spec

## Overview

Inline keyboard панель в Telegram для управления режимами агентов Империи. Юра отправляет `/panel` в личку Сократа → получает сообщение с кнопками → нажимает кнопку → агент переключается между active/sleep. Одна кнопка = один toggle.

**Агенты:**
- **main** (Сократ) — 🔒 always active, не управляется через панель
- **archimedes** (Архимед) — dev, gateway `main` (port 18789)
- **aristotle** (Аристотель) — research, gateway `main`
- **herodotus** (Геродот) — news, gateway `main`
- **platon** (Платон) — координатор, отдельный Docker (port 18795)

**Контроллер:** Сократ (main agent) — получает callbacks, выполняет config.patch + restart.

---

## 1. Архитектура

```
┌─────────────────────────────────────────────────┐
│  Юра (TG личка)                                  │
│    │                                              │
│    ├─ "/panel" ──→ Сократ ──→ send message       │
│    │                         с inline кнопками    │
│    │                                              │
│    └─ нажимает кнопку ──→ callback_data ──→      │
│         Сократ получает как текст:                │
│         "callback_data: agent:archimedes:toggle"  │
│              │                                    │
│              ▼                                    │
│         Skill agent-control:                      │
│         1. Проверяет sender_id = 473841722        │
│         2. Читает agent-modes.json                │
│         3. Определяет toggle: active↔sleep        │
│         4. Выполняет:                             │
│            ├─ main gateway agents:                │
│            │  config.patch + gateway restart       │
│            └─ platon (Docker):                    │
│               docker exec config.patch + restart  │
│         5. Обновляет agent-modes.json             │
│         6. Edit message — обновлённые кнопки      │
└─────────────────────────────────────────────────┘
```

**Принципы:**
- `/panel` работает только в личке Сократа (не в группе)
- Сократ не может усыпить сам себя — кнопка заблокирована
- Toggle-стиль: одно нажатие = переключение
- `agent-modes.json` — source of truth для состояний
- `panel_message_id` — сохраняется для edit при toggle

---

## 2. Конфигурация

### `agent-modes.json` (state file)
```json
{
  "agents": {
    "main":       { "mode": "active", "locked": true,  "label": "Сократ" },
    "archimedes": { "mode": "active", "locked": false, "label": "Архимед" },
    "aristotle":  { "mode": "sleep",  "locked": false, "label": "Аристотель" },
    "herodotus":  { "mode": "active", "locked": false, "label": "Геродот" },
    "platon":     { "mode": "active", "locked": false, "label": "Платон" }
  },
  "brainstorm": { "enabled": true },
  "panel_message_id": null
}
```

### Agent config mapping
| Agent | Gateway | Config path | Restart command |
|-------|---------|-------------|-----------------|
| archimedes | main | `agents.archimedes.enabled` | `openclaw gateway restart` |
| aristotle | main | `agents.aristotle.enabled` | `openclaw gateway restart` |
| herodotus | main | `agents.herodotus.enabled` | `openclaw gateway restart` |
| platon | platon (Docker) | `agents.default.enabled` | `docker exec platon-gateway openclaw gateway restart` |

---

## 3. Компоненты

### 3.1 Skill `agent-control/SKILL.md`

**Триггеры:**
- Входящий текст `/panel` → `render_panel()` + send
- Входящий текст `callback_data: agent:*:toggle` → `handle_toggle()`
- Входящий текст `callback_data: brainstorm:toggle` → `handle_brainstorm()`

### 3.2 Action Handlers

**`render_panel()`**
Генерирует текст сообщения + inline keyboard:

Текст:
```
🎛️ Agent Control Panel

🔒 Сократ (main) — always active
🟢 Архимед — active
💤 Аристотель — sleep
🟢 Геродот — active
🟢 Платон — active

🧠 Brainstorm: ON
```

Кнопки (inline keyboard):
```
Row 1: [🟢 Архимед] [💤 Аристотель]
Row 2: [🟢 Геродот] [🟢 Платон]
Row 3: [🧠 Brainstorm: ON]
```

**`handle_toggle(agentId)`**
1. Verify sender_id == 473841722 → else silent ignore
2. Read `agent-modes.json`
3. Check `agents[agentId].locked` → if true, ignore
4. Toggle: `active` → `sleep` / `sleep` → `active`
5. Call `apply_config(agentId, newMode)`
6. Update `agent-modes.json` (atomic write: tmp + rename)
7. Call `render_panel()` → `message(edit, panel_message_id, new buttons)`

**`apply_config(agentId, mode)`**
```bash
# For main gateway agents (archimedes, aristotle, herodotus):
openclaw gateway config.patch agents.{agentId}.enabled={true|false}
openclaw gateway restart

# For platon (Docker):
docker exec platon-gateway openclaw gateway config.patch agents.default.enabled={true|false}
docker exec platon-gateway openclaw gateway restart
```

**`handle_brainstorm()`**
1. Verify sender_id
2. Toggle `brainstorm.enabled` in agent-modes.json
3. Update SESSION-STATE file (existing mechanism)
4. Edit panel message

### 3.3 Callback Data Format

| Callback | Action |
|----------|--------|
| `agent:archimedes:toggle` | Toggle Архимед active↔sleep |
| `agent:aristotle:toggle` | Toggle Аристотель active↔sleep |
| `agent:herodotus:toggle` | Toggle Геродот active↔sleep |
| `agent:platon:toggle` | Toggle Платон active↔sleep |
| `brainstorm:toggle` | Toggle brainstorm on↔off |

Все строки < 64 bytes (Telegram limit).

---

## 4. Data Flow

### `/panel` command:
```
Юра: "/panel" (в личке Сократа)
  → Сократ получает текст "/panel"
  → Skill agent-control активируется
  → render_panel(): читает agent-modes.json
  → message(action=send, buttons=[[...]])
  → Сохраняет message_id в agent-modes.json
  → Юра видит панель с кнопками
```

### Button press:
```
Юра: нажимает [💤 Аристотель]
  → Telegram отправляет callback
  → Сократ получает: "callback_data: agent:aristotle:toggle"
  → Skill: verify sender_id ✓
  → Skill: read agent-modes.json → aristotle.mode = "sleep"
  → Skill: toggle → "active"
  → Skill: apply_config("aristotle", "active")
    → exec: openclaw gateway config.patch agents.aristotle.enabled=true
    → exec: openclaw gateway restart
  → Skill: write agent-modes.json → aristotle.mode = "active"
  → Skill: render_panel() → message(action=edit, messageId=..., buttons=[[...]])
  → Юра видит: [🟢 Аристотель] вместо [💤 Аристотель]
```

---

## 5. Error Handling

| Сценарий | Реакция |
|----------|---------|
| Неавторизованный sender | Silent ignore — не отвечать, не логировать |
| config.patch failed | Не менять agent-modes.json, edit panel с ⚠️ у агента |
| gateway restart failed | То же — ⚠️ в панели + текст ошибки внизу |
| docker exec failed (Платон) | ⚠️ в панели + "Docker unreachable" |
| Toggle на locked агента | Невозможно — кнопки нет в панели |
| panel_message_id stale/deleted | Отправить новое сообщение, обновить id |
| Одновременные нажатия | Atomic write (tmp + rename) предотвращает race condition |
| Gateway restart affects multiple agents | Ожидаемо — restart один раз, все агенты на этом gateway |

**Error display в панели:**
```
🎛️ Agent Control Panel

🔒 Сократ (main) — always active
🟢 Архимед — active
⚠️ Аристотель — error: restart failed
🟢 Геродот — active
🟢 Платон — active

❌ Last error: gateway restart failed at 09:15
```

---

## 6. Testing Strategy

| Тест | Метод |
|------|-------|
| `/panel` render | Отправить /panel, проверить текст + кнопки соответствуют agent-modes.json |
| Toggle active→sleep | Нажать кнопку 🟢 агента, проверить: config.patch вызван с enabled=false, panel updated с 💤 |
| Toggle sleep→active | Нажать кнопку 💤 агента, проверить: config.patch вызван с enabled=true, panel updated с 🟢 |
| Auth: valid sender | Callback от 473841722 → обработан |
| Auth: invalid sender | Callback от другого sender_id → ignored, panel не изменилась |
| Main lock | /panel не содержит кнопку для main, только текст 🔒 |
| Docker agent (Платон) | Toggle platon → docker exec вызван с правильными аргументами |
| Brainstorm toggle | Нажать brainstorm → SESSION-STATE обновлён + panel updated |
| Error recovery | Остановить gateway → toggle → ⚠️ в панели → запустить gateway → toggle → ✅ |
| Stale message | Удалить panel message → toggle → новое сообщение отправлено |
| State persistence | Toggle agent → restart Сократа → /panel → состояния сохранены |

---

## 7. Scope Boundaries

### ✅ В скоупе (v1)
- `/panel` команда в личке → inline keyboard
- Toggle active/sleep per agent (кроме main)
- Toggle brainstorm on/off
- Auth по sender_id (473841722)
- Edit message при каждом toggle
- agent-modes.json как persistent state
- Error display в панели
- Docker support для Платона

### ❌ НЕ в скоупе (YAGNI)
- Scheduling (sleep at 23:00, wake at 08:00) — v2
- Per-agent model switching через кнопки
- Batch operations (sleep all / wake all)
- History of mode changes
- Panel в группе (только личка)
- Confirmation dialogs (кроме main lock)
- Custom modes beyond active/sleep
- Auto-sync с реальным состоянием gateway (panel = source of truth)

---

## 8. File Structure

```
~/.openclaw/skills/agent-control/
├── SKILL.md              # Skill definition + trigger rules
└── README.md             # Usage docs

~/.openclaw/agent-control/
└── agent-modes.json      # Persistent state
```

---

## Changelog
- 2026-03-16: Initial design. Brainstorm with Sokrat in topic:4.
