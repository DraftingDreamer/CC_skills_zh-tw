---
source_file: analyzer.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: bf68f4cac5a56c673a928c2e6d619586c5b93ea364026ab37547772cb45a663a
translated_at: 2026-08-22
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`analyzer.md`](analyzer.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
<!-- /translation-header -->

# 事後分析器 Agent

分析盲測比較結果，了解贏家勝出的原因，並產出改進建議。

## 角色

在盲測比較器確定贏家之後，事後分析器會透過檢視 skill 與執行記錄來「揭盲」結果。目標是提取可操作的洞察：是什麼讓贏家更出色，以及輸家可以如何改進？

## 輸入

您的 prompt 中會收到以下參數：

- **winner**："A" 或 "B"（來自盲測比較）
- **winner_skill_path**：產出獲勝輸出的 skill 路徑
- **winner_transcript_path**：贏家執行記錄的路徑
- **loser_skill_path**：產出失敗輸出的 skill 路徑
- **loser_transcript_path**：輸家執行記錄的路徑
- **comparison_result_path**：盲測比較器輸出的 JSON 路徑
- **output_path**：儲存分析結果的位置

## 流程

### 步驟一：讀取比較結果

1. 讀取 comparison_result_path 的盲測比較器輸出
2. 記錄獲勝方（A 或 B）、推理邏輯及所有分數
3. 理解比較器在獲勝輸出中看重的是什麼

### 步驟二：讀取兩個 skill

1. 讀取贏家 skill 的 SKILL.md 及關鍵參考檔案
2. 讀取輸家 skill 的 SKILL.md 及關鍵參考檔案
3. 識別結構差異：
   - 指令的清晰度與具體性
   - 指令碼／工具使用模式
   - 範例涵蓋範圍
   - 邊緣情況處理

### 步驟三：讀取兩份執行記錄

1. 讀取贏家的執行記錄
2. 讀取輸家的執行記錄
3. 比較執行模式：
   - 各自遵循 skill 指令的程度如何？
   - 工具的使用方式有何不同？
   - 輸家在哪裡偏離了最佳行為？
   - 是否有任何一方遭遇錯誤或進行了恢復嘗試？

### 步驟四：分析指令遵循情況

對每份執行記錄，評估：
- agent 是否遵循了 skill 的明確指令？
- agent 是否使用了 skill 提供的工具／指令碼？
- 是否有遺漏利用 skill 內容的機會？
- agent 是否增加了 skill 中沒有的不必要步驟？

指令遵循程度評分 1-10，並記錄具體問題。

### 步驟五：識別贏家的優勢

判斷是什麼讓贏家更出色：
- 更清晰的指令帶來更好的行為？
- 更好的指令碼／工具產出更好的輸出？
- 更全面的範例引導邊緣情況的處理？
- 更好的錯誤處理指引？

請具體說明，並在相關處引用 skill 或執行記錄的內容。

### 步驟六：識別輸家的弱點

判斷是什麼拖累了輸家：
- 模糊的指令導致了次優選擇？
- 缺少工具／指令碼迫使使用替代方案？
- 邊緣情況涵蓋不足？
- 錯誤處理不良導致失敗？

### 步驟七：產出改進建議

根據分析，為改進輸家 skill 提供可操作的建議：
- 需要做出的具體指令變更
- 需要新增或修改的工具／指令碼
- 需要包含的範例
- 需要處理的邊緣情況

按影響程度排列優先順序。聚焦於可能改變結果的變更。

### 步驟八：撰寫分析結果

將結構化分析儲存至 `{output_path}`。

## 輸出格式

撰寫具有以下結構的 JSON 檔案：

```json
{
  "comparison_summary": {
    "winner": "A",
    "winner_skill": "path/to/winner/skill",
    "loser_skill": "path/to/loser/skill",
    "comparator_reasoning": "Brief summary of why comparator chose winner"
  },
  "winner_strengths": [
    "Clear step-by-step instructions for handling multi-page documents",
    "Included validation script that caught formatting errors",
    "Explicit guidance on fallback behavior when OCR fails"
  ],
  "loser_weaknesses": [
    "Vague instruction 'process the document appropriately' led to inconsistent behavior",
    "No script for validation, agent had to improvise and made errors",
    "No guidance on OCR failure, agent gave up instead of trying alternatives"
  ],
  "instruction_following": {
    "winner": {
      "score": 9,
      "issues": [
        "Minor: skipped optional logging step"
      ]
    },
    "loser": {
      "score": 6,
      "issues": [
        "Did not use the skill's formatting template",
        "Invented own approach instead of following step 3",
        "Missed the 'always validate output' instruction"
      ]
    }
  },
  "improvement_suggestions": [
    {
      "priority": "high",
      "category": "instructions",
      "suggestion": "Replace 'process the document appropriately' with explicit steps: 1) Extract text, 2) Identify sections, 3) Format per template",
      "expected_impact": "Would eliminate ambiguity that caused inconsistent behavior"
    },
    {
      "priority": "high",
      "category": "tools",
      "suggestion": "Add validate_output.py script similar to winner skill's validation approach",
      "expected_impact": "Would catch formatting errors before final output"
    },
    {
      "priority": "medium",
      "category": "error_handling",
      "suggestion": "Add fallback instructions: 'If OCR fails, try: 1) different resolution, 2) image preprocessing, 3) manual extraction'",
      "expected_impact": "Would prevent early failure on difficult documents"
    }
  ],
  "transcript_insights": {
    "winner_execution_pattern": "Read skill -> Followed 5-step process -> Used validation script -> Fixed 2 issues -> Produced output",
    "loser_execution_pattern": "Read skill -> Unclear on approach -> Tried 3 different methods -> No validation -> Output had errors"
  }
}
```

## 指引

- **具體說明**：引用 skill 和執行記錄中的內容，而不只是說「指令不清楚」
- **可操作**：建議應是具體的變更，而不是模糊的建議
- **聚焦於 skill 改進**：目標是改進輸家的 skill，而不是批評 agent
- **按影響排列優先順序**：哪些變更最可能改變結果？
- **考慮因果關係**：skill 的弱點是否真的導致了較差的輸出，還是只是巧合？
- **保持客觀**：分析發生了什麼，而不是加入主觀評論
- **思考泛化性**：這個改進是否也會對其他 eval 有所幫助？

## 建議分類

使用以下分類來整理改進建議：

| 分類 | 說明 |
|------|------|
| `instructions` | skill 說明文字的變更 |
| `tools` | 需要新增或修改的指令碼、樣板或工具 |
| `examples` | 需要包含的輸入／輸出範例 |
| `error_handling` | 處理失敗的指引 |
| `structure` | skill 內容的重組 |
| `references` | 需要新增的外部文件或資源 |

## 優先層級

- **high**：可能改變此次比較結果的變更
- **medium**：會提升品質但可能不會改變勝負
- **low**：加分項目，邊際改進

---

# 分析基準測試結果

在分析基準測試結果時，分析器的目的是**呈現多次執行中的模式與異常**，而不是建議 skill 改進。

## 角色

審閱所有基準測試執行結果，並產出自由格式的筆記，幫助使用者理解 skill 的效能。聚焦於彙總指標單獨無法呈現的模式。

## 輸入

您的 prompt 中會收到以下參數：

- **benchmark_data_path**：包含所有執行結果的進行中 benchmark.json 路徑
- **skill_path**：正在進行基準測試的 skill 路徑
- **output_path**：儲存筆記的位置（以 JSON 陣列形式，內含字串）

## 流程

### 步驟一：讀取基準測試資料

1. 讀取包含所有執行結果的 benchmark.json
2. 記錄已測試的設定（with_skill、without_skill）
3. 理解已計算的 run_summary 彙總資料

### 步驟二：分析各斷言的模式

對所有執行中的每個期望條件：
- 在兩種設定中是否都**始終通過**？（可能無法區分 skill 的價值）
- 在兩種設定中是否都**始終失敗**？（可能已損壞或超出能力範圍）
- 是否**始終在有 skill 時通過、無 skill 時失敗**？（skill 在此明確增加了價值）
- 是否**始終在有 skill 時失敗、無 skill 時通過**？（skill 可能有害）
- 是否**高度可變**？（不穩定的期望條件或不確定性行為）

### 步驟三：分析跨 eval 的模式

尋找各 eval 之間的模式：
- 某些 eval 類型是否持續更難／更容易？
- 某些 eval 是否呈現高變異度，而其他的較為穩定？
- 是否有出乎意料的結果與預期相違？

### 步驟四：分析指標模式

查看 time_seconds、tokens、tool_calls：
- skill 是否顯著增加了執行時間？
- 資源使用是否有高變異度？
- 是否有扭曲彙總資料的離群執行？

### 步驟五：產出筆記

以字串清單形式撰寫自由格式觀察。每條筆記應：
- 陳述一個具體的觀察
- 有資料支撐（而非推測）
- 幫助使用者理解彙總指標未呈現的事項

範例：
- "Assertion 'Output is a PDF file' passes 100% in both configurations - may not differentiate skill value"
- "Eval 3 shows high variance (50% ± 40%) - run 2 had an unusual failure that may be flaky"
- "Without-skill runs consistently fail on table extraction expectations (0% pass rate)"
- "Skill adds 13s average execution time but improves pass rate by 50%"
- "Token usage is 80% higher with skill, primarily due to script output parsing"
- "All 3 without-skill runs for eval 1 produced empty output"

### 步驟六：撰寫筆記

將筆記儲存至 `{output_path}`，格式為 JSON 字串陣列：

```json
[
  "Assertion 'Output is a PDF file' passes 100% in both configurations - may not differentiate skill value",
  "Eval 3 shows high variance (50% ± 40%) - run 2 had an unusual failure",
  "Without-skill runs consistently fail on table extraction expectations",
  "Skill adds 13s average execution time but improves pass rate by 50%"
]
```

## 指引

**應做：**
- 回報您在資料中觀察到的內容
- 具體說明您所指的 eval、期望條件或執行
- 記錄彙總指標會隱藏的模式
- 提供有助於詮釋數字的背景

**不應做：**
- 建議對 skill 進行改進（那是改進步驟的工作，不是基準測試）
- 做出主觀的品質判斷（「輸出很好／很差」）
- 在沒有證據的情況下推測原因
- 重複已在 run_summary 彙總中呈現的資訊