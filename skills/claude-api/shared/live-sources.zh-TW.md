---
source_file: live-sources.md
source_commit: 34040c9c568585f6929bedeaad110ad08f079624
source_sha256: 75aef376b170d5fb62d762f7287bf2bf8d2d7589f26323ad9765a3d9625b1022
translated_at: 2026-09-13
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `live-sources.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# 即時說明文件來源

本文件包含 WebFetch URL，用於從 platform.claude.com 和 Agent SDK 存放庫擷取最新資訊。當使用者需要可能在快取內容最後更新後已變更的最新資料時，請使用這些 URL。

## 何時使用 WebFetch

- 使用者明確要求「最新」或「當前」資訊
- 快取資料看起來不正確
- 使用者詢問快取內容未涵蓋的功能
- 使用者需要特定的 API 詳細資訊或範例

## Claude API 說明文件 URL

### 模型與定價

| 主題 | URL | 擷取提示 |
| --- | --- | --- |
| 模型概覽 | `https://platform.claude.com/docs/en/about-claude/models/overview.md` | 「擷取所有 Claude 模型的當前模型 ID、情境視窗和定價」 |
| 遷移指南 | `https://platform.claude.com/docs/en/about-claude/models/migration-guide.md` | 「擷取遷移到較新 Claude 模型時的破壞性變更、已棄用的參數和逐模型遷移步驟」 |
| 介紹 Claude Fable 5 | `https://platform.claude.com/docs/en/about-claude/models/introducing-claude-fable-5.md` | 「擷取 Claude Fable 5 和 Claude Mythos 5 的功能、API 變更和可用性階段」 |
| 定價 | `https://platform.claude.com/docs/en/pricing.md` | 「擷取每百萬 token 的輸入和輸出當前定價」 |
| 成本最佳化 | `https://platform.claude.com/docs/en/about-claude/models/optimizing-for-cost-and-intelligence.md` | 「擷取已量化的成本槓桿、快取與批次節省、依 effort 與模型的每項工作成本比較、預算控管，以及多模型使用指引」 |

### 核心功能

| 主題 | URL | 擷取提示 |
| --- | --- | --- |
| Extended Thinking | `https://platform.claude.com/docs/en/build-with-claude/extended-thinking.md` | 「擷取 extended thinking 參數、budget_tokens 需求和使用範例」 |
| Adaptive Thinking | `https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking.md` | 「擷取 adaptive thinking 設定、effort 層級和 Claude Opus 5 使用範例」 |
| Effort 參數 | `https://platform.claude.com/docs/en/build-with-claude/effort.md` | 「擷取 effort 層級、成本品質取捨和與 thinking 的互動」 |
| Tool Use | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview.md` | 「擷取工具定義綱要、tool_choice 選項和處理工具結果」 |
| 串流 | `https://platform.claude.com/docs/en/build-with-claude/streaming.md` | 「擷取串流事件類型、SDK 範例和最佳實務」 |
| Prompt Caching | `https://platform.claude.com/docs/en/build-with-claude/prompt-caching.md` | 「擷取 cache_control 用法、定價優勢和實作範例」 |

### 媒體與檔案

| 主題 | URL | 擷取提示 |
| --- | --- | --- |
| 視覺 | `https://platform.claude.com/docs/en/build-with-claude/vision.md` | 「擷取支援的圖片格式、大小限制和程式碼範例」 |
| PDF 支援 | `https://platform.claude.com/docs/en/build-with-claude/pdf-support.md` | 「擷取 PDF 處理功能、限制和範例」 |

### API 操作

| 主題 | URL | 擷取提示 |
| --- | --- | --- |
| 批次處理 | `https://platform.claude.com/docs/en/build-with-claude/batch-processing.md` | 「擷取批次 API 端點、請求格式和輪詢結果」 |
| Files API | `https://platform.claude.com/docs/en/build-with-claude/files.md` | 「擷取檔案上傳、下載、在訊息中引用、支援的類型，以及從 files-api-2025-04-14 遷移的步驟」 |
| Token 計數 | `https://platform.claude.com/docs/en/build-with-claude/token-counting.md` | 「擷取 token 計數 API 用法和範例」 |
| 頻率限制 | `https://platform.claude.com/docs/en/api/rate-limits.md` | 「擷取依層級和模型的當前頻率限制」 |
| 使用量與成本 Admin API | `https://platform.claude.com/docs/en/manage-claude/usage-cost-api.md` | 「擷取 usage_report 與 cost_report 端點、Admin API 金鑰需求、filter 與 group_by 維度、token 欄位，以及粒度（granularity）限制」 |
| 錯誤 | `https://platform.claude.com/docs/en/api/errors.md` | 「擷取 HTTP 錯誤碼、含義和重試指南」 |
| Amazon Bedrock | `https://platform.claude.com/docs/en/build-with-claude/claude-on-amazon-bedrock.md` | 「擷取每種語言的 AnthropicBedrockMantle 客戶端、`anthropic.` 前綴的模型 ID、驗證路徑、功能可用性和地區」 |
| Claude Platform on AWS | `https://platform.claude.com/docs/en/build-with-claude/claude-platform-on-aws.md` | 「擷取每種語言的 AnthropicAWS 客戶端、SigV4 驗證、憑證優先順序、短期 API 金鑰、workspace_id 和地區需求」 |
| Claude Platform on AWS — IAM 動作 | `https://platform.claude.com/docs/en/api/claude-platform-on-aws-iam-actions.md` | 「擷取每個 API 功能所需的 IAM 動作名稱、資源 ARN 和政策範例」 |

### Admin API（組織管理）

| 主題 | URL | 擷取提示 |
| --- | --- | --- |
| Admin API 指南 | `https://platform.claude.com/docs/en/manage-claude/admin-api.md` | 「擷取 Admin API 驗證方式、SDK/CLI 用法，以及成員/邀請/金鑰管理」 |
| Admin API 參考 | `https://platform.claude.com/docs/en/api/admin.md` | 「擷取 Admin API 的端點參數、回應和分頁」 |
| Workspaces | `https://platform.claude.com/docs/en/manage-claude/workspaces.md` | 「擷取透過 API 進行的 workspace 建立/列出/封存，以及成員管理」 |
| 頻率限制 API | `https://platform.claude.com/docs/en/manage-claude/rate-limits-api.md` | 「擷取組織與 workspace 層級的頻率限制報告端點與篩選器」 |
| WIF Admin | `https://platform.claude.com/docs/en/manage-claude/wif-admin-api.md` | 「擷取服務帳戶（service account）、federation issuer 與 federation 規則管理」 |
| 使用量與成本報告 | `https://platform.claude.com/docs/en/manage-claude/usage-cost-api.md` | 「擷取使用量與成本報告端點（僅支援 curl，SDK 中未提供）」 |

### 工具

| 主題 | URL | 擷取提示 |
| --- | --- | --- |
| 程式碼執行 | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/code-execution-tool.md` | 「擷取程式碼執行工具設定、檔案上傳、容器重用和回應處理」 |
| Computer Use | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/computer-use.md` | 「擷取 computer use 工具設定、功能和實作範例」 |
| Bash Tool | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/bash-tool.md` | 「擷取 bash tool 綱要、參考實作和安全考量」 |
| Text Editor | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool.md` | 「擷取 text editor 工具命令、綱要和參考實作」 |
| Memory Tool | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool.md` | 「擷取 memory tool 命令、目錄結構和實作模式」 |
| Tool Search | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool.md` | 「擷取 tool search 設定、使用時機和快取互動」 |
| 程式化工具呼叫 | `https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling.md` | 「擷取 PTC 設定、指令碼執行模型和從程式碼呼叫工具」 |
| Skills | `https://platform.claude.com/docs/en/agents-and-tools/skills.md` | 「擷取 skill 資料夾結構、SKILL.md 格式和載入行為」 |
| Skills 指南 | `https://platform.claude.com/docs/en/build-with-claude/skills-guide.md` | 「擷取 Skills API（/v1/skills）用法，以及從 skills-2025-10-02 遷移的步驟」 |

### 進階功能

| 主題 | URL | 擷取提示 |
| --- | --- | --- |
| 結構化輸出 | `https://platform.claude.com/docs/en/build-with-claude/structured-outputs.md` | 「擷取 output_config.format 用法和綱要強制執行」 |
| 壓縮 | `https://platform.claude.com/docs/en/build-with-claude/compaction.md` | 「擷取壓縮設定、觸發設定和帶壓縮的串流」 |
| 情境編輯 | `https://platform.claude.com/docs/en/build-with-claude/context-editing.md` | 「擷取情境編輯閾值、清除的內容和設定」 |
| 引用 | `https://platform.claude.com/docs/en/build-with-claude/citations.md` | 「擷取引用格式和實作」 |
| 情境視窗 | `https://platform.claude.com/docs/en/build-with-claude/context-windows.md` | 「擷取情境視窗大小和 token 管理」 |

### Managed Agents

當快取的 `shared/managed-agents-*.md` 概念文件或 `{lang}/managed-agents/README.md` 中未涵蓋 managed agents 繫結、行為或線路層級詳細資訊時，請使用這些 URL。

| 主題 | URL | 擷取提示 |
| --- | --- | --- |
| 概覽 | `https://platform.claude.com/docs/en/managed-agents/overview.md` | 「擷取高層架構以及 agents/sessions/environments/vaults 如何組合在一起」 |
| 快速入門 | `https://platform.claude.com/docs/en/managed-agents/quickstart.md` | 「擷取最小端到端 agent → environment → session → stream 程式碼路徑」 |
| Agent 設定 | `https://platform.claude.com/docs/en/managed-agents/agent-setup.md` | 「擷取 agent create/update/list-versions/archive 生命週期和參數」 |
| 定義 Outcome | `https://platform.claude.com/docs/en/managed-agents/define-outcomes.md` | 「擷取 outcome 定義、評估鉤子和成功標準設定」 |
| Sessions | `https://platform.claude.com/docs/en/managed-agents/sessions.md` | 「擷取 session 生命週期、狀態轉換、idle/terminated 語義和恢復規則」 |
| 環境 | `https://platform.claude.com/docs/en/managed-agents/environments.md` | 「擷取環境設定（cloud/networking）、管理端點和重用模型」 |
| 自託管沙盒 | `https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes.md` | 「擷取 config:{type:self_hosted}、ANTHROPIC_ENVIRONMENT_KEY、EnvironmentWorker.run/handle_item、environments.work.poller(drain)、beta_agent_toolset、ant beta:worker poll/run、webhook 驅動喚醒、記憶體存放區（ANTHROPIC_WORK_SECRET、memory_sync_interval/memory_sync_deletes）」 |
| 自託管沙盒 — 安全性 | `https://platform.claude.com/docs/en/managed-agents/self-hosted-sandboxes-security.md` | 「擷取客戶擁有的內容（強化、出口、金鑰保管、信任邊界）與 Anthropic 無法做的事」 |
| 事件與串流 | `https://platform.claude.com/docs/en/managed-agents/events-and-streaming.md` | 「擷取事件串流類型、串流優先排序、重連/去重和引導模式」 |
| 工具 | `https://platform.claude.com/docs/en/managed-agents/tools.md` | 「擷取內建工具集、自訂工具定義和工具結果線路格式」 |
| 檔案 | `https://platform.claude.com/docs/en/managed-agents/files.md` | 「擷取檔案上傳、掛載路徑、session 資源和列出/下載 session 輸出」 |
| 權限政策 | `https://platform.claude.com/docs/en/managed-agents/permission-policies.md` | 「擷取權限政策類型（`always_allow` / `always_ask` / `auto`）、`auto` 的三種結果、`evaluated_permission` + `evaluation` 事件欄位和按工具設定」 |
| 多 Agent | `https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration.md` | 「擷取多 agent 組合模式、子 agent 呼叫和結果交接」 |
| 可觀測性 | `https://platform.claude.com/docs/en/managed-agents/observability.md` | 「擷取 managed agents 公開的日誌、追蹤和使用遙測」 |
| Webhook | `https://platform.claude.com/docs/en/managed-agents/webhooks.md` | 「擷取 webhook 端點登錄、HMAC 簽名驗證、支援的事件類型和傳送語義」 |
| GitHub | `https://platform.claude.com/docs/en/managed-agents/github.md` | 「擷取 github_repository 資源形狀、多存放庫掛載和 token 輪換」 |
| MCP Connector | `https://platform.claude.com/docs/en/managed-agents/mcp-connector.md` | 「擷取 agents 上的 MCP 伺服器宣告和 session 時的 vault 憑證注入」 |
| Vault | `https://platform.claude.com/docs/en/managed-agents/vaults.md` | 「擷取 vault 建立、憑證新增/輪換、OAuth 更新形狀和封存」 |
| Skills | `https://platform.claude.com/docs/en/managed-agents/skills.md` | 「擷取 managed agents 的 skill 封裝和載入模型」 |
| 記憶體 | `https://platform.claude.com/docs/en/managed-agents/memory.md` | 「擷取記憶體資源形狀、範圍和生命週期」 |
| 入門 | `https://platform.claude.com/docs/en/managed-agents/onboarding.md` | 「擷取首次執行設定、先決條件和帳戶/地區需求」 |
| 雲端容器 | `https://platform.claude.com/docs/en/managed-agents/cloud-containers.md` | 「擷取雲端容器執行時、映像設定和網路/儲存旋鈕」 |
| 遷移 | `https://platform.claude.com/docs/en/managed-agents/migration.md` | 「擷取從較早 API/預覽形狀到 GA managed agents 的遷移路徑」 |

### Anthropic CLI

`ant` CLI 提供對 Claude API 的終端機存取。每個 API 資源都公開為子命令。它是從版本控制的 YAML 建立 agent 和環境的推薦方式（`ant beta:agents create < agent.yaml`——請參見 `shared/anthropic-cli.md`），也為指令碼和互動式檢查公開 sessions 和其他所有 API 資源。

| 主題 | URL | 擷取提示 |
| --- | --- | --- |
| Anthropic CLI | `https://platform.claude.com/docs/en/api/sdks/cli.md` | 「擷取 CLI 安裝、驗證、命令結構和 beta:agents/environments/sessions 命令」 |
| `ant beta:sessions connect` | `https://platform.claude.com/docs/en/cli-sdks-libraries/cli/sessions-connect.md` | 「擷取互動式 session 檢視器：快捷鍵、工具呼叫允許/拒絕提示、`--web` 本機檢視器及其 URL/存活規則」 |
| 驗證概覽 | `https://platform.claude.com/docs/en/manage-claude/authentication.md` | 「擷取憑證選項（API 金鑰、互動式 OAuth 登入、Workload Identity Federation）和各自的使用時機」 |
| WIF 參考 | `https://platform.claude.com/docs/en/manage-claude/wif-reference.md` | 「擷取憑證優先順序、profile 設定檔綱要和設定目錄結構」 |

---

## Claude API SDK 存放庫

當語言特定的繫結（類別、方法、命名空間、欄位）未涵蓋在快取的 `{lang}/` skill 文件或上方的 managed agents 說明文件中時，請 WebFetch 這些。SDK 包含對 `/v1/agents`、`/v1/sessions`、`/v1/environments` 和相關資源的 beta managed agents 支援——在存放庫中搜尋 `BetaManagedAgents`、`beta.agents`、`beta.sessions` 或該語言的等效命名空間。

| SDK | URL | 擷取提示 |
| --- | --- | --- |
| Python | `https://github.com/anthropics/anthropic-sdk-python` | 「擷取 beta managed-agents 命名空間、類別和方法簽名（`client.beta.agents`、`client.beta.sessions`）」 |
| TypeScript | `https://github.com/anthropics/anthropic-sdk-typescript` | 「擷取 beta managed-agents 命名空間、類別和方法簽名（`client.beta.agents`、`client.beta.sessions`）」 |
| Java | `https://github.com/anthropics/anthropic-sdk-java` | 「擷取 beta managed-agents 類別、建構器和方法簽名（`client.beta().agents()`、`BetaManagedAgents*`）」 |
| Go | `https://github.com/anthropics/anthropic-sdk-go` | 「擷取 beta managed-agents 類型和方法簽名（`client.Beta.Agents`、`BetaManagedAgents*` 事件類型）」 |
| Ruby | `https://github.com/anthropics/anthropic-sdk-ruby` | 「擷取 beta managed-agents 方法和參數形狀（`client.beta.agents`、`client.beta.sessions`）」 |
| C# | `https://github.com/anthropics/anthropic-sdk-csharp` | 「擷取 beta managed-agents 類別和方法簽名（NuGet 套件、`BetaManagedAgents*` 類型）」 |
| PHP | `https://github.com/anthropics/anthropic-sdk-php` | 「擷取 beta managed-agents 類別和方法簽名（`$client->beta->agents`、`BetaManagedAgents*` 參數）」 |

每個 SDK 存放庫也在 `examples/` 下附有可執行的程式——包括 refusal-fallback / `fallbacks` 範例（客戶端中介軟體登錄、fallback 狀態、伺服器端 `fallbacks` 參數）。擷取這些以取得精確的語言特定語法，而非翻譯另一種語言的範例。

### SDK 主版本升級指南

跨主版本升級 SDK 套件本身的權威異動清單。內建的 `{lang}/claude-api/sdk-upgrade.md` 是可執行版本；若兩者不一致，以存放庫指南為準。

| SDK | URL | 擷取提示 |
| --- | --- | --- |
| Python（0.x → 1.x） | `https://github.com/anthropics/anthropic-sdk-python/blob/main/MIGRATION.md` | 「擷取每項破壞性變更及其前後程式碼、新的最低 Python 版本需求，以及升級指令」 |

---

## 備援策略

若 WebFetch 失敗（網路問題、URL 已變更）：

1. 使用語言特定文件中的快取內容（注意快取日期）
2. 告知使用者資料可能已過時
3. 建議他們直接查看 platform.claude.com 或 GitHub 存放庫
