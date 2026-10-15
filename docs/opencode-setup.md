# OpenCode 配置

本指南说明如何在 OpenCode 中使用 Agent Skills。可复用的资产是 `skills/` 目录里的 Markdown skill；本仓库根的 `AGENTS.md` 文件是仓库作用域的，不应被复制到其它项目。

## 概览

OpenCode 从多个位置发现 Skill。Agent Skills 提供两种可选用法：

- **Agent 驱动的工作流：** Skill 通过内置的 `skill` 工具和你为自己项目写的项目本地 `AGENTS.md` 自动选择。
- **命令驱动的工作流：** 用 `.opencode/commands/` 手工调用生命周期命令（可选）。

## 安装

把 Skill 引入项目有两种方式：

1. 用 `skills` CLI 安装（最快）。
2. 克隆本仓库并手工复制 Skill 目录。

无论哪一步之后，都要在你自己项目里创建一份 `AGENTS.md`，以及若想要的话复制 `.opencode/commands/*.md` 文件。

### 选项 1：用 `npx skills` 安装

最快路径是开源的 [`skills` CLI](https://github.com/vercel-labs/skills)：

```bash
npx skills add addyosmani/agent-skills            # 安装所选 Skill
npx skills add addyosmani/agent-skills --list     # 安装前浏览
```

安装单个 Skill：

```bash
npx skills add addyosmani/agent-skills --skill spec-driven-development
```

默认情况下，`npx skills` 会把 Skill 装到一个工具特定的目录（通常是 `.claude/skills/` 或一个共享位置）。OpenCode 会发现放在那里的 Skill，因为它会读 `.claude/skills/<name>/SKILL.md` 与通用的 `.agents/skills/<name>/SKILL.md` 路径。

若 Skill 落在 OpenCode 不会扫描的位置，把它们复制或软链到下面列出的发现路径之一，例如：

```bash
mkdir -p .opencode/skills
cp -r .claude/skills/<skill-name> .opencode/skills/
```

> **注意：** 按 Skill 安装只会复制 Skill 目录本身。若某 Skill 引用 `references/` 下的共享文件，请把那些文件也复制进已安装的 Skill 目录、或安装整包。背景见 [#361](https://github.com/addyosmani/agent-skills/issues/361)。

### 选项 2：克隆本仓库

1. 克隆仓库：

```bash
git clone https://github.com/addyosmani/agent-skills.git
```

2. 把所需的 Skill 复制到 OpenCode Skill 发现路径之一。

#### 项目本地安装

```bash
mkdir -p .opencode/skills
cp -r /path/to/agent-skills/skills/<skill-name> .opencode/skills/
```

例如安装 `spec-driven-development` 与 `incremental-implementation`：

```bash
mkdir -p .opencode/skills
cp -r /path/to/agent-skills/skills/spec-driven-development .opencode/skills/
cp -r /path/to/agent-skills/skills/incremental-implementation .opencode/skills/
```

#### 全局安装

```bash
mkdir -p ~/.config/opencode/skills
cp -r /path/to/agent-skills/skills/<skill-name> ~/.config/opencode/skills/
```

#### 跨兼容路径

OpenCode 也会发现放在 Claude 兼容或通用 agent 路径下的 Skill：

- `.claude/skills/<name>/SKILL.md`
- `~/.claude/skills/<name>/SKILL.md`
- `.agents/skills/<name>/SKILL.md`
- `~/.agents/skills/<name>/SKILL.md`

若你已经在 Claude Code 与 OpenCode 之间共享 Skill，上面任何位置都管用。

### 要复制什么

复制 `skills/` 下的目录（例如 `skills/spec-driven-development/`）。每个目录必须包含 `SKILL.md` 文件。不要复制仓库根的 `AGENTS.md` 或 `CLAUDE.md`；那两份文件配置的是本仓库自身的开发。

## 项目 `AGENTS.md`

在你的**项目根**创建 `AGENTS.md`。这是告诉 OpenCode 何时、如何调用已安装 Skill 的系统提示词。不同于仓库作用域的 `AGENTS.md`（位于 `addyosmani/agent-skills` 内），它属于你的项目本体，应适配你的技术栈。

下面是一份可粘贴到你项目 `AGENTS.md` 的模板：

```markdown
# Agent Skills (OpenCode)

This project uses skills installed under `.opencode/skills/` (or a compatible path).

## Core Rules

- If a task matches a skill, invoke it with the `skill` tool before acting.
- Skills are located in `.opencode/skills/<skill-name>/SKILL.md`.
- Follow the skill workflow strictly; do not partially apply it.
- Never skip required steps such as spec, plan, or test when a skill demands them.

## Intent → Skill Mapping

Map the user's intent to the matching skill automatically:

- Feature / new functionality → `spec-driven-development`, then `incremental-implementation` and `test-driven-development`
- Planning / breakdown → `planning-and-task-breakdown`
- Bug / failure / unexpected behavior → `debugging-and-error-recovery`
- Code review → `code-review-and-quality`
- Refactoring / simplification → `code-simplification`
- API or interface design → `api-and-interface-design`
- UI work → `frontend-ui-engineering`

## Execution Model

For every request:

1. Determine if any skill applies (even a small chance).
2. Load the skill with `skill({ name: "<skill-name>" })`.
3. Follow the skill workflow exactly.
4. Only proceed to implementation once required steps are complete.
```

把它存为 `AGENTS.md` 放进你项目根。OpenCode 会自动加载。

> **注意：** `addyosmani/agent-skills` 仓库根的 `AGENTS.md` 是为在本仓库上工作的贡献者准备的，不应复制到其他项目。见 [CONTRIBUTING.md](../CONTRIBUTING.md#repo-scoped-files)。

## 工作原理

### 1. Skill 发现

OpenCode 依次走下列路径（先项目本地，后全局）：

- `.opencode/skills/<name>/SKILL.md`
- `~/.config/opencode/skills/<name>/SKILL.md`
- `.claude/skills/<name>/SKILL.md`
- `~/.claude/skills/<name>/SKILL.md`
- `.agents/skills/<name>/SKILL.md`
- `~/.agents/skills/<name>/SKILL.md`

每个 Skill 的 frontmatter 必须包含合法的 `name` 与 `description`。

### 2. 自动调用 Skill

当项目的 `AGENTS.md` 指示 Agent 使用 Skill 时，Agent 会评估每个请求并把它映射到合适的 Skill。

示例：

- "build a feature" → `incremental-implementation` + `test-driven-development`
- "design a system" → `spec-driven-development`
- "fix a bug" → `debugging-and-error-recovery`
- "review this code" → `code-review-and-quality`

### 3. 生命周期映射（隐式命令）

OpenCode 不强制要求斜杠命令；若你更喜欢斜杠命令，见下一节。在 Agent 驱动模式下，生命周期是隐式映射的：

- DEFINE → `spec-driven-development`
- PLAN → `planning-and-task-breakdown`
- BUILD → `incremental-implementation` + `test-driven-development`
- VERIFY → `debugging-and-error-recovery`
- REVIEW → `code-review-and-quality`
- SHIP → `shipping-and-launch`

### 复制可选的斜杠命令

若你更喜欢显式命令，把本仓库里的示例命令文件复制到你的项目里，并改成调用你安装的 Skill：

```bash
mkdir -p .opencode/commands
cp /path/to/agent-skills/.opencode/commands/*.md .opencode/commands/
```

> **注意：** 本仓库目前没有把 `.opencode/commands/*.md` 合并到 `main` 分支。你可创建自己的命令文件，或关注 PR #200，提案就是添加它们。文件到位后，上述模式适用。

一个典型的命令文件长这样：

```markdown
---
description: Break work into small verifiable tasks
---

Invoke the planning-and-task-breakdown skill. Read the spec and create tasks with acceptance criteria.
```

存为 `.opencode/commands/plan.md`，即可在 OpenCode 中启用 `/plan`。

## 用法示例

### 示例 1：特性开发

用户：
```
Add authentication to this app
```

Agent 行为：
- 检测到特性工作
- 调用 `spec-driven-development`
- 在写代码前产出 spec
- 转入规划与实现 Skill

### 示例 2：修 Bug

用户：
```
This endpoint is returning 500 errors
```

Agent 行为：
- 调用 `debugging-and-error-recovery`
- 复现 → 定位 → 修复 → 加防御

### 示例 3：代码评审

用户：
```
Review this PR
```

Agent 行为：
- 调用 `code-review-and-quality`
- 应用结构化评审（正确性、设计、可读性等）

## Agent 预期

为了让 OpenCode 正确工作，Agent 应：

- 在动手前始终检查是否有 Skill 适用
- 在适用时用 `skill` 工具加载 Skill
- 绝不跳过必需的工作流（spec、plan、test 等）
- 不要直接跳到实现

这些规则由你项目的 `AGENTS.md` 强制，而不是由 Skill 副本本身。

## 限制

- Skill 调用依赖模型是否守规。
- OpenCode 不会自动安装 Skill；按需复制或安装目录。
- 若某 Skill 引用 `references/` 下的文件，手工安装时你可能也需要复制那些。

## 小结

1. 装好你需要的 Skill，要么用 `npx skills add addyosmani/agent-skills`，要么从仓库克隆复制到 `.opencode/skills/`（项目）、`~/.config/opencode/skills/`（全局），或 `.claude/skills/` / `.agents/skills/` 这类跨兼容路径。
2. 创建你自己的项目本地 `AGENTS.md`，写上上述规则与意图映射。
3. OpenCode 发现 Skill，并由你的 `AGENTS.md` 引导 Agent 调用它们。
4. 可选添加 `.opencode/commands/*.md` 提供显式斜杠命令。

这让可复用资产（Skill）与仓库专属配置（`addyosmani/agent-skills` 根的 `AGENTS.md`）保持分离。