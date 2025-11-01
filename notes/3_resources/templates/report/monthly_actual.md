---
tags: [monthly, report, actual]
type: actual
month: <% tp.date.now("YYYY-MM") %>
---

# 📅 月報（実績）：<% tp.date.now("YYYY年MM月") %>

## 📊 実績集計（暫定）
```dataviewjs
// 1. この月報がどの月かを決める
const cur = dv.current();

function monthFromAny(v, fallbackName) {
  // Obsidian の日付プロパティっぽいとき
  if (v && typeof v === "object" && typeof v.toISODate === "function") {
    return moment(v.toISODate(), "YYYY-MM-DD", true).startOf("month");
  }
  // 文字列のとき
  if (typeof v === "string") {
    const m = moment(v, ["YYYY-MM", "YYYY/MM/DD", "YYYY-MM-DD"], true);
    if (m.isValid()) return m.startOf("month");
  }
  // ファイル名 2025-11-actual.md みたいなとき
  if (fallbackName) {
    const m = moment(fallbackName.slice(0, 7), "YYYY-MM", true);
    if (m.isValid()) return m.startOf("month");
  }
  return moment.invalid();
}

const mStart = monthFromAny(cur.month, cur.file.name);
if (!mStart.isValid()) {
  dv.paragraph("⚠ この月報には month: YYYY-MM がありません。");
} else {
  const mEnd = mStart.clone().endOf("month");

  // 2. 日報を全部とって、この月に入るものだけ拾う
  const pages = dv.pages('"2_areas/report/daily"').array();

  function dailyDate(p) {
    // プロパティに date がある → / と - 両方見る
    if (p.date) {
      const m = moment(p.date, ["YYYY-MM-DD", "YYYY/MM/DD"], true);
      if (m.isValid()) return m;
      // dataviewのdateオブジェクト
      if (typeof p.date === "object" && typeof p.date.toISODate === "function") {
        return moment(p.date.toISODate(), "YYYY-MM-DD", true);
      }
    }
    // なければファイル名
    return moment(p.file.name, "YYYY-MM-DD", true);
  }

  // まずは全タスクをフラットに集める
  const raw = [];
  for (const p of pages) {
    const d = dailyDate(p);
    if (!d.isValid()) continue;
    if (!d.isBetween(mStart, mEnd, "day", "[]")) continue;

    const tasks = p.file.tasks ?? [];
    for (const t of tasks) {
      const text = t.text ?? "";

      // タスク名をクリーンにする（タグ・時間・進捗を落とす）
      const cleanTask = text
        .replace(/\s*@pm\([^)]*\)/g, "")
        .replace(/\s*@work\([^)]*\)/g, "")
        .replace(/\s*@ad-hoc\([^)]*\)/g, "")
        .replace(/\s*⏰\s*[0-9.]+h/g, "")
        .replace(/\s*⏱\s*[0-9.]+h/g, "")
        .replace(/\s*🔼\s*[0-9]+%/g, "")
        .trim();

      // pm/work をそれぞれ match で素直に取る
      const pm   = text.match(/@pm\([^)]*\)/)?.[0] ?? "";
      const work = text.match(/@work\([^)]*\)/)?.[0] ?? "";

      const plan = Number(text.match(/⏰\s*([0-9.]+)h/)?.[1] ?? "0");
      const act  = Number(text.match(/⏱\s*([0-9.]+)h/)?.[1] ?? "0");

      raw.push({
        task: cleanTask || "(タスクなし)",
        pm,
        work,
        plan,
        act,
      });
    }
  }

  // 3. タスク名で集約する
  const agg = new Map();

  for (const r of raw) {
    const key = r.task;
    if (!agg.has(key)) {
      agg.set(key, {
        task: r.task,
        pm: r.pm,
        work: r.work,
        plan: 0,
        act: 0,
      });
    }
    const item = agg.get(key);
    item.plan += r.plan;
    item.act  += r.act;

    // pm / work は最初に出てきたものを保持、なければ後続で埋める
    if (!item.pm && r.pm) item.pm = r.pm;
    if (!item.work && r.work) item.work = r.work;
  }

  const result = Array.from(agg.values())
    // 合計時間が大きい順に並べると月報っぽくなる
    .sort((a, b) => b.plan - a.plan);

  if (result.length === 0) {
    dv.paragraph("📭 この月には日報タスクが見つかりませんでした。");
  } else {
    dv.table(
      ["タスク", "業務区分", "PJ区分", "計画合計[h]", "実績合計[h]"],
      result.map(r => [
        r.task,
        r.pm,
        r.work,
        r.plan.toFixed(1),
        r.act.toFixed(1),
      ])
    );
  }
}
```

## 🔁 振り返り
- 達成事項：
- 改善：
- 次月：

<%*
const fname = tp.date.now("YYYY-MM") + "-actual.md";
await tp.file.move("2_areas/report/monthly/" + fname);
%>
