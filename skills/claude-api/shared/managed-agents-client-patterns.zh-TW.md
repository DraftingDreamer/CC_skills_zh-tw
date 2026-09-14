---
source_file: managed-agents-client-patterns.md
source_commit: 34040c9c568585f6929bedeaad110ad08f079624
source_sha256: f3d752675e225892f97da4d7bffbefd430841d6655b2c53a926f212121759d55
translated_at: 2026-09-13
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `managed-agents-client-patterns.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Managed Agents — 常見客戶端模式

在驅動 Managed Agent session 時客戶端需要實作的模式，以可運行的 SDK 範例為基礎。

程式碼範例使用 TypeScript — 其他語言遵循相同的形狀；等效程式碼請參見 `{lang}/managed-agents/README.md`（cURL 和 C#：`curl/managed-agents.md`）。

---

## 1. 無損串流重連

**問題：** SSE 沒有重播功能。若連線在 session 進行中斷開，單純的重連會從「現在」重新開啟串流，您會靜默遺漏中間發出的每個事件。

**解決方案：** 在重連時，*先*透過 `events.list()` 擷取完整事件歷史，再消費即時串流，並在即時串流追上時對事件 ID 進行去重。

```ts
const seenEventIds = new Set<string>()
const stream = await client.beta.sessions.events.stream(session.id)

// Stream is now open and buffering server-side. Read history first.
for await (const event of client.beta.sessions.events.list(session.id)) {
  seenEventIds.add(event.id)
  handle(event)
}

// Tail the live stream. Dedupe only gates handle() - terminal checks must run
// even for already-seen events, or a terminal event that was in the history
// response gets skipped by `continue` and the loop never exits.
for await (const event of stream) {
  if (!seenEventIds.has(event.id)) {
    seenEventIds.add(event.id)
    handle(event)
  }
  if (event.type === 'session.status_terminated') break
  if (event.type === 'session.status_idle' && event.stop_reason.type !== 'requires_action') break
}
```

---

## 2. `processed_at` — 已排隊 vs 已處理

串流上的每個事件都帶有 `processed_at`（ISO 8601），在事件完成處理時設定。對於客戶端送出的事件（`user.message`、`user.interrupt`、`user.tool_confirmation`），在事件排隊等待前面的事件時為 `null`，在 agent 處理後才填入——因此同一事件在串流上出現兩次，一次帶有 `null`，一次帶有時間戳。（例外：在 session 因預算暫停時送出的 `user.interrupt` 會被接受並忽略——它根本不會出現；請參見 `shared/managed-agents-events.md` § 達到 session 預算。）

**三種事件類型跳過排隊階段：** `user.define_outcome`、`user.custom_tool_result` 和 `user.tool_result` 在收到時即被處理，並帶有已填入的 `processed_at` 回傳。假設「第一次出現永遠是 `null`」的待處理 → 已確認 UI 對這些事件永遠不會清除——將第一次出現就帶有填入的 `processed_at` 視為立即確認。

```ts
for await (const event of stream) {
  if (event.type === 'user.message') {
    if (event.processed_at == null) onQueued(event.id)
    else onProcessed(event.id, event.processed_at)
  }
}
```

用於驅動您送出的任何內容的待處理 → 已確認 UI 狀態。如何將本地渲染的樂觀訊息對應到伺服器指派的 `event.id`，這是特定應用程式的事（通常透過 `events.send()` 的回傳值或 FIFO 排序）。

---

## 3. 中斷執行中的 session

將 `user.interrupt` 作為普通事件送出。Session 繼續執行直到到達安全邊界，然後進入 idle。

```ts
await client.beta.sessions.events.send(session.id, {
  events: [{ type: 'user.interrupt' }],
})

// Drain until the session is truly done - see Pattern 5 for the full gate.
for await (const event of stream) {
  if (event.type === 'session.status_terminated') break
  if (
    event.type === 'session.status_idle' &&
    event.stop_reason.type !== 'requires_action'
  ) break
}
```

參考：`interrupt.ts` — 在看到 `span.model_request_start` 時立即送出中斷，耗盡到 idle，然後透過 `sessions.retrieve()` 驗證。

---

## 4. `tool_confirmation` 往返

當呼叫評估為 `ask`——工具有 `permission_policy: { type: 'always_ask' }`，或有 `{ type: 'auto' }` 且伺服器未能做出判定——`agent.tool_use` / `agent.mcp_tool_use` 事件帶有 `evaluated_permission === 'ask'`，session 進入 idle 等待決定。以 `user.tool_confirmation` 回應。

```ts
for await (const event of stream) {
  if ((event.type === 'agent.tool_use' || event.type === 'agent.mcp_tool_use') && event.evaluated_permission === 'ask') {
    await client.beta.sessions.events.send(session.id, {
      events: [{
        type: 'user.tool_confirmation',
        tool_use_id: event.id,         // not a toolu_ id - use event.id
        result: 'allow',               // or 'deny'
        // deny_message: '...',        // optional, only with result: 'deny'
      }],
    })
  }
}
```

關鍵點：
- `tool_use_id` 是 `event.id`（通常是 `sevt_...`），**不是** `toolu_...` ID。
- `result` 是 `'allow' | 'deny'`。使用 `deny_message` 告知模型您拒絕的*原因*——它會回傳給 agent。
- 多個待處理工具：對每個帶有 `evaluated_permission === 'ask'` 的 `agent.tool_use` / `agent.mcp_tool_use` 事件各回應一次。
- 以 `evaluated_permission === 'ask'` 作為判斷依據，而非您所設定的政策——它同時涵蓋 `always_ask` 和 `auto` 未能判定的情況。伺服器在 `auto` 策略下**拒絕**的呼叫（`evaluated_permission === 'deny'`，`evaluation.evaluated_permission.reason_code === 'high_risk'`）永遠不會進入此流程：agent 會收到錯誤工具結果，session 繼續執行；為此類呼叫送出確認會得到 400。
- 記錄 `event.evaluation` 以供稽核（`type` + `reason_code`），並容忍未能識別的 `type` 或 `reason_code`——對已知值分支處理，對未知值直接傳遞。

參考：`tool-permissions.ts`。

---

## 5. 正確的 idle 中斷閘門

不要單獨在 `session.status_idle` 時中斷。Session 會短暫進入 idle — 例如在並行工具執行之間、等待 `user.tool_confirmation` 時，或等待 `user.custom_tool_result` 時。當 idle 且 `stop_reason` 不是 `requires_action` 時中斷（終止狀態，或 `budget_reached` — 只有預算更新才能恢復，因此除非您打算變更或移除預算，否則中斷），或在 `session.status_terminated` 時中斷。

```ts
for await (const event of stream) {
  handle(event)
  if (event.type === 'session.status_terminated') break
  if (event.type === 'session.status_idle') {
    if (event.stop_reason.type === 'requires_action') continue // waiting on you - handle it
    break // end_turn, retries_exhausted, or budget_reached - see list below
  }
}
```

`session.status_idle` 上的 `stop_reason.type` 值：
- `requires_action` — agent 在等待客戶端事件（工具確認、自訂工具結果）。處理它，不要中斷。**自託管例外：** 若 session 進入 `requires_action`-idle，卻沒有待處理的 `agent.tool_use` / `agent.mcp_tool_use`（`ask`）或 `agent.custom_tool_use` 可供回應，代表 worker 未能完成已認領的工作項目（通常是記憶體存放區掛載錯誤，僅記錄在 worker 主機上）。不要對此無限期 `continue`——請讓錯誤浮現、修復主機，並送出 `user.interrupt` 將工作重新排入佇列（`shared/managed-agents-self-hosted-sandboxes.md` § 記憶體存放區 → 疑難排解）。
- `retries_exhausted` — 終止失敗。中斷，然後查看 `sessions.retrieve()` 的錯誤狀態。
- `end_turn` — 正常完成。
- `budget_reached` — session 達到其消費上限並暫停。非終止且不可由任何事件恢復：變更（通常是提高）或移除 session 的 `budget` 以恢復，或將其視為完成。緊接在此 idle 之前會有帶有最終費用的 `session.usage` 事件。請參見 `shared/managed-agents-core.md` § Session 預算。

---

## 6. idle 後的狀態寫入競爭

SSE 串流在 session 的可查詢狀態反映之前，稍早發出 `session.status_idle`。在 idle 時中斷並立即呼叫 `sessions.delete()` 或 `sessions.archive()` 的客戶端，會偶發性地收到 400 錯誤「cannot delete/archive while running」。

清理前先輪詢：

```ts
let s
for (let i = 0; i < 10; i++) {
  s = await client.beta.sessions.retrieve(session.id)
  if (s.status !== 'running') break
  await new Promise(r => setTimeout(r, 200))
}
if (s?.status !== 'running') {
  await client.beta.sessions.archive(session.id)
} // else: still running after 2s - don't archive, let it settle or escalate
```

---

## 7. 先開串流，後送事件

永遠在送出啟動事件**之前**開啟串流。否則 agent 可能在您的消費者附加之前處理事件並發出最初的事件，您就會遺漏它們。

```ts
const stream = await client.beta.sessions.events.stream(session.id)
await client.beta.sessions.events.send(session.id, {
  events: [{ type: 'user.message', content: [{ type: 'text', text: 'Hello' }] }],
})
for await (const event of stream) { /* ... */ }
```

`Promise.all([stream, send])` 的形式也可以，但先開串流更簡單且效果相同——串流在開啟的瞬間就開始緩衝。

---

## 8. 檔案掛載注意事項

**掛載的資源有與您上傳的檔案不同的 `file_id`。** Session 建立時會製作 session 範圍的副本。

```ts
const uploaded = await client.beta.files.upload({ file, purpose: 'agent_resource' })
// uploaded.id         -> the original file
const session = await client.beta.sessions.create({
  /* ... */
  resources: [{ type: 'file', file_id: uploaded.id, mount_path: '/workspace/data.csv' }],
})
// session.resources[0].file_id !== uploaded.id  <- different IDs
```

透過 `files.delete(uploaded.id)` 刪除原始檔案；session 範圍的副本會隨 session 垃圾回收。`mount_path` 必須是絕對路徑——請參見 `shared/managed-agents-environments.md`。

---

## 9. 非 MCP API 和 CLI 的密鑰——透過自訂工具保留在主機端

**問題：** 您希望 agent 呼叫需要密鑰（API 金鑰、token、服務帳戶憑證）的第三方 API 或執行 CLI，但您無法或不想將密鑰交給 vault。

**先查：** 對於雲端環境，現在的第一選擇是 vault `environment_variable` 憑證——agent 的 shell 看到的是不透明的佔位符，真正的密鑰在出口處被替換。請參見 `shared/managed-agents-tools.md` → Vault。在以下情況時使用此模式替代：**自託管沙盒**（環境變數憑證尚不支援）、因本地格式驗證拒絕佔位符的客戶端、絕不能離開您基礎設施的密鑰，或需要主機端二進位檔的呼叫。

**解決方案：** 將已驗證的呼叫移到您這端。在 agent 上宣告自訂工具；當 agent 發出 `agent.custom_tool_use` 時，您的協作器（讀取 SSE 串流的處理程序）以自己的憑證執行呼叫，並以 `user.custom_tool_result` 回應。容器永遠看不到金鑰。

```ts
// Agent template: declare the tool, no credentials
tools: [{ type: 'custom', name: 'linear_graphql', input_schema: { /* query, vars */ } }]

// Orchestrator: handle the call with host-side creds
for await (const event of stream) {
  if (event.type === 'agent.custom_tool_use' && event.name === 'linear_graphql') {
    const result = await linear.request(event.input.query, event.input.vars) // host's key
    await client.beta.sessions.events.send(session.id, {
      events: [{
        type: 'user.custom_tool_result',
        custom_tool_use_id: event.id,
        content: [{ type: 'text', text: JSON.stringify(result) }],
      }],
    })
  }
}
```

相同的形狀適用於 `gh` CLI、本地 eval 指令碼，或任何其他需要主機端驗證或二進位檔的工具。

**安全說明：** 這不會暴露公開端點。`agent.custom_tool_use` 到達您的協作器已用您的 Anthropic API 金鑰持有開啟的 SSE 串流，而 `user.custom_tool_result` 在相同金鑰下透過 `events.send()` 回傳。您的協作器是客戶端，不是伺服器——沒有未驗證的監聽。

**不要將 API 金鑰嵌入系統 prompt 或使用者訊息作為因應方法。** Prompt 和訊息儲存在 session 的事件歷史中，由 `events.list()` 回傳，並包含在壓縮摘要中——放在那裡的密鑰會在 session 的生命週期內持久保存並可透過 API 讀取。
