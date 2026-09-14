---
source_file: SKILL.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 81c5002c6643b0de7b8710b00e7a9038daa6fb9b68d59870ee6adb12da8d10f8
translated_at: 2026-08-21
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`SKILL.md`](SKILL.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
>
> **原檔 YAML frontmatter**
> - `name`: `web-artifacts-builder`
> - `description`（中譯）：一套使用現代前端網頁技術（React、Tailwind CSS、shadcn/ui）建構精細多元件 claude.ai HTML artifact 的工具組。適用於需要狀態管理、路由或 shadcn/ui 元件的複雜 artifact——不適用於簡單的單檔 HTML/JSX artifact。
> - `license`: Complete terms in LICENSE.txt
<!-- /translation-header -->

# Web Artifacts Builder

若要建構強大的前端 claude.ai artifact，請依照以下步驟：
1. 使用 `scripts/init-artifact.sh` 初始化前端專案
2. 編輯生成的程式碼以開發你的 artifact
3. 使用 `scripts/bundle-artifact.sh` 將所有程式碼打包成單一 HTML 檔
4. 將 artifact 展示給使用者
5. （可選）測試 artifact

**技術堆疊**：React 18 + TypeScript + Vite + Parcel（打包）+ Tailwind CSS + shadcn/ui

## 設計與樣式指引

非常重要：為了避免所謂的「AI 濫造」（AI slop），請避免使用過多置中版面、紫色漸層、統一圓角，以及 Inter 字體。

## 快速入門

### 步驟一：初始化專案

執行初始化指令碼以建立新的 React 專案：
```bash
bash scripts/init-artifact.sh <project-name>
cd <project-name>
```
這會建立一個完整設定的專案，包含：
- ✅ React + TypeScript（透過 Vite）
- ✅ Tailwind CSS 3.4.1 搭配 shadcn/ui 主題系統
- ✅ 已設定路徑別名（`@/`）
- ✅ 預裝 40 個以上的 shadcn/ui 元件
- ✅ 包含所有 Radix UI 相依套件
- ✅ 已為打包設定 Parcel（透過 .parcelrc）
- ✅ Node 18+ 相容性（自動偵測並固定 Vite 版本）

### 步驟二：開發你的 Artifact

若要建構 artifact，請編輯生成的檔案。請參閱下方的**常見開發工作**以取得指引。

### 步驟三：打包成單一 HTML 檔

若要將 React 應用程式打包成單一 HTML artifact：
```bash
bash scripts/bundle-artifact.sh
```
這會建立 `bundle.html`——一個包含所有 JavaScript、CSS 與相依套件的自給自足 artifact。此檔案可直接在 Claude 對話中以 artifact 形式分享。

**需求**：你的專案根目錄必須有 `index.html` 檔案。

**指令碼的操作內容**：
- 安裝打包相依套件（parcel、@parcel/config-default、parcel-resolver-tspaths、html-inline）
- 建立支援路徑別名的 `.parcelrc` 設定
- 使用 Parcel 建構（不含 source map）
- 使用 html-inline 將所有資產內嵌到單一 HTML 中

### 步驟四：與使用者分享 Artifact

最後，在對話中與使用者分享打包好的 HTML 檔案，讓他們以 artifact 形式檢視。

### 步驟五：測試／視覺化 Artifact（可選）

注意：這是完全可選的步驟。只有在必要或有請求時才執行。

若要測試／視覺化 artifact，請使用可用的工具（包括其他 skill 或如 Playwright 或 Puppeteer 等內建工具）。一般而言，避免預先測試 artifact，因為這會在請求與完成品可見之間增加延遲。若有請求或發現問題，在展示 artifact 後再進行測試。

## 參考資料

- **shadcn/ui 元件**：https://ui.shadcn.com/docs/components