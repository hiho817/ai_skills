---
name: zhiyu-police
description: Scan, explain, and safely revise Mainland Chinese or simplified-Chinese technical wording into Taiwan Traditional-Chinese academic usage. Use for thesis chapters, research papers, robot/control documents, and requests such as 「檢查支語」、「改成台灣用語」、「兩岸術語統一」 or 「支語警察」.
---

# 支語警察

以臺灣學術寫作與工程慣用語檢查繁體中文文件；先區分真正的地區用語、臺灣亦常用的術語與不得改寫的引用原文，再做最小範圍修訂。

## 工作流程

1. 先讀取專案術語表、既有名詞備忘錄與文件上下文。若使用者要求修訂，先確認工作區既有變更。
2. 搜尋 `references/terminology.md` 的高信心詞彙。一次檢查完整文件或使用者指定範圍。
3. 對「可安全替換」項目直接修訂；對「需依語意判斷」項目逐句改寫，禁止批量置換。
4. 不改寫文獻正式題名、引文原文、程式 API、ROS topic、變數名或專有產品名稱。
5. 修訂後重新掃描目標詞，檢查術語一致性、Markdown/LaTeX 完整性及差異範圍。回報修改數量與仍保留的例外。

## 判斷準則

- 優先採用臺灣學術詞彙；有爭議時，先查國家教育研究院「雙語詞彙、學術名詞暨辭書資訊網」與臺灣大專校院論文或官方技術文件。
- 不把簡體字以外的所有差異都視為錯誤。`優化`、`數據`、`魯棒性`、`建模` 等詞在臺灣學術文本亦可見，除非使用者要求特定風格，否則提出建議或依全篇一致性處理。
- `位姿` 不可直接改為「姿態」：pose 通常含位置與方向；若內容涵蓋兩者，改為「位置與姿態」或依既有術語表保留「位姿」。只有 orientation/attitude 才用「姿態」。
- `構型` 用於機構幾何或關節配置；軟體與濾波器設定使用「組態」或「設定」；一般 arrangement 才依語句使用「配置」。
- `量測`用於感測器輸出與實驗量得的數值；測繪、尺寸或動詞語境可使用「測量」。

## 範圍與呈現

先列出候選用語與判斷理由，再在獲授權修訂時實作。對文件修訂，以「原詞 → 建議詞」摘要呈現，並說明任何保留用語的原因。

詳細對照與來源見 `references/terminology.md`。
