---
source_file: managed-agents-multiagent.md
source_commit: 34040c9c568585f6929bedeaad110ad08f079624
source_sha256: b937599363615432ba9f69c3cc53616d2e7844907fc5a3f744355a55b9d38a1c
translated_at: 2026-09-13
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `managed-agents-multiagent.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Managed Agents — 多 Agent Session

協調者 agent 可以在一個 session 中委派給其他 agent。所有 agent **共用容器和檔案系統**；每個都在自己的**執行緒**中執行——一個具有獨立對話歷史、模型、系統 prompt、工具、MCP 伺服器和 skill（來自該 agent 自己的設定）的情境隔離事件串流。執行緒是持久的：協調者可以向它之前呼叫的子 agent 發送後續訊息，而那個子 agent 保留其先前的輪次。

SDK 在所有 `client.beta.{agents,sessions}.*` 呼叫上自動設定 `managed-agents-2026-04-01` beta header；多 agent 不需要額外的 header。

---

## 何時使用——從 `self` 開始，然後新增更便宜的 worker

**若 agent 的工作分成獨立的部分**——幾個要研究的來源、許多要處理的檔案或記錄、任何形狀為「研究 N 件事，然後摘要」的東西——或一個部分會用大量閱讀填滿情境，**請使用多 agent session，而非一個長的單執行緒迴圈。** 每個委派的部分在有全新情境視窗的自己執行緒中執行，執行緒在同一個容器中並行執行，且只有每個子 agent 的報告回傳，因此協調者的情境保持小。沒有協作程式碼要寫：協調者自動獲得委派工具並決定何時使用它們，而您的客戶端仍然建立一個 session 並讀取一個串流。

**第一步——最小有用的 roster 是 agent 本身。** 新增一個唯一條目為 `{"type": "self"}` 的 `multiagent` 區塊。協調者可以將自包含的子任務交給自己的副本——相同的模型、系統 prompt 和工具，但沒有進一步委派的能力——並組合它們的報告。其他什麼都不變。

```python
agent = client.beta.agents.create(
    name="Research assistant",
    description="Researches a question end to end. A copy can be spawned to own one well-scoped sub-question.",
    model="claude-opus-5",
    system="You are a research assistant. When a request splits into independent sub-questions, delegate each to a copy of yourself, one self-contained task per copy, then verify and combine their reports.",
    tools=[{"type": "agent_toolset_20260401"}],
    multiagent={"type": "coordinator", "agents": [{"type": "self"}]},  # the only change vs. a single agent
)

session = client.beta.sessions.create(agent=agent.id, environment_id=env.id)  # unchanged
```

**第二步——將閱讀繁重的工作移到更便宜的模型。** 委派的研究工作主要是搜尋、閱讀和提取：許多輸入 token，幾乎沒有困難的推理。在較小的現代模型（Claude Haiku 4.5，或當 worker 需要更多判斷力時用 Claude Sonnet 5）上建立第二個 agent，帶有窄範圍的 `system` prompt 和只需要的工具，並將其列在 `self` 旁邊。roster 條目只是一個參考：worker 在自己的 `model`、`system` 和 `tools` 上執行，其 token 以自己模型的費率計費。大型模型將其 token 用於規劃、檢查和綜合；小型模型做大量閱讀。

```python
worker = client.beta.agents.create(
    name="Web researcher",
    description="Fast, low-cost, read-only researcher. Give it one well-scoped question; it searches, reads, and reports findings with sources.",
    model="claude-haiku-4-5",
    system="Answer exactly the question you are given. Search and read as much as you need, then report concise findings with a source URL or file path for every claim.",
    tools=[{
        "type": "agent_toolset_20260401",
        "default_config": {"enabled": False},
        "configs": [{"name": n, "enabled": True} for n in ("read", "glob", "grep", "web_fetch", "web_search")],
    }],
)

lead = client.beta.agents.create(
    name="Research lead",
    description="Plans and synthesizes research. A copy can be spawned to own one large sub-analysis.",
    model="claude-opus-5",
    system="Plan the work. Delegate each independent, reading-heavy question to Web researcher, one self-contained task per spawn, several in parallel. Keep verification and the final synthesis for yourself; spawn a copy of yourself only for a sub-analysis that needs your full capability.",
    tools=[{"type": "agent_toolset_20260401"}],
    multiagent={"type": "coordinator", "agents": [worker.id, {"type": "self"}]},
)
```

**第三步——新增專門的專家。** 當子任務需要不同技能時，給每個 agent 自己的模型、窄範圍的 `system` prompt 和只需要的工具——並以 ID 列在 `self` 旁邊的 roster 上。這裡 lead 自己做出變更，向幾個唯讀審查者執行緒發送相同的審查摘要以進行獨立審查（一個已列 roster 的 agent 可以生成多次），並向測試撰寫者交付一個自包含的摘要；然後它去重複找到的問題，在採取行動前對照程式碼逐一核查，修復，並讓測試撰寫者重新執行。

```python
reviewer = client.beta.agents.create(
    name="Concurrency reviewer",
    description="Read-only reviewer for race conditions, deadlocks, lost updates, and retry/idempotency bugs. Give it the changed file paths and the invariants that must hold; it reports findings with file:line evidence. Spawn several on the same change for independent reviews.",
    model="claude-sonnet-5",
    system="Review only the files you are pointed at. Look for concurrency bugs: unsynchronized shared state, lock ordering, non-atomic read-modify-write, retries without idempotency. Report each finding as file:line, the interleaving that triggers it, and a suggested fix; say plainly if you found none.",
    tools=[{"type": "agent_toolset_20260401", "default_config": {"enabled": False},
            "configs": [{"name": n, "enabled": True} for n in ("read", "glob", "grep")]}],
)
test_writer = client.beta.agents.create(
    name="Test writer",
    description="Writes and runs tests. Give it the module path, the behavior to pin down, and the test command; it adds test files, runs them, and reports results with output.",
    model="claude-sonnet-5",
    system="Write focused tests for the behavior you are given, run them with the command you are given, and report pass/fail, the relevant output, and the paths of files you added. Do not edit non-test code; if the code under test looks wrong, report that instead.",
    tools=[{"type": "agent_toolset_20260401", "default_config": {"enabled": True},
            "configs": [{"name": n, "enabled": False} for n in ("web_fetch", "web_search")]}],
)
lead = client.beta.agents.create(
    name="Engineering lead",
    description="Plans and makes code changes and integrates specialist reports. A copy can be spawned to own one independent change.",
    model="claude-opus-5",
    system="Make the change yourself. Then, in parallel, send the changed paths and invariants to three Concurrency reviewers and the module path and test command to Test writer. Merge and de-duplicate the reviewers' findings, check each against the code before acting on it, fix, and have Test writer re-run. Keep design decisions and the final summary for yourself.",
    tools=[{"type": "agent_toolset_20260401"}],
    multiagent={"type": "coordinator", "agents": [reviewer.id, test_writer.id, {"type": "self"}]},
)
```

相同的形式適合不同專家的流水線：快速文件提取器（例如在 Claude Haiku 4.5 上）為每個輸入文件寫一個 JSON 檔案，一個驗證者對照其來源核查每個檔案，以及一個 lead 套用修正並將最終表格寫入 `/mnt/session/outputs/`。在每個任務中放入輸入和輸出路徑：執行緒共用容器的檔案系統，而非彼此的對話。

- **適合的情況：** 跨來源的並行研究；閱讀大量材料而不填滿協調者的情境；具有窄範圍 prompt 和工具集的專家，而非一個 agent 攜帶所有工具。**不適合的情況：** 小型單步驟任務——每次委派都有來回開銷和重新簡報的成本。
- **為協調者閱讀而撰寫 `name` 和 `description`。** 協調者從每個 roster 條目的 name 和 description 中選擇要生成誰（`self` 條目以協調者自己的名稱列出），因此請說明每個 agent 擅長什麼以及要交付什麼。名稱在 roster 中必須唯一；不要將 agent 命名為 `self`。
- **在協調者的 `system` prompt 中說明如何委派**——要交給誰、多少個同時、保留什麼給自己，以及什麼太小不值得委派（`shared/model-migration.md` 中的*委派給子 agent* 示範 prompt 是起點）。子 agent 不看協調者的任何對話，因此每個任務必須帶有它需要的路徑、限制和報告格式。生成立即回傳；子 agent 的報告在後來的協調者輪次中到達。
- **Web 工具的網域清單只會逐層收斂，絕不會放寬。** roster agent 的 `web_search` / `web_fetch` 呼叫，同時受其自己的 `allowed_domains` / `blocked_domains`、呼叫它的每個 agent 的清單，以及協調者目前清單的約束（允許清單取交集，封鎖清單取聯集）。請讓每個 roster agent 的允許清單保持在協調者的允許清單範圍內——彼此不相交的清單會讓工具仍然存在，但每次呼叫都以 `url_not_allowed` 失敗。請參見 `shared/managed-agents-tools.md` § Web search 與 web fetch 設定。
- **限制：** 1–20 個 roster 條目（最多一個 `self`；每個已列 roster 的 agent 可以生成多次），一個委派層級（roster 成員不能有自己的 `multiagent`），以及每個 session 最多 25 個並行執行緒——若長時間 session 需要更多，封存已完成的執行緒（見下方「中斷和封存執行緒」）。

以下各節是 roster、執行緒、事件和客戶端處理的參考；平台指南是 `https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration.md`。

---

## 在協調者上宣告 roster

`multiagent` 是 `agents.create()` / `agents.update()` 上的**頂層欄位**——**不是** `tools[]` 條目。`agents` 列出 1–20 個 roster 條目。`sessions.create()` 上沒有任何變更——roster 從協調者的設定解析。

```python
orchestrator = client.beta.agents.create(
    name="Engineering lead",
    model="claude-opus-5",
    system="You coordinate engineering work. Delegate code review to the reviewer and test writing to the test agent.",
    tools=[{"type": "agent_toolset_20260401"}],
    multiagent={
        "type": "coordinator",
        "agents": [
            reviewer.id,                                            # bare string - latest version
            {"type": "agent", "id": test_writer.id, "version": 4},  # pinned version
            {"type": "self"},                                       # the coordinator itself
        ],
    },
)

session = client.beta.sessions.create(agent=orchestrator.id, environment_id=env.id)
```

| Roster 條目 | 形狀 | 備注 |
|---|---|---|
| 字串簡寫 | `"agent_abc123"` | 引用已儲存 agent 的最新版本。 |
| Agent 參考 | `{type: "agent", id, version?}` | 省略 `version` 以固定協調者儲存時的最新版本。 |
| Self | `{type: "self"}` | 協調者可以生成自己的副本。 |
| Advisor | `{type: "advisor", model}` | Session 的主執行緒在輪次中途可以諮詢的模型。每個 roster 最多一個。見「Advisor」。 |

若 session 以 `agent_with_overrides` 建立（見 `shared/managed-agents-core.md` → 為 session 覆寫 agent 設定），那些覆寫套用到**協調者及其 `self` 副本**。以 ID 引用的 roster agent 始終使用其自己的原始設定——覆寫不會傳播到它們。

協調者的執行緒收到用於操作 roster 的委派工具：`list_agents`（查看 roster）和 `send_to_agent`（向成員交付任務或訊息）。Roster 中最多 **20 個唯一 agent**；協調者可以生成每個的**多個副本**。**只有一個委派層級**——且是強制執行的，而非靜默展平：將自身帶有 `multiagent.agents` roster 的 agent 列入 roster 會使建立或更新失敗並帶有驗證錯誤。

**推理地理固定必須在 roster 中一致。** 當 agent 固定推理地理（`model.inference_geo`——見 `shared/managed-agents-core.md` § 固定推理地理位置）時，協調者的固定和每個 roster 成員的固定必須全部相同或全部未設定。不一致的 roster 是 400 驗證錯誤，無論是 agent 儲存時還是 session 建立的 `model` 覆寫更改任何固定時。

---

## 執行緒

Session 層級的事件串流是**主執行緒**——它顯示協調者的追蹤加上子 agent 活動的精簡檢視（執行緒狀態轉換和跨執行緒訊息，而非每個子 agent 工具呼叫）。透過每個執行緒的端點深入到特定子 agent：

| 操作 | HTTP | SDK（`client.beta.sessions.threads.*`） |
|---|---|---|
| 列出執行緒 | `GET /v1/sessions/{sid}/threads` | `.list(session_id)` |
| 取得一個 | `GET /v1/sessions/{sid}/threads/{tid}` | `.retrieve(thread_id, session_id=...)` |
| 封存 | `POST /v1/sessions/{sid}/threads/{tid}/archive` | `.archive(thread_id, session_id=...)` |
| 列出執行緒事件 | `GET /v1/sessions/{sid}/threads/{tid}/events` | `.events.list(thread_id, session_id=...)` |
| 串流執行緒事件 | `GET /v1/sessions/{sid}/threads/{tid}/stream` | `.events.stream(thread_id, session_id=...)` |

每個 `SessionThread` 帶有 `id`、`status`（`running` | `idle` | `rescheduling` | `terminated`）、`agent`（agent 設定的已解析快照——`id`、`name`、`model`、`system`、`tools`、`skills`、`mcp_servers`、`version`——advisor 執行緒除外，其 `agent` 是兩欄位的 advisor 形式 `{"type": "advisor", "model": ...}`——見「Advisor」）、`parent_thread_id`（主執行緒為 null，包含在列表中）、`archived_at` 和選用的 `stats`/`usage`。每個執行緒的 `usage.list_cost` 數字**不**加總為 session 總計——session 數字另外包含 session 執行時間，且每個數字都獨立四捨五入；session 層級的 `usage.list_cost` 是權威的。**Session 狀態聚合執行緒狀態**——若任何執行緒為 `running`，`session.status` 就為 `running`。最多 **25 個並行執行緒**（advisor 執行緒豁免——見「Advisor」）。排空每個執行緒的串流時，在 `session.thread_status_idle` 時中斷（並像對 session 層級 idle 一樣檢查其 `stop_reason`）。

**Session 預算是跨所有執行緒的一個共用上限**——沒有按執行緒的上限。每個執行緒的消耗以其自己所服務的模型定價，執行緒在共用上限達到時獨立暫停（`stop_reason: budget_reached`）；一個執行緒可以在另一個完成其進行中請求時暫停。等待 `requires_action` 的執行緒在 session 層級上優先於上限。見 `shared/managed-agents-core.md` § Session 預算。

---

## 多 agent 事件（在 session 串流上）

| 事件 | Payload 重點 | 意義 |
|---|---|---|
| `session.thread_created` | `session_thread_id`、`agent_name` | 新的執行緒被建立。 |
| `session.thread_status_running` | `session_thread_id`、`agent_name` | 執行緒開始活動。 |
| `session.thread_status_idle` | `session_thread_id`、`agent_name`、**`stop_reason`** | 執行緒正在等待輸入——或在 session 的共用預算暫停（`stop_reason: budget_reached`）。檢查 `stop_reason`（與 `session.status_idle.stop_reason` 形狀相同）。 |
| `session.thread_status_rescheduled` | `session_thread_id`、`agent_name` | 執行緒在可重試錯誤後重新排程。 |
| `session.thread_status_terminated` | `session_thread_id`、`agent_name` | 執行緒結束——完成工作並自我終止（advisor 諮詢執行緒——見「Advisor」）、被封存，或遇到終端錯誤。 |
| `agent.thread_message_sent` | `to_session_thread_id`、`to_agent_name`、`content` | *這個*執行緒向另一個執行緒發送了訊息。在主要串流上：協調者向 agent 發送了任務或後續訊息。 |
| `agent.thread_message_received` | `from_session_thread_id`、`from_agent_name`、`content` | 訊息從另一個執行緒到達*這個*執行緒。在主要串流上：agent 向協調者發送了報告或問題。 |

> **方向是相對於攜帶事件的執行緒串流的**，而非相對於協調者。相同的委派任務在主要串流上是 `agent.thread_message_sent`，在子執行緒自己的串流上是 `agent.thread_message_received`。一旦您在讀取子執行緒串流，將 `_received` 視為「子 agent 完成了」是錯誤的。

---

## 預覽子 agent 的文字

每個執行緒的串流接受與 session 層級串流相同的 `event_deltas[]` 參數，因此您可以在模型生成時觀察子 agent 的文字：

```
GET /v1/sessions/{sid}/threads/{tid}/stream?event_deltas%5B%5D=agent.message
```

**預覽是以執行緒為範圍的。** 子執行緒的預覽只在那個子執行緒的串流上傳送，永不跨發到 session 層級串流（其預覽範圍僅限主執行緒）。因此即時觀察子 agent 意味著開啟其執行緒串流——無論您傳入什麼，session 串流都不會顯示它。

> 警告：**只有純 assistant 文字預覽。** 子 agent *對其協調者的回覆*乘坐 `agent.thread_message_sent`，永不被預覽。因此只報告回去的 worker 即使在正確執行緒上正確選擇加入，也完全不串流任何 delta。若要從子 agent 獲得即時預覽，其 prompt 必須讓它先在自己的執行緒中以純 assistant 訊息寫出答案，然後才向協調者報告。每個連線執行一個累加器，並在 `session.thread_status_idle` 時退出讀取迴圈。選擇加入、累積和核對的詳細資訊：`shared/managed-agents-events.md` → 即時預覽。

---

## Advisor

`{"type": "advisor", "model": "<model id>"}` roster 條目為 session 的**主執行緒**提供 advisor：一個它可以在輪次中途諮詢以獲得策略指引的模型（規劃方式、擺脫困境、在完成前審查工作）。條目恰好有兩個欄位——`type` 和 `model`——可以與任何其他 roster 形式並排；只有這個條目的 roster 也可以。advisor 也作為 Messages API 上的伺服器工具可用（`advisor_20260301`——見 `shared/tool-use-concepts.md` → Advisor）；Managed Agents 介面在設定和傳送上有所不同：roster 條目**沒有 `max_uses`、`max_tokens` 或 `caching` 欄位**，且建議透過執行緒事件到達，而非透過 `advisor_tool_result` 區塊。

```python
agent = client.beta.agents.create(
    name="Backend engineer",
    model="claude-sonnet-5",
    system="You implement backend features end to end.",
    multiagent={
        "type": "coordinator",
        "agents": [{"type": "advisor", "model": "claude-opus-5"}],
    },
)
```

（Claude Opus 5 是預設的 advisor 選擇。它是一個已編輯的 advisor——agent 在伺服器端讀取其建議，但客戶端看到 `[{"type": "redacted"}]`；見下方「明文 vs 已編輯傳送」。對於客戶端可讀的建議，像 `claude-opus-4-8` 這樣的明文 advisor 只有在 agent 自己的模型是 `claude-opus-4-8` 或以下時才有效——在 Claude Opus 5、Claude Fable 5.1 或 Claude Mythos 5.1 上的 agent 只能與已編輯的 advisor 配對，因此客戶端可讀的建議對它們不可用（配對表：`shared/tool-use-concepts.md`）。）

**規則：**
- **每個 roster 最多一個 advisor 條目。** 該條目佔用保留的 roster 名稱 `anthropic.advisor`——也列出字面上命名為 `anthropic.advisor` 的成員的 roster 是 400。在回應中，advisor 條目無論提交位置如何都**最後**回顯在 roster 中。
- **配對在 agent 儲存時驗證：** advisor 模型必須達到最低能力標準，且 agent 自己的模型不能比其 advisor 更有能力（相等可以配對）。無效配對 → 400。有效配對反映 Messages advisor 工具的執行器 ↔ advisor 表（`shared/tool-use-concepts.md`）。
- **只有主執行緒諮詢它。** Advisor 不是 roster agent：對協調者的 `list_agents` 工具不可見，透過 `send_to_agent` 不可達，roster agent 也無法諮詢它。

**諮詢如何運作。** 每次諮詢作為平台生成的名為 `anthropic.advisor` 的執行緒運行，完成後自我終止；建議作為 `agent.thread_message_received` 事件傳送到主執行緒。典型的事件順序（保留名稱乘坐生命週期事件上的 `agent_name` 和傳送上的 `from_agent_name`）：

1. `session.thread_created`
2. `session.thread_status_running`
3. `agent.thread_message_received` — 建議
4. `session.thread_status_idle`（`stop_reason: end_turn`）
5. `session.thread_status_terminated`

諮詢不發出 `agent.tool_use` 和 `agent.thread_message_sent`，且**建議傳送不保證在 advisor 執行緒的 idle/terminated 事件之前**——不要將那些視為「建議已傳送」。

**明文 vs 已編輯傳送。** 您的客戶端是否可以讀取建議取決於 advisor 模型的政策，反映 Messages advisor 工具的結果變體：在那裡回傳明文的模型在這裡傳送可讀的文字內容；回傳已編輯結果的模型在每個客戶端介面上傳送 `[{"type": "redacted"}]` 作為訊息內容，而 agent 在伺服器端仍然讀取完整的建議。Advisor thinking 永不顯現。客戶端不能自行送出 `redacted` 區塊——包含一個的事件是 400。

**失敗和中斷。** 失敗的諮詢——或透過帶有 advisor 執行緒的 `session_thread_id` 的 `user.interrupt` 放棄的諮詢——永不使 agent 的輪次失敗：agent 在一般通知後繼續。諮詢期間的 session 層級 `user.interrupt` 像往常一樣停止整個 session（每個執行緒，包括主執行緒），終止 advisor 執行緒，不傳送建議。

**執行緒、計費、快取。** Advisor 執行緒**豁免於 25 個並行執行緒的限制**。它們出現在 session 的執行緒列表中，`agent` 設定為按設定的 advisor 形式（`{"type": "advisor", "model": ...}`），`parent_thread_id` 設定為主執行緒。諮詢以 advisor 模型的費率計費；其 token 出現在 advisor 執行緒的使用量和 session 的總計中。Advisor 端的 prompt 快取是自動的——無需設定。

**移除 advisor：** 以省略該條目的 roster 更新 agent；若 advisor 是 roster 的唯一條目，用 `"multiagent": null` 清除 roster。

---

## 來自子 agent 執行緒的工具權限和自訂工具

當子 agent 需要您的客戶端（工具呼叫暫停等待核准——`always_ask`，或 `auto` 策略下伺服器未能做出判定——或自訂工具結果），請求**跨發到主執行緒**，`session_thread_id` 識別原始執行緒——因此您只需要監看 session 串流。以 `user.tool_confirmation`（帶有 `tool_use_id`）或 `user.custom_tool_result`（帶有 `custom_tool_use_id`）回覆，並**回顯來自原始事件的 `session_thread_id`**（SDK 參數類型和說明字串期望它）。伺服器也透過工具使用 ID 路由，因此回顯是雙重保障而非關鍵——但請包含它。

```python
for event_id in stop.event_ids:
    pending = events_by_id[event_id]
    confirmation = {
        "type": "user.tool_confirmation",
        "tool_use_id": event_id,
        "result": "allow",
    }
    if pending.session_thread_id is not None:
        confirmation["session_thread_id"] = pending.session_thread_id
    client.beta.sessions.events.send(session.id, events=[confirmation])
```

相同的模式適用於 `user.custom_tool_result`。

**`auto` 在多 agent session 中的行為。** 只有您在主執行緒上的 `user.message` 事件能讓伺服器允許原本在 `auto` 策略下會被拒絕的呼叫；子 agent 執行緒中的任何內容都不具備這種效力（您的客戶端不在那裡發送訊息，協調者傳給子 agent 的訊息也不帶有這種效力）。伺服器在 `auto` 策略下拒絕的呼叫**不會**跨發到主執行緒——其事件和錯誤工具結果只出現在子 agent 自己的執行緒串流上，子 agent 繼續執行。

---

## 中斷和封存執行緒

- **不帶 `session_thread_id` 的 `user.interrupt` 中斷 session 中每個未封存的執行緒，包括主執行緒**——它不是僅限主執行緒的停止。傳入 `session_thread_id` 以針對一個執行緒。
- **對等待 `requires_action` 的子執行緒**，中斷以*錯誤*工具結果（`"Tool execution was interrupted before completion. Please retry."`）關閉每個待處理的工具呼叫，並直接重新發出帶有 `stop_reason: end_turn` 的 `session.thread_status_idle`——不對模型取樣。對已 `idle` 的執行緒，中斷是無操作——但有一個例外：自託管環境上的 session，若其 worker 未能完成已認領的工作項目（例如記憶體存放區掛載錯誤），會停在 `idle`，此時送出 `user.interrupt` 會將該工作重新排入佇列，讓下一次 worker 認領時重試（`shared/managed-agents-self-hosted-sandboxes.md` § 記憶體存放區 → 疑難排解）。
- **封存需要執行緒處於 idle，而 `requires_action` 算作 idle**——停在待處理工具呼叫上的執行緒可以直接封存。只有*執行中*的執行緒必須先中斷。

---

## 陷阱

- **不要將 roster 放在 `sessions.create()` 或 `tools[]` 中。** `multiagent` 是頂層 agent 欄位；更新協調者，然後啟動引用它的 session。
- **不要假設共用情境。** 執行緒共用檔案系統，但不共用對話歷史或工具。若協調者需要子 agent 對某事採取行動，它必須在委派訊息中說明（或將其寫入磁碟）。
- **深度 > 1 是驗證錯誤。** 將自身帶有 `multiagent.agents` roster 的 agent 列 roster 會使建立或更新失敗——只有 session 的協調者委派。

如需 Python 以外的語言繫結，請 WebFetch `https://platform.claude.com/docs/en/managed-agents/multiagent-orchestration.md`（見 `shared/live-sources.md`）。
