---
source_file: managed-agents-webhooks.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 38746f77a3ee971f9b1f7d47cec9e753131bf271fd795aed5c3da1764c9a6ac1
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `managed-agents-webhooks.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Managed Agents — Webhook

Anthropic 可在 Managed Agents 資源狀態變更時，對您的 HTTPS 端點發送 POST 請求——作為持有 SSE 串流或輪詢的替代方案。Payload 是**精簡的**（僅含事件類型 + 資源 ID）；收到後請擷取資源以取得目前狀態。每次傳送都有 HMAC 簽名。

> **方向很重要。** 本頁說明的是 *Anthropic → 您* 的 session/vault 狀態通知。它**不**涵蓋*第三方 → 您*的 webhook（例如觸發 session 的 GitHub push 處理器呼叫 `sessions.create()`）——那是您這端的普通應用程式碼，沒有 Anthropic 特定的線路格式。

---

## 登錄端點（僅限 Console）

Console → **管理 → Webhook**。目前尚無程式化的端點管理 API。可在同一頁面進行密鑰輪換。

| 欄位 | 限制 |
|---|---|
| URL | 連接埠 443 的 HTTPS，公開可解析的主機名稱 |
| 事件類型 | 依 `data.type` 訂閱——端點只接收它訂閱的類型 |
| 簽名密鑰 | `whsec_` 前綴，32 位元組，**建立時僅顯示一次**——請儲存 |

---

## 驗證簽名

每次傳送都帶有 `webhook-id`、`webhook-timestamp` 和 `webhook-signature` headers。**使用 SDK 的 `client.beta.webhooks.unwrap()`** — 它驗證簽名、拒絕超過約 5 分鐘的 payload，並回傳已解析的事件。它從 `ANTHROPIC_WEBHOOK_SIGNING_KEY` 讀取 `whsec_` 密鑰。請原封不動地傳遞 headers；不要針對單一 `X-Webhook-Signature` header 手工驗證，那不是線路格式。

```python
import anthropic
from flask import Flask, request

client = anthropic.Anthropic()  # reads ANTHROPIC_WEBHOOK_SIGNING_KEY from env
app = Flask(__name__)


@app.route("/webhook", methods=["POST"])
def webhook():
    try:
        event = client.beta.webhooks.unwrap(
            request.get_data(as_text=True),
            headers=dict(request.headers),
        )
    except Exception:
        return "invalid signature", 400

    if event.id in seen_event_ids:  # dedupe retries - id is per-event, not per-delivery
        return "", 204
    seen_event_ids.add(event.id)

    match event.data.type:
        case "session.status_idled":
            session = client.beta.sessions.retrieve(event.data.id)
            notify_user(session)
        case "vault_credential.refresh_failed":
            alert_oncall(event.data.id)

    return "", 204
```

將**原始請求主體**傳遞給 `unwrap()` — 重新序列化 JSON 的框架（Express `.json()`、Flask `.get_json()`）會改變位元組並破壞 MAC。其他語言請查閱 SDK 存放庫中的 `beta.webhooks.unwrap` 繫結（`shared/live-sources.md`）；不要手工驗證。

---

## Payload 信封

```json
{
  "type": "event",
  "id": "whe_9d5c1f7e...",
  "created_at": "2026-03-18T14:05:22Z",
  "data": {
    "type": "session.status_idled",
    "id": "session_01XYZ...",
    "organization_id": "8a3d2f1e-...",
    "workspace_id": "c7b0e4d9-..."
  }
}
```

依 `data.type` 分流，以 `data.id` 擷取資源，回傳任何 **2xx** 確認收到。`created_at` 是*事件發生*的時間，而非傳送嘗試的時間——`webhook-timestamp` header 是嘗試傳送時的時鐘（見傳送行為）。

頂層 `id` 與 `webhook-id` header 相同，它是按*事件*計算的，而非按傳送計算——每次重試都帶著相同的值。以它進行去重。

---

## 支援的 `data.type` 值

| `data.type` | 觸發時機 |
|---|---|
| `session.status_scheduled` | Session 已建立並準備好接受事件 |
| `session.status_run_started` | Agent 執行開始（每次轉換到 `running` 時） |
| `session.status_idled` | Agent 等待輸入（工具審批、自訂工具結果或下一條訊息）——或因 session 預算暫停。webhook payload 是精簡的——請列出 session 的事件並查看最新的 `session.status_idle` 事件的 `stop_reason`（session 物件本身沒有 `stop_reason` 欄位）：若為 `budget_reached`，進一步的 `user.message` 事件會回傳 400，只有預算變更/移除才能恢復 session（`shared/managed-agents-core.md` § Session 預算） |
| `session.status_rescheduled` | 發生暫時性錯誤；session 正在自動重試 |
| `session.status_terminated` | Session 結束——**完成或錯誤時**，而非僅在錯誤時 |
| `session.thread_created` | 多 agent：協調者開啟了新的子 agent 執行緒，或正在諮詢 session 的 advisor（`shared/managed-agents-multiagent.md` → Advisor） |
| `session.thread_idled` | 僅限子執行緒：子 agent 執行緒在等待輸入——或因 session 達到預算上限而暫停。當整個 session 因上限暫停時，也會觸發 `session.status_idled` webhook，且串流的 `session.status_idle` 事件帶有 `stop_reason: budget_reached`——除非另一個執行緒在等待工具詢問（在 session 層級工具詢問優先於預算上限）（`shared/managed-agents-core.md` § Session 預算）。 |
| `session.thread_terminated` | 執行緒結束——子 agent 完成工作，或執行緒被封存。**僅限子執行緒**；主執行緒的結束以 `session.status_terminated` 呈現 |
| `session.outcome_evaluation_ended` | Outcome 評分器完成一次迭代 |
| `session.updated` | Session 屬性變更（名稱、設定） |
| `session.deleted` | Session 永久刪除——沒有留下可擷取的物件；將事件本身視為最終狀態 |
| `vault.archived` | Vault 已封存 |
| `vault.created` | Vault 已建立 |
| `vault.deleted` | Vault 已刪除——每個底層憑證也會觸發 `vault_credential.deleted`。沒有留下可擷取的物件；將事件本身視為最終狀態 |
| `vault_credential.archived` | 憑證已封存（直接或透過 vault 封存） |
| `vault_credential.created` | Vault 憑證已建立 |
| `vault_credential.deleted` | 憑證已刪除（直接或透過 vault 刪除）。沒有留下可擷取的物件；將事件本身視為最終狀態 |
| `vault_credential.refresh_failed` | MCP OAuth vault 憑證更新失敗 |
| `agent.created` | Agent 已建立 |
| `agent.updated` | 已發布新的 agent 版本。不建立新版本的更新**不會**觸發此事件。 |
| `agent.archived` | Agent 已封存 |
| `agent.deleted` | Agent 永久刪除——沒有留下可擷取的物件；將事件本身視為最終狀態 |
| `deployment.created` | 排程部署已建立 |
| `deployment.updated` | 部署屬性變更（例如排程已編輯） |
| `deployment.paused` | 部署已暫停——依請求，或在排程執行因**不可恢復**錯誤（封存的 agent、缺少的環境）失敗時自動暫停。可恢復的失敗（包含頻率限制）**不**會自動暫停。 |
| `deployment.unpaused` | 部署已恢復；排程繼續 |
| `deployment.archived` | 部署已封存——直接，或因 agent 封存/刪除而封存 |
| `deployment.deleted` | 部署永久刪除——沒有留下可擷取的物件；將事件本身視為最終狀態 |
| `deployment_run.started` | 一次**排程**執行已開始。手動執行**不**發出 `deployment_run.*` 事件。 |
| `deployment_run.succeeded` | 排程執行已建立其 session。`data.id`（執行記錄 ID）與執行的 `.started` 事件相同——擷取部署執行記錄取得其 `session_id`，然後訂閱 session 事件追蹤工作。 |
| `deployment_run.failed` | 排程執行未建立 session。`data.id` 與執行的 `.started` 事件相同——擷取部署執行記錄取得 `error.type` / `error.message`。 |
| `environment.created` | 環境已建立 |
| `environment.updated` | 環境已更新且至少有一個欄位變更。無操作的更新不發出任何事件。 |
| `environment.archived` | 環境已封存。重複封存已封存的環境不發出任何事件。 |
| `environment.deleted` | 環境已刪除，包含對已封存環境的刪除。沒有留下可擷取的物件；將事件本身視為最終狀態 |
| `memory_store.created` | 記憶體存放區已建立——由您，或由複製其中一個存放區的 Anthropic 操作的處理程序 |
| `memory_store.archived` | 記憶體存放區已封存。重複封存已封存的存放區不發出任何事件。 |
| `memory_store.deleted` | 記憶體存放區已刪除，包含對已封存存放區的刪除。**不**為每個記憶體發送個別事件——此單一事件即為訊號。沒有留下可擷取的物件；將其視為最終狀態 |

> **刻意沒有 `memory_store.updated`。** 個別記憶體和記憶體版本完全不發出 webhook 事件，環境的自託管工作項目也不發出。若您需要追蹤每個記憶體的變更，請輪詢記憶體版本端點（`shared/managed-agents-memory.md`）。

> 這些是 **webhook** `data.type` 值——與 SSE 事件類型（`session.status_idle`、`span.outcome_evaluation_end` 等，在 `shared/managed-agents-events.md` 中）是不同的命名空間。不要在 webhook 處理器中重用 SSE 常數。

---

## 傳送行為與注意事項

- **重複。** 一個端點可能不只一次收到相同事件；每次嘗試都帶有相同的頂層 `event.id`（= `webhook-id` header）。以它進行去重。
- **訂閱範圍。** 事件只到達**在事件發出時**已訂閱其類型的端點。在沒有任何訂閱時發出的事件永遠不會傳送，稍後訂閱也不會補傳——在需要某類型之前先訂閱。
- **無順序保證。** 事件不按發生順序傳送：`session.status_idled` 可能在 `session.outcome_evaluation_ended` 之前到達，而同一資源的 `.deleted` 可能在 `.archived` 之前到達。**依您擷取的資源驅動狀態，而非依到達順序。**
- **重試：每個端點每個事件最多三次嘗試**，使用 5 到 120 秒之間的抖動指數退避。觸發自動停用的回應永遠不會重試。**最後一次嘗試失敗後事件被丟棄**——不排隊，也沒有丟失訊號。Webhook 不是持久日誌：若您必須觀察每次轉換，請透過列出或擷取資源進行核對。
- **`webhook-timestamp` 在每次嘗試時重新蓋戳**，因此重試不會失敗 SDK 的五分鐘新鮮度檢查。它計算的是*傳送嘗試*的時間；事件發生時間請使用 payload 的 `created_at`。
- **自動停用——三個觸發條件**，每個都會設定 `disabled_reason`，均可從 Console 恢復（停用期間發出的事件**不會**重播）：
  - `3xx` 回應。從不跟隨重新導向；在第一次嘗試時立即停用。原因：`auto-disabled: endpoint URL returned a redirect (3xx)`。
  - URL 在連線時解析到非公開 IP。立即停用。原因：`auto-disabled: endpoint URL resolved to an invalid address`。
  - 持續失敗達一段持續時間。原因：`auto-disabled after sustained delivery failures`。**觸發條件是持續時間，而非傳送次數**——單一 `2xx` 重置視窗，因此一個不穩定的事件不能停用端點。
- **精簡 payload 是刻意的。** 不要期望 webhook 主體上有 `stop_reason`（請列出 session 的事件取得——session 物件本身沒有 `stop_reason` 欄位）、`outcome_evaluations`、憑證密鑰等——請擷取資源。
