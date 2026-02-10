# Проектирование задач 1–19 (дизайн-док)

## 0. Контекст и текущее состояние
- Backend: NestJS + PostgreSQL, SQL напрямую, миграции через node-pg-migrate.
- Роли (app_role): admin, project_manager, executor, accountant. В БД уже есть lead_specialist, но фронт/ACL его не используют.
- Ключевые таблицы: departments, users, user_roles, projects, project_members, project_sections, tasks, time_entries, day_offs, contractors.
- Ограничения табеля: time_entries.hours in (0.5, 1), дневная сумма <= 1, запрет списания на done-задачи (триггеры).
- Задачи сейчас требуют project_id; задачи WBS создаются триггером из project_sections.
- UI: /projects, /projects/:id (tabs), /tasks (канбан), /time-tracking (табель), /organization (структура и контрагенты).

## 1. Кластеры и зависимости (кратко)
- К1 Роли и орг-структура (1, 6, 10, 12, 14).
- К2 Типы задач: личные/бухгалтерские/проектные (4, 13).
- К3 Учет времени и ставки (2, 3, 5, 11, 17).
- К4 Субподряд и согласование (7, 8, 9).
- К5 WBS и Гантт (15, 19).
- К6 UX консистентность (16, 18).

## 2. К1 Роли и орг-структура (1, 6, 10, 12, 14)

### 2.1 Сущности и схема данных
- app_role: добавить значения `department_head`, `approval_admin` (lead_specialist уже есть).
- user_departments (новая таблица):
  - id (uuid), user_id, department_id, is_primary (bool), created_at.
  - unique(user_id, department_id), index(user_id), index(department_id).
- users.department_id:
  - оставить как «основное подразделение» для обратной совместимости;
  - синхронизировать с user_departments.is_primary.
- projects.department_id (uuid, nullable): «владельческое» подразделение проекта.

### 2.2 Миграции (минимальные)
- ALTER TYPE app_role: добавить `department_head`, `approval_admin`.
- Создать user_departments + backfill из users.department_id.
- Добавить projects.department_id + индекс.
- Проверка для подрядчиков: contractor_id и user_departments несовместимы (валидация в сервисе + CHECK при необходимости).

### 2.3 ACL и правила доступа
- lead_specialist (Глав спец):
  - может создавать/редактировать задачи внутри проектов, где он участник;
  - может назначать других исполнителей;
  - не видит бюджет проекта и ставки сотрудников; нет доступа к орг-структуре.
- department_head:
  - полномочия администратора, но только в пределах своих подразделений;
  - доступ к пользователям/проектам/задачам/табелям только по пересечению департаментов.
- admin:
  - глобальный доступ без ограничений по подразделениям.

### 2.4 API и валидации
- /users:
  - принять/возвращать departmentIds[] и primaryDepartmentId;
  - фильтры: /users?departmentId=..., /users?contractorId=...;
  - запрет contractorId + departmentIds одновременно.
- /projects:
  - поддержать departmentId при создании/обновлении;
  - фильтр /projects?departmentId=...;
  - при отсутствии departmentId — оставить null.
- /departments:
  - department_head получает только свои подразделения;
  - admin/accountant видят все (как сейчас).

### 2.5 UI/навигация/сценарии
- Role mapping: добавить lead_specialist, department_head, approval_admin в Layout/RoleContext/RequireRole.
- Organization:
  - мультивыбор подразделений для пользователя + выбор «основного»;
  - роль department_head в списке ролей.
- Projects:
  - фильтр по подразделениям;
  - дефолтный фильтр = primaryDepartmentId (если есть), иначе «Все».
- ProjectTeam:
  - фильтры в диалоге «Добавить участника»: Подразделение и Контрагент;
  - для пользователя с несколькими департаментами показывать, если он входит хотя бы в один выбранный.
- ProjectProperties / CreateProjectDialog:
  - поле «Подразделение проекта»;
  - бюджет скрывать для lead_specialist.

### 2.6 Риски/краевые случаи
- Старые проекты без department_id: нужен backfill (например, по manager.primaryDepartmentId).
- Пользователь без подразделения: показывать все проекты, но фильтр доступен.
- Подрядчики в департаментах: должна быть жесткая валидация.

## 3. К2 Типы задач: личные/бухгалтерские/проектные (4, 13)

### 3.1 Сущности и схема данных
- task_type (enum): `project`, `personal`, `accounting`, `project_time`, `subcontract`.
- tasks.project_id: сделать nullable.
- tasks.created_by (uuid, not null) — кто создал.
- tasks.owner_id (uuid, nullable) — владелец (обычно = assignee для personal).
- tasks.department_id (uuid, nullable) — для ACL department_head и фильтров (берется из owner/creator).
- CHECK: если section_id задан, то task_type = project и project_id NOT NULL.

### 3.2 Миграции
- Создать enum task_type, добавить колонки task_type, created_by, owner_id, department_id.
- Ослабить NOT NULL на tasks.project_id.
- Backfill:
  - существующие задачи: task_type = project;
  - created_by = manager_id проекта (если есть) или NULL/system;
  - department_id = projects.department_id (если есть).
- Обновить триггер sync_task_for_section: при создании задач для WBS выставлять task_type=project.

### 3.3 ACL и правила доступа
- personal: видят только owner/assignee, creator и admin/department_head (в своем департаменте).
- accounting: видят assignee (бухгалтер), creator, admin/department_head.
- project: доступ по правилам проекта (как сейчас).
- lead_specialist может создавать accounting задачи, но без доступа к бюджету проекта.

### 3.4 API и валидации
- POST /tasks:
  - поддержать taskType, projectId (nullable), ownerId, assigneeId, departmentId.
  - валидация комбинаций (sectionId -> project task; personal -> assignee=owner).
- GET /tasks:
  - фильтры: taskType, includePersonal, includeAccounting, departmentId.
  - спец-флаг forTimeTracking=true для возврата «служебных» задач (см. К3).
- GET /projects/:id/tasks:
  - отдавать только project/subcontract задачи.

### 3.5 UI/навигация/сценарии
- /tasks доступен для accountant (доска его задач).
- CreateTaskDialog:
  - режим «Вне проекта» (personal/accounting);
  - для accounting — выбор бухгалтера с отдельным шильдиком в списке.
- Канбан:
  - фильтр по типу задач (Проектные / Личные / Бухгалтерские);
  - визуальный бейдж «Бухгалтер» у задач, назначенных на role=accountant.

### 3.6 Риски/краевые случаи
- Личные задачи не должны попадать в прогресс проекта и аналитику.
- Разграничить права на переassign личных задач (по умолчанию только creator/admin).

## 4. К3 Учет времени и ставки (2, 3, 5, 11, 17)

### 4.1 Сущности и схема данных
- time_entries:
  - убрать CHECK hours IN (0.5, 1);
  - убрать дневной лимит в триггере;
  - сохранить уникальность (user_id, work_date, task_id).
- Разрешить списание на done-задачи (убрать блок в validate_time_entry).
- project_time задачи для ГИПа: task_type=project_time с уникальностью (project_id, assignee_id, task_type).
- Ставка админа: использовать users.daily_rate (уже есть), показывать и учитывать в табеле.

### 4.2 Миграции
- Удалить time_entries_hours_check.
- Обновить функцию validate_time_entry:
  - убрать дневной лимит;
  - не запрещать status=done;
  - если task.status = new — переводить в in_progress только если не done.
- Добавить уникальный индекс для project_time задач (partial index).

### 4.3 ACL и правила доступа
- Любая роль может списывать время на свои задачи (включая admin/accountant).
- department_head видит табели пользователей своего департамента.
- project_manager: read табели участников проекта (как сейчас).

### 4.4 API и валидации
- /time-tracking/entries:
  - допускаются любые положительные значения (или фикс 1 для «+»).
- /tasks?forTimeTracking=true:
  - возвращать project + personal + accounting + project_time задачи для текущего пользователя.

### 4.5 UI/навигация/сценарии
- Табель (/time-tracking):
  - заменить 0.5/1 на «+»;
  - разрешить несколько задач в один день;
  - обновить легенду и итоговые подсчеты.
- Админ-табель:
  - в Organization добавить вкладку «Админы» или открыть табель админов в общем списке;
  - разрешить TimeTrackingDialog для admin.
- ГИП:
  - в табеле показывать «задачи проекта» (project_time) для проектов, где он менеджер, даже без назначенных задач.

### 4.6 Риски/краевые случаи
- Стоимость по дням:
  - при списании на несколько задач в один день стоимость может «размножаться»;
  - рекомендуем считать стоимость по уникальным work_date (для payroll), а распределение по задачам — отдельной метрикой.
- Исторические 0.5/1: требуется решение — мигрировать или оставить как есть.

## 5. К4 Субподряд и согласование (7, 8, 9)

### 5.1 Сущности и схема данных
- tasks.task_type = subcontract.
- tasks.approval_status (enum: pending, approved, rejected).
- tasks.subcontract_cost_requested, tasks.subcontract_cost_final (numeric).
- task_approvals (новая таблица):
  - id, task_id, approver_id, decision (approved/rejected), final_cost, decided_at.
  - unique(task_id, approver_id), index(task_id).

### 5.2 Миграции
- Добавить enum approval_status и поля в tasks.
- Создать task_approvals + индексы.

### 5.3 ACL и правила доступа
- project_manager:
  - может создавать subcontract задачи для подрядчика;
  - видит pending задачи, но они недоступны исполнителю.
- approval_admin:
  - видит заявки, может согласовать/отклонить, указать финальную цену.
- executor:
  - видит subcontract задачи только со статусом approved.

### 5.4 API и валидации
- POST /tasks (taskType=subcontract):
  - обязательны assigneeId (подрядчик) и subcontract_cost_requested;
  - approval_status = pending.
- GET /subcontracts/requests:
  - доступ только approval_admin; фильтр по status.
- POST /subcontracts/:taskId/approve:
  - сохраняет approval + final_cost;
  - при 2 approvals → approval_status=approved и subcontract_cost_final заполняется.
- POST /subcontracts/:taskId/reject:
  - status=rejected, задача скрывается из общих списков.

### 5.5 UI/навигация/сценарии
- Создание субподрядной задачи (ГИП):
  - переключатель «Субподряд» + поле стоимости;
  - карточка серым, статус «Ожидает согласования».
- Раздел «Заявки» (approval_admin):
  - список заявок с проектом, исполнителем и предложенной ценой;
  - действия «Согласовать/Отклонить», поле финальной цены.
- Табель:
  - отдельная вкладка «Субподряд» с approved задачами.

### 5.6 Риски/краевые случаи
- Двойные согласования: требовать 2 разных approver_id.
- Изменение цены после первого согласования: определить политику (фиксировать или требовать пересогласование).

## 6. К5 WBS и Гантт (15, 19)

### 6.1 Сущности и схема данных
- project_sections:
  - добавить parent_id (uuid, nullable), level (int), section_path (int[] для сортировки);
  - глубина ограничена 3 уровнями (level <= 2).

### 6.2 Миграции
- Добавить parent_id, level, section_path + индексы.
- Backfill:
  - парсить section_code ("1.2.3") → section_path [1,2,3], level=2.
  - некорректные коды отправлять в конец сортировки.

### 6.3 API и валидации
- /projects/:id/sections:
  - сортировка по section_path, затем section_code;
  - валидация: depth <= 3.

### 6.4 UI/навигация/сценарии
- ProjectSectionsTable:
  - автосортировка подразделов (1.1, 1.2, 1.10);
  - запрет добавления 4-го уровня (ошибка/disable).
- Gantt:
  - синхронизировать ширину колонок таймлайна для заголовка и строк;
  - фиксировать вычисление chartStart/chartEnd, чтобы шкала не ломалась на weeks/months;
  - исправить кликабельность баров (факт) и подгонку по ширине.

### 6.5 Риски/краевые случаи
- section_code с буквами/пробелами ломает сортировку — нужен fallback.

## 7. К6 UX консистентность (16, 18)

### 7.1 UI/поведение
- Confirm dialogs «Удалить?»:
  - ProjectTeam (удаление участника), ProjectSectionsTable (удаление раздела), ProjectFiles, Organization/ContractorDetails и др.
- Required asterisk:
  - отметить обязательные поля в формах, где есть необязательные (Users, Projects, Contractors, Tasks, Departments).
  - использовать единый компонент/паттерн (Label + "*").

### 7.2 Риски
- Диалоги на мобильных: соблюдать текущие паттерны shadcn/ui, не менять механику закрытия.

## 8. Порядок внедрения (шаги)
1) К1 Роли и орг-структура (ACL + multi-department + фильтры проектов).
2) К2 Типы задач (personal/accounting) + обновление задач/видимости.
3) К3 Табель (переход на «+», admin-табель, project_time задачи, done-задачи).
4) К4 Субподряд + заявки + вкладка табеля.
5) К5 WBS сортировка/глубина + фиксы Ганта.
6) К6 UX: подтверждения и звездочки.

## 9. Матрица зависимостей
| Кластер | Зависит от | Комментарий |
| --- | --- | --- |
| К1 Роли и орг-структура | - | База для ACL и фильтров |
| К2 Типы задач | К1 | Нужны роли и департаменты |
| К3 Учет времени и ставки | К1, К2 | Типы задач и ACL |
| К4 Субподряд и согласование | К1, К3 | approval_admin и вкладка табеля |
| К5 WBS и Гантт | - | Независимо от ролей |
| К6 UX консистентность | - | Лучше после основных экранов |

## 10. Список файлов, которые вероятно придется менять

### Backend
- `backend/migrations/*.js` (новые миграции по ролям, user_departments, projects.department_id, tasks/task_type, approvals, time_entries).
- `backend/src/common/auth/auth.types.ts` (новые роли).
- `backend/src/auth/auth.service.ts` (типизация ролей).
- `backend/src/users/users.service.ts` (multi-department, ACL, фильтры).
- `backend/src/departments/*` (department_head scoping).
- `backend/src/projects/projects.service.ts` (departmentId, фильтры, ACL).
- `backend/src/tasks/tasks.service.ts` (task_type, personal/accounting, subcontract, approvals, ACL).
- `backend/src/time-tracking/time-tracking.service.ts` (новые правила табеля, done-задачи, project_time).
- `backend/src/seed.ts` (seed новых ролей/полей).

### Frontend
- `work-stream-alchemy-main/src/App.tsx` (маршруты: заявки, админ-табель).
- `work-stream-alchemy-main/src/components/Layout.tsx` (nav + новые роли).
- `work-stream-alchemy-main/src/lib/routes.ts` (redirect по ролям).
- `work-stream-alchemy-main/src/pages/Projects.tsx` (фильтр по подразделениям).
- `work-stream-alchemy-main/src/components/project/CreateProjectDialog.tsx` (departmentId, required).
- `work-stream-alchemy-main/src/components/project/ProjectProperties.tsx` (departmentId, бюджет для lead_specialist).
- `work-stream-alchemy-main/src/components/project/ProjectTeam.tsx` (фильтры + confirm delete).
- `work-stream-alchemy-main/src/pages/Tasks.tsx` (фильтры типов, доступ бухгалтера).
- `work-stream-alchemy-main/src/components/project/ProjectTasks.tsx` (бейдж бухгалтер, фильтры типов).
- `work-stream-alchemy-main/src/components/tasks/*` (task_type, бухгалтерский шильдик).
- `work-stream-alchemy-main/src/pages/TimeTracking.tsx` + `work-stream-alchemy-main/src/components/TimeTrackingDialog.tsx` (логика «+», вкладка субподряда, admin-табель).
- `work-stream-alchemy-main/src/pages/Organization.tsx` (multi-department, роли, табель админов, required).
- `work-stream-alchemy-main/src/pages/ContractorDetails.tsx` (required/confirm).
- `work-stream-alchemy-main/src/components/project/ProjectSectionsTable.tsx` (глубина, сортировка, confirm).
- `work-stream-alchemy-main/src/components/project/ProjectGantt.tsx` (верстка/таймлайн).
- `work-stream-alchemy-main/src/components/RoleSwitcher.tsx` (новые роли в демо).

## 11. UX/конфликты для пересмотра
- Стоимость при «плюс-табеле»: нужна договоренность, как считать cost при нескольких задачах в день.
- Личные vs проектные задачи: где показывать по умолчанию (канбан/табель).
- Subcontract approval: политика финальной цены после первого согласования.
