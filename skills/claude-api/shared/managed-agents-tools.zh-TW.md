---
source_file: managed-agents-tools.md
source_commit: 34040c9c568585f6929bedeaad110ad08f079624
source_sha256: 3238b1e2c76a31af87a2ed90a3dba5d0293a9682001fbc8c7c23947ab0bb6470
translated_at: 2026-09-13
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`managed-agents-tools.md`](managed-agents-tools.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
<!-- /translation-header -->

# Managed Agents - 工具與 Skills

## 工具

### 伺服器工具 vs 用戶端工具

| 類型 | 誰執行它 | 運作方式 |
|---|---|---|
| **預建 Claude Agent 工具**（`agent_toolset_20260401`） | Anthropic，在 session 的容器上（對於 `cloud` 環境；對於 `self_hosted`，**您的** worker 提供並執行檔案/bash 工具——見 `shared/managed-agents-self-hosted-sandboxes.md`）。`web_search` / `web_fetch` 在這兩種環境類型中都始終在 Anthropic 的伺服器上執行。 | 檔案操作、bash、網路搜尋等。一次啟用所有工具或用 `enabled: true/false` 個別設定；用 `allowed_domains` / `blocked_domains` 限制網路工具。 |
| **MCP 工具**（`mcp_toolset`） | Anthropic 的編排層 | 由已連接的 MCP 伺服器公開的功能。透過工具組按伺服器授予存取權。 |
| **自訂工具** | **您**——您的應用程式處理呼叫並返回結果 | Agent 發出帶有輸入的 `agent.custom_tool_use` 事件，session 進入 `idle`，您發回帶有輸出的 `user.custom_tool_result` 事件。 |

**建議：** 透過 `agent_toolset_20260401` 啟用所有預建工具，然後按需個別停用。

**版本控制：** 工具組是版本化的靜態資源。當底層工具改變時，會建立新的工具組版本（因此是 `_20260401`），讓您始終確切知道您取得的是什麼。

### Agent 工具組

`agent_toolset_20260401` 提供這些內建工具：

| 工具 | 說明 |
| --- | --- |
| `bash` | 在 shell session 中執行 bash 指令 |
| `read` | 從本地檔案系統讀取檔案，包括文字、圖片、PDF 和 Jupyter notebooks |
| `write` | 將檔案寫入本地檔案系統 |
| `edit` | 在檔案中執行字串替換 |
| `glob` | 使用 glob 模式進行快速檔案模式匹配 |
| `grep` | 使用 regex 模式進行文字搜尋 |
| `web_fetch` | 從 URL 擷取內容 |
| `web_search` | 在網路上搜尋資訊 |

啟用完整工具組：

```json
{
  "tools": [
    { "type": "agent_toolset_20260401" }
  ]
}
```

### 每個工具的設定

覆蓋個別工具的預設值。此範例除了 bash 之外全部啟用：

```json
{
  "tools": [
    {
      "type": "agent_toolset_20260401",
      "default_config": { "enabled": true },
      "configs": [
        { "name": "bash", "enabled": false }
      ]
    }
  ]
}
```

| 欄位 | 必填 | 說明 |
|---|---|---|
| `type` | 是 | `"agent_toolset_20260401"` |
| `default_config` | 否 | 套用到所有工具。`{ "enabled": bool, "permission_policy": {...} }` |
| `configs` | 否 | 每個工具的覆蓋：`[{ "name": "...", "type": "...", "enabled": bool, "permission_policy": {...} }]`。`name` 識別工具（上表中的值）；`type` 在請求中是可選的（與 `name` 相同的值；伺服器會推斷它），在回應中始終存在。`web_search` / `web_fetch` 條目也接受網路設定——見下方 § 網路搜尋和網路擷取設定。 |

> **型別化 SDK：** 每個 `configs` 條目是每個內建工具一個成員的聯合型別（八個：`BetaManagedAgentsWebFetchToolConfigParams`、`...WebSearchToolConfigParams`、`...BashToolConfigParams`、...），以 `type` 為識別子。Python/TypeScript/Ruby 的字典和雜湊只帶 `name` + `enabled` + `permission_policy` 保持不變。在 Go、Java、C# 和 PHP 中，`configs` 是聯合型別本身——從其每個工具型別構建每個條目（Go：`BetaManagedAgentsAgentToolConfigUnionParamsUnion{OfWebFetch: &anthropic.BetaManagedAgentsWebFetchToolConfigParams{...}}`——分支是 `OfBash` / `OfRead` / `OfWrite` / `OfEdit` / `OfGlob` / `OfGrep` / `OfWebFetch` / `OfWebSearch`；Java：`.addConfig(BetaManagedAgentsWebFetchToolConfigParams.builder()...build())`；C#：`new BetaManagedAgentsWebFetchToolConfigParams { Enabled = false }`；PHP：`BetaManagedAgentsWebFetchToolConfigParams::with(enabled: false)`）。針對所有工具共享一個設定型別的 SDK 撰寫的程式碼必須更新其構建條目的方式。

### 權限策略

控制伺服器執行的工具（agent 工具組 + MCP）是自動執行、等待您的核准，或讓伺服器評估每次呼叫。不適用於自訂工具（您的應用程式執行這些）。

| 策略 | 行為 |
|---|---|
| `always_allow` | 工具自動執行。agent 工具組的預設值。 |
| `always_ask` | Session 發出 `session.status_idle`（`stop_reason.type: requires_action`）並暫停，直到您發送 `user.tool_confirmation` 事件。MCP 工具組的預設值。 |
| `auto` | 伺服器評估每次呼叫（工具 + 輸入 + 迄今的 session 內容）並**執行、拒絕，或暫停等待您的核准**。兩種工具組都不預設為 `auto`。見下文 §`auto`。 |

```json
{
  "type": "agent_toolset_20260401",
  "default_config": {
    "enabled": true,
    "permission_policy": { "type": "always_allow" }
  },
  "configs": [
    { "name": "bash", "permission_policy": { "type": "always_ask" } }
  ]
}
```

**回應 `always_ask`**（以及暫停的 `auto` 呼叫）：發送 `user.tool_confirmation` 事件，將 `tool_use_id` 設為觸發的 `agent.tool_use` / `agent.mcp_tool_use` 事件的**事件 ID**（`sevt_...`，不是 `toolu_` ID）。多個確認可以放在一個 `events` 請求中：

```json
{ "type": "user.tool_confirmation", "tool_use_id": "sevt_abc123", "result": "allow" }
{ "type": "user.tool_confirmation", "tool_use_id": "sevt_def456", "result": "deny", "deny_message": "Read .env.example instead" }
```

拒絕上可選的 `deny_message` 作為被拒絕的工具結果傳遞給 agent，讓它可以調整方法。`evaluated_permission` 不是 `"ask"` 的事件的 `user.tool_confirmation` 會被拒絕並返回 400——這包括伺服器在 `auto` 策略下拒絕的呼叫；您的客戶端無法覆蓋它們。

#### `auto`——讓伺服器評估每次呼叫

在接受 `permission_policy` 的任何位置設定 `{"type": "auto"}`：工具組的 `default_config` 或個別的 `configs` 條目，在 agent 工具組或 `mcp_toolset` 上均可。由於評估考慮呼叫的輸入以及迄今的 session 內容，對同一工具的兩次呼叫可能得到不同的處理。每次呼叫恰好有三種結果之一：

| 結果 | 發生什麼 |
|---|---|
| **執行** | 伺服器判定呼叫安全——在不通知您客戶端的情況下，如同 `always_allow` 一樣執行。 |
| **拒絕** | 伺服器評估呼叫為高風險——工具不執行。agent 收到錯誤工具結果（`Permission to use {tool_name} has been denied.`，`is_error: true`），session **繼續執行**，您的客戶端無法覆蓋拒絕。 |
| **暫停** | 伺服器未能做出判定——session 恰好如同 `always_ask` 一樣暫停；以 `user.tool_confirmation` 回應。 |

```json
{
  "name": "Ops Agent",
  "model": "claude-opus-5",
  "mcp_servers": [{ "type": "url", "name": "github", "url": "https://mcp.example.com/github" }],
  "tools": [
    {
      "type": "agent_toolset_20260401",
      "default_config": { "permission_policy": { "type": "auto" } },
      "configs": [{ "name": "bash", "permission_policy": { "type": "always_ask" } }]
    },
    {
      "type": "mcp_toolset",
      "mcp_server_name": "github",
      "default_config": { "permission_policy": { "type": "auto" } }
    }
  ]
}
```

在 Python、TypeScript 和 Ruby 中以無類型字典 / 物件字面值 / hash 傳入相同形狀。有類型的 SDK（Go、Java、C#、PHP）需要隨每個 SDK 功能發布版本附帶的 `auto` 策略生成類型——在此之前，使用無類型語言或透過 cURL / `ant` 建構請求。Python 和 TypeScript 也只從新增它的發布版本起才對 `{"type": "auto"}` 進行類型檢查（wire API 無論如何都接受它）。

**評估信任什麼。** 伺服器將 session 內容視為評估材料，而非要遵循的指令。您在 `user.message` 事件中發送的文字（包括您在那裡轉發的終端使用者文字）算作*您的意圖*，可以讓伺服器允許原本可能拒絕的呼叫——儘管某些呼叫無論如何都會被評估為高風險。工具結果、擷取的網頁、MCP 伺服器回應或 session 執行緒間的訊息中的相同文字則不具此效力。如果您在 `user.message` 中轉發不受信任的終端使用者輸入，伺服器也會將其讀作您的意圖，可能讓呼叫被允許——對您不想讓該終端使用者不經審核就執行的工具，請設定 `always_ask`。

> **`auto` 不是人工審核點。** 伺服器判定為安全的呼叫在任何人看到之前就已執行，其效果可能無法復原。如果必須由人工在工具呼叫執行前審核，請對該工具使用 `always_ask`。

#### `evaluated_permission` 和 `evaluation`——查看每次呼叫的評估結果

在**任何**策略下，每個 `agent.tool_use` 和 `agent.mcp_tool_use` 事件都帶有 `evaluated_permission`（`"allow" | "ask" | "deny"`）——權限檢查的結果。大多數事件還帶有 `evaluation` 物件，其 `type` 說明產生結果的策略；在 `auto` 下還加入伺服器的判定，以及對於 `ask` / `deny` 加入 `reason_code`：

```json
{
  "type": "agent.tool_use",
  "id": "sevt_01pqr...",
  "name": "bash",
  "input": { "command": "rm -rf /workspace/reports" },
  "evaluated_permission": "deny",
  "evaluation": {
    "type": "auto",
    "evaluated_permission": { "type": "deny", "reason_code": "high_risk" }
  },
  "processed_at": "2026-03-25T14:05:12Z"
}
```

| `evaluation` | 頂層 `evaluated_permission` | 含義 |
|---|---|---|
| `{"type": "always_allow"}` | `"allow"` | 已解析策略為 `always_allow`；呼叫已執行。 |
| `{"type": "always_ask"}` | `"ask"` | 已解析策略為 `always_ask`；暫停等待您的核准。 |
| `{"type": "auto", "evaluated_permission": {"type": "allow"}}` | `"allow"` | 伺服器判定呼叫安全；已執行。 |
| `{"type": "auto", "evaluated_permission": {"type": "ask", "reason_code": "indeterminate"}}` | `"ask"` | 伺服器未能做出判定；暫停等待您的核准。 |
| `{"type": "auto", "evaluated_permission": {"type": "deny", "reason_code": "high_risk"}}` | `"deny"` | 伺服器評估呼叫為高風險並予以拒絕。 |

- 在 `auto` 形式上，嵌套的 `evaluated_permission.type` 始終等於事件的頂層 `evaluated_permission`。
- `reason_code` 供您的客戶端分支處理並保存於稽核記錄中——不是要顯示給終端使用者的文字。
- 當 agent 命名的工具未在 session 中啟用時（伺服器在未評估任何策略的情況下拒絕：`evaluated_permission: "deny"`，無 `evaluation`），以及在欄位存在之前記錄的事件上（對 `"allow"` 讀作 `always_allow`，對 `"ask"` 讀作 `always_ask`），`evaluation` **不存在**。
- 編寫您的客戶端以容忍無法識別的 `evaluation.type` 或 `reason_code`。
- `agent.custom_tool_use` 事件不帶這兩個欄位（自訂工具不受權限策略管轄）。

要只啟用特定工具，將預設值翻轉為關閉並逐個工具選擇啟用：

```json
{
  "tools": [
    {
      "type": "agent_toolset_20260401",
      "default_config": { "enabled": false },
      "configs": [
        { "name": "bash", "enabled": true },
        { "name": "read", "enabled": true }
      ]
    }
  ]
}
```

### 網路搜尋和網路擷取設定（網域過濾）

`web_search` 和 `web_fetch` 無論環境類型如何都在 Anthropic 的伺服器上執行，所以環境的 `networking` 策略**不**管轄它們（見 `shared/managed-agents-environments.md` -> Networking）。要控制它們能到達的地方，在工具的 `configs` 條目上設定 `allowed_domains`（只有這些主機）**或** `blocked_domains`（永遠不是這些主機）——永遠不要在同一個條目上同時設定兩者。每個工具攜帶自己的列表。Console 中的組織層級網路搜尋/擷取設定只適用於 Messages API，不適用於 Managed Agents session。

```json
{
  "type": "agent_toolset_20260401",
  "configs": [
    {
      "type": "web_search",
      "name": "web_search",
      "allowed_domains": ["docs.example.com", "arxiv.org"],
      "user_location": { "type": "approximate", "country": "US", "timezone": "America/Los_Angeles" }
    },
    {
      "type": "web_fetch",
      "name": "web_fetch",
      "blocked_domains": ["ads.example.com"],
      "max_content_tokens": 50000
    }
  ]
}
```

| 設定 | 適用於 | 說明 |
|---|---|---|
| `allowed_domains` | `web_search`、`web_fetch` | 工具唯一可以到達的主機。在同一個條目上與 `blocked_domains` 互斥。 |
| `blocked_domains` | `web_search`、`web_fetch` | 工具無法到達的主機。 |
| `max_content_tokens` | `web_fetch` | 進入情境的擷取*文字*內容的正整數上限（二進位內容如 PDF 不受上限限制）。 |
| `user_location` | `web_search` | `{ "type": "approximate", city?, region?, country?（2 字母大寫 ISO 3166-1）, timezone?（IANA）}`——至少一個可選欄位。 |

**執行時行為：** 不在其列表範圍內的 `web_fetch` 呼叫會向 agent 返回錯誤結果（`agent.tool_result` 上的 `is_error: true`，內容名稱為 `url_not_allowed`）；`web_search` 靜默地省略不在其列表範圍內的結果。在 Console 中，agent 表單有網路工具的允許/阻止列表控制項；`user_location` 和 `max_content_tokens` 在 agent 的 **Raw** 視圖中設定。

**網域列表規則**（違規 -> 400 `invalid_request_error`，在 agent 建立/更新和提供 `tools` 的 session 建立/更新時；訊息列出清單和從零開始的索引，例如 `allowed_domains.0: IP addresses are not supported...`）：

- 每個列表 1-64 個網域，每個 1-255 個字元。空列表會被拒絕——省略欄位或發送 `null` 表示「無限制」。列表中的重複項會被拒絕。
- 只有純主機名稱：`example.com`，而非 `https://example.com`、`example.com:443` 或 `*.example.com`。不區分大小寫；單一尾隨 `/` 會被忽略。
- 列出的網域覆蓋自身**及其子網域**（`example.com` 覆蓋 `docs.example.com`；`docs.example.com` 不覆蓋 `example.com` 或 `api.example.com`）。`www.` 是普通子網域——列出裸網域以同時涵蓋兩者。
- 拒絕：任何形式的 IP 位址；裸 TLD/登錄後綴（`com`、`co.uk`）；單一標籤名稱（`intranet`）；`localhost` 以及以 `.localhost`、`.local`、`.internal`、`.localdomain`、`.invalid` 結尾的主機；非 ASCII（使用 `xn--` Punycode）。
- `web_fetch` 網域不能攜帶路徑。`web_search` 網域可以攜帶路徑後綴（`example.com/blog`，沒有空格 / `?` / `#` / `$ , | ^ !`），但供應商將其作為 URL 模式匹配——優先使用純主機名稱。
- 提供者相關的同時拒絕：Anthropic 爬蟲可能無法存取的網域、不受支援的 `user_location.country`（訊息以 `not a country the search provider supports` 結尾）、無效的 IANA `timezone`。

Session 在首次初始化工具時重新檢查設定；若之前接受的設定不再有效，它會發出 `session.error` 並進入 `idle` 而不重試。透過 session 工具更新修復（`shared/managed-agents-core.md` -> 在 session 中途更新 agent 設定），也更新 agent 讓新 session 得到修復，然後發送新的 `user.message`。

**Multiagent 層疊**（見 `shared/managed-agents-multiagent.md`）：路徑上到一個執行緒的每個列表都同時適用——名單 agent 受其自身列表、呼叫它的每個 agent 的列表，以及協調者的*當前*列表約束。允許列表取交集，阻止列表取聯集，所以名單 agent 只能縮窄而永遠不能擴大。不相交的允許列表讓工具可用但每次呼叫都失敗 `url_not_allowed`（工具描述告知模型）——讓名單允許列表在協調者的列表範圍內。`max_content_tokens` 和 `user_location`**不**合併：自身值 -> 呼叫者的 -> 協調者的。`{type: "self"}` 條目遵循協調者。結果評分器（`shared/managed-agents-outcomes.md`）在不帶網路工具的情況下執行。更新閒置 session 的工具會從下一個輪次起改變協調者對所有執行緒的列表；名單 agent 自己的列表保持在 session 建立時定義的樣子。

**與 Messages API `web_search_20260209` / `web_fetch_20260209` 工具相比：** 相同的 `allowed_domains` / `blocked_domains` 詞彙，但 64 個條目上限、`web_fetch` 網域上沒有路徑，以及沒有 `max_uses`、`citations` 或 `cache_control`。若從 Messages API 遷移，這些從每個請求移到了 agent 上的一次性設定。

### 自訂工具（用戶端）

自訂工具由**您的應用程式**執行，而非 Anthropic。流程：

1. Agent 決定使用工具 -> session 發出帶有輸入的 `agent.custom_tool_use` 事件
2. Session 進入 `idle` 等待您
3. 您的應用程式執行工具
4. 您發回帶有輸出的 `user.custom_tool_result` 事件
5. Session 恢復 `running`

不需要權限策略——您就是執行者。

```json
{
  "tools": [
    {
      "type": "custom",
      "name": "get_weather",
      "description": "Fetch current weather for a city.",
      "input_schema": {
        "type": "object",
        "properties": {
          "city": { "type": "string", "description": "City name" }
        },
        "required": ["city"]
      }
    }
  ]
}
```

### MCP 伺服器

MCP（Model Context Protocol）伺服器公開標準化的第三方功能（例如 Asana、GitHub、Linear）。**設定分布在 agent 和保管庫之間：**

1. **Agent 建立**宣告要連接的伺服器（`type`、`name`、`url`——無驗證）。Agent 的 `mcp_servers` 陣列沒有驗證欄位。
2. **Vault** 儲存 OAuth 憑證。透過 session 建立時的 `vault_ids` 附加。

這讓機密不進入可重用的 agent 定義。每個保管庫憑證綁定到一個 MCP 伺服器 URL；Anthropic 按 URL 將憑證與伺服器匹配。

**Agent 端——宣告伺服器（無驗證）：**

| 欄位 | 必填 | 說明 |
|---|---|---|
| `type` | 是 | `"url"` |
| `name` | 是 | 唯一名稱——由 `mcp_toolset.mcp_server_name` 引用 |
| `url` | 是 | MCP 伺服器的端點 URL（Streamable HTTP transport） |

```json
{
  "mcp_servers": [
    { "type": "url", "name": "linear", "url": "https://mcp.linear.app/mcp" }
  ],
  "tools": [
    { "type": "mcp_toolset", "mcp_server_name": "linear" }
  ]
}
```

**Session 端——附加保管庫：**

```json
{
  "agent": "agent_abc123",
  "environment_id": "env_abc123",
  "vault_ids": ["vlt_abc123"]
}
```

> 提示：**每個工具啟用：** `mcp_toolset` 接受 `default_config: {enabled: false}` + `configs: [{name, enabled: true}]` 以實現允許列表模式。MCP `configs` 條目**只**接受 `name`（伺服器報告的裸工具名稱）、`enabled` 和 `permission_policy`——沒有 `type` 欄位，也沒有 `web_search` / `web_fetch` 在 agent 工具組中接受的任何網路設定。

> 提示：**在執行中的 session 上修改工具/MCP 伺服器：** `sessions.update()` 可以在 session 處於 `idle` 時替換 `agent.tools` 和 `agent.mcp_servers`——一個不觸及 agent 物件的 session 本地覆蓋。`vault_ids` 是僅建立時可設定的。見 `shared/managed-agents-core.md` -> 在 session 中途更新 agent 設定。

**大型工具輸出。** 若工具返回超過 **100,000 個字元（大約 25,000 個 token）**，輸出會自動轉存到沙箱中的一個檔案——agent 收到截斷的預覽加上檔案路徑，可以 `read` 完整內容。不需要任何設定。閾值是以*字元*計算，而非 token，且同時適用於內建 agent 工具和 MCP 工具。

**無效的保管庫憑證不會阻止 session 建立。** 若某個已宣告 MCP 伺服器的保管庫憑證無效，session 仍可成功建立；`session.error` 事件描述 MCP 驗證失敗，驗證會在下一個 `session.status_idle` -> `session.status_running` 轉換時重試。

> 警告：**MCP 驗證 token ≠ REST API token。** 託管 MCP 伺服器（`mcp.notion.com`、`mcp.linear.app` 等）通常需要 **OAuth bearer token**，而非服務的原生 API 金鑰。Notion 的 `ntn_` 整合 token 可以針對 Notion 的 REST API 進行驗證，但**不能**作為 Notion MCP 伺服器的保管庫憑證。這是不同的驗證系統。

### 保管庫——憑證儲存

**保管庫**儲存 Anthropic 代您管理的憑證。兩種憑證類別：

- **MCP 憑證**（`mcp_oauth`、`static_bearer`）——以 `mcp_server_url` 為鍵。當 agent 連接到該 URL 的伺服器時，token 會自動注入。**匹配是正規化的，而非位元組精確的：** 方案和主機是小寫的，預設埠和尾隨斜線被移除，所以主機大小寫、明確的預設埠或尾隨斜線不會破壞匹配。不同的路徑、子網域或*非預設*埠則會。若無匹配，連接會在未驗證的情況下嘗試。`mcp_oauth` token 透過標準 OAuth 2.0 `refresh_token` grant 自動重新整理。這是驗證 MCP 伺服器的唯一方式。
- **環境變數**（`environment_variable`）——以 `secret_name`（環境變數名稱）為鍵。沙箱只看到**不透明的佔位符**；真實機密在**出口時**替換到出站請求中。用於透過環境變數驗證的任何服務：CLI（`aws`、`gcloud`、`stripe`）、SDK，或從 `bash` 工具直接 `curl` 呼叫。

您提供的機密欄位（`token`、`access_token`、`refresh_token`、`client_secret`、`secret_value`）是只寫的——絕不在 API 回應中返回。

#### 憑證與沙箱

保管庫儲存憑證；這些憑證**永遠不進入沙箱**。這是有意為之的安全邊界——在沙箱中執行的程式碼（包括 agent 撰寫的任何內容）無法讀取或洩露保管庫憑證，即使在 prompt 注入攻擊下也是如此。相反，憑證由 Anthropic 端的代理在請求**離開沙箱後**注入：

- **MCP 工具呼叫**透過 Anthropic 端的代理路由，該代理從保管庫獲取憑證並將其新增到出站請求。
- **附加 GitHub 存放庫上的 Git 操作**（`git pull`、`git push`、GitHub REST 呼叫）透過 git 代理路由，該代理以相同方式注入 `github_repository` 資源的 `authorization_token`。
- **環境變數憑證**在沙箱中作為不透明佔位符出現；真實值在出口時替換佔位符，僅針對憑證允許的主機的請求。替換覆蓋請求的**標頭和本體**——嵌入在 **URL 路徑**中的機密永遠不會被替換，所以路徑機密端點（例如 Slack incoming-webhook URL）無法使用保管庫；改用基於標頭的驗證（對於 Slack：在 `Authorization` 中的 bot token 透過 `chat.postMessage`）。

**當保管庫憑證不適用時**（例如自託管沙箱——那裡還不支援 `environment_variable`），**登記一個自訂工具：** agent 發出 `agent.custom_tool_use`，您的協調器（已持有憑證）執行呼叫並透過同一個已驗證的事件串流返回 `user.custom_tool_result`。不公開任何公開端點；沙箱永遠看不到機密。見 `shared/managed-agents-client-patterns.md` -> Pattern 9。

**不要把 API 金鑰放在系統 prompt 或使用者訊息中作為解決方法**——它們會持久化在 session 的事件歷史記錄中。

> 以前內部稱為 TAT（Tool/Tenant Access Tokens）。

**流程：**

1. 建立保管庫（`client.beta.vaults.create(...)`）——每個租戶/使用者一個，或一個共享的，取決於您的模型
2. 向其新增憑證（`client.beta.vaults.credentials.create(...)`）——MCP 憑證以 MCP 伺服器 URL 為鍵；環境變數憑證以 `secret_name` 為鍵
3. 在 session 建立時透過 `vault_ids: ["vlt_..."]` 引用保管庫
4. Anthropic 在 OAuth token 過期前自動重新整理，並在執行時替換機密

**MCP OAuth 憑證形式**：

```json
{
  "display_name": "Notion (workspace-foo)",
  "auth": {
    "type": "mcp_oauth",
    "mcp_server_url": "https://mcp.notion.com/mcp",
    "access_token": "<current access token>",
    "expires_at": "2026-04-02T14:00:00Z",
    "refresh": {
      "refresh_token": "<refresh token>",
      "client_id": "<your OAuth client_id>",
      "token_endpoint": "https://api.notion.com/v1/oauth/token",
      "token_endpoint_auth": { "type": "none" }
    }
  }
}
```

`refresh` 區塊是啟用自動重新整理的關鍵——`token_endpoint` 是 Anthropic 發布 `refresh_token` grant 的地方。`token_endpoint_auth` 是識別子聯合型別：

| `type` | 形式 | 使用時機 |
|---|---|---|
| `"none"` | `{type: "none"}` | 公開 OAuth 客戶端（無機密） |
| `"client_secret_basic"` | `{type: "client_secret_basic", client_secret: "..."}` | 機密客戶端，透過 HTTP Basic auth 傳遞機密 |
| `"client_secret_post"` | `{type: "client_secret_post", client_secret: "..."}` | 機密客戶端，在請求本體中傳遞機密 |

若您只有無法重新整理的存取 token，則完全省略 `refresh`——它在過期前都有效，然後 agent 失去存取權。

> 提示：**取得 OAuth token。** 初始存取和重新整理 token 的取得方式取決於 MCP 伺服器——請參閱其文件。一旦您擁有它們，使用上面的形式將它們儲存在保管庫憑證中；Anthropic 從那裡透過 `refresh.token_endpoint` 自動重新整理。

**環境變數憑證形式**：

```json
{
  "display_name": "Twilio API key for sandbox",
  "auth": {
    "type": "environment_variable",
    "secret_name": "TWILIO_API_KEY",
    "secret_value": "sk-your-secret-here",
    "networking": {
      "type": "limited",
      "allowed_hosts": ["api.twilio.com", "*.twilio.com"]
    }
  }
}
```

`networking.allowed_hosts` 控制機密可以替換的出站主機——`{"type": "limited", "allowed_hosts": [...]}` 或 `{"type": "unrestricted"}`（若您無法提前列舉網域）。強烈建議限制：它防止金鑰被發送到未授權的主機。

**`injection_location`**（可選，是 `networking` 的同輩欄位）控制機密在出站請求的**哪個部分**被替換——`{header: bool, body: bool}`。兩者是獨立的：`allowed_hosts` 限定替換請求的*目標主機*；`injection_location` 限定所有這些主機中請求的*哪些部分*替換機密。大多數服務從請求標頭讀取 API 金鑰，所以 `{"header": true}` 是較窄的設定——請求本體通常由 agent 正在處理的內容組成，使本體成為較廣的暴露面。在禁用位置的佔位符**既不被替換也不被移除**——字面上不透明的佔位符字串在那個位置被發送給第三方。

| 操作 | `injection_location` 語義 |
|---|---|
| 建立憑證 | 完全省略欄位 -> 兩個位置都啟用。提供物件 -> 省略的任何欄位預設為 `false`（`{"header": true}` 建立僅標頭憑證）。 |
| 更新憑證 | 欄位**個別合併**——`{"body": false}` 停用本體替換，`header` 保持不變。對於執行中的 session，更新在 session 的下一次操作時生效。 |

憑證必須至少啟用一個位置；會停用兩者的建立或更新返回 400，物件或任一欄位的明確 `null` 也是如此（改為省略）。回應始終返回兩個欄位及其解析後的值。

> 警告：**Console 中建立的憑證預設僅標頭**——與 API 不同，省略欄位時兩者都啟用。若您的客戶端在請求本體中發送機密（例如表單編碼的 token 請求），佔位符會字面通過，服務會以其自身的驗證錯誤拒絕它。在 Console 表單中勾選本體注入，或用 `{"injection_location": {"body": true}}` `POST` 憑證。

> 警告：**兩個網路層，兩者都需要。** 憑證上的 `networking.allowed_hosts` 控制哪些請求*使用機密*，而非哪些請求*被允許*。Agent 也必須能夠在**環境層級**到達該網域（`unrestricted`，或環境的 `allowed_hosts` 中列出的主機——見 `shared/managed-agents-environments.md`）。任一層缺少網域意味著機密替換的請求失敗。

> 警告：**用戶端驗證警告。** 替換在出口時發生，而非在沙箱內——在進行網路請求之前在本地*格式*驗證憑證的客戶端（例如檢查金鑰以 `sk-` 開頭的 CLI）會看到不透明的佔位符，可能在啟動時失敗。若客戶端在任何網路呼叫之前拒絕憑證，那就是原因。

> 提示：**最小化金鑰範圍。** Agent 可以執行金鑰允許的任何操作；比任務需要更廣泛權限的金鑰在 agent 行為意外時會增加爆炸半徑。

**自託管沙箱不支援**——`environment_variable` 憑證需要 Anthropic 管理的出口。見 `shared/managed-agents-self-hosted-sandboxes.md`。

**限制（所有憑證類型）：**

- **每個保管庫唯一鍵。** `mcp_server_url`（MCP 憑證）和 `secret_name`（環境變數憑證）在保管庫中的活躍憑證中必須唯一；重複會返回 409。
- **金鑰是不可變的。** 機密值、`display_name`，以及（在環境變數憑證上）`injection_location` 可以更新；要修改 `mcp_server_url`、`secret_name`、`token_endpoint` 或 `client_id`，封存憑證並建立新的。封存會清除機密並釋放金鑰以便替換。
- **每個保管庫最多 20 個憑證。**
- 憑證按提供的方式儲存，**直到 session 執行時才驗證**——無效的憑證在 session 期間作為驗證或下游錯誤出現，這些錯誤會被發出，但不會阻止 session 繼續。

**範圍：** 保管庫是工作區範圍的。API 工作區中具有開發者及以上角色的任何人都可以建立、讀取（僅中繼資料——機密是只寫的）和附加保管庫。`vault_ids` 可以在 session **建立**時設定，但不能透過 session 更新（SDK 文件字串說「Not yet supported; requests setting this field are rejected」）。

---

## Skills

Skills 是可重用的、基於檔案系統的資源，為您的 agent 提供領域特定的專業知識：工作流程、情境和最佳實踐，將通用 agent 轉變為專家。不同於 prompt（針對一次性任務的對話層級指示），skills 按需載入，消除了在多次對話中重複提供相同指導的需要。

Skills 透過兩種方式到達 agent：**附加**到 agent 的 `skills` 陣列，或**從掛載在 session 上的 GitHub 存放庫載入**（見下方 § 從 GitHub 存放庫獲取 Skills）。Agent 在任務相關時會自動使用它們：

| 類型 | 說明 |
|---|---|
| **預建 Anthropic skills** | 常見的文件任務（PowerPoint、Excel、Word、PDF）。按名稱引用（例如 `xlsx`）。 |
| **自訂 skills** | 您在組織中透過 Skills API 建立的 skills。按 `skill_id` + 可選的 `version` 引用。 |

**每個 agent 最多 20 個 skills。** Agent 建立使用 `managed-agents-2026-04-01`；管理自訂 skill 定義的獨立 Skills API 使用 `skills-2025-10-02`。

### 在 session 上啟用 skills

Skills 透過 `agents.create()` 附加到 **agent** 定義：

```ts
const agent = await client.beta.agents.create(
  {
    name: "Financial Agent",
    model: "claude-opus-5",
    system: "You are a financial analysis agent.",
    skills: [
      { type: "anthropic", skill_id: "xlsx" },
      { type: "custom", skill_id: "skill_abc123", version: "latest" },
    ],
  }
);
```

Python：

```python
agent = client.beta.agents.create(
    name="Financial Agent",
    model="claude-opus-5",
    system="You are a financial analysis agent.",
    skills=[
        {"type": "anthropic", "skill_id": "xlsx"},
        {"type": "custom", "skill_id": "skill_abc123", "version": "latest"},
    ]
)
```

**Skill 引用欄位：**

| 欄位 | Anthropic skill | 自訂 skill |
|---|---|---|
| `type` | `"anthropic"` | `"custom"` |
| `skill_id` | Skill 名稱（例如 `"xlsx"`、`"docx"`、`"pptx"`、`"pdf"`） | Skills API 中的 skill ID（例如 `"skill_abc123"`） |
| `version` | `"latest"` 或特定版本號 | `"latest"` 或特定版本號 |

`version` 在**兩種**類型上都是可選的，預設為 `"latest"`——不是僅限自訂 skill。

### 從 GitHub 存放庫獲取 Skills

Skills 也可以存放在您的程式碼庫中。當 session 透過 `github_repository` 資源掛載存放庫時（見 `shared/managed-agents-environments.md` -> GitHub Repositories），存放庫的根目錄 `.claude/skills` 會在 session 啟動時被掃描，每個找到的 skill 都對 agent 可用：它看到每個已發現 skill 的名稱、描述和沙箱路徑，並在任務匹配時讀取 skill 的 `SKILL.md`（加上它附帶的任何指令碼/資源）。

**Agent 可以發現 `.claude/skills/<skill-name>/` 中的任何 skill**——在存放庫根目錄下一個目錄層級深。以下位置的 Skills 不可發現：裸 `.claude/skills/SKILL.md`（無 skill 目錄）、嵌套更深的任何內容（`.claude/skills/tools/code-review/SKILL.md`）、`.claude` 之外的 `skills/` 目錄，或套件子目錄中的 `.claude/skills`（儘管 agent 在讀取該子樹下的檔案時仍然可以發現它們）。`SKILL.md` 格式與上傳的自訂 skills 相同。

> 警告：**存放庫 skills 是 agent 指示——將它們視為您的信任邊界的一部分。** 任何可以提交到已掛載存放庫的人（合併的外部 PR、受損的相依、貢獻者）都可以新增或編輯 `.claude/skills/` 內容，平台在 session 啟動時載入它，沒有審查步驟——像 `bash` 和 `web_fetch` 這樣的 session 工具賦予了注入的指示真正的能力。只掛載您信任的存放庫，並在掛載有外部貢獻者的存放庫之前審計 `.claude/skills/`。

規則：
- **僅限雲端沙箱**——自託管沙箱不支援 `github_repository` 資源，所以無法載入存放庫 skills。
- **在 session 啟動時掃描一次**，從當時簽出的存放庫狀態（資源的 `checkout` 分支/提交，否則預設分支）。session 中途推送的提交不會被接收——啟動新 session 以使用更新的 skills。新增到*執行中* session 的存放庫也不會被掃描。
- **與附加的 skills 共存。** 若存放庫 skill 與附加的 skill（或另一個已掛載存放庫的 skill）共享名稱，兩者都可用，各自以其自己的路徑宣布。

### Skills API

| 操作 | 方法 | 路徑 |
| --- | --- | --- |
| 建立 Skill | `POST` | `/v1/skills` |
| 列出 Skills | `GET` | `/v1/skills` |
| 取得 Skill | `GET` | `/v1/skills/{id}` |
| 刪除 Skill | `DELETE` | `/v1/skills/{id}` |
| 建立版本 | `POST` | `/v1/skills/{id}/versions` |
| 列出版本 | `GET` | `/v1/skills/{id}/versions` |
| 取得版本 | `GET` | `/v1/skills/{id}/versions/{version}` |
| 刪除版本 | `DELETE` | `/v1/skills/{id}/versions/{version}` |
