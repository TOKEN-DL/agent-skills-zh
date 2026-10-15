# 在 GitHub Copilot CLI 上使用 agent-skills

独立的 `copilot` 命令行工具会把本仓库作为插件安装，并发现 `skills/` 中的全部 Skill。对于 VS Code 里的 Copilot，请参见 [copilot-setup.md](copilot-setup.md)，配置与调用模型都不一样。

## 安装

**从本仓库的市场** ， 先注册市场，再从那里安装。`addy-agent-skills` 是本仓库声明的市场名，不是 GitHub 全局注册表，这个名字只在 `marketplace add` 之后才解析得到：

```bash
copilot plugin marketplace add addyosmani/agent-skills
copilot plugin install agent-skills@addy-agent-skills
```

**直接来自仓库**，不注册市场：

```bash
copilot plugin install addyosmani/agent-skills
```

**从本地克隆**，用于会话作用域的开发安装：

```bash
git clone https://github.com/addyosmani/agent-skills.git
copilot --plugin-dir /path/to/agent-skills
```

`--plugin-dir` 只在该会话加载插件，不持久化安装，编辑 Skill 时用它。

## 校验

```bash
copilot plugin list   # agent-skills@addy-agent-skills
copilot skill list    # 插件的 Skill，连同内建的
```

在交互式会话中，`/skills list` 显示同样的目录。

## 你能得到的，以及得不到的

仓库根的 `plugin.json` 是 Copilot CLI 读取的清单，它优先于 `.claude-plugin/plugin.json`（后者属于 Claude Code）。根清单只声明 name、version 和 description，因此组件路径回退到默认值：

- **Skill 可用。** 没有显式 path 时，CLI 用约定俗成的 `skills/` 目录，Skill 在那里被发现。
- **生命周期命令不可用。** 根清单没有 `commands` 字段，所以 `/spec`、`/plan`、`/build`、`/test`、`/review`、`/ship` 都不会注册。这些文件在 `.claude/commands/` 里，属于 Claude Code 命令。

关于清单优先级与各组件路径默认值，参见 [CLI 插件参考](https://docs.github.com/en/copilot/reference/copilot-cli-reference/cli-plugin-reference) 与 [创建插件](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/plugins-creating)。

## 用法

直接叫出 Skill 名字，或描述你的任务让 Agent 路由：

> Use the spec-driven-development skill to write a spec for [the feature].

> Use the test-driven-development skill: write a failing test for this bug first, then fix it.

> Use the code-review-and-quality skill to review my staged changes.

## 排错

| 症状 | 排查点 |
|---------|---------------|
| `plugin install` 解析不到 `agent-skills@addy-agent-skills` | 先跑 `copilot plugin marketplace add addyosmani/agent-skills`，这个名字只有在市场注册后才解析得到。也可以直接从仓库安装。 |
| 插件已装但没有 Skill | `copilot plugin list` 确认插件，再 `copilot skill list`（或会话中的 `/skills list`）看发现了什么。 |
| `/spec`、`/build` 等找不到 | 这是预期，不算装坏了：根清单没注册命令。改用名字叫 Skill 即可。 |
| 本地改了 Skill，但 CLI 显示的还是旧版 | 开新会话，或者用 `--plugin-dir` 指向你的克隆。 |