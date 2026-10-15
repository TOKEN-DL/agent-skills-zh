# simplify-ignore hook

为 `/code-simplify` 提供代码块级保护。标记不应被简化的代码，模型看不到它们。

## 安装

1. 标注你想保护的代码块：

```js
/* simplify-ignore-start: perf-critical */
// manually unrolled XOR ， 3x faster than a loop
result[0] = buf[0] ^ key[0];
result[1] = buf[1] ^ key[1];
result[2] = buf[2] ^ key[2];
result[3] = buf[3] ^ key[3];
/* simplify-ignore-end */
```

2. 把 hooks 添加到 `.claude/settings.json`：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Read",
        "hooks": [{ "type": "command", "command": "bash \"${CLAUDE_PROJECT_DIR}/hooks/simplify-ignore.sh\"" }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "bash \"${CLAUDE_PROJECT_DIR}/hooks/simplify-ignore.sh\"" }]
      }
    ],
    "Stop": [
      {
        "hooks": [{ "type": "command", "command": "bash \"${CLAUDE_PROJECT_DIR}/hooks/simplify-ignore.sh\"" }]
      }
    ]
  }
}
```

3. 运行 `/code-simplify`，被保护的代码块会变成 `/* BLOCK_de115a1d: perf-critical */` 占位符。模型在看不到被保护实现的情况下，仍能基于周边代码进行推理。

> **注意：** hook 把临时备份存在 `.claude/.simplify-ignore-cache/`。请确保该路径在你的 `.gitignore` 里。

## 工作原理

一个脚本、三种 hook 事件：

| 事件 | 动作 |
|---|---|
| `PreToolUse Read` | 备份文件，把代码块原地替换成 `BLOCK_<hash>` 占位符 |
| `PostToolUse Edit\|Write` | 把占位符再展开回原始代码，保存模型的改动，重新过滤 |
| `Stop` | 展开磁盘上每个文件里残留的占位符，让通过 `Edit`/`Write` 之外的路径所做出的改动得以保留；保留被整体还原的文件（`git checkout`/`stash` 回退、撤销操作）而不是过期的备份；只有在没有任何占位符残留时才回退到备份 |

每个代码块用 `shasum`/`sha1sum` 取内容哈希（8 hex chars），这样即便模型复制或重排了占位符也能明确往返。缓存按项目作用域，避免跨会话干扰。

## 标注语法

```js
/* simplify-ignore-start */           // 基础：仅隐藏代码块
/* simplify-ignore-start: reason */   // 带原因：会出现在占位符里
/* simplify-ignore-end */
```

任意注释风格都能用（`//`、`/*`、`#`、`<!--`）。每个文件允许多块，也支持单行代码块。占位符保留原始注释语法（如 Python 是 `# BLOCK_xxx`，HTML 是 `<!-- BLOCK_xxx -->`）。

## 崩溃恢复

若 Claude Code 崩溃且没触发 Stop hook，磁盘上的文件或许仍留着 `BLOCK_<hash>` 占位符。要手动恢复：

```bash
echo '{}' | bash hooks/simplify-ignore.sh
```

备份存在项目目录下的 `.claude/.simplify-ignore-cache/`。
若在 hook 生效时某次改动被回退，被取代的会话备份会以 `<id>.superseded.<pid>` 形式保留在那里；不再需要时请自行审阅并删除。

## 已知限制

- **单行代码块会隐藏整行。** 若 `simplify-ignore-start` 与 `simplify-ignore-end` 与其他代码出现在同一行，模型看到的将是整行隐藏，而非仅标注部分。请为标注单独起一行。
- **注释后缀识别仅覆盖 `*/` 与 `-->`。** 使用非标准注释闭合符的模板引擎（ERB `%>`、Blade `--}}`）可能产生不平衡的占位符。请改用 `#` 或 `//` 风格注释。
- **回退展开是渐进的，而不是精确的。** 若模型改动了占位符的格式（例如改了原因文本），它会按 全占位符 → 前缀+哈希+后缀 → 仅哈希 的顺序逐级尝试。仅哈希的回退可能留下装饰性残留（如多余的 `:` 或原因文本）。发生这种情况时会向 stderr 打印警告。
- **占位符被删除时会回退到备份。** 若一次编辑连同 `BLOCK_<hash>` 行一起被删除，或整文件被替换，hook 就再也无法定位那块被保护代码。这种 `Edit` 或 `Write` 会通过退出码 2 把应恢复的占位符回传给模型，并且备份不会被更新。若到 `Stop` 时占位符仍然缺失，hook 恢复备份并打印警告；改过的文件没有丢失，但被以 `.claude/.simplify-ignore-cache/<id>.recovered` 形式保留在缓存里（路径会随警告一起打印），由你手动合并回去。那些保留占位符的改动（Bash 编辑、格式器、外部编辑器）则直接留在文件里。
- **文件改名会留下占位符。** 若模型通过 Shell 命令改名或移动文件，新文件会保留 `BLOCK_<hash>` 占位符。原代码被存为 `<old-filename>.recovered`，在会话结束时保留。你必须手动把恢复出来的代码搬回新文件。

## 依赖

- `jq`、`shasum` 或 `sha1sum`（自动检测）、Bash 3.2+