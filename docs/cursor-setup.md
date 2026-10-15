# 在 Cursor 上使用 agent-skills

如何用当前受支持的项目上下文（而不是过时的单一巨型文件或 Kaizen 专属布局）把 [agent-skills](../README.md) 接入 **Cursor**。

本指南中所有 `.cursor/` 路径都存在于**你的**项目（或 `~/.cursor/` 下的用户作用域）。本仓库不附带 `.cursor/` 目录：你需要自己创建它，并把 skill 同步进去，这正是[第 1 步](#1把-skill-装进-cursorskills)所做的事。

---

## Cursor 当前支持什么

Cursor 把 **rules**（简短策略）与 **skills**（完整工作流）组合在一起：

| 层 | 路径 | 角色 |
|-------|------|------|
| **项目规则（Project rules）** | `.cursor/rules/*.mdc` | 始终生效或按文件作用域的指令（`alwaysApply`、`globs`） |
| **项目技能（Project skills）** | `.cursor/skills/<skill-name>/SKILL.md` | Agent 自动发现的工作流；在任务匹配 skill `description` 时被读入 |
| **用户规则（User rules）** | Cursor Settings → Rules | 账号级策略 |
| **用户技能（User skills）**（可选） | `~/.cursor/skills/` | 在每个工作区都可用的全局 Skill |

文档：[Rules](https://docs.cursor.com/context/rules) · [Skills](https://docs.cursor.com/context/skills)（Cursor 更新文档时 URL 可能跳转）。

### Rule 与 Skill 的差别

- **Rule** ， 简洁、稳定（"用 conventional commits"、"为公开 Python API 加类型注解"）。一个文件最好只管一件事；不要贴大段指南。
- **Skill** ， 本仓库里的分步流程（`test-driven-development`、`code-review-and-quality` 等）。**不要**把整份 `SKILL.md` 拷进 rule；那样会与 `.cursor/skills/` 重复，浪费上下文。

### 旧版（新建配置时请避开）

| 旧版 | 推荐 |
|--------|--------|
| 根目录 `.cursorrules` | `.cursor/rules/*.mdc` |
| 把 `SKILL.md` 拷到 `.cursor/rules/` | `.cursor/skills/<name>/SKILL.md` |
| "把 10 个 skill 当作 always-on 规则全开" | 1–2 条精简的 `alwaysApply` 规则 + 按需 Skill |

---

## 推荐的项目布局

```text
your-project/
├── .cursor/
│   ├── rules/                    # 简短的 .mdc 策略（你自己的）
│   │   └── agent-skills.mdc      # 可选："使用项目 Skill" 的提示
│   └── skills/                   # Cursor Agent 加载的内容
│       ├── using-agent-skills/
│       ├── test-driven-development/
│       ├── code-review-and-quality/
│       └── …                     # 从 agent-skills 同步 + 你自己的 Skill
└── agent-skills/                 # 可选：git submodule 或 vendored 克隆
    └── skills/                   # 仅作为上游源
```

**给 Agent 的真理源：** `.cursor/skills/`。
把 `agent-skills/skills/`（或克隆的 [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills)）当作**上游**，同步进 `.cursor/skills/`，不要只改上游就指望 Cursor 看见。

---

## 配置（任意仓库）

#### 1. 把 Skill 装进 `.cursor/skills/`

**从 agent-skills 的本地克隆**（项目根或其它位置）：

```bash
mkdir -p .cursor/skills
rsync -a /path/to/agent-skills/skills/ .cursor/skills/
```

**首次复制、不覆盖你自己的 Skill：**

```bash
rsync -a --ignore-existing /path/to/agent-skills/skills/ .cursor/skills/
```

**上游更新后：**

```bash
rsync -a /path/to/agent-skills/skills/ .cursor/skills/
```

每个 Skill 文件夹必须至少包含带 YAML frontmatter 的 `SKILL.md`：

```yaml
---
name: test-driven-development
description: Drives development with tests. Use when implementing logic, fixing bugs, or changing behavior.
---
```

Cursor 用 `description`（以及相关元数据）判断何时应用 Skill。

#### 2. 添加最少的项目规则（可选，但有用）

创建 `.cursor/rules/agent-skills.mdc`：

```markdown
---
description: Use agent-skills workflows from .cursor/skills
alwaysApply: true
---

Before non-trivial technical work:

1. Route via `.cursor/skills/using-agent-skills/SKILL.md`.
2. Read and follow the matching skill under `.cursor/skills/<name>/SKILL.md`.
3. Open `reference.md` in that folder when the skill links to it.
4. Prefer project skills over guessing; user does not need to say "read skill" each time.
```

为仓库特有标准（风格、语言、栈）添加**独立的** `.mdc` 文件，让每个文件聚焦一件事。

**Rule 文件格式：**

```markdown
---
description: Shown in Cursor rule UI
alwaysApply: false
globs: "**/*.{ts,tsx}"
---

# Your rule content
```

| 字段 | 用途 |
|-------|-----|
| `alwaysApply: true` | 该项目的每次聊天 |
| `globs` | 当匹配文件出现在上下文时 |
| `alwaysApply: false` + 无 globs | Agent 主动请求 / 手工 rule（Cursor UI） |

#### 3. 用户级 Skill（可选）

把你希望在所有项目里都有的 Skill 复制或安装到 `~/.cursor/skills/`。用于不属于 agent-skills 的全栈指南（例如语言模式）。

项目级 `.cursor/skills/` 中的 Skill 在**本仓库**的工作流里优先。

#### 4. 校验

1. **Settings → Rules** ， 列出项目的 `.mdc` 文件。
2. **Agent chat** ， `.cursor/skills/` 中的 Skill 出现在 Skill 列表里（若你的 Cursor 构建支持）。
3. 跑一个能映射到某 Skill 的任务（例如 "add a feature with tests first"），不指定文件名，路由生效时 Agent 应打开 `test-driven-development`。

---

## Agent 该如何使用 Skill

1. **发现** ， `using-agent-skills` 把任务阶段映射到 Skill 名。
2. **读入** ， 完整流程在 `.cursor/skills/<name>/SKILL.md`。
3. **深挖** ， 当 Skill 指向时，打开 `reference.md`、`references/*.md` 或链接到的检查清单。
4. **组合** ， 例如 `incremental-implementation` + `api-and-interface-design` 做一个 API 切片。

用户显式的措辞（"follow TDD"、"use code-review-and-quality"）在 Agent 跑偏时仍有用。

### 阶段 → Skill（速查）

| 你正在…… | Skill |
|----------|--------|
| 澄清需求 | `interview-me`、`idea-refine`、`spec-driven-development` |
| 规划任务 | `planning-and-task-breakdown` |
| 实现 | `incremental-implementation`、`frontend-ui-engineering`、`api-and-interface-design` |
| 测试 | `test-driven-development`、`browser-testing-with-devtools` |
| 调试 | `debugging-and-error-recovery` |
| 评审 | `code-review-and-quality`、`code-simplification` |
| 安全 / 性能 | `security-and-hardening`、`performance-optimization` |
| Git / CI / 发布 | `git-workflow-and-versioning`、`ci-cd-and-automation`、`shipping-and-launch` |

完整树：`skills/using-agent-skills/SKILL.md`。

---

## 不该做的事

| 避免 | 改用 |
|-------|------------|
| 把所有 Skill 粘进一条 rule | 同步进 `.cursor/skills/` |
| 维护两份逐渐发散的副本 | 从上游 `rsync`；提交 `.cursor/skills/` |
| 写一堆 `alwaysApply: true` 规则 | 一条路由 rule + 聚焦的 globs rule |
| 只靠 `.cursorrules` | 迁移到 `.mdc` + Skill |
| 指望 `agent-skills/agents/*.md` 自动加载 | 粘贴到聊天里，或抽成简短 rule |

---

## 上下文技巧

- **always-on** rule 保持精简（路由 + 1–2 条不可让步）。
- 长检查清单与合理化表格让 **skill** 承担。
- 阶段特定的 **globs** rule 仅在需要时加（例如 `**/*.py`、`**/components/**`）。
- 跳过验证步骤时，用 Skill 名字提醒。

---

## `agents/` 目录

`agent-skills/agents/` 下的文件（例如 code-reviewer persona）**不会**被 Cursor 自动加载。处理方式：

- 引用等价的 Skill（`code-review-and-quality`）。
- 把 agent Markdown 粘贴到聊天里做一次性评审。
- 把**简短的**检查清单抽成 `.mdc` rule。

---

## 排错

| 症状 | 排查点 |
|---------|--------|
| Skill 始终不被使用 | 是否在 `.cursor/skills/<name>/` 下？frontmatter `description` 合法？ |
| Rule 被忽略 | 后缀是 `.mdc`？`alwaysApply` / `globs` 正确？ |
| 工作流是旧的 | 重新 `rsync` 自 `agent-skills/skills/` |
| 指令重复 | 从 rule 中移除 Skill 内容；保留单一来源 |
| 选错了 Skill | 收窄自定义 Skill 的 `description`；在聊天里点名 |
| 在用 `/` 挂载 Skill 之后立刻出现 "An unexpected error occurred on our servers"，且该聊天后续每条消息都失败 | Cursor 侧的已知问题，见 [#494](https://github.com/addyosmani/agent-skills/issues/494)：Cursor 把挂载的 `SKILL.md` 拉进请求、失败发生在它那一侧、与 Skill 本身无关。新开一个聊天，改用纯文本点名 Skill（"Use the test-driven-development skill to …"）。 |

---

## 新项目清单

- [ ] `mkdir -p .cursor/skills` 并从 `agent-skills/skills/` 同步
- [ ] 可选：`.cursor/rules/agent-skills.mdc` 含路由提示
- [ ] 把仓库特有 rule 作为独立精简 `.mdc` 文件添加
- [ ] 提交 `.cursor/skills/` 与 `.cursor/rules/`（团队共享行为）
- [ ] 除非旧工具链要求，否则跳过巨型 `.cursorrules`

---

## 另见

- [getting-started.md](getting-started.md)
- [../README.md](../README.md) ， Cursor 速记
- 上游：[github.com/addyosmani/agent-skills](https://github.com/addyosmani/agent-skills)