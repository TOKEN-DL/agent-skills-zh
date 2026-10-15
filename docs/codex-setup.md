# 在 Codex 上使用 agent-skills

本仓库同时也是一个 [Codex 插件](https://developers.openai.com/codex/plugins/build)。Claude Code 用的根级 `skills/` 目录同样被 Codex 消费，因此无需复制或重复任何文件。

## 安装

```bash
codex plugin marketplace add addyosmani/agent-skills
codex plugin add agent-skills@agent-skills
```

> 需要 Codex CLI v0.122 或更高版本。旧版调用更早的命令 `codex marketplace add`。参见 [Codex CLI 文档](https://developers.openai.com/codex/cli)。

第一条命令把本仓库注册为 `agent-skills` 市场。第二条命令从该市场安装并启用 `agent-skills` 插件。安装后请启动新的 Codex 会话，让 Skill 被发现。

本地克隆也行：

```bash
codex plugin marketplace add /path/to/your/clone
codex plugin add agent-skills@agent-skills
```

## 用法

安装后，在 Codex 聊天中用 `@` 调用一个 Skill（例如 `@spec-driven-development`），或者直接描述任务，让 Codex 自己挑。`skills/` 下全部 25 个 Skill 都可调用。

[Codex 使用渐进式加载](https://developers.openai.com/codex/skills)：它从每个 Skill 的 `name` 与 `description` 起步，按需挑选 Skill，只有在被选中后才加载 `SKILL.md` 全文。**不要**再把 `using-agent-skills/SKILL.md` 粘进 `AGENTS.md`、系统提示词、或其他常驻上下文：那样会让本技能包的 meta-router 叠加在 Codex 原生路由之上，徒增路由开销。Meta-skill 可以随包保留；上述警告只是反对把它的全文灌进常驻上下文。

## 工作原理

- `.codex-plugin/plugin.json` ， 仓库根的 Codex 插件清单。把 `skills` 指向 `./skills/`，并提供 Codex 要求的元数据。
- `.agents/plugins/marketplace.json` ， 市场条目，把仓库根（`./`）声明为插件来源。
- `skills/<name>/SKILL.md` ， 不变。Codex 与 Claude Code 共享同一套 `name` + `description` frontmatter 格式，因此一个文件同时服务两个平台。

`.claude/commands/` 中的斜杠命令与 `agents/` 中的 persona 仍是 Claude Code 专属。`hooks/` 下的 `SessionStart` 脚本是面向无原生 Skill 路由宿主的独立小工具，两份插件都没有接它。在 Codex 上，直接调用底层 Skill（如 `@spec-driven-development`）而非斜杠命令（`/spec`）。