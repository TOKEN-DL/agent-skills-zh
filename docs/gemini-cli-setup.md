# 在 Gemini CLI 上使用 agent-skills

## 配置

### 选项 1：作为 Skill 安装（推荐）

Gemini CLI 内置技能系统，会自动发现 `.gemini/skills/` 或 `.agents/skills/` 目录中的 `SKILL.md`。每个 Skill 在任务匹配时按需激活。

**从仓库安装：**

```bash
gemini skills install https://github.com/addyosmani/agent-skills.git --path skills
```

**或从本地克隆安装：**

```bash
git clone https://github.com/addyosmani/agent-skills.git
gemini skills install /path/to/agent-skills/skills/
```

**只为当前工作区安装：**

```bash
gemini skills install /path/to/agent-skills/skills/ --scope workspace
```

工作区作用域安装的 Skill 落在 `.gemini/skills/`（或 `.agents/skills/`）。用户级 Skill 落在 `~/.gemini/skills/`。

安装后用下列命令校验：

```
/skills list
```

Gemini CLI 会自动把 Skill 名称与描述注入到提示词里。当它识别到匹配的任务时，会在加载 Skill 全文前请求授权。

### 选项 2：GEMINI.md（持久上下文）

对想"始终加载"为项目常驻上下文（而不是按需激活）的 Skill，把它们加到项目的 `GEMINI.md`：

```bash
# 创建 GEMINI.md，把核心 Skill 作为持久上下文
cat /path/to/agent-skills/skills/incremental-implementation/SKILL.md > GEMINI.md
echo -e "\n---\n" >> GEMINI.md
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md >> GEMINI.md
```

也可以通过从独立文件导入来模块化：

```markdown
# 项目指令

@skills/test-driven-development/SKILL.md
@skills/incremental-implementation/SKILL.md
```

用 `/memory show` 校验已加载的上下文，用 `/memory reload` 在改动后刷新。

> **Skill vs GEMINI.md：** Skill 是按需激活的专家知识，让上下文窗口保持清爽；GEMINI.md 提供每次提示词都会加载的持久上下文。Skill 用于阶段特定的工作流，GEMINI.md 用于始终生效的项目约定。

## 推荐配置

### 始终生效（GEMINI.md）

把以下作为每次会话的持久上下文加入：

- `incremental-implementation` ， 以小而可验证的切片构建
- `code-review-and-quality` ， 五维度评审

### 按需（Skills）

把以下装成 Skill，仅在相关时激活：

- `test-driven-development` ， 实现逻辑或修 bug 时激活
- `spec-driven-development` ， 开始新项目或新功能时激活
- `frontend-ui-engineering` ， 构建 UI 时激活
- `security-and-hardening` ， 安全评审时激活
- `performance-optimization` ， 性能工作时激活

## 进阶配置

### MCP 集成

本技能包中许多 Skill 利用 [Model Context Protocol（MCP，模型上下文协议）](https://modelcontextprotocol.io/) 工具与环境交互。例如：

- `browser-testing-with-devtools` 使用 `chrome-devtools` MCP 扩展。
- `performance-optimization` 能受益于性能相关的 MCP 工具。

要启用这些，确保 Gemini CLI 配置（`~/.gemini/config.json`）里已经安装了相关 MCP 扩展。

### 会话钩子（Session Hooks）

Gemini CLI 支持会话生命周期钩子。可借此在会话开始时自动注入上下文或运行校验脚本。

若想复刻其他工具中 `agent-skills` 的体验，可配置一个 `SessionStart` 钩子来提醒可用的 Skill 或加载一个 meta-skill。本仓库中的 `hooks/session-start.sh` 是为此准备好的脚本：它把 `using-agent-skills` meta-skill 以标准 `SessionStart` JSON 信封打印到 stdout。

### 显式上下文加载

可以用 `@` 符号在提示词里显式加载任意 Skill 到当前会话：

```markdown
Use the @skills/test-driven-development/SKILL.md skill to implement this fix.
```

在你想确保某个工作流被执行、又不等待自动发现时很有用。

## 斜杠命令

本仓库在 `.gemini/commands/` 下提供 9 条斜杠命令：8 条生命周期命令外加 `/webperf` 专家审计。在项目根跑时，Gemini CLI 自动发现它们。

| 命令 | 作用 |
|---------|--------------|
| `/spec` | 写代码前先写一份结构化 spec |
| `/constraints` | 定义并强制项目的质量基线 |
| `/planning` | 把工作拆成小而可验证的任务 |
| `/build` | 增量实现下一个任务 |
| `/test` | 跑 TDD 工作流，红、绿、重构 |
| `/review` | 五维度代码评审 |
| `/code-simplify` | 在不改变行为的前提下降低复杂度 |
| `/ship` | 通过并行 persona 扇出跑发布前检查清单 |
| `/webperf` | 对面向浏览器的应用审计 Core Web Vitals 与性能问题 |

每条命令都会自动调用对应的 Skill，无需手动加载。

> **注意：** 用 `/planning` 代替 `/plan`，`/plan` 与 Gemini CLI 内部命令名冲突。

## 使用技巧

1. **优先用 Skill 而非 GEMINI.md** ， Skill 按需激活，让上下文窗口保持聚焦。仅当希望 Skill 始终生效时才放进 GEMINI.md。
2. **Skill 描述很重要** ， 每个 `SKILL.md` 在 frontmatter 里都有一个 `description` 字段，告诉 Agent 何时激活。本仓库的描述针对跨所有支持工具（Claude Code、Gemini CLI 等）的自动发现做了优化，明确写出 Skill 的*作用*与*何时触发*。
3. **用 persona 做评审** ， 申请结构化代码评审时，复制 `agents/code-reviewer.md` 内容。
4. **与 references 配合** ， 当处理测试或性能等特定质量领域时，引用 `references/` 中的检查清单。