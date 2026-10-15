---
name: security-and-hardening
description: 给代码加固以防漏洞。在审计一个输入处理器的漏洞、在处理用户输入 / 认证 / 数据存储 / 外部集成、或在检查一个登录流是否对 OWASP Top Ten 安全时使用。在构建任何接受不可信数据、管理用户会话、与第三方服务交互的特性时使用。在审计依赖的已知漏洞、对包管理器审计发现分流、评估新包的供应链风险时使用。在涉及到个人数据或隐私合规（GDPR、CCPA）时使用。
---

# Security and Hardening

## 概述

面向 Web 应用的安全先行开发实践。把每一个外部输入视为敌意，把每一个密钥视为神圣，把每一次授权检查视为强制。安全不是一个阶段 ， 它是落在每一行碰用户数据、认证或外部系统的代码上的约束。

## 何时使用

- 构建任何接受用户输入的东西
- 落实认证或授权
- 储存或传输敏感数据
- 集成外部 API 或服务
- 加文件上传、webhook 或回调
- 处理支付或 PII 数据

## 流程：先建威胁模型

不带着威胁模型挂上去的防御都是猜。在加固之前，花 5 分钟像攻击者一样思考：

1. **画信任边界。** 不可信数据在哪里跨进你的系统？HTTP 请求、表单字段、文件上传、webhook、第三方 API、消息队列，以及 **LLM 输出** ， 还有那些因为 OS 交到你手上而看起来是"内部"的本地值：另一个进程的命令行或环境、共享卷上的文件名、job payload 里的路径。信任跟着谁*写*的那个值走，不是哪条通道递的它。每个边界都是攻击面。
2. **列资产。** 什么值得偷、值得毁？凭证、PII、支付数据、管理动作、资金流转。
3. **在每个边界跑一遍 STRIDE** ， 是个快速透镜，不是仪式：

| 威胁 | 问 | 常见缓解 |
|---|---|---|
| **S**poofing（伪装） | 能不能假装用户 / 服务？ | 认证、签名校验 |
| **T**ampering（篡改） | 数据能不能被改（传输中 / 静态）？ | 完整性校验、参数化查询、HTTPS |
| **R**epudiation（否认） | 动作能不能事后被否认？ | 安全事件的审计日志 |
| **I**nformation disclosure（信息泄露） | 数据会不会漏？ | 加密、字段白名单、通用错误 |
| **D**enial of service（拒绝服务） | 能不能把它压垮？ | 限流、输入 size 上限、超时 |
| **E**levation of privilege（提权） | 用户能不能拿到不该有的权限？ | 授权检查、最小权限 |

4. **跟用例一起写 abuse case。** 对每个特性，问"我怎么能乱用它？" ， 然后把它做成你的第一条测试。

如果给一个特性数不出信任边界，你就还没准备好加固它。这是 OWASP **A04: Insecure Design** ， 大多数漏洞都从设计开始，不是从代码。

## 三层边界系统

### 永远做（无例外）

- 在系统边界处**校验全部外部输入**（API routes、form handlers）
- **参数化所有数据库查询** ， 永远不要把用户输入拼进 SQL
- **编码输出**以防 XSS（用框架的自动转义，不要绕过它）
- 外部通信**用 HTTPS**
- 用 bcrypt / scrypt / argon2 **散列密码**（永远不要存明文）
- **设安全响应头**（CSP、HSTS、X-Frame-Options、X-Content-Type-Options）
- 会话 cookie 用 **httpOnly, secure, sameSite**
- 每次发布前**跑检测到的包管理器自带审计**，针对已提交的 lockfile

### 先问（需要人批准）

- 加新的认证流或改认证逻辑
- 存新一类敏感数据（PII、支付信息）
- 加新的外部服务集成
- 改 CORS 配置
- 加文件上传处理器
- 改限流或节流
- 授予更高的权限或角色

### 永远不做

- **永远不要把密钥**提交到版本控制（API key、密码、令牌）
- **永远不要记敏感数据**（密码、令牌、完整信用卡号）
- **永远不要把客户端校验**当作安全边界
- **永远不要为省事关掉安全响应头**
- **永远不要**在用户提供数据上用 `eval()` 或 `innerHTML`
- **永远不要**把会话存在客户端可访问的存储里（localStorage 放 auth token）
- **永远不要**向用户暴露堆栈跟踪或内部错误细节

## 加固控制

下面是工作流；每条的具体实现见 [references/hardening-patterns.md](references/hardening-patterns.md)。到你写那块的代码时再开那一节，别提前。

### 注入、XSS 与访问控制

- 每个查询都参数化。永远不要从输入字符串构建 SQL、NoSQL 或 shell 命令。
- 用框架的自动转义对输出编码。如果实在必须用原生 HTML，先用白名单 sanitizer。
- 在每次请求上都查**授权**，不只是认证：已认证的用户必须拥有或被允许访问那个具体资源（A01、IDOR）。

模式：[Injection](references/hardening-patterns.md#injection)、[XSS](references/hardening-patterns.md#cross-site-scripting-xss)、[Access control](references/hardening-patterns.md#broken-access-control)。

### 认证与会话

- 密码用 bcrypt（≥12 rounds）、scrypt 或 argon2 散列。会话密钥从环境来，不从代码。
- 会话 cookie 设 `httpOnly`、`secure`、`sameSite: 'lax'` 或 `'strict'`（CSRF 防御；`'none'` 把 cookie 发到跨站请求），配有限 `maxAge`。

模式：[Authentication](references/hardening-patterns.md#broken-authentication)。

### 响应头、CORS 与响应

- 每个响应都有安全头（helmet 或框架等价物）；CSP 从 `default-src 'self'` 出发，向收紧方向调整。
- CORS 限定到配置里的明确 origin 列表。带凭证时不能用 `*`。
- 响应前剥掉敏感字段（`passwordHash`、重置 token）。错误体通用；具体内容只进服务端日志。

模式：[Misconfiguration](references/hardening-patterns.md#security-misconfiguration)、[Sensitive data exposure](references/hardening-patterns.md#sensitive-data-exposure)。

### 输入校验与上传

- 在边界用 schema 做校验：白名单的形状、长度、枚举、格式。拒绝时给 422 与结构化 details；下游代码只用解析后的、有类型的值。
- 上传：白名单 MIME 类型、size 上限、需要时校验内容（magic bytes）。扩展名什么都证明不了。

模式：[Schema validation](references/hardening-patterns.md#schema-validation-at-boundaries)、[File upload](references/hardening-patterns.md#file-upload-safety)。

### 服务端拉取（SSRF）

任何用户能影响的 URL ， webhook、import-from-URL、图片代理、链接预览 ， 都可以被打到内部服务。白名单 scheme 与 host，**解析所有** DNS 记录并拒绝任何私有 / 保留地址（loopback、link-local `169.254.169.254`、private、unique-local，IPv4 与 IPv6 都要），禁止重定向。这道检查仍有 DNS-rebinding TOCTOU 的空隙：高危场景下，钉死解析出的 IP 或在前面加一层过滤代理。

模式：[SSRF](references/hardening-patterns.md#server-side-request-forgery-ssrf)。

### 对派生路径的破坏性操作

一次 delete、move、overwrite 的安全程度等于命名它目标的那个值的可靠程度，而信任跟着谁*写*了那个值走，而不是哪条通道递的：另一个进程的命令行同样像 form 字段一样是被攻击者控制的。shape 检验只能证明良好性，不能证明授权。在调用之前，要求满足这三条：解析后的目标（解析 symlink 之后）落在**白名单根**下、距离该根至少**深一层**、带着**在操作前读到的 ownership 证据**。被拒时记下被拒的目标、停下；永远不要回退到一个更宽的默认路径。

为什么这道检查读起来强却更弱（marker 自报、check/use 竞争）：[Destructive paths](references/hardening-patterns.md#destructive-operations-on-derived-paths)。实操代码：`../../references/security-checklist.md`。

### 限流

API 整体限流，认证端点严格（每 15 分钟约 10 次）。一旦多进程服务流量，进程内计数器会悄悄变成 `max × instances`，或在 serverless 上根本不响：限流器要背一个共享存储。

模式：[Rate limiting](references/hardening-patterns.md#rate-limiting)。

### 密钥

密钥从环境来。`.env.example` 用占位符提交；真 `.env*` 文件与密钥材料被 gitignore 掉；提交前 grep staged diff。**一个密钥到了远端的那一刻起就已经被攻陷：先轮换，再清理历史。**

模式：[Secrets management](references/hardening-patterns.md#secrets-management)。

### 依赖与供应链

1. **找安装边界与包管理器。** 用拥有 lockfile 的工作区根，或只要在该工作区外才用独立嵌套项目。相互核 `packageManager`（如存在）、lockfile 与 CI；对不上或多套 lockfile 就停。钉死包管理器版本。
2. **首次执行前阻断依赖脚本。** 用脚本关闭或一份写明的 fail-closed policy 启动，审核挂起脚本源，只批准最小集合，把 policy 提交，再用一次干净 frozen / immutable 安装来验。永远不要一揽子批准。
3. **每次发布前对已提交 lockfile 跑自带审计。** 按**可达性**（runtime、build、test、deploy 路径）与修复可用性分流 critical / high。永远不要自动套强制修复（`npm audit fix --force` 或等价命令），因为强制修复可能跨过声明的依赖范围；要预览、读 changelog、逐升级测试。每个延后都要带理由与审查日期。
4. **审计只能匹配已知公告。** 它们抓不到新出的恶意或 typosquat 包（如 `cross-env` vs `crossenv`）。一起审查新依赖、lockfile diff 与脚本策略变化：所有权、维护、发布年龄、来源、传递图。能支持的地方验证注册表签名（`npm audit signatures`、`pnpm audit signatures`），缺失视为要进一步调查的信号、不是定罪的证据（A06、LLM03）。

分流决策树：[Dependency audit triage](references/hardening-patterns.md#dependency-audit-triage)。包管理器矩阵与 install-script 闸门：`../../references/security-checklist.md`。

### 个人数据与隐私

加固问"攻击者能读到它吗？" 隐私问"我们*到底该不该*持有它、持有多久？" 最便宜保护、最便宜泄露、最便宜合规的数据，是你从未收集过的；把个人数据当负债来最小化。

- **加字段就分类**（非个人、PII、敏感），按类处理。你保护不了、或履行删除请求那些数据，如果你找不到它。
- **只按声明的目的收集。** "以后可能有用"是潜在泄露面，不是目的。PII 不要进遥测（`observability-and-instrumentation` Skill 从运维侧也讲过同样的话）。
- **一开始就设保留期，然后真删。** 任何一个存个人数据的存储都得有 TTL 与可工作的删除路径，包括备份、缓存、搜索索引与分析副本。
- **支持你司法辖区要求的数据主体权利**（GDPR、CCPA 之类）：导出、更正、删除。Schema 要让用户数据找得到、删得掉，而不是被擦不回去地铺到各系统。
- **同意守门采集与第三方共享、且可审计。** 把 PII 送给分析、广告或 LLM 供应商就是共享；供应商需要数据处理协议。让 region 成为可配策略，而不是硬编码假设。

分类表：[Data classification](references/hardening-patterns.md#data-classification)。一次隐私事件起的就是泄露通报钟：用 `debugging-and-error-recovery` Skill 跑事后复盘。

### AI / LLM 特性

调一个 LLM ， 聊天机器人、摘要、Agent、RAG ， 增加一片新攻击面；映射到 [OWASP Top 10 for LLM Applications (2025)](https://genai.owasp.org/llm-top-10/)：

- **模型输出是不可信输入**（LLM05）。绝不进 `eval`、SQL、shell、`innerHTML` 或文件路径；防御式解析，对 schema 校验，再编码。
- **prompt 可被劫持**（LLM01）。上下文里的不可信文本 ， 用户消息、抓来的页面、PDF ， 可能带指令。系统 prompt 不是安全边界；用代码强制权限。
- **密钥、其它租户数据、完整系统 prompt 不要放上下文窗口**（LLM02、LLM07）；划分工具权限，校验每个工具参数，破坏性动作要有确认（LLM06）；限制 token、请求频率、递归深度（LLM10）；按租户隔离 RAG embeddings，索引前校验文档（LLM08）。

模式：[LLM output handling](references/hardening-patterns.md#llm-output-handling)。

## 评审清单

签批前走一遍 `../../references/security-checklist.md`：覆盖认证、授权、输入、数据保护与隐私、头与 CORS、依赖与供应链、AI/LLM 与错误处理，以及 OWASP 速查表。

## 常见借口

| Rationalization | Reality |
|---|---|
| "这是内部工具，安全不重要" | 内部工具也会被攻破。攻击者挑最薄弱环节。 |
| "我们等会再加安全" | 后期补安全的代价比一开始就放进来的 10 倍。现在加。 |
| "没人会专门来利用这个" | 自动扫描会找到它。靠隐秘不是安全。 |
| "框架已经把安全管好了" | 框架提供工具，不提供保证。你仍得正确使用它们。 |
| "这只是原型" | 原型也会变生产。从第一天就有安全习惯。 |
| "威胁建模在这儿是过度" | 5 分钟"我会怎么攻击这个"能挡掉后面任何控制都难补的设计缺陷。 |
| "它是 LLM 输出，只是文字而已" | 那段"文字"可以是一条 SQL、一段脚本标签、一条 shell 命令。把它当任何不可信输入来对待。 |
| "审计过了所以这个依赖安全" | 审计匹配已知公告。它们不抓新出的恶意包，也不替未审过的安装脚本背书。 |
| "先收着，以后可能要" | 不持有的数据不会泄露、不会被告、不会误删。"以后要"是泄露面、不是目的。 |
| "删除请求我们手工处理" | 手工擦漏备份、缓存与分析副本。如果 schema 找不到用户的数据，你就履行不了请求 ， 按能履行来设计。 |
| "合规是法务的事" | 导出、删除、保留、同意，是 schema 与代码。法务在你把 PII 撒到十个系统后无法补上。 |

## 红旗

- 用户输入直接进数据库查询、shell 命令或 HTML 渲染
- 一次 delete、move 或 overwrite，其目标来自 payload、配置值、或另一个进程命令行，只靠 path 的 shape 检验
- 源码或提交历史里有密钥
- 没有认证 / 授权检查的 API endpoint
- 缺 CORS 配置或通配符 `*` 来源
- 认证端点没有限流，或一个进程内限流器在超过一个实例前面
- 堆栈跟踪或内部错误暴露给用户
- 依赖里有已知 critical 漏洞、一份安装边界上有冲突的 lockfile、不可复现的安装、或被一揽子批准的脚本
- 服务端拉用户提供的 URL 但没白名单（SSRF）
- LLM / 模型输出直接进 query、DOM、shell 或 `eval`
- 密钥、PII 或完整系统提示词被放进 LLM 上下文窗口
- 个人数据没有声明目的、保留限制或删除路径就采集
- PII 被发给分析 / 广告 / LLM 供应商而没有同意或数据处理协议
- "删除我的账户"只是 flip 一个 flag，但个人数据还留在存储和备份里

## 验证

做完安全相关代码：

- [ ] 自带审计没有未处置的可达 critical / high 发现；CI 守住权威 lockfile 并阻断未审过的依赖脚本
- [ ] 源码或 git 历史里没有密钥
- [ ] 所有用户输入在系统边界被校验
- [ ] 破坏性文件系统操作解析 symlink、之后核白名单根、最小深度、ownership 才执行
- [ ] 受保护的每个端点都做了认证与授权检查
- [ ] 响应里有安全头（在浏览器 DevTools 里核）
- [ ] 错误响应不暴露内部细节
- [ ] 认证端点有活跃的限流，在多实例背后有共享存储
- [ ] 服务端 URL 拉取按白名单过（没有 SSRF）
- [ ] LLM / 模型输出在被用之前被校验与编码（如有 AI 特性）
- [ ] 个人数据被分类、按声明目的最小化、有保留限制
- [ ] 删除与导出请求端到端工作（含备份、缓存、分析副本）
