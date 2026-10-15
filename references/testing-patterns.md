# Testing Patterns Reference (JavaScript/TypeScript)

JS/TS 测试模式速查 ， Jest、React Testing Library、Supertest 与 Playwright ， 展示 `test-driven-development` Skill 的通用原则。这些原则（Arrange-Act-Assert、命名、Mock 纪律、反模式）适用于任何生态；这里展示的语法与工具是 JS/TS 专属的。在别的栈下，用同一套原则换上该仓库自己的测试框架与命令。

## 目录

- [测试结构（Arrange-Act-Assert）](#测试结构arrange-act-assert)
- [测试命名约定](#测试命名约定)
- [常见断言](#常见断言)
- [Mock 模式](#mock-模式)
- [React / 组件测试](#react--组件测试)
- [API / 集成测试](#api--集成测试)
- [E2E 测试（Playwright）](#e2e-测试playwright)
- [测试反模式](#测试反模式)

## 测试结构（Arrange-Act-Assert）

```typescript
it('描述期望行为', () => {
  // Arrange：搭测试数据与前置条件
  const input = { title: 'Test Task', priority: 'high' };

  // Act：执行要测的动作
  const result = createTask(input);

  // Assert：核结果
  expect(result.title).toBe('Test Task');
  expect(result.priority).toBe('high');
  expect(result.status).toBe('pending');
});
```

## 测试命名约定

```typescript
// 模式：[单元] [期望行为] [条件]
describe('TaskService.createTask', () => {
  it('创建一个带默认 pending 状态的任务', () => {});
  it('title 为空时抛 ValidationError', () => {});
  it('裁掉 title 里的空白', () => {});
  it('为每个任务生成一个唯一 ID', () => {});
});
```

## 常见断言

```typescript
// 相等
expect(result).toBe(expected);           // 严格相等（===）
expect(result).toEqual(expected);        // 深相等（对象 / 数组）
expect(result).toStrictEqual(expected);  // 深相等 + 类型匹配

// 真值
expect(result).toBeTruthy();
expect(result).toBeFalsy();
expect(result).toBeNull();
expect(result).toBeDefined();
expect(result).toBeUndefined();

// 数字
expect(result).toBeGreaterThan(5);
expect(result).toBeLessThanOrEqual(10);
expect(result).toBeCloseTo(0.3, 5);      // 浮点

// 字符串
expect(result).toMatch(/pattern/);
expect(result).toContain('substring');

// 数组 / 对象
expect(array).toContain(item);
expect(array).toHaveLength(3);
expect(object).toHaveProperty('key', 'value');

// 错误
expect(() => fn()).toThrow();
expect(() => fn()).toThrow(ValidationError);
expect(() => fn()).toThrow('specific message');

// 异步
await expect(asyncFn()).resolves.toBe(value);
await expect(asyncFn()).rejects.toThrow(Error);
```

## Mock 模式

### Mock 函数

```typescript
const mockFn = jest.fn();
mockFn.mockReturnValue(42);
mockFn.mockResolvedValue({ data: 'test' });
mockFn.mockImplementation((x) => x * 2);

expect(mockFn).toHaveBeenCalled();
expect(mockFn).toHaveBeenCalledWith('arg1', 'arg2');
expect(mockFn).toHaveBeenCalledTimes(3);
```

### Mock 模块

```typescript
// Mock 整个模块
jest.mock('./database', () => ({
  query: jest.fn().mockResolvedValue([{ id: 1, title: 'Test' }]),
}));

// Mock 特定导出
jest.mock('./utils', () => ({
  ...jest.requireActual('./utils'),
  generateId: jest.fn().mockReturnValue('test-id'),
}));
```

### 只在边界做 Mock

```
要 Mock 的：                    不要 Mock 的：
├── 数据库调用                  ├── 内部工具函数
├── HTTP 请求                   ├── 业务逻辑
├── 文件系统操作                ├── 数据转换
├── 外部 API 调用               ├── 校验函数
└── 时间 / 日期（如必要）      └── 纯函数
```

## React / 组件测试

```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';

describe('TaskForm', () => {
  it('用输入的数据提交表单', async () => {
    const onSubmit = jest.fn();
    render(<TaskForm onSubmit={onSubmit} />);

    // 按可访问 role / label 找元素（不要用 test ID）
    await screen.findByRole('textbox', { name: /title/i });
    fireEvent.change(screen.getByRole('textbox', { name: /title/i }), {
      target: { value: '新任务' },
    });
    fireEvent.click(screen.getByRole('button', { name: /create/i }));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({ title: '新任务' });
    });
  });

  it('空 title 时显示校验错误', async () => {
    render(<TaskForm onSubmit={jest.fn()} />);

    fireEvent.click(screen.getByRole('button', { name: /create/i }));

    expect(await screen.findByText(/title is required/i)).toBeInTheDocument();
  });
});
```

## API / 集成测试

```typescript
import request from 'supertest';
import { app } from '../src/app';

describe('POST /api/tasks', () => {
  it('创建一个任务并返回 201', async () => {
    const response = await request(app)
      .post('/api/tasks')
      .send({ title: 'Test Task' })
      .set('Authorization', `Bearer ${testToken}`)
      .expect(201);

    expect(response.body).toMatchObject({
      id: expect.any(String),
      title: 'Test Task',
      status: 'pending',
    });
  });

  it('输入非法时返回 422', async () => {
    const response = await request(app)
      .post('/api/tasks')
      .send({ title: '' })
      .set('Authorization', `Bearer ${testToken}`)
      .expect(422);

    expect(response.body.error.code).toBe('VALIDATION_ERROR');
  });

  it('未认证时返回 401', async () => {
    await request(app)
      .post('/api/tasks')
      .send({ title: 'Test' })
      .expect(盘);
  });
});
```

## E2E 测试（Playwright）

```typescript
import { test, expect } from '@playwright/test';

test('用户能创建并完成任务', async ({ page }) => {
  // 导航并登录
  await page.goto('/');
  await page.getByRole('textbox', { name: /email/i }).fill('test@example.com');
  await page.getByLabel(/password/i).fill('testpass123');
  await page.getByRole('button', { name: /log in/i }).click();

  // 创建任务
  await page.getByRole('button', { name: /new task/i }).click();
  await page.getByRole('textbox', { name: /title/i }).fill('买杂货');
  await page.getByRole('button', { name: /create/i }).click();

  // 确认任务出现
  const task = page.getByRole('listitem', { name: /买杂货/i });
  await expect(task).toBeVisible();

  // 完成任务
  await task.getByRole('checkbox', { name: /完成 买杂货/i }).check();
  await expect(task).toHaveCSS('text-decoration-line', 'line-through');
});
```

## 测试反模式

| 反模式 | 问题 | 更好做法 |
|---|---|---|
| 测实现细节 | 重构时挂 | 测输入 / 输出 |
| 一切都做 snapshot | 没人审 snapshot diff | 断言具体值 |
| 共享可变状态 | 测试互相污染 | 每条测试 setup / teardown |
| 测第三方代码 | 浪费时间、不是你的 bug | Mock 边界 |
| 跳过测试让 CI 过 | 藏真实 bug | 修或删 |
| 永久 `test.skip` | 死代码 | 删或修 |
| 断言太宽 | 抓不到回归 | 写得具体 |
| 不处理 async 错误 | 错误被吞、假通过 | async 测试总是 `await` |
