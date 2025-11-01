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
const dest = `2_areas/report/weekly/${weekId}-actual.md`;
await tp.file.move(dest);

// YAMLをここで全部書く（←これが大事）
tR += `---\n`;
tR += `tags: [weekly, report, actual]\n`;
tR += `type: actual\n`;
tR += `week: ${weekId}\n`;
tR += `start_date: ${start.format("YYYY-MM-DD")}\n`;
tR += `end_date: ${end.format("YYYY-MM-DD")}\n`;
tR += `---\n`;
%>

# 🗓 週報（実績）：<% tp.date.now("GGGG年 第WW週") %>

## 📊 今週の実績集計
```dataview
TABLE file.link AS 日報, date
FROM "2_areas/report/daily"
WHERE contains(file.name, "2025-W") = false
SORT date ASC
```


## 📊 実績サマリ

```dataviewjs
// ===== 1. この週報の期間を取得 =====
const cur = dv.current();

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

const start = toMomentAny(cur.start_date);
const end   = toMomentAny(cur.end_date);

if (!start.isValid() || !end.isValid()) {
  dv.paragraph("⚠ start_date / end_date が設定されていません。テンプレートが正しく動いているか確認してください。");
} else {
  // ===== 2. 対象期間内の日報を取得 =====
  const pages = dv.pages('"2_areas/report/daily"').array();

  function toDailyMoment(p) {
    if (p.date) {
      const m = moment(p.date, ["YYYY-MM-DD", "YYYY/MM/DD"], true);
      if (m.isValid()) return m;
      if (typeof p.date === "object" && typeof p.date.toISODate === "function") {
        return moment(p.date.toISODate(), "YYYY-MM-DD", true);
      }
    }
    return moment(p.file.name, "YYYY-MM-DD", true);
  }

  // ===== 3. タスクを抽出 =====
  const raw = [];
  for (const p of pages) {
    const d = toDailyMoment(p);
    if (!d.isValid() || !d.isBetween(start, end, "day", "[]")) continue;

    const tasks = p.file.tasks ?? [];
    for (const t of tasks) {
      const text = t.text ?? "";

      const cleanTask = text
        .replace(/\s*@pm\([^)]*\)/g, "")
        .replace(/\s*@work\([^)]*\)/g, "")
        .replace(/\s*@ad-hoc\([^)]*\)/g, "")
        .replace(/\s*⏰\s*[0-9.]+h/g, "")
        .replace(/\s*⏱\s*[0-9.]+h/g, "")
        .replace(/\s*🔼\s*[0-9]+%/g, "")
        .trim();

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

  // ===== 4. タスク名で集約 =====
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

    if (!item.pm && r.pm) item.pm = r.pm;
    if (!item.work && r.work) item.work = r.work;
  }

  // ===== 5. 表示 =====
  const result = Array.from(agg.values())
    .sort((a, b) => b.plan - a.plan);

  if (result.length === 0) {
    dv.paragraph(`📭 ${start.format("MM/DD")}〜${end.format("MM/DD")} の期間には該当タスクがありません。`);
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
- 成果：
- 問題：
- 次週：

