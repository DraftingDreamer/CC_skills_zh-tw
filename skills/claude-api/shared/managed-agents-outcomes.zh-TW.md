---
source_file: managed-agents-outcomes.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 742b89e66d22dcb605c89a667aa5572ce768a6b81448f624a1f73efa2cedfca2
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `managed-agents-outcomes.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Managed Agents — Outcomes（成果）

**outcome** 將 session 從*對話*提升為*工作*：您指定何謂「完成」，框架便執行「迭代 → 評分 → 修訂」的迴圈，直到產出物符合評分標準、達到 `max_iterations`，或被中斷為止。獨立的**評分器**（擁有獨立情境視窗）依您的評分標準為每次迭代打分，並將各評分維度的差距回饋給 agent。

SDK 會在所有 `client.beta.sessions.*` 呼叫上自動設定 `managed-agents-2026-04-01` beta header；outcome 功能無需額外的 header。

---

## `user.define_outcome` 事件

Outcome 不是 `sessions.create()` 上的欄位。您先建立一般 session，然後送出 `user.define_outcome` 事件。Agent 收到事件後即開始作業——**請勿同時送出 `user.message`** 來啟動。

您可以在 session 的 `initial_events` 陣列中傳入單一 `user.define_outcome`，將兩個呼叫合併為一次——相同的事件、相同的規則、一次往返（請參見 `shared/managed-agents-core.md` → 用 `initial_events` 初始化 session）。在該陣列中放入超過一個 `user.define_outcome`，或放入沒有 `rubric` 的 `user.define_outcome`，整個建立請求會以 400 拒絕。

```python
session = client.beta.sessions.create(
    agent=AGENT_ID,
    environment_id=ENVIRONMENT_ID,
    title="Financial analysis on Costco",
)

client.beta.sessions.events.send(
    session_id=session.id,
    events=[
        {
            "type": "user.define_outcome",
            "description": "Build a DCF model for Costco in .xlsx",
            "rubric": {"type": "text", "content": RUBRIC_MD},
            # or: "rubric": {"type": "file", "file_id": rubric.id}
            "max_iterations": 5,  # optional; default 3, max 20
        }
    ],
)
```

| 欄位 | 型別 | 備註 |
|---|---|---|
| `type` | `"user.define_outcome"` | |
| `description` | string | 任務描述。這是 agent 的工作目標——不需要另外送出 `user.message`。 |
| `rubric` | `{type: "text", content}` \| `{type: "file", file_id}` | **必填。** 使用明確且可獨立評分的條件撰寫 Markdown。透過 `client.beta.files.upload(...)`（beta `files-api-2025-04-14`）上傳一次，即可在多個 session 中重複使用。 |
| `max_iterations` | int | 選填。預設 **3**，最大 **20**。 |

事件會帶著伺服器指派的 `outcome_id` 和 `processed_at` 回傳至串流上。

> **撰寫評分標準。** 使用明確、可評分的條件（「CSV 有數值型的 `price` 欄位」），而非模糊語句（「資料看起來不錯」）——評分器會對每個條件獨立評分，因此模糊條件會產生雜訊迴圈。若您沒有評分標準，可讓 Claude 分析一份已知正確的產出物，並將分析結果轉換為評分標準。

---

## 特定於 Outcome 的事件

這些事件會與一般的 `agent.*` / `session.*` 事件一起出現在標準事件串流上（`sessions.events.stream` / `.list`）。

| 事件 | 關鍵 payload | 意義 |
|---|---|---|
| `span.outcome_evaluation_start` | `outcome_id`、`iteration`（從 0 開始） | 評分器開始對第 N 次迭代評分。 |
| `span.outcome_evaluation_ongoing` | `outcome_id` | 評分器執行中的心跳。評分器的推理過程是不透明的——您只能看到它正在工作，而不是它的思考內容。 |
| `span.outcome_evaluation_end` | `outcome_evaluation_start_id`、`outcome_id`、`iteration`、`result`、`explanation`、`usage` | 評分器完成一次迭代。`result` 決定接下來的行動（見下表）。 |

### `span.outcome_evaluation_end.result`

| `result` | 後續行動 |
|---|---|
| `satisfied` | Session → `idle`。此 outcome 的終止狀態。 |
| `needs_revision` | Agent 開始下一次迭代。 |
| `max_iterations_reached` | 不再進行評分循環。Agent 可能執行最後一次修訂，然後 session → `idle`。 |
| `failed` | Session → `idle`。評分標準與任務根本不符（例如描述和評分標準相互矛盾）。 |
| `interrupted` | 在 outcome 進行中時收到 `user.interrupt` 時發出——**即使評分尚未開始**。在這種情況下，`outcome_evaluation_start_id` 是空字串而非事件 ID，因此請在使用前先確認它不為空。（例外：在 session 預算暫停時送出的中斷會被接受並忽略——請參見 `shared/managed-agents-events.md` § 達到 session 預算。） |

```json
{
  "type": "span.outcome_evaluation_end",
  "id": "sevt_01jkl...",
  "outcome_evaluation_start_id": "sevt_01def...",
  "outcome_id": "outc_01a...",
  "result": "satisfied",
  "explanation": "All 12 criteria met: revenue projections use 5 years of historical data, ...",
  "iteration": 0,
  "usage": { "input_tokens": 2400, "output_tokens": 350, "cache_creation_input_tokens": 0, "cache_read_input_tokens": 1800 },
  "processed_at": "2026-03-25T14:03:00Z"
}
```

---

## 查詢狀態與取得交付成果

**狀態** — 監聽串流上的 `span.outcome_evaluation_end`，或輪詢 session 並讀取 `outcome_evaluations`：

```python
session = client.beta.sessions.retrieve(session.id)
for ev in session.outcome_evaluations:
    print(f"{ev.outcome_id}: {ev.result}")  # outc_01a...: satisfied
```

**交付成果** — agent 寫入 `/mnt/session/outputs/`。session 進入 idle 後，透過 Files API 並以 `scope_id=session.id` 取得。這是 `shared/managed-agents-environments.md` → Session 輸出中所記載的相同機制（包含 `files.list` 上的雙 beta header 需求）。

---

## 互動規則與注意事項

- **一次只能有一個 outcome。** 只有在前一個 outcome 的終止 `span.outcome_evaluation_end`（`satisfied` / `max_iterations_reached` / `failed` / `interrupted`）出現後，才能串接送出下一個 `user.define_outcome`。Session 會在串接的 outcome 之間保留歷史。
- **允許但非必要的引導。** 您*可以*在 outcome 進行中送出 `user.message` 事件來調整方向，但 agent 已知道要持續工作直到終止——不要送出「繼續」之類的提示。（例外：在預算達到而暫停的 session（`stop_reason: budget_reached`）只接受 settle 事件——在此處送出引導性的 `user.message` 或串接的 `user.define_outcome` 會收到 400；請參見 `shared/managed-agents-events.md` § 達到 session 預算。）
- **`user.interrupt` 會暫停當前 outcome** — 它將 `result` 標記為 `"interrupted"` 並讓 session 進入 `idle`，準備好接受新的 outcome 或對話輪次。（例外：在 session 預算暫停時送出，中斷會被接受並忽略，outcome 維持活躍——請參見 `shared/managed-agents-events.md` § 達到 session 預算。）
- **終止後，session 可重複使用** — 繼續對話或定義新的 outcome。
- **Outcome ≠ session 建立欄位。** 不要將 `outcome`、`rubric` 或 `description` 放在 `sessions.create()` 上——outcome 一律透過 `user.define_outcome` 事件送出。
- **Idle 中斷閘門保持不變。** 在您的消耗迴圈中，繼續使用 `event.type === 'session.status_idle' && event.stop_reason?.type !== 'requires_action'` — **不要**僅以 `span.outcome_evaluation_end` 為閘門（在 `needs_revision` 時 session 仍在運行）。請參見 `shared/managed-agents-client-patterns.md` Pattern 5。

如需原始 HTTP 格式及 Python 以外的各語言 SDK 繫結，請 WebFetch `https://platform.claude.com/docs/en/managed-agents/define-outcomes.md`（見 `shared/live-sources.md`）。
