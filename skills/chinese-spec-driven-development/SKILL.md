---
name: spec-driven-development
description: 在写代码前先建规格。在启动新项目、新特性或重大改动、还没有任何规格时使用。在起草 PRD 或需求文档（含目标与范围）、需求不清 / 模糊 / 只是一个模糊想法时使用。在单一需求跨越几个独立可测能力、需要在写规格前先拆成能力的模块图时使用。
---

# Spec-Driven Development

## 概述

在任何代码之前写一份结构化的规格。规格是你与人类工程师之间的共同真源 ， 它定义我们要构建什么、为什么、怎么算它做好了。没有规格就写代码是在猜。

## 何时使用

- 启动一个新项目或新特性
- 需求模糊或不完整
- 改动跨多个文件或模块
- 你即将做一个架构决策
- 实现任务耗时超过 30 分钟

**不用：** 单行修复、typo 修正，或需求明确自包含的改动。

## 有闸的工作流

Spec-driven development 包含四个阶段，前置一个范围检查（阶段 0）只在请求捆绑多个独立可测能力时激活。一阶段没验收不要进下一阶段。

```
SPECIFY ──→ PLAN ──→ TASKS ──→ IMPLEMENT
   │          │        │          │
   ▼          ▼        ▼          ▼
 人类       人类      人类       人类
 评审       评审      评审       评审
```

### 阶段 0：范围检查

大多数请求描述的是单一能力。当本请求如此时，跳过这阶段直接进 Specify ， 阶段 0 是为那个例外存在的，不是规则，也不让单能力的特性有任何层级。

**检测。** 在下列情况下，先拆再写规格：

- 请求里点名了互不重叠、有各自消费者或数据的能力（如 identity、billing、notifications、reporting）
- 验收标准会聚成"可以分开交付、独立验证"的群
- 砍掉或替换一个能力不会逼着改其它能力的需求

**在写任何规格之前先出一份能力图。** 小而可审 ， 一张模块表加构建顺序，不是项目计划：

```markdown
# Capability Map: [Initiative Name]

| Module id | Responsibility | Depends on |
|---|---|---|
| identity | Accounts, sessions, SSO | ， |
| billing | Plans, invoices, payments | identity |
| notifications | Email 和 webhook 扇出 | identity |
| reporting | Usage 仪表板 | billing, notifications |

Build order: identity → billing, notifications → reporting
```

- **稳定的 module id。** kebab-case，一次定下，整个 initiative 中不改。规格、plan、下游命令用这些 id 选活，不靠猜哪份规格激活。
- **依赖方向，不能成环。** 箭头单向。两个模块互相需要，就是一个。
- **接口住在边界。** 图记录 `billing` 依赖 `identity`；两者间的契约写在提供方模块的规格里（设计见 `api-and-interface-design`）。

**这张图跟其它阶段一样有人工闸。** 在任何模块规格被写下前，人评审模块边界、依赖方向、构建顺序。整张图错代价大；看十行不贵。

**然后逐模块递归。** 按依赖顺序，对每个模块跑 Specify → Plan → Tasks → Implement。每个模块拿自己的规格，限定到那个模块的目标、边界、成功标准。把批准过的能力图存在项目根，每个模块的规格跟它放一起、用 module id 命名（`SPEC-identity.md`、`SPEC-billing.md`），图本身（不是文件名猜测）是"有什么"的索引。

### 阶段 1：Specify

从一个高层的愿景开始。问人澄清问题直到需求具体。

**立刻亮假设。** 在写任何规格内容之前，列出你在假设什么：

```
我正在做的假设：
1. 这是一个 Web 应用（不是原生移动）
2. 认证用基于 cookie 的 session（不是 JWT）
3. 数据库是 PostgreSQL（基于已有 Prisma schema）
4. 我们只支持现代浏览器（不支持 IE11）
→ 现在就纠正我，否则我按这些继续。
```

不要默默把含糊的需求补全。规格的全部目的就是把误解*在写代码之前*暴露出来 ， 假设是最危险的误解形态。

**写一份覆盖这六个核心区域的规格文档：**

1. **目标** ， 我们在构建什么、为什么。用户是谁。成功长什么样。
2. **命令** ， 带 flags 的完整可执行命令，不只是工具名。
   ```
   Build: npm run build
   Test: npm test -- --coverage
   Lint: npm run lint --fix
   Dev: npm run dev
   ```
3. **项目结构** ， 源码放在哪、测试在哪、文档在哪。
   ```
   src/           → Application source code
   src/components → React components
   src/lib        → Shared utilities
   tests/         → Unit and integration tests
   e2e/           → End-to-end tests
   docs/          → Documentation
   ```
4. **代码风格** ， 一段真实代码片段抵得过三段描述。包含命名约定、格式规则、好输出的例子。
5. **测试策略** ， 什么框架、测试放哪、覆盖率预期、关心什么用什么测试层级。
6. **边界** ， 三层系统：
   - **始终做：** 提交前跑测试、遵守命名约定、校验输入
   - **先问：** 数据库 schema 变更、加依赖、改 CI 配置
   - **永远不做：** 提交密钥、编辑 vendor 目录、未经批准移除失败的测试

**规格模板：**

```markdown
# Spec: [Project/Feature Name]

## Objective
[我们要构建什么、为什么。User story 或验收标准。]

## Tech Stack
[框架、语言、关键依赖与版本]

## Commands
[Build, test, lint, dev ， 完整命令]

## Project Structure
[目录布局与描述]

## Code Style
[示例片段 + 关键约定]

## Testing Strategy
[框架、测试位置、覆盖率要求、测试层级]

## Boundaries
- Always: [...]
- Ask first: [...]
- Never: [...]

## Success Criteria
[怎么算这事做了 ， 具体、可测的条件]

## Open Questions
[任何还未解决、需要人的问题]
```

**外部规格工具：** 这套工作流与格式无关。如果项目已经在用 OpenSpec 或别的规格系统，沿用那个系统的产物格式与存储约定，不要造重复的 `SPEC.md`。这个 Skill 管澄清、内容与审批闸；外部工具管已批准规格的呈现方式。

**把指令重新框成成功标准。** 当收到模糊需求时，把它们译成具体条件：

```
需求："让 dashboard 更快"

重新框出的成功标准：
- 仪表板 LCP < 2.5s 在 4G 网络
- 首次数据加载 < 500ms
- 加载过程无布局偏移（CLS < 0.1）
→ 这些目标对吗？
```

这让你循环、重试、解决问题时朝一个清晰的目标走，而不是去猜"更快"是什么意思。

**写完规格后停（关键）。** 规格保存之后：

1. 总结规格并列出 Open Questions。
2. 请人批准或提出修改。
3. **立即停止本轮。** 不要在那一轮启动阶段 2、调起 `planning-and-task-breakdown`、或写代码。规划要在人另开一轮批准规格后再开始。

### 阶段 2：Plan

用被验证的规格生成一份技术实施计划：

1. 找出主要组件与它们之间的依赖
2. 确定实现顺序（什么先建）
3. 标出风险与缓解策略
4. 区分可并行 vs 必须串行
5. 定义阶段间的检查点

> 详细说明按 `planning-and-task-breakdown`，那是规范化依赖图与纵切片技法的唯一来源。上面的要点是轻量摘要；如有出入，`planning-and-task-breakdown` 优先。
>
> **输出约定：** 把 plan 保存到 `tasks/plan.md`，任务清单写在 `planning-and-task-breakdown` 定义的 task list target（默认 `tasks/todo.md`；项目可能指定外部跟踪器）。若 `tasks/` 不存在则创建。下游命令（`/build` 等）期待这些默认。

计划要可审：人应该能读一下说"对，是这个方向"或"不，改 X"。

### 阶段 3：Tasks

把计划拆成离散、可实现的任务：

- 每条任务应该在一个专注的会话里能完成
- 每条带明确的验收标准
- 每条带一个验证步骤（测试、构建、手工检查）
- 任务按依赖排序，不按主观重要性
- 任何任务不应该改超过约 5 个文件

> 详细的任务体量、依赖排序看 `planning-and-task-breakdown`；那是规范化机制。下文模板是轻量的内联形态；如与 `planning-and-task-breakdown` 出入，`planning-and-task-breakdown` 优先。

**任务模板：**
```markdown
- [ ] Task: [Description]
  - Acceptance: [完成时必须为真]
  - Verify: [怎么确认 ， 测试命令、构建、手工检查]
  - Files: [会碰到的文件]
```

### 阶段 4：Implement

按 `skills/incremental-implementation/SKILL.md`（`incremental-implementation`）与 `skills/test-driven-development/SKILL.md`（`test-driven-development`）逐条执行任务。用 `skills/context-engineering/SKILL.md`（`context-engineering`）在每一步加载正确的规格节与源文件，而不是把整份规格倾倒给 Agent。

## 让规格活着

规格是活文档，不是一次性产物：

- **决策变了就更新** ， 发现数据模型需要变，先改规格，再实现。
- **范围变了就更新** ， 加 / 砍特性要反映在规格里。
- **把规格提交** ， 规格跟代码一起进版本控制。
- **在 PR 里引用规格** ， 每个 PR 链回它实现的规格段。

## 常见借口

| Rationalization | Reality |
|---|---|
| "这很简单，不需要规格" | 简单任务不需*长*规格，但仍要验收标准。两行规格就行。 |
| "我写完代码再补规格" | 那是文档、不是规格。规格的价值是在*代码之前*强迫清晰。 |
| "规格会拖慢我们" | 15 分钟的规格挡下几小时的返工。15 分钟的瀑布，比 15 小时的 debug 强。 |
| "需求反正会变" | 这正是规格是活文档的原因。过期的规格仍然比没有好。 |
| "用户知道他想要什么" | 即便是清晰的需求也有隐式假设。规格把这些假设亮出来。 |
| "这是一个大特性，拆是 overhead" | 如果验收标准能聚成"独立可测"的群，一份冗余规格逼着每个下游任务为整份契约去推导。一份十行的能力图是廉价的替代。 |
| "我在 Plan 阶段再拆" | Plan 切分的是规格内的任务。那时过大的工件已经存在 ， 模块边界与依赖方向必须先于规格敲定，不是之后。 |

## 红旗

- 没有任何书面需求就开始写代码
- 在弄清"完成"是什么意思之前问"要不直接开建？"
- 实现规格或任务清单里没提到的特性
- 做架构决策时不文档化
- 因为"明显要做什么"就跳过规格
- 同一轮里写完规格就开始 Plan 或代码
- 一份规格涵盖了几个独立可测的能力
- 模块边界或构建顺序是到了实现期才决定的，因为事前没批准能力图

## 验证

进入实现前确认：

- [ ] 规格覆盖了六个核心区域
- [ ] 人审过并批准了规格
- [ ] 那一轮在保存规格后就结束了；批准是在另一轮给出的
- [ ] 成功标准是具体且可测的
- [ ] 边界（Always / Ask First / Never）已定义
- [ ] 规格被保存到仓库里的文件
- [ ] 如果请求捆绑了几个独立可测能力，在写任何模块规格之前先批准了能力图（模块 id、依赖方向、构建顺序）
- [ ] 每个模块规格源自能力图里的某个 module id
