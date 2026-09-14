---
source_file: model-migration.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 646531ed152701c571771d38306b1d6ac358270a7ca8763b39a9e694916b08c9
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `model-migration.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->
# 模型遷移指南

> **如果你是透過 `/claude-api migrate` 進入：** 這就是正確的檔案。請依序執行下列步驟，不要把它們摘要後回傳給使用者。在接觸任何檔案前，先從步驟 0（確認範圍）開始。

說明如何將現有程式碼移轉至較新的 Claude 模型。涵蓋重大變更、已棄用參數，以及已退役模型的可直接替換項。

如需最新的權威版本（每種支援語言都含程式碼範例），請從 `shared/live-sources.md` 取得 **Migration Guide** URL 並使用 WebFetch。這份檔案是整合後、內建於 skill 的參考；每當模型發布或重大變更可能已改變現況時，請改以線上文件為準。

**這份檔案很大。** 請使用下方的章節名稱跳轉（或在本檔案中使用 `Grep` 搜尋標題文字）。先閱讀步驟 0 和步驟 1——它們適用於每次遷移；接著只閱讀你要遷移至之模型的個別目標章節。

| 章節 | 使用時機 |
|---|---|
| 步驟 0：確認遷移範圍 | 一律需要——任何編輯之前 |
| 步驟 1：分類每個檔案 | 一律需要——決定要替換、並列新增或略過 |
| 各 SDK 語法參考 | 將本指南中的 Python 範例轉換為 TypeScript / Go / Ruby / Java / C# / PHP |
| 目標模型／退役模型替代項 | 選擇目標模型 |
| 依來源模型列出的重大變更 | 遷移至 Opus 4.6／Sonnet 4.6 |
| 遷移至 Opus 4.7 | 遷移至 Opus 4.7（重大變更、靜默預設值、行為變化） |
| Opus 4.7 遷移檢查清單 | 4.7 的必要與選用項目，標記為 `[BLOCKS]`／`[TUNE]` |
| 遷移至 Opus 4.8 | 遷移至 Opus 4.8（沒有新的重大變更；工作階段中途的 system prompt；行為重新調整） |
| Opus 4.8 遷移檢查清單 | 4.8 的必要與選用項目，標記為 `[BLOCKS]`／`[TUNE]` |
| 遷移至 Claude Opus 5 | Opus 4.8 -> Claude Opus 5（停用 thinking 時受 effort 限制；對話中途變更工具；每回合 effort 與工作任務預算；重新調整冗長度、過度驗證與範圍） |
| Claude Opus 5 遷移檢查清單 | Claude Opus 5 的必要與選用項目，標記為 `[BLOCKS]`／`[TUNE]` |
| 遷移至 Claude Sonnet 5 | Sonnet 4.6 -> Claude Sonnet 5（預設開啟自適應思考；非預設取樣參數會回傳 400；新 tokenizer；用於 coding／agent 式工作的 `xhigh` effort；高解析度視覺；行為重新調整） |
| Claude Sonnet 5 遷移檢查清單 | 必要與選用項目，標記為 `[BLOCKS]`／`[TUNE]` |
| 遷移至 Claude Fable 5.1 | 遷移至 Claude Fable 5.1 或 Claude Mythos 5.1（始終開啟 thinking；絕不回傳原始思維鏈；拒絕處理；資料保留；行為變化與 prompt 指引） |
| Claude Fable 5.1 遷移檢查清單 | Claude Fable 5.1 的必要與選用項目，標記為 `[BLOCKS]`／`[TUNE]` |
| 從 Claude Fable 5 遷移至 Claude Fable 5.1 | Claude Fable 5／Claude Opus 5／Claude Mythos 5 -> Claude Fable 5.1 或 Claude Mythos 5.1（強制 `tool_choice` 會回傳 400；「保留 thinking」——模型繫結區塊與歷程編輯檢查；每則訊息的 effort；每回合只能附加的提醒；`display: "updates"` 進度更新；更便宜的快取讀取；行為重新調整） |
| 從 Claude Fable 5 遷移至 Claude Fable 5.1 的檢查清單 | Claude Fable 5 -> Claude Fable 5.1 遷移的必要與選用項目，標記為 `[BLOCKS]`／`[TUNE]` |
| 驗證遷移 | 編輯後——執行階段抽查 |

**摘要：** 變更模型 ID 字串。如果你使用 `budget_tokens`，請改用 `thinking: {type: "adaptive"}`。如果你使用助手預填，它們在 Opus 4.6 和 Sonnet 4.6 上都會回傳 400——請改用其中一種預填替代方式（最常見的是 `output_config.format`；請參閱「依來源模型列出的重大變更」中的表格）。如果你要從 Sonnet 4.5 移至 Sonnet 4.6，請明確設定 `effort`——4.6 的預設值是 `high`。移除 `effort-2025-11-24` 和 `fine-grained-tool-streaming-2025-05-14` beta 標頭（4.6 已正式推出）；改用自適應思考後，移除 `interleaved-thinking-2025-05-14`（只有使用過渡用的 `budget_tokens` 應急措施時才保留）。接著把 `client.beta.messages.create` 改回 `client.messages.create`。降低任何強硬的「CRITICAL: YOU MUST」工具指示；4.6 更嚴格遵循 system prompt。

---

## 步驟 0：確認遷移範圍

**在任何 Write、Edit 或 MultiEdit 呼叫之前，都要確認範圍。** 如果使用者的請求沒有明確指定單一檔案、特定目錄或明確的檔案清單，**請先詢問——不要開始編輯**。這項規則不可協商：即使是「遷移我的程式碼庫」、「把我的專案移至 X」、「升級至 Sonnet 4.6」或單獨一句「遷移至 Opus 4.7」這類祈使語氣的請求，範圍仍然模糊，必須提出澄清問題。「我的專案」、「我的程式碼」、「我的程式碼庫」、「全部」、「到處」或「整個存放庫」等詞語都是**模糊的，而非指示性的**——它們說明了要做什麼，卻沒有說明在哪裡做。請先詢問再執行。

請明確列出常見範圍，等待回答後再接觸任何檔案：

1. 整個工作目錄
2. 特定子目錄（例如 `src/`、`app/`、`services/billing/`）
3. 特定檔案或檔案清單

請把這些內容整理成一個澄清問題，讓使用者可以在一輪中回答。**只有在範圍已經明確無歧義時，才可不詢問直接進行**——例如使用者指定了確切檔案（「將 `extract.py` 遷移至 Sonnet 4.6」）、指向特定目錄（「將 `services/billing/` 下的所有內容遷移至 Opus 4.6」）、列出特定檔案（「更新 `a.py` 和 `b.py`」），或已在較早一輪回答範圍問題。如果只根據提示就能精確回答「這項變更會碰到哪些檔案？」，即可繼續；否則請詢問。

**實作範例。** 如果使用者說「把我的專案移至 Opus 4.6。我希望所有適合的地方都使用自適應思考」，你不知道「我的專案」是指整個工作目錄、只有 `src/`、只有正式環境程式碼，還是其他範圍——「`everywhere`」讓意圖很清楚（更新*範圍內*的每個呼叫位置），但範圍本身仍未定義。不要開始編輯。請回覆：

> 開始編輯前，請確認範圍。我可以遷移：
> 1. 工作目錄中的每個 `.py` 檔案
> 2. 只有 `src/` 下的檔案（正式環境程式碼）
> 3. 你指定的特定子目錄或檔案清單
>
> 你要選哪一項？

接著等待回答。對「遷移至 Opus 4.7」和單獨一句「幫我升級至 Sonnet 4.6」也適用相同規則——編輯前先詢問。

**估算範圍問題（大型存放庫）。** 詢問前，先取得每個目錄的數量，讓使用者可以明確選擇：

```sh
rg -l "<old-model-id>" --type-not md | cut -d/ -f1 | sort | uniq -c | sort -rn
```

在範圍問題中呈現這份分解（例如「在 3 個目錄中找到 217 個參照：api/（130）、api-go/（62）、routing/（25）。要遷移哪些？」）。調查前也要確認 `git status` 是乾淨的——意外的修改表示有並行程序；請停止並調查後再繼續。

---

## 步驟 1：分類每個檔案

不是每個包含舊模型 ID 的檔案都是 API 的**呼叫端**。編輯前，請將每個檔案分類至下列其中一個類別——正確的處理方式各不相同：

| # | 類別 | 形式 | 動作 |
|---|---|---|---|
| 1 | **呼叫 API／SDK** | `client.messages.create(model=...)`、`anthropic.Anthropic()`、請求承載 | 替換模型 ID，**並**套用下方目標版本的重大變更檢查清單。 |
| 2 | **定義或提供模型** | 模型登錄表、OpenAPI 規格、路由／佇列設定、模型原則列舉、產生的目錄 | 舊項目**保留**（該模型仍由服務提供）。詢問要不要 (a) 並列新增模型、(b) 維持不動，或 (c) 退役舊模型——絕不可盲目替換。**如果無法詢問，預設採用 (a)：並列新增模型並標記它**——替換會把仍在正式環境中提供的模型從登錄中移除。 |
| 3 | **以不透明字串參照 ID** | UI 備援常數、功能閘門子字串檢查、一般測試 fixture、標籤剖析器、環境預設值 | 通常替換字串，並確認任何剖析器／正規表示式／子字串比對都能處理新 ID——但先檢查下方的子案例。 |
| 4 | **帶尾碼的變體 ID** | 如 `-fast`、`-1024k`、`-200k`、`[1m]`、含日期快照的 `claude-<model>-<suffix>` | 這些是部署／路由識別字，不是公開模型 ID。**不要假設存在對應的新模型。** 先在登錄表中確認；如果不存在，就保留字串並加以標記。**例外：`-fast` 字串（例如 `claude-opus-4-6-fast`）由下方的 Fast Mode 章節處理**，它會將字串改寫為 Opus 4.8，加上 `speed="fast"` 與 `fast-mode-2026-02-01` beta，而不是原樣保留。 |

**類別 3 的子案例——替換字串參照前，請檢查：**

 - **功能閘門**（例如 `if 'opus-4-6' in model_id:` 用來啟用功能）-> **並列新增新 ID**，不要替換。舊模型仍由服務提供，也仍具備該功能，因此替換會讓仍經過此處的舊模型流量靜默地失去功能。如果你知道沒有舊模型流量會命中此閘門（單一呼叫端的程式碼庫已完整遷移），可以替換；不確定時，請並列新增。
 - **登錄表驗證測試**（例如 `assert "claude-X" in supported_models`、`test_X_has_N_clusters`）-> **並列新增新模型的判斷；保留舊判斷。** 舊模型仍由服務提供，因此原有判斷仍有效；但登錄表也應包含新模型，所以新模型也要加以判斷。判斷方式：如果測試在清單中參照多個模型版本，就是登錄表測試；如果只有一個模型在結構中與自身比較，就是一般 fixture。
 - **凍結／產生的快照** -> **重新產生**，不要手動編輯。
 - **與定義端耦合**（例如透過共用的 `conftest` 種子清單傳遞模型授權的整合測試，或對計費層級／速率限制群組列舉、產生的 SKU／定價目錄進行判斷）-> **先確認定義端已有新模型項目。** 如果沒有，請新增種子項目（以最接近的現有層級作為暫時值）；如果你無法有把握地完成，請詢問使用者如何填入定義端。**不要略過測試。** 未填入定義端就進行替換，會使測試在執行階段失敗。

特別遷移測試時：重大變更參數（`temperature`、`top_p`、`budget_tokens`）通常不存在——測試 fixture 很少在佔位模型上設定取樣參數。仍然必須執行重大變更掃描，但預期大多數結果會是乾淨的。

**先找出刻意標記的同步點。** 許多程式碼庫會用 `MODEL LAUNCH`、`KEEP IN SYNC`、`@model-update` 或類似的註解標記，指出每次模型發布都必須變更的位置。在廣泛搜尋模型 ID 之前，先依存放庫採用的慣例執行 Grep——這些標記會指向承載關鍵變更的位置。

---

## 各 SDK 語法參考

本指南中的程式碼範例使用 Python。**每個官方 Anthropic SDK 都有相同欄位**——Stainless 會根據相同的 OpenAPI 規格產生全部 7 個 SDK，因此 JSON 欄位名稱是一對一對應，差別只有大小寫慣例。使用下列表格，將 Python 範例轉換為你要遷移的 SDK。

> **將型別與方法名稱寫入客戶程式碼前，請對照 SDK 原始碼確認。** 從 `shared/live-sources.md` 的 SDK 原始碼表（每個 SDK 一列）對相關存放庫執行 WebFetch，確認確切的符號——尤其是具型別的 SDK（Go、Java、C#），其 union／builder 名稱可能與 JSON 形狀不同。不要猜測下表或 `<lang>/claude-api/README.md` 中沒有的型別名稱。


### `thinking`——`budget_tokens` -> adaptive

| SDK | 之前 | 之後 |
|---|---|---|
| Python | `thinking={"type": "enabled", "budget_tokens": N}` | `thinking={"type": "adaptive"}` |
| TypeScript | `thinking: { type: 'enabled', budget_tokens: N }` | `thinking: { type: 'adaptive' }` |
| Go | `Thinking: anthropic.ThinkingConfigParamOfEnabled(N)` | `Thinking: anthropic.ThinkingConfigParamUnion{OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{}}` |
| Ruby | `thinking: { type: "enabled", budget_tokens: N }` | `thinking: { type: "adaptive" }` |
| Java | `.thinking(ThinkingConfigEnabled.builder().budgetTokens(N).build())` | `.thinking(ThinkingConfigAdaptive.builder().build())` |
| C# | `Thinking = new ThinkingConfigEnabled { BudgetTokens = N }` | `Thinking = new ThinkingConfigAdaptive()` |
| PHP | `thinking: ['type' => 'enabled', 'budget_tokens' => N]` | `thinking: ['type' => 'adaptive']` |

### 取樣參數——`temperature`／`top_p`／`top_k`

（在 Opus 4.7 上完全移除欄位；在 Claude 4.x 上，`temperature` 與 `top_p` 最多保留其中一個。）

| SDK | 要移除的欄位 |
|---|---|
| Python | `temperature=...`, `top_p=...`, `top_k=...` |
| TypeScript | `temperature: ...`, `top_p: ...`, `top_k: ...` |
| Go | `Temperature: anthropic.Float(...)`, `TopP: anthropic.Float(...)`, `TopK: anthropic.Int(...)` |
| Ruby | `temperature: ...`, `top_p: ...`, `top_k: ...` |
| Java | `.temperature(...)`, `.topP(...)`, `.topK(...)` |
| C# | `Temperature = ...`, `TopP = ...`, `TopK = ...` |
| PHP | `temperature: ...`, `topP: ...`, `topK: ...` |

### 預填替代方式——透過 `output_config.format` 的結構化輸出

| SDK | 移除（最後的助手回合） | 新增 |
|---|---|---|
| Python | `{"role": "assistant", "content": "..."}` | `output_config={"format": {"type": "json_schema", "schema": SCHEMA}}` |
| TypeScript | `{ role: 'assistant', content: '...' }` | `output_config: { format: { type: 'json_schema', schema: SCHEMA } }` |
| Go | trailing `anthropic.MessageParam{Role: "assistant", ...}` | `OutputConfig: anthropic.OutputConfigParam{Format: anthropic.JSONOutputFormatParam{...}}` |
| Ruby | `{ role: "assistant", content: "..." }` | `output_config: { format: { type: "json_schema", schema: SCHEMA } }` |
| Java | trailing `Message.builder().role(ASSISTANT)...` | `.outputConfig(OutputConfig.builder().format(JsonOutputFormat.builder()...build()).build())` |
| C# | trailing `new Message { Role = "assistant", ... }` | `OutputConfig = new OutputConfig { Format = new JsonOutputFormat { ... } }` |
| PHP | trailing `['role' => 'assistant', 'content' => '...']` | `outputConfig: ['format' => ['type' => 'json_schema', 'schema' => $SCHEMA]]` |

### `thinking.display`——重新選擇摘要推理（Opus 4.7）

| SDK | 新增 |
|---|---|
| Python | `thinking={"type": "adaptive", "display": "summarized"}` |
| TypeScript | `thinking: { type: 'adaptive', display: 'summarized' }` |
| Go | `Thinking: anthropic.ThinkingConfigParamUnion{OfAdaptive: &anthropic.ThinkingConfigAdaptiveParam{Display: anthropic.ThinkingConfigAdaptiveDisplaySummarized}}` |
| Ruby | `thinking: { type: "adaptive", display: "summarized" }` (or `display_:` when constructing the model class directly) |
| Java | `.thinking(ThinkingConfigAdaptive.builder().display(ThinkingConfigAdaptive.Display.SUMMARIZED).build())` |
| C# | `Thinking = new ThinkingConfigAdaptive { Display = Display.Summarized }` |
| PHP | `thinking: ['type' => 'adaptive', 'display' => 'summarized']` |

對於表格中未列出的任何欄位，Python 範例中的 JSON 索引鍵直接轉換即可：Python／TypeScript／Ruby 使用 `snake_case`，PHP 的具名引數使用 `camelCase`，Go／C# 的結構欄位使用 `PascalCase`，Java 的 builder 方法使用 `camelCase`。

---

## 說明你做的每項變更

對沒讀過版本資訊的使用者來說，遷移編輯常看起來毫無理由——移除 `temperature`、刪除預填、改寫 system prompt 句子。**對每項編輯，都要告訴使用者你改了什麼以及為什麼改**，並連結到促成該變更的特定 API 或行為變化。請在工作過程中的摘要裡這樣做，不要只在最後說明。

對 **system prompt 的編輯**尤其要說清楚。使用者理所當然會珍惜自己的 prompt，而 prompt 調整是判斷性決策（不是 API 的硬性要求）。對任何 prompt 編輯：

- 引述變更前與變更後的文字。
- 說明促成變更的行為變化（例如：「Opus 4.7 會依工作複雜度校準回應長度，因此我加入明確的長度指示」，或「4.6 更照字面遵循指示，因此『CRITICAL: YOU MUST use the search tool』現在會過度觸發——已放寬為『Use the search tool when...』」）。
- 清楚說明哪些 prompt 編輯是**選用的調整**（語氣、長度、subagent 指引），哪些程式碼編輯是**避免 400 所必要的變更**（取樣參數、`budget_tokens`、預填）。絕不要把選用的 prompt 變更說成必要。

如果要一次套用多項 prompt 調整，請以短清單提出，讓使用者可以逐項接受或拒絕，不要默默改寫其 system prompt。

---

## 開始遷移前

1. **確認目標模型 ID。** 只使用 `shared/models.md` 中的精確字串——不要在別名後附加日期尾碼（`claude-opus-4-6`，不是 `claude-opus-4-6-20251101`）。猜測 ID 會得到 404。
2. **使用這份檢查清單確認程式碼使用哪些功能：**
   - `thinking: {type: "enabled", budget_tokens: N}` -> 在 Opus 4.6／Sonnet 4.6 上遷移至自適應思考（仍可運作，但已棄用）
   - 助手回合預填（以 `role: "assistant"` 結尾的 `messages`）-> 在 Opus 4.6／Sonnet 4.6 上必須變更（會回傳 400）
   - `messages.create()` 上的 `output_format` 參數 -> 所有模型都必須變更（全 API 已棄用）
   - `max_tokens > ~16000` -> 任何模型都必須使用串流（超過約 16K 可能造成 SDK HTTP 逾時）。使用串流時，目前每個模型都能達到 128K，只有 Haiku 4.5 上限為 64K
   - Beta 標頭 `effort-2025-11-24`、`fine-grained-tool-streaming-2025-05-14`、`interleaved-thinking-2025-05-14` -> 4.6 已正式推出，請移除它們，並將 `client.beta.messages.create` 改成 `client.messages.create`
   - 從 Sonnet 4.5 移至 Sonnet 4.6 且未設定 `effort` -> 4.6 預設為 `high`，可能改變延遲／成本特性
   - 含有 `CRITICAL`、`MUST`、`If in doubt, use X` 等文字的 system prompt -> 在 4.6 上可能過度觸發（見「Prompt 行為變化」）
   - 如果來自 3.x／4.0／4.1：也要檢查取樣參數（`temperature` + `top_p`）、工具版本（`text_editor_20250728`）、`refusal` + `model_context_window_exceeded` 停止原因，以及工具參數尾端換行的處理
3. **先對單一請求進行測試。** 對新模型執行一次呼叫、檢查回應，然後再推出。

---

## 目標模型（建議目標）

| 目前使用... | 遷移至 | 原因 |
| ------------------------------------- | ------------------ | ------------------------------------------------- |
| Claude Mythos Preview（`claude-mythos-preview`） | `claude-mythos-5-1`（Project Glasswing 後繼者）或 `claude-fable-5-1`（GA） | 相同 tokenizer 家族——主要只需替換模型 ID；移除 `thinking` 設定與預填；請參閱「遷移至 Claude Fable 5.1」 |
| Claude Fable 5（`claude-fable-5`） | `claude-fable-5-1` | 相同層級、相同每 token 價格、相同 tokenizer；三項重大變更（強制 `tool_choice` 會回傳 400、「保留 thinking」）——請參閱「從 Claude Fable 5 遷移至 Claude Fable 5.1」 |
| Claude Mythos 5（`claude-mythos-5`） | `claude-mythos-5-1` | 與 claude-fable-5 -> claude-fable-5-1 相同的路徑；請參閱「從 Claude Fable 5 遷移至 Claude Fable 5.1」下的 Claude Mythos 5.1 章節 |
| Opus 4.8 | `claude-opus-5` | 目前的 Opus。兩項重大變更（預設開啟 thinking；停用 thinking 時受 `high` effort 限制），另加 prompt 重新調整——請參閱「遷移至 Claude Opus 5」 |
| Opus 4.7 | `claude-opus-5` | 套用 Opus 4.8 章節（prompt 重新調整，沒有新的重大變更），再套用 Claude Opus 5 章節 |
| Opus 4.6 | `claude-opus-5` | 先套用 Opus 4.7 重大變更，再做 4.8 重新調整，最後套用 Claude Opus 5 章節 |
| Opus 4.0／4.1／4.5／Opus 3 | `claude-opus-5` | 依序套用 4.6 -> 4.7 -> 4.8 -> Claude Opus 5（自適應思考、移除取樣參數，然後重新調整） |
| Sonnet 4.6 | `claude-sonnet-5` | 以 Sonnet 的成本提供接近 Opus 的 agent 式與 coding 工作品質；預設開啟自適應思考；請參閱「遷移至 Claude Sonnet 5」 |
| Sonnet 4.0／4.5／3.7／3.5 | `claude-sonnet-5` | 先套用 Sonnet 4.6 的變更，再套用 Claude Sonnet 5 章節 |
| Haiku 3／3.5 | `claude-haiku-4-5` | 速度最快且最具成本效益 |

除非呼叫端明確選擇其他模型，否則預設使用其層級最新的 Opus。Opus 遷移採分層方式：如果你使用 Opus 4.6 或更早版本，請依序套用各版本的章節，直到目標版本為止（例如 4.5 -> 4.8 表示依序套用 4.6、4.7 和 4.8 章節）。4.7 -> 4.8 沒有新的重大變更——請參閱下方的「遷移至 Opus 4.8」。

---

## 退役模型替代項

這些模型會回傳 404——請立即更新：

| 退役模型 | 退役日期 | 可直接替換項 |
| ----------------------------- | ------------- | -------------------- |
| `claude-3-7-sonnet-20250219`  | Feb 19, 2026  | `claude-sonnet-5` |
| `claude-3-5-haiku-20241022`   | Feb 19, 2026  | `claude-haiku-4-5`   |
| `claude-3-opus-20240229`      | Jan 5, 2026   | `claude-opus-4-8`    |
| `claude-3-5-sonnet-20241022`  | Oct 28, 2025  | `claude-sonnet-5` |
| `claude-3-5-sonnet-20240620`  | Oct 28, 2025  | `claude-sonnet-5` |
| `claude-3-sonnet-20240229`    | Jul 21, 2025  | `claude-sonnet-5` |
| `claude-2.1`, `claude-2.0`    | Jul 21, 2025  | `claude-sonnet-5` |

## 已棄用模型（即將退役）

| 模型 | 退役日期 | 替代項 |
| ----------------------------- | ------------- | -------------------- |
| `claude-3-haiku-20240307`     | Apr 19, 2026  | `claude-haiku-4-5`   |
| `claude-opus-4-20250514`      | June 15, 2026 | `claude-opus-4-8`    |
| `claude-sonnet-4-20250514`    | June 15, 2026 | `claude-sonnet-5` |

---

## 依來源模型列出的重大變更

### 從 Sonnet 4.5 遷移至 Sonnet 4.6（effort 預設值變更）

Sonnet 4.5 沒有 `effort` 參數；Sonnet 4.6 預設為 `high`。如果你只切換模型字串而不做其他事，可能會看到延遲和 token 使用量明顯增加。請明確設定 `effort`。

**建議起始點：**

| 工作負載 | 起始值 | 備註 |
| ------------------------------------------------- | -------------- | -------------------------------------------------------------------------------------------------------- |
| 聊天、分類、內容產生 | `low` | 搭配 `thinking: {"type": "disabled"}` 時，效能會與 Sonnet 4.5 的無思考模式相近或更好 |
| 大多數應用程式（平衡） | `medium` | 品質與成本的預設最佳平衡點 |
| Agent 式 coding、工具密集型工作流程 | `medium` | 搭配自適應思考及寬裕的 `max_tokens`（使用串流時最多 128K——Sonnet 4.6 的上限） |
| 自主多步驟 agent、長期迴圈 | `high` | 如果延遲／token 成為疑慮，降至 `medium` |
| 電腦使用 agent | `high` + adaptive | Sonnet 4.6 的最佳電腦使用準確度來自 adaptive + high |

特別針對無思考的聊天工作負載：

```python
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=8192,
    thinking={"type": "disabled"},
    output_config={"effort": "low"},
    messages=[{"role": "user", "content": "..."}],
)
```

**改用 Opus 4.6 的時機：** 最困難且時間跨度最長的問題——大型程式碼遷移、深入研究、長時間自主工作。Sonnet 4.6 在快速完成和成本效益方面勝出。

### 遷移至 Opus 4.6／Sonnet 4.6（從任何較舊模型）

**1. 手動延伸思考已棄用——請使用自適應思考。**

`thinking: {type: "enabled", budget_tokens: N}`（具有固定 token 預算的手動延伸思考）已在 Opus 4.6 和 Sonnet 4.6 上棄用。請將它替換為 `thinking: {type: "adaptive"}`，讓 Claude 自行決定何時思考以及思考多少。自適應思考也會自動啟用交錯思考（不需要 beta 標頭）。

```python
# Old (still works on older models, deprecated on 4.6)
response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=16000,
    thinking={"type": "enabled", "budget_tokens": 8000},
    messages=[...]
)

# New (Opus 4.6 / Sonnet 4.6)
response = client.messages.create(
    model="claude-opus-4-6",  # or "claude-sonnet-4-6"
    max_tokens=16000,
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},  # optional: low | medium | high | max
    messages=[...]
)
```

自適應思考是長期目標，而且在內部評估中優於手動延伸思考。可以遷移時就遷移。

**過渡用應急措施：** 手動延伸思考在 Opus 4.6 和 Sonnet 4.6 上仍然*可運作*（已棄用，未來版本會移除）。如果遷移期間需要硬性上限——例如在調整 `effort` 之前，先限制失控工作負載的 token 花費——可以暫時保留 `budget_tokens`，並搭配明確的 `effort` 值，之後再在後續變更中移除。`budget_tokens` 必須嚴格小於 `max_tokens`：

```python
# Transitional only - deprecated, plan to remove
client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16384,
    thinking={"type": "enabled", "budget_tokens": 8192},  # must be < max_tokens
    output_config={"effort": "medium"},
    messages=[...],
)
```

如果使用者在 4.6 上詢問「thinking 預算」，建議回答 `effort`——使用 `low`、`medium`、`high` 或 `max`，而不是 token 數量。

**2. Effort 參數（僅限 Opus 4.5、Opus 4.6、Sonnet 4.6）。**

控制思考深度與整體 token 花費。它放在 `output_config` 內，不是頂層。預設值是 `high`。Fable 5、Opus 4.6 及後續版本、Sonnet 5 和 Sonnet 4.6 支援 `max`——在 Sonnet 4.5 和 Haiku 4.5 上會發生錯誤。

```python
output_config={"effort": "medium"}  # often the best cost / quality balance
```

### 遷移至 4.6 系列（Opus 4.6 與 Sonnet 4.6）

**3. 助手回合預填會回傳 400（Opus 4.6 與 Sonnet 4.6）。**

最後助手回合的預填回應在 Opus 4.6 和 Sonnet 4.6 上都不再支援——兩者都會回傳 400。在對話的*其他位置*新增助手訊息（例如 few-shot 範例）仍可運作。請依預填原本的用途選擇相符的替代方式：

| 預填用於 | 替代方式 |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| 強制輸出 JSON／YAML／schema | 使用 `output_config.format` 搭配 `json_schema`——請參閱下方範例 |
| 強制分類標籤 | 使用含有效標籤列舉欄位的工具，或使用結構化輸出 |
| 略過前言（`Here is the summary:\n`） | System prompt 指示：「Respond directly without preamble. Do not start with phrases like Here is... or Based on....」 |
| 避開不理想的拒絕 | 通常已不再需要——4.6 的拒絕更恰當。單純的使用者回合 prompt 就足夠。 |
| 繼續被中斷的回應 | 將續接內容移至使用者回合：「Your previous response was interrupted and ended with `[last text]`. Continue from there.」 |
| 注入提醒／內容填充 | 改為注入使用者回合。對複雜的 agent harness，透過工具呼叫或壓縮期間公開內容。 |

```python
# Old (fails on Opus 4.6 / Sonnet 4.6) - prefill forcing JSON shape
messages=[
    {"role": "user", "content": "Extract the name."},
    {"role": "assistant", "content": "{\"name\": \""},
]

# New - structured outputs replace the prefill
response = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=1024,
    output_config={"format": {"type": "json_schema", "schema": {...}}},
    messages=[{"role": "user", "content": "Extract the name."}],
)
```

**4. 對 `max_tokens > ~16K` 使用串流（所有模型）；只有 Haiku 4.5 的上限較低，為 64K。**

無論模型為何，高 `max_tokens` 的非串流請求都會遇到 SDK HTTP 逾時——輸出超過約 16K 時都使用串流。目前每個模型的可串流上限都是 128K，只有 Haiku 4.5 上限為 64K。

```python
with client.messages.stream(model="claude-opus-4-6", max_tokens=64000, ...) as stream:
    message = stream.get_final_message()
```

**5. 工具呼叫的 JSON 跳脫方式可能不同（Opus 4.6 與 Sonnet 4.6）。**

兩個 4.6 模型都可能產生含 Unicode 或斜線跳脫的工具呼叫 `input` 欄位。一律使用 `json.loads()`／`JSON.parse()` 進行剖析——絕不要對序列化輸入做原始字串比對。

### 所有模型

**6. `output_format` -> `output_config.format`（全 API）。**

`messages.create()` 上舊的頂層 `output_format` 參數已棄用。請改用 `output_config.format`。這不僅限於 4.6——適用於每個模型。

---

## 要在 4.6 上移除的 Beta 標頭

4.5 必須使用的數個 beta 標頭，在 4.6 上已正式推出，應予移除。保留它們雖然無害，卻會造成誤導；移除後也能將 SDK 呼叫位置從 `client.beta.messages.create(...)` 改回 `client.messages.create(...)`。

| 標頭 | 4.6 上的狀態 | 動作 |
| ----------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------- |
| `effort-2025-11-24` | Effort 參數已正式推出 | 移除 |
| `fine-grained-tool-streaming-2025-05-14` | 已正式推出 | 移除 |
| `interleaved-thinking-2025-05-14` | 自適應思考會自動啟用交錯思考 | 使用自適應思考時移除；在搭配手動延伸思考的 Sonnet 4.6 上仍可運作，但該路徑已棄用 |
| `token-efficient-tools-2025-02-19` | 已內建於所有 Claude 4 以上模型 | 移除（無作用） |
| `output-128k-2025-02-19` | 已內建於 Claude 4 以上模型 | 移除（無作用） |

移除所有這些標頭並完成遷移至自適應思考後，就可以將 SDK 呼叫位置從 beta 命名空間改回一般命名空間：

```python
# Before
response = client.beta.messages.create(
    model="claude-opus-4-5",
    betas=["interleaved-thinking-2025-05-14", "effort-2025-11-24"],
    ...
)

# After
response = client.messages.create(
    model="claude-opus-4-6",
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},
    ...
)
```

---

## 從 3.x／4.0／4.1 -> 4.6 時的其他變更

如果你是從 Opus 4.1、Sonnet 4、Sonnet 3.7 或較舊的 Claude 3.x 模型直接跳到 4.6，請套用上方所有內容，*再加上*本節項目。已使用 Opus 4.5／Sonnet 4.5 的使用者可以略過本節。

**1. 取樣參數：`temperature` 或 `top_p`，不可同時使用。**

同時傳入兩者會在每個 Claude 4 以上模型上發生錯誤：

```python
# Old (3.x only - errors on 4+)
client.messages.create(temperature=0.7, top_p=0.9, ...)

# New
client.messages.create(temperature=0.7, ...)  # or top_p, not both
```

**2. 更新工具版本。**

4 以上不支援舊版工具。**`type` 和 `name` 欄位都會變更**——`text_editor_20250728` 與 `str_replace_based_edit_tool` 是一組；只更新其中一個而不更新另一個會回傳 400。也請從文字編輯器整合中移除 `undo_edit` 命令：

| 舊版 | 新版 |
| ------------------------------------------------- | ------------------------------------------------------- |
| `text_editor_20250124` + `str_replace_editor` | `text_editor_20250728` + `str_replace_based_edit_tool` |
| `code_execution_*`（較早版本） | `code_execution_20260521` |
| `undo_edit` 命令 | *（不再支援——刪除呼叫位置）* |

```python
# Before
tools = [{"type": "text_editor_20250124", "name": "str_replace_editor"}]

# After - BOTH fields change
tools = [{"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"}]
```

**3. 處理 `refusal` 停止原因。**

Claude 4 以上可能在回應中回傳 `stop_reason: "refusal"`。如果你的程式碼只處理 `end_turn`／`tool_use`／`max_tokens`，請新增一個分支：

```python
if response.stop_reason == "refusal":
    # Surface the refusal to the user; do not retry with the same prompt
    ...
```

**4. 處理 `model_context_window_exceeded` 停止原因（4.5 以上）。**

它不同於 `max_tokens`：表示模型碰到的是*情境視窗*上限，而不是要求的輸出上限。請同時處理兩者：

```python
if response.stop_reason == "model_context_window_exceeded":
    # Context window exhausted - compact or split the conversation
    ...
elif response.stop_reason == "max_tokens":
    # Requested output cap hit - retry with higher max_tokens or stream
    ...
```

**5. 工具呼叫字串參數會保留尾端換行（4.5 以上）。**

4.5 和 4.6 會保留舊模型會移除的尾端換行。如果你的工具實作會對工具呼叫 `input` 值進行精確字串比對（例如 `if name == "foo"`），請確認模型傳送 `"foo\n"` 時仍能比對成功。在接收端使用 `.rstrip()` 正規化通常是最簡單的修正方式。

**6. Haiku：速率限制在世代之間重設。**

Haiku 4.5 有自己的速率限制集區，與 Haiku 3／3.5 分開。如果你在遷移時逐步增加流量，請在[API 速率限制](https://platform.claude.com/docs/en/api/rate-limits)中檢查你層級的 Haiku 4.5 限制——對 Haiku 3.5 流量而言綽綽有餘的配額，在 4.5 上處理相同流量時可能需要提升層級。

---

## Prompt 行為變化（Opus 4.5／4.6、Sonnet 4.6）

這些變更不會破壞程式碼，但在 4.5 及更早版本上有效的 prompt，可能在 4.6 上過度或不足觸發。請依需要調整。若要對本次遷移以外的日期化 prompt 文字進行持續性的模型通用稽核——包括 skill 與工具描述——請閱讀 `shared/prompt-audit.md`（或呼叫 `/claude-api prompt-audit`）。

**1. 強硬指示會造成過度觸發。** Opus 4.5 和 4.6 比早期模型更嚴格遵循 system prompt。為了*克服*舊模型不情願而撰寫的 prompt，現在過於強硬：

| 之前（在 4.0／4.5 上有效） | 之後（在 4.6 上使用） |
| ------------------------------------------- | ----------------------------------------- |
| `CRITICAL: You MUST use this tool when...`  | `Use this tool when...`                   |
| `Default to using [tool]`                   | `Use [tool] when it would improve X`      |
| `If in doubt, use [tool]`                   | *(delete - no longer needed)*             |

如果模型現在對某個工具或 skill 過度觸發，修正方式幾乎總是放寬措辭，而不是增加更多防護。

**2. 過度思考與過度探索（Opus 4.6）。** 在較高的 `effort` 設定下，Opus 4.6 會在回答前探索更多內容。如果這消耗太多 thinking token，請先降低 `effort`（`medium` 通常是最佳平衡點），再加入限制推理的文字指示。

**3. 過度熱衷於產生 subagent（Opus 4.6）。** Opus 4.6 強烈偏好委派給 subagent。如果你看到它為直接使用 `grep` 或 `read` 就能解決的事情產生 subagent，請加入指引：「只有在平行或獨立的工作流中才使用 subagent。對單一檔案讀取或循序操作，直接處理。」

**4. 過度工程化（Opus 4.5／4.6）。** 兩個模型都可能在要求之外新增檔案、抽象層或防禦式錯誤處理。如果你要最小化變更，請在 prompt 中明確說明：「只做直接要求的變更。不要為不可能發生的情境新增 helper、抽象層或錯誤處理。」

**5. LaTeX 數學輸出（Opus 4.6）。** Opus 4.6 預設使用 LaTeX（`\frac{}{}`、`$...$`）處理數學與技術內容。如果需要純文字，請明確指示：「所有數學都使用純文字格式——不要使用 LaTeX、`$` 或 `\frac{}{}`。除法使用 `/`，指數使用 `^`。」

**6. 略過口頭摘要（4.6 系列）。** 4.6 模型更簡潔，可能在工具呼叫後略過摘要段落，直接跳到下一個動作。如果你依賴這些摘要來掌握進度，請加入：「完成涉及工具使用的工作後，提供你所做工作的簡短摘要。」

**7. 「Think」作為觸發詞（停用 thinking 的 Opus 4.5）。** 停用 `thinking` 時，Opus 4.5 對 *think* 這個字特別敏感，可能比你希望的更深入推理。改用 `consider`、`evaluate` 或 `reason through`。

---

## 模型 ID 重新命名快速參考

| 舊字串（遷移來源） | 新字串 |
| ------------------------------ | ------------------ |
| `claude-opus-4-8`              | `claude-opus-5`     |
| `claude-opus-4-7`              | `claude-opus-5`     |
| `claude-opus-4-6`              | `claude-opus-5`     |
| `claude-opus-4-5`              | `claude-opus-5`     |
| `claude-opus-4-1`              | `claude-opus-5`     |
| `claude-opus-4-0`              | `claude-opus-5`     |
| `claude-mythos-preview` | `claude-mythos-5-1`（Project Glasswing）或 `claude-fable-5-1` |
| `claude-fable-5`            | `claude-fable-5-1`     |
| `claude-mythos-5`           | `claude-mythos-5-1`    |
| `claude-sonnet-4-6`            | `claude-sonnet-5`|
| `claude-sonnet-4-5`            | `claude-sonnet-5`|
| `claude-sonnet-4-0`            | `claude-sonnet-5`|

較舊的別名（`claude-opus-4-7`、`claude-opus-4-6`、`claude-opus-4-5`、`claude-sonnet-4-6`、`claude-sonnet-4-5` 等）仍然有效；如果需要時間後再升級，可以暫時釘選——完整的舊版清單請參閱 `shared/models.md`。

### Amazon Bedrock 模型 ID

如果程式碼使用 `AnthropicBedrockMantle` 用戶端（Python `anthropic[bedrock]`、TypeScript `@anthropic-ai/bedrock-sdk`、Java `BedrockMantleBackend`、Go `bedrock.NewMantleClient` 等），或目標為 `https://bedrock-mantle.{region}.api.aws/anthropic`，表示它在 **Amazon Bedrock 上的 Claude** 執行。本指南中的所有重大變更在此都同樣適用——它提供相同的 Messages API 形狀——但模型 ID 會帶有 `anthropic.` 提供者前置詞：

| 第一方 ID | Bedrock ID |
|---|---|
| `claude-opus-4-8` | `anthropic.claude-opus-4-8` |
| `claude-opus-5` | `anthropic.claude-opus-5` |
| `claude-fable-5-1` | `anthropic.claude-fable-5-1` |
| `claude-fable-5` | `anthropic.claude-fable-5` |
| `claude-mythos-5-1` | `anthropic.claude-mythos-5-1` (us-east-1 only, not publicly listed) |
| `claude-opus-4-7` | `anthropic.claude-opus-4-7` |
| `claude-sonnet-5` | `anthropic.claude-sonnet-5` |
| `claude-haiku-4-5` | `anthropic.claude-haiku-4-5` |

遷移 Bedrock 檔案時，套用與第一方相同的重新命名表列，然後保留／新增 `anthropic.` 前置詞。對 Bedrock 用戶端**不要**產生第一方的 `claude-*` ID——那會回傳 400。

**Bedrock 略過：** `code_execution_*` 工具版本檢查清單項目與 **Task Budgets** 章節——兩者在 Bedrock 上都不可用（各功能的表格請參閱 `shared/platform-availability.md`）。本指南的其他所有內容——`effort`、自適應／延伸思考、`output_config.format`、`thinking.display`、細粒度工具串流、token 計數——在 Bedrock 上都可用。

> **範圍外：** 舊版 Amazon Bedrock 整合（使用 ARN 版本化 ID，例如 `anthropic.claude-3-5-sonnet-20241022-v2:0` 的 `InvokeModel`／`Converse` API）使用不同的請求形狀與模型 ID 格式。本指南不涵蓋它；如果使用者正在兩種 Bedrock 整合之間遷移，請對 `shared/live-sources.md` 中的 Bedrock 頁面執行 WebFetch。

### AWS 上的 Claude Platform

如果程式碼使用 `AnthropicAWS`／`AnthropicAws`／`anthropicaws.NewClient`／`AnthropicAwsClient`（或目標為 `https://aws-external-anthropic.{region}.api.aws`），表示它在 **AWS 上的 Claude Platform** 執行——由 Anthropic 操作，API 每日同步。模型 ID 是**不帶前置詞的第一方**字串；逐字套用上方的重新命名表，以及本指南所有重大變更章節。沒有任何內容需要略過。**不要**新增 `anthropic.` 前置詞（那是 Amazon Bedrock，另一項服務）。用戶端／驗證詳細資料請參閱 `shared/claude-platform-on-aws.md`。

---

## 遷移檢查清單

每個項目都有標記：**`[BLOCKS]`** 項目若遺漏，會造成 400 錯誤、無限迴圈、靜默逾時或錯誤的工具選擇——請將這些視為程式碼編輯，而不是建議。**`[TUNE]`** 項目是品質／成本調整。

對每個呼叫 `messages.create()`／相等 SDK 方法的檔案：

- [ ] **[BLOCKS]** 將 `model=` 字串更新為新別名
- [ ] **[BLOCKS]** 將 `budget_tokens` 替換為 `thinking={"type": "adaptive"}`（在 Opus 4.6／Sonnet 4.6 上已棄用）
- [ ] **[BLOCKS]** 將 `format` 從頂層 `output_format` 移至 `output_config.format`
- [ ] **[BLOCKS]** 如果目標是 Opus 4.6 或 Sonnet 4.6，移除所有助手回合預填（請參閱預填替代表）
- [ ] **[BLOCKS]** 如果 `max_tokens > ~16000`，切換至串流（否則 SDK HTTP 逾時）
- [ ] **[TUNE]** 確認工具輸入處理會剖析 JSON，而不是對序列化輸入做原始字串比對（4.6 可能以不同方式跳脫 Unicode／斜線；大多數 SDK 已將 `block.input` 公開為已剖析的物件）
- [ ] **[TUNE]** 明確設定 `output_config={"effort": "..."}`——尤其是從 Sonnet 4.5 移至 Sonnet 4.6 時（4.6 預設為 `high`）
- [ ] **[TUNE]** 移除已正式推出的 beta 標頭：`effort-2025-11-24`、`fine-grained-tool-streaming-2025-05-14`、`token-efficient-tools-2025-02-19`、`output-128k-2025-02-19`；使用自適應思考後移除 `interleaved-thinking-2025-05-14`
- [ ] **[TUNE]** 移除所有 beta 後，將 `client.beta.messages.create(...)` -> `client.messages.create(...)`
- [ ] **[TUNE]** 檢查 system prompt 中強硬的工具文字（`CRITICAL:`、`MUST`、`If in doubt`），並放寬語氣

**從 3.x／4.0／4.1 開始時的額外項目：**
- [ ] **[BLOCKS]** 移除 `temperature` 或 `top_p` 其中之一（兩者同時傳入會在 Claude 4 以上回傳 400）
- [ ] **[BLOCKS]** 將文字編輯器工具的 `type` 更新為 `text_editor_20250728`
- [ ] **[BLOCKS]** 將文字編輯器工具的 `name` 更新為 `str_replace_based_edit_tool`——**只變更 `type` 而保留 `name: "str_replace_editor"` 會回傳 400**
- [ ] **[BLOCKS]** 將程式碼執行工具更新為 `code_execution_20260521`
- [ ] **[BLOCKS]** 刪除所有 `undo_edit` 命令呼叫位置
- [ ] **[TUNE]** 新增 `stop_reason == "refusal"` 的處理
- [ ] **[TUNE]** 新增 `stop_reason == "model_context_window_exceeded"` 的處理（4.5 以上）
- [ ] **[TUNE]** 確認工具參數字串比對可容許尾端換行（4.5 以上會保留）
- [ ] **[TUNE]** 如果要移至 Haiku 4.5，檢查速率限制層級（與 Haiku 3.x 分開的集區）

**驗證：**
- [ ] 執行一次測試請求，檢查 `response.stop_reason`、`response.usage`，以及工具使用／thinking 行為是否符合預期

對快取的 prompt：渲染順序與雜湊輸入沒有變更，因此現有的 `cache_control` 斷點仍可運作。不過，**變更模型字串會使現有快取失效**——新模型的第一個請求會重新寫入快取。

---

## 遷移至 Opus 4.7

> **本指南寫出的模型 ID `claude-opus-4-7` 即為權威值。** 使用者要求遷移至 Opus 4.7 時，請原樣寫入 `model="claude-opus-4-7"`。**不要**使用 WebFetch 驗證——本指南是遷移目標 ID 的真實來源。對應項目存在於 `shared/models.md`。

Claude Opus 4.7 在發布時是 Anthropic 能力最強的模型，如今則是前一代 Opus（目前版本是 Opus 4.8——請參閱下方的「遷移至 Opus 4.8」）。它高度自主，在長期 agent 式工作、知識工作、視覺工作與記憶工作上表現出色。本節總結 4.7 發布時的新內容，也仍是從 Opus 4.6 或更早版本來的呼叫端所需遵循的分層重大變更路徑。本節建立在上方 4.6 遷移之上——如果呼叫端要從 Opus 4.5 或更早版本直接跳轉，先套用 4.6 變更，再套用本節，最後套用 4.8 章節。

**已使用 Opus 4.6 者的摘要：** 將模型 ID 更新為 `claude-opus-4-7`，移除任何剩餘的 `budget_tokens` 與取樣參數（兩者在 Opus 4.7 上都會回傳 400），為 `max_tokens` 預留更多空間，並使用新模型重新以 `count_tokens()` 建立基準；如果會向使用者呈現推理，重新啟用 `thinking.display: "summarized"`；重新調整 `effort`——它在 4.7 上比任何先前的 Opus 都更重要。

### 重大變更（在 Opus 4.7 上會回傳 400）

**已移除延伸思考。**

`thinking: {type: "enabled", budget_tokens: N}` 不再受到 Claude Opus 4.7 或後續模型支援，並會回傳 400 錯誤。切換至自適應思考（`thinking: {type: "adaptive"}`），再使用 effort 參數控制思考深度。Claude Opus 4.7 預設**關閉**自適應思考：沒有 `thinking` 欄位的請求會在不思考的情況下執行，與 Opus 4.6 行為相同。請明確設定 `thinking: {type: "adaptive"}` 以啟用。

```python
# Before (Opus 4.6)
client.messages.create(
    model="claude-opus-4-6",
    max_tokens=64000,
    thinking={"type": "enabled", "budget_tokens": 32000},
    messages=[{"role": "user", "content": "..."}],
)

# After (Opus 4.7)
client.messages.create(
    model="claude-opus-4-7",
    max_tokens=64000,
    thinking={"type": "adaptive"},
    output_config={"effort": "high"},  # or "max", "xhigh", "medium", "low"
    messages=[{"role": "user", "content": "..."}],
)
```

如果呼叫端沒有使用延伸思考，則不需要變更——thinking 預設關閉，也可以明確設定 `thinking={"type": "disabled"}`。

完全刪除 `budget_tokens` 的傳遞邏輯。替代用的 `effort` 值請參閱下方的 **在 Opus 4.7 上選擇 effort 層級**——`budget_tokens` 沒有精確的一對一對應。

**已移除取樣參數。**

`temperature`、`top_p` 和 `top_k` 參數在 Claude Opus 4.7 上不再接受。包含這些參數的請求會回傳 400 錯誤。請從請求承載中移除這些欄位。在 Claude Opus 4.7 上，建議使用 prompt 引導模型行為。如果你使用 `temperature = 0` 來追求決定性，請注意它在舊模型上也從未保證輸出完全相同。

```python
# Before - errors on Opus 4.7
client.messages.create(temperature=0.7, top_p=0.9, ...)

# After
client.messages.create(...)  # no sampling params
```

- **如果目的是決定性**——使用 `effort: "low"` 搭配更嚴謹的 prompt。
- **如果目的是創意變化**——prompt 替代方式取決於使用情境；**請詢問使用者**希望如何產生變化。如果無法詢問，請加入符合情境的指示，例如*「選擇偏離分布且有趣的內容」*——文字產生可用*「在不同回應中變化措辭與結構」*；前端／設計則使用下方**設計與前端 coding**中的提出 4 個方向方法。

### 在 Opus 4.7 上選擇 effort 層級

`budget_tokens` 控制要*思考*多少；`effort` 控制要思考多少以及要*執行*多少，因此沒有精確的一對一對應。**coding 與 agent 式使用情境要獲得最佳結果，請使用 `xhigh`；大多數對智慧敏感的使用情境至少使用 `high`。** 嘗試其他層級，進一步調整 token 使用量與智慧：

| 層級 | 使用時機 | 備註 |
| --- | --- | --- |
| `max` | 值得在上限測試的智慧密集型工作 | 某些使用情境可能獲得提升，但 token 使用量增加時可能收益遞減，也容易過度思考 |
| `xhigh` | **大多數 coding 與 agent 式使用情境** | 最適合這些情境的設定；Claude Code 的預設值 |
| `high` | 一般對智慧敏感的使用情境 | 平衡 token 使用量與智慧；大多數對智慧敏感的工作建議最低使用此層級 |
| `medium` | 需要減少 token 使用量、願意用智慧換取成本的成本敏感情境 | |
| `low` | 短小、範圍明確且對延遲敏感、但不對智慧敏感的工作負載 | |

### 靜默的預設值變更（沒有錯誤，但行為不同）

**思考內容預設省略。**

思考區塊仍會在 Claude Opus 4.7 的回應串流中出現，但除非你明確選擇，`thinking` 欄位會是空的。這是相對於 Claude Opus 4.6 的靜默變更；4.6 的預設值是回傳摘要思考文字。若要在 Claude Opus 4.7 上恢復摘要思考內容，請將 `thinking.display` 設為 `"summarized"`。**區塊欄位名稱未變更**——`thinking` 類型區塊仍是 `block.thinking`；不要重新命名。

**偵測方式：** 任何從 `thinking` 類型區塊讀取 `block.thinking`（或相等欄位）並呈現在 UI、記錄或追蹤中的程式碼。**修正的是請求參數，不是回應處理**——將 `display: "summarized"` 加到 `thinking` 參數：

```python
thinking={"type": "adaptive", "display": "summarized"}  # "display" is new on Opus 4.7; values: "omitted" (default) | "summarized"
```

Claude Opus 4.7 的預設值是 `"omitted"`。如果 thinking 內容從未在任何地方呈現，不需要變更。如果產品會將推理串流給使用者，新預設值會看起來像是在輸出開始前長時間暫停；請設定 `display: "summarized"`，恢復思考期間可見的進度。

**更新 token 計數。**

Claude Opus 4.7 和 Claude Opus 4.6 的 token 計數方式不同。同一段輸入文字在 Claude Opus 4.7 上產生的 token 數高於 Claude Opus 4.6，而 `/v1/messages/count_tokens` 對 Claude Opus 4.7 回傳的 token 數也會不同於對 Claude Opus 4.6 回傳的數值。Claude Opus 4.7 的 token 效率可能隨工作負載形狀而變化。Prompt 調整、`task_budget` 與 `effort` 有助於控制成本並確保適當的 token 使用量。請注意，這些控制可能會犧牲模型智慧。**更新 `max_tokens` 參數以預留更多空間，包括壓縮觸發條件。** Claude Opus 4.7 在標準 API 定價下提供 1M 情境視窗，沒有長情境 premium。

其他要檢查的項目：

- 以 4.6 為基準的用戶端 token 估算器（tiktoken 風格的近似值）
- 將 token 乘以固定每 token 費率的成本計算器
- 以測得的 token 數為依據的速率限制重試閾值

對呼叫端 prompt 的代表性樣本，使用 `claude-opus-4-7` 重新執行 `client.messages.count_tokens()` 建立基準。不要套用一律的乘數。對成本敏感的工作負載，可以考慮將 `effort` 降低一級（例如 `high` -> `medium`）。對 agent 式迴圈，可以考慮採用下方的 Task Budgets。

### 新功能：Task Budgets（beta）

Opus 4.7 引入**工作任務預算**——告訴 Claude 完整 agent 式迴圈（思考 + 工具呼叫 + 最終輸出）有多少 token。模型會看到持續倒數，並在預算消耗時用它來排定工作優先順序、從容收尾。

這是模型知悉的**建議**，不是硬性上限。它不同於 `max_tokens`；後者仍是每次回應的強制上限，而且*不會*公開給模型。想讓模型自我節制時使用 `task_budget`；想以硬性上限限制使用量時使用 `max_tokens`。

需要 beta 標頭 `task-budgets-2026-03-13`：

```python
client.beta.messages.create(
    betas=["task-budgets-2026-03-13"],
    model="claude-opus-4-7",
    max_tokens=64000,
    thinking={"type": "adaptive"},
    output_config={
        "effort": "high",
        "task_budget": {"type": "tokens", "total": 128000},
    },
    messages=[...],
)
```

開放式 agent 式工作使用寬裕預算，對延遲敏感的工作則收緊。**`task_budget.total` 的最小值是 20,000 token。** 如果預算對工作過於嚴格，模型可能完成得不夠徹底，並把預算視為限制。**除非確定預算值正確，否則遷移期間不要新增 `task_budget`**——如果能執行工作負載並測量，就這樣做；否則請詢問使用者預算值，不要猜測。這是抵銷 agent 式工作負載 token 計數變化的主要手段。

### 能力改進

**高解析度視覺。** Opus 4.7 是第一個支援高解析度影像的 Claude 模型。影像長邊的最大解析度是 **2576 像素**（高於 Opus 4.6 及更早版本的 1568px）。這會提升視覺密集型工作負載的效果，尤其是電腦使用與螢幕擷取／artifact／文件理解。模型回傳的座標現在會一對一對應實際影像像素，因此不需要縮放因數計算。

高解析度支援在 Opus 4.7 上**自動啟用**——不需要 beta 標頭，也不需要用戶端選擇。模型開箱即接受較大的輸入並回傳像素精確的座標。

**Token 成本。** Opus 4.7 上的全解析度影像最多可能使用先前模型約 3 倍的影像 token（每張最多約 4784 token，相較於先前約 1,600 token 的上限）。如果不需要額外保真度，可以在傳送前於用戶端縮小影像以控制成本——但**遷移期間不要預設新增縮小處理**。如果不確定管線是否需要這種保真度，請詢問使用者，不要猜測。在 Opus 4.7 上對代表性影像使用 `count_tokens()` 建立新基準，再回應測得的成本變化。

除了解析度外，Opus 4.7 也改善低階感知（指向、測量、計數）以及自然影像的邊界框定位與偵測。

**知識工作。** 在模型會以視覺驗證自身輸出的工作上有明顯提升——`.docx` 修訂、`.pptx` 編輯，以及以程式分析圖表／圖形（例如使用影像處理程式庫進行像素層級的資料轉錄）。如果 prompt 含有「回傳前再次檢查投影片版面」之類的腳手架，請嘗試移除後重新建立基準。

**記憶。** Opus 4.7 更擅長寫入與使用檔案系統型記憶。如果 agent 在回合之間維護 scratchpad、筆記檔或結構化記憶儲存區，該 agent 應該更能自行記下筆記並在後續工作中運用。

**面向使用者的進度更新。** Opus 4.7 在長時間 agent 軌跡中提供更規律、品質更高的中途更新。如果 system prompt 含有「每 3 次工具呼叫後摘要進度」之類的腳手架，請嘗試移除以避免面向使用者的文字過多。如果 Opus 4.7 更新的長度或內容不符合使用情境，請在 prompt 中明確描述更新應呈現的樣子並提供範例。

### 即時網路安全防護

涉及禁止或高風險主題的請求可能導致拒絕。

### Fast Mode：僅限 Claude Opus 5／Opus 4.8

Fast mode 在 Claude Opus 5 和 Opus 4.8 上可用。只有當呼叫端程式碼實際使用 fast mode 時才提出此內容（例如 `model="claude-opus-4-6-fast"`，或在不支援的模型上使用 `speed="fast"`）；如果程式碼中沒有出現「fast」一詞，就不要提及 Fast Mode。

看到 `model="claude-opus-4-6-fast"`（或任何已退役的 `-fast` 模型字串）時，**遷移編輯是**將 fast-mode 流量移至 Claude Opus 5，目前支援 fast 的預設模型（如果呼叫端要留在該層級，Opus 4.8 也可用）：

```python
# Request fast mode on Claude Opus 5.
client.beta.messages.create(
    model="claude-opus-5", max_tokens=4096,
    speed="fast", betas=["fast-mode-2026-02-01"],
    messages=[...],
)
```

也就是：將模型切換至 Claude Opus 5（或 Opus 4.8），以支援的方式要求 fast mode，使用 beta `client.beta.messages....` 端點、`fast-mode-2026-02-01` beta 旗標，以及頂層請求參數 `speed="fast"`（各語言形式請見 SKILL.md § Fast Mode）。Opus 4.7 fast mode 也已移除，因此也不要停留在 Opus 4.7。**不要**將程式碼留在已退役的 `-fast` 模型字串上——失敗方式依版本而不同：`claude-opus-4-6-fast` 已退役，API 會**靜默回退**至標準 Opus 4.6（不會報錯，呼叫端會在未察覺的情況下失去 fast-mode 速度）；`claude-opus-4-7-fast` 以及在 Opus 4.7 上使用 `speed="fast"` 則會回傳**API 錯誤**（硬性失敗——請求會直接中斷，不會降級）。無論哪種情況，現在都遷移至 Opus 4.8 fast mode。

### 行為變化（可透過 prompt 調整）

這些變更不會破壞任何內容，但為 Opus 4.6 調整的 prompt 可能產生不同結果。Opus 4.7 比 4.6 更容易引導，因此小幅 prompt 調整通常就能縮小差距。

**更照字面遵循指示。** Claude Opus 4.7 比 Claude Opus 4.6 更照字面且明確地解讀 prompt，尤其是在較低 effort 層級。它不會把對一個項目的指示靜默泛化到另一個項目，也不會推斷你沒有提出的要求。這種字面主義的優點是精準與較少反覆。對 prompt 經過仔細調整的 API 使用、結構化擷取，以及需要可預測行為的管線，它通常表現更好。遷移至 Claude Opus 4.7 時，檢查 prompt 與 harness 可能特別有幫助。

**冗長度會依工作複雜度校準。** Opus 4.7 會依它判斷的工作複雜度調整回應長度，而不是預設固定冗長度——簡單查詢的回答較短，開放式分析則長得多。如果產品依賴特定長度或風格，請明確調整 prompt。要降低冗長度：

> *"Provide concise, focused responses. Skip non-essential context, and keep examples minimal."*

如果看到特定類型的過度冗長（例如過度解釋），請加入針對那些問題的指示。顯示理想簡潔程度的正面範例，通常比負面範例或告訴模型不要做什麼的指示更有效。**不要**假設現有的「保持簡潔」指示應該移除——先測試。

**語氣與寫作風格。** Opus 4.7 更直接、更有主見，較少使用 Opus 4.6 溫暖風格中的先肯定式措辭和 emoji。與任何新模型一樣，長文寫作的散文風格可能改變。如果產品依賴特定聲音，請以新的基準重新評估風格 prompt。如果需要更溫暖或更口語的聲音，請明確指定：

> *"Use a warm, collaborative tone. Acknowledge the user's framing before answering."*

**`effort` 比任何先前的 Opus 都更重要。** Opus 4.7 更嚴格遵循 `effort` 層級，尤其是在低端。在 `low` 和 `medium` 下，它會將工作範圍限制在要求內容，而不是額外擴充——這有利於延遲和成本，但中等工作在 `low` 下有思考不足的風險。

- 如果複雜問題出現膚淺推理，請將 `effort` 提高至 `high` 或 `xhigh`，不要用 prompt 迂迴處理。
- 如果為了延遲必須維持 `effort` 為 `low`，請加入針對性的指引：「這項工作需要多步驟推理。回答前請仔細思考整個問題。」
- **在 `xhigh` 或 `max` 下，請設定大的 `max_tokens`**，讓模型有空間在工具呼叫與 subagent 之間思考和執行。從 64K 開始再調整。（`xhigh` 是 Opus 4.7 的新 effort 層級，介於 `high` 和 `max` 之間。）

自適應思考的觸發方式也可以引導。如果模型思考頻率高於預期——大型或複雜的 system prompt 可能造成這種情況——請加入：「思考會增加延遲，只有在能有意義地改善回答品質時才應使用——通常是需要多步驟推理的問題。不確定時，直接回答。」

**預設較少使用工具。** Opus 4.7 傾向比 4.6 少用工具、更多使用推理。大多數情況下結果更好，但依賴工具的產品（搜尋／擷取、函式呼叫、電腦使用步驟）可能會降低工具使用率。可用兩個手段：

- **提高 `effort`**——`high` 或 `xhigh` 在 agent 式搜尋與 coding 中會顯著增加工具使用，對知識工作尤其有用。
- **用 prompt 要求**——在工具描述或 system prompt 中明確說明何時、如何使用工具，並鼓勵模型多使用工具：

> *"When the answer depends on information not present in the conversation, you MUST call the `search` tool before answering - do not answer from prior knowledge."*

**預設較少使用 subagent。** Opus 4.7 傾向比 4.6 產生更少 subagent。這可以引導——明確說明何時適合委派。例如對 coding agent：

> *"Do NOT spawn a subagent for work you can complete directly in a single response (e.g. refactoring a function you can already see). Spawn multiple subagents in the same turn when fanning out across items or reading multiple files."*

**設計與前端 coding。** Opus 4.7 的設計直覺比 4.6 更強，具有一致的預設樣式：溫暖奶油／米白背景（約 `#F4F1EA`）、襯線標題字體（Georgia、Fraunces、Playfair）、斜體文字強調，以及陶土／琥珀色點綴。這對編輯、旅宿與作品集簡介效果良好，但用於儀表板、開發工具、金融科技、醫療或企業應用程式會顯得不合適——投影片也會像網頁 UI 一樣出現這種風格。

預設樣式會持續存在。一般指示（「不要使用奶油色」、「做得乾淨簡約」）通常會把模型推向另一套固定色盤，而不會產生多樣性。兩種方法可靠：

1. **指定具體替代方案。** 模型會精確遵循明確規格——提供確切的十六進位色碼、字體與版面限制。
2. **讓模型在建立前提出選項。** 這會打破預設，讓使用者掌握控制權：

   > *"Before building, propose 4 distinct visual directions tailored to this brief (each as: bg hex / accent hex / typeface - one-line rationale). Ask the user to pick one, then implement only that direction."*

如果呼叫端先前依賴 `temperature` 產生設計多樣性，請使用方法 (2)——每次執行都會產生有實質差異的方向。

Opus 4.7 也比先前模型需要更少的前端設計 prompt，便能避免一般化的「AI slop」美學。早期模型需要冗長的反 slop 片段；Opus 4.7 只需更短的提醒，就能產生獨特、有創意的前端。這段內容與上述多樣性方法搭配效果良好：

> *"NEVER use generic AI-generated aesthetics like overused font families (Inter, Roboto, Arial, system fonts), cliched color schemes (particularly purple gradients on white or dark backgrounds), predictable layouts and component patterns, and cookie-cutter design that lacks context-specific character. Use unique fonts, cohesive colors and themes, and animations for effects and micro-interactions."*

**互動式 coding 產品。** Opus 4.7 在單一使用者回合的自主、非同步 coding agent，與多個使用者回合的互動、同步 coding agent 之間，token 使用量和行為可能不同。具體來說，在互動設定中它傾向使用更多 token，主要是因為它會在使用者回合後進行更多推理。這能改善長期互動 coding 工作階段的長期一致性、遵循指示與 coding 能力，但也會增加 token 使用量。要同時最大化 coding 產品的效能與 token 效率，請使用 `effort: "xhigh"` 或 `"high"`、加入自主功能（例如自動模式），並減少使用者需要的人工互動次數。

限制必要的使用者互動時，請在第一個人類回合預先指定工作、意圖與相關限制。事先提供明確、完整且正確的工作描述，有助於在減少使用者回合後額外 token 使用量的同時最大化自主性與智慧——因為 Opus 4.7 比先前模型更自主，這種使用模式有助於提升效能。相反地，透過多個使用者回合逐步傳遞的模糊或不完整 prompt，往往會降低 token 效率，有時也降低效能。

**程式碼審查。** Opus 4.7 找出錯誤的能力明顯優於先前模型，召回率與精確度都更高。不過，如果程式碼審查 harness 是為較早模型調整的，初期可能顯示*較低*召回率——這很可能是 harness 效應，而不是能力退化。當審查 prompt 說「只回報高嚴重度問題」、「保持保守」或「不要挑剔」時，Opus 4.7 比早期模型更忠實地遵循：它同樣徹底調查並找出錯誤，然後拒絕回報它判定低於指定門檻的發現。精確度提高，但測得的召回率可能下降，即使底層找錯能力已改善。

建議的 prompt 文字：

> *"Report every issue you find, including ones you are uncertain about or consider low-severity. Do not filter for importance or confidence at this stage - a separate verification step will do that. Your goal here is coverage: it is better to surface a finding that later gets filtered out than to silently drop a bug. For each finding, include your confidence level and an estimated severity so a downstream filter can rank them."*

不必有實際的第二步也可以使用這段內容，但把信心過濾移出發現步驟通常更有幫助。如果 harness 有獨立的驗證／去重／排名階段，請明確告訴模型，在發現階段它的工作是涵蓋，而不是過濾。如果希望單次自我過濾，請具體說明門檻，不要使用「重要」這類定性詞——例如「回報任何可能造成行為錯誤、測試失敗或誤導性結果的錯誤；只有純風格或命名偏好的挑剔項目才略過。」針對部分評估反覆調整 prompt，以驗證召回率或 F1 的提升。

**電腦使用。** 電腦使用支援新上限 2576px／3.75MP 以下的各種解析度。以 **1080p** 傳送影像能在效能與成本間取得良好平衡。對特別重視成本的工作負載，**720p** 或 **1366×768** 是成本較低且效能良好的選項。請測試以找出使用情境的理想設定；試驗 `effort` 也有助於調整行為。

---

## Opus 4.7 遷移檢查清單

每個項目都有標記：如果遺漏，**`[BLOCKS]`** 項目會造成 400 錯誤、無限迴圈、靜默截斷或空輸出——請將這些視為程式碼編輯，而不是建議。**`[TUNE]`** 項目是品質／成本調整——請以建議的形式呈現給使用者。

以 **「If...」** 或 **「At...」** 開頭的 `[BLOCKS]` 項目是有條件的。在處理清單前，先掃描檔案確認條件：是否會將 thinking 文字呈現給 UI／記錄？是否將 `output_config.effort` 設為 `"x-high"` 或 `"max"`？是否為安全工作負載？是否為多回合 agent 式迴圈？只套用符合條件的項目。

- [ ] **[BLOCKS]** 將 `thinking: {type: "enabled", budget_tokens: N}` 替換為 `thinking: {type: "adaptive"}` + `output_config.effort`；完全刪除 `budget_tokens` 傳遞邏輯
- [ ] **[BLOCKS]** 從請求建立邏輯中移除 `temperature`、`top_p`、`top_k`
- [ ] **[BLOCKS]** 如果 thinking 內容會呈現給使用者或儲存在記錄中：新增 `thinking.display: "summarized"`（否則呈現的文字會是空的）
- [ ] **[BLOCKS]** 在 `output_config.effort` 為 `xhigh` 或 `max` 時：將 `max_tokens` 設為 >= 64000（否則輸出會在思考中途截斷）
- [ ] **[TUNE]** 為 `max_tokens` 與壓縮觸發條件預留更多空間；針對代表性 prompt，對 `claude-opus-4-7` 重新執行 `count_tokens()` 建立新基準（不要套用一律的乘數）
- [ ] **[TUNE]** 在回應測得的變化**之前**，重新建立成本與速率限制儀表板的基準
- [ ] **[TUNE]** 逐個路由重新評估 `effort`——coding／agent 式工作使用 `xhigh`，大多數對智慧敏感的工作至少使用 `high`；它在 4.7 上比任何先前的 Opus 都更重要
- [ ] **[TUNE]** 多回合 agent 式迴圈：採用 API 原生 Task Budgets（`output_config.task_budget`、beta `task-budgets-2026-03-13`，最小 20k token）——這用於限制迴圈的*累積*花費；每回合深度由 `effort` 控制
- [ ] **[TUNE]** 檢查是否有依賴 4.6 泛化意圖的模糊或不完整指示，並更新為更清楚或精確的指示——4.7 會照字面遵循
- [ ] **[TUNE]** 工具使用工作負載：在工具描述中加入明確的何時／如何使用指引（4.7 較少主動使用工具）
- [ ] **[TUNE]** 冗長度：變更現有長度指示前先測試——4.7 會依工作複雜度校準長度，因此要針對所需輸出調整，不要預設某個方向
- [ ] **[TUNE]** 移除強制進度更新腳手架（*「每 N 次工具呼叫後……」*）
- [ ] **[TUNE]** 移除知識工作驗證腳手架（*「再次檢查投影片版面……」*）並重新建立基準
- [ ] **[TUNE]** 如果需要更溫暖／更口語的聲音，加入語氣指示；在寫作密集型路由上重新評估風格 prompt
- [ ] **[TUNE]** 存在 subagent 工具時：加入明確的產生／不要產生指引
- [ ] **[TUNE]** 前端／設計輸出：指定具體色盤／字體，或在建立前讓模型提出 4 個視覺方向（預設奶油／襯線樣式會持續存在）
- [ ] **[TUNE]** 互動式 coding 產品：使用 `effort: "xhigh"` 或 `"high"`，加入自主功能（例如自動模式）以減少人類互動，並在第一回合預先指定工作／意圖／限制
- [ ] **[TUNE]** 程式碼審查 harness：移除或放寬「只回報高嚴重度」／「保持保守」過濾，讓模型以信心 + 嚴重度回報每個發現；將過濾移至下游步驟（4.7 更照字面遵循嚴重度過濾，可能降低測得的召回率）
- [ ] **[TUNE]** 視覺密集型管線（螢幕擷取、圖表、文件理解）：為了準確度提升，保留影像原生解析度至長邊 2576px；從座標處理中移除任何縮放因數計算（座標現在與像素一對一）。不需要 beta 標頭／選擇——高解析度在 Opus 4.7 上自動啟用。
- [ ] **[TUNE]** 電腦使用管線：以 1080p 傳送螢幕擷取，以取得良好的效能／成本平衡（成本敏感工作負載可使用 720p 或 1366×768）；試驗 `effort` 以調整行為
- [ ] **[TUNE]** 成本敏感的影像管線：4.7 的全解析度影像最多使用約 4784 token，而先前模型約 1,600 token（約 3 倍）。上傳前在用戶端縮小可避免增加，但**不要預設縮小**——如果不確定是否需要保真度，請詢問使用者。對代表性影像使用 `count_tokens()` 建立基準，再回應成本變化。

---

## 遷移至 Opus 4.8

> **本指南寫出的模型 ID `claude-opus-4-8` 即為權威值。** 使用者要求遷移至 Opus 4.8 時，請原樣寫入 `model="claude-opus-4-8"`。**不要**使用 WebFetch 驗證——本指南是遷移目標 ID 的真實來源。對應項目存在於 `shared/models.md`。

Claude Opus 4.8 是我們能力最強的 Opus 層級模型——高度自主，具備最先進的長期 agent 式執行、知識工作與記憶能力。它建立在上方的 Opus 4.7 遷移之上。如果呼叫端要從 Opus 4.6 或更早版本跳轉，先套用 4.6 和 4.7 章節，再套用本節。

**沒有新的重大變更。** Opus 4.8 保持與 Opus 4.7 相同的請求介面。已在 4.7 上運作的相同呼叫在 4.8 上都能原樣運作——只使用自適應思考（`thinking: {type: "enabled", budget_tokens: N}` 仍會回傳 400；請使用 `{type: "adaptive"}`），取樣參數（`temperature`、`top_p`、`top_k`）仍被拒絕，最後助手回合預填仍會回傳 400，`thinking.display` 仍預設為 `"omitted"`，而 `low`／`medium`／`high`／`xhigh`／`max` effort 層級、Task Budgets（beta）與高解析度視覺的行為都與 4.7 相同。因此 4.7 -> 4.8 遷移就是**替換模型 ID 加上重新調整 prompt**——除模型字串外，沒有必要的程式碼編輯。

**已使用 Opus 4.7 者的摘要：** 將模型 ID 替換為 `claude-opus-4-8`。不需要其他變更即可避免錯誤。接著針對行為變化重新調整 prompt：4.8 比 4.7 產生*更多*敘述（若想保持 4.7 式簡潔，可加入安靜預設）；以更溫暖、較少保留的聲音寫作；更加審慎且更常詢問（加入自主指引以拉回詢問率）；對搜尋、subagent、檔案型記憶與自訂工具更保守（加入明確的「何時使用」觸發條件）。對長期 agent 式工作，在一個完整規格、定義良好的回合中預先提供完整工作說明，並以高 effort 執行。

### 沒有新的 API 重大變更（承襲自 4.7）

這些內容都從 Opus 4.7 原樣延續——只有在呼叫端來自 Opus 4.6 或更早版本時才套用（前後範例與 SDK 特定語法請參閱上方的**遷移至 Opus 4.7**章節）：

- `thinking: {type: "enabled", budget_tokens: N}` -> 400。使用 `thinking: {type: "adaptive"}` + `output_config.effort`。
- `temperature`、`top_p`、`top_k` -> 400。移除它們，改用 prompt 引導。
- 最後 assistant 回合預填 -> 400。使用 `output_config.format`（結構化輸出）或 system prompt 指示。
- `thinking.display` 預設為 `"omitted"`；如果要將推理呈現給使用者，設定為 `"summarized"`。

如果呼叫端已使用 Opus 4.7 且這些項目都乾淨，這裡不需要變更。

### 新 API 功能：工作階段中途的 system prompt

你可以將可信指示直接放入 `messages` 陣列的 `{"role": "system", ...}` 項目，在工作階段中途傳遞——不必編輯頂層 system prompt，也不會使 prompt 快取失效。可用於應用程式在工作階段中途得知的事項：使用者提供了非同步內容、切換模式（已啟用自動核准）、磁碟上的檔案已變更、剩餘 token 預算下降。

```python
messages=[
    {"role": "user", "content": [{"type": "tool_result", "tool_use_id": "...", "content": "..."}]},
    {"role": "system", "content": "This project's codebase is Go. Write code in Go."},
]
```

請將這些內容寫成**情境，而不是命令**。陳述事實，讓 Claude 依此行動；避免覆寫式文字（「忽略使用者說的話」、「無論使用者的請求為何」、「不理會先前的指示」）。Claude 受過訓練，會保護使用者免於看似與其利益相悖的指示；這項保護也適用於 system 角色。不需要 beta 標頭；Claude Opus 4.8 可用。如需快取放置詳細資料及較舊模型的 `<system-reminder>` 備援方式，請參閱 `shared/prompt-caching.md` 和 `shared/agent-design.md`。

### 能力改進

**長期 agent 式執行。** Opus 4.8 在長時間、自主的 agent 式工作上處於最先進水準——能完成不需人類修正的複雜重構與通宵 coding 執行。要充分發揮它的能力，**在一個定義清楚的初始回合中預先提供完整工作規格，並以高 effort 執行**（`effort: "high"` 或 `"xhigh"`）。它的長期一致性部分來自每一步進行更多推理；配合清楚的預先目標，這種更聰明的規劃通常比先前的前沿模型產生更有效率、*也*更準確的輸出。「預先提供清楚目標」原則對應兩個產品介面：在 Claude Code 中，`/goal` 為執行設定方向；使用 **Managed Agents（CMA）** 時，透過 **Outcome**（含可評分規準的 `user.define_outcome`——harness 會執行 iterate -> grade -> revise 迴圈）說明「完成」的樣子，請參閱 `shared/managed-agents-outcomes.md`。

**Effort 是要測試的維度，不是固定設定。** 先前模型中，許多使用者會直覺使用 `xhigh` 來最大化智慧。Opus 4.8 的智慧上限更高，因此**先以 `high` 作為預設並反覆調整**，不要預設使用 `xhigh`。在自己的評估集上測試 `medium`、`high` 和 `xhigh`，依路由衡量智慧 <-> 延遲 <-> 成本取捨——關係不是單調的：較高 effort 前期常能在 agent 式工作上*減少*回合數與總成本，而某些工作使用 `medium` 就能在更短時間內得到同等結果。將 `max` 保留給極度困難且不在意延遲的情境。上方 **遷移至 Opus 4.7** 的各層級 effort 表在 4.8 上原樣適用。

**寫作聲音與清晰度。** 測試者一致認為 4.8 的散文比先前模型更清楚、更溫暖、較少保留，且可測得的 AI 口頭慣性更少——尤其是在較高 effort 下，接近專家級散文與結構。這大致是 4.7 變化的**相反方向**（4.7 更簡短直接，也較少先肯定）。如果你為了對抗 4.7 的簡短，或為了注入溫暖而新增風格 prompt，保留前請以新基準重新評估——它們現在可能矯枉過正。4.8 也是更強的思考夥伴：更深思熟慮、更願意提出異議，也更可能從情境推斷正確答案。

**程式碼審查與偵錯。** 找出真實錯誤的能力與解釋清晰度都比 4.7 更強——4.7 需要更多嘗試的一次修正，4.8 可以完成；也能正確辨識間歇性 flaky 測試，而不是在一次乾淨執行後宣告「已修好」。4.7 的注意事項仍適用：如果審查 harness 說「只回報高嚴重度問題」或「保持保守」，4.8 會照字面遵循，即使底層找錯能力提高，測得的召回率仍可能下降。請告訴模型回報所有內容並在下游過濾（或再審查一次）——建議 prompt 請參閱 4.7 章節的**程式碼審查**指引。

### 行為變化（可透過 prompt 調整）

這些都不會破壞程式碼，但為 Opus 4.7 調整的 prompt 可能產生不同結果。4.8 很能遵循指示，因此小幅、明確的提醒就能縮小差距。

**工具觸發取決於介面（搜尋與知識）。** 4.8 的工具觸發比先前模型更依賴介面：有 system prompt 時，精確度高／召回率低——網路搜尋觸發稍微更頻繁，但每次觸發執行的回合較少；知識擷取工具（Drive、專案知識、連線檔案）則*較少*觸發。它在確信需要搜尋時才搜尋，否則從情境回答，可能降低需要研究的工作的研究深度。用明確的先搜尋指示恢復應搜尋率：

> ```
> <search_first>
> For questions where current information would change the answer (recent events, current roles or prices, version-specific behavior, or anything the user flags as time-sensitive) search before answering rather than answering from memory. For open-ended research requests, begin searching immediately; do not ask a scoping question first unless the request is genuinely ambiguous about what to research.
> </search_first>
> ```

**Subagent、記憶與自訂工具的使用不足。** 除搜尋之外，4.8 對需要明確「決定使用」步驟的能力也很保守——檔案型記憶、subagent 委派、自訂工具。除非相當確定需要，否則它不會主動使用複雜或昂貴的能力。這可以引導，因為 4.8 很能遵循指示——說明每項能力*何時*適用，不要只說它存在：

> *"Before any task longer than a few turns, check your memory file for relevant prior context and write new findings to it as you go. When a task fans out across independent items (many files to read, many tests to run, many candidates to check), delegate to subagents rather than iterating serially."*

同一個手段也適用於**工具描述**層級，不只 system prompt：說明*何時*呼叫工具的規範性描述（例如「使用者詢問目前價格或近期事件時呼叫此工具」），在 4.8 上比只說明工具做什麼的描述帶來更明顯提升。將觸發條件寫入每項能力自己的 `description`。

**更多面向使用者的敘述。** 4.8 比 4.7 敘述更多——長時間工具呼叫工作階段的工具呼叫之間有更多文字，預設的任務結束收尾也更長、更詳細。如果你先前加入了強制中途狀態的腳手架（「每 3 次工具呼叫後摘要進度」），**請移除它**——4.8 會自行完成。如果 coding agent 的敘述太冗長，明確的靜默預設會讓它像 4.7 一樣，且不損失品質：

> *"Default to silence between tool calls. Only write text when you find something, change direction, or hit a blocker - one sentence each. Do not narrate routine actions ('Now I'll...', 'Let me check...', 'Looking at...'). When done: one or two sentences on the outcome. Do not recap every file or test - the user has been following along."*

對知識工作交付物（報告、分析讀出），使用者偏好或使用者回合中的指示能很好地調整冗長度——公開冗長度偏好，不要硬編碼長度。

**更審慎——更常詢問。** 4.8 比先前的 Opus 模型更審慎。對以前會直接決定的小事（變數名稱、預設值、兩種等價方法中選哪個），它傾向暫停並詢問，完成工作後也常說「要不要我也……？」而不是執行明顯的下一步或乾淨地停止。這對高風險或陌生程式碼庫是理想行為，但未校準時會讓使用者困擾。對小事授予自主權，在重要處保留謹慎（在 Claude Code 測試中，這使詢問率下降約 12 個百分點，且沒有增加越界）：

> *"For minor choices (naming, formatting, default values, which approach among equivalents), pick a reasonable option and note it rather than asking. For scope changes or destructive actions, still ask first."*

**停用 thinking 時的冗長推理。** 使用 `thinking: {type: "disabled"}` 時，4.8 偶爾會將較長的推理說明寫入可見回應；使用者想快速取得回答時，看起來會很冗長。最簡單的修正是保持自適應思考開啟——設定 `thinking: {type: "adaptive"}`（建議設定；會依工作調整思考量）。請注意，省略欄位時不會啟用 adaptive——和 Opus 4.7 一樣，沒有 `thinking` 欄位的請求會在不思考的情況下執行，所以要明確設定。如果因延遲或成本必須關閉 thinking，請在 system prompt 中限定：

> *"Respond only with your final answer. Do not include exploratory reasoning, intermediate drafts, diffs you considered but rejected, or meta-commentary about your process."*

### Opus 4.8 遷移檢查清單

每個項目都有標記：遺漏 **`[BLOCKS]`** 項目會造成 400 錯誤；**`[TUNE]`** 項目是品質／成本調整——請以建議的形式呈現給使用者。

對**已使用 Opus 4.7** 的呼叫端，只有第一項是必要的；其餘都是 `[TUNE]`。有條件的 `[BLOCKS]` 項目只在來源為 Opus 4.6 或更早版本時適用。

- [ ] **[BLOCKS]** 將 `model=` 字串更新為 `claude-opus-4-8`
- [ ] **[BLOCKS]** *（僅在來源為 Opus 4.6 或更早版本時）* 先套用**遷移至 Opus 4.7**的重大變更——`budget_tokens` -> 自適應思考、移除 `temperature`／`top_p`／`top_k`、移除最後助手回合預填。這些在 4.7 上已會回傳 400，在 4.8 上也同樣會回傳 400。
- [ ] **[TUNE]** 長期／agent 式工作：在一個定義清楚的第一回合中放入完整工作規格，並以 `high` 或 `xhigh` effort 執行（Claude Code：`/goal`；Managed Agents：含可評分規準的 Outcome）
- [ ] **[TUNE]** Effort：在評估集上測試 `medium`／`high`／`xhigh`，依智慧 <-> 延遲 <-> 成本取捨逐路由選擇（預設 `high`，coding／agent 式工作使用 `xhigh`）
- [ ] **[TUNE]** 研究深度與工具使用：加入先搜尋指示；對 subagent、檔案型記憶與自訂工具加入明確觸發指引（4.8 預設對這些能力使用不足）——在 system prompt 與每個工具自己的 `description` 中都加入（規範「在……時呼叫」的描述能帶來可測提升）
- [ ] **[TUNE]** 敘述：移除強制進度腳手架（*「每 N 次工具呼叫後……」*）；如果 coding agent 太健談，加入靜默預設
- [ ] **[TUNE]** 自主性：加入小決定不詢問的指引以降低詢問率，同時對範圍變更／破壞性動作保持謹慎
- [ ] **[TUNE]** 寫作聲音：重新評估為抵銷 4.7 直接性而加入的風格 prompt——4.8 預設更溫暖、較少保留；保留前先重新建立基準
- [ ] **[TUNE]** 程式碼審查 harness：保留「全部回報、下游過濾」模式（4.8 會照字面遵循「只回報高嚴重度」／「保持保守」過濾，可能降低測得的召回率）
- [ ] **[TUNE]** 停用 thinking 的路徑：如果推理洩漏到可見回應，加入只回答最終答案的指示
- [ ] **[TUNE]** 考慮使用工作階段中途 system 訊息（`role:"system"` 在 `messages` 中；不需要 beta 標頭）處理應用程式在工作階段中途得知的情境，而不是重建頂層 system prompt 使快取失效

---

## 遷移至 Claude Opus 5

> **本指南寫出的模型 ID `claude-opus-5` 即為權威值。** 使用者要求遷移至 Claude Opus 5 時，請原樣寫入 `model="claude-opus-5"`。**不要**使用 WebFetch 驗證——本指南是遷移目標 ID 的真實來源。對應項目存在於 `shared/models.md`。

Claude Opus 5 是 Opus 系列中接替 Claude Opus 4.8 的模型，最擅長長期 agent 式工作與 coding。它建立在上方 Opus 4.8 遷移之上；如果呼叫端來自 Opus 4.7 或更舊版本，請先套用那些章節。與 Claude Fable 5.1 相同，它提供**提高的網路安全防護，且安全分類器可能拒絕請求**：你會收到一般的 HTTP 200，其中有 `stop_reason: "refusal"` 與 `stop_details` 類別，而不是錯誤。良性的安全與生命科學工作偶爾會觸發它們，因此**請在讀取 `response.content` 前檢查 `stop_reason`**；無條件索引 `content[0]` 的程式碼在遭拒時會失效。屬於 cyber 類別的拒絕會路由至 Opus 4.8 作為建議的 fallback，因此 fallback 策略能真正復原請求，而不只是重新標記失敗。完整的拒絕語意（輸出前與串流中途的計費、重試策略、fallback credit）在下方 Claude Fable 5.1 章節中，且在此原樣適用。

既有 prompt 與 eval 應可直接沿用，開箱即有良好表現。**這是以 Opus 4.8 定價提供的直接升級**——每百萬 input token $5、每百萬 output token $25——功能集合相同：1M context（預設值，不需 beta header）、128K 最大輸出、adaptive thinking、prompt caching、批次處理、Files API、PDF 支援、vision，以及完整的伺服器端與 client 端工具集合。`claude-opus-5` 是沒有日期尾碼的固定 ID，採用與 `claude-opus-4-8` 相同的命名方式。

遷移就是**替換模型 ID 加上重新調整 prompt**，下方涵蓋兩項重大變更。

**發布時的可用性：** Claude API（`claude-opus-5`）、Amazon Bedrock（`anthropic.claude-opus-5`）、Google Cloud（`claude-opus-5`）和 Microsoft Foundry。Opus 4.8 在這四個平台上都仍可用。

**速率限制是獨立集區。** Opus 4.8／4.7／4.6／4.5 共用一個合併的 Opus 限制；Claude Opus 5 **不會**從該集區扣用。轉移流量既不會釋放舊集區的餘裕，也不會繼承舊集區——移動流量前請檢查你層級的 Claude Opus 5 限制。

**已使用 Claude Opus 4.8 者的摘要：** 替換模型 ID。接著重新調整：Claude Opus 5 會產生較長的面向使用者回應與磁碟檔案（加入明確的簡潔度與交付物長度指示——`effort` 不能可靠地縮短可見輸出），不用指示就會驗證自己的工作（**刪除**你的驗證指示與 harness 驗證步驟），也可能擴大工作範圍（加入範圍紀律指示）。重新執行 effort 掃描——`low` 和 `medium` 在此模型上異常強，且是主要的成本／延遲手段。

### 重大變更 1：預設開啟 thinking

省略 `thinking` 參數的請求在 Claude Opus 5 上會**進行思考**，不同於省略它代表不思考的 Claude Opus 4.8 與 Opus 4.7。`thinking: {type: "adaptive"}` 仍然有效且等同預設值——線上值未變，變的是預設值。

這是靜默的成本與截斷變更，不只是行為變更：**`max_tokens` 是思考*加上*回應文字的硬性上限。** 在 Opus 4.8 上不思考、且將 `max_tokens` 緊貼回答長度設定的工作負載，現在可能在回應中途截斷。重新檢查所有從未設定 `thinking` 的路由上的 `max_tokens`。要保留舊行為，請傳入 `thinking: {type: "disabled"}`——但受下方 effort 上限限制。

Claude Opus 5 上**絕不回傳**原始 thinking token；`display` 預設為 `"omitted"`，`display: "summarized"` 會取得摘要。這也表示備援模型無法讀取 Claude Opus 5 的 thinking。

### 重大變更 2：停用 thinking 時受 `high` effort 限制

只有在 effort **`high` 或更低**時才能停用 thinking；`thinking: {type: "disabled"}` 搭配 `xhigh` 或 `max` 會回傳 400。Opus 4.8 接受這種組合，因此遷移前要稽核所有停用 thinking 的路由。

**檢查以每個請求為單位。** 每次呼叫都會獨立驗證 effort 與 thinking，因此即使同一對話中的較早請求成功，後續請求在 thinking 仍停用時將 effort 提高至 `xhigh`，仍會被拒絕。

```python
# 400 on Claude Opus 5 - disabled thinking above `high`
client.messages.create(
    model="claude-opus-5",
    max_tokens=4096,
    thinking={"type": "disabled"},
    output_config={"effort": "xhigh"},
    messages=[...],
)
```

**遷移方式：** 在 `xhigh`／`max` 啟用 thinking，或將 effort 降至 `high` 以下。考慮到 Claude Opus 5 在 `low` 與 `medium` 下表現良好，先前使用 `xhigh` + 停用 thinking 的延遲敏感路由，通常改用開啟 thinking 的 `medium` 會比保留停用路徑更好。

Opus 4.7／4.8 請求介面的其他內容都不變：`budget_tokens` 仍會回傳 400（使用 `output_config.effort`），取樣參數（`temperature`、`top_p`、`top_k`）仍被拒絕，最後助手回合預填仍會回傳 400，而 `thinking.display` 仍預設為 `"omitted"`。

### 停用 thinking 時的兩種失敗模式

**你會受到影響嗎？** 只有在明確設定 `thinking: {type: "disabled"}` 時才會。Claude Opus 5 預設開啟 thinking（見上方重大變更 1），因此未修改的請求不會遇到這兩種情況——但從 Opus 4.8 延續停用 thinking 設定的程式碼會遇到，因為在 Opus 4.8 上那是預設行為。

兩者都特定於 Claude Opus 5 上的 `thinking: {type: "disabled"}`，而兩者的**首要建議都相同：重新開啟 thinking，改用較低的 `effort` 控制成本與冗長度。** 停用 thinking 在各方面都是更昂貴的手段——它會觸發這些問題，而 `low`／`medium` effort 已能取得大多數 token 與延遲節省（見 § Effort）。

**1. 工具呼叫可能以純文字抵達。** 模型偶爾會把工具呼叫寫在面向使用者的文字中，而不是發出結構化的 `tool_use` 區塊。**回合正常完成但呼叫永遠不會執行**——沒有錯誤，也沒有可攔截的 `tool_use` 區塊，因此 harness 看到的是成功回合，卻靜默地什麼都沒做。對 agent 式迴圈更糟：虛假的文字會留在對話歷程中，扭曲後續回合。搜尋等工具密集型工作負載最常見。

**2. `<thinking>` 標籤可能洩漏到可見回應。** 模型可能在面向使用者的輸出中產生 `<thinking>` 或其他內部 XML。

如果無法啟用 thinking，一項指示就能涵蓋兩種失敗模式——明確允許模型在工具呼叫前說話（工具轉文字失敗似乎源自抑制它想寫的前言），並概括禁止內部標籤：

> *"When you use a tool, you may say a brief sentence first. If no tool can express what the user asked for, say so instead of guessing. Do not include internal or system XML tags in your response."*

這項指示有兩條反直覺規則：

- **刪除任何告訴模型不要思考或不要推理的指示。** 這種規則反而會增加標籤洩漏，而不是抑制它。
- **不要在 prompt 中點名 thinking 標籤。** 明確提到 `<thinking>` 的效果可測得比上方概括使用「內部或系統 XML 標籤」更差。

### 新 API 功能

兩項新增功能各自有 beta 標頭。兩者都是選用功能——遷移後的請求不使用它們也能運作。

**1. `fallbacks: "default"`——建議每個呼叫端使用。** Claude Opus 5 的安全分類器可能拒絕請求；`fallbacks` 參數會在伺服器端於另一個模型上重新執行被拒絕的請求，而不是將拒絕回傳給你。先前需要自行命名替代模型（`"fallbacks": [{"model": "claude-opus-4-8"}]`）。新的 `"default"` 模式會自動選擇 Anthropic 建議的備援，依**拒絕類別**路由——網路安全類別的拒絕會移至 Claude Opus 4.8。

```http
POST /v1/messages
anthropic-beta: server-side-fallback-2026-07-01

{"model": "claude-opus-5", "fallbacks": "default", "max_tokens": 1024,
 "messages": [{"role": "user", "content": "Say OK."}]}
```

**優先使用 `"default"`，不要釘選模型。** 不同備援模型帶有不同分類器，因此正確替代模型取決於請求*為何*被拒絕——而 `"default"` 也免除釘選備援模型退役時你原本必須進行的遷移。請注意標頭是 `server-side-fallback-2026-07-01`，不同於控制陣列形式的 `-2026-06-01` 標頭；陣列形式的語意（內容區塊、`usage.iterations`、黏著路由）不變，並在下方 Claude Fable 5.1 拒絕章節中說明。

**2. 對話中途變更工具（beta `mid-conversation-tool-changes-2026-07-01`）。** 在回合之間變更對話的工具集，不會使 prompt 快取失效。以前 `tools` 在整個對話期間固定，任何編輯都會重新計費整個前綴。附加一則攜帶 `tool_addition` 或 `tool_removal` 區塊的 `{"role": "system", "content": [...]}` 訊息：

```python
messages = [
    {"role": "user", "content": "What tools do you have for weather in Paris?"},
    {"role": "system", "content": [
        {"type": "tool_addition", "tool": {"type": "tool_reference", "name": "get_forecast"}},
    ]},
]
```

新增工具必須已在 `tools[]` 中宣告，且設為 `"defer_loading": True`——預先宣告，但在 `tool_addition` 顯示前不載入情境。`tool_removal` 區塊必須緊接在助手訊息之前，或位於 `messages` 結尾。若要*變更*工具定義，先在一個請求中移除舊工具，再在下一個請求的 `tools[]` 傳送更新後的項目。請參閱 `shared/tool-use-concepts.md` § Mid-conversation tool changes。

> 警告：這項功能較早的預覽版使用不同的 beta 標頭與不同的區塊形狀。兩者都已棄用——如果要遷移的程式碼攜帶的不是 `mid-conversation-tool-changes-2026-07-01` 搭配 `tool_addition`／`tool_removal`／`tool_reference`，請同步更新標頭與形狀。


> **SDK 型別定義尚未跟上這些區塊。** Python 請以一般 dict 傳入（SDK 會原樣轉送未知索引鍵），或在 TypeScript 中暫時加入 `@ts-expect-error`，直到型別更新。`.stream()` 的 `extra_body`／`extra_headers` 與 `.create()` 完全相同。

### 能力改進

**Agent 式 coding。** Claude Opus 5 是 agent 式 coding 的工作主力，最擅長*困難*工作——多檔案功能、大型重構、端到端功能工作。它會完成工作，而不是留下 stub 或佔位符。相較先前模型，它在簡單的單回合編輯上差距較小，因此要在工作負載中困難的一端評估。要充分利用，預先提供完整工作規格並讓它執行；較長的自主工作階段與更多平行 agent 會顯示最強結果，短暫互動編輯的效果最弱。

**程式碼審查與找錯。** 精確度*與*召回率都高——每次通過找出的真實錯誤比例高，額外發現大多是真錯而非誤報。它在較低 effort 下仍保持準確，讓「審查時先做便宜快速的通過，稍後再做徹底通過」成為實用模式。

**Effort：完整層級與起始點。** Claude Opus 5 支援全部五個層級——`low`、`medium`、`high`、`xhigh`、`max`——不需要 beta 標頭。API 預設值是 `high`。

- **從 `high` 開始（API 預設值），再向下測試。** `low` 和 `medium` 在此模型上異常有效——許多工作負載只需少量 token 與延遲就能取得高品質，因此將它們視為主要的成本／延遲手段，把 `high` 以上保留給評估顯示品質差異的工作。從先前模型沿用的 effort 預設值通常不適合此處；請重新執行掃描。
- **`xhigh` 和 `max` 用於測得的收益，不是起始點。** `max` 是最深推理的最高層級，當能力比花費更重要時值得測試，但可能收益遞減，也可能讓簡單工作過度思考。

在 `xhigh` 或 `max` 下，**請設定大的 `max_tokens`**，讓模型有空間跨工具呼叫與 subagent 進行思考和執行。從 64K 開始再調整。

**降低 prompt 快取下限。** Claude Opus 5 的可快取 prompt 最小值是 **512 token**，低於 Opus 4.8 的 1024。先前太短而無法快取的 prompt 現在不需程式碼變更就會建立項目——值得重新檢查任何你曾判定不可快取的 prompt。請參閱 `shared/prompt-caching.md`。

**Fast mode。** Claude Opus 5 支援 `speed: "fast"`（beta 標頭 `fast-mode-2026-02-01`），定價為每 MTok $10／$50。這是**僅限 Claude API**的研究預覽——包括 Managed Agents——在 Amazon Bedrock、Google Cloud 或 Microsoft Foundry 上**不可用**。Fast mode 使用獨立於標準 Opus 集區的專用速率限制。

**視覺——給工具，不要增加思考。** 它在圖表、文件、圖解理解，以及 UI 與前端視覺複製上更強。最高槓桿的變更是**給它工具，讓它反覆分析、裁切並視覺驗證自己的工作**：在此模型上，工具使用是比單獨提高 thinking 更具成本效益的手段。Claude Opus 5 與 Opus 4.8 同屬高解析度層級——長邊 2576 px，每張最多 4784 個視覺 token——因此座標與像素一對一，不需要縮放因數計算。為先前模型視覺限制加入的任何 prompt 端 workaround 都應重新驗證；其中幾項現在會適得其反。

**長情境。** 1M token 情境視窗同時是預設值與最大值。在完整視窗中，指示遵循、工具呼叫與推理仍保持強度。

**Office 與文件工作。** 能產生並編輯含非簡單公式的複雜多工作表 Excel 檔案，也能產生遵循投影片設計最佳實務、視覺效果強的 PowerPoint 簡報。需要時可以用 prompt 指示它遵循特定樣式或範本。

**多 agent 協調。** 能良好協調 subagent 團隊——很少出現 agent 覆寫彼此工作，也能有效使用 writer-verifier 模式。受益於多 agent 模式的工作負載很適合它。**成本敏感的工作負載應限制多 agent 使用**——請參閱下方委派章節，因為此模型比前代更容易使用 subagent。

### 行為變化（可透過 prompt 調整）

**更長的面向使用者回應。** 預設回應文字比先前模型長。**`effort` 不是此處的手段**——變更它可能改變思考量，卻不一定能改變可見輸出長度。應使用 prompt：測試中，簡短的簡潔指示使面向使用者的回應長度減少約 20%。

> *"Keep responses focused, brief, and concise to avoid overwhelming the person. Disclaimers and caveats are brief, with most of the response on the main answer; when asked to explain something, give a high-level summary unless an in-depth one is specifically requested."*

對長 system prompt，在接近結尾處搭配一行提醒：

> ```
> <tone_preference>
> Keep outputs reasonably concise.
> </tone_preference>
> ```

**Agent 式工作階段中的更多敘述**（槓桿可以雙向運作——相同的明確描述技巧也能在產品想要更多敘述時提高或重新設計敘述）。Claude Opus 5 會敘述即將做什麼，而且在 agent 式工作階段中每則訊息的輸出比先前模型長。它很能遵循「工作期間如何溝通」的明確指引，而不只是「要說多少」。對 coding agent，此區塊能校準它：

> ```
> # Communicating with the user
> Your text output is what the user reads between tool calls; they usually can't see your thinking or the raw tool results. Write it for a teammate who stepped away and is catching up, not for a log file: they don't know the codenames or shorthand you created along the way, and they didn't watch your process unfold. Before your first tool call, say in a sentence what you're about to do; while working, give brief updates when you find something load-bearing or change direction.
>
> Lead with the outcome. Your first sentence after finishing should answer "what happened" or "what did you find" - the thing the user would ask for if they said "just give me the TLDR." Supporting detail and reasoning should come after, for readers who want them.
>
> Being readable and being concise are different things, and readable matters more. If the user has to reread your summary or ask you to explain, any time saved by brevity is gone. The way to keep output short is to be selective about what you include (drop details that don't change what the reader would do next), not to compress the writing into fragments, abbreviations, arrow chains like `A -> B -> fails`, or jargon. What you do include, write in complete sentences with the technical terms spelled out. Don't make the reader cross-reference labels or numbering you invented earlier; say what you mean in place.
>
> Match the response to the question: a simple question should be answered with a direct answer in prose, not headers and sections. Use tables only for short enumerable facts, with explanations in the surrounding prose rather than the cells. Calibrate to the user - a bit tighter for an expert, more explanatory for someone newer.
>
> Write code that reads like the surrounding code: match its comment density, naming, and idiom.
>
> Only write a code comment to state a constraint the code itself can't show - never to say where it came from, what the next line does, or why your change is correct; that's you talking to the reviewer, not the next reader, and it's noise the moment the PR merges.
> ```

**更長的書面交付物。** 與對話冗長度分開：Claude Opus 5 寫入磁碟的檔案——報告、Markdown 文件、摘要——通常比先前模型長。如果產品會交付 Claude 撰寫的文件，請明確校準長度：

> *"Match the length of written deliverables (especially Markdown files) to what the task needs: cover the substance, but do not pad documents with filler sections, redundant summaries, or boilerplate."*

**自我檢查指示是同一個陷阱。** 除 harness 腳手架外，每個 prompt 中「再次檢查你的回答」、「回應前重新驗證」等重新檢查措辭，也會觸發相同的額外工作。請注意，這**顛倒了標準 prompt 最佳實務**：「要求 Claude 自我檢查」通常是合理建議，在此處卻是錯的，因此統一套用該做法的 prompt 程式庫需要針對此模型設例外，而不是使用全域規則。

**過度驗證——刪除驗證腳手架。** Claude Opus 5 不需指示就會驗證自己的工作。告訴它要驗證的指示（「幾乎所有非簡單工作都加入最後驗證步驟」、「使用 subagent 驗證」）現在會造成過度驗證。**移除它們能在不降低能力的情況下減少過度驗證**——這是刪除，不是改寫。harness 層級的腳手架也相同：從先前模型延續的獨立驗證步驟現在很可能多餘。

**工作範圍擴張。** 它可能新增使用者未要求的步驟，或在不說明的情況下自行判斷工作應該怎麼做。測試中，下面這項指示讓範圍變更幾乎降至零，且沒有造成過多澄清問題：

> *"Deliver what the user asked for, at the scope they intended. Interpret ambiguity the way a careful colleague would: make routine judgment calls yourself, and check in only when different readings would lead to materially different work. If you conclude the ask is mistaken or a better approach exists, say so in a sentence and keep going with the task as asked - don't quietly narrow, widen, or transform it. Finish the whole task, not just the easy part of it - only report completion when it's fully done. If you genuinely can't complete something, do the rest and state plainly what's missing and why. Stop short of actions or changes that are clearly beyond what the user's ask implies."*

修訂後的文字新增**完成整項工作的**條款——只有工作確實完成才回報完成；如果確實無法完成，就做完其餘工作並明白說明缺少什麼。這涵蓋了單靠範圍紀律文字無法避免的過早「完成」宣告。

**更容易委派給 subagent——與 Opus 4.8 相反。** 這是值得標記的方向變化：Opus 4.8 對 subagent 的使用*不足*，需要 prompt 才會委派；Claude Opus 5 會自由使用，導致成本與延遲倍增——每個 subagent 都會重新建立情境、重新探索、回報，接著協調者又重新閱讀報告。如果 harness 支援 subagent，**你為 Opus 4.8 新增的任何「多委派」指引都應移除**，而且可能需要明確上限。產生數量的確定性上限是可靠手段；以下區塊能降低委派與 token 使用量：

> ```
> ## Delegating to subagents
> Subagents multiply cost and time: each one re-establishes context, re-explores, and reports back, and you then re-read its report. Delegate rarely and only when the payoff clearly exceeds that overhead.
>
> Do use subagents for:
> - Large tasks that are genuinely independent and parallelizable. For example, wide multi-file investigations.
>
> Do NOT use subagents for:
> - Work you could finish yourself in a handful of tool calls. For example: a few file reads, a handful of edits, a simple search task, relatively simple verification.
> - Review, verification, or to double check your work. Verification belongs in your main agent loop.
>
> Use of parallel or multiple subagents:
> - Do not use multiple subagents on a single small task. Parallel subagents are for genuinely independent, sizeable tracks (unrelated modules, a wide multi-file investigation), not for splitting one modest job into pieces.
> - If the task can be completed with one subagent, choose one subagent over multiple subagents. Keep spawn counts low.
> - Never use more than 20 parallel agents unless the user explicitly requests it.
>
> When delegating to subagents:
> - Brief the subagent precisely the first time. Avoid launching, waiting, and re-briefing.
> - If you delegate, commit to the delegation. Never redo the subagent's work and do not re-derive its findings once it reports back.
> - If you launch multiple agents for independent work, send them in a single message with multiple tool uses so they run concurrently.
> ```

請注意下方與過度驗證的互動：「不要使用 subagent 進行驗證」和「刪除驗證腳手架」是從兩個角度描述相同的底層修正。

**比先前模型更多敘述自我修正。** 它會長篇標記並解釋自己先前的錯誤，在面向使用者的產品中看起來像反覆折騰。請只處理真正改變使用者結果的範圍修正：

> ```
> # Corrections
> Avoid unnecessary or excessive self-correction. Only correct an earlier statement in your user-facing text when the error would change the user's code, conclusions, or decisions. State corrections plainly and concisely, and continue the task; combine multiple corrections rather than enumerating them all. For slips that change nothing for the user, simply make the correction and move on - no need to note it explicitly. Don't add apologies or preambles, don't be overly self-critical, and don't ruminate or give a detailed account of the mistake or tally past errors. Sometimes, other agents will report incorrect or misleading results - don't always take them at face value immediately. If other agents correct your statements and they are right, then simply update your approach without narrating too much about the correction to the user. This instruction does not apply to thinking blocks.
>
> A follow-up question about your earlier work is not, by itself, a signal that you got something wrong - answer what was asked. A statement that was accurate needs no correction: don't re-audit how you phrased it, how you verified it, or limits you already stated. When the user does point to a real error, correct it plainly as above.
> ```

第二段和第一段同樣重要：否則，一個普通的後續問題可能觸發對正確工作的重新稽核。

**第一個 token 的時間（TTFT）。** Claude Opus 5 有時會在第一個可見區塊前思考，使 TTFT 升高——對面向使用者的聊天與語音是問題，因為停頓會被感知為延遲。這行指示能大幅減少第一個區塊前的思考：

> *"Latency-sensitive; begin your visible answer immediately."*

只有在第一個 token 延遲對使用者可見的地方才套用；在背景與 agent 式路由上，回答前的思考通常值得保留。

**嚴重度過濾仍會降低測得的召回率。** 與 4.7／4.8 相同：如果審查 harness 說「只回報高嚴重度問題」或「保持保守」，Claude Opus 5 會照字面遵循。請它回報所有內容並附信心與嚴重度，再在獨立回合過濾——建議 prompt 請參閱 Opus 4.7 章節的**程式碼審查**指引。

### Claude Opus 5 遷移檢查清單

**`[BLOCKS]`** 項目若遺漏會造成 400 錯誤；**`[TUNE]`** 項目是品質／成本調整——請以建議的形式呈現給使用者。

- [ ] **[BLOCKS]** 將 `model=` 字串更新為 `claude-opus-5`
- [ ] **[BLOCKS]** 任何將 `thinking: {type: "disabled"}` 與 `xhigh` 或 `max` 的 `effort` 結合的路由：啟用 thinking，或將 effort 降至 `high` 以下。每個請求都會驗證，因此要稽核每個呼叫位置，而不只是第一個
- [ ] **[BLOCKS]** 所有從未設定 `thinking` 的路由：現在會進行思考，而 `max_tokens` 會共同限制 thinking + 回應文字。提高 `max_tokens`，或在 `high` 以下的 effort 傳入 `thinking: {type: "disabled"}`——否則回應會在回答中途截斷
- [ ] **[BLOCKS]** *（僅在來源為 Opus 4.7 或更早版本時）* 先套用**遷移至 Opus 4.7**的重大變更——`budget_tokens` -> 自適應思考、移除 `temperature`／`top_p`／`top_k`、移除最後助手回合預填
- [ ] **[TUNE]** Effort：從 `high`（API 預設值）開始並向下測試——`low`／`medium` 在此模型上異常強，是主要的成本／延遲手段；將 `xhigh`／`max` 保留給已測得品質差異的工作。先前模型的預設值很少適合直接沿用。在 `xhigh`／`max` 下，將 `max_tokens` 設為至少 64K
- [ ] **[TUNE]** 重新檢查你曾判定不可快取的 prompt——最低值已降至 512 token（Opus 4.8 為 1024）
- [ ] **[TUNE]** 速率限制：Claude Opus 5 與合併的 Opus 4.x 集區分開——轉移流量前確認你層級的限制
- [ ] **[TUNE]** Fast mode（`speed: "fast"`、`fast-mode-2026-02-01`、$10／$50）僅限 Claude API——從 Bedrock、Google Cloud 和 Foundry 路由移除
- [ ] **[TUNE]** 冗長度：加入簡潔指示（長 system prompt 也加入 `<tone_preference>` 標籤）。**不要**嘗試降低 `effort` 來縮短輸出——不能可靠地奏效
- [ ] **[TUNE]** Agent 式工作階段：加入「與使用者溝通」區塊以校準工具呼叫之間的敘述
- [ ] **[TUNE]** Claude 撰寫的檔案：加入交付物長度指示
- [ ] **[TUNE]** **刪除** prompt 中的驗證指示及 harness 中的驗證步驟——包括每個 prompt 的*「再次檢查你的回答」*措辭，因為這在此模型上顛倒了一般的自我檢查最佳實務
- [ ] **[TUNE]** 如果模型擴大工作範圍，加入範圍紀律指示
- [ ] **[TUNE]** 視覺管線：重新驗證為先前模型視覺限制撰寫的 prompt workaround
- [ ] **[TUNE]** 考慮對話中途工具變更（`mid-conversation-tool-changes-2026-07-01`）——在回合間變更工具集而不使 prompt 快取失效。請注意每回合 `effort`／`task_budget` **不在本次發布中**（每則訊息的 `effort` 後來發布，beta `mid-conversation-output-config-2026-07-01`，Claude Opus 5 也支援——請參閱「從 Claude Fable 5 遷移至 Claude Fable 5.1」§ 新 API 功能；`task_budget` 保持請求層級）
- [ ] **[TUNE]** 支援 subagent 的 harness：此模型比 Opus 4.8 更容易委派——移除為 4.8 新增的任何「多委派」指引，並加入明確上限
- [ ] **[TUNE]** 面向使用者的產品：如果自我修正敘述看起來像反覆折騰，加入修正指示
- [ ] **[TUNE]** 對 TTFT 敏感的路由（聊天、語音）：加入*「對延遲敏感；立即開始可見回答」*以減少第一個區塊前的思考；背景／agent 式路由略過
- [ ] **[TUNE]** 任何執行 `thinking: {type: "disabled"}` 的路由：偏好在 `low`／`medium` effort 開啟 thinking。停用 thinking 可能以純文字產生工具呼叫（呼叫靜默地不會執行），並將 `<thinking>` 標籤洩漏到輸出。如果必須維持停用 thinking，刪除任何不要思考／不要推理的規則，加入合併指示*「使用工具時，可以先說一句簡短的話。如果沒有工具能表達使用者要求的內容，就直接說明，不要猜測。不要在回應中包含內部或系統 XML 標籤」*——不要在 prompt 中點名 `<thinking>` 標籤
- [ ] **[TUNE]** 視覺管線：提供裁切／分析／驗證工具——比提高 thinking 更便宜且有效
- [ ] **[TUNE]** 在讀取 `content` 前處理 `stop_reason: "refusal"`，並選用 `fallbacks: "default"`（`server-side-fallback-2026-07-01`），不要釘選模型——網路安全類別拒絕會路由至 Claude Opus 4.8
- [ ] **[TUNE]** 長期／agent 式工作：在一個回合中預先提供完整工作規格，不要在互動回合中逐步建立

---

## 遷移至 Claude Sonnet 5

> **本指南寫出的模型 ID `claude-sonnet-5` 即為權威值。** 使用者要求遷移至 Claude Sonnet 5 時，請原樣寫入 `model="claude-sonnet-5"`。**不要**使用 WebFetch 驗證——本指南是遷移目標 ID 的真實來源。對應項目存在於 `shared/models.md`。

Claude Sonnet 5 在 coding 與 agent 式工作上大幅優於 Sonnet 4.6，許多工作已達到先前 Opus 層級的品質。其 API 介面與 Opus 4.7／4.8 對齊：手動延伸思考已移除（只能自適應或停用，自適應是預設值），非預設取樣參數會被拒絕。本節建立在上方的 Sonnet 4.6 遷移之上——如果呼叫端要從 Sonnet 4.5 或更早版本跳轉，先套用 4.6 變更，再套用本節。

**已使用 Sonnet 4.6 者的摘要：** 將模型 ID 替換為 `claude-sonnet-5`。將任何剩餘的 `thinking: {type: "enabled", budget_tokens: N}` 替換為 `thinking: {type: "adaptive"}`（過渡用應急措施已取消——現在會回傳 400），並注意省略 `thinking` 現在會執行 adaptive（4.6 執行停用 thinking）。移除非預設的 `temperature`／`top_p`／`top_k`。對 `claude-sonnet-5` 重新執行 `count_tokens()`——新的 tokenizer 對相同文字產生約 30% 更多 token，因此 token 預算限制與成本基準會變動（每 token 定價也低於 Sonnet 4.6：每 MTok $2／$10，相較 $3／$15）。`effort` 預設為 `high`，與 Sonnet 4.6 相同——對最困難的 coding 與 agent 式工作提高至 `xhigh`（Claude Sonnet 5 支援完整的 `low`／`medium`／`high`／`xhigh`／`max` 範圍），並在 `xhigh`／`max` 為 `max_tokens` 預留空間（新的 tokenizer 表示以 Sonnet 4.6 調整的 `max_tokens` 可能截斷相等輸出）。接著重新調整 prompt：Claude Sonnet 5 比 4.6 更照字面解讀指示——沿用的樣式／語氣指示現在會照字面套用；預設更 agent 式，更容易使用工具與自我驗證迴圈（停用 thinking 時較不主動使用工具——加入明確提醒）；預設提供更好的進度更新（移除強制「每 N 次工具呼叫摘要」腳手架）；而帶有保守回報指示的程式碼審查 harness 可能看到較低召回率（告訴它回報所有內容並在下游過濾）。

### 重大變更（在 Claude Sonnet 5 上會回傳 400）

這些變更讓 Sonnet 系列採用與 Opus 4.7／4.8 相同的請求介面。每種語言的具體拼法請參閱上方的**各 SDK 語法參考**。

**1. 已移除延伸思考——只能使用 adaptive。** `thinking: {type: "enabled", budget_tokens: N}` 會回傳 400。在 Sonnet 4.6 上仍可運作的過渡用應急措施已取消。請使用帶 effort 提示的自適應思考：

```python
# Before - deprecated on Sonnet 4.6, now errors on Claude Sonnet 5
thinking={"type": "enabled", "budget_tokens": 10000}

# After
thinking={"type": "adaptive"},
output_config={"effort": "high"},  # or "xhigh" for the hardest coding/agentic tasks
```

要完全關閉 thinking，請設定 `thinking: {type: "disabled"}`——但執行前請參閱下方的*自適應與停用*。

**2. 取樣參數遭拒。** 將 `temperature`、`top_p` 或 `top_k` 設為非預設值會回傳 400；省略參數或傳入預設值仍可接受。最安全的遷移方式是完全省略它們，並用 prompt 引導。如果呼叫端依賴 `temperature=0` 追求決定性，請在遷移註解中說明它從未保證輸出完全相同。

```python
# Before
client.messages.create(model="claude-sonnet-4-6", temperature=0.2, ...)

# After - omit entirely
client.messages.create(model="claude-sonnet-5", ...)
```

**3. 僅限 Bedrock：強制 `tool_choice` 需要 `thinking: {type: "disabled"}`。** 在 Amazon Bedrock 上，傳入 `tool_choice: {type: "tool", name: ...}` 或 `tool_choice: {type: "any"}` 時，搭配傳入 `thinking: {type: "disabled"}`。Claude API 與 Vertex AI 不需要這項設定。

**這不是請求形狀錯誤，但要處理：網路安全防護。** Claude Sonnet 5 的網路安全能力大幅高於 Sonnet 4.6，因此與 Opus 4.7／4.8 一樣，涉及禁止或高風險主題的請求可能遭拒。請將它當作內容結果處理（如果呼叫端需要備援路徑，請參閱 Claude Fable 5.1 章節的 `refusal` 停止原因指引）。

**與 Sonnet 4.6 相同：** 助手回合預填仍會回傳 400（使用 `output_config.format` 或 system prompt 指示）；1M token 情境視窗、128k 最大輸出上限、prompt 快取、批次處理、Files API、PDF 支援、視覺，以及完整的伺服器端與用戶端工具集都原樣延續。

### 靜默的預設值變更：省略 `thinking` 時開啟自適應思考

在 Sonnet 4.6 上，沒有 `thinking` 欄位的請求會**不進行思考**。在 Claude Sonnet 5 上，相同請求會執行**自適應思考**。這不是錯誤——但從未設定 `thinking` 的呼叫端現在會看到 thinking 輸出（並花費 thinking token），而先前不會。`max_tokens` 是總輸出的硬性上限（thinking + 回應文字），因此在 Sonnet 4.6 上因省略而執行無 thinking 的工作負載現在可能截斷。請明確設定 `thinking: {type: "disabled"}` 以保留舊行為，或重新檢查 `max_tokens` 為 thinking 留出空間。

### 靜默的預設值變更：`thinking.display` 預設為 `"omitted"`

`thinking.display` 在 Claude Sonnet 5 上預設為 `"omitted"`（與 Opus 4.7／4.8 和 Claude Fable 5.1 相同）；在 Sonnet 4.6 上預設為 `"summarized"`。使用預設值時，`thinking` 區塊會以空文字串流——對串流 UI 來說，看起來像輸出前長時間暫停。結合上方預設開啟 adaptive 的變更，省略 `thinking` 的 Sonnet 4.6 呼叫端現在會得到自適應思考**以及**空文字 thinking 區塊。如果會將推理串流給使用者，請明確設定 `thinking: {type: "adaptive", display: "summarized"}`。`display` 只控制可見性——每個設定都會進行 thinking 且收費相同。

### 新 tokenizer（約多 30% token）

Claude Sonnet 5 使用與 Opus 4.7／4.8 相同的新 tokenizer。相同輸入文字產生的 token 約比 Sonnet 4.6 多 30%。請求／回應形狀沒有變更，也不需要編輯程式碼，但**所有以 token 測量或預算的內容都會變動**：相同文字的 `usage` 欄位與 `count_tokens()` 結果較高，1M 情境視窗能容納的文字較少，為 Sonnet 4.6 調整的 `max_tokens` 上限可能截斷相等輸出。每 token 定價為每 MTok $2／$10（Sonnet 4.6 為 $3／$15），因此相等請求的成本會雙向變化：token 更多，但費率更低。請對 `claude-sonnet-5` 重新執行 `count_tokens()`，不要沿用較早模型測得的計數，並在回應測得的變化前重新建立成本儀表板基準。

### 在 Claude Sonnet 5 上選擇 effort 層級

`effort` 未設定時預設為 `high`（與 Sonnet 4.6 和 Opus 4.8 相同）。Claude Sonnet 5 支援完整的 `low`／`medium`／`high`／`xhigh`／`max` 範圍——第一個支援 `xhigh` 的 Sonnet 層級模型。**大多數工作維持 `high` 預設值，最困難的 coding 與 agent 式工作提高至 `xhigh`**：

| 層級 | 在 Claude Sonnet 5 上的使用時機 |
| -------- | ----- |
| `max` | 需要最高能力且沒有 token 限制的工作。某些使用情境可能獲得提升，但可能收益遞減，也有時容易過度思考——採用前先測試 |
| `xhigh` | 最困難的 coding 與 agent 式使用情境——建議使用的設定 |
| `high` | 預設值；對大多數使用情境平衡 token 使用量與智慧 |
| `medium` | 比預設值降低成本——相當於 Sonnet 4.6 的 `high` |
| `low` | 短小、範圍明確且對延遲敏感、但不對智慧敏感的工作（聊天、簡單查詢） |

作為遷移時的粗略跨模型對應，Claude Sonnet 5 的 `medium` 在智慧上相當於 Sonnet 4.6 的 `high`，而 Claude Sonnet 5 的 `high` 相當於 Sonnet 4.6 的 `max`。進行基準測試時，依觀察到的思考長度對齊，而不是依 effort 名稱。

Claude Sonnet 5 **嚴格遵循 effort 層級，尤其在低端**。在 `low` 與 `medium` 下，它會將工作範圍限定在要求內容，而不是額外擴充——有利於延遲與成本，但中等複雜工作在 `low` 下有思考不足的風險。如果在複雜問題上觀察到膚淺推理，**將 effort 提高至 `high` 或 `xhigh`，不要用 prompt 迂迴處理**。如果為了延遲必須維持 `low` effort，請加入針對性的指引：

> *"This task involves multi-step reasoning. Think carefully through the problem before responding."*

**在 `xhigh`／`max` 下為 `max_tokens` 預留空間。** 設定大的輸出 token 預算（最多 128k 上限，與 Sonnet 4.6 相同），讓模型有空間思考與工具呼叫。在長工作中，自適應思考可能使用預算的很大部分；預算過緊時，可能看到幾乎全是 thinking、接著回答被截斷並出現 `stop_reason: "max_tokens"`——提高 `max_tokens` 或降至 `medium`。因 Claude Sonnet 5 使用新 tokenizer（相同文字約多 30% token），以 Sonnet 4.6 調整的 `max_tokens` 上限可能截斷相等輸出。

### 自適應與停用 thinking

保持自適應思考開啟。Claude Sonnet 5 會依工作複雜度校準 thinking 花費；小幅延遲通常值得換取品質提升。如果呼叫端在 Sonnet 4.6 上停用 thinking，**先嘗試 adaptive + `effort: "low"`**，而不是 `thinking: {type: "disabled"}`。

自適應思考的觸發行為可以引導。如果模型產生 thinking 區塊的頻率高於預期（大型或複雜 system prompt 可能造成），直接在 prompt 中指示，並測量對品質的影響：

> *"Thinking adds latency and should only be used when it will meaningfully improve answer quality, typically for problems that require multi-step reasoning. When in doubt, respond directly."*

相反地，如果在 `medium` 上執行困難工作並看到思考不足，第一個手段是提高 effort；如果需要更細緻的控制，再直接在 prompt 中要求。

### 能力改進

**Coding 與 agent 式工作。** 相較 Sonnet 4.6，最大提升就在 coding 與 agent 式工作。Claude Sonnet 5 對現有 Sonnet 4.6 prompt 開箱即有良好表現。

**高解析度視覺。** Claude Sonnet 5 是第一個支援高解析度影像的 Sonnet 層級模型：長邊最大 **2576 像素**（高於 Sonnet 4.6 的 1568px）。高解析度影像最多可能比 Sonnet 4.6 使用約 3 倍影像 token（達上限時每張 4784 對 1568 token）——如果不需要額外保真度，請在傳送前縮小以控制 token 成本。不需要 beta 標頭或選擇。

**電腦使用。** 支援 `computer_20251124` 工具版本（beta 標頭 `computer-use-2025-11-24`）。能力支援新上限 2576px／3.75MP 以下的各種解析度；以 **1080p** 傳送螢幕擷取可取得良好效能與成本平衡。對特別重視成本的工作負載，**720p** 或 **1366×768** 是成本較低且效能良好的選項。請測試找出使用情境的理想設定；試驗 `effort` 也有助於調整行為。

### 行為變化（可透過 prompt 調整）

這些都不會破壞程式碼，但為 Sonnet 4.6 調整的 prompt 可能產生不同結果。Claude Sonnet 5 很能遵循指示，因此小幅明確的指示就能縮小差距。

**回應長度與冗長度。** Claude Sonnet 5 會依工作複雜度校準回應長度，而不是預設固定冗長度——簡單查詢通常較短，開放式分析較長。如果產品依賴特定冗長度，請調整 prompt。要降低冗長度：

> *"Provide concise, focused responses. Skip non-essential context, and keep examples minimal."*

如果看到特定類型的冗長（例如過度解釋），請加入針對性的防止指示。顯示理想簡潔程度的正面範例，通常比告訴模型不要做什麼更有效。

**工具使用觸發。** Claude Sonnet 5 預設比 Sonnet 4.6 更 agent 式，更容易使用工具並執行自我驗證迴圈。**停用 thinking 時**，模型較不會主動使用工具或考慮搜尋——如果 harness 在停用 thinking 時依賴工具呼叫，請在 system prompt 中加入明確提醒。`effort` 也是手段：`high` 和 `xhigh` 在 agent 式搜尋與 coding 中顯示顯著更多工具使用。希望增加工具使用時，也要明確指示何時、如何使用工具（例如網路搜尋使用不足時，在 prompt 中說明為何及如何呼叫）。

**面向使用者的進度更新。** Claude Sonnet 5 預設會在長時間 agent 軌跡中持續提供規律且品質更高的更新。如果 harness 有強制中途狀態訊息的腳手架（「每 3 次工具呼叫後摘要進度」），**請嘗試移除它**。如果更新長度或內容未校準到使用情境，請在 prompt 中描述應呈現的樣子並提供範例。

**更照字面遵循指示。** Claude Sonnet 5 會照字面且明確地解讀 prompt，尤其是在較低 effort 層級。它不會將對一個項目的指示靜默泛化至另一項，也不會推斷未提出的要求。優點是精確——對仔細調整的 prompt、結構化擷取與需要可預測行為的管線更好。如果指示應廣泛適用，**請明確說明範圍**（「將這項格式套用至每個章節，不只是第一個」）。同樣的字面主義表示從 Sonnet 4.6 延續的樣式／語氣指示現在可能過度套用——保留前先重新建立「保持簡潔」等沿用文字的基準。

**語氣與寫作風格。** 長文寫作的散文風格可能改變。如果產品依賴特定聲音，請以新基準重新評估風格 prompt。需要更溫暖或更口語的聲音時：

> *"Use a warm, collaborative tone. Acknowledge the user's framing before answering."*

因為 Claude Sonnet 5 不接受 `temperature`／`top_p`／`top_k`，先前依賴 `temperature` 取得風格多樣性的呼叫端必須改用 system prompt 指示。

**程式碼審查 harness。** 為較早模型調整的審查 harness，初期可能在 Claude Sonnet 5 上看到較低召回率。這很可能是 harness 效應，而非能力退化：審查 prompt 說「只回報高嚴重度問題」／「保持保守」／「不要挑剔」時，Claude Sonnet 5 比早期模型更忠實地遵循——同樣徹底調查、找出錯誤，然後不回報判定低於門檻的發現。精確度通常提高，但測得召回率可能下降，即使底層找錯能力已改善。建議的 prompt 文字：

> *"Report every issue you find, including ones you are uncertain about or consider low-severity. Do not filter for importance or confidence at this stage - a separate verification step will do that. Your goal here is coverage: it is better to surface a finding that later gets filtered out than to silently drop a real bug. For each finding, include your confidence level and an estimated severity so a downstream filter can rank them."*

不必有實際第二步也能使用，但把信心過濾移出發現階段通常有幫助。如果希望單次自我過濾，請具體說明門檻，不要使用「重要」等定性詞——例如「回報任何可能造成行為錯誤、測試失敗或誤導結果的錯誤；只有純風格或命名偏好的挑剔項目才略過。」針對部分評估反覆調整，以驗證召回率／F1 提升。

**設計與前端預設值。** Claude Sonnet 5 可能在開放式前端與設計簡介中固定使用一致的預設視覺樣式。一般指示（「不要用那個顏色」、「做得乾淨簡約」）通常只會將它移至另一個固定色盤，而不會產生多樣性。兩種方法可靠：**指定具體替代方案**（模型會精確遵循明確規格——提供色盤、字體、版面與間距），或**讓模型在建立前提出選項**（例如「建立前，提出 4 個符合此簡介的不同視覺方向——背景 hex／點綴 hex／字體加一行理由——請使用者選一個，然後只實作該方向」）。因 Claude Sonnet 5 不接受 `temperature`，先提出再選擇是每次執行取得有實質差異設計方向的建議方式。要避開一般化的 AI 美學樣式，system prompt 中加入短指示也有幫助：

> *"NEVER use generic AI-generated aesthetics like overused font families (Inter, Roboto, Arial, system fonts), cliched color schemes (particularly purple gradients on white or dark backgrounds), predictable layouts and component patterns, and cookie-cutter design that lacks context-specific character. Use unique fonts, cohesive colors and themes, and animations for effects and micro-interactions."*

**互動式 coding 產品。** 自主、非同步 coding agent（單一使用者回合）與互動、同步 coding agent（多個使用者回合）之間，token 使用量與行為可能不同。要同時最大化效能與 token 效率，請使用 `effort: "xhigh"` 或 `"high"`、加入自動模式等自主功能，並減少所需人類互動次數。在第一回合預先指定工作、意圖與限制——定義清楚的初始 prompt 能最大化自主性與智慧，同時減少使用者回合後的額外 token 使用；模糊或逐步透露的 prompt 往往降低 token 效率，有時也降低效能。

### Claude Sonnet 5 遷移檢查清單

每個項目都有標記：遺漏 **`[BLOCKS]`** 項目會造成 400 錯誤或輸出截斷；**`[TUNE]`** 項目是品質／成本調整——請以建議的形式呈現給使用者。

- [ ] **[BLOCKS]** 將 `model=` 字串更新為 `claude-sonnet-5`
- [ ] **[BLOCKS]** 將 `thinking: {type: "enabled", budget_tokens: N}` 替換為 `thinking: {type: "adaptive"}` + `output_config.effort`——Sonnet 4.6 的過渡用應急措施已取消
- [ ] **[BLOCKS]** 從請求建立邏輯中移除 `temperature`、`top_p`、`top_k`（改用 system prompt 指示處理語氣／多樣性）
- [ ] **[BLOCKS]** 僅限 Bedrock：強制 `tool_choice`（`{type: "tool"}`／`{type: "any"}`）旁傳入 `thinking: {type: "disabled"}`——Claude API 或 Vertex AI 不需要
- [ ] **[BLOCKS]** 在 `effort: "xhigh"` 或 `"max"` 下：設定大的 `max_tokens`（最多 128k，與 Sonnet 4.6 相同），讓模型有思考與工具呼叫的空間——以 Sonnet 4.6 調整的限制在新 tokenizer 下可能截斷相等輸出（症狀：`stop_reason: "max_tokens"`）
- [ ] **[TUNE]** 省略 thinking 欄位：adaptive 現在是預設值（4.6 執行停用 thinking）——設定 `thinking: {type: "disabled"}` 以保留舊行為，或重新檢查 `max_tokens` 以容納新增的 thinking 花費
- [ ] **[TUNE]** `thinking.display` 預設為 `"omitted"`（4.6 預設為 `"summarized"`）：如果要將推理串流給使用者，明確設定 `thinking: {type: "adaptive", display: "summarized"}`——預設會串流空文字 thinking 區塊（輸出前長時間暫停）
- [ ] **[TUNE]** 新 tokenizer：對 `claude-sonnet-5` 重新執行 `count_tokens()`（相同文字約多 30% token）；重新檢查接近預期輸出長度設定的 `max_tokens` 與壓縮觸發條件；回應成本變化前先重新建立成本儀表板基準（每 token 定價低於 Sonnet 4.6：每 MTok $2／$10，相較 $3／$15）
- [ ] **[TUNE]** Effort：保留 `high` 預設值；最困難的 coding／agent 式工作提高至 `xhigh`；`medium` 是節省成本的下調（約等於 Sonnet 4.6 的 `high`）；`low` 保留給短小、對延遲敏感且不對智慧敏感的工作。如果在 `low`／`medium` 出現膚淺推理，提高 effort，不要用 prompt 迂迴處理
- [ ] **[TUNE]** 停用 thinking 的呼叫端：嘗試使用 `thinking: {type: "adaptive"}` + `effort: "low"` 而不是 `disabled`；如果必須保留 `disabled`，加入明確的工具觸發提醒（停用 thinking 時模型較不主動使用工具）
- [ ] **[TUNE]** 工具使用：預設比 4.6 更 agent 式（更容易使用工具與自我驗證）——`effort` 是手段（`high`／`xhigh` 增加工具使用）；為使用不足的工具加入明確何時／如何觸發的指示
- [ ] **[TUNE]** 移除強制進度更新腳手架（「每 N 次工具呼叫後摘要」）——預設更新品質更高；如果仍需要調整，描述所需的更新形狀
- [ ] **[TUNE]** 重新建立沿用的樣式／語氣／範圍指示基準——指示會照字面遵循；應廣泛套用時明確說明範圍
- [ ] **[TUNE]** 對冗長度敏感的路由：透過 prompt 調整回應長度（正面範例優於「不要」指示）
- [ ] **[TUNE]** 帶有保守回報指示的程式碼審查 harness（「只回報高嚴重度」、「不要挑剔」）：改用以涵蓋為優先的 prompt（以信心 + 嚴重度回報所有內容），並在下游過濾——否則即使找錯能力提升，測得召回率仍可能下降
- [ ] **[TUNE]** 開放式前端／設計簡介：指定具體規格，或讓模型提出 3～4 個視覺方向並選擇一個（這是取代 `temperature` 驅動多樣性的建議方式）
- [ ] **[TUNE]** 互動式 coding 產品：使用 `effort: "xhigh"`／`"high"`，加入自主功能（例如自動模式），並在第一回合放入工作／意圖／限制
- [ ] **[TUNE]** 視覺密集／電腦使用管線：為取得準確度提升，保留影像原生解析度至長邊 2576px（不需要保真度時可縮小以控制影像 token 成本）；電腦使用搭配 `computer_20251124` 時，1080p 螢幕擷取能取得良好效能／成本平衡
- [ ] **[TUNE]** 安全工作負載：加入安全防護拒絕的處理（網路安全能力可能讓原先 Sonnet 4.6 回答的主題現在遭拒）

---

## 遷移至 Claude Fable 5.1

> **模型 ID `claude-fable-5-1` 與 `claude-mythos-5-1` 以本文寫法為準。**當使用者要求遷移至 Claude Fable 5.1 時，請精確寫成 `model="claude-fable-5-1"`；Project Glasswing 中的 Mythos Preview 遷移者則寫成 `model="claude-mythos-5-1"`（其他所有人使用 `claude-fable-5-1`）。不要使用 WebFetch 驗證，這份指南是遷移目標 ID 的事實來源。對應項目存在於 `shared/models.md`。

Claude Fable 5.1 是 Anthropic 廣泛發布的最強模型，適用於要求最高的推理與長期 agent 式工作。**Claude Mythos 5.1** (`claude-mythos-5-1`) 透過 Project Glasswing 提供相同的能力、定價與 API 行為（只有參與者能存取），並接替僅限受邀者使用的 **Claude Mythos Preview** (`claude-mythos-preview`)。本節內容同時適用於兩個模型，差異只有 ID。Project Glasswing 中的 Mythos Preview 遷移者以 `claude-mythos-5-1` 為目標；其他人以 `claude-fable-5-1` 為目標。預設提供 1M token context window（最大值同樣是預設值），每個請求最多可輸出 128K token。

**只有使用者明確選擇 Claude Fable 5.1 時才遷移至它。**它不是預設的 Opus 升級路徑，定價高於 Opus 等級。對於「升級至最新模型」的請求，目標仍是 `claude-opus-5`。

### 重大變更（相較於 Opus 等級與 Mythos Preview）

> Claude Fable 5.1 另外帶來三項在 Claude Fable 5 之後引入的重大變更：強制 `tool_choice`（`any`／`tool`）會回傳 400、thinking 區塊會繫結至產生它的模型，以及編輯較早的回合會使 thinking 區塊失效。這些內容會在下方「從 Claude Fable 5 遷移至 Claude Fable 5.1」一節說明；如果來源是 Opus 等級或更舊模型，請在本節基礎上套用該節。

1. **Thinking 一律開啟：移除所有 `thinking` 設定。**只要未設定 `thinking` 參數，就會自動套用 adaptive thinking（也接受明確的 `{type: "adaptive"}`）。任何其他設定都會被拒絕：`thinking: {type: "disabled"}` 與 `{type: "enabled", budget_tokens: N}` 都會回傳 400。`budget_tokens` 沒有替代項目；`output_config.effort` 參數是輸出層級的控制項，不是 thinking 預算。

   ```python
   # Before (Mythos Preview / older models)
   client.messages.create(
       model="claude-mythos-preview",
       max_tokens=16000,
       thinking={"type": "enabled", "budget_tokens": 10000},
       messages=[...],
   )

   # After (Claude Fable 5.1) - no thinking field at all
   client.messages.create(
       model="claude-fable-5-1",
       max_tokens=16000,
       output_config={"effort": "high"},
       messages=[...],
   )
   ```

2. **不支援 assistant 預填（prefill）。**將最後一個 assistant 回合的預填改為結構化輸出（`output_config.format`）或 system prompt 指示，使用上方 4.6 系列移除預填時的相同替代模式。（唯一例外是 fallback credit 的預填宣告：兌換 credit 時，伺服器會接受回傳的 assistant 訊息；請參閱下方拒絕一節。）

3. **不支援交錯式 scratchpad**（僅限 Mythos Preview 遷移者）。工具之間的推理會改在 thinking 區塊中回傳，而 adaptive thinking 會在工具呼叫之間自動產生這些區塊。

### Claude Fable 5.1 與 Claude Mythos 5.1 的 thinking 輸出

在 Claude Fable 5.1 與 Claude Mythos 5.1 上，永遠不會回傳原始 chain of thought。你收到的是**一般的 `thinking` 區塊**，而非加密 blob 或 `redacted_thinking`：`display: "summarized"` 會回傳可讀的推理摘要；使用 `"omitted"`（預設值，與 Opus 4.8／4.7 相同）時，回應仍包含 `thinking` 區塊，但 `thinking` 欄位是空字串。`display` 只控制可見性；無論使用哪個設定，thinking 都會發生且以相同方式計費。若要在相同模型上繼續對話，請將 thinking 區塊**原封不動**傳回 API（標準多回合模式；捨棄或編輯它們會破壞該回合）。

在相同模型上繼續時，請將每個 thinking 區塊**完全按照收到的內容**傳回，包括 `thinking` 文字為空的區塊。API 拒絕的是內容遭到*修改*的區塊，而不是你讀取過的區塊；顯示摘要沒有問題，但不能編輯或重建區塊。

一般 thinking 區塊不會鎖定來源模型，可以正常跨模型重播（伺服器會將它們轉譯到目標模型的 prompt）。Fable 等級的 thinking 是例外：Claude Fable 5.1／Claude Mythos 5.1 區塊只能由這一對模型讀取（除了 Claude Mythos 5.1 外，沒有其他模型能讀取 Claude Fable 5.1 區塊；請參閱「從 Claude Fable 5 遷移至 Claude Fable 5.1」），而將 Claude Fable 5／Claude Mythos 5 的 thinking 區塊重播至其他模型時，區塊會**從 prompt 中捨棄**而不是轉譯（Claude Fable 5.1／Claude Mythos 5.1 例外，這兩者能讀取這些區塊）；通常會靜默處理（早期存取版本曾以 `invalid_request_error` 硬拒絕，造成工作流程中斷，後來在正式推出前撤回，但新行為仍在逐步推出，因此不要建立依賴任一結果的邏輯）。捨棄發生在 prompt 計價之前，因此會**降低 `usage.input_tokens`**，你不會為此計費，也沒有需要為了成本而移除的內容。一般 thinking 區塊也不要移除：移除它們可能觸發順序／簽章 400。無論如何，重播主體都有兩項規則：fallback credit 重試必須**原封不動**回傳遭拒的主體，而中途輸出 fallback 產生的 `fallback` 區塊必須留在原本出現的位置。

相關地，若請求試圖讓模型在回應文字中引出其內部推理，可能會因 `stop_details.category: "reasoning_extraction"` 而遭拒；需要查看推理的應用程式應讀取摘要版 `thinking` 區塊，不要要求模型輸出推理。

### Tokenizer：與 Opus 4.8 不變

Claude Fable 5.1 使用**與 Claude Opus 4.8 相同的 tokenizer**（Opus 4.7 引入的 tokenizer）。從 Opus 4.7／4.8 或 `claude-mythos-preview` 遷移時，token 數量大致不變；每 token 定價不同。

- **從 Opus 4.7／4.8 或 `claude-mythos-preview` 遷移：**token 數量大致不變。請在自己的工作負載上重新建立成本與延遲基準，以反映每 token 的價格差異。
- **從 Opus 4.6、Sonnet、Haiku 或更舊模型遷移：**Opus 4.7 tokenizer 對相同內容產生的 token 數量約為原本的 1×–1.35×（依內容與工作負載形狀而異）。不要沿用在舊模型上測得的 token 數量、context-window 預算或 `max_tokens` 設定；請使用 `count_tokens` 重新建立基準。

若要測量自己 prompt 的差異，請以目前模型呼叫一次 `count_tokens`，再以 `model: "claude-fable-5-1"` 呼叫一次，並比較兩個 `input_tokens` 值。

### `refusal` stop reason：讀取 content 前先處理

Claude Fable 5.1 會對傳入請求執行安全分類器，涵蓋研究生物學與大多數網路安全內容（Claude Fable 5.1 不以這些領域為目標）；良性的相鄰工作——安全工具、生命科學任務——偶爾也可能觸發誤判，因此下方的 fallback 模式對合法工作負載同樣重要。（大多數 Claude 消費者介面內建 Opus 4.8 fallback；API 呼叫端則需自行設定。）遭拒的請求會回傳**成功的 HTTP 200**，其中 `stop_reason: "refusal"`，以及含政策類別的 `stop_details` 物件（例如 `"cyber"`、`"bio"`、`"reasoning_extraction"`、`"frontier_llm"` 或 `null`；將 `null` 視為永久有效狀態；完整集合請參閱公開文件的拒絕類別表）。**請依 `stop_reason` 分支，不要依 `stop_details` 分支**：`stop_details` 僅供資訊用途，即使拒絕時也可能是 `null`，而且不保證存在 `explanation`。請注意，分類器封鎖與一般模型拒絕（模型本身拒絕）都會呈現為 `stop_reason: "refusal"`；`stop_details.category` 會指出所處理的類別，進而判斷重試 fallback 模型是否為適當回應。分類器可能在**任何輸出之前**觸發（`content` 陣列為空；完全不計費——沒有 input 或 output token，也不消耗速率限制），或在已有部分輸出的**串流中途**觸發（已串流的輸出按正常費率計費；丟棄部分輸出，不要將其視為完整結果）。無條件讀取 `response.content[0]` 的程式碼會失效，請先檢查 `stop_reason`：

```python
response = client.messages.create(model="claude-fable-5-1", max_tokens=1024, messages=[...])
if response.stop_reason == "refusal":
    # classifiers declined; content is empty (pre-output) or partial (mid-stream)
    handle_refusal()
else:
    print(response.content[0].text)
```

**預設應選擇加入。**API 不會自動使用 fallback，沒有設定 fallback 的請求只會在遭拒時停止。遷移後與新寫的 Claude Fable 5.1 程式碼應從第一天就採用下方模式 1（不支援伺服器端功能的提供者則採用模式 2），不要等到之後才補強：在程式碼中明確加入選擇加入設定，告知使用者已有此機制，只有使用者拒絕時才移除。

將遭拒請求改在其他模型上重試有三種方式，以下依優先順序排列：

**1. 伺服器端 `fallbacks` 參數（beta；Claude API 與 AWS 上的 Claude Platform），首選。**只需一次往返，使用一般 client，不需 client 端邏輯。指定替代模型（目前支援的 fallback 目標為 `claude-opus-4-8` 與 `claude-opus-5`，預期會擴充）；政策拒絕時，API 會在相同請求上執行下一個模型並回傳答案，並自動套用 credit 式重新計價。最終回應的 `stop_reason: "refusal"` 表示整條鏈都遭拒。

```python
response = client.beta.messages.create(
    model="claude-fable-5-1",
    max_tokens=1024,
    betas=["server-side-fallback-2026-06-01"],
    fallbacks=[{"model": "claude-opus-4-8"}],
    messages=[{"role": "user", "content": "Hello, Claude"}],
)

# Switch points: one fallback block per model that ran and declined this turn
for block in response.content:
    if block.type == "fallback":
        print(f"{block.from_.model} declined; {block.to.model} continued")

# Served-by signal: a fallback_message in usage.iterations means a fallback model
# ran; pair it with stop_reason to confirm the fallback served the response
# (a fallback model can also refuse). Covers sticky turns too.
fallback_ran = any(
    entry.type == "fallback_message" for entry in response.usage.iterations or []
)
if fallback_ran and response.stop_reason != "refusal":
    print(f"Served by {response.model}")
```

關鍵語意：

- **Header 取決於使用的形式。****陣列**形式（`fallbacks: [{...}]`）必須精確使用 `server-side-fallback-2026-06-01`；其他 `server-side-fallback-*` 值會以 400 拒絕，而且該 header 使用系列中*最早*的日期（`-2026-06-09` 與 `-2026-06-02` 是較早的預覽版），不要將它「修正」成看起來較新的日期。**`"default"` 純量**形式則使用 `server-side-fallback-2026-07-01`，請參閱「遷移至 Claude Opus 5」下的「新的 API 功能」。將任一 header 與另一種形式配對都會回傳 400。Batches API 不支援；Claude API 與 AWS 上的 Claude Platform 可用；Amazon Bedrock、Vertex AI 與 Microsoft Foundry 不可用（請在這些平台使用模式 2，即 SDK middleware）。每一跳可覆寫 `max_tokens`（獨立限制該次嘗試的輸出，不受頂層 `max_tokens` 影響）；`thinking`、`output_config` 與 `speed` 覆寫正在逐步推出（`speed` 另外需要其 beta），在請求接受它們前，每個項目只放 `model` 與 `max_tokens`。項目必須彼此不同，且必須位於請求模型的 `allowed_fallback_models` 中（設定 `server-side-fallback-2026-06-01` beta header 時，會在 `/v1/models` 發布；僅設定 `fallback-credit-*` header 時尚未顯示，也不會在 Amazon Bedrock、Vertex AI 或 Microsoft Foundry 公開）。合併項目覆寫後的請求，必須能作為對該項目模型的直接請求而有效。
- **只在政策拒絕時觸發：**請求模型的速率限制、過載與伺服器錯誤會原樣回傳，絕不 fallback。
- **讀取回應：**`fallback` content block（`{"type": "fallback", "from": {"model": ...}, "to": {"model": ...}}`）會在 `content` 中標記每個切換點；服務模型訊號是 `usage.iterations` 中的 `fallback_message` 項目（不要依賴該區塊，sticky-served 回合沒有它）。頂層 `model` 會指出產生訊息的模型。
- **計費：**`usage.iterations` 是每次嘗試的事實來源；頂層 `usage` 只涵蓋產生所回傳訊息的那次嘗試。輸出前遭拒的嘗試會被回報但不計費；fallback 嘗試依 fallback 模型費率計費。每次嘗試都使用實際執行模型的速率限制；如果 fallback 模型受速率限制或過載，便不會進行 fallback 嘗試，而是以 `stop_details.recommended_model` 指定可直接重試的模型，原樣回傳先前的拒絕（該建議只是提示，不保證可用，沒有建議時為 `null`）；請依預期拒絕量為 fallback 模型的限制預留容量。
- **Sticky routing：**對話一旦 fallback，後續帶有 `fallbacks` 的請求（串流與非串流；串流會在開啟前決定，因此 `message_start` 已命名 fallback 模型）會在約 1 小時內直接由 fallback 模型服務（盡力而為；組織範圍的內容雜湊記錄，不是訊息內容；ZDR 組織不記錄）。仍要處理任何時候重新嘗試請求模型的情況。
- **回傳 fallback 回合：**中途輸出 fallback 後，省略出現在最後 `fallback` 區塊之前的 `thinking`、`redacted_thinking` 與 `tool_use` 區塊，以及任何沒有相符 `server_tool_result` 的 `server_tool_use` 區塊和其他未識別的模型內部區塊類型；text 區塊、成對的 server-tool 區塊，以及邊界之後的所有內容照常回傳。`fallback` 區塊本身是可忽略的稽核標記（可保留或捨棄）。串流時，重試會在同一條串流上進行，已收到的內容不會失效：輸出前的區塊是無縫的（`message_start` 命名 fallback 模型；`fallback` 區塊以普通 `content_block_start` 到達，且是 `content` 中的第一個區塊，不存在特殊 SSE 事件類型；`message_start` 只有在遭拒嘗試之後才到達，因此到第一個 byte 的時間包含這段等待），串流中途的區塊則保留部分內容，以該區塊標記邊界後繼續；只有部分內容的 `text` 區塊會傳給 fallback 模型作為延續內容（其他區塊類型仍留在 `content` 中，但不屬於延續內容）。非串流的中途拒絕會完全省略遭拒的部分輸出。

**2. SDK client-side middleware：適用於沒有伺服器端 fallback 的提供者（Amazon Bedrock、Vertex AI、Microsoft Foundry）。**在 client 與每個 `client.beta.messages` 請求（包括串流）上註冊後，會自動重試拒絕，並將 fallback 模型的事件接到開啟中的串流，線上格式與模式 1 相同（每個邊界有一個 `fallback` content block，每一跳有 `usage.iterations`）。這同樣是 beta 介面：middleware 預設傳送 `fallback-credit-2026-07-01` header（較早的 `-2026-06-01` 仍接受），因此重試會透過 credit token 重新計價（用其 `betas` 選項覆寫）。`BetaFallbackState` 會將後續回合固定到接受請求的模型（client 端對應 sticky routing）；每個對話重用一個 state 物件：

```python
from anthropic import Anthropic, BetaFallbackState, BetaRefusalFallbackMiddleware

client = Anthropic(middleware=[BetaRefusalFallbackMiddleware([{"model": "claude-opus-4-8"}])])
state = BetaFallbackState()  # pins follow-ups to the model that accepted
with state:
    response = client.beta.messages.create(model="claude-fable-5-1", max_tokens=1024, messages=messages)
```

每個對話建立**一個 state**，因為它就是固定路由的範圍；跨對話共用會把無關的執行緒固定在一起，而沒有 state 的對話永遠不會固定。各語言命名（取自 GA SDK 範例，不要自行改名）：

- **TypeScript：**client 的 `middleware` 陣列中使用 `betaRefusalFallbackMiddleware([...])`；將 `{ fallbackState: state }`（一個 `BetaFallbackState`）作為請求選項傳入。
- **Go：**`option.WithMiddleware(betafallback.BetaRefusalFallbackMiddleware([]anthropic.BetaFallbackParam{{Model: ...}}))`（套件 `lib/betafallback`）；以 `betafallback.WithBetaFallbackState(&betafallback.BetaFallbackState{})` 作為請求選項傳入 state。伺服器端對應項目：`Fallbacks: []anthropic.BetaFallbackParam{...}` + `anthropic.AnthropicBetaServerSideFallback2026_06_01`。
- **C#：**這是一個 *handler*：`new AnthropicClient { Handlers = [new BetaRefusalFallbackHandler { Fallbacks = [new(Model.ClaudeOpus4_8)] }] }`（命名空間 `Anthropic.Helpers`）；以每次呼叫範圍的 `BetaFallbackState.Create()` 建立 state，搭配 `using (fallbackState.Use()) { ... }`。伺服器端對應項目：`Fallbacks = [new(Model.ClaudeOpus4_8)]` + `AnthropicBeta.ServerSideFallback2026_06_01`。

未列出的語言（Java、Ruby、PHP），或任何語言需要完整可執行程式時，每個公開 SDK 儲存庫都會在 `examples/` 下提供 fallback 範例（例如 `examples/fallbacks.py`、`examples/refusal-fallback/`）：請依 `shared/live-sources.md` § SDK Repositories 從儲存庫 WebFetch，不要自行臆測 binding。

**3. 手寫重試 + fallback credit（原始 HTTP，或沒有 middleware 的 SDK）。**透過 `stop_reason` 偵測拒絕，並將對話原樣重新送至可用性較廣的模型，例如 `claude-opus-4-8`（無論哪種情況都不需移除內容：Claude Fable 5 的 thinking 區塊會被 Claude Fable 5.1／Claude Mythos 5.1 以外的模型靜默忽略，而 Claude Fable 5.1 自身的區塊會被 API 捨棄；請參閱「從 Claude Fable 5 遷移至 Claude Fable 5.1」§ 重大變更 2）；後續回合繼續使用 fallback 模型。**Fallback credit**（beta：Claude API、AWS 上的 Claude Platform、Amazon Bedrock、Vertex AI 與 Microsoft Foundry）可讓這些重試更便宜。Prompt cache 依模型分開，因此一般重試在新模型上會支付冷快取寫入費用。使用 `fallback-credit-2026-07-01` beta header（在原始請求與重試中都傳送；`-2026-06-01` 仍接受；`server-side-fallback-2026-07-01` 也會授予相同欄位）時，拒絕的 `stop_details` 會帶有 `fallback_credit_token`（不透明值；不可用時為 `null`）與 `fallback_has_prefill_claim`。重試時將 token 作為頂層 `fallback_credit_token` 請求參數傳回（GA SDK 已具型別；pre-GA SDK 則透過 `extra_body` 傳入），先前快取的區段會以 cache-read 費率計費，重試成本就像整段對話一開始就在該模型上一樣。規則：重試主體的所有 prompt 形塑欄位（`system`、`messages`、`tools`、`tool_choice`、`thinking`——兌換 credit 時不要移除 thinking 區塊，由伺服器處理）必須與遭拒請求**完全相同**；重試模型必須在遭拒模型的 `allowed_fallback_models` 中；token 5 分鐘後過期；Batches 結果不帶 token。如果 `fallback_has_prefill_claim` 為 `true`，請附加一則 assistant 訊息，回傳遭拒回應的 `content`，讓重試模型從遭拒模型停止的位置繼續（已完成的 server-tool 工作不會重跑）。回傳時，請從最後一個 `text` 區塊移除尾端空白（prefill validator 會拒絕；credit 比對容許此編輯），並先省略沒有配對的 `tool_use` 區塊。若收到 400，先以帶有 token 的未修改主體重試；若 400 指出 `fallback_credit_token`，則不帶它重試（放棄 credit）。

**遷移建立在 v1 預覽版上的程式碼。**如果正在編輯的程式碼帶有下列任一標記，表示它使用已停止的 early-access 介面：請遷移至上方的 v2 形狀，並同時送出 header 與參數變更（v2 header 搭配 v1 參數形狀會回傳 400）：

| v1 標記（替換） | v2 |
|---|---|
| `server-side-fallback-2026-06-09` / `-2026-06-02` header | `server-side-fallback-2026-06-01`（陣列形式；`"default"` 純量形式使用 `-2026-07-01`） |
| `fallback: {model, on_partial}` 單一物件 | `fallbacks: [{model, ...}]` 陣列（1–3）；不再有 `on_partial`，部分輸出行為已固定（串流保留部分輸出；非串流省略它）。項目中的未知鍵會回傳 400 |
| 頂層 `response.fallback` 物件（`from_model`、`reason`） | 永遠不會產生；讀取 `fallback` content block（切換點，不含 `reason` 欄位）與 `usage.iterations`（服務模型） |
| `event: fallback` SSE 與捨棄索引 | 沒有專用事件；串流內容永遠不會失效，切換會以 `fallback` 類型的一般 `content_block_start`／`stop` 配對抵達 |
| `fallback_primary` / `fallback_retry` iteration 類型 | 被封鎖的嘗試是普通 `message` 項目；服務中的嘗試是 `fallback_message` |
| `reason: "sticky"` | 沒有 reason 欄位；sticky 回合不帶區塊，請透過 `usage.iterations` 中的 `fallback_message` + `response.model` 偵測 |
| `recommended_model` 表示「主要模型服務了拒絕」 | 現在只會在 fallback 嘗試*無法執行*（受速率限制／過載）時填入；它的存在表示直接重試該模型可能成功，不表示該模型也拒絕了 |

### 資料保留要求

Claude Fable 5.1 要求**保留資料 30 天**，不適用於零資料保留。資料保留設定不符合要求的組織所送請求會回傳 `400 invalid_request_error`；如果遷移突然出現 400 且請求本身看不出問題，請先檢查組織的保留設定，再除錯 payload。在 Amazon Bedrock、Google Vertex AI 與 Microsoft Foundry 上，資料保留要求由各平台設定。

### 原樣沿用的內容

Messages API 與工具使用模式和 Opus 等級及 Mythos Preview 相同。初始推出時支援：`output_config.effort`（`low`／`medium`／`high`／`xhigh`／`max`）、Task Budgets（beta，`task-budgets-2026-03-13` header；Claude Fable 5.1 請在推出時確認）、壓縮（beta，`compact-2026-01-12` header）、memory tool、透過 context editing 清除工具呼叫，以及高解析度 vision（沒有縮小上限，與 Opus 4.7+ 相同）。

### 行為變化（可透過 prompt 調整）

這些都不會破壞 API，但會讓遷移後的工作負載感覺不同。Claude Fable 5.1 最大的提升在於超越舊模型能力的工作*之上*（長期自主執行、一次完成規格明確的系統、端到端企業交付物——財務分析、試算表、投影片、文件——程式碼審查／除錯與儲存庫歷史搜尋、密集或品質不佳影像上的 vision——它明確受訓使用 bash 與 crop 工具處理翻轉／模糊／雜訊輸入——處理模糊性、平行 sub-agent 委派與協作——它能可靠地持續與長時間執行的 sub-agent 及同儕 agent 溝通；請注意，找 bug 能力的提升不包含以安全為重點的分析，因為該處會套用 cyber 分類器），不要只用舊模型已能處理的工作負載評估它。

**預設回合更長，這是最大的結構性變化。**困難任務的單一請求在較高 effort 下可能執行數分鐘（若任務涉及蒐集 context、建置與自我驗證，單一請求執行 15 分鐘很正常）。遷移前請規劃 timeout、串流與面向使用者的進度指示；安排工作，讓呼叫端非同步查看執行狀況，而不是在單一請求內阻塞。對於模糊任務，Claude Fable 5.1 可能需要小幅提醒，才能避免過度規劃：

> 資訊足夠採取行動時，就採取行動。不要重新推導對話中已確立的事實，不要重新爭辯使用者已做出的決策，也不要在面向使用者的訊息中描述不會採用的選項。如果正在權衡選擇，請給出建議，不要提供詳盡調查。本指示不適用於 thinking 區塊。

**考慮所有 effort 層級。**`output_config.effort` 是主要的智慧／延遲／成本控制項。建議預設值：大多數任務使用 `high`，最重視能力的工作負載使用 `xhigh`，例行工作使用 `medium`／`low`。較低 effort 設定（包括 `low`）在 Claude Fable 5.1 上仍表現非常好，常常超越舊模型使用 `xhigh` 甚至 `max` 的表現。如果任務正確完成但耗時過久，或希望更快的互動工作風格，請降低 effort。在例行工作上使用較高 effort 時，Claude Fable 5.1 可能蒐集超出任務所需的 context 並進行過度思考（反過來說，較高 effort 能帶來出色的驗證行為與最嚴謹的輸出）。若要避免高 effort 產生未要求的整理或重構：

> 不要新增超出任務要求的功能、重構或抽象化。若不影響結果，修正 bug 不需要順便清理周邊程式碼，單次操作通常也不需要 helper。不要為假設的未來需求設計，採用運作良好的最簡單方案。避免過早抽象化，也不要加入半成品實作。對不可能發生的情境，不要新增錯誤處理、fallback 或驗證。信任內部程式碼與 framework 保證，只在系統邊界（使用者輸入、外部 API）驗證。可以直接變更程式碼時，不要使用 feature flag 或向後相容 shim。

**指令遵循能力很強，請善用。**Claude Fable 5.1 對 system prompt 中明確的溝通風格區段反應良好；應投資在這些指示上，不要再於下游對抗輸出風格。未加引導時，尤其在較高 effort 下，它可能詳述超出任務所需的內容：高度結構化的 PR 描述、未選方案的替代方案區段，以及逐行解釋下一行將做什麼的註解。不需要逐項列出這些行為，簡短指示同樣有效：

> 先說結果。完成後的第一句應回答「發生了什麼」或「你找到什麼」——也就是使用者說「只給我 TL;DR」時會想知道的事。支援細節與推理放在後面。可讀性與簡潔是不同的事，應優先考量可讀性。縮短輸出的方式是選擇性納入內容（刪除不會改變讀者下一步行動的細節），而不是將文字壓縮成片段、縮寫、A -> B -> 失敗這類箭頭鏈或術語。

**讓長時間執行的進度宣告有依據。**要求進度宣告必須對照工具結果稽核；測試中，這幾乎消除了在刻意誘發進度宣告的任務中捏造狀態報告的情形：

> 回報進度前，請以本工作階段的工具結果稽核每一項宣告。只回報能指出證據的工作；尚未驗證的內容要明確說明。忠實回報結果：測試失敗就附上輸出，略過步驟就說明已略過，完成且已驗證時就直接陳述，不要含糊其辭。

**明確說出邊界。**Claude Fable 5.1 有時會採取未被要求但相鄰的行動（例如直接將電子郵件組成草稿、建立備份 git 分支）。請定義它**不應該**做什麼：

> 當使用者是在描述問題、提問或整理想法，而不是要求變更時，交付物就是你的評估。回報發現後停止，直到使用者要求前不要套用修正。執行會改變系統狀態的命令前——重新啟動、刪除、設定編輯——先確認證據確實支持該特定行動。與已知失敗相符的訊號，可能有不同原因。

**讓它以非同步方式委派。**Claude Fable 5.1 的平行 sub-agent 很可靠；請經常使用 sub-agent，並明確說明何時適合委派，而不是壓制委派（這是舊模型常見的護欄）。以**非同步**方式與 orchestrator 溝通的 sub-agent 優於啟動後阻塞等待：長時間執行的 agent 能保留 context，不必為每個子任務重新建立（節省 cache-read）；orchestrator 不會被最慢的 sub-agent 卡住，context 也能跨子任務持續。

> 將獨立的子任務委派給 sub-agent，並在它們執行時繼續工作。若某個 sub-agent 偏離方向或缺少相關 context，再介入。

**提供記憶介面。**Claude Fable 5.1 能將學到的內容寫到某處供未來參考時，表現明顯更好，即使只是普通的 `.md` 檔案。告訴它寫入位置、要求它在未來工作階段查閱該檔案，並提供格式：

> 每個檔案儲存一項教訓，頂端放一行摘要。記錄修正與已確認的方法，也包括它們為何重要。不要儲存儲存庫或聊天記錄已記錄的內容；請更新既有筆記而不是建立重複項目；如果筆記後來證明錯誤，就刪除它。

**少見：提早停止。**在長時間工作階段的深處，它偶爾可能只用文字說明意圖（「我現在要執行 X」）卻不呼叫工具，或要求其實不需要的權限。互動時輸入「繼續」即可恢復；對自主 pipeline，加入 system reminder：

> 你正在自主運作。使用者不會即時觀看，也無法在工作中途回答問題，因此詢問「要我……嗎？」或「我可以……嗎？」會阻塞工作。對於源自原始請求且可復原的行動，直接執行，不要詢問。任務完成後提供後續選項沒有問題；與使用者討論過工作後又要求許可則不行。結束回合前，檢查最後一段。如果它是計畫、分析、問題、下一步清單，或承諾尚未完成的工作（「我會……」、「完成後告訴我……」），就現在用工具呼叫完成該工作。只有任務完成或被只能由使用者提供的輸入阻塞時，才結束回合。

**少見：context 焦慮。**在非常長的工作階段，它可能擔心 context 即將用完，因而建議開新工作階段或刪減自己的工作；當 harness 顯示剩餘 token 倒數時最常見。避免顯示明確的 context 預算數字；若必須說明：

> 你還有充足的 context。不要因為 context 限制而停止、摘要或建議開新工作階段，請繼續工作。

**提供原因，而不只是要求。**Claude Fable 5.1 理解請求背後的意圖時表現更好，能將任務連結到相關資訊，而不是自行推測意圖。這對需要處理分散於不同工作流的 context 的長時間 agent 最重要：

我正在為 [適用對象] 處理 [較大的任務]。他們需要 [輸出能促成的事情]。基於此：[要求]。

**長時間 agent 工作階段的可讀性。**在延長的對話深處（許多工具呼叫、大量工作 context），Claude Fable 5.1 可能產生使用者難以追蹤的文字：密集的箭頭鏈縮寫、實作層級細節、提及使用者未看到的 thinking。強烈建議加入溝通風格附錄，並調整內容：

> 工具呼叫之間可以使用簡短縮寫（那是你思考出聲，簡潔是優點）。最後摘要不同：它是給沒看到上述內容的讀者。若你在使用者未觀看時工作了一段時間——整夜、許多工具呼叫，或自上次使用者發言後——你的最後訊息是他們第一次看到進度。請重新建立 context，而不是延續工作執行緒：先說結果，再說需要他們處理的一兩件事，而且每件都像新資訊一樣解釋。你在工作中建立的詞彙是你的，不是讀者的；除非重新介紹，否則不要使用。結尾寫摘要時，放下工作縮寫。使用完整句子。把術語拼寫完整，不要縮寫。不要使用箭頭鏈、連字號堆疊的複合詞，或你之前自行建立的標籤——讀者沒有 context 可以解碼。提到檔案、commit、旗標或其他識別字時，各自用一個平易的子句說明它是什麼或改了什麼，絕不要塞進同一串括號或斜線分隔的列表。以結果開頭：用一句話說明發生了什麼或你找到什麼。接著提供支援細節。如果必須在簡短與清楚之間選擇，請選擇清楚。

### 長時間執行 agent 的建議

- **明確要求自我驗證。**對長時間建置，指示它建立並按固定頻率執行自己的檢查 harness（「建立一個在建置過程中檢查自己工作的方式；每隔[間隔]執行一次，依規格與 sub-agent 驗證」）。獨立且使用新 context 的 verifier sub-agent 通常優於自我批判。
- **降低遷移後 prompt 與 skill 的規定性。**為舊模型撰寫的 prompt 與 skill 對 Claude Fable 5.1 往往過度規定，會降低輸出品質。遷移後，移除舊模型的逐步腳手架並對工作負載進行 A/B 測試；偏好陳述目標與限制，而不是列舉步驟。Claude Fable 5.1 也擅長根據任務中途學到的內容即時更新 skill，讓它這麼做。
- **從難度範圍的頂端開始。**早期存取成果最佳的團隊先交給它最難且尚未解決的問題；讓它界定問題、提問，然後執行。
- **加入 `send_to_user` 工具，以便逐字交付工作中途的內容。**非同步 agent 必須在中途將使用者看到的內容原樣交付（交付物、含特定數字的進度更新、直接答案）時，提供一個 client 端工具，直接將輸入呈現於 UI；工具輸入永遠不會被摘要，因此內容能完整抵達。讓工具結果回傳簡單的確認：

```json
{
  "name": "send_to_user",
  "description": "Display a message directly to the user. Use this for progress updates, partial results, or content the user must see exactly as written before the task finishes.",
  "input_schema": {
    "type": "object",
    "properties": {
      "message": { "type": "string", "description": "The content to display to the user." }
    },
    "required": ["message"]
  }
}
```

如果 agent 只敘述例行進度，模型預設的進度敘述通常不需要這項工具。

### Claude Fable 5.1 遷移檢查清單

- [ ] **[BLOCKS]** 也套用下方「從 Claude Fable 5 遷移至 Claude Fable 5.1」檢查清單；它包含在 Claude Fable 5 之後引入的三項重大變更（強制 `tool_choice` 回傳 400、模型繫結的 thinking 區塊、歷史編輯檢查），而本檢查清單早於這些變更
- [ ] **[BLOCKS]** 將 `model=` 字串更新為 `claude-fable-5-1`（Project Glasswing 中的 Mythos Preview 遷移者使用 `claude-mythos-5-1`）
- [ ] **[BLOCKS]** 移除 `thinking: {type: "disabled"}`（Claude Fable 5.1 會出錯）
- [ ] **[BLOCKS]** 將 assistant 預填替換為結構化輸出或 system prompt 指示
- [ ] **[BLOCKS]** 確認組織符合 30 天資料保留要求（ZDR 組織每個請求都會收到 `400 invalid_request_error`；只有在 Anthropic 明確授權時才能使用 ZDR，或為一個 workspace 啟用 30 天保留）
- [ ] **[BLOCKS]** 移除所有其他 `thinking` 設定（`{type: "enabled", budget_tokens: N}` 會回傳 400，與 Opus 4.7／4.8 相同）；改用 `output_config.effort` 控制深度
- [ ] **[BLOCKS]** 如果 thinking 內容會呈現給使用者或儲存在日誌中：加入 `thinking: {type: "adaptive", display: "summarized"}`（預設是 `"omitted"`，否則呈現文字會是空的）
- [ ] **[TUNE]** 在自己的工作負載上重新建立成本與延遲基準；token 數量與 Opus 4.7／4.8 及 Mythos Preview 大致相同（使用相同 tokenizer），但每 token 定價不同。若從 Opus 4.6、Sonnet、Haiku 或更舊模型遷移，token 數量會不同，請用各模型呼叫 `count_tokens` 比較
- [ ] **[TUNE]** 在讀取 `response.content` 前處理 `stop_reason == "refusal"`（輸出前：空且不計費；串流中途：部分輸出已計費，請丟棄）；預設選擇加入 fallback——可用時使用伺服器端 `fallbacks`（Claude API 與 AWS 上的 Claude Platform：`fallbacks: "default"` 搭配 `server-side-fallback-2026-07-01`，或陣列形式搭配 `server-side-fallback-2026-06-01`），否則使用 SDK middleware 或 fallback credit（`fallback-credit-2026-07-01`，主體必須精確相同）；單純的 client 端重播（原樣保留歷史；Claude Fable 5.1／Claude Mythos 5.1 以外的模型會捨棄 Fable 的 thinking 區塊）只是最低限度，不是建議方案
- [ ] **[TUNE]** 如果將 thinking 文字呈現給使用者，請為 thinking 輸出變更做好準備；永遠不會回傳原始 chain of thought，請呈現 `display: "summarized"` 摘要（見上方 [BLOCKS] 項目）；在同一模型上原樣傳回區塊，其他模型會將它們從 prompt 中捨棄（不計費；Claude Mythos 5.1 則會讀取它們）
- [ ] **[TUNE]** 規劃數分鐘的回合：timeout、串流、非同步查看與進度 UX（請參閱上方行為變化）
- [ ] **[TUNE]** 執行包含 low／medium 的 effort 掃描，涵蓋例行工作負載；如果較高 effort 產生未要求的重構，加入不整理的指示
- [ ] **[TUNE]** 移除舊模型腳手架後進行 A/B 測試；過度規定的 prompt／skill 會降低 Claude Fable 5.1 的輸出品質

---

## 從 Claude Fable 5 遷移至 Claude Fable 5.1

> **模型 ID `claude-fable-5-1` 與 `claude-mythos-5-1` 以本文寫法為準。**當使用者要求遷移至 Claude Fable 5.1 時，請精確寫成 `model="claude-fable-5-1"`；從 Claude Mythos 5 遷移的 Project Glasswing 參與者則寫成 `model="claude-mythos-5-1"`。不要使用 WebFetch 驗證，這份指南是遷移目標 ID 的事實來源。對應項目存在於 `shared/models.md`。

Claude Fable 5.1 在相同層級接替 Claude Fable 5，每 token 價格相同，但長時間 agent 式 coding、多步驟研究，以及文件／試算表／投影片工作更強。**Claude Mythos 5.1** (`claude-mythos-5-1`) 是提供給 Project Glasswing 參與者的相同模型（兩項差異請參閱下方「Claude Mythos 5.1」一節）。相同的 1M token context window（預設值與最大值相同）、相同的 128K 最大輸出、與 Claude Fable 5 相同的 tokenizer（token 數量不變；如果從 Opus 4.7 之前的模型遷移，預期 token 約增加 30%，請依上方「遷移至 Claude Fable 5.1」中的 tokenizer 指引操作）。可在 Claude API、Amazon Bedrock（`anthropic.claude-fable-5-1`）、AWS 上的 Claude Platform、Google Cloud 與 Microsoft Foundry（Anthropic 託管）使用。既有 Claude Fable 5 prompt 應可直接正常運作。

**只有使用者明確選擇 Claude Fable 5.1 時才遷移至它**，規則與 Claude Fable 5 相同：它不是預設的 Opus 升級路徑。對於「升級至最新模型」的請求，目標仍是 `claude-opus-5`；文件本身的定位是「先從 Claude Opus 5 開始；需要要求最高的推理與長期 agent 式工作時，或對 Claude Opus 5 提高 effort 後的評估仍不理想時，使用 Claude Fable 5.1」。

**變更一行概括：**三項重大變更（強制工具選擇回傳 400；thinking 區塊只保留給產生它們的模型或更新的模型；thinking 區塊只保留在產生它們的對話中——文件將後兩項統稱為「保留的 thinking」）、五項新增功能（每則訊息的 effort、回合範圍的 system 訊息、工具呼叫之間的進度更新、更低的 cache-read 價格、內容來源），以及 agent loop 在三個可透過 prompt 調整的面向上有不同的行為。請閱讀符合來源模型的路徑：從 Claude Fable 5 來的話，以下全部直接適用；從 Claude Opus 5 來的話，還要閱讀「從 Claude Opus 5 遷移」一節；從 Opus 4.8 或更早版本來的話，先套用上方「遷移至 Claude Fable 5.1」一節（Opus 4.7 或更早版本還要先讀之前的 Claude Opus 5 一節），再套用本節。

### 重大變更 1：拒絕強制工具使用

`tool_choice: {"type": "any"}` 與 `tool_choice: {"type": "tool", "name": "..."}` 在 Claude Fable 5.1 與 Claude Mythos 5.1 上會回傳 400 `invalid_request_error`（Mythos Preview 也已如此）；適用於 Messages API、Message Batches API 與 token-counting endpoint：

```text
tool_choice: type "tool" and "any" are not supported for this model.
```

這是模型特定的限制，不是 always-on thinking 的結果（Claude Fable 5 與 Claude Opus 5 也預設 thinking，但仍接受強制工具選擇）。`{"type": "auto"}`（預設值）與 `{"type": "none"}` 不變。`disable_parallel_tool_use: true` 搭配 `auto` 仍然有效，但現在只表示*最多*一次呼叫；它與 `any`／`tool` 搭配時原本提供的「恰好一次工具」保證已不存在。

依意圖遷移：

- **引導模型使用工具：**保留 `tool_choice: {"type": "auto"}`（或省略），並在 prompt 中說明何時使用工具（「使用 `get_weather` 工具回答」）。Claude Fable 5.1 能可靠遵循明確的工具指示，先思考也會改善它傳入的參數。如果是*應用程式*（不是使用者）要求在多回合對話的目前回合進行特定呼叫，請在最新 `user` 回合後附加一則 `role: "system"` 訊息，指名工具、說明本回合必須呼叫，並要求 Claude 以該呼叫開頭；後續請求也要保留這則訊息在歷史中。
- **保證符合結構描述的參數：**在 `auto` 下，工具定義使用 `strict: true`（結構描述中設定 `additionalProperties: false`），即可恢復 `any` 提供的參數有效性保證。（CMEK 組織無法在 Fable 模型上使用包含 `strict: true` 的結構化輸出，請只依靠指示。）
- **擷取結構化資料：**如果強制呼叫只是為了取得 JSON，請改用結構化輸出（`output_config.format`）；請參閱「依來源模型的重大變更」下的預填替代表中的 `messages.parse()`／`output_config.format` 形狀。
- **advisor 工具：**Claude Fable 5.1 或 Claude Mythos 5.1 的 *executor* 同樣拒絕強制 `tool_choice`，請改在 prompt 中引導 advisor 呼叫（請參閱 `shared/tool-use-concepts.md` § Advisor）。

```python
# Before - 400 on Claude Fable 5.1
response = client.messages.create(
    model="claude-fable-5",
    max_tokens=4096,
    tools=[get_weather_tool],
    tool_choice={"type": "tool", "name": "get_weather"},
    messages=[{"role": "user", "content": "Check Tokyo, then summarize."}],
)

# After - let it think, name the tool, keep the schema guarantee with strict tool use
get_weather_tool["strict"] = True   # schema must set additionalProperties: false
response = client.messages.create(
    model="claude-fable-5-1",
    max_tokens=4096,
    tools=[get_weather_tool],
    tool_choice={"type": "auto"},
    messages=[{"role": "user", "content": "Use the get_weather tool to check Tokyo, then summarize."}],
)
```

### 重大變更 2：thinking 區塊只保留給產生它們的模型或更新的模型

每個 `thinking` 區塊都會記錄產生它的模型。Claude Fable 5.1 與 Claude Mythos 5.1 能讀取彼此的區塊，以及 Claude Opus 5、Claude Fable 5、Claude Mythos 5 與更早模型的區塊；這些較早模型不會將推理加密在 signature 中（Opus 4.8 與更早的 Opus、Sonnet、Haiku 4.5），因此對話*移至* `claude-fable-5-1` 時會保留既有推理。它們無法讀取 Mythos Preview 的區塊。**這種繫結是單向的：除了 Claude Mythos 5.1 外，沒有其他模型能讀取 Claude Fable 5.1 區塊。**

當請求帶有接收模型無法讀取的區塊時——路由切換、在其他模型上的 client 端重試、分類器拒絕 fallback（伺服器端或 SDK middleware）——API 會在模型看到之前捨棄它：請求成功，捨棄的區塊不計入 `input_tokens` 也不計費，目標模型會不帶該推理重新規劃（切換後第一個回合預期成本與延遲較高）。捨棄的區塊會使該請求從其位置起的快取前綴改變。未設定 `thinking-binding-controls-2026-08-01` beta header 時，捨棄會靜默進行；設定後，回應會帶有頂層 `input_transformations` 陣列，以 `reason: "model_binding_mismatch"` 指出每個被捨棄的區塊（形狀如下）。Amazon Bedrock 目前設定為只讀取較窄的集合（僅自家系列），請在推出時確認。

切換模型時仍要原樣傳回 thinking 區塊；API 會丟棄目標無法讀取的區塊且不計費，因此自行移除沒有可節省的 input token；自己移除區塊可能觸發順序／簽章 400，而 fallback-credit 重試必須原樣回傳遭拒的主體。

### 重大變更 3：thinking 區塊只保留在產生它們的對話中

公開文件將本項與重大變更 2 合併在*保留的 thinking*之下（「原樣傳回區塊，讓 API 決定模型能使用哪些」）；本項是對話檢查：編輯較早回合會使之後的每個 thinking 區塊失效。相關 API 欄位名稱是 `prefix_mismatch_behavior`／`prefix_binding_mismatch`，使用的是同一項檢查。

Claude Fable 5.1 thinking 區塊的 `signature` 也會記錄產生它的對話前綴——頂層 `system` prompt、`tools` 中的工具集合，以及區塊之前的每一則訊息（使用伺服器端壓縮時，前綴從最近的壓縮區塊開始）——還包含跨回合連接至前一個 thinking 區塊的鏈結（較早 thinking 區塊不屬於前綴，但每個區塊都記錄前一個區塊，因此可以從歷史*前端*移除區塊，不能從中間移除）。傳回 transcript 時，API 會檢查此前綴是否未變。Claude Code、claude.ai、Managed Agents 與 Agent SDK 會替你維持前綴；**如果程式碼自行建立 `messages` 陣列，請在遷移前檢查**（下方有三步驟檢查）。**適用對象：**2026 年 8 月 31 日或之後建立的新帳戶（Claude API 組織、Amazon Bedrock 帳戶、Google Cloud 專案、Microsoft Foundry 資源）。Anthropic 計畫在未來模型上對所有帳戶執行此檢查，因此即使目前帳戶尚未強制，也請現在採用這些模式。對較早建立的帳戶，API 會*記錄*不相符，但只有在請求選擇加入時才採取行動：設定 `thinking.block_binding.prefix_mismatch_behavior`——包括 `"error"` 在內的**任何值**都會讓請求選擇加入 enforcement，這也是在舊組織測試的方法——或單獨送出 `thinking-binding-controls-2026-08-01` header，讓請求選擇加入 beta 的預設 `drop_block`。如果你提供工具或 framework，讓使用者用自己的 API key 執行，請在欄位已設定時測試：使用新組織的使用者會比你更早受到強制執行。若要查看自己的組織是否預設強制，請在不帶 beta header 的情況下送出會編輯歷史的請求；若回傳指出該 header 的 400，就表示已強制。平台備註：選擇加入控制項本身（beta header、`prefix_mismatch_behavior`、`input_transformations`）在 Claude API 與 AWS 上的 Claude Platform 推出時可用，之後會在 Amazon Bedrock 與 Google Cloud 依模型提供（在此之前它們會拒絕該 header），Microsoft Foundry 不提供；沒有控制項的平台無法使用選擇加入測試路徑，復原方式是移除後重試（矩陣請見 `shared/platform-availability.md`）。

**會使後續每個 thinking 區塊失效的情況：**

- 保留較晚回合的同時編輯、重新排序或移除較早回合，包括刪除舊的工具結果（請改用伺服器端工具結果清除）。
- 將每次請求的文字（提醒、狀態列、token 計數）注入較早回合，下一次請求又將它移除或重建。
- 在同一對話的請求之間重建頂層 `system` prompt 或 `tools` 陣列。
- 從執行開頭以外的位置移除 thinking 區塊（見下方）。
- 較早回合中的影像或文件 URL 在後續請求提供不同的 byte；繫結的是 byte，不是 URL 字串，因此同一檔案輪替簽署 URL 沒問題。對跨回合參照的內容，請透過 Files API 上傳一次並傳送 `file_id`，或傳送 base64。

**會讓後續區塊保持有效的情況：**只能附加的歷史，包括附加的 `role: "system"` 訊息，以及留在原處的已清除回合範圍（`clear_at`）訊息或提醒文字區塊；移除一串位於開頭的 thinking 區塊，從最舊者開始（對話中的第一個區塊，或最近壓縮區塊之後的第一個區塊，接著依序移除）；重新排序但不變更 `tools`（以名稱排序的集合繫結，請在推出時確認），以及新增從未被參照的 `defer_loading: true` 工具；變更 `system`／`tools`／`messages` 以外的請求參數（`max_tokens`、包含 `effort` 的 `output_config`、`tool_choice`、`metadata`）；新增、移動或移除 `cache_control` 標記；輪替後仍回傳相同 byte 的簽署 URL；伺服器端壓縮與 context editing，包括清除 thinking 區塊（這些不算編輯，因為檢查比較的是你送出的對話，不是伺服器編輯後的副本；壓縮後，檢查的前綴從壓縮區塊開始）。

**在執行此檢查的位置，重播已失效區塊的請求會被拒絕**，回傳 400 `invalid_request_error`，且在任何輸出前決定。重試相同主體會以相同方式失敗；token-counting endpoint 也執行同一項檢查。（在 Message Batches API 中，*未設定*的預設值會捨棄失敗區塊而不是使項目失敗；若希望批次項目失敗，請明確設定 `"error"`。）

```text
messages.5.content.0: Invalid `signature` in `thinking` block. The block is bound to a different conversation. Remove the block, or set `thinking.block_binding.prefix_mismatch_behavior` to "drop_block". That setting requires the `thinking-binding-controls-2026-08-01` value in the `anthropic-beta` header.
```

只有在請求未送出 beta header 時才會出現最後一句；訊息可能再加上一句指出第一則變更的訊息，這是可採取行動的診斷。（遭竄改或無法解密的簽章是不同的失敗：同樣的開頭子句，但**沒有**「繫結至不同對話」這句，永遠是 400，且 `prefix_mismatch_behavior` 不適用。）有兩種復原方式：

1. **從歷史中移除所有 `thinking` 與 `redacted_thinking` 區塊**（每個回合的 `text` 與 `tool_use` 區塊保留），然後重試一次，這是沒有 beta 的路徑。模型會在沒有這些區塊所帶推理的情況下回答該回合。在壓縮等邊界只捨棄一次 thinking 幾乎沒有影響；如果整合每次請求都讓自己的歷史失效，就會失去那些推理並每次重新啟動 prompt cache，可能提高每項任務的成本。請將這視為一次性復原，而不是穩態模式。
2. **要求 API 捨棄而不是回傳錯誤：**

```http
POST /v1/messages
anthropic-beta: thinking-binding-controls-2026-08-01

{"model": "claude-fable-5-1", "max_tokens": 4096,
 "thinking": {"type": "adaptive", "block_binding": {"prefix_mismatch_behavior": "drop_block"}},
 "messages": [ ...full history with thinking blocks replayed verbatim... ]}
```

`thinking.block_binding.prefix_mismatch_behavior` 可接受 `"error"` 或 `"drop_block"`。不同介面的預設值不同：沒有 header 時，強制執行的帳戶在不相符時回傳錯誤（上方的 400）；單獨送出 header **會**將請求切換為 beta 自己的預設 `drop_block`，因此請明確設定欄位，不要依賴任一預設值（header 讓你能設定欄位，也會將 `input_transformations` 加入回應）。使用 `"drop_block"` 時，API 會捨棄第一個不相符的區塊與其後每一個 thinking 區塊（直到下一個壓縮區塊，如有；包括 assistant 回合中仍在等待 `tool_result` 的 `tool_use` 區塊），請求繼續處理，且每次捨棄都會回報在回應頂層的 `input_transformations` 陣列中：

```json
"input_transformations": [
  {"type": "thinking_dropped", "path": "messages.1.content.0", "reason": "prefix_binding_mismatch"}
]
```

捨棄只適用於*該次請求*：在工作階段其餘時間繼續送出 `"drop_block"`，或自行從歷史移除失敗區塊。`reason` 是 `"prefix_binding_mismatch"`（歷史變更）或 `"model_binding_mismatch"`（對話切換模型，不是程式碼 bug）；忽略不認識的 `type` 或 `reason` 項目，因為後續檢查會加入新值。設定 header 時，具有 thinking 能力的模型每個回應都會帶此陣列（沒有捨棄時為空，絕不為 `null`）；未設定時欄位不存在。串流時，它會在 `message_start` 的 `message` 物件中抵達（伺服器端 fallback 在串流中途發生時，最終 `message_delta` 也會再抵達）。不帶 header 傳送 `block_binding` 會回傳 400，結尾為 `block_binding: Extra inputs are not permitted`。此物件可與 `thinking.type: "adaptive"` 和 `"enabled"` 一起接受，不強制對話檢查的模型也會接受它，並只回報模型檢查所造成的捨棄，因此同一請求主體可跨模型使用。推出時的 SDK 會在 beta namespace 中為它建立型別（`client.beta.messages.create(..., thinking={"type": "adaptive", "block_binding": {"prefix_mismatch_behavior": "drop_block"}}, betas=["thinking-binding-controls-2026-08-01"])`）；推出時的型別列舉名稱（例如 `PrefixMismatchBehavior`）是開放的，如果欄位尚未具型別，請改用 `extra_body`／cast。某些較舊工具將欄位寫成 `block_binding.mismatch_behavior`，這是未文件化的別名；請使用標準名稱，絕不要同時送出兩者。

**既有整合的三步驟檢查：**

1. 擷取它在幾個正常回合中送出的精確請求主體，包括產品有使用壓縮或工具變更時的情況。對每一對連續請求，比較 `system` prompt、`tools` 陣列與 `messages` 的共用前綴；直到新附加回合之前，它們應該逐 byte 相同。
2. 對 `claude-fable-5-1` 執行一般多回合工作階段，帶上 `thinking-binding-controls-2026-08-01` header 與 `prefix_mismatch_behavior: "drop_block"`，並記錄每個回應的 `input_transformations`。每回合都是空陣列表示歷史完整；`prefix_binding_mismatch` 項目表示 `path` 所指區塊之前的內容自上次請求後變更；`model_binding_mismatch` 項目表示對話切換了模型。只要平台提供該控制項（請參閱上方的平台備註；其他平台的復原方式是移除後重試），此方法可適用於任何組織，因為設定欄位會讓請求選擇加入 enforcement。在 CI 中設定 `"error"`，讓編輯使執行失敗。
3. 選擇一個正式環境設定，並在 `thinking-binding-controls-2026-08-01` header 下**明確設定**（上方說明的預設值各介面不同）：`"error"` 表示 prefix mismatch 只能代表程式碼 bug；或使用 `"drop_block"`，在降級而非失敗的情況下繼續，並監控 400 或 `input_transformations` 項目。不要讓欄位未設定：對 2026-08-31 之前建立的帳戶，沒有 header 且欄位未設定時，檢查只會在伺服器端記錄，不會有 400 或可監控的 `input_transformations`（請參閱上方的預設值備註）。

**讓 harness 相容：將每一項 transcript 編輯替換為只能附加的形式：**

| 原本的作法 | 改用此作法 |
|---|---|
| 在工作階段中途編輯 system prompt | 在工作階段開始時凍結頂層 `system`；在變更生效的位置附加 `{"role": "system", "content": "..."}` 訊息（GA，不需 header；請參閱 `shared/prompt-caching.md` § Mid-conversation system messages）。它具有 system prompt 權限，並成為後續區塊所繫結的前綴一部分。 |
| 在工作階段中途編輯 `tools` 陣列 | 在工作階段開始時於 `tools` 宣告完整集合（對一開始隱藏的工具設定 `defer_loading: true`），並在 `role: "system"` 訊息中送出 `tool_addition`／`tool_removal` 區塊（beta `mid-conversation-tool-changes-2026-07-01`；`shared/tool-use-concepts.md` § Mid-conversation tool changes）。 |
| 注入每回合提醒並在下次請求刪除 | 將它作為回合範圍的 system 訊息送出（`clear_at: "next_user_message"`，下方新增項目 2），位於 `tool_result` 訊息之後，並保留在歷史中；沒有該 beta 時，在相同 user 訊息中 `tool_result` 區塊之後加入文字區塊，較早副本留在原處。 |
| 刪除舊工具結果／在 client 端剪除舊回合 | 使用伺服器端 context editing（工具結果清除、thinking 清除）或壓縮，這些不算編輯（檢查比較的是你送出的對話）。 |
| 進行壓縮 | 優先使用伺服器端壓縮（beta `compact-2026-01-12`；其 `instructions` 參數接受自訂摘要 prompt）或 context editing，兩者都不算編輯。若在 client 端進行，**簡單壓縮**是建議形式：對話過長時，將它摘要成單一訊息，下一個請求以該摘要加上新的 user 回合開始，不要重播其他內容——沒有較早回合，也沒有較早的 thinking 區塊。帶過來的內容不會繫結舊 transcript；Claude 模型受訓處理採用此方案的長期任務，表現與更複雜的方案相近。任何壓縮都會重設 cache，不要在工具回合中間壓縮（仍等待 `tool_use` 的 `tool_result` 的 assistant 回合應原樣帶回 thinking）。摘要之前的 thinking 不會延續，因此摘要就是模型對該工作的全部認知；請告訴摘要器要保留什麼（行為變化中的壓縮 prompt，或伺服器端壓縮的 `instructions`）。 |
| 跨回合以 URL 參照影像／文件 | 透過 Files API 上傳一次並傳送 `file_id`，或傳送 base64。 |

兩種 client 端壓縮形狀會在此檢查下**失效**。*保留尾端的壓縮*（摘要較舊回合，逐字保留最近回合）會在保留回合上失敗：它們的 thinking 區塊是在完整歷史存在時建立，摘要後重播時即使保留回合本身未變，也會回傳 400；請從保留回合移除 thinking 區塊（text 與工具呼叫可以保留），或設定 `"drop_block"`。*背景（非同步）壓縮*（在關鍵路徑之外壓縮，對話繼續時替換摘要）也會以相同方式失敗，但影響更多 transcript：摘要到達時，交換點上方已有幾個較新的回合，而它們的 thinking 區塊都早於交換點；每次仍帶有交換前 thinking 區塊的請求都送出 `"drop_block"`（每次都要繼續傳送），或自行移除那些區塊；交換後第一個回應的 `input_transformations` 會精確列出哪些區塊被捨棄，或者同步壓縮。若使用壓縮 beta 的 `pause_after_compaction` 流程，且在壓縮區塊後重新插入 assistant 回合，也同樣適用：移除其 `thinking` 區塊，或送出 `"drop_block"`。從 transcript *中間*剪除個別回合會使後續每個 thinking 區塊失效，沒有任何 client 端形式能避免；請對正在變更的指示使用對話中途 system 訊息，或使用伺服器端 context editing 進行選擇性移除。

### 從 Claude Fable 5 原樣沿用的內容

API 介面、限制、每 token 定價、tokenizer、always-on adaptive thinking、拒絕處理與 `stop_details` 類別都與 Claude Fable 5 相同：除了 `{type: "adaptive"}` 外沒有 `thinking` 設定（`disabled` 與 `budget_tokens` 都會回傳 400），`display` 預設為 `"omitted"`，永遠不回傳原始 chain of thought，交錯 thinking 會自動處理（不需 header），不支援 assistant 預填，沒有非預設取樣參數，可快取 prompt 的最小長度為 512 token，支援對話中途的 system 訊息與工具變更。讀取 `content` 前必須先處理 `refusal` stop reason；分類器涵蓋與 Claude Fable 5 相同的類別（比 Claude Opus 5 只涵蓋 cyber 的分類器更廣），因此除了 `"cyber"` 外，還可能看到 `stop_details.category` 值 `"bio"` 與 `"reasoning_extraction"`。差異如下：

- **Fallback：**伺服器端 `fallbacks`（`"default"` 或陣列形式）與 SDK middleware 的運作方式和 Claude Fable 5 相同；允許的目標是 `claude-opus-4-8` 與 `claude-opus-5`，且依類別路由是在伺服器端套用、不公開（有些類別拒絕時沒有 fallback）。fallback 模型無法讀取 Claude Fable 5.1 的 thinking 區塊，因此 API 會捨棄它們（重大變更 2）。Fallback credit 的運作方式和 Claude Fable 5 相同：Claude Fable 5.1 與 Claude Mythos 5.1 在拒絕時產生 `fallback_credit_token`，可在任一允許目標上兌換（上方「遷移至 Claude Fable 5.1」拒絕一節的模式 3；Claude Mythos 5.1 的 fallback 目標截至 8 月下旬尚未接線，請在推出時確認，見下方「Claude Mythos 5.1」一節）；輸出前的拒絕不計費，credit 會退還切換模型的 prompt-cache 成本。
- **資料保留：**Claude Fable 5.1 與 Claude Mythos 5.1 和 Claude Fable 5 一樣是 Covered Models，需要保留 30 天，**除非 Anthropic 明確授權，否則不可使用零資料保留**。與 Claude Fable 5 相同，沒有 30 天保留的組織或 workspace 所送請求會回傳 `400 invalid_request_error`（「In order to access this model, your organization or workspace must have data retention enabled.」），請在除錯 payload 前檢查保留設定。（較早的推出文件草稿描述的是 404 且模型從 `/v1/models` 隱藏；最終文字是 400。如果 ID 收到 404，先檢查保留設定。）需要該模型的 ZDR 組織應聯絡 Anthropic account team（「明確授權」路徑），或為一個 workspace 啟用 30 天保留；已能存取模型的 ZDR 組織代表具備此授權，不表示要求已取消。（推出文件較早草稿描述企業可享有至 2026-12-31 的限時豁免；該句已於 8 月 28 日刪除，不要引用。）
- **Priority Tier：**Claude Fable 5.1 或 Claude Mythos 5.1 不支援（Claude Fable 5 支援）。使用 Priority Tier 的 Claude Fable 5 呼叫端遷移後會失去此功能。
- **速率限制：**Claude Fable 5.1 與 Claude Fable 5 共用一個「Fable 5.x」池（流量合併；Mythos 模型依相同條件共用另一個池）；如果遷移期間同時執行兩者，請重新建立剩餘容量基準。
- **定價：**每 MTok $10／$50，5 分鐘 cache write $12.50、1 小時 cache write $20、batch $5／$25，全部與 Claude Fable 5 相同；但**cache read 為每 MTok $0.25**（基礎 input 的 0.025 倍，其他模型為 0.1 倍；Claude Mythos 5.1 是否共用 0.025 倍費率，推出時確認），是 Claude Fable 5 費率的四分之一，也是 Claude Opus 5 的一半。重新讀取快取前綴的長時間 agent 工作階段能取得大部分節省；`shared/prompt-caching.md` 中的快取損益平衡計算也隨之改變；而且現在 miss 相對於 hit 昂貴許多，因此保持 cache warm 更重要：每則訊息的 effort 與回合範圍 system 訊息部分就是為此存在；閒置 5–60 分鐘時，以預設 5 分鐘 TTL 重新送出 `max_tokens: 0` keep-alive 通常比 1 小時 TTL 便宜（以 `stream` 關閉送出，不要搭配結構化輸出或 Batches；請參閱 `shared/prompt-caching.md` § Choosing the TTL）。預期每項任務成本低於或等於 `shared/cost-optimization.md` 中 Claude Fable 5 的數字。
- **工具介面：**工具版本與 Claude Fable 5 相同：code execution `code_execution_20250825`／`_20260120`／`_20260521`（程式化工具呼叫需要 `_20260120` 或更新版本）、tool search（`tool_search_tool_regex_20251119`、`_bm25_20251119`）、computer use `computer_20251124`、browser use、結構化輸出、具動態篩選的 web fetch（`web_fetch_20260318`），以及 advisor tool（作為 executor 或 advisor；Claude Fable 5.1／Claude Mythos 5.1 advisor 會回傳加密的 `advisor_redacted_result`）。Task budgets：beta（`task-budgets-2026-03-13`，最低 20k），請在推出時確認。
- **內容來源（新增，不需變更請求）：**Claude Fable 5.1 與 Claude Mythos 5.1 的文字在所有平台帶有 Anthropic 的統計文字浮水印（沒有額外 token 或隱藏字元，不含組織資訊）。Claude 在 code-execution sandbox 中產生且受支援的影像、音訊與影片檔案，透過 Claude API 的 Files API 下載時會帶有簽署的 C2PA Content Credentials；manifest 會增加幾 KB，因此下載檔案的大小與 checksum 會和容器內的檔案不同；文字、PDF 與 office 檔案不會簽署。Claude API 以外的平台範圍在推出時確認。
- **Bedrock／Google Cloud 上的 1M context 與 batch 300k-output beta：**推出時尚未確定；在合作夥伴平台承諾任一功能前請先確認。

### 新的 API 功能

共有三項新增功能，各自位於 beta header 之後。全部都是選用的，遷移後的請求不使用它們也能運作；但前兩項是讓 harness 保持快取友善並保留 thinking 的方式，因此修改 agent loop 前請先閱讀。

**1. 每則訊息的 effort——beta `mid-conversation-output-config-2026-07-01`。**在 Claude Fable 5.1、Claude Mythos 5.1 與 Claude Opus 5 上（Claude API；Bedrock／Google Cloud／Foundry 在推出時尚未確認，且 Claude Opus 5 不包含在 Bedrock），內容為空且含 `output_config: {effort: ...}` 的 `role: "system"` 訊息，可從該處起變更 effort 而不使 prompt cache 失效；困難步驟提高，例行步驟降低：

```http
POST /v1/messages
anthropic-beta: mid-conversation-output-config-2026-07-01

{"model": "claude-fable-5-1", "max_tokens": 4096,
 "output_config": {"effort": "high"},
 "messages": [
   {"role": "user", "content": "Plan the migration."},
   {"role": "assistant", "content": "Here's the plan: ..."},
   {"role": "system", "content": [], "output_config": {"effort": "low"}},
   {"role": "user", "content": "Now rename the config file."}
 ]}
```

值為層級名稱（`low`、`medium`、`high`、`xhigh`、`max`）。新層級從下一個 `user` 回合開始生效，直到稍後的 `role: "system"` 訊息變更它為止。只有 effort 的訊息不帶文字，因此不適用對話中途 system 訊息的放置規則，可以位於 `messages` 的任何位置，包括開頭，或 assistant 回合與下一個 user 回合之間。用這種方式降低 effort 很可靠；提高 effort 最適合大幅跳升（例如從 `low` 到 `xhigh`）。在 Claude Fable 5.1 上，優先使用此形式，不要在請求之間變更頂層值：頂層變更會重啟 cache，且引導模型的可靠性較低（它先前的回覆是在上一層級寫成，往往會保持一致），但頂層變更不會使 thinking 區塊失效。不支援的模型（包括 Claude Fable 5）會回傳 400：`output_config.effort requires a model that supports per-turn effort; this model does not`。較舊的拼法 `mid-conversation-effort-2026-08-01` 與 `per-turn-control-2026-07-01` 仍會解析至相同功能，但未文件化，不要用它們寫新程式碼。推出時確認 beta 是對所有組織開放，還是維持受限（allowlist）beta。這取代 Claude Opus 5 檢查清單中「本次推出不含 per-turn effort」的註記。

**2. 回合範圍的對話中途 system 訊息——beta `mid-conversation-system-clear-at-2026-08-21`。**harness 常需要告訴模型只對一個回合成立的事情（「執行程式碼前檢查收件匣」、「使用者看不到那項工具輸出」）。注入提醒後在下一次請求刪除屬於歷史編輯，會重啟 prompt cache；在 Claude Fable 5.1 上還會使之後的每個 thinking 區塊失效。請改用 `clear_at: "next_user_message"` 的 `role: "system"` 訊息：其文字在目前回合具有 system-prompt 權限，等稍後存在 `user` 訊息時就停止呈現。**請原樣持續傳送它**，因為它仍留在 `messages` 中，因此較早內容不變、cache 持續相符、後續 thinking 區塊維持有效，而且已清除的訊息不耗用 input token。

```http
POST /v1/messages
anthropic-beta: mid-conversation-system-clear-at-2026-08-21

{"model": "claude-fable-5-1", "max_tokens": 4096,
 "tools": [...],
 "messages": [
   {"role": "user", "content": "Run the analysis script."},
   {"role": "assistant", "content": [{"type": "tool_use", "id": "toolu_01", "name": "bash",
    "input": {"command": "python analyze.py"}}]},
   {"role": "user", "content": [{"type": "tool_result", "tool_use_id": "toolu_01",
    "content": "Analysis complete; report written."}]},
   {"role": "system", "clear_at": "next_user_message",
    "content": "Results have landed in your inbox; check it before running more code."}
 ]}
```

主要用途是工具 loop 中的每回合提醒：在每個希望模型看到提醒的 `tool_result` user 訊息後附加該訊息，並且**保留所有較早副本在原處**；只有 `tool_result` 的 user 訊息會算作下一個 user 訊息，因此較早副本已被清除（不呈現、不計費，但仍屬於 thinking 所繫結的前綴），模型只讀取最新副本。規則：`clear_at` 可取 `"never"`（預設）或 `"next_user_message"`；回合範圍訊息只能是 `text`（不可含 `tool_addition`／`tool_removal` 區塊，也不可含 `output_config`），不可帶 `cache_control`（將斷點放在前一個 user 回合），並遵循一般放置規則——一則回合範圍訊息若直接接著另一則 `user` 訊息會回傳 400，因此將一個工具回合的所有結果放在同一則 user 訊息，並在它之後放提醒。刪除、改寫、依目前狀態重建，或變更已送出副本的 `clear_at`，都和任何其他編輯一樣。使用的模型與平台和對話中途 system 訊息相同（在 Bedrock 與 Google Cloud 上，依平台傳送 beta 的方式傳送 beta 值）；推出時 SDK 可能尚未為欄位建立型別，請透過 `extra_body`／cast 傳送。**沒有 beta 時**，在相同 user 訊息的 `tool_result` 區塊後附加 `text` 區塊，並保留較早副本；模型會依最新副本行動。

**3. 工具呼叫之間的進度更新——`thinking.display: "updates"`，beta `thinking-display-updates-2026-08-18`。**Claude Fable 5.1、Claude Mythos 5.1 與 Claude Fable 5 會在工具呼叫之間寫下簡短進度更新——剛找到什麼、下一步要做什麼——每則都以自己的 `thinking` 區塊回傳，帶有自己的 signature，緊接在它所引入的工具呼叫之前，與同一位置的任何 reasoning 區塊分開。在預設 `display: "omitted"` 下，這些區塊和 reasoning 一樣會回傳空內容，因此長時間 agent 回合可能看起來沉默數分鐘。要求 `display: "updates"` 後，進度更新會以文字回傳，而 reasoning 維持隱藏：

```http
POST /v1/messages
anthropic-beta: thinking-display-updates-2026-08-18

{"model": "claude-fable-5-1", "max_tokens": 4096,
 "thinking": {"type": "adaptive", "display": "updates"},
 "tools": [...],
 "messages": [{"role": "user", "content": "Review the PRs open against our billing service."}]}
```

使用方式：在 `"updates"` 下，**任何含非空文字的 `thinking` 區塊都是進度更新**（通常是一或兩句），請將它呈現為狀態列；空區塊不要呈現（進度區塊在任何 `display` 值下都可能是空的）。串流時，進度區塊會在它引入的 `tool_use` 區塊之前，以 `thinking_delta` 事件串流文字；只要 `thinking_delta` 帶有非空文字，就將區塊視為進度更新；區塊開啟前暫停數秒是正常的。回應可能一則也沒有，模型也可能跳過任何間隔，因此請支援零個或多個。若工具呼叫或結果後不久回應因 `max_tokens`、`model_context_window_exceeded` 或 `stop_sequence` 停止，最後區塊可能是代表未完成工作的進度區塊，其文字精確為 `This part of the response was interrupted before it finished.`；若要繼續，將 assistant 回合原樣傳回，並附加新的 `user` 訊息（該回合每個 `tool_use` 都要有一個 `tool_result`）。更新會以完整長度計入 `usage.output_tokens`，而不是只計入摘要。像任何 thinking 區塊一樣，原樣回傳進度區塊。`"summarized"` 也會回傳其文字，並與 reasoning 摘要混合。所有平台都可用；在 Bedrock、Google Cloud 與 Foundry 上，依該平台傳送 beta header 的方式傳送 beta 值；沒有 beta 時，`"updates"` 會因未知 `display` 值而被拒絕。

### 從 Claude Opus 5 遷移

除了三項重大變更外：`thinking: {type: "disabled"}` 在**任何** effort 下都會回傳 400（Claude Opus 5 在 `high` 或更低層級仍接受）；請移除它，以較低 effort 控制支出，並重新檢查 `max_tokens`。Claude Opus 5 在工具呼叫*之間*寫出的文字會以 `text` 區塊回傳；Claude Fable 5.1 則以進度更新 `thinking` 區塊回傳，在預設 `"omitted"` 下為空；如果 UI 會呈現那段敘述，請設定 `display: "updates"`（或 `"summarized"`）。分類器集合更廣（除了 `cyber` 還有 `bio`、`reasoning_extraction`）。ZDR 不再適用（Claude Opus 5 可在 ZDR 下使用）。定價從每 MTok $5／$25 變為 $10／$50，cache read 為 Claude Opus 5 費率的一半；512 token 快取最小值不變。Claude Opus 5 已支援每則訊息的 effort，因此使用它的 harness 在此不需變更。

從 Opus 4.8 或更早版本來的話，先套用上方「遷移至 Claude Fable 5.1」一節（Opus 4.7 或更早版本還要先讀之前的 Claude Opus 5 一節），再套用本節；並為歷史編輯檢查預留時間：為 Opus 4.8 與更早版本撰寫的整合通常會截短舊回合、移除或重建較早訊息，或每次請求重新整理 `system` prompt，而 Opus 4.8 從未拒絕這些做法。請檢查接近 512 token 快取最小值的 prompt。

### Claude Mythos 5.1

`claude-mythos-5-1` 與 Claude Fable 5.1 是相同模型，具備相同的能力、限制、API 行為與每 token 定價（cache-read 費率在推出時確認）；只提供給獲核准的 Project Glasswing 客戶，而且除了 Claude Fable 5.1 外，這是唯一能讀取 Claude Fable 5.1 thinking 區塊的模型（也能讀取 Claude Mythos 5 的區塊，反之則不行）。切換 ID 前，請先向 account team 確認組織有權限。從 Claude Mythos 5 遷移者的角度看有兩項差異：**Claude Mythos 5.1 會執行安全防護**，其內容取決於組織獲准加入的存取計畫（Claude Mythos 5 完全不執行）；請處理 `stop_reason: "refusal"`、讀取 `stop_details.category`，並像 Claude Fable 5.1 一樣設定 fallback（其 fallback 目標截至 8 月下旬尚未接線，請在推出時確認）；此外，它**不提供於 AWS 上的 Claude Platform**（Claude API、Amazon Bedrock 僅在 us-east-1 可用且未公開列出，模型 ID 為 `anthropic.claude-mythos-5-1`、Google Cloud、Microsoft Foundry）。它與 Claude Mythos 5 共用 Mythos 速率限制池。Claude Mythos 5 的存取權是否會自動延續，推出時確認。

### 相較於 Claude Fable 5 的能力提升

差距在較高 effort 層級最明顯。共有六個面向：**長時間工作階段的 agent 式 coding**（多檔案功能、大型重構與遷移、除錯、跨數小時工作階段的程式碼審查）；**文件、試算表與投影片的知識工作**（從第一個問題到完成文件、含即時公式的試算表，或從空白頁建立投影片）；**研究與搜尋**（會追蹤找到內容的多步驟網路研究）；**vision**（PDF 中的密集圖表、申報文件與巢狀表格，在能使用 crop 與 zoom 工具時最強）；**深入 1M window 的長 context 擷取**；以及**computer use**（更可靠地操作瀏覽器與桌面應用程式，並從失敗步驟復原）。多語言表現與 Claude Fable 5 相當。以下內容等待推出時確認：它是否較少在工作途中擴大請求範圍，以及長時間工作階段開始時只給一次的指示是否能更持久；如果後者成立，請移除為 Claude Fable 5 每隔幾回合插入的重複指示並重新測試（下方每回合 batching 提醒是另一種情況：它針對下一回合的一項行為，因此測量顯示有幫助時要保留）。

### 行為變化（可透過 prompt 調整）

這些都不會破壞 API。上方「遷移至 Claude Fable 5.1」中的行為指引（更長的回合、讓進度宣告有依據、說明邊界、委派、記憶介面、可讀性附錄）仍然適用；以下是 Claude Fable 5.1 特有的差異。其中三項不需變更程式碼就會出現：它較少批次處理隱含的工具呼叫、工具呼叫之間較少敘述，以及在 `low` effort 下更常從記憶回答。

**Effort。**從 `high`（預設值）開始，即使已在 Claude Fable 5 上執行過一次，也要重新進行 effort 掃描；不同模型的層級名稱不代表相同程度的 thinking。Claude Fable 5 之上的提升遍及各層級，在較高設定最明顯；在 `medium` 下，結果大致相當於較低成本的 Claude Fable 5，因此評估顯示品質能維持時，請降至 `medium` 或 `low`。在 `high` 以上設定較大的 `max_tokens`，它是總輸出的硬限制（thinking 加回應）。在 `low` 下，Claude Fable 5.1 的每項任務成本往往可與 Opus 和 Sonnet 競爭且表現更好；在選擇更便宜模型前，請將低 effort 的 Fable 與低於 frontier 的使用量比較。每則訊息的 effort（新增功能 1）讓單一對話可混用層級而不重設 cache。

**`xhigh` 與 `max` 下的長篇交付物。**在 `xhigh`，尤其是 `max` 下，模型在開始寫作前會思考更久。當單一請求要求長篇交付物——完整重寫長文件、大型表格、完整程式碼檔案——模型可能在 thinking 中草擬大部分內容，然後又在回覆中重新寫出：等待時間更長，輸出 token 約增加一倍。最簡單的修正是在 `high`（本來就建議從此開始）執行這些請求，只在測得品質提升的位置提高層級。如果確實使用 `xhigh`／`max`，請設定 `max_tokens`，為 thinking 與回覆都保留空間，並將下方內容附加在 user 訊息末尾；它會讓文字與程式碼請求的 thinking 大幅縮短（將括號替換為請求實際的 `max_tokens`，例如 64,000）。如同此模型其他附加的每次請求備註（上方新增功能 2），後續請求要逐 byte 保留每個較早副本，且各自維持送出時的值；移除或重建其中一個就是歷史編輯，會使其後的 thinking 區塊失效：

> Everything Claude produces in one reply, including any reasoning or drafting it does before the reply, counts toward a single limit of about [max_tokens] tokens. If that limit is reached before the reply is finished, the person receives a cut-off response and has to start over. Composing an entire output or deliverable in full as reasoning and then again as a reply would double the length of the turn without improving the result, so Claude doesn't do that.
> Instead, when the person has asked for a long or effort-intensive deliverable such as a multi-section document, a large table or dataset, or a complete code file, Claude spends extra effort on understanding the request, checking the inputs Claude's answer depends on, settling the structure and other difficult decisions, and otherwise using the reasoning space to reason and the output space to write an output. If Claude plans well then it should not need to draft its output multiple times (and Claude is pretty good at planning, so this should not be an issue).

**在 agent loop 中批次處理獨立的工具呼叫。**當請求明確列出數個要擷取的項目時，Claude Fable 5.1 會平行發出這些呼叫；標準 function calling 不受影響。在長時間 agent loop 中，若下一批獨立讀取只被*隱含*要求（自訂 coding agent、bash 與編輯器 harness、computer use），它可能每回合只發出一個呼叫，而 Claude Fable 5 原本會批次處理數個；答案相同，但往返與實際耗時更多。先測量：追蹤含有多於一個工具呼叫的 assistant 回合比例，只有比例偏低時才加入提醒（過度批次處理會表現為在依賴的結果到達前就發出呼叫）。放置位置比措辭更重要——放在目前請求末尾附近的一句話，影響遠大於相同文字放在 system prompt 或工具描述中。每次把工具結果傳回時，請在該 user 訊息之後，以回合範圍的 system 訊息附加這句話（`clear_at: "next_user_message"`，新增功能 2）；沒有 beta 時，則在相同 user 訊息的 `tool_result` 區塊後附加 `text` 區塊——**每回合附加新的副本，並逐 byte 保留較早副本**；重寫較早回合以移除它們會重啟 cache，且在此模型上使其後 thinking 區塊失效。請保留「privately」一詞；沒有它時，模型有時會回答提醒本身（「不需要其他內容」），而不是在最終回覆中回答使用者：

> First privately list what you need next; then request every item that doesn't depend on another's result in this one response.

**面向使用者的進度更新。**Claude Fable 5.1 在長時間工具呼叫回合中產生的面向使用者更新比 Claude Fable 5 少，在較高 effort 與較長工具鏈中尤其如此。使用者可能看到 agent 沉默數分鐘，或看到只描述最後一步的最終訊息；它的 agent 式 coding 摘要也更短。依序處理：(1) **要求 `display: "updates"`**（新增功能 3），否則模型在工具之間的備註不會傳到你這裡；(2) **先移除為偏好更新的舊模型撰寫的 prompt 文字**（「把所有發現保留到最終回覆」、「不要敘述」），再新增任何內容；(3) 如果仍需要更多更新——配對程式設計、人機協作——加入簡短且具體的 system-prompt 行，說明何時需要面向使用者的文字：

> Before you start, say in a line what you're about to do; brief updates while you work help the user follow along. Close with a short recap that stands on its own - what you found, what you did, and what's next - so a reader who only sees the last message has the full picture.

相關地，**如果 harness 會壓縮或隱藏工具輸出，請告訴模型**；否則 Claude Fable 5.1 可能會執行命令來「顯示」使用者看不到的輸出。請以回合範圍 system 訊息交付（`clear_at: "next_user_message"`，新增功能 2）；沒有 beta 時，則在相同 user 訊息中與工具結果一起交付，並在後續請求保留原處：

> Only you see that command's output - the user's terminal shows at most a few lines of it. If the user needs to read any of it, put it in your reply.

**文字密度。**Claude Fable 5.1 的寫作通常更受偏好，但 prose 可能比 Claude Fable 5 更密集——句子更長、段落分隔更少。將「mannered prose」定義為反模式有所幫助；放在工作階段第一個 user 回合的風格指示，比放在 system prompt 中的相同文字更能持續：

> Mannered prose substitutes metaphor and flourish for direct statement. Instead of "a parameter worth varying," the mannered writer produces "a dial worth turning." Instead of "this point still matters," they write "this point earns its keep." The phrases exist to display the writer, not to convey the idea, and readers can tell. That is why mannered prose irritates: it makes the reader work harder so the writer can perform. It is also imprecise. Metaphors drag in connotations the writer did not choose and cannot control. The fix is to say what you mean. When a literal phrase is available, use it.

The short form - "Please remove all mannered prose." - also tends to work.

**格式。**較早模型在聊天中過度使用項目符號與粗體時，Claude Fable 5.1 反而會少用粗體、標題、清單與引號。**如果 prompt 含有反格式化文字，請移除它**，或替換為說明何時適合使用格式的規則：

> Use lists and bullet points when asked to, or when the content is multifaceted enough that they help with clarity. If the person explicitly requests minimal formatting, always format your responses without bullet points, headers, lists, or bold emphasis, as requested. In conversational, personal, or emotional exchanges, keep to plain prose.

摘要文件時，Claude Fable 5.1 比 Claude Fable 5 更可能重現來源文字，卻沒有將其標為引文。修正方式是在 system prompt 中放入一個完整的正確回應範例，包括使用者請求、回應，以及說明回應為何正確的一句話。將兩行 `[web_search: ...]` 替換為你自己的工具名稱，讓模型將它們讀作範本化工具輸出，而不是應直接產生的文字：

```xml
<example>
<user>look up how the Riverton Ledger and the Coast Dispatch each covered the Harbor Bridge closure and compare their reporting</user>
<response>
[web_search: Harbor Bridge closure Riverton Ledger]
[web_search: Harbor Bridge closure Coast Dispatch]
Both outlets agree on the basics: the bridge closed on March 3 after inspectors found cracked welds, and the state expects repairs to take about eight months. Where they differ is emphasis. The Ledger treats it as a local-economy story. The Dispatch frames it as a funding failure; its editorial calls the closure "entirely foreseeable." Read together, the Ledger explains who is affected now and the Dispatch explains how it came to this - neither account alone gives the whole picture.
</response>
<rationale>CORRECT: The response is organized around where the two outlets agree and differ, not as a walk through either article. Each outlet's reporting is conveyed in one or two sentences of the assistant's own indirect speech. One short marked phrase from one source; every other claim is reworded. The response is still specific and complete.</rationale>
</example>
```

**最大化長期執行能力。**Claude Fable 5.1 能執行很長的自主工作，但在複雜的非同步工作負載上，需要提醒它不要停在*描述*下一步（「接下來我會……」），或詢問原始請求已涵蓋的步驟是否要執行（「要我套用這項嗎？」）。使用者會覺得必須回覆「繼續」；這對配對程式設計尚可，卻限制模型的長期能力。兩項 system-prompt 新增內容一起使用時可降低這種情況；除非 context 緊張，否則兩項都套用，此時第一項仍保留大部分效果。第一項的開頭句（「使用者沒有在觀看」）是關鍵，請照原文保留；如果產品需要針對特定確認而停止，請加入列出這些確認事項的句子。此 prompt 可能讓模型較不常釐清模糊請求。使用任一區塊時，模型會多寫一點程式碼——主要是在已編輯的檔案中增加測試——因此請搭配上方「遷移至 Claude Fable 5.1」的「讓進度宣告有依據」稽核指示，以及下方的測試涵蓋率行。如果既有 prompt 要求模型在回報前測試或檢查工作，遷移時**保留它**；刪除驗證指示的 Claude Opus 5 指引不適用於此（暫定結論：依據的報告數量很少）。

> You are operating autonomously. The user is not watching in real time and cannot answer questions mid-task, so asking 'Want me to...?' or 'Shall I...?' will block the work. For reversible actions that follow from the original request, proceed without asking. Stop only for destructive actions or genuine scope changes the user must decide. Offering follow-ups after the task is done is fine; asking permission before doing the work is not.
>
> Exception: when the user is describing a problem, asking a question, or thinking out loud rather than requesting a change, the deliverable is your assessment. Report your findings and stop. Don't apply a fix until they ask for one.
>
> Before ending your turn, check your last paragraph. If it is a plan, an analysis, a question, a list of next steps, or a promise about work you have not done ('I'll...', 'let me know when...'), do that work now with tool calls. That includes retrying after errors and gathering missing information yourself. Do not stop because the context or session is long. End your turn only when the task is complete or you are blocked on input only the user can provide.
>
> Before running a command that changes system state (such as restarts, deletes, or config edits), check that the evidence actually supports that specific action. A signal that pattern-matches to a known failure may have a different cause.

第二項則告訴它維持使用者設定的範圍：

> \# Delivering work
> The user's request - or the plan they approved - sets the scope, and the scope is the deliverable: don't quietly narrow, widen, or swap it. Read ambiguity the way a careful colleague would: make routine judgment calls yourself, and check in only when different readings would lead to materially different work. If you see a real problem with the task as specified, say so in a sentence or two and keep building under stated assumptions; if the user hears the concern and reaffirms, that is their decision, so deliver the full request.
>
> If a question comes up partway, first do everything that doesn't depend on the answer; then state the assumption you made, or - when going ahead on a wrong guess would be unsafe or would make the work useless - put the question at the end of a turn that also delivers that progress. If one part turns out to be blocked, complete every other part in full and say exactly what you left out and why - the whole task is the deliverable, and scaling it down is the user's call, not yours. A step you have decided on is something to run, not to announce: describing the next step and ending the turn leaves it undone until the user replies.
>
> Keep changes to what the request needs. Something else you notice worth doing - cleanup or documentation the task didn't call for, a change to a file the task didn't require - is a suggestion to make at the end, not a change to make; actions clearly beyond what the ask implies, and risky or destructive ones, still need the user's go-ahead.

（公開片段使用 em dash 與省略號字元，而本檔案使用連字號與三個句點；隨附的 skill 僅使用 ASCII，這項差異不會影響模型。）

**範圍與測試涵蓋率。**要求實作開放式功能時，Claude Fable 5.1 會交付要求內容，有時還會做更多——修正附近程式碼、撰寫額外測試、將暫存檢查以永久測試檔案提交。它能良好回應明確說明哪些內容不要做的指示；使用下方 prompt 時，指南作者觀察到未要求新增內容顯著減少，提交的測試程式碼也少很多，而任務成功率沒有可測量的變化（如果只在意測試蔓延，較早且較短的形式——「將驗證指令碼放在儲存庫外，例如 /tmp，並刪除你新增的任何指令碼」——仍然有效）：

> If, while working or testing, you find a pre-existing bug, a performance concern, or behavior the task doesn't mention, don't fix, optimize or extend it in this change unless the requested behavior cannot work without it; report it as a follow-up in your summary. Where the task is ambiguous, implement the reading its wording and the surrounding code most directly support, state that assumption in your summary, and don't build for the other readings as well. Verify your work however you like; scratch scripts and quick checks need not be kept. Commit tests only where the task asks for them or this repository already keeps tests for this kind of change, sized like the neighboring test files - roughly one focused test per stated behavior - and don't turn scratch checks into additional permanent test files. This is about extras only: implement every behavior the task asks for, completely.

**低 effort 下觸發搜尋。**在 `low` effort 下，Claude Fable 5.1 呼叫搜尋或擷取工具的頻率低於 Claude Fable 5，更常憑記憶回答；對它認得但知識已過時的具名產品、模型與工具最明顯。提高這些回合的 effort（每則訊息的 effort，新增功能 1）通常是最簡單的修正。否則，請在 system prompt 中告訴它：認出名稱不等於知道目前狀態，這些名稱應依使用者寫法進行搜尋：

> When a query centers on a name you do not confidently recognize, or recognize from a fast-moving area like AI models and developer tools where the landscape shifts within months, the name itself is the thing to verify: search before answering, and include the name as the user wrote it in at least one query alongside any reformulations. This holds even when you have some background on it - partial background is exactly what makes an out-of-date answer sound authoritative, so familiarity is not a reason to skip the search.

**Vision：讓模型裁切、縮放並驗證。**Claude Fable 5.1 的純 vision 開箱即用表現更好；能反覆分析、裁切並以視覺驗證自己的工作時效果最佳。對複雜輸入——密集圖表、申報文件、PDF 中巢狀的表格、影片——請以 agent 方式執行，使用包含原始影像／影片與基本影像處理函式庫（PIL、OpenCV）的 container。如果 container 負擔太大，大部分提升來自單一 crop 工具：它接受 bounding box，回傳裁切並放大的區域（作法在 Claude Opus 5 一節的 vision 指引中）；這會以影像 token 而非 effort 來擴展測試時的計算量。在 `low` effort 下，模型可能只憑整體印象回答而不呼叫該工具，因此請檢查日誌是否有呼叫；若缺少，請提高影像回合的 effort。

**降低安全防護的誤判。**這些分類器產生的誤判少於 Claude Fable 5 上線時的分類器，而且允許在原始碼中尋找弱點；遭封鎖的請求仍會回傳 `stop_reason: "refusal"`，因此保留拒絕處理與 fallback。三種情況較容易產生誤判：compile-check 措辭（詢問「這個程式有 bug 嗎？」而不是「這個程式能無錯誤編譯嗎？」）；較不知名的程式語言（提供模型該語言是什麼及如何運作的 context，例如其文件）；以及將 base64 編碼資料回傳到模型 context 的工具（移除這些工具）。

**整個檔案重寫。**Claude Fable 5.1 比 Claude Fable 5 更可能在只需目標式編輯時重寫整個檔案，結果相同但會消耗更多輸出 token 與時間。在 system prompt（或第一則 user 訊息，效果相同）附加下方內容，可恢復小型與中型變更的目標式編輯：

> The number of tokens used to edit files is best minimized, all else being equal. Therefore, when it will not affect the end result, try to surgically edit a file rather than rewrite the entire thing.

**client 端壓縮的摘要 prompt。**明確告訴 Claude 壓縮摘要中要保留什麼時，Claude Fable 5.1 反應良好。伺服器端壓縮已經這麼做；若在 client 端壓縮（重大變更 3 下的簡單壓縮形式），下方摘要指示很有效。當摘要請求仍攜帶對話的 `tools` 時，最後一句很關鍵（重大變更 3 表示不能為一次請求刪除它們）：沒有最後一句，模型偶爾會呼叫工具，而不是寫摘要。

> Summarize the transcript inside <summary></summary> tags. Include relevant information in the summary such that this conversation will be continued by a new context window without needing to redo work or be reprovided with relevant constraints or context. Be sure to preserve: (1) any difficulties or problems that came up, and how they were handled or resolved; (2) any possibilities, options, or approaches that were raised, tried, or set aside, and why; (3) anything that was asked for, decided, agreed, ruled out, or established as a preference, constraint, or boundary - stated exactly; (4) exactly where things stand now - what has been covered, settled, or completed so far; (5) anything still open, unresolved, promised, or expected to happen next; (6) specific details that would be hard to reconstruct - names, numbers, dates, exact wording, links or references - kept exactly. Be complete on these even at the cost of length; keep everything else concise. Weight the two voices differently: keep what the user said, asked for, shared, or established carefully and close to their own words; your own explanations and reasoning can be condensed much further, to what they concluded or produced - as long as nothing in the six items above is dropped. Do not call any tools while writing this summary; respond with text only.

**coding 中不阻塞的 sub-agent。**如果 coding agent 委派給 sub-agent，當 lead 不被迫停止並等待每個 sub-agent 時，Claude Fable 5.1 會更快完成——在品質、token 使用量與成本相近的情況下，平均完成時間較短。讓啟動 sub-agent 的工具立即回傳，待結果準備好時在稍後的 user 訊息中交付給 lead；模型仍常常會選擇等待，因此也提供另一個等待 sub-agent 的工具。節省的時間來自 lead 能在其他工作上繼續進行的情況。（這延伸了上方「遷移至 Claude Fable 5.1」的非同步委派指引。）

### 從 Claude Fable 5 遷移至 Claude Fable 5.1 檢查清單

- [ ] **[BLOCKS]** 將 `model=` 字串更新為 `claude-fable-5-1`（從 Claude Mythos 5 來的 Project Glasswing 參與者使用 `claude-mythos-5-1`；先確認存取權）
- [ ] **[BLOCKS]** 移除 `tool_choice: {type: "any"}` 與 `{type: "tool", name: ...}`（也會在 `count_tokens` 與 Batches 上回傳 400）；使用 `auto` 加上 `user` 回合中的指示（若應用程式要求該呼叫，則附加 `role: "system"` 訊息）、以 `strict: true` 取得符合結構描述的參數、以結構化輸出擷取 JSON；刪除依賴強制工具呼叫的「找不到工具就重試」loop
- [ ] **[BLOCKS]** 如果來源是 Opus 等級或更舊模型（不是 Claude Fable 5），先套用上方 Claude Fable 5.1 遷移檢查清單（Opus 等級 → Fable 遷移），再閱讀「從 Claude Opus 5 遷移」§；`thinking: {type: "disabled"}` 現在在任何 effort 下都會回傳 400，工具之間的敘述移入 `thinking` 區塊，ZDR 不再適用，價格加倍
- [ ] **[BLOCKS]** 資料保留：需要保留 30 天（Covered Model；除非 Anthropic 明確授權，否則 ZDR 不可用）；ZDR 組織每個請求都會收到 `400 invalid_request_error`，與 Claude Fable 5 相同；除錯 payload 前檢查保留設定
- [ ] **[BLOCKS]** 每個回合都持續原樣傳回 `thinking` 區塊，包括空區塊與 `redacted_thinking`；歷史編輯檢查會拒絕編輯過的歷史
- [ ] **[BLOCKS]** 保留的 thinking／歷史編輯檢查（所有平台在 2026-08-31 或之後建立的新帳戶，以及設定 `prefix_mismatch_behavior` 或送出控制項 beta header 的任何請求；後續模型會對所有人強制執行）：停止在請求之間編輯歷史；凍結頂層 `system`，使用 `role: "system"` 訊息傳送工作階段中途指示，工具變更使用 `tool_addition`／`tool_removal`，每回合提醒使用回合範圍（`clear_at`）system 訊息；沒有該 beta 時，則使用附加在工具結果之後且永不刪除的 user-message 文字區塊；裁切使用伺服器端 context editing／壓縮（client 端只保留摘要），跨回合檔案使用 `file_id`。在提供控制項 beta 的平台上執行三步驟檢查（`shared/platform-availability.md`）（`prefix_mismatch_behavior: "drop_block"` + 記錄 `input_transformations`；修正每一個 `prefix_binding_mismatch`；模型切換後出現 `model_binding_mismatch` 是預期情況；CI 使用 `"error"`），再選擇正式環境設定並監控。如果你提供的工具會讓其他人使用自己的 key 執行，請在欄位已設定時測試。保留尾端與背景壓縮需要 `"drop_block"`（每次請求都要繼續傳送）或移除保留回合的 thinking；絕不要在工具回合中間壓縮
- [ ] **[TUNE]** Fallback：保留伺服器端 `fallbacks`（目標 `claude-opus-4-8`／`claude-opus-5`，路由未公開）或 SDK middleware；fallback 模型無法讀取 5.1 thinking 區塊（會被捨棄且不計費）；fallback credit 的運作方式與 Claude Fable 5 相同
- [ ] **[TUNE]** 如果使用者會觀看長時間工具呼叫回合，採用 `thinking: {type: "adaptive", display: "updates"}` 搭配 `thinking-display-updates-2026-08-18`（所有平台）；將非空 `thinking` 區塊呈現為狀態列，處理 interrupted-response sentinel，並原樣回傳它們
- [ ] **[TUNE]** 在 loop 混合困難與例行步驟時採用每則訊息的 effort（`mid-conversation-output-config-2026-07-01`；Claude Opus 5 也支援）；降低可靠，提高時需要大幅跳升；重新執行 effort 掃描（預設 `high`；`medium` 作為成本控制；`xhigh`／`max` 僅用於重視能力的工作；`low` 的每項任務成本常勝過低於 frontier 的模型）；為 `high`+ 設定 `max_tokens`
- [ ] **[TUNE]** Agent loop：測量多工具呼叫回合的比例；比例低時加入「privately 列出接下來需要什麼」提醒（每回合新附一份，保留較早副本）；加入進度更新與格式片段前，移除「把發現保留到最終回覆」／反敘述文字及反格式化規則
- [ ] **[TUNE]** 加入無人值守執行的自主性 + 範圍 prompt；harness 壓縮工具輸出時加入隱藏工具輸出備註；coding agent 加入目標式編輯與範圍／測試涵蓋率 prompt；`xhigh`／`max` 請求加入長篇交付物備註（使用實際 `max_tokens`）；client 端摘要時加入壓縮摘要 prompt；文字密集工作加入 mannered-prose 指示；搜尋產品加入名稱驗證行；vision 加入 crop 工具（或影像處理 container）
- [ ] **[TUNE]** Claude Fable 5.1 不支援 Priority Tier；速率限制與 Claude Fable 5 共用 Fable 5.x pool，請重新建立容量基準；cache read 是 Claude Fable 5 費率的四分之一（重新檢查快取損益平衡；閒置 5–60 分鐘時，使用 5 分鐘 TTL 的 `max_tokens: 0` keep-alive 通常優於 1 小時 TTL，需關閉 `stream` 傳送，不搭配結構化輸出或 Batches）；tokenizer 與 Claude Fable 5 不變，因此只有在來源不是 Claude Fable 5 時才需重新建立 token 數量基準
- [ ] **[TUNE]** 在 Claude API 上，透過 Files API 下載時，code-execution sandbox 產生的受支援影像、音訊與影片檔案帶有 C2PA manifest；大小與 checksum 會和容器內檔案不同（文字、PDF 與 office 檔案未簽署；Claude API 以外的平台範圍在推出時確認）；只針對已簽署媒體調整完整性檢查

---

## 驗證遷移

更新後，抽查實際使用的是否為新模型。將 `YOUR_TARGET_MODEL` 替換為遷移到的模型字串（例如 `claude-fable-5-1`、`claude-opus-5`、`claude-opus-4-8`、`claude-opus-4-7`、`claude-sonnet-5`、`claude-sonnet-4-6`、`claude-haiku-4-5`），並同步更新 assertion 前綴：

```python
YOUR_TARGET_MODEL = "claude-opus-5"  # or "claude-opus-4-7", "claude-sonnet-5", "claude-sonnet-4-6", "claude-haiku-4-5"
response = client.messages.create(model=YOUR_TARGET_MODEL, max_tokens=64, messages=[...])
assert response.model.startswith(YOUR_TARGET_MODEL), response.model
```

如需了解速率限制容量變化、定價或能力差異（vision、結構化輸出、effort 支援），請查詢 Models API：

```python
m = client.models.retrieve(YOUR_TARGET_MODEL)
m.max_input_tokens, m.max_tokens
m.capabilities["effort"]["max"]["supported"]
```

完整的能力查詢模式請參閱 `shared/models.md`。
