---
source_file: evaluation.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 8c99479f8a2d22a636c38e274537aac3610879e26f34e0709825077c4576f427
translated_at: 2026-08-22
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`evaluation.md`](evaluation.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
<!-- /translation-header -->

# MCP 伺服器評估指南

## 概述

本文件提供關於建立 MCP 伺服器全面評估的指引。評估測試 LLM 是否能有效地使用您的 MCP 伺服器，僅使用所提供的工具回答真實的、複雜的問題。

---

## 快速參考

### 評估需求
- 建立 10 個人類可讀的問題
- 問題必須是唯讀、獨立、非破壞性
- 每個問題需要多次工具呼叫（可能數十次）
- 答案必須是單一的、可驗證的值
- 答案必須穩定（不會隨時間改變）

### 輸出格式
```xml
<evaluation>
   <qa_pair>
      <question>Your question here</question>
      <answer>Single verifiable answer</answer>
   </qa_pair>
</evaluation>
```

---

## 評估的目的

MCP 伺服器的品質衡量標準，**不是**伺服器實作工具的完善程度，而是這些實作（輸入/輸出綱要、文件字串/說明、功能）能多有效地讓 LLM 在沒有其他情境、且**只能**存取 MCP 伺服器的情況下，回答真實且困難的問題。

## 評估概覽

建立 10 個人類可讀的問題，回答時只需要唯讀、獨立、非破壞性且冪等的操作。每個問題應該：
- 真實
- 清晰且簡潔
- 明確無歧義
- 複雜，可能需要數十次工具呼叫或步驟
- 可以用您事先識別的單一、可驗證的值來回答

## 問題指引

### 核心需求

1. **問題必須獨立**
   - 每個問題不應依賴其他問題的答案
   - 不應假設處理另一個問題時已執行過寫入操作

2. **問題只能需要非破壞性且冪等的工具使用**
   - 不應指示或要求修改狀態才能得到正確答案

3. **問題必須真實、清晰、簡潔且複雜**
   - 必須要求另一個 LLM 使用多個（可能數十個）工具或步驟才能回答

### 複雜度與深度

4. **問題必須需要深度探索**
   - 考慮需要多個子問題和連續工具呼叫的多跳問題
   - 每一步應受益於前面問題中找到的資訊

5. **問題可能需要大量分頁**
   - 可能需要翻閱多頁結果
   - 可能需要查詢舊資料（1-2 年前的）才能找到利基資訊
   - 問題必須具有難度

6. **問題必須需要深度理解**
   - 而非表面知識
   - 可以將複雜概念設計為需要證據的是非題
   - 可以使用選擇題格式，讓 LLM 必須搜尋不同的假設

7. **問題不能用直接的關鍵字搜尋解決**
   - 不要包含目標內容中的特定關鍵字
   - 使用同義詞、相關概念或改述
   - 需要多次搜尋、分析多個相關項目、提取情境，然後推導答案

### 工具測試

8. **問題應對工具回傳值進行壓力測試**
   - 可能使工具回傳大型 JSON 物件或清單，讓 LLM 應接不暇
   - 應需要理解多種資料型態：
     - ID 和名稱
     - 時間戳記和日期時間（月、日、年、秒）
     - 檔案 ID、名稱、副檔名和 MIME 類型
     - URL、GID 等
   - 應探查工具回傳所有有用資料形式的能力

9. **問題應大多反映真實的人類使用案例**
   - 是 LLM 輔助的**人類**會關心的資訊擷取任務

10. **問題可能需要數十次工具呼叫**
    - 這對情境有限的 LLM 是一種挑戰
    - 鼓勵 MCP 伺服器工具減少回傳的資訊

11. **包含有歧義的問題**
    - 可能有歧義，或需要對呼叫哪些工具做出困難的決定
    - 迫使 LLM 可能犯錯或誤解
    - 確保儘管有歧義，仍然有**單一的可驗證答案**

### 穩定性

12. **問題必須設計成答案不會改變**
    - 不要詢問依賴「當前狀態」的動態問題
    - 例如，不要計算：
      - 貼文的反應數量
      - 討論串的回覆數量
      - 頻道的成員數量

13. **不要讓 MCP 伺服器限制您建立的問題類型**
    - 建立具有挑戰性和複雜性的問題
    - 有些可能無法用現有的 MCP 伺服器工具解決
    - 問題可能需要特定的輸出格式（datetime vs. epoch time、JSON vs. MARKDOWN）
    - 問題可能需要數十次工具呼叫才能完成

## 答案指引

### 驗證

1. **答案必須可透過直接字串比較驗證**
   - 若答案可以用多種格式改寫，請在問題中清楚指定輸出格式
   - 例如：「使用 YYYY/MM/DD。」、「回答 True 或 False。」、「回答 A、B、C 或 D，除此之外什麼都不要。」
   - 答案應是單一的可驗證值，例如：
     - 使用者 ID、使用者名稱、顯示名稱、名字、姓氏
     - 頻道 ID、頻道名稱
     - 訊息 ID、字串
     - URL、標題
     - 數值
     - 時間戳記、日期時間
     - 布林值（用於是非題）
     - 電子郵件地址、電話號碼
     - 檔案 ID、檔案名稱、副檔名
     - 選擇題答案
   - 答案不需要特殊格式或複雜的結構化輸出
   - 答案將使用**直接字串比較**進行驗證

### 可讀性

2. **答案一般應優先採用人類可讀格式**
   - 例如：名稱、名字、姓氏、日期時間、檔案名稱、訊息字串、URL、是/否、真/假、a/b/c/d
   - 而非不透明的 ID（雖然 ID 也是可接受的）
   - **絕大多數**答案應是人類可讀的

### 穩定性

3. **答案必須穩定/靜止**
   - 查看舊內容（例如已結束的對話、已發布的專案、已回答的問題）
   - 基於「已關閉」的概念建立問題，這些概念始終會回傳相同的答案
   - 問題可以要求考慮固定的時間視窗，以隔離不穩定的答案
   - 依賴不太可能改變的情境
   - 範例：若要找一篇論文的名稱，要**足夠具體**，使答案不會與後來發表的論文混淆

4. **答案必須清晰且無歧義**
   - 問題必須設計成有單一的、清晰的答案
   - 答案可以透過使用 MCP 伺服器工具推導出來

### 多樣性

5. **答案必須多樣**
   - 答案應是各種型態和格式的單一可驗證值
   - 使用者概念：使用者 ID、使用者名稱、顯示名稱、名字、姓氏、電子郵件地址、電話號碼
   - 頻道概念：頻道 ID、頻道名稱、頻道主題
   - 訊息概念：訊息 ID、訊息字串、時間戳記、月、日、年

6. **答案不得是複雜結構**
   - 不是值的清單
   - 不是複雜物件
   - 不是 ID 或字串的清單
   - 不是自然語言文字
   - 除非答案可以用**直接字串比較**直接驗證
   - 且可以被真實地重現
   - LLM 不太可能以任何其他順序或格式回傳相同的清單

## 評估流程

### 步驟一：文件檢視

閱讀目標 API 的文件以了解：
- 可用的端點和功能
- 若存在歧義，從網路擷取額外資訊
- 盡可能**平行化**此步驟
- 確保每個 sub-agent 只從檔案系統或網路上檢視文件

### 步驟二：工具檢視

列出 MCP 伺服器中可用的工具：
- 直接檢視 MCP 伺服器
- 了解輸入/輸出綱要、文件字串和說明
- 在此階段**不要**自行呼叫工具

### 步驟三：建立理解

重複步驟一和二直到有充分理解：
- 多次迭代
- 思考您想要建立的任務類型
- 精煉您的理解
- 在任何階段都**不應**閱讀 MCP 伺服器實作本身的程式碼
- 使用您的直覺和理解來建立合理的、真實的但**非常具有挑戰性**的任務

### 步驟四：唯讀內容檢視

在了解 API 和工具之後，**使用** MCP 伺服器工具：
- 只使用**唯讀**和**非破壞性**操作來檢視內容
- 目標：識別特定內容（例如使用者、頻道、訊息、專案、任務）以建立真實的問題
- **不應**呼叫任何修改狀態的工具
- **不應**閱讀 MCP 伺服器實作本身的程式碼
- 用個別的 sub-agent 進行獨立探索以平行化此步驟
- 確保每個 sub-agent 只執行唯讀、非破壞性且冪等的操作
- **注意**：某些工具可能回傳**大量資料**，導致您耗盡情境
- 進行**增量、小型且有針對性**的工具呼叫以進行探索
- 在所有工具呼叫請求中，使用 `limit` 參數限制結果（< 10）
- 使用分頁

### 步驟五：任務生成

檢視內容後，建立 10 個人類可讀的問題：
- LLM 應能使用 MCP 伺服器回答這些問題
- 遵循上述所有問題和答案指引

## 輸出格式

每個 QA 配對由一個問題和一個答案組成。輸出應是具有此結構的 XML 檔案：

```xml
<evaluation>
   <qa_pair>
      <question>Find the project created in Q2 2024 with the highest number of completed tasks. What is the project name?</question>
      <answer>Website Redesign</answer>
   </qa_pair>
   <qa_pair>
      <question>Search for issues labeled as "bug" that were closed in March 2024. Which user closed the most issues? Provide their username.</question>
      <answer>sarah_dev</answer>
   </qa_pair>
   <qa_pair>
      <question>Look for pull requests that modified files in the /api directory and were merged between January 1 and January 31, 2024. How many different contributors worked on these PRs?</question>
      <answer>7</answer>
   </qa_pair>
   <qa_pair>
      <question>Find the repository with the most stars that was created before 2023. What is the repository name?</question>
      <answer>data-pipeline</answer>
   </qa_pair>
</evaluation>
```

## 評估範例

### 好問題

**範例一：需要深度探索的多跳問題（GitHub MCP）**
```xml
<qa_pair>
   <question>Find the repository that was archived in Q3 2023 and had previously been the most forked project in the organization. What was the primary programming language used in that repository?</question>
   <answer>Python</answer>
</qa_pair>
```

這個問題很好，因為：
- 需要多次搜尋才能找到已封存的存放庫
- 需要識別哪個在封存前被 fork 最多次
- 需要檢視存放庫細節以取得語言資訊
- 答案是簡單的、可驗證的值
- 基於不會改變的歷史（已關閉）資料

**範例二：需要在無關鍵字匹配下理解情境（專案管理 MCP）**
```xml
<qa_pair>
   <question>Locate the initiative focused on improving customer onboarding that was completed in late 2023. The project lead created a retrospective document after completion. What was the lead's role title at that time?</question>
   <answer>Product Manager</answer>
</qa_pair>
```

這個問題很好，因為：
- 不使用特定的專案名稱（「專注於改善客戶入門的計畫」）
- 需要從特定時間框架找到已完成的專案
- 需要識別專案負責人及其職位
- 需要從回顧文件中理解情境
- 答案是人類可讀且穩定的
- 基於已完成的工作（不會改變）

**範例三：需要多個步驟的複雜彙總（問題追蹤 MCP）**
```xml
<qa_pair>
   <question>Among all bugs reported in January 2024 that were marked as critical priority, which assignee resolved the highest percentage of their assigned bugs within 48 hours? Provide the assignee's username.</question>
   <answer>alex_eng</answer>
</qa_pair>
```

這個問題很好，因為：
- 需要按日期、優先級和狀態篩選錯誤
- 需要按負責人分組並計算解決率
- 需要理解時間戳記以確定 48 小時視窗
- 測試分頁（可能需要處理許多錯誤）
- 答案是單一的使用者名稱
- 基於特定時間段的歷史資料

**範例四：需要跨多種資料類型的綜合（CRM MCP）**
```xml
<qa_pair>
   <question>Find the account that upgraded from the Starter to Enterprise plan in Q4 2023 and had the highest annual contract value. What industry does this account operate in?</question>
   <answer>Healthcare</answer>
</qa_pair>
```

這個問題很好，因為：
- 需要理解訂閱等級的變化
- 需要識別特定時間框架內的升級事件
- 需要比較合約價值
- 必須存取帳戶行業資訊
- 答案簡單且可驗證
- 基於已完成的歷史交易

### 差問題

**範例一：答案隨時間改變**
```xml
<qa_pair>
   <question>How many open issues are currently assigned to the engineering team?</question>
   <answer>47</answer>
</qa_pair>
```

這個問題差，因為：
- 隨著問題被建立、關閉或重新指派，答案會改變
- 不基於穩定/靜止的資料
- 依賴動態的「當前狀態」

**範例二：用關鍵字搜尋太容易**
```xml
<qa_pair>
   <question>Find the pull request with title "Add authentication feature" and tell me who created it.</question>
   <answer>developer123</answer>
</qa_pair>
```

這個問題差，因為：
- 可以用精確標題的直接關鍵字搜尋解決
- 不需要深度探索或理解
- 不需要任何綜合或分析

**範例三：答案格式有歧義**
```xml
<qa_pair>
   <question>List all the repositories that have Python as their primary language.</question>
   <answer>repo1, repo2, repo3, data-pipeline, ml-tools</answer>
</qa_pair>
```

這個問題差，因為：
- 答案是可以以任何順序回傳的清單
- 難以用直接字串比較驗證
- LLM 可能以不同方式格式化（JSON 陣列、逗號分隔、換行分隔）
- 最好詢問特定的彙總（計數）或最高級（星星最多的）

## 驗證流程

建立評估後：

1. **檢視 XML 檔案**以了解綱要
2. **載入每個任務指令**，並平行地使用 MCP 伺服器和工具，嘗試**自己**解決任務以識別正確答案
3. **標記任何需要**寫入或破壞性操作的操作
4. **彙總所有正確答案**，並替換文件中任何不正確的答案
5. **移除任何 `<qa_pair>`**，其需要寫入或破壞性操作

記住要平行化解決任務以避免耗盡情境，然後彙總所有答案並在最後對檔案進行修改。

## 建立高品質評估的訣竅

1. **認真思考並提前規劃**，然後再生成任務
2. **把握機會時平行化**，以加快流程並管理情境
3. **聚焦於真實使用案例**，是人類實際想要完成的任務
4. **建立具有挑戰性的問題**，測試 MCP 伺服器能力的極限
5. **確保穩定性**，使用歷史資料和已關閉的概念
6. **驗證答案**，使用 MCP 伺服器工具自行解答問題
7. **迭代並精煉**，根據流程中學到的東西

---

# 執行評估

建立評估檔案後，您可以使用提供的評估測試框架來測試您的 MCP 伺服器。

## 設定

1. **安裝相依套件**

   ```bash
   pip install -r scripts/requirements.txt
   ```

   或手動安裝：
   ```bash
   pip install anthropic mcp
   ```

2. **設定 API 金鑰**

   ```bash
   export ANTHROPIC_API_KEY=your_api_key_here
   ```

## 評估檔案格式

評估檔案使用帶有 `<qa_pair>` 元素的 XML 格式：

```xml
<evaluation>
   <qa_pair>
      <question>Find the project created in Q2 2024 with the highest number of completed tasks. What is the project name?</question>
      <answer>Website Redesign</answer>
   </qa_pair>
   <qa_pair>
      <question>Search for issues labeled as "bug" that were closed in March 2024. Which user closed the most issues? Provide their username.</question>
      <answer>sarah_dev</answer>
   </qa_pair>
</evaluation>
```

## 執行評估

評估指令碼（`scripts/evaluation.py`）支援三種傳輸類型：

**重要：**
- **stdio 傳輸**：評估指令碼自動為您啟動和管理 MCP 伺服器行程。不要手動執行伺服器。
- **sse/http 傳輸**：在執行評估之前，您必須單獨啟動 MCP 伺服器。指令碼會連接到已在指定 URL 執行的伺服器。

### 1. 本機 STDIO 伺服器

對於本機執行的 MCP 伺服器（指令碼自動啟動伺服器）：

```bash
python scripts/evaluation.py \
  -t stdio \
  -c python \
  -a my_mcp_server.py \
  evaluation.xml
```

帶有環境變數：
```bash
python scripts/evaluation.py \
  -t stdio \
  -c python \
  -a my_mcp_server.py \
  -e API_KEY=abc123 \
  -e DEBUG=true \
  evaluation.xml
```

### 2. Server-Sent Events（SSE）

對於基於 SSE 的 MCP 伺服器（您必須先啟動伺服器）：

```bash
python scripts/evaluation.py \
  -t sse \
  -u https://example.com/mcp \
  -H "Authorization: Bearer token123" \
  -H "X-Custom-Header: value" \
  evaluation.xml
```

### 3. HTTP（Streamable HTTP）

對於基於 HTTP 的 MCP 伺服器（您必須先啟動伺服器）：

```bash
python scripts/evaluation.py \
  -t http \
  -u https://example.com/mcp \
  -H "Authorization: Bearer token123" \
  evaluation.xml
```

## 命令列選項

```
usage: evaluation.py [-h] [-t {stdio,sse,http}] [-m MODEL] [-c COMMAND]
                     [-a ARGS [ARGS ...]] [-e ENV [ENV ...]] [-u URL]
                     [-H HEADERS [HEADERS ...]] [-o OUTPUT]
                     eval_file

positional arguments:
  eval_file             Path to evaluation XML file

optional arguments:
  -h, --help            Show help message
  -t, --transport       Transport type: stdio, sse, or http (default: stdio)
  -m, --model           Claude model to use (default: claude-3-7-sonnet-20250219)
  -o, --output          Output file for report (default: print to stdout)

stdio options:
  -c, --command         Command to run MCP server (e.g., python, node)
  -a, --args            Arguments for the command (e.g., server.py)
  -e, --env             Environment variables in KEY=VALUE format

sse/http options:
  -u, --url             MCP server URL
  -H, --header          HTTP headers in 'Key: Value' format
```

## 輸出

評估指令碼會產生詳細的報告，包括：

- **彙總統計**：
  - 準確率（正確/總計）
  - 平均任務時間
  - 每個任務的平均工具呼叫次數
  - 工具呼叫總數

- **每個任務的結果**：
  - Prompt 和預期回應
  - agent 的實際回應
  - 答案是否正確（✅/❌）
  - 時間和工具呼叫細節
  - agent 對其方法的摘要
  - agent 對工具的意見回饋

### 將報告儲存至檔案

```bash
python scripts/evaluation.py \
  -t stdio \
  -c python \
  -a my_server.py \
  -o evaluation_report.md \
  evaluation.xml
```

## 完整範例工作流程

以下是建立和執行評估的完整範例：

1. **建立您的評估檔案**（`my_evaluation.xml`）：

```xml
<evaluation>
   <qa_pair>
      <question>Find the user who created the most issues in January 2024. What is their username?</question>
      <answer>alice_developer</answer>
   </qa_pair>
   <qa_pair>
      <question>Among all pull requests merged in Q1 2024, which repository had the highest number? Provide the repository name.</question>
      <answer>backend-api</answer>
   </qa_pair>
   <qa_pair>
      <question>Find the project that was completed in December 2023 and had the longest duration from start to finish. How many days did it take?</question>
      <answer>127</answer>
   </qa_pair>
</evaluation>
```

2. **安裝相依套件**：

```bash
pip install -r scripts/requirements.txt
export ANTHROPIC_API_KEY=your_api_key
```

3. **執行評估**：

```bash
python scripts/evaluation.py \
  -t stdio \
  -c python \
  -a github_mcp_server.py \
  -e GITHUB_TOKEN=ghp_xxx \
  -o github_eval_report.md \
  my_evaluation.xml
```

4. **審閱 `github_eval_report.md` 中的報告**，以：
   - 查看哪些問題通過/失敗
   - 閱讀 agent 對您的工具的意見回饋
   - 識別需要改進的領域
   - 迭代您的 MCP 伺服器設計

## 疑難排解

### 連線錯誤

若出現連線錯誤：
- **STDIO**：驗證命令和參數是否正確
- **SSE/HTTP**：檢查 URL 是否可存取且標頭是否正確
- 確保所有必要的 API 金鑰已在環境變數或標頭中設定

### 準確率低

若許多評估失敗：
- 審閱每個任務的 agent 意見回饋
- 檢查工具說明是否清晰且全面
- 驗證輸入參數是否有充分的文件說明
- 考慮工具回傳的資料是否過多或過少
- 確保錯誤訊息是可操作的

### 逾時問題

若任務超時：
- 使用更強大的模型（例如 `claude-3-7-sonnet-20250219`）
- 檢查工具是否回傳過多資料
- 驗證分頁是否正常運作
- 考慮簡化複雜的問題