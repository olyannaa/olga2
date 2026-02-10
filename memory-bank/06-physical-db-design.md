# Физическая модель БД (PostgreSQL)

## Таблицы (ключевые поля)
- users: id (uuid), full_name, email, password_hash, department_id, contractor_id, can_approve_subcontracts, daily_rate, contract_rate, must_change_password, created_at.
- user_roles: user_id, role, UNIQUE(user_id, role).
- user_departments: user_id, department_id, is_primary, UNIQUE(user_id, department_id).
- departments: id, name, description.
- contractors: id, name, inn, deleted_at.
- projects: id, name, manager_id, department_id, budget, status, start_date, end_date, organization, external_link, deleted_at.
- project_members: project_id, user_id, project_role, UNIQUE(project_id, user_id).
- project_sections: project_id, section_code, designation, section_name, start_date, planned_end_date, executor_id, actual_end_date, notes.
- tasks: project_id (nullable), section_id (nullable), title, status, task_type, assignee_id, created_by, owner_id, approval_status, subcontract_cost_requested, subcontract_cost_final, planned_start_date, planned_end_date.
- task_status_history: task_id, from_status, to_status, changed_by, changed_at.
- task_approvals: task_id, approver_id, decision, final_cost, decided_at, UNIQUE(task_id, approver_id).
- time_entries: user_id, task_id, work_date, hours, UNIQUE(user_id, work_date, task_id).
- day_offs: user_id, work_date, type, UNIQUE(user_id, work_date).
- work_calendar: work_date, is_holiday, comment (резерв).
- time_sheets: user_id, period_start, period_end (резерв).
- project_files: project_id, task_id, file_name, file_url, description, user_id.
- project_folders: project_id, nextcloud_path, last_sync_at, last_sync_status.
- chat_messages: project_id, user_id, message, created_at.
- audit_log: actor_id, entity, action, payload_json, created_at.

## Ключевые ограничения и триггеры
- users: запрет одновременного `contractor_id` и `department_id`.
- tasks: `section_id` допускается только для `task_type IN ('project','subcontract')`.
- project_time задачи: уникальность `(project_id, assignee_id, task_type)` при `task_type='project_time'`.
- project_sections → tasks:
  - триггер создаёт/обновляет задачу при вставке/изменении раздела.
- tasks:
  - триггер запрещает откат в `new`, если есть time entries;
  - проставляет `actual_start_date` при переходе в `in_progress`;
  - проставляет `actual_end_date` при `done`.
- time_entries:
  - триггер блокирует запись, если есть day‑off на дату;
  - при первой записи переводит задачу в `in_progress`.

## Индексы (основные)
- projects: status, manager_id, department_id, deleted_at.
- project_sections: project_id, executor_id.
- tasks: project_id, status, assignee_id, section_id, department_id.
- user_departments: user_id, department_id.
- project_members: project_id, user_id.

## Enum‑типы
- app_role: admin, gip, executor, accountant (legacy роли остаются в enum, но не используются).
- project_role: manager, lead_specialist, executor, accountant.
- task_status: new, in_progress, review, done.
- task_type: project, personal, accounting, project_time, subcontract.
- approval_status: pending, approved, rejected.
- project_status: active, completed, archived.
- day_status: b, o, k, v.
