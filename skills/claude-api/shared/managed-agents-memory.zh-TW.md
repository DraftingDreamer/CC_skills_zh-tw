---
source_file: managed-agents-memory.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 9f14213db20c1fcd1547e7c4882f9adde50b7b3f602b29684e5d851bf06d370a
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `managed-agents-memory.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Managed Agents — 記憶體存放區

> **公開測試版。** 記憶體存放區在 `managed-agents-2026-04-01` beta header 下發布；SDK 會在所有 `client.beta.memory_stores.*` 呼叫上自動設定。若 `client.beta.memory_stores` 不存在，請升級至最新 SDK 版本。

Session 預設是短暫的——當一個 session 結束時，agent 所學習的任何內容都會消失。**記憶體存放區**是一個跨 session 持久保存的、工作區範圍的小型文字文件集合。當存放區附加到 session 時（透過 `resources[]`），它會以檔案系統目錄的形式掛載到容器中；agent 使用普通的檔案工具讀寫它，而系統 prompt 中的說明會告知 agent 掛載點的位置。

每次對記憶體的修改都會產生一個不可變的**記憶體版本**（`memver_...`），提供稽核軌跡和時間點回滾/編輯功能。

> 警告：**請勿在記憶體存放區中儲存憑證、API 金鑰或 token。** 記憶體跨 session 持久保存，並會逐字傳回到後續情境中——寫入一次的金鑰會在後續每個掛載該存放區的 session 中重播。請改用 vault `environment_variable` 憑證（`shared/managed-agents-tools.md` → Vault）。若敏感資料已被寫入，請刪除該記憶體並編輯受影響的版本（見下方「編輯版本」）。

## 物件模型

| 物件 | ID 前綴 | 範圍 | 備註 |
| --- | --- | --- | --- |
| 記憶體存放區 | `memstore_...` | 工作區 | 透過 `resources[]` 附加到 session |
| 記憶體 | `mem_...` | 存放區 | 一個文字檔案，以 `path` 定址（各 ≤ 100KB——推薦使用多個小檔案） |
| 記憶體版本 | `memver_...` | 記憶體 | 每次修改的不可變快照；`operation` ∈ `created` / `modified` / `deleted` |

## 建立存放區

`description` 會傳遞給 agent，使其了解存放區的內容——請為模型而非人類撰寫。

```python
store = client.beta.memory_stores.create(
    name="User Preferences",
    description="Per-user preferences and project context.",
)
print(store.id)  # memstore_01Hx...
```

其他 SDK：TypeScript `client.beta.memoryStores.create({...})`；Go `client.Beta.MemoryStores.New(ctx, ...)`。完整的各語言表格請參見 `shared/managed-agents-api-reference.md` → SDK 方法參考。

存放區支援 `retrieve` / `update` / `list`（含 `include_archived`、`created_at_{gte,lte}` 篩選）/ `delete` / **`archive`**。封存使存放區變為唯讀——現有 session 附件繼續有效，新 session 無法引用；無法取消封存。

### 預載內容（選用）

在任何 session 執行之前預載參考資料。`memories.create` 在指定 `path` 建立記憶體；若該路徑已有記憶體，呼叫會回傳 `409`（`memory_path_conflict_error`，含 `conflicting_memory_id`）。存放區 ID 是第一個位置引數。

```python
client.beta.memory_stores.memories.create(
    store.id,
    path="/formatting_standards.md",
    content="All reports use GAAP formatting. Dates are ISO-8601...",
)
```

## 附加到 session

記憶體存放區放在 session 的 `resources[]` 陣列中，與 `file` 和 `github_repository` 資源並列（請參見 `shared/managed-agents-environments.md` → 資源）。記憶體存放區**僅能在 session 建立時**附加——`sessions.resources.add()` 不接受 `memory_store`。自託管環境上的 session 也以相同方式附加它們（且 `memory_store` 是那些環境**唯一**接受的資源類型）——見下方的自託管說明。

```python
session = client.beta.sessions.create(
    agent=agent.id,
    environment_id=environment.id,
    resources=[
        {
            "type": "memory_store",
            "memory_store_id": store.id,
            "access": "read_write",  # or "read_only"; default is "read_write"
            "instructions": "User preferences and project context. Check before starting any task.",
        }
    ],
)
```

| 欄位 | 必填 | 備註 |
| --- | --- | --- |
| `type` | 是 | `"memory_store"` |
| `memory_store_id` | 是 | `memstore_...` |
| `access` | — | `"read_write"`（預設）或 `"read_only"` — 在雲端掛載的檔案系統層級強制執行；在自託管沙盒上則由 worker 的 `write`/`edit` 工具和上傳路徑強制執行（見下方） |
| `instructions` | — | 此存放區的 session 特定指引，附加於存放區的 `name`/`description` 之後。≤ 4,096 字元。 |

**每個 session 最多 8 個記憶體存放區。** 當不同記憶體片段有不同擁有者或生命週期時可附加多個——例如一個唯讀共享參考存放區加上一個每個使用者的可讀寫存放區，或每個共用單一 agent 設定的終端使用者/團隊/專案一個存放區。

### Agent 如何看待它（FUSE 掛載）

每個附加的存放區掛載在 session 容器的 `/mnt/memory/<store-name>/` 路徑下。Agent 使用標準檔案工具（`bash`、`read`、`write`、`edit`、`glob`、`grep`）與其互動——沒有專用的記憶體工具。在雲端沙盒上，`access: "read_only"` 在檔案系統層級使掛載變為唯讀（在自託管沙盒上則由 worker 的 `write`/`edit` 工具和上傳路徑強制執行——見下方）；`"read_write"` 允許 agent 在其下建立、編輯和刪除檔案。每個掛載的簡短說明（名稱、路徑、`instructions`、存取權限）會自動注入系統 prompt，讓 agent 知道存放區存在，無需您特別提及。

Agent 在掛載下所做的寫入會持久保存回存放區，並產生記憶體版本，就像主機端的 `memories.update` 呼叫一樣。

**自託管沙盒：同步的本地複本，而非即時掛載。** 在 `self_hosted` 環境上，SDK worker（`EnvironmentWorker`——Python、TypeScript、Go；`ant` CLI worker 不掛載存放區）會將每個附加的存放區下載到相同的 `/mnt/memory/<store-name>/` 路徑，並定期與存放區進行協調，因此寫入只有在同步後才會對其他 session 可見，衝突以存放區為準解決，且 `read_only` 是由 worker 的工具而非檔案系統強制執行的（`bash` 仍可更改本地複本）。其他一切——同步間隔、每個 session 的 `secret`、主機準備、疑難排解——都記載於 `shared/managed-agents-self-hosted-sandboxes.md` § 記憶體存放區。在 Claude Platform on AWS 的自託管環境中不可用。

## 直接管理記憶體（主機端）

用於審查流程、更正錯誤記憶體，或帶外初始化存放區。

### 列出

回傳 `Memory | MemoryPrefix` 條目——`MemoryPrefix`（`type: "memory_prefix"`，僅含 `path`）是按階層列出時的目錄式節點。使用 `path_prefix` 來限定範圍（包含結尾斜線：`"/notes/"` 符合 `/notes/a.md` 但不符合 `/notes_backup/old.md`），並使用 `depth` 限制樹狀結構遍歷深度。傳入 `view="full"` 可在每個項目中包含 `content`；預設 `"basic"` 只回傳中繼資料。

```python
for m in client.beta.memory_stores.memories.list(store.id, path_prefix="/"):
    if m.type == "memory":
        print(f"{m.path}  ({m.content_size_bytes} bytes, sha={m.content_sha256[:8]})")
    else:  # "memory_prefix"
        print(f"{m.path}/")
```

### 讀取

```python
mem = client.beta.memory_stores.memories.retrieve(memory_id, memory_store_id=store.id)
print(mem.content)
```

`retrieve` 預設使用 `view="full"`（包含內容）；`view` 主要在列出端點上才有影響。

### 建立與更新

| 操作 | 定址方式 | 語義 |
| --- | --- | --- |
| `memories.create(store_id, path=..., content=...)` | **路徑** | 在 `path` 建立記憶體。若路徑已佔用，回傳 `409`（`memory_path_conflict_error`，含 `conflicting_memory_id`）。 |
| `memories.update(mem_id, memory_store_id=..., path=..., content=...)` | **`mem_...` ID** | 修改現有記憶體。變更 `content`、`path`（重新命名）或兩者。重新命名到已佔用的路徑會回傳相同的 `409 memory_path_conflict_error`。 |

```python
mem = client.beta.memory_stores.memories.create(
    store.id,
    path="/preferences/formatting.md",
    content="Always use tabs, not spaces.",
)

client.beta.memory_stores.memories.update(
    mem.id,
    memory_store_id=store.id,
    path="/archive/2026_q1_formatting.md",  # rename
)
```

### 樂觀並發（`update` 上的前置條件）

`memories.update` 接受 `precondition`，讓您可以先讀取再修改再寫回，而不會覆蓋並發寫入者的資料。唯一支援的類型是 `content_sha256`。不符合時 API 回傳 `409`（`memory_precondition_failed_error`）——重新讀取並針對最新狀態重試。

```python
client.beta.memory_stores.memories.update(
    mem.id,
    memory_store_id=store.id,
    content="CORRECTED: Always use 2-space indentation.",
    precondition={"type": "content_sha256", "content_sha256": mem.content_sha256},
)
```

### 刪除

```python
client.beta.memory_stores.memories.delete(mem.id, memory_store_id=store.id)
```

傳入 `expected_content_sha256` 可進行條件式刪除。

## 稽核與回滾——記憶體版本

每次修改都會建立不可變的 `memver_...` 快照。版本在父記憶體的生命週期內累積；`memories.retrieve` 永遠回傳當前最新版，版本端點提供歷史記錄。

| 觸發它的操作 | 版本上的 `operation` 欄位 |
| --- | --- |
| 在新路徑執行 `memories.create` | `"created"` |
| `memories.update` 變更 `content`、`path` 或兩者（或 agent 端對掛載的寫入） | `"modified"` |
| `memories.delete` | `"deleted"` |

每個版本還記錄 `created_by`——帶有 `type` ∈ `session_actor` / `api_actor` / `user_actor` 的執行者物件——以及在編輯後的 `redacted_at` + `redacted_by`。

### 列出版本

最新優先，分頁。可依 `memory_id`、`operation`、`session_id`、`api_key_id` 或 `created_at_gte` / `created_at_lte` 篩選。傳入 `view="full"` 可包含 `content`；預設僅回傳中繼資料。

```python
for v in client.beta.memory_stores.memory_versions.list(store.id, memory_id=mem.id):
    print(f"{v.id}: {v.operation}")
```

### 取得版本

```python
version = client.beta.memory_stores.memory_versions.retrieve(
    version_id, memory_store_id=store.id
)
print(version.content)
```

### 編輯版本

從歷史版本中清除內容，同時保留稽核軌跡（執行者 + 時間戳）。清除 `content`、`content_sha256`、`content_size_bytes` 和 `path`；其他一切保留。用於洩漏的密鑰、個人識別資訊或使用者刪除請求。

```python
client.beta.memory_stores.memory_versions.redact(version_id, memory_store_id=store.id)
```

## 端點參考

完整的 HTTP 方法/路徑表格請參見 `shared/managed-agents-api-reference.md` → 記憶體存放區 / 記憶體 / 記憶體版本。原始 HTTP 基礎路徑：

```
POST   /v1/memory_stores
POST   /v1/memory_stores/{memory_store_id}/archive
GET    /v1/memory_stores/{memory_store_id}/memories
PATCH  /v1/memory_stores/{memory_store_id}/memories/{memory_id}
GET    /v1/memory_stores/{memory_store_id}/memory_versions
POST   /v1/memory_stores/{memory_store_id}/memory_versions/{version_id}/redact
```

如需 cURL 範例及 CLI（`ant beta:memory-stores ...`），請 WebFetch `shared/live-sources.md` → Managed Agents 中的 Memory URL。
