# agent-skills 入门

agent-skills 可与任何接受 Markdown 指令的 AI 编程 Agent 配合。本指南覆盖通用方法。针对特定工具的配置见各专用指南。

想在自己动手前看一个走完一遍的演练？[交互式教程](https://skills.addy.ie/tutorials/) 走过一次全新构建、一次老项目加新特性、一次安全的自动化循环，并提供 Claude Code、Codex 或任何其他 Agent 上可直接复制的 prompt。

## Skill 如何工作

每个 Skill 都是一份 Markdown 文件（`SKILL.md`），描述一种具体的工程工作流。被加载到 Agent 上下文后，Agent 沿该工作流走，包括验证步骤、需规避的反模式与退出标准。

**Skill 不是参考文档。** 它是 Agent 遵循的分步流程。

## 快速上手（任意 Agent）

### 1. 克隆仓库

```bash
git clone https://github.com/addyosmani/agent-skills.git
```

### 2. 挑一个 Skill

浏览 `skills/` 目录。每个子目录有一份 `SKILL.md`，包含：

- **何时使用** ， 表明本 Skill 适用的触发条件
- **流程** ， 分步工作流
- **验证** ， 如何确认工作已完成
- **常见借口** ， Agent 可能用来跳过步骤的借口
- **红旗** ， 表明 Skill 正在被违反的迹象

### 3. 把 Skill 装进你的 Agent

把相关 `SKILL.md` 内容复制到 Agent 的系统提示词、规则文件或对话中。最常见的做法：

**系统提示词：** 把 Skill 内容粘贴在会话开头。

**规则文件：** 把 Skill 内容加进项目的规则文件（CLAUDE.md、.cursorrules 等）。

**对话：** 下指令时引用 Skill："按 test-driven-development 流程做这次改动。"

### 4. 必要时用 meta-skill 做发现

若你的 Agent 没有原生 Skill 路由，先加载 `using-agent-skills`。它内含一张流程图，把任务类型映射到合适的 Skill。

若你的宿主已经能从 description 自动发现并激活 Skill，**不要再把 `using-agent-skills` 粘进 always-on 系统提示词或规则文件**。那相当于给同一任务装两个路由器。安装单独的 Skill，让宿主按需激活即可。

### 既有项目无需迁移

用你的 Agent 的常规安装路径从项目根安装本包，然后继续在该项目里工作。Skill 会为匹配的任务激活；它们不要求新仓库布局，也不要求一次性转换既有代码。

不要把本仓库根的 `AGENTS.md` 或 `CLAUDE.md` 拷进项目里。那两份文件是给 agent-skills 的贡献者配置的。只加 Skill 与你的 Agent 通常读取的项目特定指令。给已立项目的渐进式铺开，遵循 [Adoption Guide](adoption-guide.md)。

## 推荐配置

要铺到真实项目？**[Adoption Guide](adoption-guide.md)** 覆盖两条端到端路径：全新项目第一天就用全套生命周期，以及为既有代码库做的增量、验证优先的铺开。下面是快速版。

### 最小配置（从这里开始）

把三个核心 Skill 装进规则文件：

1. **spec-driven-development** ， 定义要做什么
2. **test-driven-development** ， 证明它能用
3. **code-review-and-quality** ， 合并前验证质量

这三项覆盖 AI 辅助开发中最关键的质量缺口。

### 完整生命周期

要全面覆盖，按阶段装 Skill：

```
启动项目:    spec-driven-development → planning-and-task-breakdown
开发期间:    incremental-implementation + test-driven-development
合并前:       code-review-and-quality + security-and-hardening
部署前:       shipping-and-launch
```

### 按上下文加载

不要一次加载所有 Skill，那浪费上下文。加载与当前任务相关的 Skill：

- 在做 UI？→ 加 `frontend-ui-engineering`
- 调试？→ 加 `debugging-and-error-recovery`
- 搭 CI？→ 加 `ci-cd-and-automation`

## Skill 解剖

每个 Skill 都遵循同一结构：

```
YAML frontmatter（name、description）
├── 概览 ， Skill 做什么
├── 何时使用 ， 触发条件
├── 核心流程 ， 分步工作流
├── 示例 ， 代码样本与模式
├── 常见借口 ， 借口与反驳
├── 红旗 ， Skill 正在被违反的迹象
└── 验证 ， 退出标准清单
```

完整规范见 [skill-anatomy.md](skill-anatomy.md)。

## 使用 Persona

`agents/` 目录包含预配置的 Agent persona：

| Agent | 用途 |
|-------|---------|
| `code-reviewer.md` | 五维度代码评审 |
| `test-engineer.md` | 测试策略与编写 |
| `security-auditor.md` | 漏洞检测 |
| `web-performance-auditor.md` | Core Web Vitals 与性能审计（通过 `/webperf`） |

当你需要专项评审时，加载一份 agent 定义。例如，让你的编程 Agent "用 code-reviewer agent persona 评审这次改动"，并附上 agent 定义。

## 使用命令

`.claude/commands/` 目录包含 Claude Code 的斜杠命令：

| 命令 | 调用的 Skill |
|---------|---------------|
| `/spec` | spec-driven-development |
| `/constraints` | constraint-driven-development |
| `/plan` | planning-and-task-breakdown |
| `/build` | incremental-implementation + test-driven-development |
| `/build auto` | planning-and-task-breakdown → incremental-implementation + test-driven-development（整个计划，一次批准） |
| `/test` | test-driven-development |
| `/review` | code-review-and-quality |
| `/code-simplify` | code-simplification |
| `/ship` | shipping-and-launch |
| `/webperf` | web-performance-auditor（专家 agent，仅限 Web 应用） |

> **注意：** 作为 Claude Code 插件安装时，你可能会看到一条警告，_"Default commands/ folder is ignored because the manifest sets 'commands'"_。这是预期的。根的 `commands/` 目录属于 Antigravity CLI，刻意与 `.claude/commands/` 分开。所有 Claude Code 斜杠命令都从 `.claude/commands/` 正确加载；那条警告只是表面。

## 使用 References

`references/` 目录包含补充检查清单：

| Reference | 配套 Skill |
|-----------|----------|
| `testing-patterns.md` | test-driven-development |
| `performance-checklist.md` | performance-optimization |
| `security-checklist.md` | security-and-hardening |
| `accessibility-checklist.md` | frontend-ui-engineering |
| `definition-of-done.md` | 所有 Skill / 每次改动 |
| `observability-checklist.md` | observability-and-instrumentation |
| `orchestration-patterns.md` | doubt-driven-development |

当你需要 Skill 之外的详细模式时，加载一份 reference。

若用 `npx skills add ... --skill <name>` 单独装一个 Skill，只复制所选的 `skills/<name>/` 目录。Skill 仍能工作，但仓库根 `references/` 目录里的辅助检查清单路径就不可用了。改用整仓集成、克隆仓库、或把所需检查清单复制到已安装 Skill 内部的 `references/` 目录里。这个可移植性缺口在 [addyosmani/agent-skills#361](https://github.com/addyosmani/agent-skills/issues/361) 跟踪。

## Spec 与任务工件

`/spec` 与 `/plan` 命令会创建工作工件（`SPEC.md`、`tasks/plan.md`、`tasks/todo.md`）。在工作中把它们当作**活的文档**：

- 开发期间放进版本控制，让人和 Agent 共享同一份真理源。
- 范围或决策变化时更新它们。
- 若你的仓库不希望长期保留这些文件，合并前删掉，或把目录加进 `.gitignore`，工作流并不要求它们长期存在。

### 跨会话协作

同一份工件就是会话之间的交接。小任务可以在一个会话里跑完整个生命周期。任何不琐碎的工作，每个阶段（spec → plan → build → review）一个新会话能让上下文保持聚焦，把工作向前推的是已批准的文件，不是对话：

- spec ， `SPEC.md`，或你的 spec 实际所在的位置
- `tasks/plan.md` 与 `tasks/todo.md` ， 或者计划中标识的外部跟踪器，如果你用的话

**切换之前**，确认这些文件反映了仍然适用的决策、你批准过的范围、还开放的问题、下一个任务、以及当前的验证状态（哪些测试跑过、对什么跑的）。

**新会话里**，先读真实文件、看 `git status`，再做任何事。不要假定你在工件里看不到的批准。把一条记录的"测试通过"当作关于特定基线的主张：若代码已经移动、若记录里没写"对什么跑的"、或若你即将动它覆盖的区域，就重跑它涵盖的检查。若基线仍然成立，就接住它去推下一项，空出的精力是做与变更应用一致的检查，不是每次交接都跑全套。

#### 任务边界重启与 Ralph 循环

`/build auto` 能在单个会话里跑完整个已批准的计划。它不要求也不会按任务重置进程。它按任务的状态更新、验证结果与提交把每个已完成任务变成一个可重启的边界，所以一个够格的外部 harness 能在那里退出并恢复，不依赖聊天历史。

Shell 级别的 "Ralph 循环" 是 harness 行为，不是单独的 Skill 工作流。如果你用，只在当前任务到达被记录的边界之后才重启；重新进入时，先读选定的最新工件与仓库状态，再挑下一个待办任务。进程退出不是任务通过的证据，重启不得绕过审批门。见 `context-engineering` Skill 的 **Restartable Session Boundaries** 段，那里是交接清单。

不强制要求 `/spec` 与 `/plan` 包装器，在任意 Agent 里用纯请求都行，包括仅装了 Skill 的 `npx skills add` 安装：

> 读 SPEC.md，然后把它拆成可验证的小任务，含验收标准与依赖顺序。把它们存到 tasks/plan.md 与 tasks/todo.md。先别写产品代码，先把计划拿给我看。

> 读 SPEC.md、tasks/plan.md 与 tasks/todo.md，然后查实际进度，`git status`，外加重跑那些记录的验证状态不再覆盖的检查。告诉我下一个未勾选的任务、以及还开放的事，然后停下：我会在你开始前确认范围。若计划看起来不完整，就说什么缺了，而不是把它重写。

## 小贴士

1. **任何不琐碎的工作都从 spec-driven-development 开始**
2. **写代码时始终加载 test-driven-development**
3. **不要跳过验证步骤** ， 它们是 SKILL 的全部意义
4. **有选择地加载 Skill** ， 更多上下文并不总是更好
5. **用 persona 做评审** ， 不同视角抓到不同问题