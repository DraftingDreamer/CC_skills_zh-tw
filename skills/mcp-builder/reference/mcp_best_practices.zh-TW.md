---
source_file: mcp_best_practices.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 80fb4369a349447cf18ecdd7494fe7938b6065377e9f08c077cec411093a3007
translated_at: 2026-08-22
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`mcp_best_practices.md`](mcp_best_practices.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
<!-- /translation-header -->

# MCP 伺服器最佳實踐

## 快速參考

### 伺服器命名
- **Python**：`{service}_mcp`（例如 `slack_mcp`）
- **Node/TypeScript**：`{service}-mcp-server`（例如 `slack-mcp-server`）

### 工具命名
- 使用 snake_case 加上服務前綴
- 格式：`{service}_{action}_{resource}`
- 範例：`slack_send_message`、`github_create_issue`

### 回應格式
- 同時支援 JSON 和 Markdown 格式
- JSON 用於程式化處理
- Markdown 用於人類可讀性

### 分頁
- 始終遵守 `limit` 參數
- 回傳 `has_more`、`next_offset`、`total_count`
- 預設為 20-50 個項目

### 傳輸方式
- **Streamable HTTP**：用於遠端伺服器、多客戶端情境
- **stdio**：用於本機整合、命令列工具
- 避免使用 SSE（已棄用，改用 streamable HTTP）

---

## 伺服器命名慣例

遵循以下標準化命名模式：

**Python**：使用格式 `{service}_mcp`（小寫加底線）
- 範例：`slack_mcp`、`github_mcp`、`jira_mcp`

**Node/TypeScript**：使用格式 `{service}-mcp-server`（小寫加連字號）
- 範例：`slack-mcp-server`、`github-mcp-server`、`jira-mcp-server`

名稱應具通用性、描述所整合的服務、易於從任務描述推斷，且不含版本號。

---

## 工具命名與設計

### 工具命名

1. **使用 snake_case**：`search_users`、`create_project`、`get_channel_info`
2. **包含服務前綴**：預期您的 MCP 伺服器可能與其他 MCP 伺服器一起使用
   - 使用 `slack_send_message` 而非僅 `send_message`
   - 使用 `github_create_issue` 而非僅 `create_issue`
3. **以動作為導向**：以動詞開頭（get、list、search、create 等）
4. **具體明確**：避免可能與其他伺服器衝突的通用名稱

### 工具設計

- 工具說明必須精確且無歧義地描述功能
- 說明必須精確匹配實際功能
- 提供工具標註（readOnlyHint、destructiveHint、idempotentHint、openWorldHint）
- 保持工具操作聚焦且原子化

---

## 回應格式

所有回傳資料的工具都應支援多種格式：

### JSON 格式（`response_format="json"`）
- 機器可讀的結構化資料
- 包含所有可用欄位和中繼資料
- 一致的欄位名稱和型別
- 用於程式化處理

### Markdown 格式（`response_format="markdown"`，通常為預設）
- 人類可讀的格式化文字
- 使用標題、清單和格式化以提升清晰度
- 將時間戳記轉換為人類可讀格式
- 以括號顯示 ID 搭配顯示名稱
- 省略冗長的中繼資料

---

## 分頁

對於列出資源的工具：

- **始終遵守 `limit` 參數**
- **實作分頁**：使用 `offset` 或基於游標的分頁
- **回傳分頁中繼資料**：包含 `has_more`、`next_offset`/`next_cursor`、`total_count`
- **絕不將所有結果載入記憶體**：對大型資料集尤其重要
- **預設合理的限制**：20-50 個項目是常見的做法

分頁回應範例：
```json
{
  "total": 150,
  "count": 20,
  "offset": 0,
  "items": [...],
  "has_more": true,
  "next_offset": 20
}
```

---

## 傳輸選項

### Streamable HTTP

**最適合**：遠端伺服器、網路服務、多客戶端情境

**特性**：
- 透過 HTTP 進行雙向通訊
- 支援多個同時連線的客戶端
- 可部署為網路服務
- 支援伺服器對客戶端的通知

**適用時機**：
- 同時服務多個客戶端
- 部署為雲端服務
- 與網路應用程式整合

### stdio

**最適合**：本機整合、命令列工具

**特性**：
- 標準輸入/輸出串流通訊
- 設定簡單，不需要網路設定
- 作為客戶端的子行程執行

**適用時機**：
- 建立本機開發環境的工具
- 與桌面應用程式整合
- 單一使用者、單一 session 的情境

**注意**：stdio 伺服器**不應**將日誌記錄到 stdout（應使用 stderr 記錄日誌）

### 傳輸方式選擇

| 評選標準 | stdio | Streamable HTTP |
|----------|-------|----------------|
| **部署方式** | 本機 | 遠端 |
| **客戶端數量** | 單一 | 多個 |
| **複雜度** | 低 | 中 |
| **即時性** | 否 | 是 |

---

## 安全性最佳實踐

### 驗證與授權

**OAuth 2.1**：
- 使用帶有來自受信任機構憑證的安全 OAuth 2.1
- 在處理請求前驗證存取權杖
- 只接受專門針對您的伺服器的權杖

**API 金鑰**：
- 將 API 金鑰儲存於環境變數中，絕不寫入程式碼
- 在伺服器啟動時驗證金鑰
- 在驗證失敗時提供清楚的錯誤訊息

### 輸入驗證

- 清除檔案路徑以防止目錄遍歷攻擊
- 驗證 URL 和外部識別碼
- 檢查參數大小和範圍
- 防止系統呼叫中的指令注入
- 對所有輸入使用綱要驗證（Pydantic/Zod）

### 錯誤處理

- 不向客戶端公開內部錯誤
- 在伺服器端記錄與安全性相關的錯誤
- 提供有用但不洩露細節的錯誤訊息
- 在錯誤後清除資源

### DNS 重新繫結保護

對於在本機執行的 streamable HTTP 伺服器：
- 啟用 DNS 重新繫結保護
- 驗證所有傳入連線的 `Origin` 標頭
- 繫結至 `127.0.0.1` 而非 `0.0.0.0`

---

## 工具標註

提供標註以幫助客戶端理解工具行為：

| 標註 | 型別 | 預設值 | 說明 |
|------|------|--------|------|
| `readOnlyHint` | boolean | false | 工具不修改其環境 |
| `destructiveHint` | boolean | true | 工具可能執行破壞性更新 |
| `idempotentHint` | boolean | false | 以相同參數重複呼叫不會產生額外效果 |
| `openWorldHint` | boolean | true | 工具與外部實體互動 |

**重要**：標註是提示，不是安全保證。客戶端不應僅根據標註做出安全關鍵的決策。

---

## 錯誤處理

- 使用標準 JSON-RPC 錯誤碼
- 在結果物件內回報工具錯誤（而非協定層級的錯誤）
- 提供有用、具體的錯誤訊息，附帶建議的後續步驟
- 不公開內部實作細節
- 在錯誤時正確清除資源

錯誤處理範例：
```typescript
try {
  const result = performOperation();
  return { content: [{ type: "text", text: result }] };
} catch (error) {
  return {
    isError: true,
    content: [{
      type: "text",
      text: `Error: ${error.message}. Try using filter='active_only' to reduce results.`
    }]
  };
}
```

---

## 測試需求

全面的測試應涵蓋：

- **功能測試**：以有效/無效輸入驗證正確執行
- **整合測試**：測試與外部系統的互動
- **安全性測試**：驗證驗證機制、輸入清除、速率限制
- **效能測試**：檢查負載下的行為、逾時
- **錯誤處理**：確保適當的錯誤回報與資源清除

---

## 文件需求

- 提供所有工具和功能的清晰文件
- 包含可運行的範例（每個主要功能至少 3 個）
- 記錄安全性考量
- 說明所需的權限和存取層級
- 記錄速率限制和效能特性