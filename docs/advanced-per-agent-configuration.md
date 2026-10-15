# 高级按 Agent 配置

本仓库的 Skill 刻意保持可移植。每个 `SKILL.md` 的 frontmatter 都被限制在 [Agent Skills 规范](https://agentskills.io) 定义的字段内，因此同一个文件能在 Claude Code、Cursor、Gemini CLI、Antigravity、以及其他任何符合规范的客户端里使用。

若干 Agent 还支持额外的运行时控制：模型路由、工具限制、轮次上限、执行隔离。这些控制很有价值，但它们是供应商特定的，不属于共享的、可移植的 frontmatter。本文档说明每个 Agent 支持哪些控制、放在哪里，以及如何作为可选加入项应用它们而不破坏可移植性。

这是按 Agent 配置的规范参考，取代了过去向顶层 `SKILL.md` frontmatter 添加供应商字段的做法。`kind`、`model`、`temperature`、`max_turns`、`tools`、`context` 等字段**不得**出现在 Skill 的顶层。它们应当放在 `metadata` 下、或单独的按 Agent 适配器文件里，详见下文。

## 原则

1. **保持 `SKILL.md` 可移植。** Agent Skills 规范把顶层 frontmatter 留给 `name`、`description`、`license`、`compatibility`、`metadata`，以及实验性的 `allowed-tools`。供应商或客户端特有属性应放在 `metadata` 下，而非顶层。
2. **未知顶层字段不一定被忽略。** 许多解析器只读 `name` 与 `description`、静默丢掉其它，但严格的规范校验器或客户端可能标记或拒收未识别的顶层键。把供应商字段放进 `metadata` 让文件在所有地方都符合规范。
4. **可选加入，而非全局默认。** 运行时控制应是用户为自己的 Agent 与工作流添加的，而不是默认烤进仓库里每个 Skill 的。
4. **优先生成，而非手工编辑。** 当某项配置重复且机械时，从单一真理源生成，而不是手工维护副本。参见 [用脚本自动化](#用脚本自动化)。

## 供应商字段的位置

| 字段类型 | 例子 | 正确位置 |
|---|---|---|
| 规范字段 | `name`、`description`、`license`、`compatibility` | `SKILL.md` 的顶层 frontmatter |
| 供应商元数据 | 模型提示、路由标签 | `SKILL.md` 的 `metadata` 键下 |
| 运行时编排 | 子 Agent 定义、轮次上限、工具白名单 | 单独的按 Agent 适配器文件（见下文各 Agent） |

## Claude Code

Claude Code 读 `name` 与 `description` 用于发现。自定义命令与 Skill 共享同一套 frontmatter，因此下面的 `context` 与 `allowed-tools` 字段在 `.claude/commands/*.md` 与 `.claude/skills/*/SKILL.md` 中都能用。

- **`context: fork`** 在隔离的子 Agent 上下文中跑命令或 Skill，而不是在活跃会话中。这带来上下文隔离、可重复的干净状态执行、token 在分叉而非主会话消耗、且只有结果回到主上下文，让交接更清爽。可选的 **`agent`** 字段选择哪个子 Agent 类型运行该分叉（例如 `agent: Plan`）；默认 `general-purpose`。
- **`allowed-tools`** 是预批准工具，而非限制。被列出的工具在调用该命令或 Skill 的那一轮内无需权限提示运行；下一次发消息时授权撤销。工具池不会被移除任何东西。
- **`disallowed-tools`** 才是真正"限制"用的字段：在 Skill 激活期间把列出的工具从可用池里移除，这正是被动阶段（如评审或审计）所需的最小权限。限制也会在下一次发消息时撤销。注意这是 Claude Code 字段、不属于规范的六字段之一，因此应放在本地副本或适配器文件里；在公开发布的 `SKILL.md` 中它会以"意外键"错误导致打包失败。

这些是 Claude Code 约定。把共享的、已发布的 `SKILL.md` 限制在规范字段；把这些运行时控制放在你自己的本地命令或 Skill 副本里。

## Gemini CLI 与 Antigravity

Gemini CLI 与 Antigravity 用同一方式发现 Skill：匹配 `name` 与 `description`，再按需加载完整 Skill。运行时编排存在于单独的资源里，而不是 Skill 的 frontmatter 中。

- **子 Agent 定义** 存在于 `.gemini/agents/*.md`，与 Skill 不同。其 schema 是 `kind`（`local` 或 `remote`）、`model`、`temperature`、`max_turns`、`tools`。注意这是子 Agent 的 schema，不是 Skill 的 schema：两者不应合并进同一个 `SKILL.md`。
- **模型路由** 可以把轻量任务（格式化、文档、git 操作）送给更快模型，把更强模型留给认知密集型工作（调试、安全审计、接口设计）。
- **工具限制** 为每个子 Agent 强制最小权限，同时也从提示词里裁掉未用的工具 schema。
- **轮次上限**（`max_turns`）给执行封顶，避免失控的修正循环。

### 模型指南

Google 建议对 Gemini 3.x 模型省略 `temperature`，改用 `thinking_level`。不要在路由到 3.x 模型的 Skill 或子 Agent 上硬编码 `temperature`；优先用 `thinking_level`，并随模型演进而重新评估该指南。

```yaml
# .gemini/agents/security-auditor.md
---
kind: local
model: gemini-3-pro
thinking_level: high   # 而不是：temperature: 0.1
tools: [read_file, grep]
max_turns: 10
---
```

## 用脚本自动化

手工维护跨每个 Skill 的按 Agent 元数据既重复又容易失同步、还容易写错。更好的做法是保持单一真理源，并生成 Agent 特定的输出：

- 生成器读取按 Skill 的配置映射（每个 Skill 在给定 Agent 下该用哪个模型、工具白名单、轮次上限与隔离）。
- 对元数据型字段，它把它们注入每个 `SKILL.md` 的 `metadata` 键下，让文件保持规范合规。
- 对编排型字段，它产出 Agent 期望的独立适配器文件，例如 `.gemini/agents/*.md` 子 Agent 定义，不动 `SKILL.md`。

生成也允许工具校验**语义，而不仅是形状**：确认模型名是已知模型、工具名是真实工具、`max_turns` 是正整数、`temperature`（仍适用时）是范围内有效数字。仅形状的检查会让 `parseInt("1.5")` 或 `parseFloat("0.2oops")` 这样的非法值漏过。

## 状态

本文整合了 [#272](https://github.com/addyosmani/agent-skills/pull/272)、[#36](https://github.com/addyosmani/agent-skills/pull/36)、[#35](https://github.com/addyosmani/agent-skills/issues/35) 的讨论，三者都因本方案而关闭。这是按 Agent 配置的单一参考：保持共享 Skill 可移植，同时把每个 Agent 提供的高级控制文档化为可选加入。上面描述的脚本自动化是尚待达成共识的提案，尚未实施。