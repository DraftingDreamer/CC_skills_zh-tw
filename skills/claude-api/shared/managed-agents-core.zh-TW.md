---
source_file: managed-agents-core.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: f375cdf6ec7e7106540408e0f788279ee84bf304af3d0ef2a13d13470a407e0f
translated_at: 2026-09-05
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`managed-agents-core.md`](managed-agents-core.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
<!-- /translation-header -->

# Managed Agents - 核心概念

## 架構

Managed Agents 建立在四個核心概念之上：

| 概念 | 端點 | 說明 |
|---|---|---|
| **Agent** | `/v1/agents` | 一個持久化、版本化的物件，定義 agent 的能力與人格：模型、系統 prompt、工具、MCP 伺服器、skills。**必須在啟動 session 之前建立。** 見下方 Agents 章節。 |
| **Session** | `/v1/sessions` | 與 agent 的有狀態互動。透過 ID 引用預先建立的 agent + 環境 + 初始指令。產生事件串流。 |
| **Environment** | `/v1/environments` | 定義容器佈建設定的範本。 |
| **Container** | 無 | 一個隔離的運算實例，agent 的**工具**在此執行（bash、檔案操作、程式碼）。Agent 迴圈本身不在這裡執行——它在 Anthropic 的編排層上執行，並透過工具呼叫對容器採取行動。 |

```
                       +-------------------------------------+
                       |  Anthropic orchestration layer      |
Agent (config) ------->|  (agent loop: Claude + tool calls)  |
                       +--------------+----------------------+
                                      | tool calls
                                      v
Environment (template) --> Container (tool execution workspace)
                                 |
                         Session -+
                                 +-- Resources (files, repos, memory stores - attached at startup)
                                 +-- Vault IDs (MCP credential references)
                                 +-- Conversation (event stream in/out)
```

> **Agent 建立是先決條件。** Session 透過 ID 引用預先建立的 agent——`model`/`system`/`tools` 存在於 agent 物件上，絕不在 session 上。每個流程都從 `POST /v1/agents` 開始。

---

## Session 生命週期

```
rescheduling -> running <-> idle -> terminated
```

| 狀態 | 說明 |
| --- | --- |
| `idle` | Agent 已完成當前任務，正在等待輸入。它可能正在等待輸入以透過 `user.message` 繼續工作、因等待 `user.custom_tool_result` 或 `user.tool_confirmation` 而被阻塞，或因達到 session 預算上限而暫停。附帶的 `stop_reason` 包含 agent 停止工作的更多資訊。 |
| `running` | Session 已開始執行，Agent 正在積極工作。 |
| `rescheduling` | Session 在發生可重試錯誤後正在（重新）排程，準備好被編排系統接手。 |
| `terminated` | Session 已結束，處於不可逆、不可用的狀態——**可能是正常完成，也可能是因不可恢復的錯誤**。Terminated 本身並不代表失敗；需要取得 session 才能區分兩者。 |

- 當 session 處於 `running` 或 `idle` 時可以發送事件。訊息會被排隊並按順序處理。例外：在預算達上限時暫停的 session（`stop_reason: budget_reached`）只接受**結算事件**——解決已在進行中的工作的事件（`user.tool_confirmation`、`user.tool_result`、`user.custom_tool_result`、`user.interrupt`），而非開始新工作——見 § Session 預算。
- Agent 在收到新事件時從 `idle -> running` 轉換，完成後回到 `idle`。
- 錯誤以 `session.error` 事件的形式出現在串流中，而非以狀態值的形式。

每個 session 在 Anthropic Console 都有即時追蹤視圖：`https://platform.claude.com/workspaces/{workspace}/sessions/{session_id}`。建立 session 後立即列印此 URL，讓使用者可以即時觀看工具呼叫和訊息串流。**`{workspace}` 是 API 金鑰所屬的工作區**——只有當它是組織的預設工作區時才使用 `default`。Session 回應**不**包含工作區欄位，且 Console 沒有不限工作區的 session 路由，所以對於非預設工作區，請替換為工作區的 ID（在 Console URL 列中可見，或作為 API 金鑰旁的設定值公開）。連結到其他工作區中 session 的 `default` 連結會跳到**「Session not found」**頁面——那裡的 **Search workspaces** 按鈕可以找到它，但這不是自動重新導向。

### 內建 session 功能

- **情境壓縮** - 若接近最大情境，API 會自動壓縮 session 歷史記錄以保持互動繼續
- **Prompt 快取** - 歷史重複的 token 會被快取，減少處理時間和成本
- **擴展思考** - 預設開啟；`agent.thinking` 事件表示思考進度，不攜帶思考內容

### Session 操作

| 操作 | 說明 |
|---|---|
| 列出/取得 | 分頁列表或按 ID 取得單一資源 |
| 更新 | `title`、`metadata`，以及 session 本地的 `agent.tools`/`agent.mcp_servers` 可以被覆蓋（見 § 在 session 中途更新 agent 設定）。`budget` 只能被修改或移除（見 § Session 預算）。`vault_ids` 是僅建立時可設定——更新請求設定它時會被拒絕。 |
| 封存 | Session 變為**唯讀**。不可逆。 |
| 刪除 | 永久刪除 session、事件歷史記錄、容器和檢查點。 |

這些是操作/檢查呼叫——通常從終端機發出，而非應用程式程式碼。從 shell（見 `shared/anthropic-cli.md`）：

```sh
ant beta:sessions list --transform '{id,title,status,created_at}' --format jsonl
ant beta:sessions retrieve --session-id "$SID"
ant beta:sessions:events stream --session-id "$SID"   # watch events live
ant beta:sessions archive  --session-id "$SID"
ant beta:sessions delete   --session-id "$SID"
```

---

## Sessions

Session 是在環境中執行的 agent 實例。

### Session 物件

API 返回的關鍵欄位：

| 欄位 | 型別 | 說明 |
| --- | --- | --- |
| `type` | string | 始終為 `"session"` |
| `id` | string | 唯一 session ID |
| `title` | string | 人類可讀的標題 |
| `status` | string | `idle`、`running`、`rescheduling`、`terminated` |
| `created_at` | string | ISO 8601 時間戳記 |
| `updated_at` | string | ISO 8601 時間戳記 |
| `archived_at` | string | ISO 8601 時間戳記（可為 null） |
| `environment_id` | string | 環境 ID |
| `agent` | object | Agent 設定 |
| `resources` | array | 附加的檔案、存放庫和記憶體儲存 |
| `metadata` | object | 使用者提供的鍵值對（最多 8 個鍵） |
| `usage` | object | 累積使用量：token 計數、`server_tool_use`（網路搜尋/擷取請求次數）、`list_cost`（按公開標準費率計算的消費，以 `{amount, currency}` 表示，amount 是最小單位的整數字串——分）、以及 `active_seconds`（有 >=1 個執行緒執行的時間；並發執行緒重疊計算一次——不同於 `stats.active_seconds`，後者對每個執行緒時間求和） |
| `budget` | object | Session 的支出上限（若建立時有設定）——見 § Session 預算 |
| `stats` | object | 時間統計——`stats.active_seconds` 對每個執行緒時間求和，不同於 `usage.active_seconds` |

### 建立 session

**沒有 agent 的 session 毫無意義。** Session 透過 ID 引用預先建立的 agent。先透過 `agents.create()` 建立 agent，然後引用它：

```ts
// 1. Create the agent (reusable, versioned)
const agent = await client.beta.agents.create(
  {
    name: "Coding Assistant",
    model: "claude-opus-5",
    system: "You are a helpful coding agent.",
    tools: [{ type: "agent_toolset_20260401"}],
  },
);

// 2. Start a session that references it
const session = await client.beta.sessions.create(
  {
    agent: agent.id,  // string shorthand -> latest version. Or: { type: "agent", id: agent.id, version: agent.version }
    environment_id: environmentId,
    title: "Hello World Session",
  },
);
```

> 提示：**在 Console 即時觀看。** 開發時，列印一個連結讓您可以點擊到 session UI，而不是解析事件串流：`https://platform.claude.com/workspaces/{workspace}/sessions/{session.id}`。只有當 API 金鑰屬於組織的預設工作區時才對 `{workspace}` 使用 `default`；否則替換為工作區的 ID（session 回應不攜帶它——從 Console URL 列讀取或將其設為 API 金鑰旁邊的設定值）。在為本地迭代的使用者生成程式碼時包含這個 `print`/`console.log`。

**Session 建立參數：**

| 欄位 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `agent` | string 或 object | **是** | 三種形式：字串縮寫 `"agent_abc123"`（最新版本）；釘選的 `{type: "agent", id, version}`；或 `{type: "agent_with_overrides", id, version?, ...}` 以僅為這個 session 覆蓋 `model`/`system`/`tools`/`mcp_servers`/`skills`——見 § 為 session 覆蓋 agent 設定 |
| `environment_id` | string | **是** | 環境 ID |
| `title` | string | 否 | 人類可讀的名稱（出現在日誌/儀表板中） |
| `resources` | array | 否 | 在啟動時附加到容器的檔案、GitHub 存放庫或記憶體儲存。記憶體儲存只能在建立 session 時設定（無法透過 `resources.add()` 新增）。 |
| `initial_events` | array | 否 | 在建立時發送的事件，按順序處理——將建立 + 首次發送合併為一次呼叫。見下方 § 用 `initial_events` 預設 session。 |
| `vault_ids` | array | 否 | Vault ID（`vlt_*`）——帶有自動重新整理和在出口時替換的 `environment_variable` 機密的 MCP 憑證。見 `shared/managed-agents-tools.md` -> Vaults。 |
| `budget` | object | 否 | Session 支出的硬性美元上限：`{type: "limit", max_list_cost: {amount, currency}}`。**僅建立時可設定**——之後可以修改或移除，但不能新增。見 § Session 預算。 |
| `metadata` | object | 否 | 使用者提供的鍵值對 |

#### 用 `initial_events` 預設 session

不帶 `initial_events` 建立 session 時，session 以 `idle` 狀態登記，不啟動任何工作；沙箱在 session 第一次需要時才會佈建。傳遞**非空的** `initial_events` 陣列會在同一個呼叫中啟動 agent 迴圈——session **直接以 `running` 狀態建立**，從不經過 `idle`。等待 `idle -> running` 轉換以得知工作已開始的客戶端會永久等待；應改為在建立回應上檢查 `status`。

```python
session = client.beta.sessions.create(
    agent=AGENT_ID,
    environment_id=ENVIRONMENT_ID,
    initial_events=[
        {"type": "user.message", "content": [{"type": "text", "text": "Review the auth module."}]},
    ],
)
```

- **只接受 `user.message` 和 `user.define_outcome`**，最多 **50** 個事件。工具結果類型（`user.tool_confirmation`、`user.tool_result`、`user.custom_tool_result`）會被拒絕，因為還沒有 agent 輪次；`user.interrupt` 也是，因為沒有輪次可以停止。與排程部署的 `initial_events` 不同，session 的不接受 `system.message`。
- 每個事件在建立回應返回之前都會被驗證並持久化，按列表順序，帶有伺服器分配的 ID——就像在建立後立即發布到發送事件端點一樣。每個事件的內容規則與該端點相同。
- **事件不會在建立回應中回傳。** 若需要伺服器分配的 ID，用 `sessions.events.list(session.id)` 讀回它們。
- **驗證是全有或全無：** 若任何事件失敗，整個請求會被拒絕，不會建立 session。空列表等同於省略該欄位。
- 拒絕情況：超過一個 `user.define_outcome` -> 400；沒有 `rubric` 的 `user.define_outcome` -> 400；整個列表中超過 100 個檔案來源的 `document` 內容區塊 -> 400；請求體超過 32 MB -> 413。

因此，結果導向的 session 只需一次呼叫——在 `initial_events` 中傳遞一個 `user.define_outcome`，而不是建立 session 然後再發送事件（見 `shared/managed-agents-outcomes.md`）。

**Agent 設定欄位**（傳遞給 `agents.create()`，而非 `sessions.create()`）：

| 欄位 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `name` | string | **是** | 人類可讀的名稱（1-256 個字元） |
| `model` | string 或 object | **是** | Claude 模型 ID（裸字串，或接受 `id`、`speed`、`effort` 和 `inference_geo` 的物件）。支援所有 Claude 4.5+ 模型。見 § Agent 模型的努力設定和下方的 § 釘選推理地理位置。 |
| `system` | string | 否 | 系統 prompt——定義 agent 的行為（最多 100K 個字元） |
| `tools` | array | 否 | 包含三種類型：(1) 預建 Claude Agent 工具（`agent_toolset_20260401`）、(2) MCP 工具（`mcp_toolset`）和 (3) 自訂用戶端工具。最多 128 個。 |
| `mcp_servers` | array | 否 | MCP 伺服器連接——標準化的第三方功能（例如 GitHub、Asana）。最多 20 個，名稱唯一。見 `shared/managed-agents-tools.md` -> MCP Servers。 |
| `skills` | array | 否 | 自訂的「最佳實踐」情境，帶有漸進式披露。最多 20 個。見 `shared/managed-agents-tools.md` -> Skills。 |
| `description` | string | 否 | Agent 的描述（最多 2048 個字元） |
| `multiagent` | object | 否 | `{type: "coordinator", agents: [...]}` - 此 agent 可委派的名單。見 `shared/managed-agents-multiagent.md`。 |
| `metadata` | object | 否 | 任意鍵值對（最多 16 個，鍵 <=64 個字元，值 <=512 個字元） |

### Session 預算

**Session 預算**是在建立 session 時設定的可選硬性支出上限。平台持續按**公開標準費率**（session 的**標準費率成本**）為 session 消費的一切計費，並在總費用達到上限後停止發出新的模型請求。達到預算上限的 session 會**暫停並以 `stop_reason: budget_reached` 進入 `idle` 狀態**——它不會終止；歷史記錄和沙箱都會保留，修改或移除預算會自動恢復暫停的工作。

```python
session = client.beta.sessions.create(
    agent=AGENT_ID,
    environment_id=ENVIRONMENT_ID,
    budget={
        "type": "limit",
        "max_list_cost": {"amount": "2500", "currency": "USD"},  # minor units: "2500" = $25.00
    },
)
```

- `type` 始終為 `"limit"`。`max_list_cost.amount` 是**貨幣最小單位（分）的整數字串**，不帶前導零，> 0——`"2500"` 是 \$25.00，`"50"` 是五十分。使用字串而非數字，以避免任何浮點取整；小數形式如 `"25.00"` 會被拒絕。`max_list_cost.currency` 是大寫 ISO-4217；**`USD` 是唯一支援的貨幣。**
- **計入標準費率成本的項目：** 按每個服務模型的標準費率計算的模型 token、每 1,000 次網路搜尋 \$10、以及每小時 \$0.08 的 session 執行時間。標準費率成本*不是*您的合約價格——以協商折扣，session 在標準費率總額達到上限時觸發，實際計費支出可能更低。
- **執行是前置請求閘道：** 在每次模型請求之前，平台會檢查已消費的標準費率成本是否已達到上限，若已達到則暫停執行緒；跨越上限的請求會完成，所以最終數字可能每個執行中的執行緒超過上限最多一個模型請求。將預算視為新工作的界限，而非精確的停止點。
- 報告的 `list_cost` 被**四捨五入到最近的分**，而執行時比較精確金額——四捨五入可能使報告數字在精確金額的任一方向移動最多半分，所以報告 `list_cost` 等於其上限的 session 可能尚未暫停。將 `stop_reason: budget_reached`（或 `user.message` 上的 400），而非報告數字，視為已達到上限的訊號。
- **僅建立時可設定。** 為沒有預算的 session 新增預算是 400。更新接受恰好兩種修改：**修改上限**（新值可以高於或低於舊上限，但必須嚴格大於已消費的標準費率成本，否則 400：`budget.max_list_cost must be greater than the session's consumed list cost`）或**移除**（`budget: null`——`session.updated` 事件攜帶 `budget: null` 而非單獨的旗標）。因為 session 暫停時已消費成本通常略高於舊上限，請以 session 報告的 `usage.list_cost` 而非舊的 `max_list_cost` 作為新值的基礎。**移除是單向的**：已移除的預算永遠無法重新新增；若要保持上限，請修改它。
- **達到上限時，只接受結算事件**——解決已在進行中的工作而非開始新工作的事件：`user.tool_confirmation`、`user.tool_result`、`user.custom_tool_result`、`user.interrupt`。在 session 因其預算暫停時（所有執行緒都在上限處暫停）發送的 `user.interrupt` 會被接受但忽略：它不會出現在事件列表中且不改變任何內容。提高或移除預算以繼續。任何開始新工作的事件（例如 `user.message`）都是 400，列出那個列表。沒有事件可以恢復 session——只有預算修改/移除才能恢復。
- **Multiagent：** 所有執行緒共享一個預算，沒有每個執行緒的上限。執行緒獨立暫停；每個執行緒的消費按其自身服務的模型計費。待處理的工具詢問優先於上限：有一個執行緒在 `requires_action` 而另一個在 `budget_reached` 的 session 在 session 層級報告 `requires_action`——照常回答它（結算事件不受阻塞）。
- **沒有標準費率價格的模型不能設定預算：** 建立帶有預算的 session，其 agent（或任何名單 agent，包括顧問的模型）使用未定價的模型時會返回 400。若執行中的有預算 session 的使用包含了一個，則修改預算會被拒絕——移除預算以恢復。
- 達到上限時的串流行為和 `session.usage` 事件：`shared/managed-agents-events.md` § 達到 session 預算。
- 排程部署也可以攜帶預算——複製到每個觸發的 session，具有不同的更新語義（可清除且可重新新增）：`shared/managed-agents-scheduled-deployments.md` § 部署預算。

> **與 Messages API 任務預算不同。** Session 預算是對一個 session 的硬性、以美元計算、平台強制的上限。Messages API 上的 `task_budget` 是模型用來在一個 agentic 迴圈內調整節奏的諮詢性、以 token 計算的預算。

---

## Agents

**這是每個 Managed Agents 流程的起點。** Agent 物件是一個持久化的、版本化的設定——您建立一次，然後每次啟動 session 時都透過 ID 引用它。沒有 agent -> 沒有 session。

### Agent 物件

API 是**扁平的**——`model`、`system`、`tools` 等是頂層欄位，不包裝在 `agent:{}` 子物件中。

| 欄位 | 型別 | 必填 | 說明 |
| --- | --- | --- | --- |
| `name` | string | 是 | 人類可讀的名稱 |
| `model` | string 或 object | 是 | Claude 模型 ID——裸字串，或 `{id, speed?, effort?, inference_geo?}` |
| `system` | string | 否 | 系統 prompt |
| `tools` | array | 否 | Agent 工具組 / MCP 工具組 / 自訂工具 |
| `mcp_servers` | array | 否 | MCP 伺服器連接 |
| `skills` | array | 否 | Skill 引用（最多 20 個） |
| `description` | string | 否 | Agent 的描述 |
| `multiagent` | object | 否 | 協調者名單——見 `shared/managed-agents-multiagent.md` |
| `metadata` | object | 否 | 任意鍵值對 |

### 生命週期：建立一次，多次執行，就地更新

Agent 是一個**持久化資源**，而非每次執行的參數。預期模式：

```
+- setup (once) ---------+     +- runtime (every invocation) -+
| agents.create()        |     | sessions.create(             |
|   -> store agent_id    | ---> |   agent={type:..., id: ID}   |
|     in config/env/db   |     | )                            |
+------------------------+     +------------------------------+
```

**反模式：** 在每次指令碼執行頂端呼叫 `agents.create()`。這會積累孤立的 agent 物件、在每次呼叫時支付建立延遲，並破壞版本控制模型。若您看到 `agents.create()` 在每個請求或每個 cron 滴答呼叫的函式中，那是錯誤的——將其提升到一次性設定並持久化 ID。

> **推薦——以 YAML + 透過 `ant` CLI 定義 agent 和環境。** 分工是 **CLI 負責控制平面，SDK 負責資料平面**：agent 和環境是相對靜態的資源，由 `ant`（從 CI 套用的版本控制 YAML）管理；session 是動態的，由您的應用程式透過 SDK 驅動。見 `shared/anthropic-cli.md` -> *版本控制的 Managed Agents 資源* 以了解 `ant beta:agents create < agent.yaml` / `update --version N` 流程。在其他地方顯示的 SDK `agents.create()` 呼叫是等效的程式碼——當需要以程式化方式佈建時使用它，但對於人類維護的任何內容，優先使用 YAML 流程。

### Agent 模型的努力設定

將 `model` 作為物件傳遞以設定努力等級：`{"id": "claude-opus-5", "effort": "high"}`。`effort` 接受等級字串（`low`、`medium`、`high`、`xhigh`、`max`）或物件如 `{"type": "high"}`。建立/更新回應以物件形式回傳它，並用預設值填充省略的 `model` 欄位。

> 警告：**努力是僅限 agent 設定的。** 在每個 session 的 `model` 覆蓋中設定的 `effort`**不會被套用**——session 以 agent 的努力執行。要修改努力，您必須更新 agent（或讓 session 指向不同的 agent）。這是唯一一個覆蓋形式靜默地不執行任何操作而非報錯的欄位。

相同的物件形式攜帶 `speed` 用於快速模式：`{"id": "claude-opus-5", "speed": "fast"}`。

### 釘選推理地理位置（`inference_geo`）

`model` 物件也接受 `inference_geo` 以釘選服務 agent 模型請求的地理位置：`{"id": "claude-opus-5", "inference_geo": "us"}`。接受 `"us"` 或 `"global"`——不同於 Messages API，那裡的 `inference_geo` 是頂層請求參數，這裡它始終嵌套在 `model` 中，從不頂層。當未設定時，每個模型請求遵循服務時工作區的預設推理地理位置。

- **在每個階段都會驗證：** 儲存 agent 時、從中建立 session 時，以及 session 服務的每個輪次都會針對工作區的 `allowed_inference_geos` 檢查該釘選。若工作區允許列表之後縮窄使釘選不再被允許，就無法從 agent 建立新 session，且**執行中的 session 會拒絕後續輪次**——釘選從不被祖父化（工作區依賴它們合規）。
- 在不支援地理推理釘選的模型上設定 `inference_geo` 返回 400。
- **在 session 的生命週期內是固定的**——釘選無法在 session 中途改變。在 agent 上設定它，或在 session 建立時用 `model` 覆蓋設定/清除它（見 § 為 session 覆蓋 agent 設定）。
- **Multiagent 名單必須地理一致：** 協調者的釘選和每個名單成員的釘選必須全部相同或全部未設定——見 `shared/managed-agents-multiagent.md`。
- 不同於 `effort`，在每個 session 的 `model` 覆蓋中的 `inference_geo`**是會被套用的**——而且因為覆蓋完整替換 `model` 物件，省略了 `inference_geo` 的覆蓋會為那個 session 清除 agent 的釘選。

### 版本控制

每次 `POST /v1/agents/{id}`（更新）都會建立一個新的不可變版本——從 1 開始的序列整數，每次更新遞增。Agent 的歷史記錄是只可追加的——您無法編輯過去的版本。

**更新時的 `version` 是可選的。** 提供它以進行樂觀並發，或省略它以無條件套用更新：

| `version` | 行為 | 適合 |
|---|---|---|
| 已提供（必須 >= 1） | 若不匹配 agent 的當前版本則 409——**即使您發送的欄位已等於儲存的值**。重新讀取並重試。 | 互動式呼叫者；推薦的預設 |
| 省略 | 無條件套用。最近的更新靜默地替換任何並發的更新，沒有任何一方報錯。 | 宣告式套用迴圈——例如同步已簽入 agent 定義的 CI 工作，其中迴圈擁有 agent |

**更新語義。** 省略的欄位會被保留。標量欄位（`model`、`system`、`name`、`description`）會被替換；`system` 和 `description` 可以用 `null` 清除，而 `model` 和 `name` 則不行。陣列欄位（`tools`、`mcp_servers`、`skills`）會被完整替換——`null` 或 `[]` 清除它們。**`effort` 是您提供的 `model` 物件中唯一的例外：** 若模型 `id` 未改變，省略 `effort` 會保留儲存的等級；若您修改了 `id`，省略的 `effort` 會重置為新模型的預設值。其他 `model` 欄位與物件一起被替換——**提供不帶 `inference_geo` 的 `model` 會清除 agent 的推理地理位置釘選。**

**為何要版本：**
- **可重現性** - 將 session 釘選到已知良好的設定：`{type: "agent", id, version: 3}`
- **安全迭代** - 在不破壞已在舊版本上執行的 session 的情況下更新 agent
- **回滾** - 若新的系統 prompt 造成退步，在您除錯時將新 session 釘回先前的版本

**`version` 是可選的。** 省略它（或使用字串縮寫 `agent="agent_abc123"`）在 session 建立時獲取最新版本。明確傳遞它（`{type: "agent", id, version: N}`）以釘選可重現性。

**取得要釘選的版本：** `agents.create()` 和 `agents.update()` 都在回應中返回 `version`。將其與 `agent_id` 一起儲存。要取得現有 agent 的當前最新版本：`GET /v1/agents/{id}` -> `.version`。

**何時更新 vs 建立新的：** 當它在概念上是同一個 agent 帶有調整後的行為時更新（更好的 prompt、額外的工具）。當它是不同的人格/用途時建立新的 agent。經驗法則：若您會給它相同的 `name`，就更新。

### Agent 端點

| 操作 | 方法 | 路徑 |
| --- | --- | --- |
| 建立 | `POST` | `/v1/agents` |
| 列出 | `GET` | `/v1/agents` |
| 取得 | `GET` | `/v1/agents/{id}` |
| 更新 | `POST` | `/v1/agents/{id}` |
| 封存 | `POST` | `/v1/agents/{id}/archive` |

> 警告：**封存是永久的。** 封存使 agent 變為唯讀：現有 session 繼續執行，但**新的 session 無法引用它**，也沒有取消封存。由於 agent 沒有 `delete`，這是終端生命週期狀態。永遠不要封存生產 agent 作為例行清理——先向使用者確認。

### 在 Session 中使用 Agent

透過字串 ID（最新版本）或帶有明確版本的物件引用 agent：

```python
# String shorthand - uses the agent's latest version
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment_id,
)

# Or pin to a specific version (int)
session = client.beta.sessions.create(
    agent={"type": "agent", "id": agent.id, "version": agent.version},
    environment_id=environment_id,
)
```

### 為 session 覆蓋 agent 設定

第三種 `agent` 形式，`agent_with_overrides`，為**單一 session** 替換 agent 設定的部分——試驗不同的模型或授予額外工具，而無需對 agent 進行版本控制。傳遞 `id`（以及可選的 `version`；省略 = 最新，與其他兩種形式相同的預設）加上 `model`、`system`、`tools`、`mcp_servers`、`skills` 中的任何：

```python
session = client.beta.sessions.create(
    agent={
        "type": "agent_with_overrides",
        "id": agent.id,
        "model": "claude-opus-5",   # replace the agent's model for this session
        "system": None,           # clear the system prompt for this session
    },
    environment_id=environment_id,
)
```

每個可覆蓋的欄位遵循三態規則：
- **省略** -> session 從引用的 agent 版本繼承值。
- **`null`（或列表欄位的 `[]`）** -> session 以清除該欄位的狀態執行。完全適用於 `system` 和 `skills`。三個例外：`model` 永遠不可清除（`model: null` -> 400 `agent_model_required`）；當 session 的有效 `skills` 非空時清除 `tools` 返回 400（skills 需要 `read` 工具）；以及當有效 `tools` 仍包含引用 agent 某個伺服器的 `mcp_toolset` 時清除 `mcp_servers` 返回 400——在同一個請求中覆蓋 `tools` 以刪除那些條目，然後清除 `mcp_servers`。
- **一個值** -> 完整替換 agent 的值。覆蓋永遠不合併——`tools` 覆蓋必須列出 session 應有的每個工具。一個例外：`model` 覆蓋中的 `effort` 等級**不會被套用**（改為在 agent 上設定它——見 § Agent 模型的努力設定）。`model` 覆蓋中的 `inference_geo`**會被套用**——而且因為物件被完整替換，省略它的覆蓋會清除 agent 的釘選，讓 session 遵循工作區的預設推理地理位置。覆蓋的值在 session 建立時針對工作區的 `allowed_inference_geos` 進行驗證。

覆蓋是 session 本地的：它們**不**修改 agent 資源或建立新的 agent 版本。回應的 `agent` 物件反映覆蓋後的設定，而其 `id` 和 `version` 仍然識別基礎 agent——所以您可以將 session 追溯到其基礎。在 multiagent session 中，覆蓋適用於協調者及其 `{type: "self"}` 副本；透過 ID 引用的名單 agent 始終使用其自身的建立時設定（見 `shared/managed-agents-multiagent.md`）。

### 在 session 中途更新 agent 設定

`sessions.update()` 可以在**現有** session 上修改 `agent.tools` 和 `agent.mcp_servers`（包括權限策略和每個工具的網路設定——`allowed_domains`/`blocked_domains` 等，見 `shared/managed-agents-tools.md` § 網路搜尋和網路擷取設定）。更新後的網域列表適用於 session 的其餘部分。這是一個 **session 本地覆蓋**——它不會建立新的 agent 版本，也不會傳播回 agent 物件。提供的陣列是**完整替換**；要追加一個工具，`GET` session，修改，然後 `POST` 回去。Session 必須處於 `idle`——若正在執行則先中斷。`vault_ids` 是**僅建立時可設定**：SDK 中存在更新參數但 API 拒絕（「Not yet supported」）——建立 session 時附加保管庫。

在 agent 設定欄位中，只有 `tools` 和 `mcp_servers` 可以在建立 session 後改變——要以不同於 agent 值的 `model`、`system` 或 `skills` 執行，請在建立時使用 `agent_with_overrides`（上方）。（`title`、`metadata` 和 `budget` 有其自己的 session 更新路徑——見 § Session 操作 / § Session 預算。）Agent 的模型設定——包括其 `inference_geo` 釘選——以及其設定的 `system` 欄位在 session 的生命週期內是固定的；您仍然可以透過發送 `system.message` 事件在輪次之間**追加系統層級情境**（見 `shared/managed-agents-events.md` § 在 session 中途新增系統情境）。

```python
client.beta.sessions.update(
    session.id,
    agent={
        "tools": [
            {"type": "agent_toolset_20260401"},
            {"type": "mcp_toolset", "mcp_server_name": "linear"},
        ],
        "mcp_servers": [{"type": "url", "name": "linear", "url": "https://mcp.linear.app/sse"}],
    },
)
```
