# 贡献 Agent Skills

感谢你有兴趣贡献！本项目是一组面向 AI 编程 Agent 的生产级工程 Skill。

新同学请从 [docs/developer-onboarding.md](docs/developer-onboarding.md) 开始。这是一份带导游的入门路线：理解五层结构、验证回路、各类贡献路径，并告诉你应该在什么时候读本文件、[skill-anatomy.md](docs/skill-anatomy.md) 和 [evals/README.md](evals/README.md)。本文件是权威规则手册，入门指南是地图。

## 新增一个 Skill

### 在提案一个新 Skill 之前

本套件已经覆盖了大多数开发生命周期，很多提案都会与已有 Skill 或其他开放 PR 重叠。打开之前先做这些检查，避免评审辛苦地挑重复：

1. **搜索目录。** 浏览 [README 中的 Skill 列表](README.md)，扫一遍 `skills/`，看是否已有 Skill 部分或全部覆盖了你的想法。
2. **查看已开 PR。** 跑 `gh pr list --state open`（或在 PR 标签页浏览），找是否有同类提案。已有成簇的近似重复 Skill；别再加。
3. **查看被驳回的提案。** 在 [Skill 改动的驳回账本](evals/skill-impact.md) 中搜索早期与你的想法重叠的提案，并查看它们的评测证据，避免重复劳动。
4. **读格式规范。** 确认你的想法契合 [docs/skill-anatomy.md](docs/skill-anatomy.md) 的格式，即带验证的可执行工作流，而不是空泛建议。
5. **在 PR 描述里说明 "为什么需要"。** 直接说明本提案为什么没有被现有 Skill、开放 PR 或之前被驳回的提案覆盖。如果重叠了，请提议扩展现有 Skill，而不是新增。
6. **去掉模型特定的变通。** 如果某一步无法在不点名模型、模型版本或某个 Agent 私有名的情况下描述，它就不该出现在 Skill 里，描述能力本身就好（参见 [写流程、不写变通](docs/skill-anatomy.md#write-the-procedure-not-the-workaround)）。

如果你的想法是对现有 Skill 的细修，优先定向编辑那个 Skill，而不是新建目录。

### 创建 Skill

1. 在 `skills/` 下创建一个目录，名字是 kebab-case
2. 添加一份遵循 [docs/skill-anatomy.md](docs/skill-anatomy.md) 格式的 `SKILL.md`
3. 包括 YAML frontmatter 含 `name` 和 `description` 字段；顶层不要写 vendor 或 runtime 相关键（那些应该放到 `metadata` 或 per-agent 适配文件中；详见 [docs/advanced-per-agent-configuration.md](docs/advanced-per-agent-configuration.md)）
4. `description` 以 "这个 Skill 做什么"（第三人称）打头，紧接着是一或多个 `Use when` 触发条件

### Skill 质量标准

Skill 应当：

- **具体**：可执行步骤，不是空泛建议
- **可验证**：明确的退出条件和证据要求
- **经过实战检验**：基于真实的工程工作流，不是理论上的"理想态"
- **最小**：只保留真正能引导 Agent 走对的内容

### 结构

每个新 Skill 必须有：

- Skill 目录下的 `SKILL.md`
- 含合法 `name` 与 `description` 的 YAML frontmatter
- 一份位于 `evals/cases/<skill-name>.json` 的评测用例，至少 3 个正向触发、2 个负向触发（尽量带 `owner`），以及 1 个行为评测。执行类评测必须有 `evals/fixtures/` 下的真实文件支撑；偏对话形态的 Skill 可以使用评审员把关的 `kind: "dialogue"` 评测代替（见 [evals/README.md](evals/README.md)）。CI 强制这些要求。

新 Skill 通常应遵循标准解剖：

- **Overview**：这个 Skill 做什么以及为什么重要
- **When to Use**：触发条件
- **Process**：一步步工作流
- **Common Rationalizations**：Agent 跳过步骤时的借口，附反驳
- **Red Flags**：Skill 被错误应用时的信号
- **Verification**：怎么确认 Skill 已正确应用

上面的 frontmatter 字段是必需的。节级解剖是建议的范式，等价标题如 `How It Works`、`Workflow`、`Core Process` 也行，只要保持同一意图并便于阅读。

### 不应该做的事

- 不要在多个 Skill 之间复制内容；互相引用就好
- 不要新增写空泛建议而不是可执行流程的 Skill
- 辅助文件不要过早拆分；只有内容超过 100 行才拆
- 不要为了跟其它 Skill 一致而新建空的 `scripts/` 目录；只在 Skill 确实带可运行助手脚本时才加 `scripts/`
- 不要把 reference 资料放进 Skill 目录；要放到 `references/`

## 修改现有 Skill

提出改动之前，先搜索 [Skill 改动驳回账本](evals/skill-impact.md)，看是否有针对同一 Skill 的早期尝试，并查看它们的评测证据。

- 改动保持聚焦且最小
- 保留原有结构和语气
- 编辑后确认 YAML frontmatter 仍然合法

如果一个 Skill 或 description 的修改基于评测被驳回，那么在账本上加一行：日期、受影响 Skill、精简改动描述、改前/改后的 rank-1 分数、被驳回 PR 链接与结论。把这次只更新账本的提交单独落在默认分支上，不要只留在被驳回提案分支，因为关闭或强制推送提案会把记录丢掉。

## 添加宿主指南

README 中的安装小节只列维护者已安装并跑过当前版本的宿主。其余宿主在 [docs/other-hosts.md](docs/other-hosts.md) 用一行介绍：宿主名与安装命令。不另起专页、不放厂商链接、不放 Logo、表情符、截图。

- 如果你就职于该宿主背后的公司，请在 PR 中说明
- 一个宿主只有当有维护者跑通并愿意持续维护小节时，才迁入 README
- 宿主指南按开启 PR 的顺序评审

## 仓库级文件

仓库根的 `AGENTS.md` 与 `CLAUDE.md` 是给在 [`addyosmani/agent-skills`](https://github.com/addyosmani/agent-skills) 仓库本身工作的 Agent 用的。写安装指南或文档时，不要让用户把这些文件复制到自己的项目或全局 Agent 配置里；可复用的资产是 `skills/`。

## 翻译

> 以下为**上游原文**：「We don't accept translations of the documentation (README, `docs/`) or of skills and their content. Translated copies drift out of sync as skills and docs evolve, and we have no way to maintain them long-term without leaning on agent translations plus community corrections, which adds maintenance cost for limited value. Keep all skills, docs, and contributions in English.」

我们不接受对文档（README、`docs/`）或 Skill 及其内容的翻译合并。翻译副本会随 Skill 与文档的演化而漂移，我们无法在不依赖 Agent 翻译 + 社区校正的前提下长期维护，代价高而价值有限。所有 Skill、文档、贡献请保持英文。

## 测试钩子

会话启动脚本（`hooks/session-start.sh`）在接入宿主 `SessionStart` 钩子时会注入 `using-agent-skills` 元 Skill。Claude Code 插件不注册它，Claude Code 原生路由 Skill，常驻注入会让同一个任务出现两个路由（参见 [docs/getting-started.md](docs/getting-started.md)）；该脚本保留给没有原生 Skill 路由的宿主使用。`hooks/session-start-test.sh` 提供回归测试，验证脚本产生的 JSON 负载，分别在有 `jq` 和没有 `jq` 的环境下都验证。

在打开以下相关 PR 之前先跑：

- `hooks/session-start.sh`
- `skills/using-agent-skills/SKILL.md`（钩子嵌入的元 Skill 内容）

```bash
bash hooks/session-start-test.sh
```

期望输出：`session-start JSON payload OK`。脚本在断言失败时退出非零。

### 复现 no-jq 分支

钩子在 `jq` 不在 `PATH` 上时仍输出相同的 `hookSpecificOutput` 信封，由 `additionalContext` 解释 "需要 jq"。本地复现时，把 `jq` 所在目录从 `PATH` 中剥离再跑测试：

```bash
JQ_DIR=$(dirname "$(command -v jq)")
PATH=$(echo "$PATH" | tr ':' '\n' | grep -v "^${JQ_DIR}$" | tr '\n' ':' | sed 's/:$//') \
  bash hooks/session-start-test.sh
```

当 `jq` 单独放在一个目录时（如 Homebrew 的 `/opt/homebrew/bin`、手动装的 `/usr/local/bin`），上面最干净。如果 `jq` 与测试要依赖的工具（如 `/usr/bin/mktemp`）共享系统 bin，更简单的办法是用单独的包管理器安装一份 `jq` 让它有自己的 bin 目录，然后重跑。

钩子内的 `command -v jq` 在剥离 `PATH` 的环境下会失败，进入缺 jq 兜底分支，测试断言 `additionalContext` 里的 "jq is required" 提示，而不是元 Skill 正文。

## 报告 Issue

如果你遇到以下情况，请开 Issue：

- Skill 给出了错误或过时的指导
- 常见工程工作流没有被覆盖
- Skill 之间存在不一致

如果某个 Skill 的指导错了、过时了，或在你的项目里完全不适用（例如它在 Maven / Gradle 仓库里假定 `npm test`），请使用 [Skill gap](https://github.com/addyosmani/agent-skills/issues/new?template=skill-gap.yml) 模板。它会要你填写受影响的 Skill、相关节选、项目上下文、以及你替代做了什么，足够让维护者不必看自由发挥的报告就能分流。

## 许可证

提交贡献即表示你同意你的贡献以 MIT 协议授权。
