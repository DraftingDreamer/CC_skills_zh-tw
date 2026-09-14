---
source_file: SKILL.md
source_commit: 34040c9c568585f6929bedeaad110ad08f079624
source_sha256: 8227f0d1192594bfea04df6c9f1d9d727dd300c3f4b2c42d315dd9fe58b1a421
translated_at: 2026-09-13
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`SKILL.md`](SKILL.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
>
> **原檔 YAML frontmatter**
> - `name`: `claude-api`
> - `description`（中譯）：Claude API／Anthropic SDK 的參考資料，涵蓋模型 ID、定價、參數、串流、工具使用、MCP、agent、快取、token 計數與模型遷移。觸發條件：在開啟目標檔案前必須先讀取本 skill，不要因為目標「看起來只是一行」就跳過——只要 prompt 以任何形式提到 Claude／Anthropic（Claude、Anthropic、Fable、Opus、Sonnet、Haiku、`anthropic`、`@anthropic-ai`、`claude-*`、`us.anthropic.*`、`[1m]`），或使用者詢問 LLM（定價、模型選擇、限制、快取；絕不可憑記憶回答），或任務具有 LLM 形態但未指明供應商（agent／MCP／工具定義／多 agent／RAG／LLM 評審／電腦使用；對自然語言進行生成、摘要、擷取、分類、重寫、對話；偵錯拒絕、截斷、串流、工具呼叫、token）。只有在處理其他供應商時跳過（覆蓋所有觸發條件）：查詢明確提到 OpenAI／GPT／Gemini／Llama／Mistral／Cohere／Ollama，或對專案執行 `grep -rE 'openai|langchain_openai|google.generativeai|genai|mistralai|cohere|ollama'` 有命中（若未指明供應商，先執行這個 grep，不要先讀檔案）。
> - `license`: Complete terms in LICENSE.txt
<!-- /translation-header -->

# 使用 Claude 建構 LLM 驅動的應用程式

本 skill 協助您使用 Claude 建構 LLM 驅動的應用程式。根據需求選擇正確的開發介面，偵測專案語言，然後閱讀相關的語言專屬文件。

## 開始之前

掃描目標檔案（或若無目標檔案，則掃描 prompt 和專案），尋找非 Anthropic 供應商標記——`import openai`、`from openai`、`langchain_openai`、`OpenAI(`、`gpt-4`、`gpt-5`、檔名如 `agent-openai.py` 或 `*-generic.py`，或任何保持程式碼供應商中立的明確指示。若發現任何標記，停下來告訴使用者本 skill 產生 Claude/Anthropic SDK 程式碼；詢問是否要將檔案切換到 Claude，或想要非 Claude 的實作。不要用 Anthropic SDK 呼叫編輯非 Anthropic 的檔案。（例外：`prompt-audit` 子命令是非互動式的，不在此停止——它在報告的已陳述假設中記錄非 Anthropic 供應商標記，且從不提議將非 Anthropic 檔案切換到 Anthropic SDK。）

## 輸出需求

當使用者要求新增、修改或實作 Claude 功能時，程式碼必須透過以下其中一種方式呼叫 Claude：

1. **適用於專案語言的官方 Anthropic SDK**（`anthropic`、`@anthropic-ai/sdk`、`com.anthropic.*` 等）。只要支援的語言有 SDK，預設使用此方式。
2. **原始 HTTP**（`curl`、`requests`、`fetch`、`httpx` 等）——只有在使用者明確要求 cURL/REST/原始 HTTP、專案是 shell/cURL 專案，或該語言沒有官方 SDK 時才使用。

不要混用兩者——不要在 Python 或 TypeScript 專案中使用 `requests`/`fetch`，僅因為感覺更輕量。不要退回到 OpenAI 相容的轉換層。

**永遠不要猜測 SDK 用法。** 函式名稱、類別名稱、命名空間、方法簽名和 import 路徑必須來自明確的文件——本 skill 中的 `{lang}/` 檔案或 `shared/live-sources.md` 中列出的官方 SDK 存放庫或文件連結。若需要的繫結在 skill 檔案中沒有明確記載，在撰寫程式碼之前先從 `shared/live-sources.md` WebFetch 相關的 SDK 存放庫。不要從 cURL 結構或其他語言的 SDK 推斷 Ruby/Java/Go/PHP/C# API。

**若 WebFetch 或存放庫存取失敗**（網路受限、逾時、複製被封鎖）：不要持續重試——從 `{lang}/` 檔案中的模式和命名空間/套件表撰寫程式碼，在上面執行編譯器或直譯器，並根據錯誤輸出迭代。對於靜態型別的 SDK（C#、Java、Go），針對本機錯誤的編譯修復迴圈比被封鎖的網路研究更快達到可運行的程式碼。

## 預設設定

除非使用者另有要求：

對於 Claude 模型版本，請使用 Claude Opus 5，可透過確切的模型字串 `claude-opus-5` 存取。對於任何稍微複雜的事情，請預設使用自適應思考（`thinking: {type: "adaptive"}`）。最後，對於任何可能涉及長輸入、長輸出或高 `max_tokens` 的請求，請預設使用串流——它可以防止遭遇請求逾時。若不需要處理個別串流事件，使用 SDK 的 `.get_final_message()` / `.finalMessage()` 輔助方法取得完整回應。

## ⚠️ API 漂移——訓練先驗知識可能已過時

幾種常見的 Claude API 結構在 2025–2026 年發生了變化。若從訓練中回想起某個模式，在撰寫之前先對照本 skill 中的 `{lang}/` 檔案驗證——下表是最常見的漂移點：

| 領域 | 過時的先驗知識 | 當前 API |
|---|---|---|
| 擴展思考 | `thinking: {type: "enabled", budget_tokens: N}` | 在 Claude 4.6+ 模型上：`thinking: {type: "adaptive"}`。`budget_tokens` 在 Opus 4.6 / Sonnet 4.6 上已棄用，**在 Fable 5/5.1 / Sonnet 5 / Opus 5 / 4.8 / 4.7 上以 400 拒絕**。4.6 以前的模型仍使用 `budget_tokens`。 |
| 網路搜尋/網路擷取工具類型 | `web_search_20250305`、`web_fetch_20250910` | `web_search_20260209`、`web_fetch_20260209`（動態過濾）適用於 Opus 5/4.8/4.7/4.6、Sonnet 5、Sonnet 4.6。舊模型保留基本變體；在 Vertex AI 上只有基本 `web_search_20250305` 可用（Vertex 上沒有網路擷取）——見下方伺服器工具快速參考。 |
| PHP 參數名稱 | snake_case 電線名稱作為具名引數（`max_tokens`） | 頂層具名引數為 camelCase（`maxTokens`）。巢狀陣列鍵依功能而異（例如 `'taskBudget'`、`'skillID'`、`'mcp_server_name'`）——從文件範例複製確切的鍵；不要批量轉換。 |
| Managed Agents 憑證 | 透過自訂工具在主機端保管機密（保管庫推出前的唯一選項） | 保管庫 `environment_variable` 憑證——由 Anthropic 儲存，在出口時替換，在沙箱中永遠不可見（`shared/managed-agents-tools.md` → Vaults）。自訂工具在主機端仍是自託管沙箱的備用方案。 |
| Files API / Skills | `client.beta.files.*` / `client.beta.skills.*`，帶有 beta `files-api-2025-04-14` / `skills-2025-10-02` | 已出 beta：`client.files.*` / `client.skills.*`，無需 beta 標頭。當前 SDK 中 `client.beta.files` / `client.beta.skills` 相較先前版本有破壞性的結構變更，與穩定命名空間匹配——依 `shared/live-sources.md` → Files API / Skills Guide 遷移。 |

本 skill 中的 `{lang}/` 檔案優先於回憶的模式。

---

## 子命令

若對話最底部的使用者請求是純粹的子命令字串（沒有散文），搜尋本文件所有 **Subcommands** 表格——包括附加到下方各節的任何表格——並直接遵循匹配的 Action 欄。這讓使用者可以透過 `/claude-api <subcommand>` 呼叫特定流程。若文件中沒有匹配的表格，將請求視為一般散文。

| 子命令 | 動作 |
|---|---|
| `migrate` | 將現有的 Claude API 程式碼遷移到更新的模型。**立即閱讀 `shared/model-migration.md`** 並依序遵循：步驟 0（確認範圍——在任何編輯之前詢問哪些檔案/目錄）、步驟 1（分類每個檔案），然後是各目標的重大變更章節。不要總結指南——執行它。若使用者未指定目標模型，在詢問範圍問題的同一輪中詢問要遷移到哪個模型。套用各目標變更後，依照 `shared/prompt-audit.md` 審核範圍內的 prompt 文字、工具描述和請求程式碼——為舊模型撰寫的 prompt 是每次遷移的一部分，而且它不會自行宣告。 |
| `prompt-audit` | 審核現有的 prompt、skill 和工具描述，找出為舊模型撰寫的過時模式（「殘留物」）。**立即閱讀 `shared/prompt-audit.md`** 並依序遵循：步驟 0（從請求和存放庫確定範圍與目標模型——在報告中陳述假設，不要停下來詢問）、清查、溯源，然後模式掃描。完整產出兩份交付物——審核報告（帶有 `file:line`、模式、為何對目標模型已過時、信心度的發現）和建議的 diff——無需暫停確認；只有在請求明確要求時才套用編輯。不要總結指南——執行它。 |
| `upgrade` | 跨主要版本升級專案的 Anthropic SDK 相依——目前是 Python SDK，`anthropic` 0.x → 1.x。尾隨的字詞可以指定語言和/或範圍（`upgrade python`、`upgrade python sdk src/`）。**立即閱讀 `python/claude-api/sdk-upgrade.md`** 並依序遵循：步驟 0（確認範圍，然後確定當前和目標版本——在寫入 pin 之前必須有已發布的 1.x），步驟 1 清查，每個編號章節，然後是驗證和報告。不要總結指南——執行它。若偵測到或指定的語言在本 skill 中沒有 `sdk-upgrade.md`，說明該 SDK 尚未捆綁主要版本升級指南，並指向該 SDK 的 CHANGELOG（`shared/live-sources.md` 中的存放庫）；不要從 Python 指南即興創作一個。這不是模型遷移——要將程式碼移到更新的 Claude 模型，請使用 `migrate`。 |
| `cost-optimize` | 降低現有 Claude API 程式碼的執行成本，同時不犧牲輸出品質。**立即閱讀 `shared/cost-optimization.md`** 並依序遵循：步驟 0（確定範圍、品質標準和基準），token 設定檔——當使用者有 Admin API 金鑰時透過 Usage and Cost Admin API 測量，當應用程式有自己的 `response.usage` 記錄時從中取得（詢問），否則從程式碼估算——然後節省排名最高的槓桿清單（以美元、帳單的百分比或相對等級報價，取決於您有哪些資料來源），免費勝利（快取、輸入 token 整理、迴圈整理、輸出 token 整理、批次）在取捨（預算、努力、模型選擇、多模型）之前；任何獲得一席之地的槓桿都成為其自己的 diff——預設提議，當使用者要求並批准時套用並對照覆蓋其流量的評測衡量——且「不建議變更」是一個有效的結果。兩條常規：每次執行模型的執行都花費真實資金，所以先獲得使用者的批准；當某個槓桿缺少情境時，與使用者互動地完成它——此工作流程預計不會一次完成審核。不要總結指南——執行它；將設定檔和排名計劃呈現給使用者是執行它的一部分。 |

---

## 語言偵測

在閱讀程式碼範例之前，確定使用者使用的語言（例外：對於 `prompt-audit` 子命令，跳過本節的詢問步驟——審核是非互動式的，其清查與語言無關；當無法推斷語言時，不詢問繼續並在報告中陳述假設）：

1. **查看專案檔案**以推斷語言：

   - `*.py`、`requirements.txt`、`pyproject.toml`、`setup.py`、`Pipfile` → **Python** — 從 `python/` 讀取
   - `*.ts`、`*.tsx`、`package.json`、`tsconfig.json` → **TypeScript** — 從 `typescript/` 讀取
   - `*.js`、`*.jsx`（無 `.ts` 檔案）→ **TypeScript** — JS 使用相同的 SDK，從 `typescript/` 讀取
   - `*.java`、`pom.xml`、`build.gradle` → **Java** — 從 `java/` 讀取
   - `*.kt`、`*.kts`、`build.gradle.kts` → **Java** — Kotlin 使用 Java SDK，從 `java/` 讀取
   - `*.scala`、`build.sbt` → **Java** — Scala 使用 Java SDK，從 `java/` 讀取
   - `*.go`、`go.mod` → **Go** — 從 `go/` 讀取
   - `*.rb`、`Gemfile` → **Ruby** — 從 `ruby/` 讀取
   - `*.cs`、`*.csproj` → **C#** — 從 `csharp/` 讀取
   - `*.php`、`composer.json` → **PHP** — 從 `php/` 讀取

2. **若偵測到多種語言**（例如同時有 Python 和 TypeScript 檔案）：

   - 檢查使用者目前的檔案或問題與哪種語言相關
   - 若仍有歧義，詢問：「我偵測到 Python 和 TypeScript 檔案。您在 Claude API 整合中使用哪種語言？」

3. **若無法推斷語言**（空專案、無原始碼檔案或不支援的語言）：

   - 使用 AskUserQuestion，選項為：Python、TypeScript、Java、Go、Ruby、cURL/raw HTTP、C#、PHP
   - 若 AskUserQuestion 不可用，預設為 Python 範例並注記：「顯示 Python 範例。若需要其他語言，請告訴我。」

4. **若偵測到不支援的語言**（Rust、Swift、C++、Elixir 等）：

   - 建議從 `curl/` 使用 cURL/raw HTTP 範例，並注意社群 SDK 可能存在
   - 提供以 Python 或 TypeScript 範例作為參考實作

5. **若使用者需要 cURL/raw HTTP 範例**，從 `curl/` 讀取。

### 語言專屬功能支援

上述每種 SDK 語言都同時支援 beta Tool Runner 和 Managed Agents（beta）——Python（`@beta_tool` 裝飾器）、TypeScript（`betaZodTool` + Zod）、Java（標註類別）、Go（`toolrunner` 套件中的 `BetaToolRunner`）、Ruby（`BaseTool` + `tool_runner`）、C#（`BetaToolRunner` + 原始 JSON 綱要）、PHP（`BetaRunnableTool` + `toolRunner()`）；程式碼進入點在下方的 Tool Use Patterns 快速參考中。cURL 是原始 HTTP（無 SDK 功能）並支援 Managed Agents。

> **Managed Agents 程式碼範例**：見下方 `## Managed Agents (Beta)` 章節中的閱讀指南。

---

## 我應該使用哪個介面？

> **從簡單開始。** 預設使用能滿足需求的最簡單層級。單一 API 呼叫和工作流程能處理大多數使用案例——只有在任務真正需要開放式的、模型驅動的探索時才使用 agent。「最簡單」意味著擁有最少的程式碼：對於託管、排程或記憶體支援的 agent，Managed Agents 通常是最簡單的選項（沒有迴圈程式碼、沒有狀態檔案、沒有排程器），即使它是更大的平台。

| 使用案例 | 層級 | 推薦介面 | 原因 |
| --- | --- | --- | --- |
| 分類、摘要、擷取、問答 | 單一 LLM 呼叫 | **Claude API** | 一個請求，一個回應 |
| 批次處理或嵌入 | 單一 LLM 呼叫 | **Claude API** | 特殊端點 |
| 程式碼控制邏輯的多步驟管線 | 工作流程 | **Claude API + tool use** | 您自行編排迴圈 |
| 帶有自訂工具的自訂 agent | Agent | **Claude API + tool use** | 最大彈性 |
| 帶有工作區的伺服器管理有狀態 agent | Agent | **Managed Agents** | Anthropic 執行迴圈並託管工具執行沙箱 |
| 持久化、版本化的 agent 設定 | Agent | **Managed Agents** | Agent 是儲存物件；session 綁定到版本 |
| 帶有檔案掛載的長時間多輪 agent | Agent | **Managed Agents** | 每個 session 的容器、SSE 事件串流、Skills + MCP |
| 按排程執行的 agent（cron、「每晚」） | Agent | **Managed Agents** — 排程部署 | 部署自動觸發 session；無需用戶端排程器 |

> **注意：** 當希望 Anthropic 執行 agent 迴圈*並*託管工具執行的容器時，Managed Agents 是正確的選擇——檔案操作、bash、程式碼執行都在每個 session 的工作區中執行。若想自行託管計算資源或執行自訂工具執行時，Claude API + tool use 是正確的選擇——使用 tool runner 進行 agentic 迴圈——其每輪鉤子仍給您審批閘道、日誌記錄、錯誤攔截和條件執行（見 `shared/tool-use-concepts.md`）——或在想完全擁有整個迴圈時使用手動迴圈。

> **雲端供應商存取。** **Claude Platform on AWS** 是 Anthropic 營運的，具有同日 API 同等性——請見 `shared/claude-platform-on-aws.md` 進行用戶端設定。關於 **Claude Platform on AWS**、**Amazon Bedrock**、**Google Vertex AI** 和 **Microsoft Foundry** 的各功能可用性，請見 `shared/platform-availability.md`——該表格是本 skill 的唯一可信來源；不要從其他地方推斷可用性。

### 建構 Agent：四種方法

一旦確實需要 agent（開放式、模型驅動的工具使用），有四種不同的建構方式。兩個獨立的問題區分它們：**誰提供框架**（agent 迴圈 + 情境管理）和**誰提供部署**（agent 執行的基礎設施）。Tool Runner 和 Claude Agent SDK 都只提供*框架*——您仍需自行託管和部署——這就是為什麼它們容易混淆。Managed Agents（CMA）是唯一同時提供**框架**和**託管部署**的選項；手動迴圈兩者都不提供。

| # | 方法 | 您撰寫 | 框架和部署 | 可用工具 | 使用時機 |
|---|---|---|---|---|---|
| 1 | **Claude API — 手動迴圈** | 自行撰寫 `while stop_reason == "tool_use"` 迴圈 | 您建構框架；您自行託管 | 只有您定義的工具 | 想完全擁有*整個*迴圈——沒有 beta 相依，或 Tool Runner 的每輪鉤子不適合的控制流程 |
| 2 | **Claude API — Tool Runner**（`client.beta.messages.tool_runner` + `@beta_tool` / `betaZodTool`） | 只是工具函式 | SDK 提供迴圈（**僅框架**）；您自行託管 | 只有您定義的工具 | 無需手動撰寫迴圈的自訂工具 agent（大多數情況）。每輪鉤子仍提供審批閘道、錯誤攔截、結果修改（例如 `cache_control`）、重試、串流和壓縮 |
| 3 | **Managed Agents**（REST、beta） | Agent 設定 + 您的工具結果 | Anthropic 提供框架**並**託管每個 session 的沙箱（**框架 + 部署**） | Anthropic 託管的沙箱（bash、檔案、程式碼執行）+ Skills/MCP + 您的工具 | 想要 Anthropic 執行迴圈*並*託管每個 session 的工作區；持久化/版本化的設定；長時間執行的 session |
| 4 | **Claude Agent SDK** — *獨立產品*（`claude-agent-sdk` / `@anthropic-ai/claude-agent-sdk`） | 一個 prompt + 選項 | SDK 提供 Claude Code 框架 + 內建工具（**僅框架**）；您自行託管 | 內建的 Read/Write/Edit/Bash/Glob/Grep/WebSearch/WebFetch + MCP + 子 agent | 想要一個包含所有功能的編碼/檔案系統 agent 在自己的基礎設施上執行 |

框架/部署的分離是關鍵的心智模型：選項 1、2、4 都**把部署留給您**；只有選項 3（CMA）新增了託管部署。選項 1–3 是本 skill 生成的內容；選項 4 是另一個有自己文件的函式庫——見下方的釐清說明。

> **Tool Runner ≠ Claude Agent SDK。** 這兩者聽起來很像但卻是不同的套件：
> - **Tool Runner** 是常規 Anthropic API SDK（`anthropic` / `@anthropic-ai/sdk`）的一部分，透過 `client.beta.messages.tool_runner` 存取。它自動化請求 → 執行 → 迴圈週期*針對您定義的工具*。沒有內建工具、沒有檔案系統存取、沒有沙箱——您提供每個工具並託管計算資源。它是上方的選項 2，是 `POST /v1/messages` 之上的薄輔助層。
> - **Claude Agent SDK**（`claude-agent-sdk` / `@anthropic-ai/claude-agent-sdk`）是打包為函式庫的 Claude Code。它包含內建工具（檔案讀/寫/編輯、bash、grep、網路搜尋）、完整的 agent 迴圈、情境管理、鉤子、子 agent、權限和 session。您呼叫 `query(prompt, options)`，它驅動一切。
>
> 兩者都是**僅框架——您自行託管和部署。** 差別在於框架的範圍：Tool Runner 迴圈*您*定義的工具（帶有每輪鉤子用於審批、攔截、結果修改和重試——但沒有內建工具）；Agent SDK 是帶有內建工具的完整 Claude Code 框架。兩者都不提供託管部署——那是 **Managed Agents（CMA）** 新增的（Anthropic 託管迴圈和每個 session 的沙箱）。
>
> **本 skill 涵蓋 Claude API 和 Managed Agents（選項 1–3）；不生成 Claude Agent SDK 程式碼。** 若使用者實際上想要 Claude Agent SDK，指引他們到其文件（`code.claude.com/docs/en/agent-sdk`）——不要用 API Tool Runner 替代它，反之亦然。

### 我應該建構 Agent 嗎？

在選擇 agent 層級之前，檢查所有四個標準：

- **複雜度** — 任務是多步驟且難以提前完全規定的嗎？
- **價值** — 結果是否值得更高的成本和延遲？
- **可行性** — Claude 能完成這類任務嗎？
- **錯誤成本** — 錯誤能否被捕獲和恢復？（測試、審查、回滾）

若對這些問題中的任何一個回答是「否」，停留在更簡單的層級（單一呼叫或工作流程）。

---

## 架構

一切都透過 `POST /v1/messages` 進行。工具和輸出限制是這個單一端點的功能——不是獨立的 API。

**使用者定義的工具** — 您定義工具（透過裝飾器、Zod 綱要或原始 JSON），SDK 的 tool runner 處理呼叫 API、執行您的函式，並迴圈直到 Claude 完成。若要完全控制，可以手動撰寫迴圈。

**伺服器端工具** — 在 Anthropic 基礎設施上執行的 Anthropic 託管工具。程式碼執行完全在伺服器端（在 `tools` 中宣告，Claude 自動執行程式碼）。電腦使用可以是伺服器託管或自託管。

**結構化輸出** — 限制 Messages API 回應格式（`output_config.format`）和/或工具參數驗證（`strict: true`）。推薦的方法是 `client.messages.parse()`，它會自動根據您的綱要驗證回應。注意：舊的 `output_format` 參數已棄用；在 `messages.create()` 上使用 `output_config: {format: {...}}`。

**支援端點** — Batches（`POST /v1/messages/batches`）、Files（`POST /v1/files`）、Token Counting（`POST /v1/messages/count_tokens`——見 `shared/token-counting.md`）和 Models（`GET /v1/models`、`GET /v1/models/{id}`——即時能力/情境視窗探索）用於支援 Messages API 請求。

---

## 當前模型（快取：2026-06-24）

| 模型 | 模型 ID | 情境 | 輸入 \$/1M | 輸出 \$/1M |
| --- | --- | --- | --- | --- |
| Claude Fable 5.1 | `claude-fable-5-1` | 1M | \$10.00 | \$50.00 |
| Claude Mythos 5.1（僅 Project Glasswing） | `claude-mythos-5-1` | 1M | \$10.00 | \$50.00 |
| Claude Fable 5 | `claude-fable-5` | 1M | \$10.00 | \$50.00 |
| Claude Opus 5 | `claude-opus-5` | 1M | \$5.00 | \$25.00 |
| Claude Opus 4.8 | `claude-opus-4-8` | 1M | \$5.00 | \$25.00 |
| Claude Opus 4.7 | `claude-opus-4-7` | 1M | \$5.00 | \$25.00 |
| Claude Opus 4.6 | `claude-opus-4-6` | 1M | \$5.00 | \$25.00 |
| Claude Sonnet 5 | `claude-sonnet-5` | 1M | \$2.00 | \$10.00 |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | 1M | \$3.00 | \$15.00 |
| Claude Haiku 4.5 | `claude-haiku-4-5` | 200K | \$1.00 | \$5.00 |

**合作夥伴定價：** 上述價格為 Anthropic 第一方 API 費率——同樣適用於 Microsoft Foundry 上的 Claude，透過 Microsoft Marketplace 按標準 API 費率計費。Amazon Bedrock 和 Vertex AI 上的 Claude 由合作夥伴營運，定價不同——請見 [Bedrock](https://aws.amazon.com/bedrock/pricing/) 或 [Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/pricing#claude-models)。若要 WebFetch，使用 `shared/live-sources.md` 中的 Pricing 行。

**除非使用者明確指定不同的模型，否則始終使用 `claude-opus-5`。** 這是不可商量的。不要使用 `claude-sonnet-5`、`claude-sonnet-4-6` 或任何其他模型，除非使用者字面上說「使用 sonnet」或「使用 haiku」。不要為了成本降級——那是使用者的決定，不是您的。只有當使用者明確要求 Claude Fable 5.1、「fable」或 Anthropic 最強大的模型時，才使用 `claude-fable-5-1`——它與 Opus 系列有不同的 API 行為（見下文）以及超過 Opus 級別的定價。**只使用表格中確切的模型 ID 字串——它們本身已完整；不要附加日期後綴**（`claude-sonnet-4-6`，絕不是 `claude-sonnet-4-6-20251114` 或任何其他附有日期後綴的變體）。若使用者要求不在表格中的舊模型（例如「opus 4.5」、「sonnet 3.7」），閱讀 `shared/models.md` 以獲取確切的 ID——不要自行建構。

### Claude Fable 5.1（`claude-fable-5-1`）——最強大的廣泛發布模型

Claude Fable 5.1 是 Anthropic 最強大的廣泛發布模型，適用於最苛刻的推理和長期 agentic 工作；以下內容也適用於 **Claude Mythos 5.1**（`claude-mythos-5-1`，Project Glasswing——相同的能力、定價和 API 介面；它執行依賴存取程式的安全措施，所以下方的 `refusal` 處理也適用於此；Claude Mythos 5 的繼任者，Claude Mythos 5 不執行任何安全分類器）。1M 情境視窗（最大值也是預設值），128K 最大輸出。與 Opus 級別的關鍵 API 差異——詳見 `shared/model-migration.md` → 遷移到 Claude Fable 5.1：

- **思考始終開啟** — 完全省略 `thinking` 參數（或發送 `{type: "adaptive"}`）。任何其他明確設定都會被拒絕：`{type: "disabled"}` 和 `{type: "enabled", budget_tokens: N}` 都回傳 400。用 `output_config.effort` 控制深度（支援 `low` 至 `xhigh` 和 `max`）。
- **不回傳原始思維鏈** — 回應帶有常規 `thinking` 區塊（不是 `redacted_thinking`）：`display: "summarized"` 回傳可讀摘要，`"omitted"`（預設）將 `thinking` 欄位保留為空字串。重放規則：在同一個模型上原樣回傳思考區塊；其他模型靜默地忽略它們（不計費——無需刪除；Claude Mythos 5.1 改為讀取它們）；詳情在 `shared/model-migration.md`。
- **分詞器** — 與 Opus 4.8 相同的分詞器（在 Opus 4.7 中引入）。從 Opus 4.7/4.8 遷移時 token 計數大致不變；每個 token 的定價不同。從 Opus 4.6、Sonnet、Haiku 或更舊版本來的，用 `count_tokens` 重新建立基準（Opus 4.7 分詞器使用大約 1×–1.35× 的 token）。
- **`refusal` 停止原因——處理它，並預設選擇備援** — 安全分類器可能拒絕請求（HTTP 200、`stop_reason: "refusal"`，帶有 `stop_details` 類別）；在讀取 `content` 之前始終檢查 `stop_reason`。**在撰寫 `claude-fable-5-1` 或 `claude-opus-5` 程式碼時，預設包含伺服器端 `fallbacks` 參數。** 最簡形式：`betas: ["server-side-fallback-2026-07-01"]` + `fallbacks: "default"`，它按拒絕類別路由，這樣您永遠不需要維護模型列表。（舊的陣列形式——`betas: ["server-side-fallback-2026-06-01"]` + `fallbacks: [{"model": "claude-opus-4-8"}]`——仍然有效；Claude API 和 Claude Platform on AWS——在 Bedrock、Vertex 和 Foundry 上，改用 SDK 的用戶端 `BetaRefusalFallbackMiddleware` + `BetaFallbackState`）。告訴使用者您已啟用它；只有在他們拒絕時才刪除。完整語義（計費、中途串流拒絕、信用重新定價）在 `shared/model-migration.md` → refusal 章節。**各語言的程式碼範例在 `{lang}/claude-api/README.md` § Refusal Fallbacks，只涵蓋陣列形式**——對於 `"default"` 模式，遵循 `shared/model-migration.md` → 遷移到 Claude Opus 5 → 新 API 功能中的原始 HTTP 結構，並將 `fallbacks: [{...}]` 替換為 `fallbacks: "default"` 加上 `-2026-07-01` 標頭；其餘請求不變。
- **無助手預填** — 與其餘 4.6+ 系列相同。
- **需要 30 天資料保留** — Claude Fable 5.1 在零資料保留下不可用，除非 Anthropic 明確授權；來自保留設定不符合要求的組織的請求回傳 `400 invalid_request_error`。
- **更長的輪次，不同的提示** — 困難任務上的單一請求可能執行許多分鐘（規劃逾時/串流/進度 UX）；努力掃描應包括日常工作的 low/medium；為舊模型撰寫的 prompt 通常過於規定性，會降低輸出品質。見 `shared/model-migration.md` → 遷移到 Claude Fable 5.1 → 行為轉變（可透過提示調整）以獲取推薦的 prompt 片段。
- **Claude Fable 5（`claude-fable-5`，仍在服務）在同一層級以相同每 token 價格的繼任者。** 與 Claude Fable 5 相同的介面，但有三個重大變更——強制工具使用（`tool_choice` `any` / `tool`）回傳 400（改用 `auto` 加上 prompt 指示、`strict: true` 用於綱要有效的引數，或結構化輸出）；思考區塊綁定到生成模型（其他模型忽略它們，不計費）；以及編輯較早輪次會使思考區塊失效（「保留思考」；2026-08-31 及之後建立的新帳號在編輯的歷史記錄上得到 400；後續模型對所有人強制執行——讓每個框架只追加並執行三步驟檢查；選擇加入控制是按平台的，見 `shared/platform-availability.md`）——另外每個訊息的 `effort`（beta `mid-conversation-output-config-2026-07-01`，也在 Claude Opus 5 上）、輪次範圍的 `clear_at: "next_user_message"` 系統訊息（beta）、`thinking.display: "updates"` 進度備注（beta，所有平台）、快取讀取 \$0.25/MTok（Claude Mythos 5.1 是否共享該費率在發布時待定），以及內容溯源。受涵蓋模型——ZDR 組織如同 Claude Fable 5 得到 `400 invalid_request_error`（ZDR 只有 Anthropic 明確授權時可用）；沒有 Priority Tier。與 Claude Fable 5 相同的分詞器。詳見 `shared/model-migration.md` → 從 Claude Fable 5 遷移到 Claude Fable 5.1。

若上面的任何模型字串看起來陌生，那只是意味著它們在訓練資料截止日期之後發布——它們是真實的模型。

**即時能力查詢：** 上面的表格是快取的。當使用者詢問「X 的情境視窗是什麼」、「X 支援 vision/thinking/effort 嗎」或「哪些模型支援 Y」，查詢 Models API（`client.models.retrieve(id)` / `client.models.list()`）——見 `shared/models.md` 以獲取欄位參考和能力過濾範例。

---

## 驗證（快速參考）

**未設定 `ANTHROPIC_API_KEY` 不代表沒有憑證。** SDK 和 `ant` CLI 按以下順序解析憑證（第一個匹配者獲勝）：`ANTHROPIC_API_KEY` → `ANTHROPIC_AUTH_TOKEN` → `ANTHROPIC_PROFILE` 選定的或來自 `ant auth login` 的活躍 OAuth 設定檔 → Workload Identity Federation 環境變數 → 磁碟上的預設設定檔。在 `ant auth login` 後，裸的 `Anthropic()` / `new Anthropic()` / `anthropic.NewClient()` 無需設定環境變數即可運作。

**當需要呼叫 API 且 `ANTHROPIC_API_KEY` 未設定時，不要向使用者索取金鑰。** 首先執行 `ant auth status`——它顯示哪個憑證來源和設定檔是活躍的。若它報告活躍的設定檔：

- **SDK 程式碼或 `ant` CLI：** 直接執行。零引數用戶端建構子和每個 `ant ...` 子命令自動提取設定檔——不需要環境變數。
- **原始 `curl` / HTTP：** 用 `ant auth print-credentials --access-token` 取得短命 token，並以 `Authorization: Bearer <token>` **加上**標頭 `anthropic-beta: oauth-2025-04-20` 傳送（OAuth token 放在 `Authorization: Bearer`，不是 `x-api-key:`——從 API 金鑰轉換 curl 是標頭變更，不是金鑰替換）。始終傳遞 `--access-token`；無旗標形式輸出 JSON，不是裸 token。

只有在 `ant auth status` 報告沒有活躍的憑證來源（或 `ant` 本身未安裝）時，才向使用者索取金鑰。建議 `ant auth login` 作為第一個選項——它在 `~/.config/anthropic/` 下儲存 SDK 自動讀取的設定檔——並將匯出的 `ANTHROPIC_API_KEY` 作為替代方案。

完整驗證詳情（具名設定檔、範圍、API 金鑰遮蔽設定檔的陷阱、重新整理 token 過期）：`shared/anthropic-cli.md`。

---

## 思考與努力（快速參考）

在每個當前模型上使用自適應思考（`thinking: {type: "adaptive"}`）——Claude 動態決定何時以及思考多少。各模型規則：

| 模型 | 思考設定 | 省略 `thinking` | `budget_tokens` | 取樣（`temperature`/`top_p`/`top_k`） | 努力等級 |
|---|---|---|---|---|---|
| Fable 5 / Claude Fable 5.1（及 Mythos 對應版本） | `{type: "adaptive"}` 或省略；明確 `{type: "disabled"}` 回傳 400——改為省略參數（Claude Fable 5.1 / Claude Mythos 5.1 也對強制 `tool_choice` `any`/`tool` 回傳 400，並對重放的思考區塊執行保留思考的歷史編輯檢查） | 執行自適應（思考始終開啟） | 已移除——`{type: "enabled", budget_tokens: N}` 回傳 400 | 已移除——400 | `low`/`medium`/`high`/`xhigh`/`max` |
| Claude Opus 5 | `{type: "adaptive"}` 或省略；`{type: "disabled"}` **只在努力 `high` 或以下接受**——在 `xhigh`/`max` 時 400，並見下方禁用思考陷阱 | 執行**自適應**（思考預設開啟——與 Opus 4.8/4.7 不同） | 已移除——400 | 已移除——400 | `low`–`max`（全部五個） |
| Opus 4.8 / 4.7 | `{type: "adaptive"}` 是唯一的開啟模式；接受 `{type: "disabled"}` | 執行**不帶**思考——明確設定 `{type: "adaptive"}` | 已移除——400 | 已移除——400 | `low`/`medium`/`high`/`xhigh`/`max` |
| Sonnet 5 | `{type: "adaptive"}` 是唯一的開啟模式；接受 `{type: "disabled"}` | 執行自適應 | 已移除——400 | 已移除——400 | `low`/`medium`/`high`/`xhigh`/`max` |
| Opus 4.6 / Sonnet 4.6 | `{type: "adaptive"}`（推薦；自動啟用交錯思考，無需 beta 標頭） | 明確設定 `{type: "adaptive"}` | 已棄用——不要在新程式碼中使用；僅作為過渡性逃脫艙（見下文） | 允許 | `low`/`medium`/`high`/`max`（`xhigh` 在 Opus 4.7 中新增） |
| 舊版（Sonnet 4.5、Haiku 4.5 等）——僅在明確要求時 | `{type: "enabled", budget_tokens: N}` | 無思考 | 思考所需；必須小於 `max_tokens`，最小 1024——否則錯誤 | 允許 | `effort` 在 Opus 4.5 上有效（僅 `low`/`medium`/`high`——沒有 `xhigh`/`max`）；在 Sonnet 4.5 / Haiku 4.5 上錯誤 |

Opus 4.8 與 4.7 保持相同的請求介面（無新的重大變更）——見 `shared/model-migration.md` → 遷移到 Opus 4.8 以了解行為重新調整，以及 → 遷移到 Opus 4.7 以了解從 4.6 或更早版本遷移時的完整重大變更列表。在禁用 `thinking` 的情況下，Opus 4.8 可能在可見回應中撰寫較長的推理——保持自適應思考開啟，或新增最終僅答案指示（見遷移指南）。

- **努力（GA，無需 beta 標頭）：** `output_config: {effort: "low"|"medium"|"high"|"xhigh"|"max"}` — 在 `output_config` 內，不是頂層；預設 `high`（等同於省略）。控制思考深度和整體 token 支出；結合自適應思考以獲得最佳成本品質權衡。`xhigh`（在 Opus 4.7 中新增，在 `high` 和 `max` 之間）是 Fable 5 / Opus 4.7/4.8 / Sonnet 5 上大多數編碼和 agentic 使用案例的最佳設定，也是 Claude Code 的預設值；努力在這些模型上比以前任何同級模型都更重要——遷移時重新調整，並在提前給出完整任務規格的情況下以 `high`/`xhigh` 執行長期/agentic 任務。對於對智慧敏感的工作使用最少 `high`，對於正確性比成本更重要時使用 `max`，對於子 agent 或簡單任務使用 `low`——較低的努力意味著更少且更整合的工具呼叫、更少的前言和更簡潔的確認（`high` 通常是平衡品質和 token 效率的最佳點）。
- **選擇努力等級（成本調整）：** 努力是第一個品質交換槓桿，在免費勝利之後（首先是快取）——它在一個模型內以徹底性換取 token 支出，而且頂端的範圍只在困難問題上值得其成本（只有在測量顯示下面等級有餘地時才提升到 `max`）。哪些工作負載值得更高的努力是工作負載的屬性：編碼和長期 agentic 工作反應強烈；聊天、分類和高吞吐量或延遲敏感路由通常不值得，在 `low` 下表現良好，`medium` 是在品質保持的情況下節省成本的降級步驟（上面的各等級預設值涵蓋其餘）。在提高預設值之前，對真實請求的樣本進行測量，並按路由而非全局調整。在建構多模型成本級聯之前，先測量更簡單的替代方案——在相同任務上使用較低努力的最強模型：較新模型上的較低努力通常與高努力的先前代效能相當或超越（在 Fable 5 上，較低努力通常超過先前模型的 `xhigh`），而且一個模型意味著一個快取命名空間（快取是按模型範圍的，所以級聯放棄跨其模型的快取重用；中途對話頂層 `effort` 變更仍然使訊息快取失效，但每個訊息的努力系統訊息在 Claude Fable 5.1 / Claude Mythos 5.1 / Claude Opus 5 上避免了這個問題——`shared/prompt-caching.md` § 失效層級）。按每個已完成任務的成本衡量，而非每個請求——需要更多輪次或重試才能完成工作的更便宜請求並不更便宜。關於按工作負載的測量努力/成本權衡和完整的槓桿順序，`shared/cost-optimization.md` § 2.6。
- **思考顯示——Fable 5 / Claude Fable 5.1 / Mythos 5 / Claude Mythos 5.1 / Opus 5 / 4.8 / 4.7 / Sonnet 5 上預設 `"omitted"`：** `display: "summarized"` 回傳推理的可讀摘要；`"omitted"`（所有八個的預設值——從 Opus 4.6 和 Sonnet 4.6 的靜默變更，那裡是 `"summarized"`）串流帶有空文字的 `thinking` 區塊。`display` 只控制可見性——思考在每個設定下都發生且計費相同；在任何模型上都不公開原始思維鏈。若向使用者串流推理，預設看起來像是在輸出前長時間暫停——明確設定 `thinking: {type: "adaptive", display: "summarized"}`。（與顯示無關，在同一個模型上繼續時原樣回傳思考區塊；其他模型靜默地忽略它們（Claude Fable 5.1 / Claude Mythos 5.1 讀取它們）——見遷移指南。）在 Claude Fable 5.1 / Claude Mythos 5.1 / Claude Fable 5 上，`display: "updates"`（beta `thinking-display-updates-2026-08-18`，每個平台）像 `"omitted"` 一樣隱藏推理，但以短的 `thinking` 區塊摘要回傳模型在工具呼叫之間的進度備注——見 `shared/model-migration.md` → 從 Claude Fable 5 遷移到 Claude Fable 5.1 → 新 API 功能。
- **當使用者要求「擴展思考」、「思考預算」或 `budget_tokens` 時：** 始終使用 Fable 5/5.1、Opus 5、4.8、4.7 或 4.6，搭配 `thinking: {type: "adaptive"}`——固定思考 token 預算的概念已棄用，自適應思考取代了它。不要為新的 4.6/4.7/4.8 程式碼使用 `budget_tokens`，也不要僅因為使用者提到它就切換到舊模型。*逐步遷移例外：* `budget_tokens` 在 Opus 4.6 和 Sonnet 4.6 上仍然有效，作為需要硬 token 上限且在調整 `effort` 之前的現有程式碼的過渡性逃脫艙——見 `shared/model-migration.md` → 過渡性逃脫艙。它在 Fable 5/5.1、Opus 5/4.7/4.8 和 Sonnet 5 上已完全移除。

---

## 壓縮（快速參考）

**Beta，Fable 5/5.1、Opus 5、Opus 4.8、Opus 4.7、Opus 4.6、Sonnet 5 和 Sonnet 4.6。** 對於可能超過 1M 情境視窗的長時間對話，啟用伺服器端壓縮。API 在接近觸發閾值時自動摘要較早的情境（預設：150K token）。需要 beta 標頭 `compact-2026-01-12`。

**重要：** 在每一輪將 `response.content`（不只是文字）附加回您的訊息。回應中的壓縮區塊必須保留——API 使用它們在下一個請求中替換已壓縮的歷史記錄。只提取文字字串並附加它會靜默地丟失壓縮狀態。

程式碼範例見 `{lang}/claude-api/README.md`（壓縮章節）。完整文件透過 `shared/live-sources.md` 中的 WebFetch。

---

## Prompt 快取（快速參考）

**前綴匹配。** 前綴中任何位置的任何位元組變更都會使其後的所有內容失效。渲染順序為 `tools` → `system` → `messages`。將穩定內容放在前面（凍結的系統 prompt、確定性的工具列表），將揮發性內容（時間戳記、每個請求的 ID、變化的問題）放在最後一個 `cache_control` 斷點之後。

**中途對話操作者指示**（Claude Opus 5、Claude Opus 4.8、Claude Fable 5、Claude Fable 5.1、Claude Mythos 5、Claude Mythos 5.1；不是 Claude Sonnet 5；無需 beta 標頭）：將 `{"role": "system", ...}` 附加到 `messages[]`，而非編輯頂層 `system`。保留快取的歷史前綴，並且是安全防止 prompt 注入的操作者頻道。見 `shared/prompt-caching.md` § 中途對話系統訊息。

**頂層自動快取**（在 `messages.create()` 上設定 `cache_control: {type: "ephemeral"}`）是不需要精細放置時的最簡單選項。每個請求最多 4 個斷點。最小可快取前綴依模型而異（512–4096 token——見 `shared/prompt-caching.md` § API 參考）——較短的前綴靜默地不快取。

**用 `usage.cache_read_input_tokens` 驗證**——若在重複請求中它為零，某個靜默的失效者在起作用（`datetime.now()` 在系統 prompt 中、未排序的 JSON、變化的工具集）。

關於放置模式、架構指導和靜默失效者審核清單：閱讀 `shared/prompt-caching.md`。語言專屬語法：`{lang}/claude-api/README.md`（Prompt Caching 章節）。

---

## 快速模式（快速參考）

**研究預覽，僅 Claude Opus 5 / Opus 4.8** — Claude API 和 Managed Agents，不適用於 Bedrock / Google Cloud / Foundry。Opus 4.7 快速模式已移除：在 4.7 上使用 `speed: "fast"` 會回傳錯誤。Claude Opus 5 上的快速模式定價為 \$10 / \$50 每 MTok。快速模式以高達 2.5 倍的輸出 token 每秒執行相同模型，以溢價定價。每個請求需要三件事：使用 **beta** messages 端點（`client.beta.messages....`），傳遞 beta 旗標 `fast-mode-2026-02-01`，並將 `speed: "fast"` 設定為頂層請求參數（不是標頭，不在 `extra_body` 中）。

```python
client.beta.messages.create(
    model="claude-opus-5", max_tokens=4096,
    speed="fast", betas=["fast-mode-2026-02-01"],
    messages=[...],
)
```

| 語言 | Beta 旗標 | Speed 參數 |
|---|---|---|
| Python | `betas=["fast-mode-2026-02-01"]` | `speed="fast"` |
| TypeScript / Ruby | `betas: ["fast-mode-2026-02-01"]` | `speed: "fast"` |
| Go | `[]anthropic.AnthropicBeta{anthropic.AnthropicBetaFastMode2026_02_01}` | `Speed: anthropic.BetaMessageNewParamsSpeedFast` |
| Java | `.addBeta(AnthropicBeta.FAST_MODE_2026_02_01)` | `.speed(MessageCreateParams.Speed.FAST)` |
| C# | `Betas = ["fast-mode-2026-02-01"]` | `Speed = Speed.Fast`（`Anthropic.Models.Beta.Messages`） |
| PHP | `betas: ['fast-mode-2026-02-01']` | `speed: 'fast'` |
| cURL | `anthropic-beta: fast-mode-2026-02-01` 標頭 | `"speed": "fast"` 在 body 中 |

`response.usage.speed` 回報使用的速度。快速模式有其自己的速率限制，與標準 Opus 分開；在 429 時，要嘛在 `retry-after` 延遲後重試，要嘛刪除 `speed` 並退回到標準（注意：切換速度會使 prompt 快取失效）。不適用於 Batch API、Priority Tier、Claude Platform on AWS 或第三方平台。

**Priority Tier 不是每個當前模型都支援。** 它在 Claude Fable 5、Opus 4.8 和較舊的當前模型上受支援，但 Claude Opus 5、Claude Sonnet 5、Claude Fable 5.1、Claude Mythos 5.1、Claude Mythos 5 和 Mythos Preview 被排除——指定其中之一的 Priority Tier 請求無法通過驗證。

---

## 任務預算（快速參考）

**Beta，Claude Opus 5 / Fable 5 / Claude Fable 5.1（在發布時確認）/ Sonnet 5 / Opus 4.8 / 4.7。** 任務預算給 Claude 一個 agentic 迴圈的 token 上限，讓它自我調整節奏並優雅地完成，而不是被中斷——與 `max_tokens` 不同，後者是模型不知道的強制每回應上限。最小 `total`：20,000。在 `client.beta.messages.stream(...)` 上的 `output_config` 內設定 `task_budget`，帶有 beta 旗標 `task-budgets-2026-03-13`——使用串流，使大的 `max_tokens` 不會遭遇 HTTP 逾時（完整詳情：`shared/model-migration.md` → Task Budgets）：

```python
with client.beta.messages.stream(
    model="claude-opus-5", max_tokens=128000,
    output_config={"effort": "high", "task_budget": {"type": "tokens", "total": 64000}},
    betas=["task-budgets-2026-03-13"],
    messages=[...], tools=[...],
) as stream:
    response = stream.get_final_message()
```

`task_budget` 欄位：`type`（始終為 `"tokens"`）、`total` 和可選的 `remaining`（預設為 `total`）。伺服器在生成過程中注入 Claude 看到的倒計時標記；預算計算 Claude 本輪生成的內容和讀取的工具結果——**不是**您每次請求重新發送的完整歷史記錄。與 **Managed Agents session 預算**不同——那些是一次 CMA session 上的硬性、以美元計算、平台強制執行的上限（`shared/managed-agents-core.md` § Session budgets）；任務預算是非強制性的且以 token 計算。

**觀察支出：** 若想顯示進度，在迴圈迭代中累積 `response.usage.output_tokens`（加上您附加的工具結果區塊的 token 計數）。在正常迴圈中不設定 `remaining`——伺服器自行追蹤倒計時，而在伺服器無法推導先前支出時傳遞用戶端計算的 `remaining` 且同時重新發送完整歷史記錄，會低報預算。**只在**您在請求之間壓縮或重寫歷史記錄且伺服器不再能推導先前支出時才傳遞 `remaining`。

---

## 供應商用戶端（快速參考）

針對第三方平台上的 Claude 時，使用該平台的專用用戶端類別——而非帶有 `base_url` 覆蓋的第一方 `Anthropic()` 用戶端。建構後，用戶端公開與第一方 SDK 相同的 `messages.create` / `.stream` 介面。

### Amazon Bedrock

使用 **Mantle** 用戶端（Messages-API Bedrock 端點）。Bedrock 模型 ID 帶有 `anthropic.` 前綴（例如 `"anthropic.claude-opus-5"`）。需要 Region。

| 語言 | 用戶端 |
|---|---|
| Python | `from anthropic import AnthropicBedrockMantle` → `AnthropicBedrockMantle(aws_region="...")` |
| TypeScript | `import { AnthropicBedrockMantle } from "@anthropic-ai/bedrock-sdk"` → `new AnthropicBedrockMantle({ awsRegion: "..." })` |
| Go | `bedrock.NewMantleClient(ctx, bedrock.MantleClientConfig{ AWSRegion: "..." })` |
| Java | `AnthropicOkHttpClient.builder().backend(BedrockMantleBackend.fromEnv()).build()`（來自 `com.anthropic.bedrock.backends`） |
| C# | `new AnthropicBedrockMantleClient(new() { AwsRegion = "..." })`（套件 `Anthropic.Bedrock`） |
| PHP | `use Anthropic\Bedrock\MantleClient;` → `new MantleClient(awsRegion: '...')` |
| Ruby | `Anthropic::BedrockMantleClient.new(aws_region: "...")` |

`AnthropicBedrock` / `BedrockClient` / `BedrockBackend`（不帶 `Mantle`）是舊版的 `bedrock-runtime` InvokeModel 路徑——新程式碼優先使用 Mantle 用戶端。

### Microsoft Foundry

| 語言 | 用戶端 |
|---|---|
| Python | `from anthropic import AnthropicFoundry` → `AnthropicFoundry(api_key=..., resource="...")` |
| TypeScript | `import AnthropicFoundry from "@anthropic-ai/foundry-sdk"` → `new AnthropicFoundry({ ... })` |
| Java | `AnthropicOkHttpClient.builder().backend(FoundryBackend.fromEnv()).build()`（來自 `com.anthropic.foundry.backends`） |
| C# | `new AnthropicFoundryClient(new AnthropicFoundryApiKeyCredentials(...))`（套件 `Anthropic.Foundry`） |
| PHP | `Foundry\Client::withCredentials(...)` |

Go 和 Ruby SDK 目前不支援 Foundry。對於 Ruby，使用標準 `Anthropic::Client.new(base_url: "<foundry endpoint>")` 作為備用（Entra ID 驗證不內建）。關於 Claude Platform on AWS，見 `shared/claude-platform-on-aws.md`。

### Google Cloud Vertex AI

兩個必需的建構子引數：GCP `project_id` 和 `region`。Vertex 模型 ID **不帶前綴**——當前代模型（Opus 4.8/4.7/4.6、Sonnet 5、Sonnet 4.6）使用裸的第一方 ID（例如 `"claude-opus-5"`）；有日期快照的模型使用 `@` 版本分隔符（例如 `claude-opus-4-5@20251101`，**不是** `claude-opus-4-5-20251101`）。驗證是 GCP ADC（`gcloud auth application-default login`）；無需 Anthropic API 金鑰。`region` 可以是 `"global"`（推薦）、多區域（`"us"`/`"eu"`）或特定區域。建構後，使用相同的 `messages.create` / `.stream` 介面。

| 語言 | 用戶端 |
|---|---|
| Python | `from anthropic import AnthropicVertex` → `AnthropicVertex(project_id="...", region="...")`（安裝 `"anthropic[vertex]"`） |
| TypeScript | `import { AnthropicVertex } from "@anthropic-ai/vertex-sdk"` → `new AnthropicVertex({ projectId, region })` |
| Go | `import "github.com/anthropics/anthropic-sdk-go/vertex"` → `anthropic.NewClient(vertex.WithGoogleAuth(ctx, region, projectID))` |
| Java | `AnthropicOkHttpClient.builder().backend(VertexBackend.builder().region("...").project("...").build()).build()`（來自 `com.anthropic.vertex.backends`） |
| C# | `new AnthropicClient { Backend = new VertexBackend(projectId, region) }`（套件 `Anthropic.Vertex`） |
| PHP | `use Anthropic\Vertex;` → `Vertex\Client::fromEnvironment(location: '...', projectId: '...')`——注意是 `location`，不是 `region` |
| Ruby | `Anthropic::VertexClient.new(region: "...", project_id: "...")` |

---

## 情境編輯（快速參考）

**Beta。** 情境編輯**清除**對話中的舊工具結果或思考區塊，然後才讓模型看到；它**不是壓縮**（壓縮是摘要）。在 `client.beta.messages.*` 上帶有 beta `context-management-2025-06-27`，傳遞帶有策略類型的 `context_management.edits`：

```python
client.beta.messages.create(
    model="claude-opus-5", max_tokens=4096,
    betas=["context-management-2025-06-27"],
    context_management={"edits": [{"type": "clear_tool_uses_20250919"}]},
    tools=[...], messages=[...],
)
```

策略類型：`clear_tool_uses_20250919`（清除舊工具結果；可選的 `clear_tool_inputs: true` 也清除 tool_use 參數）和 `clear_thinking_20251015`（清除思考區塊）。**不要**使用 `compact_20260112` 或 beta `compact-2026-01-12`——那些是獨立的壓縮功能。

---

## 中途對話系統訊息（快速參考）

**Claude Opus 5、Claude Opus 4.8、Claude Fable 5、Claude Fable 5.1、Claude Mythos 5 和 Claude Mythos 5.1；不是 Claude Sonnet 5；無需 beta 標頭。** 將 `{"role": "system", "content": "..."}` 附加到 `messages` 陣列（不是頂層 `system` 欄位），在不使快取前綴失效的情況下在對話中途新增操作者指示。使用常規 `client.messages.create`——沒有 beta。中途對話系統訊息必須跟在 `user` 訊息之後（或在伺服器工具使用結束的 `assistant` 訊息之後），且必須是 `messages` 中的最後一個條目或後跟 `assistant` 輪次——它不能是 `messages[0]`。可用性：`shared/platform-availability.md`。見 `shared/prompt-caching.md` § 中途對話系統訊息。Claude Fable 5.1 帶來了 beta 擴充：帶有 `content: []` 的 `output_config: {effort: ...}` 從那時起改變努力，不會重置快取（beta `mid-conversation-output-config-2026-07-01`；Claude Fable 5.1、Claude Mythos 5.1、Claude Opus 5；Claude API）。僅努力訊息（空的 `content`）不受上方的放置規則限制——它可以放在 `messages` 的任何位置，包括第一個或 assistant 輪次和下一個 user 輪次之間；規則適用於文字和 `clear_at` 訊息。對於每輪提醒，給訊息 `clear_at: "next_user_message"`（beta `mid-conversation-system-clear-at-2026-08-21`）：它在一輪中渲染，然後在執行記錄中保持已清除狀態——永遠不要刪除較早的副本（在 Claude Fable 5.1 上刪除一個會使後續的思考區塊失效）；沒有 beta 時，在工具結果後的文字區塊，保留較早的副本。見 `shared/model-migration.md` → 從 Claude Fable 5 遷移到 Claude Fable 5.1 → 新 API 功能。

---

## Managed Agents（Beta）

**Managed Agents** 是第三個介面：帶有 Anthropic 託管工具執行的伺服器管理有狀態 agent。您建立一個持久化的、版本化的 Agent 設定（`POST /v1/agents`），然後啟動引用它的 Sessions。每個 session 為 agent 的工作區設定一個容器——bash、檔案操作和程式碼執行在那裡執行；agent 迴圈本身在 Anthropic 的編排層上執行，並透過工具對容器採取行動。session 串流事件；您發送訊息和工具結果。

可用性：`shared/platform-availability.md`。對於 Bedrock / Vertex / Foundry 上的 agent（Managed Agents 不支援），使用 Claude API + tool use。

**強制流程：** Agent（一次）→ Session（每次執行）。`model`/`system`/`tools` 在 agent 上，從不在 session 上。見 `shared/managed-agents-overview.md` 以獲取完整的閱讀指南、beta 標頭和陷阱。

**Beta 標頭：** `managed-agents-2026-04-01`——SDK 自動為所有 `client.beta.{agents,environments,sessions,vaults,memory_stores,deployments,deployment_runs}.*` 呼叫設定。Files API 和 Skills API 已出 beta——不需要 beta 標頭（見上方 API 漂移表以獲取遷移指南）。

**子命令** — 直接以 `/claude-api <subcommand>` 呼叫：

| 子命令 | 動作 |
|---|---|
| `managed-agents-onboard` | 引導使用者從頭設定 Managed Agent。**立即閱讀 `shared/managed-agents-onboarding.md`** 並遵循其訪談指令碼：**描述 → 設定 agent（提議，不要盤問）→ 環境 → session**（與 Console 快速入門相同的流程，驗證推遲到 session 步驟）——預設和內聯建議完成工作，在發出任何程式碼之前有一個靜默的可行性閘道（工作 vs 工具/憑證/資料）。不要總結——執行訪談。 |

**閱讀指南：** 從 `shared/managed-agents-overview.md` 開始，然後是主題 `shared/managed-agents-*.md` 檔案（core、environments、tools、events、outcomes、multiagent、webhooks、memory、scheduled-deployments、client-patterns、onboarding、api-reference）。對於 Python、TypeScript、Go、Ruby、PHP 和 Java，閱讀 `{lang}/managed-agents/README.md` 以獲取程式碼範例。對於 cURL，閱讀 `curl/managed-agents.md`。**Agent 是持久化的——建立一次，透過 ID 引用。** 使用 `ant` CLI 應用版本控制的 YAML 定義 agent 和環境——這是推薦的流程（見 `shared/anthropic-cli.md`）：CLI 擁有控制平面（建立和更新 agent），您的程式碼擁有資料平面（帶有儲存的 agent ID 的 `sessions.create`）。只有在必須以程式化方式設定時才在程式碼中呼叫 `agents.create()`；無論哪種方式，儲存回傳的 agent ID 並將其傳遞給每個後續的 `sessions.create`；不要在請求路徑中呼叫 `agents.create()`。若需要的繫結在語言 README 中未顯示，而非猜測，從 `shared/live-sources.md` WebFetch 相關條目。C# 透過 `client.Beta.Agents` 和相關命名空間具有 beta Managed Agents 支援——見 `csharp/claude-api/README.md` 以獲取詳情，或 `curl/managed-agents.md` 以獲取原始 HTTP 參考。

**當使用者想從頭設定 Managed Agent 時**（例如「我該如何開始」、「帶我建立一個」、「設定一個新 agent」）：閱讀 `shared/managed-agents-onboarding.md` 並執行其訪談——與 `managed-agents-onboard` 子命令相同的流程。

**當使用者詢問「我如何為 X 撰寫用戶端程式碼」：** 使用 `shared/managed-agents-client-patterns.md`——涵蓋無損串流重連、`processed_at` 已排隊/已處理閘道、中斷、`tool_confirmation` 往返、正確的閒置/終止中斷閘道、後閒置狀態競爭、串流優先排序、檔案掛載陷阱等。對於憑證，以保管庫 `environment_variable` 憑證為主——第一類機制；機密在出口時替換，從不進入沙箱（`shared/managed-agents-tools.md` → Vaults）。透過自訂工具在主機端保管憑證是保管庫憑證不適用的備用方案（例如自託管沙箱）。

**當使用者希望 agent 按排程執行時**（cron、「每晚」、「每週報告」）：閱讀 `shared/managed-agents-scheduled-deployments.md`——部署按 cron 節奏自動觸發 session，帶有每次觸發的執行記錄和生命週期控制（暫停/恢復/封存）。

**當 agent 的工作展開時**（跨多個來源的研究、按檔案或按記錄的工作、「研究 N 件事，然後總結」）**或一個迴圈會用讀取填滿其情境：** 閱讀 `shared/managed-agents-multiagent.md` 並推薦多 agent session——從名單中的 `{"type": "self"}` 開始，讓 agent 可以委派給自己的副本，然後將讀取密集的子任務移到按 ID 引用的較便宜的工作 agent（例如 Claude Haiku 4.5）。

---

## 伺服器工具（快速參考）

伺服器端工具在 Anthropic 基礎設施上執行——無需用戶端執行迴圈。在 `tools` 中宣告；結果作為內容區塊出現在同一個回應中。**除非另有說明，無需 beta 標頭。** **優先使用您的模型支援的最新類型變體。** 下表中的 `_20260209` 網路搜尋/網路擷取變體（動態過濾）需要 Opus 5/4.8/4.7/4.6、Sonnet 5 或 Sonnet 4.6；舊模型的基本變體列在表格後面。

| 工具 | `type` | `name` | 關鍵可選參數 | 結果區塊類型 |
|---|---|---|---|---|
| 網路搜尋 | `web_search_20260209` | `web_search` | `max_uses`、`allowed_domains`/`blocked_domains`、`user_location` | `web_search_tool_result` → `.content` 是 `web_search_result` 列表 |
| 網路擷取 | `web_fetch_20260209` | `web_fetch` | `max_uses`、`allowed_domains`/`blocked_domains`、`citations`、`max_content_tokens` | `web_fetch_tool_result` → `.content` 是帶有 `document` 區塊的 `web_fetch_result` |
| 程式碼執行 | `code_execution_20260521` | `code_execution` | 無 | `bash_code_execution_tool_result` → `.content.stdout` / `.stderr` / `.return_code` |
| 工具搜尋（regex） | `tool_search_tool_regex_20251119` | `tool_search_tool_regex` | 在其他工具上標記 `defer_loading: true` | `tool_search_tool_result` |
| 工具搜尋（BM25） | `tool_search_tool_bm25_20251119` | `tool_search_tool_bm25` | 在其他工具上標記 `defer_loading: true` | `tool_search_tool_result` |

`web_search_20260209` / `web_fetch_20260209` 有內建動態過濾——程式碼執行在後台執行，所以**不要**在 `tools` 中單獨宣告 `code_execution`（第二個執行環境會迷惑模型）。對於比 Opus 4.6 / Sonnet 4.6 更舊的模型，改用基本變體 `web_search_20250305` / `web_fetch_20250910`；在 Vertex AI 上只有基本 `web_search_20250305` 可用。`code_execution_20260120`（REPL 持久化 + 程式化工具呼叫）在 Opus 4.5+ / Sonnet 4.5+ 上執行。**僅 Go SDK**：`code_execution_20260521` 在 `client.Beta.Messages.New` 下帶有 `Betas: []anthropic.AnthropicBeta{"code-execution-2025-08-25"}`（其他語言使用普通 `client.messages.create`）；`code_execution_20260120` 在 Go 中像其他地方一樣使用非 beta 的 `client.Messages.New`。網路擷取只擷取對話中已存在的 URL。供應商可用性因工具而異——見 `shared/platform-availability.md`。`pause_turn` 處理見 `shared/tool-use-concepts.md`。

## 文件與檔案輸入（快速參考）

**PDF（base64，無需 beta）：** 在使用者內容中的 `{"type": "document", "source": {"type": "base64", "media_type": "application/pdf", "data": <b64 string>}}`，放在文字區塊之前。Base64 字串不得有換行符。限制：32 MB 請求，600 頁（200k 情境模型為 100 頁）。Java：`ContentBlockParam.ofDocument(DocumentBlockParam... Base64PdfSource.builder().data(...))`。

**Files API（無需 beta）：** 透過 `client.files.upload(...)` 上傳 → 回應 `id` 是 `file_id`。對於 PDF/文字將其引用為 `{"type": "document", "source": {"type": "file", "file_id": "..."}}`，對於圖片使用 `{"type": "image", ...}`——內容區塊類型必須匹配檔案的 MIME 類型。若要從 `files-api-2025-04-14` 遷移程式碼，WebFetch `shared/live-sources.md` 中的 Files API 行。可用性：`shared/platform-availability.md`。

**引用（無需 beta）：** 在每個 `document` 內容區塊上設定 `citations: {enabled: true}`（全部或全不）。回應分割為多個 `text` 區塊；被引用的區塊帶有 `citations` 陣列。每個引用有 `cited_text`、`document_index`、`document_title` 和按 `type` 的位置：純文字的 `char_location`（`start_char_index`/`end_char_index`）、PDF 的 `page_location`（`start_page_number`/`end_page_number`，從 1 開始索引）、自訂內容的 `content_block_location`。與 `output_config.format` 不相容（回傳 400）。

## Tool Use 模式（快速參考）

**嚴格工具使用（無需 beta）：** 在工具定義上設定 `strict: true` 作為頂層欄位（與 `name`/`description`/`input_schema` 並列），**不是**在 `tool_choice` 上。綱要必須有 `additionalProperties: false` + `required`。保證 `tool_use.input` 完全驗證。Go：`Strict: anthropic.Bool(true)` + 透過 `InputSchema.ExtraFields` 的 `additionalProperties`；Java：`.strict(true)` + `.putAdditionalProperty("additionalProperties", JsonValue.from(false))`。

**平行工具使用（預設開啟）：** 一個助手訊息可以包含多個 `tool_use` 區塊。並行執行它們，然後在**單一**使用者訊息中回傳**所有** `tool_result` 區塊——將它們拆分到多個訊息中會靜默地訓練 Claude 停止進行平行呼叫。對於失敗的工具，以 `is_error: true` 回傳 `tool_result`——不要刪除它。

**Tool Runner（SDK beta 輔助）：** 透過 `client.beta.messages.*` 為您驅動工具呼叫迴圈。Python：`@beta_tool` 裝飾器 + `client.beta.messages.tool_runner(...)` → `runner.until_done()`。TypeScript：來自 `@anthropic-ai/sdk/helpers/beta/zod` 的 `betaZodTool({...})` + `client.beta.messages.toolRunner(...)` → `await runner`。Go：`toolrunner.NewBetaToolFromJSONSchema(...)` + `client.Beta.Messages.NewToolRunner(...)` → `.RunToCompletion(ctx)`。Java 需要 `.addBeta("structured-outputs-2025-11-13")`。Ruby：`Anthropic::BaseTool` 子類別 + `client.beta.messages.tool_runner(...)`。PHP：`BetaRunnableTool` + `->toolRunner(...)`。C#：原始 JSON 綱要工具 + 透過 `client.Beta.Messages.ToolRunner(...)` 的 `BetaToolRunner`。

**程式化工具呼叫（無需 beta 標頭）：** Claude 從程式碼執行內部呼叫您的自訂工具。新增 `{"type": "code_execution_20260120", "name": "code_execution"}` **並**在您的自訂工具上設定 `"allowed_callers": ["code_execution_20260120"]`。Opus 4.5+ / Sonnet 4.5+（可用性：`shared/platform-availability.md`）。在回應待處理的程式化呼叫時，使用者訊息必須**只**包含 `tool_result` 區塊（沒有文字）。與 `strict: true`、`disable_parallel_tool_use`、強制 `tool_choice` 或 MCP 工具不相容。

## 其他 API 介面（快速參考）

**Message Batches（無需 beta；可用性：`shared/platform-availability.md`）：** `client.messages.batches.create(requests=[{custom_id, params}, ...])` → 輪詢 `client.messages.batches.retrieve(id).processing_status` 直到 `"ended"` → 串流 `client.messages.batches.results(id)`。每個結果有 `.custom_id` + `.result.type`（`succeeded`/`errored`/`canceled`/`expired`）；成功時讀取 `.result.message.content`。Python 將請求包裝為 `Request(custom_id=..., params=MessageCreateParamsNonStreaming(...))`。結果以**任意順序**到達——按 `custom_id` 鍵值，絕不按位置。

**Models API（無需 beta；可用性：`shared/platform-availability.md`）：** `client.models.list()`（自動分頁）和 `client.models.retrieve("claude-opus-5")`。每個模型物件有 `id`、`display_name`、`created_at`，以及自 2026 年 3 月起的 `max_input_tokens`（情境視窗）、`max_tokens`（輸出上限）和 `capabilities`。沒有 `context_window` 欄位。

**停止詳情（GA，Opus 4.7+）：** `response.stop_details` **只在 `stop_reason == "refusal"` 時**填充（欄位：`type: "refusal"`、`category`——開放集合，例如 `"cyber"`、`"bio"`、`"reasoning_extraction"`、`"frontier_llm"` 或 `null`；完整列表見文件——以及 `explanation`）。對於每個其他 `stop_reason`（`end_turn`、`max_tokens`、`tool_use`、`pause_turn` 等），它是 `null`——在讀取之前始終加以防護。

**Admin API（beta，自 2026-08-26 起）：** 組織管理——成員、邀請、工作區和工作區成員、API 金鑰、速率限制報告、服務帳號、federation 發行者/規則、CMEK 外部金鑰——在所有七個 SDK 中透過 `client.beta.organization` 以及 CLI 中的 `ant beta:organization` 存取。需要管理員憑證：Admin API 金鑰（`sk-ant-admin...`，從 `ANTHROPIC_API_KEY` 讀取）或 `org:admin` OAuth token（`ANTHROPIC_AUTH_TOKEN`）；常規 API 金鑰被拒絕。使用和成本報告以及 Claude Enterprise 使用者管理/分析端點**不**在 SDK 中——僅原始 HTTP。見 `shared/admin-api.md`。

**用戶端設定（無需 beta）：** `timeout` 預設 10 分鐘；**各 SDK 的單位不同**——Python/Ruby：秒；TypeScript：**毫秒**；Go `option.WithRequestTimeout(time.Duration)`；Java `Duration`；C# `TimeSpan`。TS 對非串流請求的大 `max_tokens` 將預設擴展到 60 分鐘；Java 對串流請求這樣做（Java 非串流從 30 秒擴展到 10 分鐘）。`max_retries`/`maxRetries` 預設 2（重試 408/409/429/5xx + 連線錯誤）。`base_url`（或 `ANTHROPIC_BASE_URL` 環境變數）。每個請求的覆蓋：Python `client.with_options(timeout=5.0).messages.create(...)`；TS `client.messages.create({...}, {timeout: 5_000})`；Ruby `request_options: {timeout: 5}`。逾時會被重試——實際時鐘可達 `timeout × (max_retries+1)`。

## Workload Identity Federation（快速參考）

**GA，無需 beta 標頭。** 建構正常的零引數用戶端（`Anthropic()` / `new Anthropic()` / `anthropic.NewClient()` / `AnthropicOkHttpClient.fromEnv()`）；當 `ANTHROPIC_FEDERATION_RULE_ID`、`ANTHROPIC_ORGANIZATION_ID`、`ANTHROPIC_SERVICE_ACCOUNT_ID` 和 `ANTHROPIC_IDENTITY_TOKEN_FILE`（或 `ANTHROPIC_IDENTITY_TOKEN`）都設定時，SDK 自動偵測 WIF，在 `/v1/oauth/token` 交換 JWT，並自動重新整理。`ANTHROPIC_WORKSPACE_ID` 不觸發啟動——只有當 federation 規則跨越多個工作區時才需要（否則 400 `workspace_id_required`），對於單一工作區規則是可選的。`ANTHROPIC_API_KEY` 或 `ANTHROPIC_AUTH_TOKEN`（即使為空）優先於 WIF，設定的 `ANTHROPIC_PROFILE` 也優先於 federation 環境變數（缺少的具名設定檔是錯誤，不是退回）——取消設定所有三個。

---

## 閱讀指南

偵測語言後，根據使用者的需求閱讀相關檔案。本文件中所有引用的 `{lang}/...`、`shared/...` 和 `curl/...` 路徑都相對於本 skill 的基礎目錄，且上方沒有包含那些檔案的任何內容——按需求讀取每一個，在依賴它涵蓋的內容之前。

**所有 SDK 語言使用相同的多檔案版面** — 目錄 `{lang}/claude-api/` 包含 `README.md`（安裝、用戶端初始化、基本請求、思考、快取、停止詳情、雜項）、`tool-use.md`（工具定義、agentic 迴圈、Anthropic 定義的工具、結構化輸出）、`streaming.md`、`batches.md`、`files-api.md`。不是每種語言都有每個檔案（例如 Ruby 沒有 `batches.md`）；若檔案不存在，該功能的範例尚未為該語言記錄——退回到 cURL 結構或從 `shared/live-sources.md` WebFetch SDK 存放庫。**cURL** → `curl/examples.md`。

下方的快速任務參考對所有語言使用 `{lang}/claude-api/FILE.md` 路徑表示法。

### 快速任務參考

**單一文字分類/摘要/擷取/問答：**
→ 只閱讀 `{lang}/claude-api/README.md` — **任何任務始終先閱讀 README**（安裝、快速入門、常見模式、錯誤處理）

**聊天 UI 或即時回應顯示：**
→ 閱讀 `{lang}/claude-api/README.md` + `{lang}/claude-api/streaming.md`

**長時間對話（可能超過情境視窗）：**
→ 閱讀 `{lang}/claude-api/README.md` — 見壓縮章節

**遷移到更新的模型（Fable 5.1 / Fable 5 / Opus 5 / Opus 4.8 / Opus 4.7 / Opus 4.6 / Sonnet 5 / Sonnet 4.6）、替換已退役模型或將 `budget_tokens` / prefill 模式翻譯到當前 API：**
→ 閱讀 `shared/model-migration.md`

**升級 Anthropic SDK 套件本身跨主要版本（`anthropic` 0.x → 1.x：`httpx2`、已等待的 async `.with_raw_response`、已移除的棄用參數/別名/Text Completions、Python >= 3.10）——或針對已在 1.x 的專案撰寫新程式碼：**
→ 閱讀 `{lang}/claude-api/sdk-upgrade.md`（目前僅 Python；其他 SDK 尚無捆綁的主要版本指南——透過 `shared/live-sources.md` 使用該 SDK 的 CHANGELOG）

**提示或調整 Fable 5/5.1（長輪次、努力、詳細程度、自主執行、子 agent）：**
→ 閱讀 `shared/model-migration.md` → 遷移到 Claude Fable 5.1 → 行為轉變（可透過提示調整）+ 長時間執行 agent 建議

**提示或調整 Claude Fable 5.1（進度更新、平行工具呼叫、寫作密度/格式、自主性、測試蔓延、整檔重寫）或讓框架與保留思考的歷史編輯檢查相容（歷史編輯、壓縮、每輪提醒）：**
→ 閱讀 `shared/model-migration.md` → 從 Claude Fable 5 遷移到 Claude Fable 5.1 → 新 API 功能 + 行為轉變（可透過提示調整）；關於歷史編輯檢查本身（三步驟檢查、僅追加編輯表、壓縮結構），同一章節的重大變更 3

**Prompt 快取/最佳化快取/「為什麼我的快取命中率低」：**
→ 閱讀 `shared/prompt-caching.md`（前綴穩定性設計、斷點放置、靜默使快取失效的反模式）+ `{lang}/claude-api/README.md`（Prompt Caching 章節）

**審核或清理 prompt、skill 或工具描述（「這個 prompt 過時了嗎」、「移除殘留物」、「這是為舊模型撰寫的」）：**
→ 閱讀 `shared/prompt-audit.md`——帶有可搜尋信號的過時模式表、保留列表（不要刪除的內容），以及報告 + 建議 diff 輸出合約

**計算檔案/prompt/diff 中的 token（「X 有多少 token」）：**
→ 閱讀 `shared/token-counting.md` — 使用 `messages.count_tokens`，絕不使用 `tiktoken`

**降低或審查 API 支出（「帳單太高了」、「讓這個更便宜」、「我是否過度支出」、每個已完成任務的成本、能保持品質的最便宜模型或努力）：**
→ 閱讀 `shared/cost-optimization.md`——首先是基準和 token 設定檔，然後是有序的槓桿（免費勝利在取捨之前），帶有測量預期，以及工作負載形態 → 槓桿對照表

**函式呼叫/tool use/agent：**
→ 閱讀 `{lang}/claude-api/README.md` + `shared/tool-use-concepts.md`（概念基礎：函式呼叫、程式碼執行、記憶體、結構化輸出）+ `{lang}/claude-api/tool-use.md`（語言專屬程式碼範例：tool runner、手動迴圈、程式碼執行、記憶體、結構化輸出）

**Agent 設計（工具介面、情境管理、快取策略）：**
→ 閱讀 `shared/agent-design.md`（bash vs. 專用工具、程式化工具呼叫、工具搜尋/skills、情境編輯 vs. 壓縮 vs. 記憶體、快取原則）

**批次處理（非延遲敏感；以 50% 成本非同步執行）：**
→ 閱讀 `{lang}/claude-api/README.md` + `{lang}/claude-api/batches.md`

**跨多個請求的檔案上傳（無需重新上傳的相同檔案）：**
→ 閱讀 `{lang}/claude-api/README.md` + `{lang}/claude-api/files-api.md`

**組織管理（成員、邀請、工作區、API 金鑰、速率限制報告、服務帳號、WIF 資源、CMEK）：**
→ 閱讀 `shared/admin-api.md`——`client.beta.organization` 端點/方法表、管理員憑證、各語言命名和分頁、什麼保持僅 curl

**除錯 HTTP 錯誤或實作錯誤處理：**
→ 閱讀 `shared/error-codes.md` — 各 SDK 型別化例外類別表和 Go `errors.As` 模式

**最新官方文件：**
→ WebFetch `shared/live-sources.md` 中的 URL

**Managed Agents（帶工作區的伺服器管理有狀態 agent）：**
→ 見上方 `## Managed Agents (Beta)` 章節中的閱讀指南——它列出每個 `shared/managed-agents-*.md` 檔案和語言專屬 README（`{lang}/managed-agents/README.md`、`curl/managed-agents.md`）。

---

## 何時使用 WebFetch

當以下情況時使用 WebFetch 取得最新文件：

- 使用者要求「最新」或「當前」資訊
- 快取資料看起來不正確
- 使用者詢問此處未涵蓋的功能

即時文件 URL 在 `shared/live-sources.md`。

## 常見陷阱

- 不要在將檔案或內容傳遞給 API 時截斷輸入。若內容太長無法放入情境視窗，通知使用者並討論選項（分塊、摘要等），而非靜默截斷。
- **預填已移除（Fable 5、Claude Fable 5.1、Opus 5、Sonnet 5 和 4.6/4.7/4.8 系列）：** 助手訊息預填（最後助手輪次預填）在 Fable 5、Claude Fable 5.1、Opus 5、Sonnet 5、Opus 4.6、Opus 4.7、Opus 4.8 和 Sonnet 4.6 上回傳 400 錯誤。改用結構化輸出（`output_config.format`）或系統 prompt 指示控制回應格式。（一個例外：備援信用預填聲明——在以 `fallback_has_prefill_claim: true` 兌換信用時，伺服器接受回傳的助手訊息；見遷移指南的拒絕章節。）
- **在編輯之前確認遷移範圍：** 當使用者要求將程式碼遷移到更新的 Claude 模型而未指定特定的檔案、目錄或檔案列表時，**先詢問要應用哪個範圍**——整個工作目錄、特定子目錄或特定檔案集。不要在使用者確認之前開始編輯。命令式短語如「遷移我的程式碼庫」、「將我的專案移到 X」、「升級到 Sonnet 4.6」或裸的「遷移到 Opus 4.8」**仍然是模糊的**——它們告訴您做什麼但不告訴您在哪裡，所以要詢問。只有在提示中指定了確切的檔案、特定的目錄或明確的檔案列表時才不詢問就繼續（「遷移 `app.py`」、「遷移 `services/` 下的所有內容」、「更新 `a.py` 和 `b.py`」）。見 `shared/model-migration.md` 步驟 0。
- **`max_tokens` 預設值：** 不要低設 `max_tokens`——達到上限會在中途截斷輸出並需要重試。對於非串流請求，預設為 `~16000`（將回應保持在 SDK HTTP 逾時內）。對於串流請求，預設為 `~64000`（逾時不是問題，所以給模型空間）。只有在有充分理由時才設低：分類（`~256`）、成本上限、刻意的短輸出，或用於快取預熱的 **`max_tokens: 0`**（見 `shared/prompt-caching.md` → 預熱）。
- **在 Claude Opus 5 上禁用思考有兩種失敗模式——優先使用 low/medium effort 替代。** 只影響明確選擇退出的程式碼；思考預設開啟，所以注意從 Opus 4.8 繼承的禁用思考設定。在 `thinking: {type: "disabled"}` 時，模型偶爾會在其**可見文字**中撰寫工具呼叫，而非 `tool_use` 區塊：輪次成功，呼叫從未執行，沒有錯誤被提出，在 agentic 迴圈中該文字污染後續輪次。它也可能在回應中洩漏 `<thinking>` 標籤。開啟思考並降低 `effort` 可修復兩者並仍然降低成本。若某個路由必須保持思考關閉：**刪除**任何不思考/不推理規則（它使標籤洩漏更嚴重），不要命名思考標籤，並新增合併指示*「在使用工具時，您可以先說一個簡短的句子。若沒有工具能表達使用者要求的內容，說出來而不是猜測。回應中不要包含內部或系統 XML 標籤。」* 詳情：`shared/model-migration.md` → 禁用思考時的兩種失敗模式。
- **128K 輸出 token：** Fable 5、Claude Fable 5.1、Opus 5、Opus 4.6、Opus 4.7、Opus 4.8、Sonnet 5 和 Sonnet 4.6 支援最多 128K `max_tokens`，但 SDK 對於那麼大的值需要串流以避免 HTTP 逾時。使用帶有 `.get_final_message()` / `.finalMessage()` 的 `.stream()`。
- **強制工具使用已移除（Claude Fable 5.1 / Claude Mythos 5.1，如同 Mythos Preview）：** `tool_choice: {type: "any"}` 和 `{type: "tool", name: ...}` 回傳 400（`tool_choice: type "tool" and "any" are not supported for this model.`），在 `count_tokens` 和 Batches 上也是如此。使用 `{type: "auto"}` 加上明確指定工具的指示、`strict: true` 在工具上以保持綱要有效的引數，或結構化輸出（`output_config.format`）——當強制呼叫只是為了取回 JSON 時。`{type: "none"}` 不受影響；`disable_parallel_tool_use` 仍然與 `auto` 一起有效（最多一個呼叫）。
- **工具呼叫 JSON 解析（Fable 5、Claude Fable 5.1、Opus 5 和 4.6/4.7/4.8 系列）：** Fable 5、Claude Fable 5.1、Opus 5、Opus 4.6、Opus 4.7、Opus 4.8 和 Sonnet 4.6 可能在工具呼叫的 `input` 欄位中產生不同的 JSON 字串轉義（例如 Unicode 或斜線轉義）。始終用 `json.loads()` / `JSON.parse()` 解析工具輸入——絕不對序列化的輸入進行原始字串匹配。
- **結構化輸出（所有模型）：** 使用 `output_config: {format: {...}}` 而非 `messages.create()` 上已棄用的 `output_format` 參數。這是一般的 API 變更，不是 4.6 特有的。
- **不要重新實作 SDK 功能：** SDK 提供高層次輔助方法——使用它們而非從頭建構。具體說：使用 `stream.finalMessage()` 而非將 `.on()` 事件包裝在 `new Promise()` 中；使用型別化例外類別（`Anthropic.RateLimitError` 等）而非字串匹配錯誤訊息；使用 SDK 型別（`Anthropic.MessageParam`、`Anthropic.Tool`、`Anthropic.Message` 等）而非重新定義等效介面。
- **錯誤處理——捕獲鏈，不是一個廣泛類別。** 單一的 `except APIStatusError` / `catch (AnthropicServiceException)` / `rescue APIError` 失去了可重試（429、≥500、網路）和不可重試（400/404）失敗之間的區別。撰寫最具體優先的鏈——例如 `NotFoundError` → `RateLimitError` → `APIStatusError` → `APIConnectionError`（或 Go 等效：`errors.As` 成 `*anthropic.Error` 然後 `switch apierr.StatusCode { case 404: ...; case 429: ...; default: ... }`）。各語言的類別名稱和命名空間在 `shared/error-codes.md`。
- **不要研究 SDK 型別——先寫。** 若型別名稱未在本 skill 所含文件中顯示，從語言專屬文件中的命名空間/套件表撰寫程式碼檔案，讓編譯器的錯誤指向正確的名稱。不要在寫作之前花費幾輪在 WebFetch、SDK 存放庫複製或編譯並執行單獨的反射程式來發現型別名稱——先產生原始碼檔案，然後修復編譯器回報的內容。對已安裝的 SDK 進行快速的 `strings` / `jar tf` / `javap` 以定位名稱是可以接受的（幾秒鐘回傳），但不要超出這個範圍。帶有錯誤型別名稱的檔案是可恢復的；在沒有撰寫任何檔案的情況下在探索上花費的 session 是無法恢復的。
- **Bash 和文字編輯器工具是 Anthropic 定義的，無綱要。** 宣告 `{"type": "bash_20250124", "name": "bash"}` / `{"type": "text_editor_20250728", "name": "str_replace_based_edit_tool"}`——無 `input_schema`。帶有您自己綱要的名為 `"bash"` 的自訂工具是不同的工具。處理器路徑和安全性檢查在 `shared/tool-use-concepts.md` § 用戶端工具。
- **顧問工具模型配對。** 顧問工具的 `model` 必須至少與請求的頂層 `model` 一樣強大——例如執行者 `claude-sonnet-5` → 顧問 `claude-opus-4-8` 或 `claude-opus-4-7`。無效的配對回傳 400。配對表在 `shared/tool-use-concepts.md` § Advisor。可用性：`shared/platform-availability.md`。
- **Agent Skills ≠ Managed Agents。** 若要讓 Claude 透過 Agent Skills 生成 `.pptx`/`.xlsx`/等，呼叫 `client.beta.messages.create`，帶有 `container={"skills": [...]}`、`code_execution_20260521` 工具，以及 `code-execution-2025-08-25` beta（Skills 已出 beta——不需要 `skills-2025-10-02` 標頭）。不要在這裡使用 `client.beta.agents` / `sessions` / `environments`——那些是 Managed Agents 介面，不是 Agent Skills。
- **MCP 連接器需要兩個半部分。** 單獨的 `mcp_servers=[{type:"url", url, name}]` 被拒絕為驗證錯誤——也要新增帶有 beta `mcp-client-2025-11-20` 的 `tools=[{type:"mcp_toolset", mcp_server_name:<same name>}]`。可用性：`shared/platform-availability.md`。
- **`inference_geo` 是直接的頂層請求參數** — `client.messages.create(..., inference_geo="us")` / `.inferenceGeo("us")`。不要放在 `extra_body` / `putAdditionalBodyProperty` 中。（僅 Messages API——在 Managed Agents 上，`inference_geo` 改為巢狀在 agent 的 `model` 物件內，從不頂層；見 `shared/managed-agents-core.md` § 釘選推理地理位置。）在 Opus 4.6 / Sonnet 4.6 及更新版本上支援；可用性：`shared/platform-availability.md`。`response.usage.inference_geo` 回報推理執行的位置。
- **細粒度工具串流不是 beta 功能。** 在工具定義上設定 `eager_input_streaming: true`，並呼叫常規 `client.messages.stream(...)`。沒有 beta 標頭，也沒有 `client.beta.*` 路徑。
- **快取診斷是 beta。** 使用帶有 beta `cache-diagnosis-2026-04-07` 的 `client.beta.messages.*`。在第一輪傳遞 `diagnostics: {previous_message_id: null}`，在後續輪次傳遞 `diagnostics: {previous_message_id: <previous response id>}`；結果在 `response.diagnostics` 上。可用性：`shared/platform-availability.md`。
- **記憶體工具類型是 `memory_20250818`。** 宣告 `{"type": "memory_20250818", "name": "memory"}`。Go 在 `client.Beta.Messages.New` 上使用 beta 命名空間類型 `{OfMemoryTool20250818: &anthropic.BetaMemoryTool20250818Param{}}`；Python/TypeScript/Ruby/PHP/C# 使用非 beta 的 `client.messages.create`；Java 同時有非 beta 的 `MemoryTool20250818` 和 beta tool-runner 路徑。Python/TypeScript 提供 `BetaAbstractMemoryTool` / `betaMemoryTool` 輔助方法用於實作後端。
- **使用功能實際支援的模型。** 某些功能受限於特定的模型層級——快速模式僅 Claude Opus 5 / Opus 4.8（且僅 Claude API），任務預算（僅 Messages API——Managed Agents session 預算沒有模型層級限制）僅 Claude Opus 5 / Fable 5 / Claude Fable 5.1（在發布時確認）/ Sonnet 5 / Opus 4.8 / 4.7，顧問工具需要有效的執行者↔顧問配對。若使用者的 prompt 指定了某個功能不支援的模型，改用支援的模型，並在輸出中注記替換。
- **不要為 SDK 資料結構定義自訂型別：** SDK 匯出所有 API 物件的型別。對於訊息使用 `Anthropic.MessageParam`，對於工具定義使用 `Anthropic.Tool`，對於工具結果使用 `Anthropic.ToolUseBlock` / `Anthropic.ToolResultBlockParam`，對於回應使用 `Anthropic.Message`。定義您自己的 `interface ChatMessage { role: string; content: unknown }` 重複了 SDK 已提供的內容，並丟失了型別安全。
- **回報和文件輸出：** 對於生成報告、文件或視覺化的任務，程式碼執行沙箱預裝了 `python-docx`、`python-pptx`、`matplotlib`、`pillow` 和 `pypdf`。Claude 可以生成格式化檔案（DOCX、PDF、圖表）並透過 Files API 回傳它們——對於「報告」或「文件」類型的請求，考慮這種方式而非純 stdout 文字。
- **伺服器工具錯誤不會提出例外。** 網路搜尋和網路擷取錯誤以 HTTP 200 回傳，帶有 `web_search_tool_result` / `web_fetch_tool_result` 區塊，其 `content` 是單一的錯誤物件（例如 `{error_code: "max_uses_exceeded"}`）——不是提出的例外。對於網路搜尋，成功的 `content` 是*列表*；錯誤的 `content` 是*物件*——在索引之前先分支。
- **Managed Agents 的網路工具會忽略環境的 `networking`。** `web_search` / `web_fetch` 在雲端與自託管環境中都於 Anthropic 伺服器上執行，Console 組織層級的網路設定僅套用於 Messages API。請在工具集的 `configs` 項目中，使用 `allowed_domains` **或** `blocked_domains`（不可同時使用；每個清單 1–64 個純主機名稱；涵蓋子網域；兩種工具都拒絕 IP、裸 TLD、單一標籤名稱與 `localhost` 類名稱；路徑尾碼只允許用於 `web_search`）限制它們——`shared/managed-agents-tools.md` § Web search & web fetch settings。
- **程式碼執行輸出區塊類型：** `code_execution_20260521` 回傳 `bash_code_execution_tool_result`（帶有 `.content.stdout`），**不是**舊版的裸 `code_execution_tool_result`。迭代 `response.content` 並匹配正確的類型。
- **工具搜尋：絕不延遲所有工具。** 搜尋工具本身不能有 `defer_loading: true`，且 `tools` 中至少有一個工具必須是非延遲的，否則 API 回傳 400 `All tools have defer_loading set`。
