---
source_file: SKILL.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 51b7349e77ec63b7744a6f63647e7566a0b4d2e301121cc10e8c2113af6556a2
translated_at: 2026-08-21
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`SKILL.md`](SKILL.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
>
> **原檔 YAML frontmatter**
> - `name`: `webapp-testing`
> - `description`（中譯）：使用 Playwright 與本地端網頁應用程式互動及測試的工具組。支援驗證前端功能、偵錯 UI 行為、擷取瀏覽器截圖，以及檢視瀏覽器日誌。
> - `license`: Complete terms in LICENSE.txt
<!-- /translation-header -->

# 網頁應用程式測試

若要測試本地端網頁應用程式，請撰寫原生 Python Playwright 指令碼。

**可用的輔助指令碼**：
- `scripts/with_server.py`——管理伺服器生命週期（支援多個伺服器）

**始終先以 `--help` 執行指令碼**以查看使用說明。除非先嘗試直接執行指令碼後確認真的需要自訂解法，否則請勿讀取原始碼。這些指令碼可能相當龐大，會污染你的 context window。它們的設計是以黑盒指令碼的方式直接呼叫，而非讀入你的 context window。

## 決策樹：選擇你的方式

```
User task → Is it static HTML?
    ├─ Yes → Read HTML file directly to identify selectors
    │         ├─ Success → Write Playwright script using selectors
    │         └─ Fails/Incomplete → Treat as dynamic (below)
    │
    └─ No (dynamic webapp) → Is the server already running?
        ├─ No → Run: python scripts/with_server.py --help
        │        Then use the helper + write simplified Playwright script
        │
        └─ Yes → Reconnaissance-then-action:
            1. Navigate and wait for networkidle
            2. Take screenshot or inspect DOM
            3. Identify selectors from rendered state
            4. Execute actions with discovered selectors
```
## 範例：使用 with_server.py

若要啟動伺服器，先執行 `--help`，然後使用輔助工具：

**單一伺服器：**
```bash
python scripts/with_server.py --server "npm run dev" --port 5173 -- python your_automation.py
```
**多個伺服器（例如後端 + 前端）：**
```bash
python scripts/with_server.py \
  --server "cd backend && python server.py" --port 3000 \
  --server "cd frontend && npm run dev" --port 5173 \
  -- python your_automation.py
```
若要建立自動化指令碼，只包含 Playwright 邏輯（伺服器由輔助工具自動管理）：
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True) # Always launch chromium in headless mode
    page = browser.new_page()
    page.goto('http://localhost:5173') # Server already running and ready
    page.wait_for_load_state('networkidle') # CRITICAL: Wait for JS to execute
    # ... your automation logic
    browser.close()
```
## 偵察後再行動模式

1. **檢查渲染後的 DOM**：
   ```python
   page.screenshot(path='/tmp/inspect.png', full_page=True)
   content = page.content()
   page.locator('button').all()
   ```

2. **從檢查結果辨識選擇器**

3. **使用找到的選擇器執行動作**

## 常見陷阱

❌ **不要**在動態應用程式等待 `networkidle` 之前就檢查 DOM
✅ **請先**等待 `page.wait_for_load_state('networkidle')` 再進行檢查

## 最佳實踐

- **將打包指令碼視為黑盒**——若要完成任務，考慮 `scripts/` 中是否有指令碼可以幫忙。這些指令碼可靠地處理常見的複雜工作流程，而不會佔用 context window。使用 `--help` 查看用法，然後直接呼叫。
- 使用 `sync_playwright()` 撰寫同步指令碼
- 使用完畢後始終關閉瀏覽器
- 使用描述性選擇器：`text=`、`role=`、CSS 選擇器或 ID
- 加入適當的等待：`page.wait_for_selector()` 或 `page.wait_for_timeout()`

## 參考檔案

- **examples/**——展示常見模式的範例：
  - `element_discovery.py`——在頁面上探索按鈕、連結與輸入框
  - `static_html_automation.py`——使用 file:// URL 處理本地端 HTML
  - `console_logging.py`——在自動化過程中擷取主控台日誌