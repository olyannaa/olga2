# План внедрения требований v2 (без пересоздания БД)

## 1. Миграции ролевой модели и бэкфилл
- Цель: привести роли к модели v2 (Админ/ГИП/Исполнитель/Бухгалтер), деприкейтить approval_admin/department_head без потери данных, `project_manager` оставить как внутренний ключ “ГИП”.
- Файлы/модули: `backend/migrations/*`, `backend/src/common/auth/auth.types.ts`, `backend/src/auth/auth.service.ts`, `backend/src/seed.ts`.
- Миграции: добавить значения `lead_specialist`, `accountant` в `project_role`; добавить `users.can_approve_subcontracts boolean not null default false`; DML-бэкфилл `can_approve_subcontracts=true` для `approval_admin`, добавить роль `admin` тем же пользователям; `department_head` перенести в `admin` с сохранением `user_departments` (без права согласований); оставить старые роли как совместимость до следующего релиза.
- Риски: старые токены без нового флага; невозможность удалить enum-значения; конфликт прав при двойных ролях.
- Критерии приемки: миграции применяются без потери данных; бывшие `approval_admin` получают `admin` и `can_approve_subcontracts=true`; бывшие `department_head` получают `admin` без права согласований; новые проектные роли доступны для назначения.

## 2. Бэкенд: ACL и проверка прав по новой модели
- Цель: обновить проверки доступа на роли v2 и проектные роли; запретить видимость задач бухгалтера для исполнителей/главспецов.
- Файлы/модули: `backend/src/projects/projects.service.ts`, `backend/src/tasks/tasks.service.ts`, `backend/src/time-tracking/time-tracking.service.ts`, `backend/src/departments/departments.service.ts`, `backend/src/contractors/contractors.service.ts`, `backend/src/users/users.service.ts`.
- Миграции: нет (используются изменения из задачи 1).
- Риски: регрессии прав, несовпадение с фронтом, утечки бюджета/аналитики.
- Критерии приемки: админ/ГИП имеют полный доступ к проекту (админ с ограничением по `user_departments` — только свои подразделения); главспец управляет задачами и разделами, но не видит бюджет/аналитику; исполнитель видит только свои задачи; задачи бухгалтера скрыты от исполнителей/главспецов; согласования доступны только админам с `can_approve_subcontracts`.

## 3. Фронтенд: обновление ролей и навигации
- Цель: привести UI и маршруты к ролям v2, убрать obsolete-роль из выбора и навигации.
- Файлы/модули: `work-stream-alchemy-main/src/components/Layout.tsx`, `work-stream-alchemy-main/src/components/RoleSwitcher.tsx`, `work-stream-alchemy-main/src/App.tsx`, `work-stream-alchemy-main/src/lib/routes.ts`, `work-stream-alchemy-main/src/pages/Organization.tsx`, `work-stream-alchemy-main/src/pages/Projects.tsx`, `work-stream-alchemy-main/src/components/project/ProjectTeam.tsx`, `work-stream-alchemy-main/src/components/project/ProjectProperties.tsx`, `work-stream-alchemy-main/src/components/project/ProjectAnalytics.tsx`, `work-stream-alchemy-main/src/components/project/ProjectChat.tsx`.
- Миграции: нет.
- Риски: рассинхрон ролей между фронтом и бэком; неверные роли по умолчанию.
- Критерии приемки: в UI нет “Админ с согласованием” и “Руководитель подразделения”; `project_manager` отображается как “ГИП”; доступы к разделам совпадают с backend.

## 4. Субподряд: создание из свойств проекта и согласования
- Цель: дать ГИП/админу создавать субподряд в свойствах проекта, с ожиданием 2 согласований.
- Файлы/модули: `work-stream-alchemy-main/src/components/project/ProjectProperties.tsx`, `work-stream-alchemy-main/src/components/tasks/CreateTaskDialog.tsx`, `work-stream-alchemy-main/src/pages/Approvals.tsx`, `backend/src/tasks/tasks.service.ts`.
- Миграции: нет.
- Риски: дубли заявок, некорректная связь с проектом/исполнителем, обход согласований.
- Критерии приемки: субподряд создается в “Свойствах” и отображается серым до 2 согласований; доступ к выполнению появляется только после статуса `approved`.

## 5. Таймшиты: “+” на уровне проекта и перенос смены статусов дня
- Цель: дать ГИП/админу ставить “+” по проекту; перенос b/k/o/v на клик по дате; добавить пояснение.
- Файлы/модули: `work-stream-alchemy-main/src/pages/TimeTracking.tsx`, `backend/src/tasks/tasks.service.ts`, `backend/src/time-tracking/time-tracking.service.ts`.
- Миграции: нет.
- Риски: массовое создание `project_time` задач; путаница между задачами и проектными днями.
- Критерии приемки: “+” ставится на уровне проекта для ГИП/админа; b/k/o/v меняется по клику на дату; подсказка поясняет правила.

## 6. Канбан проекта: бухгалтер двигает свои задачи
- Цель: разрешить бухгалтеру перетаскивать свои задачи на проектном канбане.
- Файлы/модули: `work-stream-alchemy-main/src/components/project/ProjectTasks.tsx`.
- Миграции: нет.
- Риски: возможность двигать не свои задачи (если нет фильтра).
- Критерии приемки: бухгалтер может перетаскивать только назначенные ему задачи, остальные роли — без изменений.

## 7. WBS: сортировка и глубина разделов
- Цель: автоматически сортировать подразделы и ограничить глубину до 3 уровней.
- Файлы/модули: `backend/src/projects/projects.service.ts`, `work-stream-alchemy-main/src/components/project/ProjectSectionsTable.tsx`, `work-stream-alchemy-main/src/components/project/ProjectProperties.tsx`, `work-stream-alchemy-main/src/components/project/CreateProjectDialog.tsx`.
- Миграции: опционально добавить check-constraint на глубину `section_code` (<= 3 уровней) или серверную валидацию.
- Риски: существующие разделы глубже 3 уровней; некорректный порядок при нечисловых кодах.
- Критерии приемки: список разделов сортируется по иерархии (1, 1.1, 1.2, 2); нельзя создать 4-й уровень; визуальные отступы корректны.

## 8. Подтверждение удаления
- Цель: показывать диалог “Удалить?” везде, где есть удаление.
- Файлы/модули: `work-stream-alchemy-main/src/pages/Organization.tsx`, `work-stream-alchemy-main/src/components/project/ProjectTeam.tsx`, `work-stream-alchemy-main/src/components/project/ProjectProperties.tsx`, `work-stream-alchemy-main/src/pages/ContractorDetails.tsx`, `work-stream-alchemy-main/src/pages/Projects.tsx`.
- Миграции: нет.
- Риски: несогласованный UX, случайные удаления.
- Критерии приемки: любое удаление требует подтверждения с “Да/Отмена”.

## 9. Обязательные поля
- Цель: отметить обязательные поля звездочкой во всех формах с необязательными полями.
- Файлы/модули: `work-stream-alchemy-main/src/components/tasks/CreateTaskDialog.tsx`, `work-stream-alchemy-main/src/components/tasks/EditTaskDialog.tsx`, `work-stream-alchemy-main/src/components/project/CreateProjectDialog.tsx`, `work-stream-alchemy-main/src/components/project/ProjectProperties.tsx`, `work-stream-alchemy-main/src/pages/Organization.tsx`, `work-stream-alchemy-main/src/pages/ContractorDetails.tsx`.
- Миграции: нет.
- Риски: несоответствие UI и серверной валидации.
- Критерии приемки: все обязательные поля визуально отмечены, ошибки валидации совпадают с UI.

## 10. Баг верстки диаграммы Ганта
- Цель: найти и устранить сбой верстки в Ганте.
- Файлы/модули: `work-stream-alchemy-main/src/components/project/ProjectGantt.tsx`.
- Миграции: нет.
- Риски: невозможность воспроизвести; регрессия при масштабировании.
- Критерии приемки: баг воспроизводится по шагам, фикс проверен в day/week/month режимах и на мобильной ширине.

## 11. Убрать кнопку “Новая задача” в канбане проекта
- Цель: убрать создание задач из проектного канбана.
- Файлы/модули: `work-stream-alchemy-main/src/components/project/ProjectTasks.tsx`.
- Миграции: нет.
- Риски: пользователи теряют быстрый доступ к созданию задач (нужна компенсация через общую страницу задач/свойства).
- Критерии приемки: кнопка отсутствует, остальные функции канбана работают.
