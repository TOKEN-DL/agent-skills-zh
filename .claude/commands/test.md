---
description: 运行 TDD 工作流，先写失败测试，再实现，再验证。Bug 修复用 Prove-It 模式
---

调用 agent-skills:test-driven-development Skill。

新功能：

1. 写描述预期行为的测试（它们应 FAIL）
2. 实现代码使其通过
3. 保持测试绿灯的同时重构

Bug 修复（Prove-It 模式）：

1. 写一个复现 bug 的测试（必须 FAIL）
2. 确认该测试失败
3. 实现修复
4. 确认该测试通过
5. 跑完整测试套件查回归

与浏览器相关的问题，同时调用 agent-skills:browser-testing-with-devtools 通过 Chrome DevTools MCP 验证。