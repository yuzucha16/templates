---
type: project
project_id: {{project_id}}
status: active
owner: [[オーナー名]]
deadline: 2025-12-31
tags: [project/{{project_id}}, status/active]
---

# Project: {{project_name}}

## Purpose (第1層: 目的)
- このPJの意義・背景を書く

## Goals / Milestones (第2層)
- [ ] ゴール1 (due::2025-05-31)
- [ ] ゴール2 (due::2025-09-30)

## Deliverables (第2層)
- [[deliverables/requirements_spec.md]]
- [[deliverables/design_doc.md]]
- [[deliverables/test_plan.md]]

## Phases (第3層)
- [[phases/01_requirements.md]]
- [[phases/02_design.md]]
- [[phases/03_implementation.md]]
- [[phases/04_test.md]]

---
## KPI View (マネジメント向け)
```dataview
TABLE
  deadline as "期限",
  (round((length(filter(file.tasks, (t) => t.completed)) / max(1, length(file.tasks))) * 100)) + "%" as "進捗率",
  length(filter(file.tasks, (t) => !t.completed && t.due && date(t.due) < date(now()))) as "期限超過タスク数"
FROM "projects"
WHERE project_id = this.project_id
```

## Chain View（数珠つなぎ）
```dataview
TABLE phase as "工程", file.link as "ノート", join(inputs, ", ") as "Input", join(outputs, ", ") as "Output", status as "状態"
FROM "projects"
WHERE type = "phase" AND project_id = this.project_id
SORT order ASC
```

## Task Overview（担当者別タスク）
```dataview
TABLE WITHOUT ID file.link as "ノート", choice(t.section, t.text, t.text) as "タスク", t.due as "期限", t.completed as "完了"
FROM "projects"
FLATTEN file.tasks as t
WHERE project_id = this.project_id
SORT t.due ASC
```
