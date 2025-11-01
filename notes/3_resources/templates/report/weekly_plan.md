<%*
/**
 * 来週の週報(計画)を作るテンプレ
 * - 基準日は「今日 + 7日」
 * - 週の境界は 水曜 → 火曜
 * - ファイルは 2_areas/report/weekly/ に置く
 */
const base = moment().add(7, "days");              // ← 来週を基準にする
const week = base.format("GGGG-[W]WW");            // 例: 2025-W45

// 水曜→火曜のレンジを作る
const dow   = base.day();                          // 0=日,1=月,2=火,3=水...
const start = base.clone().subtract((dow - 3 + 7) % 7, "days");
const end   = start.clone().add(6, "days");

// ここで保存先も決める（ファイル名とweekを絶対にズラさない）
const dest = `2_areas/report/weekly/${week}-plan.md`;
await tp.file.move(dest);

// ここで YAML を丸ごと出力する
tR += `---\n`;
tR += `tags: [weekly, report, plan]\n`;
tR += `type: plan\n`;
tR += `week: ${week}\n`;
tR += `start_date: ${start.format("YYYY-MM-DD")}\n`;
tR += `end_date: ${end.format("YYYY-MM-DD")}\n`;
tR += `---\n`;
%>
# 🗓 週報 (計画)

## 🧮 月報からの引用（Dataview参照）
```dataviewjs
const cur = dv.current();
const startStr = cur.start_date ?? null;

if (!startStr) {
  dv.paragraph("⚠ start_date がないので週を特定できません。");
  return;
}

const weekStart = moment(startStr, "YYYY-MM-DD");
const targetMonth = weekStart.format("YYYY-MM");

const monthlyPlans = dv.pages('"2_areas/report/monthly"')
  .where(p => (p.type && p.type == "plan") || (p.tags ?? []).includes("plan"))
  .where(p => {
    if (p.month) {
      return moment(p.month, "YYYY-MM").format("YYYY-MM") === targetMonth;
    }
    const m = p.file.name.slice(0, 7);
    return m === targetMonth;
  })
  .sort(p => p.file.name, "desc")
  .array();   // ← ここだけ変更

if (monthlyPlans.length === 0) {
  dv.paragraph(`📭 ${targetMonth} の月報(計画)が見つかりませんでした。`);
  return;
}

// 以下は前と同じ
const monthly = monthlyPlans[0];
const content = await dv.io.load(monthly.file.path);
const lines = content.split("\n");

const rows = [];
for (const line of lines) {
  if (!line.trim().startsWith("- [")) continue;

  const text = line.trim().replace(/^- \[[ xX]\]\s*/, "");

  const dueMatch  = text.match(/完了予定::\s*([0-9]{4}-[0-9]{2}-[0-9]{2})/);
  const planMatch = text.match(/計画\[h\]::\s*([0-9.]+)/);

  const cleanTask = text
    .replace(/完了予定::.*$/,"")
    .replace(/計画\[h\]::.*$/,"")
    .trim();

  rows.push({
    file: monthly.file.link,
    task: cleanTask,
    due:  dueMatch ? dueMatch[1] : "",
    plan: planMatch ? Number(planMatch[1]).toFixed(1) : ""
  });
}

if (rows.length === 0) {
  dv.paragraph("📭 月報(計画)にタスクがありません。`- [ ] ... 完了予定:: ... 計画[h]:: ...` で書くとここに出ます。");
} else {
  dv.table(
    ["File", "上位タスク", "完了予定", "計画[h]"],
    rows.map(r => [r.file, r.task, r.due, r.plan])
  );
}
}```

## 🎯 今週の重点テーマ
- 月報計画のうち、今週の主対象：
  - [ ] @pm(1課) 進捗報告書テンプレート刷新
  - [ ] @work(PJ1) 設計レビュー対応
  - [ ] @work(PJ2) 実装サポート

## ✅ 週次タスク計画
| 区分 | タスク | 計画[h] | 備考 |
|------|--------|---------|------|
| @pm(部共通) | 進捗報告書テンプレ修正 | 3 |  |
| @work(PJ1) | 設計レビュー対応 | 4 |  |
| @ad-hoc(突発) | 緊急対応枠 | 2 |  |




