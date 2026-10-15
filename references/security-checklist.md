# 安全检查清单（Security Checklist）

面向 Web 应用安全的快速参考。与 `chinese-security-and-hardening` Skill 配合使用。

## 目录

- [威胁建模（从这里开始）](#威胁建模从这里开始)
- [提交前检查](#提交前检查)
- [身份认证（Authentication）](#身份认证authentication)
- [授权（Authorization）](#授权authorization)
- [输入校验（Input Validation）](#输入校验input-validation)
- [安全响应头（Security Headers）](#安全响应头security-headers)
- [跨域资源共享配置（CORS）](#跨域资源共享配置cors)
- [数据保护（Data Protection）](#数据保护data-protection)
- [依赖安全（Dependency Security）](#依赖安全dependency-security)
- [AI / LLM 安全](#ai--llm-安全)
- [错误处理（Error Handling）](#错误处理error-handling)
- [OWASP Top 10 速查](#owasp-top-10-速查)
- [OWASP Top 10 for LLMs 速查](#owasp-top-10-for-llms-速查)

## 威胁建模（从这里开始）

在动手上控制之前，先花五分钟像攻击者那样思考：

- [ ] 信任边界已映射（请求、上传、Webhook、第三方 API、LLM 输出，以及由你无法控制进程写入的本地值）
- [ ] 资产已具名（凭据、PII、支付数据、管理员动作、资金流转）
- [ ] 沿每个边界跑一遍 STRIDE（伪造 Spoofing、篡改 Tampering、抵赖 Repudiation、信息泄露 Info disclosure、拒绝服务 DoS、权限提升 Elevation）
- [ ] 滥用用例（abuse case）与用例并列书写（"我该怎么误用这个？"）

## 提交前检查

- [ ] 代码中不含机密（`git diff --cached | grep -i "password\|secret\|api_key\|token"`）
- [ ] `.gitignore` 覆盖：`.env`、`.env.local`、`*.pem`、`*.key`
- [ ] `.env.example` 使用占位值（而不是真实机密）

## 身份认证（Authentication）

- [ ] 密码使用 bcrypt（≥12 轮）、scrypt 或 argon2 哈希
- [ ] 会话 Cookie：`httpOnly`、`secure`、`sameSite: 'lax'`
- [ ] 已配置会话过期（合理的 max-age）
- [ ] 登录端点限流（15 分钟内 ≤10 次尝试）
- [ ] 密码重置 token：限时（≤1 小时）、一次性
- [ ] 多次失败后账户锁定（可选，需附带通知）
- [ ] 敏感操作支持 MFA（可选但建议）

## 授权（Authorization）

- [ ] 每个受保护端点都校验身份
- [ ] 每个资源访问都校验所有权 / 角色（防止 IDOR（Insecure Direct Object Reference，不安全的直接对象引用））
- [ ] 管理员端点要求管理员角色校验
- [ ] API Key 限定到最小必要权限
- [ ] JWT（JSON Web Token，JSON 网络令牌）已校验（签名、过期、签发方）

## 输入校验（Input Validation）

- [ ] 系统边界处校验全部用户输入（API 路由、表单处理器）
- [ ] 校验使用白名单（而不是黑名单）
- [ ] 字符串长度受约束（min/max）
- [ ] 数值范围已校验
- [ ] 邮箱、URL、日期格式使用正规库校验
- [ ] 文件上传：限定类型、限制大小、校验内容
- [ ] SQL 查询参数化（不要字符串拼接）
- [ ] HTML 输出已转义（使用框架的自动转义）
- [ ] 重定向前校验 URL（防止开放重定向）
- [ ] 服务端 URL 获取走白名单；私有 / 保留 IP 段被屏蔽（防止 SSRF（Server-Side Request Forgery，服务端请求伪造））
- [ ] 破坏性路径操作（删除 / 移动 / 覆盖）：先解析符号链接、根目录走白名单、限定最小深度，调用前先读取所有权证据

### 破坏性路径操作

针对由数据指定的目标的隔离措施：先解析，再判定，并把解析结果视为"候选"，而不是"授权"：

```typescript
import { realpath, readFile } from 'node:fs/promises';
import { resolve, relative, isAbsolute, join, sep } from 'node:path';

const ALLOWED_ROOTS = ['/var/lib/myapp/sessions']; // 白名单，不是模式
const MIN_DEPTH = 1;                               // 这样根目录永远不会被当作目标

async function resolveDeletable(candidate: string, expectedOwner: string) {
  const target = await realpath(resolve(candidate)); // 校验之前先解析符号链接
  const inRoot = ALLOWED_ROOTS.some((root) => {
    const rel = relative(root, target);
    // `rel === '..'` / `'../'` 这两种仅限，朴素的 `startsWith('..')` 也会
    // 拒掉一个名为 `..cache` 的合法子项。
    if (rel === '' || rel === '..' || rel.startsWith(`..${sep}`) || isAbsolute(rel)) return false;
    return rel.split(sep).length >= MIN_DEPTH;
  });
  if (!inRoot) throw new Error(`refusing: outside allowed roots (${target})`);

  const owner = await readFile(join(target, '.owner'), 'utf8').catch(() => null);
  if (owner?.trim() !== expectedOwner) throw new Error(`refusing: unproven owner (${target})`);
  return target;
}
```

这一段没有做的事，以及在引用该代码片段时必须显式说明的事：

- **标记位是"自陈的"（self-attestation）。** 任何能在根目录里写入的进程都能写入 `.owner`。`expectedOwner` 必须来自认证状态，并且该标记位在被当作授权（而不是"与错误派生目标的合理性校验"）前，需要有完整性保护（限制所有权，或加 MAC（Message Authentication Code，消息认证码））。
- **返回路径意味着存在 check/use 竞态。** 在不受信任进程可能在校验与调用之间替换祖先节点的场景下，应当改用 no-follow 描述符、在根目录下的语义，或保证该层级在调用期间不可变。

## 安全响应头（Security Headers）

```
Content-Security-Policy: default-src 'self'; script-src 'self'
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
X-XSS-Protection: 0  （已禁用，依赖 CSP（Content Security Policy，内容安全策略））
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
```

## 跨域资源共享配置（CORS）

```typescript
// 限制性配置（推荐）
cors({
  origin: ['https://yourdomain.com', 'https://app.yourdomain.com'],
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization'],
})

// 生产环境严禁：
cors({ origin: '*' })  // 允许任何来源
```

## 数据保护（Data Protection）

- [ ] API 响应排除敏感字段（`passwordHash`、`resetToken` 等）
- [ ] 不记录敏感数据（密码、token、完整信用卡号）
- [ ] PII（Personally Identifiable Information，个人可识别信息）静态加密（如法规要求）
- [ ] 所有外部通信走 HTTPS
- [ ] 数据库备份已加密
- [ ] 个人数据已分类、按明示目的收集，并最小化
- [ ] 个人数据设有保留期限与可用的删除路径（包括备份、缓存、索引）
- [ ] 必要时支持导出 / 删除（数据主体）请求；第三方共享需有同意与数据处理协议（DPA，Data Processing Agreement）

## 依赖安全（Dependency Security）

首先定位**安装边界（installation boundary）**。若某个包被父级 `workspaces` 声明命中，则使用该 workspace 根；否则使用同时拥有其清单与依赖图的最近项目根。在该边界处交叉核对 `packageManager`（若存在）、锁文件与 CI 命令。若它们彼此不一致或该边界上并存多个管理器的锁文件，就停下来。嵌套项目只有在位于父 workspace 之外时才是独立的；独立子项目使用不同管理器是合法的。

| 管理器 / 版本信号 | 冻结 / 不可变 CI 安装 | 已知公告审计 |
|---|---|---|
| npm（`package-lock.json` 或 `npm-shrinkwrap.json`） | `npm ci` | `npm audit` |
| pnpm | `pnpm install --frozen-lockfile` | `pnpm audit` |
| Yarn 2+ | `yarn install --immutable` | `yarn npm audit -A -R` |
| Yarn 1 | `yarn install --frozen-lockfile` | `yarn audit` |

对未列出的管理器或版本，请查阅其官方文档；不要替换成别的管理器的命令或较新默认。

### 安装脚本门禁（Install-Script Gate）

不要在默认未经验证的客户端上，先用一次普通安装来"发现"依赖生命周期脚本。

1. 用禁用依赖脚本的方式引导，或使用有据可查的默认拒绝（default-deny）策略并以失败关闭（fail-closed）方式强制执行。
2. 在批准前检查脚本源码与包版本的确切内容。
3. 在安装边界处记录最严格的本地允许 / 拒绝策略，并提交到仓库。
4. 在该策略下运行一次干净的冻结 / 不可变安装，并验证所需包仍能构建。

**即时快照：** 包管理器默认值与命令名变化很快。在依赖此矩阵前，务必对照所钉版本客户端当前的官方文档进行验证。

| 管理器版本 | 本地（native）策略 |
|---|---|
| 未经验证的细粒度审批的 npm | 用 `npm ci --ignore-scripts` 引导，或在打算整项目阻断时固化 `ignore-scripts=true`。保持脚本禁用，或在审查通过前有意识地升级。 |
| npm 11.18.x（在 11.18.0 上验证） | 未审查的依赖脚本默认会带警告运行。在普通安装前强制 `strict-allow-scripts=true`，随后在安装边界使用 workspace 不感知的 `npm install-scripts ls`；批准项需按版本钉死，拒绝项需按名称宽口径。 |
| npm 12.x（在 12.0.1 上验证） | 未审查的依赖脚本默认会被跳过；`strict-allow-scripts=true` 会让它们的存在在执行前直接使安装失败。使用同样的 `npm install-scripts` 审查与批准流程。 |
| pnpm 11+ | 使用 `pnpm approve-builds` 并提交 `allowBuilds` 决议；`strictDepBuilds` 默认为 `true`，未审查的构建会失败。 |
| pnpm 10.26–10.x | 显式配置 `allowBuilds`，或使用 `pnpm approve-builds` 配合旧版 `onlyBuiltDependencies` / `ignoredBuiltDependencies` 列表。设置 `strictDepBuilds: true`；其 v10 默认是 `false`。 |
| pnpm 10.1–10.25 | `pnpm approve-builds` 会记录旧版列表；在支持的位置启用 `strictDepBuilds`（10.3+）。 |
| 较旧或未知的 pnpm | 用 `pnpm install --frozen-lockfile --ignore-scripts` 引导。除非钉死版本文档化了可强制策略，否则保持脚本禁用。 |
| Yarn 4.14+ | 依赖 postinstall 默认禁用。仅通过顶层 `dependenciesMeta.<package>.built: true` 授予必需例外。 |
| Yarn 2–4.13 | 在 `.yarnrc.yml` 中设置 `enableScripts: false`，然后仅通过顶层 `dependenciesMeta.<package>.built: true` 授予必需例外；不要全局启用脚本。 |
| Yarn 1 | 用 `yarn install --ignore-scripts` 引导；除非每个必需例外都按钉死版本的文档化流程审查过，否则保持脚本禁用。 |

权威参考：[npm install-scripts](https://docs.npmjs.com/cli/v11/commands/npm-install-scripts/)、[安装策略](https://docs.npmjs.com/cli/v11/commands/npm-install/)、[CLI 发布](https://github.com/npm/cli/releases)；[pnpm approve-builds](https://pnpm.io/cli/approve-builds)、[构建设置](https://pnpm.io/settings#allowbuilds)；[Yarn 安全](https://yarnpkg.com/features/security)、[清单](https://yarnpkg.com/configuration/manifest#dependenciesMeta)。

**供应链卫生**（公告审计逮不到新出现的恶意包）：

- [ ] 每个项目 / workspace 根只有一份权威锁文件已提交，且 CI 永远不会重写它
- [ ] 严重 / 高危发现需按可达性分类处置；延期需有理由与复核日期
- [ ] 强制审计修复（`npm audit fix --force` 或等效）绝不自动化；修复差异与变更日志需审查
- [ ] 在管理器支持时验证注册表签名 / 来源（provenance）
- [ ] 依赖生命周期脚本在首次执行前被阻断，仅通过钉死管理器的本地策略批准
- [ ] 新增依赖需审查：所有权、维护状态、发布年龄、来源、传递图、typosquatting（误植域名）

## AI / LLM 安全

针对任何调用 LLM（Large Language Model，大语言模型）的特性（聊天机器人、摘要器、Agent、RAG（Retrieval-Augmented Generation，检索增强生成））：

- [ ] 模型输出视为不可信，绝不能放进 `eval` / SQL / Shell / `innerHTML` / 文件路径
- [ ] 假设存在 prompt injection（提示词注入）；权限用代码强制，而不是系统提示词
- [ ] 机密、跨租户数据、完整系统提示词不得进入上下文窗口
- [ ] 工具 / Agent 权限已限定；破坏性或不可逆动作需要确认
- [ ] 设置 token、速率与递归 / 循环上限（限制消耗）

## 错误处理（Error Handling）

```typescript
// 生产环境：通用错误，不暴露内部细节
res.status(500).json({
  error: { code: 'INTERNAL_ERROR', message: 'Something went wrong' }
});

// 生产环境严禁：
res.status(500).json({
  error: err.message,
  stack: err.stack,         // 暴露内部细节
  query: err.sql,           // 暴露数据库细节
});
```

## OWASP Top 10 速查

| # | 漏洞 | 防护 |
|---|---|---|
| 1 | 访问控制失效（Broken Access Control） | 每个端点做认证与所有权校验 |
| 2 | 加密失效（Cryptographic Failures） | HTTPS、强哈希、代码中无机密 |
| 3 | 注入（Injection） | 参数化查询、输入校验 |
| 4 | 不安全设计（Insecure Design） | 威胁建模、规范驱动开发 |
| 5 | 安全配置错误（Security Misconfiguration） | 安全响应头、最小权限、审计依赖 |
| 6 | 易受攻击组件（Vulnerable Components） | 生态依赖审计（`npm audit`、`pip-audit` 等），保持依赖更新，最小依赖 |
| 7 | 认证失效（Auth Failures） | 强密码、限流、会话管理 |
| 8 | 数据完整性失效（Data Integrity Failures） | 校验更新与依赖、签名工件 |
| 9 | 日志与监控失效（Logging Failures） | 记录安全事件、不记录机密 |
| 10 | SSRF（服务端请求伪造） | 校验 / 白名单 URL、限制出站请求 |

## OWASP Top 10 for LLMs 速查

面向带有 LLM 能力的应用。参见 [OWASP GenAI Security Project](https://genai.owasp.org/llm-top-10/)。

| ID | 风险 | 防护 |
|---|---|---|
| LLM01 | Prompt Injection（提示词注入） | 不要把系统提示词当作边界；用代码强制权限 |
| LLM02 | 敏感信息泄露（Sensitive Information Disclosure） | 机密 / PII 不进 prompt；过滤输出 |
| LLM03 | 供应链（Supply Chain） | 像对待任何依赖一样审查模型、数据集与插件 |
| LLM04 | 数据与模型投毒（Data and Model Poisoning） | 使用受信模型源、验证完整性；审查微调与 RAG 数据 |
| LLM05 | 输出处理不当（Improper Output Handling） | 把输出视为不可信；校验、参数化、转义 |
| LLM06 | 过度代理（Excessive Agency） | 限定工具权限；破坏性动作需要确认 |
| LLM07 | 系统提示词泄露（System Prompt Leakage） | 假设系统提示词会泄露；里面不放机密 |
| LLM08 | 向量与嵌入弱点（Vector and Embedding Weaknesses） | RAG 嵌入按租户分区；索引前校验文档 |
| LLM09 | 错误信息（Misinformation） | 用引用支撑答案；关键断言需校验；保留人在回路 |
| LLM10 | 无界消耗（Unbounded Consumption） | 限制 token、请求速率与循环 / 递归深度 |