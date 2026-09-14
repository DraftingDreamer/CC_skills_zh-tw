---
source_file: SKILL.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 0f4592dcb53cf2b5d6b7febee6b4152018b565551a1c29e3c612f57b218ab295
translated_at: 2026-08-22
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`SKILL.md`](SKILL.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
>
> **原檔 YAML frontmatter**
> - `name`: `mcp-builder`
> - `description`（中譯）：建立高品質 MCP（Model Context Protocol）伺服器的指南，讓 LLM 能透過精心設計的工具與外部服務互動。當使用 Python（FastMCP）或 Node/TypeScript（MCP SDK）建構 MCP 伺服器以整合外部 API 或服務時，請使用本 skill。
> - `license`: Complete terms in LICENSE.txt
<!-- /translation-header -->

# MCP 伺服器開發指南

## 概述

建立 MCP（Model Context Protocol）伺服器，讓 LLM 能透過精心設計的工具與外部服務互動。MCP 伺服器的品質，衡量於它能多有效地讓 LLM 完成真實世界的任務。

---

# 流程

## 🚀 高層級工作流程

建立高品質 MCP 伺服器涉及四個主要階段：

### 第一階段：深度研究與規劃

#### 1.1 了解現代 MCP 設計

**API 涵蓋範圍 vs. 工作流程工具：**
在全面的 API 端點涵蓋範圍與專用工作流程工具之間取得平衡。工作流程工具對特定任務可能更方便，而全面的涵蓋範圍則讓 agent 有彈性組合操作。效能因用戶端而異——某些用戶端受益於結合基本工具的程式碼執行，而另一些則使用更高層次的工作流程效果更好。在不確定的情況下，優先考慮全面的 API 涵蓋範圍。

**工具命名與可發現性：**
清晰、描述性的工具名稱能幫助 agent 快速找到正確的工具。使用一致的前綴（例如 `github_create_issue`、`github_list_repos`）和以動作為導向的命名方式。

**情境管理：**
agent 受益於簡潔的工具說明，以及篩選/分頁結果的能力。設計能回傳聚焦、相關資料的工具。某些用戶端支援程式碼執行，能幫助 agent 有效地篩選和處理資料。

**可操作的錯誤訊息：**
錯誤訊息應透過具體建議和後續步驟，引導 agent 找到解決方案。

#### 1.2 研讀 MCP 協定文件

**瀏覽 MCP 規格：**

從 sitemap 開始找到相關頁面：`https://modelcontextprotocol.io/sitemap.xml`

然後以 `.md` 後綴取得特定頁面的 markdown 格式（例如 `https://modelcontextprotocol.io/specification/draft.md`）。

要審閱的關鍵頁面：
- 規格概覽與架構
- 傳輸機制（streamable HTTP、stdio）
- 工具、資源和 prompt 的定義

#### 1.3 研讀框架文件

**推薦技術堆疊：**
- **語言**：TypeScript（高品質 SDK 支援，在許多執行環境中具有良好的相容性，例如 MCPB。此外 AI 模型擅長生成 TypeScript 程式碼，受益於其廣泛使用、靜態型別和良好的 linting 工具）
- **傳輸方式**：遠端伺服器使用 Streamable HTTP，採用無狀態 JSON（比有狀態 session 和串流回應更易於擴展和維護）。本機伺服器使用 stdio。

**載入框架文件：**

- **MCP 最佳實踐**：[📋 查看最佳實踐](./reference/mcp_best_practices.md) - 核心指引

**TypeScript（推薦）：**
- **TypeScript SDK**：使用 WebFetch 載入 `https://raw.githubusercontent.com/modelcontextprotocol/typescript-sdk/main/README.md`
- [⚡ TypeScript 指南](./reference/node_mcp_server.md) - TypeScript 模式與範例

**Python：**
- **Python SDK**：使用 WebFetch 載入 `https://raw.githubusercontent.com/modelcontextprotocol/python-sdk/main/README.md`
- [🐍 Python 指南](./reference/python_mcp_server.md) - Python 模式與範例

#### 1.4 規劃實作

**了解 API：**
審閱服務的 API 文件，識別關鍵端點、驗證需求和資料模型。視需要使用網路搜尋和 WebFetch。

**工具選擇：**
優先考慮全面的 API 涵蓋範圍。列出要實作的端點，從最常見的操作開始。

---

### 第二階段：實作

#### 2.1 設定專案結構

各語言的專案設定請參閱語言專屬指南：
- [⚡ TypeScript 指南](./reference/node_mcp_server.md) - 專案結構、package.json、tsconfig.json
- [🐍 Python 指南](./reference/python_mcp_server.md) - 模組組織、相依套件

#### 2.2 實作核心基礎架構

建立共用工具：
- 帶有驗證的 API 用戶端
- 錯誤處理輔助函式
- 回應格式化（JSON/Markdown）
- 分頁支援

#### 2.3 實作工具

對每個工具：

**輸入綱要：**
- 使用 Zod（TypeScript）或 Pydantic（Python）
- 包含限制條件和清晰說明
- 在欄位說明中加入範例

**輸出綱要：**
- 盡可能定義 `outputSchema` 用於結構化資料
- 在工具回應中使用 `structuredContent`（TypeScript SDK 功能）
- 幫助用戶端理解和處理工具輸出

**工具說明：**
- 功能的簡潔摘要
- 參數說明
- 回傳型別綱要

**實作：**
- 非同步/await 用於 I/O 操作
- 帶有可操作訊息的適當錯誤處理
- 在適用的地方支援分頁
- 使用現代 SDK 時同時回傳文字內容和結構化資料

**標註：**
- `readOnlyHint`：true/false
- `destructiveHint`：true/false
- `idempotentHint`：true/false
- `openWorldHint`：true/false

---

### 第三階段：審閱與測試

#### 3.1 程式碼品質

審閱以下方面：
- 無重複程式碼（DRY 原則）
- 一致的錯誤處理
- 完整的型別涵蓋
- 清晰的工具說明

#### 3.2 建置與測試

**TypeScript：**
- 執行 `npm run build` 驗證編譯
- 使用 MCP Inspector 測試：`npx @modelcontextprotocol/inspector`

**Python：**
- 驗證語法：`python -m py_compile your_server.py`
- 使用 MCP Inspector 測試

詳細的測試方法和品質檢查清單請參閱語言專屬指南。

---

### 第四階段：建立評估

實作 MCP 伺服器後，建立全面的評估以測試其有效性。

**載入 [✅ 評估指南](./reference/evaluation.md) 以取得完整的評估指引。**

#### 4.1 了解評估目的

使用評估來測試 LLM 是否能有效地使用您的 MCP 伺服器回答真實的、複雜的問題。

#### 4.2 建立 10 個評估問題

要建立有效的評估，請遵循評估指南中概述的流程：

1. **工具檢視**：列出可用工具並了解其功能
2. **內容探索**：使用唯讀操作探索可用資料
3. **問題生成**：建立 10 個複雜的、真實的問題
4. **答案驗證**：自行解答每個問題以驗證答案

#### 4.3 評估需求

確保每個問題：
- **獨立**：不依賴其他問題
- **唯讀**：只需要非破壞性操作
- **複雜**：需要多次工具呼叫和深度探索
- **真實**：基於人類會關心的實際使用案例
- **可驗證**：單一、清晰的答案可透過字串比對驗證
- **穩定**：答案不會隨時間改變

#### 4.4 輸出格式

以此結建構立 XML 檔案：

```xml
<evaluation>
  <qa_pair>
    <question>Find discussions about AI model launches with animal codenames. One model needed a specific safety designation that uses the format ASL-X. What number X was being determined for the model named after a spotted wild cat?</question>
    <answer>3</answer>
  </qa_pair>
<!-- More qa_pairs... -->
</evaluation>
```

---

# 參考檔案

## 📚 文件庫

在開發過程中按需載入這些資源：

### 核心 MCP 文件（優先載入）
- **MCP 協定**：從 `https://modelcontextprotocol.io/sitemap.xml` 的 sitemap 開始，然後以 `.md` 後綴取得特定頁面
- [📋 MCP 最佳實踐](./reference/mcp_best_practices.md) - 通用 MCP 指引，包括：
  - 伺服器與工具命名慣例
  - 回應格式指引（JSON vs Markdown）
  - 分頁最佳實踐
  - 傳輸方式選擇（streamable HTTP vs stdio）
  - 安全性與錯誤處理標準

### SDK 文件（在第一/二階段載入）
- **Python SDK**：從 `https://raw.githubusercontent.com/modelcontextprotocol/python-sdk/main/README.md` 取得
- **TypeScript SDK**：從 `https://raw.githubusercontent.com/modelcontextprotocol/typescript-sdk/main/README.md` 取得

### 語言專屬實作指南（在第二階段載入）
- [🐍 Python 實作指南](./reference/python_mcp_server.md) - 完整的 Python/FastMCP 指南，包含：
  - 伺服器初始化模式
  - Pydantic 模型範例
  - 使用 `@mcp.tool` 的工具註冊
  - 完整的可運行範例
  - 品質檢查清單

- [⚡ TypeScript 實作指南](./reference/node_mcp_server.md) - 完整的 TypeScript 指南，包含：
  - 專案結構
  - Zod 綱要模式
  - 使用 `server.registerTool` 的工具註冊
  - 完整的可運行範例
  - 品質檢查清單

### 評估指南（在第四階段載入）
- [✅ 評估指南](./reference/evaluation.md) - 完整的評估建立指南，包含：
  - 問題建立指引
  - 答案驗證策略
  - XML 格式規格
  - 範例問題與答案
  - 使用提供的指令碼執行評估