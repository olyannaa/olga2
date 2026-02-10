# Реализация задач v4 (фактическое состояние)

## 1. «Мои задачи»: фильтр для ГИПа
- `work-stream-alchemy-main/src/pages/Tasks.tsx` — `shouldFilterByAssignee` включает `admin` и `gip`, запрос уходит с `assigneeId`.

## 2. Субподряд в таблице разделов
- `work-stream-alchemy-main/src/components/project/ProjectProperties.tsx` — формирует `subcontractSectionIds`.
- `work-stream-alchemy-main/src/components/project/ProjectSectionsTable.tsx` — колонка «Субподряд» и бейдж.

## 4–5. Канбан проекта: шильдик и имя исполнителя
- `work-stream-alchemy-main/src/components/project/ProjectTasks.tsx` — бейдж «Субподряд», показ имени исполнителя и `title` на аватаре.

## 6–7. Таймшит: фильтры проектов
- `work-stream-alchemy-main/src/pages/TimeTracking.tsx` — фильтр пустых проектов на вкладке «Субподряд» для admin/gip и для бухгалтера на обеих вкладках.

## 8. Исполнитель меняет статус в таймшите
- `backend/src/tasks/tasks.service.ts` — `listTasksForTimeTracking` возвращает `assignee_id`.
- `work-stream-alchemy-main/src/pages/TimeTracking.tsx` — `canChangeTaskStatus` проверяет `assigneeId`.
