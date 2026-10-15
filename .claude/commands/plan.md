---
description: 把工作拆成可验证的小任务，每个都带验收标准与依赖顺序
---

调用 agent-skills:planning-and-task-breakdown Skill。

读现有 spec（SPEC.md 或等效文件）与相关代码段。然后：

1. 进入计划模式，只读，不改代码
2. 识别组件间的依赖图
4. 纵向切片（每个任务是一条完整路径，而不是横向分层）
4. 为每个任务写出验收标准与验证步骤
5. 在阶段之间加入检查点
6. 把计划交给人工审阅

把计划保存到 tasks/plan.md，把任务列表保存到 tasks/todo.md。

若 tasks/plan.md 或 tasks/todo.md 已经存在且对不同工作含有未勾选的任务，停下询问再写，绝不静默覆盖未完成的计划。