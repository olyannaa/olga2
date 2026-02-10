# Логическая модель данных

## Основные сущности и связи
- User 1..N UserRole.
- User 1..N UserDepartment (многие подразделения + primary).
- Department 1..N Users (через UserDepartment).
- Contractor 1..N Users (контрагентские исполнители).
- Project 1..N ProjectSection.
- Project 1..N ProjectMember (Project ↔ User + роль в проекте).
- ProjectSection 1..1 Task (тип `project`, создаётся триггером).
- Task 1..N TaskStatusHistory.
- Task 1..N TimeEntry.
- Task 0..N TaskApproval (для субподрядов).
- Project 1..N Task (ручные проектные/субподрядные).
- User 1..N TimeEntry и 1..N DayOff.
- Project 1..N ProjectFile и 1..N ChatMessage.
- Project 1..1 ProjectFolder (stub).
- User 1..N AuditLog.

## Сущности (актуальный состав полей)
- User: id, full_name, email, password_hash, daily_rate, contract_rate, contractor_id, department_id (primary), can_approve_subcontracts.
- UserRole: user_id, role.
- UserDepartment: user_id, department_id, is_primary.
- Contractor: id, name, inn, deleted_at.
- Department: id, name, description.
- Project: id, name, description, manager_id, organization, external_link, budget, status, start_date, end_date, department_id, deleted_at.
- ProjectMember: project_id, user_id, project_role.
- ProjectSection: project_id, section_code, designation, section_name, start_date, planned_end_date, executor_id, actual_end_date, notes.
- Task: id, project_id (nullable), section_id (nullable), title, status, task_type, assignee_id, created_by, owner_id, approval_status, subcontract_cost_requested, subcontract_cost_final, planned_start_date, planned_end_date.
- TaskApproval: task_id, approver_id, decision, final_cost.
- TaskStatusHistory: task_id, from_status, to_status, changed_by.
- TimeEntry: user_id, task_id, work_date, hours.
- DayOff: user_id, work_date, type.
- WorkCalendar: work_date, is_holiday, comment (резерв).
- ProjectFile: project_id, task_id, file_name, file_url, description, user_id.
- ProjectFolder: project_id, nextcloud_path, last_sync_status.
- ChatMessage: project_id, user_id, message, created_at.
- AuditLog: actor_id, entity, action, payload_json, created_at.

## Ключевые связи и правила
- WBS‑раздел и связанная задача синхронизируются триггером (название, исполнитель).
- Subcontract = Task с `task_type=subcontract` + approval_status.
- Contractor users не могут иметь подразделения и должны иметь роль `executor`.
