---
source_file: platform-availability.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: b11e6ef8162daeb15ce12dff4187f5e28055bb7cef47f2b104f2b806a873d8aa
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `platform-availability.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# 平台可用性

各功能在哪些提供者平台上可用。**此表格是本 skill 的唯一事實來源**——其他地方的個別功能章節會指向此處，而非重複說明可用性。在為第三方平台（Bedrock、Vertex、Foundry）或 AWS 上的 Claude Platform 撰寫程式碼時，請先查閱此表格；若該功能在那裡不受支援，則改用第一方 Claude API 介面或採用不同方式。

欄位說明：**1P** = 第一方 Claude API，**P-AWS** = AWS 上的 Claude Platform（Anthropic 自行運營，具備同日更新的功能對等性），**Bedrock** = Amazon Bedrock，**Vertex** = Google Cloud Vertex AI，**Foundry** = Microsoft Foundry。是 = GA，beta = beta，否 = 不支援。

| 功能 | 1P | P-AWS | Bedrock | Vertex | Foundry | 備註 |
|---|---|---|---|---|---|---|
| Messages、串流、tool use | 是 | 是 | 是 | 是 | 是 | 核心 API |
| PDF 輸入 | 是 | 是 | 是 | 是 | beta | |
| 結構化輸出 / 嚴格 tool use | 是 | 是 | 是 | 是 | beta | |
| 自適應思考 / effort | 是 | 是 | 是 | 是 | beta | |
| 延伸思考 | 是 | 是 | 是 | 是 | beta | |
| Prompt 快取（5分鐘、1小時） | 是 | 是 | 是 | 是 | 是 | |
| 自動 prompt 快取 | 是 | 是 | 是 | 是 | 是 | 舊版 Bedrock 整合（Opus 4.6 及更早版本）對頂層 `cache_control` 會拒絕並回傳 400——該處僅支援明確設定快取斷點 |
| Token 計數 | 是 | 是 | 是 | 是 | beta | |
| Citations | 是 | 是 | 是 | 是 | beta | |
| 搜尋結果內容區塊 | 是 | 是 | 是 | 是 | beta | |
| 細粒度工具串流 | 是 | 是 | 是 | 是 | 是 | |
| 壓縮 | beta | beta | beta | beta | beta | |
| 情境編輯 | beta | beta | beta | beta | beta | |
| 情境視窗（1M） | 是 | 是 | 是 | 是 | beta | |
| `inference_geo`（資料駐留） | 是 | 是 | 否 | 否 | 否 | |
| **伺服器端工具** | | | | | | |
| &nbsp;&nbsp;網路搜尋 | 是 | 是 | 否 | 是 | beta | Vertex：僅基本 `web_search_20250305`（不支援 `_20260209` 動態篩選） |
| &nbsp;&nbsp;網路擷取 | 是 | 是 | 否 | 否 | beta | |
| &nbsp;&nbsp;程式碼執行 | 是 | 是 | 否 | 否 | beta | |
| &nbsp;&nbsp;工具搜尋 | 是 | 是 | 是 | 是 | beta | Bedrock：僅 InvokeModel API，不支援 Converse |
| &nbsp;&nbsp;Advisor 工具 | beta | beta | 否 | 否 | 否 | |
| **客戶端實作工具** | | | | | | |
| &nbsp;&nbsp;Bash、文字編輯器、記憶體 | 是 | 是 | 是 | 是 | beta | |
| &nbsp;&nbsp;電腦使用 | beta | beta | beta | beta | beta | |
| **Agentic / 協作流程** | | | | | | |
| &nbsp;&nbsp;Agent Skills（Messages API） | 是 | 是 | 否 | 否 | beta | |
| &nbsp;&nbsp;程式化工具呼叫 | 是 | 是 | 否 | 否 | beta | |
| &nbsp;&nbsp;MCP 連接器 | beta | beta | 否 | 否 | beta | |
| &nbsp;&nbsp;Managed Agents | beta | beta | 否 | 否 | 否 | Foundry：否（推斷；Foundry 文件亦未提及） |
| &nbsp;&nbsp;自託管沙盒 | beta | beta | 否 | 否 | 否 | P-AWS：worker 以 IAM/SigV4，或搭配 `AnthropicSelfHostedEnvironmentAccess` 的 AWS Console API 金鑰進行驗證（Console 環境金鑰在那裡無效）；自託管環境上的 session 無法附加記憶體存放區；不支援 `GET /v1/environments/{id}/work` 清單端點，其他 work 端點正常 |
| **API 端點** | | | | | | |
| &nbsp;&nbsp;Message Batches | 是 | 是 | 否 | 否 | 否 | |
| &nbsp;&nbsp;Files API | 是 | 是 | 否 | 否 | beta | |
| &nbsp;&nbsp;Models API | 是 | 是 | 否 | 否 | 否 | |
| **其他** | | | | | | |
| &nbsp;&nbsp;對話中途系統訊息 | 是 | 是 | 是 | 是 | 否 | Claude Opus 5、Claude Opus 4.8、Claude Fable 5、Claude Fable 5.1、Claude Mythos 5、Claude Mythos 5.1；不含 Claude Sonnet 5。Bedrock：僅 InvokeModel passthrough，不支援 ARN 版本化模型 |
| &nbsp;&nbsp;回合範圍（`clear_at`）系統訊息 | beta | beta | beta | beta | 否 | 與對話中途系統訊息相同的模型；beta `mid-conversation-system-clear-at-2026-08-21`（在 Bedrock/Vertex 上將此值當作 beta 傳遞） |
| &nbsp;&nbsp;每則訊息 `effort`（系統訊息 `output_config`） | beta | 否 | 否 | 否 | 否 | Claude Fable 5.1、Claude Mythos 5.1、Claude Opus 5；beta `mid-conversation-output-config-2026-07-01`；發布時僅限 Claude API（Bedrock/Vertex/Foundry 未確認；Claude Opus 5 在 Bedrock 上排除在外） |
| &nbsp;&nbsp;`thinking.display: "updates"` | beta | beta | beta | beta | beta | Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5；beta `thinking-display-updates-2026-08-18`（依平台傳遞該 beta 值）；未傳遞時 `"updates"` 會被拒絕，視為未知的 `display` 值 |
| &nbsp;&nbsp;Thinking 區塊繫結控制 | beta | beta | 依模型而定 | 依模型而定 | 否 | `thinking.block_binding` + `input_transformations`；beta `thinking-binding-controls-2026-08-01`（在 Bedrock 上透過 `anthropic_beta` 主體欄位傳遞）；此 beta 在 Bedrock/Vertex 上依模型陸續開放——開放前該 header 會被拒絕；歷史編輯的強制規則本身遵循 `shared/model-migration.md` → 從 Claude Fable 5 遷移到 Claude Fable 5.1 中的帳號建立時間規則 |
| &nbsp;&nbsp;伺服器端 `fallbacks` | beta | beta | 否 | 否 | 否 | `"default"` → beta `server-side-fallback-2026-07-01`；陣列形式 → beta `server-side-fallback-2026-06-01` |
| &nbsp;&nbsp;快速模式 | beta | 否 | 否 | 否 | 否 | 研究預覽版，beta `fast-mode-2026-02-01`，僅限第一方 API |
| &nbsp;&nbsp;快取診斷 | beta | 否 | 否 | 否 | 否 | 僅限第一方 API |
| &nbsp;&nbsp;任務預算 | beta | beta | 否 | 否 | 否 | Beta header `task-budgets-2026-03-13`；第三方平台可用性未記載——假設不支援 |
