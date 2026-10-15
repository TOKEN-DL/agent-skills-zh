---
name: git-workflow-and-versioning
description: 搭 Git 工作流实践。在做任何代码改动时使用。在提交、分支、处理冲突、把混乱的工作树切分成干净的原子提交、开或评审 PR、推到远端、或需要在多条并行流上组织工作时使用。在发版、选语义版本号、做 tag、写 changelog 时使用。
---

# Git Workflow and Versioning

## 概述

Git 是你的安全网。把 commit 当存档点、把 branch 当沙盒、把 history 当文档。AI Agent 用代码的速度极高，训练有素的版本控制就成了让改动可控、可审、可逆的机制。

## 何时使用

总是。每一次代码改动都经手 git。

## 核心原则

### Trunk-Based Development（推荐）

让 `main` 始终可部署。在存活期短的功能分支上工作，1-3 天内合回 main。生存期长的开发分支是被藏起来的成本 ， 它们分叉、生合并冲突、拖慢集成。DORA 研究反复显示：trunk-based development 与高效能工程团队显著相关。

```
main ──●──●──●──●──●──●──●──●──●──  (始终可部署)
        ╲      ╱  ╲    ╱
         ●──●─╱    ●──╱    ← 短期功能分支（1-3 天）
```

这是推荐的默认。用 gitflow 或长期分支的团队也可以套用这些原则（原子提交、小改动、有描述性的 message）到自己的分支模型 ， 真正重要的是提交纪律，不是具体的分支策略。

- **dev 分支是成本。** 分支每多活一天，就多积累一天的合并风险。
- **release 分支可以接受。** 当你需要在 main 继续推进的同时稳定一个发布时。
- **特性开关 > 长分支。** 比起把未完成的工作压在分支上几周，更愿意把不完整的特性放在开关后面部署。

### 1. 早提、多提

每次成功增量配一条 commit。不要堆积大量未提交的改动。

```
工作模式：
  实现一片 → 测试 → 验证 → 提交 → 下一片

而不是这样：
  全部实现 → 希望没坏 → 一条巨型 commit
```

commit 是存档点。如果下一次改动炸了，你可以立刻回到上次已知好状态。

### 2. 原子提交

每条 commit 做一件逻辑上的事：

```
# 好：每条 commit 自包含
git log --oneline
a1b2c3d 加任务创建接口 + 校验
d4e5f6g 加任务创建表单组件
h7i8j9k 把表单接到 API 并加加载状态
m1n2o3p 加任务创建测试（单元 + 集成）

# 坏：所有事混在一起
git log --oneline
x1y2z3a 加任务特性、修侧栏、升依赖、重构 utils
```

### 3. 有描述性的 message

commit message 解释 *为什么*，不只 *做了什么*：

```
# 好：解释意图
feat: 给注册接口加邮箱校验

阻止非法邮箱格式进数据库。
在路由处理层用 Zod schema 校验，
与 auth.ts 里已有的校验模式保持一致。

# 坏：复述 diff 一眼就能看到的东西
update auth.ts
```

**格式：**
```
<type>: <简短描述>

<可选主体，解释为什么、而不是什么>
```

**类型：**
- `feat` ， 新特性
- `fix` ， Bug 修复
- `refactor` ， 既不修 bug 也不加特性的代码改动
- `test` ， 加或更新测试
- `docs` ， 仅文档
- `chore` ， 工具、依赖、配置

### 4. 把关注点分开

不要把格式改动跟行为改动混在一起。不要把重构跟特性混在一起。每类改动都应该是独立 commit ， 理想上还是独立 PR：

```
# 好：分开关注点
git commit -m "refactor: 把校验逻辑抽到共享工具"
git commit -m "feat: 给注册加手机号校验"

# 坏：混在一起
git commit -m "重构校验 + 加手机号字段"
```

**把重构跟特性分开。** 重构和特性是两种不同的改动 ， 分别提。每条改动更易审、可回滚、也容易在历史里读懂。小清理（重命名变量）可以视评审者偏好并入功能提交。

### 5. 给改动定体量

目标每次提交 / PR 约 100 行。超过 1000 行的要拆。`code-review-and-quality` 给出了拆分策略。

```
~100 行   → 易审、易回滚
~300 行   → 一个逻辑改动可以接受
~1000 行  → 拆成更小的改动
```

## 分支策略

### 特性分支

```
main (始终可部署)
  │
  ├── feature/task-creation    ← 一个特性一条分支
  ├── feature/user-settings    ← 并行工作
  └── fix/duplicate-tasks      ← Bug 修复
```

- 从 `main`（或团队默认分支）拉
- 分支存活期短（1-3 天内合）， 长期分支是被藏起来的成本
- 合入后删分支
- 对未完成特性，更偏好特性开关而不是长期分支

### 分支命名

```
feature/<short-description>   → feature/task-creation
fix/<short-description>       → fix/duplicate-tasks
chore/<short-description>     → chore/update-deps
refactor/<short-description>  → refactor/auth-module
```

## 用 Worktree 工作

并行 AI Agent 工作用 git worktree 同时跑多条分支：

```bash
# 给一个特性分支建一个 worktree
git worktree add ../project-feature-a feature/task-creation
git worktree add ../project-feature-b feature/user-settings

# 每个 worktree 是个独立目录，带自己的分支
# Agent 能并行工作而不互相干扰
ls ../
  project/              ← main 分支
  project-feature-a/    ← task-creation 分支
  project-feature-b/    ← user-settings 分支

# 完成后合并并清理
git worktree remove ../project-feature-a
```

好处：
- 多个 Agent 能同时在不同特性上工作
- 不需要切分支（每个目录有自己的分支）
- 如果一条实验失败，删 worktree ， 没东西丢
- 改动在显式合并前保持隔离

## 存档点模式

```
Agent 开始工作
    │
    ├── 做改动
    │   ├── 测试通过？ → 提交 → 继续
    │   └── 测试挂？ → 回滚到上次提交 → 排查
    │
    ├── 做另一个改动
    │   ├── 测试通过？ → 提交 → 继续
    │   └── 测试挂？ → 回滚到上次提交 → 排查
    │
    └── 特性完成 → 全部 commit 构成一段干净历史
```

这个模式让你一次最多只丢一段增量。如果 Agent 跑偏了，`git reset --hard HEAD` 直接把你带回到上次成功状态。

## 改动摘要

做完任何修改，提供一份结构化摘要。这让评审更易、文档化范围纪律、暴露未打算的改动：

```
CHANGES MADE:
- src/routes/tasks.ts: 给 POST 接口加校验中间件
- src/lib/validation.ts: 用 Zod 加 TaskCreateSchema

THINGS I DIDN'T TOUCH (intentionally):
- src/routes/auth.ts: 同样有校验缺口，但超出本次范围
- src/middleware/error.ts: 错误格式可以改进（另开任务）

POTENTIAL CONCERNS:
- Zod schema 严格 ， 拒绝额外字段，请确认这是想要的
- 加了 zod 作为依赖（72KB gzipped）， 已经在 package.json 里了
```

这个模式早早就抓到错误假设，给评审者一张清晰的"这次改了什么"地图。"没动"那段尤其重要 ， 显示出你守住了范围、没做不请自来的装修。

## 提交前卫生

每次提交之前：

```bash
# 1. 看即将提交什么
git diff --staged

# 2. 确保没有密钥
git diff --staged | grep -i "password\|secret\|api_key\|token"

# 3. 跑测试
npm test

# 4. 跑 lint
npm run lint

# 5. 跑类型检查
npx tsc --noEmit
```

用 git hook 自动：

```json
// package.json (用 lint-staged + husky)
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{md,json}": ["prettier --write"]
  }
}
```

## 处理生成的文件

- **提交生成文件**仅当项目期望它们时（如 `package-lock.json`、Prisma migrations）
- **不要提交**构建产物（`dist/`、`.next/`）、环境文件（`.env`）、IDE 配置（`.vscode/settings.json` 除非共享）
- **有 `.gitignore`** 覆盖：`node_modules/`、`dist/`、`.env`、`.env.local`、`*.pem`

## 用 Git 做调试

```bash
# 找到引入 bug 的 commit
git bisect start
git bisect bad HEAD
git bisect good <known-good-commit>
# Git checkout 中间 commit；在每条上跑测试来缩小范围

# 看最近改了什么
git log --oneline -20
git diff HEAD~5..HEAD -- src/

# 看一行最后一次被谁改
git blame src/services/task.ts

# 按关键词搜 commit message
git log --grep="validation" --oneline
```

## 发版与版本号

commit 是你*自己*追踪改动的方式；**版本号**是你*消费者*追踪它的方式。一旦有什么别的东西依赖你的代码 ， 另一个团队、一个发出去的包、一个部署好的客户端 ， "main 上最新"就不再能回答"我在跑什么，升级是不是安全？"版本号与 changelog 是回答它的契约。

### 语义版本号

对带消费者的东西，用 `MAJOR.MINOR.PATCH` 编号，让数字带语义：

```
  MAJOR  破坏性变更 ， 消费者必须改代码才能升级
  MINOR  新功能，向后兼容 ， 升级安全
  PATCH  Bug 修复，向后兼容 ， 升级安全
```

数字是承诺，所以代码要对得上。一个"patch"但改了消费者依赖的行为，是穿着伪装的主要变更（Hyrum 定律 ， 见 `api-and-interface-design` Skill）。拿不准这次变更是不是破坏，就当它是；意外的主要变更比破坏的消费者便宜。

### 给发布做 tag，让 tag 当真源

发布是历史上的一个不可变节点，不是会动的分支。打 tag 让它能被重现：

```bash
git tag -a v1.4.0 -m "Release 1.4.0"
git push origin v1.4.0
```

从 tag 衍生版本号，不要在散落的文件里手工编辑它，让工件、tag 和 changelog 不可能打架。

### 写一份给"人"看的 changelog

changelog 不是 `git log`。它是为消费者精心编辑的、对"改了什么、我在不在乎？"的答案 ， 按 `Added / Changed / Fixed / Deprecated / Removed / Security` 分组，最新在上，每条围绕用户影响而非内部机制。

```markdown
## [1.4.0] - 2025-06-12
### Added
- 通过 CSV 批量导入任务
### Fixed
- 周期任务截止日期的时区漂移
### Deprecated
- `GET /v1/tasks/all` ， 用分页版的 `GET /v1/tasks`（2.0 里移除）
```

在做出变更的同一份改动里写这一条 ， 趁着影响还新鲜，不是发版时从 commit 考古重建。破坏性变更要有迁移说明和弃用窗口（按 `deprecation-and-migration` Skill）；实际打 release 包是 `shipping-and-launch` Skill 的活 ， 这一节是喂给它的版本契约。

## 常见借口

| Rationalization | Reality |
|---|---|
| "特性完了再提交" | 一条巨型 commit 不可能审、debug、回滚。把每一片都提交。 |
| "message 不重要" | message 就是文档。未来的你（与未来的 Agent）要能看懂改了什么、为什么。 |
| "我等会 squash 一下" | squash 抹掉了开发的叙事。一开始就用干净的增量提交。 |
| "分支就是 overhead" | 短期分支是免费的、能防止冲突工作撞车。长期分支才是问题 ， 1-3 天内合。 |
| "拆分之后再做" | 大改动更难看、更危险部署、更难回滚。提交前先拆，不是之后。 |
| "我不需要 .gitignore" | 直到带着生产密钥的 `.env` 被提交了。立刻设。 |
| "这是个小修，bump patch" | 看消费者能观测什么。他们依赖的行为变化，无论是 diff 多小，就是 major。 |
| "changelog 就是 commit log" | commit 是给你用的；changelog 是给消费者的，按影响精心编辑。从 raw commit 生成的 changelog 把重要的事埋了。 |
| "我们发版时再写 changelog" | 到那时候影响是从记忆里重建的，一半已经没了。在改动里把这条写了。 |

## 红旗

- 大量未提交的改动在累积
- 提交 message 像 "fix"、"update"、"misc"
- 格式改动混行为改动
- 项目里没有 `.gitignore`
- 把 `node_modules/`、`.env` 或构建产物提交了
- 长期分支与 main 显著分叉
- 向共享分支 force-push
- 破坏性变更以 minor 或 patch 版本号发布
- 发布无 tag，或版本号被手工编辑与 tag 脱钩
- 一个用户面版本没有 changelog 条目，或 changelog 只是堆 commit message

## 验证

每条 commit：

- [ ] Commit 做了一件逻辑事
- [ ] message 解释为什么、符合 type 约定
- [ ] 提交前测试通过
- [ ] diff 里没有密钥
- [ ] 没有"纯格式"改动混行为改动
- [ ] `.gitignore` 覆盖了标准的忽略项

每个发布（任何带消费者的东西）：

- [ ] 版本 bump 与变更匹配：破坏 → major，加性 → minor，修复 → patch
- [ ] 给发布打了 tag，版本号从 tag 衍生，不是手工编辑后与 tag 脱钩
- [ ] changelog 有按影响分组、为人写的本版本条目
