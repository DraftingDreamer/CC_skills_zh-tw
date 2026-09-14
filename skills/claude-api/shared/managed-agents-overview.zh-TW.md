---
source_file: managed-agents-overview.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: a250fd240dbddc63c5821612051f4a40fbc7fd40dbb6091299b708d0c594f89b
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `managed-agents-overview.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Managed Agents — 概覽

Managed Agents 為每個 session 配置一個容器作為 agent 的工作區。Agent 迴圈在 Anthropic 的協作層執行；容器是 agent 的*工具*執行的地方——bash 指令、檔案操作、程式碼。您建立持久的 **Agent** 設定（模型、系統 prompt、工具、MCP 伺服器、skills），然後啟動引用它的 **Session**。Session 將事件串流回傳給您；您送入使用者訊息和工具結果。

## 警告：強制流程：Agent（一次）→ Session（每次執行）

**Agent 為獨立物件的原因：版本控制。** Agent 是持久的、有版本的設定——每次更新都建立新的不可變版本，session 在建立時固定到某個版本。這讓您可以迭代 agent（調整 prompt、新增工具）而不影響正在執行的 session，在變更退步時回滾，以及並行 A/B 測試版本。若每次執行都 `agents.create()`，這些都無法做到。

每個 session 都引用一個預先建立的 `/v1/agents` 物件。建立一次 agent，儲存 ID，並在多次執行中重複使用。

| 步驟 | 呼叫 | 頻率 |
|---|---|---|
| 1 | `POST /v1/agents` — `model`、`system`、`tools`、`mcp_servers`、`skills` 放在這裡 | **一次。** 儲存 `agent.id` **和** `agent.version`。 |
| 2 | `POST /v1/sessions` — `agent: "agent_abc123"` 或 `{type: "agent", id, version}` | **每次執行。** 字串簡寫使用最新版本。 |

如果您要在 session 主體上寫 `sessions.create()` 並帶有 `model`、`system` 或 `tools` ——**停下**。那些欄位放在 `agents.create()` 上。Session 只接受*指標*。

**產生程式碼時，請將設置與執行時分離。** `agents.create()` 屬於設置指令碼（或受防護的 `if agent_id is None:` 區塊），而非熱路徑的頂部。若使用者的程式碼每次呼叫都呼叫 `agents.create()`，他們在積累孤立的 agent 並白白付出建立延遲。正確的形式是：將 agent 定義為版本控制的 YAML 清單，用 `ant beta:agents create < agent.yaml`（或受防護的設置指令碼——請參見 `shared/anthropic-cli.md`）一次套用，持久化回傳的 ID（設定檔、環境變數、密鑰管理器），並讓每次執行載入 ID 並呼叫 `sessions.create()`。

**若要更改 agent 的行為，請使用 `POST /v1/agents/{id}`——不要建立新的。** 每次更新都會升版；執行中的 session 保留其固定版本，新的 session 取得最新版本（或透過 `{type: "agent", id, version}` 明確固定）。請參見 `shared/managed-agents-core.md` → Agents → 版本控制。若要在**一個執行中的 session** 上更改 `tools`/`mcp_servers` 而不觸及 agent 物件，請使用 `sessions.update()`（`vault_ids` 僅在 session 建立時附加）——請參見 `shared/managed-agents-core.md` → 在 session 進行中更新 agent 設定。

## Beta Header

Managed Agents 處於測試版。SDK 自動設定所需的 beta header：

| Beta Header | 啟用的功能 |
| --- | --- |
| `managed-agents-2026-04-01` | Agents、Environments、Sessions、Events、Session Resources、Session Threads、Outcomes、Multiagent、Vaults、Credentials、Memory Stores、Deployments |
| `skills-2025-10-02` | Skills API（用於管理自訂 skill 定義） |
| `files-api-2025-04-14` | Files API 的檔案上傳功能 |

**各 beta header 的適用範圍：** SDK 在 `client.beta.{agents,environments,sessions,vaults,memory_stores,deployments,deployment_runs}.*` 呼叫上自動設定 `managed-agents-2026-04-01`，在 `client.beta.files.*` / `client.beta.skills.*` 呼叫上自動設定 `files-api-2025-04-14` / `skills-2025-10-02`。呼叫 Managed Agents 端點時，您**不**需要新增 Skills 或 Files beta header。在原始 HTTP 上，Managed Agents header **單獨就能授予 Files API 存取**，因此上傳檔案用作 session 資源時，不需要在旁邊加 `files-api-2025-04-14`。（直接透過 cURL 的 Skills API 呼叫仍需要 `skills-2025-10-02`；`ant` CLI 和 SDK 會為您送出。）**例外——session 範圍的檔案列出：** `client.beta.files.list({scope_id: session.id})` 是一個接受 Managed Agents 參數的 Files 端點，因此需要**兩個** header。在該呼叫上明確傳入 `betas: ["managed-agents-2026-04-01"]`（SDK 新增 Files header；您新增 Managed Agents header）。請參見 `shared/managed-agents-environments.md` → Session 輸出。

## 閱讀指南

| 使用者想要... | 閱讀這些文件 |
| --- | --- |
| **從零開始 / 「幫我設定 agent」** | `shared/managed-agents-onboarding.md` — 引導式訪談（WHERE→WHO→WHAT→WATCH），然後輸出程式碼 |
| 了解 API 如何運作 | `shared/managed-agents-core.md` |
| 查看完整端點參考 | `shared/managed-agents-api-reference.md` |
| **建立 agent**（必要的第一步） | `shared/managed-agents-core.md`（Agents 節）+ 語言檔案 |
| 更新/版本化 agent | `shared/managed-agents-core.md`（Agents → 版本控制）——更新，不要重新建立 |
| 建立 session | `shared/managed-agents-core.md` + `{lang}/managed-agents/README.md`（cURL/C#：`curl/managed-agents.md`） |
| 設定工具和權限 | `shared/managed-agents-tools.md` |
| 限制 `web_search` / `web_fetch` 能連到哪些網站；在地化搜尋；限制擷取內容長度 | `shared/managed-agents-tools.md`（§ 網路搜尋與網路擷取設定）——工具集 `configs` 項目上的 `allowed_domains` / `blocked_domains` / `user_location` / `max_content_tokens`；**非**環境的 `networking` |
| 設定 MCP 伺服器 | `shared/managed-agents-tools.md`（MCP 伺服器節） |
| 串流事件 / 處理 tool_use | `shared/managed-agents-events.md` + 語言檔案 |
| 透過 webhook 收到 session 狀態變更通知（無需輪詢） | `shared/managed-agents-webhooks.md` — Console 登錄端點、HMAC 驗證、精簡 payload + 擷取 |
| 定義 outcome / 評分標準導向的迭代迴圈 | `shared/managed-agents-outcomes.md` — `user.define_outcome` 事件、評分器、`span.outcome_evaluation_*` 事件 |
| 協調多個 agent / 子 agent / 執行緒 | `shared/managed-agents-multiagent.md` — agent 上的 `multiagent: {type: "coordinator", agents: [...]}`、session 執行緒、跨執行緒工具確認 |
| 設定環境 | `shared/managed-agents-environments.md` + 語言檔案 |
| 在您自己的基礎設施 / VPC 中執行工具（自託管沙盒） | `shared/managed-agents-self-hosted-sandboxes.md` — `config:{type:"self_hosted"}`、`ANTHROPIC_ENVIRONMENT_KEY`、`EnvironmentWorker.run()` / `ant beta:worker poll` |
| 上傳檔案 / 附加存放庫 | `shared/managed-agents-environments.md`（資源） |
| 給 agent 跨 session 的持久記憶體 | `shared/managed-agents-memory.md` — 記憶體存放區、`memory_store` session 資源、前置條件、版本/編輯。自託管沙盒上：`shared/managed-agents-self-hosted-sandboxes.md` § 記憶體存放區（SDK worker 同步本地複本） |
| 不寫程式碼檢視 session（逐字稿、各工具統計、成本、執行緒） | `shared/managed-agents-events.md` — Console session 檢視器說明；深層連結 `?event={event_id}` |
| 將 agent/環境定義為版本控制的 YAML；從 shell 驅動 API | `shared/anthropic-cli.md` — `ant beta:agents create < agent.yaml`、`--transform`、`@file` 內嵌 |
| 儲存憑證（MCP 驗證、CLI/SDK 的 API 金鑰） | `shared/managed-agents-tools.md`（Vault 節）— `mcp_oauth` / `static_bearer` / `environment_variable` |
| 呼叫需要密鑰的非 MCP API / CLI | `shared/managed-agents-tools.md`（Vault 節）— `environment_variable` 憑證，在出口處替換。若不適用（例如自託管沙盒），`shared/managed-agents-client-patterns.md` Pattern 9 透過自訂工具將密鑰保留在主機端 |
| 在重複 cron 排程上執行 agent | `shared/managed-agents-scheduled-deployments.md` — 部署、部署執行記錄、暫停/自動暫停 |
| 以硬性美元預算限制 session 消費 | `shared/managed-agents-core.md`（§ Session 預算）— 在 session 建立時設定 `budget`、`budget_reached` 暫停、變更/移除以恢復。部署：`shared/managed-agents-scheduled-deployments.md` § 部署預算 |
| 固定模型推理執行位置（資料駐留） | `shared/managed-agents-core.md`（§ 固定推理地理位置）— agent 上的 `model.inference_geo`、每個 session 的覆寫、roster 一致性 |
| 從程式碼庫載入 skill 而非上傳 | `shared/managed-agents-tools.md`（§ 來自 GitHub 存放庫的 Skills）— session 啟動時的根目錄 `.claude/skills` 探索 |
| 給 session 一個在執行中可諮詢的 advisor | `shared/managed-agents-multiagent.md`（§ Advisor）— roster 條目 `{type: "advisor", model}`、諮詢執行緒、明文 vs 已編輯的傳送 |

## 常見陷阱

- **先 Agent，再 Session——無例外** — session 的 `agent` 欄位**只**接受字串 ID 或 `{type: "agent", id, version}`。`model`、`system`、`tools`、`mcp_servers`、`skills` 是 **`POST /v1/agents` 的頂層欄位**，從不放在 `sessions.create()` 上。若使用者尚未建立 agent，那是每個範例的第零步。
- **Agent 一次，而非每次執行** — `agents.create()` 是設置步驟。儲存回傳的 `agent_id` 並重複使用；不要在熱路徑頂部呼叫 `agents.create()`。若 agent 的設定需要更改，請 `POST /v1/agents/{id}`——每次更新建立新版本，session 可以固定到特定版本以確保重現性。
- **MCP 驗證透過 vault** — agent 的 `mcp_servers` 陣列只宣告 `{type, name, url}`（沒有驗證）。憑證放在 vault（`client.beta.vaults.credentials.create`）中，並透過 `vault_ids` 附加到 session。Anthropic 使用儲存的更新 token 自動更新 OAuth token。Vault 也儲存非 MCP 服務（CLI、SDK、直接 API 呼叫）的 `environment_variable` 憑證——在出口處替換，在沙盒中永不可見。
- **在第一次執行前核對資源** — 一個有明確請求但缺少工具、憑證、資料掛載或情境的 session 會在執行中途發現缺口，然後掙扎並放棄。在建立 session 之前，確認任務中的每個動作都對應到已設定的工具/MCP 伺服器，每個 MCP 伺服器都有 vault 憑證，每個引用的檔案/主機都已掛載/可達。幫助使用者設定時，請執行 `shared/managed-agents-onboarding.md` → §3 起飛前可行性檢查中的核對流程。
- **串流以獲取事件** — `GET /v1/sessions/{id}/events/stream` 是即時接收 agent 輸出的主要方式。
- **SSE 串流沒有重播——重連時需要整合** — 若在 `agent.tool_use`、`agent.mcp_tool_use` 或 `agent.custom_tool_use` 待解析時串流中斷（前兩者需要 `user.tool_confirmation`，最後者需要 `user.custom_tool_result`），session 會死鎖（客戶端斷線 → session 進入 idle → 重連發生 → 沒有客戶端解析）。在每次（重新）連線時：以 `GET /v1/sessions/{id}/events/stream` 開啟串流，擷取 `GET /v1/sessions/{id}/events`，依事件 ID 去重，然後繼續。請參見 `shared/managed-agents-events.md` → 串流中斷後重連。
- **不要以 HTTP 函式庫逾時作為掛鐘上限** — `requests` 的 `timeout=(c, r)` 和 `httpx.Timeout(n)` 是*每個區塊*的讀取逾時；它們在每個位元組後重設，因此緩慢滴水的連線可以無限期阻塞。對原始 HTTP 輪詢的硬性截止時間，請在迴圈層級追蹤 `time.monotonic()` 並明確中止。優先使用 SDK 的 `sessions.events.stream()` / `sessions.events.list()`，而非手工 HTTP。請參見 `shared/managed-agents-events.md` → 接收事件。
- **訊息佇列** — 您可以在 session 為 `running` 或 `idle` 時送出事件；它們按順序處理。不需要在收到回應後再送下一條訊息。例外：因預算暫停的 session（`stop_reason: budget_reached`）只接受結算事件——變更或移除預算以恢復（`shared/managed-agents-core.md` § Session 預算）。
- **環境 `config.type` 是 `"cloud"` 或 `"self_hosted"`** — `cloud` 在 Anthropic 的基礎設施上執行容器；`self_hosted` 將工具執行移到您自己的（請參見 `shared/managed-agents-self-hosted-sandboxes.md`）。
- **封存在每個資源上都是永久的** — 封存 agent、環境、session、vault、憑證或記憶體存放區使其變為唯讀且無法取消封存。對於 agent、環境和記憶體存放區，封存的資源無法被新 session 引用（現有 session 繼續）。不要對生產 agent、環境或記憶體存放區呼叫 `.archive()` 作為清理——**在封存前務必與使用者確認**。
