# Skill 解剖

本文描述 agent-skills 的 Skill 文件结构与格式。在贡献新 Skill 或理解既有 Skill 时作为指南。

## 文件位置

每个 Skill 都放在 `skills/` 下的独立目录里：

```
skills/
  skill-name/
    SKILL.md           # 必需：Skill 定义
    scripts/           # 可选：Skill 工作流使用的可跑辅助脚本
    references/        # 可选：Skill 特定的参考文档
    supporting-file.md # 可选：按需加载的参考资料
```

`SKILL.md` 是唯一必需的文件。只有当 Skill 真正需要时才加 `scripts/` 或 `references/`；简单的 Skill 直接省略。

## SKILL.md 格式

### Frontmatter（必需）

```yaml
---
name: skill-name-with-hyphens
description: Guides agents through [task/workflow]. Use when [specific trigger conditions].
---
```

**规则：**
- `name`：小写，连字符分隔。必须与目录名匹配。
- `description`：以第三人称描述 Skill 做了什么，然后给出 1 条或多条明确的 "Use when" 触发条件。同时包含*做什么*与*何时用*。最多 1024 字符。
- 顶层键被限制在 [Agent Skills 规范](https://agentskills.io) 字段：`name`、`description`、`license`、`compatibility`、`metadata`、以及实验性的 `allowed-tools`。`model`、`tools`、`max_turns`、`context` 等供应商或运行时控制项应放在 `metadata` 下或单独的按 Agent 适配器文件里；见 [docs/advanced-per-agent-configuration.md](advanced-per-agent-configuration.md)。校验器会拒绝任何其它顶层键。

已发布的 name 是兼容性标识符。尤其地，`browser-testing-with-devtools` 是稳定的上游名，因为其它 Skill 直接按这个名字引用它。下游目录可以重命名它，但那是该目录自己负责别名或迁移映射；本仓库不维护仅下游的别名。

**为什么这要紧：** Agent 通过 description 来发现 Skill。description 会被注入到系统提示词中，因此它必须同时告诉 Agent "Skill 提供什么" 与 "何时激活它"。不要把工作流总结进 description，若 description 写了流程步骤，Agent 可能照着那份总结走，而不读完整的 Skill。

### 标准章节（推荐模式）

上面的 frontmatter 约定是必需的。下面是推荐的章节布局，不是死板模板：等价的章节标题在清晰表达同样目的时也是可接受的。

```markdown
# Skill 标题

## 概览
一两句话解释这个 Skill 做什么、为什么重要。

## 何时使用
- 触发条件列表（症状、任务类型）
- 何时**不**用（排除项）

## [核心流程 / 工作流 / 步骤]
推荐流程，分解为编号步骤或阶段。
需要时含代码示例。
在存在决策点的位置用流程图（ASCII）。

## [特定技巧 / 模式]
针对具体场景的详细指导。
代码示例、模板、配置。

## 常见借口
| 借口 | 现实 |
|---|---|
| Agent 用于跳过步骤的借口 | 为什么借口是错的 |

## 红旗
- 表明 Skill 正在被违反的行为模式
- 评审中要注意的事

## 验证
完成 Skill 流程后，确认：
- [ ] 退出标准清单
- [ ] 证据要求
```

## 章节用途

### 概览（Overview）
Skill 的"电梯演讲"。应回答：这个 Skill 做什么，Agent 为什么该遵循它？

### 何时使用（When to Use）
帮助 Agent 与人判断本 Skill 是否适用于当前任务。包含正向触发（"用于 X 时"）与负向排除（"不用于 Y"）。

### 核心流程（Core Process）
Skill 的核心。Agent 要遵循的分步工作流。必须具体且可执行，不是含糊建议。

**好：** "跑 `npm test` 并确认全部测试通过"
**差：** "确保测试能跑"

### 常见借口（Common Rationalizations）
一份打磨良好的 Skill 最具特色的部分。这些是 Agent 用于跳过重要步骤的借口，配以反驳。它阻止 Agent 自圆其说地绕开流程。

想想每次 Agent 说"我稍后补测试"或"这够简单，跳过 spec 吧"的场景，把它们写在这里，给出事实性的反驳。

### 红旗（Red Flags）
Skill 被违反时的可观察迹象。在代码评审与自我监督时有用。

### 验证（Verification）
退出标准。一份 Agent 用来确认 Skill 流程已完成的清单。每个复选框都应能用证据（测试输出、构建结果、截图等）验证。

## 支持文件

仅当出现下列情况时创建支持文件：

- 参考资料超过 100 行（保持主 SKILL.md 聚焦）
- 需要代码工具或脚本
- 检查清单足够长，值得独立成文件

50 行以内的模式与原则直接放正文。

若 Skill 不需要可跑辅助脚本，不要为了与其他 Skill 对齐而建一个空 `scripts/` 目录。空目录只是噪音，不会改变 Skill 的工作方式。

## 共享 References

被多于一个 Skill 使用的检查清单，测试、安全、性能、可访问性、done 定义，放在仓库根的 `references/`，刻意*不*放进任何 Skill 目录内。

这是技能包级别的设计选择。Agent Skills 规范把 Skill 描述为自包含，但本仓里有几个 Skill 指向同一份检查清单。把它们集中到一处会强迫你二选一：要么把检查清单复制到每个引用它的 Skill，要么挑一个 Skill 持有它、其它 Skill 钻进那个目录。两者都会随时间漂移。仓库根单份副本始终是真理源。

权衡是可移植性：整仓安装（例如 Claude Code marketplace 插件）会带上 `references/`，但按 Skill 安装只复制 `skills/<name>/`，仓库根同级目录会被丢下，那些链接就解析不到。这个空白在 [#361](https://github.com/addyosmani/agent-skills/issues/361) 跟踪。

当前约定：只被一个 Skill 用的资料是该 Skill 目录内的支持文件；跨 Skill 共享的资料进 `references/`。

## 上下文效率

Skill 按需加载：启动时上下文中只有 Skill 名与 description。完整 `SKILL.md` 只在 Agent 判定该 Skill 相关时才加载。为了让加载便宜：

- **`SKILL.md` 控制在 500 行以内。** 详细参考资料移到支持文件。
- **description 写得具体。** 精准的 description 帮助 Agent 在恰当时机激活 Skill，其他情况跳过。
- **渐进披露。** 只在工作流到达时，才引用那些会被读入的支持文件。
- **优先用脚本而非内联代码。** 跑脚本不占上下文；只占它输出的内容。内联代码块每次加载都要付。
- **文件引用控制在一层。** 从 `SKILL.md` 直接链到支持文件，而不是经中间文档链下去。

## 脚本要求

当 Skill 携带 `scripts/` 下的可跑辅助脚本时，每个脚本遵循以下约定：

- 用 `#!/bin/bash` shebang。
- 用 `set -e` 实现 fail-fast。
- 状态消息写 stderr：`echo "Message" >&2`。
- 机器可读输出（JSON）写 stdout。
- 含清理临时文件的 trap。
- 脚本路径以仓库相对路径引用：`skills/<skill-name>/scripts/<script>.sh`。

## 写作原则

1. **流程胜于知识。** Skill 是工作流，不是参考文档。要步骤，不要事实。
2. **具体胜于笼统。** "跑 `npm test`" 胜过 "确认测试能用"。
3. **证据胜于假设。** 每个验证复选框都需要证据。
4. **抗合理化。** 每个值得跳过的步骤都需要在借口表里有对应反驳。
5. **渐进披露。** 主 SKILL.md 是入口。支持文件按需加载。
6. **有 token 意识。** 每个章节都要为存在给出理由。删掉它若不会改变 Agent 行为，那就删掉。
7. **模型中立。** 写流程本身，别写绕路。见 [写流程本身，别写绕路](#写流程本身别写绕路)。

## 写流程本身，别写绕路

本技能包在许多 Agent 与模型世代上跑。为某个模型把某个调用弄错而存在的步骤，对每个其他模型都是负债：它限制了本来能直接解决问题的 Agent，也花掉了更强模型做实际工作所需的轮次。跨模型迁移测量（[WikiSkill，arXiv:2608.27454](https://arxiv.org/abs/2608.27454)）显示：按某个模型的执行失败来撰写的 Skill，可能让更强模型表现得*比不用 Skill 还差*。

**规则：** 若一步无法在不提某个模型、模型版本、或某个 Agent 私有工具名的情况下被合理化，那它应当进 issue，而不是进 Skill。描述能力（"跑那个聚焦的测试命令"、"写入那个文件"），而不是某个运行时恰巧暴露的机制（`run_command`、`write_to_file`）。

提案 Skill 内容前要查两种气味：

- **过度规定的机制。** 在只说目标就够时硬规定精确命令形式或序列化细节。若理由是"不然模型 X 会做错"，那只是绕路，不是流程。
- **碎片化的诊断。** 一步步的检查序列，本可一步确认同一件事。每多一步多花一轮，轮次预算是有限的。

这是关于 Skill *内容*的；`references/` *路径* 的可移植性是另一回事（见 [共享 References](#共享-references)）。

## 命名约定

- Skill 目录：`lowercase-hyphen-separated`
- Skill 文件：`SKILL.md`（始终大写）
- 支持文件：`lowercase-hyphen-separated.md`
- 共享 references：存在仓库根的 `references/`，不嵌在 Skill 目录内（理由见 [共享 References](#共享-references)）。
- Skill 专属 references：单份支撑文档可作为松散文件留在 Skill 目录下（见上面"支持文件"项）；当若干相关文档随 Skill 一起走时，自包含、可分发 Skill 的新惯例是把它们收进 Skill 目录下的 `references/`，让 Skill 自带支持文档。

## 跨 Skill 引用

按名字引用其它 Skill：

```markdown
按 `test-driven-development` Skill 写测试。
若构建崩了，使用 `debugging-and-error-recovery` Skill。
```

不要在 Skill 之间复制内容，用引用与链接代替。

## 必需 vs 推荐

必需：

- 一份 `skills/<skill-name>/SKILL.md` 文件
- 合法 YAML frontmatter，含 `name` 与 `description`
- description 同时包含 Skill 的作用与何时使用

推荐：

- 上面展示的标准章节流
- 当 `How It Works`、`Core Process`、`Workflow` 等同类标题在 Skill 里读起来更自然时使用它们
- 只有当能让主 `SKILL.md` 保持聚焦时才用支持文件