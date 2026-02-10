# On-Prem Runbook (VM)

## 0) Что нужно заранее
- IP/VM доступ по SSH и sudo.
- Решение по домену и порту (обычно 80/443 или отдельный порт).
- Docker и Docker Compose на VM.

## 1) Подготовка VM
1) Обновить систему и установить Docker:
   - Ubuntu/Debian: `sudo apt update && sudo apt install -y docker.io docker-compose-plugin`
2) Добавить пользователя в группу docker:
   - `sudo usermod -aG docker $USER` и перелогиниться.
3) Открыть только нужный порт для UI (80/443 или `HTTP_PORT`).
4) Закрыть внешний доступ к Postgres (не публиковать 5432 наружу).

## 2) Загрузка проекта
1) Склонировать репозиторий:
   - `git clone <repo_url> /opt/iset-group`
2) Перейти в каталог:
   - `cd /opt/iset-group`

## 3) Конфигурация окружения
1) Создать `backend/.env` по образцу `backend/.env.example`.
2) Заполнить как минимум:
   - `JWT_ACCESS_SECRET` и `JWT_REFRESH_SECRET` (сильные значения).
   - `CORS_ORIGINS=https://your-domain.tld` (можно несколько через запятую).
   - `PORT=4000` (обычно оставить).
   - `BOOTSTRAP_ADMIN_EMAIL` и `BOOTSTRAP_ADMIN_NAME` (если хотите свои).
3) Для on-prem можно задать переменные запуска:
   - `HTTP_PORT=80` (или 8080).
   - `POSTGRES_PASSWORD=<strong_password>`.
   - `BACKUP_INTERVAL_SECONDS=86400` (по умолчанию раз в сутки).
   - `BACKUP_RETENTION_DAYS=7` (по умолчанию хранить 7 дней).

## 4) Запуск стека
1) Собрать и поднять контейнеры:
   - `HTTP_PORT=80 POSTGRES_PASSWORD=... docker compose up -d --build`
2) Проверить состояние:
   - `docker compose ps`
3) Посмотреть логи:
   - `docker compose logs -f backend`
4) Найти в логах строку с `Bootstrap admin created` и сохранить пароль.

## 5) Первый вход
- Войти под bootstrap admin и сменить пароль в разделе организации.
- Если нужен стартовый набор пользователей, выполните на пустой БД:
  - `docker compose exec backend npm run seed`
  - Примечание: seed пропускается, если в таблице users уже есть записи.
- После этого смените пароли в UI или напрямую в БД.

## 6) Бэкапы БД (автоматически)
- Сервис `db-backup` делает `pg_dump` в `/backups` раз в сутки.
- Проверка наличия бэкапов:
  - `docker compose exec db-backup ls -lh /backups`
- Каталог с бэкапами хранится в volume `iset_backups`.

## 7) Восстановление БД (пример)
1) Остановить сервисы, кроме db:
   - `docker compose stop backend frontend db-backup`
2) Восстановить из дампа:
   - `docker compose exec -T db pg_restore -U ${POSTGRES_USER:-postgres} -d ${POSTGRES_DB:-iset} --clean /backups/<file>.dump`
3) Запустить сервисы обратно:
   - `docker compose up -d`

## 8) Рекомендованные проверки после запуска
- UI открывается по `http://<host>:<HTTP_PORT>`.
- API отвечает по `/api/health` (если добавите healthcheck) или базовые эндпойнты.
- В `backend` нет ошибок миграций.
- Бэкап создается в течение первых суток (или уменьшить интервал для теста).

## 9) TLS и домен
- Если нужен HTTPS, поставьте reverse-proxy (Nginx/Caddy) на VM и проксируйте на `HTTP_PORT`.
- Убедитесь, что `CORS_ORIGINS` соответствует внешнему домену.
