# 在 Windsurf 上使用 agent-skills

## 配置

### 项目级规则（Project Rules）

Windsurf 用 `.windsurfrules` 存放项目特定的 Agent 指令：

```bash
# 把最重要的几个 Skill 合并到一个规则文件
cat /path/to/agent-skills/skills/test-driven-development/SKILL.md > .windsurfrules
echo "\n---\n" >> .windsurfrules
cat /path/to/agent-skills/skills/incremental-implementation/SKILL.md >> .windsurfrules
echo "\n---\n" >> .windsurfrules
cat /path/to/agent-skills/skills/code-review-and-quality/SKILL.md >> .windsurfrules
```

### 全局规则（Global Rules）

对想跨项目生效的 Skill，把它们加进 Windsurf 的全局规则：

1. 打开 Windsurf → Settings → AI → Global Rules
2. 粘贴你最常用的 Skill 的内容

## 推荐配置

保持 `.windsurfrules` 聚焦在 2-3 个核心 Skill 上，避免超出上下文限制：

```
# .windsurfrules
# 本项目必备的 agent-skills

[粘贴 test-driven-development 的 SKILL.md]

---

[粘贴 incremental-implementation 的 SKILL.md]

---

[粘贴 code-review-and-quality 的 SKILL.md]
```

## 使用技巧

1. **有取舍** ， Windsurf 的上下文有限。挑选能解决你最大质量缺口的 Skill。
2. **在对话中引用** ， 工作进入特定阶段时，把对应的 Skill 内容粘贴进聊天（例如构建鉴权时粘贴 `security-and-hardening`）。
3. **把 references 当清单用** ， 粘贴 `references/security-checklist.md`，让 Windsurf 逐项核对。