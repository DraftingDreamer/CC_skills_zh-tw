# CC_skills_zh-tw — Anthropic 官方 Agent Skills 繁體中文學習對照庫

[繁體中文](README.md) | [English](README.en.md) | [简体中文](README.zh-CN.md)

> **免責聲明**：本專案為**非官方**的社群學習與翻譯專案，與 Anthropic, PBC. **無隸屬關係，未經其背書或贊助**。「Claude」與「Anthropic」為 Anthropic, PBC. 之商標，本專案僅為指稱原始技術來源而提及。

---

## 專案核心宗旨：深入學習與掌握官方 Agent Skills

[anthropics/skills](https://github.com/anthropics/skills) 是 Anthropic 官方為 Claude Code 與自主 AI 代理所設計的提示詞與技能標準實踐。每個 Skill 不僅是可呼叫的工具指引，更蘊含了官方團隊對於 **提示詞工程（Prompt Engineering）**、**工具調度決策（Tool Use Governance）**、**邊界安全防護** 與 **結構化工作流程** 的頂尖思維。

本專案將官方開源的 **15 個核心技能** 完整翻譯為**台灣正體中文**，並維持**原文與譯文逐位元組並存**。我們的主要目的不是為了替代既有工具，而是**幫助中文使用者與開發者透徹學習、理解與借鏡官方設計 Skill 的架構與精髓**。

---

## 收錄技能清單（15 個官方開源技能）

本專案已全數完成以下 15 個技能的翻譯，依功能領域整理如下：

| 技能目錄 (Skill) | 正體中文名稱 | 做什麼用的？（核心功能說明） |
|---|---|---|
| [`claude-api`](skills/claude-api/) | Claude API 全方位指南 | 深入解析 Claude API、Managed Agents 託管架構、工具權限策略與 CLI 實踐手冊。 |
| [`mcp-builder`](skills/mcp-builder/) | MCP 伺服器建置專家 | 指引開發者從架構規劃、協定實作到安全驗證，完整打造 Model Context Protocol 伺服器。 |
| [`skill-creator`](skills/skill-creator/) | 技能建立與效能評測 | 官方標準 Skill 開發指南，涵蓋目錄架構設計、評測指標建立與 Prompt 表現最佳化。 |
| [`frontend-design`](skills/frontend-design/) | 前端美學與介面設計 | 突破平庸模板預設風格，打造具備獨特美感、精緻排版與直覺體驗的高品質前端 UI。 |
| [`web-artifacts-builder`](skills/web-artifacts-builder/) | 互動 Web 元件建構 | 指引 Claude 撰寫完整自包含、具現代互動性與美觀視覺的單頁 Web 應用與 Artifacts。 |
| [`webapp-testing`](skills/webapp-testing/) | 網頁應用自動化測試 | 運用 Playwright 進行端對端（E2E）自動化測試、介面行為驗證與即時除錯。 |
| [`canvas-design`](skills/canvas-design/) | 畫布視覺排版設計 | 運用專業平面設計哲學與留白原則，設計高質感的靜態海報、文宣與藝術版面。 |
| [`algorithmic-art`](skills/algorithmic-art/) | 演算法生成藝術 | 指導 Claude 運用 p5.js 與數學演算法，創作原創、富美感的互動與靜態生成藝術。 |
| [`brand-guidelines`](skills/brand-guidelines/) | 品牌視覺設計規範 | 規範產出物符合 Anthropic 官方標準色彩、字型排版與品牌視覺風格。 |
| [`theme-factory`](skills/theme-factory/) | 視覺風格主題工廠 | 提供專業的調色盤與字型組合方案，快速為各類產出物賦予一致且優雅的風格主題。 |
| [`doc-coauthoring`](skills/doc-coauthoring/) | 結構化文件共創 | 指引 Claude 採用漸進式共同起草工作流，撰寫技術白皮書、架構規範與長篇報告。 |
| [`internal-comms`](skills/internal-comms/) | 內部溝通與專案通訊 | 規範團隊高效撰寫狀況更新、週報、架構決策紀錄（ADR）與事故檢討報告（Post-mortem）。 |
| [`slack-gif-creator`](skills/slack-gif-creator/) | Slack 動態 GIF 製作 | 針對 Slack 檔案大小與播放限制，調校並製作流暢且吸睛的客製化動態 GIF。 |
| [`discernment-nudge`](skills/discernment-nudge/) | 批判性思考引導 | 促使 Claude 保持敏銳洞察與客觀中立，避免過度迎合或阿諛，提供真正有深度的反思建議。 |
| [`academy-guide`](skills/academy-guide/) | 官方學院導引 | 協助使用者與代理了解 Anthropic 官方教學資源體系與技能架構。 |

### ⚠️ 未收錄之企業級技能（4 個 Anthropic 專有授權技能）

因上游官方採用 **Anthropic Proprietary（專有授權）** 條款，明文規定禁止製作衍生作品與公開散布，故本公開學習庫**未收錄**以下 4 項技能，以嚴格遵守智慧財產權與官方授權規範：

| 技能目錄 (Skill) | 正體中文名稱 | 做什麼用的？（核心功能說明） |
|---|---|---|
| `docx` | Word 檔案建立與編輯 | 協助使用者建立、讀取、編輯或轉換專業的 Word 文件（.docx）與範本。 |
| `pdf` | PDF 處理與操作 | 處理 PDF 檔案的各項操作，包含擷取文字、合併拆分、建立表單與 OCR 辨識。 |
| `pptx` | PowerPoint 簡報生成 | 協助建立、編輯、解析與操作 PowerPoint 簡報檔案。 |
| `xlsx` | Excel 試算表處理 | 協助建立、編輯、解析與操作 Excel 試算表及結構化資料。 |

---

## 本對照庫的設計特點

- **原文原汁原味，一字不動**：每個 skill 目錄下的英文原檔與上游 commit **逐位元組相同**，包含原始 `LICENSE.txt`。學習時可隨時比對英文原文。
- **譯文獨立存放（`*.zh-TW.md`）**：翻譯檔案命名為 `*.zh-TW.md`（例如 `SKILL.md` 的譯文為 `SKILL.zh-TW.md`）。Claude 安裝時本來就內建這些英文 Skill，譯文獨立另存既不會干擾 AI 的工具載入機制，又方便人類讀者隨時查閱。
- **版本可追蹤性**：每份譯文頂部的 YAML frontmatter 均記錄對應的來源檔 commit 與 SHA-256 雜湊值，方便隨時追蹤上游版本演進。

```text
skills/<skill>/
├── SKILL.md            # 上游原檔（英文提示詞與指令，完全未修改）
├── SKILL.zh-TW.md      # 台灣正體中文譯文（方便人類研讀與學習對照）
├── LICENSE.txt         # 上游開源授權原檔
└── reference/x.md + x.zh-TW.md
```

---

## 翻譯標準與品質保證

為提供最高品質的中文技術學習體驗，本專案嚴格遵循以下規範：
1. **微軟與 VS Code 正體中文標準**：軟體術語嚴格採用台灣在地化權威標準（如：`程式碼`而非代碼、`伺服器`而非服務器、`記憶體`而非內存、`非同步`而非異步）。
2. **防範過度繁化錯字**：針對常見繁簡轉換陷阱進行人工與自動化白名單校驗（如：`乾淨`不誤作幹淨、`複製`不誤作復制、`檔案`不誤作文件）。
3. **代碼與 API 逐字元保留**：所有命令列指令、程式碼區塊、API 欄位名（如 `evaluated_permission`、`deny_message`）與 YAML 鍵名 100% 保持英文原樣，確保研讀時能直接對應真實程式碼。

---

## 授權與法律邊界

| 對象 | 授權條款 | 說明 |
|---|---|---|
| 上游原始檔案 | Apache License 2.0 — © 2026 Anthropic, PBC. | 依 Apache 2.0 條款原樣保留與散布。 |
| `*.zh-TW.md` 譯文 | Apache License 2.0 — © 2026 CC_skills_zh-tw contributors | 遵循開源社群分享精神開放使用。 |

完整聲明請見 [`NOTICE`](NOTICE) 與 [`LICENSE`](LICENSE)。

---

## 相關來源

- [anthropics/skills](https://github.com/anthropics/skills) — Anthropic 官方 Agent Skills 原始倉庫
