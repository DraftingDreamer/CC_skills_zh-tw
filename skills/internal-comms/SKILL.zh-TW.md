---
source_file: SKILL.md
source_commit: 41bbe19d1a1a7eaab5e7bb9050a417e5c6cffc8f
source_sha256: 067b7587a344a928fc6534ef66b1bcd591fc7c26d207ea7ca3334aeb678d6475
translated_at: 2026-08-13
---

<!-- translation-header -->
> **譯文說明**｜本檔是 [`SKILL.md`](SKILL.md) 的台灣正體中文譯文，供閱讀理解之用；原檔一字未改，實際使用請以原文為準。
>
> **原檔 YAML frontmatter**
> - `name`: `internal-comms`
> - `description`（中譯）：一套協助我撰寫各類內部通訊的資源，套用我公司慣用的格式。當被要求撰寫任何形式的內部通訊（狀態報告、高層更新、3P 更新、公司電子報、常見問答（FAQ）、事故報告、專案更新等）時，Claude 都應使用本 skill。
> - `license`: Complete terms in LICENSE.txt
<!-- /translation-header -->

## 何時使用本 skill
撰寫內部通訊時，本 skill 適用於：
- 3P 更新（Progress, Plans, Problems：進度、計畫、問題）
- 公司電子報
- 常見問答（FAQ）回覆
- 狀態報告
- 高層更新
- 專案更新
- 事故報告

## 如何使用本 skill

撰寫任何內部通訊時：

1. **從請求判斷通訊類型**
2. **從 `examples/` 目錄載入對應的指引檔案**：
    - `examples/3p-updates.md`：用於 Progress/Plans/Problems（進度、計畫、問題）團隊更新
    - `examples/company-newsletter.md`：用於全公司電子報
    - `examples/faq-answers.md`：用於回答常見問題
    - `examples/general-comms.md`：用於不屬於以上任何一類的情況
3. **依該檔案中的具體指示**進行格式、語氣與內容蒐集

若通訊類型不符合任何現有指引，請提出釐清問題，或詢問更多關於期望格式的背景資訊。

## 關鍵字
3P 更新、公司電子報、公司通訊、每週更新、FAQ、常見問題、更新、內部通訊
