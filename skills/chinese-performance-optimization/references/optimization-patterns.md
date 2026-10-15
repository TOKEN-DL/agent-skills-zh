# 性能优化模式

`performance-optimization` Skill 第 3 步里提到的反模式的成例修复。Skill 给出规则与识别信号；本文给出一份具体实现。第 2 步识别出瓶颈后再开需要的章节 ， 不打算自上而下读。

清单、测量命令、TTFB 诊断、缓存的读 / 写策略都在 `../../../references/performance-checklist.md`。

## N+1 查询（后端）

```typescript
// 反：N+1 ， 每个 task 一条查 owner
const tasks = await db.tasks.findMany();
for (const task of tasks) {
  task.owner = await db.users.findUnique({ where: { id: task.ownerId } });
}

// 正：单条带 join / include
const tasks = await db.tasks.findMany({
  include: { owner: true },
});
```

## 无界的数据获取

```typescript
// 反：拉全部记录
const allTasks = await db.tasks.findMany();

// 正：带 limit + 稳定排序分页
const tasks = await db.tasks.findMany({
  take: 20,
  skip: (page - 1) * 20,
  orderBy: { createdAt: 'desc' },
});
```

## 查询无视索引

"加索引"是猜；查询计划才是测量：

```sql
EXPLAIN ANALYZE
SELECT id, title FROM tasks
WHERE owner_id = 42 ORDER BY created_at DESC LIMIT 20;
```

输出里有三件事决定修法：

| 你看到了 | 意思 |
|---|---|
| 在大表上你预期走索引却出现 `Seq Scan` | 这个谓词上没有可用索引 |
| `rows=` 估计值比实际差一个量级 | 统计过时；规划器在错信息上做选择 |
| 扫描节点上面有 `Sort` 节点 | 索引能覆盖过滤但覆盖不了 `ORDER BY` |

为**查询的形态**建索引，而不是为孤立的列。在复合索引里，等值列在前，再是范围或排序列：

```sql
CREATE INDEX idx_tasks_owner_created ON tasks (owner_id, created_at DESC);
```

**索引帮不上忙的情况：**

| 场景 | 为什么 |
|---|---|
| 低选择性、查的是 dominant 值（一个 `status` 列 95% `active`，按 `active` 过滤） | 顺序扫描确实更便宜；规划器会无视索引。按稀有值过滤是反例，部分索引就派上用场 |
| 前导通配（`LIKE '%term'`） | 没有前缀 B 树就不能 seek；要 trigram 或全文 |
| 对列施加函数（`WHERE lower(email) = ?`） | 纯列索引用不上；要改为对表达式建索引 |
| 写频繁的表 | 每个索引都是每次 `INSERT` / `UPDATE` 的税；既量读的收益，也量写的代价 |

改完再跑一次 `EXPLAIN ANALYZE`。一个没改计划的索引要回退（第 4 步），它还免不了：每次写仍然要付。

## 连接池耗尽

信号很特别：**所有**接口同时变慢，慢的时间花在等连接、不是执行上，数据库里大部分会话显示空闲。

```typescript
// 反：每请求或每模块一个池 ， 在 serverless 下被实例数一乘就耗光
// 数据库的连接上限
// 正：一个进程一个池，相对数据库上限定尺寸
const pool = new Pool({
  max: 10,                        // instances × max 不能超过 max_connections
  idleTimeoutMillis: 30_000,
  connectionTimeoutMillis: 5_000, // 快失败而不是无限排队
});
```

**更大不更快。** 一个比数据库能并发执行的连接数还大的池只是把你的队列搬到数据库去，那儿更难看。当实例数无界（serverless、autoscaling）时，修复办法是用一个多路复用连接的代理（pgbouncer、RDS Proxy），而不是把 `max` 抬高。

## 缺图片优化（前端）

```html
<!-- 反：无尺寸、无格式优化 -->
<img src="/hero.jpg" />

<!-- 正：Hero / LCP 图 ， 艺术方向 + 分辨率切换，高优先级 -->
<!--
  两个技法组合用：
  - 艺术方向（media）：按断点不同裁切 / 取景
  - 分辨率切换（srcset + sizes）：按屏幕密度选文件
-->
<picture>
  <!-- 移动：竖向裁切（8:10） -->
  <source
    media="(max-width: 767px)"
    srcset="/hero-mobile-400.avif 400w, /hero-mobile-800.avif 800w"
    sizes="100vw"
    width="800"
    height="1000"
    type="image/avif"
  />
  <source
    media="(max-width: 767px)"
    srcset="/hero-mobile-400.webp 400w, /hero-mobile-800.webp 800w"
    sizes="100vw"
    width="800"
    height="1000"
    type="image/webp"
  />
  <!-- 桌面：横向裁切（2:1） -->
  <source
    srcset="/hero-800.avif 800w, /hero-1200.avif 1200w, /hero-1600.avif 1600w"
    sizes="(max-width: 1200px) 100vw, 1200px"
    width="1200"
    height="600"
    type="image/avif"
  />
  <source
    srcset="/hero-800.webp 800w, /hero-1200.webp 1200w, /hero-1600.webp 1600w"
    sizes="(max-width: 1200px) 100vw, 1200px"
    width="1200"
    height="600"
    type="image/webp"
  />
  <img
    src="/hero-desktop.jpg"
    width="1200"
    height="600"
    fetchpriority="high"
    alt="Hero 图描述"
  />
</picture>

<!-- 正：折以下图片 ， 懒加载 + 异步解码 -->
<img
  src="/content.webp"
  width="800"
  height="400"
  loading="lazy"
  decoding="async"
  alt="内容图描述"
/>
```

## 不必要的重渲染（React）

```tsx
// 反：每次渲染都创建新对象，让子组件都重渲染
function TaskList() {
  return <TaskFilters options={{ sortBy: 'date', order: 'desc' }} />;
}

// 正：稳定引用
const DEFAULT_OPTIONS = { sortBy: 'date', order: 'desc' } as const;
function TaskList() {
  return <TaskFilters options={DEFAULT_OPTIONS} />;
}

// 对渲染开销大的组件用 React.memo
const TaskItem = React.memo(function TaskItem({ task }: Props) {
  return <div>{/* 贵渲染 */}</div>;
});

// 对昂贵的计算用 useMemo
function TaskStats({ tasks }: Props) {
  const stats = useMemo(() => calculateStats(tasks), [tasks]);
  return <div>{stats.completed} / {stats.total}</div>;
}
```

## 大 bundle 体积

```typescript
// 现代打包器（Vite、webpack 5+）会自动对 named imports 做 tree-shake，
// 前提是依赖发的是 ESM 且 package.json 标了 `sideEffects: false`。
// 改 import 风格之前先 profile ， 真正的收益来自切分与懒加载。

// 正：对重、罕用的特性用 dynamic import
const ChartLibrary = lazy(() => import('./ChartLibrary'));

// 正：路由级代码切分，包在 Suspense 里
const SettingsPage = lazy(() => import('./pages/Settings'));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <SettingsPage />
    </Suspense>
  );
}
```

## 缺缓存（后端）

缓存昂贵且读远多于写的东西。缓存一个本来就很快的查询，只会多加一跳网络、一份 staleness bug、和一份淘汰策略要维护，换不到任何东西。

**有意识地挑层：**

| 层 | 对谁可见 | 何时用 | 代价 |
|---|---|---|---|
| 进程内（`Map`、LRU） | 一个实例 | 小、热、每实例 staleness 可接受 | 每实例独立漂移；失效只到一个 |
| 共享（Redis、Memcached） | 所有实例 | 实例间必须一致，或者值贵到必须共担 | 一跳网络，又一个要跑要监控的服务 |
| CDN / 边缘 | 大家，按 URL | 响应公开且对某 key 完全一致 | 失效是难的部分；假定你不能快速召回一份坏响应 |

```typescript
// 缓存频繁读、罕变的数据
const CACHE_TTL = 5 * 60 * 1000; // 5 分钟
let cachedConfig: AppConfig | null = null;
let cacheExpiry = 0;

async function getAppConfig(): Promise<AppConfig> {
  if (cachedConfig && Date.now() < cacheExpiry) {
    return cachedConfig;
  }
  cachedConfig = await db.config.findFirst();
  cacheExpiry = Date.now() + CACHE_TTL;
  return cachedConfig;
}

// 静态资产的 HTTP 缓存头
app.use('/static', express.static('public', {
  maxAge: '1y',           // 缓存 1 年
  immutable: true,        // 永不重验证（用文件名的内容哈希）
}));

// API 响应的 Cache-Control
res.set('Cache-Control', 'public, max-age=300'); // 5 分钟
```

**Key 设计决定正确性。** 任何改变响应的输入都得在 key 里：租户、locale、权限、特性开关。漏掉查看者的 key 就是把 A 用户的响应发给 B 用户 ， 而这会被作为性能胜利上线。

**只挑一种失效策略，不要挑三种：**

| 策略 | 权衡 |
|---|---|
| TTL | 最简单。接受至 TTL 的 staleness，所以明说可接受的窗口 |
| 基于事件或 tag | 写时新鲜，但写的人现在也得懂缓存拓扑 |
| 版本化 key（`user:42:profile:v7`） | 从不失效，只是停止读旧 key。占内存直到淘汰 |

**防雪崩。** 一个热 key 过期、所有并发请求同时 miss，源头吃满了流量 ， 这就是缓存如何变成故障而不是挡了故障。重算期间服务旧值（`stale-while-revalidate`），或者把并发 miss 合并到一个 in-flight promise 后面，让 N 个等待者只换来一次重算。

**不要缓存：** 任何 staleness 会变成正确性 bug 的东西（余额、权限、结账时库存）、或 key 没标用户的 per-user 数据。见 `../../../references/performance-checklist.md#caching-strategies` 看请求合并、写策略、负缓存、与缓存清单。
