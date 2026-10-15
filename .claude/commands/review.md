---
description: 跨五个维度的代码评审，正确性、可读性、架构、安全、性能
---

调用 agent-skills:code-review-and-quality Skill。

按下列五个维度评审当前改动（已暂存或最近提交）：

1. **正确性（Correctness）** ， 是否符合 spec？边界情况已处理？测试充分？
2. **可读性（Readability）** ， 命名清晰？逻辑直白？组织合理？
3. **架构（Architecture）** ， 遵循既有模式？边界清晰？抽象层级合适？
4. **安全（Security）** ， 输入已校验？机密安全？鉴权已检？（使用 security-and-hardening Skill）
5. **性能（Performance）** ， 没有 N+1 查询？无无界操作？（使用 performance-optimization Skill）

把发现分类为 Critical、Required、Optional 或 Nit。
输出一份带具体 file:line 引用与修复建议的结构化评审。