# ISET Group — платформа управления проектами

В репозитории находится full‑stack система управления проектами: фронтенд на React + Vite и бэкенд на NestJS + PostgreSQL.

## Стек

- Фронтенд: React, Vite, TypeScript, Tailwind, shadcn/ui
- Бэкенд: NestJS, TypeScript, PostgreSQL
- Аутентификация: JWT (access + refresh)

## Структура репозитория

- `backend/` — NestJS API, миграции, сиды
- `work-stream-alchemy-main/` — Vite React UI
- `artifacts/` — QA‑скриншоты и снапшоты (опционально)
- `docker-compose.yml` - запуск полного стека (Postgres + API + UI)

## Требования

- Node.js 18+ (и npm)
- PostgreSQL 16 (или Docker)
- Docker + Docker Compose (для контейнерного запуска)

## Переменные окружения

### Бэкенд (`backend/.env`)

Создайте `backend/.env`:

```
DATABASE_URL=postgres://postgres:postgres@localhost:5432/iset
JWT_ACCESS_SECRET=replace-with-strong-secret
JWT_REFRESH_SECRET=replace-with-strong-secret
JWT_ACCESS_TTL=15m
JWT_REFRESH_TTL=7d
CORS_ORIGINS=http://localhost:5173,http://localhost
PORT=4000
```

Примечания:
- `DATABASE_URL` обязателен.
- `JWT_ACCESS_TTL` и `JWT_REFRESH_TTL` необязательны.
- `PORT` по умолчанию `4000`.
- `CORS_ORIGINS` - список разрешенных origin через запятую.
- `BOOTSTRAP_ADMIN_EMAIL` и `BOOTSTRAP_ADMIN_NAME` - параметры первичного admin (опционально).

### Фронтенд (`work-stream-alchemy-main/.env`)

Создайте `work-stream-alchemy-main/.env`, если нужен нестандартный API URL:

```
VITE_API_URL=http://localhost:4000
```

Если не задано, используется `http://localhost:4000`.

## Запуск через Docker (локально или на сервере)

1) Создайте `backend/.env` по образцу `backend/.env.example` и задайте реальные `JWT_*` и `CORS_ORIGINS`.

2) Запустите весь стек:

```
docker compose up -d --build
```

3) Откройте UI:

- http://localhost:8080

Примечания:
- Внешний порт UI: `HTTP_PORT` (по умолчанию `8080`).
- Порт БД на хосте: `DB_PORT` (по умолчанию `5432`).
- API доступен через `/api` на том же домене/порту, CORS настраивается через `CORS_ORIGINS`.
- Бэкапы БД выполняются автоматически сервисом `db-backup` в volume `iset_backups`.
- Параметры бэкапа: `BACKUP_INTERVAL_SECONDS` (по умолчанию 86400), `BACKUP_RETENTION_DAYS` (по умолчанию 7).
- При первом запуске, если в БД нет пользователей, создается bootstrap admin. Пароль выводится в логи backend.

On-prem запуск: см. `ONPREM-RUNBOOK.md`.

## Быстрый старт (локально без Docker)

1) Запустить PostgreSQL через Docker (опционально):

```
docker compose up -d db
```

2) Установить зависимости бэкенда:

```
cd backend
npm install
```

3) Применить миграции и сиды:

```
npm run migrate:up
npm run seed
```

4) Запустить API:

```
npm run start:dev
```

5) В новом терминале установить зависимости фронтенда и запустить UI:

```
cd work-stream-alchemy-main
npm install
npm run dev
```

6) Открыть UI:

- http://localhost:5173

## Логины по умолчанию (seed)

- admin@iset.local / admin123
- pm@iset.local / pm123
- executor1@iset.local / exec123
- executor2@iset.local / exec123
- accountant@iset.local / acc123

## Production сборка

### Бэкенд

```
cd backend
npm install
npm run build
npm run start
```

### Фронтенд

```
cd work-stream-alchemy-main
npm install
npm run build
npm run preview
```

`npm run preview` поднимает локальный статический сервер. Для продакшена отдавайте `work-stream-alchemy-main/dist` через ваш веб‑сервер.

## Полезные команды

Бэкенд (`backend/`):

- `npm run start:dev` — запуск API в dev‑режиме
- `npm run build` — сборка в `dist/`
- `npm run start` — запуск собранного API
- `npm run migrate:up` — применить миграции
- `npm run migrate:down` — откатить последнюю миграцию
- `npm run seed` — заполнить демо‑данными

Фронтенд (`work-stream-alchemy-main/`):

- `npm run dev` — Vite dev‑сервер
- `npm run build` — production‑сборка
- `npm run preview` — локальный preview сборки
- `npm run lint` — ESLint

## Частые проблемы

- `DATABASE_URL is required` — убедитесь, что есть `backend/.env` и запущен Postgres.
- `EADDRINUSE` — порт занят; остановите процесс или поменяйте `PORT` / порт Vite.
- `Seed skipped: users already exist` — сиды выполняются только если таблица users пустая.
- `401 Unauthorized` в UI — проверьте JWT‑секреты и перелогиньтесь.

## QA артефакты

Если проводите UI‑проверки, сохраняйте скриншоты/снапшоты в `artifacts/` для удобства отчетности.
