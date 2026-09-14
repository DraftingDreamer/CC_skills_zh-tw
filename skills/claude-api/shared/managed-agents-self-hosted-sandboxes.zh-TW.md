---
source_file: managed-agents-self-hosted-sandboxes.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 3228ab72afbe3ab1a0ac0f5a41995e1764c947cf17ce22a858b41b67262b71ee
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `managed-agents-self-hosted-sandboxes.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Managed Agents — 自託管沙盒

使用 `config.type: "self_hosted"` 時，**agent 迴圈保留在 Anthropic 的協作層**，但**工具執行移至您控制的基礎設施**——bash、檔案操作和程式碼在您的容器中執行，因此檔案系統內容和沙盒的網路流出都不會離開您的環境。（`web_search` / `web_fetch` 是例外：這兩種工具在兩種環境類型中都在 Anthropic 的伺服器上執行——請在 agent 工具集中用 `allowed_domains` / `blocked_domains` 限制它們，見 `shared/managed-agents-tools.md` § Web search 與 web fetch 設定。）工具輸入/輸出仍會流向 Anthropic 的控制平面，讓模型能看到結果；agent 的 skills 以及任何已附加記憶體存放區的內容，皆由 Anthropic 儲存並在 session 期間複製到您的沙盒中（記憶體變更會同步回去——見 § 記憶體存放區）。與 `config.type: "cloud"`（由 Anthropic 執行容器）形成對比。連線方式為**純出站**：您的 worker 對 Anthropic 的工作佇列進行長輪詢；Anthropic 絕不主動連入您的網路。

## 流程

```
1. Create environment:      config: {type: "self_hosted"}        -> env_...
2. Generate environment key (Console, on the environment page)   -> sk-ant-oat01-...  as ANTHROPIC_ENVIRONMENT_KEY
3. Run a worker:            EnvironmentWorker.run()  or  ant beta:worker poll
4. Sessions reference       environment_id=env_... exactly as for cloud
```

## 建立環境

```python
client = anthropic.Anthropic()

environment = client.beta.environments.create(
    name="self-hosted", config={"type": "self_hosted"}
)
```

`{"type": "self_hosted"}` 是完整的設定——沒有集區、容量或網路子欄位；這些由您自行控制。

## 執行 worker — SDK（主要路徑）

`EnvironmentWorker` 封裝了輪詢 → 派送 → 工具執行的迴圈。`.run()` 是永久執行迴圈（持續迴圈直到被取消）。`.handle_item()` / `.handleItem()` / `.HandleItem()` 在不輪詢的情況下服務**一個已認領**的工作項目——各 ID 依序退回至 `ANTHROPIC_WORK_ID` / `ANTHROPIC_ENVIRONMENT_ID` / `ANTHROPIC_SESSION_ID`，金鑰依序退回至 worker 自身的 `environment_key`、再退回至 `ANTHROPIC_ENVIRONMENT_KEY`，每個 session 的 secret 則退回至 `ANTHROPIC_WORK_SECRET`，因此在 `ant beta:worker poll --on-work` 容器內部它不需要任何引數。它會自行忽略（並強制停止）非 session 的工作項目。沒有 `run_one()` 這個方法；認領工作由 `.run()` 或由下方的中階輪詢器完成。

**Python — 永久執行：**

```python
import asyncio
import contextlib
import os
import signal
from anthropic import AsyncAnthropic
from anthropic.lib.environments import EnvironmentWorker


async def main() -> None:
    environment_key = os.environ["ANTHROPIC_ENVIRONMENT_KEY"]
    environment_id = os.environ["ANTHROPIC_ENVIRONMENT_ID"]
    async with AsyncAnthropic(auth_token=environment_key) as client:
        worker = EnvironmentWorker(
            client,
            environment_id=environment_id,
            environment_key=environment_key,
            workdir="/workspace",
        )
        task = asyncio.create_task(worker.run())
        # Cancel the task (don't kill the process): the worker stops its in-flight
        # work item and uploads changed memory files before exiting.
        loop = asyncio.get_running_loop()
        for signum in (signal.SIGINT, signal.SIGTERM):
            loop.add_signal_handler(signum, task.cancel)
        with contextlib.suppress(asyncio.CancelledError):
            await task


asyncio.run(main())
```

**TypeScript — 永久執行：**

```typescript
import Anthropic from "@anthropic-ai/sdk";
import { EnvironmentWorker } from "@anthropic-ai/sdk/helpers/beta/environments";

const environmentKey = process.env.ANTHROPIC_ENVIRONMENT_KEY!;
const environmentId = process.env.ANTHROPIC_ENVIRONMENT_ID!;
const client = new Anthropic({ authToken: environmentKey });
const ctrl = new AbortController();
process.once("SIGTERM", () => ctrl.abort());
process.once("SIGINT", () => ctrl.abort());

await new EnvironmentWorker({
  client,
  environmentId,
  environmentKey,
  workdir: "/workspace",
  signal: ctrl.signal
}).run();
```

**自訂工具。** `EnvironmentWorker` 預設執行內建工具集。若要新增或替換工具，請使用 `AgentToolContext(workdir=, client=, session_id=)` 搭配 `beta_agent_toolset(env)` / `betaAgentToolset(env)`，並將產生的工具傳遞給較低層級的 `tool_runner()`。附加到 agent 的 skills 在工具呼叫開始前會下載到 `{workdir}/skills/<name>/`（`AgentToolContext` 在給定 `client` 和 `session_id` 時處理此操作）。下載的 skill 檔案由 CLI 和 SDK 自動標記為可執行；若您自行實作 skills 下載，則需自行設定權限。

> **執行環境依賴：** SDK helper 需要 `/bin/bash` 位於該精確路徑（不透過 `PATH` 查詢）。TypeScript SDK 另外需要 `PATH` 上有 `unzip` 和 `tar`，以及 Node.js 22+；Python 和 Go 使用各自的標準函式庫進行封存檔解壓縮。記憶體存放區另外需要 POSIX 主機（Linux 或 macOS——不支援 Windows，worker 以 `O_NOFOLLOW` 開啟記憶體檔案），且該主機的 `/mnt/memory` 須可寫入——見 § 記憶體存放區。

**檔案工具的存取限制。** `AgentToolContext` 將 `read`/`write`/`edit`/`glob`/`grep` 限制在工作目錄加上 `allowed_roots`（`allowedRoots` / `AllowedRoots`）之內；`write` 和 `edit` 也會拒絕 `read_only_roots`（`readOnlyRoots` / `ReadOnlyRoots`）底下的路徑。`EnvironmentWorker` 會自行將 session 的記憶體存放區目錄加入這些清單。這只是檔案工具的防護機制——它**不會**限制 `bash`。舊有的 `unrestricted_paths` 選項已不再接受（傳入會引發錯誤）；請改為將目錄加入 `allowed_roots`。

## 執行 worker — `ant` CLI（固定工具）

`ant` CLI 附帶一個使用固定內建工具集（`bash`、`read`、`write`、`edit`、`glob`、`grep`）的 worker。依照 `shared/anthropic-cli.md` 安裝，然後：

```sh
export ANTHROPIC_ENVIRONMENT_KEY=sk-ant-oat01-...
ant beta:worker poll --environment-id env_... --workdir /workspace
```

- `--workdir` 是工具操作的目錄（預設 `.`）；工具呼叫被沙盒限制在其中。
- `--environment-key` 覆寫環境變數。
- `--on-work <script>` 在每個工作項目執行您的指令碼（例如為每個 session 啟動全新容器——見下方容器協作）。
- `--unrestricted-paths`、`--max-idle`（預設 `60s`）、`--log-format`——請執行 `ant beta:worker poll --help`。
- 旗標退回至環境變數（`ANTHROPIC_ENVIRONMENT_ID`、`ANTHROPIC_ENVIRONMENT_KEY`）。
- 在耗盡進行中的工作後，於 SIGTERM/SIGINT 上正常退出。
- **固定工具集**——如需自訂工具，請使用上方的 SDK worker。
- **不會掛載記憶體存放區。** 附加了存放區的 session 仍能執行，但 agent 在該存放區的 `/mnt/memory/<store-name>/` 目錄下找不到任何東西，也不會有任何內容同步回去。若要將 CLI 輪詢器與記憶體存放區結合使用，請將 `ant beta:worker poll --on-work` 留在主機上，並在每個 session 的沙盒內執行 **SDK** worker（`EnvironmentWorker.handle_item()`）——見 § 記憶體存放區 → 每個 session 一個沙盒。

在 `--on-work` 容器中，以 `ant beta:worker run --workdir <dir>` 作為進入點執行（若 session 需要記憶體存放區，則執行 SDK worker）。

## Webhook 驅動的喚醒（替代永久執行）

為 `session.status_run_started` 註冊 webhook（請參見 `shared/managed-agents-webhooks.md`），驗證傳送內容，然後用輪詢器**清空**佇列（`drain=True` 在佇列清空時停止；`block_ms=None` 為非阻塞；`auto_stop=False` 是因為 `handle_item` 會自行強制停止該項目），並將每個已認領的項目交給 `handle_item()`。**不要在 HTTP 處理常式內 `await` 清空動作**——session 執行的時間會超過 webhook 傳送逾時，因此請先確認送達，再將清空動作作為背景工作執行（`asyncio.create_task` / 已分離的 promise / 從 `context.Background()` 衍生的 goroutine），並讓處理程序保持存活直到它完成：

```python
import asyncio
import os
import anthropic

environment_key = os.environ["ANTHROPIC_ENVIRONMENT_KEY"]
environment_id = os.environ["ANTHROPIC_ENVIRONMENT_ID"]
client = anthropic.AsyncAnthropic(
    auth_token=environment_key,
)  # reads ANTHROPIC_WEBHOOK_SIGNING_KEY from env for webhooks.unwrap()


async def handle(raw: bytes, headers: dict[str, str]) -> dict:
    event = client.beta.webhooks.unwrap(raw.decode(), headers=headers)
    if event.data.type != "session.status_run_started":
        return {"status": "ignored"}
    asyncio.create_task(drain())  # keep a reference if your framework may GC it
    return {"status": "accepted"}


async def drain() -> None:
    async for work in client.beta.environments.work.poller(
        environment_id=environment_id,
        environment_key=environment_key,
        block_ms=None,
        reclaim_older_than_ms=2000,
        drain=True,
        auto_stop=False,
    ):
        await client.beta.environments.work.worker(workdir="/workspace").handle_item(
            work_id=work.id,
            environment_id=environment_id,
            session_id=work.data.id,
            environment_key=environment_key,
            work_secret=work.secret,  # lets the worker mount the session's memory stores
        )
```

TypeScript：形式相同，使用 `client.beta.webhooks.unwrap(body, {headers})`、`client.beta.environments.work.poller({environmentId, environmentKey, blockMs: null, reclaimOlderThanMs: 2000, drain: true, autoStop: false})`，以及 `client.beta.environments.work.worker({workdir}).handleItem({workId, environmentId, sessionId, environmentKey, workSecret: work.secret})`。Go：同樣沒有 `RunOne` 這種便利方法——`environments.NewWorkPoller(ctx, client, environments.WorkPollerOptions{EnvironmentID, EnvironmentKey, BlockMs: param.Null[int64](), ReclaimOlderThanMs: param.NewOpt[int64](2000), Drain: true, AutoStop: param.NewOpt(false)})`，然後對每個 `poller.Next()` 項目，在從 `context.Background()` 衍生的 goroutine 中執行 `worker.HandleItem(ctx, environments.HandleItemOptions{WorkID: item.ID, EnvironmentID: item.EnvironmentID, SessionID: item.Data.ID, EnvironmentKey, WorkSecret: item.Secret})`。務必傳遞工作項目的 `secret`，否則具有記憶體存放區的 session 會在認領時失敗。`handle_item` 會自行跳過非 session 的工作項目，因此清空迴圈不需要檢查 `work.data.type`。

## 容器協作（中階）

`EnvironmentWorker.run()` 在同一個處理程序中輪詢並執行工具。若要讓每個 session 在**各自的**容器中執行，請在薄型協作器中使用中階輪詢器——Python 為 `client.beta.environments.work.poller(environment_id=, environment_key=, drain=, block_ms=, reclaim_older_than_ms=, auto_stop=)`；TypeScript 為來自 `@anthropic-ai/sdk/helpers/beta/environments` 的 `new WorkPoller({client, environmentId, environmentKey, autoStop})`——並針對每個產生的 `work` 項目，啟動一個注入了這些環境變數的全新容器，其進入點執行 `ant beta:worker run` 或 `EnvironmentWorker(...).handle_item()`（若 session 附加了記憶體存放區則為必要）。`block_ms` 為 1–999（或 `None` 表示非阻塞）；`reclaim_older_than_ms` 重新認領租給已故 worker 的項目；`drain` 在佇列清空後停止；`auto_stop` 在迭代器退出後發布停止訊號（當啟動的容器擁有停止呼叫時設為 `False`）。Go：`environments.NewWorkPoller(ctx, client, environments.WorkPollerOptions{EnvironmentID, EnvironmentKey, BlockMs, ReclaimOlderThanMs, Drain, AutoStop: param.NewOpt(false)})`，搭配 `poller.Next()` / `poller.Current()` / `poller.Err()`。

| 環境變數 | 值 |
|---|---|
| `ANTHROPIC_SESSION_ID` | `work.data.id` |
| `ANTHROPIC_WORK_ID` | `work.id` |
| `ANTHROPIC_ENVIRONMENT_ID` | `work.environment_id` |
| `ANTHROPIC_ENVIRONMENT_KEY` | 直接傳遞 |
| `ANTHROPIC_BASE_URL` | 直接傳遞 |
| `ANTHROPIC_WORK_SECRET` | `work.secret`——worker 內部掛載記憶體存放區所需的每個 session 憑證。`ant beta:worker poll --on-work` **不會**為衍生的指令碼設定它；請從 stdin 上的工作項目 JSON 中讀取它（`jq -r '.secret // empty'`）並傳入。僅傳入服務該 session 的沙盒；絕不記錄它。 |

當您自行派送容器時，請跳過 `work.data.type != "session"` 的項目（`handle_item` 會為您執行此檢查）。

## 記憶體存放區

自託管環境上的 session 附加記憶體存放區的方式與雲端 session 完全相同——在建立 session 時使用 `resources=[{"type": "memory_store", "memory_store_id": ..., "access": ...}]`，每個 session 最多 8 個（見 `shared/managed-agents-memory.md`）。差別在於*由誰實體化它們*：在雲端環境中，Anthropic 會掛載即時的 FUSE 檔案系統；在自託管環境中，則由 **SDK worker**（`EnvironmentWorker`，或其 `handle_item()` / `handleItem()` / `HandleItem()`）下載一份工作複本並同步它。需要 Python、TypeScript 或 Go SDK；`ant` CLI worker 以及 C#/Java/PHP/Ruby SDK 不會掛載存放區。在 AWS 上的 Claude Platform 不可用。

當它認領一個 session 已附加存放區的工作項目時，**worker 會執行以下動作**：

1. 將每個存放區下載到 `/mnt/memory/` 底下的掛載路徑——該路徑由存放區的名稱衍生而來，不是可設定的欄位（例如名為「User Preferences」的存放區對應 `/mnt/memory/user-preferences/`）；與雲端 session 使用的路徑相同，且 session 的系統 prompt 會向 agent 描述它。使用工作項目的每個 session `secret` 進行驗證。
2. 將這些目錄新增到檔案工具的 `allowed_roots`，並將 `access: "read_only"` 的存放區新增到 `read_only_roots`，讓 agent 對記憶體使用一般的 `read`/`write`/`edit`/`glob`/`grep` 工具。
3. 在工具呼叫後進行協調，每個同步間隔最多一次（預設 15 秒）：遠端變更會寫入磁碟，agent 變更過的檔案會被上傳。
4. Session 結束時：進行最終同步、將待處理的上傳作業清空至多 30 秒、移除這些目錄。在 session 中途被*取消*的 worker 會跳過最終同步，但仍會上傳變更過的檔案並移除目錄；被*終止*的 worker 則完全不會執行收尾。

Anthropic 端的存放區仍是唯一真實來源——記憶體版本、版本編輯，以及 Console 檢視/編輯的運作方式與雲端 session 相同，且 agent 的記憶體讀寫會以一般工具事件的形式出現在事件串流中。由於同步是按間隔進行的，一個自託管 session 寫入的變更，只有在雙方都完成同步後（通常遠低於一分鐘），才會對另一個執行中的 session 可見；雲端 session 幾乎能立即看到彼此的變更。每個存放區目錄都有一個標記檔案 `.anthropic-memory-store`——請勿動它；標記檔案遺失或被更改的目錄，worker 不會對其進行同步。

**準備主機。** 僅限 POSIX（Linux/macOS）；建議使用區分大小寫的檔案系統。啟動 worker 之前：

```bash
sudo mkdir -p /mnt/memory && sudo chown "$USER" /mnt/memory
```

**請勿**自行建立每個存放區的目錄——worker 會在 session 啟動時建立每個存放區的目錄，**若該路徑下已存在某個東西則拒絕該工作項目**，並在 session 結束時將其移除。由此衍生兩條規則：(a) 兩個 session 無法在同一台主機上同時掛載相同的存放區（它們需要相同的路徑）——請讓每個 session 擁有自己的沙盒；(b) 正常停止 workers。`EnvironmentWorker` 不會安裝任何訊號處理常式：請自行將 SIGTERM/SIGINT 接上取消動作（TypeScript 為中止 `signal`、Go 為取消 context、Python 為取消執行 `run()` / `handle_item()` 的工作），傳送 SIGTERM，並在強制終止之前至少留出 30 秒。若 worker 在收尾之前被終止，請在下一個附加該存放區的 session 啟動前，移除 `/mnt/memory/` 底下遺留的目錄——其中未同步的編輯將會遺失。

**每個 session 一個沙盒**（來自 § 容器協作的模式）會自動滿足規則 (a)。請將 `ant beta:worker poll --on-work`（或 SDK 輪詢器）留在主機上；圍繞 SDK worker（而非 `ant beta:worker run`）建構每個 session 的映像——它的進入點會建構 `EnvironmentWorker` 並呼叫 `handle_item()`，此方法會從 `ANTHROPIC_*` 變數讀取 session/work/environment ID，並從 `ANTHROPIC_WORK_SECRET` 讀取每個 session 的 secret（或明確傳入 `work_secret=` / `workSecret` / `WorkSecret`）。`--on-work` 不會為產生指令碼設定 `ANTHROPIC_WORK_SECRET`，因此請從 stdin 上的工作項目 JSON 中讀取它：

```bash
#!/bin/bash
# spawn.sh - called once per claimed work item; the work item arrives as JSON on stdin
ANTHROPIC_WORK_SECRET="$(jq -r '.secret // empty')"
export ANTHROPIC_WORK_SECRET
exec docker run --rm \
  -e ANTHROPIC_SESSION_ID -e ANTHROPIC_WORK_ID -e ANTHROPIC_ENVIRONMENT_ID \
  -e ANTHROPIC_ENVIRONMENT_KEY -e ANTHROPIC_BASE_URL -e ANTHROPIC_WORK_SECRET \
  my-sdk-worker-image
```

每個 session 的進入點只有幾行——不需要任何引數，`handle_item()` 會讀取轉發過來的 `ANTHROPIC_*` 變數，包括 `ANTHROPIC_WORK_SECRET`；請將訊號接上取消動作，讓即使被停止的容器仍會完成上傳：

```python
import asyncio, contextlib, os, signal
from anthropic import AsyncAnthropic
from anthropic.lib.environments import EnvironmentWorker


async def main() -> None:
    async with AsyncAnthropic(auth_token=os.environ["ANTHROPIC_ENVIRONMENT_KEY"]) as client:
        task = asyncio.create_task(EnvironmentWorker(client, workdir="/workspace").handle_item())
        loop = asyncio.get_running_loop()
        for signum in (signal.SIGINT, signal.SIGTERM):
            loop.add_signal_handler(signum, task.cancel)
        with contextlib.suppress(asyncio.CancelledError):
            await task


asyncio.run(main())
```

TypeScript：`new EnvironmentWorker({ client, workdir: "/workspace", signal: controller.signal }).handleItem()`，搭配 `process.once("SIGTERM"/"SIGINT", () => controller.abort())`。Go：`signal.NotifyContext(ctx, os.Interrupt, syscall.SIGTERM)`，然後 `environments.NewEnvironmentWorker(client, environments.EnvironmentWorkerOptions{Workdir: "/workspace"}).HandleItem(ctx, environments.HandleItemOptions{})`。

映像需要可寫入的 `/mnt/memory`；記憶體目錄**不需要**繫結掛載到主機——worker 會在沙盒退出前完成上傳，且被捨棄的沙盒不會留下任何需要清理的東西。若要提早停止容器，請用一個會被進入點轉為取消動作的訊號，而非直接終止，這樣上傳才會照常執行。

**設定同步**——兩個 `EnvironmentWorker` 選項（Python 中為建構函式或 `client.beta.environments.work.worker()` 工廠函式；TypeScript 中為選項物件；Go 中為 `environments.EnvironmentWorkerOptions`）：

| 選項 | Python / TypeScript / Go | 行為 |
|---|---|---|
| 同步間隔 | `memory_sync_interval`（秒）/ `memorySyncIntervalMs`（毫秒）/ `MemorySyncInterval`（時間長度） | 預設 15 秒，最小值 5 秒。較短的間隔會縮小過時的時間窗，但代價是更多記憶體存放區請求。`None` / `null` / 負的時間長度會**完全停用記憶體支援**——存放區既不下載也不同步，即使 session 附加了存放區、其系統 prompt 仍會描述它們，但 session 執行時不會有這些存放區。僅在 session 從不附加存放區的 worker 上停用。啟用時，若附加存放區的 session 收到的工作項目沒有 `secret`，會**失敗**而非在沒有記憶體的情況下執行。 |
| 刪除傳播 | `memory_sync_deletes` / `memorySyncDeletes` / `MemorySyncDeletes` | `"enabled"`（預設——一旦後續同步確認檔案確實已消失，就從存放區中刪除）、`"log_only"`（相同的檢查，但只記錄本應刪除的內容——用於在信任 `enabled` 之前先進行稽核）、`"disabled"`（絕不從存放區刪除）。Go：`environments.MemorySyncDeletesEnabled`（零值）/ `LogOnly` / `Disabled`。上傳/下載不受影響。 |

舉例來說，每 10 秒同步一次，且只*記錄*本應刪除的內容：Python 為 `EnvironmentWorker(client, environment_id=..., environment_key=..., workdir="/workspace", memory_sync_interval=10, memory_sync_deletes="log_only")`；TypeScript 為 `new EnvironmentWorker({ client, environmentId, environmentKey, workdir: "/workspace", memorySyncIntervalMs: 10_000, memorySyncDeletes: "log_only" })`；Go 為 `environments.EnvironmentWorkerOptions{..., MemorySyncInterval: 10 * time.Second, MemorySyncDeletes: environments.MemorySyncDeletesLogOnly}`。

**唯讀存放區與衝突。** 對於 `access: "read_only"`，`write`/`edit` 會拒絕該目錄下的變更（這是唯一會以工具錯誤形式傳達給 agent 的記憶體錯誤），且不會有任何內容上傳；記憶體存放區端點也會拒絕以該 session 的 `secret` 進行的寫入。`bash` 的編輯不會在本地被封鎖——它們永遠不會同步，且下一次遠端變更會覆寫它們。衝突的解決方式**以存放區為準**：若 agent 變更的檔案自上次同步後在遠端也發生了變更，worker 會在下一次同步時保留存放區的版本、覆寫本地檔案，並記錄一則警告——`write`/`edit` 仍會成功，且不會有錯誤傳達給 agent；它可以重新讀取並重新套用。

**疑難排解。** 掛載和背景同步失敗會被*記錄*，但不會回報給 session。若存放區在認領時無法掛載，worker 會讓該工作項目失敗——session 不會發出任何錯誤事件，並停留在 `idle`（`requires_action` 停止原因）。

| 日誌行/症狀 | 原因 | 修法 |
|---|---|---|
| `the work item carried no sessions token`（Go：`ErrSessionMemoryNoToken`），工作項目失敗 | 每個 session 的 `secret` 未送達 worker——貴組織尚未啟用自託管記憶體，或您的產生指令碼未轉發它 | 將 `ANTHROPIC_WORK_SECRET` 轉發進沙盒。若處理程序內 worker（輪詢＋執行在同一個處理程序）仍記錄此訊息，請聯絡支援團隊 |
| `something already exists at the memory store's path` | 被終止的 worker 留下的殘留目錄 | 移除該指名的目錄（其中未同步的編輯會遺失） |
| `cannot create the memory store's folder` + `the worker host must make this mount path writable` | worker 的使用者無法在 `/mnt/memory` 底下建立目錄 | `mkdir -p /mnt/memory && chown <worker-user> /mnt/memory` |
| session 在認領後不久呈現 `idle` 並帶有 `requires_action`，沒有錯誤事件 | worker 在上述掛載錯誤上讓工作項目失敗 | 修復主機，然後傳送 `user.interrupt`——工作會被重新排入佇列，下一次認領會重試掛載 |

## 監控與控制

這些是**控制平面**呼叫——使用 `x-api-key`（而非環境金鑰）進行驗證；需要 `managed-agents-2026-04-01` beta header。**請從 worker 主機外部呼叫它們**——在 worker 主機上設定 `ANTHROPIC_API_KEY` 會將組織範圍的憑證暴露給 agent 工具呼叫。

| SDK（`client.beta.environments.work.*`） | REST | CLI | 回傳 |
|---|---|---|---|
| `stats(environment_id)` | `GET /v1/environments/{id}/work/stats` | `ant beta:environments:work stats` | `{type:"work_queue_stats", depth, pending, oldest_queued_at, workers_polling}` |
| `stop(work_id, environment_id=)` | `POST /v1/environments/{id}/work/{work_id}/stop` | `ant beta:environments:work stop` | `work.state` |

## 與 `cloud` 的差異

| 關注點 | `cloud` | `self_hosted` |
|---|---|---|
| 容器生命週期、強化、網路 | Anthropic | **您**——以非 root 執行、唯讀根檔案系統、移除特權；網路流出由您的 VPC/防火牆決定——`web_search` / `web_fetch` 除外，它們無論如何都在 Anthropic 的伺服器上執行（請針對每個工具用 `allowed_domains` / `blocked_domains` 限制它們） |
| `file` / `github_repository` 資源掛載 | Anthropic 掛載到容器 | **您**——透過 `sessions.create(metadata={...})` 傳遞指標，並由您的協作器在派送前擷取/複製 |
| `memory_store` 資源 | 由 Anthropic 掛載於 `/mnt/memory/<name>/`（即時 FUSE 掛載） | **透過 SDK worker 支援**（Python / TypeScript / Go 的 `EnvironmentWorker`），會將每個存放區下載到 `/mnt/memory/<store-name>/` 並按間隔同步——見 § 記憶體存放區。`ant` CLI worker 不會掛載它；C#、Java、PHP 或 Ruby SDK 不提供此功能。`memory_store` 是自託管環境接受的**唯一**資源類型——`file` / `github_repository` 仍會被拒絕，並回傳 400 訊息「Environment env_... is a self-hosted environment. `resources` are not supported with self-hosted environments.」（以自託管環境為目標的部署遵循相同規則；Console 的部署表單不為它們提供記憶體存放區選項——請使用 API/SDK） |
| Vault `environment_variable` 憑證 | 支援（在 Anthropic 管理的出口處替換） | **尚不支援**——出口由您負責，因此沒有地方可以替換該密鑰。請使用 MCP 憑證或主機端自訂工具（`shared/managed-agents-client-patterns.md` Pattern 9） |
| 內建工具 | 透過 `agent_toolset_20260401` | 由您的 worker 提供（`EnvironmentWorker` 預設 / `beta_agent_toolset(env)` / `ant` CLI 固定集） |
| Skills 下載 | 自動 | `EnvironmentWorker` / `AgentToolContext` 擷取到 `{workdir}/skills/`（需要 `client` + `session_id`） |
| AWS 上的 Claude Platform | 支援 | 支援——worker 使用 AWS IAM（SigV4）或 AWS Console 產生的 API 金鑰進行驗證（Console 產生的環境金鑰對 AWS 端點無效）；請將 `AnthropicSelfHostedEnvironmentAccess` 受管政策附加到 worker 的主體上。在那裡，自託管環境上的 session **無法附加記憶體存放區**（在建立 session 時即遭拒絕）；雲端環境則照常附加。 |
| SDK worker helper | 所有 SDK | **僅限 Python、TypeScript、Go**（`EnvironmentWorker` / 輪詢器不在 Java、Ruby、PHP 或 C# 中）——請使用這三者之一或 `ant` CLI |

## 憑證

| 憑證 | 格式 | 範圍 |
|---|---|---|
| `ANTHROPIC_ENVIRONMENT_KEY` | `sk-ant-oat01-...` | 一個環境的工作佇列。在 Console 中產生（「Generate environment key」）。作為 `auth_token=` / `authToken` 傳入客戶端，**且**作為 `environment_key=` / `environmentKey` 傳入 `EnvironmentWorker`。存放在密鑰管理器中；洩露時輪換。 |
| `ANTHROPIC_WEBHOOK_SIGNING_KEY` | `whsec_...` | Webhook 簽名驗證（若使用 webhook 驅動的喚醒）。SDK 在 `client.beta.webhooks.unwrap()` 中自動讀取此環境變數。 |
| 工作項目的 `secret`（`ANTHROPIC_WORK_SECRET`） | 每個 session 各自的值，由 Anthropic 在已認領的工作項目上核發 | 用於張貼（POST）該 session 的事件，以及讀寫附加給它的記憶體存放區。您不會自行產生它；處理程序內 worker 會從工作項目中取得它，而在每個 session 一個沙盒的模式下，您需要自行將它轉發進沙盒（或明確傳入 `work_secret=` / `workSecret` / `WorkSecret`）。請比照環境金鑰處理：只傳入服務該 session 的沙盒，絕不放進映像、共用磁碟區或日誌中。 |

## 安全性——您的責任

容器強化；沙盒的出口限制（沒有預設限制；伺服器端的 `web_search` / `web_fetch` 只由它們的 `allowed_domains` / `blocked_domains` 管控）；`ANTHROPIC_ENVIRONMENT_KEY` 的保管和輪換；在執行不受信任程式碼時每個信任邊界使用一個工作區 + 環境；工具處理程序的最小權限；日誌保留和清除。**Anthropic 無法**：快速吊銷洩露的環境金鑰、驗證您的映像或供應鏈、在您的容器內對工具執行進行沙盒化，或在工具輸出到達您的基礎設施後強制執行保留政策。**記憶體存放區**仍由 Anthropic 託管（含版本歷史），但 `/mnt/memory/` 底下的工作複本在 session 期間屬於您：worker 會在收尾時刪除它，被終止的 worker 會將其遺留下來，而共用檔案系統的 session 之間的權限/隔離則是您的責任。`read_only` 存放區受保護的對象是*上傳*，而非本地修改——`bash` 仍可變更本地複本（該 session 中後續的工具呼叫會讀到變更後的複本，直到存放區下一次變更該記憶體為止）；若 agent 連本地檢視都不得更動，請停用 `bash` 或以唯讀方式掛載該路徑。完整檢查清單請參見 `shared/live-sources.md` 中的自託管沙盒安全頁面。
