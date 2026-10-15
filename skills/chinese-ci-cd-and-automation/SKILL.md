---
name: ci-cd-and-automation
description: 自动化 CI/CD 流水线搭建。设置或修改构建与部署流水线时使用。需要自动化质量门槛、在 CI 配置测试 runner、或建立部署策略时使用。
---

# CI/CD and Automation

## 概述

把质量门槛自动化，那样任何变更都不在没通过测试、Lint、类型检查和构建之前到达生产。CI/CD 是每个 Skill 的强制机制 ， 它捕获人和 Agent 都漏掉的，且在每一次改动上都稳定执行。

**Shift Left：** 尽可能在流水线早期抓问题。在 Lint 阶段抓个 bug 几分钟搞定；同一个 bug 在生产里抓到要几小时。检查项往上游推 ， 静态分析先于测试，测试先于 staging，staging 先于生产。

**Faster is Safer：** 批量更小、发布更频繁反而降低风险。一次部署 3 个变更比一次部署 30 个好排查。频繁的发版让团队对发版流程本身建立信心。

## 何时使用

- 给新项目搭 CI 流水线
- 加或改自动化检查
- 配置部署流水线
- 改动应该触发自动化验证
- 调试 CI 失败

## 质量门槛流水线

每次改动都要过这些门槛才能合并：

```
Pull Request Opened
    │
    ▼
┌─────────────────┐
│   LINT CHECK     │  eslint, prettier
│   ↓ pass         │
│   TYPE CHECK     │  tsc --noEmit
│   ↓ pass         │
│   UNIT TESTS     │  jest/vitest
│   ↓ pass         │
│   BUILD          │  npm run build
│   ↓ pass         │
│   INTEGRATION    │  API/DB 测试
│   ↓ pass         │
│   E2E (optional) │  Playwright/Cypress
│   ↓ pass         │
│   SECURITY AUDIT │  npm audit
│   ↓ pass         │
│   BUNDLE SIZE    │  bundlesize check
└─────────────────┘
    │
    ▼
  准备评审
```

**任何门槛都不能跳过。** Lint 挂了修 Lint ， 不要禁用规则。测试挂了修代码 ， 不要跳过测试。

## GitHub Actions 配置

### 基础 CI 流水线

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Lint
        run: npm run lint

      - name: Type check
        run: npx tsc --noEmit

      - name: Test
        run: npm test -- --coverage

      - name: Build
        run: npm run build

      - name: Security audit
        run: npm audit --audit-level=high
```

### 含数据库集成测试

```yaml
  integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: ci_user
          POSTGRES_PASSWORD: ${{ secrets.CI_DB_PASSWORD }}
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - name: Run migrations
        run: npx prisma migrate deploy
        env:
          DATABASE_URL: postgresql://ci_user:${{ secrets.CI_DB_PASSWORD }}@localhost:5432/testdb
      - name: Integration tests
        run: npm run test:integration
        env:
          DATABASE_URL: postgresql://ci_user:${{ secrets.CI_DB_PASSWORD }}@localhost:5432/testdb
```

> **注意：** 即便 CI 专用测试库，也用 GitHub Secrets 而不是硬编码凭据。这能养成好习惯、避免在其它上下文意外复用测试凭据。

### E2E 测试

```yaml
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'
      - run: npm ci
      - name: Install Playwright
        run: npx playwright install --with-deps chromium
      - name: Build
        run: npm run build
      - name: Run E2E tests
        run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-report
          path: playwright-report/
```

## 把 CI 失败喂回 Agent

CI 与 AI Agent 组合的威力在于反馈回路。CI 挂的时候：

```
CI 挂了
    │
    ▼
复制失败输出
    │
    ▼
喂给 Agent：
"CI 流水线挂了，错误是：
[粘贴具体错误]
修一下并在本地验证后再 push。"
    │
    ▼
Agent 修 → push → CI 再跑
```

**关键模式：**

```
Lint 失败 → Agent 跑 `npm run lint --fix` 并提交
类型错误  → Agent 读错误位置并修类型
测试失败 → Agent 跟 debugging-and-error-recovery Skill
构建错误 → Agent 检查配置和依赖
```

## 部署策略

### Preview 部署

每个 PR 拿一份 preview 部署做手工测试：

```yaml
# 给 PR 部署 preview（Vercel/Netlify 等）
deploy-preview:
  runs-on: ubuntu-latest
  if: github.event_name == 'pull_request'
  steps:
    - uses: actions/checkout@v4
    - name: Deploy preview
      run: npx vercel --token=${{ secrets.VERCEL_TOKEN }}
```

### 特性开关

特性开关把部署与发布解耦。把没做完或带风险的特性部署在开关后面，这样你可以：

- **代码上了但没启用。** 早合到 main，准备好再启用。
- **回退不用重新部署。** 关掉开关而不是 revert 代码。
- **金丝雀新特性。** 1% → 10% → 100% 打开。
- **跑 A/B 测试。** 比较有和没这特性时的行为。

```typescript
// 简单的特性开关模式
if (featureFlags.isEnabled('new-checkout-flow', { userId })) {
  return renderNewCheckout();
}
return renderLegacyCheckout();
```

**开关生命周期：** 创建 → 测试时启用 → Canary → 全量 → 删除开关与死代码。永不消亡的开关就成技术债 ， 创建时设个清理日期。

### 分阶段放量

```
PR 合入 main
    │
    ▼
  Staging 部署（自动）
    │ 手工验收
    ▼
  生产部署（手工触发或 staging 后自动）
    │
    ▼
  监控错误（15 分钟窗口）
    │
    ├── 检测到错误 → 回滚
    └── 干净 → 完成
```

### 回滚方案

每次部署都应该是可逆的：

```yaml
# 手工回滚 workflow
name: Rollback
on:
  workflow_dispatch:
    inputs:
      version:
        description: '回滚到的版本'
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - name: 回滚部署
        run: |
          # 部署指定的上一版本
          npx vercel rollback ${{ inputs.version }}
```

## 环境管理

```
.env.example       → 已提交（给开发者的模板）
.env                → 不提交（本地开发）
.env.test           → 已提交（测试环境，无真实密钥）
CI secrets          → 存放在 GitHub Secrets / vault
生产 secrets        → 存放在部署平台 / vault
```

CI 永远不该有生产 secrets。CI 测试用独立的 secrets。

## 超越 CI 的自动化

### Dependabot / Renovate

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
```

### Build Cop 角色

指定某人对构建保持绿有责任。当构建挂了，Build Cop 的工作是修或回滚 ， 而不是那个弄出问题的提交者。这避免破损的构建不断累积而大家指望别人去修。

### PR 检查

- **必需评审：** 合并前至少 1 人批准
- **必需状态检查：** 合并前 CI 必须通过
- **分支保护：** main 上不能 force-push
- **自动合并：** 所有检查通过且已批准则自动合并

## CI 优化

当流水线超过 10 分钟时，按这个影响顺序套策略：

```
CI 流水线慢了？
├── 缓存依赖
│   └── 用 actions/cache 或 setup-node cache 选项处理 node_modules
├── 并行跑任务
│   └── 把 lint、typecheck、test、build 拆成独立并行任务
├── 只跑改过的
│   └── 用 path filter 跳过无关任务（如纯文档 PR 不跑 e2e）
├── 用 matrix 构建
│   └── 把测试套件分片到多个 runner
├── 优化测试套件
│   └── 把慢测试从关键路径拆出去，改为定时跑
└── 用更大的 runner
    └── GitHub-hosted 更大 runner 或自管用于 CPU 重的构建
```

**例：缓存与并行：**
```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci
      - run: npm run lint

  typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci
      - run: npx tsc --noEmit

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci
      - run: npm test -- --coverage
```

## 常见借口

| Rationalization | Reality |
|---|---|
| "CI 太慢了" | 优化流水线（见上），不要跳过。5 分钟的流水线能省下几小时的 debug。 |
| "这次改动很无聊，跳过 CI" | 无聊的改动也能破构建。无聊改动的 CI 也很快。 |
| "测试 flaky，重新跑一下" | Flaky 测试掩盖真实 bug、浪费大家时间。去修 flakiness。 |
| "我们等会再加 CI" | 没有 CI 的项目会积攒破损的状态。第一天就设好。 |
| "手工测试就够了" | 手工测不可扩展、不可重复。能自动的就自动。 |

## 红旗

- 项目里没有 CI 流水线
- CI 失败被忽略或静默
- 在 CI 里关测试让流水线过
- 没有 staging 验证就直接上生产
- 没有回滚机制
- 密钥存在代码里或 CI 配置里（不在 secret 管理器）
- CI 时间长却没有优化动作

## 验证

搭或修改 CI 之后：

- [ ] 全部质量门槛都在（lint、types、tests、build、audit）
- [ ] 流水线在每个 PR 和 push 到 main 时都跑
- [ ] 失败阻止合并（分支保护已配）
- [ ] CI 结果反馈到开发循环中
- [ ] 密钥放在 secret 管理器，不在代码里
- [ ] 部署有回滚机制
- [ ] 流水线的测试套件部分跑在 10 分钟内
