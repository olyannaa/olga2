# Архитектура (текущее состояние)

## Общая схема
- Web UI: React + Vite SPA.
- Backend API: NestJS (REST).
- PostgreSQL: основная БД с миграциями node-pg-migrate.
- Аудит: записи в `audit_log`.
- Интеграции: Nextcloud и хранение файлов — пока заглушки (метаданные + `project_folders`).

## Слой приложений
- UI обращается к API напрямую (база URL из `VITE_API_URL`, по умолчанию `http://localhost:4000`).
- API покрывает модули: auth, users, departments, projects, tasks, time-tracking, contractors.
- Реалтайм не используется; чат работает через REST‑запросы.

## Слой данных
- Триггеры БД:
  - синхронизация задачи с WBS‑разделом;
  - запрет отката статуса задачи при наличии time entries;
  - проверка day‑off и авто‑перевод задачи в `in_progress` при записи времени.

## Развертывание (on‑prem)
- Docker Compose поднимает Postgres + API + UI.
- Проектные файлы — ссылки/URL; Nextcloud подключается позже.
