---
source_file: SKILL.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 1120b3769e2985cefb3d25be981b1f914abeba57ae079b83c20c666c164fa9fe
translated_at: 2026-08-21
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`SKILL.md`](SKILL.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
>
> **原檔 YAML frontmatter**
> - `name`: `brand-guidelines`
> - `description`（中譯）：將 Anthropic 官方品牌色彩與字體排印套用到任何適合呈現 Anthropic 視覺風格的 artifact。凡涉及品牌色彩或樣式指引、視覺格式化、或公司設計規範時，請使用本 skill。
> - `license`: Complete terms in LICENSE.txt
<!-- /translation-header -->

# Anthropic 品牌樣式

## 概覽

若要取用 Anthropic 官方品牌識別與樣式資源，請使用本 skill。

**關鍵字**：branding、企業識別、視覺識別、後製處理、樣式套用、品牌色彩、字體排印、Anthropic 品牌、視覺格式化、視覺設計

## 品牌指引

### 色彩

**主色系：**

- 深色：`#141413` — 主要文字與深色背景
- 淺色：`#faf9f5` — 淺色背景及深色底上的文字
- 中灰：`#b0aea5` — 次要元素
- 淺灰：`#e8e6dc` — 低調背景

**輔助色：**

- 橘色：`#d97757` — 第一輔助色
- 藍色：`#6a9bcc` — 第二輔助色
- 綠色：`#788c5d` — 第三輔助色

### 字體排印

- **標題**：Poppins（備援字體：Arial）
- **內文**：Lora（備援字體：Georgia）
- **注意**：為取得最佳效果，請預先在你的環境中安裝這兩款字體

## 功能特色

### 智慧字體套用

- 將 Poppins 字體套用到標題（24pt 及以上）
- 將 Lora 字體套用到內文
- 若自訂字體不可用，自動備援至 Arial／Georgia
- 在所有系統上維持可讀性

### 文字樣式

- 標題（24pt 以上）：Poppins 字體
- 內文：Lora 字體
- 根據背景色智慧選色
- 保留文字層次與格式

### 形狀與輔助色

- 非文字的形狀元素使用輔助色
- 依序循環套用橘色、藍色、綠色
- 在符合品牌規範的前提下維持視覺活潑感

## 技術細節

### 字體管理

- 優先使用系統已安裝的 Poppins 與 Lora 字體
- 自動備援至 Arial（標題）與 Georgia（內文）
- 無需額外安裝字體——直接使用現有的系統字體
- 為取得最佳效果，請預先在你的環境中安裝 Poppins 與 Lora

### 色彩套用

- 使用 RGB 色彩值以精確匹配品牌色
- 透過 python-pptx 的 RGBColor 類別套用
- 在不同系統上維持色彩還原度