---
source_file: managed-agents-events.md
source_commit: 34040c9c568585f6929bedeaad110ad08f079624
source_sha256: cf449696835d8188b6ec27f90a24c517b47756e3f6a5f8f957760ac2a060473e
translated_at: 2026-09-13
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `managed-agents-events.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Managed Agents — 事件與引導

## 事件

### 送出事件

透過 `POST /v1/sessions/{id}/events` 向 session 送出事件。

| 事件類型 | 何時送出 |
| --- | --- |
| `user.message` | 送出使用者訊息 |
| `user.interrupt` | 在 agent 執行時中斷它 |
| `user.tool_confirmation` | 核准/拒絕已暫停等待核准的工具呼叫（`always_ask` 政策，或 `auto` 策略下伺服器未能做出判定時） |
| `user.custom_tool_result` | 提供自訂工具呼叫的結果 |
| `user.define_outcome` | 啟動評分標準導向的迭代迴圈——見 `shared/managed-agents-outcomes.md` |
| `system.message` | 為本輪次及之後的每個輪次追加特權系統層級情境；見「在 session 中途新增系統情境」 |

#### 在 session 中途新增系統情境（`system.message`）

agent 定義上的 `system` 欄位設定頂層系統 prompt，對 session 的生命週期是固定的。`system.message` 事件**追加**到 session 的系統情境作為 `role: "system"` 輪次——它不取代那個 prompt。內容套用到此輪次和所有後續輪次。用於不同的角色、修訂的限制或應影響後續行為的執行時擷取情境：

```python
client.beta.sessions.events.send(
    session.id,
    events=[
        {
            "type": "system.message",
            "content": [
                {"type": "text", "text": "The user's current timezone is America/New_York."},
            ],
        },
    ],
)
```

限制：

- **模型限制：Claude Opus 5、Claude Opus 4.8、Claude Sonnet 5、Claude Fable 5.1 和 Claude Mythos 5.1。** 只檢查 agent 的**主要**模型——`system.message` 只落在主要執行緒上，因此不考慮子 agent 模型。在不支援的主要模型上，事件被拒絕並帶有 `model_does_not_support_mid_conversation_system` 驗證錯誤。
- **當 session 因 `stop_reason: requires_action` 而閒置**（等待 `user.custom_tool_result` / `user.tool_confirmation` 阻塞）時，`system.message` 只有在**同一個請求中跟在工具結果事件後面**才被接受。單獨送出——或與 `user.message` 一起送出——會被拒絕，直到待處理的工具事件解決。
- `content` 接受 1–1000 個文字項目。

### 接收事件

三種方法：

1. **串流（SSE）**：`GET /v1/sessions/{id}/events/stream` — 即時 Server-Sent Events。**長連線**——伺服器定期發送心跳以保持連線活躍。
2. **輪詢**：`GET /v1/sessions/{id}/events` — 分頁事件列表（查詢參數：`limit` 預設 1000，`page`）。**立即回傳**——這是普通的分頁 GET，不是長輪詢。
3. **Webhook**：Anthropic 將 session 狀態轉換 POST 到您的 HTTPS 端點——精簡 payload（僅 ID）、HMAC 簽名、Console 登錄。見 `shared/managed-agents-webhooks.md`。

**免程式碼檢查——Console session 檢視器**（Console 側邊欄 -> **Managed Agents** -> **Sessions**；僅限 Developer 與 Admin）。在使用者自行解析串流之前，可先引導他們來這裡除錯：session 清單（ID、名稱、狀態、agent、輸入/輸出 token 數、成本；可依狀態/建立時間篩選、依 ID 搜尋）；多 agent session 中每個執行緒各佔一列的**時間軸縮圖(timeline minimap)**；依模型請求分組的**逐字稿(transcript)**（thinking、附帶輸入/結果的工具呼叫、串流文字），並附有 **Filter events** 篩選欄位（可比對 ID、類型、工具名稱或文字；按 Enter 可在相符項目間跳轉）與複製/下載為 JSON 功能（篩選啟用時匯出篩選後的結果）；以及可用 `d` 鍵切換的 **Inspector** 側邊面板，內含五個分頁——**Session**（詳細資訊、中繼資料、相對於預算的累計成本圖表）、**Events**（依伺服器順序排列的原始事件，每個事件皆有 JSON，另有針對頁面開啟期間串流訊息的 **Deltas** 檢視）、**Tools**（每個已設定工具的呼叫次數、失敗次數、中位數耗時；可跳至任何呼叫）、**Resources**（掛載的檔案、存放庫、含每個 session 記憶體變更的記憶體存放區、`/mnt/session/outputs` 檔案、`/workspace/skills` 下的 skill）、**Threads**（每個執行緒的狀態、情境大小、成本；目前執行緒的情境大小圖表；可切換執行緒）。以 session URL 上的 `?event={event_id}` 建立深層連結——方便連同 `shared/managed-agents-core.md` 中的 Console 連結一起納入錯誤回報。

所有**持久化**事件都帶有 `id`、`type` 和 `processed_at`（ISO 8601），在事件完成處理時設定。在您送出的事件上，`processed_at` 在事件仍排在較早事件後面等待時為 `null`——**除了** `user.define_outcome`、`user.custom_tool_result` 和 `user.tool_result`，這些在收到時就被處理並帶有已填寫的 `processed_at` 回顯。僅限串流的 `event_start` / `event_delta` 預覽事件（見「即時預覽」）只帶有它們所預覽事件的 `id`。

> 警告：**健壯的輪詢（原始 HTTP）。** 若您繞過 SDK 並自己實作輪詢迴圈，不要依賴 `requests` 或 `httpx` 逾時作為掛鐘上限——它們是**每個區塊**的讀取逾時，每次有位元組到達時重置。緩慢滴水的回應（心跳、wedge 的分塊編碼主體、行為異常的 proxy）即使設定了 `timeout=(5, 60)` 或 `httpx.Timeout(120)` 也能讓呼叫無限期阻塞。兩個函式庫都沒有內建「總掛鐘」逾時。對於硬性截止時間：在迴圈層級追蹤 `time.monotonic()` 並在單一請求超過您的預算時中斷/取消（例如透過監看執行緒，或在非同步 httpx 外圍使用 `asyncio.wait_for()`）。**優先使用 SDK** ——`client.beta.sessions.events.stream()` 和 `client.beta.sessions.events.list()` 能合理地處理逾時 + 重試。
>
> 若 `GET /v1/sessions/{id}/events`（分頁）在 headers 之後掛起，您可能誤打了 `GET /v1/sessions/{id}/events/stream` 或發生了伺服器端停滯——回報它；不要視為客戶端設定問題。

### 事件類型（接收）

事件類型使用點記法，依命名空間分組：

| 事件類型 | 說明 |
| --- | --- |
| `agent.message` | Agent 文字輸出 |
| `agent.thinking` | Agent 正在思考的進度訊號——它**不**帶有思維內容 |
| `agent.tool_use` | Agent 使用了內建工具（`agent_toolset_20260401`）。帶有 `evaluated_permission`（`allow`/`ask`/`deny`）與通常存在的 `evaluation`——見 `shared/managed-agents-tools.md` §`evaluated_permission` 與 `evaluation` |
| `agent.tool_result` | 內建工具的結果 |
| `agent.mcp_tool_use` | Agent 使用了 MCP 工具。帶有 `evaluated_permission` 與通常存在的 `evaluation`，與 `agent.tool_use` 相同 |
| `agent.mcp_tool_result` | MCP 工具的結果 |
| `agent.custom_tool_use` | Agent 呼叫了自訂工具——session 進入閒置，您以 `user.custom_tool_result` 回應 |
| `agent.thread_context_compacted` | 對話情境已壓縮 |
| `session.status_idle` | Agent 已完成當前任務，正在等待輸入。它要麼在等待繼續工作的 `user.message`，要麼等待 `user.custom_tool_result` 或 `user.tool_confirmation` 而阻塞，要麼因為 session 預算上限已達到而暫停。附帶的 `stop_reason` 包含有關 Agent 為何停止工作的更多資訊。 |
| `session.status_running` | Session 已開始執行，Agent 正在積極工作。 |
| `session.status_rescheduled` | Session 在可重試錯誤發生後重新排程，準備被協作系統接手。 |
| `session.status_terminated` | Session 結束且不可逆地無法使用——**完成或錯誤時**，而非僅在錯誤時。 |
| `session.updated` | Session 更新更改了至少一個欄位——只帶有更改的欄位（移除預算的事件帶有 `budget: null`） |
| `session.usage` | Session 累計使用量和追蹤列表成本的快照——見「達到 session 預算」 |
| `session.error` | 處理過程中發生錯誤 |
| `span.model_request_start` | 模型推理開始 |
| `span.model_request_end` | 模型推理完成 |
| `span.outcome_evaluation_start` / `_ongoing` / `_end` | outcome 導向 session 的評分器進度——見 `shared/managed-agents-outcomes.md` |
| `session.thread_created` | 子 agent 執行緒生成（多 agent），或 advisor 諮詢開始（執行緒名稱 `anthropic.advisor`）——見 `shared/managed-agents-multiagent.md` |
| `session.thread_status_running` / `_idle` / `_rescheduled` / `_terminated` | 執行緒狀態轉換——主要在多 agent session 中出現，但單一 agent session 的主執行緒在因 session 預算暫停時也會發出 `_idle`（見「達到 session 預算」）。`_idle` 帶有 `stop_reason`。 |
| `agent.thread_message_sent` / `_received` | 跨執行緒訊息，帶有 `to_session_thread_id` / `from_session_thread_id`（多 agent） |

串流也會回顯使用者送出的事件（`user.message`、`user.interrupt`、`user.tool_confirmation`、`user.tool_result`、`user.custom_tool_result`、`user.define_outcome`）——在 session 因預算暫停時送出的 `user.interrupt` 除外，它被接受並忽略，永不出現（見「達到 session 預算」）。

僅限串流的 delta 預覽事件（`event_start`、`event_delta`）是 `{domain}.{action}` 命名慣例的唯一例外——見下方「即時預覽」；它們從不出現在 `GET /v1/sessions/{id}/events` 中。

---

## 即時預覽

預設情況下，assistant 文字以緩衝的 `agent.message` 事件到達串流——只有在產生它們的模型請求完成後才發出。**即時預覽**讓您在模型仍在生成時以增量方式渲染該文字。緩衝的 `agent.message` 始終是權威記錄；忽略預覽的客戶端仍會收到完整、正確的串流。線路格式**不是** Messages API 串流：delta 類型是 `content_delta`，而非 `content_block_delta`，因此 Messages API 累加器程式碼不能直接移植。

**每個串流連線選擇加入**，透過新增 `event_deltas[]` 查詢參數，每個要預覽的事件類型重複一次。接受的值：`agent.message`、`agent.thinking`——任何其他值回傳 400，超過 100 個值的請求也是如此。**兩個串流端點都接受它：** session 層級串流（`GET /v1/sessions/{id}/events/stream`）和每個 session 執行緒自己的串流（`GET /v1/sessions/{sid}/threads/{tid}/stream`）。在 shell 中，引號括住 URL 或將括號百分比編碼為 `%5B%5D`——裸 `[]` 是 glob 模式。

**預覽是以執行緒為範圍的。** 連線只預覽它所讀取的執行緒。子執行緒的預覽在那個子執行緒的串流上傳送，*永不*跨發到 session 層級串流（其預覽範圍僅限主執行緒）。若要在模型生成時觀察子 agent 的文字，請開啟那個子 agent 的執行緒串流——見 `shared/managed-agents-multiagent.md`。每個連線執行一個累加器實例。

```python
stream = client.beta.sessions.events.stream(
    session_id=session.id,
    event_deltas=["agent.message"],
)
```

當預覽事件開始時，串流發出帶有即將到來事件的 `type` 和 `id` 的 `event_start`；對於 `agent.message`，它後面跟著帶有增量文字的 `event_delta` 事件：

```json
{"type": "event_start", "event": {"type": "agent.message", "id": "sevt_01abc..."}}
{"type": "event_delta", "event_id": "sevt_01abc...", "delta": {"type": "content_delta", "index": 0, "content": {"type": "text", "text": "Here is the summary"}}}
```

`event_start` 和 `event_delta` 沒有自己的 `id` 或 `processed_at`——它們帶有的唯一識別碼是它們所預覽事件的 `id`。對於 `agent.thinking`，**只**發出 `event_start`（「thinking 已開始」訊號）——不跟隨 delta，結束預覽的緩衝 `agent.thinking` 也不帶有思維內容。它是進度訊號，不是內容載體；沒有可讀出的內容。

**累積並核對模式。** 將預覽視為以 `(event_id, index)` 為鍵的暫存緩衝區。在 `event_start` 時，為宣告的 `id` 建立一個空條目。在每個 `event_delta` 時，將 `delta.content.text` 追加到 `(event_id, delta.index)` 並渲染執行中的文字。當緩衝的 `agent.message` 到達時，依 `id` 比對，**丟棄累積的預覽**，並改為渲染訊息的內容。識別碼始終一致：`event_start.event.id`、每個 `event_delta.event_id` 和緩衝事件的 `id` 是相同的值。在正常輪次中順序是固定的：`session.status_running` → `span.model_request_start` → `event_start` → `event_delta`* → 緩衝的 `agent.message` → `span.model_request_end`。若輪次出錯或被中斷，緩衝的事件可能永不到達，但 `span.model_request_end` 仍然到達——當您看到它時關閉所有未核對的預覽。Python/TypeScript/Go SDK 附帶實作此功能的累加器輔助工具；在其他 SDK 中，將手動模式套用到生成的事件類型。

**該模式依賴的兩個保證：** 按到達順序、以 `(event_id, index)` 為鍵串接預覽的 delta，產生緩衝事件中 `content[index].text` 的*前綴*（是前綴，不一定是全文——delta 在負載下可能被丟棄）；以及連線每個 `event_id` 最多發出一個 `event_start`，緩衝事件是那個連線為那個 `id` 傳送的最後一個東西。

**限制：**
- **盡力而為** — 在負載下伺服器可能丟棄事件的 delta；您接收到連續前綴後不再有那個事件的 delta。緩衝的 `agent.message` 仍然完整到達。永遠不要將累積的預覽視為最終狀態。
- **重連時沒有重播** — delta 只傳送到選擇加入的連線，且僅在連線開啟期間；這對 session 層級串流和每個執行緒串流同樣適用。在模型請求開始後才開啟的連線不會收到那個進行中事件的 delta。中斷後，請遵循「串流中斷後重連」中的整合模式——歷史擷取回傳在間隙期間發出的所有緩衝事件；錯過的 delta 無法重新請求。
- **一個執行緒，僅限文字** — 預覽涵蓋連線正在讀取的執行緒上的 assistant 文字。工具使用、工具結果、MCP 結果和*任何其他*執行緒上的活動永遠不會在那個連線上被預覽。
- **永不持久化** — `event_start` / `event_delta` 只存在於即時 SSE 串流上，從不在 `GET /v1/sessions/{id}/events` 或任何執行緒的事件歷史中。

**疑難排解：**

| 您看到 | 它的意義 |
| --- | --- |
| 緩衝事件但沒有 `event_start` / `event_delta` | 這個連線沒有選擇加入（`event_deltas[]` 是每個連線的，不是每個 session 的），或輪次在不同的執行緒上執行。列出 `GET /v1/sessions/{sid}/threads` 以找出是哪個。 |
| 串流 URL 上的 404 | 錯誤的路徑或 ID，或請求沒有帶 managed-agents beta header——執行緒端點是 beta 限制的，沒有它它們就不存在。執行緒路徑是 `/threads/{tid}/stream`，**不是** `/threads/{tid}/events/stream`（不存在）也不是 `/events/stream`（僅 session 層級）。 |
| 命名 `event_deltas` 的 400 | 只接受 `agent.message` 和 `agent.thinking`，最多 100 個值。 |

---

## 引導模式

透過事件介面驅動 session 的實用模式。

### 串流優先排序

**在送出事件之前開啟串流。** 串流只傳送在它開啟*之後*發生的事件——它不重播當前狀態或歷史事件。若您先送出訊息再開啟串流，早期事件（包含快速狀態轉換）會以單一批次緩衝到達，您失去了即時回應它們的能力。

```ts
// Correct - stream and send concurrently
const [response] = await Promise.all([
  streamEvents(sessionId),   // opens SSE connection
  sendMessage(sessionId, text),
]);

// Wrong - events before stream opens arrive as a single buffered batch
await sendMessage(sessionId, text);
const response = await streamEvents(sessionId);
```

**如需完整歷史**，請使用 `GET /v1/sessions/{id}/events`（分頁列表）——串流只提供從連線開始的即時事件。

### 串流中斷後重連

**SSE 串流沒有重播。** 若您的連線中斷（httpx 讀取逾時、網路問題）並重連，您只會收到重連*之後*發出的事件。在間隙期間發出的任何事件都從串流中丟失。

**整合模式：** 在每次（重新）連線時，將串流與歷史擷取重疊並依事件 ID 去重：

```python
def connect_with_consolidation(client, session_id):
    # 1. Open the SSE stream first
    stream = client.beta.sessions.events.stream(session_id=session_id)

    # 2. Fetch history to cover any gap
    history = client.beta.sessions.events.list(
        session_id=session_id,
    )

    # 3. Yield history first, then stream - dedupe by event.id
    seen = set()
    for ev in history.data:
        seen.add(ev.id)
        yield ev
    for ev in stream:
        if ev.id not in seen:
            seen.add(ev.id)
            yield ev
```

### 訊息佇列

**不需要等待回應再送下一條訊息。** 使用者事件在伺服器端排隊並按順序處理。這對使用者快速傳送後續訊息的聊天橋接器很有用：

```ts
// All three go into one session; agent processes them in order
await sendMessage(sessionId, "Summarize the README");
await sendMessage(sessionId, "Actually also check the CONTRIBUTING guide");
await sendMessage(sessionId, "And compare the two");
// Stream once - agent responds to all three as a coherent turn
```

可以隨時向 Session 送出事件。無需等待特定的 session 狀態才能透過 `client.beta.sessions.events.send()` 排入新事件。一個例外：因預算暫停的 session（`stop_reason: budget_reached`）只接受結算事件——`user.message` 在那裡是 400。見「達到 session 預算」。

### 中斷

`user.interrupt` 事件**跳過佇列**（在任何待處理的使用者訊息之前）並強制 session 進入 `idle`。例外：當 session 因預算暫停時，中斷被接受並忽略——它永不被持久化且不改變任何事（見「達到 session 預算」）。用於「停止」/「算了」/「取消」命令：

```ts
await client.beta.sessions.events.send(sessionId, {
  events: [{ type: 'user.interrupt' }],
});
```

Agent 在任務中途停止。它不會將中斷視為訊息——它只是停止。送出後續 `user` 事件來說明改做什麼。若 outcome 處於活躍狀態，中斷也會將 `span.outcome_evaluation_end.result` 標記為 `"interrupted"`（見 `shared/managed-agents-outcomes.md`）——雖然在預算暫停時不會，因為中斷在那裡被接受並忽略（見「達到 session 預算」）。

**被中斷的輪次以 `stop_reason: end_turn` 結束**——與自行完成的輪次帶有的值相同。沒有中斷特定的 stop reason，因此排空迴圈無法僅從 `stop_reason` 區分兩者；請追蹤您送出了中斷。

**對一個已經 `idle` 的 session 送出中斷，通常是無操作（no-op）。** 例外是自託管環境上的 session，其 worker 未能完成已認領的工作項目（例如記憶體存放區掛載錯誤）：它會停在 `idle` 且 `stop_reason: requires_action`，沒有錯誤事件，此時 `user.interrupt` 會將該工作重新排入佇列，供下一次 worker 認領時重試（`shared/managed-agents-self-hosted-sandboxes.md` § 記憶體存放區 → 疑難排解）。

**在多 agent session 中，省略 `session_thread_id` 中斷每個未封存的執行緒，包括主執行緒**——它不是僅限主執行緒的。傳入 `session_thread_id` 以停止一個執行緒。見 `shared/managed-agents-multiagent.md`。

> **注意**：在目前的實作中，中斷事件可能有空的 ID。疑難排解時，請使用 `processed_at` 時間戳記和周圍事件 ID。（不適用於在預算上限時送出的中斷——那個事件永不被持久化，因此沒有可定位的東西。）

### 達到 session 預算

使用預算建立的 session（見 `shared/managed-agents-core.md` § Session 預算）暫停而非超支。在每個模型請求之前，平台檢查消耗的列表成本是否已達到上限，若有則暫停執行緒，session 進入 `stop_reason: budget_reached` 的閒置而非終止。在串流上，暫停按序到達三個事件：

1. `session.thread_status_idle`，帶有 `stop_reason: budget_reached`，對每個執行緒在它暫停時。當執行緒的最後一個請求同時跨越上限並完成其輪次時，那個執行緒報告 `stop_reason: end_turn`，而 session 仍報告 `budget_reached`——以 **session 層級**的 `stop_reason` 為鍵（而非執行緒層級的）來偵測暫停。
2. `session.usage` — session 累計使用量和追蹤列表成本的快照。
3. `session.status_idle`，帶有 `stop_reason: budget_reached`。`session.usage` 事件總是緊接在這個閒置之前。

在達到上限時，session 只接受**結算事件**（`user.tool_confirmation`、`user.tool_result`、`user.custom_tool_result`、`user.interrupt`）；任何啟動新工作的事件（包含 `user.message`）都是 400，並列出那個清單。在 session 因預算暫停時（所有執行緒都在上限暫停）送出的 `user.interrupt` 被接受並忽略：它不出現在事件列表中，也不改變任何事。提高或移除預算以繼續。當一個執行緒在等待工具詢問，而另一個在上限暫停時，session 層級的 `stop_reason` 是 `requires_action`，而非 `budget_reached`——結算詢問不觸發模型請求，因此照常回應。

**沒有任何事件可以恢復因上限暫停的 session。** 改為更新 session 的預算：將其更改到高於消耗的列表成本的值（高於或低於舊的上限），或用 `"budget": null` 移除它。接受的更新自動恢復暫停的工作。

**`session.usage`** 帶有 session 的累計 token 總數、`list_cost`（`{amount, currency}`，四捨五入到最近的分）、`active_seconds`（並發執行緒重疊計算一次——執行時間成本定價的數字）、`server_tool_use` 計數（`web_search_requests` 和 `web_fetch_requests`——資訊性的，目前始終為 0，因為 web fetch 未計費）以及設定了預算時 session 的 `budget` 回顯。它出現在事件列表和 session 串流中——串流讀者無需額外擷取就能看到達到上限的工作的最終成本；子執行緒自己的串流不帶有它。相同的總數在 session 物件的 `usage` 欄位中，每個執行緒自己的 `usage` 帶有每個執行緒的 `list_cost` 和 `active_seconds`——但每個執行緒的成本**不**加總為 session 總計：session 數字另外包含 session 執行時間，且每個數字都獨立四捨五入，因此 session 數字是權威的。若要強制執行消費限制，請設定預算而非輪詢使用量並自行中斷 session——平台的閘道在每個模型請求之前執行。

### 事件 payload

某些事件帶有超出狀態變更本身的有用中繼資料：

`session.status_idle` — 包含 `stop_reason` 欄位，詳細說明 session 停止的原因以及使用者需要採取的進一步行動類型。
```json
{
  "id": "sevt_456",
  "processed_at": "2026-04-07T04:27:43.197Z",
  "stop_reason": {
    "event_ids": [
      "sevt_123"
    ],
    "type": "requires_action"
  },
  "type": "status_idle"
}
```

`span.model_request_end` 包含用於成本追蹤和效率分析的 `model_usage` 欄位：

```json
{
  "type": "span.model_request_end",
  "id": "sevt_456",
  "is_error": false,
  "model_request_start_id": "sevt_123",
  "model_usage": {
    "cache_creation_input_tokens": 0,
    "cache_read_input_tokens": 6656,
    "input_tokens": 3571,
    "output_tokens": 727
  },
  "processed_at": "2026-04-07T04:11:32.189Z"
}
```

**`agent.thread_context_compacted`** — 當對話歷史被摘要以符合情境時發出。包含 `pre_compaction_tokens` 以便您知道壓縮了多少：

```json
{
  "id": "sevt_abc123",
  "processed_at": "2026-03-24T14:05:15.787Z",
  "type": "agent.thread_context_compacted"
}
```

### 封存

處理完 session 後，封存它以釋放資源：

```ts
await client.beta.sessions.archive(sessionId);
```

> 封存 **session** 是例行清理——session 是每次執行的，是可丟棄的。**不要將此推廣到 agent 或環境**：那些是持久的、可重複使用的資源，封存它們是永久的（無法取消封存；新 session 無法引用它們）。見 `shared/managed-agents-overview.md` → 常見陷阱。
