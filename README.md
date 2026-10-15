# Agent Skills（汉化版）

**面向 AI 编程 Agent 的生产级工程技能集合。**

技能把资深工程师在构建软件时使用的工作流、质量门槛和最佳实践封装起来。这些技能被打包成 Skill，让 AI Agent 在开发的每一个阶段都能稳定地遵循它们。

> 这是 `addyoshani/agent-skills` 的**独立汉化版本**。汉化规则：根目录命名为 `agent-skills-zh`，每个 Skill 子目录加 `chinese-` 前缀（例：`spec-driven-development` → `chinese-spec-driven-development`），原文所有 key / 路径 / 命令 / URL / Issue 编号保留不变。汉化产物不参与上游版本同步；如发现内容与上游不一致，请以 [上游仓库](https://github.com/addyosmani/agent-skills) 为准。

<a href="https://trendshift.io/repositories/25200" target="_blank"><img src="https://trendshift.io/api/badge/repositories/25200" alt="addyosmani%2Fagent-skills | Trendshift" style="width: 250px; height: 55px;" width="250" height="55"/></a>

![Addy's Agent Skills](https://addyosmani.com/assets/images/addys-agent-skills.jpg)

```
  DEFINE          PLAN           BUILD          VERIFY         REVIEW          SHIP
 ┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐      ┌──────┐
 │ Idea │ ───▶ │ Spec │ ───▶ │ Code │ ───▶ │ Test │ ───▶ │  QA  │ ───▶ │  Go  │
 │Refine│      │  PRD │      │ Impl │      │Debug │      │ Gate │      │ Live │
 └──────┘      └──────┘      └──────┘      └──────┘      └──────┘      └──────┘
  /spec          /plan          /build        /test         /review       /ship
```

---

## 命令

9 个斜杠命令对应完整开发生命周期，每个命令都会自动激活对应的 Skill。

| 你在做的事 | 命令 | 关键原则 |
|-------------------|---------|---------------|
| 厘清要构建什么 | `/spec` | 先写规格，再写代码 |
| 规划实现路径 | `/plan` | 任务要小、要原子化 |
| 增量构建 | `/build` | 一次只切一片 |
| 证明它能跑 | `/test` | 测试就是证据 |
| 设定质量底线 | `/constraints` | 一次定清楚，处处都强制 |
| 合并前复查 | `/review` | 提升代码健康度 |
| 审计 Web 性能 | `/webperf` | 先测量，再优化 |
| 简化代码 | `/code-simplify` | 清晰胜于取巧 |
| 上线到生产 | `/ship` | 更快、更安全 |

在 Claude Code 中，这些会以插件命令形式安装。如果某个命令与内置命令同名（例如 `/review`），请使用带命名空间的形式：`/agent-skills:review`。

希望规格就绪后少一些手动步骤？**`/build auto`** 会自动生成实施计划，并在一次审批通过后连续执行每个任务。批量审批一次后由它自动推进，自动消解的是任务之间的人为停留，而不是验证环节：每个任务依然走测试驱动，并单独提交，失败或有风险的操作会暂停。

Skill 也会根据你正在做的事自动激活，例如设计 API 会触发 `api-and-interface-design`，构建 UI 会触发 `frontend-ui-engineering`，依此类推。

---

## 快速开始

**最快路径，任何 Agent 都只需一行命令。** 开源的 [skills CLI](https://github.com/vercel-labs/skills) 会把 Skill 安装到 70+ 个 Agent 中（Claude Code、Cursor、Codex、Copilot、Cline 等等）：

```bash
npx skills add addyosmani/agent-skills            # 安装全部 25 个 Skill
npx skills add addyosmani/agent-skills --list     # 安装前先浏览
```

或者按需挑选部分 Skill：

```bash
npx skills add addyosmani/agent-skills --skill code-review-and-quality   # 合并前的五维度评审
npx skills add addyosmani/agent-skills --skill interview-me              # 一次一个问题地澄清需求
npx skills add addyosmani/agent-skills --skill test-driven-development   # 红绿重构强制落地
```

> **只装一个 Skill 时：** 单 Skill 的 `npx` 安装只复制 `skills/<name>/`，不会带上仓库级的 `references/` 目录。Skill 本身能正常工作，但路径指向的共享清单无法访问。可以用整仓集成、改用 `git clone` 仓库，或者把所需清单复制到安装出的 Skill 内的 `references/` 目录。这个可移植性缺口已在 [#361](https://github.com/addyosmani/agent-skills/issues/361) 中追踪。

倾向于使用原生集成？请从下面挑选工具。

<details>
<summary><b>Claude Code（推荐）</b></summary>

**Marketplace 安装：**

```
/plugin marketplace add addyosmani/agent-skills
/plugin install agent-skills@addy-agent-skills
```

> **SSH 出错？** Marketplace 通过 SSH 克隆仓库。如果你没有在 GitHub 上配置 SSH 密钥，要么[添加你的 SSH 密钥](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)，要么用完整 HTTPS URL 强制在 `marketplace-add` 步骤中使用 HTTPS 克隆：
> ```bash
> /plugin marketplace add https://github.com/addyosmani/agent-skills.git
> /plugin install agent-skills@addy-agent-skills
> ```
>
> 如果 `/plugin install` 仍然报 `git@github.com: Permission denied (publickey)`（Windows 或 macOS），建议用 Git 的全局配置把 GitHub 的 SSH URL 重写成 HTTPS，以便子进程能正常克隆：
> ```bash
> git config --global url."https://github.com/".insteadOf git@github.com:
> ```

**本地 / 开发模式：**

```bash
git clone https://github.com/addyosmani/agent-skills.git
claude --plugin-dir /path/to/agent-skills
```

</details>

<details>
<summary><b>Cursor</b></summary>

把工作流 Skill 放到 `.cursor/skills/`（从 `agent-skills/skills/` 同步），简短的策略放到 `.cursor/rules/*.mdc`，不要把完整的 Skill 粘贴进规则文件。这两个目录都在项目里，本仓库并不随带 `.cursor/` 文件夹。详情见 [docs/cursor-setup.md](docs/cursor-setup.md)。

</details>

<details>
<summary><b>Antigravity CLI</b></summary>

作为原生插件安装 Skill 和子 Agent。在某些 Antigravity CLI 版本中，旧版命令的 TOML 会显示已转换，但其包装的命令无法被发现，请直接调用底层的命名空间 Skill。参见 [docs/antigravity-setup.md](docs/antigravity-setup.md#lifecycle-workflows-and-command-compatibility)。

**从仓库安装：**

```bash
agy plugin install https://github.com/addyosmani/agent-skills.git
```

**从本地克隆安装：**

```bash
git clone https://github.com/addyoshani/agent-skills.git
agy plugin install ./agent-skills
```

> 注：上面第二行命令中的 `agent-skills` 路径指的是克隆到本地的目录名。如果用作汉化版本，目录应改为 `agent-skills-zh` 并在 Skill 名字前加 `chinese-` 前缀。

</details>

<details>
<summary><b>Gemini CLI</b></summary>

安装为原生 Skill 走自动发现，或者加入 `GEMINI.md` 作为持久上下文。参见 [docs/gemini-cli-setup.md](docs/gemini-cli-setup.md)。

**从仓库安装：**

```bash
gemini skills install https://github.com/addyosmani/agent-skills.git --path skills
```

**从本地克隆安装：**

```bash
gemini skills install ./agent-skills/skills/
```

</details>

<details>
<summary><b>Windsurf</b></summary>

把 Skill 内容加入 Windsurf 的规则配置中。参见 [docs/windsurf-setup.md](docs/windsurf-setup.md)。

</details>

<details>
<summary><b>OpenCode</b></summary>

复制 Skill 到 `.opencode/skills/`（或 `~/.config/opencode/skills/`），并在项目级添加一份 `AGENTS.md`，使用内建的 `skill` 工具由 Agent 自动执行。可以在 `.opencode/commands/` 中添加可选的斜杠命令。

详见 [docs/opencode-setup.md](docs/opencode-setup.md)。

</details>

<details>
<summary><b>GitHub Copilot</b></summary>

将 `agents/` 下的 Agent 定义用作 Copilot 角色，Skill 内容放在 `.github/copilot-instructions.md`。生命周期类斜杠命令（`/spec`、`/plan`、`/build`、`/test`、`/review`、`/ship`）是 Claude Code 的命令，Copilot 里没有；改用 Skill 名称直接调用（如 `/spec-driven-development`）。参见 [docs/copilot-setup.md](docs/copilot-setup.md)。

如果使用独立的 `copilot` CLI，请作为插件安装，参见 [docs/copilot-cli-setup.md](docs/copilot-cli-setup.md)。

</details>

<details>
  <summary><b>Kiro IDE 与 CLI </b></summary>
  Kiro 的 Skill 放在 `.kiro/skills/` 下，可以放在 Project 或 Global 层级。Kiro 也支持 Agents.md。详见 Kiro 文档 https://kiro.dev/docs/skills/
</details>

<details>
<summary><b>Codex</b></summary>

安装为原生 Codex 插件（Codex CLI v0.122+）：

```bash
codex plugin marketplace add addyosmani/agent-skills
codex plugin add agent-skills@agent-skills
```

第一条命令注册 Marketplace，第二条安装插件。Codex 通过 `.codex-plugin/plugin.json` 直接读取根目录的 `skills/`。安装后，在对话中使用 `@` 调用 Skill，例如 `@spec-driven-development`。本地安装与排错见 [docs/codex-setup.md](docs/codex-setup.md)。

</details>

<details>
<summary><b>Command Code</b></summary>

通过内建的 `cmd skills` 命令原生安装。Command Code 会克隆仓库、发现每一个 `SKILL.md`，并安装到 `.commandcode/skills/`：

```bash
cmd skills add addyosmani/agent-skills            # 选择要安装的 Skill（项目级）
cmd skills add addyosmani/agent-skills --global   # 装到所有项目（~/.commandcode/skills/）
cmd skills add addyosmani/agent-skills -s spec-driven-development  # 安装某一个 Skill
```

安装完成之后，TUI 的斜杠菜单里就会出现，例如 `/spec-driven-development`。参见 [docs/commandcode-setup.md](docs/commandcode-setup.md)。

</details>

<details>
<summary><b>其他 Agent</b></summary>

Skill 是纯 Markdown，可以被任何接受系统提示词或指令文件的 Agent 直接使用。参见 [docs/getting-started.md](docs/getting-started.md)。上面没有列出的、但已支持安装本套件的宿主放在 [docs/other-hosts.md](docs/other-hosts.md)。

</details>

---

## 落地

已经装好了？怎么把套件在代码库里推开，取决于你的项目。[**采纳指南**](docs/adoption-guide.md) 给出两条路径：新项目从第 1 天就上完整生命周期；存量代码库走增量、可验证优先。

---

## 全部 25 个 Skill

上面的命令只是入口。整套包含 25 个 Skill，24 个生命周期 Skill 加上 `using-agent-skills` 元 Skill。每个 Skill 都是结构化的工作流：步骤、验证门槛、防甩锅表。也可以直接引用任意 Skill。

### Meta：找到适合当前任务的 Skill

| Skill | 作用 | 何时使用 |
|-------|-------------|----------|
| [using-agent-skills](skills/chinese-using-agent-skills/SKILL.md) | 把进入的工作映射到合适的工作流，并定义共享的操作规则 | 启动会话，或决定使用哪个 Skill |

### Define：把要做的事说清楚

| Skill | 作用 | 何时使用 |
|-------|-------------|----------|
| [interview-me](skills/chinese-interview-me/SKILL.md) | 一次一个问题地从用户那里抽出真正想要的东西（而不是他们以为想要的东西），直到置信度约 95% | 需求笼统，或用户直接说 "interview me" / "grill me" |
| [idea-refine](skills/chinese-idea-refine/SKILL.md) | 用结构化的发散、收敛思考，把模糊想法变成具体提案 | 你有一个粗糙的概念需要先发散 |
| [spec-driven-development](skills/chinese-spec-driven-development/SKILL.md) | 在写代码之前，先写一份覆盖目标、命令、结构、代码风格、测试、边界的 PRD | 启动新项目、新功能，或做重要改动 |
| [constraint-driven-development](skills/chinese-constraint-driven-development/SKILL.md) | 通过问询得出项目的质量底线，提供合理的默认阈值，把每条写入 CONSTRAINTS.md，按代价排好检查项，捕获 agent 静默关闭检查或跳过测试以求 "绿" 的行为 | 没有书面标准，或 agent 写得比谁读得都快 |

### Plan：拆分任务

| Skill | 作用 | 何时使用 |
|-------|-------------|----------|
| [planning-and-task-breakdown](skills/chinese-planning-and-task-breakdown/SKILL.md) | 把规格拆成可验证的小任务，配验收标准和依赖顺序 | 规格已经具备，需要可实施的小单元 |

### Build：写代码

| Skill | 作用 | 何时使用 |
|-------|-------------|----------|
| [incremental-implementation](skills/chinese-incremental-implementation/SKILL.md) | 横向薄片，实现、测试、验证、提交；用特性开关、安全默认值、可回滚的设计 | 改动跨越多个文件 |
| [test-driven-development](skills/chinese-test-driven-development/SKILL.md) | 红绿重构、测试金字塔（80/15/5）、测试分层、DAMP over DRY、Beyonce Rule、浏览器测试 | 写逻辑、修 bug、改变行为 |
| [context-engineering](skills/chinese-context-engineering/SKILL.md) | 在合适的时机喂给 agent 正确的信息（规则文件、上下文打包、MCP 集成） | 启动会话、切换任务，或输出质量开始下降 |
| [source-driven-development](skills/chinese-source-driven-development/SKILL.md) | 任何框架决策以官方文档为依据，验证、引用来源、标出未经证实的部分 | 你希望任何框架 / 库都有权威、带引用的代码 |
| [doubt-driven-development](skills/chinese-doubt-driven-development/SKILL.md) | 对飞行中每一次非平凡决策做对抗式 fresh-context 复核：CLAIM → EXTRACT → DOUBT → RECONCILE → STOP，可选用户授权的跨模型升级 | 风险高（生产、安全、不可逆）、在不熟悉的代码里工作，或现在验证比将来调试更便宜 |
| [frontend-ui-engineering](skills/chinese-frontend-ui-engineering/SKILL.md) | 组件架构、设计系统、状态管理、响应式、WCAG 2.1 AA 可访问性 | 构建或修改面向用户的界面 |
| [api-and-interface-design](skills/chinese-api-and-interface-design/SKILL.md) | 契约先行、Hyrum 定律、单版本规则、错误语义、边界校验 | 设计 API、模块边界、公有接口 |

### Verify：证明它能跑

| Skill | 作用 | 何时使用 |
|-------|-------------|----------|
| [browser-testing-with-devtools](skills/chinese-browser-testing-with-devtools/SKILL.md) | Chrome DevTools MCP 提供运行时数据：DOM 检查、控制台日志、网络抓包、性能 profiling | 构建或调试任何运行在浏览器里的东西 |
| [debugging-and-error-recovery](skills/chinese-debugging-and-error-recovery/SKILL.md) | 五步分诊：复现、定位、缩小、修、设防；停线规则；安全回退 | 测试失败、构建打断、行为异常 |

### Review：合并前的质量门槛

| Skill | 作用 | 何时使用 |
|-------|-------------|----------|
| [code-review-and-quality](skills/chinese-code-review-and-quality/SKILL.md) | 五维度评审、变更体量（约 100 行）、严重级标签（Nit/Optional/FYI）、评审节奏、拆分策略 | 合并任何改动前 |
| [code-simplification](skills/chinese-code-simplification/SKILL.md) | Chesterton's Fence、500 行规则、在保留行为的前提下降低复杂度 | 代码能跑但比应有的难读、难维护 |
| [security-and-hardening](skills/chinese-security-and-hardening/SKILL.md) | OWASP Top 10 预防、认证模式、密钥管理、依赖审计、三层边界 | 处理用户输入、认证、数据存储、外部集成 |
| [performance-optimization](skills/chinese-performance-optimization/SKILL.md) | 先测量后优化：Core Web Vitals 阈值、profiling 流程、bundle 分析、反模式检测 | 有性能需求，或怀疑性能回退 |

### Ship：稳定上线

| Skill | 作用 | 何时使用 |
|-------|-------------|----------|
| [git-workflow-and-versioning](skills/chinese-git-workflow-and-versioning/SKILL.md) | Trunk-based development、原子提交、变更体量（约 100 行）、commit 作为存档点的范式 | 做任何代码改动（永远适用） |
| [ci-cd-and-automation](skills/chinese-ci-cd-and-automation/SKILL.md) | Shift Left、更快更安全、特性开关、质量门槛流水线、失败反馈环 | 配置或修改构建、部署流水线 |
| [deprecation-and-migration](skills/chinese-deprecation-and-migration/SKILL.md) | 代码即负债、强制 vs 顾问式弃用、迁移模式、僵尸代码清理 | 下线旧系统、迁移用户、停用特性 |
| [documentation-and-adrs](skills/chinese-documentation-and-adrs/SKILL.md) | 架构决策记录（ADR）、API 文档、内联文档规范，文档化 "为什么" | 拍架构决策、改 API、上功能 |
| [observability-and-instrumentation](skills/chinese-observability-and-instrumentation/SKILL.md) | 结构化日志、RED 指标、OpenTelemetry 链路追踪、基于症状的告警，边写边埋点 | 加可观测性，或任何会在生产中跑的东西 |
| [shipping-and-launch](skills/chinese-shipping-and-launch/SKILL.md) | 上线前清单、特性开关生命周期、分阶段放量、回滚预案、监控配置 | 准备上生产 |

---

## Agent 角色

针对性强、预配置的角色，用于专项评审：

| Agent | 角色 | 视角 |
|-------|------|------|
| [code-reviewer](agents/code-reviewer.md) | 资深主任工程师 | 五维度代码评审，标准是 "staff engineer 会通过吗" |
| [test-engineer](agents/test-engineer.md) | 测试专家 | 测试策略、覆盖率分析、Prove-It 模式 |
| [security-auditor](agents/security-auditor.md) | 安全工程师 | 漏洞发现、威胁建模、OWASP 评估 |
| [web-performance-auditor](agents/web-performance-auditor.md) | Web 性能工程师 | Core Web Vitals 审计（Quick/Deep 模式），配"指标诚实原则"；以 `/webperf` 调用 |

决策矩阵、编排规则、角色与 Skill / 命令如何协同见 [docs/agents.md](docs/agents.md)。

---

## 参考清单

需要时由 Skill 加载的速查资料：

| 参考文件 | 覆盖内容 |
|-----------|--------|
| [definition-of-done.md](references/definition-of-done.md) | 跨项目都需越过的"完成"基线，区别于每个任务的验收标准 |
| [testing-patterns.md](references/testing-patterns.md) | 测试结构、命名、Mock、React/API/E2E 示例、反模式（JavaScript/TypeScript） |
| [security-checklist.md](references/security-checklist.md) | 提交前检查、认证、输入校验、响应头、CORS、OWASP Top 10 |
| [performance-checklist.md](references/performance-checklist.md) | Core Web Vitals 目标、前后端清单、测量命令 |
| [accessibility-checklist.md](references/accessibility-checklist.md) | 键盘导航、屏幕阅读器、视觉设计、ARIA、测试工具 |
| [observability-checklist.md](references/observability-checklist.md) | 值班提问、结构化日志、RED/USE 指标、链路追踪、基于症状的告警、上线前门 |
| [orchestration-patterns.md](references/orchestration-patterns.md) | 官方背书的多角色编排模式、反模式，以及 "personas don't invoke personas" 的规则 |

---

## Skill 是怎么工作的

每个 Skill 都有统一的结构：

```
┌─────────────────────────────────────────────────┐
│  SKILL.md                                       │
│                                                 │
│  ┌─ Frontmatter（元数据）────────────────────┐  │
│  │ name: lowercase-hyphen-name               │  │
│  │ description: Guides agents through [task].│  │
│  │              Use when…                    │  │
│  └───────────────────────────────────────────┘  │
│  Overview         → 这个 Skill 是干嘛的        │
│  When to Use      → 触发条件                    │
│  Process          → 一步步工作流                │
│  Rationalizations → 借口与反驳                  │
│  Red Flags        → 出错的信号                  │
│  Verification     → 证据要求                    │
└─────────────────────────────────────────────────┘
```

**关键设计取舍：**

- **流程，不是散文。** Skill 是 Agent 要遵循的工作流，不是参考资料。每个都有步骤、检查点和退出条件。
- **防甩锅。** 每个 Skill 都附了一张"借口与反驳"表，覆盖 Agent 跳过步骤的常见理由（如"我待会补测试"），并给出反驳。
- **证据是不可商量的。** 每个 Skill 末尾都列出证据要求：测试通过、构建输出、运行时数据。"看着对"永远不够。
- **渐进披露。** `SKILL.md` 是入口；辅助参考文件仅在需要时加载，token 用量最低。

---

## 项目结构

可移植的核心放在共享目录中。宿主专属路径是各 Agent 自身的发现约定，不是品牌别名；改或者合并会破坏那些扫描固定路径的工具。

| 层 / 消费者 | 仓库路径 | 用途 |
|---|---|---|
| 共享工作流核心 | `skills/`（25 个 Skill） | 被每个集成使用的可移植 `SKILL.md` 工作流 |
| 共享评审材料 | `agents/`（4 个角色）、`references/`（7 个清单） | 由整仓安装带入的专家评审和清单 |
| Claude Code 适配层 | `.claude/commands/`（9 个命令）、`.claude-plugin/`、`hooks/` | 斜杠命令封装、Marketplace 元数据、生命周期钩子 |
| Gemini CLI 适配层 | `.gemini/commands/`（9 个命令） | Gemini 原生 TOML 命令封装 |
| Antigravity CLI 适配层 | `commands/`（9 个命令）、`plugin.json` | 旧版 TOML 封装和根目录插件清单；参见 [已知包装限制](docs/antigravity-setup.md#lifecycle-workflows-and-command-compatibility) |
| Codex 适配层 | `.codex-plugin/`、`.agents/plugins/` | Codex 插件元数据、Marketplace 注册；Codex 直接消费 `skills/` |
| GitHub Copilot CLI 适配层 | `plugin.json` | 根插件元数据；Copilot CLI 按约定发现 `skills/`，并不注册生命周期包装命令 |
| 贡献者工具链 | `scripts/`（13 个脚本）、`evals/`（25 个 case 文件）、`.github/workflows/` | 校验、路由评测、CI |
| 文档 | `docs/` | 通用指南和按工具的安装指南 |

没有自带适配目录的工具，会把共享的 `skills/` 核心装进或复制到它们自己的原生位置。详见 [快速开始](#快速开始) 里的安装链接。

---

## 为什么需要 Agent Skills？

AI 编程 Agent 默认走最短路径，往往意味着跳过规格、测试、安全评审，以及让软件可靠的那些实践。Agent Skills 通过结构化的工作流把资深工程师带到生产代码上的纪律强加进去。

每个 Skill 都封装了来之不易的工程判断：什么时候写规格、测什么、怎么评审、什么时候上线。这些不是泛泛的提示词，而是带强烈观点的、流程驱动的工作流，正是它们把生产级工作与原型级工作区分开。

这些 Skill 沉淀了 Google 工程文化的最佳实践，包括 [《Software Engineering at Google》](https://abseil.io/resources/swe-book) 和 Google 的 [工程实践指南](https://google.github.io/eng-practices/) 中的若干概念。API 设计里有 Hyrum 定律，测试里有 Beyonce Rule 和测试金字塔，代码评审里有变更体量与评审节奏，简化里有 Chesterton's Fence，Git 工作流里有 trunk-based development，CI/CD 里有 Shift Left 与特性开关，还专门有一个弃用 Skill 把代码视作负债。这些不是抽象原则，它们被嵌入 Agent 一步步遵循的工作流。

---

## 与同类的对比

想和 [Superpowers](https://github.com/obra/superpowers) 或 [Matt Pocock 的 skills](https://github.com/mattpocock/skills) 比一比？请看 [**docs/comparison.md**](docs/comparison.md)，里面有诚实的横向对比，包括一次受控的 [端到端对比实验](https://www.linkedin.com/pulse/superpowers-vs-agent-skills-faster-shipping-safer-reasoning-om-mishra-dzakf/) 链接。

---

## 贡献

Skill 应当**具体**（可执行的步骤，而不是空泛建议）、**可验证**（明确的退出条件和证据要求）、**经过实战检验**（基于真实工作流）、**最小化**（只保留真正能引导 Agent 走对的部分）。

格式规范见 [docs/skill-anatomy.md](docs/skill-anatomy.md)，贡献指南见 [CONTRIBUTING.md](CONTRIBUTING.md)，关于在不破坏可移植的 `SKILL.md` frontmatter 的前提下应用模型路由、工具限制等 per-agent 运行时控制见 [docs/advanced-per-agent-configuration.md](docs/advanced-per-agent-configuration.md)。

---

## 团队

agent-skills 的构建与维护者：

| | 姓名 | GitHub | 角色 |
|---|------|--------|------|
| <img src="https://github.com/addyosmani.png?size=120" width="60" height="60" alt="Addy Osmani"> | **Addy Osmani** | [@addyosmani](https://github.com/addyosmani) | 创建者 |
| <img src="https://github.com/federicobartoli.png?size=120" width="60" height="60" alt="Federico Bartoli"> | **Federico Bartoli** | [@federicobartoli](https://github.com/federicobartoli) | 协作者 |
| <img src="https://github.com/nucliweb.png?size=120" width="60" height="60" alt="Joan León"> | **Joan León** | [@nucliweb](https://github.com/nucliweb) | 协作者 |

---

## 许可证

MIT，可以在你的项目、团队和工具中使用这些 Skill。

---

## 译注（非上游原文）

这是个人维护的汉化版本，方便中文使用者阅读。**上游明确不接受文档 / Skill 的翻译合并**（见 CONTRIBUTING.md 中的 "Translations" 一节），此汉化仓库长期与上游可能不同步；如发现内容过时，请优先以 [上游仓库](https://github.com/addyosmani/agent-skills) 为准，并欢迎提 Issue 提醒更新。目录命名约定：

- 仓库根目录：`agent-skills-zh`
- 每个 Skill 子目录：`chinese-<原 skill 名>`
- 内部子目录、文件原名保留
- 所有 key / 路径 / 命令 / Issue 编号 / URL / commit SHA 等技术锚点保持原文

### 建议的 GitHub 仓库主题（Topics）

在 GitHub 仓库页 **About → Topics** 处建议贴入下列主题，便于检索：

`agent-skills` `claude-code` `claude-skills` `ai-agents` `agentic-coding` `developer-tools` `chinese` `simplified-chinese` `translation` `spec-driven-development` `test-driven-development` `code-review` `security` `performance` `tutorial` `documentation` `mit-license`

### 仓库描述建议

`Addy Osmani 的 agent-skills 仓库简体中文翻译版，包含 25 个 Skill、4 个 agent persona、跨 Claude Code / Cursor / Gemini CLI / Antigravity / OpenCode / Windsurf / Copilot / Codex / Command Code 的安装指南。`
