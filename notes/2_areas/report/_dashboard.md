---
tags:
  - report
  - dashboard
---

# 📈 report dashboard

※このノートは `2_areas/report/` に置く想定です  
※月報は翌月分を15日に作る運用でも、ここは「今月分」を表示します

## 🟦 1. 月報の計画（今月）

```dataview
table file.link as "File", month, period
from "2_areas/report/monthly"
where dateformat(month, "YYYY-MM") = dateformat(date(today), "YYYY-MM")
   or dateformat(month, "YYYY-MM") = dateformat(date(today) + dur(1 month), "YYYY-MM")
sort month asc
limit 5
```

---

## 🟦 2. 日報からの実績集計（今月）

```dataviewjs
const month = moment().format("YYYY-MM");

const tasks = dv.pages('"2_areas/report/daily"')
  .where(p => p.file.name.startsWith(month))
  .flatMap(p => p.file.tasks)
  .filter(t => t.text);

function pick(text, re, def="未分類") {
  const m = text.match(re);
  return m ? m[1] : def;
}

let bucket = { pm: {}, work: {}, adhoc: {} };

for (let t of tasks) {
  const plan   = parseFloat((t.text.match(/⏰ ([0-9.]+)h/)||[0,0])[1]);
  const actual = parseFloat((t.text.match(/⏱ ([0-9.]+)h/)||[0,0])[1]);

  const pm    = pick(t.text, /@pm\(([^)]+)\)/, null);
  const work  = pick(t.text, /@work\(([^)]+)\)/, null);
  const adhoc = pick(t.text, /@ad-hoc\(([^)]+)\)/, null);

  if (pm) {
    if (!bucket.pm[pm]) bucket.pm[pm] = {plan:0, actual:0};
    bucket.pm[pm].plan   += plan  || 0;
    bucket.pm[pm].actual += actual|| 0;
  }
  if (work) {
    if (!bucket.work[work]) bucket.work[work] = {plan:0, actual:0};
    bucket.work[work].plan   += plan  || 0;
    bucket.work[work].actual += actual|| 0;
  }
  if (adhoc) {
    if (!bucket.adhoc[adhoc]) bucket.adhoc[adhoc] = {plan:0, actual:0};
    bucket.adhoc[adhoc].plan   += plan  || 0;
    bucket.adhoc[adhoc].actual += actual|| 0;
  }
}

dv.header(3, "🗂 PM別 実績（今月）");
dv.table(["PM分類","計画[h]","実績[h]","乖離[h]"],
  Object.entries(bucket.pm).map(([k,v]) => [k, v.plan.toFixed(1), v.actual.toFixed(1), (v.actual - v.plan).toFixed(1)])
);

dv.header(3, "💼 Work別 実績（今月）");
dv.table(["PJ分類","計画[h]","実績[h]","乖離[h]"],
  Object.entries(bucket.work).map(([k,v]) => [k, v.plan.toFixed(1), v.actual.toFixed(1), (v.actual - v.plan).toFixed(1)])
);

dv.header(3, "⚡ 突発(Ad-hoc)（今月）");
dv.table(["分類","計画[h]","実績[h]","乖離[h]"],
  Object.entries(bucket.adhoc).map(([k,v]) => [k, v.plan.toFixed(1), v.actual.toFixed(1), (v.actual - v.plan).toFixed(1)])
);
```

---

## 🟦 3. 今週の進捗（水曜→火曜）

```dataviewjs
// ===== 設定 =====
const ROOT = "2_areas/report/daily";

// 0. 全dailyを取る（必ず .array() にする）
const allPages = dv.pages(`"${ROOT}"`).array();
if (allPages.length === 0) {
  dv.paragraph("まだ日報がありません");
  return;
}

// 1. 日付を取る関数（frontmatter優先→ファイル名）
const getDate = (p) => {
  if (p.date) {
    const m = moment(p.date);
    if (m.isValid()) return m;
  }
  const m2 = moment(p.file.name, "YYYY-MM-DD", true);
  return m2.isValid() ? m2 : null;
};

// 2. 最新の日報の日付を基準に「その週(水→火)」を決める
const latest = allPages
  .map(p => ({ p, d: getDate(p) }))
  .filter(x => x.d && x.d.isValid())
  .sort((a, b) => b.d.valueOf() - a.d.valueOf())[0];

const base = latest.d.clone();          // この日付を基準にする
const dow  = base.day();                // 0=日,1=月,...,3=水
const start = base.clone().subtract((dow - 3 + 7) % 7, "days").startOf("day"); // その週の水曜
const end   = start.clone().add(6, "days").endOf("day");                        // 翌週火曜

// 3. その週に入っている daily だけに絞る
const weekPages = allPages.filter(p => {
  const d = getDate(p);
  return d && d.isBetween(start, end, null, "[]");
});

// 4. タスクっぽい行を全部拾う（タスクが無くても本文から拾う）
const tasks = [];
for (const p of weekPages) {
  // 4-1. dataviewが見つけてくれたチェックボックス
  const dvTasks = (p.file.tasks ?? []).map(t => ({
    text: t.text ?? "",
    source: p.file.path,
  }));

  // 4-2. 念のため内容を読み込んで @work(...) / ⏱ ... を拾う
  const content = await dv.io.load(p.file.path);
  const lineTasks = content
    .split("\n")
    .filter(l => /@pm|@work|@ad-hoc|⏰|⏱|🔼/.test(l))
    .map(l => ({
      text: l.trim(),
      source: p.file.path,
    }));

  tasks.push(...dvTasks, ...lineTasks);
}

// 5. 集計（タグは自動検出）
const tagPattern = /@(pm|work|ad-hoc)\(([^)]+)\)/g;
const results = {};

for (const t of tasks) {
  const text = t.text ?? "";

  const planMatch   = text.match(/⏰ ([0-9.]+)h/);
  const actualMatch = text.match(/⏱ ([0-9.]+)h/);
  const progMatch   = text.match(/🔼 ([0-9]+)%/);

  const plan   = planMatch   ? parseFloat(planMatch[1]) : 0;
  const actual = actualMatch ? parseFloat(actualMatch[1]) : 0;
  const prog   = progMatch   ? parseInt(progMatch[1])    : null;

  let m;
  while ((m = tagPattern.exec(text)) !== null) {
    const key = `@${m[1]}(${m[2]})`;     // 例: @work(PJ1), @pm(部共通)

    if (!results[key]) {
      results[key] = { plan: 0, actual: 0, progSum: 0, progCount: 0 };
    }
    results[key].plan   += plan;
    results[key].actual += actual;
    if (prog !== null) {
      results[key].progSum   += prog;
      results[key].progCount += 1;
    }
  }
}

// 6. 表にする
const rows = Object.entries(results)
  .sort((a, b) => b[1].actual - a[1].actual)
  .map(([k, v]) => {
    const diff = v.actual - v.plan;
    const avg  = v.progCount ? (v.progSum / v.progCount).toFixed(0) + "%" : "";
    return [
      k,
      v.plan.toFixed(1),
      v.actual.toFixed(1),
      diff.toFixed(1),
      avg,
    ];
  });

// 7. 合計行
const totalPlan   = rows.reduce((s, r) => s + Number(r[1]), 0);
const totalActual = rows.reduce((s, r) => s + Number(r[2]), 0);
const totalDiff   = totalActual - totalPlan;

// 8. 出力
dv.header(3, `📆 今週 (${start.format("YYYY-MM-DD")}〜${end.format("YYYY-MM-DD")}) 実績`);
dv.table(
  ["分類", "計画[h]", "実績[h]", "乖離[h]", "進捗(平均)"],
  [
    ...rows,
    ["— 合計 —", totalPlan.toFixed(1), totalActual.toFixed(1), totalDiff.toFixed(1), ""],
  ]
);
```

---

## 🟦 4. 日報の入力確認（直近7件）

```dataview
list
from "2_areas/report/daily"
sort file.name desc
limit 7
```

---

## 📝 5. メモ
- 今月の所感：
- 来月に寄せるもの：
- 突発が多かった要因：
