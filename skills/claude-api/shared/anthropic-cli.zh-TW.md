---
source_file: anthropic-cli.md
source_commit: 34040c9c568585f6929bedeaad110ad08f079624
source_sha256: 4f8b4319fc096b48f311eff782a3b818a872933852ecb3e811157352cb618210
translated_at: 2026-09-13
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `anthropic-cli.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Anthropic CLI（`ant`）

`ant` CLI 將每個 Claude API 資源公開為 shell 子命令。與 `curl` 相比：請求主體由型別化旗標或以管道傳遞的 YAML 構建，而非手寫 JSON；`@path` 將檔案內容內嵌到任何字串欄位；`--transform` 以 GJSON 路徑提取欄位（無需 `jq`）；列出端點自動分頁（以 `--max-items N` 限制總結果；`--limit` 只設定伺服器頁面大小）；`beta:` 前綴自動設定正確的 `anthropic-beta` header。

## 何時使用 CLI vs SDK

**CLI 用於控制平面，SDK 用於資料平面。** Agent 和環境是相對靜態的資源，您用 `ant` 定義、設定和偵錯——將 YAML 提交到您的存放庫，從 CI 套用，從終端機檢查。Session 是動態的，由您的應用程式透過 SDK 驅動——每個任務建立、串流事件、回應工具呼叫、整合到您的產品。兩者都打同一個 API；分工在於呼叫位置，而非可能性。

| | 控制平面 → `ant` | 資料平面 → SDK |
|---|---|---|
| 資源 | agents、environments、skills、vaults、files | sessions、events |
| 頻率 | 每次部署一次 / 臨時 | 每個任務 / 每個輪次 |
| 所在位置 | 存放庫中的 `*.yaml` + CI + 終端機 | 應用程式碼 |
| 典型呼叫 | `create < agent.yaml`、`update --version N`、`list`、`retrieve`、`archive`、`--debug` | `sessions.create()`、`events.stream()`、`events.send()` |

## 安裝與驗證

```sh
# macOS
brew install anthropics/tap/ant
xattr -d com.apple.quarantine "$(brew --prefix)/bin/ant"

# Linux / WSL - pick the release from github.com/anthropics/anthropic-cli/releases
curl -fsSL "https://github.com/anthropics/anthropic-cli/releases/download/v${VERSION}/ant_${VERSION}_$(uname -s | tr A-Z a-z)_$(uname -m | sed -e s/x86_64/amd64/ -e s/aarch64/arm64/).tar.gz" \
  | sudo tar -xz -C /usr/local/bin ant

# Or from source (Go 1.22+)
go install github.com/anthropics/anthropic-cli/cmd/ant@latest
```

**驗證** — CLI 解析憑證的方式與 SDK 相同（第一個符合的獲勝）：明確旗標，然後 `ANTHROPIC_API_KEY`，然後 `ANTHROPIC_AUTH_TOKEN`，然後 `ANTHROPIC_PROFILE` 選取的或使用中的 profile，然後 Workload Identity Federation 環境變數，然後磁碟上的預設 profile。以 `ANTHROPIC_BASE_URL` 或 `--base-url` 覆寫主機。

- **API 金鑰**：在環境中設定 `ANTHROPIC_API_KEY`。
- **OAuth profile**（無需管理靜態金鑰）：`ant auth login` 開啟瀏覽器，交換短暫 token，並在 `$ANTHROPIC_CONFIG_DIR` 下儲存 profile（Linux/macOS 預設為 `~/.config/anthropic/`，Windows 為 `%APPDATA%\Anthropic`——設定使用 `configs/<profile>.json`，token 使用 `credentials/<profile>.json`）。後續的 `ant`（和 SDK）呼叫自動取用——登入後裸露的 `Anthropic()` 客戶端即可運作，但直接讀取 `ANTHROPIC_API_KEY` 的指令碼則不行。Claude Code 和 Claude Agent SDK 遵循相同的 profile 解析。`ant auth status` 顯示哪個憑證來源和 profile 獲勝（僅報告狀態——不要將其退出碼作為健康檢查使用）；`ant auth logout` 清除使用中的 profile（`--all` 清除所有 profile）。在沒有瀏覽器的遠端主機上，`ant auth login --no-browser` 印出授權 URL 並在終端機接受回傳的代碼。
- **非互動式工作負載**（CI、伺服器、容器）：互動式登入是為了在您自己的機器上進行開發——改用 Workload Identity Federation（請透過 `shared/live-sources.md` 查看驗證文件）。

> **第一大驗證陷阱：** profile 只在沒有設定 API 金鑰時才被諮詢。過期匯出的 `ANTHROPIC_API_KEY` 會靜默覆寫每個 profile——請求會打到該金鑰所屬的任何組織/工作區。`ant auth status` 顯示哪個來源獲勝；在依賴 profile 之前請取消設定金鑰（或每個命令：`env -u ANTHROPIC_API_KEY ant …`）。**真正地**取消設定它——空的 `ANTHROPIC_API_KEY=""` 仍然獲得其優先順序位置，並以空金鑰驗證。相同的覆蓋反向適用於 Claude Code：在 `ant auth login` 之後，Claude Code 可能會警告 profile 和其自己的 `/login` 憑證之間的驗證衝突——保留一個（使用 profile 並在 Claude Code 中 `/logout`，或 `ant auth logout` 保留 Claude Code 自己的登入）。

**命名 profile** — 互動式登入 token 綁定到單一組織+工作區，API 只顯示屬於該工作區的資源。如果您建立的 agent、session 或檔案「消失了」，通常原因是 token 的範圍是與建立它的工作區不同的工作區（`ant auth status` 顯示使用中的工作區）。多工作區工作意味著每個工作區一個 profile：

```sh
ant auth login --profile <name>                  # creates the profile if it doesn't exist; org/workspace picker in browser
ant auth login --profile <name> --workspace-id wrkspc_01...   # bind directly, skip the picker
ant profile activate <name>                      # switch the default profile
ant --profile <name> models list                 # one-off; equivalent: ANTHROPIC_PROFILE=<name> ant models list
ant profile list                                 # inspect
ant profile set workspace_id wrkspc_01... --profile <name>    # edit config keys (workspace_id, base_url, organization_id, ...)
```

`ant profile set` 編輯現有 profile 的設定——它從不建立 profile，且**不**重新綁定已發布的憑證；請在該 profile 下再次執行 `ant auth login` 以為新目標發行 token。將 `ANTHROPIC_PROFILE` 指向不存在的 profile 是錯誤，而非降級。更新 token 最終會強制過期（不隨使用而滑動）——當先前運作的 profile 開始驗證失敗時，請在偵錯其他任何事情之前重新執行 `ant auth login`。

**範圍** — profile 的 OAuth 範圍集在登入時請求（`--scope`），並持久保存在 profile 上（`scope` 也是 `profile set` 設定鍵；像其他設定編輯一樣，更改它需要新的 `ant auth login` 才能生效）。特權範圍——例如組織管理端點的 `org:admin`——**不**在預設範圍集中：明確傳入您想要的完整集合（`ant auth login --profile admin --scope "... org:admin"`），且伺服器只在您的角色實際擁有它時授予特權範圍。由於範圍集附著在 profile 發行的每個 token 上，請將特權工作放在專用 profile（`admin` vs `default`），日常推理使用非特權的，以 `--profile`/`ANTHROPIC_PROFILE` 切換。查看 `ant auth login --help` 了解當前範圍列表，查看 `ant auth status` 了解使用中 token 攜帶的範圍。

將使用中的憑證傳遞給子處理程序或原始 HTTP 指令碼：

```sh
# Bare access token - for curl's Authorization header
curl https://api.anthropic.com/v1/messages \
  -H "Authorization: Bearer $(ant auth print-credentials --access-token)" \
  -H "anthropic-version: 2023-06-01" \
  -H "anthropic-beta: oauth-2025-04-20" \
  -H "content-type: application/json" \
  -d '{"model": "claude-opus-5", "max_tokens": 1024, "messages": [{"role": "user", "content": "Hello"}]}'

# .env format - sets ANTHROPIC_AUTH_TOKEN (and ANTHROPIC_BASE_URL if the profile has one).
# Output is bare KEY=value (no `export`), so use `set -a` to auto-export for child processes:
set -a; eval "$(ant auth print-credentials --env)"; set +a
python my_script.py   # SDK picks up ANTHROPIC_AUTH_TOKEN
```

OAuth token 放在 `Authorization: Bearer`（而非 `x-api-key:`）**加上 `anthropic-beta: oauth-2025-04-20` header** — 從 API 金鑰轉換原始 curl/httpx 指令碼是 header 變更，而非金鑰替換。beta header 需求依端點而異（某些端點在沒有它的情況下也能運作；`/v1/messages` 則不行）——請始終送出它，以免切換端點時請求中斷。token 是短暫的，透過環境變數傳遞時不會自動更新，因此在長時間執行的指令碼中過期前請重新執行 `print-credentials`（`print-credentials` 本身在需要時會更新 token）。如果同時設定了 `ANTHROPIC_API_KEY` 和 `ANTHROPIC_AUTH_TOKEN`，SDK 會同時送出兩者，API 拒絕請求——在 `eval` `--env` 輸出之前請取消設定 `ANTHROPIC_API_KEY`。

**陷阱：** 不帶旗標的 `ant auth print-credentials` 印出完整的憑證 JSON，而非裸 token——將它放在 `Authorization` header 中會產生空回應或 HTTP/2 協定錯誤。Headers 請始終使用 `--access-token`（它始終讀取命名/使用中的 profile；設定的 `ANTHROPIC_API_KEY` 不覆寫憑證印出）。

## 命令結構

```
ant <resource>[:<subresource>] <action> [flags]
```

Beta 資源（agents、sessions、environments、deployments、skills、vaults、memory stores）位於 `beta:` 下——CLI 自動送出正確的 `anthropic-beta` header，因此不要自行傳入，除非使用 `--beta <header>` 覆寫。對於自託管環境，`ant beta:worker poll/run` 和 `ant beta:environments:work stats/stop` 驅動和監控工作佇列——請參見 `shared/managed-agents-self-hosted-sandboxes.md`。

```sh
ant models list
ant messages create --model claude-opus-5 --max-tokens 1024 --message '{role: user, content: "Hello"}'
ant beta:agents retrieve --agent-id agent_01...
ant beta:sessions:events list --session-id session_01...
```

`ant --help` 列出資源；在任何子命令後附加 `--help` 查看其旗標。

## 全域旗標

| 旗標 | 用途 |
| --- | --- |
| `--format` | `auto`（預設：TTY 時美化輸出，管道時緊湊輸出）、`json`、`jsonl`、`yaml`、`pretty`、`raw`、`explore`（互動式 TUI） |
| `--transform` | 套用到回應的 GJSON 路徑（在列出端點上按項目套用）。不在 `--format raw` 時套用。 |
| `-r`、`--raw-output` | 若轉換結果是字串，不帶引號印出（jq 語義）。與 `--transform` 搭配用於純量擷取。 |
| `--max-items` | 限制自動分頁列出端點回傳的總結果（與 `--limit` 不同，後者是伺服器頁面大小）。 |
| `--format-error` / `--transform-error` | 與 `--format`/`--transform` 相同，套用到錯誤回應。`-r` 不適用於錯誤路徑——用 `--format-error yaml` 取得不帶引號的錯誤純量。 |
| `--base-url` | 覆寫 API 主機 |
| `--debug` | 印出完整的 HTTP 請求 + 回應到 stderr（API 金鑰已編輯） |

## 輸出——`--transform` + `--format`

`--transform` 接受 [GJSON 路徑](https://github.com/tidwall/gjson/blob/master/SYNTAX.md)。在列出端點上它**按項目**執行，而非在信封上。

```sh
ant beta:agents list --transform '{id,name,model}' --format jsonl
```

**提取純量用於 shell：** 搭配 `-r`（`--raw-output`——不帶引號印出字串，jq 風格）：

```sh
AGENT_ID=$(ant beta:agents create --name "My Agent" --model '{id: claude-sonnet-5}' \
  --transform id -r)
```

## 輸入——旗標、stdin、`@file`

**旗標** — 純量欄位直接對應。結構化欄位接受寬鬆 YAML 語法（無引號鍵）或嚴格 JSON。可重複旗標建立陣列（每個 `--tool`、`--event`、`--message` 追加一個元素）：

```sh
ant beta:agents create \
  --name "Research Agent" \
  --model '{id: claude-opus-5}' \
  --tool '{type: agent_toolset_20260401}' \
  --tool '{type: custom, name: search_docs, input_schema: {type: object, properties: {query: {type: string}}}}'
```

**stdin** — 以管道傳遞完整的 JSON 或 YAML 主體。與旗標合併；衝突時旗標優先（對陣列欄位，任何旗標**替換**整個 stdin 陣列——而非追加）。引號括住 heredoc 分隔符（`<<'YAML'`）以在主體內停用 shell 展開：

```sh
ant beta:agents create <<'YAML'
name: Research Agent
model: claude-opus-5
system: |
  You are a research assistant. Cite sources for every claim.
tools:
  - type: agent_toolset_20260401
YAML
```

**`@file` 參考** — 將檔案內容內嵌到任何字串值欄位。在結構化旗標值內，引號括住路徑。二進位檔自動 base64；使用 `@file://`（文字）或 `@data://`（base64）強制指定。使用 `\@` 跳脫開頭的字面 `@`。

```sh
ant beta:agents create --name "Researcher" --model '{id: claude-sonnet-5}' --system @./prompts/researcher.txt

ant messages create --model claude-opus-5 --max-tokens 1024 \
  --message '{role: user, content: [
    {type: document, source: {type: base64, media_type: application/pdf, data: "@./scan.pdf"}},
    {type: text, text: "Extract the text from this scanned document."}
  ]}' \
  --transform 'content.0.text' -r
```

原生接受檔案路徑的旗標（例如 `beta:files upload` 上的 `--file`）接受不帶 `@` 的裸路徑。

## 版本控制的 Managed Agents 資源

這是定義 agent 和環境的推薦流程——將 YAML 提交到您的存放庫，透過 `create`（第一次）/ `update`（之後）同步。欄位參考請參見 `shared/managed-agents-core.md`。

```yaml
# summarizer.agent.yaml
name: Summarizer
model: claude-sonnet-5
system: |
  You are a helpful assistant that writes concise summaries.
tools:
  - type: agent_toolset_20260401
```

```sh
# Create (once) - capture the ID
AGENT_ID=$(ant beta:agents create < summarizer.agent.yaml --transform id -r)

# Update (CI) - needs ID + current version (optimistic lock)
ant beta:agents update --agent-id "$AGENT_ID" --version 1 < summarizer.agent.yaml
```

相同的模式用於環境（`ant beta:environments create|update < env.yaml`），然後用兩個 ID 啟動 session：

```sh
ant beta:sessions create --agent "$AGENT_ID" --environment-id "$ENV_ID" --title "Task"
ant beta:sessions:events send --session-id "$SID" \
  --event '{type: user.message, content: [{type: text, text: "Summarize X"}]}'
ant beta:sessions:events list --session-id "$SID" --transform 'content.0.text' -r
ant beta:sessions:events stream --session-id "$SID"   # live event stream
```

### 將終端機連接到 session（`ant beta:sessions connect`）

`ant beta:sessions connect <session-id>` 將您的終端機連接到既有的 session：它載入逐字稿、即時跟進，並讓您介入——送出訊息、中斷，或允許/拒絕等待核准的工具呼叫。Ctrl+C 中斷連接；session 繼續執行，重新連接會重新載入完整歷史記錄。若 session 已 `terminated` 或封存則為唯讀。

```sh
ant beta:sessions connect sesn_011CZkZAtmR3yMPDzynEDxu7          # terminal view
ant beta:sessions connect sesn_011CZkZAtmR3yMPDzynEDxu7 --web    # Console session viewer, served locally
```

| 按鍵 | 動作 |
|---|---|
| Enter | 以 `user.message` 送出輸入（Alt+Enter / Ctrl+J 換行） |
| Esc | 中斷正在執行的 agent（`user.interrupt`） |
| Ctrl+O | 切換詳細資訊：工具輸入/結果、token 使用量、狀態事件（`--verbose` / `-v` 以展開模式啟動） |
| PgUp / PgDn | 捲動；向上捲動暫停跟進，End 恢復 |
| Ctrl+C（或在空白輸入上按 Ctrl+D）| 中斷連接 |

當呼叫等待核准時（`always_ask`，或 `auto` 策略下伺服器未能做出判定），輸入列變為 **Allow tool call?**，選項有 **Yes** / **No** / **No, and tell the agent why**——CLI 送出 `user.tool_confirmation`，您輸入的理由作為 `deny_message`。在多 agent session 中，終端機檢視器只跟進主執行緒（包含協調者與子 agent 之間的訊息）。

`--web` 從本機 `127.0.0.1` 伺服器提供 Console 的 session 檢視器，印出 URL 並開啟瀏覽器（`--no-browser` 可跳過）。URL 只能使用一次，有效期兩分鐘（重新整理同一個分頁沒問題；若要在其他地方開啟，請重新執行指令）。頁面僅與本機 `ant` 程序通訊，由它發起 API 呼叫，因此憑證永不離開 CLI；伺服器執行直到 Ctrl+C。與終端機檢視器不同，瀏覽器檢視器可跟進多 agent session 的每個執行緒。

需要互動式終端機（`--web` 除外）——指令碼請使用 `ant beta:sessions:events stream` / `send`（見下文）。

### 互動式 session 迴圈（先開串流再送事件）

`ant beta:sessions:events stream` 只傳送串流開啟*之後*發出的事件——因此在送出啟動事件之前先開啟它，以避免錯過早期事件。使用 process substitution（行程替換）將串流保持在檔案描述符上，送出，然後讀取：

```sh
exec {stream}< <(ant beta:sessions:events stream --session-id "$SID" \
  --transform '{type,text:content.#(type=="text").text,err:error.message}' --format yaml)

ant beta:sessions:events send --session-id "$SID" > /dev/null <<'YAML'
events:
  - type: user.message
    content:
      - type: text
        text: Summarize the repo README
YAML

type=
while IFS= read -r -u "$stream" line; do
  case "$line" in
    type:\ session.status_idle) break ;;
    type:\ session.error)
      IFS= read -r -u "$stream" next || next=
      case "$next" in err:\ *) msg=${next#err: } ;; *) msg=unknown ;; esac
      printf '\n[Error: %s]\n' "$msg"; break ;;
    type:\ *) type=${line#type: } ;;
    text:*)
      [[ $type == agent.message ]] || continue
      val=${line#text: }
      case "$val" in '|-'|'|') ;; *) printf '%s' "$val" ;; esac ;;
    \ \ *)
      if [[ $type == agent.message ]]; then printf '%s\n' "${line#  }"; fi ;;
  esac
done
exec {stream}<&-
```

這適用於互動式探索和示範。對於需要回應 `agent.tool_use` / `agent.custom_tool_use` 事件、在斷線後重連或對 `events.list` 去重的應用程式碼，請使用 SDK——請參見 `shared/managed-agents-client-patterns.md`。

## 指令碼模式

列出端點上的 `--transform id -r` 每行輸出一個裸 ID——與 `xargs` 組合，或使用 `--max-items N` 限制結果集而無需透過管道傳給外部工具：

```sh
FIRST=$(ant beta:agents list --transform id -r --max-items 1)
ant beta:agents:versions list --agent-id "$FIRST" --transform '{version,created_at}' --format jsonl
```

錯誤塑形反映成功路徑（注意：`-r` 不適用於錯誤輸出——在這裡用 `--format-error yaml` 取得不帶引號的純量）：

```sh
ant beta:agents retrieve --agent-id bogus --transform-error error.message --format-error yaml 2>&1
```

Shell 自動補全：`ant @completion {zsh|bash|fish|powershell}`。

如需完整的、始終最新的參考（包含每個端點的旗標），請 WebFetch `shared/live-sources.md` 中的 **Anthropic CLI** URL。
