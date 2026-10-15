# 变更日志（CHANGELOG）

本文件按时间倒序记录本汉化版的发布历史。版本号遵循语义化版本（SemVer），`zh` 后缀标识为汉化版分支。

格式参考 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)。

## [Unreleased]

### 待办

- 跟进上游 `v0.6.13` 之后的新版本
- 在 issue 中跟踪上游新增/弃用 Skill 的汉化优先级
- 校对 `evals/cases/*.json` 中 trigger prompt 翻译是否需要更新

## [0.1.0-zh] - 2026-10-15

首次汉化发布。基于上游 `addyosmani/agent-skills` 仓库某次快照（与上游 `plugin.json` v0.6.13 同步）整库翻译。

### 范围

#### 文档（11 类，约 100 个文件，约 700 KB）

- 4 个根级文档：`README.md`、`AGENTS.md`、`CLAUDE.md`、`CONTRIBUTING.md`
- 5 个插件 / 市场清单配置：`plugin.json`、`.claude-plugin/{plugin,marketplace}.json`、`.codex-plugin/plugin.json`、`.agents/plugins/marketplace.json`
- 4 个 agent persona（系统提示词）：`code-reviewer.md`、`security-auditor.md`、`test-engineer.md`、`web-performance-auditor.md`
- 25 个 Skill 的 `SKILL.md`（每个 `chinese-<name>/SKILL.md`）
- 3 个 Skill 内嵌 reference：`chinese-constraint-driven-development/references/floor-guard.md`、`chinese-performance-optimization/references/optimization-patterns.md`、`chinese-security-and-hardening/references/hardening-patterns.md`
- 3 个 `idea-refine` 辅助文件：`frameworks.md`、`refinement-criteria.md`、`examples.md`
- 7 个根级 reference：`accessibility-checklist.md`、`definition-of-done.md`、`observability-checklist.md`、`testing-patterns.md`、`security-checklist.md`、`performance-checklist.md`、`orchestration-patterns.md`
- 2 个 hook 文档：`hooks/SDD-CACHE.md`、`hooks/SIMPLIFY-IGNORE.md`
- 1 个 `.claude/rules/skills-contributing.md` 护栏
- 2 个 evals 顶层文档：`evals/README.md`、`evals/skill-impact.md`
- 27 份斜杠命令文件：9 个 `.claude/commands/*.md` + 9 个 `.gemini/commands/*.toml` + 9 个 `commands/*.toml`（Antigravity）
- 17 个 `docs/*.md` 安装指南
- 1 份 `.github/ISSUE_TEMPLATE/skill-gap.yml`（汉化版，列出 `chinese-*` Skill 名）

#### 命名重映射

- 仓库根：`agent-skills` → `agent-skills-zh`
- Skill 目录：`skills/<name>` → `skills/chinese-<name>`（每个 Skill 加 `chinese-` 前缀）
- 内部子目录与文件名保留原名

#### 翻译约束

- 所有 key、路径、CLI、Issue 编号、URL、commit SHA、env 变量名、人名、品牌名、链接保留英文
- frontmatter 的 `name` / `disable-model-invocation` 等 boolean 字段不译；`description` 与 `argument-hint` 译为中文
- 行业术语首次出现用「中文（English）」括号注释
- 严格不使用 em-dash `—`，一律以 `，` 或 `（...）` 代替
- 代码块保留英文（技术锚点）

### 不翻译的部分（按 `tran-skill-zh` skill §F 排除）

- `LICENSE`（MIT 许可原文）
- `package-lock.json`（无）
- `scripts/*.js` 与 `scripts/*.sh`（评测契约 / 校验脚本）
- `evals/cases/*.json`（trigger phrases 与 Skill description 必须 1:1 对齐上游才能让评测跑通）
- `evals/plugin/*`（同上）
- `evals/fixtures/*`（评测输入样本）
- `.github/workflows/test-plugin-install.yml`（CI 工作流）
- `.gitignore`、`.gitattributes`

### 已知风险

- 上游文档 / Skill 仅英文（上游明确拒绝合并翻译 PR），本汉化版会随时间漂移
- 评测 `evals/cases/*.json` 仍为英文，但 Skill `description` 为中文，可能影响 Tier 2 路由的 TF-IDF 打分；`run-evals.js --min-rank1 95` 跑通标准尚未在汉化版验证
- 与上游不同步周期：建议每 1–2 个月 rebase 一次

### 与上游的差异

| 项 | 上游 | 本汉化版 |
|---|---|---|
| 仓库根 | `agent-skills` | `agent-skills-zh` |
| Skill 目录 | `skills/<name>/` | `skills/chinese-<name>/` |
| 文档语言 | 英文 | 简体中文 |
| `evals/cases` | 英文 trigger prompts | 不复制 |
| `evals/README.md` | 英文 | 简体中文 |
| `.github/ISSUE_TEMPLATE` | 英文，引用 `<name>` | 汉化，引用 `chinese-<name>` |

### 致谢

- 感谢 [Addy Osmani](https://github.com/addyosmani) 与 [Federico Bartoli](https://github.com/federicobartoli) 与 [Joan León](https://github.com/nucliweb) 创建并维护上游 `agent-skills` 仓库
- 上游许可证：MIT

[Unreleased]: https://example.com/agent-skills-zh/compare/v0.1.0-zh...HEAD
[0.1.0-zh]: https://example.com/agent-skills-zh/releases/tag/v0.1.0-zh
