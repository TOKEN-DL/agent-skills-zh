# 采用指南：新项目 vs. 既有代码库

如何铺开 agent-skills，取决于代码库的生命阶段。一个全新项目可以从第一个提交就采用完整生命周期。一个有数年底蕴的代码库则需要一条渐进路径，尊重既有之物、它的约定、未被记录的决策、以及那些你不太想盲目触碰的测试空白地带。

本指南覆盖两条路径。安装机制见 [getting-started.md](getting-started.md) 与按工具的配置指南。每个 Skill 的作用见 [README 的 Skill 目录](../README.md#all-25-skills)。

---

## 你走哪条路？

| 信号                     | 全新项目（Greenfield）          | 既有代码库（Brownfield）                        |
| -------------------------- | ------------------------------- | -------------------------------------------- |
| 代码库年龄                | 几天到几周                   | 数月到数年                              |
| 测试覆盖                  | 你从第一天就控制着它 | 不均匀；有些区域未测                  |
| 约定                       | 边做边定义 | 既定，往往未被文档化              |
| 团队习惯                | 正在形成 | 根深蒂固（无论好坏）             |
| 一次坏 Agent 改动的风险 | 波及面小                | 可能弄坏没人记得怎么修的东西 |
| 采用策略                | **立即上完整生命周期** | **增量、验证优先**          |

若你介于两者之间（一个已经上生产的年轻项目），从 brownfield 路径开始、加速走，最终会汇聚到同一终态。

---

## 路径 A ｜ Greenfield：从第一天起的完整生命周期

新项目是最佳场景：没有要保留的旧包袱，因此 Skill 的质量关卡几乎不费力、且从第一次提交起就开始复利。

### Day 0 ｜ 安装并接入

1. 装本包（`npx skills add addyosmani/agent-skills`，或你工具的原生集成，参见 [getting-started.md](getting-started.md)）。
2. 若宿主没有原生 Skill 路由器，加载 `using-agent-skills`（meta-skill），让 Agent 能自己把工作路由到合适的 Skill。在有原生路由的宿主上，安装单独的 Skill，不要把 meta-skill 作为 always-on 上下文预加载。
3. 加一份简短的项目规则文件（`CLAUDE.md`、`.cursorrules` 等），写明你的技术栈、命令与边界；`context-engineering` 描述了那里该写什么。

### Day 0 ｜ 先定义再构建

按顺序为项目的第一个真正功能跑一遍生命周期：

```
/spec   →  SPEC.md            （spec-driven-development）
/plan   →  tasks/plan.md      （planning-and-task-breakdown）
/build  →  一次切一片        （incremental-implementation + test-driven-development）
/review →  每次合并前         （code-review-and-quality）
/ship   →  上线时             （shipping-and-launch）
```

`/build auto` 适合 greenfield：一次批准计划，每个任务仍按 TDD 跑、各自独立提交。Spec 与 plan 工件（`SPEC.md`、`tasks/`）是活的文档，在工作进行时放进版本控制。若一项特性跨多个会话，那些文件也是交接物，参见 [跨会话协作](getting-started.md#working-across-sessions)。

### 从一开始就当作 always-on

- **test-driven-development** ， 覆盖债最早最好避开。
- **git-workflow-and-versioning** ， 原子提交与约 100 行的改动是习惯，不是后置改造。
- **security-and-hardening** ， 鉴权、输入校验、机密处理是结构性的；之后补是迁移工程。
- **documentation-and-adrs** ， 最初的架构决策正好是两年后没人记得*为什么*的那批。现在写 ADR 能避免路径 B 里描述的 brownfield 考古。

### 随项目成长而加入

| 何时                                | 加载                                                          |
| ----------------------------------- | ------------------------------------------------------------- |
| 第一份公开 API 或模块边界 | `api-and-interface-design`                                    |
| 第一份 UI 工作                       | `frontend-ui-engineering`（+ `browser-testing-with-devtools`） |
| 第一条 CI 流水线                   | `ci-cd-and-automation`                                        |
| 第一次生产部署             | `observability-and-instrumentation`、`shipping-and-launch`    |
| 出现性能要求             | 性能优化（`performance-optimization`）                        |

### Greenfield 反模式

- **"它只是个原型"式跳过 `/spec`。** 原型会变成产品。Spec 是你为这个代码库写过的最便宜的工件。
- **把全部 25 个 Skill 一次塞进每个会话。** 那浪费上下文、稀释重要的那些。按阶段加载；让宿主原生路由器或 `using-agent-skills` 来路由，但别两个都上。
- **"有东西要观察时再说"地推迟可观测性。** 一边构建一边打点；事后补结构化日志是你自己造出来的 Path B 问题。

---

## 路径 B ｜ Brownfield：增量、验证优先

在既有代码库里，风险谱系反过来：危险不是"做错事"，而是"改了没人完整说清过行为的东西"。因此采用顺序从**读、保护**代码库的 Skill 开始，然后才轮到**改**它的 Skill。

### 阶段 1 ｜ 上下文与只读 Skill

目标：Agent 在动手改之前理解代码库。

1. **`context-engineering` 优先。** 写项目规则文件描述真实的约定，那些写在代码里的，不是 wiki 里的。包括构建 / 测试命令、目录含义、已知雷区（"别碰 `legacy/billing`，它没测试、还有三处已知绕路"）。
2. **`code-review-and-quality` 用于评审进入的改动。** 评审是零风险且立刻有价值：五维度评审及其严重度标签（区分哪些拦合并 Critical / Required、哪些不拦）适用于任何 PR，不管代码库处于什么状态。
3. **`debugging-and-error-recovery` 用于你本来就要修的 bug。** 五步分诊（复现 → 定位 → 缩小 → 修复 → 防御）在陌生代码里特别亮眼，而"防御"那一步正好开始搭建你本来没有的回归套件。
4. **`doubt-driven-development` 作为安全网。** 旧代码恰好就是这个 Skill 瞄准的"陌生代码、改错代价高"的场景。对 Agent 关于旧系统如何工作的主张做对抗式的新上下文评审，能在它们变成提交前抓到自信的幻觉。

### 阶段 2 ｜ 改之前先有测试

目标：Agent 要动的每个区域先有安全网。

- **`test-driven-development`，有选择地应用。** 不要瞄准全局覆盖；瞄准*计划改动处*的覆盖。对未测的旧行为，写 characterization tests，把代码当下做的事（不管对错）钉死的测试，再做任何修改。Beyonce Rule 适用：若 Agent 喜欢一个行为到依赖它的程度，那就该给它加个测试。
- **`code-simplification` 用于最严重的热点。**Chesterton's Fence 是操作原理：该 Skill 强迫 Agent 在删代码前先弄清*为什么*存在。保持行为的简化加上 characterization tests，是让旧代码变得可改的最低风险路径。
- **`git-workflow-and-versioning` 处处用。** 小原子提交在 brownfield 里*更重要*：当一处对旧代码的改动引发微妙的破坏时，一个约 100 行的提交是可二分定位的；一个 2000 行的"现代化"提交不能。

### 阶段 3 ｜ 新工作跑完整生命周期

目标：双速采用，旧代码留在 Phase 1–2 治理下；**新特性得到 greenfield 待遇**。

- 在旧代码库里加新特性？`/spec → /plan → /build → /review`。Spec 的边界段就是声明特性可以 / 不可以碰哪些旧表面。
- **`api-and-interface-design` 在接缝处使用。** 当新代码必须与旧代码对话时，先契约式设计边界。Hyrum's Law 在有数年底蕴的代码库里不是理论，有人依赖每个可观察行为，包括 bug。
- **`security-and-hardening` 作为审计，再作为门禁。** 先在现有攻击面上跑一遍（鉴权、输入处理、依赖，光是依赖审计通常就值回票价），把发现归档，再对新改动强制执行。

### 阶段 4 ｜ 偿还、弃用、可观测

- **`deprecation-and-migration`** 是 brownfield 的头牌 Skill：把代码当负债、强建议 vs. 可选建议的弃用、僵尸代码清理，给你一条让旧表面缩小的纪律之路，不只是包住它。
- **`observability-and-instrumentation`** 沿你真正调试的路径事后补：先在首要事件源上加结构化日志与 RED 指标。
- **`performance-optimization`** 在回归成为问题时上，它的"先度量"规则能避开经典旧代码陷阱，优化那些从来不是瓶颈的代码。

### Brownfield 反模式

- **"大爆炸"式采用。** 在第一天把完整生命周期全堆到一个旧代码库上，会产出为已存在代码写的 spec，以及没有安全网的 refactor。要排顺序。
- **让 Agent refactor 未测代码。** 没有 characterization tests，就别 refactor。这是 brownfield 采用中最昂贵的捷径。
- **以"代码就是文档"为由跳过 `/context-engineering`。** Agent 会从它偶然读到的最差文件里推断约定。把真实的告诉它。
- **把旧系统的行为默认视为错的。** Chesterton's Fence：那个奇怪的 retry 循环可能是承重的。先理解，再改。
- **什么闸都不棘轮化。** 采用应当让质量单调变好：每一阶段加一道不再撤的门。若一个月后你叫不出现在多了哪些以前没有的强制项，那铺开就停滞了。

---

## 两条路径汇聚

两条都以同一终态结尾：新建工作走 `/spec → /plan → /build → /review → /ship`，always-on 的 TDD 与 git 纪律，合并前的评审门，按阶段而非批量加载 Skill。Greenfield 几天就到；brownfield 一个季度到，差异恰好是（旧代码库从来没有过的）上下文、特性化测试、边界。

|                        | Greenfield                     | Brownfield                               |
| ---------------------- | ------------------------------ | ---------------------------------------- |
| 首个加载的 Skill     | 原生路由器，或 `using-agent-skills` + `/spec` | `context-engineering`                    |
| 第一份交付的价值  | 第一项功能规格化、测试先行 | 零风险评审与更安全的 bug 修复    |
| TDD 姿态            | 从第一个提交起全面 | 有选择：计划改的地方有测试 |
| 重构规则          | 很少（几乎没什么可重构） | 先写 characterization tests，始终如此     |
| 最危险的反模式 | 跳过 spec                | refactor 未测代码                |
| 到完整生命周期的时间 | 第一天                        | 约一个季度，期间双速运行       |