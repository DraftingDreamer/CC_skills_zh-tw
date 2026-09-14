---
source_file: managed-agents-api-reference.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: abb04671f430a4542012d8d25bc8b3932704142302a1eeb4c4d7a33d46314595
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `managed-agents-api-reference.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Managed Agents — 端點參考

所有端點都需要 `x-api-key` 和 `anthropic-version: 2023-06-01` headers。Managed Agents 端點額外需要 `anthropic-beta` header。

> 大多數使用者應該將 agent 和環境定義為以 `ant` CLI 套用的版本控制 YAML——見 `shared/anthropic-cli.md`。以下端點是 CLI 和 SDK 驅動的底層 API。

## Beta Headers

```
anthropic-beta: managed-agents-2026-04-01
```

SDK 為所有 `client.beta.{agents,environments,sessions,vaults,memory_stores,deployments,deployment_runs}.*` 呼叫自動新增此 header。Skills 端點使用 `skills-2025-10-02`；Files 端點使用 `files-api-2025-04-14`。

---

## SDK 方法參考

所有資源都在 `beta` 命名空間下。Python 和 TypeScript 共用相同的方法名稱。

| 資源 | Python / TypeScript（`client.beta.*`） | Go（`client.Beta.*`） |
| --- | --- | --- |
| Agents | `agents.create` / `retrieve` / `update` / `list` / `archive` | `Agents.New` / `Get` / `Update` / `List` / `Archive` |
| Agent Versions | `agents.versions.list` | `Agents.Versions.List` |
| Environments | `environments.create` / `retrieve` / `update` / `list` / `delete` / `archive` | `Environments.New` / `Get` / `Update` / `List` / `Delete` / `Archive` |
| Environment Work（自託管） | `environments.work.poller` / `stats` / `stop` | 見 `shared/managed-agents-self-hosted-sandboxes.md` |
| Sessions | `sessions.create` / `retrieve` / `update` / `list` / `delete` / `archive` | `Sessions.New` / `Get` / `Update` / `List` / `Delete` / `Archive` |
| Session Events | `sessions.events.list` / `send` / `stream` | `Sessions.Events.List` / `Send` / `StreamEvents` |
| Session Threads | `sessions.threads.list` / `retrieve` / `archive`；`sessions.threads.events.list` / `stream` | `Sessions.Threads.List` / `Get` / `Archive`；`Sessions.Threads.Events.List` / `StreamEvents` |
| Session Resources | `sessions.resources.add` / `retrieve` / `update` / `list` / `delete` | `Sessions.Resources.Add` / `Get` / `Update` / `List` / `Delete` |
| Deployments | `deployments.create` / `update` / `pause` / `unpause` / `archive` / `run` | 尚未說明文件化——WebFetch SDK 存放庫（`shared/live-sources.md`） |
| Deployment Runs | `deployment_runs.list` / `retrieve`（TS：`deploymentRuns.*`） | 尚未說明文件化——WebFetch SDK 存放庫（`shared/live-sources.md`） |
| Vaults | `vaults.create` / `retrieve` / `update` / `list` / `delete` / `archive` | `Vaults.New` / `Get` / `Update` / `List` / `Delete` / `Archive` |
| Credentials | `vaults.credentials.create` / `retrieve` / `update` / `list` / `delete` / `archive` / `mcp_oauth_validate` | `Vaults.Credentials.New` / `Get` / `Update` / `List` / `Delete` / `Archive` / `McpOauthValidate` |
| Memory Stores | `memory_stores.create` / `retrieve` / `update` / `list` / `delete` / `archive` | `MemoryStores.New` / `Get` / `Update` / `List` / `Delete` / `Archive` |
| Memories | `memory_stores.memories.create` / `retrieve` / `update` / `list` / `delete` | `MemoryStores.Memories.New` / `Get` / `Update` / `List` / `Delete` |
| Memory Versions | `memory_stores.memory_versions.list` / `retrieve` / `redact` | `MemoryStores.MemoryVersions.List` / `Get` / `Redact` |

**需要注意的命名差異：**
- Agents 和 Session Threads **沒有 delete**——只有 `archive`。封存是**永久的**：agent 變成唯讀，新 session 無法引用它，且沒有取消封存。在封存生產環境 agent 之前請與使用者確認。Environments、Sessions、Vaults、Credentials 和 Memory Stores 同時有 `delete` 和 `archive`；Session Resources、Files、Skills 和 Memories 只有 `delete`；Memory Versions 兩者都沒有——只有 `redact`。
- Session resources 使用 `add`（而非 `create`）。
- Go 的事件串流是 `StreamEvents`（而非 `Stream`）。
- 自託管 worker 類別是來自 `anthropic.lib.environments` / `@anthropic-ai/sdk/helpers/beta/environments` / `anthropic-sdk-go/lib/environments` 的 `EnvironmentWorker`；`client.beta.environments.work.worker(...)` 是回傳同一個類別的工廠方法，與 `environments.work.poller/stats/stop` 等客戶端方法並存。

**Agent 簡寫：** session 建立時的 `agent` 接受三種形式——裸字串（`agent="agent_abc123"`，最新版本）、固定參考 `{type: "agent", id, version}`，或 `{type: "agent_with_overrides", id, version?, model?, system?, tools?, mcp_servers?, skills?}` 以僅為此 session 覆寫那些欄位（見 `shared/managed-agents-core.md` → 為 session 覆寫 agent 設定）。

**Model 簡寫：** agent 建立時的 `model` 接受裸字串（`model="claude-opus-5"`——使用 `standard` 速度）或完整設定物件，它在 `id` 旁邊帶有 `speed`、`effort` 和 `inference_geo`：`{id: "claude-opus-5", speed: "fast"}`、`{id: "claude-opus-5", effort: "high"}`、`{id: "claude-opus-5", inference_geo: "us"}`。`effort` 接受層級字串（`low`/`medium`/`high`/`xhigh`/`max`）或 `{type: "<level>"}`，且**僅是 agent 設定**——每個 session 的 `model` 覆寫中的 `effort` 被忽略。`inference_geo`（`"us"` | `"global"`）固定服務 agent 模型請求的地理位置，且與 `effort` 不同，在每個 session 的 `model` 覆寫中**會**套用。見 `shared/managed-agents-core.md` → Agent 模型的 effort / 固定推理地理位置。注意：`speed: "fast"` 在 Claude Opus 5 和 Opus 4.8 上支援——僅限 Claude API，包括 Managed Agents，但不包括 Amazon Bedrock、Google Cloud 或 Microsoft Foundry。Opus 4.7 快速模式已移除；Opus 4.7 上的 `speed: "fast"` 回傳錯誤。

---

## Agents

**每個流程的第一步。** Session 需要預先建立的 agent——在 `managed-agents-2026-04-01` 下沒有行內 agent 設定。

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `GET` | `/v1/agents` | ListAgents | 列出 agent |
| `POST` | `/v1/agents` | CreateAgent | 建立已儲存的 agent 設定 |
| `GET` | `/v1/agents/{agent_id}` | GetAgent | 取得 agent 詳細資訊 |
| `POST` | `/v1/agents/{agent_id}` | UpdateAgent | 更新 agent 設定。`version` 是**選用的**：提供它（≥ 1）以進行樂觀並行控制——不符回傳 409——或省略它以進行無條件的最後寫入獲勝更新。 |
| `POST` | `/v1/agents/{agent_id}/archive` | ArchiveAgent | 封存 agent。使其**唯讀**；現有 session 繼續，新 session 無法引用它。沒有取消封存——這是終端狀態。 |
| `GET` | `/v1/agents/{agent_id}/versions` | ListAgentVersions | 列出 agent 版本 |

## Sessions

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `GET` | `/v1/sessions` | ListSessions | 列出 session（分頁） |
| `POST` | `/v1/sessions` | CreateSession | 建立新 session |
| `GET` | `/v1/sessions/{session_id}` | GetSession | 取得 session 詳細資訊 |
| `POST` | `/v1/sessions/{session_id}` | UpdateSession | 更新 session `metadata`/`title`、`agent.tools`/`agent.mcp_servers`（session 本地覆寫；session 必須為 `idle`），或 `budget`——更改上限（更高或更低；新值必須超過消耗的列表成本）或用 `null` 移除；移除是單向的，且預算建立後永不能新增。`vault_ids` 是僅限建立時的（在更新時被拒絕）。見 `shared/managed-agents-core.md` → 在 session 進行中更新 agent 設定 / Session 預算。 |
| `DELETE` | `/v1/sessions/{session_id}` | DeleteSession | 刪除 session |
| `POST` | `/v1/sessions/{session_id}/archive` | ArchiveSession | 封存 session |

## Events

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `GET` | `/v1/sessions/{session_id}/events` | ListEvents | 列出事件（輪詢，分頁） |
| `POST` | `/v1/sessions/{session_id}/events` | SendEvents | 送出事件（使用者訊息、工具結果） |
| `GET` | `/v1/sessions/{session_id}/events/stream` | StreamEvents | 透過 SSE 串流事件。選用的 `event_deltas[]=agent.message` / `agent.thinking` 選擇加入即時預覽的 `event_start`/`event_delta` 事件——見 `shared/managed-agents-events.md` § 即時預覽。 |

## Session Threads

多 agent session 中的每個子 agent 事件串流。見 `shared/managed-agents-multiagent.md`。

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `GET` | `/v1/sessions/{session_id}/threads` | ListThreads | 列出執行緒（分頁） |
| `GET` | `/v1/sessions/{session_id}/threads/{thread_id}` | GetThread | 取得一個執行緒（帶有 `agent` 快照、`status`、`parent_thread_id`、`stats`、`usage`） |
| `POST` | `/v1/sessions/{session_id}/threads/{thread_id}/archive` | ArchiveThread | 封存執行緒 |
| `GET` | `/v1/sessions/{session_id}/threads/{thread_id}/events` | ListThreadEvents | 列出一個執行緒的過去事件（分頁） |
| `GET` | `/v1/sessions/{session_id}/threads/{thread_id}/stream` | StreamThreadEvents | 透過 SSE 串流一個執行緒（SDK：`threads.events.stream`） |

## Session Resources

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `GET` | `/v1/sessions/{session_id}/resources` | ListResources | 列出附加到 session 的資源 |
| `POST` | `/v1/sessions/{session_id}/resources` | AddResource | 附加 `file` 或 `github_repository` 資源（SDK 方法：`add`，而非 `create`）。`memory_store` 資源只在 session 建立時附加。自託管環境**只**接受 `memory_store`（在建立時），`file` / `github_repository` 在那裡會被拒絕。 |
| `GET` | `/v1/sessions/{session_id}/resources/{resource_id}` | GetResource | 取得單一資源 |
| `POST` | `/v1/sessions/{session_id}/resources/{resource_id}` | UpdateResource | 更新資源 |
| `DELETE` | `/v1/sessions/{session_id}/resources/{resource_id}` | DeleteResource | 從 session 移除資源 |

## Environments

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `POST` | `/v1/environments` | CreateEnvironment | 建立環境 |
| `GET` | `/v1/environments` | ListEnvironments | 列出環境 |
| `GET` | `/v1/environments/{environment_id}` | GetEnvironment | 取得環境詳細資訊 |
| `POST` | `/v1/environments/{environment_id}` | UpdateEnvironment | 更新環境 |
| `DELETE` | `/v1/environments/{environment_id}` | DeleteEnvironment | 刪除環境。回傳 204。 |
| `POST` | `/v1/environments/{environment_id}/archive` | ArchiveEnvironment | 封存環境。使其**唯讀**；現有 session 繼續，新 session 無法引用它。沒有取消封存——這是終端狀態。 |
| `GET` | `/v1/environments/{environment_id}/work/stats` | WorkQueueStats | 自託管工作佇列深度/待處理/worker 數量。`x-api-key` 驗證。見 `shared/managed-agents-self-hosted-sandboxes.md`。 |
| `POST` | `/v1/environments/{environment_id}/work/{work_id}/stop` | StopWork | 自託管：停止已認領的工作項目。`x-api-key` 驗證。 |

對於 `type: "self_hosted"`，`config` 是裸 `{"type": "self_hosted"}`——`networking` 和 `packages` 不適用。（`networking` 在兩種類型中都不會控管 `web_search` / `web_fetch`——這些是在 agent 工具集中以 `allowed_domains` / `blocked_domains` 逐工具限制的；見 `shared/managed-agents-tools.md`。）

## Deployments

排程部署（`depl_` ID）在重複的 cron 排程上執行 agent——每次觸發建立一個 session。見 `shared/managed-agents-scheduled-deployments.md`，了解概念指南（cron/DST 語義、失敗行為、生命週期）。

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `POST` | `/v1/deployments` | CreateDeployment | 建立排程部署 |
| `POST` | `/v1/deployments/{deployment_id}` | UpdateDeployment | 更新部署設定（見 `shared/managed-agents-scheduled-deployments.md`） |
| `POST` | `/v1/deployments/{deployment_id}/pause` | PauseDeployment | 抑制排程觸發（可逆；仍允許手動執行） |
| `POST` | `/v1/deployments/{deployment_id}/unpause` | UnpauseDeployment | 從下一次發生時恢復（不補跑） |
| `POST` | `/v1/deployments/{deployment_id}/archive` | ArchiveDeployment | **終端**——排程停止，部署變為不可變 |
| `POST` | `/v1/deployments/{deployment_id}/run` | RunDeployment | 立即觸發手動執行（`trigger_context.type: "manual"`）；暫停時仍可執行 |

## Deployment Runs

每次觸發嘗試（排程或手動）都寫入一個 `deployment_run` 記錄（`drun_` ID），帶有已建立的 `session_id` 或 `error.type`（`environment_archived`、`agent_archived`、`vault_not_found`、`session_rate_limited`、`service_unavailable`）。

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `GET` | `/v1/deployment_runs?deployment_id=...` | ListDeploymentRuns | 列出部署的執行（分頁；用 `has_error=true` 篩選失敗） |
| `GET` | `/v1/deployment_runs/{deployment_run_id}` | GetDeploymentRun | 以 ID 取得單一執行（`deployment_run.*` webhook 事件以 `data.id` 帶有此值） |

## Vaults

Vault 儲存 Anthropic 代您管理的憑證——MCP 憑證（帶自動更新的 OAuth 或靜態 bearer token）和在出口替換到對外請求的 `environment_variable` 憑證。透過 `vault_ids` 附加到 session。見 `managed-agents-tools.md` § Vault 了解概念指南和憑證形狀。

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `POST` | `/v1/vaults` | CreateVault | 建立 vault |
| `GET` | `/v1/vaults` | ListVaults | 列出 vault |
| `GET` | `/v1/vaults/{vault_id}` | GetVault | 取得 vault 詳細資訊 |
| `POST` | `/v1/vaults/{vault_id}` | UpdateVault | 更新 vault |
| `DELETE` | `/v1/vaults/{vault_id}` | DeleteVault | 刪除 vault |
| `POST` | `/v1/vaults/{vault_id}/archive` | ArchiveVault | 封存 vault |

## Credentials

憑證是儲存在 vault 內的個別密鑰。

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `POST` | `/v1/vaults/{vault_id}/credentials` | CreateCredential | 建立憑證 |
| `GET` | `/v1/vaults/{vault_id}/credentials` | ListCredentials | 列出 vault 中的憑證 |
| `GET` | `/v1/vaults/{vault_id}/credentials/{credential_id}` | GetCredential | 取得憑證中繼資料 |
| `POST` | `/v1/vaults/{vault_id}/credentials/{credential_id}` | UpdateCredential | 更新憑證 |
| `DELETE` | `/v1/vaults/{vault_id}/credentials/{credential_id}` | DeleteCredential | 刪除憑證 |
| `POST` | `/v1/vaults/{vault_id}/credentials/{credential_id}/archive` | ArchiveCredential | 封存憑證 |
| `POST` | `/v1/vaults/{vault_id}/credentials/{credential_id}/mcp_oauth_validate` | McpOauthValidate | 驗證 MCP OAuth 憑證 |

## Memory Stores

跨 session 持久的工作區範圍記憶體。在 `resources[]` 中透過 `{"type": "memory_store", "memory_store_id": ...}` 條目附加到 session（僅限 session 建立時）。見 `shared/managed-agents-memory.md`，了解概念指南、FUSE 掛載 agent 介面、先決條件和版本控制。

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `POST` | `/v1/memory_stores` | CreateMemoryStore | 建立存放區（`name`、`description`、`metadata`） |
| `GET` | `/v1/memory_stores` | ListMemoryStores | 列出存放區（`include_archived`、`created_at_{gte,lte}`） |
| `GET` | `/v1/memory_stores/{memory_store_id}` | GetMemoryStore | 取得存放區詳細資訊 |
| `POST` | `/v1/memory_stores/{memory_store_id}` | UpdateMemoryStore | 更新存放區 |
| `DELETE` | `/v1/memory_stores/{memory_store_id}` | DeleteMemoryStore | 刪除存放區 |
| `POST` | `/v1/memory_stores/{memory_store_id}/archive` | ArchiveMemoryStore | 封存存放區。使其**唯讀**；現有 session 繼續，新 session 無法引用它。沒有取消封存。 |

## Memories

存放區內的個別文字文件（各 ≤ 100KB）。`create` 在 `path` 建立並在路徑被佔用時回傳 `409`（`memory_path_conflict_error`，帶有 `conflicting_memory_id`）；`update` 以 `mem_...` ID 更改（重新命名和/或內容）。只有 `update` 接受 `precondition`（`{"type": "content_sha256", "content_sha256": ...}`）——不符時回傳 `409`（`memory_precondition_failed_error`）。列表端點接受 `view: "basic"|"full"`（控制是否填充 `content`；`retrieve` 預設為 `full`）。

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `GET` | `/v1/memory_stores/{memory_store_id}/memories` | ListMemories | 回傳 `Memory \| MemoryPrefix`；依 `path_prefix`、`depth` 篩選 |
| `POST` | `/v1/memory_stores/{memory_store_id}/memories` | CreateMemory | 在 `path` 建立（SDK：`memories.create`）；若被佔用則回傳 `409 memory_path_conflict_error` |
| `GET` | `/v1/memory_stores/{memory_store_id}/memories/{memory_id}` | GetMemory | 讀取一個記憶體（預設 `view="full"`） |
| `PATCH` | `/v1/memory_stores/{memory_store_id}/memories/{memory_id}` | UpdateMemory | 以 ID 更改 `content`、`path` 或兩者；選用的 `precondition` |
| `DELETE` | `/v1/memory_stores/{memory_store_id}/memories/{memory_id}` | DeleteMemory | 刪除（選用的 `expected_content_sha256`） |

## Memory Versions

每次更改的不可變快照（`memver_...`）——稽核和回滾介面。`operation` ∈ `created` / `modified` / `deleted`。

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `GET` | `/v1/memory_stores/{memory_store_id}/memory_versions` | ListMemoryVersions | 最新優先；依 `memory_id`、`operation`、`session_id`、`api_key_id`、`created_at_{gte,lte}` 篩選 |
| `GET` | `/v1/memory_stores/{memory_store_id}/memory_versions/{version_id}` | GetMemoryVersion | 列表欄位 + 完整 `content` |
| `POST` | `/v1/memory_stores/{memory_store_id}/memory_versions/{version_id}/redact` | RedactMemoryVersion | 清除 `content`/`content_sha256`/`content_size_bytes`/`path`；保留 actor + 時間戳記 |

## Files

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `POST` | `/v1/files` | UploadFile | 上傳檔案 |
| `GET` | `/v1/files` | ListFiles | 列出檔案 |
| `GET` | `/v1/files/{file_id}` | GetFile | 取得檔案中繼資料（SDK 方法：`retrieve_metadata`） |
| `GET` | `/v1/files/{file_id}/content` | DownloadFile | 下載檔案內容 |
| `DELETE` | `/v1/files/{file_id}` | DeleteFile | 刪除檔案 |

## Skills

| 方法 | 路徑 | 操作 | 說明 |
| --- | --- | --- | --- |
| `POST` | `/v1/skills` | CreateSkill | 建立 skill |
| `GET` | `/v1/skills` | ListSkills | 列出 skill |
| `GET` | `/v1/skills/{skill_id}` | GetSkill | 取得 skill 詳細資訊 |
| `DELETE` | `/v1/skills/{skill_id}` | DeleteSkill | 刪除 skill |
| `POST` | `/v1/skills/{skill_id}/versions` | CreateVersion | 建立 skill 版本 |
| `GET` | `/v1/skills/{skill_id}/versions` | ListVersions | 列出 skill 版本 |
| `GET` | `/v1/skills/{skill_id}/versions/{version}` | GetVersion | 取得 skill 版本 |
| `DELETE` | `/v1/skills/{skill_id}/versions/{version}` | DeleteVersion | 刪除 skill 版本 |

---

## 請求/回應 Schema 快速參考

### CreateAgent 請求主體

**始終從這裡開始。** `model`、`system`、`tools`、`mcp_servers`、`skills` 是此物件上的頂層欄位——它們**不**放在 session 上。

```json
{
  "name": "string (required, 1-256 chars)",
  "model": "claude-opus-5 (required - bare string, or {id, speed?, effort?, inference_geo?} object)",
  "description": "string (optional, up to 2048 chars)",
  "system": "string (optional, up to 100,000 chars)",
  "tools": [
    { "type": "agent_toolset_20260401" }
  ],
  "skills": [
    { "type": "anthropic", "skill_id": "xlsx" },
    { "type": "custom", "skill_id": "skill_abc123", "version": "1" }
  ],
  "mcp_servers": [
    {
      "type": "url",
      "name": "github",
      "url": "https://api.githubcopilot.com/mcp/"
    }
  ],
  "multiagent": {
    "type": "coordinator",
    "agents": [
      "agent_abc123",
      { "type": "agent", "id": "agent_def456", "version": 4 },
      { "type": "self" }
    ]
  },
  "metadata": {
    "key": "value (max 16 pairs, keys <=64 chars, values <=512 chars)"
  }
}
```

> 限制：`tools` 最多 128，`skills` 最多 20，`mcp_servers` 最多 20（唯一名稱）。`multiagent.agents` 1–20 個條目（字串 ID | `{type:"agent",id,version?}` | `{type:"self"}` | `{type:"advisor",model}`，最多一個 advisor）——見 `shared/managed-agents-multiagent.md`。

### CreateSession 請求主體

```json
{
  "agent": "agent_abc123 (required - string shorthand for latest version, or {type: \"agent\", id, version} object)",
  "environment_id": "env_abc123 (required)",
  "title": "string (optional)",
  "resources": [
    {
      "type": "github_repository",
      "url": "https://github.com/owner/repo (required)",
      "authorization_token": "ghp_... (required)",
      "mount_path": "/workspace/repo (optional - defaults to /workspace/<repo-name>)",
      "checkout": { "type": "branch", "name": "main" }
    }
  ],
  "initial_events": [
    { "type": "user.message", "content": [{ "type": "text", "text": "Review the auth module." }] }
  ],
  "vault_ids": ["vlt_abc123 (optional - vault credentials: MCP auth + environment variables)"],
  "budget": {
    "type": "limit",
    "max_list_cost": { "amount": "2500", "currency": "USD" }
  },
  "metadata": {
    "key": "value"
  }
}
```

> `agent` 欄位接受字串 ID、`{type: "agent", id, version}` 或 `{type: "agent_with_overrides", id, version?, ...}` 用於 session 本地覆寫 `model`/`system`/`tools`/`mcp_servers`/`skills`。在覆寫形式外，那些欄位存在於 agent 上，而非這裡。`model` 覆寫中的 `effort` 被忽略——在 agent 上設定它。`model` 覆寫中的 `inference_geo` **會**套用（省略它會清除此 session 的 agent 固定）。
>
> **`budget`**（選用，僅限建立時）是 session 以列表定價消費的硬性上限；`amount` 是小單位（分——`"2500"` = 25.00 美元）的整數字串，僅限 `USD`。之後可以透過 session 更新更改或移除，永不能新增。見 `shared/managed-agents-core.md` → Session 預算。
>
> **`initial_events`**（選用，最多 50 個）在建立時送出事件並在同一個呼叫中啟動 agent 迴圈。只接受 `user.message` 和 `user.define_outcome`——沒有 `system.message`，也沒有任何工具結果類型。驗證是全有或全無。見 `shared/managed-agents-core.md` → 以 `initial_events` 為 session 播種。
>
> **`checkout`** 接受 `{type: "branch", name: "..."}` 或 `{type: "commit", sha: "..."}`。省略以使用存放庫的預設分支。

### CreateEnvironment 請求主體

```json
{
  "name": "string (required)",
  "description": "string (optional)",
  "config": {
    "type": "cloud | self_hosted",
    "networking": {
      "type": "unrestricted | limited (union - see SDK types)"
    },
    "packages": { }
  },
  "metadata": { "key": "value" }
}
```

### CreateDeployment 請求主體

```json
{
  "name": "Weekly compliance scan",
  "agent": "agent_abc123 (required - same shapes as CreateSession)",
  "environment_id": "env_abc123 (required)",
  "initial_events": [
    { "type": "user.message", "content": [{ "type": "text", "text": "Run the weekly compliance scan." }] }
  ],
  "schedule": {
    "type": "cron",
    "expression": "0 20 * * 5",
    "timezone": "America/New_York"
  }
}
```

> 選用的 session 設定（`resources`、`vault_ids` 等）以與 CreateSession 相同的方式支援，包括 `budget`——複製到每個觸發的 session 上；與 session 的不同，它可以在不存在時新增，並在清除後重新新增（見 `shared/managed-agents-scheduled-deployments.md` § 部署預算）。回應包括 `status`、`paused_reason` 和 `schedule.upcoming_runs_at`（下次觸發時間）。見 `shared/managed-agents-scheduled-deployments.md`。

### SendEvents 請求主體

```json
{
  "events": [
    {
      "type": "user.message",
      "content": [
        {
          "type": "text",
          "text": "Hello"
        }
      ]
    }
  ]
}
```

> `system.message` 事件（追加此輪次及之後輪次的系統層級情境）使用帶有 `type: "system.message"` 的相同信封——在 Claude Opus 5、Claude Opus 4.8、Claude Sonnet 5、Claude Fable 5.1 和 Claude Mythos 5.1 上支援，只對 agent 的*主要*模型核查；見 `shared/managed-agents-events.md` § 在 session 中途新增系統情境。

### Define Outcome 事件

```json
{
  "type": "user.define_outcome",
  "description": "Build a DCF model for Costco in .xlsx",
  "rubric": { "type": "file", "file_id": "file_01..." },
  "max_iterations": 5
}
```

> `rubric` 是必填的：`{type: "text", content}` 或 `{type: "file", file_id}`。`max_iterations` 預設 3，最大 20。以 `outcome_id` + `processed_at` 回顯。見 `shared/managed-agents-outcomes.md`。

### 工具結果事件

```json
{
  "type": "user.custom_tool_result",
  "custom_tool_use_id": "sevt_abc123",
  "content": [{ "type": "text", "text": "Result data" }],
  "is_error": false
}
```

---

## 錯誤處理

Managed Agents 端點使用標準的 Anthropic API 錯誤格式。錯誤以 HTTP 狀態碼和包含 `type`、`error` 和 `request_id` 的 JSON 主體回傳：

```json
{
  "type": "error",
  "error": {
    "type": "invalid_request_error",
    "message": "Description of what went wrong"
  },
  "request_id": "req_011CRv1W3XQ8XpFikNYG7RnE"
}
```

向 Anthropic 回報問題時請包含 `request_id`——它讓我們能端到端追蹤請求。內部的 `error.type` 是以下之一：

| 狀態 | 錯誤類型 | 說明 |
|---|---|---|
| 400 | `invalid_request_error` | 請求格式錯誤或缺少必要參數 |
| 401 | `authentication_error` | 無效或缺少 API 金鑰 |
| 403 | `permission_error` | API 金鑰沒有此操作的權限 |
| 404 | `not_found_error` | 請求的資源不存在 |
| 409 | `invalid_request_error` | 請求與資源的當前狀態衝突（例如，向已封存的 session 送出） |
| 413 | `request_too_large` | 請求主體超過允許的最大大小 |
| 429 | `rate_limit_error` | 請求過多——檢查頻率限制 headers 以取得重試時機 |
| 500 | `api_error` | 發生內部伺服器錯誤 |
| 529 | `overloaded_error` | 服務暫時超載——以退避方式重試 |

注意 `409 Conflict` 帶有 `error.type: "invalid_request_error"`（沒有單獨的 `conflict_error` 類型）；同時檢查 HTTP 狀態和 `message` 以區分衝突和其他無效請求。

---

## 分頁

大多數 Managed Agents 列表端點使用 `page` / `next_page` 游標方案：

| 欄位 | 位置 | 備注 |
|---|---|---|
| `limit` | query | 每頁最多項目數 |
| `page` | query | 來自先前回應的不透明游標——在這裡傳入 `next_page` 或 `prev_page` 值 |
| `order` | query | 在支援排序的端點上為 `asc` / `desc`。游標編碼產生它的請求的 `order`——以不同的 `order` 重用它回傳 400。其他參數（篩選器、`limit`）可以在分頁請求之間更改。 |
| `next_page` | response | 下一頁的游標；沒有更多結果時為 `null` |
| `prev_page` | response | 支援向後分頁的端點上前一頁的游標——目前**只有 `GET /v1/sessions`**。第一頁時為 `null`。在不支援的端點上，欄位**不存在**（而非 `null`）。 |

每個 SDK 都公開自動分頁的迭代器，遵循 `next_page`。在 Python 和 TypeScript 中，直接迭代列表結果；其他 SDK 透過單獨的方法公開迭代器（迭代普通列表結果回傳一頁）。SDK 自動分頁是**僅限向前**——若要回到上一頁，請自己從回應讀取 `prev_page` 並將其作為 `page` 參數傳回。

> 警告：某些端點使用**不同的**游標方案：Message Batches、Files、Models 和幾個管理 API 端點採用 `after_id`/`before_id` 並回傳 `has_more`/`first_id`/`last_id`，而非 `page`/`next_page`。某些 `page` 方案端點（例如 `GET /v1/skills`）也在 `next_page` 旁邊回傳 `has_more` 布林值。查看端點的參考頁面以了解其確切的分頁欄位。

---

## 頻率限制

Managed Agents 端點有每個組織的每分鐘請求（RPM）限制，與您的 [Messages API token 限制](https://platform.claude.com/docs/en/api/rate-limits) 分開。Session 內的模型推理仍從您組織的標準 ITPM/OTPM 限制中提取。

| 端點群組 | 範圍 | RPM | 最大並行數 |
|---|---|---|---|
| 建立操作（Agents、Sessions、Vaults） | 組織 | 300 | — |
| 所有其他操作（Agents、Sessions、Vaults） | 組織 | 600 | — |
| 所有操作（Environments） | 組織 | 60 | 5 |

Files 和 Skills 端點使用標準的按層級[頻率限制](https://platform.claude.com/docs/en/api/rate-limits)。

當超過限制時，API 回傳 `429` 帶有 `rate_limit_error`（回應信封見[錯誤處理](#錯誤處理)）和 `retry-after` header，指示重試前等待的秒數。Anthropic SDK 讀取此 header 並自動重試。
