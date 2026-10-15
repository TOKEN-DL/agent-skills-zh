---
description: 防重复护栏（添加或修改 Skill 时）
paths:
  - "skills/**"
---

# 添加或修改 Skill

本仓库已覆盖大部分开发生命周期，因此大多数新 Skill 想法都会与既有 Skill 或既有 PR 重叠。在新建 `skills/<name>/` 目录或大幅改动既有 Skill 之前：

- 先跑 [CONTRIBUTING.md](../../CONTRIBUTING.md#before-proposing-a-new-skill) 中的预检：检索目录、查看开放 PR（`gh pr list --state open`），并证明确实存在空白点。
- 优先扩展既有 Skill，而不是增加近似重复。若想法与既有 Skill 重叠，改既有 Skill 而非新建目录。
- 把 `SKILL.md` 控制在该范围 [docs/skill-anatomy.md](../../docs/skill-anatomy.md) 之内，绝不要在 Skill 之间重复内容，而是相互引用而不是复制。

CONTRIBUTING.md 是完整流程的唯一真理源；本规则只指向它，而不是重复它的清单。