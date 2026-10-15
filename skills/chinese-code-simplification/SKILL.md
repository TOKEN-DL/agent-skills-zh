---
name: code-simplification
description: 让代码更清晰。重构代码以提升清晰度而不改变行为时使用。代码能跑但比应有的难读、难维护、难扩展时使用。评审已经积累不必要复杂度的代码时使用。
---

# Code Simplification

> 灵感来自 [Claude Code Simplifier plugin](https://github.com/anthropics/claude-plugins-official/blob/main/plugins/code-simplifier/agents/code-simplifier.md)。在这里被改造成模型无关、流程驱动的 Skill，适配任何 AI 编程 Agent。

## 概述

通过减少复杂度同时精确保留行为来简化代码。目标不是行数更少 ， 是让代码更易读、更易懂、更易改、更易 debug。每次简化必须过一道简单测试："新团队成员是否比读原版更快理解？"

## 何时使用

- 一个特性跑通且测试通过，但实现感觉比必需的更重
- 代码评审时有人标出可读性或复杂度问题
- 遇到深度嵌套逻辑、长函数、命名不清
- 重构带时间压力写出来的代码
- 把散落在多个文件的关联逻辑整合到一起
- 合入引入了重复或不一致的代码后

**不用：**
- 代码本来就干净且易读 ， 别为了简化而简化
- 还没弄懂代码在做什么 ， 先理解再简化
- 代码是性能关键的，"更简单"的版本明显慢
- 你准备完全重写模块 ， 简化要丢的代码是浪费

## 五条原则

### 1. 精确保留行为

别改代码做什么 ， 只改它怎么表达。输入、输出、副作用、错误行为、边界情况都必须保持不变。如果你拿不准某个简化是否保留行为，就别做。

```
每次改动前问：
→ 这对每个输入产生同样的输出吗？
→ 这保留了同样的错误行为吗？
→ 这保留了同样的副作用与顺序吗？
→ 已有测试不改动就仍然通过吗？
```

### 2. 跟项目约定

简化是让代码更像代码库的样子，不是把外部偏好强加进来。简化前：

```
1. 读 CLAUDE.md / 项目约定
2. 看相邻代码怎么处理类似模式
3. 对齐项目风格：
   - import 顺序与模块系统
   - 函数声明风格
   - 命名约定
   - 错误处理模式
   - 类型注解深度
```

破坏一致性的简化不是简化 ， 是 churn。

### 3. 清晰胜于取巧

显式代码好过紧凑代码，如果紧凑版本要花点脑力才能 parse：

```typescript
// 不清晰：密集的三元链
const label = isNew ? 'New' : isUpdated ? 'Updated' : isArchived ? 'Archived' : 'Active';

// 清晰：可读的映射
function getStatusLabel(item: Item): string {
  if (item.isNew) return 'New';
  if (item.isUpdated) return 'Updated';
  if (item.isArchived) return 'Archived';
  return 'Active';
}
```

```typescript
// 不清晰：链式 reduce 嵌内联逻辑
const result = items.reduce((acc, item) => ({
  ...acc,
  [item.id]: { ...acc[item.id], count: (acc[item.id]?.count ?? 0) + 1 }
}), {});

// 清晰：命名中间步骤
const countById = new Map<string, number>();
for (const item of items) {
  countById.set(item.id, (countById.get(item.id) ?? 0) + 1);
}
```

### 4. 维持平衡

简化有一种失效模式：过度简化。这些陷阱要盯住：

- **过激内联** ， 删一个给概念命名的 helper 让调用点更难看
- **合并不相关的逻辑** ， 两个简单函数合成一个复杂函数，反而更不简单
- **删"不必要"抽象** ， 有的抽象是为可扩展或可测试而存在的，不是因为复杂
- **优化行数** ， 行数少不是目标；易理解才是

### 5. 限定在你改动的范围

默认只简化最近改过的代码。避免顺手重构不相关的代码，除非显式要求扩大范围。不收敛的简化在 diff 里制造噪声、增加不必要的回归风险。

## 简化流程

### 步骤 1：动笔前先理解（Chesterton's Fence）

在改或删任何东西前，先弄懂它为什么存在。这就是 Chesterton's Fence：你在路上看到一道栅栏却不知道为什么，先别拆。先理解原因，再判断原因是否还成立。

```
简化前先答：
- 这段代码的职责是什么？
- 谁调用它？它调用了谁？
- 边界情况与错误路径是什么？
- 有没有测试定义预期行为？
- 为什么它可能被写成这样？（性能？平台约束？历史原因？）
- 看 git blame：这段代码的原始上下文是什么？
```

回答不了，就还没准备好简化。先多读上下文。

### 步骤 2：识别简化机会

扫这些模式 ， 每条都是具体信号，不是模糊气味：

**结构性复杂度：**

| 模式 | 信号 | 简化 |
|---------|--------|----------------|
| 深度嵌套（3+ 层） | 控制流难跟 | 把条件抽成 guard 子句或 helper |
| 长函数（50+ 行） | 多职责 | 拆成有描述性名字的专注函数 |
| 嵌套三元 | 要脑内栈才能 parse | 换成 if/else 链、switch、或查找表 |
| 布尔参数 flag | `doThing(true, false, true)` | 换成 options 对象或拆成多个函数 |
| 重复条件 | 同样的 `if` 检查出现在多处 | 抽成命名良好的谓词函数 |

**命名与可读性：**

| 模式 | 信号 | 简化 |
|---------|--------|----------------|
| 通用命名 | `data`、`result`、`temp`、`val`、`item` | 改成描述内容：`userProfile`、`validationErrors` |
| 缩写 | `usr`、`cfg`、`btn`、`evt` | 用全拼除非缩写通用（`id`、`url`、`api`） |
| 误导命名 | 名为 `get` 的函数同时也改状态 | 改成反映真实行为的名字 |
| 解释"做什么"的注释 | `// increment counter` 在 `count++` 上 | 删注释 ， 代码够了 |
| 解释"为什么"的注释 | `// Retry because the API is flaky under load` | 保留 ， 表达代码说不出的意图 |

**冗余：**

| 模式 | 信号 | 简化 |
|---------|--------|----------------|
| 重复逻辑 | 同样 5+ 行散在多处 | 抽成共享函数 |
| 死代码 | 不可达分支、不用的变量、注释掉的块 | 删（确认真死后再删） |
| 不必要的抽象 | 没加价值的 wrapper | 内联 wrapper，直接调底层 |
| 过度工程模式 | 工厂套工厂、单策略的策略模式 | 换成简单直接的做法 |
| 冗余类型断言 | 强转给已经推导出的类型 | 删断言 |

### 步骤 3：渐进地改动

一次做一个简化。每次跑测试。**重构提交与特性 / 修复提交分开。** 一次 PR 既重构又加特性，那是两个 PR ， 拆开。

```
每个简化：
1. 做改动
2. 跑测试套件
3. 如果测试通过 → 提交（或继续下一个）
4. 如果测试挂了 → 回滚重新想
```

避免把多个简化批量进一次未测试的改动。一旦挂了，你得知道是哪个简化弄挂的。

**500 行规则：** 如果重构要动 500+ 行，投资自动化（codemods、sed 脚本、AST 转换）而不是手工改。手工改到这个体量既容易错、又审得累。

### 步骤 4：验证结果

所有简化做完，退一步看整个：

```
前后对比：
- 简化版是不是真的更易理解？
- 是否引入与代码库不一致的新模式？
- diff 干不干净、好不好审？
- 同事会批准这次改动吗？
```

如果"简化"版更难理解或难审，回滚。不是每次简化尝试都会成功。

## 语言特定指引

### TypeScript / JavaScript

```typescript
// 简化：没必要的 async 包装
// 改前
async function getUser(id: string): Promise<User> {
  return await userService.findById(id);
}
// 改后
function getUser(id: string): Promise<User> {
  return userService.findById(id);
}

// 简化：啰嗦的条件赋值
// 改前
let displayName: string;
if (user.nickname) {
  displayName = user.nickname;
} else {
  displayName = user.fullName;
}
// 改后
const displayName = user.nickname || user.fullName;

// 简化：手工拼数组
// 改前
const activeUsers: User[] = [];
for (const user of users) {
  if (user.isActive) {
    activeUsers.push(user);
  }
}
// 改后
const activeUsers = users.filter((user) => user.isActive);

// 简化：冗余 boolean 返回
// 改前
function isValid(input: string): boolean {
  if (input.length > 0 && input.length < 100) {
    return true;
  }
  return false;
}
// 改后
function isValid(input: string): boolean {
  return input.length > 0 && input.length < 100;
}
```

### Python

```python
# 简化：啰嗦的字典构建
# 改前
result = {}
for item in items:
    result[item.id] = item.name
# 改后
result = {item.id: item.name for item in items}

# 简化：嵌套条件 + early return
# 改前
def process(data):
    if data is not None:
        if data.is_valid():
            if data.has_permission():
                return do_work(data)
            else:
                raise PermissionError("No permission")
        else:
            raise ValueError("Invalid data")
    else:
        raise TypeError("Data is None")
# 改后
def process(data):
    if data is None:
        raise TypeError("Data is None")
    if not data.is_valid():
        raise ValueError("Invalid data")
    if not data.has_permission():
        raise PermissionError("No permission")
    return do_work(data)
```

### React / JSX

```tsx
// 简化：啰嗦的条件渲染
// 改前
function UserBadge({ user }: Props) {
  if (user.isAdmin) {
    return <Badge variant="admin">Admin</Badge>;
  } else {
    return <Badge variant="default">User</Badge>;
  }
}
// 改后
function UserBadge({ user }: Props) {
  const variant = user.isAdmin ? 'admin' : 'default';
  const label = user.isAdmin ? 'Admin' : 'User';
  return <Badge variant={variant}>{label}</Badge>;
}

// 简化：通过中间组件做 props drilling
// 改前 ， 先评估 context 或组合是不是更好的方案。
// 这是判断题 ， 标记，不要自动重构。
```

## 常见借口

| Rationalization | Reality |
|---|---|
| "能跑，没必要动" | 难读的代码坏的时候难修。早点简化在将来每次改动都省时间。 |
| "行数少就是简单" | 一行嵌套三元并不比五行 if/else 简单。简不简单是看理解速度，不是行数。 |
| "我顺手把这个无关的也简化下" | 不收敛的简化制造噪声 diff、带来你无意改的代码的回归风险。保持专注。 |
| "类型已经自文档了" | 类型记的是结构，不是意图。一个好名字比类型签名更解释 *为什么*。 |
| "这个抽象以后可能用" | 别留投机性抽象。现在不用，那它就是没挣到价值的复杂度。删了，需要时再加。 |
| "原作者肯定有理由" | 也许。看 git blame ， 套 Chesterton's Fence。但堆积的复杂度常常并无理由，只是时间压力下的迭代残留。 |
| "我加这个特性的同时重构一下" | 把重构和特性分开。混在一起既难审、又难回滚、也难从历史里读懂。 |

## 红旗

- 简化要求改测试才能通过（你大概改了行为）
- "简化"后反而比原来更长、更难看
- 按自己偏好而不是项目约定改名
- 为了"让代码更干净"而删错误处理
- 简化自己没完全理解的代码
- 把一堆简化打包进一笔难审的大提交
- 不被要求就对范围外做重构

## 验证

完成一轮简化后：

- [ ] 已有测试不改动就全部通过
- [ ] 构建成功且无新告警
- [ ] Lint / formatter 通过（无风格退化）
- [ ] 每个简化都是可审、可渐进的
- [ ] diff 干净 ， 没有不相关的改动混入
- [ ] 简化后的代码遵守项目约定（对照 CLAUDE.md 或等价文件核过）
- [ ] 没有任何错误处理被删或弱化
- [ ] 没有死代码被留下（不用的 import、不可达分支）
- [ ] 同事或评审 Agent 会把这次改动视为净改进
