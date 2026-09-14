# CC_skills_zh-tw — Traditional Chinese Learning Reference for Official Anthropic Agent Skills

[繁體中文](README.md) | [English](README.en.md) | [简体中文](README.zh-CN.md)

> **Disclaimer**: This is an **unofficial** community learning and translation project, and is **not affiliated with, endorsed by, or sponsored by Anthropic, PBC.** "Claude" and "Anthropic" are trademarks of Anthropic, PBC., and are mentioned here solely to attribute the original technical source.

---

## Core Purpose: In-depth Learning & Understanding of Official Agent Skills

[anthropics/skills](https://github.com/anthropics/skills) represents Anthropic's official collection of prompt engineering and capability specifications designed for Claude Code and autonomous AI agents. Each skill is more than a callable prompt; it embodies the Anthropic team's best practices in **Prompt Engineering**, **Tool Governance**, **Defensive Boundary Control**, and **Structured Agent Workflows**.

This project provides complete, high-quality **Taiwan Traditional Chinese (`zh-TW`)** translations of all **15 open-source skills**, maintained side-by-side with the original English files. The primary objective is **to help Chinese-speaking developers and users thoroughly study, understand, and learn from Anthropic's architectural design and prompt craftsmanship**.

---

## List of Included Skills (15 Official Open-Source Skills)

All 15 open-source skills are completely translated and curated below:

| Skill Directory | Traditional Chinese Name | Core Function & What It Does |
|---|---|---|
| [`claude-api`](skills/claude-api/) | Claude API 全方位指南 | Comprehensive reference manual for Claude API, Managed Agents architecture, tool permission policies, and CLI workflows. |
| [`mcp-builder`](skills/mcp-builder/) | MCP 伺服器建置專家 | Guides developers from architecture and protocol implementation to security auditing when building Model Context Protocol servers. |
| [`skill-creator`](skills/skill-creator/) | 技能建立與效能評測 | Official skill engineering guide covering directory scaffolding, benchmark evaluations, and prompt performance optimization. |
| [`frontend-design`](skills/frontend-design/) | 前端美學與介面設計 | Guides the creation of distinctive, production-grade frontend UI/UX that breaks away from generic template aesthetics. |
| [`web-artifacts-builder`](skills/web-artifacts-builder/) | 互動 Web 元件建構 | Instructs Claude to author self-contained, reactive, and visually refined web artifacts and single-page apps. |
| [`webapp-testing`](skills/webapp-testing/) | 網頁應用自動化測試 | Leverages Playwright for end-to-end (E2E) automated browser testing, UI interaction verification, and live debugging. |
| [`canvas-design`](skills/canvas-design/) | 畫布視覺排版設計 | Applies graphic design philosophy, whitespace discipline, and typography hierarchy to static posters and visual layouts. |
| [`algorithmic-art`](skills/algorithmic-art/) | 演算法生成藝術 | Directs Claude to craft algorithmic and generative artwork using p5.js and mathematical logic. |
| [`brand-guidelines`](skills/brand-guidelines/) | 品牌視覺設計規範 | Enforces official Anthropic brand colors, typographic hierarchy, and visual design standards. |
| [`theme-factory`](skills/theme-factory/) | 視覺風格主題工廠 | Provides curated color palettes and typography pairings to rapidly skin applications with cohesive aesthetics. |
| [`doc-coauthoring`](skills/doc-coauthoring/) | 結構化文件共創 | Guides Claude through structured co-authoring workflows for drafting technical whitepapers, architectural specs, and in-depth reports. |
| [`internal-comms`](skills/internal-comms/) | 內部溝通與專案通訊 | Standardizes high-efficiency team status reports, weekly briefs, Architectural Decision Records (ADRs), and incident post-mortems. |
| [`slack-gif-creator`](skills/slack-gif-creator/) | Slack 動態 GIF 製作 | Crafts optimized, engaging animated GIFs tailored specifically to Slack file constraints and display requirements. |
| [`discernment-nudge`](skills/discernment-nudge/) | 批判性思考引導 | Nudges Claude to maintain keen discernment and objective neutrality, resisting sycophancy to deliver authentic analytical value. |
| [`academy-guide`](skills/academy-guide/) | 官方學院導引 | Orients developers and agents within Anthropic's official educational resource ecosystem and skill structures. |

### ⚠️ Exclusion of 4 Proprietary Office Skills

Upstream official skills `docx`, `pdf`, `pptx`, and `xlsx` are licensed under **Anthropic Proprietary** terms ("Source-Available, not open source"), which explicitly forbid derivative works and redistribution. Consequently, this public repository **does not include** these four proprietary skills:

| Skill Directory | Traditional Chinese Name | Core Function & What It Does |
|---|---|---|
| `docx` | Word 檔案建立與編輯 | Assists users in creating, reading, editing, or converting professional Word documents (.docx) and templates. |
| `pdf` | PDF 處理與操作 | Handles various PDF operations including text extraction, merging, splitting, forms creation, and OCR processing. |
| `pptx` | PowerPoint 簡報生成 | Assists in creating, editing, parsing, and manipulating PowerPoint presentation files. |
| `xlsx` | Excel 試算表處理 | Assists in creating, editing, parsing, and manipulating Excel spreadsheets and structured data. |

---

## Design Highlights of This Reference Library

- **Byte-for-byte Original Fidelity**: Original English files under each skill folder are **100% byte-for-byte identical** to the upstream source commit, including original `LICENSE.txt` files. Learners can always cross-reference the original text.
- **Dedicated Translation Files (`*.zh-TW.md`)**: Translations reside alongside originals in `*.zh-TW.md` files (e.g., `SKILL.zh-TW.md`). Because Claude already ships with these skills built-in, keeping translations separate ensures zero interference with AI runtime tool-loading while offering human readers immediate readability.
- **Full Version Traceability**: YAML frontmatter on every translation documents the upstream source commit and SHA-256 hash for transparent version tracking.

```text
skills/<skill>/
├── SKILL.md            # Upstream original (unmodified English prompts & instructions)
├── SKILL.zh-TW.md      # Taiwan Traditional Chinese translation (for learning & reference)
├── LICENSE.txt         # Upstream original license
└── reference/x.md + x.zh-TW.md
```

---

## Translation Quality & Standards

To ensure an exceptional technical reading and learning experience:
1. **Microsoft & VS Code zh-Hant Terminology**: Software domain terms strictly follow Taiwan localization standards.
2. **Anti-Overconversion Defense**: Validated against character over-conversion pitfalls via automated whitelists and manual auditing.
3. **Verbatim Code & API Preservation**: CLI commands, code fences, API parameters (e.g., `evaluated_permission`, `deny_message`), and YAML keys remain untouched in English.

---

## Licensing & Proprietary Exclusion

| Target | License | Notes |
|---|---|---|
| Upstream original files | Apache License 2.0 — © 2026 Anthropic, PBC. | Retained and distributed under Apache 2.0 terms. |
| `*.zh-TW.md` translations | Apache License 2.0 — © 2026 CC_skills_zh-tw contributors | Shared openly in the spirit of community learning. |

See [`NOTICE`](NOTICE) and [`LICENSE`](LICENSE) for complete details.

---

## Source Attribution

- [anthropics/skills](https://github.com/anthropics/skills) — Official Anthropic Agent Skills Repository
