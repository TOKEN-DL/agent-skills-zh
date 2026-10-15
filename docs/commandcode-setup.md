# 在 Command Code 上使用 agent-skills

[Command Code](https://commandcode.ai) 内置技能系统。自带的 `cmd skills` 命令会克隆一个 GitHub 仓库、递归发现所有 `SKILL.md`，然后安装你挑选的那些。

Command Code 二进制名为 `cmd`（在 Windows 上还叫 `cmdc`，又叫 `command-code`）。下面示例统一使用 `cmd`。

## 安装

**项目作用域**（默认；安装到当前 git 根的 `.commandcode/skills/`）：

```bash
cmd skills add addyosmani/agent-skills
```

在交互式终端里这会弹出多选菜单，让你挑选 25 个 Skill 中的子集。管道 / 非交互调用则安装发现到的所有 Skill。

**安装指定 Skill：**

```bash
cmd skills add addyosmani/agent-skills -s spec-driven-development
```

**用户作用域**（安装到 `~/.commandcode/skills/`，在每个项目里都生效）：

```bash
cmd skills add addyosmani/agent-skills --global
```

**其他支持的形态：**

```bash
cmd skills add addyosmani/agent-skills@main            # 指定分支
cmd skills add addyosmani/agent-skills/skills/interview-me   # 仓库内的指定路径
cmd skills add addyosmani/agent-skills --force         # 已安装时覆盖 / 升级
```

## 管理

```bash
cmd skills list                       # 列出已安装的 Skill（项目 + 用户 + 内置）
cmd skills remove spec-driven-development           # 移除项目级 Skill
cmd skills remove spec-driven-development --global  # 移除用户级 Skill
```

`add` 上的 `--force` 会重新拉取并覆盖既有 Skill，这就是升级到最新版本的方式。

## 用法

已安装的 Skill 自动被发现，并出现在 TUI 的斜杠菜单里，标签为 `[skill]`：

```
/spec-driven-development   [skill] Write a spec before writing code…
```

输入 `/` 浏览，或者开始输入 Skill 名做筛选。用 `/skills` 启用 / 禁用 Skill。

## Skill 存放位置

Command Code 从下列位置发现 Skill（项目条目相对最近的 git 根解析）：

| 作用域 | 路径 |
|-------|------|
| 项目 | `.commandcode/skills/<name>/SKILL.md` |
| 项目（agents 兼容） | `.agents/skills/<name>/SKILL.md` |
| 用户 | `~/.commandcode/skills/<name>/SKILL.md` |
| 用户（agents 兼容） | `~/.agents/skills/<name>/SKILL.md` |

`cmd skills add` 默认写入 `.commandcode/skills/`（项目）或 `~/.commandcode/skills/`（`--global`）。