# SQL-запросы для портала онбординга (PostgreSQL)

Все запросы написаны для базы данных, описанной в ER-диаграмме. Основные таблицы: `Users`, `Roles`, `Tasks`, `EquipmentRequests`, `OnboardingFlows`, `Comments`, `TaskApprovals`.

---

## 1. Список текущих задач для новичка

**Назначение:** отобразить на дашборде новичка все задачи, которые ещё не выполнены, с сортировкой по сроку выполнения.

```sql
SELECT id, title, deadline, status
FROM tasks
WHERE assignee_id = 123 
  AND status != 'completed'
ORDER BY deadline ASC;
```

**Примечание:** в реальном запросе `assignee_id` будет браться из сессии текущего пользователя.

---

## 2. Прогресс выполнения чек-листа (в процентах)

**Назначение:** показать на дашборде процент завершения всех задач для конкретного новичка.

```sql
SELECT 
  ROUND(COUNT(*) FILTER (WHERE status = 'completed') * 100.0 / COUNT(*), 2) AS progress_percent
FROM tasks
WHERE assignee_id = 123;
```

---

## 3. Список подопечных для наставника

**Назначение:** отобразить наставнику всех новичков, закреплённых за ним, с количеством их задач.

```sql
SELECT u.id, u.name, COUNT(t.id) AS total_tasks
FROM users u
LEFT JOIN tasks t ON u.id = t.assignee_id
WHERE u.mentor_id = 456
GROUP BY u.id, u.name;
```

---

## 4. Количество заявок на оборудование по типам

**Назначение:** HR-отчёт – сколько заявок каждого типа оборудования подано.

```sql
SELECT equipment_type, COUNT(*) AS request_count
FROM equipment_requests
GROUP BY equipment_type
ORDER BY request_count DESC;
```

---

## 5. Среднее время выполнения задач по новичкам (в днях)

**Назначение:** оценить, сколько в среднем уходит на выполнение одной задачи для каждого новичка.

```sql
SELECT assignee_id, 
       AVG(EXTRACT(DAY FROM (updated_at - created_at))) AS avg_days
FROM tasks
WHERE status = 'completed' AND updated_at IS NOT NULL
GROUP BY assignee_id;
```

**Примечание:** если нет даты завершения (`updated_at`), задача считается не завершённой.

---

## 6. Список задач, ожидающих утверждения руководителем

**Назначение:** показать руководителю задачи, которые требуют его подтверждения (статус `'pending_approval'`).

```sql
SELECT t.id, t.title, u.name AS assignee_name, t.deadline
FROM tasks t
JOIN users u ON t.assignee_id = u.id
WHERE t.status = 'pending_approval'
  AND t.assignee_id IN (
    SELECT id FROM users WHERE mentor_id IS NOT NULL
  );
```

---

## 7. Комплексный отчёт по новичкам для HR

**Назначение:** сводка – сколько новичков всего, средний прогресс по чек-листам, общее количество заявок на оборудование.

```sql
SELECT 
  COUNT(DISTINCT u.id) AS newbies_count,
  COALESCE(ROUND(AVG(comp.progress), 2), 0) AS avg_progress,
  COUNT(eq.id) AS total_equipment_requests
FROM users u
LEFT JOIN (
  SELECT assignee_id, 
         COUNT(*) FILTER (WHERE status = 'completed') * 100.0 / COUNT(*) AS progress
  FROM tasks
  GROUP BY assignee_id
) comp ON u.id = comp.assignee_id
LEFT JOIN equipment_requests eq ON u.id = eq.user_id
WHERE u.role_id = (SELECT id FROM roles WHERE name = 'Новичок');
```

**Пояснение:** `COALESCE` нужен, чтобы при отсутствии задач показывать 0 вместо NULL. 
