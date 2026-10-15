<!--
  This document is for developers evaluating the project. It is NOT a skill and
  is not meant to be loaded into an agent's context. It lives in docs/ so it
  stays out of the agent's working set.
-->

# agent-skills 的对比

常被问到的：**agent-skills** 与其他两个最常被提到的"为编程 Agent 准备的 Skill 集合"，**Superpowers**（Jesse Vincent / obra）和 **Matt Pocock's skills**，关系如何。三者都好、共享不少基因、也都值得借鉴。这里诚实地画出它们*形态*上的差别，方便你挑契合自己工作方式的，或者跨三个都借一点。

> **TL;DR** ， 它们为不同瞬间优化。**agent-skills** 把*完整产品生命周期*（Define、Plan、Build、Verify、Review、Ship）组织起来，配评审 persona、抗合理化护栏、以及仓库内的评测框架，验证 Skill 是否真在路由与按设计表现。**Superpowers** 偏向*自主、推理密集型*运行，用子 Agent、严格流水线、工作树隔离。**Matt Pocock's skills** 是一套*锐利、有立场的 Claude Code 工具箱*，蒸馏自一位专家的日常工作流，带有招牌的"grill me"审讯循环。抽象地说没有"最好"；看摆在你面前的工作。

---

## 一眼对比

| | **agent-skills** | **Superpowers** | **Matt Pocock's skills** |
|---|---|---|---|
| **核心理念** | 把资深工程师完整生命周期编码为 Skill | ， |
| **组织原则** | SDLC **阶段**（Define 到 Ship）配 meta-skill 路由器 | 单一纪律循环：brainstorm、plan、execute、review | 一套策划过的、聚焦可组合的命令工具箱 |
| **目录规模** | 25 个 Skill 覆盖整个生命周期 | 约 14 个 Skill，深耕内部 build 循环 | 约 30 个 Skill，分组为 engineering / productivity / in-progress / deprecated |
| **生命周期覆盖** | 广：想法打磨、API 与 UI 设计、安全、性能、CI/CD、可观测性、弃用、ADR、上线 | 深但窄：TDD、调试、规划、评审、Skill 编写 | Define 与 Build 偏重：grilling、PRD、issue、TDD、架构、bug 分诊、知识管理 |
| **入口** | 斜杠命令按阶段 1:1 映射（`/spec` `/plan` `/build` `/test` `/review` `/code-simplify` `/ship`，外加 `/webperf`），配 `/build auto` 整计划模式 | Skill 链式流水线（`brainstorming`、`writing-plans`、`subagent-driven-development`） | 斜杠命令（`/grill-me`、`/tdd`、`/to-prd`、`/diagnosing-bugs`、`/grill-with-docs`） |
| **独特机制** | 每个 Skill 配抗合理化表与红旗；`/ship` 并行评审 **persona**；参考清单；CI 中**三层评测框架** | 子 Agent 驱动开发配任务评审（规范 + 质量）与修正循环；git worktree 隔离；Skill 写 Skill，加压测试 | **grilling** 原语（一次一个问题，走设计树）；基于接缝的 TDD；显式区分用户触发 vs. 模型触发；问题跟踪器集成 |
| **质量度量** | 触发、路由、行为评测对目录跑（仓库内，部分进 CI） | 加压测试方法论是其哲学核心；评测套件现住在独立仓库 | 仓库内未附带 |
| **覆盖的工具** | Claude Code、Cursor、Gemini CLI、Antigravity、OpenCode、Windsurf、Copilot、Kiro、Codex、Command Code，外加 `npx skills` CLI | 覆盖面最广且最活跃迭代的之一：Claude Code、Codex、Cursor、Copilot CLI、OpenCode、Kimi、Factory Droid、Antigravity、Pi | Claude Code 优先，通过 `npx skills add` 分发；其他 Agent 保真度参差 |
| **治理** | 主动评审与合并社区贡献；每个 Skill 上线带评测 | 大体单人作者；社区 PR 积压较多 | 单人作者、自合并、公开演进 |
| **最适合** | 把一项特性从始至终驱动起来，每个阶段都有人工门 | 长跑、自主、推理密集或探索性工作 | 务实、久经考验的日常循环，在需求与 TDD 上最强 |

*（我们刻意不提 star 数与采用数字：博客里引法差异巨大、每周都在变。三者都活跃使用与维护。）*

---

## 三个项目，自各言其用

### Superpowers（obra）

由可组合 Skill 搭建的完整软件开发*方法论*。它赌的是**自主与前置推理**。一次会话跑一条刻意的线性流水线：苏格拉底式 brainstorming，把 spec 写成带日期、只允许一次交接；计划写得能被"一名热情但品味差、缺乏上下文的初级工程师"执行；接着是子 Agent 驱动执行，每个任务由一个全新子 Agent 实现，由任务评审员对"规范符合度"与"代码质量"两者都签字才关闭，对任何被它标记的事触发修正循环，最后由整支分支评审用最强模型做收尾。Git worktree 把并行工作隔离，它的 `writing-skills` Skill 把 TDD 应用到文档本身：没有失败测试的 Skill 不上线。

它的长处是真的：交出去一大块，回头时拿到一个已经被对照规范评审过的结果，并带强护栏防止 Agent 自圆其说绕过流程。代价就是那条纪律的另一面。覆盖窄（它是内部 build 方法论，不是"安全到上线"生命周期）；单条流水线在小改动上可能显得重；近期的方向正是在还那笔账，例如把任务评审合并为一位，让速度翻倍、token 减半。它的社区最响亮的呼声，多 Agent 团队执行，目前还没进盒。

**仓库：** <https://github.com/obra/superpowers>

### Matt Pocock's skills

Matt 把他在用的 Claude Code Skill 开源了出来，这个集合已经长成一套广博、有立场的工具箱，公开组织，并配有可见的 `in-progress/` 与 `deprecated/` 目录，把 Skill 所宣扬的纪律也示范了出来。重头戏不是 TDD，是 **grilling**：一个一次一个问题、沿设计树每条分支按依赖顺序走、为每问附推荐答案、偏好读代码库而不是问、并且除非你确认达成共识否则拒绝继续的可复用审讯循环。`grill-me` 与 `grill-with-docs` 是它的薄包装。TDD Skill 也很有特色、且坦率不羁（"重构不属于循环，它属于代码评审"），并刻意在用户触发与模型触发的 Skill 之间做区分，把 Agent 上下文当作稀缺预算。

长处在于真与锐利：这是一个好工程师怎么出货，不是一群委员会对框架的想象，需求审讯循环也确实优秀。代价：实际是 Claude Code 优先（其他 Agent 暴露稳定性毛边）；没有仓库内评测抓回归；Build 之后生命周期覆盖薄；某些 Skill 耦合到个人安装向导与跟踪器约定。近期工作正在从单会话 Skill 推到经由 issue tracker 的多会话编排（in-progress 的 `wayfinder`）。

**仓库：** <https://github.com/mattpocock/skills> · 相关：<https://github.com/mattpocock/agent-rules-books>

### agent-skills（本项目）

agent-skills 把**完整产品生命周期**组织为 Skill，配一个 meta-skill（`using-agent-skills`）把任务路由到合适那个。每个 Skill 带一张**Common Rationalizations**表（Agent 用于跳过步骤的借口，每个都反驳）和**Red Flags**。斜杠命令按生命周期阶段 1:1 映射；`/build auto` 一次跑完整个已批准的计划；`/ship` 并行扇出评审 **persona**（`code-reviewer`、`security-auditor`、`test-engineer`、`web-performance-auditor`），再把它们合并为 go/no-go。它在每个阶段保留一个人工门、上线七份参考清单含 Definition of Done、并配一条命令在多数主流 Agent 工具上完成单条安装。

更新且当前的差异化点：一个**三层评测框架**住在仓库里。Tier 1 校验结构，Tier 2 校验每个 Skill 的 description 是否带上了用户实际说的词汇、以及没有两份 description 在路由上冲突（确定性、在 CI 里跑），Tier 3 把 Agent 的真实执行轨迹与每 Skill 的期望对照打分。另两个目前都没在仓库里提供这种目录级的度量。诚实的反面代价：agent-skills 没有像 Superpowers 那么强的单一强观点"run"，三者也都还没把跨会话持久记忆做透。

---

## 真刀真枪的对比：Superpowers vs. agent-skills

Om Mishra 跑了一次对照实验，同模型、同仓库、同 prompt，唯一改动 Skill 框架，写了一篇：

**["Superpowers vs Agent-Skills: Faster Shipping, Safer Reasoning"](https://www.linkedin.com/pulse/superpowers-vs-agent-skills-faster-shipping-safer-reasoning-om-mishra-dzakf/)** ， Om Mishra

公平地总结：

- **agent-skills** 更快动代码（约 8 分钟 vs 约 12）、跑**更多验证通过**（7 vs 5，含完整测试套件）。那次更广的验证抓到了一处*在*该特性*之外*的兼容性问题，特性自身的测试漏了。他在那一项任务上给 agent-skills 颁了**验证深度**奖。
- **Superpowers** 投入更多**前置架构推理**，对演进中的生产系统与无既定模式可循的探索性工作，他仍然把它当作日常驱动。
- Token 效率基本相当；两者都各重排了一次。

这是一个开发者的单任务实验，不是基准，但它具体地把核心权衡摆了出来：**广覆盖的纪律验证 vs. 重前置推理**。他自己下的结论也是诚实的：按手上的任务挑工具。

---

## 怎么决定用哪个

一眼表告诉你形态怎样。这是实践中的挑选。

### 从工作形态出发

- **一整项特性，从头到尾？** agent-skills。三者中只有它能从 spec 一路带你过安全、性能、上线，每个阶段都有相等的门，所以没人悄悄跳过评审或飞行前检查。
- **一大块含糊的活，你想交出去走开？** Superpowers。它的流水线与子 Agent 评审就为长时间运行、回头拿到一份已经对照规范评审过的结果而生。
- **快速、聚焦的日常循环，尤其先把需求弄清再动代码？** Matt Pocock's skills。Grilling 循环是三者中最锋利的需求工具，工具箱也不碍手。

### 然后按你真正在意的加权

- **覆盖广度**（安全、性能、CI/CD、可观测性、上线）：agent-skills 显然胜选；其他两个聚焦内部循环。
- **长跑的自主性**：Superpowers，按设计就是。
- **小改动低开销**：Pocock 的工具箱最轻；agent-skills 提供中间档（小改动可以直跳 `/test` 与 `/review`）；Superpowers 流程最重。
- **对 Skill 本身的可靠性有信心**：agent-skills 是三者中唯一带目录级评测在仓库里的，因此 description 或路由回归会在 CI 失败，而不是后来神秘地"为啥这个 Skill 没触发"。
- **需求审讯**：Pocock 的 grilling 是参考实现；agent-skills 的 `interview-me` 思路相近，并新增可选的协作模式。
- **平台覆盖**：agent-skills 与 Superpowers 几乎在所有地方都能跑；Pocock 在 Claude Code 上最舒服。
- **每一步人工门 vs. 全程无人值守**：agent-skills 默认检查点；Superpowers 有意减少中途介入。

### 具体场景

- *"上线一个新端点含鉴权、测试、安全关。"* agent-skills：`/spec` 到 `/ship`，结尾由 security-auditor 与 test-engineer persona 扇出。
- *"隔夜 refactor 一个乱子系统，早上来 review。"* Superpowers：把计划丢给它，让 subagent-driven development 与任务评审员跑。
- *"我有个模糊点子，老让 Agent 猜需求。"* Pocock 的 `grill-me`（或 agent-skills 的 `interview-me`），在动代码前把意图钉死。
- *"修一个明确的 bug，test-first。"* 三者都行；挑你已经装的、最轻的那个。
- *"让一个团队在一个仓库里标准化使用 Agent。"* agent-skills：阶段命令、persona、共享检查清单给团队一份公共词汇，评测让自定义 Skill 不至于跑偏。
- *"自己造 Skill 并知道它们正确触发。"* agent-skills，并借 Superpowers 的加压测试纪律，给那些绝对不能被合理化掉的。

### 个人 vs. 团队

对个人开发者，喜好与节奏赢：挑一款默认契合你既有工作方式的，最低开销选项往往也是用得最多的。对团队，算账方式偏向共享结构与可强制执行的护栏：具名阶段、评审 persona、检查清单、在 CI 失败的评测，正是这些让五个工程师（与他们的 Agent）汇聚到同一标准而不是五个私人工作流。这是 agent-skills 为之设计的场景，不过团队也完全可以在其中采纳 Superpowers 的评审纪律或 Pocock 的 grilling 循环。

### 共同的边界（三者都成立）

三者都还没把**跨会话持久记忆**做透：一个会话里 Agent 学到的东西很少能干净地带到下一个。三者都在围着它转（learnings 文件、交接工件、跟踪器支撑的计划映射）。如果那是你的痛点，请知道你现在站在任一款的发货边界，且眼下预期要自己缝一部分。

---

## 把它们组合

不必唯一选，但组合要谨慎。它们是 Markdown Skill，不是运行时，因此挑*单个* Skill 借来用效果不错：把 Pocock 的 `grill-me`、一个 Superpowers 隔离模式、或一份具体检查清单拉进你的主配置里。

不可行的是把其中两款**同时作为你的活动路由器**。堆叠的 meta-skill 会争抢命令名（同一处定义了两次的 `/tdd`）、在路由逻辑上竞争、引入不同的 TDD 哲学，因此你得到的是不可预测的行为，而不是二者的优点。挑一款框架作主路由器，其余按需点菜。

---

## 出处

- Superpowers：<https://github.com/obra/superpowers>
- Matt Pocock's skills：<https://github.com/mattpocock/skills>
- Om Mishra，*Superpowers vs Agent-Skills*：<https://www.linkedin.com/pulse/superpowers-vs-agent-skills-faster-shipping-safer-reasoning-om-mishra-dzakf/>

*看到本档案里关于其它项目有不准确的地方？开个 issue 或 PR。我们宁愿公平也不要捧场。*