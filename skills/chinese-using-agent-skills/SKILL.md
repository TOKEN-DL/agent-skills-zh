---
name: using-agent-skills
description: 发现并调用 Agent Skill。在启动会话、或需要为当前工作决定该用哪个 Skill / 哪个工作流时使用。这是统领其它 Skill 如何被发现和调用的元 Skill。
---

# 使用 Agent Skills

## 概述

Agent Skills 是一组按开发阶段组织的工程工作流 Skill。每个 Skill 都封装了资深工程师遵循的具体流程。这个元 Skill 帮你为当前任务发现并套用恰当的 Skill。

## Skill 发现

任务进来时，先识别开发阶段，再套用对应的 Skill：

```
任务进入
    │
    ├── 还不清楚想要什么？ ──────→ interview-me
    ├── 有粗略概念，想要更多变体？ → idea-refine
    ├── 新项目 / 新功能 / 重大改动？ ──→ spec-driven-development
    ├── 没有写下来的质量底线？ ──→ constraint-driven-development
    ├── 有规格，需要拆任务？ ──────→ planning-and-task-breakdown
    ├── 在写代码？ ────────────→ incremental-implementation
    │   ├── UI 工作？ ─────────────────→ frontend-ui-engineering
    │   ├── API 工作？ ────────────────→ api-and-interface-design
    │   ├── 需要更清晰的上下文？ ─────→ context-engineering
    │   ├── 需要文档化核验的代码？ ───→ source-driven-development
    │   └── 风险高 / 不熟悉的代码？ ──→ doubt-driven-development
    ├── 在写 / 跑测试？ ────────→ test-driven-development
    │   └── 涉及浏览器？ ───────────→ browser-testing-with-devtools
    ├── 出了问题？ ──────────────→ debugging-and-error-recovery
    ├── 在评审代码？ ───────────────→ code-review-and-quality
    │   ├── 太复杂？ ─────────────→ code-simplification
    │   ├── 安全顾虑？ ───────→ security-and-hardening
    │   └── 性能顾虑？ ────→ performance-optimization
    ├── 在提交 / 分支？ ─────────→ git-workflow-and-versioning
    ├── 在搭 CI/CD 流水线？ ──────────→ ci-cd-and-automation
    ├── 在做弃用 / 迁移？ ────────→ deprecation-and-migration
    ├── 在写文档 / ADR？ ───────────→ documentation-and-adrs
    ├── 在加日志 / 指标 / 告警？ ───→ observability-and-instrumentation
    └── 准备上线 / 部署？ ─────────→ shipping-and-launch
```

## 核心操作行为

这些行为任何时候都适用，跨所有 Skill 都是不可商量的。

### 1. 显式列出假设

在动手实现任何非平凡改动前，先把你的假设明列出来：

```
我正在做的假设：
1. [关于需求的假设]
2. [关于架构的假设]
3. [关于范围的假设]
→ 现在就纠正我，或者我按这些继续。
```

不要默默地把含糊的需求补全。最常见的失败模式是带着错误假设一路冲、不核对。把不确定性提前说出来，比来回返工便宜。

### 2. 主动管理"我不懂"

遇到前后不一致、需求冲突、规格不清时：

1. **停下来。** 不要靠猜继续。
2. 把具体的疑问点明。
3. 摆出权衡或问出澄清问题。
4. 等有了结论再继续。

**不好的：** 默默挑一种解读然后赌它对。
**好的：** "规格里看到的是 X，但现存代码是 Y。哪个优先？"

### 3. 该反对就反对

你不是应声虫。当一个方案有明显问题时：

- 直接点出问题
- 说出具体的坏处（能量化就量化 ， "这会多 ~200ms 延迟" 而非 "这可能慢一些"）
- 提出替代方案
- 用户如果掌握全部信息依然要采纳，那就接受

谄媚是一种失效模式。"当然！" 然后照着糟糕的方案实现，对谁都没帮助。诚实的反对比假意的认同更有价值。

### 4. 强制保持简单

你的默认倾向就是过度复杂。要主动抵挡。

在收尾任何实现前，问一下：
- 能写得更短吗？
- 这些抽象是否挣回了它们带来的复杂度？
- 资深工程师看了会不会说 "为什么不直接……"？

写 1000 行而 100 行就够了，那就是失败。宁可乏味、显而易见的方案。取巧是要付代价的。

### 5. 守住范围

只动要你动的地方。

不要：
- 删你看不懂的注释
- 顺手"清理"与你任务正交无关的代码
- 把顺手改动嫁接到旁边系统
- 没有明确授权就删看起来没在用的代码
- 因为"似乎有用"就加规格之外的特性

你是外科精度，不是主动上门装修。

### 6. 验证，不假设

每个 Skill 都包含一个验证步骤。验证不通过，任务就没完成。"看着对" 永远不够 ， 必须有证据（测试通过、构建输出、运行时数据）。

每个 Skill 自带的验证是局部检查。跨项目通用的"完成"门槛（无论当前激活哪个 Skill 都要过得去）是 Definition of Done：测试通过、无回归、行为在运行时被验证、文档已更新。参见 `../../references/definition-of-done.md`。它补足任务级的验收标准，而不是替代。

## 失败模式

这些"看着像产量"但是制造问题的细微错误：

1. 不核对就做错误假设
2. 自己困惑却不管 ， 迷路还硬往前推
3. 看到不一致没说
4. 面对非显然选择时不给权衡
5. 对有问题的方案谄媚（一口一个 "当然！"）
6. 让代码、API 过度复杂
7. 改动与任务正交无关的代码或注释
8. 删自己没完全理解的东西
9. 因为"显而易见"就不写规格直接动手
10. 因为"看着对"就跳过验证

## Skill 使用规则

1. **开工前检查是否有适用的 Skill。** Skill 封装的流程能预防常见错误。
2. **Skill 是工作流，不是建议。** 按步骤顺序做。不要跳过验证。
3. **多个 Skill 可以叠加。** 一个功能实现可能会依次走 `idea-refine` → `spec-driven-development` → `planning-and-task-breakdown` → `incremental-implementation` → `test-driven-development` → `code-review-and-quality` → `code-simplification` → `shipping-and-launch`。
4. **拿不准就从规格开始。** 任务非平凡且没有规格的话，从 `spec-driven-development` 走起。

## 生命周期顺序

对一个完整的功能，典型的 Skill 顺序是：

```
1.  interview-me                → 抽出用户真正想要的东西
2.  idea-refine                 → 把模糊想法打磨
3.  spec-driven-development     → 定义要构建什么
4.  planning-and-task-breakdown → 拆成可验证的小块
5.  context-engineering         → 加载对的上下文
6.  source-driven-development   → 对照官方文档做核验
7.  incremental-implementation  → 一片一片构建
8.  observability-and-instrumentation → 边写边埋点（与 7-9 并行，不是之后再加）
9.  doubt-driven-development    → 飞行中对每次非平凡决策做对抗式复核
10. test-driven-development     → 证明每一片能跑
11. code-review-and-quality     → 合并前评审
12. code-simplification         → 在保留行为前提下削减不必要复杂度
13. git-workflow-and-versioning → 干净的提交历史
14. documentation-and-adrs      · 记录决策
15. deprecation-and-migration   · 必要时让旧系统退场并把用户安全迁走
16. shipping-and-launch         · 安全地上线
```

不是每个任务都要走完所有 Skill。一个 bug 修复可能只需要：`debugging-and-error-recovery` → `test-driven-development` → `code-review-and-quality`。

## 速查

| 阶段 | Skill | 一句话总结 |
|-------|-------|-----------------|
| Define | interview-me | 在任何计划、规格、代码出现前，先把用户真正想要的东西挖出来 |
| Define | idea-refine | 用结构化的发散 / 收敛思考打磨想法 |
| Define | spec-driven-development | 代码之前的"要做什么"和验收标准 |
| Plan | planning-and-task-breakdown | 拆成小而可验证的任务 |
| Build | incremental-implementation | 薄薄一片，每片测完再扩 |
| Build | source-driven-development | 实现前对照官方文档核验 |
| Build | doubt-driven-development | 对每次非平凡决策做对抗式的 fresh-context 复核 |
| Build | context-engineering | 在对的时间喂给对的上下文 |
| Build | frontend-ui-engineering | 兼顾可访问性的高质量 UI |
| Build | api-and-interface-design | 接口稳定、契约清晰 |
| Verify | test-driven-development | 先写失败的测试，再让它通过 |
| Verify | browser-testing-with-devtools | Chrome DevTools MCP 提供运行时验证 |
| Verify | debugging-and-error-recovery | 复现 → 定位 → 修 → 防 |
| Review | code-review-and-quality | 五维度评审配质量门槛 |
| Review | code-simplification | 在保留行为的前提下削减不必要复杂度 |
| Review | security-and-hardening | OWASP 预防、输入校验、最小权限 |
| Review | performance-optimization | 先测量，只优化真正有影响的 |
| Ship | git-workflow-and-versioning | 原子提交、干净历史 |
| Ship | ci-cd-and-automation | 每次改动走自动化质量门槛 |
| Ship | deprecation-and-migration | 安全地让旧系统退场、把用户迁走 |
| Ship | documentation-and-adrs | 文档化"为什么"，而不只是"做了什么" |
| Ship | observability-and-instrumentation | 结构化日志、RED 指标、链路追踪、基于症状的告警 |
| Ship | shipping-and-launch | 上线前清单、监控、回滚预案 |
