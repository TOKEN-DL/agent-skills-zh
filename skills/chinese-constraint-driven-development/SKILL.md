---
name: constraint-driven-development
description: 把项目的质量底线变成书面契约，阻止 Agent 偷偷降低它。通过访谈了解哪些维度重要，提供合理的默认阈值，把一切都记在 CONSTRAINTS.md 里，并且在 diff 里盯"被弱化的底线" ， 新的 @ts-ignore 或 eslint-disable 抑制、被跳过或删除的测试、被去掉的断言、未实现的 stub、被下调的阈值。在没有写下来的质量底线时、用户说"配约束"或"立我们的标准"时、用户希望他在意的维度（可访问性、Web 性能、覆盖率）被设为强约束时、Agent 一直静默关闭检查或跳过测试以求"绿"、需要一个覆盖率或性能阈值但不知道选什么数字、或 Agent 写得比谁读得都快时使用。
---

# Constraint-Driven Development

## 概述

本包里的其它 Skill 描述"好"长什么样。`code-review-and-quality` 给你五维度。`test-driven-development` 给你一个循环。`security-and-hardening` 给你一份威胁清单。这些全活在散文里 ， Agent 会读、可能遵守、也可能不，会话结束就跟着没了。

这个 Skill 产出的是另一种东西：**本项目**质量底线的书面记录，带数字、跨会话、并可以被机械检查。

理由重要。你写代码的时候，读它就知道写得好不好。Agent 一个下午写的比你这周读的多，评判就得搬出你的脑子、搬到跑在循环边上的检查里。那些检查得存在、用你实际选过的数字、且足够贴近工作，Agent 才会去修自己的产出。

Spec-driven development 决定要做什么。Test-driven development 证明它能跑。Constraint-driven development 定义"够格上线"是什么意思 ， 比有人在 PR 里争辩要早。

## 何时使用

在以下情况下应用：

- 启动一个项目或一个重要特性，还没有写下来的质量底线
- 用户要求"set up constraints"、"add quality gates"、"define our standards" 或 "stop the agent shipping junk"
- Agent 在产出没人逐行读的量
- CI 里有检查，但没人能说哪些阻止合并、哪些只是装饰
- 覆盖率、性能、可访问性的数字每个 PR 都重新吵一遍
- 即将跑 `/build auto` 或别的自主循环，而它和 main 之间只有 Agent 自己写的测试套件

**何时不用：**
- 项目已经有 `CONSTRAINTS.md` 且用户没在改它 ， 读它、跟着它走
- 一次性脚本、尖峰、可丢的原型
- 用户此时想要一次代码评审（`code-review-and-quality`）或搭一条 CI 流水线（`ci-cd-and-automation`）
- 还没跑出产品 / 市场契合的代码，两周预期寿命 ， 下面的底线还是值得做，剩下的不值得

## 加载约束

访谈需要一个在场的用户。**不要在非交互式上下文**（CI、`/loop`、autonomous run）跑它。如果在这些环境里、约束缺失，套下面的"底线"，注明你套了，把剩下的 flag 给人类。

## 流程

### 步骤 1：先察觉，再问

不要问你能读到的。问第一个问题前，收集：

| 内容 | 去哪看 |
|------|---------------|
| 语言与栈 | `package.json`、`pyproject.toml`、`go.mod`、`Cargo.toml` |
| Test runner | dev dependencies、`test` 脚本、已有测试文件 |
| 现有 linter | `eslint.config.*`、`biome.json`、`.ruff.toml` |
| 当前的覆盖率 | `coverage/` 输出，或先跑一次套件 |
| CI | `.github/workflows/`、`.gitlab-ci.yml` |
| Agent harness | `.claude/`、`.codex/`、`AGENTS.md` |

用两行报告你看到的，然后只问剩下的。

### 步骤 2：四个问题，每个都有默认

跟着 `interview-me` 的一次一问一次的纪律，但有一个变化：每个问题都有默认，所以"我不知道"是个完整答案且仍然产出一个能用的配置。

```
Q1：除底线之外，你希望哪些被强制？
    (a) 新代码的测试覆盖率
    (b) 安全扫描
    (c) 性能预算
    (d) 可访问性
    (e) 架构边界
猜：(a) 和 (b) ， 你已经有测试 runner，且你在处理用户输入。
拿不准时默认：(a) 和 (b)。
说说每项的代价：(c) 和 (d) 要能跑的 URL，(e) 要写一条规则文件。
```

```
Q2：当 Agent 在任务中途挂了，是阻止还是警告？
猜：阻止。你在无人值守地跑 Agent，没人读的警告等于不警告。
拿不准时默认：底线就阻止，其余前两周先警告。
```

```
Q3：你心里有目标数字，还是按今天的实测值守住？
猜：实测。大部分团队没数字，凭空捏的数字会被忽略。
拿不准时默认：实测并守住。见下方"Ratchets"。
```

```
Q4：Agent 交活前你能忍受的最慢的检查是多少？
猜：约 90 秒。再久你就会停跑。
拿不准时默认：任务结束 90 秒，CI 不限。
```

止于四个。十二个问题的接入流程产出一份没人懂的配置，以及后悔开头的用户。

### 步骤 3：写 CONSTRAINTS.md

仓库根一份文件。任意 harness 上的任意 Agent 都能读，对它的改动也会进该进 review。

```markdown
# Constraints

Last reviewed: 2026-08-08 by @addy

## Floor (always enforced, no setup required)
- 不准新的抑制注释：`@ts-ignore`、`eslint-disable`、`# noqa`、`# type: ignore`
- 不准未实现的 stub：`throw new Error("Not implemented")`、空的 `catch {}`
- 不准在没有理由（写在 commit message 里）的情况下跳过或删除测试
- 不准密钥进源码
- 本文件不被降低以让某次改动过

## Enforced with numbers

| Dimension | Rule | Checked by | Runs at |
|-----------|------|-----------|---------|
| Types | 零类型错误 | `tsc --noEmit` | 每次编辑 |
| Lint | 我们配置下零错误 | `biome check` | 每次编辑 |
| Secrets | 源码里没有密钥 | `gitleaks detect --redact` | 每次编辑 |
| Coverage | 改动行 ≥ 80% 覆盖 | `vitest run --coverage` + git diff | 任务结束，CI |
| Security: code | 无 high 发现 | `semgrep scan --config p/default` | CI |
| Security: deps | 没有 high 或更严重的 | `osv-scanner scan source -r .` | CI |
| Accessibility | 零 critical 或 serious | `axe $PREVIEW_URL --tags wcag2a,wcag2aa,wcag21aa` | preview 部署 |
| Performance | LCP ≤ 2500ms，CLS ≤ 0.1 | `lighthouse $PREVIEW_URL --output=json` | preview 部署 |

每行都点明出裁决的命令。一个有数字但这列为空的维度，只是愿望，不是约束。

## Measured, not yet enforced

| Metric | Today | Direction |
|--------|-------|-----------|
| 项目覆盖率 | 62.4% | 不得下降 |
| Bundle size (main) | 184 kB | 不得增长 |

## Exceptions

| ID | Rule | Path | Reason | Owner | Expires |
|----|------|------|--------|-------|---------|
| W1 | `no-explicit-any` | `src/legacy/**` | 重写在 ENG-441 跟踪 | @addy | 2026-11-01 |
```

然后给 `AGENTS.md` 和 `CLAUDE.md` 各加一行：`Read CONSTRAINTS.md before writing code. Do not weaken it to make a change pass.`

### 步骤 4：为每个维度装上它需要的

每个维度的代价就是装一个。不要给用户留一个数字但没机制；如果现有的"事实标准"已经在做这件事就不要自己造一个 checker ， 这些工具在这里被列出，是因为它们的规则格式与阈值正是生态里其它东西所对齐的，所以团队已有的配置都能继续 work。

| 维度 | 工具 | 安装 | 运行 | 卡控点 |
|-----------|------|---------|-----|---------|
| Types (TS) | tsc | 已经有了 | `tsc --noEmit` | 任一错误 |
| Types (Python) | mypy | `pip install mypy` | `mypy .` | 任一错误 |
| Lint | 你已有的配置 | 已经有了 | `eslint .` / `biome check` / `ruff check` | 任一错误 |
| Coverage (JS) | 你的测试 runner | 已经有了 | `vitest run --coverage`（或 `jest --coverage`） | 改动行的覆盖率 |
| Coverage (Python) | pytest-cov | `pip install pytest-cov` | `pytest --cov --cov-report=lcov` | 同上 |
| Security: code | Semgrep | `pipx install semgrep` | `semgrep scan --config p/default --config p/owasp-top-ten` | 任一 high 发现 |
| Security: secrets | gitleaks | `brew install gitleaks` | `gitleaks detect --redact --no-banner` | 任一发现 |
| Security: dependencies | osv-scanner | `brew install osv-scanner` | `osv-scanner scan source -r .` | high 或更严重 |
| Performance: page | Lighthouse | `npm i -D lighthouse` | `lighthouse $URL --output=json --quiet` | LCP、CLS、performance score |
| Performance: bundle | size-limit | `npm i -D size-limit` | `size-limit --json` | 每入口的字节预算 |
| Accessibility | axe-core | `npm i -D @axe-core/cli` | `axe $URL --tags wcag2a,wcag2aa,wcag21aa` | 零 critical 或 serious |
| Architecture | dependency-cruiser | `npm i -D dependency-cruiser` | `depcruise --validate src` | 任一违反 |
| Assertion quality | Stryker | `npm i -D @stryker-mutator/core` | `stryker run --mutate <changed files>` | 变异分数 |

如果跳过，下面 5 件事会咬你：

1. **`--redact` 在 gitleaks 上不是可选。** 不加它，匹配到的密钥会落到 agent 的对话记录里 ， 密钥就是这么进日志、摘要、commit message 的。报告规则和位置，不要报告值。
2. **Lighthouse 和 axe 需要 URL。** 它们只能对着跑起来的应用工作，所以它们属于运行时阶段、对着 preview 部署或先启动一个本地 server。如果项目没有 URL 可打 ， 一个 CLI、一个库、一个桌面 app ， 直说、把这个维度拿掉，而不是造一个跑不起来的检查。
3. **把开销大的限定到 diff。** `stryker run --mutate` 对整个仓库要几小时，会被关掉；只对这次改动碰到的文件，不到一分钟。Semgrep 同理，要传路径清单。
4. **覆盖率不必跑两遍套件。** 读取你的套件已经写出来的 lcov，跟 `git diff` 取交。再跑一遍套件拿个数字，是让人最快讨厌这件事的方式。
5. **Semgrep 注册表的规则免费跑；再分发前查一下许可证。** `opengrep` 是一个 drop-in 分叉，同一套规则格式与 JSON 输出，如果你的法务在意这个。

把每一条加到项目自己的脚本，这样不需要 Agent 也能复现：

```json
{
  "scripts": {
    "check:fast": "tsc --noEmit && eslint . && gitleaks detect --redact --no-banner",
    "check:task": "npm run check:fast && vitest run --coverage",
    "check:full": "npm run check:task && semgrep scan --config p/default && osv-scanner scan source -r ."
  }
}
```

这个映射比工具本身更重要。`check:fast` 是改完就跑，`check:task` 是 Agent 觉得做完了时跑，`check:full` 在 CI 里跑。

命令现在活在两处 ， `CONSTRAINTS.md` 里的 `Checked by` 列和这些脚本。`CONSTRAINTS.md` 是规范源：它把"为什么"和每条命令放一起，且会进入 review。脚本是必须与之对齐的便利封装，不是第二个真源；如果它们漂移，文件赢。

### 步骤 5：把它挂到生命周期上

最大的错误就是到处都跑。一项让 Agent 卡住的检查会被关掉，被人关掉的门槛比没门槛更糟，因为底线看起来还在。

| 阶段 | 命令 | 跑什么 | 预算 |
|-------|---------|-----------|---------|
| BUILD | `/build` | 类型、lint、密钥、底线 | < 5s，只改过的文件 |
| VERIFY | `/test` | 相关测试，改动行的覆盖率 | < 90s |
| REVIEW | `/review` | 全部，加下面的守护 | 几分钟 |
| SHIP | `/ship` | 方向性检查，无回归 | CI |

让这件事可控的两条规则：

1. **限定到 diff。** 检查这次改动碰到的行，而不是整个仓库。改动行的覆盖率是 Agent 能推动的数字，项目覆盖率是它继承下来的。
2. **代价决定摆放。** 任何超过几秒的都搬出编辑循环。整个仓库的变异测试要几小时；对这次改动碰到的文件不到一分钟，这是"会跑"和"不会跑"的差别。

### 步骤 6：守护底线本身

会有人指出，如果 Agent 自己写代码、自己写检查，那检查什么也证明不了。说对一半，值得用工程绕开。

Agent 不编巧妙的漏洞。它们撞到红的检查，挑去绿最便宜的路。在评审时盯 diff 里这五种动作：

1. **阈值改了。** 预算被下调、严重级被降、某项检查被从快阶段拿掉。把 `CONSTRAINTS.md` 与其在分支点的状态对比。
2. **测试被放宽。** 加了 `.skip`、删了测试文件、从留在仓库的测试里抽走了断言。
3. **checker 被静音。** 新增 `@ts-ignore` 或 `eslint-disable`。四个抑制值得特别盯，因为它们关掉你正依赖的检查：`istanbul ignore` 把代码从覆盖率丢出去而不是测它；`Stryker disable` 藏住幸存的 mutant；`nosemgrep` 和 `gitleaks:allow` 对安全发现也这样做。
4. **活没干完。** 抛的 stub、空 `catch` 把失败变沉默、`TODO` 站在本来该有实现的位置。
5. **出现了例外。** Exceptions 表里出现了一行，谁也没讨论过。

这一切所需的工具只有 `git diff`。提升底线应该悄无声息；松动底线应该响亮。

与带数字的维度不同，底线没有它自己的事实标准工具。被要求去强制它的 Agent 倾向于从零写一个 checker，两个 Agent 就写两份不同的。在这个 Skill 的 [references/floor-guard.md](references/floor-guard.md) 里附这五条检查的一份参考实现（diff 限定的、退出 `0`/`1`/`2`、模式可按生态调）。去适配它而不是重造它，原因与每个维度都点一个事实标准工具一样：这样机制跨运行、跨栈保持一致。

**不是所有检查都同样循环。** 用一个问题来排：Agent 能靠"写不管用的代码"让这个检查过吗？

- **外部** ， axe-core 编码了 WCAG、`osv-scanner` 读漏洞库、Lighthouse 测真实浏览器。Agent 没法和这些吵。
- **项目** ， 你的 lint 规则、你的层次边界。由人拥有这个文件。
- **套件** ， 你自己的测试。最有用，也确实是最循环的。

一条全是第三种的底线，不如有一条带"外界意见"的。检查至少一条外部约束在列。

### 步骤 7：当你心里没数字时的 ratchet

一个 62% 覆盖率的代码库直接设 80%，就是无限红的 build，外加一支学会忽视红的 build 的团队。

另一种选择不要决定：记下当前位置，然后拒绝变差。今天的数字与一个方向写进"Measured, not yet enforced"表。每次检查都对照记录值比照，不比照心愿。数字变好就更新；数字掉了就是发现。

这也回答了一个对训练的合理反对。模型被奖励"过测试"，你能在几秒内评估。架构腐烂几个月才显现，从来不会进入权重。ratchet 就是写进"build 能看见的地方"的、丢失的惩罚。

## 合理的默认

用户没意见时用这些。它们的设计就是"第一天大多数代码库都能满足"。

| 约束 | 默认 | 为什么是它 |
|------------|---------|-----------------|
| 改动行的覆盖率 | ≥ 80% | 高到强制一条测试，低到允许一行配置 |
| 项目覆盖率 | 今天的值，不得下降 | 采用不需要争辩 |
| 变异分数（如启用） | 起步 ≥ 60% | 之前没变过异的套件典型水平；80% 算成熟 |
| 依赖漏洞 | 不出现 high 或更严重 | 低于它通常是噪声 |
| LCP | ≤ 2500 ms | Core Web Vitals "good" 阈值 |
| CLS | ≤ 0.1 | 同上 |
| 可访问性 | 零 critical 或 serious 的 axe 违规 | 中等与轻微常常有争议 |
| 例外有效期 | 90 天 | 足够排修复计划，又记得住 |
| Ratchet 容差 | 0.5% | 在不相关文件动起来时吸收漂移 |

数字和理由一起给。一个没理由的阈值总会被下次撞到它的人删掉。

## 升级路径

约束在三个力度上工作。从第一档起：

1. **只写下来。** `CONSTRAINTS.md` 存在且 Agent 会读。零代价、抓诚实的错误、依赖 Agent 守规矩。
2. **脚本化。** 一条 `npm run check`（或 `make check`）跑快检查，挂进 agent 的 post-edit 钩子和 CI。确定性、无新依赖。
3. **工具有背。** 专门的 runner 处理 diff 限定、预算、ratchet、守护检查。当配置大过一段 shell 脚本时走这条。floor-guard 参考在 [references/floor-guard.md](references/floor-guard.md) 里，是守护检查那一半的起点。

大多数项目到 2 就行。当你维护的 check-shell 多于约三十行时，进到 3。

**第一次跑可以只跑底线。** floor guard 只看 diff、不需要安装，所以可以在第一天就强制底线，再随你装每个工具逐步加带数字的维度，而不是先给每个 checker 立项再让第一次提交有保护。需要全局安装的安全工具（gitleaks、osv-scanner）也可以 CI-only，先把笔记本弄干净；每个维度的运行位置在 `Runs at` 列里声明。

## 常见借口

| Rationalization | Reality |
|---|---|
| "代码定型后再加约束" | 代码在它"能乱"的窗口里定型 |
| "测试就是约束" | 你写的测试证明你同意你自己；它对"新代码覆盖率"、"依赖风险"、"bundle 增长"一个都没说 |
| "我们做不到 80% 覆盖率" | 那就别 80%。记今天的位置并守住 |
| "这会让 Agent 慢下来" | 只是你把慢检查放进了快循环。这是摆放问题，不是反约束的论据 |
| "我会记住我们的标准" | Agent 不会，而 Agent 在写大多数代码 |
| "约束会卡住我们发布" | 一份带负责人和日期的例外就能解锁。删约束让每个人永远解锁 |

## 红旗

注意到下面任何一条就停下来重新评估：

- 访谈问了超过 4 个问题，或产出一份用户解释不了的配置
- 设了代码库今天就过不了的预算，又没计划怎么达到
- 一个维度被写进 CONSTRAINTS.md 但背后没工具
- 当有事实标准的 checker 时，却手写了一个
- 全部约束都由项目自己的测试套件做裁判，没有任何外部意见
- `CONSTRAINTS.md` 在与那个失败的特性同一条提交里被改了
- 一份例外没有负责人，或到期日比一年还远
- Agent 提的不是修代码，而是调阈值
- 慢检查被放进了编辑循环，有人开始用 `--no-verify` 跳过
- `CONSTRAINTS.md` 自打写出来就没人再开过

## 验证

这个 Skill 被正确应用时：

- [ ] `CONSTRAINTS.md` 存在，里面的每个数字都给了理由
- [ ] 底线被强制，且能在当前代码库上不改任何东西就过
- [ ] 用户选的每个维度都装好了工具，且有今天能跑的命令
- [ ] 每条约束说明跑在哪里，快阶段保持在几秒内
- [ ] 至少有一条约束是外部的（不由本项目自己的测试评判）
- [ ] 仅测量的指标记下今天值和方向
- [ ] 例外有负责人和到期日
- [ ] `AGENTS.md` 或 `CLAUDE.md` 指向本文件
- [ ] 在当前分支上跑一次，产出的失败没有用户不同意的

## 参见

- `interview-me` ， 这次访谈借用的一次一问一次的纪律
- `code-review-and-quality` ， 怎么评；这个 Skill 决定评什么
- `ci-cd-and-automation` ， 这些约束要跑的流水线
- `test-driven-development` ， 覆盖率和变异约束测的套件
- `security-and-hardening` ， 安全维度应当包含什么
- `performance-optimization` ， 性能数字从哪来
