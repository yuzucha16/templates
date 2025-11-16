<%*
/**
 * 週報（実績）用
 * 基準：今日を含む「水曜→火曜」
 */
const today = moment();

// 今日が何曜日か (0=日,1=月,2=火,3=水,...)
const dow = today.day();

// 今週の水曜にそろえる
const start = today.clone().subtract((dow - 3 + 7) % 7, "days").startOf("day");
// 火曜まで 6日足す
const end   = start.clone().add(6, "days").endOf("day");

// ISO週番号（ファイル名に使う）
const weekId = start.format("GGGG-[W]WW");

// 保存先はここ
const dest = `2_areas/report/weekly/${weekId}.md`;
await tp.file.move(dest);

// YAMLをここで全部書く（←これが大事）
tR += `---\n`;
tR += `tags: [weekly, report]\n`;
tR += `week: ${weekId}\n`;
tR += `start_date: ${start.format("YYYY-MM-DD")}\n`;
tR += `end_date: ${end.format("YYYY-MM-DD")}\n`;
tR += `---\n`;
%>

# 🗓 週報（実績）：<% tp.date.now("GGGG年 第WW週") %>

```dataviewjs
const DT = dv.luxon.DateTime;
const dailyFolder = "2_areas/report/daily";

// 期間（frontmatter のゆるい解釈）
function parseLoose(s, inherit) {
  if (!s) return null;
  s = String(s).trim().replaceAll("／","/").replaceAll("-","/");
  for (const f of ["yyyy/M/d","yyyy/MM/dd","M/d","MM/dd","d"]) {
    const d = DT.fromFormat(s, f); if (d.isValid) return d;
  }
  if (inherit) {
    if (/^\d{1,2}$/.test(s)) return inherit.set({ day: +s });
    const md = s.match(/^(\d{1,2})\/(\d{1,2})$/);
    if (md) return inherit.set({ month:+md[1], day:+md[2] });
  }
  return null;
}
let start = parseLoose(dv.current().start_date, DT.now()) || DT.now().startOf("week");
let end   = parseLoose(dv.current().end_date,   start)     || start.endOf("week");
start = start.startOf("day"); end = end.endOf("day");

// 直読
function ymd(d){ return d.toFormat("yyyy-LL-dd"); }
function tryPage(path){ return dv.page(path.replace(/\.md$/,"")) ?? dv.page(path) ?? null; }
const pages = [];
for (let d = start; d <= end; d = d.plus({ days: 1 })) {
  const p = tryPage(`${dailyFolder}/${ymd(d)}.md`);
  if (p) pages.push(p);
}

// 抽出＆集計
const catRe = /@([^(（]+)[(（]([^/／)）]+)[/／]([^)）]+)[)）]/; // @上位(中位/リソース)
const estRe = /⏰\s*([0-9.]+)h/;  // 計画
const actRe = /⏱\s*([0-9.]+)h/;  // 実績
const agg = new Map(); // key = l1|l2|l3

for (const p of pages) for (const t of (p.file.tasks ?? [])) {
  const m = t.text.match(catRe); if (!m) continue;
  const l1 = m[1].trim(), l2 = m[2].trim(), l3 = m[3].trim();
  const est = parseFloat(t.text.match(estRe)?.[1] ?? "0") || 0;
  const act = parseFloat(t.text.match(actRe)?.[1] ?? "0") || 0;
  const k = `${l1}||${l2}||${l3}`;
  if (!agg.has(k)) agg.set(k, { l1, l2, l3, est:0, act:0 });
  const r = agg.get(k); r.est += est; r.act += act;
}

// 分母
const round1 = x => Math.round(x*10)/10;
const pct = (num, den) => den > 0 ? `${round1(num/den*100)}%` : "—";
const totalEst = [...agg.values()].reduce((s,r)=>s+r.est, 0);
const totalAct = [...agg.values()].reduce((s,r)=>s+r.act, 0);

// 行生成
const rows = [...agg.values()]
  .map(r => [
    r.l1, r.l2, r.l3,
    round1(r.est),            // 計画h
    round1(r.act),            // 実績h
    pct(r.act, totalAct || 0),// 工数割合%
    pct(r.est, totalEst || 0),// 計画比%
    pct(r.act, r.est)         // 達成率%
  ])
  .sort((a,b)=> a[0].localeCompare(b[0]) || a[1].localeCompare(b[1]) || a[2].localeCompare(b[2]));

dv.table(["第1層","第2層","リソース","計画h","実績h","工数割合%","計画比%","達成率%"], rows);

```

## 📊 今週の実績集計
```dataview
TABLE file.link AS 日報, date
FROM "2_areas/report/daily"
WHERE contains(file.name, "2025-W") = false
SORT date ASC
```


## 📊 実績サマリ

```dataviewjs
const DT = dv.luxon.DateTime;
const dailyFolder = "2_areas/report/daily";

// 期間
function parseLoose(s, inherit) {
  if (!s) return null;
  s = String(s).trim().replaceAll("／","/").replaceAll("-","/");
  for (const f of ["yyyy/M/d","yyyy/MM/dd","M/d","MM/dd","d"]) {
    const d = DT.fromFormat(s, f); if (d.isValid) return d;
  }
  if (inherit) {
    if (/^\d{1,2}$/.test(s)) return inherit.set({ day: +s });
    const md = s.match(/^(\d{1,2})\/(\d{1,2})$/);
    if (md) return inherit.set({ month:+md[1], day:+md[2] });
  }
  return null;
}
let start = parseLoose(dv.current().start_date, DT.now()) || DT.now().startOf("week");
let end   = parseLoose(dv.current().end_date,   start)     || start.endOf("week");
start = start.startOf("day"); end = end.endOf("day");

// 直読
function ymd(d){ return d.toFormat("yyyy-LL-dd"); }
function tryPage(path){ return dv.page(path.replace(/\.md$/,"")) ?? dv.page(path) ?? null; }
const pages = [];
for (let d = start; d <= end; d = d.plus({ days: 1 })) {
  const p = tryPage(`${dailyFolder}/${ymd(d)}.md`);
  if (p) pages.push(p);
}

// 抽出＆層別集計
const catRe = /@([^(（]+)[(（]([^/／)）]+)[/／]([^)）]+)[)）]/;
const estRe = /⏰\s*([0-9.]+)h/;  // 計画
const actRe = /⏱\s*([0-9.]+)h/;  // 実績
const L1 = new Map(), L2 = new Map(), L3 = new Map();

for (const p of pages) for (const t of (p.file.tasks ?? [])) {
  const m = t.text.match(catRe); if (!m) continue;
  const l1 = m[1].trim(), l2 = m[2].trim(), l3 = m[3].trim();
  const est = parseFloat(t.text.match(estRe)?.[1] ?? "0") || 0;
  const act = parseFloat(t.text.match(actRe)?.[1] ?? "0") || 0;

  if (!L1.has(l1)) L1.set(l1, { est:0, act:0 });
  if (!L2.has(l2)) L2.set(l2, { est:0, act:0 });
  if (!L3.has(l3)) L3.set(l3, { est:0, act:0 });
  L1.get(l1).est += est; L1.get(l1).act += act;
  L2.get(l2).est += est; L2.get(l2).act += act;
  L3.get(l3).est += est; L3.get(l3).act += act;
}

// 分母（計画と実績）
const round1 = x => Math.round(x*10)/10;
const pct = (num, den) => den > 0 ? `${round1(num/den*100)}%` : "—";
const sumEst = m => [...m.values()].reduce((s,v)=>s+v.est, 0);
const sumAct = m => [...m.values()].reduce((s,v)=>s+v.act, 0);

// L1
const totalEst1 = sumEst(L1), totalAct1 = sumAct(L1);
const r1 = [...L1.entries()]
  .map(([k,v]) => [k, round1(v.est), round1(v.act),
                    pct(v.act, totalAct1), // 工数割合%
                    pct(v.est, totalEst1), // 計画比%
                    pct(v.act, v.est)      // 達成率%
                   ])
  .sort((a,b)=> b[2]-a[2]);
dv.header(3, "第1層（目的軸）");
r1.length ? dv.table(["第1層","計画h","実績h","工数割合%","計画比%","達成率%"], r1) : dv.paragraph("(該当なし)");

// L2
const totalEst2 = sumEst(L2), totalAct2 = sumAct(L2);
const r2 = [...L2.entries()]
  .map(([k,v]) => [k, round1(v.est), round1(v.act),
                    pct(v.act, totalAct2), // 工数割合%
                    pct(v.est, totalEst2), // 計画比%
                    pct(v.act, v.est)      // 達成率%
                   ])
  .sort((a,b)=> b[2]-a[2]);
dv.header(3, "第2層（粒度軸）");
r2.length ? dv.table(["第2層","計画h","実績h","工数割合%","計画比%","達成率%"], r2) : dv.paragraph("(該当なし)");

// L3
const totalEst3 = sumEst(L3), totalAct3 = sumAct(L3);
const r3 = [...L3.entries()]
  .map(([k,v]) => [k, round1(v.est), round1(v.act),
                    pct(v.act, totalAct3), // 工数割合%
                    pct(v.est, totalEst3), // 計画比%
                    pct(v.act, v.est)      // 達成率%
                   ])
  .sort((a,b)=> b[2]-a[2]);
dv.header(3, "第3層（リソース軸）");
r3.length ? dv.table(["第3層","計画h","実績h","工数割合%","計画比%","達成率%"], r3) : dv.paragraph("(該当なし)");

```

## 🔁 振り返り
- 成果：
- 問題：
- 次週：

