---
name: documentation-and-adrs
description: 记录决策与文档。需要文档化一个架构决策（ADR）或设计选择背后的理由、修改公开 API、上线特性、或者需要记录上下文让未来的工程师和 Agent 能理解代码库时使用。
---

# Documentation and ADRs

## 概述

文档记录的是决策，而不是代码本身。最有价值的文档捕捉 *为什么* ， 上下文、约束、做出这个决策的取舍。代码展示了 *做了什么*；文档解释了 *为什么这么做*，以及 *考虑了哪些替代方案*。这份上下文对未来在代码库里工作的人和 Agent 都是必不可少的。

## 何时使用

- 做一个重要的架构决策
- 在多个竞争方案间抉择
- 增加或修改一个公开 API
- 上线一个改变用户可见行为的特性
- 让新团队成员（或 Agent）入门项目
- 当你发现自己在反复解释同一件事

**不用：** 不要文档化显而易见的代码。不要加只是复述代码的注释。不要为一次性的原型写文档。

## 架构决策记录（ADR）

ADR 捕捉重要技术决策背后的推理。它们是你能写的最有价值的文档。

### 何时写 ADR

- 选框架、库或重要依赖时
- 设计数据模型或数据库 schema
- 选认证策略
- 决定 API 架构（REST vs GraphQL vs tRPC）
- 在构建工具、托管平台、基础设施间选择
- 任何一旦反悔就要付出可观代价的决策

### 先匹配已有约定

在创建 ADR 之前，先翻一下现有仓库上下文，看看有没有既有约定 ， 现有 ADR、项目说明、ADR 相关配置或工具（例如 `.adr-dir` 文件）。既有约定优先于下面的默认值。匹配：

- **位置与格式** ， 例 `docs/adr/*.md`、`Documentation/Decisions/*.rst`、MADR 布局，或 `adr-tools` 工具链。匹配既有目录、文件扩展名、标记语言（Markdown vs reStructuredText）。
- **编号与命名** ， 沿用既有序列与文件命名（`ADR-004-Title.rst`、`0004-title.md` ...）；不要从 001 重启、也不要引入第二种编号方式。
- **章节标题** ， 复用项目既有的标题集合，而不是硬塞本模板的标题。

如果可用证据互相矛盾，先把冲突亮出来而不是默默引入另一种编号。只有在找不到任何约定的情况下，才用下面的默认。

### ADR 模板

把 ADR 存在 `docs/decisions/`，连续编号（除非项目已在用其它位置 ， 见上）：

```markdown
# ADR-001: 用 PostgreSQL 作为主数据库

## Status
Proposed | Accepted | Superseded by ADR-XXX | Deprecated

## Date
2025-01-15

## Context
任务管理应用需要一个主数据库。关键要求：
- 关系数据模型（用户、任务、团队，有关系）
- 任务状态变更的 ACID 事务
- 支持任务内容的全文搜索
- 可用托管 hosting（小团队、有限 ops 能力）

## Decision
用 PostgreSQL + Prisma ORM。

## Alternatives Considered

### MongoDB
- 优点：schema 灵活、上手容易
- 缺点：我们的数据本质上是关系型；需要手工管理关系
- 否决：文档库里的关系数据要么写复杂 join、要么冗余

### SQLite
- 优点：零配置、内嵌、读快
- 缺点：并发写有限、生产无可用托管
- 否决：不适合多用户 Web 应用的生产

### MySQL
- 优点：成熟、广泛支持
- 缺点：PostgreSQL 的 JSON 支持、全文搜索、生态工具更好
- 否决：PostgreSQL 更匹配我们的功能要求

## Consequences
- Prisma 提供类型安全的数据库访问与迁移管理
- 可以用 PostgreSQL 的全文搜索，不引入 Elasticsearch
- 团队需要懂 PostgreSQL（标准技能、低风险）
- 托管在托管服务上（Supabase、Neon 或 RDS）
```

### ADR 生命周期

```
PROPOSED → ACCEPTED → (SUPERSEDED 或 DEPRECATED)
```

- **不要删旧 ADR。** 它们记录的是历史上下文。
- 决策有变时写新 ADR，引用并取代旧的。

## 内联文档

### 何时该加注释

注释 *为什么*，而不是 *什么*：

```typescript
// 反例：复述代码
// 把计数器 +1
counter += 1;

// 好：解释非显然意图
// 限流用滑动窗口 ， 在窗口边界处重置计数器，而非按固定时间表，
// 以防窗口边缘的突发攻击
if (now - windowStart > WINDOW_SIZE_MS) {
  counter = 0;
  windowStart = now;
}
```

### 何时不加注释

```typescript
// 不要给自解释代码加注释
function calculateTotal(items: CartItem[]): number {
  return items.reduce((sum, item) => sum + item.price * item.quantity, 0);
}

// 不要留 TODO 拖到以后
// TODO: 加错误处理  ← 直接加

// 不要留注释掉的代码
// const oldImplementation = () => { ... }  ← 直接删，git 里有历史
```

### 文档化已知坑

```typescript
/**
 * IMPORTANT: 这个函数必须在首次渲染前调用。
 * 如果在水合之后调用，会因为 SSR 期间主题 context 不可用，
 * 导致样式闪烁。
 *
 * 设计原理详见 ADR-003。
 */
export function initializeTheme(theme: Theme): void {
  // ...
}
```

## API 文档

对公开 API（REST、GraphQL、库接口）：

### 与类型同行内联（TypeScript 首选）

```typescript
/**
 * 创建一个新任务。
 *
 * @param input - 任务创建数据（title 必填，description 可选）
 * @returns 带服务端生成 ID 与时间戳的已创建任务
 * @throws {ValidationError} 当 title 为空或超过 200 字符
 * @throws {AuthenticationError} 当用户未认证
 *
 * @example
 * const task = await createTask({ title: '买杂货' });
 * console.log(task.id); // "task_abc123"
 */
export async function createTask(input: CreateTaskInput): Promise<Task> {
  // ...
}
```

### 用 OpenAPI / Swagger 描述 REST API

```yaml
paths:
  /api/tasks:
    post:
      summary: 创建一个任务
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateTaskInput'
      responses:
        '201':
          description: 任务已创建
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Task'
        '422':
          description: 校验错误
```

## README 结构

每个项目都应该有 README，覆盖：

```markdown
# Project Name

一段话说明这个项目做什么。

## Quick Start
1. 克隆仓库
2. 安装依赖：`npm install`
3. 设置环境：`cp .env.example .env`
4. 跑 dev 服务：`npm run dev`

## Commands
| Command | Description |
|---------|-------------|
| `npm run dev` | 启动开发服务 |
| `npm test` | 跑测试 |
| `npm run build` | 生产构建 |
| `npm run lint` | 跑 linter |

## Architecture
项目结构和关键设计决策的简要总览。
链 ADR 详情。

## Contributing
如何贡献、编码规范、PR 流程。
```

## Changelog 维护

对已上线特性：

```markdown
# Changelog

## [1.2.0] - 2025-01-20
### Added
- 任务分享：用户可以把任务分享给团队成员 (#123)
- 任务分配时的邮件通知 (#124)

### Fixed
- 快速点创建按钮出现的重复任务 (#125)

### Changed
- 任务列表每页 50 条（之前 20）以提升 UX (#126)
```

## 面向 Agent 的文档

对 AI Agent 上下文做特别考虑：

- **CLAUDE.md / 规则文件** ， 文档化项目约定，让 Agent 跟着走
- **Spec 文件** ， 保持更新，让 Agent 建对东西
- **ADR** ， 帮 Agent 理解过去的决策（防止再决策一次）
- **内联坑警告** ， 防止 Agent 掉进已知陷阱

## 文稿评审

技术内容核对完后，再为将来的读者审一下文稿。更清楚的措辞不应改变决策或限制。

1. 标记必须保留的细节：名字、数字与单位、命令、路径、链接、条件、权衡、不确定性。对照可用代码与决策上下文核；任何核不实的就标记。
2. 用读者真正需要的具体信息，替代套话开头、形容词与重复总结。不要拿"未被支持的声明"换成编出来的例子或更强的承诺。保留必要的技术重复与项目既有术语。
3. 把改过的文档与原文及出处比对。被遗漏的限定或 caveat 要恢复；命令与代码示例要核仍然按文档工作；未解决的事实问题要解释清楚，而不是抛光掉。

## 常见借口

| Rationalization | Reality |
|---|---|
| "代码自文档" | 代码展示 what。不展示 why、否决了哪些替代方案、什么约束适用。 |
| "API 稳定了再写文档" | 写文档时 API 会先稳定。文档就是设计的第一关验证。 |
| "没人读文档" | Agent 读。未来的工程师读。三个月后的你自己读。 |
| "ADR 是 overhead" | 一份 10 分钟 ADR 能挡住 6 个月后关于同个决策的两小时辩论。 |
| "注释会过时" | 注释 *why* 是稳定的。注释 *what* 会过时 ， 所以只写前者。 |

## 红旗

- 架构决策没有书面理由
- 公开 API 没有文档或类型
- README 不教人怎么跑这个项目
- 注释掉的代码取代删除
- 几周没动的 TODO 注释
- 做了重要架构选择的项目没有 ADR
- 文档复述代码而不是讲意图

## 验证

文档化完成后：

- [ ] 所有重要的架构决策都有 ADR
- [ ] README 覆盖快速开始、命令、架构总览
- [ ] API 函数有参数与返回类型文档
- [ ] 已知坑在内联该出现的地方被记录
- [ ] 没有注释掉的代码残留
- [ ] 规则文件（CLAUDE.md 等）当前且准确
- [ ] 文稿编辑保留了源细节、决策权衡与限定，不引入未被支持的承诺
