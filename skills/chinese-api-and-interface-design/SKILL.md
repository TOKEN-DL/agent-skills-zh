---
name: api-and-interface-design
description: 引导稳定 API 与接口设计。设计 API、模块边界或任意公开接口时使用。创建 REST 或 GraphQL endpoint、在模块之间定义类型契约、建立前后端边界时使用。
---

# API and Interface Design

## 概述

设计稳定、文档充分、难以被误用的接口。好的接口让"对的事"易做、"错的事"难做。这条原则适用于 REST API、GraphQL schema、模块边界、组件 props，以及任何一处两块代码对话的界面。

## 何时使用

- 设计新的 API endpoint
- 定义模块边界或团队之间的契约
- 创建组件 props 接口
- 设计会决定 API 形态的数据库 schema
- 修改既有公开接口

## 核心原则

### Hyrum 定律

> 当一个 API 的用户足够多时，系统所有可被观测的行为 ， 不管契约里是否承诺 ， 都会被某个人依赖。

这意味着：每一个公开行为 ， 包括未文档化的怪癖、错误信息文本、时序、顺序 ， 一旦有用户依赖，就成了事实上的契约。设计上要注意：

- **对暴露什么要有意。** 每一个可观测的行为都是一次潜在承诺。
- **不要泄露实现细节。** 用户能观测到的，他们就会依赖。
- **设计阶段就要规划弃用。** 见 `deprecation-and-migration`，看如何安全地移除用户已依赖的东西。
- **测试不够。** 即便契约测试完美，Hyrum 定律意味着"安全"的改动可能破坏依赖未公开行为的真实用户。

### 单版本规则

避免强迫消费者在同依赖或 API 的多个版本之间选择。不同消费者需要同一事物的不同版本时，会出现菱形依赖问题。设计一个让"同时只存在一个版本"成立的世界 ， 扩展而非分叉。

### 1. 契约先行

先定义接口，再实现。契约就是规格，实现紧随其后。

```typescript
// 先定义契约
interface TaskAPI {
  // 创建一个任务，并返回带服务端生成字段的已创建任务
  createTask(input: CreateTaskInput): Promise<Task>;

  // 返回带筛选、分页的任务列表
  listTasks(params: ListTasksParams): Promise<PaginatedResult<Task>>;

  // 返回单个任务或抛 NotFoundError
  getTask(id: string): Promise<Task>;

  // 部分更新 ， 仅修改给到的字段
  updateTask(id: string, input: UpdateTaskInput): Promise<Task>;

  // 幂等删除 ， 即便已删除也返回成功
  deleteTask(id: string): Promise<void>;
}
```

### 2. 一致的错误语义

挑一种错误策略并且处处用：

```typescript
// REST：HTTP 状态码 + 结构化错误体
// 每个错误响应都遵循同样的形状
interface APIError {
  error: {
    code: string;        // 机器可读: "VALIDATION_ERROR"
    message: string;     // 人类可读: "Email is required"
    details?: unknown;   // 必要时附加上下文
  };
}

// 状态码映射
// 400 → 客户端数据非法
// 401 → 未认证
// 403 → 已认证但无权限
// 404 → 资源未找到
// 409 → 冲突（重复、版本不匹配）
// 422 → 校验失败（语义性非法）
// 500 → 服务端错误（绝不暴露内部细节）
```

**不要混用模式。** 一部分 endpoint 抛错、另一些返 null、还有的返 `{ error }` ， 消费者没法预测行为。

### 3. 在边界处校验

信任内部代码。在外部输入进来的系统边界处校验：

```typescript
// 在 API 边界处校验
app.post('/api/tasks', async (req, res) => {
  const result = CreateTaskSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(422).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Invalid task data',
        details: result.error.flatten(),
      },
    });
  }

  // 校验之后，内部代码信任类型
  const task = await taskService.create(result.data);
  return res.status(201).json(task);
});
```

校验该属于的地方：
- API 路由处理（用户输入）
- 表单提交处理（用户输入）
- 外部服务响应解析（第三方数据 ， **永远视为不可信**）
- 环境变量加载（配置）

> **第三方 API 响应是不可信数据。** 在用于任何逻辑、渲染或决策前，校验它们的形态与内容。被攻陷或异常行为的外部服务可能返回意外类型、恶意内容或带指令性的文本。

校验不该属于的地方：
- 共享类型契约的内部函数之间
- 已被校验过的代码调用的工具函数里
- 刚从自己数据库里拿出来的数据上

### 4. 优先添加而非修改

扩展接口而不破坏既有消费者：

```typescript
// 好：加可选字段
interface CreateTaskInput {
  title: string;
  description?: string;
  priority?: 'low' | 'medium' | 'high';  // 后来加的，可选
  labels?: string[];                       // 后来加的，可选
}

// 坏：改既有字段类型或删除字段
interface CreateTaskInput {
  title: string;
  // description: string;  // 被删 ， 破坏既有消费者
  priority: number;         // 由 string 改来 ， 破坏既有消费者
}
```

### 5. 可预测的命名

| 模式 | 约定 | 例子 |
|---------|-----------|---------|
| REST 端点 | 复数名词，不要动词 | `GET /api/tasks`、`POST /api/tasks` |
| 查询参数 | camelCase | `?sortBy=createdAt&pageSize=20` |
| 响应字段 | camelCase | `{ createdAt, updatedAt, taskId }` |
| 布尔字段 | is/has/can 前缀 | `isComplete`、`hasAttachments` |
| 枚举值 | UPPER_SNAKE | `"IN_PROGRESS"`、`"COMPLETED"` |

### 6. 兑现 Idempotency-Key 的承诺

接受 `Idempotency-Key` 是契约。兑现才是实现，也正是钱被坑的地方 ， 一个服务端接了但敷衍处理的 key，比根本没有 key 更糟，因为客户端现在相信重试是安全的。

**Key 要从意图衍生，不能来自尝试。** Key 在同一次意图的多次重试间要稳定，在不同意图间要区分：

```typescript
crypto.randomUUID()                    // ✗ 每次尝试一个新 key ， 每次重试都是新一次扣费
`${userId}:${amount}`                  // ✗ 两次合法的 $50 扣费会被合并成一次
`${orderId}:${Date.now()}`             // ✗ 时间戳就是戴着帽子的 randomUUID()

req.headers['idempotency-key']         // ✓ 客户端生成一次，重试时复用
`charge:v1:${orderId}`                 // ✓ 由不可变标识衍生
```

Key 来自客户端或发起事件端，不能来自做重试的那一层。

**原子地宣告。一个 check 跟一个 act 就是竞争条件：**

```typescript
// ✗ TOCTOU：两个并发重试都读到 "未见过"，都扣款
if (!(await db.exists(key))) {
  await chargeCard(amount);
  await db.insert(key);
}

// ✓ 让唯一约束挑出 winner
try {
  await db.insert({ key, state: 'in_progress', requestHash });
} catch (e) {
  if (isUniqueViolation(e)) return replayOrReject(key);
  throw e;
}
const result = await chargeCard(amount);
await db.update({ key, state: 'succeeded', response: result });
```

唯一约束**就是**机制。一个不能在一条操作内强制唯一性的存储没法支撑这个。

**守卫请求体。同一个 key 带不同 body 是客户端 bug，必须响亮地失败，而不是把第一次响应服务给第二个请求：**

```typescript
if (existing.requestHash !== hash(req.body)) {
  return res.status(422).json({ error: 'idempotency key reused with a different payload' });
}
```

**为飞行中的重复请求做个决定。** 第一次请求还在跑时第二次到了 ， 重试风暴下的常态：

| 策略 | 响应 | 何时用 |
|---|---|---|
| 拒绝 | `409 Conflict` | 客户端能稍后重试；最简单最安全 |
| 等待 | 阻塞等结果（有上限） | 调用方需要同步拿结果 |
| 返回挂起 | `202` + 状态 URL | 长时间运行的副作用 |

不要因为第一次"看起来卡了"就把第二个调用放过去。一个命运未卜的挂起请求，正是重复发生时代价最大的时刻。

**每次调用都有三种结果，不是两种：成功、失败、未知。** 超时告诉你"这次副作用是否生效"都没说。在调用外部之前先记录意图，这样调用和响应之间的崩溃会留下"这有东西待解"的证据，而不是事后又被静悄悄重试的扣费。

**Key 的保留期按"最长重试链"算，不要按"磁盘成本"算。** Key 必须比任何能再投递同一次意图的路径更长：包括一周后被回放的死信队列、任何供应商争议窗口。一个 24 小时 TTL 的 key 落在 7 天的 DLQ 后面，等于预先埋好的重复。

## REST API 模式

### 资源设计

```
GET    /api/tasks              → 列出任务（带查询参数筛选）
POST   /api/tasks              → 创建一个任务
GET    /api/tasks/:id          → 获取单个任务
PATCH  /api/tasks/:id          → 更新一个任务（部分）
DELETE /api/tasks/:id          → 删除一个任务

GET    /api/tasks/:id/comments → 列出任务的评论（子资源）
POST   /api/tasks/:id/comments → 给任务加一条评论
```

### 分页

列表 endpoint 需要分页：

```typescript
// 请求
GET /api/tasks?page=1&pageSize=20&sortBy=createdAt&sortOrder=desc

// 响应
{
  "data": [...],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 142,
    "totalPages": 8
  }
}
```

### 筛选

用查询参数做筛选：

```
GET /api/tasks?status=in_progress&assignee=user123&createdAfter=2025-01-01
```

### 部分更新 (PATCH)

接受部分对象 ， 只更新给到的：

```typescript
// 只有 title 改，其它都保留
PATCH /api/tasks/123
{ "title": "Updated title" }
```

## TypeScript 接口模式

### 用可辨识联合表示变体

```typescript
// 好：每个变体显式
type TaskStatus =
  | { type: 'pending' }
  | { type: 'in_progress'; assignee: string; startedAt: Date }
  | { type: 'completed'; completedAt: Date; completedBy: string }
  | { type: 'cancelled'; reason: string; cancelledAt: Date };

// 调用方得到类型收窄
function getStatusLabel(status: TaskStatus): string {
  switch (status.type) {
    case 'pending': return 'Pending';
    case 'in_progress': return `In progress (${status.assignee})`;
    case 'completed': return `Done on ${status.completedAt}`;
    case 'cancelled': return `Cancelled: ${status.reason}`;
  }
}
```

### Input / Output 分离

```typescript
// Input：调用方提供
interface CreateTaskInput {
  title: string;
  description?: string;
}

// Output：系统返回（包含服务端生成字段）
interface Task {
  id: string;
  title: string;
  description: string | null;
  createdAt: Date;
  updatedAt: Date;
  createdBy: string;
}
```

### 用品牌类型标识 ID

```typescript
type TaskId = string & { readonly __brand: 'TaskId' };
type UserId = string & { readonly __brand: 'UserId' };

// 防止把 UserId 错传给要 TaskId 的地方
function getTask(id: TaskId): Promise<Task> { ... }
```

## 常见借口

| Rationalization | Reality |
|---|---|
| "我们等会再文档化 API" | 类型就是文档。先定义它们。 |
| "现在还不需要分页" | 任何东西到了 100+ 就要。开局就加分页。 |
| "PATCH 太复杂，就用 PUT 吧" | PUT 每次都要完整对象。PATCH 才是客户端真想要的。 |
| "需要时再做版本化" | 没版本化的破坏性变更会破坏消费者。从一开始就为扩展而设计。 |
| "没人用那个未文档化行为" | Hyrum 定律：可观测的，就有人依赖。每个公开行为都是承诺。 |
| "我们维护两套就好" | 多版本叠加维护成本、产生菱形依赖。优先单版本规则。 |
| "内部 API 不需要契约" | 内部消费者也是消费者。契机能防止耦合、让并行工作成为可能。 |
| "接 Idempotency-Key 头就够了" | 头是契约；把 key 和结果一起落库才是实现。接了但不兑现的 key 让客户端以为重试安全 ， 实际上不是。 |
| "我们的队列保证 exactly-once" | 跨消费者崩溃没有任何队列能做到 ， broker 的 ack 与你的副作用并不在同一笔事务里。设计成 at-least-once + 幂等处理。 |
| "重复请求很罕见" | 它们是*相关的*。依赖降级时重试正好是高峰 ， 这是重复最可能出现、最贵的时刻。 |

## 红旗

- 不同条件下返回不同形状的 endpoint
- 跨 endpoint 错误格式不一致
- 校验零散落在内部代码而不是边界
- 改既有字段的破坏性变更（类型变更、删除）
- 列表 endpoint 不分页
- REST URL 里出现动词（`/api/createTask`、`/api/getUsers`）
- 第三方 API 响应未经校验 / 消毒就直接用
- 一条 select 后跟一条 insert 当幂等 key 守卫 ， 那是竞争，不是守卫
- 幂等 key 从 UUID、时间戳或任何"每次重试都重新生成"的东西衍生
- 同样的 key 接了不同请求体，悄无声息地返回第一次的响应
- Key 的保留窗口比"能再投递同一次请求"的路径还短

## 验证

设计完一个 API 后：

- [ ] 每个 endpoint 都有 typed 输入与输出 schema
- [ ] 错误响应遵循同一种格式
- [ ] 校验只在系统边界处发生
- [ ] 列表 endpoint 支持分页
- [ ] 新字段是加性的、可选（向后兼容）
- [ ] 命名跨所有 endpoint 一致
- [ ] API 文档或类型与实现一起提交
- [ ] 改变状态的 endpoint 要么兑现幂等 key，要么文档化"不安全重试"
- [ ] Key 在一次原子操作里宣告，由唯一约束守护
- [ ] 同一 key 带不同请求体是响亮失败，而不是错误地重放旧响应
- [ ] 飞行中重复请求的响应是有意选择（409、等待、或 202），不是随意
- [ ] Key 的保留期比"最长的可重投递路径"（含死信回放）还长
