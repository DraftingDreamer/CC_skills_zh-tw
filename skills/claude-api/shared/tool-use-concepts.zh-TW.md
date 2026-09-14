---
source_file: tool-use-concepts.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 0c5db4fa5a0b60f0e47e473b7735bf94c62b35c6d345f2cfed06409deb9b8f2e
translated_at: 2026-09-05
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`tool-use-concepts.md`](tool-use-concepts.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
<!-- /translation-header -->

# Tool Use 概念

本文涵蓋使用 Claude API 進行 tool use 的概念基礎。語言專屬的程式碼範例請見 `python/`、`typescript/` 或其他語言資料夾。關於要公開哪些工具、如何管理長期執行 agent 中的情境，以及快取策略的決策啟發式，請見 `agent-design.md`。

## 使用者定義工具

### 工具定義結構

> **注意：** 使用 Tool Runner（beta）時，工具綱要會從函式簽名（Python）、Zod 綱要（TypeScript）、標註類別（Java）、`jsonschema` struct 標籤（Go）或 `BaseTool` 子類別（Ruby）自動生成。下方的原始 JSON 綱要格式適用於手動方式——包括 PHP 的 `BetaRunnableTool`（用 run 閉包包裝手寫的綱要）——或沒有 tool runner 支援的 SDK。

每個工具需要名稱、描述和輸入的 JSON Schema：

```json
{
  "name": "get_weather",
  "description": "Get current weather for a location",
  "input_schema": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "City and state, e.g., San Francisco, CA"
      },
      "unit": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "description": "Temperature unit"
      }
    },
    "required": ["location"]
  }
}
```

**工具定義的最佳實踐：**

- 使用清晰、描述性的名稱（例如 `get_weather`、`search_database`、`send_email`）
- 撰寫詳細的描述——Claude 用這些來決定何時使用工具。要**規定性地說明*何時*呼叫**，而不只是說明它做什麼（例如「當使用者詢問當前價格或近期事件時呼叫此工具」）。在最新的 Opus 模型上，模型對工具的使用較為保守，描述中的觸發條件可以明顯提高應呼叫率。
- 為每個屬性包含描述
- 對有固定值集合的參數使用 `enum`
- 在 `required` 中標記真正必填的參數；讓其他參數帶有預設值成為可選
---

### 工具選擇選項

控制 Claude 何時使用工具：

| 值 | 行為 |
| --- | --- |
| `{"type": "auto"}` | Claude 自行決定是否使用工具（預設） |
| `{"type": "any"}` | Claude 必須至少使用一個工具 |
| `{"type": "tool", "name": "..."}` | Claude 必須使用指定的工具 |
| `{"type": "none"}` | Claude 不能使用工具 |

任何 `tool_choice` 值也可以包含 `"disable_parallel_tool_use": true`，強制 Claude 每次回應最多只使用一個工具。預設情況下，Claude 可能在單次回應中請求多個工具呼叫。

**Claude Fable 5.1、Claude Mythos 5.1 和 Mythos Preview 拒絕強制工具使用：** `{"type": "any"}` 和 `{"type": "tool", "name": ...}` 在這些模型上返回 400（`tool_choice: type "tool" and "any" are not supported for this model.`——在 `count_tokens` 和 Batches 上也是如此）。這是模型特定的限制（Claude Fable 5 和 Claude Opus 5 接受它們）。改用 `{"type": "auto"}` 並在 prompt 中陳述期望（「使用 get_weather 工具回答」）——工具上的 `strict: true` 保持 `any` 給您的綱要有效引數保證——或在強制呼叫只是為了擷取 JSON 時使用結構化輸出（`output_config.format`）。`auto` 和 `none` 不受影響；帶有 `auto` 的 `disable_parallel_tool_use` 仍意味著最多一個呼叫（與 `any`/`tool` 的「恰好一個」組合已不存在）。將 `tool_choice` `any` 與 `strict: true` 組合只在支援強制工具使用的模型上適用。見 `shared/model-migration.md` -> 從 Claude Fable 5 遷移到 Claude Fable 5.1。

---

### Tool Runner 與手動迴圈

**Tool Runner（推薦）：** SDK 的 tool runner 自動處理 agentic 迴圈——它呼叫 API、偵測工具使用請求、執行您的工具函式、將結果回饋給 Claude，並重複直到 Claude 停止呼叫工具。適用於 Python、TypeScript、Java、Go、Ruby、PHP 和 C# SDK（beta）。Python SDK 還提供 MCP 轉換輔助方法（`anthropic.lib.tools.mcp`），可將 MCP 工具、prompt 和資源轉換以供 tool runner 使用——詳見 `python/claude-api/tool-use.md`。**任何自訂工具 agent 都預設使用 tool runner。**

**Tool runner 不是黑盒——「我需要控制」很少是降級到手動迴圈的理由。** 每次迭代在工具執行*前*產出 assistant 訊息並讓您介入，所以大多數「細粒度控制」需求都可以滿足，無需手寫迴圈：

- **人工在迴路批准 / 閘道**——在工具的 run 函式中設置閘道（返回「使用者拒絕」結果而非執行），或在產出的訊息中檢查工具呼叫，並用 `set_messages_params()` / `setMessagesParams()` / `append_messages()` / `pushMessages()` 覆蓋待處理請求，在工具執行*前*允許或拒絕。只有當您不介入時，runner 才會自動執行您的函式。
- **錯誤攔截**——在工具結果返回給 Claude 之前檢查它（`generate_tool_call_response()` / `generateToolResponse()`）；提前停止或自行處理。
- **結果修改**——在結果返回之前修改它（例如新增 `cache_control` 用於 prompt 快取，或轉換輸出）。
- **每輪重試 / 參數更改**——例如提高 `max_tokens` 並重新執行被截斷的輪次；用 `max_iterations` 限制整個迴圈。
- **串流和自動壓縮**均受支援。

這些鉤子是 SDK 輔助功能，而非獨立的 API 參數——確切的方法名稱和範例，請 WebFetch `shared/live-sources.md` -> *Claude API SDK Repositories* 中列出的每語言 SDK 存放庫（tool runner 輔助方法位於每個存放庫的 `tools.md` / `helpers.md`）。附帶的 `python/claude-api/tool-use.md` 和 `typescript/claude-api/tool-use.md` 顯示基本的 tool runner 設定。

**不要因為這些誤解而降級到手動迴圈：**

- Tool runner 不需要 Zod/Pydantic——`betaTool()`（TS）和 `@beta_tool`（Python）接受原始 JSON Schema；其他 SDK 使用普通的 struct/map/class。
- Runner 讓偵測最終輪次*更容易*，而非更難——當 Claude 停止呼叫工具時迭代結束，最後產出的訊息就是最終回應。大多數 SDK 還提供一次性變體（`runner.until_done()` / `runner.runUntilDone()` / `RunToCompletion()`）。
- 確認 / 批准閘道可以與 runner 一起使用（見下方安全性）。

**手動 Agentic 迴圈：** 只有在想完全擁有*整個*迴圈時才使用——您需要 runner 未公開的控制（例如自訂傳輸、SDK 無法構建的請求形狀、在 runner 不支援的 SDK 上進行每個 token 的串流）、不想承擔 beta 相依，或您的控制流程不適合 runner 的每輪鉤子（例如在迴圈中途交錯無關的工作）。批准閘道、日誌記錄、攔截、結果修改和條件執行**不**需要它——tool runner 涵蓋這些（上方）。迴圈直到 `stop_reason == "end_turn"`，始終追加完整的 `response.content` 以保留 tool_use 區塊，並確保每個 `tool_result` 包含匹配的 `tool_use_id`。

**伺服器端工具的停止原因：** 使用伺服器端工具（程式碼執行、網路搜尋等）時，API 執行伺服器端採樣迴圈。若此迴圈達到其預設的 10 次迭代上限，回應將有 `stop_reason: "pause_turn"`。要繼續，重新發送使用者訊息和 assistant 回應，然後再發出一個 API 請求——伺服器將從中斷處恢復。**不要**新增額外的使用者訊息如「繼續。」——API 偵測到尾隨的 `server_tool_use` 區塊並知道自動恢復。

```python
# Handle pause_turn in your agentic loop
if response.stop_reason == "pause_turn":
    messages = [
        {"role": "user", "content": user_query},
        {"role": "assistant", "content": response.content},
    ]
    # Make another API request - server resumes automatically
    response = client.messages.create(
        model="claude-opus-5", messages=messages, tools=tools
    )
```

**注意：** SDK tool runner 不會自動恢復 `pause_turn`（截至 `@anthropic-ai/sdk` 0.110.0 / `anthropic` 0.116.0）——暫停的輪次結束 runner 並作為最終訊息返回，沒有錯誤。在 TypeScript 中，您可以在迭代本體內恢復（將暫停的 assistant 輪次推回 runner）；在 Python 中，runner 無法在迴圈中途恢復——用追加了暫停輪次的新 runner 重新啟動，或在手動迴圈中處理 `pause_turn`。每種語言的 `tool-use.md` 中有對應模式。

設定 `max_continuations` 上限（例如 5）以防止無限迴圈。完整指南請見：`https://platform.claude.com/docs/en/build-with-claude/handling-stop-reasons`

> **安全性：** Tool runner 在 Claude 請求時自動執行您的工具函式。對於有副作用的工具（發送電子郵件、修改資料庫、財務交易），驗證輸入並在破壞性操作前設置人工批准閘道。**Tool runner 和手動迴圈都**支援此功能——使用 tool runner 時，在工具的 run 函式內設置閘道（提示使用者並返回「使用者拒絕」結果而非執行），或在每個產出的訊息中檢查工具呼叫，並用 `set_messages_params()` / `setMessagesParams()` 接管訊息歷史記錄，在工具執行*前*允許或拒絕（只有當您不介入時才自動執行您的函式）；使用手動迴圈時，在呼叫函式之前內聯設置閘道。

---

### 處理工具結果

當 Claude 使用工具時，回應包含 `tool_use` 區塊。您必須：

1. 用提供的輸入執行工具
2. 在 `tool_result` 訊息中發回結果
3. 繼續對話

**工具結果中的錯誤處理：** 當工具執行失敗時，設定 `"is_error": true` 並提供有資訊性的錯誤訊息。Claude 通常會確認錯誤並嘗試不同的方法或請求澄清。

**多個工具呼叫：** Claude 可以在單次回應中請求多個工具。在繼續之前處理所有工具——在**單一**使用者訊息中發回所有結果。

---

## 伺服器端工具：程式碼執行

程式碼執行工具讓 Claude 在安全的沙箱容器中執行程式碼。與使用者定義工具不同，伺服器端工具在 Anthropic 的基礎設施上執行——您不在用戶端執行任何東西。只需包含工具定義，Claude 就會處理其餘的事。

### 關鍵事實

- 在隔離的容器中執行（1 CPU、5 GiB RAM、5 GiB 磁碟）
- 無網路存取（完全沙箱化）
- 預裝資料科學函式庫的 Python 3.11
- 容器持久化 30 天，可跨請求重用
- 與網路搜尋 / 網路擷取工具一起使用時免費；否則每個組織每月 1,550 個免費小時後 \$0.05 / 小時

### 工具定義

工具不需要綱要——只需在 `tools` 陣列中宣告：

```json
{
  "type": "code_execution_20260120",
  "name": "code_execution"
}
```

Claude 自動獲得 `bash_code_execution`（執行 shell 命令）和 `text_editor_code_execution`（建立 / 查看 / 編輯檔案）的存取權。

### 預裝的 Python 函式庫

- **資料科學**：pandas、numpy、scipy、scikit-learn、statsmodels
- **視覺化**：matplotlib、seaborn
- **檔案處理**：openpyxl、xlsxwriter、pillow、pypdf、pdfplumber、python-docx、python-pptx
- **數學**：sympy、mpmath
- **工具程式**：tqdm、python-dateutil、pytz、sqlite3

可以在執行時透過 `pip install` 安裝額外的套件。

### 支援的上傳檔案類型

| 類型 | 副檔名 |
| --- | --- |
| 資料 | CSV、Excel（.xlsx/.xls）、JSON、XML |
| 圖片 | JPEG、PNG、GIF、WebP |
| 文字 | .txt、.md、.py、.js 等 |

### 容器重用

跨請求重用容器以維持狀態（檔案、已安裝套件、變數）。從第一個回應中提取 `container_id` 並傳遞給後續請求。

### 回應結構

回應包含交錯的文字和工具結果區塊：

- `text`——Claude 的說明
- `server_tool_use`——Claude 正在做什麼
- `bash_code_execution_tool_result`——程式碼執行輸出（檢查 `return_code` 以了解成功 / 失敗）
- `text_editor_code_execution_tool_result`——檔案操作結果

> **安全性：** 在將下載的檔案寫入磁碟之前，始終用 `os.path.basename()` / `path.basename()` 清理檔名，以防止路徑穿越攻擊。將檔案寫入專用的輸出目錄。

---

## 伺服器端工具：網路搜尋和網路擷取

網路搜尋和網路擷取讓 Claude 搜尋網路並擷取頁面內容。它們在伺服器端執行——只需包含工具定義，Claude 就會自動處理查詢、擷取和結果處理。

### 工具定義

```json
[
  { "type": "web_search_20260209", "name": "web_search" },
  { "type": "web_fetch_20260209", "name": "web_fetch" }
]
```

### 動態過濾（Claude Opus 5 / Fable 5 / Opus 4.8 / Opus 4.7 / Opus 4.6 / Sonnet 5 / Sonnet 4.6）

`web_search_20260209` 和 `web_fetch_20260209` 版本支援**動態過濾**——Claude 撰寫並執行程式碼在搜尋結果進入情境視窗之前過濾它們，提高準確性和 token 效率。動態過濾內建在這些工具版本中並自動啟動；您不需要單獨宣告 `code_execution` 工具或傳遞任何 beta 標頭。

```json
{
  "tools": [
    { "type": "web_search_20260209", "name": "web_search" },
    { "type": "web_fetch_20260209", "name": "web_fetch" }
  ]
}
```

沒有動態過濾的話，先前的 `web_search_20250305` 版本也可用。

> **注意：** 只有當您的應用程式出於自身目的（資料分析、檔案處理、視覺化）需要程式碼執行，且獨立於網路搜尋時，才包含獨立的 `code_execution` 工具。在 `_20260209` 網路工具旁邊包含它會建立第二個執行環境，可能讓模型困惑。

---

## 伺服器端工具：程式化工具呼叫

使用標準工具呼叫時，每次工具呼叫都是一個往返：Claude 呼叫，結果進入 Claude 的情境，Claude 推理，然後呼叫下一個工具。連鎖呼叫會累積延遲和 token——大多數中間資料再也不會需要。

程式化工具呼叫讓 Claude 將那些呼叫組合成一個指令碼。指令碼在程式碼執行容器中執行；當它呼叫工具時，容器暫停，呼叫執行，結果返回到執行中的程式碼（而非 Claude 的情境）。指令碼用正常的控制流程處理結果。只有最終輸出返回給 Claude。當連鎖多個工具呼叫，或中間結果很大且應在到達情境視窗之前過濾時使用它。

完整文件請 WebFetch：

- URL：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/programmatic-tool-calling`

---

## 伺服器端工具：工具搜尋

工具搜尋工具讓 Claude 從大型函式庫中動態發現工具，無需將所有定義載入情境視窗。當您有許多工具但每個請求只有少數相關時使用。發現的工具綱要會追加到請求，而非換入——這保留了 prompt 快取（見 `agent-design.md` §代理快取）。

完整文件請 WebFetch：

- URL：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/tool-search-tool`

---

## 對話中途工具變更（Beta）

**Beta 標頭 `mid-conversation-tool-changes-2026-07-01`；Claude Opus 5 及以後。** 通常 `tools` 在對話的生命週期內是固定的——編輯它會改變 prompt 前綴的最前面並使整個快取失效（見 `prompt-caching.md` § 失效層級）。此功能讓您在輪次之間新增和移除工具，同時快取的前綴得以保留。

兩種操作都是追加到 `messages[]` 的 `{"role": "system", ...}` 訊息上的內容區塊，兩者都透過 `tool_reference` 按名稱引用工具：

```python
# Removal - must sit immediately before an assistant message, or last in messages.
{"role": "system", "content": [
    {"type": "tool_removal", "tool": {"type": "tool_reference", "name": "get_weather"}},
]}

# Addition - surfaces a tool declared up front with defer_loading.
{"role": "system", "content": [
    {"type": "tool_addition", "tool": {"type": "tool_reference", "name": "get_forecast"}},
]}
```

**您計劃新增的工具必須已用 `"defer_loading": True` 宣告在 `tools[]` 中。** 延遲的工具在請求中已知但不載入到模型的情境中，直到 `tool_addition` 使它們浮現：

```python
tools = [
    {"name": "get_weather", "description": "Get weather",
     "input_schema": {"type": "object", "properties": {"city": {"type": "string"}}}},
    {"name": "get_forecast", "description": "Get 5-day forecast",
     "input_schema": {"type": "object", "properties": {"city": {"type": "string"}}},
     "defer_loading": True},
]
```

**要變更工具的定義**，在兩個請求中進行：在第一個請求發送舊定義的 `tool_removal`，然後在下一個請求中以更新後的條目繼續對話。

> 警告：早期預覽版使用了不同的 beta 標頭和不同的區塊形狀；兩者都已棄用。使用帶有 `tool_addition` / `tool_removal` / `tool_reference` 的 `mid-conversation-tool-changes-2026-07-01`。

SDK 型別落後這些區塊——在 Python 中將它們作為普通 dict 傳遞，或在 TypeScript 中新增 `@ts-expect-error`。

**在此和工具搜尋之間選擇：** 工具搜尋是為了*發現*——Claude 自行從大型函式庫中找到它需要的東西。對話中途工具變更是為了*控制*——您的應用程式決定工具集已改變（模式切換、一個可用的資源、您想移除的功能）並明確說明。

---

## Agent Skills（Messages API）

Agent Skills 打包了 Claude 在相關時載入的任務特定指示和檔案（例如 Anthropic 預建的 `pptx`、`xlsx`、`pdf`、`docx` skills）。在 **Messages API** 上，skills 透過 `container` 參數與程式碼執行工具一起啟用——這**不是** Managed Agents 介面，**不使用** `client.beta.agents` / `sessions` / `environments`。可用性：見 `shared/platform-availability.md`。

每個請求都需要：

1. 帶有 `code-execution-2025-08-25` beta 旗標的 `client.beta.messages.create(...)`（Skills 已出 beta——不需要 `skills-2025-10-02` 標頭）。
2. `container={"skills": [{"type": "anthropic", "skill_id": "<id>", "version": "latest"}]}`——skills 列表選擇哪些 skills 在執行容器內可用。
3. `tools=[{"type": "code_execution_20260521", "name": "code_execution"}]`——skills 透過容器中的程式碼執行來執行。

```python
response = client.beta.messages.create(
    model="claude-opus-5", max_tokens=16000,
    betas=["code-execution-2025-08-25"],
    container={"skills": [{"type": "anthropic", "skill_id": "pptx", "version": "latest"}]},
    tools=[{"type": "code_execution_20260521", "name": "code_execution"}],
    messages=[{"role": "user", "content": "Create a 3-slide presentation on X"}],
)
```

生成的檔案（`.pptx`、`.xlsx`、...）寫入容器內；回應攜帶每個檔案的 file ID。透過將該 ID 傳遞給 Files API 下載（`client.files.download(file_id)` / `GET /v1/files/{id}/content`）。

透過 `GET /v1/skills`（無需 beta 標頭）列出可用的 skills。

---

## MCP 連接器（Beta）

MCP 連接器讓 Claude 直接從 Messages API 呼叫遠端 MCP 伺服器上託管的工具——Anthropic 在伺服器端建立 MCP 連接。需要在 `client.beta.messages.create(...)` 上帶有 beta 旗標 `mcp-client-2025-11-20`。可用性：見 `shared/platform-availability.md`。

**需要兩個參數一起使用：**

- `mcp_servers`——伺服器連接定義的陣列：`[{"type": "url", "url": "<server URL>", "name": "<server-name>", "authorization_token": "<optional>"}]`
- `tools`——必須包含按名稱引用伺服器的 `mcp_toolset` 條目：`[{"type": "mcp_toolset", "mcp_server_name": "<server-name>"}]`

toolset 中的 `mcp_server_name` 必須與 `mcp_servers` 中的某個 `name` 匹配。省略 `mcp_toolset` 條目會被作為驗證錯誤拒絕——`mcp_servers` 中的每個伺服器必須被恰好一個工具組引用。

```python
client.beta.messages.create(
    model="claude-opus-5", max_tokens=1024,
    betas=["mcp-client-2025-11-20"],
    mcp_servers=[{"type": "url", "url": "https://example/sse", "name": "example-mcp"}],
    tools=[{"type": "mcp_toolset", "mcp_server_name": "example-mcp"}],
    messages=[...],
)
```

Go 使用型別化常數 `anthropic.AnthropicBetaMCPClient2025_11_20`；舊的 `...2025_04_04` 常數已棄用。

可選的工具組欄位：`default_config`（所有工具的預設值，例如 `{"enabled": false}` 用於允許列表模式）和 `configs`（按工具名稱鍵值的每工具覆蓋）。

---

## Tool Use 範例

您可以在工具定義中直接提供工具呼叫範例，以展示使用模式並減少參數錯誤。這幫助 Claude 理解如何正確格式化工具輸入，尤其是對於具有複雜綱要的工具。

完整文件請 WebFetch：

- URL：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use`

---

## 用戶端工具：電腦使用

電腦使用讓 Claude 與桌面環境互動（截圖、滑鼠、鍵盤）。它是用戶端工具——您的應用程式提供環境並執行 Claude 請求的動作；Anthropic 即時處理截圖和動作請求，但不託管環境或保留資料。

完整文件請 WebFetch：

- URL：`https://platform.claude.com/docs/en/agents-and-tools/computer-use/overview`

---

## 情境編輯

情境編輯從執行記錄中清除陳舊的工具結果和思考區塊，作為長期執行的 agent 累積輪次的一種方式。與壓縮（摘要）不同，情境編輯是修剪——已清除的內容被移除，而非替換。當舊的工具輸出不再相關且想在不失去對話結構的情況下保持執行記錄精簡時使用它。

**Beta。** 使用帶有 beta `context-management-2025-06-27` 的 `client.beta.messages.*`。透過帶有策略類型 `clear_tool_uses_20250919`（清除舊工具結果；可選的 `clear_tool_inputs: true` 也清除 tool_use 參數）或 `clear_thinking_20251015`（清除思考區塊）的 `context_management.edits` 進行設定。這些**不是**壓縮類型——帶有 beta `compact-2026-01-12` 的 `compact_20260112` 是獨立的壓縮功能。

完整文件請 WebFetch：

- URL：`https://platform.claude.com/docs/en/build-with-claude/context-editing`

---

## 伺服器端工具：顧問（Beta）

顧問工具將較快、較低成本的**執行者**模型（請求上的頂層 `model`）與提供中途策略指導的較高智慧**顧問**模型（工具定義內的 `model` 欄位）配對。執行者做大部分的 token 生成；顧問在規劃時被諮詢。可用性：見 `shared/platform-availability.md`。

### 工具定義

```json
{
  "type": "advisor_20260301",
  "name": "advisor",
  "model": "claude-opus-4-8"
}
```

工具定義上的可選欄位：

- `max_uses`——每個請求的顧問諮詢上限。超過它會讓 `advisor_tool_result` 區塊的 `content` 成為錯誤物件 `{"type": "advisor_tool_result_error", "error_code": "max_uses_exceeded"}`——下方酬載形狀表中內容聯合型別的第三個成員。
- `max_tokens`——限制顧問每次呼叫的總輸出（思考 + 文字）。在上限處，結果區塊攜帶 `stop_reason: "max_tokens"` 並在執行者看到的建議後追加截斷說明；伺服器還在顧問的 prompt 中發出剩餘 token 預算區塊，讓它自我調整到上限。
- `caching`——顧問自己 prompt 的快取控制，與快取斷點相同形狀：`"caching": {"type": "ephemeral", "ttl": "5m"}`（`ttl` 是 `"5m"` 或 `"1h"`，預設 `"5m"`）。每次呼叫在該 TTL 寫入快取條目，使對話後續呼叫可以讀取穩定前綴。省略 = 顧問 prompt 不快取。

**顧問模型必須至少與執行者一樣強大。** 無效的配對返回 `400 invalid_request_error`。有效配對：

| 執行者（請求 `model`） | 有效顧問（工具 `model`） |
|---|---|
| `claude-haiku-4-5` / `claude-sonnet-4-6` / `claude-sonnet-5` / `claude-opus-4-6` / `claude-opus-4-7` | `claude-opus-5`、`claude-fable-5-1`、`claude-mythos-5-1`、`claude-fable-5`、`claude-mythos-5`、`claude-opus-4-8` 或 `claude-opus-4-7` |
| `claude-opus-4-8` | `claude-opus-5`、`claude-fable-5-1`、`claude-mythos-5-1`、`claude-fable-5`、`claude-mythos-5` 或 `claude-opus-4-8` |
| `claude-opus-5` | `claude-opus-5`、`claude-fable-5-1`、`claude-mythos-5-1`、`claude-fable-5` 或 `claude-mythos-5` |
| `claude-fable-5` | `claude-fable-5-1`、`claude-mythos-5-1`、`claude-fable-5`、`claude-mythos-5` 或 `claude-opus-5` |
| `claude-mythos-5` | `claude-mythos-5-1`、`claude-fable-5-1`、`claude-mythos-5`、`claude-fable-5` 或 `claude-opus-5` |
| `claude-fable-5-1` / `claude-mythos-5-1` | `claude-mythos-5-1`、`claude-fable-5-1`、`claude-mythos-5`、`claude-fable-5` 或 `claude-opus-5`——而且這些執行者拒絕強制 `tool_choice`，所以從 prompt 中暗示顧問呼叫（`-5-1` 顧問返回加密的 `advisor_redacted_result`，如同 claude-opus-5 / claude-fable-5 / claude-mythos-5） |

> 警告：**顧問的酬載形狀因顧問模型而異。** 回應區塊始終是 `advisor_tool_result`；變化的是其 **`content`**，一個可辨別的聯合型別：
>
> | `content` 類型 | 欄位 | 時機 |
> |---|---|---|
> | `advisor_result` | `text`、`stop_reason` | 顧問返回純文字（例如 Opus 4.8） |
> | `advisor_redacted_result` | `encrypted_content`、`stop_reason` | 顧問返回加密輸出——Claude Opus 5、Claude Fable 5.1、Claude Mythos 5.1、Claude Fable 5、Claude Mythos 5 |
> | `advisor_tool_result_error` | `error_code` | 諮詢失敗——`max_uses_exceeded`、`prompt_too_long`、`too_many_requests`、`overloaded`、`unavailable`、`execution_time_exceeded` 或 `model_not_found` |
>
> 所以按 `advisor_tool_result.content` 類型切換，而非按區塊類型。無條件讀取 `.text` 的程式碼從 Claude Opus 5 顧問什麼都得不到，因為酬載在 `encrypted_content` 下——您無法讀取它，只能重播它。

透過帶有 `betas=["advisor-tool-2026-03-01"]`（或 `anthropic-beta: advisor-tool-2026-03-01` 標頭）的 `client.beta.messages.create(...)` 呼叫。在多輪對話中，將完整的 `response.content`——包括任何 `advisor_tool_result` 區塊——追加回下一輪的 `messages`。若在後續輪次從 `tools` 移除顧問工具，而歷史記錄中仍包含 `advisor_tool_result` 區塊，API 返回 400。

> **Managed Agents 上的顧問：** CMA session 也支援顧問，設定為 agent 的 multiagent 名單中的 `{"type": "advisor", "model"}` 條目，而非工具定義——沒有 `max_uses`/`max_tokens`/`caching` 選項，建議以 session 事件串流上的執行緒事件傳遞，而非 `advisor_tool_result` 區塊。見 `shared/managed-agents-multiagent.md` -> 顧問。

---

## 用戶端工具：記憶體

記憶體工具讓 Claude 透過記憶體檔案目錄跨對話儲存和擷取資訊。Claude 可以建立、讀取、更新和刪除在 session 之間持久化的檔案。

### 關鍵事實

- 用戶端工具——您透過自己的實作控制儲存
- 支援命令：`view`、`create`、`str_replace`、`insert`、`delete`、`rename`
- 在 `/memories` 目錄中的檔案上操作
- Python、TypeScript 和 Java SDK 提供實作記憶體後端的輔助類別 / 函式

> **安全性：** 永遠不要在記憶體檔案中儲存 API 金鑰、密碼、token 或其他機密。處理個人識別資訊（PII）時要謹慎——在持久化使用者資料之前檢查資料隱私法規（GDPR、CCPA）。參考實作沒有內建的存取控制；在多使用者系統中，在工具處理器中實作每使用者的記憶體目錄和驗證。

完整實作範例請 WebFetch：

- 文件：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/memory-tool.md`

---

## 用戶端工具：Bash 和文字編輯器

Bash 和文字編輯器工具是 **Anthropic 定義的、無綱要**工具。只需透過 `type` 和 `name` 宣告它們——輸入綱要內建在模型中且無法修改。**不要傳遞 `input_schema`**，也不要定義一個碰巧叫做 `"bash"` 的自訂工具——那會建立一個沒有內建行為的使用者定義工具。

兩者都是**用戶端執行**：Claude 返回 `tool_use` 區塊，您的程式碼在本地執行動作，然後您發回 `tool_result`。API 是無狀態的；您的應用程式在輪次之間維護 shell session 或檔案系統。

### Bash 工具宣告

```json
{"type": "bash_20250124", "name": "bash"}
```

| 語言 | 宣告 |
|---|---|
| Python / TypeScript / Ruby / cURL | 普通物件 `{"type": "bash_20250124", "name": "bash"}` |
| Go | `anthropic.ToolUnionParam{OfBashTool20250124: &anthropic.ToolBash20250124Param{}}` |
| Java | `.addTool(ToolBash20250124.builder().build())` 來自 `com.anthropic.models.messages` |
| C# | `Tools = [new ToolBash20250124()]` 來自 `Anthropic.Models.Messages` |
| PHP | `tools: [new \Anthropic\Messages\ToolBash20250124()]` |

Claude 的 `tool_use.input` 包含 `{"command": "<string>"}` 或 `{"restart": true}`。先檢查 `restart`（重置 session，返回確認字串）；否則執行 `command` 並返回合併的 stdout + stderr。

> **安全性——命令是不受信任的模型輸出。** 在隔離的環境中執行（容器、VM 或受限使用者）；套用允許的可執行檔的**允許列表**並拒絕 shell 運算子（`&&`、`|`、`;`、反引號、`$()`）；設置逾時和資源限制；記錄每個命令。黑名單是不夠的。

### 文字編輯器工具宣告

```json
{"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"}
```

可選欄位：`max_characters` 以限制 `view` 輸出。Java 公開型別化的 `ToolTextEditor20250728` 建構器（`com.anthropic.models.messages`）；其他靜態型別的 SDK 遵循相同的命名模式——確切的類別見 `{lang}/claude-api/tool-use.md` 中的 Anthropic-Defined Tools 章節。

> **安全性——`path` 是不受信任的模型輸出。將每個檔案操作限制在固定的專案根目錄內。** 在執行任何命令之前，將模型提供的 `path` 解析為其標準形式，並驗證它仍在您的專案根目錄內；若它逃逸（`..`、符號連結、根目錄外的絕對路徑、URL 編碼的穿越如 `%2e%2e%2f`）則拒絕請求。使用您語言的內建路徑工具程式（例如 Python `pathlib.Path.resolve()` 然後檢查 `.is_relative_to(root)`）。永遠不要直接在原始的 `path` 值上呼叫 `open()` / `writeFile` / `unlink`。

`tool_use.input.command` 是以下之一：

| `command` | 其他輸入 | 動作 |
|---|---|---|
| `view` | `path`、可選的 `view_range` | 返回檔案內容或目錄列表 |
| `create` | `path`、`file_text` | 用 `file_text` 建立 / 覆蓋檔案。若檔案已存在則建立備份。 |
| `str_replace` | `path`、`old_str`、`new_str` | 替換恰好一個出現；若 0 個或 >1 個匹配則報錯 |
| `insert` | `path`、`insert_line`、`insert_text` | 在第 `insert_line` 行後插入 `insert_text`（0 = 檔案開頭） |

對於兩種工具，在發生錯誤時返回 `{"type": "tool_result", "tool_use_id": "...", "content": "<error text>", "is_error": true}` 讓 Claude 可以恢復。

---

## 結構化輸出

結構化輸出約束 Claude 的回應遵循特定的 JSON 綱要，確保有效的、可解析的輸出。這不是一個獨立的工具——它增強 Messages API 回應格式和 / 或工具參數驗證。

兩個功能可用：

- **JSON 輸出**（`output_config.format`）：控制 Claude 的回應格式
- **嚴格工具使用**（`strict: true`）：保證有效的工具參數綱要

**支援的模型：** Claude Fable 5、Claude Mythos 5、Claude Fable 5.1、Claude Mythos 5.1、Claude Opus 5、Claude Opus 4.8、Claude Sonnet 5 和 Claude Haiku 4.5。遺留模型（Claude Opus 4.5、Claude Opus 4.1）也支援結構化輸出。

> **推薦：** 使用 `client.messages.parse()` 自動依您的綱要驗證回應。直接使用 `messages.create()` 時，使用 `output_config: {format: {...}}`。某些 SDK 方法（例如 `.parse()`）也接受 `output_format` 便利參數，但 `output_config.format` 是 API 層級的標準參數。

### JSON 綱要限制

**支援：**

- 基本型別：object、array、string、integer、number、boolean、null
- `enum`、`const`、`anyOf`、`allOf`、`$ref`/`$def`
- 字串格式：`date-time`、`time`、`date`、`duration`、`email`、`hostname`、`uri`、`ipv4`、`ipv6`、`uuid`
- `additionalProperties: false`（所有 object 必填）

**不支援：**

- 遞迴綱要
- 數值約束（`minimum`、`maximum`、`multipleOf`）
- 字串約束（`minLength`、`maxLength`）
- 複雜的陣列約束
- `additionalProperties` 設為 `false` 以外的任何值

Python 和 TypeScript SDK 透過從傳送給 API 的綱要中移除不支援的約束並在用戶端驗證它們，自動處理不支援的約束。

### 重要注意事項

- **首次請求延遲：** 新綱要產生一次性編譯成本。後續帶有相同綱要的請求使用 24 小時快取。
- **拒絕：** 若 Claude 因安全原因拒絕（`stop_reason: "refusal"`），輸出可能不符合您的綱要。
- **Token 限制：** 若 `stop_reason: "max_tokens"`，輸出可能不完整。增加 `max_tokens`。
- **與以下不相容：** Citations（返回 400 錯誤）、訊息預填。
- **與以下相容：** Batches API、串流、token 計算、擴展思考。

---

## 有效 Tool Use 的技巧

1. **提供詳細描述：** Claude 大量依賴描述來理解何時以及如何使用工具
2. **使用具體的工具名稱：** `get_current_weather` 比 `weather` 好
3. **驗證輸入：** 在執行之前始終驗證工具輸入
4. **優雅地處理錯誤：** 返回有資訊性的錯誤訊息讓 Claude 可以適應
5. **限制工具數量：** 太多工具可能使模型困惑——保持集合精準聚焦
6. **測試工具互動：** 在各種情境中驗證 Claude 是否正確使用工具

詳細的 tool use 文件請 WebFetch：

- URL：`https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview`
