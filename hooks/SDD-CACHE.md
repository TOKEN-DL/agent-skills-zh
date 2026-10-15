# sdd-cache hook

为 [`source-driven-development`](../skills/source-driven-development/SKILL.md) 提供跨会话的引用缓存，跳过冗余的 `WebFetch` 调用，同时不削弱该 Skill "以最新文档为准"的保证。

## 为什么

`source-driven-development` 会为每个框架相关决策抓取官方文档。同一项目跨多个会话工作意味着反复抓取相同页面。把内容作为本地记忆缓存会与 Skill 矛盾，文档会变化，过期缓存就掩盖了这种变化。

本 hook 把抓取内容缓存到磁盘，但**每次复用时都通过 HTTP `If-None-Match` / `If-Modified-Since` 向源服务器重新校验**。仅当服务器返回 `304 Not Modified` 时才从缓存服务，这是"新鲜校验"，不是"读记忆"。

## 安装

1. 把 hooks 添加到 `.claude/settings.json`（个人使用可放到 `.claude/settings.local.json`）：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "WebFetch",
        "hooks": [
          {
            "type": "command",
            "command": "bash \"${CLAUDE_PROJECT_DIR}/hooks/sdd-cache-pre.sh\"",
            "timeout": 10
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "WebFetch",
        "hooks": [
          {
            "type": "command",
            "command": "bash \"${CLAUDE_PROJECT_DIR}/hooks/sdd-cache-post.sh\"",
            "async": true,
            "timeout": 10
          }
        ]
      }
    ]
  }
}
```

`${CLAUDE_PROJECT_DIR}` 解析为你启动 Claude Code 时所在的目录。上面片段在 hook 与项目同仓时可直接工作。若你把 `agent-skills` 安装到了别处（例如作为共享插件放在 `~/agent-skills`），把 `${CLAUDE_PROJECT_DIR}/hooks/...` 替换成每个脚本的绝对路径。

2. 确保 `.claude/sdd-cache/` 在你的 `.gitignore` 中（本仓库已包含）。

3. 像往常一样使用 `/source-driven-development`（或该 Skill）。Skill 与 Agent 工作流都不需要改动，缓存是透明的。

## 心智模型

按 URL 索引的 HTTP 资源缓存。新鲜度由源站通过 `ETag` / `Last-Modified` 委派校验；不设 TTL，不把 prompt 放进 key。

存储的 body 不是原始 HTML，`WebFetch` 会用调用方的 prompt 把每次响应通过模型做后处理，因此我们缓存的是某个 Agent 对该页面的"阅读"。Key 只用 URL 以便不同会话复用；原始 prompt 作为元数据保留，并在命中消息里透出，让下一个 Agent 判断那次旧阅读是否适用。

## 工作原理

每个 URL 一条缓存项，以 JSON 存于 `.claude/sdd-cache/<sha>.json`：

| 事件 | 动作 |
|---|---|
| `PreToolUse WebFetch` | 若条目存在，发起带 `If-None-Match` / `If-Modified-Since` 的 `HEAD` 请求。`304` 时阻断抓取，通过 stderr 把缓存内容返回给 Agent，并把原始 prompt 作为元数据透出。否则放行抓取。 |
| `PostToolUse WebFetch` | 捕获响应，发起 `HEAD` 请求记录当前 `ETag` / `Last-Modified`，并存储 `{url, prompt, etag, last_modified, content, fetched_at}`。 |

**新鲜度规则：**

- 仅当源站确认 `304 Not Modified` 时才返回条目。
- 没有 `ETag` 或 `Last-Modified` 头的条目永不缓存，没有验证器的话，hook 之后就无法校验新鲜度，缓存就变成了信任记忆。
- 缓存 key 是 `sha256(url)`。同一 URL 用不同 prompt 查询会命中同一条目；缓存 body 反映首次抓取时的 prompt，且 prompt 与命中一起展示，让 Agent 自行决定复用还是手动重新抓取。

**Agent 看到的内容：**

- 缓存命中：`WebFetch` 通过退出码 2 阻断。Claude Code 把 hook 的 stderr 负载作为工具错误回传给 Agent，这是缓存命中的"应有信号"，不是失败。负载以 `[sdd-cache] Cache hit for <url>` 为前缀，把缓存 body 包在 `----- BEGIN CACHED CONTENT -----` / `----- END CACHED CONTENT -----` 标记之间，让 Agent 能像 `WebFetch` 刚刚返回一样直接使用。
- 未命中或已过期：`WebFetch` 正常执行；结果被存起来供下次使用。

Skill 本身不变。它继续沿 `DETECT → FETCH → IMPLEMENT → CITE`。Hook 只改变了 `FETCH` 执行时的底层动作。

## 本地测试

### 1. 直接对脚本做冒烟测试

```bash
# 模拟 PostToolUse 负载：缓存一个页面
echo '{
  "tool_input": {
    "url": "https://react.dev/reference/react/useActionState",
    "prompt": "extract the signature"
  },
  "tool_response": "useActionState(action, initialState) returns [state, formAction, isPending]"
}' | bash hooks/sdd-cache-post.sh

# 查看存储的条目
ls .claude/sdd-cache/
cat .claude/sdd-cache/*.json | jq .

# 模拟同 URL + 同 prompt 的下一次 PreToolUse
echo '{
  "tool_input": {
    "url": "https://react.dev/reference/react/useActionState",
    "prompt": "extract the signature"
  }
}' | bash hooks/sdd-cache-pre.sh
echo "exit=$?"
```

预期：

- 第一条命令在 `.claude/sdd-cache/` 下创建一个文件（仅当服务器返回了 `ETag` 或 `Last-Modified` 时）。
- 第二条命令在源站回复 `304` 时退出码为 `2`、stderr 输出缓存内容；否则静默退出码为 `0`。

### 2. 在真实会话里做端到端

1. 按上文把 hooks 注册到 `.claude/settings.local.json`。
2. 在本仓库启动 Claude Code 会话。
3. 让 Agent 抓取一个文档页（例如 "fetch `https://react.dev/reference/react/useActionState` and summarize"）。
4. 校验 `.claude/sdd-cache/` 下出现了文件。
5. 让 Agent 用相同 prompt 再次抓取同一页面。
6. 校验第二次 `WebFetch` 被阻断，缓存内容被返回（可在会话转录里看到带 `[sdd-cache]` 前缀的工具错误）。

### 3. 新鲜度校验

为确认缓存在文档变化时会失效，构造一次 `ETag` 不匹配。挑一条具体条目，一旦缓存多于一个文件，`*.json` 通配就不再安全：

```bash
# 挑你想破坏的条目（替换成实际文件名）
ENTRY=.claude/sdd-cache/e49c9f378670cfbb1d7d871b6dee16d9.json

# 把它的 ETag 改成源站不会认的值
jq '.etag = "W/\"stale-etag-forced\""' "$ENTRY" > "$ENTRY.tmp" && mv "$ENTRY.tmp" "$ENTRY"

# 下一次 PreToolUse 应未命中（服务器返回 200，而非 304）
echo '{"tool_input":{"url":"...", "prompt":"..."}}' | bash hooks/sdd-cache-pre.sh
echo "exit=$?"   # 预期 0（放行抓取）
```

### 4. 调试

两个 hook 在调试模式开启时都会把带时间戳的事件写入 `.claude/sdd-cache/.debug.log`。可用下列任一方式开启：

```bash
# 方式 A：环境变量（按会话）
SDD_CACHE_DEBUG=1 claude

# 方式 B：哨兵文件（持久）
mkdir -p .claude/sdd-cache && touch .claude/sdd-cache/.debug
# 关闭：rm .claude/sdd-cache/.debug
```

日志捕获 URL、检测到的 `tool_response` 形态、HEAD 状态、以及每次命中或未命中的原因。在缓存未命中看起来不正常时很有用（通常是源站不再下发验证器）。

## 已知限制

- **body 由 prompt 塑形。** 命中返回的是之前 Agent 对该页面的阅读，并把原始 prompt 透出，让当前 Agent 自行判定是否适用。若不适用，删掉 `.claude/sdd-cache/` 下的对应文件以强制重新抓取。
- **每次缓存写入多付一次 HEAD。** Claude Code 没有暴露 `WebFetch` 已经收到的响应头，因此 post hook 不得不重新查询源站来获取 `ETag` / `Last-Modified`。每次未命中多一次往返，这是把它保持为"纯 hook、零核心改动"的代价。
- **没有 `ETag` 或 `Last-Modified` 的服务器永不缓存。** 大多数官方文档站（react.dev、docs.djangoproject.com、developer.mozilla.org）都会下发验证器；不下发的站总是被重新抓取。
- **行为不当的服务器可能返回错的 `304`。** 这是需要诊断的服务端 bug，而不是缓存要去捍卫的常量；我们不会用 TTL 来遮盖它。看到过期条目就删掉。
- **缓存是本地的、按项目隔离。** 没有跨团队共享缓存。加一层需要带签名的内容寻址存储，超出本 hook 范围。

## 依赖

- `jq`
- `curl`
- `shasum` 或 `sha256sum`（自动检测）
- Bash 3.2+