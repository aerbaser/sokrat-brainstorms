# 🧠 Agent Health Monitor

| | |
|---|---|
| Дата | 2026-03-16 |
| Тип | Feature |
| Режим | Single (Claude) |
| Статус | 🔮 Draft |

## Summary
Система мониторинга здоровья агентов Империи (main/Сократ, Архимед, Аристотель, Геродот + Платон). Двухслойная архитектура: shell watchdog (cron, независим от OpenClaw) + OpenClaw skill (расширенная аналитика). Интеграция в существующий Brain dashboard на localhost:3333. Алерты в TG топик + личку для критичных.

## Документы
- [Spec](spec.md)
