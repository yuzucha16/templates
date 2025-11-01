---
tags: [reporting, dashboard]
---

# 📈 report dashboard (with charts)

※このノートは `2_areas/report/` に置く想定です  
※ObsidianのDataviewJSだけで動くように、Chartプラグイン不要の簡易バー表示にしてあります

---

## 🟦 1. 月報の計画（今月）

```dataview
table サブ分類 as "分類", 想定時間[h/月] as "計画[h]", 備考
from "2_areas/report/monthly"
where month = dateformat(date(today), "YYYY-MM")
sort 想定時間[h/月] desc
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

// ===== ここから簡易グラフ描画 =====
const chartRoot = dv.el("div", "", {cls: "ops-chart-root"});
const style = document.createElement("style");
style.textContent = `
.ops-chart-root { display: flex; flex-wrap: wrap; gap: 1rem; margin-top: 1rem; }
.ops-chart { border: 1px solid var(--background-modifier-border); padding: .75rem; border-radius: .5rem; min-width: 240px; }
.ops-chart-title { font-weight: 600; margin-bottom: .5rem; }
.ops-chart-bars { display: flex; gap: .5rem; align-items: flex-end; min-height: 120px; }
.ops-bar { width: 1.8rem; background: var(--interactive-accent); border-radius: .25rem .25rem 0 0; position: relative; }
.ops-bar.plan { background: rgba(130, 130, 130, .35); }
.ops-bar-label { font-size: .7rem; text-align: center; margin-top: .25rem; }
.ops-bar-val { position: absolute; top: -1.2rem; left: 50%; transform: translateX(-50%); font-size: .65rem; background: rgba(0,0,0,.35); color: white; padding: 1px 4px; border-radius: 3px; white-space: nowrap; }
`;
document.head.appendChild(style);

function renderGroup(title, obj) {
  const maxVal = Math.max(...Object.values(obj).map(v => Math.max(v.plan, v.actual, 0)), 1);
  const wrap = chartRoot.createDiv({cls: "ops-chart"});
  wrap.createDiv({cls: "ops-chart-title", text: title});
  const barsWrap = wrap.createDiv({cls: "ops-chart-bars"});
  for (const [k,v] of Object.entries(obj)) {
    // 計画バー
    const planBar = barsWrap.createDiv({cls: "ops-bar plan"});
    planBar.style.height = (v.plan / maxVal * 100) + "%";
    if (v.plan > 0) planBar.createDiv({cls: "ops-bar-val", text: v.plan.toFixed(1)});
    // 実績バー
    const actBar = barsWrap.createDiv({cls: "ops-bar"});
    actBar.style.height = (v.actual / maxVal * 100) + "%";
    if (v.actual > 0) actBar.createDiv({cls: "ops-bar-val", text: v.actual.toFixed(1)});
    // ラベル
    const label = wrap.createDiv({cls: "ops-bar-label", text: k});
    label.style.maxWidth = "4.5rem";
  }
}

if (Object.keys(bucket.pm).length) renderGroup("PM別 計画 vs 実績", bucket.pm);
if (Object.keys(bucket.work).length) renderGroup("Work別 計画 vs 実績", bucket.work);
if (Object.keys(bucket.adhoc).length) renderGroup("突発(Ad-hoc) 計画 vs 実績", bucket.adhoc);
```

---

## 🟦 3. 今週の進捗（水曜→火曜）

```dataviewjs
const today = moment();
const dow = today.day(); // 0=日
const start = moment(today).subtract((dow - 3 + 7) % 7, "days").startOf("day");
const end   = moment(start).add(6, "days").endOf("day");

const weekTasks = dv.pages('"2_areas/report/daily"')
  .where(p => {
    const d = moment(p.file.name, "YYYY-MM-DD", true);
    return d.isValid() && d.isBetween(start, end, null, "[]");
  })
  .flatMap(p => p.file.tasks);

function sumByRe(list, re) {
  return list
    .filter(t => re.test(t.text))
    .map(t => parseFloat((t.text.match(/⏱ ([0-9.]+)h/)||[0,0])[1]))
    .reduce((a,b)=>a+b,0);
}

const weekData = [
  ["@pm(部共通)", sumByRe(weekTasks, /@pm\(部共通\)/)],
  ["@pm(1課)",    sumByRe(weekTasks, /@pm\(1課\)/)],
  ["@work(PJ1)",  sumByRe(weekTasks, /@work\(PJ1\)/)],
  ["@work(PJ2)",  sumByRe(weekTasks, /@work\(PJ2\)/)],
  ["@ad-hoc(突発)",sumByRe(weekTasks, /@ad-hoc\(突発\)/)],
];

dv.header(3, `📆 今週(${start.format("YYYY-MM-DD")}〜${end.format("YYYY-MM-DD")}) 実績`);
dv.table(["分類","実績[h]"], weekData);

// 簡易横棒チャート
const wrap = dv.el("div", "", {cls: "ops-week-chart"});
const s = document.createElement("style");
s.textContent = `
.ops-week-chart { margin-top: .75rem; }
.ops-week-row { display: flex; align-items: center; gap: .5rem; margin-bottom: .25rem; }
.ops-week-label { width: 7rem; font-size: .7rem; }
.ops-week-bar { height: .6rem; background: var(--interactive-accent); border-radius: .3rem; }
`;
document.head.appendChild(s);

const maxWeek = Math.max(...weekData.map(([_,v]) => v), 1);
for (const [label, val] of weekData) {
  const row = wrap.createDiv({cls: "ops-week-row"});
  row.createDiv({cls: "ops-week-label", text: label});
  const bar = row.createDiv({cls: "ops-week-bar"});
  bar.style.width = (val / maxWeek * 100) + "%";
  bar.setAttr("title", val.toFixed(1) + "h");
}
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
