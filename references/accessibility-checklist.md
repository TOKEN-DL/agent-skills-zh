# Accessibility Checklist

WCAG 2.1 AA 合规的速查。与 `frontend-ui-engineering` Skill 一起用。

## 目录

- [基础检查](#基础检查)
- [常见 HTML 模式](#常见-html-模式)
- [测试工具](#测试工具)
- [速查：ARIA Live Regions](#速查aria-live-regions)
- [常见反模式](#常见反模式)

## 基础检查

### 键盘导航
- [ ] 所有交互元素都能用 Tab 聚焦
- [ ] 聚焦顺序跟着视觉 / 逻辑顺序
- [ ] 焦点可见（聚焦元素上有 outline / ring）
- [ ] 自定义控件有键盘支持（Enter 激活、Escape 关闭）
- [ ] 没有键盘陷阱（用户能随时 Tab 离开一个组件）
- [ ] 页首有"跳到正文"链接 ， 键盘聚焦时（至少）可见
- [ ] 模态打开时圈住焦点、关闭时把焦点交回去

### 屏幕阅读器
- [ ] 所有图片有 `alt` 文本（装饰图则 `alt=""`）
- [ ] 所有表单 input 有对应 label（`<label>` 或 `aria-label`）
- [ ] 按钮与链接的文本是描述性的（不要"点这里"）
- [ ] 仅图标的按钮有 `aria-label`
- [ ] 页面有一个 `<h1>`，标题层级不跳
- [ ] 动态内容变化会被宣布（`aria-live` 区域）
- [ ] 表格有 `<th>` 表头与 scope

### 视觉
- [ ] 文本对比度 ≥ 4.5:1（正文）或 ≥ 3:1（大字，18px+）
- [ ] UI 组件相对背景 ≥ 3:1
- [ ] 颜色不是传达信息的唯一手段
- [ ] 文本能放大到 200% 不破布局
- [ ] 没有每秒闪 3 次以上的内容

### 表单
- [ ] 每个 input 有可见 label
- [ ] 必填字段被标明（不能只用颜色）
- [ ] 错误信息具体且关联到字段
- [ ] 错误状态除颜色外还能看到（图标、文字、边框）
- [ ] 表单提交错误总览可聚焦
- [ ] 已知字段使用 autocomplete（如 `type="email" autocomplete="email"`）

### 内容
- [ ] 声明语言（`<html lang="en">`）
- [ ] 页面有描述性的 `<title>`
- [ ] 链接与正文不同（不能仅靠颜色）
- [ ] 触屏目标 ≥ 44x44px 在移动端
- [ ] 空状态是有意义的（不是空白屏）

## 常见 HTML 模式

### Button vs. Link

```html
<!-- 用 <button> 做动作 -->
<button onClick={handleDelete}>删除任务</button>

<!-- 用 <a> 做导航 -->
<a href="/tasks/123">查看任务</a>

<!-- 永远不要用 div / span 当按钮 -->
<div onClick={handleDelete}>删除</div>  <!-- 反 -->
```

### 表单 Label

```html
<!-- 显式 label 关联 -->
<label htmlFor="email">邮箱地址</label>
<input id="email" type="email" required />

<!-- 隐式包裹 -->
<label>
  邮箱地址
  <input type="email" required />
</label>

<!-- 隐藏 label（首选可见 label）-->
<input type="search" aria-label="搜索任务" />
```

### ARIA 角色

```html
<!-- 导航 -->
<nav aria-label="主导航">...</nav>
<nav aria-label="页脚链接">...</nav>

<!-- 状态消息 -->
<div role="status" aria-live="polite">任务已保存</div>

<!-- 告警消息 -->
<div role="alert">错误：标题必填</div>

<!-- 模态对话框 -->
<dialog aria-modal="true" aria-labelledby="dialog-title">
  <h2 id="dialog-title">确认删除</h2>
  ...
</dialog>

<!-- 加载状态 -->
<div aria-busy="true" aria-label="加载任务中">
  <Spinner />
</div>
```

### 可访问的列表

```html
<ul role="list" aria-label="任务">
  <li>
    <input type="checkbox" id="task-1" aria-label="完成：买杂货" />
    <label htmlFor="task-1">买杂货</label>
  </li>
</ul>
```

## 测试工具

```bash
# 自动化审计
npx axe-core          # 程序化无障碍测试
npx pa11y             # CLI 无障碍检查

# 在浏览器里
# Chrome DevTools → Lighthouse → Accessibility
# Chrome DevTools → Elements → Accessibility tree

# 屏幕阅读器测试
# macOS：VoiceOver（Cmd + F5）
# Windows：NVDA（免费）或 JAWS
# Linux：Orca
```

## 速查：ARIA Live Regions

| 值 | 行为 | 用于 |
|-------|----------|---------|
| `aria-live="polite"` | 在下一个停顿被宣布 | 状态更新、保存确认 |
| `aria-live="assertive"` | 立即宣布 | 错误、时效性告警 |
| `role="status"` | 同 `polite` | 状态消息 |
| `role="alert"` | 同 `assertive` | 错误消息 |

## 常见反模式

| 反模式 | 问题 | 修 |
|---|---|---|
| `div` 当按钮 | 不可聚焦、没键盘支持 | 用 `<button>` |
| 缺 `alt` | 屏幕阅读器看不见图片 | 加描述性 `alt` |
| 仅靠颜色表达状态 | 色盲用户不可见 | 加图标、文字或图案 |
| 自动播放媒体 | 令人迷失、不能停 | 加控制、不要自动播放 |
| 自定义下拉缺 ARIA | 键盘 / 屏幕阅读器不可用 | 用原生 `<select>` 或合规的 ARIA listbox |
| 删掉焦点轮廓 | 用户看不到自己在哪 | 把轮廓样式化、不要删 |
| 空链接 / 按钮 | "链接"被宣布但没说明 | 加文本或 `aria-label` |
| `tabindex > 0` | 打乱自然 tab 序 | 只用 `tabindex="0"` 或 `-1` |
