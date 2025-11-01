---
tags: [weekly, report]
week: <% tp.date.now("GGGG-[W]WW") %>
---

<%*
const today = moment();
const start = today.clone().weekday(3);      // 今週水曜
const end   = start.clone().add(6, "days");  // 翌週火曜
tR += `start_date:: ${start.format("YYYY-MM-DD")}
`;
tR += `end_date:: ${end.format("YYYY-MM-DD")}
`;
%>

# 🗓 週報：<% tp.date.now("GGGG年 第WW週") %>
期間：<%*
const today2 = moment();
const start2 = today2.clone().weekday(3);
const end2   = start2.clone().add(6, "days");
tR += `${start2.format("YYYY-MM-DD")}〜${end2.format("YYYY-MM-DD")}`;
%>

## 🎯 1. 今週の重点テーマ
- 月報計画のうち、今週の主対象：
  - [ ] @pm(1課) 進捗報告書テンプレート刷新
  - [ ] @work(PJ1) 設計レビュー対応
  - [ ] @work(PJ2) 実装サポート

---

## 🧮 2. 月報からの引用（Dataview参照）

```dataview
table タスク as "上位タスク", 想定完了週 as "完了予定", 想定工数[h] as "計画[h]"
from "2_areas/report/monthly"
where month = dateformat(date(today), "YYYY-MM")
sort 想定完了週 asc
```

---

## ✅ 3. 今週のタスク計画（週次分解）

| 区分 | タスク | 計画[h] | 実績[h] | 進捗[%] |
|------|---------|----------|----------|----------|
| @pm(1課) | 報告書フォーマット試作 | 3 | | |
| @work(PJ1) | 設計レビュー（第1回） | 4 | | |
| @work(PJ2) | 実装動作確認 | 5 | | |
| @ad-hoc(突発) | 緊急対応枠 | 2 | | |

---

## 📊 4. 実績サマリ（自動集計）

```dataview
table file.link as "日報", text as "タスク",
  regexreplace(text, ".*@(pm\([^)]*\)).*", "$1") as "業務区分",
  regexreplace(text, ".*@(work\([^)]*\)).*", "$1") as "PJ区分",
  regexreplace(regexreplace(text, ".*⏰ ([0-9.]+)h.*", "$1"), ".*", "0") as "計画[h]",
  regexreplace(regexreplace(text, ".*⏱ ([0-9.]+)h.*", "$1"), ".*", "0") as "実績[h]",
  regexreplace(regexreplace(text, ".*🔼 ([0-9]+)%.*", "$1"), ".*", "0") as "進捗[%]"
from "2_areas/report/daily"
where date >= date(this.start_date) and date <= date(this.end_date)
sort file.name asc
```

---

## 🔁 5. 週次振り返り
- 成果：
- 遅延・課題：
- 次週の重点：

<%*
const year = tp.date.now("YYYY");
const weekFile = tp.date.now("GGGG-[W]WW");
const dest = `2_areas/report/weekly/${year}/${weekFile}.md`;
await tp.file.move(dest);
%>
