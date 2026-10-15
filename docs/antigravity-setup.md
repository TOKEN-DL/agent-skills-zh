# 在 Antigravity CLI（agy）上使用 agent-skills

`agent-skills` 包可以作为原生插件装到 Antigravity CLI（`agy`），让 Agent 能访问结构化工作流与 persona。

## 配置

### 选项 1：原生插件安装（推荐）

Antigravity CLI 有[一等公民的插件系统](https://www.agy.dev/docs/plugins/)，能注册 Skill 与 agent。本仓库也带了旧版命令 TOML，但受影响的 `agy` 版本不会暴露它们的转换包装；参见 [生命周期工作流与命令兼容性](#生命周期工作流与命令兼容性)。

**从远程仓库安装：**

```bash
agy plugin install https://github.com/addyosmani/agent-skills.git
```

**从本地克隆安装：**

1. 克隆仓库：
   ```bash
   git clone https://github.com/addyosmani/agent-skills.git
   ```
2. 用 `agy` 安装插件：
   ```bash
   agy plugin install /path/to/agent-skills
   ```

这会校验插件，并安装到全局 Antigravity 配置目录（`~/.gemini/config/plugins/agent-skills/`）。

> **注意：** 在当前 `agy` 版本中，插件落在 `~/.gemini/config/plugins/`，而不是旧版所用的 `~/.gemini/antigravity-cli/plugins/`。如果在旧路径找不到插件，先查 `~/.gemini/config/plugins/agent-skills/`。

### 选项 2：从 Gemini CLI 导入

若你已经把 `agent-skills` 装在旧版 Gemini CLI 安装下，可以直接导入：

```bash
agy plugin import gemini
```

装好后，校验当前激活插件：

```bash
agy plugin list
```

---

## 生命周期工作流与命令兼容性

Antigravity 的[迁移工具](https://www.agy.dev/docs/cli/gcli-migration/)会把 `commands/*.toml` 中的 9 个旧版定义报告为"已转换为 Skill"。在受影响的 `agy` 1.1.x 版本中，校验通过，但转换后的包装器不会出现在斜杠命令或 Skill 目录里。因此，`agy plugin validate` 成功只说明文件格式正确，并不代表 `/build` 等简短包装可用。这件事在 [agent-skills #445](https://github.com/addyosmani/agent-skills/issues/445) 与上游 [antigravity-cli #788](https://github.com/google-antigravity/antigravity-cli/issues/788) 跟踪。

在该限制未解除前，直接用原生插件的 Skill：

| 想用的包装器 | Antigravity 直接调用 | 备注 |
|------------------|-------------------------------|-------|
| `/spec` | `/agent-skills:spec-driven-development` | 写代码前先写一份结构化 spec |
| `/constraints` | `/agent-skills:constraint-driven-development` | 定义并强制项目的质量基线 |
| `/planning` | `/agent-skills:planning-and-task-breakdown` | Antigravity 自带的 `/planning` 命令是独立的 plan-mode 控制 |
| `/build` | `/agent-skills:incremental-implementation` | 同时调用 `/agent-skills:test-driven-development`；包装器独占的 `/build auto` 编排不可用 |
| `/test` | `/agent-skills:test-driven-development` | 跑 red-green-refactor 工作流 |
| `/review` | `/agent-skills:code-review-and-quality` | 跑五维度评审工作流 |
| `/code-simplify` | `/agent-skills:code-simplification` | 在不改变行为的前提下降低复杂度 |
| `/ship` | `/agent-skills:shipping-and-launch` | 包装器的自动 persona 扇出不可用；分别调用专家 agent |
| `/webperf` | 从 `/agents` 选择 `web-performance-auditor` | 这条工作流是 persona 而非 Skill |

不要为了绕过此问题而给 TOML 文件加 YAML frontmatter。Gemini CLI 用严格解析器读取并行 TOML 命令格式，`---` frontmatter 会让这些文件变成无效 TOML，却不会改变 Antigravity 的转换行为。

---

## 技能与发现

Antigravity 自动发现插件 `skills/` 目录中的 Skill。
* Antigravity 按需把用户任务与意图匹配到相关 Skill。
* 若任务匹配某个 Skill，Agent 会先加载它并向你请求执行授权。

---

## 校验与验证

要校验你的本地插件结构正确、且包含所有 Skill，命令：

```bash
agy plugin validate /path/to/agent-skills
```

然后开新会话，输入 `/agent-skills:` 检查带命名空间的 Skill 目录。若校验器报告命令已转换、但 `/build` 返回 `No matches`，按上面直接调用表使用；重装、或给 TOML 加 `name` 字段都不能解决已知的导入器限制。

---

## 工作原理

### 1. 按需激活 Skill
Antigravity CLI 自动发现已安装插件 `skills/` 目录中的 `SKILL.md`。根据其 frontmatter 中的触发描述，Agent 在检测到匹配的开发者意图时，会动态激活相应的工作流。

例如当你让 Agent：

- **设计新系统** → 它会建议 / 激活 `spec-driven-development`。
- **实现一个特性** → 它会激活 `incremental-implementation` 与 `test-driven-development`。
- **修一个 bug** → 它会激活 `debugging-and-error-recovery`。

### 2. 专家 Agent Persona
插件从 `agents/` 目录注册可复用的子 Agent 定义：

- `code-reviewer.md`
- `security-auditor.md`
- `test-engineer.md`

可在会话中直接调用，或在用子 Agent 委派任务时调用。

---

## 配置与定制

### 项目特定约束（`AGENTS.md`）
为强制严格的 Skill 守门（例如要求在写代码前有 spec 或 plan），把 `AGENTS.md` 复制或链接到工作区根目录。Antigravity CLI 读这个文件，让 Agent 行为与计划阶段对齐你团队的约定。

### 沙箱模式
若想以受限终端权限跑 Skill 或脚本（跑第三方验证测试时为安全），用下列命令启动 CLI：

```bash
agy --sandbox
```

---

## 使用技巧

1. **保持插件为最新：** 用下列命令升级 CLI 或检查新插件版本：
   ```bash
   agy update
   ```
2. **执行前审查：** 当 Agent 用这些 Skill 执行复杂重构任务时，按 `Ctrl+r` 进入 **Artifact Review**（工件评审）屏幕，提交前查看、编辑或批准代码。
3. **控制权限：** 仅在你想绕开手工工具批准提示的可信本地项目里用 `--dangerously-skip-permissions` 标志。