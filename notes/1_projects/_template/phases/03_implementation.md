---
type: phase
project_id: your_project
phase: implementation
order: 3
status: active
owner: [[担当者名]]
due: 2025-05-31
inputs: ["deliverables/design_doc.md"]     # 例: ["deliverables/requirements_spec.md"]
outputs: []   # 例: ["deliverables/design_doc.md"]
tags: [project/your_project, phase/implementation]
---

# Phase: implementation (3)

## Scope
- この工程のスコープを明記

## Plan
- [ ] タスクA (due::2025-05-10)
- [ ] タスクB (due::2025-05-20)

## Notes
- 決定事項やメモ

## Risks
- リスクと対策

---
## Kanban（メンバー向け）
```mermaid
kanban
  section 未着手
    タスクA
  section 進行中
    タスクB
  section 完了
```
