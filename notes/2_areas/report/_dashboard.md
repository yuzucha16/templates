---
tags: [dashboard, report]
title: ダッシュボード
updated: <% tp.date.now("YYYY-MM-DD") %>
---

# 🗓 レポート ダッシュボード

---
## 🕒 進行状況（今週）

```dataviewjs
//
// 🕒 進行状況（今週）
// 1. 最新の週報(実績)を1枚見つける
// 2. その週報の start_date/end_date で日報をフィルタ
// 3. タスクを @pm(...) / @work(...) / @ad-hoc(...) に分類して工数を合計
//

// 1. 最新の週報(実績)を探す
const weekly = dv.pages('"2_areas/report/weekly"')
  .where(p => (p.type ?? "") === "actual")
  .sort(p => p.file.name, "desc")
  .array()[0];

if (!weekly) {
  dv.paragraph("📭 週報(実績)がまだありません。");
} else {
  // 週報が持っている期間
  function toMomentAny(v) {
    if (v && typeof v === "object" && typeof v.toISODate === "function") {
      return moment(v.toISODate(), "YYYY-MM-DD", true);
    }
    if (typeof v === "string") {
      const m = moment(v, ["YYYY-MM-DD", "YYYY/MM/DD"], true);
      if (m.isValid()) return m;
    }
    return moment.invalid();
  }

  const start = toMomentAny(weekly.start_date);
  const end   = toMomentAny(weekly.end_date);

  if (!start.isValid() || !end.isValid()) {
    dv.paragraph("⚠ この週報には start_date / end_date がありません。");
  } else {
    // 2. 日報を全部とって、この週の範囲だけにする
    const dailies = dv.pages('"2_areas/report/daily"').array();

    function dailyDate(p) {
      if (p.date) {
        const m = moment(p.date, ["YYYY-MM-DD", "YYYY/MM/DD"], true);
        if (m.isValid()) return m;
        if (typeof p.date === "object" && typeof p.date.toISODate === "function") {
          return moment(p.date.toISODate(), "YYYY-MM-DD", true);
        }
      }
      return moment(p.file.name, "YYYY-MM-DD", true);
    }

    // 3. 分類ごとの集計器
    const buckets = new Map();  // key -> {plan, act, count}
    function ensureBucket(key) {
      if (!buckets.has(key)) {
        buckets.set(key, { plan: 0, act: 0, count: 0 });
      }
      return buckets.get(key);
    }

    for (const p of dailies) {
      const d = dailyDate(p);
      if (!d.isValid() || !d.isBetween(start, end, "day", "[]")) continue;

      const tasks = p.file.tasks ?? [];
      for (const t of tasks) {
        const txt = t.text ?? "";

        // タグ検出
        const pm   = txt.match(/@pm\(([^)]*)\)/);
        const work = txt.match(/@work\(([^)]*)\)/);
        const adh  = txt.match(/@ad-hoc\(([^)]*)\)/);

        let key = "その他";
        if (pm)   key = `@pm(${pm[1]})`;
        else if (work) key = `@work(${work[1]})`;
        else if (adh)  key = `@ad-hoc(${adh[1]})`;

        const plan = Number(txt.match(/⏰\s*([0-9.]+)h/)?.[1] ?? "0");
        const act  = Number(txt.match(/⏱\s*([0-9.]+)h/)?.[1] ?? "0");

        const b = ensureBucket(key);
        b.plan += plan;
        b.act  += act;
        b.count += 1;
      }
    }

    // 合計バケットを作る
    let totalPlan = 0, totalAct = 0;
    for (const [, v] of buckets) {
      totalPlan += v.plan;
      totalAct  += v.act;
    }
    buckets.set("— 合計 —", { plan: totalPlan, act: totalAct, count: 0 });

    // 表示用に並び替え（合計を最後に）
    const rows = Array.from(buckets.entries())
      .filter(([k,_]) => k !== "— 合計 —")
      .sort((a, b) => b[1].plan - a[1].plan);

    rows.push(["— 合計 —", buckets.get("— 合計 —")]);

    // 4. 出力
    dv.paragraph(`🗓 対象週: ${start.format("YYYY-MM-DD")} 〜 ${end.format("YYYY-MM-DD")}`);

    dv.table(
      ["分類", "計画[h]", "実績[h]", "乖離[h]", "進捗[%]"],
      rows.map(([name, v]) => {
        const diff = v.act - v.plan;
        const prog = v.plan > 0 ? (v.act / v.plan * 100).toFixed(1) : (v.act > 0 ? "100.0" : "0.0");
        return [
          name,
          v.plan.toFixed(1),
          v.act.toFixed(1),
          diff.toFixed(1),
          prog
        ];
      })
    );
  }
}
```

---

## 📅 今月の月報

```dataview
TABLE WITHOUT ID
  link(file.link, file.name) AS "月報ファイル",
  type AS "タイプ",
  month AS "対象月",
  file.mtime AS "更新日時"
FROM "2_areas/report/monthly"
SORT file.name DESC
LIMIT 2
```

---

## 📆 今週の週報

```dataview
TABLE WITHOUT ID
  link(file.link, file.name) AS "週報ファイル",
  type AS "タイプ",
  week AS "週番号",
  start_date AS "開始日",
  end_date AS "終了日"
FROM "2_areas/report/weekly"
SORT file.name DESC
LIMIT 3
```

---

## 🧾 最近の日報

```dataview
TABLE WITHOUT ID
  link(file.link, file.name) AS "日報",
  date AS "日付",
  file.mtime AS "更新日時"
FROM "2_areas/report/daily"
SORT date DESC
LIMIT 7
```

---

## 📈 実績サマリ（週次）

```dataviewjs
const weeklies = dv.pages('"2_areas/report/weekly"')
  .where(p => p.type && p.type == "actual")
  .sort(p => p.file.name, 'desc')
  .limit(1)
  .array();

if (weeklies.length == 0) {
  dv.paragraph("📭 実績週報がまだありません。");
} else {
  const w = weeklies[0];
  dv.paragraph(`📘 最新週報: [${w.file.name}](${w.file.path})`);
  dv.paragraph(`期間: ${w.start_date}〜${w.end_date}`);
}
```

---

## 📊 実績サマリ（月次）

```dataviewjs
const monthlies = dv.pages('"2_areas/report/monthly"')
  .where(p => p.type && p.type == "actual")
  .sort(p => p.file.name, 'desc')
  .limit(1)
  .array();

if (monthlies.length == 0) {
  dv.paragraph("📭 実績月報がまだありません。");
} else {
  const m = monthlies[0];
  dv.paragraph(`📘 最新月報: [${m.file.name}](${m.file.path})`);
  dv.paragraph(`対象月: ${m.month}`);
}
```

---
## 🪞 メモ・ToDoリンク

- [[2_areas/report/daily/]]
- [[2_areas/report/weekly/]]
- [[2_areas/report/monthly/]]
