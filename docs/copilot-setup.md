# 在 GitHub Copilot 上使用 agent-skills

本指南覆盖 VS Code 中的 Copilot。独立的 `copilot` 命令行工具见 [copilot-cli-setup.md](copilot-cli-setup.md)。

**一次安装到底给你什么：** 就是这些 Skill。每个装好的 Skill 会变成一条按其 frontmatter `name` 命名的斜杠命令，`/spec-driven-development`、`/test-driven-development` 等等。`npx skills add addyosmani/agent-skills` 与下面的手工复制都只装 Skill。两种路径都不会复制本仓库的短生命周期包装器（`/spec`、`/plan`、`/build`、`/test`、`/review`、`/ship`），那些是 `.claude/commands/` 里的 Claude Code 命令。直接用 Skill 的完整名字，或加你自己的别名，见 [生命周期工作流](#生命周期工作流)。

> **`/spec`、`/plan`、`/build`、`/test`、`/review`、`/code-simplify`、`/webperf` 与 `/ship` 在 Copilot 中永远不会出现。** 它们是 Claude Code 命令，而 Copilot 不读 `.claude/commands/`。装好之后看不到它们是预期，不是漏步骤。改用 Skill 自己的名字，`/spec-driven-development` 对应 `/spec`、`/planning-and-task-breakdown` 对应 `/plan`，依此类推；完整映射见 [生命周期工作流](#生命周期工作流)。在 VS Code 里你可以按需添加[简短别名](#可选的短-spec-形式别名)。

## 配置

### Copilot 指令

Copilot 支持通过仓库里的 `.github/skills`、`.claude/skills` 或 `.agents/skills` 目录创建 agent Skill。

```bash
mkdir -p .github/skills/test-driven-development .github/skills/code-review-and-quality

# 为核心 Skill 创建文件
cat /path/to/agent-skills/skills/test-driven-development/SKILL.md > .github/skills/test-driven-development/SKILL.md
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md > .github/skills/code-review-and-quality/SKILL.md
```

无论走上面哪条路径，结果都是在三条项目路径之一下、每个 Skill 目录一份 `SKILL.md`。用全局 / 用户标志运行的安装会写到别处，查看它的输出确认实际路径。在 Copilot Chat 跑 `/skills` 打开 **Configure Skills** 菜单，确认发现到的内容。

更多细节参考 [Creating agent skills for GitHub Copilot](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/coding-agent/create-skills) 与 VS Code 的 [agent skills 指南](https://code.visualstudio.com/docs/agent-customization/agent-skills)。

### Agent Persona（*.agent.md）

Copilot 支持专项 agent persona。使用 agent-skills 的 agent：

> **重要：** GitHub Copilot 要求自定义 agent 文件名为 `*.agent.md`。
> 命名为 `*.md` 的文件会被 Copilot 静默忽略。
> 细节见 [VS Code 自定义 agent 文档](https://code.visualstudio.com/docs/copilot/customization/custom-agents#_custom-agent-file-structure)。

```bash
# 创建 agents 目录并复制 agent 定义
mkdir -p .github/agents
cp /path/to/agent-skills/agents/code-reviewer.md .github/agents/code-reviewer.agent.md
cp /path/to/agent-skills/agents/test-engineer.md .github/agents/test-engineer.agent.md
cp /path/to/agent-skills/agents/security-auditor.md .github/agents/security-auditor.agent.md
```

在 Copilot Chat 中调用 agent：

- `@code-reviewer Review this PR`
- `@test-engineer Analyze test coverage for this module`
- `@security-auditor Check this endpoint for vulnerabilities`

### 自定义指令（用户级）

对想跨所有仓库生效的 Skill：

1. 打开 VS Code → Settings → GitHub Copilot → Custom Instructions
2. 添加你最常用的 Skill 摘要

## 推荐配置

### .github/copilot-instructions.md

GitHub Copilot 通过 `.github/copilot-instructions.md` 支持项目级指令。

```markdown
# 项目编码标准

## 测试
- 先写测试再写代码（TDD）
- 修 bug：先写失败测试再修（Prove-It 模式）
- 测试层级：unit > integration > e2e（用能抓住行为的最低层级）
- 每次改动后跑 `npm test`

## 代码质量
- 跨五维度评审：正确性、可读性、架构、安全、性能
- 每个 PR 必须通过：lint、类型检查、测试、构建
- 代码与版本控制里没有机密

## 实现
- 以小而可验证的增量构建
- 每个增量：实现 → 测试 → 验证 → 提交
- 不要把格式改动与行为改动混在一起

## 边界
- 始终：提交前跑测试，校验用户输入
- 先问：数据库 schema 变更、新增依赖
- 永不：提交机密、删除失败测试、跳过验证
```

### 专家 Agent

在 Copilot Chat 里用各 agent 做针对性的评审流程。

## 生命周期工作流

Skill 默认是用户可调用的，因此一旦被发现，整个生命周期都在 Skill 自己的名字下可用，无需额外配置：

| 工作流 | Copilot 调用 | 备注 |
|----------|--------------------|-------|
| Define | `/spec-driven-development` | 写代码前先写一份结构化 spec |
| Plan | `/planning-and-task-breakdown` | 产出 `tasks/plan.md` 与 `tasks/todo.md` |
| Build | `/incremental-implementation` | 配 `/test-driven-development`；一次切一片 |
| Verify | `/test-driven-development` | red-green-refactor，bug 用 Prove-It |
| Review | `/code-review-and-quality` | 五维度评审 |
| Ship | `/shipping-and-launch` | 上线就绪 |

输 `/` 浏览本工作区里实际加载的内容。

自然语言是兜底，在任何 Copilot 表面都管用，不管斜杠命令有没有出现：

> Use the spec-driven-development skill to write a spec for [the feature].

> Use the code-review-and-quality skill to review my staged changes.

### 可选的短 `/spec` 形式别名

VS Code 的扩展宿主本地 agent 支持[prompt files](https://code.visualstudio.com/docs/agent-customization/prompt-files)，位于 `.github/prompts/<name>.prompt.md`，每份都变成 `/<name>` 斜杠命令。Agent Host **不使用** prompt files，在那儿用上面 Skill 名字。

别名是一条对单个 Skill 的薄快捷方式，仅此而已。这些别名并不实现本仓库 Claude Code 命令里的编排，`/build auto` 的"一次批准、整计划实施"，或 `/ship` 的并行 persona 扇出。在 Copilot 上复现这些是另起一份工作，不是单文件别名能搞定的。把 Skill 名字作默认；只有当短形式值得多一份文件时再去弄别名。

创建一条，不覆盖既有文件：

```bash
mkdir -p .github/prompts
[ -e .github/prompts/spec.prompt.md ] || cat > .github/prompts/spec.prompt.md <<'EOF'
---
description: Write a structured spec before writing code
---

Use the spec-driven-development skill.

Ask clarifying questions about the objective and target users, core features and
acceptance criteria, stack preferences and constraints, and known boundaries.
Then write a spec covering objective, commands, project structure, code style,
testing strategy, and boundaries. Save it as SPEC.md in the project root and
confirm with me before any code is written.
EOF
```

重载窗口，然后输 `/spec`。

其余命令，照搬 `mkdir`/`cat` 块、把文件名换掉（文件名*就是*命令名），并把 `description` 与正文从本表取。**替换 frontmatter 下方的整段正文**，不只是 Skill 名，SPEC.md 的 spec 正文与澄清问题指令只属于 Define，留在 `/plan` 或 `/test` 里会让它们写成 spec。

| 别名文件 | `description` | 完整正文，frontmatter 下方全部 |
|------------|---------------|--------------------------------------------------|
| `.github/prompts/plan.prompt.md` | 把已批准的 spec 拆成有序可验证的任务 | Use the planning-and-task-breakdown skill. Read the spec, then break the work into small, independently verifiable tasks, each with acceptance criteria and explicit dependency order. Save the result to `tasks/plan.md` and `tasks/todo.md`. Write no product code ， show me the plan and wait for my approval. |
| `.github/prompts/build.prompt.md` | test-first 实现下一个计划任务 | Use the incremental-implementation and test-driven-development skills. Read `tasks/plan.md` and `tasks/todo.md`, then take the next unchecked task and only that one. Write a failing test first, make it pass, refactor, run the suite, and tick the task off. Stop there and report what changed. |
| `.github/prompts/test.prompt.md` | 在满足的代码之前先写测试 | Use the test-driven-development skill. For new behavior, write a failing test that captures it before any implementation. For a bug, reproduce it with a failing test first, then fix it. Run the suite after each step and show me the red and the green output. |

自己写正文，或从本表取，不要逐字复制 `.claude/commands/*.md`：那些文件按 `agent-skills:<name>` 引用 Skill，那是 Claude Code 插件命名空间，对 Copilot 没意义。

### 如果 Skill 斜杠命令没出现

先排除预期情况：若你只是想念那些短生命周期包装器（`/spec`、`/plan`、`/build`、`/test`、`/review`、`/ship`），那没毛病，它们是 Claude Code 命令，永远不会出现在 Copilot（见顶部提示）。下面步骤适用于当 Skill 自己的名字（例如 `/spec-driven-development`）也缺失的情况。

按顺序排查，`/spec-driven-development` 缺失与 `/spec` 缺失原因不同：

1. **检查 Skill 落到了哪里。** 每个 Skill 都要有自己的目录，里面含 `SKILL.md`。本指南用的项目级路径是 `.github/skills/`、`.claude/skills/`、`.agents/skills/`，但用全局 / 用户标志运行的安装写到工作区之外（`~/.agents/skills/` 与类似位置），因此若 Skill 不在这三条里，可能只是装在了个人位置。参照 [VS Code agent skills 文档](https://code.visualstudio.com/docs/agent-customization/agent-skills) 确认实际路径，若想项目作用域就重新装到项目里。
2. **检查 frontmatter。** `name` 必须存在且合法，它*就是*斜杠命令。frontmatter 退出用户调用的 Skill 不会显示。
3. **检查 Configure Skills 菜单。** 跑 `/skills`，确认该 Skill 在这里被启用。
4. **开启新会话。** 新加的 Skill 不一定在对话中途被拾起。
5. **检查宿主。** 若 `/spec-driven-development` 工作、但你的 `/spec` 别名不工作，你大概率在 Agent Host，它不读 prompt files。
6. **检查版本。** VS Code（**Help → About**）与 Copilot Chat 扩展；Skill 作为斜杠命令是新功能，若任一落后请更新。

## 使用技巧

1. **指令保持精炼** ， Copilot 指令越聚焦越好。总结关键规则，而不是把整份 Skill 文件塞进去。
2. **用 agent 做评审** ， code-reviewer、test-engineer、security-auditor 三个 agent 为 Copilot 的 agent 模型而设计。
3. **在聊天中引用** ， 当工作进入特定阶段，把相关 Skill 内容粘贴到 Copilot Chat 里作为上下文。
4. **与 PR 评审结合** ， 配置 Copilot 用 code-reviewer agent persona 评审 PR。