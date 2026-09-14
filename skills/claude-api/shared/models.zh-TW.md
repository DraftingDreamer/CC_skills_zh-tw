---
source_file: models.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 247f3f7adb64943c70a78643e2d6ebc48baa8da3d5369e1329153bd9dbc952f0
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `models.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Claude 模型目錄

**只使用本文件中列出的精確模型 ID。** 絕不要猜測或自行構造模型 ID——錯誤的 ID 會導致 API 錯誤。盡可能使用別名。如需最新資訊，請 WebFetch `shared/live-sources.md` 中的 Models Overview URL，或直接查詢 Models API（見下方「以程式探索模型」）。

## 以程式探索模型

如需**即時**功能資料——情境視窗、最大輸出 token 數、功能支援（thinking、視覺、effort、結構化輸出等）——請查詢 Models API 而非依賴下方的快取表格。當使用者問「X 的情境視窗是多少」、「模型 X 支援視覺/thinking/effort 嗎」、「哪些模型支援功能 Y」，或想在執行時依功能選擇模型時，請使用此方法。

```python
m = client.models.retrieve("claude-opus-4-8")
m.id                 # "claude-opus-4-8"
m.display_name       # "Claude Opus 4.8"
m.max_input_tokens   # context window (int)
m.max_tokens         # max output tokens (int)

# capabilities is an untyped nested dict - bracket access, check ["supported"] at the leaf
caps = m.capabilities
caps["image_input"]["supported"]                       # vision
caps["thinking"]["types"]["adaptive"]["supported"]     # adaptive thinking
caps["effort"]["max"]["supported"]                     # effort: max (also low/medium/high)
caps["structured_outputs"]["supported"]
caps["context_management"]["compact_20260112"]["supported"]

# filter across all models - iterate the page object directly (auto-paginates); do NOT use .data
[m for m in client.models.list()
 if m.capabilities["thinking"]["types"]["adaptive"]["supported"]
 and m.max_input_tokens >= 200_000]
```

頂層欄位（`id`、`display_name`、`max_input_tokens`、`max_tokens`）是具型別的屬性。`capabilities` 是字典——使用括號存取，而非屬性存取。API 會為每個模型回傳完整功能樹，在每個葉節點帶有 `supported: true/false`，因此括號鏈不需要 `.get()` 防護。TypeScript SDK：相同的方法名稱，迭代時也自動分頁。

### 原始 HTTP

```bash
curl https://api.anthropic.com/v1/models/claude-opus-4-8 \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01"
```

```json
{
  "id": "claude-opus-4-8",
  "display_name": "Claude Opus 4.8",
  "max_input_tokens": 1000000,
  "max_tokens": 128000,
  "capabilities": {
    "image_input": {"supported": true},
    "structured_outputs": {"supported": true},
    "thinking": {"supported": true, "types": {"enabled": {"supported": false}, "adaptive": {"supported": true}}},
    "effort": {"supported": true, "low": {"supported": true}, ..., "max": {"supported": true}},
    ...
  }
}
```

## 目前模型（推薦）

| 友善名稱 | 別名（請使用此） | 完整 ID | 情境視窗 | 最大輸出 | 狀態 |
|---|---|---|---|---|---|
| Claude Fable 5.1 | `claude-fable-5-1` | — | 1M | 128K | 啟用中 |
| Claude Mythos 5.1 | `claude-mythos-5-1` | — | 1M | 128K | 啟用中（僅限 Project Glasswing） |
| Claude Fable 5 | `claude-fable-5` | — | 1M | 128K | 啟用中 |
| Claude Mythos 5 | `claude-mythos-5` | — | 1M | 128K | 啟用中（僅限 Project Glasswing） |
| Claude Opus 5 | `claude-opus-5` | — | 1M | 128K | 啟用中 |
| Claude Opus 4.8 | `claude-opus-4-8` | — | 1M | 128K | 啟用中 |
| Claude Opus 4.7 | `claude-opus-4-7` | — | 1M | 128K | 啟用中 |
| Claude Opus 4.6 | `claude-opus-4-6` | — | 1M | 128K | 啟用中 |
| Claude Sonnet 5 | `claude-sonnet-5` | — | 1M | 128K | 啟用中 |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | — | 1M | 128K | 啟用中 |
| Claude Haiku 4.5 | `claude-haiku-4-5` | `claude-haiku-4-5-20251001` | 200K | 64K | 啟用中 |

### 模型說明
- **Claude Fable 5.1** — Anthropic 迄今公開發布的最強大模型，適用於最嚴苛的推理和長期 agent 工作。是 Claude Fable 5 的後繼者，屬於同一層級、每 token 定價相同（每 MTok \$10/\$50；快取讀取每 MTok \$0.25——為 0.025 倍，即 Claude Fable 5 的四分之一；batch \$5/\$25）；在長時間執行的 agent 程式設計、涉及文件/試算表/簡報的知識工作、多步驟研究、視覺、長情境檢索和電腦使用方面更強。API 介面與 Claude Fable 5 相同（thinking 永遠啟用、不支援 prefill、無取樣參數、`refusal` 停止原因、512 token 快取最低值），但有三個破壞性變更：強制 tool use（`tool_choice` 為 `any` / `tool`）會回傳 400；thinking 區塊會繫結到產生它的模型（只有 Claude Mythos 5.1 能讀取它們——其他模型會捨棄它們）；以及編輯較早的回合會使 thinking 區塊失效（「preserved thinking」；2026-08-31 當天或之後建立的新帳號，編輯過的歷史紀錄會收到 400；更晚推出的模型會對所有人強制執行此規則；選擇加入的控制項依平台而異——見 `shared/platform-availability.md`）。新增每則訊息 `effort`、回合範圍的 `clear_at` 系統訊息、`thinking.display: "updates"` 進度更新，以及內容來源追溯（content provenance）。與 Claude Fable 5 相同的分詞器；1M 情境（預設），128K 最大輸出。Covered Model：需要 30 天資料保留（僅在 Anthropic 明確授權時才可使用 ZDR）——ZDR 組織會收到 `400 invalid_request_error`，與 Claude Fable 5 相同。無 Priority Tier；與 Fable 5.x 共用頻率限制池。請參見 `shared/model-migration.md` → 從 Claude Fable 5 遷移到 Claude Fable 5.1。
- **Claude Fable 5** / **Claude Mythos 5**（`claude-fable-5` / `claude-mythos-5`） — 前一代的 Fable / Mythos 版本：層級、限制和每 token 定價與 Claude Fable 5.1 相同，Claude Fable 5.1 在此基礎上新增了三個破壞性 API 變更（見上文；此處的快取讀取為每 MTok \$1，而非 Claude Fable 5.1 的 \$0.25）；目前仍提供服務，可依 ID 選用。Claude Mythos 5 未執行任何安全分類器，因此不會出現 `stop_reason: "refusal"`。新工作建議優先使用 claude-fable-5-1。
- **Claude Mythos 5.1** — 與 Claude Fable 5.1 是相同的模型（功能、限制、每 token 定價、API 行為皆相同），僅提供給經核准的 Project Glasswing 客戶；是 Claude Mythos 5 的後繼者（Claude Mythos 5 本身則繼承自邀請制的 `claude-mythos-preview`）。與 Claude Mythos 5 不同，它會執行依存取權限方案而定的安全防護機制，因此請處理 `stop_reason: "refusal"`。AWS 上的 Claude Platform 不提供此模型。僅在組織參與 Project Glasswing 時使用；否則請使用 `claude-fable-5-1`。
- **Claude Opus 5** — 適用於複雜 agent 程式設計和企業工作；比 Claude Opus 4.8 有大幅進步，在深度推理、agent 和長期工作，以及測試時計算擴展方面最強，定價是 Claude Fable 5.1 的一半（Claude Fable 5.1 仍是最高功能層）。安全分類器可能回傳 `stop_reason: "refusal"`——在讀取 `content` 之前請先處理。以 Opus 4.8 的定價（每 MTok \$5/\$25）直接升級，功能集相同。thinking 預設啟用（省略 `thinking` 執行 adaptive；`{type: "adaptive"}` 等效），且 `thinking: {type: "disabled"}` 僅在 effort `high` 或以下可用——與 `xhigh`/`max` 搭配會回傳 400。原始 thinking token 永不回傳。完整 effort 梯從 `max` 開始；512 token prompt 快取最低（低於 Opus 4.8 的 1024）；僅限 Claude API 的快速模式。增強的網路安全保障。獨立於合併的 Opus 4.x 池的頻率限制桶。1M 情境視窗（預設和最大），128K 最大輸出。請參見 `shared/model-migration.md` → 遷移到 Claude Opus 5。
- **Claude Opus 4.8** — Opus 4 系列中最強大的模型——高度自主，在長期 agent 工作、知識工作和記憶方面達到業界頂尖水準；更清晰、更溫暖的寫作風格。API 介面與 Opus 4.7 相同（僅限 adaptive thinking；移除了取樣參數和 `budget_tokens`）。以標準 API 定價提供 1M 情境視窗（無長情境溢價）。請參見 `shared/model-migration.md` → 遷移到 Opus 4.8——4.7 → 4.8 的遷移是模型 ID 更換加 prompt 重新調整，沒有新的破壞性變更。
- **Claude Opus 4.7** — 上一代 Opus。高度自主；在長期 agent 工作、知識工作、視覺和記憶方面表現強勁。僅限 adaptive thinking；移除了取樣參數和 `budget_tokens`。1M 情境視窗。請參見 `shared/model-migration.md` → 遷移到 Opus 4.7。
- **Claude Opus 4.6** — 舊版 Opus。支援 adaptive thinking（推薦），128K 最大輸出 token（大輸出需要串流）。1M 情境視窗。
- **Claude Sonnet 5** — Sonnet 層中速度與智能的最佳結合；在程式設計和 agent 工作方面接近 Opus 品質。Adaptive thinking 預設啟用（省略 `thinking` 執行 adaptive）；移除了手動 `budget_tokens`；拒絕非預設的取樣參數。`effort` 支援 `low`/`medium`/`high`/`xhigh`/`max`。新分詞器（相同文字比 Sonnet 4.6 多約 30% token）。高解析度視覺（2576px）。1M 情境視窗，128K 最大輸出。請參見 `shared/model-migration.md` → 遷移到 Claude Sonnet 5。
- **Claude Sonnet 4.6** — 上一代 Sonnet。支援 adaptive thinking（推薦）。1M 情境視窗。128K 最大輸出 token。
- **Claude Haiku 4.5** — 適用於簡單任務，速度最快且最具成本效益的模型。

## 舊版模型（仍啟用）

| 友善名稱 | 別名（請使用此） | 完整 ID | 狀態 |
|---|---|---|---|
| Claude Opus 4.5 | `claude-opus-4-5` | `claude-opus-4-5-20251101` | 啟用中 |
| Claude Opus 4.1 | `claude-opus-4-1` | `claude-opus-4-1-20250805` | 已棄用（2026-08-05 退役——請遷移到 `claude-opus-5`） |
| Claude Sonnet 4.5 | `claude-sonnet-4-5` | `claude-sonnet-4-5-20250929` | 啟用中 |

## 已棄用模型（即將退役）

| 友善名稱 | 別名（請使用此） | 完整 ID | 狀態 | 退役時間 |
|---|---|---|---|---|
| Claude Sonnet 4 | `claude-sonnet-4-0` | `claude-sonnet-4-20250514` | 已棄用 | 待定 |
| Claude Opus 4 | `claude-opus-4-0` | `claude-opus-4-20250514` | 已棄用 | 待定 |
| Claude Haiku 3 | — | `claude-3-haiku-20240307` | 已棄用 | 2026 年 4 月 19 日 |

## 已退役模型（不再可用）

| 友善名稱 | 完整 ID | 退役時間 |
|---|---|---|
| Claude Sonnet 3.7 | `claude-3-7-sonnet-20250219` | 2026 年 2 月 19 日 |
| Claude Haiku 3.5 | `claude-3-5-haiku-20241022` | 2026 年 2 月 19 日 |
| Claude Opus 3 | `claude-3-opus-20240229` | 2026 年 1 月 5 日 |
| Claude Sonnet 3.5 | `claude-3-5-sonnet-20241022` | 2025 年 10 月 28 日 |
| Claude Sonnet 3.5 | `claude-3-5-sonnet-20240620` | 2025 年 10 月 28 日 |
| Claude Sonnet 3 | `claude-3-sonnet-20240229` | 2025 年 7 月 21 日 |
| Claude 2.1 | `claude-2.1` | 2025 年 7 月 21 日 |
| Claude 2.0 | `claude-2.0` | 2025 年 7 月 21 日 |

## 解析使用者請求

當使用者依名稱要求模型時，請用此表格找到正確的模型 ID：

| 使用者說... | 使用此模型 ID |
|---|---|
| "fable"、"最強大的模型" | `claude-fable-5-1` |
| "最強大" | `claude-fable-5-1` |
| "mythos"、"mythos 5.1" | `claude-mythos-5-1`（僅限 Project Glasswing 參與者；否則使用 `claude-fable-5-1`） |
| "fable 5"、"mythos 5"（前一版本） | `claude-fable-5` / `claude-mythos-5`（仍提供服務；新工作建議優先使用 `claude-fable-5-1`） |
| "mythos preview" | `claude-mythos-5-1`（`claude-mythos-preview` 的繼承者——請參見遷移指南） |
| "opus" | `claude-opus-5` |
| "opus 5" | `claude-opus-5` |
| "opus 4.8" | `claude-opus-4-8` |
| "opus 4.7" | `claude-opus-4-7` |
| "opus 4.6" | `claude-opus-4-6` |
| "opus 4.5" | `claude-opus-4-5` |
| "opus 4.1" | `claude-opus-4-1`（已棄用，2026-08-05 退役——建議 `claude-opus-5`） |
| "opus 4"、"opus 4.0" | `claude-opus-4-0`（已棄用——建議 `claude-opus-5`） |
| "sonnet"、"平衡" | `claude-sonnet-5` |
| "sonnet 5" | `claude-sonnet-5` |
| "sonnet 4.6" | `claude-sonnet-4-6` |
| "sonnet 4.5" | `claude-sonnet-4-5` |
| "sonnet 4"、"sonnet 4.0" | `claude-sonnet-4-0`（已棄用——建議 `claude-sonnet-5`） |
| "sonnet 3.7" | 已退役——建議 `claude-sonnet-5` |
| "sonnet 3.5" | 已退役——建議 `claude-sonnet-5` |
| "haiku"、"快"、"便宜" | `claude-haiku-4-5` |
| "haiku 4.5" | `claude-haiku-4-5` |
| "haiku 3.5" | 已退役——建議 `claude-haiku-4-5` |
| "haiku 3" | 已棄用——建議 `claude-haiku-4-5` |
