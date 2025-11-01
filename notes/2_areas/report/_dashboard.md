---
tags: [reporting, dashboard]
---

# 📈 report dashboard

※このノートは `2_areas/report/` に置く想定です  
※月報は翌月分を15日に作る運用でも、ここは「今月分」を表示します

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

const weekPmDept   = sumByRe(weekTasks, /@pm\(部共通\)/);
const weekPm1      = sumByRe(weekTasks, /@pm\(1課\)/);
const weekPJ1      = sumByRe(weekTasks, /@work\(PJ1\)/);
const weekPJ2      = sumByRe(weekTasks, /@work\(PJ2\)/);
const weekAdhoc    = sumByRe(weekTasks, /@ad-hoc\(突発\)/);

dv.header(3, `📆 今週(${start.format("YYYY-MM-DD")}〜${end.format("YYYY-MM-DD")}) 実績`);
dv.table(["分類","実績[h]"], [
  ["@pm(部共通)", weekPmDept.toFixed(1)],
  ["@pm(1課)",    weekPm1.toFixed(1)],
  ["@work(PJ1)",  weekPJ1.toFixed(1)],
  ["@work(PJ2)",  weekPJ2.toFixed(1)],
  ["@ad-hoc(突発)",weekAdhoc.toFixed(1)],
]);
```

---

## 🟦 4. 日報の入力確認（直近7件）

```dataview
list
from "2_areas/ops_reporting/daily"
sort file.name desc
limit 7
```

---

## 📝 5. メモ
- 今月の所感：
- 来月に寄せるもの：
- 突発が多かった要因：
