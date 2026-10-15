# 开发者入门

本指南面向**在** agent-skills 仓库上做事的人：贡献 Skill、改文档、完善评测框架。若你想在自己项目里*用*这些 Skill，你应该看的是 [getting-started.md](getting-started.md)。

这是一篇引导式的导览，不是规则书。规则住在 [CONTRIBUTING.md](../CONTRIBUTING.md)（贡献流程）、[skill-anatomy.md](skill-anatomy.md)（Skill 格式）、[evals/README.md](../evals/README.md)（评测框架）；本文告诉你何时读哪一份、它们如何拼起来。

---

## 1. 心智模型

仓库有五层可组合的组件。理解每层*用于什么*能避开最常见的贡献错误（把参考资料塞进 Skill、造一个会路由到其它 persona 的 persona、跨 Skill 复制内容）。

| 层 | 位置 | 职责 | 一句话 |
|---|---|---|---|
| **Skill** | `skills/<name>/SKILL.md` | 带验证关卡的分步工作流 | *怎么用* |
| **Persona** | `agents/<role>.md` | 带视角与输出格式的角色 | *谁来做* |
| **Command** | `.claude/commands/`、`.gemini/commands/`、`commands/` | 用户面入口；编排层 | *何时用* |
| **References** | `references/*.md` | Skill 按需拉入的检查清单 | *查什么* |
| **Evals** | `evals/cases/<name>.json` | 证明 Skill 触发正确且行为正确 | *真有效吗* |

两条值得早早内化的结构性规则：

- **用户（或斜杠命令）才是编排者。** Persona 绝不会调用其它 persona；本仓库背书的多 persona 模式只有并行扇出 + 合并（见 [references/orchestration-patterns.md](../references/orchestration-patterns.md)）。
- **不复制，用引用。** Skill 之间互相链接、链接到 `references/`，而不是复述内容。同样的规则适用于文档，包括本文。

一个把人绊倒过的作用域警告：仓库根的 `AGENTS.md` 与 `CLAUDE.md` 配置的是工作于*本仓库*的 Agent。它们不是可复用资产，安装指南绝不应当让用户把它们拷进自己的项目；可复用的资产是 Skill。

注意命令存在于三个并行目录（Claude Code、Gemini CLI、Antigravity）。改了一处，CI 会检查它们之间的对齐，见 §3。

## 2. 本地配置

```bash
git clone https://github.com/addyosmani/agent-skills.git
cd agent-skills
```

没有构建步骤、也没有 `package.json`；校验器是纯 Node 脚本。你需要：

- **Node 24**（CI 钉死的版本）用于 `scripts/` 校验器
- **bash**（+ 推荐 `jq`）用于 hook 回归测试
- **`gh` CLI** 用于在提案 Skill 前做重复 PR 检查
- **Claude Code** 仅在你本地跑 Tier 3 行为评测时才需要

想拿本地检出版本链路把包跑起来：

```bash
claude --plugin-dir /path/to/agent-skills
```

## 3. 验证循环

仓库自己吃自己的狗粮：验证对 Skill 不可让步，对仓库的贡献也不可让步。CI 跑的所有不依赖 Claude Code CLI 的检查，你都能在本地几秒内跑完。如下，按职责分组（工作流把这些分摊到多个 job 里）：

```bash
# Tier 1，结构：frontmatter、命名、必需章节、版式与行预算
node scripts/validate-skills.js

# 从 skill 到 references/ 的链接都解析，包括 #anchor 锚点
node scripts/validate-reference-links.js

# 每个插件清单都带同一版本
node scripts/validate-versions.js

# 跨三个命令目录的命令对齐与描述同步
node scripts/validate-commands.js

# Spec / plan / todo 工件路径在每个命令表面与各 docs 间一致
node scripts/validate-artifact-paths.js

# 校验器与评测 runner 的单元测试
# （PowerShell 下，列出文件而非用 glob）
node --test scripts/*-test.js scripts/lib/*-test.js

# Tier 2，触发与路由：正向 prompt 排进 top-k，负向不冲突。
# CI 强制 rank-1 下限，所以照 CI 的方式跑。
node scripts/run-evals.js --min-rank1 95

# Tier 3，行为（按需，消耗 token；--dry-run 仅打印计划）
node scripts/run-evals.js --behavioral <skill-name> --dry-run

# Hook 回归测试，改了 hooks/ 或 skills/using-agent-skills/SKILL.md
# （session-start 脚本内嵌该文件）时必跑
bash hooks/session-start-test.sh
bash hooks/sdd-cache-test.sh
bash hooks/simplify-ignore-test.sh
```

CI 在 Ubuntu 上跑这一组，再在 macOS 与 Windows 上重跑校验器与单元测试，让路径处理与换行符也被覆盖。还有两个 job 需要 Claude Code CLI、不在上述块里：`claude plugin validate .` 检查 marketplace 与插件清单；marketplace 安装（`claude plugin marketplace add ./`，然后 `claude plugin install agent-skills@addy-agent-skills --scope user`）证明插件能从一个检出版本安装。若你动了 `plugin.json`、`.claude-plugin/` 清单、或安装路径读取的任何东西，就跑那两个。当工作流加了步骤时，本块应加同一行；若两者不一致，[工作流文件](../.github/workflows/test-plugin-install.yml) 是真理源。

三个评测层级即使你从不碰评测框架也值得理解，因为 Tier 2 红通常意味着*改你 Skill 的 description*，不是改评测：Tier 2 是路由的词法近似（description 上的词干化 TF-IDF），它盯的两种失败模式是 description 漏掉用户实际说的词、以及 description 过宽盖过了正确 Skill。完整设计、schema 与可信度规则见 [evals/README.md](../evals/README.md)。

每个 PR 前跑相关子集。一个 Tier 1 + Tier 2 + 命令对齐全绿的 PR 是可评审的；否则会先因形式问题被打回，没人看内容。

## 4. 贡献路径

### 路径 1：修或改进既有 Skill（最常见、最佳首份 PR）

1. 改动保持聚焦与最小化；保留 Skill 的结构与语气。
2. 若你改了 frontmatter `description`，预期会有 Tier 2 影响；跑 `node scripts/run-evals.js`，检查该 Skill 的 trigger prompt 仍能排上。
3. 跑 Tier 1 确认 frontmatter 仍合法。

### 路径 2：提案新 Skill（门槛更高，先做预检）

目录已覆盖大部分生命周期，所以举证责任在空白一方。动笔之前，跑 [CONTRIBUTING.md](../CONTRIBUTING.md#before-proposing-a-new-skill) 的预检：检索目录、查看开放 PR（`gh pr list --state open`；近重复集群已经存在）、确认构想符合 [skill-anatomy.md](skill-anatomy.md)，并在 PR 描述里明确证明空白点。若与既有 Skill 重叠，对那个 Skill 做有针对性的编辑，胜过新建目录。

新 Skill 是一组而不是单文件地上线：`skills/<kebab-case-name>/SKILL.md`、一份匹配的 `evals/cases/<name>.json`、以及只有在带可跑辅助脚本时才有的 `scripts/` 目录（参考资料放 `references/`，永远不要放进 Skill 内部）。精确的 frontmatter 规则、章节要解、评测 case 最小项落在 [CONTRIBUTING.md](../CONTRIBUTING.md#structure) 与 [skill-anatomy.md](skill-anatomy.md)；从那里取，而不是本导览，避免两者发散。

一条值得内化而不是去查的点：写 trigger prompt 时用用户实际会用的方式改写；把 description 复制进 prompt 是在钻评测的空子，什么也证明不了。

### 路径 3：文档、References、评测框架

- 文档与 Skill **只接受英文**；翻译不接受，因为会漂移（理由见 [CONTRIBUTING.md](../CONTRIBUTING.md#translations)）。
- 改 `scripts/run-evals.js` 或评测 schema 时，应保持与 skill-creator 的 `evals.json` schema 兼容（行为层逐字采纳；这种兼容是特性，不是巧合）。
- 改 `hooks/` 或 `skills/using-agent-skills/SKILL.md`（session-start 脚本嵌入了该文件）的任何东西都需要跑那三个 hook 回归测试（§3）。改 `scripts/` 的任何东西都需要其单元测试。

## 5. PR 前清单

- [ ] Tier 1 绿：`node scripts/validate-skills.js`，动了引用 `references/` 的 Skill 时再加 `node scripts/validate-reference-links.js`
- [ ] Tier 2 照 CI 方式绿：`node scripts/run-evals.js --min-rank1 95`
- [ ] 动了任何命令目录时，命令对齐与工件路径绿：`node scripts/validate-commands.js` 与 `node scripts/validate-artifact-paths.js`
- [ ] 动了 `scripts/` 时，单元测试绿：`node --test scripts/*-test.js scripts/lib/*-test.js`
- [ ] 动了 `hooks/` 或 `using-agent-skills` 时，hook 测试绿：`bash hooks/session-start-test.sh`、`bash hooks/sdd-cache-test.sh`、`bash hooks/simplify-ignore-test.sh`
- [ ] 新 Skill → 评测 case 文件就位，且 trigger / 行为计数达下限
- [ ] 新 Skill → 在 PR 描述中证明空白；目录与开放 PR 已查
- [ ] 无内容重复；交叉引用而非复制
- [ ] 改动小而聚焦（本仓库自身的 `code-review-and-quality` 改动大小指南同样适用于此处的贡献）

## 6. 建议阅读顺序

1. [README.md](../README.md)：目录与生命周期图（10 分钟）
3. `skills/using-agent-skills/SKILL.md`：从 Agent 视角看路由如何工作
4. 完整读一个成熟的 Skill（例如 `test-driven-development`）：通过例子内化解剖
5. [skill-anatomy.md](skill-anatomy.md)：格式规范，现在带着上下文
6. [evals/README.md](../evals/README.md)：三个层级与 case 格式
7. [CONTRIBUTING.md](../CONTRIBUTING.md) + [AGENTS.md](../AGENTS.md)：规则与仓库作用域的 Agent 配置