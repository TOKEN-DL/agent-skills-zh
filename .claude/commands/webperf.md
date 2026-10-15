---
description: 通过 web-performance-auditor persona 跑 Web 性能审计
---

`/webperf` 专门面向 Web 应用。对工具库、CLI 或没有浏览器侧输出的纯服务端代码不要使用。

## 判定模式

**Deep 模式** ， 下列任一可用时启用：

- A Lighthouse JSON 报告文件（例如 `npx lighthouse <url> --output json --output-path ./report.json`，或 Chrome DevTools MCP CLI 的 `npx -p chrome-devtools-mcp chrome-devtools lighthouse_audit --output-format=json`）
- A PageSpeed Insights JSON 响应（含 Lighthouse + CrUX）
- A CrUX API 响应（需要 `CRUX_API_KEY` 或 `GOOGLE_API_KEY`）
- A DevTools 性能轨迹
- 一个在线 URL 加 harness 中配置好的 `chrome-devtools` MCP 服务器（Agent 可通过 `lighthouse_audit` 与 `performance_*` 工具直接捕获指标）
- 本地调用的 Chrome DevTools MCP CLI（通过 `npx -p chrome-devtools-mcp chrome-devtools <tool>`，或在 `npm i -g chrome-devtools-mcp` 之后），用户跑像 `chrome-devtools lighthouse_audit --output-format=json` 这样的命令，并把 JSON 输出交给 Agent

**Quick 模式** ， 上面都没有时的默认。Agent 在源码中扫描结构性反模式，并把每个发现标为 `potential impact`。

## 跑审计

派生 `web-performance-auditor` 子 Agent。显式传给子 Agent：

- 待评审的文件、组件或 diff
- 任何工件路径（Lighthouse JSON、PSI JSON、CrUX 响应、轨迹）或粘贴的 JSON 内容
- 已知的 URL 或页面名
- 一句关于你预期的模式（Quick 还是 Deep）的注释，让 Agent 在本应是 Deep 时能指出缺失输入

子 Agent 返回一张记分卡（仅用已获取的值填充）、一份按优先级排序的发现清单、正面观察、以及主动建议。

## 输出

把完整审计报告返回给用户。无需综合或合并，这是单 persona 命令。