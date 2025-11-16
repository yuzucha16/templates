---
title: 日報ダッシュボード
type: dashboard
tags: [report, dashboard]
---

# 📊 日報ダッシュボード
---
## 🎯 KPIサマリ（直近5日）

```dataviewjs
const reports = dv.pages('"2_areas/report/daily"')
  .where(p => p.date && p.date >= dv.date("today") - dv.duration("5 days"));

const totalHours = reports.array().reduce((acc, p) => acc + (p.実績時間 || 0), 0);
const avgHours = reports.length ? (totalHours / reports.length).toFixed(1) : 0;

const avg達成率 = reports.length ? (reports.array()
  .reduce((acc, p) => acc + (p.達成率 || 0), 0) / reports.length).toFixed(1) : 0;

const avg計画比 = reports.length ? (reports.array()
  .reduce((acc, p) => acc + (p.計画比 || 0), 0) / reports.length).toFixed(1) : 0;

dv.paragraph(`
| 指標 | 値 |
|:--|--:|
| 🕒 合計実績時間（5日間） | **${totalHours.toFixed(1)} h** |
| ⏱ 平均実績時間/日 | **${avgHours} h** |
| 📈 平均達成率 | **${avg達成率}%** |
| 📊 平均計画比 | **${avg計画比}%** |
`);
```

---
## 📅 直近5日の日報サマリ

```dataview
TABLE
  dateformat(date, "MM/dd (ddd)") AS "日付",
  実績時間 AS "実績[h]",
  工数割合 AS "工数割合[%]",
  計画比 AS "計画比[%]",
  達成率 AS "達成率[%]",
  file.link AS "リンク"
FROM "2_areas/report/daily"
WHERE date >= date(today) - dur(5 days)
SORT date DESC
```

---

## 🕓 最近アクセスしたノート

```dataview
TABLE file.link AS "ノート", dateformat(file.mtime, "MM/dd HH:mm") AS "最終更新"
FROM "2_areas"
SORT file.mtime DESC
LIMIT 5
```

---

## 📈 実績時間 推移（直近5日）

```dataviewjs
const rows = dv.pages('"2_areas/report/daily"')
  .where(p => p.date && p.date >= dv.date("today") - dv.duration("5 days"))
  .sort(p => p.date, 'asc')
  .map(p => ({ date: p.date, time: p.実績時間 || 0 }));

const labels = rows.map(r => dv.date(r.date).toFormat("MM/dd"));
const data = rows.map(r => r.time);

if (labels.length) {
  dv.paragraph("### 📊 実績時間グラフ");
  renderChart({
    type: "bar",
    data: {
      labels: labels,
      datasets: [{
        label: "実績時間[h]",
        data: data
      }]
    },
    options: {
      plugins: {
        legend: { display: false }
      },
      scales: {
        y: { beginAtZero: true }
      }
    }
  });
} else {
  dv.paragraph("📭 データがありません。");
}
```

---

## 🧭 補足
- `2_areas/report/daily/` 以下の日報が対象（frontmatter に `date:` を含む想定）  
- フィールド名は `実績時間`, `計画比`, `達成率`, `工数割合` に対応  
- グラフ描画には `DataviewJS + Chart.js` 対応環境が必要（Obsidian Charts Plugin でも可）  
