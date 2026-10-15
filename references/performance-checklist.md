# 性能检查清单（Performance Checklist）

面向 Web 应用性能的快速参考清单。与 `chinese-performance-optimization` Skill 配合使用。

## 目录

- [Core Web Vitals 目标值](#core-web-vitals-目标值)
- [TTFB 诊断](#ttfb-诊断)
- [前端清单](#前端清单)
- [后端清单](#后端清单)
- [缓存策略](#缓存策略)
- [测量命令](#测量命令)
- [常见反模式](#常见反模式)

## Core Web Vitals 目标值

| 指标 | 良好 | 需改进 | 差 |
|--------|------|------------|------|
| LCP（Largest Contentful Paint，最大内容绘制时间） | ≤ 2.5s | ≤ 4.0s | > 4.0s |
| INP（Interaction to Next Paint，下一次绘制的交互延迟） | ≤ 200ms | ≤ 500ms | > 500ms |
| CLS（Cumulative Layout Shift，累计布局偏移） | ≤ 0.1 | ≤ 0.25 | > 0.25 |

## TTFB 诊断

当 TTFB（Time To First Byte，首字节时间）偏慢（> 800ms）时，按 DevTools（开发者工具）Network 水流逐项排查：

- [ ] **DNS 解析**慢 → 为已知来源添加 `<link rel="dns-prefetch">` 或 `<link rel="preconnect">`
- [ ] **TCP / TLS 握手**慢 → 启用 HTTP/2，考虑边缘部署，校验 keep-alive
- [ ] **服务端处理**慢 → 后端打点、检查慢查询、加缓存

## 前端清单

### 图片
- [ ] 图片使用现代格式（WebP、AVIF）
- [ ] 图片有响应式尺寸（`srcset` 与 `sizes`）
- [ ] 图片与 `<source>` 元素显式声明 `width` 与 `height`（防止艺术指导下的 CLS）
- [ ] 首屏以下图片使用 `loading="lazy"` 与 `decoding="async"`
- [ ] 首屏 / LCP 图片使用 `fetchpriority="high"`，不延迟加载

### JavaScript
- [ ] 初始加载 Bundle（打包后产物）大小 ≤ 200KB gzip 压缩后
- [ ] 对路由与重型特性使用动态 `import()` 做代码分割
- [ ] 启用 tree shaking（确认依赖提供 ESM（ECMAScript Modules，ECMAScript 模块）并标记 `sideEffects: false`）
- [ ] `<head>` 中无阻塞 JavaScript（用 `defer` 或 `async`）
- [ ] 重计算卸载到 Web Worker（如适用）
- [ ] 对相同 props 重渲染的组件使用 `React.memo()`
- [ ] `useMemo()` / `useCallback()` 仅在打点显示有收益时使用
- [ ] 长任务（> 50ms）拆块以保持主线程可用，这是 INP 的主要杠杆
- [ ] 长循环里使用 `yieldToMain` 模式，使输入事件可以在块之间运行
- [ ] 在支持时优先用现代调度 API：`scheduler.yield()`（首选）、带优先级的 `scheduler.postTask()`、`isInputPending()` 仅在需要时让步
- [ ] 可推迟、非紧急工作用 `requestIdleCallback`（分析 flush、预取、预热）
- [ ] 非关键工作移出事件处理器（如分析、日志），避免延迟对交互的响应
- [ ] 第三方脚本用 `async` / `defer` 加载、审计大小，并对重型资源（如聊天挂件、嵌入）前面挡一层 facade（外观代理）

### CSS
- [ ] 关键 CSS 内联或预加载
- [ ] 非关键样式不阻塞渲染
- [ ] 生产环境无 CSS-in-JS 运行时代价（用构建期抽取）

### 字体
- [ ] 限制在 2–3 个字体族，每个 2–3 个字重（每多一个字重就多一个请求）
- [ ] 仅使用 WOFF2 格式（体积最小、通用支持，跳过 WOFF / TTF / EOT）
- [ ] 尽量自托管（第三方字体 CDN 会带来 DNS + TCP + TLS 往返）
- [ ] LCP 关键字体预加载：`<link rel="preload" as="font" type="font/woff2" crossorigin>`
- [ ] `font-display: swap`（非关键字体用 `optional`）避免 FOIT（Flash of Invisible Text，字体加载前不可见文本闪烁）阻塞渲染
- [ ] 通过 `unicode-range` 子集化，仅下发每页需要的字形
- [ ] 需要多字重 / 多样式时考虑可变字体（一个文件替代多个）
- [ ] 用 `size-adjust`、`ascent-override`、`descent-override` 调整后备字体度量，降低字体切换时的 CLS
- [ ] 任何自定义字体之前先考虑系统字体栈

### 网络
- [ ] 静态资源使用长 `max-age` + 内容哈希缓存
- [ ] API 响应在适当时缓存（`Cache-Control`）
- [ ] 启用 HTTP/2 或 HTTP/3
- [ ] 为已知来源预连接（`<link rel="preconnect">`）
- [ ] 在关键非图片资源上使用 `fetchpriority`（例如关键的 `<link rel="preload">`、首屏 `<script>`），不只用在 `<img>`
- [ ] 无不必要重定向

### 渲染
- [ ] 无 layout thrashing（强制同步布局）
- [ ] 动画使用 `transform` 与 `opacity`（GPU 加速）
- [ ] 长列表使用虚拟化（如 `react-window`）
- [ ] 无不必要的整页重渲染
- [ ] 屏幕外区块使用 `content-visibility: auto` 与 `contain-intrinsic-size`，跳过不可见区的布局 / 绘制
- [ ] 无 `unload` 事件处理器，HTML 响应无 `Cache-Control: no-store`，保留 bfcache（back/forward cache，后退 / 前进缓存）资格

## 后端清单

### 数据库
- [ ] 无 N+1 查询模式（用预加载 / join）
- [ ] 查询有合适的索引
- [ ] 列表端点分页（绝不要 `SELECT * FROM table`）
- [ ] 已配置连接池
- [ ] 已开启慢查询日志

#### 查询计划
- [ ] **修复前**就抓取 `EXPLAIN ANALYZE`，而不是只在修复后，它就是基线
- [ ] 大表上的 `Seq Scan` 已理解：是缺索引、索引不可用，还是确实不值得加索引
- [ ] 估算与实际 `rows=` 在一个数量级以内（若不是，先刷新统计信息再动索引）
- [ ] 没有本可被复合索引吸收的 `Sort` 节点
- [ ] 变更后重新检查计划，若索引未改变计划，应回退

#### 索引策略
- [ ] 复合索引列顺序先等值、再范围 / 排序
- [ ] 索引覆盖查询形态（过滤 + 排序），而不是孤立地覆盖某一列
- [ ] 对热读路径考虑 covering index（覆盖索引，index-only scan 避免回表）
- [ ] 不为低选择性列的*主导值*建索引；偏态索引仍能服务于"罕见值"查询（`WHERE status = 'failed'`）
- [ ] 查询使用函数时（如 `lower(email)`），用表达式索引
- [ ] 前缀通配搜索用全文或 trigram 索引，而不是 B-tree
- [ ] 对写密集表衡量写代价（每个索引都要为每次 `INSERT` / `UPDATE` 付税）
- [ ] 删除未使用与重复索引（它们增加写成本、却买不来收益）

#### 连接池
- [ ] 每个进程一个池，而不是每请求或每模块一个
- [ ] `实例数 × pool 上限` 保持在数据库 `max_connections` 以下
- [ ] 设置 `connectionTimeoutMillis`，使耗尽时快速失败而不是无限排队
- [ ] 扩容前先诊断耗尽：找出谁持有连接（长事务、缺失 `await`、泄漏的 client）
- [ ] Serverless / 自动伸缩场景前接多路复用代理（pgbouncer、RDS Proxy），而不是用更大的池

### API
- [ ] 响应时间 < 200ms（p95）
- [ ] 请求处理器中无同步重计算
- [ ] 批量操作代替循环单次调用
- [ ] 响应压缩（gzip / brotli）
- [ ] 适当的缓存（内存、Redis、CDN）

### 基础设施
- [ ] 静态资源用 CDN
- [ ] 服务器靠近用户（或边缘部署）
- [ ] 已配置横向扩缩（按需）
- [ ] 负载均衡器的健康检查端点

## 缓存策略

决策依据（哪一层、用什么失效策略、什么永不缓存）放在 `chinese-performance-optimization` Skill 里。本节覆盖读写模式与检查清单。

### 读写模式

| 模式 | 工作方式 | 适用场景 | 注意 |
|---|---|---|---|
| **Cache-aside**（懒加载） | 应用查缓存，未命中则读源站并回填 | 默认选择；读密集、能容忍首次冷命中 | 每次未命中都会打源站，因此需要防击穿 |
| **Read-through** | 缓存层自己在未命中时加载 | 想把加载路径集中在一处，而不是每个调用点都重复 | 隐藏源站延迟；慢源站看起来像慢缓存 |
| **Write-through** | 写同时到达缓存与源站，同步 | 写入后读取绝不能看到旧值 | 给每次写入增加缓存延迟 |
| **Write-behind**（write-back） | 写命中缓存，源站异步更新 | 写密集，且源站是瓶颈 | 若缓存先于 flush 死亡则有数据丢失窗口。需要可辩护的持久性 |

### 负缓存（Negative Caching）

把"结果不存在"也缓存起来。一个每次查找都未命中的 key（循环探测的不存在用户 ID、未找到的资源）会让每次请求都打到源站，这种"缓存"只保护了幸福路径。

- 用一个显式的"未找到"哨兵值，配合比正常条目**更短**的 TTL
- 负缓存 TTL 足够短，让新建资源及时可见
- 绝不要让源站*错误*成为负缓存条目，否则失败的一分钟会变成持续的失败

### 请求合并（请求合并防击穿，Request Coalescing）

一次重计算、N 个等待者。避免热 key 过期时把全部并发负载都打到源站：

```typescript
const inFlight = new Map<string, Promise<unknown>>();

function loadOnce<T>(key: string, fetcher: () => Promise<T>): Promise<T> {
  const existing = inFlight.get(key) as Promise<T> | undefined;
  if (existing) return existing;
  const p = fetcher().finally(() => inFlight.delete(key));
  inFlight.set(key, p);
  return p;
}
```

对共享缓存，相同思路需要分布式锁，或 `stale-while-revalidate`，让等待者返回旧值而不是阻塞。

### 缓存检查清单
- [ ] 已先度量被缓存调用确实昂贵（缓存一个快速调用多一跳、毫无收益）
- [ ] 读写比支持缓存（重读远多于写）
- [ ] 缓存 key 包含响应会随其变化的所有输入：租户、查看者、权限、特性开关
- [ ] 未把每用户数据缓存到不标识用户的 key 下
- [ ] 选择了一种失效策略（TTL、事件 / 标签、或带版本的 key），而不是意外的混合
- [ ] 可接受的陈旧窗口已写明，而不是默认敲入的 TTL 数字
- [ ] 热 key 有防击穿（合并、锁、或 `stale-while-revalidate`）
- [ ] 负结果用更短 TTL 缓存；源站错误永不缓存
- [ ] 淘汰策略与内存上限已设置（无上限缓存就是内存泄漏）
- [ ] 监控命中率，没人度量的缓存只是假设，低命中率就是纯开销
- [ ] 没有把"过期即正确性 bug"的内容缓存（余额、权限、结账库存）

## 测量命令

### INP 现场数据与 DevTools 工作流

1. **先看现场数据**，查 [CrUX Vis](https://developer.chrome.com/docs/crux/vis) 或你的 RUM（Real User Monitoring，真实用户监控）工具的真实用户 INP，再做优化
2. **定位慢交互**，打开 DevTools → Performance 面板 → 边交互边录制；观察点击 / 按键触发的长任务
3. **中端 Android 真机测试**，INP 问题常常只在较慢硬件上出现；用真机或 DevTools CPU 节流（4×–6× 降速）

```bash
# Lighthouse CLI
npx lighthouse https://localhost:3000 --output json --output-path ./report.json

# Bundle 分析
npx webpack-bundle-analyzer stats.json
# 或 Vite：
npx vite-bundle-visualizer

# 检查 Bundle 大小
npx bundlesize
```

```javascript
// 代码中接入 Web Vitals
import { onLCP, onINP, onCLS } from 'web-vitals';
onLCP(console.log);
onINP(console.log);
onCLS(console.log);

// 带交互级细节的 INP（attribution 构建）
import { onINP } from 'web-vitals/attribution';
onINP(({ value, attribution }) => {
  const { interactionTarget, inputDelay, processingDuration, presentationDelay } = attribution;
  console.log({ value, interactionTarget, inputDelay, processingDuration, presentationDelay });
});
```

## 常见反模式

| 反模式 | 影响 | 修复 |
|---|---|---|
| N+1 查询 | 数据库负载线性增长 | 用 join、includes 或批加载 |
| 无界查询 | 内存耗尽、超时 | 始终分页、加 LIMIT |
| 缺失索引 | 数据增长后读变慢 | 为过滤 / 排序列加索引 |
| 不读计划就建索引 | 付出写成本、读收益未证明 | 修复前后都跑 `EXPLAIN ANALYZE`；计划未变则回退 |
| 冗余 / 未使用索引 | 每次写入都付税 | 审计使用统计、删除无读索引 |
| 每请求一个连接池 | 高负载下打满 `max_connections` | 每进程一个池；serverless 用代理 |
| 缓存 key 漏掉查看者 | 一个用户数据被发给另一用户 | key 包含租户、查看者、权限 |
| 无界缓存 | 披着优化外衣的内存泄漏 | 设置淘汰策略与内存上限 |
| 热 key 上的缓存击穿 | 过期时源站扛全部并发负载 | 合并未命中，或 `stale-while-revalidate` |
| Layout thrashing | 抖动、丢帧 | 先批量读 DOM，再批量写 |
| 未优化图片 | LCP 慢、带宽浪费 | 用 WebP、响应式尺寸、懒加载 |
| 大体积 Bundle | Time to Interactive 慢 | 代码分割、tree shaking、审计依赖 |
| 阻塞主线程 | INP 差、UI 无响应 | 用 `scheduler.yield()` / `yieldToMain` 切长任务、卸载到 Web Worker |
| 内存泄漏 | 内存持续增长、最终崩溃 | 清理监听器、定时器、refs |