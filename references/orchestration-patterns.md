# 编排模式（Orchestration Patterns）

本仓库背书的 Agent 编排模式目录，以及需要规避的反模式。在新增协调多 persona 的斜杠命令之前，或在引入"包裹"已有 persona 的新 persona 之前，请先阅读本参考。

治理规则：**用户（或斜杠命令）才是编排者。Persona 不会调用其他 persona。Skill 是 persona 工作流里的强制跳点。**

---

## 背书的模式

### 1. 直接调用（无编排）

单一 persona、单一视角、单一工件。默认且成本最低的选项。

```
user → code-reviewer → report → user
```

**适用场景：** 工作是同一工件上的同一视角，并且你能用一句话讲清。

**示例：**
- "Review this PR" → `code-reviewer`
- "Find security issues in `auth.ts`" → `security-auditor`
- "What tests are missing for the checkout flow?" → `test-engineer`

**成本：** 一轮往返。你应该一直把它拿来与编排模式对比的基线。

---

### 2. 单一 persona 斜杠命令

用项目 Skill 包裹一个 persona 的斜杠命令。免去用户每次重复解释工作流。

```
/review → code-reviewer（加载 code-review-and-quality skill）→ report
```

**适用场景：** 同一个单一 persona 调用反复出现，且配置相同。

**本仓库示例：** `/review`、`/test`、`/code-simplify`。

**成本：** 与直接调用相同。斜杠命令只是"保存起来的 prompt"。

**反向信号：** 若斜杠命令的主体内容主要是"决定调用哪个 persona"，请直接删除它，让用户直接调用 persona。

---

### 3. 并行扇出 + 合并

多个 persona 并行处理同一输入，各自产出独立报告。在主 Agent 上下文里的合并步骤把它们综合成单一决策。

```
                    ┌─→ code-reviewer    ─┐
/ship → 扇出  ───┼─→ security-auditor ─┤→ 合并 → go/no-go + 回滚
                    └─→ test-engineer    ─┘
```

**适用场景：**
- 子任务真正独立（无共享可变状态、无顺序依赖）
- 每个子 Agent 受益于自己的上下文窗口
- 合并步骤足够小、能放进主上下文
- 时钟延迟（wall-clock latency）重要

**本仓库示例：** `/ship`。

**成本：** N 个并行子 Agent 上下文 + 一次合并轮次。比直接调用高，但时钟更快；因为每个子 Agent 仍专注于自己的单一视角，报告质量也更好。

**采用此模式前的验证清单：**
- [ ] 我能否让所有子 Agent 同时跑、而没有顺序问题？
- [ ] 每个 persona 产出的是不同*种类*的发现，而不是同一发现的另一种角度？
- [ ] 合并步骤能放进主 Agent 剩余的上下文窗口？
- [ ] 用户等待时间足够长，并行效果可被察觉？

若任一项答案为"否"，回退到直接调用或单一 persona 命令。

---

### 4. 由用户驱动的顺序串行命令

用户按既定顺序运行斜杠命令，跨步骤携带上下文（或 commit 历史）。没有编排 Agent，用户本人就是编排者。

```
user runs:  /spec  →  /plan  →  /build  →  /test  →  /review  →  /ship
```

**适用场景：** 工作流存在依赖（每一步需要上一步输出），且步骤之间需要人类判断。

**本仓库示例：** 整个 DEFINE → PLAN → BUILD → VERIFY → REVIEW → SHIP 生命周期。

**成本：** 每一步一个子 Agent 上下文。编排层零成本，因为没有编排 Agent。

**为何不自动化它：** 一个 LLM "生命周期编排者"会（a）因要为下一步做摘要而损失步骤之间的细微信息，（b）跳过在错误方向上能早期拦住的人工检查点，（c）通过额外转述轮次让 token 成本翻倍。

---

### 5. 研究隔离（上下文保留）

当任务需要阅读大量资料、且这些资料不应污染主上下文时，派生一个研究子 Agent，只回吐关于归类的精简摘要。

```
主 Agent → 研究子 Agent（读 50 个文件）→ 摘要 → 主 Agent 继续
```

**适用场景：**
- 主会话需要保持专注于下游任务
- 调查结果远小于其消费的输入
- 决策质量需要主 Agent 在之后仍有空间思考

**示例：** "Find every call site of this deprecated API across the monorepo"、"Summarize what these 30 ADRs say about caching"。

**成本：** 一个隔离的子 Agent 上下文。只要替代方案是把成百个文件塞进主上下文，这笔账就划算。

**在 Claude Code 上，使用内置 `Explore` 子 Agent**，而不是自定义一个研究 persona。`Explore` 跑在 Haiku 上、写 / 编辑工具被禁用，并且就是为此模式而生。仅当 `Explore` 不适用时才自定义研究子 Agent（例如你需要模型无法自行推断的领域特定系统提示）。

---

## Claude Code 兼容性

本目录与宿主（harness）无关，但多数读者会在 Claude Code 上跑它。下面是每个模式映射到 Claude Code 原语的方式，以及平台在哪些地方替我们执行了规则。

### Persona 放在哪

插件子 Agent 放在插件根的 `agents/` 下。本仓库是一个插件（`.claude-plugin/plugin.json`），因此 `agents/code-reviewer.md`、`agents/security-auditor.md`、`agents/test-engineer.md` 在插件启用时会被自动发现。无需路径配置。

### 子 Agent vs. Agent Teams

Claude Code 有两种并行原语。模式 3（并行扇出 + 合并）映射到**子 Agent（subagents）**。若需要队友彼此能相互沟通，改用**Agent Teams**。

| | 子 Agent | Agent Teams |
|--|-----------|-------------|
| 协调方式 | 主 Agent 扇出，子 Agent 只回报 | 队友彼此消息互通，共享任务列表 |
| 上下文 | 每个子 Agent 拥有自己的上下文窗口 | 每个队友拥有自己的上下文窗口 |
| 适用场景 | 独立任务产出报告 | 需要协作讨论的工作 |
| 状态 | 稳定 | 实验性，需 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` |
| 成本 | 较低 | 较高，每个队友都是一个独立的 Claude 实例 |

**本仓库的 persona 在两种模式下都能工作。** 当作为子 Agent 被派生（例如被 `/ship`）时，它们向主会话回报。当作为队友被派生（`Spawn a teammate using the security-auditor agent type...`）时，它们能彼此挑战对方的发现。Persona 定义本身不变，只有派生上下文变了。

一个细节：persona 中的 `skills` 与 `mcpServers` frontmatter 字段在它作为子 Agent 时生效，但**作为队友时会被忽略**，队友会从项目和用户设置加载 Skill 与 MCP 服务器，与普通会话一致。若某 persona 依赖某个特定 Skill 或 MCP 服务器，要在会话级把它配置好，使两种模式下都可用。

### 平台强制执行的规则

目录里两条规则不仅是约定，Claude Code 会强制执行：

- **"Subagents cannot spawn other subagents"**（子 Agent 不能派生其他子 Agent，原文出自官方文档）。反模式 B（persona 调用 persona）与反模式 D（深层 persona 树）在 Claude Code 上结构上无法存在。
- **"No nested teams"**（团队不能嵌套），队友不能派生自己的团队。同样在团队层面挡住上述反模式。

这意味着你可以放心采用本目录中的模式，不必担心贡献者不小心搭出反模式，它们根本加载不起来。

### 需要了解的内置子 Agent

在自定义子 Agent 前，先检查下面这些是否覆盖了该角色：

| 内置 | 用途 |
|----------|---------|
| `Explore` | 只读代码库搜索与分析。用于模式 5（研究隔离）。 |
| `Plan` | 计划模式期间的只读研究。 |
| `general-purpose` | 需要探索与修改能力的多步任务。 |

不要重新定义这些。把你的专家 persona（code-reviewer、security-auditor、test-engineer）叠在它们之上。

### 插件 Agent 的 frontmatter 限制

插件子 Agent **不支持** `hooks`、`mcpServers`、`permissionMode` frontmatter 字段，这些会被静默忽略。若未来某个 persona 需要其中之一，请用户把文件复制到 `.claude/agents/` 或 `~/.claude/agents/` 即可。

在插件 Agent 中**确实生效**的字段：`name`、`description`、`tools`、`disallowedTools`、`model`、`maxTurns`、`skills`、`memory`、`background`、`effort`、`isolation`、`color`、`initialPrompt`。若想按 persona 优化（例如 `test-engineer` 覆盖率扫描用 Haiku、`code-reviewer` 用 Sonnet、`security-auditor` 用 Opus），可以为每个 persona 设置 `model`。

### 并行派生多个子 Agent

在 Claude Code 上，并行扇出（模式 3）需要在**同一条 assistant 轮次中发出多个 Agent 工具调用**。串行轮次会让执行也串行。`/ship` 显式写明了这点。任何新的编排命令都应当如此。

---

## 实战案例：用 Agent Teams 做竞争假设调试

这个例子展示何时**伸手拿 Agent Teams** 而不是 `/ship` 的子 Agent 扇出。两种模式远看很像，都派生同一个三人组，但价值来源完全不同。

### 场景

> *Checkout 偶尔会卡住约 30 秒后再完成。大约每 50 次会话出现一次。日志里没有错误。上周发版后开始。*

合理的根因（互斥，且都符合症状）：

1. 新增的支付确认流程里有竞态
2. 一个鉴权检查偶尔落到一次慢的同步网络调用
3. 某条随购物车大小放大的查询缺索引
4. 一个不稳定的第三方 API，SDK 在超时前静默重试

单一 Agent 会挑第一个看似合理的理论就停手。`/ship` 风格的子 Agent 扇出只会让每个 persona 独立报告，但报告彼此不见面，因此没办法排除错误的理论。

这正是 Agent Teams 文档描述的场景：*"With multiple independent investigators actively trying to disprove each other, the theory that survives is much more likely to be the actual root cause."*（多个独立调查者主动试图推翻彼此，能存活的理论才更可能是真实根因。）

### 为何这不是 `/ship` 的活

| | `/ship`（子 Agent） | Agent Teams |
|--|--------------------|-------------|
| 子 Agent 看到 | 同一份 diff、不同的视角 | 共享任务列表、彼此的消息 |
| 输出 | 三份独立报告 → 一次合并 | 对抗式辩论 → 共识根因 |
| 适用 | 你想要对已知工件下一份裁决 | 你想在假设之中*找出*真正的工件 |

`/ship` 是裁决；Agent Teams 是调查。

### 一次性环境配置

Agent Teams 是实验性特性。在 `~/.claude/settings.json`：

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

需要 Claude Code v2.1.32 或更高版本。本仓库的 persona 会被自动加载，不必手写 team-config 文件。

### 触发 prompt

在主会话里以自然语言输入：

```
Users report checkout checkout 间歇性挂起 ~30 秒，上周发版后开始，日志没有错误。

Create an agent team to debug this with competing hypotheses. Spawn
three teammates using the existing agent types:

  - code-reviewer  ， investigate race conditions and blocking calls
                     in the checkout code path
  - security-auditor ， investigate auth checks, session handling,
                       and any synchronous network calls added recently
  - test-engineer  ， propose tests that would distinguish between the
                     hypotheses and check coverage gaps in checkout

Have them message each other directly to challenge each other's
theories. Update findings as consensus emerges. Only converge when
two teammates agree they can disprove the others'.
```

主会话派生三名队友，引用既有 persona 名。Persona 主体作为附加指令**追加**到每位队友的系统提示词（在主会话安装的团队协调指令之上）；上面的 prompt 是它们的任务。

### 流程

1. 每位队友在自己的上下文窗口里、以自己的视角探查代码库。
2. 队友用 `message` 互相直接发送发现。主会话不必中转。
3. 共享任务列表显示谁在调查什么，随时可见（in-process 模式下用 `Ctrl+T`，split 模式下在 tmux 面板）。
4. 当 `code-reviewer` 发现一个本应是串行的 `Promise.all`，它向 `security-auditor` 发消息确认那次鉴权调用不在竞态里。`security-auditor` 检查并回复，要么确认竞态就是真实问题，要么给出反证。
5. `test-engineer` 针对正在胜出的理论提出一个聚焦的集成测试，团队在宣布共识前用它验证。
6. 主会话综合收敛后的发现并向你呈现。

你可以随时通过 `Shift+Down` 切换到某个队友并输入，用来把走错路的调查者拉回正轨。

### 何时清理

当调查落到根因时，告诉主会话：

```
Clean up the team
```

始终通过主会话清理，而不是由某个队友清理（依据文档：队友缺乏完整的团队上下文来做清理）。

### 成本预期

三名 Sonnet 队友跑大约 10–15 分钟调查，其成本显著高于同样这三名 persona 由 `/ship` 派生为子 Agent。理由是*结论质量*，对于修错代价昂贵的生产环境调试，多花的 token 是划算的。对于例行 PR review，留在 `/ship`。

### 本场景中的反模式

**不要**把它改造成一个扇出子 Agent 的 `/debug` 斜杠命令。子 Agent 不能彼此发消息，你会丢掉让该模式起作用的对抗式辩论。若某工作流反复出现，把上面的触发 prompt 存为片段，而不要包成错误使用子 Agent 的斜杠命令。

### 何时**不**用 Agent Teams

- 对已知 diff 的可投产裁决 → 用 `/ship`（子 Agent）。
- 对单一工件的单一专家视角 → 直接调用 persona。
- 顺序生命周期（spec → plan → build）→ 用户驱动的斜杠命令（模式 4）。
- 读多产少的研究 → 内置 `Explore` 子 Agent。

只在队友**必须**互相挑战才能得到正确答案时，伸手用 Agent Teams。

---

## 反模式

### A. 路由 persona（"元编排者"）

负责决定调用哪个 persona 的 persona。

```
/work → router-persona → "this needs a review" → code-reviewer → router（转述）→ user
```

**为何失败：**
- 纯路由层、没有任何领域价值
- 增加两轮转述跳点 → 信息丢失 + 大约 2× token 成本
- 用户本就想要 review；他们本可以直接调 `/review`
- 重复了斜杠命令与 `AGENTS.md` 中的意图映射已经做的事

**该怎么做：** 新增或打磨斜杠命令。在 `AGENTS.md` 中记录意图 → 命令的映射。

---

### B. 调用其他 persona 的 persona

一个 `code-reviewer` 在内部发现鉴权代码时调用 `security-auditor`。

**为何失败：**
- Persona 设计为产出单一视角；把它们串起来就废掉了这点
- 调用 persona 所传的摘要会丢掉被调 persona 需要的上下文
- 失败模式叠加（谁的输出格式胜出？规则属于谁？）
- 把成本藏起来

**该怎么做：** 让调用 persona 在它的报告里*建议*一次后续审计。由用户或斜杠命令运行第二遍。

---

### C. 转述的顺序编排者

一个 Agent 替用户依次调 `/spec`、`/plan`、`/build` 等。

**为何失败：**
- 丢失在错误方向上能早期拦住的人工检查点
- 每次交接都是上下文摘要，长流水线上的累积漂移
- Token 成本翻倍：每步多一轮编排者 + 子 Agent
- 在判断最重要的节点剥夺了用户主动权

**该怎么做：** 让用户保留编排者身份。在 `README.md` 中写明推荐顺序，由用户自己调用。

---

### D. 深层 persona 树

`/ship` 调用 `pre-ship-coordinator`，后者再调 `quality-coordinator`，后者才调 `code-reviewer`。

**为何失败：**
- 每一层只增加延迟与 token，没有任何决策价值
- 调试变成多层调查
- 叶子 persona 因多轮摘要丢掉了上下文

**该怎么做：** 把编排深度控制在最多 1 层（斜杠命令 → persona）。合并动作在主 Agent 完成。

---

## 决策流

当考虑一个新的编排工作流时，按本流程走：

```
工作是对单一工件的单一视角吗？
├── 是 → 直接调用。停。
└── 否  → 同样的组合会重复出现吗？
         ├── 否  → 直接调用，临时搭。停。
         └── 是 → 子任务独立吗？
                  ├── 否  → 由用户跑的顺序斜杠命令（模式 4）。
                  └── 是 → 并行扇出 + 合并（模式 3）。
                           对照上面清单验证。
                           若任一项未过 → 回退到单一 persona 命令（模式 2）。
```

---

## 何时向本目录新增模式

仅当满足以下条件后再新增条目：

1. 你在真实工作中至少用过该模式两次
2. 你能指出本仓库里一个具体工件作为示范
3. 你能解释为什么既有模式无法胜任
4. 你能描述它的反模式阴影（人们会错误搭出的样子）

过早的目录条目只会变成"没人会照做"的愿望清单。