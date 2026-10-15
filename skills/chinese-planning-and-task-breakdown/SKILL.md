---
name: planning-and-task-breakdown
description: 把工作拆成有序任务。在已经有规格或明确需求、需要把工作拆成可实现的任务时使用。在任务大到无从下手、需要估算范围、可以并行处理时使用。
---

# Planning and Task Breakdown

## 概述

把工作拆成小而可验证、验收标准明确的任务。好的任务拆分是一条 Agent 能可靠完成、一条会搞出乱麻的分界。每个任务都应该足够小，能在一个专注的会话里实现、测试、验证。

## 何时使用

- 已有规格，需要把它拆成可实现的单元
- 一个任务太大或太模糊无从下手
- 工作要跨多个 Agent 或会话做并行
- 需要向人说明范围
- 实现顺序不明显

**不用：** 单文件且范围明显的工作，或规格里已经带定义良好的任务。

## 规划流程

### 步骤 1：进入规划模式

动手写代码之前，先以只读模式操作：

- 读规格和仓库里相关段落
- 识别现有模式与约定
- 映射组件间的依赖
- 记下风险和未知项

**规划阶段不要写代码。** 产物是 `tasks/plan.md` 的规划文档与任务清单（见"输出文件"，默认 `tasks/todo.md`），不是实现。

### 步骤 2：识别依赖图

映射出谁依赖谁：

```
Database schema
    │
    ├── API models/types
    │       │
    │       ├── API endpoints
    │       │       │
    │       │       └── Frontend API client
    │       │               │
    │       │               └── UI components
    │       │
    │       └── Validation logic
    │
    └── Seed data / migrations
```

实现顺序按依赖图自底向上：先建地基。

### 步骤 3：纵向切

不要先把整套数据库建好，再做 API，再做 UI ， 每条都建一条贯穿全栈的纵切片：

**反（横切）：**
```
任务 1：建完整数据库 schema
任务 2：建所有 API endpoints
任务 3：建所有 UI 组件
任务 4：拼起来
```

**正（纵切）：**
```
任务 1：用户可以注册账号（schema + API + 注册 UI）
任务 2：用户可以登录（认证 schema + API + 登录 UI）
任务 3：用户可以创建任务（任务 schema + API + 创建 UI）
任务 4：用户可以查看任务列表（查询 + API + 列表 UI）
```

每条纵切片都交付可工作、可测试的功能。

### 步骤 4：写任务

每条任务按下面结构写，无论是 markdown 任务清单还是外部跟踪器（见"输出文件"）：

```markdown
## Task [N]: [简短描述性标题]

**Description:** 一段话说明本任务完成什么。

**Acceptance criteria:**
- [ ] [具体、可验证的条件]
- [ ] [具体、可验证的条件]

**Verification:**
- [ ] 测试通过：[仓库自己的聚焦测试命令]
- [ ] 构建成功：[仓库的构建命令]
- [ ] 手动检查：[描述需要核验什么]

**Dependencies:** [本任务依赖的任务编号，或 "None"]

**Files likely touched:**
- `src/path/to/file.ts`
- `tests/path/to/test.ts`

**Estimated scope:** [Small: 1-2 files | Medium: 3-5 files | Large: 5+ files]
```

### 步骤 5：定顺序与检查点

任务这样排：

1. 依赖被满足（先建地基）
2. 每条任务都让系统保持可工作状态
3. 每 2-3 条任务后有一个验证检查点
4. 高风险任务靠前（快速试错）

显式把检查点加到任务清单：

```markdown
## Checkpoint: After Tasks 1-3
- [ ] 全部测试通过
- [ ] 应用构建无错
- [ ] 核心用户流跑通
- [ ] 与人复核后再继续
```

## 任务体量指南

| 体量 | 文件 | 范围 | 例子 |
|------|-------|-------|---------|
| **XS** | 1 | 单一函数 / 配置变更 | 加一条校验规则 |
| **S** | 1-2 | 一个组件或一个 endpoint | 加一个新 API endpoint |
| **M** | 3-5 | 一条特性切片 | 用户注册流程 |
| **L** | 5-8 | 跨多组件特性 | 带筛选与分页的搜索 |
| **XL** | 8+ | **太大 ， 进一步拆分** | ， |

任务如果是 L 或更大，就应该再拆。Agent 在 S 和 M 任务上表现最好。

**何时再拆：**
- 超过一个专注会话（约 2+ 小时 Agent 工作）
- 没法在 3 个或更少的验收要点里讲清
- 触及两个或更多独立子系统（如 auth 与 billing）
- 任务标题里出现 "and"（两个任务的信号）

## 输出文件

- **规划文档：** 把实施规划存为 `tasks/plan.md`。**永远**是 markdown ， 设计决策、风险、开放问题不能干净地塞进单个 tracker issue。
- **任务清单：** 在**任务清单 target**（见下）里记录每条任务。

如果 `tasks/` 目录不存在就建。

**不要覆盖未完成的规划。** 写 `tasks/plan.md` / `tasks/todo.md` 之前，先看它们是否已存在且仍带未勾选任务：

- 同一工作被重新规划（用户让你修订或扩展本规划）→ 就地更新现有文件。
- 不同的工作 → **停下来问。** 未勾选的任务可能在另一个会话还在做。不要自己删、改名或覆盖现有文件；把这个冲突摆出来，让用户决定（先把老规划收尾、明确放弃、或者告诉你新规划去哪）。

同样的规则适用于外部任务清单 target：不要批量关闭或删另一个规划的开放 tracker 项来给新的让位。

### 任务清单 Target

任务清单 target 是任务与检查点的登记位置。下面一次定义，其它 Skill 都引用它。

- **默认：`tasks/todo.md` 的 markdown 清单。** 这是 `/build` 与下游工具期望的约定。除非项目另有约定，否则就用它。
- **外部跟踪器：** 如果项目的 Agent 规则（`CLAUDE.md`、`AGENTS.md` 等）或用户指定了一个 issue tracker（GitHub Issues、Jira、Linear、`bd`/beads），就改成每个任务建一个 tracker 项，把步骤 4 的结构映射过去：验收标准与验证步骤放在项 body，依赖用 tracker 的链接机制（`bd dep add`、`blocked by` 等）。把步骤 5 的检查点也建成 tracker 项，或者按"无法对位"的写在规划文档里。

用外部跟踪器时，在 `tasks/plan.md` 里写明（例如 "Tasks tracked in Linear project FOO"），让下游步骤和后续会话知道去哪看；同时把规划文档里的 Task List 节保留为按顺序的 tracker 项 ID 或链接索引，不要复制一份清单。

## 规划文档模板

```markdown
# Implementation Plan: [Feature/Project Name]

## Overview
[一段总结我们要建什么]

## Architecture Decisions
- [关键决策 1 与理由]
- [关键决策 2 与理由]

## Task List

### Phase 1: Foundation
- [ ] Task 1: ...
- [ ] Task 2: ...

### Checkpoint: Foundation
- [ ] 测试通过，构建干净

### Phase 2: Core Features
- [ ] Task 3: ...
- [ ] Task 4: ...

### Checkpoint: Core Features
- [ ] 端到端流程跑通

### Phase 3: Polish
- [ ] Task 5: ...
- [ ] Task 6: ...

### Checkpoint: Complete
- [ ] 全部验收标准达成
- [ ] 准备评审

## Risks and Mitigations
| Risk | Impact | Mitigation |
|------|--------|------------|
| [风险] | [High/Med/Low] | [策略] |

## Open Questions
- [需要人回答的问题]
```

任务放在外部跟踪器时，Task List 这一节保留为按顺序的 tracker 项 ID 或链接的索引，而不是复制的清单。

## 可并行机会

有多 Agent 或多会话可用时：

- **可并行：** 独立的特性切片、对已落地特性的测试、文档
- **必须串行：** 数据库迁移、共享状态变更、依赖链
- **要协调：** 共享 API 契约的特性（先定契约，再并行）

## 常见借口

| Rationalization | Reality |
|---|---|
| "我边写边弄明白" | 这就是搞出乱麻和返工的来历。10 分钟规划省下几小时。 |
| "任务是显而易见的" | 还是写下来。明确的任务会暴露隐藏的依赖和被遗忘的边界情况。 |
| "规划就是 overhead" | 规划本身就是任务。没有规划的"实现"只是打字。 |
| "这些我能一直 hold 在脑子里" | 上下文窗口是有限的。写下来的规划能跨会话、跨压缩。 |
| "老的 `tasks/plan.md` 老旧了，我替换就好" | 未勾选的任务可能在另一个会话做了一半。盖掉它们就毁了只在工作状态里才有的信息。停下来问。 |

## 红旗

- 还没写任务清单就开始实现
- 在仍有未勾选任务的 `tasks/plan.md` / `tasks/todo.md` 里覆盖别的活，且没问
- 项目已经指定外部 tracker，却写 `tasks/todo.md`（或者两边都放）
- 任务只写 "implement the feature"，没有验收标准
- 规划里没有验证步骤
- 所有任务都是 XL 体量
- 任务之间没有检查点
- 没考虑依赖顺序

## 验证

开始实现前，确认：

- [ ] 每条任务都带验收标准
- [ ] 每条任务都带验证步骤
- [ ] 任务依赖被识别且排序正确
- [ ] 任务记录在任务清单 target（默认 `tasks/todo.md`）
- [ ] 任何预先存在的未完成规划没被擅自覆盖
- [ ] 没有任务需要改动超过 ~5 个文件
- [ ] 各主要阶段之间有检查点
- [ ] 有人已经审过并批准了规划

## 参见

验收标准是任务级的，回答"我们做对了么？"。它们建立在项目级的 Definition of Done 之上 ， 这是每个任务都要越过的常设门槛。参见 `../../references/definition-of-done.md`。
