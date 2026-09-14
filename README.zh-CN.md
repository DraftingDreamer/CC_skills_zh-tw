# CC_skills_zh-tw — Anthropic 官方 Agent Skills 繁体中文学习对照库（含简体中文导读）

[繁體中文](README.md) | [English](README.en.md) | [简体中文](README.zh-CN.md)

> **免责声明**：本项目为**非官方**社区学习与翻译项目，与 Anthropic, PBC. **无隶属关系，未获其认可或赞助**。“Claude”与“Anthropic”为 Anthropic, PBC. 之商标，本项目仅为标明原始技术来源而提及。

---

## 项目核心宗旨：深入学习与掌握官方 Agent Skills

[anthropics/skills](https://github.com/anthropics/skills) 是 Anthropic 官方为 Claude Code 与自主 AI 代理所设计的提示词与技能标准实践。每个 Skill 不仅是一组可供调用的工具指南，更蕴含了官方团队在 **提示词工程（Prompt Engineering）**、**工具调用治理（Tool Governance）**、**安全边界防护** 以及 **结构化工作流程** 方面的顶级设计思想。

本项目将官方开源的 **15 个核心技能** 完整翻译为**中国台湾正体中文（zh-TW）**，并保持**原文与译文逐字节并存**。本项目的核心目标是**帮助中文使用者与开发者透彻学习、理解与借鉴官方设计 Skill 的架构与精髓**。

---

## 收录技能清单（15 个官方开源技能）

本项目已全数完成以下 15 个开源技能的翻译，按功能领域整理如下：

| 技能目录 (Skill) | 中文名称 | 做什么用的？（核心功能说明） |
|---|---|---|
| [`claude-api`](skills/claude-api/) | Claude API 全方位指南 | 深入解析 Claude API、Managed Agents 托管架构、工具权限策略与 CLI 实践手册。 |
| [`mcp-builder`](skills/mcp-builder/) | MCP 服务器构建专家 | 指引开发者从架构规划、协议实现到安全审计，完整打造 Model Context Protocol 服务器。 |
| [`skill-creator`](skills/skill-creator/) | 技能创建与效能评测 | 官方标准 Skill 开发指南，涵盖目录结构设计、评测基准建立与 Prompt 表现优化。 |
| [`frontend-design`](skills/frontend-design/) | 前端美学与界面设计 | 突破平庸模板预设风格，打造具备独特美感、精致排版与直观体验的高品质前端 UI。 |
| [`web-artifacts-builder`](skills/web-artifacts-builder/) | 交互式 Web 组件构建 | 指引 Claude 编写完整自包含、具现代交互性与优美视觉的单页 Web 应用与 Artifacts。 |
| [`webapp-testing`](skills/webapp-testing/) | 网页应用自动化测试 | 运用 Playwright 进行端到端（E2E）自动化测试、界面交互验证与实时排错。 |
| [`canvas-design`](skills/canvas-design/) | 画布视觉排版设计 | 运用专业平面设计哲学与留白原则，设计高质感的静态海报、文宣与艺术版面。 |
| [`algorithmic-art`](skills/algorithmic-art/) | 算法生成艺术 | 指导 Claude 运用 p5.js 与数学算法，创作原创且富美感的互动与静态生成艺术。 |
| [`brand-guidelines`](skills/brand-guidelines/) | 品牌视觉设计规范 | 规范产出物符合 Anthropic 官方标准色彩、字体排版与品牌视觉风格。 |
| [`theme-factory`](skills/theme-factory/) | 视觉风格主题工厂 | 提供专业的调色板与字体搭配方案，快速为各类产出物赋予一致且优雅的风格主题。 |
| [`doc-coauthoring`](skills/doc-coauthoring/) | 结构化文档共创 | 指引 Claude 采用渐进式共同起草工作流，撰写技术白皮书、架构规范与长篇报告。 |
| [`internal-comms`](skills/internal-comms/) | 内部沟通与项目通讯 | 规范团队高效撰写状态更新、周报、架构决策记录（ADR）与事故复盘报告（Post-mortem）。 |
| [`slack-gif-creator`](skills/slack-gif-creator/) | Slack 动态 GIF 制作 | 针对 Slack 文件大小与播放限制，调校并制作流畅且吸睛的定制化动态 GIF。 |
| [`discernment-nudge`](skills/discernment-nudge/) | 批判性思维引导 | 促使 Claude 保持敏锐洞察与客观中立，避免过度迎合或阿谀，提供真正有深度的反思建议。 |
| [`academy-guide`](skills/academy-guide/) | 官方学院导引 | 协助使用者与代理理解 Anthropic 官方教学资源体系与技能架构。 |

### ⚠️ 未收录之企业级技能（4 个 Anthropic 专有许可证技能）

官方上游的 `docx`、`pdf`、`pptx`、`xlsx` 四个技能采用 **Anthropic 专有许可证（Proprietary）**，明确禁止制作衍生作品与公开发布。因此本公开学习库**未收录**上述 4 项专有技能，以严格遵守版权合规规范：

| 技能目录 (Skill) | 中文名称 | 做什么用的？（核心功能说明） |
|---|---|---|
| `docx` | Word 文档创建与编辑 | 协助用户创建、读取、编辑或转换专业的 Word 文档（.docx）与模板。 |
| `pdf` | PDF 文档创建与渲染 | 处理 PDF 文件的各项操作，包含提取文本、合并拆分、创建表单与 OCR 识别。 |
| `pptx` | PowerPoint 演示文稿生成 | 协助创建、编辑、解析与操作 PowerPoint 演示文稿文件。 |
| `xlsx` | Excel 电子表格处理 | 协助创建、编辑、解析与操作 Excel 电子表格及结构化数据。 |

---

## 本对照库的特色设计

- **英文原文原汁原味，一字不动**：每个 skill 目录下的英文原文件与上游 commit **逐字节一致**，包含原始 `LICENSE.txt`。学习时可随时对照英文原文。
- **独立译文存放（`*.zh-TW.md`）**：翻译文件命名为 `*.zh-TW.md`（例如 `SKILL.md` 对应译文为 `SKILL.zh-TW.md`）。Claude 安装时本身就内置了这些英文 Skill，译文独立存储既不会干扰 AI 的工具加载机制，又便于人类读者随时研读。
- **版本透明可追溯**：每份译文顶部的 YAML frontmatter 均记录对应的来源文件 commit 与 SHA-256 校验和，便于随时比对上游版本更新。

```text
skills/<skill>/
├── SKILL.md            # 上游原文件（英文提示词与指令，完全未修改）
├── SKILL.zh-TW.md      # 台湾正体中文译文（便于中文用户对照学习与研读）
├── LICENSE.txt         # 上游原始开源许可证
└── reference/x.md + x.zh-TW.md
```

---

## 翻译标准与品质把控

为保障高水准的中文技术研读体验：
1. **正体中文术语规范**：专业术语严格遵循权威标准（如代码、服务器、内存、异步等在地化对齐）。
2. **严防过度转换错字**：针对常见简繁转换陷阱建立白名单机制，杜绝生硬错字。
3. **代码与 API 逐字符保留**：命令行指令、代码块、API 字段名（如 `evaluated_permission`、`deny_message`）与 YAML 键名 100% 保持英文原貌。

---

## 许可证与合规声明

| 对象 | 许可证条约 | 说明 |
|---|---|---|
| 上游原始文件 | Apache License 2.0 — © 2026 Anthropic, PBC. | 遵循 Apache 2.0 原样保留与分发。 |
| `*.zh-TW.md` 译文 | Apache License 2.0 — © 2026 CC_skills_zh-tw contributors | 秉持开源社区分享精神开放使用。 |

详见 [`NOTICE`](NOTICE) 与 [`LICENSE`](LICENSE)。

---

## 原始来源

- [anthropics/skills](https://github.com/anthropics/skills) — Anthropic 官方 Agent Skills 原始仓库
