---
name: frontend-ui-engineering
description: 构建生产级、可访问、响应式的前端 UI。在构建或修改界面与页面、创建组件、落实布局、满足 WCAG 可访问性、管理状态、或需要输出看起来和用起来都是生产级而不是 AI 生成时使用。
---

# Frontend UI Engineering

## 概述

构建生产级的用户界面：可访问、高性能、视觉精致。目标是 UI 看起来像是有设计感的工程师在顶级公司造的，而不是 AI 生成的。这代表真正的设计系统遵循、合适的可访问性、有思考的交互模式、以及没有泛泛的"AI 美感"。

## 何时使用

- 构建新 UI 组件或页面
- 修改既有用户界面
- 实现响应式布局
- 加交互或状态管理
- 修视觉或 UX 问题

## 组件架构

### 文件结构

把组件相关的所有东西就近放：

```
src/components/
  TaskList/
    TaskList.tsx          # 组件实现
    TaskList.test.tsx     # 测试
    TaskList.stories.tsx  # Storybook stories（如使用）
    use-task-list.ts      # 自定义 hook（复杂状态时）
    types.ts              # 组件专属类型（如需要）
```

### 组件模式

**优先组合，而非配置：**

```tsx
// 好：可组合
<Card>
  <CardHeader>
    <CardTitle>Tasks</CardTitle>
  </CardHeader>
  <CardBody>
    <TaskList tasks={tasks} />
  </CardBody>
</Card>

// 避免：过度配置
<Card
  title="Tasks"
  headerVariant="large"
  bodyPadding="md"
  content={<TaskList tasks={tasks} />}
/>
```

**保持组件专注：**

```tsx
// 好：做一件事
export function TaskItem({ task, onToggle, onDelete }: TaskItemProps) {
  return (
    <li className="flex items-center gap-3 p-3">
      <Checkbox checked={task.done} onChange={() => onToggle(task.id)} />
      <span className={task.done ? 'line-through text-muted' : ''}>{task.title}</span>
      <Button variant="ghost" size="sm" onClick={() => onDelete(task.id)}>
        <TrashIcon />
      </Button>
    </li>
  );
}
```

**把数据获取与展示分开：**

```tsx
// 容器：管数据
export function TaskListContainer() {
  const { tasks, isLoading, error, refetch } = useTasks();

  if (isLoading) return <TaskListSkeleton />;
  if (error) return <ErrorState message="加载任务失败" retry={refetch} />;
  if (tasks.length === 0) return <EmptyState message="还没有任务" />;

  return <TaskList tasks={tasks} />;
}

// 展示：管渲染
export function TaskList({ tasks }: { tasks: Task[] }) {
  return (
    <ul role="list" className="divide-y">
      {tasks.map(task => <TaskItem key={task.id} task={task} />)}
    </ul>
  );
}
```

## 状态管理

**挑能 work 的最简单方案：**

```
本地状态 (useState)              → 组件专属的 UI 状态
提升的状态                        → 2-3 个兄弟组件共享
Context                          → 主题、认证、locale（读多写少）
URL 状态 (searchParams)          → 筛选、分页、可分享的 UI 状态
服务端状态 (React Query, SWR)    → 带缓存的远端数据
全局 store (Zustand, Redux)      → 应用级共享的复杂客户端状态
```

**避免超过 3 层深的 prop drilling。** 如果你把 props 穿过根本不用它们的中间组件，引入 context 或重构组件树。

## 设计系统遵循

### 参考驱动的 UI 质量

当产品需要独特的视觉方向时，先收集证据再决定布局：

1. 查可信的参考目录，或者用产品团队给的参考。
2. 看两到三个相关页面。记下层级、密度、导航、控件、响应式行为、交互状态的决策。
3. 在实现前把这些决策变成简短的设计合约。命名页面的工作、主要动作、必备状态、响应式规则、要拒绝的模式。
4. 用产品自己的组件、token、内容、视觉语言重建有用的结构。不要照搬别家的品牌、专有文本、图像或确切布局。

参考是证据，不是模板。如果拿不到参考，把假设文档化，用产品既有的设计系统核验结果。

### 避免"AI 美感"

AI 生成的 UI 有可识别的模式。全部避开：

| AI 默认 | 为什么有问题 | 生产级 |
|---|---|---|
| 到处都是紫 / 靛蓝 | 模型默认走"视觉安全"的调色板，让每个应用看起来一样 | 用项目实际的调色板 |
| 过度的渐变 | 渐变制造视觉噪声、与多数设计系统冲突 | 平面或克制渐变对齐设计系统 |
| 一切都圆角（rounded-2xl） | 最大圆角信号"友好"，但忽略真实设计里圆角的层级 | 从设计系统挑一致的 border-radius |
| 通用 hero section | 模板驱动的布局，与真实内容或用户需要无关 | 内容先行的布局 |
| Lorem ipsum 式文案 | 占位符藏起真实内容才会暴露的布局问题（长度、断行、溢出） | 现实的占位内容 |
| 处处超大 padding | 平铺的慷慨 padding 毁掉视觉层级、浪费屏幕 | 一致的间距尺度 |
| 库存卡套件网格 | 统一网格是忽略信息优先级和扫描模式的布局捷径 | 服务目标的布局 |
| 重阴影设计 | 层叠阴影加深度，与内容抢戏、低端设备渲染慢 | 设计系统要的克制或无阴影 |

### 间距与布局

用一致的间距尺度。不要现造值：

```css
/* 用尺度：0.25rem 步进（或项目实际用的尺度）*/
/* 好 */  padding: 1rem;      /* 16px */
/* 好 */  gap: 0.75rem;       /* 12px */
/* 坏 */   padding: 13px;      /* 不在任何尺度上 */
/* 坏 */   margin-top: 2.3rem; /* 不在任何尺度上 */
```

### 排版

尊重类型层级：

```
h1 → 页面标题（每页一个）
h2 → 节标题
h3 → 子节标题
body → 默认正文
small → 次要 / 辅助文字
```

不要跳级。不要拿 heading 样式给非标题内容用。

### 颜色

- 用语义色 token：`text-primary`、`bg-surface`、`border-default` ， 不要原始十六进制
- 保证足够对比（正文 4.5:1，大字 3:1）
- 不要单靠颜色传递信息（再加图标、文字或图案）

## 可访问性（WCAG 2.1 AA）

每个组件必须满足：

### 键盘导航

```tsx
// 每个交互元素都必须键盘可访问
<button onClick={handleClick}>点我</button>            // ✓ 默认可聚焦
<div onClick={handleClick}>点我</div>               // ✗ 不可聚焦
<div role="button" tabIndex={0} onClick={handleClick} // ✓ 但首选 <button>
     onKeyDown={e => {
       if (e.key === 'Enter') handleClick();
       if (e.key === ' ') e.preventDefault();
     }}
     onKeyUp={e => {
       if (e.key === ' ') handleClick();
     }}>
  点我
</div>
```

### ARIA 标签

```tsx
// 标没有可见文字的交互元素
<button aria-label="关闭对话框"><XIcon /></button>

// 标表单输入
<label htmlFor="email">邮箱</label>
<input id="email" type="email" />

// 当没有可见 label 时用 aria-label
<input aria-label="搜索任务" type="search" />
```

### 焦点管理

```tsx
// 内容变化时移动焦点
function Dialog({ isOpen, onClose }: DialogProps) {
  const closeRef = useRef<HTMLButtonElement>(null);

  useEffect(() => {
    if (isOpen) closeRef.current?.focus();
  }, [isOpen]);

  // 打开时把焦点圈在对话框内
  return (
    <dialog open={isOpen}>
      <button ref={closeRef} onClick={onClose}>关闭</button>
      {/* 对话框内容 */}
    </dialog>
  );
}
```

### 有意义的空态和错态

```tsx
// 别让屏幕一片空白
function TaskList({ tasks }: { tasks: Task[] }) {
  if (tasks.length === 0) {
    return (
      <div role="status" className="text-center py-12">
        <TasksEmptyIcon className="mx-auto h-12 w-12 text-muted" />
        <h3 className="mt-2 text-sm font-medium">没有任务</h3>
        <p className="mt-1 text-sm text-muted">从一个新任务开始。</p>
        <Button className="mt-4" onClick={onCreateTask}>新建任务</Button>
      </div>
    );
  }

  return <ul role="list">...</ul>;
}
```

## 响应式设计

先为移动端做设计，再扩展：

```tsx
// Tailwind：移动端优先的响应式
<div className="
  grid grid-cols-1      /* 移动端：单列 */
  sm:grid-cols-2        /* 小屏：2 列 */
  lg:grid-cols-3        /* 大屏：3 列 */
  gap-4
">
```

在这些断点测试：320px、768px、1024px、1440px。

## 加载与过渡

```tsx
// 骨架屏（不要用 spinner 加载内容）
function TaskListSkeleton() {
  return (
    <div className="space-y-3" aria-busy="true" aria-label="正在加载任务">
      {Array.from({ length: 3 }).map((_, i) => (
        <div key={i} className="h-12 bg-muted animate-pulse rounded" />
      ))}
    </div>
  );
}

// 乐观更新以感知加速
function useToggleTask() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: toggleTask,
    onMutate: async (taskId) => {
      await queryClient.cancelQueries({ queryKey: ['tasks'] });
      const previous = queryClient.getQueryData(['tasks']);

      queryClient.setQueryData(['tasks'], (old: Task[]) =>
        old.map(t => t.id === taskId ? { ...t, done: !t.done } : t)
      );

      return { previous };
    },
    onError: (_err, _taskId, context) => {
      queryClient.setQueryData(['tasks'], context?.previous);
    },
  });
}
```

## 参见

详细的可访问性要求与测试工具，见 `../../references/accessibility-checklist.md`。

## 常见借口

| Rationalization | Reality |
|---|---|
| "可访问性是锦上添花" | 很多司法管辖区是法律要求，也是工程质量标准。 |
| "我们等会再补响应式" | 后期补响应式是从一开始就做的 3 倍难。 |
| "设计还没定，那就先不管样式" | 用设计系统的默认值。没样式的 UI 给评审者留下破损的第一印象。 |
| "这只是原型" | 原型会变生产代码。一开始就把基础打对。 |
| "AI 美感现在能用就行" | 它信号"质量差"。从一开始就上项目实际的设计系统。 |

## 红旗

- 组件超过 200 行（拆）
- 内联样式或任意像素值
- 缺错误状态、加载状态或空状态
- 没测过键盘导航
- 颜色是状态的唯一信号（红 / 绿不带文字或图标）
- 通用 "AI 风格"（紫色渐变、超大卡片、库存布局）

## 验证

构建完 UI 后：

- [ ] 组件渲染时控制台没有错误
- [ ] 所有交互元素都键盘可访问（用 Tab 走一遍页面）
- [ ] 屏幕阅读器能传达页面内容与结构
- [ ] 响应式：在 320、768、1024、1440px 都工作
- [ ] 加载、空、错、成功、权限状态按需处理
- [ ] 遵循项目设计系统（间距、颜色、字号）
- [ ] 渲染结果过了一次 UI 专门的完工闸评审
- [ ] 在 dev tools 或 axe-core 里没有可访问性告警
