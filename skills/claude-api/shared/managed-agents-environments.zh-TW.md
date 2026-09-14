---
source_file: managed-agents-environments.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: faa3bb1a55a8bfc308d7884329ed2ed9fa2b9f64f39696bb51aeb3c69b86b264
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `managed-agents-environments.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Managed Agents — 環境與資源

## 環境

建立 session 需要 `environment_id`。環境是**可重複使用的設定模板**，用於在 Anthropic 基礎設施中啟動容器——您可能會為不同的使用案例建立不同的環境（例如資料視覺化 vs 網頁開發，使用不同的套件集）。Anthropic 負責擴展、容器生命週期和工作協作。

**環境名稱必須唯一。** 使用現有名稱建立環境會回傳 409。

### 網路

| 網路政策 | 說明 |
| --- | --- |
| `unrestricted` | 完整出口（法律封鎖清單除外） |
| `limited` | 預設拒絕；透過 `allowed_hosts` / `allow_package_managers` / `allow_mcp_servers` 選擇開放 |

```json
{
  "networking": {
    "type": "limited",
    "allow_package_managers": true,
    "allow_mcp_servers": true,
    "allowed_hosts": ["api.example.com"]
  }
}
```

三個 `limited` 欄位均為選填。`allow_package_managers`（預設 `false`）允許 PyPI/npm 等；`allow_mcp_servers`（預設 `false`）允許 agent 設定的 MCP 伺服器端點，無需將它們列在 `allowed_hosts` 中。

**MCP 注意事項：** 在 `limited` 網路下，請設定 `allow_mcp_servers: true` 或將每個 MCP 伺服器網域加入 `allowed_hosts`。否則容器無法連到它們，工具會靜默失敗。

**套件注意事項：** 在 `limited` 網路下，`packages` 需要 `allow_package_managers: true`；否則請求會以 400 失敗。只將 registry 列在 `allowed_hosts` 中是不夠的。

**`networking` 不控管 `web_search` / `web_fetch`。** 這些工具在 Anthropic 的伺服器上執行（雲端*和*自託管環境皆然），因此 `limited` 出口和 `allowed_hosts` 都不會限制它們。若要限制它們可連到的網站，請在 agent 工具集中該工具的 `configs` 條目上設定 `allowed_domains` / `blocked_domains`——請參見 `shared/managed-agents-tools.md` § Web search 與 web fetch 設定。

### 建立環境

SDK 會自動新增 `managed-agents-2026-04-01`。TypeScript：

```ts
const env = await client.beta.environments.create({
  name: "my_env",
  config: {
    type: "cloud",
    networking: { type: "unrestricted" },
  },
});
```

### 自託管沙盒

若要在**您自己的基礎設施**中執行工具，而非 Anthropic 的，請設定 `config: {type: "self_hosted"}` — agent 迴圈保留在 Anthropic 這端，但 `bash` / 檔案操作 / 程式碼在您透過出站輪詢 worker 控制的容器中執行。`networking` 區塊不適用（您控制出口）。資源掛載（`file`、`github_repository`）和記憶體存放區的行為不同——關於 worker、憑證和雲端與自託管的比較，請參見 `shared/managed-agents-self-hosted-sandboxes.md`。

### 環境 CRUD

| 操作 | 方法 | 路徑 | 備註 |
| --- | --- | --- | --- |
| 建立 | `POST` | `/v1/environments` | |
| 列出 | `GET` | `/v1/environments` | 分頁（`limit`、`after_id`、`before_id`） |
| 取得 | `GET` | `/v1/environments/{id}` | |
| 更新 | `POST` | `/v1/environments/{id}` | 變更只套用到**新的**容器；現有 session 保留原始設定 |
| 刪除 | `DELETE` | `/v1/environments/{id}` | 回傳 204。 |
| 封存 | `POST` | `/v1/environments/{id}/archive` | 使其**唯讀**；現有 session 繼續，新 session 無法引用它。無法取消封存——終止狀態。 |

---

## 資源

將檔案、GitHub 存放庫和記憶體存放區附加到 session。資源在 session 建立時解析，因此錯誤的 `file_id` 或無法連到的存放庫會在建立呼叫時呈現，而非在執行中途。建立 session **本身**不啟動工作或配置沙盒——沒有 `initial_events` 時 session 只是被登錄，沙盒在 session 首次需要時才啟動（請參見 `shared/managed-agents-core.md` → 用 `initial_events` 初始化 session）。每個 session 最多 **999 個檔案資源**。每個 session 支援多個 GitHub 存放庫。關於 `type: "memory_store"` 資源（持久跨 session 記憶體——每個 session 最多 8 個），請參見 `shared/managed-agents-memory.md`。

### 檔案上傳（輸入——主機 → agent）

先透過 Files API 上傳檔案，然後以 `file_id` + `mount_path` 引用：

```ts
// 1. Upload
const file = await client.beta.files.upload({
  file: fs.createReadStream("data.csv"),
  purpose: "agent",
});

// 2. Attach as a session resource
const session = await client.beta.sessions.create({
  agent: agent.id,
  environment_id: envId,
  resources: [
    { type: "file", file_id: file.id, mount_path: "/workspace/data.csv" }
  ],
});
```

**`mount_path` 為必填**，且必須是絕對路徑。父目錄會自動建立。Agent 工作目錄預設為 `/workspace`。檔案以唯讀方式掛載——agent 將修改後的版本寫入新路徑。

### Session 輸出（輸出——agent → 主機）

Agent 可在 session 期間將檔案寫入 `/mnt/session/outputs/`。這些檔案由 Files API 自動捕獲，之後可列出和下載：

```ts
// After the turn completes, list output files scoped to this session:
for await (const f of client.beta.files.list({
  scope_id: session.id,
  betas: ["managed-agents-2026-04-01"],
})) {
  console.log(f.filename, f.size_bytes);
  const resp = await client.beta.files.download(f.id);
  const text = await resp.text();
}
```

**需求：**
- Agent 必須啟用 `write` 工具（或 `bash`）才能建立輸出檔案。
- Session 範圍的 `files.list` / `files.download` 捕獲寫入 `/mnt/session/outputs/` 的輸出。
- 篩選參數是 **`scope_id`**（REST 查詢參數 `?scope_id=<session_id>`）。SDK 的 files 資源只自動新增 `files-api-2025-04-14` header，因此請明確傳入 `betas: ["managed-agents-2026-04-01"]`（或在原始 HTTP 上同時傳入兩個 header）——沒有它，API 可能以未知欄位拒絕 `scope_id`。需要 `@anthropic-ai/sdk` ≥ 0.88.0 / `anthropic`（Python）≥ 0.92.0——舊版不對 `scope_id` 進行型別化。`ant` CLI 目前**尚未**公開此旗標；請使用 SDK 或 curl。
- 請逐字傳入 `sessions.create()` 回傳的 session ID（例如 `sesn_011CZx...`）——API 會驗證前綴。
- 在 `session.status_idle` 和輸出檔案出現在 `files.list` 之間有短暫的索引延遲（約 1–3 秒）。若結果為空，請重試一兩次。

> **`scope_id` 篩選不可用時的備援方案**（舊版 SDK，或端點回傳錯誤）：發送後續 `user.message` 請求 agent 讀取 `/mnt/session/outputs/` 下的每個檔案並回傳內容。Agent 將檔案主體以 `agent.message` 文字串流回傳。這只適用於文字檔案且需要消耗輸出 token——用於解除阻塞，而非作為主要路徑。

這為您提供了雙向檔案橋：輸入上傳參考資料，輸出下載 agent 產出物。

### GitHub 存放庫

在初始化期間（agent 開始執行之前）將 GitHub 存放庫複製到 session 容器中。Agent 可透過 `bash`（`git`）讀取、編輯、提交和推送。每個 session 支援多個存放庫——每個存放庫新增一個 `resources` 條目。存放庫會快取，因此使用相同存放庫的未來 session 啟動更快。

掛載存放庫時，也會載入其根目錄 `.claude/skills` 中儲存的 skill——每個 session 探索一次，從 session 開始時簽出的存放庫狀態（僅限雲端沙盒）。請參見 `shared/managed-agents-tools.md` → 來自 GitHub 存放庫的 Skills。

存放庫在 session 的生命週期內附加——若要變更掛載的存放庫，請建立新的 session。您**可以**透過 `client.beta.sessions.resources.update(resource_id, {session_id, authorization_token})` 在執行中的 session 上輪換存放庫的 `authorization_token`；resource `id` 在 session 建立時和 `resources.list()` 中回傳。

**欄位：**

| 欄位 | 必填 | 備註 |
|---|---|---|
| `type` | 是 | `"github_repository"` |
| `url` | 是 | GitHub 存放庫 URL |
| `authorization_token` | 是 | 具有存放庫存取權限的 GitHub Personal Access Token。**永遠不會在 API 回應中回顯。** |
| `mount_path` | 否 | 存放庫複製的路徑。預設為 `/workspace/<repo-name>`。 |
| `checkout` | 否 | `{type: "branch", name: "..."}` 或 `{type: "commit", sha: "..."}`。預設為存放庫的預設分支。 |

**Token 權限層級**（細粒度 PAT）：
- `Contents: Read` — 僅複製
- `Contents: Read and write` — 推送變更和建立 pull request

**驗證方式：** `authorization_token` 永遠不會放入容器中。針對附加存放庫的 `git pull` / `git push` 和 GitHub REST 呼叫，會透過 Anthropic 端的 git proxy 路由，在請求離開沙盒後注入 token。容器中執行的程式碼——包括 agent 寫入的任何內容——無法讀取或洩漏它。

> 重要：**若要建立 pull request**，您還需要 GitHub **MCP 伺服器**存取——`github_repository` 資源只提供檔案系統 + git 存取。請參見 `shared/managed-agents-tools.md` → MCP 伺服器。PR 工作流程為：在掛載的存放庫中編輯檔案 → 透過 `bash` 推送分支（透過 git proxy 使用 `authorization_token` 驗證）→ 透過 MCP `create_pull_request` 工具建立 PR（透過 vault 驗證）。

**TypeScript：**

```ts
// 1. Create the agent - declare GitHub MCP (no auth here)
const agent = await client.beta.agents.create(
  {
    name: 'GitHub Agent',
    model: 'claude-opus-5',
    mcp_servers: [
      { type: 'url', name: 'github', url: 'https://api.githubcopilot.com/mcp/' },
    ],
    tools: [
      { type: 'agent_toolset_20260401', default_config: { enabled: true } },
      { type: 'mcp_toolset', mcp_server_name: 'github' },
    ],
  },
);

// 2. Start a session - attach vault for MCP auth + mount the repo
const session = await client.beta.sessions.create({
  agent: agent.id,
  environment_id: envId,
  vault_ids: [vaultId],  // vault contains the GitHub MCP OAuth credential
  resources: [
    {
      type: 'github_repository',
      url: 'https://github.com/owner/repo',
      authorization_token: process.env.GITHUB_TOKEN,  // repo clone token (!= MCP auth)
      checkout: { type: 'branch', name: 'main' },
    },
  ],
});
```

**Python：**

```python
import os

agent = client.beta.agents.create(
    name="GitHub Agent",
    model="claude-opus-5",
    mcp_servers=[{
        "type": "url",
        "name": "github",
        "url": "https://api.githubcopilot.com/mcp/",
    }],
    tools=[
        {"type": "agent_toolset_20260401", "default_config": {"enabled": True}},
        {"type": "mcp_toolset", "mcp_server_name": "github"},
    ],
)

session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=env_id,
    vault_ids=[vault_id],  # vault contains the GitHub MCP OAuth credential
    resources=[{
        "type": "github_repository",
        "url": "https://github.com/owner/repo",
        "authorization_token": os.environ["GITHUB_TOKEN"],  # repo clone token (!= MCP auth)
        "checkout": {"type": "branch", "name": "main"},
    }],
)
```

---

## Files API

上傳和管理用於 session 資源的檔案，以及下載 agent 寫入 `/mnt/session/outputs/` 的檔案。

| 操作 | 方法 | 路徑 | SDK |
| --- | --- | --- | --- |
| 上傳 | `POST` | `/v1/files` | `client.beta.files.upload({ file })` |
| 列出 | `GET` | `/v1/files?scope_id=...` | `client.beta.files.list({ scope_id, betas: ["managed-agents-2026-04-01"] })` |
| 取得中繼資料 | `GET` | `/v1/files/{id}` | `client.beta.files.retrieveMetadata(id)` |
| 下載 | `GET` | `/v1/files/{id}/content` | `client.beta.files.download(id)` → `Response` |
| 刪除 | `DELETE` | `/v1/files/{id}` | `client.beta.files.delete(id)` |

List 上的 `scope_id` 篩選將結果限定為該 session 寫入 `/mnt/session/outputs/` 的檔案。沒有篩選時，您會取得上傳到帳戶的所有檔案。
