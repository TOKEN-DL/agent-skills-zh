# Agent 角色（Personas）

扮演单一角色、单一视角的专家 persona。每个 persona 都是一个 Markdown 文件，被宿主（Claude Code、Cursor、Copilot 等）当作系统提示词读入。

| Persona | 角色 | 适用场景 |
|---------|------|----------|
| [code-reviewer](../agents/code-reviewer.md) | 高级主任工程师 | 合并前的五维度评审 |
| [security-auditor](../agents/security-auditor.md) | 安全工程师 | 漏洞检测、OWASP 式审计 |
| [test-engineer](../agents/test-engineer.md) | QA 工程师 | 测试策略、覆盖分析、Prove-It 模式 |
| [web-performance-auditor](../agents/web-performance-auditor.md) | Web 性能工程师 | Core Web Vitals 审计、加载 / 渲染 / 网络分析 |

## Persona 与 Skill、命令的关系

三层，各司其职：

| 层 | 是什么 | 例子 | 组合中的定位 |
|-------|-----------|---------|------------------|
| **Skill** | 带步骤与退出标准的工作流 | `code-review-and-quality` | *怎么用*，在 persona 或命令内部调用 |
| **Persona** | 带有视角与输出格式的角色 | `code-reviewer` | *谁来做*，采用视角、产出报告 |
| **命令** | 用户面入口 | `/review`、`/ship` | *什么时候用*，组合 persona 与 skill |

用户（或斜杠命令）才是编排者。**Persona 不会调用其他 persona。** Skill 是 persona 工作流内的强制跳点。

## 何时用哪个

### 直接调用 persona

当你想要当前改动的单一视角、且用户在回路中时选这个。

- "Review this PR" → 直接调用 `code-reviewer`
- "Are there security issues in `auth.ts`?" → 直接调用 `security-auditor`
- "What tests are missing for the checkout flow?" → 直接调用 `test-engineer`
- "Audit Core Web Vitals on the product page" → 直接调用 `web-performance-auditor`

### 斜杠命令（背后是单一 persona）

当有一个你会反复使用、否则每次都要重述的工作流时选这个。

- `/webperf` → 包装 `web-performance-auditor`，用于 Web 应用的性能专项审计

`/review` 与 `/test` 不是 persona 的包装器。它们在主会话（对话的上下文所在之处）跑 `code-review-and-quality` 与 `test-driven-development` Skill。要让 `code-reviewer` 或 `test-engineer` 在全新上下文中跑，直接调用 persona，或跑 `/ship`。

### 斜杠命令（编排器，扇出）

仅当**独立**调查可以并行跑、产出报告、再由一个 Agent 合并时，才选这个。

- `/ship` → 并行扇出 `code-reviewer` + `security-auditor` + `test-engineer`，再把它们各自的报告综合成 go/no-go 决策

这是本仓库背书的唯一编排模式。完整模式目录与反模式见 [references/orchestration-patterns.md](../references/orchestration-patterns.md)。

## 决策矩阵

```
工作是对单一工件单一视角的吗？
├── 是 → 直接调用 persona
└── 否  → 子任务独立吗（无共享可变状态、无顺序依赖）？
         ├── 是 → 并行扇出的斜杠命令（例如 /ship）
         └── 否  → 由用户按顺序跑的斜杠命令（/spec → /plan → /build → /test → /review）
```

## 实战：合法的编排

`/ship` 是本仓库规范化的扇出编排器：

```
/ship
  ├── (并行) code-reviewer    → 评审报告
  ├── (并行) security-auditor → 审计报告
  └── (并行) test-engineer    → 覆盖报告
                  ↓
        合并阶段（主 Agent）
                  ↓
        go / no-go 决策 + 回滚计划
```

为何有效：
- 每个子 Agent 处理同一份 diff，但产出*不同视角*
- 它们彼此没有依赖，真正的并行，实打实的时钟节省
- 每个都在全新的上下文窗口里跑，主会话保持清爽
- 合并步骤小且受益于完整上下文，所以留在主 Agent

## 实战：非法编排（不要造这种）

`meta-orchestrator` persona 的工作是"决定调用哪个 persona"：

```
/work-on-pr → meta-orchestrator
                  ↓ （决定"这次需要一次评审"）
              code-reviewer
                  ↓ （返回）
              meta-orchestrator（转述结果）
                  ↓
              user
```

为何失败：
- 纯路由层，没有任何领域价值
- 多出两轮转述跳点 → 信息丢失 + 2× token 成本
- 用户本就想要 review；让他们直接调 `/review` 即可
- 重复了斜杠命令与 `AGENTS.md` 意图映射已经做的事

## Persona 规则

1. 一个 persona 是一个角色、一个输出格式。若你发现自己在加第二个角色，那就再开一个 persona。
2. **Persona 不会调用其他 persona。** 组合是斜杠命令或用户的职责。在 Claude Code 上这也是硬性平台约束，*"子 Agent 不能派生其他子 Agent"*，所以规则会替你强制。
3. Persona 可以调用 Skill（*怎么用*）。
4. 每个 persona 文件以一段 "Composition" 块结尾，说明它在编排中的位置。

## Claude Code 互操作

本仓库的 persona 既可作为 Claude Code 子 Agent 运行，也可作为 Agent Teams 队友，无需修改：

- **作为子 Agent：** 启用本插件即自动发现（无需 path 配置）。对在线程 Agent 工具使用 `subagent_type: code-reviewer`（或 `security-auditor`、`test-engineer`）。`/ship` 是规范示例。
- **作为 Agent Teams 队友**（实验性，需要 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`）：派生队友时引用相同的 persona 名。Persona 的正文会被**追加到**队友的系统提示词上（作为附加指令，而非替换），因此你的 persona 文本叠在主会话安装的团队协调指令（SendMessage、任务列表工具等）之上。

子 Agent 只把结果报告给主 Agent。Agent Teams 让队友彼此直接发消息。当报告足够时用子 Agent；当子 Agent 需要彼此挑战对方发现时（例如竞争假设调试）用 Agent Teams。完整映射见 [references/orchestration-patterns.md](../references/orchestration-patterns.md)。

插件 Agent 不支持 `hooks`、`mcpServers`、`permissionMode` frontmatter，这些字段会被静默忽略。在此处新增 persona 时不要依赖它们。

## 新增 persona

1. 用既有 persona 同款的 frontmatter 格式创建 `agents/<role>.md`。
2. 定义角色、范围、输出格式与规则。
3. 在末尾加一段 **Composition** 块（直接调用的时机 / 通过哪个命令调用 / 不要被其他 persona 调用）。
4. 把 persona 加到本文件顶部的表格中。
5. 若该 persona 引入新的编排模式，请在 `references/orchestration-patterns.md` 中记录，而不是在 persona 文件里自创模式。