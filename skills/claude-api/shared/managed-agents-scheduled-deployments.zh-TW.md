---
source_file: managed-agents-scheduled-deployments.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 65312565b26ce16176c63185f9b392f0a93ffc4ef46180a5eff32ca3678a47d3
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `managed-agents-scheduled-deployments.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Managed Agents — 排程部署

**排程部署**按照重複的 cron 排程執行 agent——每次觸發都會自動建立一個 session。適合用於有規律周期的工作：每日夜間分類、每週合規掃描、每小時監控。

需要 `managed-agents-2026-04-01` beta header（SDK 會在 `client.beta.deployments.*` / `client.beta.deployment_runs.*` 呼叫上自動設定）。

## 建立部署

部署將 session 所需的一切（agent、環境、選用的檔案 / GitHub / 記憶體存放區 / vault）加上 `schedule` 和啟動每次執行的 `initial_events` 打包在一起：

- `agent` 和 `environment_id` 為必填——形狀與 `sessions.create` 相同（請參見 `shared/managed-agents-core.md`）。以**自託管**環境為目標的部署可以附加 `memory_store` 資源（需要 SDK worker——見 `shared/managed-agents-self-hosted-sandboxes.md` § 記憶體存放區）；`file` 和 `github_repository` 資源則需要雲端環境。Console 部署表單不提供自託管環境的記憶體存放區選項——請透過 API/SDK 附加。
- `initial_events` 必須至少包含一個啟動事件——`user.message` **或** `user.define_outcome`。（部署的 `initial_events` 也接受 `system.message`，session 的不接受。）
- `schedule` 接受 cron `expression` 和 IANA `timezone`。最大粒度為分鐘級。

```bash
curl -fsSL https://api.anthropic.com/v1/deployments \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: managed-agents-2026-04-01" \
  -H "content-type: application/json" \
  -d @- <<EOF
{
  "name": "Weekly compliance scan",
  "agent": "$AGENT_ID",
  "environment_id": "$ENVIRONMENT_ID",
  "initial_events": [
    {"type": "user.message", "content": [{"type": "text", "text": "Run the weekly compliance scan."}]}
  ],
  "schedule": {
    "type": "cron",
    "expression": "0 20 * * 5",
    "timezone": "America/New_York"
  }
}
EOF
```

```python
deployment = client.beta.deployments.create(
    name="Weekly compliance scan",
    agent=agent.id,
    environment_id=environment.id,
    initial_events=[
        {
            "type": "user.message",
            "content": [{"type": "text", "text": "Run the weekly compliance scan."}],
        },
    ],
    schedule={
        "type": "cron",
        "expression": "0 20 * * 5",
        "timezone": "America/New_York",
    },
)
```

回應為部署物件（ID 前綴 `depl_`）。查看 `schedule.upcoming_runs_at`——下次執行時間——確認排程解析是否符合預期：

```json
{
  "id": "depl_01xyz",
  "status": "active",
  "paused_reason": null,
  "schedule": {
    "type": "cron",
    "expression": "0 20 * * 5",
    "timezone": "America/New_York",
    "last_run_at": null,
    "upcoming_runs_at": ["2026-05-09T00:00:00Z", "2026-05-16T00:00:00Z", "2026-05-23T00:00:00Z"]
  }
}
```

`upcoming_runs_at` 反映精確設定的排程，但**為了分散負載，實際執行時間會加入抖動：最多為執行間隔的 15%，下限 5 秒，上限 9 分鐘。** 因此，每小時的部署最多可能延遲 9 分鐘觸發；不要建立假設列出時間戳的下游截止時間。每個組織最多 **1000 個排程部署**（如需更多請聯絡 Anthropic 支援）。

### Cron 與時區語義

- **表達式：** 標準 POSIX cron（`分鐘 小時 日 月 星期`）。
- **時區：** IANA 識別字（例如 `"America/Los_Angeles"`）。
- **夏令時：** 依字面掛鐘時間比對——`"0 20 * * *"` 在 `America/New_York` 時區無論 EST/EDT 均於本地時間晚上 8:00 觸發。

> 警告：**夏令時邊界：** 在春季撥快時計時不存在的掛鐘時間（例如凌晨 2 點）會被**跳過**；在秋季撥回時出現兩次的時間會**觸發兩次**。當不能接受漏觸或重複執行時，請在本地時間凌晨 1–3 點視窗之外排程，或改用 UTC。

## 部署預算

部署接受與 session 相同的 `budget` 物件（`{type: "limit", max_list_cost: {amount, currency}}`——小單位分字串，僅限 `USD`；請參見 `shared/managed-agents-core.md` § Session 預算）。上限在**觸發時複製到每個 session 上**，該 session 的行為與任何有預算限制的 session 完全相同。

部署預算的更新語義與 session 不同：

- `budget` 可在**建立和更新**時接受——並非僅限於建立時。
- 更新時傳入 `budget: null` 可**清除**預算，且清除後的預算**可以稍後重新新增**——沒有單向門。
- 變更從**下一個觸發的 session** 起生效——已在執行中的 session 保留建立時的上限（透過各自的 session 更新來修改那些 session）。

## 部署執行記錄

每次觸發嘗試——無論成功與否——都會寫入一筆**部署執行記錄**（前綴 `drun_`），讓您能夠獨立於 session 生命週期稽核失敗。成功的執行記錄帶有已建立的 `session_id`；按照慣例透過事件串流（`shared/managed-agents-events.md`）或 webhook（`shared/managed-agents-webhooks.md`）追蹤該 session。失敗的執行記錄帶有 `error`，其 `type` 說明 session 建立被拒絕的原因。

```python
# All runs for a deployment
for run in client.beta.deployment_runs.list(deployment_id=deployment.id):
    print(run.created_at, run.session_id or run.error.type)

# Failures only
for run in client.beta.deployment_runs.list(deployment_id=deployment.id, has_error=True):
    print(run.created_at, run.error.type, run.error.message)
```

```typescript
for await (const run of client.beta.deploymentRuns.list({
  deployment_id: deployment.id,
  has_error: true,
})) {
  console.log(run.created_at, run.error?.type, run.error?.message);
}
```

原始 HTTP：`GET /v1/deployment_runs?deployment_id=...&has_error=true`。透過 ID 取得單一執行記錄：`GET /v1/deployment_runs/{deployment_run_id}`（SDK：`client.beta.deployment_runs.retrieve(run_id)`）——`deployment_run.*` webhook 事件帶有執行記錄 ID 作為 `data.id`。

失敗的執行記錄範例：

```json
{
  "type": "deployment_run",
  "id": "drun_01abc124",
  "deployment_id": "depl_01xyz",
  "trigger_context": { "type": "schedule", "scheduled_at": "2026-05-09T00:00:00Z" },
  "session_id": null,
  "error": { "type": "environment_archived", "message": "environment `env_01abc` is archived" },
  "agent": { "type": "agent", "id": "agent_01ghi789", "version": 3 },
  "created_at": "2026-05-09T00:00:01Z"
}
```

錯誤類型包括 `environment_archived`、`agent_archived`、`vault_not_found`、`session_rate_limited` 和 `service_unavailable`。

每次**排程**執行的結果（已開始/已成功/已失敗）以及每次部署生命週期變更（已建立/已更新/已暫停/已恢復/已封存/已刪除）也會以 webhook 事件方式傳送——關於 `deployment.*` 和 `deployment_run.*` 事件類型，請參見 `shared/managed-agents-webhooks.md`——讓您無需輪詢即可應對。手動執行**不會**發出 `deployment_run.*` webhook 事件。

## 生命週期：暫停 / 恢復 / 封存

| 操作 | SDK | 效果 |
|---|---|---|
| 暫停 | `client.beta.deployments.pause(id)` | 從此刻起抑制排程觸發。已在執行中的 session 繼續執行。**暫停期間仍允許手動執行。** 設定 `paused_reason: {"type": "manual"}`。 |
| 恢復 | `client.beta.deployments.unpause(id)` | 從下一個排程時間點恢復。**不會補觸漏過的觸發。** 清除 `paused_reason`。 |
| 封存 | `client.beta.deployments.archive(id)` | **終止狀態** — 排程停止，部署無法再被修改。可逆的操作請使用暫停。 |

原始 HTTP：`POST /v1/deployments/{deployment_id}/pause`（同樣適用 `/unpause`、`/archive`）。

### 失敗行為

- **頻率限制：** 立即記錄為 `session_rate_limited` 執行記錄，**不重試** — 排程只在下次出現時再試。（session 內部 API 呼叫的頻率限制由 session 自行處理。）
- **其他失敗執行記錄**（例如 `environment_archived`、`vault_not_found`、`service_unavailable`）：執行記錄記錄 `error.type` — 監控執行記錄並修復所指資源，或暫停部署。
- **Agent 已封存：** 部署在同一操作中自動**封存**（終止狀態）。**Agent 已刪除：** 下一次排程觸發會偵測到缺少的 agent 並屆時封存部署。無論哪種情況，都不會記錄部署執行記錄，也不會再建立 session。

## 手動執行

`POST /v1/deployments/{deployment_id}/run`（SDK：`client.beta.deployments.run(id)`）立即建立 session 並寫入帶有 `trigger_context.type: "manual"` 的執行記錄。用於在**提交排程之前測試部署**——記住即使在部署暫停時也可以使用。
