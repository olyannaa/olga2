# Аналитика и отчетность

## Ключевые метрики (реализация backend)
- Прогресс проекта = `COUNT(tasks.status='done') / COUNT(tasks)` по `project_id`.
- Трудозатраты = сумма `time_entries.hours` по задачам проекта.
- Стоимость по таймшитам = сумма `hours * daily_rate` для участников проекта.
- Стоимость по контрактам = сумма `contract_rate` (фиксированная) для участников с контрактом.
- Общая стоимость = `timesheets + contracts`.

## Источники данных
- `tasks` + `task_status_history` → прогресс/статусы.
- `time_entries` → трудозатраты.
- `users.daily_rate / users.contract_rate` → стоимость.
- `projects.budget` → бюджет (отображается в UI).

## Реализованные отчеты
- Проектная аналитика: `/projects/:id/analytics`.
- Сводка по трудозатратам: `/time-tracking/summary` (admin/accountant).

## Ограничения
- В расчетах учитываются все задачи проекта (`project_id`), включая `project_time` и `subcontract`.
- Экспорт в Excel и отдельные отчеты — не реализованы в API/UI.
