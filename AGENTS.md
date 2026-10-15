# AGENTS.md

当 AI 编程 Agent（Claude Code、Cursor、Copilot、Antigravity 等）在仓库中工作时，本文件提供指导。

> **作用范围：** 本文件是给在 [`addyosmani/agent-skills`](https://github.com/addyosmani/agent-skills) 仓库本身工作的 Agent 准备的，不应被复制到其它项目或全局 Agent 配置中；可复用的资产是 `skills/` 里的 Skill，不是本文件。

## 仓库总览

本仓库是一组面向 Claude.ai 和 Claude Code 的资深软件工程师 Skill 集合。Skill 是打包好的指令和脚本，用于扩展 Claude 与你的编程 Agent 的能力。

## OpenCode 集成

OpenCode 使用**由 Skill 驱动的执行模型**，由 `skill` 工具与本仓库的 `/skills` 目录提供。

### 核心规则

- 如果任务与某个 Skill 匹配，你必须调用它
- Skill 放在 `skills/<skill-name>/SKILL.md`
- 即使感觉可以直写，只要 Skill 适用就不能直接实现
- 始终严格按 Skill 指示执行（不可部分采纳）

### 意图 → Skill 映射

Agent 应当自动把用户意图映射到 Skill：

- 新功能 / 新行为 → `spec-driven-development`，然后 `incremental-implementation`、`test-driven-development`
- 规划 / 拆分 → `planning-and-task-breakdown`
- Bug / 失败 / 意外行为 → `debugging-and-error-recovery`
- 代码评审 → `code-review-and-quality`
- 重构 / 简化 → `code-simplification`
- API 或接口设计 → `api-and-interface-design`
- UI 工作 → `frontend-ui-engineering`

### 生命周期映射（隐式命令）

OpenCode 不支持像 `/spec`、`/plan` 这种斜杠命令。

Agent 必须在内部遵循以下生命周期：

- DEFINE → `spec-driven-development`
- PLAN → `planning-and-task-breakdown`
- BUILD → `incremental-implementation` + `test-driven-development`
- VERIFY → `debugging-and-error-recovery`
- REVIEW → `code-review-and-quality`
- SHIP → `shipping-and-launch`

### 执行模型

每一次请求：

1. 判断是否有 Skill 适用（哪怕只是 1% 的可能）
2. 用 `skill` 工具调用相应 Skill
3. 严格遵循 Skill 工作流
4. 只有在必须步骤（spec、plan 等）完成后，才能进入实施

### 防甩锅

下列想法都不对，必须忽略：

- "这事太小了，用不到 Skill"
- "我直接快速实现一下就行"
- "我先收集一下上下文"

正确行为：

- 始终先检查并使用 Skill

这样才能让 OpenCode 行为上贴近 Claude Code 的完整工作流强制。

## 编排：角色、Skill 与命令

本仓库有三层可组合的抽象。它们职责不同，不要混淆：

- **Skill**（`skills/<name>/SKILL.md`），带步骤和退出条件的工作流。回答 "how"。意图匹配时是必经环节。
- **角色**（`agents/<role>.md`），带视角与输出格式的角色设定。回答 "who"。
- **斜杠命令**（`.claude/commands/*.md`），面向用户的入口。回答 "when"，是编排层。

组合规则：**用户（或斜杠命令）是编排者；角色不调用其它角色。** 角色可以调用 Skill。

本仓库背书的多角色编排模式只有一种：**并行扇出 + 合并步骤**，由 `/ship` 用于同时调度 `code-reviewer`、`security-auditor`、`test-engineer` 并合成它们的报告。不要造一个"路由型"角色去决定该调哪个角色，那是斜杠命令和意图映射的活。

详见 [docs/agents.md](docs/agents.md) 的决策矩阵，以及 [references/orchestration-patterns.md](references/orchestration-patterns.md) 的完整模式目录。

**Claude Code 互操作：** `agents/` 下的角色既可作为 Claude Code 的 subagent（由本插件的 `agents/` 目录自动发现），也可作为 Agent Teams 的队友（通过名字生成）。两个平台约束与我们规则一致：subagent 不能生成其它 subagent，teams 不能嵌套。本插件的 Agent 会静默忽略 frontmatter 中的 `hooks`、`mcpServers`、`permissionMode` 字段。

## 新建一个 Skill

> **动手之前：** 先按 [CONTRIBUTING.md](CONTRIBUTING.md#before-proposing-a-new-skill) 中的预检步骤搜索目录、检查已开 PR（`gh pr list --state open`），并确认想法契合 [docs/skill-anatomy.md](docs/skill-anatomy.md)，在 PR 描述里说明 "为什么需要"。大多数新 Skill 的想法都会和已有 Skill 或已开 PR 重叠；优先扩展已有 Skill，不要堆叠近似重复。CONTRIBUTING.md 是这套工作流的单一权威。

本仓库的 Skill 以 markdown 优先：每个 Skill 都活在 `skills/<kebab-case-name>/SKILL.md`，带 YAML frontmatter（`name`、`description`），并遵循分节解剖（Overview、When to Use、Process、Common Rationalizations、Red Flags、Verification）。只有当 Skill 自带可运行助手脚本时再加 `scripts/`；多数 Skill 只有 markdown，且不存在每 Skill 的 zip 包。

完整格式、命名约定、frontmatter 规则、辅助文件阈值与写作原则见 [docs/skill-anatomy.md](docs/skill-anatomy.md)，它是 Skill 结构的单一权威。不要在这里复述这些规则，做链接。
