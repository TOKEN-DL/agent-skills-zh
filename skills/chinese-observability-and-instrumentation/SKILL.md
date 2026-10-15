---
name: observability-and-instrumentation
description: 给代码埋点让生产行为可见、可诊断。在加日志、指标、追踪或告警时使用。在让任何会在生产跑的特性需要证据时使用。在生产问题被报上来但你从现有数据看不出发生了什么时使用。
---

# Observability and Instrumentation

## 概述

观测不到自己行为的代码，没法运营。Observability 是从外部回答"系统在做什么、为什么？"的能力，用代码吐出的遥测。Instrumentation 不是上线后的补丁 ， 它和特性一起写，就像测试一样。如果一个特性上线时没有遥测，第一份用户报上来的 bug 就成了考古而不是查询。

## 何时使用

- 构建任何会在生产跑的特性
- 新增服务、endpoint、后端任务、外部集成
- 一场生产事故花了太久才定位（"我们看不出发生了什么"）
- 配置或评审告警规则
- 评审一个加了 I/O、重试、队列或跨服务调用的 PR

**不是用于：**
- 排查正在发生的故障 ， 用 `debugging-and-error-recovery` Skill（Observability 是让那个 Skill 下次变快）
- Profile 与优化已测出的慢 ， 用 `performance-optimization` Skill
- 上线日监控清单与回滚触发器 ， 看 `shipping-and-launch` Skill；这个 Skill 管的是供给它的埋点

## 流程

### 1. 在埋点前先把"能跑"定义清楚

没有问题的遥测是噪声。加任何埋点之前，写 2-4 个值班工程师会问这个特性的问题：

```
特性：checkout 支付重试
值班会问的问题：
1. 首次尝试成功 vs 重试后成功的支付比例？
2. 当支付最终失败时，为什么？（供应商错误？超时？校验？）
3. 支付供应商是否比平时慢？
→ 下面每个信号都必须能回答这些之一。
```

如果连问题都说不出来，你就还没准备好埋点 ， 会什么日志都写、什么都学不到。

### 2. 给每个问题挑对的信号

| 信号 | 回答 | 成本画像 | 例子 |
|---|---|---|---|
| **结构化日志** | "在这种情况下发生了什么？" | 每事件一记；随流量增长 | `payment_failed` 携供应商错误码 |
| **指标** | "有多频繁/多快，聚合看？" | 每序列固定一条；查询便宜 | 供应商调用的 p99 延迟 |
| **追踪** | "时间花在了跨服务的哪里？" | 每请求一条；通常采样 | 一单慢的 checkout，按跳数拆开 |

经验法则：metrics 告诉你**什么**出了事，traces 告诉你**哪里**，logs 告诉你**为什么**。

### 3. 结构化日志

记录事件，不是散文。每行日志是一个 JSON 对象，带稳定的事件名和机器可读字段：

```typescript
// 反例：字符串内插 ， 不可查、不一致
logger.info(`Payment ${id} failed for user ${userId} after ${n} retries`);

// 好：稳定事件名 + 结构化字段
logger.warn({
  event: 'payment_failed',
  paymentId: id,
  provider: 'stripe',
  errorCode: err.code,
  attempt: n,
}, 'payment failed');
```

**日志级别 ， 用一致：**

| 级别 | 含义 | 值班动作 |
|---|---|---|
| `error` | 不变量被破坏；可能需要有人来处理 | 排查 |
| `warn` | 降级但已处理（重试成功，用了兜底） | 关注趋势 |
| `info` | 重要的业务事件（订单下单，任务完成） | 无 |
| `debug` | 调试细节 | 生产默认关 |

**关联 ID 是强制的。** 在系统边界生成（或接受）一个请求 ID，并把它附到每一行日志、span、外部调用。没有它，没法从交叉的日志里把单个请求重建起来：

```typescript
// Express：每个请求一个子 logger，向下游传 ID
app.use((req, res, next) => {
  req.id = req.headers['x-request-id'] ?? crypto.randomUUID();
  req.log = logger.child({ requestId: req.id });
  res.setHeader('x-request-id', req.id);
  next();
});
```

**当多个入口写入同一个日志，记一下入口点。** 关联 ID 标识一次跑，但不说是哪条代码路径起的它。同一个 job 被调度器、重放端点和手工 CLI 跑到，在同一个 sink 里产出可互换的行，所以要把一行归因就只能用排除法 ， 交叉读调度器的历史、进程表、部署日志 ， 这种论断只要外部记录恰好还在就成立。在每次跑的起点打一个"入口点"印章，紧挨着关联 ID，同样一起传下去：

```typescript
// 给每条入口用一次 helper：这条跑的 logger 带这两个字段。
// 用 `entryPoint` 不是 `source` ， ECS 把 `source.*` 留给网络字段。
export const runLog = (entryPoint: 'scheduler' | 'replay_endpoint' | 'cli', runId: string) =>
  logger.child({ entryPoint, requestId: runId });

// 调度 tick       -> runLog('scheduler', crypto.randomUUID())
// POST /jobs/:id/replay -> runLog('replay_endpoint', req.id)
// CLI 调用         -> runLog('cli', process.env.RUN_ID ?? crypto.randomUUID())
```

这两个字段必须跨过与关联 ID 一样的边界 ， 队列元数据、HTTP 头 ， 否则 worker 再衍生出入口点就要猜。一个仅"和入口点相关"的字段只是提示，不是归因：能调用这个 job 的任何东西都能复现它。

**永远不日志密钥、token、密码或完整 PII。** 这是 `security-and-hardening` Skill 的一条硬规则 ， 遥测管道是经典的数据泄漏路径。用白名单字段；不要把整份请求体都记录。

### 4. 指标

对请求驱动的服务，给每个 endpoint 与每个外部依赖做 **RED**：**R**ate（每秒请求）、**E**rrors（失败率）、**D**uration（延迟直方图，不是均值）。对资源（队列、池、主机），用 **USE**：**U**tilization、**S**aturation、**E**rrors。

如 tracing，供应商无关的路径是 OpenTelemetry metrics API（与步骤 5 同一套 SDK 与上下文）。下面的例用 Prometheus 的 `prom-client` ， 这是常见后端之一，不是唯一选择；RED/USE 与基数规则两边都是同样的。

```typescript
import { Histogram } from 'prom-client';

const httpDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP 请求时长',
  labelNames: ['method', 'route', 'status_class'],  // '2xx'，不是 '200'
  buckets: [0.05, 0.1, 0.25, 0.5, 1, 2.5, 5],
});
```

**基数是失败模式。** 每个唯一 label 组合就是独立时间序列。Label 必须来自小且固定的集合（路由模板、状态类、供应商名）。永远不要把 user ID、原始 URL、错误消息或任何无界值当 label ， 那属于 logs 与 traces。

```
OK 当 label:    route="/api/tasks/:id"   status_class="5xx"   provider="stripe"
绝不:           user_id、email、request_id、完整 URL、错误消息文本
```

跟踪平均值从来不要，看分位数永远要看：平均值把"占 1% 用户的糟糕体验"藏起来。用直方图读 p50/p95/p99。

### 5. 分布式追踪

用 OpenTelemetry ， 它是供应商无关的标准、自动埋点覆盖了 HTTP、gRPC 与常见 DB 客户端，几乎零代码：

```typescript
// tracing.ts ， 必须先于其它东西被 import
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';

const sdk = new NodeSDK({
  serviceName: 'checkout-service',
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

只在有意义的内部工作单元上加手工 span（如 `applyDiscounts`、`chargeProvider`），并附上值班会用来过滤的属性。上下文跨过每一个 async 边界传过去 ， HTTP 头、队列消息元数据 ， 否则 trace 在裂口处死掉。默认 head-based 低采样率采样；如果后端支持 tail sampling 就把错误全采。

### 6. 告警

**按用户能感到的症状告警**，而不是按原因：

```
症状（值得叫醒）：           原因（仪表板，不是叫醒）：
错误率 > 1% 持续 5 分钟        CPU 在 85%
p99 延迟 > 2s                一个 pod 重启了
队列年龄 > 10 分钟           磁盘在 70%
```

按原因告警会在没出毛病时乱响、放过你预料不到的故障。按症状告警正好在用户受伤时响，无所谓原因。

每条告警的规则：

1. **必须可执行。** 如果响应是"忽略它，它会自愈"，删掉这条告警。
2. **它链一条 runbook** ， 即便只有三行：什么意思，先查什么，找谁升级。
3. **它有阈值与持续时长**，由 SLO 或历史数据支撑，不是猜。
4. 只用两个严重级：**叫醒**（面向用户，立刻行动）和**开 ticket**（退化，本周处理）。再加一层就是噪声，教会人忽略一切。

#### 写 Runbook

上面第 2 条要求每条告警链一条 runbook。runbook 的工作是不让读者动脑就回答三个问题：发生什么，先查什么，没解决就找谁。存在 `docs/runbooks/`，按告警命名。

**最小可用的 runbook（三行）：**

```markdown
# Runbook: High Error Rate on /api/tasks
**意思：** 数据库连接池可能用尽，或一次糟糕的部署。
**先查：** `SELECT count(*) FROM pg_stat_activity WHERE backend_type = 'client backend';`
  ， 如果 count > pool 限制，看第 2 步。（对换为你用的数据库的等价命令。）
**找谁：** #db-oncall 或工程 on-call 轮值。
```

**何时扩到三行以上：** 只有在单靠"先查这一条"还不够让你决策时，加。一份 5 步的 runbook 覆盖最常见的三种原因，比一份 20 步的覆盖每种边界情形的、没人看的好。

**保持 runbook 的时效。** 关闭每次用过的 runbook 的事故时，一并修 runbook ， 过时的 runbook 建立虚假的信心。如果一步是错的或缺的，先修，再标事故已处理。

### 7. 校验遥测本身

Instrumentation 是代码；它也可能是错的。宣告工作完成前，触发那些路径、看实际输出：

- 强制 staging 里出一个错 → 用 `requestId` 在日志里找到它，确认字段是结构化的（不是 `[object Object]`）
- 发测试流量 → 确认指标序列出现了、带的 label 与预期且数值合理
- 在追踪 UI 里跟一条跨服务的请求 → 跨度不破
- 每条新告警发一次（临时调阈值）→ 确认它落到对的频道、runbook 链接打得开

## 常见借口

| Rationalization | Reality |
|---|---|
| "功能跑起来我再加日志" | "之后"会变成"第一次事故之后"，而那恰恰是发现自己是盲的最贵时刻。边写边埋。 |
| "日志多就是观测强" | 无结构噪声让事故变慢、不是更快。三条可查的事件胜三百行散文。 |
| "console.log 现在够用了" | 无结构输出无法过滤、关联或告警。结构化 logger 多花 5 分钟一次。 |
| "出问题我们看板子就好" | 没有定义问题时搭的看板让你看到一切、除了答案。从值班问题开始。 |
| "什么都加告警，之后我们调" | 噪声告警教会人忽略它。调优从不发生；漏掉的真告警倒是来了。 |
| "把 user ID 当 metric label 更易 debug" | 这也会让你的指标后端倒下。高基数查找属于 logs 与 traces。 |
| "我们的两个服务加追踪是过度" | 跨服务延迟问题日志答不了，两个服务就已出现。自动埋点让代价微小。 |

## 红旗

- 一个加 retries、队列或外部调用的特性 PR，零新增遥测
- 用字符串内插拼日志行而不是结构化字段
- 没有关联 / 请求 ID ， 每行日志都是孤儿
- 一条日志流被调度器、webhook 与手工跑喂进来，但没有字段说明哪条出的行
- 指标用 user ID、原始 URL、错误消息文本当 label（基数炸弹）
- 延迟用平均数跟踪，没分位数
- 告警每天响、被确认但不行动
- 告警挂在原因（CPU、内存）叫醒人，而用户侧错误率没监控
- 密钥、token、完整请求体出现在日志里
- "在我机器上能跑" 是一个生产特性健康的唯一证据

## 验证

为一个特性做完埋点，确认：

- [ ] 值班会问的问题写下来了，每个信号能映射到其中一个
- [ ] 全部日志输出是结构化（JSON），每行有稳定事件名和关联 ID
- [ ] 凡是被多条入口写入的日志 sink 都带一个入口点字段，在跑开始时设上、跟关联 ID 一起传下去，不能下游再猜
- [ ] 任何日志行都没有密钥、token 或未经去标识化的 PII（抽查实际输出）
- [ ] 每个新 endpoint 与每个外部依赖都有 RED 指标，label 集合有界
- [ ] 延迟用直方图；p95/p99 可查
- [ ] 在追踪 UI 里能端到端跟一个请求，跨度不断
- [ ] 每条新告警是按症状、链 runbook、测发过至少一次
- [ ] 在 staging 诱导出一个故障，只用遥测、不读源码也能定位

取一份汇总版（含上线前 instrumentation 闸），见 `../../references/observability-checklist.md`。
