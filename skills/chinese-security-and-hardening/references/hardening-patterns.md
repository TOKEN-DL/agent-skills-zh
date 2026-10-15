# 加固模式

`security-and-hardening` Skill 的代码模式与决策辅助。Skill 给出规则；本文给出一份具体实现。写那块代码时再开需要的章节 ， 不打算自上而下读。

共享清单（威胁建模、提交前检查、安全响应头、CORS、包管理器矩阵、安装脚本闸、OWASP 速查表）在 `../../../references/security-checklist.md`。

## OWASP Top 10 预防模式

这是预防模式，不是排名。2021 年的排序见 `../../../references/security-checklist.md#owasp-top-10-quick-reference`。

### 注入

```typescript
// 反：通过字符串拼接做 SQL 注入
const query = `SELECT * FROM users WHERE id = '${userId}'`;

// 正：参数化查询
const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);

// 正：用带参数化输入的 ORM
const user = await prisma.user.findUnique({ where: { id: userId } });
```

### 认证失效

```typescript
// 密码散列
import { hash, compare } from 'bcrypt';

const SALT_ROUNDS = 12;
const hashedPassword = await hash(plaintext, SALT_ROUNDS);
const isValid = await compare(plaintext, hashedPassword);

// 会话管理
app.use(session({
  secret: process.env.SESSION_SECRET,  // 从环境来，不从代码
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,     // 不能被 JavaScript 访问
    secure: true,       // 仅 HTTPS
    sameSite: 'lax',    // CSRF 防护
    maxAge: 24 * 60 * 60 * 1000,  // 24 小时
  },
}));
```

### 跨站脚本（XSS）

```typescript
// 反：把用户输入当 HTML 渲染
element.innerHTML = userInput;

// 正：用框架的自动转义（React 默认就这么做）
return <div>{userInput}</div>;

// 如果真的必须渲染 HTML，先 sanitize
import DOMPurify from 'dompurify';
const clean = DOMPurify.sanitize(userInput);
```

### 访问控制失效

```typescript
// 总要查授权，不只是认证
app.patch('/api/tasks/:id', authenticate, async (req, res) => {
  const task = await taskService.findById(req.params.id);

  // 核已认证用户拥有这个资源
  if (task.ownerId !== req.user.id) {
    return res.status(403).json({
      error: { code: 'FORBIDDEN', message: '无权修改该任务' }
    });
  }

  // 继续更新
  const updated = await taskService.update(req.params.id, req.body);
  return res.json(updated);
});
```

### 安全配置失效

```typescript
// 安全响应头（用 Express 的 helmet）
import helmet from 'helmet';
app.use(helmet());

// Content Security Policy
app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"],
    styleSrc: ["'self'", "'unsafe-inline''],  // 能收紧就收紧
    imgSrc: ["'self'", 'data:', 'https:'],
    connectSrc: ["'self'"],
  },
}));

// CORS ， 限定到已知来源
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || 'http://localhost:3000',
  credentials: true,
}));
```

### 敏感数据暴露

```typescript
// 永远不要在 API 响应里返回敏感字段
function sanitizeUser(user: UserRecord): PublicUser {
  const { passwordHash, resetToken, ...publicFields } = user;
  return publicFields;
}

// 把密钥放在环境变量里
const API_KEY = process.env.STRIPE_API_KEY;
if (!API_KEY) throw new Error('STRIPE_API_KEY 未配置');
```

### 服务端请求伪造（SSRF）

任何时候只要服务端拉一个用户影响的 URL ， webhook、import-from-URL、图片代理、链接预览 ， 攻击者都能把它指向内部服务（云元数据、`localhost`、私有 IP）。

```typescript
// 反：拉用户给的任意 URL
await fetch(req.body.webhookUrl);

// 正：白名单 scheme + host，若任一解析出的 IP 是私有的就拒，禁止重定向
import { lookup } from 'node:dns/promises';
import ipaddr from 'ipaddr.js';

const ALLOWED_HOSTS = new Set(['hooks.example.com']);

async function assertSafeUrl(raw: string): Promise<URL> {
  const url = new URL(raw);
  if (url.protocol !== 'https:') throw new Error('仅允许 https');
  if (!ALLOWED_HOSTS.has(url.hostname)) throw new Error('host 不在白名单');
  // 解析所有记录；任何一条私有 / 保留地址就视为失败
  const addrs = await lookup(url.hostname, { all: true });
  if (addrs.some((a) => ipaddr.parse(a.address).range() !== 'unicast')) {
    throw new Error('私有 / 保留 IP');
  }
  return url;
}

await fetch(await assertSafeUrl(req.body.webhookUrl), { redirect: 'error' });
```

`range() !== 'unicast'` 这条覆盖了 loopback、link-local `169.254.169.254`（云元数据、SSRF 的头号目标）、private、unique-local 范围，IPv4 与 IPv6 都包括。

**注意 ， 这仍有一个 TOCTOU 空隙。** `fetch` 在校验完之后会再次解析 DNS，所以攻击者用一个短 TTL 记录就能在校验与连接之间重新绑定到一个内部 IP。高危场景下，解析一次后钉死 IP 再连接，或者在前面加一层过滤代理（`request-filtering-agent` / `ssrf-req-filter`）。

## 输入校验模式

### 在边界用 schema 做校验

```typescript
import { z } from 'zod';

const CreateTaskSchema = z.object({
  title: z.string().max(200).trim().min(1),
  description: z.string().max(2000).optional(),
  priority: z.enum(['low', 'medium', 'high']).default('medium'),
  dueDate: z.string().datetime().optional(),
});

// 在路由处理里校验
app.post('/api/tasks', async (req, res) => {
  const result = CreateTaskSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(422).json({
      error: {
        code: 'VALIDATION_ERROR',
        message: '输入不合法',
        details: result.error.flatten(),
      },
    });
  }
  // result.data 现在是类型化且校验过的
  const task = await taskService.create(result.data);
  return res.status(201).json(task);
});
```

### 文件上传安全

```typescript
// 限制文件类型与大小
const ALLOWED_TYPES = ['image/jpeg', 'image/png', 'image/webp'];
const MAX_SIZE = 5 * 1024 * 1024; // 5MB

function validateUpload(file: UploadedFile) {
  if (!ALLOWED_TYPES.includes(file.mimetetype)) {
    throw new ValidationError('不允许的文件类型');
  }
  if (file.size > MAX_SIZE) {
    throw new ValidationError('文件过大 (上限 5MB)');
  }
  // 不要信文件扩展名 ， 需要时查 magic bytes
}
```

### 对派生路径的破坏性操作

一次 delete、move 或 overwrite 的安全程度，等于命名它目标的那个值的可信度。从内核、job payload、或兄弟服务里读这个值，证明的只是它*从哪里来*，谁*写了*它 ， 另一个进程的命令行一样是攻击者可控的。shape 校验（"绝对路径，至少一层深"）只能证明良好性，常被误当作授权；这就是一个清理 routine 把根目录删了而不是叶子文件。

破坏性调用之前，要求满足这三条：解析后的目标（解析 symlink 之后，不要在原始字符串上比较）落在**白名单根**下；至少距离该根**深一层**，所以根本身永远不是目标；带着**证据证明它是你的**，在调用之前、且在任何会移走证据的拆解之前读 ， 不然"不存在"和"不是我的"分不清。拒绝时记下被拒的目标、停下：清理时回退到更宽的默认路径就是要防的失败。实操示例在 `../../../references/security-checklist.md#destructive-path-operations`。

两道限制，因为这道检查读起来强、用起来没这么强。树里的标记是自证 ， 能写到那里的人就能写这个标记 ， 所以期望的所有者必须来自已认证的状态，且标记需要完整性保护（限制所有，或一个 MAC）才配作授权。解析路径、然后操作*名称*本身是一个 check/use race，凡是 untrusted 进程能换祖先的地方：在共享卷上，用 descriptor 持有目标、并用 no-follow + 在根下的操作、或保证期间层级不会变。

## 依赖审计分流

包管理器审计报告已知公告；它不能证明一个包可信、或有漏洞的代码真的可达。用这个决策树：

```
本地包管理器审计报了一条漏洞
├── 严重度: critical 或 high
│   ├── 这段有漏洞的代码在 runtime / build / test / deploy 路径上可达吗？
│   │   ├── 是 --> 立即修（更新、打补丁、或替换依赖）
│   │   └── 否（确认这几条路径上都没用到） --> 尽快修，但不是阻塞器
│   └── 有可用的修复吗？
│       ├── 有 --> 更新到打了补丁的版本
│       └── 没有 --> 找缓解、加进白名单（带审查日期）或考虑替换依赖
├── 严重度: moderate
│   ├── 生产可达？ --> 下个发布周期修
│   └── 仅 dev 用？ --> 顺手修，记在 backlog
└── 严重度: low
    └── 跟踪，在常规依赖更新时修
```

**关键问题：**
- 这个有漏洞的函数在你的代码路径里真的会被调到吗？
- 这个依赖是 runtime 还是仅 dev？
- 在你的部署上下文里，这个漏洞能被利用吗（如纯客户端应用里的服务端漏洞）？

延后修时，文档化理由、设个审查日期。

## 限流

```typescript
import rateLimit from 'express-rate-limit';

// API 通用限流
app.use('/api/', rateLimit({
  windowMs: 15 * 60 * 1000, // 15 分钟
  max: 100,                   // 每个窗口 100 次
  standardHeaders: true,
  legacyHeaders: false,
}));

// 对认证端点更严
app.use('/api/auth/', rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 10,  // 每个 15 分钟 10 次
}));
```

**一旦多于一个进程就在共享存储里计数。** `express-rate-limit` 默认在进程内存里计。在负载均衡后面，每个实例有自己的计数器，所以实际限制是 `max × instances`；在 serverless 或边缘运行时上，每次新调用都从零起算，上面的认证限流可能根本不会生效。传一个共享 `store`（`rate-limit-redis` 的 Redis），或者用一条 HTTP 思路的限流，它能在长 TCP 连接不存在的地方工作（如 `@upstash/ratelimit`）：

```typescript
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const authLimiter = new Ratelimit({
  redis: Redis.fromEnv(),                       // UPSTASH_REDIS_REST_URL + _TOKEN
  limiter: Ratelimit.slidingWindow(10, '15 m'), // 跨所有实例、每个 15 分钟 10 次
});
const { success } = await authLimiter.limit(`login:${req.ip}`);
if (!success) return res.status(429).end();
```

## 密钥管理

```
.env 文件：
  ├── .env.example  → 提交（带占位符值的模板）
  ├── .env          → 不提交（含真实密钥）
  └── .env.local    → 不提交（本地覆盖）

.gitignore 必须包含：
  .env
  .env.local
  .env.*.local
  *.pem
  *.key
```

**提交前总要检查：**
```bash
# 检查不小心被暂存的密钥
git diff --cached | grep -i "password\|secret\|api_key\|token"
```

**一旦密钥被提交，轮换它。** 仅删那一行、改写历史都不够 ， 假定它到达远端的那一刻起就已经被攻陷。先撤销并重发这份 key，再从历史里清掉。

## 数据分类

加字段时就把个人数据分类好，让日后导出与删除请求找得到它：

| 类别 | 例子 | 处理 |
|---|---|---|
| **非个人** | 聚合、匿名化计数 | 常规处理 |
| **个人（PII）** | 姓名、邮箱、IP、设备 / 用户 ID | 最小化、访问控制、纳入导出 / 删除 |
| **敏感** | 健康、金融、位置、生物特征、政府 ID、未成年人相关 | 额外的采集基础、更严的访问、通常加密 + 审计日志 |

## LLM 输出处理

```typescript
// 反：把模型输出当成命令或标记
const sql = await llm.generate(`为这个写 SQL: ${userQuestion}`);
await db.query(sql);                                   // 任意查询执行
container.innerHTML = await llm.reply(userMessage);   // 通过模型存储的 XSS

// 正：模型输出是数据 ， 防御式解析、再校验、再编码
let intent;
try {
  intent = CommandSchema.parse(JSON.parse(await llm.replyJson(userMessage)));
} catch {
  throw new ValidationError('意外模型输出'); // JSON.parse 或 schema 失败
}
await runAllowlistedAction(intent.action, intent.params);
container.textContent = await llm.reply(userMessage);
```
