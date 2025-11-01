<%*
const base = moment().add(1, "month");   // 来月
const ym   = base.format("YYYY-MM");
const periodStart = base.clone().startOf("month").format("YYYY-MM-DD");
const periodEnd   = base.clone().add(1, "month").startOf("month").format("YYYY-MM-DD");

const dest = `2_areas/report/monthly/${ym}-plan.md`;
await tp.file.move(dest);

tR += `---\n`;
tR += `tags: [monthly, report, plan]\n`;
tR += `type: plan\n`;
tR += `month: ${ym}\n`;
tR += `period: ${periodStart}〜${periodEnd}\n`;
tR += `---\n`;
%>

# 📅 月報（計画）

## 🎯 今月の重点方針
- 技術テーマ：
- 組織テーマ：
- 主な課題・リスク：
- 成果目標：

## 📊 計画工数サマリ（時間目安）

- [ ] PJ1 スプリント4準備 完了予定:: 2025-12-05 計画[h]:: 3
- [ ] 部共通 年末レポート 完了予定:: 2025-12-20 計画[h]:: 2
- [ ] PJ2 顧客レビュー資料 作成  計画[h]:: 1.5

---

## 📅 月次イベント・定常業務
- 部内定例（毎週水曜）
- PJ1レビュー会（第2・第4木曜）
- 品質会議（第3火曜）

---

## ✅ 重点タスク（上位粒度）

| 区分          | タスク           | 目標       | 想定完了週 | 想定工数[h] |
| ----------- | ------------- | -------- | ----- | ------- |
| @pm(1課)     | 進捗報告書テンプレート刷新 | フォーマット統一 | W49   | 5       |
| @work(PJ1)  | 設計書完成・レビュー完了  | 設計完了     | W50   | 12      |
| @work(PJ2)  | 実装支援・動作確認     | 安定化      | W51   | 10      |
| @ad-hoc(突発) | 緊急対応枠         | 突発吸収     | -     | 15      |

| 区分          | サブ分類        | 想定時間[h/月] | 備考          |
| ----------- | ----------- | --------- | ----------- |
| @pm(部共通)    | 部内定例、資料作成   | 10        | 定例・部報告・管理業務 |
| @pm(1課)     | メンバー指導、課内進捗 | 8         | 会議・レビュー対応   |
| @work(PJ1)  | 設計完了＋レビュー   | 30        | フェーズ目標あり    |
| @work(PJ2)  | 実装支援＋動作確認   | 25        | 軽作業中心       |
| @ad-hoc(突発) | 緊急調整・特命対応   | 15        | バッファ領域      |
