# Autonomous Product Development Pipeline

## Overview

Полностью автономный pipeline от идеи до delivery в multi-agent системе Платона. Event-driven архитектура с GitHub как backbone, 9 фаз, supervised auto mode (30-мин auto-continue), параллельная разработка.

## Context

**Система Платона** (Docker, отдельный OpenClaw instance):

| ID | Эмодзи | Модель | Роль |
|----|--------|--------|------|
| `main` (Платон) | 🏛️ | Sonnet | Координатор, event dispatcher, spec reviewer |
| `brainstorm` | 🧠 | Opus/Codex | Brainstorm, spec generation |
| `architect` | 🏗 | Sonnet | Triage, планы, GitHub issues, decomposition |
| `ops` | ⚙️ | Sonnet | AO мониторинг, CI setup, deploy, health |

**Coding agents:** ACP сессии (Codex/Claude Code) — спавнятся через AO (Agent Orchestrator)

**Инфраструктура:** GitHub (aerbaser), Docker, AO, CI/CD через GitHub Actions, VPS.

**Существующие скиллы:**
- `brainstorming` — Q&A → spec
- `issue-architect` — идеальные AO-ready issues
- `plan-to-issues` — декомпозиция плана на пакеты
- `ci-architect` — генерация CI DAG
- `ao-ops` — управление Agent Orchestrator
- `ci-forensics` — диагностика CI failures

**Группа Telegram:** `-1003822323654`
- topic:3 — Codex Brainstorm
- topic:4 — Claude Brainstorm
- topic:6 — Results
- topic:7 — Ops
- topic:10 — Architect

## Design Decisions

| Решение | Выбор | Обоснование |
|---------|-------|-------------|
| Уровень автономности | Supervised auto, 30-мин auto-continue | Человек видит прогресс, может вмешаться, но не блокирует |
| Primary product type | Web apps (80%), CLI + skills как extensions | Покрывает основной кейс, расширяемо |
| Источник идей | GitHub Issues backlog + AI-генерация (v2) | v1 — ручной backlog, AI idea gen — v2 |
| AI↔AI brainstorm | Research-first: Brainstorm + web_search → Платон-review | Решения на данных, не фантазиях |
| Оркестрация | AO (Agent Orchestrator) через Ops | AO уже есть, параллельные ACP сессии |
| Тестирование | Гибрид: dev = unit (TDD), QA = integration + E2E | Два уровня защиты |
| UI тесты | Playwright primary, Agent Browser fallback | Стабильность, CI-friendly |
| Delivery | Auto-deploy staging, manual prod trigger | Безопасность production |
| Feedback | Формальная ретро + метрики после каждого проекта | Метрики без анализа бесполезны |

---

## 1. Architecture

### Pipeline — 9 фаз, связанных через GitHub events

```
┌─────────────┐   label:approved   ┌─────────────┐   research:done   ┌──────────────┐
│  1. IDEATE  │ ─────────────────▶ │ 2. RESEARCH │ ────────────────▶│ 3. BRAINSTORM │
│  (Backlog)  │                    │ (Brainstorm │                  │  (Brainstorm  │
│             │                    │  web_search)│                  │   + spec)     │
└─────────────┘                    └─────────────┘                  └──────┬───────┘
                                                                          │ spec:ready
┌─────────────┐   issues:ready     ┌────────────┐                  ┌─────▼────────┐
│ 5. DEVELOP  │ ◀──────────────── │ 4. DECOMPOSE│ ◀──────────────│  Платон       │
│ (AO + ACP)  │                    │ (Architect) │   spec:approved │  (Review)     │
└──────┬──────┘                    └────────────┘                  └──────────────┘
       │ pr:ready
┌──────▼──────┐   tests:pass      ┌───────────┐   review:pass    ┌─────────────┐
│  6. TEST    │ ─────────────────▶│ 7. REVIEW  │ ───────────────▶│  8. DEPLOY  │
│  (CI + E2E) │                    │(AI Review) │                  │   (Ops)     │
└─────────────┘                    └───────────┘                  └──────┬──────┘
                                                                         │ deploy:done
                                                                  ┌─────▼───────┐
                                                                  │  9. RETRO   │
                                                                  │  (Платон)   │
                                                                  └─────────────┘
```

### Event Bus = GitHub Labels + Comments + PR status

- Каждая фаза меняет label на issue/PR → следующий агент реагирует
- **Платон (main)** — event dispatcher: мониторит GitHub events, роутит на нужного агента
- Состояние pipeline = набор labels на GitHub issue (single source of truth)
- 30-мин auto-continue: если human notification отправлено и нет ответа → продолжаем

### Agent Responsibility Map

| Агент | Фазы | Навыки |
|-------|-------|--------|
| **Платон** | Event dispatch, Phase 4 review, Phase 9 retro | Координация, sessions_spawn |
| **Brainstorm** | Phase 2 (research), Phase 3 (spec) | brainstorming, web_search |
| **Architect** | Phase 4 (decompose → issues) | issue-architect, plan-to-issues, ci-architect |
| **Ops** | Phase 5 (AO management), Phase 6 (CI), Phase 8 (deploy) | ao-ops, ci-forensics, ci-architect |
| **ACP sessions** | Phase 5 (coding), Phase 6 (unit tests) | Codex/Claude Code через AO |

### Notification Layer

- Каждый переход между фазами → уведомление в Telegram
- topic:4 — для фаз 2-3 (brainstorm)
- topic:10 — для фазы 4 (issues)
- topic:7 — для фаз 5-6, 8 (dev/CI/deploy)
- topic:6 — для финальных результатов (7-9)
- Человек может написать "стоп" в любой момент → pipeline paused

---

## 2. Components — детально по фазам

### Phase 1 — IDEATE (Backlog Manager)

**Агент:** Платон (main)
**Где:** GitHub Issues в project repo с label `idea`

**Input:**
- Человек создаёт issue с label `idea`
- (v2) Brainstorm агент генерирует идеи и создаёт issue с label `idea:ai-generated`

**Scoring:**
- Каждая идея: `impact` (1-5) × `feasibility` (1-5) = score
- Score записывается в комментарий к issue
- Формат: `📊 Impact: 4/5 | Feasibility: 5/5 | Score: 20/25`

**Приоритизация:**
- Человек ставит label `idea:approved` на идеи к реализации
- Pipeline берёт approved с наивысшим score
- Одновременно один проект в pipeline (очередь для остальных)

### Phase 2 — RESEARCH (Brainstorm agent)

**Агент:** Brainstorm (🧠)
**Trigger:** Issue получает label `pipeline:research`

**Процесс:**
1. Платон спавнит Brainstorm с задачей research
2. Brainstorm использует `web_search` + `web_fetch` для сбора данных:
   - Аналоги на рынке
   - Технические подходы
   - Best practices
3. Результат → `research.md` в комментарий к GitHub issue
4. Label → `pipeline:research-done`

**Output:** Markdown с секциями:
- Рынок/аналоги (3-5 ключевых)
- Технические подходы (с trade-offs)
- Рекомендация
- Источники (URLs)

**Timeout:** 15 мин. Если Brainstorm не отвечает → Платон пинг → 5 мин → skip research, продолжить brainstorm с тем что есть.

### Phase 3 — BRAINSTORM (Brainstorm agent)

**Агент:** Brainstorm (🧠)
**Trigger:** Label `pipeline:brainstorm`
**Skill:** `brainstorming` (SKILL.md)

**AI↔AI Brainstorm:**
1. Платон формирует промпт на основе issue + research.md
2. Brainstorm запускает полный brainstorming flow (skill):
   - Phase 1: контекст из repo
   - Phase 2: вопросы → **Платон отвечает** (AI↔AI, sessions_send)
   - Phase 3: подходы
   - Phase 4: дизайн по секциям → Платон approve
   - Phase 5: spec.md → commit → push в `sokrat-brainstorms`
3. Brainstorm отправляет ссылку на spec.md в topic:6

**Критично:** Платон отвечает на вопросы Brainstorm агента как прокси-пользователь, используя research data + контекст проекта. Если уверенность < 50% → эскалация к человеку.

**Output:** `spec.md` в `sokrat-brainstorms/products/<name>/spec.md`

### Phase 4 — SPEC REVIEW + DECOMPOSE

**Агенты:** Платон (review) → Architect (decompose)

**4a — Review (Платон):**
- Читает spec.md
- Проверяет: полнота, реализуемость, YAGNI
- Если проблемы → возвращает Brainstorm с комментариями
- Если ОК → label `pipeline:spec-approved`
- Уведомление в topic:6: "Spec approved. Начинаю decompose."
- 30-мин window: человек может вмешаться

**4b — Decompose (Architect):**
**Skill:** `issue-architect` + `plan-to-issues`

1. Architect читает spec.md
2. Создаёт GitHub issues:
   - Каждый issue = один branch = один PR
   - Labels: `ao-ready`, `priority:N`, `phase:N`
   - Acceptance criteria = конкретные проверки
   - Dependencies: `depends-on: #XX`
3. Создаёт milestone на GitHub
4. Если > 10 файлов в issue → разбивает на sub-issues
5. Label → `pipeline:issues-ready`

**Issue format (AO-ready):**
```markdown
## Context
[Ссылка на spec, что реализуем]

## Task
[Конкретные файлы, функции, endpoints]

## Acceptance Criteria
- [ ] Unit tests pass
- [ ] Endpoint returns 200
- [ ] Integration test added

## Dependencies
depends-on: #XX (если есть)
```

### Phase 5 — DEVELOP (AO + ACP sessions)

**Агенты:** Ops (AO management) → ACP sessions (Codex/Claude Code)
**Trigger:** Label `pipeline:issues-ready` на milestone

**Ops запускает AO:**
1. `ao start` для проекта
2. `ao lifecycle-worker <project> --interval-ms 30000`
3. AO берёт issues с label `ao-ready` и спавнит ACP сессии

**AO как оркестратор:**
- Смотрит dependency graph
- Параллельно: 2-3 independent issues одновременно
- Последовательно: dependent issues ждут completion
- Для каждого issue: branch → code → tests → PR

**Coding flow (per issue):**
1. AO спавнит ACP сессию (Codex или Claude Code)
2. ACP читает issue, пишет код + unit тесты (TDD)
3. `git push` → PR создаётся
4. CI запускается автоматически
5. Если CI fail → AO re-спавнит ACP на fix

**Branch strategy:**
- `main` — protected, only merge via PR
- `ao/<issue-number>-<short-name>` — рабочие ветки
- Squash merge после approve

**Merge:**
- AO автоматически мержит PR если CI pass + no conflicts
- При конфликтах → AO спавнит ACP на resolve

**Мониторинг:**
- Ops отслеживает через `ao status`, `ao session ls`
- Timeout: 30 мин на issue. Если ACP завис → Ops убивает сессию, переспавнивает
- Статус в topic:7

### Phase 6 — TEST (CI + E2E)

**Агенты:** Ops (CI monitoring) + ACP sessions (E2E тесты)
**Trigger:** Все PRs merged → label `pipeline:test`

**Три уровня тестирования:**

**Level 1 — Unit (уже в Phase 5):**
- Каждый ACP пишет unit тесты вместе с кодом (TDD)
- CI запускает: `bun test` / `npm test`
- Coverage threshold: 70%

**Level 2 — Integration:**
- Ops спавнит ACP сессию для integration тестов
- Тестирует: API endpoints, data flow между компонентами
- Framework: тот же что unit (Vitest/Jest), но с реальными dependencies

**Level 3 — E2E / UI (для web apps):**
- **Tool:** Playwright (primary)
- Ops спавнит ACP сессию для E2E:
  1. ACP читает spec → пишет Playwright тесты
  2. Тестирует: навигация, формы, кнопки, flows
  3. Visual snapshots для regression
- **Fallback:** Agent Browser tool для интерактивной проверки
- CI запускает: `npx playwright test`

**Если тесты падают:**
1. Ops анализирует лог (skill: `ci-forensics`)
2. Создаёт fix issue с label `bug` + `ao-ready`
3. AO спавнит ACP на fix
4. Re-run тестов
5. Max 3 retry. Если после 3 fail → эскалация к человеку

**Output:** Test report в комментарии к milestone issue

### Phase 7 — CODE REVIEW (AI Review)

**Агент:** Платон (main) спавнит review subagent
**Trigger:** Label `pipeline:review`

**Процесс:**
1. Платон спавнит subagent с задачей code review
2. Subagent проверяет:
   - Соответствие spec
   - Code quality, naming, structure
   - Security issues (SQL injection, XSS, secrets)
   - Performance (N+1, unnecessary re-renders)
   - Test coverage
3. Результат:
   - ✅ Approved → label `pipeline:review-pass`
   - ❌ Issues → GitHub review comments → fix issues → retry
4. Max 3 review rounds. После 3 → эскалация.

**Review checklist:**
```
□ Spec compliance — все requirements из spec реализованы
□ No dead code — нет unused imports/functions
□ Error handling — все async operations wrapped
□ Security — no secrets, no injection vectors
□ Tests — coverage adequate, meaningful assertions
□ Performance — no obvious N+1, unnecessary loops
```

### Phase 8 — DEPLOY (Ops)

**Агент:** Ops (⚙️)
**Trigger:** Label `pipeline:deploy`

**Staging auto-deploy:**
1. Ops настраивает deployment:
   - Docker build → local container
   - Cloudflare tunnel для доступа
   - `.env` из secrets
2. Запускает staging
3. Smoke test (curl endpoints, basic health)
4. Уведомление в topic:6: "Staging ready: [URL]"

**Production:**
- Manual trigger только (человек пишет "деплой в прод")
- Ops выполняет production deploy

**Rollback:**
- Если staging smoke fail → auto-rollback
- Production rollback → manual trigger

### Phase 9 — RETRO (Платон)

**Агент:** Платон (main)
**Trigger:** Deploy done → label `pipeline:retro`

**Формальная ретроспектива:**
1. Платон собирает метрики:
   - Общее время pipeline (час/дни)
   - Количество issues / PRs / re-tries
   - Test pass rate (first attempt)
   - Review rounds
   - Human interventions (сколько раз человек вмешался)
2. Анализ:
   - Что сработало хорошо
   - Что затормозило pipeline
   - Какие фазы нужно улучшить
3. Output:
   - `retro.md` в `sokrat-brainstorms/products/<name>/`
   - Learnings → memory файлы Платона
   - Метрики → `metrics.json`

**Feedback loop:**
- Learnings из ретро влияют на следующий pipeline:
  - Улучшенные промпты для brainstorm
  - Обновлённые шаблоны issues
  - Новые checklist items для review

---

## 3. Data Flow

### Артефакты по фазам

```
Phase 1 (Ideate)     → GitHub Issue с label `idea:approved`
Phase 2 (Research)   → research.md (комментарий к issue)
Phase 3 (Brainstorm) → spec.md (sokrat-brainstorms repo)
Phase 4 (Decompose)  → GitHub Issues с labels `ao-ready`
Phase 5 (Develop)    → Code + PRs в project repo
Phase 6 (Test)       → Test reports (CI artifacts + comments)
Phase 7 (Review)     → Review comments на PRs
Phase 8 (Deploy)     → Running staging + URL
Phase 9 (Retro)      → retro.md + metrics.json
```

### Кто что создаёт

```
┌─────────────────────────────────────────────────────┐
│  Brainstorm  ──▶ research.md + spec.md              │
│  Architect   ──▶ GitHub issues (AO-ready)           │
│  Ops + AO    ──▶ code + tests (→ PR на project repo)│
│  Ops         ──▶ CI config + staging deploy         │
│  Платон      ──▶ review + retro + coordination      │
└─────────────────────────────────────────────────────┘
```

---

## 4. End-to-End Example

**Scenario:** "Dashboard для мониторинга агентов" → delivery

```
1. Юра создаёт issue #42 "Agent Health Dashboard" → label `idea`
2. Юра ставит `idea:approved` → Платон видит → `pipeline:research`
3. Платон спавнит Brainstorm → web_search аналогов, tech stack
4. Brainstorm: research.md → комментарий на #42 → `pipeline:research-done`
5. Платон → `pipeline:brainstorm` → Brainstorm запускает brainstorming skill
6. AI↔AI brainstorm: Brainstorm задаёт вопросы ← Платон отвечает
7. spec.md → push в `sokrat-brainstorms` → `pipeline:spec-ready`
8. Платон ревьюит spec → OK → `pipeline:spec-approved`
9. Уведомление Юре → 30 мин → auto-continue → `pipeline:decompose`
10. Architect: spec → 5 issues (#43-#47), milestone "Health Dashboard"
11. `pipeline:issues-ready` → Ops запускает AO
12. AO: #43 + #44 параллельно (нет зависимостей), #45 ждёт (#43 dependency)
13. PRs, CI, unit тесты → merge
14. `pipeline:test` → Ops спавнит E2E (Playwright)
15. E2E pass → `pipeline:review`
16. AI code review → approved → `pipeline:deploy`
17. Ops: Docker build → staging → Cloudflare tunnel → URL
18. Юре: "🏛️ Dashboard готов: [staging URL]. Ревью?"
19. Юра: "ОК" → `pipeline:retro`
20. Платон: retro.md, metrics, learnings
```

**Timeline estimate:** 2-4 часа (simple web app), 6-12 часов (complex product)

---

## 5. Error Handling

### По фазам

| Фаза | Ошибка | Действие |
|------|--------|----------|
| Research | web_search timeout | Retry ×2, skip research |
| Brainstorm | Agent unresponsive | Платон пинг, 5 мин timeout, restart session |
| Decompose | Spec unclear | Architect пишет вопросы → Brainstorm → Платон |
| Develop | ACP hang | Ops: `ao session kill`, re-spawn. Max 3 retries |
| Develop | Merge conflict | AO спавнит ACP для resolve |
| Test | Flaky test | Re-run ×2. Если consistently fails → fix issue |
| Test | All tests fail | Stop pipeline, эскалация к человеку |
| Deploy | Build fail | Ops: ci-forensics → fix → retry |
| Deploy | Staging crash | Rollback, fix issue → retry |

### Poison Pill Protection

Если issue не может быть resolved после 3 ACP respawns:
1. Label `pipeline:blocked`
2. Comment с диагностикой
3. Уведомление человеку: "⚠️ Issue #X заблокирован после 3 попыток: [причина]"
4. Pipeline продолжает с другими issues (skip blocked)

### Recovery After Restart

Если Платон перезапустился:
1. Читает GitHub labels → восстанавливает состояние pipeline
2. Каждая фаза idempotent — можно перезапустить
3. Labels = single source of truth → no lost state

---

## 6. Label State Machine

```
idea → idea:approved → pipeline:research → pipeline:research-done →
pipeline:brainstorm → pipeline:spec-ready → pipeline:spec-approved →
pipeline:decompose → pipeline:issues-ready → pipeline:develop →
pipeline:test → pipeline:review → pipeline:review-pass →
pipeline:deploy → pipeline:deploy-done → pipeline:retro → pipeline:complete
```

**Blocked states:** `pipeline:blocked`, `pipeline:human-needed`

**Каждый label = ровно один владелец:**

| Label | Кто ставит | Кто реагирует |
|-------|-----------|---------------|
| `idea:approved` | Человек | Платон |
| `pipeline:research` | Платон | Brainstorm |
| `pipeline:research-done` | Brainstorm | Платон |
| `pipeline:brainstorm` | Платон | Brainstorm |
| `pipeline:spec-ready` | Brainstorm | Платон |
| `pipeline:spec-approved` | Платон | Architect |
| `pipeline:decompose` | Платон | Architect |
| `pipeline:issues-ready` | Architect | Ops |
| `pipeline:develop` | Ops | AO |
| `pipeline:test` | Ops | Ops |
| `pipeline:review` | Ops | Платон |
| `pipeline:review-pass` | Платон | Ops |
| `pipeline:deploy` | Платон/Ops | Ops |
| `pipeline:deploy-done` | Ops | Платон |
| `pipeline:retro` | Платон | Платон |
| `pipeline:complete` | Платон | — |

---

## 7. Testing Strategy — Detail

### Unit Tests (Phase 5, inline)
- **Кто:** ACP sessions (Codex/Claude Code)
- **Когда:** Одновременно с кодом (TDD)
- **Что:** Functions, utilities, API handlers
- **Framework:** Vitest (Bun), Jest (Node)
- **Coverage:** ≥70%

### Integration Tests (Phase 6a)
- **Кто:** ACP session, спавненная Ops
- **Когда:** После merge всех PRs
- **Что:** API endpoints, data flow, auth
- **Framework:** Vitest/Jest + supertest
- **Env:** Testcontainers для DB/Redis

### E2E / UI Tests (Phase 6b)
- **Кто:** ACP session, спавненная Ops
- **Когда:** После integration pass
- **Что:** User flows, формы, навигация, кнопки
- **Framework:** Playwright
- **Скрипт:**
  1. ACP читает spec → выделяет user flows
  2. Пишет `tests/e2e/*.spec.ts`
  3. Запускает `npx playwright test`
  4. При failure: скриншот + trace → анализ → fix

### Visual Regression (Phase 6c, v2)
- **Tool:** Playwright screenshots → diff
- **Baseline:** первый успешный run = baseline
- **Threshold:** 1% pixel diff

---

## 8. Scope Boundaries

### In scope (v1)
- [x] 9-phase pipeline с label state machine
- [x] AI↔AI brainstorm (Brainstorm ↔ Платон)
- [x] Automated spec → issues decomposition
- [x] AO-driven parallel development
- [x] 3-level testing (unit + integration + E2E)
- [x] AI code review
- [x] Staging auto-deploy
- [x] Retro + metrics

### Out of scope (v2+)
- [ ] AI idea generation (auto-create issues)
- [ ] Multi-project parallel pipelines
- [ ] Production auto-deploy
- [ ] Visual regression testing
- [ ] Performance/load testing
- [ ] A/B testing
- [ ] User analytics integration

---

## 9. Implementation Priority

| # | Компонент | Критичность | Зависимости |
|---|-----------|-------------|-------------|
| 1 | **Label state machine** — GitHub labels + Платон monitor loop | Core | — |
| 2 | **Phase 5: Dev** — AO + ACP sessions (core value) | Core | AO setup |
| 3 | **Phase 4: Decompose** — Architect issue creation | Core | issue-architect skill |
| 4 | **Phase 6: Test** — CI + E2E | Core | CI config |
| 5 | **Phase 3: Brainstorm** — AI↔AI flow | High | brainstorming skill |
| 6 | **Phase 2: Research** — Brainstorm web_search | Medium | web_search tool |
| 7 | **Phase 7: Review** — AI code review | High | — |
| 8 | **Phase 8: Deploy** — Staging automation | Medium | Docker, Cloudflare |
| 9 | **Phase 9: Retro** — Metrics collection | Low | — |
| 10 | **Phase 1: Ideate** — Backlog scoring | Low | — |

---

## Changelog

- 2026-03-16: Initial spec. Architecture: Платон + Brainstorm + Architect + Ops + AO. 9 phases, label state machine, AI↔AI brainstorm.
- 2026-03-16: **Revised** — replaced Империя agents (Сократ/Архимед/Аристотель/Геродот) with Платон ecosystem agents. Research → Brainstorm (web_search). Dev orchestration → AO (Agent Orchestrator) via Ops. Aligned with actual Platon architecture.
