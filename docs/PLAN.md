# docs/PLAN.md — fayna-industrial-iot

## Фаза 0 — Extract & sanitize ✅

Витяг Industrial IoT Bridge з внутрішнього проєкту, знеособлення (клієнт-специфічна
логіка, облікові дані та карти регістрів виключені), приведення до REPO_STANDARD,
fresh git history без AI-підписів. (Процедура Project Sunset.)

## Фаза 1 — Senior-grade pass ✅

- Пакетна структура `src/bridge` (main/machine_tester/scanner/odoo_api/db_handler/plc_simulator).
- Повна типізація публічних функцій, специфічні `except`, `logging` у бібліотечному коді.
- Env-var driven конфігурація (`config/settings.py`) — без секретів у коді.
- `pytest` юніт-тести без фізичного заліза та без живого Odoo; CI (ruff + gitleaks + mypy + pytest).

## Фаза 2 — Demo hardening

- End-to-end smoke-прогін з бандл-симулятором: `plc_simulator` → `main` (demo mode)
  → SQLite-буфер, без фізичного заліза та без живого Odoo.
- README-скріншот/GIF демо-сесії (лог мосту + показання регістрів).

## Фаза 3 — Покриття тестами

- Мок Modbus-клієнта → тест `scanner.read_machine_state()` end-to-end.
- Мок Odoo XML-RPC → тест `odoo_api` + SQLite-буфер (черга при недоступному Odoo).
- Coverage-репорт, ціль ≥70% критичного шляху (scanner + db_handler + odoo_api).

## Checkpoints

- Кінець Фази 1 → готове до публічного портфоліо-показу.
- Кінець Фази 3 → integration-шлях покритий без живого заліза/Odoo в CI.
