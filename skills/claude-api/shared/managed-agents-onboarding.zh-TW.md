---
source_file: managed-agents-onboarding.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: d6f0fae12e1ec04e5a96863f928f9b62d09ab860c2653d13a62b6b282d832682
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `managed-agents-onboarding.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Managed Agents — 上線流程

> **透過 `/claude-api managed-agents-onboard` 呼叫？** 您來對地方了。請執行以下訪談——不要把它摘要給使用者，直接問問題。

Claude Managed Agents 是一個託管 agent：Anthropic 執行 agent 迴圈，並為每個 session 配置一個沙盒容器，供 agent 的工具執行（或您自己的 worker，使用 `self_hosted` 環境——請參見 `shared/managed-agents-self-hosted-sandboxes.md`）。您提供 **agent 設定**（工具、skills、模型、系統 prompt——可重複使用、有版本控制）和**環境設定**（沙盒——可在 agent 之間重複使用）。每次執行都是一個 **session**。

流程分四個節拍——**描述 → agent → 環境 → session**——與 Console 快速入門的流程相同，哲學也相同：**憑證之前先驗證價值**。使用者在任何詢問驗證之前就從想法到可執行的 session；每個憑證在設計使其相關時（§2）被*標記*，並在 session 設定時（§4）一次性*收集*，在那裡它被繫結（`sessions.create()`）並被使用（冒煙測試）。請對照閱讀 `shared/managed-agents-core.md`——它對每個選項有完整說明；本文件是訪談指引。

---

## 1. 描述任務

**以一句話提示語開頭，配合單一開放式問題——不要猜測，不要問卷調查。** 用您自己的話：

> Managed Agents 是託管的——Anthropic 執行 agent 迴圈、沙盒和基礎設施；您只需定義 agent。我們會分三步完成：agent、它執行的環境，然後是一個即時測試 session。那麼：描述一下您想要的 agent——它應該做什麼，以及是什麼觸發它（一個人、一個事件，還是排程）？

讓他們完整回答後再進行任何設定。

## 2. 設定 agent——提議，而非詢問

他們的描述完成了訪談的工作。從中起草 agent 設定，並**以提案形式呈現，內嵌您的建議**——使用者回應具體的設定，而不是回答問題清單。最多一個批次的後續問題，用於真正的缺口。在描述給您機會的地方提出建議：

- **工具** — 預設啟用完整的預建工具集（`agent_toolset_20260401`：`bash`、`read`、`write`、`edit`、`glob`、`grep`、`web_fetch`、`web_search`）。**建議 MCP 伺服器**用於工作中提及的任何第三方服務（GitHub、Linear、Slack……）——並在建議時標記每個服務隱含的憑證（「Linear MCP → 您在啟動時需要 Linear API token」），讓 §4 的驗證步驟成為形式，而非意外。收集本身等到 §4。自訂工具僅在使用者自己的應用程式必須回應呼叫時才使用（名稱、說明、輸入綱要——其處理器程式碼由他們負責；不要幫他們生成）。
- **Skills** — 當工作產生這些產出物時，**建議**預建的 `xlsx`/`docx`/`pptx`/`pdf`；自訂則用 `skill_id`（每個 agent 總計最多 20 個，預建 + 自訂合計）。
- **Outcome** — 若描述暗示可查核的「完成」標準（或您可以在後續問題中引出它們：不是「一份好報告」而是「每個 SKU 有數值型 `price` 欄位的 CSV」），**建議 Outcome 啟動方式**——框架依評分標準評分並迭代（`shared/managed-agents-outcomes.md`）。
- **現有資源** — 磁碟上的存放庫（`github_repository`：URL、選用的 `mount_path`/`checkout`；token 在 §4 來），要初始化的檔案（Files API 上傳 → `{type: "file", file_id, mount_path}`；唯讀），若工作有引用它們。
- **模型** — 預設 `claude-opus-5`；最難的長期工作使用 `claude-fable-5-1`（`shared/model-migration.md` → 遷移到 Claude Fable 5.1）。

> 重要：**PR 建立也需要 GitHub MCP 伺服器** — `github_repository` 掛載僅是檔案系統。在掛載中編輯 → 透過 `bash` 推送分支 → 透過 MCP `create_pull_request` 工具開啟 PR。

每個選項的完整說明：`shared/managed-agents-tools.md`（工具集、MCP、自訂工具、skills）、`shared/managed-agents-environments.md`（存放庫、檔案）。

## 3. 環境

通常只需零個或一個問題：

- **重用或建立？** 環境在 agent 之間共用——先檢查是否有現有的。
- **網路** — 預設不限制出口。只有在使用者需要出口控制時才切換到 `limited`——然後設定 `allow_mcp_servers: true` 或在 `allowed_hosts` 中列出每個 MCP 伺服器網域，否則這些工具會靜默失敗。
- **建議 `self_hosted`** 當有明確跡象時：工具必須在其自己的基礎設施上執行、密鑰不能離開它，或者他們需要雲端容器沒有的二進位檔/資料（`shared/managed-agents-self-hosted-sandboxes.md`；在 AWS 上的 Claude Platform 上，worker 以 IAM 而非環境金鑰進行驗證，且那裡的 session 無法附加記憶體存放區）。否則用 `cloud`——對於簡單工作不要主動提出。

## 4. Session — 驗證，然後測試執行

**驗證在這裡發生——收集 §2 中標記的憑證，現在設定已確定：** 一個 vault（現有的或 `vaults.create()`）+ 在 §2 中宣告的每個 MCP 伺服器的 `vaults.credentials.create()`、工作使用的 API 金鑰的 `environment_variable` 憑證（在出口處替換；沙盒看到的是佔位符），以及每個存放庫掛載的 `authorization_token`。憑證是只寫的；MCP 憑證依 URL 匹配伺服器並自動更新。請參見 `shared/managed-agents-tools.md` → Vault。

**靜默可行性閘門——在輸出任何內容之前自行執行此操作；只呈現缺口。** 逐條走過工作：每個動詞對應到已啟用的工具或 MCP 伺服器（「開啟 PR」→ GitHub MCP，而非只是掛載）；每個 MCP 伺服器和存放庫掛載都有來自驗證步驟的憑證；每個外部主機在網路選擇下可達；工作引用的每個檔案/存放庫/資料集都已掛載；「完成」是可查核的。若有缺漏，說明並解決——不要輸出您已知資源不足的設定。

**啟動——選一個，絕不兩個都選：**
- `user.message` — 對話式。
- `user.define_outcome` + 評分標準 — 當 §2 確定使用 Outcome 時；框架迭代並評分直到評分標準通過。
- **排程形式？** 完全跳過每個 session 的啟動——建立一個**部署**（`deployments.create()` 搭配 `schedule` + `initial_events`）；每次觸發都會自動建立 session。請參見 `shared/managed-agents-scheduled-deployments.md`。

要內建到執行時程式碼中的機制：session 建立時解析資源（錯誤的掛載在那裡就會出現，在 token 之前），但本身不配置沙盒；在送出啟動事件*之前*開啟事件串流；在 `session.status_terminated` 或帶有任何非 `requires_action` `stop_reason` 的 `session.status_idle` 時中斷——終止狀態，或 `budget_reached`（非終止，只有預算變更/移除才能恢復）（`shared/managed-agents-client-patterns.md` Pattern 5）；使用量在 `span.model_request_end` 上；產出物在 `/mnt/session/outputs/`（`files.list({scope_id: session.id, ...})`）。

## 5. 整合——輸出程式碼

從最後一個回答直接到程式碼——不要前言，不要關於設置與執行時的說教；兩區塊結構自然展示它。產生**兩個明確分離的區塊**：

**區塊 1 — 設置（執行一次，儲存 ID）。** 優先使用 **YAML 檔案 + `ant` CLI** — agent 和環境是版本控制的定義，使用者應該提交並從 CI 套用：

1. `<name>.agent.yaml`（扁平：`name`、`model`、`system`、`tools`、`mcp_servers`、`skills`）和 `<name>.environment.yaml`
2. ```sh
   AGENT_ID=$(ant beta:agents create < <name>.agent.yaml --transform id -r)
   ENV_ID=$(ant beta:environments create < <name>.environment.yaml --transform id -r)
   # CI sync: ant beta:agents update --agent-id "$AGENT_ID" --version N < <name>.agent.yaml
   ```

若使用者要求 SDK 備援——且在 AWS 上的 Claude Platform 上**必須使用 SDK**，因為驗證是 SigV4 而 `ant` CLI 沒有 SigV4 模式（使用 `shared/claude-platform-on-aws.md` 中的平台客戶端）：標記為 `# ONE-TIME SETUP — run once, save the IDs` 並呼叫 `environments.create()` → `agents.create()`。

> 警告：**部署比 MA 介面的其他部分更新。** 在輸出 `ant beta:deployments ...` 或 `client.beta.deployments` / `client.beta.deployment_runs` 呼叫之前，驗證使用者已安裝的 CLI/SDK 是否公開這些功能（`ant beta:deployments --help`；`hasattr(client.beta, "deployments")`）。若無，針對 `POST /v1/deployments` 輸出帶有 `managed-agents-2026-04-01` beta header 的原始 HTTP（使用 Bearer token 從 `ant auth print-credentials` 驗證時再加上 `oauth-2025-04-20`），並留下標記，指出哪些內容在升級後簡化為 SDK 呼叫。

**排程形式？部署是設置，而非執行時。** 在 agent/環境 ID 存在後，在區塊 1 中建立它（`deployments.create()` 搭配 `schedule` + `initial_events`）。區塊 2 就**不是** session 迴圈——沒有每次執行需要送出的啟動事件。改為輸出：一個手動執行觸發器（`POST /v1/deployments/{id}/run`），讓使用者可以立即測試而不必等待第一次觸發——手動執行兼作冒煙測試——再加上一個擷取 helper（最新的 `deployment_runs` 條目 → `session_id` → Console URL + `files.list(scope_id=session_id)` 取得產出物）。

**區塊 2 — 執行時（每次呼叫；對話和 Outcome 形式）。** 在偵測到的語言中的 SDK 程式碼（Python/TS/cURL——SKILL.md → 語言偵測）；不要在這裡輸出 shell 迴圈：

1. 從設定/環境載入 `agent_id` + `env_id`
2. `sessions.create(agent=AGENT_ID, environment_id=ENV_ID, resources=[...], vault_ids=[...])`，然後印出 Console URL 讓使用者即時觀看：`https://platform.claude.com/workspaces/default/sessions/{session.id}`（將 `default` 替換為其工作區代稱）
3. **當工作依賴 MCP 伺服器、憑證或受限主機時進行冒煙測試** — 這些失敗不在 `sessions.create()` 時出現，只在第一次使用時出現。一個廉價的探針輪次（「確認您可以連到 <service> 並列出 1-2 個項目；不要開始任務」），驗證後再送出真正的啟動事件。若沒有外部依賴則跳過。
4. 開啟串流 → 送出 §4 的啟動事件 → 以 §4 的終止閘門迴圈。

> 警告：**永遠不要在同一個未受防護的區塊中輸出 `agents.create()` 和 `sessions.create()`** — 這會教導每次執行都建立新 agent，這是第一大反模式。單一指令碼請求：將建立包在 `if not os.getenv("AGENT_ID"):` 中。

從 `{lang}/managed-agents/README.md` 中取得您偵測到的語言的精確語法（cURL 和 C#：使用 `curl/managed-agents.md` 作為線路層級參考）。不要自造欄位名稱。
