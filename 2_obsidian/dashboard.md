# Dashboard

## Active Projects (KPI)
```dataview
TABLE
  status as "状態",
  deadline as "期限",
  (round((length(filter(file.tasks, (t) => t.completed)) / max(1, length(file.tasks))) * 100)) + "%" as "進捗率",
  length(filter(file.tasks, (t) => !t.completed && t.due && date(t.due) < date(now()))) as "期限超過タスク数"
FROM "projects"
WHERE contains(file.folder, "projects/") AND status = "active"
SORT deadline ASC
```

## 今週のタスク（全プロジェクト横断）
```tasks
not done
due this week
```

## 最近の更新
```dataview
LIST
FROM ""
SORT file.mtime DESC
LIMIT 15
```
