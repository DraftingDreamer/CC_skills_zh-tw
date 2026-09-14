---
source_file: SKILL.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: dcd4803e61e913e6fc27294184cd3a71f09f5e924ff20c8a9a20173e7b3c2bcf
translated_at: 2026-08-22
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`SKILL.md`](SKILL.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
>
> **原檔 YAML frontmatter**
> - `name`: `skill-creator`
> - `description`（中譯）：建立新 skill、修改並改進現有 skill，以及衡量 skill 效能。當使用者想要從頭建立 skill、編輯或最佳化現有 skill、執行 eval 來測試 skill、以變異度分析進行 skill 效能基準測試，或最佳化 skill 的 description 以提升觸發準確率時，請使用本 skill。
<!-- /translation-header -->

# Skill Creator

用於建立新 skill 並反覆改進的 skill。

概括而言，建立 skill 的流程如下：

- 決定您希望 skill 做什麼，以及大致應如何實現
- 撰寫 skill 的草稿
- 建立幾個測試 prompt，並在這些 prompt 上執行「具備 skill 的 claude」
- 協助使用者對結果進行質性與量性評估
  - 當執行在背景進行時，若沒有現成的量性 eval，則起草一些（若已有，可以直接使用或在認為需要修改時加以調整）。然後向使用者說明（或者，若已存在，說明已有的那些）
  - 使用 `eval-viewer/generate_review.py` 指令碼向使用者展示結果，也讓他們查看量性指標
- 根據使用者對結果評估的意見（以及量性基準測試中發現的任何明顯缺陷）重寫 skill
- 重複直到您滿意為止
- 擴大測試集並在更大規模上再試一次

您使用本 skill 時的工作，是找出使用者在這個流程中的所在位置，然後介入並協助他們推進這些階段。例如，也許他們說「我想建立一個做 X 的 skill」。您可以幫助縮小他們的意思、撰寫草稿、撰寫測試案例、確定他們想要如何評估、執行所有 prompt，並重複這個過程。

另一方面，也許他們已經有了 skill 的草稿。在這種情況下，您可以直接進入評估／迭代的環節。

當然，您應該始終保持彈性，如果使用者說「我不需要跑一堆評估，就憑感覺來吧」，您也可以那樣做。

完成 skill 之後（但同樣地，順序是彈性的），您也可以執行 skill description 改進器——我們有一個獨立的指令碼——以最佳化 skill 的觸發。

好的。

## 與使用者溝通

skill creator 很可能被各種熟悉程度的人使用，從完全不懂程式術語到相當熟悉電腦的人都有。Claude 的強大能力正在激勵水電工開啟終端機、父母與祖父母搜尋「如何安裝 npm」——雖然這是很新近才開始出現的趨勢。另一方面，大多數使用者可能是相當熟悉電腦的。

所以請留意上下文線索，了解如何措辭！以下給您一些預設情況下的判斷依據：

- 「evaluation」與「benchmark」算是邊界情況，但可以接受
- 對於「JSON」和「assertion」，您需要看到使用者明確的線索表明他們知道這些是什麼，才能不加解釋地使用

如有疑問，簡短解釋術語是可以的，若不確定使用者是否理解，請隨時用簡短定義釐清術語。

---

## 建立 skill

### 掌握意圖

從理解使用者的意圖開始。目前的對話中可能已包含使用者想要捕捉的工作流程（例如，他們說「把這個變成一個 skill」）。若是如此，請先從對話記錄中提取答案——使用的工具、步驟的順序、使用者做出的修正、觀察到的輸入/輸出格式。使用者可能需要填補空缺，並應在繼續下一步之前確認。

1. 這個 skill 應使 Claude 能夠做什麼？
2. 這個 skill 應在何時觸發？（哪些使用者片語/情境）
3. 預期的輸出格式是什麼？
4. 我們應該設定測試案例來驗證 skill 是否有效嗎？具有客觀可驗證輸出的 skill（檔案轉換、資料提取、程式碼生成、固定工作流程步驟）受益於測試案例。具有主觀輸出的 skill（寫作風格、藝術）通常不需要。根據 skill 類型建議適當的預設設定，但讓使用者決定。

### 訪談與研究

主動詢問關於邊緣情況、輸入/輸出格式、範例檔案、成功標準和相依關係的問題。等到這部分確定後再撰寫測試 prompt。

檢查可用的 MCP——若對研究有用（搜尋文件、尋找類似 skill、查閱最佳實踐），若有 sub-agent 可用則透過 sub-agent 平行研究，否則直接進行。帶著脈絡準備好，以減少對使用者的負擔。

### 撰寫 SKILL.md

根據使用者訪談，填寫以下元件：

- **name**：skill 識別碼
- **description**：何時觸發、做什麼。這是主要的觸發機制——同時包含 skill 的功能和使用時機。所有「何時使用」的資訊放在這裡，而不是放在正文中。注意：目前 Claude 有「觸發不足」skill 的傾向——在有用的時候不使用它們。為了對抗這一點，請讓 skill 的 description 稍微「積極主動」一些。例如，不要寫「如何建立顯示 Anthropic 內部資料的簡單快速儀表板。」，而是寫「如何建立顯示 Anthropic 內部資料的簡單快速儀表板。請確保在使用者提到儀表板、資料視覺化、內部指標或想要展示任何公司資料時使用本 skill，即使他們沒有明確要求『儀表板』。」
- **compatibility**：所需工具、相依項（可選，很少需要）
- **其餘的 skill 內容 :)**

### Skill 撰寫指南

#### Skill 的結構

```
skill-name/
├── SKILL.md (required)
│   ├── YAML frontmatter (name, description required)
│   └── Markdown instructions
└── Bundled Resources (optional)
    ├── scripts/    - Executable code for deterministic/repetitive tasks
    ├── references/ - Docs loaded into context as needed
    └── assets/     - Files used in output (templates, icons, fonts)
```

#### 漸進式揭露

skill 使用三層載入系統：
1. **Metadata**（name + description）——始終在 context 中（約 100 字）
2. **SKILL.md 正文**——skill 觸發時即在 context 中（理想情況下 < 500 行）
3. **附帶資源**——按需（無限制，指令碼可在不載入的情況下執行）

這些字數是近似值，若有需要可以寫得更長。

**關鍵模式：**
- 保持 SKILL.md 在 500 行以內；若接近此限制，請增加額外的層級，並附上清楚的指引說明使用 skill 的模型應去哪裡繼續。
- 在 SKILL.md 中清楚地參照檔案，並附上何時讀取的指引
- 對大型參考檔案（> 300 行），請包含目錄

**領域組織**：當 skill 支援多個領域/框架時，按變體組織：
```
cloud-deploy/
├── SKILL.md (workflow + selection)
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```
Claude 只讀取相關的參考檔案。

#### 無驚訝原則

這不言而喻，但 skill 絕不可包含惡意軟體、漏洞利用程式碼或任何可能危害系統安全的內容。若依 description 描述，skill 的內容不應讓使用者對其意圖感到驚訝。不要配合建立具有誤導性的 skill，或設計用來促進未授權存取、資料洩露或其他惡意活動的 skill。不過，像「角色扮演成 XYZ」這類要求是可以的。

#### 寫作模式

在指令中優先使用命令式語氣。

**定義輸出格式**——您可以這樣做：
```markdown
## Report structure
ALWAYS use this exact template:
# [Title]
## Executive summary
## Key findings
## Recommendations
```

**範例模式**——包含範例很有用。您可以像這樣格式化它們（但若範例中有「Input」和「Output」，您可能需要稍作調整）：
```markdown
## Commit message format
**Example 1:**
Input: Added user authentication with JWT tokens
Output: feat(auth): implement JWT-based authentication
```

### 寫作風格

試著解釋您要求模型做某事的**原因**，而不是用重手的「MUST」。使用心智理論，試著讓 skill 通用，而非過度局限於特定範例。先撰寫草稿，然後以全新的視角審視並改進它。

### 測試案例

撰寫 skill 草稿後，想出 2-3 個實際的測試 prompt——那種真實使用者會說的話。與使用者分享：「這裡有幾個我想試試的測試案例。這些看起來正確嗎，還是您想要增加更多？」然後執行它們。

將測試案例儲存至 `evals/evals.json`。先不要撰寫斷言——只是 prompt。您將在下一步驟中，在執行進行的同時起草斷言。

```json
{
  "skill_name": "example-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "User's task prompt",
      "expected_output": "Description of expected result",
      "files": []
    }
  ]
}
```

完整綱要（包含 `assertions` 欄位，稍後會新增）請參閱 `references/schemas.md`。

## 執行和評估測試案例

本節是一個連續的序列——不要中途停止。不要使用 `/skill-test` 或任何其他測試 skill。

將結果放在 `<skill-name>-workspace/` 中，作為 skill 目錄的同層目錄。在工作區內，按迭代組織結果（`iteration-1/`、`iteration-2/` 等），每個測試案例在其中有一個目錄（`eval-0/`、`eval-1/` 等）。不要預先建立所有這些——隨需建立目錄。

### 步驟一：在同一輪中生成所有執行（有 skill 和基線）

對每個測試案例，在同一輪中生成兩個 sub-agent——一個帶 skill，一個不帶。這很重要：不要先生成帶 skill 的執行，然後回頭再做基線。一次全部啟動，讓它們差不多同時完成。

**帶 skill 的執行：**

```
Execute this task:
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**基線執行**（相同 prompt，但基線取決於情境）：
- **建立新 skill**：完全沒有 skill。相同 prompt，無 skill 路徑，儲存至 `without_skill/outputs/`。
- **改進現有 skill**：舊版本。在編輯前，對 skill 建立快照（`cp -r <skill-path> <workspace>/skill-snapshot/`），然後將基線 sub-agent 指向快照。儲存至 `old_skill/outputs/`。

為每個測試案例撰寫一個 `eval_metadata.json`（斷言暫時可以為空）。根據測試內容給每個 eval 一個描述性名稱——而不只是「eval-0」。也以這個名稱作為目錄名稱。若本次迭代使用了新的或修改過的 eval prompt，請為每個新的 eval 目錄建立這些檔案——不要假設它們會從上一個迭代延續。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 步驟二：執行進行時，起草斷言

不要只是等待執行完成——您可以善用這段時間。為每個測試案例起草量性斷言，並向使用者說明。若 `evals/evals.json` 中已有斷言，請審閱並解釋它們檢查的內容。

好的斷言是客觀可驗證的，且有描述性名稱——它們在基準測試檢視器中應讀起來清晰，讓瀏覽結果的人立即理解每個斷言在檢查什麼。主觀的 skill（寫作風格、設計品質）最好進行質性評估——不要強行對需要人工判斷的事物加上斷言。

一旦起草完成，更新 `eval_metadata.json` 檔案和 `evals/evals.json` 中的斷言。也向使用者說明他們在檢視器中會看到什麼——包括質性輸出和量性基準測試。

### 步驟三：執行完成時，擷取計時資料

當每個 sub-agent 任務完成時，您會收到包含 `total_tokens` 和 `duration_ms` 的通知。立即將此資料儲存至執行目錄的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

這是擷取此資料的唯一機會——它透過任務通知傳遞，不會持久化到其他地方。隨到隨處理每個通知，而不要嘗試批次處理。

### 步驟四：評分、彙總並啟動檢視器

所有執行完成後：

1. **評分每次執行**——生成一個評分器 sub-agent（或直接評分），讀取 `agents/grader.md`，並對照輸出評估每個斷言。將結果儲存至每個執行目錄的 `grading.json`。grading.json 的 expectations 陣列必須使用 `text`、`passed` 和 `evidence` 欄位（而非 `name`/`met`/`details` 或其他變體）——檢視器依賴這些確切的欄位名稱。對於可以程式化檢查的斷言，請撰寫並執行指令碼，而不是用眼睛看——指令碼更快、更可靠，且可以在各迭代中重複使用。

2. **彙總成基準測試**——從 skill-creator 目錄執行彙總指令碼：
   ```bash
   python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
   ```
   這會產生 `benchmark.json` 和 `benchmark.md`，其中包含各設定的 pass_rate、時間和 token，以及平均值 ± 標準差和差異值。若手動生成 benchmark.json，請參閱 `references/schemas.md` 了解檢視器所預期的確切綱要。
   每個 with_skill 版本放在其基線對應版本之前。

3. **進行分析師複盤**——讀取基準測試資料，呈現彙總統計可能隱藏的模式。請參閱 `agents/analyzer.md`（「分析基準測試結果」章節）了解要注意的事項——例如始終通過的斷言（無論 skill 如何，意味著非區分性）、高變異度 eval（可能不穩定），以及時間/token 的取捨。

4. **啟動檢視器**，同時包含質性輸出和量性資料：
   ```bash
   nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
   ```
   對於迭代 2+，也傳入 `--previous-workspace <workspace>/iteration-<N-1>`。

   **Cowork / 無頭環境：** 若 `webbrowser.open()` 無法使用或環境沒有顯示器，請使用 `--static <output_path>` 寫入獨立的 HTML 檔案，而不是啟動伺服器。當使用者點擊「Submit All Reviews」時，意見回饋將作為 `feedback.json` 檔案下載。下載後，將 `feedback.json` 複製到工作區目錄，以便下次迭代取用。

   注意：請使用 generate_review.py 建立檢視器；不需要撰寫自訂 HTML。

5. **告知使用者**，例如：「我已在您的瀏覽器中開啟結果。有兩個分頁——『Outputs』讓您點選每個測試案例並留下意見回饋，『Benchmark』顯示量性比較。完成後，請回來告訴我。」

### 使用者在檢視器中看到的內容

「Outputs」分頁一次顯示一個測試案例：
- **Prompt**：給定的任務
- **Output**：skill 產出的檔案，盡可能在行內渲染
- **Previous Output**（迭代 2+）：折疊的上次迭代輸出
- **Formal Grades**（若有執行評分）：折疊的斷言通過/失敗
- **Feedback**：自動儲存的文字框
- **Previous Feedback**（迭代 2+）：上次的評論，顯示在文字框下方

「Benchmark」分頁顯示統計摘要：各設定的通過率、計時和 token 使用量，以及各 eval 的細分和分析師觀察。

導航透過上/下按鈕或方向鍵進行。完成後，他們點擊「Submit All Reviews」，將所有意見回饋儲存至 `feedback.json`。

### 步驟五：讀取意見回饋

當使用者告訴您已完成時，讀取 `feedback.json`：

```json
{
  "reviews": [
    {"run_id": "eval-0-with_skill", "feedback": "the chart is missing axis labels", "timestamp": "..."},
    {"run_id": "eval-1-with_skill", "feedback": "", "timestamp": "..."},
    {"run_id": "eval-2-with_skill", "feedback": "perfect, love this", "timestamp": "..."}
  ],
  "status": "complete"
}
```

空白的意見回饋表示使用者認為沒問題。把改進重點放在使用者有具體抱怨的測試案例上。

當您完成時，終止檢視器伺服器：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 改進 skill

這是循環的核心。您已執行測試案例，使用者已審閱結果，現在您需要根據他們的意見回饋讓 skill 更好。

### 如何思考改進

1. **從意見回饋中泛化。** 這裡發生的大事是：我們正試圖建立可以使用百萬次（也許真的，也許更多，誰知道呢）的 skill，跨越許多不同的 prompt。您和使用者在這裡反覆迭代只有幾個範例，因為這有助於移動更快。使用者對這些範例瞭若指掌，很容易評估新的輸出。但若您和使用者共同開發的 skill 只對這些範例有效，那它就毫無用處。與其進行細緻繁瑣的過度擬合修改，或壓倒性的限制性 MUST，若有頑固的問題，您可以嘗試擴展並使用不同的比喻，或建議不同的工作模式。嘗試成本相對低廉，也許您會找到絕妙的方案。

2. **保持 prompt 精簡。** 移除沒有發揮作用的部分。確保讀取執行記錄，而不只是最終輸出——若看起來 skill 讓模型浪費了大量時間在無益的事情上，您可以嘗試去掉那些讓它這樣做的 skill 部分，看看會發生什麼。

3. **解釋原因。** 努力解釋您要求模型做某事背後的**原因**。現今的 LLM 很**聰明**。它們有良好的心智理論，當有好的框架時，可以超越死板的指令，讓事情真正發生。即使使用者的意見回饋是簡短或沮喪的，也要試著真正理解任務，以及使用者寫了什麼、為何這樣寫，然後將這種理解傳遞到指令中。若發現自己在用全大寫的 ALWAYS 或 NEVER，或使用超級僵硬的結構，那是一個黃色警告——若可能，重新表述並解釋您要求的事情為何重要。那是一種更人性化、更有力、更有效的方法。

4. **尋找跨測試案例的重複工作。** 讀取測試執行的執行記錄，注意是否所有 sub-agent 都獨立撰寫了類似的輔助指令碼，或對某事採取了相同的多步驟方法。若所有 3 個測試案例都導致 sub-agent 撰寫 `create_docx.py` 或 `build_chart.py`，這是一個強烈的信號，表明 skill 應該附帶這個指令碼。寫一次，放在 `scripts/` 中，並告訴 skill 使用它。這樣可以節省每次未來調用重新造輪子的時間。

這個任務相當重要（我們試圖在這裡每年創造數十億美元的經濟價值！），而您的思考時間不是瓶頸；慢慢來，真正思索。我建議撰寫草稿修訂版，然後重新審視它並進行改進。真正試著進入使用者的思維，理解他們想要和需要什麼。

### 迭代循環

改進 skill 後：

1. 將改進應用至 skill
2. 將所有測試案例重新執行至新的 `iteration-<N+1>/` 目錄，包括基線執行。若您正在建立新 skill，基線始終是 `without_skill`（無 skill）——這在各迭代間保持不變。若您正在改進現有 skill，請根據判斷決定什麼作為基線最合理：使用者帶來的原始版本，還是上一個迭代。
3. 使用 `--previous-workspace` 指向上一個迭代來啟動審閱器
4. 等待使用者審閱並告訴您已完成
5. 讀取新的意見回饋，再次改進，重複

持續直到：
- 使用者說他們滿意
- 意見回饋全為空（一切看起來都好）
- 您沒有在做出有意義的進展

---

## 進階：盲測比較

對於希望更嚴格比較 skill 兩個版本的情況（例如，使用者問「新版本真的更好嗎？」），有一個盲測比較系統。詳情請讀取 `agents/comparator.md` 和 `agents/analyzer.md`。基本想法是：將兩個輸出交給一個獨立 agent，而不告訴它哪個是哪個，讓它判斷品質。然後分析為什麼贏家會贏。

這是可選的，需要 sub-agent，大多數使用者不需要它。人工審閱循環通常就足夠了。

---

## Description 最佳化

SKILL.md frontmatter 中的 description 欄位是決定 Claude 是否調用 skill 的主要機制。建立或改進 skill 後，提議最佳化 description 以提升觸發準確率。

### 步驟一：生成觸發 eval 查詢

建立 20 個 eval 查詢——混合應觸發和不應觸發的。儲存為 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

查詢必須是真實的，是 Claude Code 或 Claude.ai 使用者實際會輸入的。不是抽象的請求，而是具體且詳細的請求。例如，檔案路徑、關於使用者工作或情況的個人脈絡、欄位名稱和值、公司名稱、URL。一點點背景故事。有些可能是小寫或包含縮寫、錯字或非正式用語。混合使用不同長度，並聚焦於邊緣情況，而不是讓它們顯而易見（使用者將有機會確認）。

不好的：`"Format this data"`、`"Extract text from PDF"`、`"Create a chart"`

好的：`"ok so my boss just sent me this xlsx file (its in my downloads, called something like 'Q4 sales final FINAL v2.xlsx') and she wants me to add a column that shows the profit margin as a percentage. The revenue is in column C and costs are in column D i think"`

對於**應觸發**的查詢（8-10 個），考慮涵蓋範圍。您需要同一意圖的不同措辭——有些正式，有些隨意。包括使用者沒有明確說出 skill 名稱或檔案類型但明顯需要它的案例。加入一些不常見的使用案例，以及此 skill 與另一個競爭但應獲勝的案例。

對於**不應觸發**的查詢（8-10 個），最有價值的是「幾乎命中」——與 skill 共享關鍵字或概念，但實際需要不同內容的查詢。考慮相鄰領域、天真的關鍵字匹配會觸發但實際上不應該的模糊措辭，以及觸及 skill 功能但在另一個工具更合適的情境下的案例。

要避免的關鍵事：不要讓不應觸發的查詢顯然不相關。「寫一個費波那契函式」作為 PDF skill 的負面測試太容易了——它沒有測試任何東西。負面案例應該是真正難以區分的。

### 步驟二：與使用者審閱

使用 HTML 樣板向使用者展示 eval 集合以供審閱：

1. 讀取 `assets/eval_review.html` 中的樣板
2. 替換佔位符：
   - `__EVAL_DATA_PLACEHOLDER__` → eval 項目的 JSON 陣列（不帶引號——它是一個 JS 變數賦值）
   - `__SKILL_NAME_PLACEHOLDER__` → skill 的名稱
   - `__SKILL_DESCRIPTION_PLACEHOLDER__` → skill 目前的 description
3. 寫入暫存檔（例如 `/tmp/eval_review_<skill-name>.html`）並開啟：`open /tmp/eval_review_<skill-name>.html`
4. 使用者可以編輯查詢、切換應觸發、新增/移除條目，然後點擊「Export Eval Set」
5. 檔案下載至 `~/Downloads/eval_set.json`——若有多個版本（例如 `eval_set (1).json`），請檢查 Downloads 資料夾中最新的版本

這個步驟很重要——糟糕的 eval 查詢會導致糟糕的 description。

### 步驟三：執行最佳化循環

告知使用者：「這需要一些時間——我將在背景執行最佳化循環，並定期查看進度。」

將 eval 集合儲存至工作區，然後在背景執行：

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

使用您系統 prompt 中的模型 ID（為目前工作階段提供動力的模型），這樣觸發測試就能與使用者實際體驗相符。

執行期間，定期追蹤輸出，向使用者更新當前迭代和分數情況。

這個功能會自動處理完整的最佳化循環。它將 eval 集合分成 60% 訓練集和 40% 保留測試集，評估目前的 description（每個查詢執行 3 次以獲得可靠的觸發率），然後呼叫 Claude 根據失敗的案例提出改進建議。它對新舊 description 在訓練集和測試集上重新評估，最多迭代 5 次。完成後，它會在瀏覽器中開啟一個 HTML 報告，顯示各迭代的結果，並返回帶有 `best_description` 的 JSON——依測試分數（而非訓練分數）選擇，以避免過度擬合。

### Skill 觸發的運作方式

了解觸發機制有助於設計更好的 eval 查詢。skill 以其 name + description 出現在 Claude 的 `available_skills` 清單中，Claude 根據該 description 決定是否查閱 skill。重要的是：Claude 只對它自己無法輕易處理的任務查閱 skill——簡單的一步查詢，如「讀取這個 PDF」，即使 description 完全匹配，也可能不會觸發 skill，因為 Claude 可以直接用基本工具處理。複雜的、多步驟的或專業的查詢，在 description 匹配時，可靠地觸發 skill。

這意味著您的 eval 查詢應該足夠實質，讓 Claude 真的能從查閱 skill 中受益。簡單的查詢如「讀取檔案 X」是差的測試案例——無論 description 品質如何，它們都不會觸發 skill。

### 步驟四：應用結果

從 JSON 輸出中取出 `best_description`，並更新 skill 的 SKILL.md frontmatter。向使用者展示前後對比，並回報分數。

---

### 打包並呈現（僅在 `present_files` 工具可用時）

確認您是否有 `present_files` 工具的存取權限。若沒有，跳過這個步驟。若有，打包 skill 並向使用者呈現 .skill 檔案：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包後，將結果 `.skill` 檔案路徑告知使用者，以便他們安裝。

---

## Claude.ai 特定指令

在 Claude.ai 中，核心工作流程相同（草稿 → 測試 → 審閱 → 改進 → 重複），但由於 Claude.ai 沒有 sub-agent，某些機制會改變。以下是需要調整的地方：

**執行測試案例**：沒有 sub-agent 意味著無法平行執行。對每個測試案例，讀取 skill 的 SKILL.md，然後遵循其指令自己完成測試 prompt。一次完成一個。這比獨立 sub-agent 的嚴格性低（您撰寫了 skill，也在執行它，所以您有完整的脈絡），但這是一個有用的健全性檢查——而人工審閱步驟可以彌補。跳過基線執行——只使用 skill 完成請求的任務即可。

**審閱結果**：若無法開啟瀏覽器（例如 Claude.ai 的 VM 沒有顯示器，或您在遠端伺服器上），完全跳過瀏覽器審閱器。取而代之，直接在對話中呈現結果。對每個測試案例，顯示 prompt 和輸出。若輸出是使用者需要查看的檔案（如 .docx 或 .xlsx），將其儲存至檔案系統並告訴他們在哪裡，以便他們下載和檢視。以內聯方式請求意見回饋：「這樣看起來如何？有什麼想要改變的嗎？」

**基準測試**：跳過量性基準測試——它依賴基線比較，而沒有 sub-agent 時這沒有意義。聚焦於來自使用者的質性意見回饋。

**迭代循環**：和之前一樣——改進 skill、重新執行測試案例、請求意見回饋——只是中間沒有瀏覽器審閱器。若您有檔案系統，仍然可以將結果組織到迭代目錄中。

**Description 最佳化**：這部分需要 `claude` CLI 工具（具體是 `claude -p`），僅在 Claude Code 中可用。若您在 Claude.ai 上，跳過它。

**盲測比較**：需要 sub-agent。跳過它。

**打包**：`package_skill.py` 指令碼在任何有 Python 和檔案系統的地方都能運作。在 Claude.ai 上，您可以執行它，使用者可以下載結果的 `.skill` 檔案。

**更新現有 skill**：使用者可能要求您更新現有 skill，而不是建立新的。在這種情況下：
- **保留原始名稱。** 記下 skill 的目錄名稱和 `name` frontmatter 欄位——保持不變。例如，若已安裝的 skill 是 `research-helper`，輸出 `research-helper.skill`（而非 `research-helper-v2`）。
- **在編輯前複製到可寫入的位置。** 已安裝的 skill 路徑可能是唯讀的。複製到 `/tmp/skill-name/`，在那裡編輯，並從副本打包。
- **若手動打包，先在 `/tmp/` 中暫存**，然後複製到輸出目錄——直接寫入可能因權限問題而失敗。

---

## Cowork 特定指令

若您在 Cowork 中，主要需要知道的是：

- 您有 sub-agent，所以主要工作流程（平行生成測試案例、執行基線、評分等）都能運作。（但是，若您遇到嚴重的超時問題，可以串行而非平行執行測試 prompt。）
- 您沒有瀏覽器或顯示器，所以在生成 eval 檢視器時，請使用 `--static <output_path>` 寫入獨立的 HTML 檔案，而不是啟動伺服器。然後提供一個連結讓使用者點擊，在瀏覽器中開啟 HTML。
- 由於某種原因，Cowork 設定似乎讓 Claude 在執行測試後不願意生成 eval 檢視器，所以再次強調：無論您是在 Cowork 還是在 Claude Code 中，執行測試後，您應該始終在自己評估輸入並嘗試自行修正之前，先使用 `generate_review.py` 生成 eval 檢視器供人工審閱。在此我要全大寫說：在自行評估輸入之前，先生成 eval 檢視器。您希望盡快讓它們呈現在人類面前！
- 意見回饋的運作方式不同：由於沒有執行中的伺服器，檢視器的「Submit All Reviews」按鈕將下載 `feedback.json` 作為檔案。然後您可以從那裡讀取它（可能需要先請求存取權限）。
- 打包能運作——`package_skill.py` 只需要 Python 和檔案系統。
- Description 最佳化（`run_loop.py` / `run_eval.py`）在 Cowork 中應該沒問題，因為它透過 subprocess 使用 `claude -p`，而不是瀏覽器，但請將其留到您已完全完成 skill，且使用者同意它狀態良好後再進行。
- **更新現有 skill**：使用者可能要求您更新現有 skill，而不是建立新的。請遵循上方 claude.ai 章節中的更新指引。

---

## 參考檔案

agents/ 目錄包含專用 sub-agent 的指令。需要生成相關 sub-agent 時讀取它們。

- `agents/grader.md` — 如何對照輸出評估斷言
- `agents/comparator.md` — 如何對兩個輸出進行盲測 A/B 比較
- `agents/analyzer.md` — 如何分析一個版本勝過另一個版本的原因

references/ 目錄有額外的文件：
- `references/schemas.md` — evals.json、grading.json 等的 JSON 結構

---

再次重申核心循環以加深印象：

- 弄清楚 skill 的主題
- 草擬或編輯 skill
- 在測試 prompt 上執行「具備 skill 的 claude」
- 與使用者一起評估輸出：
  - 建立 benchmark.json 並執行 `eval-viewer/generate_review.py` 幫助使用者審閱它們
  - 執行量性 eval
- 重複直到您和使用者滿意
- 打包最終 skill 並回傳給使用者。

若您有 TodoList 之類的東西，請加入步驟，以確保不會忘記。若您在 Cowork 中，請特別在 TodoList 中加入「建立 evals JSON 並執行 `eval-viewer/generate_review.py`，讓人工審閱測試案例」，以確保它發生。

祝您好運！