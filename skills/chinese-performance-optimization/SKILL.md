---
name: performance-optimization
description: 优化前端、后端、查询与数据库的应用性能。在有性能需求、怀疑性能回退、Core Web Vitals 或加载时间要改善、N+1 查询模式需要修、或 profile 找到瓶颈时使用。
---

# Performance Optimization

## 概述

先测量后优化。没有测量的性能工作就是猜 ， 猜导致过早优化，只增加复杂度、不改善真正重要的东西。先 profile，找出实际瓶颈，修它，再测一遍。只优化测量证明重要的事。

## 何时使用

- 规格里有性能要求（加载时间预算、响应时间 SLA）
- 用户或监控报告慢
- Core Web Vitals 分数低于阈值
- 怀疑改动引入了回归
- 做的特性要处理大数据集或高流量

**不用：** 没有证据有问题之前别优化。过早优化增加复杂度，付出比性能收益更多。

## Core Web Vitals 目标

| 指标 | 好 | 需改进 | 差 |
|--------|------|-------------------|------|
| **LCP** (Largest Contentful Paint，最大内容绘制) | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| **INP** (Interaction to Next Paint，交互到下次绘制) | ≤ 200ms | ≤ 500ms | > 500ms |
| **CLS** (Cumulative Layout Shift，累计布局偏移) | ≤ 0.1 | ≤ 0.25 | > 0.25 |

## 优化流程

```
1. MEASURE   → 用真实数据建立基线
2. IDENTIFY  → 找实际瓶颈（不是假设的）
3. FIX       → 处理具体瓶颈
4. VERIFY    → 再测量；保留或回退
5. GUARD     → 加监控或测试防回归
```

### 步骤 1：测量

两条互补路径 ， 两条都用：

- **合成（Lighthouse、DevTools Performance 面板）：** 受控条件、可复现。最适合 CI 回归检测与隔离具体问题。
- **RUM（web-vitals 库、CrUX）：** 真实用户数据、真实条件。验证修复确实改善了用户体验必用。

**前端：**
```javascript
// 合成：在 Chrome DevTools（或 CI）跑 Lighthouse
// Chrome DevTools → Performance 面板 → Record
// Chrome DevTools MCP → 性能 trace

// RUM：在代码里用 web-vitals 库
import { onLCP, onINP, onCLS } from 'web-vitals';

onLCP(console.log);
onINP(console.log);
onCLS(console.log);
```

**后端：**
```javascript
// 响应时间日志
// 应用性能监控（APM）
// 带时序的数据库查询日志

// 简单时序
console.time('db-query');
const result = await db.query(...);
console.timeEnd('db-query');
```

### 从哪里开始测

用症状决定先测什么：

```
什么慢了？
├── 首次页面加载
│   ├── bundle 大？--> 量 bundle 大小，看代码分割
│   ├── 服务端响应慢？--> 在 DevTools Network 瀑布里量 TTFB
│   │   ├── DNS 长？--> 给已知来源加 dns-prefetch / preconnect
│   │   ├── TCP/TLS 长？--> 开 HTTP/2，查边缘部署，keep-alive
│   │   └── Waiting（服务端）长？--> profile 后端，查查询与缓存
│   └── 渲染阻塞资源？--> 看网络瀑布里阻塞的 CSS/JS
├── 交互卡顿感
│   ├── 点击时 UI 卡？--> profile 主线程，找长任务（>50ms）
│   ├── 表单输入卡？--> 看重渲染、受控组件开销
│   └── 动画掉帧？--> 看布局抖动、强制 reflow
├── 导航后的页面
│   ├── 数据加载？--> 量 API 响应时间，查瀑布
│   └── 客户端渲染？--> profile 组件渲染时间，找 N+1 fetch
└── 后端 / API
    ├── 单接口慢？--> profile 数据库查询，查索引
    ├── 全接口慢？--> 看连接池、内存、CPU
    └── 间歇性慢？--> 看锁竞争、GC 暂停、外部依赖
```

### 步骤 2：找出瓶颈

按类别列常见瓶颈：

**前端：**

| 症状 | 可能原因 | 排查 |
|---------|-------------|-------------|
| LCP 慢 | 图片大、阻塞资源、服务端慢 | 看网络瀑布、图片大小 |
| CLS 高 | 图片无尺寸、加载晚的内容、字体跳动 | 看布局位移归因 |
| INP 差 | 主线程 JS 重、DOM 更新大 | 在 Performance trace 里找长任务 |
| 首屏慢 | bundle 大、网络请求多 | 看 bundle 大小、代码分割 |

**后端：**

| 症状 | 可能原因 | 排查 |
|---------|-------------|-------------|
| API 响应慢 | N+1 查询、缺索引、查询未优化 | 查数据库查询日志 |
| 内存涨 | 漏引用、无界缓存、payload 大 | Heap snapshot 分析 |
| CPU 峰值 | 同步重计算、正则回溯 | CPU profile |
| 延迟高 | 缺缓存、重复计算、网络跳 | 在栈里追踪请求 |

### 步骤 3：修瓶颈

修步骤 2 找出的那一个，不要修别的。下面的反模式是常见嫌犯；每条给出规则与识别信号，并在 [references/optimization-patterns.md](references/optimization-patterns.md) 链到带示例的修复。直到代码到了那个位置再开那个，不要提前。

**后端**

- **N+1 查询。** 每行一条查询是最常见的后端瓶颈。把关系用同一查询拿到（join / include），而不是在循环里。 [模式](references/optimization-patterns.md#n1-queries-backend)。
- **无界的数据获取。** 每一个列表接口都按 limit + 稳定顺序分页。 [模式](references/optimization-patterns.md#unbounded-data-fetching)。
- **查询无视索引。** "加索引"是猜；`EXPLAIN ANALYZE` 才是测量。在该出索引的地方出了 `Seq Scan`、`rows=` 估计差一个量级、扫描上面有 `Sort` 节点，每个都指向不同的修法；`rows=` 估计差是统计过时，跑 `ANALYZE` 而非加索引。按查询形态建索引（等值列在前，再是范围或排序列）。普通索引帮不上低选择度主值的查询（用部分索引服务稀有值）、前导通配（需要 trigram 或全文搜索）、或对列施加函数的查询（对表达式建索引，形如 `WHERE lower(email) = ?`），每加一个索引都让每次写付出代价。改完重跑计划；没改计划的索引要回滚。 [模式](references/optimization-patterns.md#queries-that-ignore-their-index)。
- **连接池耗尽。** 信号：*所有*接口同时变慢、时间花在等连接上而不是执行、数据库主要显示空闲会话。一个进程一个池，size 让 `instances × max` 不超数据库连接上限。更大不更快，只是把队列搬到数据库更看不见的地方。在实例无界（serverless、autoscaling）时，用代理（pgbouncer、RDS Proxy）多路复用而非把 `max` 抬高。 [模式](references/optimization-patterns.md#connection-pool-exhaustion)。
- **缺缓存。** 缓存昂贵且读远多于写的东西；缓存已经很快的查询只会多一跳网络和一份 staleness 漏洞。挑层有意识（进程内、共享、CDN）。会改变响应的输入必须在 key 里（租户、locale、权限、特性开关）：漏掉查看者的 key，就是把一用户的数据发给另一用户。挑一种失效策略（TTL、事件或 tag 触发、版本化 key）并明确说出可接受的 staleness 窗口。给热 key 防雪崩：在重新算时服旧值，或把并发未命中合并到单个 in-flight promise 后面。永不缓存必须新鲜的（余额、权限、结账时库存）。 [模式](references/optimization-patterns.md#missing-caching-backend)；请求合并、写策略与负缓存见 `../../references/performance-checklist.md`。

**前端**

- **缺图片优化。** 每张图声明 `width` 和 `height`（CLS）。LCP 图得 `fetchpriority="high"`、用现代格式（AVIF、WebP）通过 `<picture>`、并配 `srcset`/`sizes` 做分辨率切换；折以下图片用 `loading="lazy"` 与 `decoding="async"`。 [模式](references/optimization-patterns.md#missing-image-optimization-frontend)。
- **不必要的重渲染。** 渲染时创建的对象或函数每次都是新引用，让拿到它的每个子组件都重渲染。把常量提上去；`React.memo` 与 `useMemo` 留给 profile 显示的贵活 ， 一味用是另一种代价。 [模式](references/optimization-patterns.md#unnecessary-re-renders-react)。
- **大 bundle。** 现代打包器自己就能对 ESM named import 做 tree-shake；真正的收益是路由级代码切分与 `Suspense` 后面的重、罕见特性。改 import 风格之前先 profile。 [模式](references/optimization-patterns.md#large-bundle-size)。

### 步骤 4：验证（保留或回退）

修是假设，直到重测。这一步决定它活下来。

**按量基线的同一方式再量一遍：** 同一条命令、同样的条件、同样的固定预算（墙钟、样本数或请求数）。冷缓存取基线、热缓存出结果，量的就是缓存，不是你的改动。

**一次只改一个。** 三个优化一起上的产出只有一个数字、你也没法归因。如果必须一起上，先把每个隔离着量。

**赢过噪声，不只是均值。** 把测量重复几次，对比差值与"次间方差"。在 ±5% 方差里的 3% 收益不是收益；是另一个样本。

然后，严格决策：

| 对基线的结果 | 动作 |
|---|---|
| 越阈值、测试绿 | **保留。** 用前后数字写进 commit message。 |
| 在噪声范围里（无可测量变化） | **回退。** |
| 更差 | **回退。** |
| 好了，但测试红 | **回退。** 穿着赢的衣服的回归。 |

**"中性"是回退、不是保留。** 这是团队最常跳过的步骤：改动已经写完了，扔掉感觉浪费，它就默默合上了，代码库积累下从未给过回报的复杂度。你保留了什么、就维护它一辈子。让它自证。

**正确性守门指标。** 套件保持绿*并且*数字动。"优化"用扔掉产品需要的工作（跳过校验、缓存必须新鲜的东西、删一个本来承重的 `await`）赢了，那是回归，不是赢。

#### 把每次尝试都记下来，含回退的

被回退的工作不会留在 git 历史里，这就是为什么同一个死点子下个季度又被试一次。留一份短账，让被丢弃的 点子保持丢弃：

| 点子 | 基线 → 结果 | 判定 | 原因 |
|---|---|---|---|
| 把行组件 memo 起来 | INP 240ms → 235ms | 回退 | 在噪声内（±15ms）。行不是瓶颈。 |
| 把列表虚拟化 | INP 240ms → 90ms | 保留 | 长任务在 trace 里消失。 |
| 给 API 来源 preconnect | LCP 2.8s → 2.8s | 回退 | 本来就是同源。 |

PR 描述里一节、或仓库里一份 `PERF.md`，都行。重要的是下个人（或下一个 Agent）跑下一个实验前先读，别重跑一次已经失败过的。

### 步骤 5：防回归

守卫用户真正能感到的指标，不是所有能拿到的数字。用当初让这笔修复合理的那同一个 LCP、INP、p95 延迟或其它首要指标。

当面向用户时用两层互补：

- **合成 CI 闸：** 用性能预算在合并前抓可复现的回归。重复噪声较大的测量、或用中位数 / 趋势对比，让次间方差不会把闸变成 flaky。
- **现场监控：** 在 RUM 数据上对有意义的 p75 变化告警。用归因过的 `web-vitals` 数据定位原因；把 CrUX 的滚动窗口当确认、不当立即告警。

任一层响了，回到步骤 1 再建立一份新基线，再提修。

**设预算并强制：**

```
JavaScript bundle: < 200KB gzipped（首屏）
CSS: < 50KB gzipped
图片: < 200KB 每张（折以上）
字体: < 100KB 总
API 响应时间: < 200ms（p95）
Time to Interactive: < 3.5s 在 4G
Lighthouse Performance score: ≥ 90
```

**CI 中强制：**
```bash
# bundle size check
npx bundlesize --config bundlesize.config.json

# Lighthouse CI
npx lhci autorun
```

## 参见

详细性能清单、优化命令与反模式参考，见 `../../references/performance-checklist.md`。


## 常见借口

| Rationalization | Reality |
|---|---|
| "我们等会再优化" | 性能债会叠加。现在把显而易见的反模式修了，微优化往后放。 |
| "在我机器上挺快" | 你的机器不是用户的。在代表性硬件与网络上 profile。 |
| "这次优化显而易见" | 如果你没测，你不知道。先 profile。 |
| "用户察觉不到 100ms" | 研究显示 100ms 延迟影响转化率。用户察觉到的比你以为多。 |
| "框架会把性能管好" | 框架能阻止一些问题，但修不了 N+1 查询或超大 bundle。 |
| "查询慢，加索引" | 先看计划。索引可能已经在但不可用，且每个索引永久让写付代价。 |
| "直接缓存它" | 缓存已经很便宜的调用啥都没买到、多一份 staleness 漏洞。缓存既贵又读远多于写的东西。 |
| "把池 size 抬高，连接用光了" | 比数据库能承接更大的池，只是把队列搬到更看不见的地方。找谁占着连接。 |
| "没帮多少，但也没害" | 中性改动要回退。你要永久维护它、没换回来任何东西。 |
| "我们写都写了，保留一下" | 沉没成本。测量不在乎改动花了多久。 |
| "提升是显然的，不用再量" | 那重测便宜、能证明它。未量化的赢，是中性复杂度默默上线的方式。 |

## 红旗

- 没有 profile 数据支撑的"优化"
- 数据获取里有 N+1 查询模式
- 加索引没在前后用查询计划支撑
- 缓存 key 漏掉响应所依赖的输入（租户、locale、查看者）
- 缓存没设 staleness 窗口、也没失效策略
- 用抬高连接池 size 应对耗尽、不查谁占着连接
- 列表接口没分页
- 图片没尺寸、懒加载或响应式尺寸
- bundle 涨了没人 review
- 生产里没性能监控
- 到处 `React.memo`、`useMemo`（用多跟用少一样差）
- 没重测就留的优化
- 几个优化并到一个测量里，没法归因到单个改动
- "赢"是要改、跳过或删一条测试才换来的
- 同一个失败的优化试了不止一次，因为没人记第一次

## 验证

任何性能相关改动之后：

- [ ] 有前后数字（具体值）
- [ ] 用与基线同一种方式重测（同一命令、同一条件）
- [ ] 改进超过次间方差，不只是均值
- [ ] 没超过基线的改动被回退，而非作为"中性"保留
- [ ] 每次尝试都记下，保留与回退都记，所以死了的 点子不会被再跑
- [ ] 找到了具体瓶颈并处理
- [ ] Core Web Vitals 在 "Good" 区间
- [ ] bundle size 没有显著增加
- [ ] 新数据获取代码里没有 N+1
- [ ] 任何新索引被前后 query plan 支撑，且其写代价被考虑
- [ ] 任何新缓存都说明它用啥 key、如何过期
- [ ] 测过的用户侧指标有合成预算或现场监控能发现回退
- [ ] 已有测试仍通过（优化没破坏行为）
