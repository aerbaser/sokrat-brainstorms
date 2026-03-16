# Autonomous Product Development Pipeline

## Overview

Полностью автономный pipeline от идеи до delivery в multi-agent системе Платона. Event-driven архитектура с GitHub как backbone, 9 фаз, supervised auto mode (30-мин auto-continue), параллельная разработка.

## Context

**Система:** 5 агентов на VPS через OpenClaw:
- **Сократ** — координатор, event dispatcher
- **Архимед** — dev-оркестратор, управляет ACP сессиями
- **Аристотель** — research, анализ
- **Платон** — brainstorm, spec generation
- **Геродот** — reporting

**Инфраструктура:** GitHub (aerbaser), Docker, Codex/Claude Code как ACP coding agents, CI/CD через GitHub Actions, один VPS.

**Существующие скиллы:** brainstorming, superpowers, ci-architect, issue-architect, plan-to-issues, coding agents.

## Design Decisions

| Решение | Выбор | Обоснование |
|---------|-------|-------------|
| Уровень автономности | Supervised auto, 30-мин auto-continue | Человек видит прогресс, может вмешаться, но не блокирует |
| Primary product type | Web apps (80%), CLI + skills как extensions | Покрывает основной кейс, расширяемо |
| Источник идей | GitHub Issues backlog + AI-генерация (v2) | v1 — ручной backlog, AI idea gen — v2 |
| AI↔AI brainstorm | Research-first: Аристотель → Платон → Сократ-review | Решения на данных, не фантазиях |
| Оркестрация | Параллельный оркестратор (Архимед), 2-3 issues | Скорость важнее простоты |
| Тестирование | Гибрид: dev = unit (TDD), QA = integration + E2E | Два уровня защиты |
| UI тесты | Playwright primary, Agent Browser fallback | Стабильность, CI-friendly |
| Delivery | Auto-deploy staging, manual prod trigger | Безопасность production |
| Feedback | Формальная ретро + метрики после каждого проекта | Метрики без анализа бесполезны |

---

## 1. Architecture

### Pipeline — 9 фаз, связанных через GitHub events

```
┌─────────────┐   label:approved   ┌───────────┐   research:done   ┌──────────────┐
│  1. IDEATE  │ ─────────────────▶ │ 2. RESEARCH│ ────────────────▶│ 3. BRAINSTORM │
│  (Backlog)  │                    │(Аристотель)│                  │   (Платон)    │
└─────────────┘                    └───────────┘                   └──────┬───────┘
                                                                         │ spec:ready
┌─────────────┐   issues:ready     ┌────────────┐                  ┌────▼────────┐
│ 5. DEVELOP  │ ◀──────────────── │ 4. DECOMPOSE│ ◀──────────────│  Сократ      │
│  (Архимед)  │                    │(Spec→Issues)│   spec:approved │  (Review)    │
└──────┬──────┘                    └────────────┘                  └─────────────┘
       │ pr:ready
┌──────▼──────┐   tests:pass      ┌───────────┐   review:pass    ┌─────────────┐
│  6. TEST    │ ─────────────────▶│ 7. REVIEW  │ ───────────────▶│  8. DEPLOY  │
│ (QA Agent)  │                    │(AI Review) │                  │  (Staging)  │
└─────────────┘                    └───────────┘                  └──────┬──────┘
                                                                         │ deploy:done
                                                                  ┌─────▼───────┐
                                                                  │  9. RETRO   │
                                                                  │ (Feedback)  │
                                                                  └─────────────┘
```

### Event Bus = GitHub Labels + Comments + PR status

- Каждая фаза меняет label на issue/PR → следующий агент реагирует
- Сократ — event dispatcher: мониторит GitHub events, роутит на нужного агента
- Состояние pipeline = набор labels на GitHub issue (single source of truth)
- 30-мин auto-continue: если human notification отправлено и нет ответа — продолжаем

### Notification Layer

- Каждый переход между фазами → уведомление в Telegram
- Topic:3 (Brainstorm) — для фаз 1-4
- Topic:6 (Results) — для фаз 5-9
- Человек может написать "стоп" в любой момент → pipeline paused

---

## 2. Components — детально по фазам

### Phase 1 — IDEATE (Backlog Manager)

**Где:** GitHub Issues в repo `sokrat-core` с label `idea`

**Input:**
- Человек создаёт issue с label `idea`
- (v2) Агент создаёт issue с label `idea:ai-generated`

**Scoring:**
- Каждая идея: `impact` (1-5) × `feasibility` (1-5) = score
- Score записывается в комментарий к issue
- Формат: `📊 Impact: 4/5 | Feasibility: 5/5 | Score: 20/25`

**Приоритизация:**
- Человек ставит label `idea:approved` на идеи к реализации
- Pipeline берёт approved с наивысшим score
- Одновременно один проект в pipeline (очередь для остальных)

### Phase 2 — RESEARCH (Аристотель)

**Trigger:** Issue получает label `pipeline:research`

**Действия:**
1. Web search конкурентов и аналогов
2. Анализ best practices для выбранного tech stack
3. Сбор технических ограничений
4. Компиляция результатов в `research.md`

**Output:**
- Комментарий на issue с research summary
- Файл `research.md` в repo
- Label `pipeline:research-done`

**Время:** 10-20 минут

### Phase 3 — BRAINSTORM (Платон)

**Trigger:** Label `pipeline:brainstorm`

**Input:** Текст issue + research.md от Аристотеля

**Действия:**
1. Self-brainstorm на основе собранных данных
2. Платон отвечает на вопросы brainstorming skill, используя research как факты
3. Генерит spec.md через стандартный процесс (architecture → components → data flow → error handling → testing → scope)

**Output:**
- `spec.md` в `sokrat-brainstorms/products/<name>/spec.md`
- Label `pipeline:spec-ready`

**Время:** 15-30 минут

### Phase 4 — SPEC REVIEW + DECOMPOSE (Сократ + Архитектор)

**Trigger:** Label `pipeline:spec-review`

**4a — Review (Сократ):**
- Ревьюит spec через subagent-reviewer (Phase 6 brainstorming skill)
- Max 5 итераций review loop
- Approve → label `pipeline:spec-approved`
- Notification человеку → 30 мин auto-continue

**4b — Decompose (после approve):**
- Парсит spec.md → извлекает компоненты, endpoints, UI pages
- Создаёт GitHub issues (один issue = один branch = один PR)

**Формат каждого issue:**
```markdown
## Title
[Конкретная задача]

## Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2
- [ ] Criterion 3

## Technical Notes
[Из spec — что конкретно делать]

## Dependencies
Blocked by: #N (если есть)
```

**Labels:** `priority:high/med/low`, `type:frontend/backend/infra/test`
**Milestone:** `<project-name> v1`
**Dependency links:** `Blocked by #N` в body

**Output:** N issues созданы, label `pipeline:issues-ready`

### Phase 5 — DEVELOP (Архимед → ACP sessions)

**Trigger:** Label `pipeline:dev`

**Архимед как оркестратор:**
1. Берёт issues из milestone, строит dependency graph
2. Спавнит 2-3 ACP сессии параллельно (только issues без блокирующих зависимостей)
3. Мониторит прогресс каждой сессии

**Каждая ACP сессия:**
- Создаёт branch от main: `feat/<issue-number>-<short-name>`
- Пишет код + unit тесты (TDD style)
- Создаёт PR с описанием (что сделано, acceptance criteria checklist)
- CI запускается автоматически

**Merge strategy:**
- Каждый PR проходит CI (lint + typecheck + unit tests)
- Архимед мержит PR в main (squash merge)
- Перед merge — rebase на latest main
- При conflict: auto-resolve если trivial, sequential merge если complex

**Branch naming:** `feat/<issue-number>-<short-description>`
**Commit format:** `feat(#N): description` / `fix(#N): description` / `test(#N): description`

### Phase 6 — TEST (QA Agent)

**Trigger:** Все dev issues в milestone merged → label `pipeline:test`

**QA Agent — отдельная ACP сессия:**

**Level 2 — Integration тесты:**
1. Читает spec.md + исходный код
2. Пишет integration test suite:
   - API endpoints (request → response, status codes, validation)
   - Database operations (CRUD, constraints)
   - Service interactions (auth → API → DB flow)
3. Framework: Supertest (Node) / httpx (Python), testcontainers для DB

**Level 3 — E2E тесты (Playwright):**
1. Извлекает user stories из spec
2. Пишет Playwright тесты:
   - Critical user flows (CRUD operations)
   - Формы: валидация, submit, error states
   - Навигация: все роуты, 404
   - Responsive: desktop (1280px) + mobile (375px)
3. Setup: Docker compose (app + DB) + headless Chromium

**Test failure loop:**
```
Tests fail
    │
    ▼
QA Agent анализирует → создаёт bug issue:
    - Failing test name
    - Error message
    - Expected vs actual
    - Suggested fix area
    │
    ▼
Архимед назначает на ACP сессию → fix → push → re-test
    │
    ├─ pass → pipeline continues
    └─ fail → iterate (max 3, потом escalate)
```

**Output:** Test report в PR comment, label `pipeline:tests-pass`

### Phase 7 — CODE REVIEW (AI Reviewer)

**Trigger:** Label `pipeline:review`

**Кто:** Отдельная ACP сессия с review-specific prompt ("свежие глаза")

**Checklist:**
- [ ] Code quality: naming, structure, DRY
- [ ] Security: injection, auth, secrets exposure
- [ ] Архитектурное соответствие spec.md
- [ ] Edge cases: null handling, error states, boundary conditions
- [ ] Performance: N+1 queries, unnecessary re-renders, large payloads
- [ ] Test quality: meaningful assertions, edge case coverage

**Output:**
- Approve → label `pipeline:review-pass`
- Request changes → comments на PR, конкретные fix tasks → назад в Phase 5
- Max 2 review rounds, потом escalate

### Phase 8 — DEPLOY (Staging)

**Trigger:** Label `pipeline:deploy`

**Steps:**
1. Docker build (multi-stage, production config)
2. Push image to registry (GitHub Container Registry)
3. Deploy на staging VPS (docker-compose up)
4. Smoke test: health check endpoint + basic page load
5. Генерация changelog (из PR descriptions)

**Output:**
- Staging URL отправляется в Telegram (topic:6)
- Changelog в Telegram
- Label `pipeline:deployed`

**Production deploy:**
- Только ручной trigger
- Человек пишет "деплой в прод" или через GitHub Actions manual dispatch
- Same Docker image что на staging (promote, не rebuild)

### Phase 9 — RETRO (Feedback Loop)

**Trigger:** Label `pipeline:retro` (после успешного deploy)

**Метрики собираются автоматически:**

| Метрика | Как считается |
|---------|---------------|
| Total time | Время от `pipeline:research` до `pipeline:deployed` |
| Time per phase | Timestamps на каждом label change |
| CI fail count | Кол-во failed CI runs |
| Review iterations | Кол-во review rounds |
| Bug issues created | Issues с label `bug:auto` |
| Test coverage | Coverage report из CI |
| Lines of code | `git diff --stat` main..before vs after |

**Retro report:**
```markdown
# 🔄 Retro: [Project Name]

## Метрики
- Общее время: X часов
- Фазы: Research 15m, Brainstorm 25m, Dev 3h, Test 45m, Review 20m, Deploy 10m
- CI fails: N
- Bugs found in testing: N
- Review iterations: N
- Test coverage: N%

## Что пошло хорошо
- [автоматически: фазы без retries]

## Что можно улучшить
- [автоматически: фазы с retries, длинные фазы]

## Learnings
- [конкретные паттерны для запоминания]
```

**Куда идёт:**
- Retro report → Telegram (topic:6)
- Learnings → memory файлы (для self-improvement skill)
- Metrics → `metrics.json` (накопительно, для трендов)
- Label `pipeline:done`

---

## 3. Data Flow

### Три слоя данных:

```
┌─────────────────────────────────────────────────────────┐
│                    GITHUB (Source of Truth)               │
│                                                           │
│  Issues          PRs            Files           Labels    │
│  ┌──────┐       ┌──────┐      ┌──────────┐    ┌───────┐ │
│  │idea  │──────▶│code  │─────▶│spec.md   │    │pipeline│ │
│  │body  │       │diff  │      │research.md│   │:phase  │ │
│  │score │       │tests │      │retro.md  │    │:status │ │
│  └──────┘       └──────┘      └──────────┘    └───────┘ │
└──────────────────────┬──────────────────────────────────┘
                       │ events (label change, PR merge, comment)
┌──────────────────────▼──────────────────────────────────┐
│                 СОКРАТ (Event Dispatcher)                 │
│                                                           │
│  Monitor loop:                                            │
│  1. Poll GitHub events (каждые 2 мин)                    │
│  2. Match event → pipeline phase                          │
│  3. Dispatch to agent via sessions_spawn                  │
│  4. Track active sessions + timeouts                      │
│  5. Handle 30-min auto-continue                           │
│                                                           │
│  State tracking:                                          │
│  ┌─────────────────────────────────────┐                 │
│  │ pipeline-state.json (per project)   │                 │
│  │ { phase, started_at, agent,         │                 │
│  │   issues[], notified_at, human_ack }│                 │
│  └─────────────────────────────────────┘                 │
└──────────────────────┬──────────────────────────────────┘
                       │ sessions_spawn / sessions_send
┌──────────────────────▼──────────────────────────────────┐
│                    АГЕНТЫ (Workers)                       │
│                                                           │
│  Аристотель ──▶ research.md (→ issue comment)            │
│  Платон     ──▶ spec.md (→ sokrat-brainstorms repo)      │
│  Архимед    ──▶ code + tests (→ PR на project repo)      │
│  QA Agent   ──▶ test report (→ PR comment)               │
│  Reviewer   ──▶ review (→ PR review)                     │
│  Deployer   ──▶ staging URL (→ Telegram)                 │
└─────────────────────────────────────────────────────────┘
```

### Артефакты

| Артефакт | Где | Формат |
|----------|-----|--------|
| Идея | GitHub Issue body | Markdown |
| Score | Issue comment | `impact:N feasibility:N score:N` |
| Research | Issue comment + `research.md` | Markdown |
| Spec | `sokrat-brainstorms/products/<name>/spec.md` | Markdown |
| Decomposed issues | GitHub Issues (milestone) | Standard GH format |
| Code | PR на project repo | Branch per issue |
| Test report | PR comment + CI artifacts | Markdown + JSON |
| Review | PR review | GH review comments |
| Deploy log | Telegram + `deploy.log` | Text |
| Changelog | Telegram + `CHANGELOG.md` | Markdown |
| Retro | Telegram + `retro.md` + memory | Markdown + JSON |
| Metrics | `metrics.json` | JSON (cumulative) |

### Пример flow для одного проекта:

1. Issue `#42 "Task manager app"` — label `idea:approved`, score 20
2. Сократ видит → `pipeline:research` → спавнит Аристотеля
3. Аристотель: research.md → комментарий на #42 → `pipeline:research-done`
4. Сократ → `pipeline:brainstorm` → спавнит Платона
5. Платон: issue + research → spec.md → push в `sokrat-brainstorms` → `pipeline:spec-ready`
6. Сократ → уведомление → 30 мин → auto-continue → `pipeline:decompose`
7. Декомпозиция: spec → issues #43, #44, #45 (milestone "task-manager-v1")
8. Архимед: #43 + #44 параллельно (нет зависимостей), #45 ждёт (#43 dependency)
9. PR #43 merged → Архимед стартует #45. PR #44 merged.
10. Все issues closed → `pipeline:test` → QA agent: integration + E2E → pass
11. `pipeline:review` → AI review → approve
12. `pipeline:deploy` → Docker build → staging → ссылка в Telegram
13. `pipeline:retro` → метрики + learnings → `pipeline:done`

---

## 4. Error Handling

### 4.1 — Типы ошибок и реакция

| Фаза | Что может сломаться | Реакция | Max retries |
|------|---------------------|---------|-------------|
| RESEARCH | Web search timeout, API limit | Retry через 5 мин. 3 fails → skip research + notify | 3 |
| BRAINSTORM | Spec пустой/бессмысленный | Reviewer reject → re-brainstorm. 2 fails → escalate | 2 |
| DECOMPOSE | Spec слишком абстрактный | Назад в brainstorm с комментарием "нужна конкретика" | 1 |
| DEVELOP | ACP сессия crash/timeout | Restart на том же branch. Corrupted → fresh branch | 2 |
| DEVELOP | CI fails (lint, typecheck) | ACP получает CI output, фиксит, push. Max 3 fix-cycles | 3 |
| DEVELOP | Merge conflict | Rebase. Trivial → auto. Complex → sequential merge | 2 |
| TEST | Тесты fail | Bug issue → fix → re-test. Max 3 цикла | 3 |
| REVIEW | Changes requested | Назад в dev с конкретными comments. Max 2 rounds | 2 |
| DEPLOY | Docker build fail | Fix Dockerfile PR → re-deploy. Infra → notify human | 2 |
| DEPLOY | Smoke test fail | Назад в test фазу с ошибкой | 1 |

### 4.2 — Escalation chain

```
Retry (автоматически)
    │ max retries exceeded
    ▼
Notify human в Telegram (ошибка + что пробовали)
    │ 30 мин нет ответа
    ▼
Pipeline PAUSED (label: pipeline:blocked)
    │ human вмешался
    ▼
Resume с текущей фазы
```

### 4.3 — Timeout policy

| Фаза | Max время | При timeout |
|------|-----------|-------------|
| Research | 30 мин | Skip + notify |
| Brainstorm | 60 мин | Kill, retry once |
| Decompose | 20 мин | Retry once |
| Dev (per issue) | 120 мин | Kill, reassign |
| Test | 60 мин | Kill, retry |
| Review | 30 мин | Auto-approve ⚠️ |
| Deploy | 15 мин | Kill, notify human |

### 4.4 — Poison pill protection

Если один проект в цикле retry → fail → retry больше 3 раз на одной фазе:
- Pipeline останавливается для ЭТОГО проекта
- Label `pipeline:poisoned`
- Notification с полным логом
- Другие проекты НЕ блокируются

### 4.5 — State recovery

Если Сократ перезапустился:
1. Читает `pipeline-state.json`
2. Проверяет GitHub labels (labels > local state при конфликте)
3. Возобновляет мониторинг

---

## 5. Testing Strategy

### Три уровня

```
┌─────────────────────────────────────────────┐
│     Level 3: E2E / UI (QA Agent)            │
│  Playwright: user flows, forms, navigation  │
│  ПОСЛЕ merge всех issues                    │
├─────────────────────────────────────────────┤
│     Level 2: Integration (QA Agent)         │
│  API, DB, service interactions              │
│  ПОСЛЕ merge всех issues                    │
├─────────────────────────────────────────────┤
│     Level 1: Unit (Dev Agent, TDD)          │
│  Functions, components, utils               │
│  ВМЕСТЕ с кодом                             │
└─────────────────────────────────────────────┘
```

### Level 1: Unit (Dev)
- TDD preferred
- Каждая функция — happy path + edge case минимум
- Framework: Vitest (frontend), Jest/pytest (backend)
- CI gate: coverage ≥ 70%

### Level 2: Integration (QA Agent)
- API endpoints: request → response, status codes, validation
- DB operations: CRUD, constraints, migrations
- Service interactions: auth → API → DB flow
- Framework: Supertest/httpx + testcontainers

### Level 3: E2E (QA Agent + Playwright)
- Critical user flows из spec
- Формы: валидация, submit, errors
- Навигация: все роуты + 404
- Responsive: desktop (1280px) + mobile (375px)
- Headless Chromium в CI

### По типу продукта

| Тип | Unit | Integration | E2E |
|-----|------|-------------|-----|
| Web app | ✅ Vitest + Jest | ✅ API + DB | ✅ Playwright |
| CLI tool | ✅ Unit | ✅ Command integration | ❌ |
| OpenClaw skill | ✅ Unit | ✅ Skill mock | ❌ |

### Agent Browser (fallback)
- Primary: Playwright (детерминистичный, CI-native)
- Fallback: Agent Browser — визуальная проверка по запросу человека
- Скриншоты → Telegram

---

## 6. Scope

### v1 (In Scope)

- ✅ Полный pipeline: idea → spec → issues → code → test → review → staging → retro
- ✅ Web apps как primary target
- ✅ GitHub Issues + Labels как state machine
- ✅ Параллельная разработка (2-3 issues)
- ✅ Unit + Integration + E2E тесты
- ✅ AI code review
- ✅ Auto-deploy на staging (Docker)
- ✅ Формальная ретро + метрики
- ✅ 30-мин auto-continue
- ✅ Poison pill protection
- ✅ Single VPS

### v2+ (Out of Scope)

- ❌ Multi-repo projects
- ❌ Production auto-deploy
- ❌ Visual regression testing
- ❌ Performance/load testing
- ❌ AI-генерация идей по cron
- ❌ A/B testing / feature flags
- ❌ Multi-VPS / cloud
- ❌ Production мониторинг + auto-rollback
- ❌ User feedback collection

### Явные ограничения v1

- Один проект в pipeline одновременно
- Один VPS для staging
- GitHub как единственный git provider
- Docker-only деплой
- Английские labels/branches, русская коммуникация

---

## 7. Label State Machine

Полная карта labels и переходов:

```
idea → idea:approved → pipeline:research → pipeline:research-done
→ pipeline:brainstorm → pipeline:spec-ready → pipeline:spec-review
→ pipeline:spec-approved → pipeline:decompose → pipeline:issues-ready
→ pipeline:dev → pipeline:dev-done → pipeline:test → pipeline:tests-pass
→ pipeline:review → pipeline:review-pass → pipeline:deploy
→ pipeline:deployed → pipeline:retro → pipeline:done
```

**Error labels:**
- `pipeline:blocked` — ждёт human intervention
- `pipeline:poisoned` — застрял, требует ручного разбора
- `bug:auto` — баг найденный в testing

**Auxiliary labels:**
- `priority:high`, `priority:med`, `priority:low`
- `type:frontend`, `type:backend`, `type:infra`, `type:test`
- `idea:ai-generated` (v2)

---

## 8. Agent Responsibilities Summary

| Агент | Роль в pipeline | Фазы |
|-------|----------------|------|
| **Сократ** | Event dispatcher, координатор, spec reviewer | Мониторинг всех фаз, Phase 4a |
| **Аристотель** | Research, data collection | Phase 2 |
| **Платон** | Brainstorm, spec generation | Phase 3 |
| **Архимед** | Dev orchestrator, merge manager | Phase 5, bug fixes |
| **QA Agent** | Integration + E2E тесты | Phase 6 |
| **AI Reviewer** | Code review | Phase 7 |
| **Deployer** | Docker build + staging deploy | Phase 8 |
| **Геродот** | Retro report, metrics | Phase 9 |

*QA Agent, AI Reviewer, Deployer — это не отдельные постоянные агенты, а ACP сессии спавнимые по требованию.*

---

## 9. Implementation Priority

Порядок реализации pipeline (инкрементальный):

1. **Label state machine** — GitHub labels + Сократ monitor loop
2. **Phase 5: Dev** — Архимед + ACP сессии (core value)
3. **Phase 6: Test** — QA agent + CI integration
4. **Phase 4: Decompose** — spec → issues автоматизация
5. **Phase 3: Brainstorm** — AI↔AI self-brainstorm
6. **Phase 2: Research** — Аристотель automation
7. **Phase 7: Review** — AI code review
8. **Phase 8: Deploy** — Docker staging pipeline
9. **Phase 9: Retro** — Metrics + feedback loop
10. **Phase 1: Ideate** — Backlog management + scoring

*Логика: начинаем с dev core (Phases 4-6), потом расширяем в обе стороны.*
