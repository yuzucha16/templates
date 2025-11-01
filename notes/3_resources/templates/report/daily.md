---
tags: [daily, report]
type: actual
date: <% tp.file.title %>
---

# 📅 日報：<% tp.file.title %>

## ✅ 今日のToDo
- [ ] 部内MTG準備 @pm(部共通) ⏰ 1.0h ⏱1.0h 
- [ ] コードレビュー @work(PJ1) ⏰ 2.0h ⏱3.0h 
- [ ] 会議調整 @ad-hoc(突発) ⏰ 0.5h⏱1.0h 

## 🔁 振り返りメモ
- 今日できた：
- 困ったこと：
- 明日やること：

<%*
await tp.file.move("2_areas/report/daily/" + tp.file.title + ".md");
%>
