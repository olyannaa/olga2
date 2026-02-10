# Повторная QA-проверка кластеров (K1-K4) - 20.01.2026

## Статус
- K1: Пройдено; ACL и API-проверки выполнены (см. сценарии/риски).
- K2: Пройдено; задачи бухгалтера и фильтры типов проверены.
- K3: Пройдено; UI/легенда табеля и API проверены, строки project_time присутствуют.
- K4: Пройдено; действия согласования и фильтры проверены.

## Внесенные исправления
- Логин и API-запросы теперь идут через /api по VITE_API_URL в work-stream-alchemy-main/.env (dev-переопределение в work-stream-alchemy-main/.env.development.local).
- /approvals: исправлены сводка и фильтры (frontend) + поддержка status=all (backend).
- /projects/:id/tasks: диалог создания задачи показывает текущее имя проекта (frontend).
- /time-tracking: вкладка субподряда строит список проектов с фолбэком из задач (frontend).
- /approvals approve/reject: повторные согласования одним аппрувером больше не дают 500 (backend).

## Найденные проблемы
1) [Критично][Исправлено] Логин не отправлял /api/auth/login.
   Шаги: /login -> ввести admin@iset.local/admin123 -> Войти.
   Ожидание: POST /api/auth/login, сохранение токенов, редирект.
   Факт (до фикса): запрос не уходил в /api, редиректа не было.

2) [Высокая][Исправлено] Создание задач в /tasks не отправляло POST /api/tasks.
   Шаги: /tasks -> "Новая задача" -> ввести название -> "Создать".
   Ожидание: POST /api/tasks, карточка появляется в колонке.
   Факт (до фикса): запрос отсутствовал / форма не сохраняла.

3) [Высокая][Исправлено] "Согласовать" в /approvals не отправлял POST /api/subcontracts/:id/approve.
   Шаги: /approvals -> нажать "Согласовать" для pending заявки.
   Ожидание: POST /api/subcontracts/:id/approve, UI обновляется.
   Факт (до фикса): запрос отсутствовал / статус не менялся.

4) [Средняя][Исправлено] /approvals сводка/фильтр не показывали согласованные заявки.
   Шаги: /approvals -> фильтр "Согласованные".
   Ожидание: список согласованных, корректные счетчики.
   Факт (до фикса): счетчики 0, список пустой.

5) [Средняя][Исправлено] Диалог проектной задачи показывал "Проект: -".
   Шаги: Project Alpha -> вкладка "Задачи" -> "Новая задача".
   Ожидание: отображается имя проекта.
   Факт (до фикса): "Проект: -".

6) [Средняя][Исправлено] /time-tracking вкладка "Субподряд" не показывала задачи.
   Шаги: contractor1 -> /time-tracking -> вкладка "Субподряд".
   Ожидание: субподрядные задачи отображаются.
   Факт (до фикса): пустой/не отображающийся список.

7) [Средняя][Исправлено] Повторное согласование одним аппрувером давало 500 (duplicate key task_approvals_unique).
   Шаги: /approvals -> нажать "Согласовать" дважды тем же пользователем.
   Ожидание: идемпотентная обработка или ясная ошибка без 500.
   Факт (до фикса): 500 Internal Server Error.

## Проверенные сценарии (K1-K4)
- K1: /organization UI для admin/accountant/department_head; отделы и сотрудники видны; ACL-поведение проверено.
- K1: API /users?departmentId и /users?contractorId возвращают ожидаемых пользователей.
- K1: PATCH /users для contractor + department возвращает 400 "Contractor users cannot have departments".
- K1: Логин использует /api/auth/login и сохраняет токены.
- K2: /tasks для бухгалтера показывает бухгалтерские задачи (QA Accounting 1); фильтр типов: Проектные/Личные/Бухгалтерские/Субподряд.
- K2: /projects/:id/tasks не содержит personal/accounting задачи.
- K2: Создание личной задачи в /tasks (QA Personal 5).
- K3: /time-tracking (admin) — легенда видна; time-tracking API содержит несколько задач в один день, часы > 1, и записи для done-задач.
- K3: PM видит строки "Project management" в табеле.
- K3: contractor1 — вкладка "Субподряд" показывает субподрядные задачи.
- K4: /approvals согласование (QA Subcontract 3), фильтр "Согласованные" показывает список.
- K4: /approvals отклонение (QA Subcontract 4), фильтр "Отклоненные" показывает список.
- K4: PM видит субподрядные задачи с контрагентом/статусом; contractor1 не видит pending субподряд в /tasks.

## Тестовые данные
- Создано: QA Personal 5 (личная задача).
- Создано: QA Subcontract 4 (субподряд, 25 000 ₽, contractor1) -> отклонено.
- Обновлено: QA Subcontract 3 -> согласовано (2 согласования).

## Риски/неопределенности
- lead_specialist доступ к /organization: UI редиректил на /projects. Если нужен частичный доступ, требуется уточнение.
- /time-tracking вкладка "Субподряд": в автоматизации переключение подтверждено через Enter; клик мышью нужно перепроверить вручную.

## Команды
- docker-compose down
- docker-compose up -d --build
- docker-compose up -d --build backend
- docker-compose logs --tail=200 backend
- docker-compose exec -T db psql -U postgres -d iset -c "SELECT COUNT(*) AS projects FROM projects;"
- docker-compose exec -T db psql -U postgres -d iset -c "SELECT id, name FROM projects ORDER BY created_at;"
- docker-compose exec -T db psql -U postgres -d iset -c "SELECT name, status FROM projects;"
- docker-compose exec -T db psql -U postgres -d iset -c "SELECT name, department_id FROM projects;"
- docker-compose exec -T db psql -U postgres -d iset -c "SELECT id, title, task_type, status, project_id, approval_status, subcontract_cost_requested, subcontract_cost_final FROM tasks ORDER BY created_at;"
- docker-compose exec -T db psql -U postgres -d iset -c "SELECT email, array_agg(r.role ORDER BY r.role) AS roles FROM users u LEFT JOIN user_roles r ON r.user_id = u.id GROUP BY u.email ORDER BY u.email;"
- docker-compose exec -T db psql -U postgres -d iset -c "SELECT work_date, task_id, hours FROM time_entries ORDER BY work_date, task_id;"
