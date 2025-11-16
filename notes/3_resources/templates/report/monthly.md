---
tags:
  - monthly
  - report
month: <% tp.date.now("YYYY-MM") %>
---

# 📅 月報（実績）：<% tp.date.now("YYYY年MM月") %>

## 📊 実績集計（暫定）
```dataviewjs
const DT = dv.luxon.DateTime;
const dailyFolder = "2_areas/report/daily";   // 日報フォルダ

// --- month 正規化（frontmatter: 文字列/日付、無ければファイル名） ---
function normalizeMonth(cur) {
  let m = cur.month;
  if (m && typeof m === "object") {
    if (typeof m.toFormat === "function") return m.toFormat("yyyy-LL");
    if (m instanceof Date) return DT.fromJSDate(m).toFormat("yyyy-LL");
  }
  if (typeof m === "string") {
    const mm = m.match(/^(\d{4})[-/](\d{1,2})$/);
    if (mm) return `${mm[1]}-${mm[2].padStart(2,"0")}`;
  }
  const fn = (cur.file?.name ?? "");
  const mx = fn.match(/^(\d{4}-\d{2})$/);
  return mx ? mx[1] : null;
}

const ym = normalizeMonth(dv.current());
if (!ym) {
  dv.paragraph('month を解釈できませんでした。例: frontmatter に month: "2025-11" またはファイル名を 2025-11.md');
} else {
  const first = DT.fromFormat(ym + "-01", "yyyy-LL-dd").startOf("day");
  const last  = first.endOf("month").endOf("day");

  // --- 日付→パス直読 ---
  function ymd(d){ return d.toFormat("yyyy-LL-dd"); }
  function tryPage(path){ return dv.page(path.replace(/\.md$/,"")) ?? dv.page(path) ?? null; }
  const pages = [];
  for (let d = first; d <= last; d = d.plus({ days: 1 })) {
    const p = tryPage(`${dailyFolder}/${ymd(d)}.md`);
    if (p) pages.push(p);
  }

  // --- 抽出＆集計（@上位(中位/リソース), ⏰計画, ⏱実績） ---
  const catRe = /@([^(（]+)[(（]([^/／)）]+)[/／]([^)）]+)[)）]/;
  const estRe = /⏰\s*([0-9.]+)h/;
  const actRe = /⏱\s*([0-9.]+)h/;

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

  // --- 出力（割合3種を右に） ---
  const round1 = x => Math.round(x*10)/10;
  const pct = (num, den) => den > 0 ? `${round1(num/den*100)}%` : "—";

  const totalEst = [...agg.values()].reduce((s,r)=>s+r.est, 0);
  const totalAct = [...agg.values()].reduce((s,r)=>s+r.act, 0);

  const rows = [...agg.values()]
    .map(r => [
      r.l1, r.l2, r.l3,
      round1(r.est),             // 計画h
      round1(r.act),             // 実績h
      pct(r.act, totalAct),      // 工数割合%
      pct(r.est, totalEst),      // 計画比%
      pct(r.act, r.est)          // 達成率%
    ])
    .sort((a,b)=> a[0].localeCompare(b[0]) || a[1].localeCompare(b[1]) || a[2].localeCompare(b[2]));

  dv.header(3, `月報 実績集計（${ym}）`);
  rows.length ? dv.table(["第1層","第2層","リソース","計画h","実績h","工数割合%","計画比%","達成率%"], rows)
              : dv.paragraph("(該当なし)");
}

```


## 📊 実績サマリ
```dataviewjs
const DT = dv.luxon.DateTime;
const dailyFolder = "2_areas/report/daily";

function normalizeMonth(cur) {
  let m = cur.month;
  if (m && typeof m === "object") {
    if (typeof m.toFormat === "function") return m.toFormat("yyyy-LL");
    if (m instanceof Date) return DT.fromJSDate(m).toFormat("yyyy-LL");
  }
  if (typeof m === "string") {
    const mm = m.match(/^(\d{4})[-/](\d{1,2})$/);
    if (mm) return `${mm[1]}-${mm[2].padStart(2,"0")}`;
  }
  const fn = (cur.file?.name ?? "");
  const mx = fn.match(/^(\d{4}-\d{2})$/);
  return mx ? mx[1] : null;
}

const ym = normalizeMonth(dv.current());
if (!ym) {
  dv.paragraph('month を解釈できませんでした。例: frontmatter に month: "2025-11" またはファイル名を 2025-11.md');
} else {
  const first = DT.fromFormat(ym + "-01", "yyyy-LL-dd").startOf("day");
  const last  = first.endOf("month").endOf("day");

  function ymd(d){ return d.toFormat("yyyy-LL-dd"); }
  function tryPage(path){ return dv.page(path.replace(/\.md$/,"")) ?? dv.page(path) ?? null; }
  const pages = [];
  for (let d = first; d <= last; d = d.plus({ days: 1 })) {
    const p = tryPage(`${dailyFolder}/${ymd(d)}.md`);
    if (p) pages.push(p);
  }

  const catRe = /@([^(（]+)[(（]([^/／)）]+)[/／]([^)）]+)[)）]/;
  const estRe = /⏰\s*([0-9.]+)h/;
  const actRe = /⏱\s*([0-9.]+)h/;

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

  const round1 = x => Math.round(x*10)/10;
  const pct = (num, den) => den > 0 ? `${round1(num/den*100)}%` : "—";
  const sumEst = m => [...m.values()].reduce((s,v)=>s+v.est, 0);
  const sumAct = m => [...m.values()].reduce((s,v)=>s+v.act, 0);

  // 第1層
  const totalEst1 = sumEst(L1), totalAct1 = sumAct(L1);
  const r1 = [...L1.entries()]
    .map(([k,v]) => [k, round1(v.est), round1(v.act), pct(v.act, totalAct1), pct(v.est, totalEst1), pct(v.act, v.est)])
    .sort((a,b)=> b[2]-a[2]);

  // 第2層
  const totalEst2 = sumEst(L2), totalAct2 = sumAct(L2);
  const r2 = [...L2.entries()]
    .map(([k,v]) => [k, round1(v.est), round1(v.act), pct(v.act, totalAct2), pct(v.est, totalEst2), pct(v.act, v.est)])
    .sort((a,b)=> b[2]-a[2]);

  // 第3層
  const totalEst3 = sumEst(L3), totalAct3 = sumAct(L3);
  const r3 = [...L3.entries()]
    .map(([k,v]) => [k, round1(v.est), round1(v.act), pct(v.act, totalAct3), pct(v.est, totalEst3), pct(v.act, v.est)])
    .sort((a,b)=> b[2]-a[2]);

  dv.header(2, `📊 月報 実績サマリ（${ym}）`);
  dv.header(3, "第1層（目的軸）");
  r1.length ? dv.table(["第1層","計画h","実績h","工数割合%","計画比%","達成率%"], r1) : dv.paragraph("(該当なし)");
  dv.header(3, "第2層（粒度軸）");
  r2.length ? dv.table(["第2層","計画h","実績h","工数割合%","計画比%","達成率%"], r2) : dv.paragraph("(該当なし)");
  dv.header(3, "第3層（リソース軸）");
  r3.length ? dv.table(["第3層","計画h","実績h","工数割合%","計画比%","達成率%"], r3) : dv.paragraph("(該当なし)");
}

```

## 🔁 振り返り
- 達成事項：
- 改善：
- 次月：

<%*
const fname = tp.date.now("YYYY-MM") + ".md";
await tp.file.move("2_areas/report/monthly/" + fname);
%>
