# agent-skills

这是 agent-skills 项目，一组面向 AI 编程 Agent 的生产级工程 Skill。

> **作用范围：** 本文件配置工作在 [`addyosmani/agent-skills`](https://github.com/addyosmani/agent-skills) 仓库本身的 Agent，不是给其它项目用的。不要复制到别的项目或全局 Agent 配置；可复用的资产是 `skills/` 下的 Skill。

## 项目结构

```
skills/       → 核心 Skill（每个子目录一份 SKILL.md）
agents/       → 可复用 Agent 角色（code-reviewer、test-engineer、security-auditor、web-performance-auditor）
hooks/        → 会话生命周期钩子
.claude/commands/ → 斜杠命令（/spec、/plan、/build、/test、/review、/code-simplify、/ship；外加专用 /webperf 审计）
references/   → 补充清单（测试、性能、安全、可访问性、可观测性）
evals/        → Skill 评测用例 + 评测框架（见 evals/README.md）
docs/         → 各工具的安装指南
```

## 按阶段划分的 Skill

**Define：** interview-me、idea-refine、spec-driven-development
**Plan：** planning-and-task-breakdown
**Build：** incremental-implementation、test-driven-development、context-engineering、source-driven-development、doubt-driven-development、frontend-ui-engineering、api-and-interface-design
**Verify：** browser-testing-with-devtools、debugging-and-error-recovery
**Review：** code-review-and-quality、code-simplification、security-and-hardening、performance-optimization
**Ship：** git-workflow-and-versioning、ci-cd-and-automation、deprecation-and-migration、documentation-and-adrs、observability-and-instrumentation、shipping-and-launch

## 约定

- 每个 Skill 都在 `skills/<name>/SKILL.md`
- YAML frontmatter，含 `name` 和 `description` 两个字段
- `description` 以 "这个 Skill 做什么"（第三人称）打头，后接触发条件 "Use when..." 等
- 每个 Skill 都包含：Overview、When to Use、Process、Common Rationalizations、Red Flags、Verification
- 共享 reference 放在根 `references/` 目录；面向自包含、可分发的 Skill 的新约定是把自有 reference 放到 `skills/<name>/references/`
- 辅助文件仅在内容超过 100 行时创建

## 贡献

新增 Skill 或大改现有 Skill 之前，先按 [CONTRIBUTING.md](CONTRIBUTING.md#before-proposing-a-new-skill) 中的预检步骤：搜索目录、检查已开 PR，确认想法契合 [docs/skill-anatomy.md](docs/skill-anatomy.md)，并说明 "为什么需要"。优先扩展现有 Skill 而不是新建近似重复的目录。CONTRIBUTING.md 是这套工作流的单一权威；不要在这里或其它地方重复它的清单，做链接即可。

## 命令

- `npm test`：不适用（本项目是文档项目）
- 校验：检查所有 SKILL.md 是否都有合法的 YAML frontmatter（name + description）
- 评测：`node scripts/run-evals.js`，对每个 Skill 跑触发/路由评测（CI）；`--behavioral <skill>` 跑评分评测

## Pull Request

PR 目标是上游仓库的默认分支。典型 fork 场景中上游远端是 `upstream`，fork 仓库是 `origin`，但远端名细节不重要。

- 在打开 PR 前，先搜索上游仓库的开放 PR 与 issue，看是否有工作触及相同文件或规则。如有重叠，协调进行（在它之上构建、对齐规则，或在它合并后 rebase），而不是打开冲突的 PR。
- 优先小而专的 PR，而非对共享面广的文件（如 `scripts/` 下的）做大幅重构；后者更容易撞车。

## 边界

- 始终：创建新 Skill 目录前先跑 CONTRIBUTING.md 中的预检
- 始终：新增 Skill 遵循 skill-anatomy.md 中的格式
- 始终：开新 PR 之前先检查上游 PR 与 issue 的重叠
- 永远：不要添加写空泛建议而不是可执行流程的 Skill
- 永远：不要在多个 Skill 之间重复内容，要相互引用
