---
source_file: prompt-caching.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 07d3ee1266f2825adc1413c4e561b1a09ba9304617f4057f782b0c87a72640fd
translated_at: 2026-09-06
---
<!-- translation-header -->
> 譯文說明：本文件由 DraftingDreamer 翻譯自 `prompt-caching.md`。程式碼區塊內容保持原文不變。
<!-- /translation-header -->

# Prompt Caching — 設計與最佳化

本文件說明如何設計 prompt 構建程式碼以實現有效的快取。語言特定的語法，請參見各語言 README 或單一文件的「## Prompt Caching」節。

## 一切都從這個不變數推導

**Prompt 快取是前綴比對。前綴中任何位置的任何變更都會使其後的一切失效。**

快取金鑰由渲染後的 prompt 到每個 `cache_control` 斷點的精確位元組衍生而來。位置 N 的單一位元組差異——一個時間戳記、一個重新排序的 JSON 鍵、列表中不同的工具——使位置 ≥ N 的所有斷點快取失效。

渲染順序為：`tools` → `system` → `messages`。最後一個 system 區塊上的斷點將 tools 和 system 一起快取。

圍繞此限制設計 prompt 構建路徑。正確排序，大多數快取自然運作。排錯了，再多 `cache_control` 標記也無濟於事。

---

## 最佳化現有程式碼的工作流程

當被要求新增或最佳化快取時：

1. **追蹤 prompt 組裝路徑。** 找出 `system`、`tools` 和 `messages` 的構建位置。識別流入其中的每個輸入。
2. **依穩定性分類每個輸入：**
   - 從不變更 → 屬於 prompt 的早期部分，在任何斷點之前
   - 每個 session 變更 → 屬於全域前綴之後，按 session 快取
   - 每個輪次變更 → 屬於末尾，在最後一個斷點之後
   - 每個請求變更（時間戳記、UUID、隨機 ID）→ **消除或移至最末端**
3. **確認渲染順序符合穩定性順序。** 穩定內容必須實際上先於易變內容。若時間戳記插入到 system prompt 標頭中，無論標記如何，其後的一切都無法快取。
4. **在穩定性邊界放置斷點。** 見下方的放置模式。
5. **審查靜默失效因素。** 見反模式表。

---

## 放置模式

### 跨許多請求共用的大型 system prompt

在最後一個 system 文字區塊上放置斷點。若有工具，它們渲染在 system 之前——最後一個 system 區塊上的標記將 tools + system 一起快取。

```json
"system": [
  {"type": "text", "text": "<large shared prompt>", "cache_control": {"type": "ephemeral"}}
]
```

### 多輪對話

在最近追加輪次的最後一個內容區塊上放置斷點。每個後續請求重用整個先前對話前綴。較早的斷點仍是有效的讀取點，因此隨著對話增長，命中率逐步累積。

```json
// Last content block of the last user turn
messages[-1].content[-1].cache_control = {"type": "ephemeral"}
```

### 共用前綴、變化後綴

許多請求共用大型固定前言（few-shot 範例、擷取的文件、指令），但最後的問題不同。將斷點放在**共用**部分的末尾，而非整個 prompt 的末尾——否則每個請求都寫入不同的快取項目，永遠不會被讀取。

```json
"messages": [{"role": "user", "content": [
  {"type": "text", "text": "<shared context>", "cache_control": {"type": "ephemeral"}},
  {"type": "text", "text": "<varying question>"}  // no marker - differs every time
]}]
```

### 對話中途的 system 訊息

**Claude Opus 5、Claude Opus 4.8、Claude Fable 5、Claude Fable 5.1、Claude Mythos 5，以及 Claude Mythos 5.1；無需 beta header。不適用於 Claude Sonnet 5**——在那裡使用頂層 `system`。（來源對 Claude Sonnet 5 有衝突：模型設定標記為支援，但所有規範說明文件頁面都省略了它。視為不支援並捕捉 400。）當操作者指令在對話中途到達——模式切換、更新的情境、動態注入的狀態——將其作為 `{"role": "system", "content": "..."}` 追加到 `messages[]`，而非編輯頂層 `system`。編輯頂層 `system` 會更改整個對話歷史前面的前綴，因此每個已快取的輪次都會在不命中快取的情況下重新處理；`role: "system"` 訊息位於歷史之後，使快取的前綴保持完整。

```json
// Top-level system stays byte-identical; new instruction goes after the cached history
"system": [{"type": "text", "text": "<stable core>", "cache_control": {"type": "ephemeral"}}],
"messages": [
  ...history,
  {"role": "user", "content": "..."},
  {"role": "system", "content": "Terse mode enabled - keep responses under 40 words."}
]
```

這也是在使用者輪次中嵌入操作者指令作為文字（`<system-reminder>` 模式）的防 prompt 注入替代方案：兩者的快取特性相同，但 `role: "system"` 是無法偽造的操作者通道，而使用者/工具內容中的文字可以被任何寫入使用者可見輸入的東西偽造。

必須跟在 `role: "user"` 訊息之後（或以伺服器工具使用結尾的 `assistant` 訊息之後），且必須是 `messages` 中的最後一個條目或後面跟著 `assistant` 輪次；不能是 `messages[0]`——初始 prompt 請使用頂層 `system`。內容僅限文字。不支援的模型回傳 400（`BadRequestError`：`role 'system' is not supported on this model`）；捕捉該錯誤並退回到將指令放入使用者輪次的 `<system-reminder>` 區塊。

**工具迴圈中的每輪提醒：輪次範圍的訊息，永不刪除。** 注入到歷史中、又在下一個請求時移除的提醒是一次歷史編輯——快取會從那個點開始錯失，且在 Claude Fable 5.1 / Claude Mythos 5.1 上，之後所有的 thinking 區塊都會失效。請改為給 `role: "system"` 訊息 `clear_at: "next_user_message"`（beta `mid-conversation-system-clear-at-2026-08-21`；與對話中途 system 訊息相同的模型和平台）：它只在一輪中渲染，之後便以清除狀態留在逐字稿中——不耗費輸入 token、不具快取資格（在其上設定 `cache_control` 會是 400；請將斷點放在前一個使用者輪次上），但仍是前綴的一部分。在每則 `tool_result` 訊息之後附加一份新複本，並保留較早的複本；若不使用該 beta，則在同一則使用者訊息中 `tool_result` 區塊之後放一個 `text` 區塊，並保留較早的複本。另外，每則訊息的 effort（beta `mid-conversation-output-config-2026-07-01`；Claude Fable 5.1、Claude Mythos 5.1、Claude Opus 5；限 Claude API）：帶有 `content: []` 和 `output_config: {effort: ...}` 的 `role: "system"` 訊息，會從下一個使用者輪次起變更 effort，且**不會**造成頂層 `effort` 變更所引發的 messages 快取失效，並且不受放置規則限制（可以放在任何位置）——見下方的失效層次結構，以及 `shared/model-migration.md` → 從 Claude Fable 5 遷移到 Claude Fable 5.1 → 新的 API 功能。

### 每次都從頭變化的 Prompt

不要快取。若前 1K token 每個請求都不同，就沒有可重用的前綴。新增 `cache_control` 只會付出快取寫入的溢價而毫無讀取。不要加它。

---

## 架構指導

這些決策比標記放置更重要。先修復這些。

**保持 system prompt 凍結。** 不要將「current date: X」、「mode: Y」、「user name: Z」插入 system prompt——那些位於前綴的前面，使下游所有內容失效。改為在 `messages` 中注入動態情境——在支援的地方作為 `{"role": "system", ...}` 訊息（見上方「對話中途的 system 訊息」），否則作為使用者訊息中的文字。第 5 輪的訊息不會使第 5 輪之前的任何快取失效。

**不要在對話中途更換工具或模型。** 工具在位置 0 渲染；新增、移除或重新排序工具會使整個快取失效。切換模型也是如此（快取是以模型為範圍的）。若您需要「模式」，不要替換工具集——給 Claude 一個記錄模式切換的工具，或將模式作為訊息內容傳入。確定性地序列化工具（依名稱排序）。

**分叉操作必須重用父操作的精確前綴。** 副計算（摘要、壓縮、子 agent）通常會啟動一個單獨的 API 呼叫。若分叉以任何差異重建 `system` / `tools` / `model`，它就完全錯過了父操作的快取。逐字複製父操作的 `system`、`tools` 和 `model`，然後在末尾追加特定於分叉的內容。

---

## 靜默失效因素

審查程式碼時，在所有流入 prompt 前綴的內容中搜尋這些：

| 模式 | 為何破壞快取 |
|---|---|
| `datetime.now()` / `Date.now()` / `time.time()` 在 system prompt 中 | 前綴每個請求都變更 |
| `uuid4()` / `crypto.randomUUID()` / 請求 ID 在內容早期 | 同上——每個請求都是唯一的 |
| 沒有 `sort_keys=True` 的 `json.dumps(d)` / 迭代 `set` | 非確定性序列化 → 前綴位元組不同 |
| f-string 將 session/使用者 ID 插入 system prompt | 每個使用者的前綴不同；無法跨使用者共用 |
| 條件式 system 節（`if flag: system += ...`） | 每種旗標組合都是不同的前綴 |
| `tools=build_tools(user)` 其中集合因使用者而異 | 工具在位置 0 渲染；沒有任何內容可以跨使用者快取 |

修復方法：將動態部分移到最後一個斷點之後、使其具確定性，或若它不是關鍵的就刪除它。

---

## API 參考

```json
"cache_control": {"type": "ephemeral"}              // 5-minute TTL (default)
"cache_control": {"type": "ephemeral", "ttl": "1h"} // 1-hour TTL
```

- 每個請求最多 **4** 個 `cache_control` 斷點。
- 可放在任何內容區塊上：system 文字區塊、工具定義、訊息內容區塊（`text`、`image`、`tool_use`、`tool_result`、`document`）。
- `messages.create()` 上的頂層 `cache_control` 自動放在最後一個可快取區塊上——當不需要細粒度放置時是最簡單的選項（§ 自動與明確斷點）。
- 在 Claude API、AWS 上的 Claude Platform，以及 Microsoft Foundry 上，快取是按工作區隔離的（在 Amazon Bedrock 和 Google Cloud 上則是按組織隔離），且絕不會跨組織共用。同一個 prompt 若分散在不同工作區的流量會寫入並讀取各自獨立的條目——在把命中率低歸咎於 prompt 之前，請先檢查這一點。
- 最小可快取前綴視模型而定。較短的前綴即使有標記也靜默不快取——沒有錯誤，只有 `cache_creation_input_tokens: 0`：

| 模型 | 最小值 |
|---|---:|
| Claude Opus 5、Claude Fable 5、Claude Mythos 5、Claude Fable 5.1、Claude Mythos 5.1 | 512 tokens |
| Opus 4.8、Claude Sonnet 5、Sonnet 4.6、Sonnet 4.5、Opus 4.1、Opus 4、Sonnet 4 | 1024 tokens |
| Opus 4.7、Mythos Preview、Haiku 3.5 | 2048 tokens |
| Opus 4.6、Opus 4.5、Haiku 4.5 | 4096 tokens |

**最小值在跨代之間不是單調的**——最新模型是 512，但 Opus 4.6/4.5 和 Haiku 4.5 是 4096。3K token 的 prompt 在 Claude Opus 5、Opus 4.8 和 Sonnet 4.5 上快取，在 Opus 4.6 或 Haiku 4.5 上靜默不快取。Claude Opus 5 將 Opus 4.8 的最小值減半（1024 → 512），因此先前太短而無法快取的 prompt 現在無需更改程式碼就能建立條目。

這些最小值適用於模型可用的**每個**平台——舊的 Amazon Bedrock 對 Claude Fable 5.1 的覆寫已移除，且沒有任何按平台的例外。

**經濟學：** 快取讀取成本約為基礎輸入價格的 0.1×——**在 Claude Fable 5.1 上為 0.025×**（每 MTok $0.25；Claude Mythos 5.1 是否共用此費率，在發布時仍未確定），這會讓下方的每個損益平衡點按比例下移。快取寫入成本為 **5 分鐘 TTL 1.25×，1 小時 TTL 2×**。損益平衡取決於 TTL：5 分鐘 TTL，兩個請求達到損益平衡（1.25× + 0.1× = 1.35× vs 未快取的 2×）；1 小時 TTL，您至少需要三個請求（2× + 0.2× = 2.2× vs 未快取的 3×）。1 小時 TTL 讓條目在突發性流量的間隙中保持活躍，但加倍的寫入成本意味著它需要更多讀取才能回本。

### 選擇 TTL

在兩種 TTL 下，快取讀取都會以零額外成本重新整理該條目的計時器。存活時間是從寫入或讀取該條目的請求的**開始**時間起算——生成時間也算在內，因此一次 4 分鐘的生成，只留給下一個請求約 1 分鐘的時間得以在 5 分鐘條目過期前開始。共用前綴且彼此開始時間相差不到 5 分鐘的請求，能讓 5 分鐘快取無限期保持溫熱——1 小時 TTL 在那裡除了讓寫入價格加倍之外沒有任何好處。請依共用前綴的請求之間「開始時間到開始時間」的間隔來選擇：

| 共用前綴的請求之間「開始時間到開始時間」的間隔 | TTL |
|---|---|
| 低於 5 分鐘（連續流量；輪次生成時間遠低於 5 分鐘的 agent 迴圈） | 5 分鐘——每個請求都會重新整理它；嚴格來說更便宜 |
| 5–60 分鐘（20 分鐘後才回覆的使用者；讀取間隔超過 5 分鐘的 agentic 附加工作或生成） | 1 小時——唯一能讓 2× 的寫入成本回本的區間 |
| 超過一小時 | 兩者都沒有直接幫助——按排程重新預熱（§ 預熱快取）或接受冷未命中 |

**Claude Fable 5.1 / Claude Mythos 5.1：keep-alive 通常比 1 小時 TTL 更便宜。** 由於 Claude Fable 5.1 上的快取讀取費率是 0.025 倍（其他地方是 0.1 倍；Claude Mythos 5.1 是否共用此費率，發布時仍未確定——見上方的經濟學）未命中相對於命中要昂貴得多，而讀取幾乎是免費的——因此對於 5–60 分鐘的間隔，與其為 1 小時 TTL 支付 2 倍的寫入成本，不如維持預設的 5 分鐘 TTL，並在閒置期間、於條目即將過期前不久，用 `max_tokens: 0` 重新送出前一個請求。該請求會重新整理條目的計時器，且只計費一次便宜的快取讀取（沒有輸出 token）。以 Claude Fable 5.1 的價格來說，除非停頓經常逼近一小時，否則這會比 1 小時 TTL 更划算。`max_tokens: 0` 依循 § 預熱一節中列出的拒絕組合；在這些模型上可能出現的是 `stream: true`、結構化輸出和 Batches（強制 `tool_choice` 和 `thinking.type: "enabled"` 在這裡本來就已是 400）。傳送 keep-alive 時請關閉 `stream`——串流是傳輸層的選項，不屬於已快取的前綴，因此為這一個請求關閉它不會有任何成本——而在請求無法這樣調整的情況下（結構化輸出 `output_config.format`，或位於 Message Batches 請求內），請改用 1 小時 TTL。Prompt 快取頁面（`shared/live-sources.md`）提供了一個範例工作負載的成本比較，以及一個 keep-alive 請求的範例。

在 Claude API 上，快取讀取在大多數模型上也不計入輸入 token 的頻率限制（Haiku 3.5 是文件記載的例外——見頻率限制文件），因此讓條目跨間隙保持活躍，除了降低成本外，還能提升有效輸送量。

---

## 自動與明確斷點

自動快取是請求上的一個頂層 `cache_control` 欄位，而非放在任何內容區塊上。系統會將斷點放在最後一個可快取區塊上，並隨著對話增長向前移動；若最後一個區塊不是合格的目標，它會靜默向後尋找最近的合格區塊，若找不到則跳過快取。自動斷點預設為 5 分鐘 TTL（頂層欄位可接受 `ttl: "1h"`），並會佔用 4 個斷點名額中的一個。它可以與同一請求中的明確標記並存，但有兩種已記載的 400 情況：4 個名額已全部被明確標記佔用，以及最後一個區塊上有一個 TTL 與頂層欄位不同的明確標記（若該處的明確標記 TTL 相同，則自動快取會變成無操作）。

對多輪對話而言，自動是正確的預設選擇——也就是上方的多輪對話放置模式，且無需管理標記。請在以下情況使用明確斷點：

| 情況 | 為何自動不是正確的工具 |
|---|---|
| Prompt 以每個請求各不相同的內容結尾（擷取的資料列、每個請求的情境、一次性的問題） | 自動斷點會落在這個獨特尾端之後，因此每個請求都要為永遠不會被讀回的位元組支付寫入溢價——純屬額外開銷。特徵是：每個請求都有 `cache_creation_input_tokens`，而 `cache_read_input_tokens` 永遠涵蓋不到完整的共用前綴。請改為在共用部分的末尾放置明確標記（§ 共用前綴、變化後綴）。 |
| 各區段以不同頻率變化（工具從不變、情境每天變、對話每輪變） | 自動只會放置一個斷點；多個穩定性邊界需要明確標記。 |
| 某個區塊應為 1 小時 TTL，另一個應為 5 分鐘 | 逐區塊 TTL 需要明確標記——且較長 TTL 的條目必須出現在較短的條目之前（1 小時的條目必須出現在任何 5 分鐘條目之前）。 |
| 單一輪次新增超過 20 個位置（連續的 tool_use 串、以及 tool_result 串，各自收合為一個位置） | 回溯可能會錯過前一個條目——見 § 20 個區塊的回溯視窗。 |
| 不支援自動快取的平台或整合（查閱 `shared/platform-availability.md`） | 頂層欄位在那裡會被拒絕——只能使用明確標記。 |

**適用於 agent 迴圈的穩健組合：** 在靜態 system 前綴的最後一個區塊上放置一個明確斷點——這個昂貴的共用部分因此獲得一個保證的讀取點，不受 `messages` 中之後發生的任何事影響——再加上針對不斷增長的對話尾端使用頂層自動快取（在自動快取可用之處——見 `shared/platform-availability.md`）。

---

## 驗證快取命中

回應的 `usage` 物件報告快取活動：

| 欄位 | 意義 |
|---|---|
| `cache_creation_input_tokens` | 此請求寫入快取的 token（您支付了約 1.25× 的寫入溢價） |
| `cache_read_input_tokens` | 此請求從快取提供的 token（您支付了約 0.1×） |
| `input_tokens` | 以全價處理的 token（未快取） |

若在具有相同前綴的重複請求中 `cache_read_input_tokens` 為零，就有靜默失效因素在起作用——比較兩個請求之間渲染的 prompt 位元組差異以找到它。

**`input_tokens` 只是未快取的餘量。** 總 prompt 大小 = `input_tokens + cache_creation_input_tokens + cache_read_input_tokens`。若您的 agent 執行了幾個小時但 `input_tokens` 顯示 4K，其餘的都是從快取提供的——查看總和，而非單一欄位。

語言特定的存取：`response.usage.cache_read_input_tokens`（Python/TS/Ruby）、`$message->usage->cacheReadInputTokens`（PHP）、`resp.Usage.CacheReadInputTokens`（Go/C#）、`.usage().cacheReadInputTokens()`（Java）。

**每次變更後都要驗證，而不只是在初次設定時。** 生產環境中代價最高的快取失效是無聲的：請求持續成功，只是帳單變高了——沒有錯誤，也沒有任何東西會宣告這件事。典型的形態是一次退化，而非一開始就沒做好：快取在寫好時能運作，然後後續對 prompt 組裝的一次變更（system prompt 中新增的動態欄位、一個會重寫歷史的功能、一份不再具確定性的工具清單）使每個請求都錯失命中，且數月都沒被人發現。`usage` 欄位是快取確實在運作的唯一真實依據。每當 prompt 組裝程式碼變更時就重新檢查它們，並且比起只看一次，更該偏好一種常態檢查——例如整合測試斷言第二個相同的請求會顯示 `cache_read_input_tokens > 0`，或是對 usage 欄位進行監控。

**健康迴圈的特徵。** 寫入只會針對超出目前最高快取命中點的差量計費，因此在穩定的多輪迴圈中，每個請求應該讀取目前為止累積的一切，並只寫入最後一輪新增的部分：

- `cache_read_input_tokens`——先前的整個前綴；隨輪次增長
- `cache_creation_input_tokens`——大致上是前一次 assistant 輸出加上新追加的輸入；相對於整個對話很小
- `input_tokens`——僅是最後一個斷點之後的尾端部分

若 `cache_creation_input_tokens` 反而在每個請求上都接近整個對話的大小，那麼要嘛是前綴在斷點之前的部分被重寫了，要嘛這次寫入的發生原因是 payload 差異比對和快取診斷都無法定位的——在啟用 thinking、且模型會剝除前一輪 thinking 區塊的情況下，失效是發生在伺服器端的（見 § 失效層次結構）；而單一輪次新增超過 20 個位置（平行工具呼叫串會收合為一個位置——見 § 20 個區塊的回溯視窗）會將前一個條目推出回溯範圍，導致每個請求都以逐位元組相同的 payload 重寫整段對話（見 § 20 個區塊的回溯視窗）。請先從模型與輪次形狀排除這兩種「什麼都看不出來」的情況。讀取只能落在前一個請求寫入斷點的位置上，因此 usage 欄位只能告訴您前綴*確實*壞掉了（讀取會崩塌，經常變成零），卻不能告訴您在哪裡壞掉——下方的 payload 差異比對或快取診斷能定位出確切的位置。

**找出失效因素。** 記錄幾個連續的請求 payload（完整的 JSON 主體），並比較相鄰的成對差異。在持續增長的對話中，相鄰的 payload 理應在末尾（新追加的輪次）有差異；必須逐位元組相同的是重疊部分——前一個請求的 prompt 應該原封不動地以前綴的形式重新出現在下一個請求中。比對之前請先去除 `cache_control` 標記：移動中的標記在相鄰請求之間必然不同，這不是一種失效因素（先前標記過的區塊仍是快取命中）。在重疊區域內第一個剩餘的差異點，就是失效發生的位置。這能抓到一整類程式碼審查會漏掉的錯誤——非確定性序列化、程式庫重新排序鍵或欄位、一個在請求之間變化、但在單次請求內不變的值。在 Claude API 上，快取診斷（beta header `cache-diagnosis-2026-04-07`）一旦您選擇加入，就會在伺服器端執行這種比對：請在**每個**請求上都送出此 header——指紋只會為帶有該 header 的請求儲存，因此事後才補加只會以 `previous_message_not_found` 失敗——然後將前一個回應的 `id` 作為 `diagnostics.previous_message_id` 傳入，回應的 `diagnostics` 物件就會指出兩個請求在何處出現分歧（model、system、tools 或訊息歷史）。不需要記錄 payload。可用性：見 `shared/platform-availability.md`。

**無法解釋的寫入：** `usage.cache_creation` 會按 TTL 拆分 `cache_creation_input_tokens`（`ephemeral_5m_input_tokens` / `ephemeral_1h_input_tokens`）。當請求已經在使用快取時，像網路搜尋這樣的伺服器工具，會在工具結果之後自動插入一次 5 分鐘的快取寫入——這是發生在您沒有標記的位置上的寫入；這是預期行為，不是失效因素。

---

## 失效層次結構

並非每個參數變更都會使一切失效。API 有三個快取層，變更只會使自己的層及以下的層失效：

| 變更 | Tools 快取 | System 快取 | Messages 快取 |
|---|:---:|:---:|:---:|
| 工具定義（新增/移除/重新排序） | 否 | 否 | 否 |
| 模型切換 | 否 | 否 | 否 |
| `speed`、網路搜尋、引用切換 | 是 | 否 | 否 |
| System prompt 內容 | 是 | 否 | 否 |
| `tool_choice`、圖片 | 是 | 是 | 否 |
| `thinking` 或 `effort` 變更 | 因模型而異 | 因模型而異 | 否 |
| 訊息內容 | 是 | 是 | 否 |

含義：您可以逐請求變更 `tool_choice` 而不失去 tools+system 快取，且訊息內容的變更完全不會動到它。Thinking 和 `effort` 的變更則永遠會使 messages 快取失效，而在那些將 thinking 設定渲染在 tools 和 system 之前的模型上，這些變更也會連帶使那些快取失效——請針對每條路由釘死 thinking 和 effort 設定，而不要逐請求變動它們。只有工具定義和模型變更，才會在每個模型上都強制完全重建。

**這些行中有三個有保留快取的逃生出口**——tools 那一行、system prompt 那一行，以及（在 Claude Fable 5.1 / Claude Mythos 5.1 / Claude Opus 5 上）`effort` 那一行——各自透過將變更移出頂層請求、移入 `messages[]` 中快取前綴之後的 system 訊息來達成。「注入後刪除」的提醒模式有它自己的出口：在使用者訊息中 `tool_result` 區塊之後附加一個文字區塊，永不刪除。**可用性因行而異**——它們並非一起開放的：

| 會失效的頂層變更 | 保留快取的形式 | 可用於 |
|---|---|---|
| 工具定義（新增/移除） | `tool_addition` / `tool_removal` 區塊——見 `shared/tool-use-concepts.md` § 對話中途的工具變更 | Claude Opus 5 起，在 `mid-conversation-tool-changes-2026-07-01` 後面 |
| System prompt 內容 | 一則 `{"role": "system", "content": "..."}` 訊息——見上方「對話中途的 system 訊息」 | Claude Opus 5、Claude Opus 4.8、Claude Fable 5、Claude Fable 5.1、Claude Mythos 5、Claude Mythos 5.1——**現在即可使用**，無需 beta header |
| 每輪提醒（注入後於下一個請求刪除） | 一則輪次範圍的 `clear_at: "next_user_message"` system 訊息，留在逐字稿中——見上方「對話中途的 system 訊息」（不使用該 beta 時：在 `tool_result` 區塊之後放一個文字區塊，並保留較早的複本） | 與對話中途 system 訊息相同的模型，在 `mid-conversation-system-clear-at-2026-08-21` 後面 |
| `effort` 變更 | 一則 `{"role": "system", "content": [], "output_config": {"effort": ...}}` 訊息——見 `shared/model-migration.md` → 從 Claude Fable 5 遷移到 Claude Fable 5.1 | Claude Fable 5.1、Claude Mythos 5.1、Claude Opus 5，在 `mid-conversation-output-config-2026-07-01` 後面 |
| 被捨棄的 thinking 區塊（將 Claude Fable 5.1 / Claude Mythos 5.1 的區塊回放給無法讀取它的模型，或歷史編輯檢查的 `drop_block`） | 沒有——API 會在該請求上捨棄該區塊，且 messages 快取會從該位置起變更；tools 和 system 快取則保持完整。接收端模型能讀取、且原封不動傳回的區塊，則會讓快取保持完整 | - |

模型切換沒有逃生出口：快取是以模型為範圍的。在一個模型上保持主迴圈，並為較便宜的子任務生成子 agent（見 `agent-design.md` § Agent 的快取）。

**Thinking 區塊與 messages 快取（因模型而異）。** 在 Claude Fable 5、Claude Fable 5.1、Claude Mythos 5、Claude Mythos 5.1、Mythos Preview、Opus 4.5 及更新版本，以及 Sonnet 4.6 及更新版本上，前一輪的 thinking 區塊預設會被保留，因此在啟用 thinking 的情況下傳入一則一般（非 tool_result）的使用者訊息，仍會讓 messages 快取保持有效。在更早的 Opus 和 Sonnet 模型，以及所有到 Haiku 4.5 為止的 Haiku 模型上，同一個請求會把先前已快取的 thinking 區塊從情境中剝除，且第一個被剝除的區塊之後的每則訊息都會跌出快取——在 agent 迴圈中，這會表現為：在一般使用者訊息緊接在工具使用之後的輪次上，`cache_creation_input_tokens` 出現尖峰。（在請求之間切換 thinking 開/關，是另一種對所有模型都成立的 messages 快取失效因素——見上方的層次結構表。變更 `output_config.effort` 的行為與變更 thinking 參數相同；明確設定模型的預設 effort 等同於省略它，因此釘死預設值不會有任何成本。）

---

## 20 個區塊的回溯視窗

每個斷點最多向後走 **20 個位置**以尋找先前的快取條目。在 Claude API 上，連續一串 `tool_use` 區塊算作一個位置，連續一串 `tool_result` 區塊也是如此，因此有許多*平行*工具呼叫的輪次不會把前一個請求的條目推出視窗；但新增超過 20 個位置的其他內容的輪次（長串的循序工具迴圈、許多文字/圖片區塊）仍然可能——下一個請求的斷點會找不到先前的快取，並靜默錯失。

修復：在長輪次中每約 15 個位置放置一個中間斷點，或將標記放在距前一輪次最後一個已快取區塊 20 個位置以內的區塊上。

---

## 並發請求時序

快取條目只有在第一個回應**開始串流後**才可讀。N 個具有相同前綴的並行請求都付全價——它們都無法讀取彼此還在寫入的內容。

對於扇出模式：送出 1 個請求，等待第一個串流 token（而非完整回應），然後發出剩餘的 N−1 個。它們會讀取第一個剛寫入的快取。

同樣的算術也影響多 agent 設計：N 個平行 worker，各自在相同情境上組裝略有差異的 prompt，會寫入 N 個各自獨立的快取條目，且互相都讀不到對方的。當輸入成本佔主導時，改用較少的並行路線、共用逐位元組相同的前綴——或改由一個 worker 依序執行 N 次——能把那些寫入轉換為讀取。

## 預熱快取

要消除*第一個*真實請求的快取未命中延遲，在啟動時（或按間隔）發送 **`max_tokens: 0`** 請求。API 執行 prefill——在您的 `cache_control` 斷點寫入快取——並立即回傳 `content: []`、`stop_reason: "max_tokens"` 和已填寫的 `usage` 區塊（無輸出 token 計費；`cache_creation_input_tokens` 上的正常快取寫入費用）。

**何時預熱**——預熱以*現在*的快取寫入費用換取*下一個*真實請求的較低 TTFT。當三個條件都成立時值得：(a) 第一個請求的延遲對使用者可見（聊天/語音/互動——而非背景作業），(b) 共用前綴夠大，冷寫入速度明顯慢，以及 (c) 流量*之前*有一個時機觸發它——應用程式啟動、worker 開機、部署後、排程視窗開始。

| 跳過預熱的時機... | 因為 |
|---|---|
| 流量是連續的（請求間隔 ≤ TTL） | 第一個真實請求預熱快取，之後每個都命中；單獨的預熱呼叫是純粹的額外寫入 |
| 前綴小或低於可快取最小值 | 冷寫入的成本可忽略 |
| 前綴因請求/使用者而異 | 沒有共用的東西可預熱 |
| 您要推測性地預熱許多不同的前綴 | 每個約 1.25× 的寫入；成本可能超過您節省的延遲 |

**排程重新預熱：** 只有在流量有比 TTL 更長的間隙時才需要。若真實請求的到達頻率高於每 5 分鐘，它們本身就能保持快取的熱度——不要加間隔重新預熱。對於有長時間閒置間隙的突發性流量，要麼在快取 TTL 到期前重新預熱，要麼切換到 `ttl: "1h"` 並減少預熱頻率。

```python
client.messages.create(
    model="claude-opus-5",
    max_tokens=0,
    system=[{
        "type": "text",
        "text": SYSTEM_PROMPT,
        "cache_control": {"type": "ephemeral"},
    }],
    messages=[{"role": "user", "content": "warmup"}],
)
```

**斷點放置：** 將 `cache_control` 放在**與真實請求共用的最後一個區塊**上（system prompt 或工具定義）——**不是**在佔位符使用者訊息上，**不是**透過頂層自動快取（那會以佔位符為金鑰）。佔位符可以是任何非空白字串；它在 prefill 期間被讀取但永不被回答。

**拒絕的組合：** 在 `stream: true`、`thinking.type: "enabled"`、`output_config.format`、`tool_choice` 為 `{"type":"tool"}` 或 `{"type":"any"}`，或在 Message Batches 請求內時，`max_tokens: 0` 是 `invalid_request_error`。

**TTL 仍適用**——預設快取每至少 5 分鐘重新預熱，或使用 1 小時 TTL。這取代了舊的 `max_tokens: 1` 解決方案（沒有需要丟棄的單 token 回覆，無輸出 token 計費，意圖明確）。
