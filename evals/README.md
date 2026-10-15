# Skill 评测（Skill Evals）

本仓库如何衡量它的 Skill 是否真的工作：是否在应该**触发**的时候触发、是否彼此**保持区分**、以及是否如各自承诺的那样**改变 Agent 的行为**。

## 前人经验（与我们的取舍）

对 `SKILL.md` Skill 的评测，社区尚无统一标准，但有两种做法领跑：

- **Anthropic 的 skill-creator v2** 定义了每 Skill 一份 `evals.json`（prompt + `expectations[]`，由转录评分）以及对 description 与示例 prompt 的触发准确度测试。我们采用它的 [`evals.json` schema](https://github.com/anthropics/skills/tree/main/skills/skill-creator) 用于行为层，并增加一个可选的 `kind` 字段以选择被评分的工件。
- **Superpowers**（obra）用 bash + `claude -p` + prompt fixtures 测试 Skill。我们的行为 runner 也遵循同一无头 `claude` 模式，评分细则取自 `expectations[]`。

两者都没有给出针对多 Skill *目录* 的**确定性、CI 安全**检查，每个 Skill 的 description 是否带上了用户实际会说的词汇、以及两份 description 之间是否冲突。这就是下面要介绍的 Tier 2，也是本仓库的增量。

## 三个层级

| 层级 | 校验内容 | 运行方式 | 成本 |
|---|---|---|---|
| 1. 结构（Structural） | Frontmatter、命名、必需章节、版式与行预算、`references/` 链接、命令对齐、工件路径、人工版本 | CI（`validate-skills.js`、`validate-reference-links.js`、`validate-commands.js`、`validate-artifact-paths.js`、`validate-versions.js`） | 免费 |
| 2. 触发与路由（Trigger & Routing） | 正向 prompt 把对应 Skill 排在 top-k；负向 prompt 不应如此；没有两份 description 接近冲突 | CI（`run-evals.js --min-rank1 95`） | 免费 |
| 3. 行为（Behavioral） | 遵循该 Skill 的 Agent 满足它的 `expectations[]` | 按需（`run-evals.js --behavioral`） | Token |

Tier 2 是路由的**词法近似**（在 description 上做词干化 TF-IDF）。它不能评判语义，那是 Tier 3 的工作，但它能逮住真实触发 bug 中占主导的两种失败模式：description 漏掉用户实际说的词（假阴性）、以及 description 过宽盖过正确 Skill（假阳性）。Tier 2 失败通常意味着*修 description*，而不是修评测。

## 运行

```bash
# Tier 2 ， 确定性，在 CI 中运行
node scripts/run-evals.js
node scripts/run-evals.js --min-rank1 95  # 强制当前路由底线

# Tier 3 ， 行为，把每个评测过一遍无头 claude 再评分
node scripts/run-evals.js --behavioral test-driven-development            # 消耗 token
node scripts/run-evals.js --behavioral test-driven-development --dry-run  # 只打印计划
```

Tier 3 支持两种行为工件 kind。`execution` 是默认：每个评测跑在一次性 git 仓库里，`files[]` 中的真实项目输入由 `evals/fixtures/` 物化出来并提交作为基线，评分器对完整的 `--output-format stream-json --verbose` 执行轨迹评分，含工具调用。`dialogue` 留给"交付物即对话本身"的 Skill：它不需要 fixture，评分器在 assistant 的对话回合上评判，不要求文件编辑或命令。把 `dialogue` 视为人工审核的豁免，而不是执行 Skill 的一般逃逸通道。

执行器使用显式权限模式（`--permission-mode acceptEdits` 加上预批准的工具列表）运行，使 execution 评测能真正编辑文件、运行命令、检视 diff、做 commit，而不是被拒绝后只做叙述。轨迹在评分器 prompt 中被圈为不可信数据，并通过 stdin 传给评分器（它们可能上 MB，argv 会撞到 OS 参数长度上限）；执行器与评分器调用都带超时；评分器输出在写入 `evals/results/`（已 gitignore）之前会被校验为 JSON，形状符合 skill-creator 的 `grading.json`。纪律类 Skill 也包含针对时间压力、沉没成本与权威压力的压力 case；这些 case 校验当 prompt 试图说服跳过流程时，工作流依然成立。

## 插件评测（Claude Code）

`claude plugin eval`（Claude Code 2.1.269 或更高）会一次加载整个插件、全部 Skill，跑 `evals/plugin/` 下的 cases，并对每个 case 在带/不带插件两种状态下打分。它补足 Tier 2 而非取代：Tier 2 检查 description 是否带上了用户实际说的词汇；这里检查 Claude Code 自身的路由器在自然 prompt 上是否挑中该 Skill，以及它给回复带来了什么（`Δ` 列）。

```bash
claude plugin eval . --no-publish                                          # 每个 case、两种手臂、各 3 次
claude plugin eval . --case code-review-fires --runs 1 --ablation none     # 单 case 的低成本迭代
```

一个 case 是一个目录，内含 `prompt.md`（frontmatter：turn 与 tool 上限；body：prompt，原样发送）与 `graders/` 下每个文件一份评分器。报告落在 `evals/plugin/results/<timestamp>/`（已 gitignore）。每次运行、每个 `llm` 评分器都是你账号上的真实模型调用，因此按需运行、永不进 CI，与 Tier 3 一致。

两条规则让它可信。manifest 中的 `experimental.evals` key 把扫描范围限定在 `evals/plugin/`，因此 `evals/` 下别处的 fixture 永远不会被当作 case 读。而 `tool_used: Skill` 评分器只表示 Skill 是否被触发；在双臂运行时它会从分数里排除，因此要把它与对回复本身的评分器配对使用。`code-review-fires` 展示了该模式：触发指示器、一个看 Skill 严重度分类法（Critical、Required、Nit、Optional、Consider、FYI）作为标题或标签的自由 `regex` 评分器、以及一个针对预埋 off-by-one 的 `llm` 判定。`code-review-stays-quiet` 是精度检查：review Skill 不应在测试优先（test-first）请求上触发，其 `Δ` 应为 0；另一个不计分的指示器记录 test-driven-development 是否被触发。

首批运行教给我们的（Claude Code 2.1.278，claude-opus-5）。Skill 调用是随机的：review Skill 在该 case 用的措辞下，7 次里有 5 次触发；test-driven-development 在测试优先 prompt 下，7 次里有 2 次触发。因此要读 `Δ` 与报告本身，而不是依赖出口码，或者按文档里的 CI 示例那样传 `--threshold 0.8`。在一次运行中 review Skill 没触发，回复却仍把它的严重度分类法作为标题使用，这在九次基线运行中从未出现；加载的 Skill description 可能在没调用的情况下也塑造了回复。措辞重要，description 是杠杆。用原 description 时，"Review this diff before I merge it"、"Can you do a proper code review of this change before I merge it?" 与 "Review this pull request for me before it goes into main" 几乎不会触发 Skill：模型靠自身就把 diff 评审掉了，做得不错但少了 verdict、严重度层级与五维度判定。光描述 Skill 的产出没用，光在 description 中提 diff / pull request 也没用；真正起作用的是说"即使 diff 是内联粘贴的"也适用本 Skill，因为模型本来就把这种情况当成"我自己就能搞定"。在钉死模型的 claude-opus-5-5 上，这一句话把四种 review 措辞的触发从 7/27 提到 21/27，九个相邻 must-not-fire 的 prompt 两种情况下都是 0/27，最朴素的措辞从 0/9 提到 3/9，变好了，没解决。`code-review-stays-quiet-on-commit-message` 守护那一句话：内联粘贴的 diff 加上提交信息请求不应触发 review Skill。同一天的两条实操规则：报数字时一定钉死 `--model` 并记录 Claude Code 版本，因为 CLI 会自更新、默认模型在一早一晚之间换了两次；给 case 留 10 以上的 turn 预算，因为模型可能通过 slash 命令（`agent-skills:review`，然后……（行截断）

同样的杠杆用到 test-driven-development（钉死 claude-opus-5-5，每个 prompt 跑六次）。原 description 下，"Add unit tests for this function"、"Fix this bug ... here is the function"、"Change this function so that size can never exceed 100" 都以 0/6 触发 Skill，而流程化的 "Write a failing test for this bug before we fix it" 以 6/6 触发：模型直接写测试或写修复。光补尾巴还不够；真正起作用的是把这三种请求都点出来，并说"即使代码内联粘贴、请求里压根没提测试"也适用本 Skill。这句话把四个 prompt 从 6/24 提到 19/24（加测试 6/6，修 bug 6/6，改行为 1/6），在所有试过的变体下把八个相邻 must-not-fire 的 prompt 维持在 0/24。改一行行为是底线：轨迹显示一次无工具调用的单轮编辑，模型不会为它考虑流程。`tdd-fires-on-pasted-bug` 与 `tdd-stays-quiet-on-debug` 就是守护那句话的对子：前者检查 Skill 在粘贴的 bug 上触发且回复在修复前先用失败测试复现；后者检查一个带 diff 的"哪儿坏了"问题不会把 TDD Skill 拉进来。第一个里，Skill 每次都触发，但回复把失败测试放在修复之前的，带插件的 4 次里有 2 次、不带的 3 次里 0 次：判定器有意严格，因为那个顺序就是 Skill 带来的，而漏掉的（先修、后给一个测试并加注说"它在旧代码上本来会失败"）属于 Skill 在没有 Shell 的会话里的"遵循"差距，不是路由差距。

## 评测 case 格式

每个 Skill 一个文件：`evals/cases/<skill-name>.json`。

```json
{
  "skill_name": "test-driven-development",
  "trigger": {
    "positive": [
      { "prompt": "Write a failing test for this bug before fixing it", "top_k": 3 }
    ],
    "negative": [
      { "prompt": "Update the architecture diagram in the docs", "owner": "documentation-and-adrs" }
    ]
  },
  "evals": [
    {
      "id": 1,
      "kind": "execution",
      "prompt": "Finance filed the reconciliation bug written up in BUG.md. Fix it.",
      "expected_output": "A failing reproduction test for the lost-cent case, a fix preserving both README invariants (exact sum, earliest-shares fairness), the fairness invariant covered by its own test, full suite passing",
      "files": [
        "test-driven-development"
      ],
      "expectations": [
        "A test reproducing the lost-cent case from BUG.md is added and shown failing before src/split.js is modified",
        "The final implementation satisfies the full README fairness invariant (leftover cents go to the earliest shares): splitCents(10000, 3) returns [3334, 3333, 3333] as BUG.md expects and splitCents(100, 7) returns [15, 15, 14, 14, 14, 14, 14] as the README example shows; dumping the whole remainder on a single share would violate both",
        "The fairness invariant from the README has its own test case in the suite on an input with remainder of at least 2 (such as splitCents(100, 7)), where dumping the whole remainder on one share would fail it, beyond the reported lost-cent case",
        "The full suite is run with the repository's own command after the fix"
      ]
    }
  ]
}
```

- `evals[]` 使用 skill-creator 的核心 schema（`id`、`prompt`、`expected_output`、可选 `files[]`、`expectations[]`）加上本仓库一个可选的 `kind`。`kind` 必须是 `execution` 或 `dialogue`，默认 `execution` 以保持兼容。Execution 评测需要非空 `files[]`；路径相对 `evals/fixtures/`，可指向文件或项目目录。Dialogue 评测可以省略 `files[]`，因为转录本身就是工件。Expectations 是评分器对照相关工件校验的可验证陈述，是行为，不是措辞。
- `trigger` 是本仓库的扩展。`positive` prompt 是理应路由到本 Skill 的真实用户问法（默认 `top_k` 为 3；为该 Skill 的招牌问法收紧到 1）。`negative` prompt 属于*别的* Skill；本 Skill 不能对它们排第一。能指认时把那个 Skill 写在 `owner` 里：runner 会断言该 owner **排在本 Skill 之先**，把 negative 变成真正的"成对路由"测试，而不是那种"prompt 什么也没命中所以空过"的检查。

**写好 trigger prompt：** 用用户真实会说的方式改写；不要照抄 description（那是钻评测的空子）。若一个真实 prompt 因为 description 缺它的词汇而排不上，那是真实发现，去改 description。

## 新增 Skill

每个 Skill 上线都带一份评测文件。新增 `skills/<name>/` 时，同时新增 `evals/cases/<name>.json`，至少 3 个 positive trigger、2 个 negative trigger、1 个 behavioral eval。Execution 评测必须由 `evals/fixtures/<name>/` 撑场；仅在该 Skill 的交付物*确为*对话本身时，才用 `kind: "dialogue"`。缺 case 文件、case 数量不足、未知的 kind、无效的 fixture 路径、缺失必需 fixture 都是 CI 错误。

## 需关注的指标

Tier 2 运行会打印 **trigger rank-1 rate**（被排在第一位而非仅 top-k 的正向 prompt 占比）。CI 跑时传 `--min-rank1 95`，在已入库 100% 基线下方留出有用的余量，让一次不相关的 description 改动不至于立刻让 CI 红。路由改进就抬高底线；永远不要为让回归过线而压低它。数字掉下来说明 description 在互相靠近。冲突检查在 description 两两相似度 ≥75% 时报错，≥50% 时警告。当这些评测显示 description 词汇空白（参见 [#351](https://github.com/addyosmani/agent-skills/issues/351) 看原始例子），改 description 而不是 prompt。