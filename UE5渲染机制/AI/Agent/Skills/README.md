# 当前 Skills 清单

这是一份 2026-07-19 的 Codex skill 快照，方便在 Obsidian 中检索、阅读和维护。共有 75 个：70 个用户管理的 skill 位于本目录，5 个系统 skill 位于 `_system/`。每个 skill 的触发条件和具体流程以其 `SKILL.md` 为准。

> 快照按精简后的目标集合制作，未收录英文 OpenSpec 的 4 个 skill 及 `quick-commit`。当前环境拦截了对源目录的永久删除，因此这 5 个源目录仍需在本机手动删除；它们不在本快照和本次 Git 提交中。

## 怎么选

不要按数量去调用 skill。先按任务选一个主 skill；只有在工作天然跨边界时再组合第二个。例如：写 Vault 技术文档时用 `obsidian-technical-documentation` + `obsidian-git-workflow`；开发功能时用 `implement` 或中文 OpenSpec；量化回测时用 `backtesting-frameworks`，需要风险评价再加 `risk-metrics-calculation`。

| 能力域 | 数量 | 适用问题 |
| --- | ---: | --- |
| 文档、知识与教学 | 11 | 写 README、技术笔记、ADR、代码说明、交接和教学 |
| 代码工程与架构 | 10 | 实现、审查、调试、重构与模块边界 |
| 测试、前端与浏览器自动化 | 10 | 单测、集成测、Web 验证与前端界面 |
| 工程基础设施与安全 | 4 | API 安全、数据库、依赖和 Git 冲突 |
| 量化研究与风险 | 2 | 回测偏差控制和风险指标 |
| 研究、评审与规范 | 8 | 调研、双轴评审、中文 OpenSpec、规格和任务拆分 |
| 产品发现、规划与沟通 | 16 | PRD、优先级、定位、实验、战略叙事与反馈 |
| 思维校验 | 8 | 概率、证据、系统、预演失败与科学排查 |
| Skill 发现 | 1 | 寻找和安装可能已有的能力 |
| Codex 系统能力 | 5 | 图片、OpenAI 文档、插件和 skill 的创建/安装 |

## 按能力域浏览

### 文档、知识与教学

`create-readme`、`doc-generator`、`documentation`、`documentation-and-adrs`、`explain-code`、`handoff`、`obsidian-git-workflow`、`obsidian-technical-documentation`、`teach`、`ubiquitous-language`、`writing-shape`。

其中，`doc-generator` 只负责从代码推导 docstring、JSDoc、接口参考和变更片段；README、架构文档和 Obsidian 技术笔记分别交给更专门的 skill。

### 代码工程与架构

`code-review`、`codebase-design`、`diagnosing-bugs`、`domain-modeling`、`edit-file`、`fix-bug`、`implement`、`improve-codebase-architecture`、`refactor`、`request-refactor-plan`。

### 测试、前端与浏览器自动化

`frontend-design`、`generate-test`、`integration-testing`、`javascript-typescript-jest`、`playwright-best-practices`、`playwright-cli`、`pytest-coverage`、`tdd`、`vitest`、`webapp-testing`。

### 工程基础设施与安全

`api-security-testing`、`database-schema-designer`、`dependency-update`、`resolving-merge-conflicts`。

### 量化研究与风险

`backtesting-frameworks`、`risk-metrics-calculation`。

### 研究、评审与规范

`mp-code-review`、`mp-research`、`openspec-archiving-cn`、`openspec-context-loading-cn`、`openspec-implementation-cn`、`openspec-proposal-creation-cn`、`to-spec`、`to-tickets`。

中文 OpenSpec 保留，用于“先加载上下文，再写提案，再实施，最后归档”的较大变更；小改动不必强行走它。

### 产品发现、规划与沟通

`ab-test-designer`、`design-sprint`、`feature-prioritization-assistant`、`jobs-to-be-done`、`okrs`、`opportunity-solution-trees`、`positioning-canvas`、`prd-writer`、`prototype`、`radical-candor`、`shape-up`、`stakeholder-update-generator`、`strategic-narrative`、`trustworthy-experiments`、`user-feedback-synthesizer`、`working-backwards`。

### 思维校验

`thinking-bayesian`、`thinking-in-bets`、`thinking-inversion`、`thinking-map-territory`、`thinking-pre-mortem`、`thinking-probabilistic`、`thinking-scientific-method`、`thinking-systems`。

这些不是每次都要显式调用的流程工具；只有涉及预测、实验、复杂排障或高风险决策时才值得使用。

### Skill 发现与系统能力

- `find-skills`：在当前能力不足时，先找现有 skill 再决定是否新建。
- `_system/imagegen`、`_system/openai-docs`、`_system/plugin-creator`、`_system/skill-creator`、`_system/skill-installer`：Codex 自带能力，分别覆盖图片、官方 OpenAI 文档、插件和 skill 的创建或安装。

## 维护约定

- 把本目录当作可追溯快照，不在这里直接修改运行时 skill。
- 要调整运行时 skill，先修改 `C:\Users\27648\.codex\skills` 中的源文件，验证后再同步本目录并提交。
- 定期删除或归档已不用的 skill，而不是追求某个固定数量。关键是让每个保留 skill 的触发条件清晰、描述简短、用途不重叠。
