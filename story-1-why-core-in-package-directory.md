---
story_id: 1
title: 为什么 agent-core 在 packages/ 目录里，看起来却像第三方包？
date: 2026-08-20
author: notfresh
status: recorded
upstream_repo: MoonshotAI/kimi-code
upstream_commit: 842e699a6 (Kimi For Coding, 初始提交)
tags: [kimi-code, monorepo, 架构, agent-core]
---

# Story 1: 为什么 agent-core 看起来像第三方包

## 问题

读 kimi-code 源码时的一个困惑：`packages/agent-core` 明明是这个仓库的**核心**，但它的 package.json、目录结构、版本管理方式，看起来都像是一个**独立的第三方库**，而不是"主程序的一部分"。为什么？

## 答案（基于源码事实）

这个"像第三方"的感觉不是错觉——它本来就是按**独立可发布库**的标准写的，而且是刻意的架构决策。证据分三层：

### 1. 形态是"发布物"的形态

`packages/agent-core/package.json` 有一套完整的、独立库才有的配置：

- 自己的版本号 `0.15.8`，与 CLI（`@moonshot-ai/kimi-code` `0.37.2`）**独立演进**
- 完整的 `exports` / `imports` 映射、`files: ["dist"]`、`keywords`、`repository`、`homepage`
- 有自己的 `CHANGELOG.md`（0.15.8 一路记到很老的版本），git 历史里有 `ci: release packages` 提交——它真实走过 changesets 发布流水线
- 内部依赖全是 `workspace:^` 指向更底层的包（`kaos`、`kosong`、`oauth`、`protocol`），发布时替换为真实版本号

### 2. 架构上刻意解耦：CLI 不允许直接碰它

依赖链（证据：`apps/kimi-code/package.json` devDependencies）：

```
apps/kimi-code (CLI/TUI)
  └─ devDependencies 里只有 @moonshot-ai/kimi-code-sdk   ← 没有 agent-core！
packages/node-sdk (kimi-code-sdk)
  └─ devDependencies 里有 @moonshot-ai/agent-core、agent-core-v2
```

`apps/kimi-code/AGENTS.md` 有硬规则：

> apps/kimi-code may only use core capabilities through `@moonshot-ai/kimi-code-sdk`.
> Do not import `@moonshot-ai/agent-core` directly in app code.

即 CLI 这个壳被强制只能通过 SDK 这个"合同层"跟引擎说话。三层架构：**壳（TUI）→ 门面（SDK）→ 引擎（agent-core）**。

### 3. 它是完整的"agent 引擎"，不是应用零件

`src/` 下：`agent`、`loop`、`session`、`skill`、`tools`、`mcp`、`plugin`、`profile`、`rpc`、`services`、`flags`、`di`…… 完整自洽，有自己的测试、自己的 typecheck、自己的 AGENTS.md，完全不需要知道 TUI 长什么样。

## 为什么这么设计

1. **内核与壳分离**：agent-core 是"通用 agent 引擎"，CLI/TUI 只是它的一个宿主。将来其他产品形态（IDE 插件、网页端、server）复用同一引擎，不必重新实现。
2. **SDK 是唯一官方门面**：对外只承诺 SDK 的 API 面，引擎内部随便改，只要 SDK 不变就不破坏消费者。`agent-core` 与 `agent-core-v2` 并存、kap-server 走 v2、老 CLI 走 v1，全靠这层隔离才能并行演进。
3. **版本独立演进**：每个包独立版本号 + changelog，changesets 自动计算依赖链，只 bump 受影响的包。

## 一个补充（容易误读的点）

`agent-core` 本身现在是 `"private": true`，**并不发布到 npm**。真正对外发布的是 `kimi-code-sdk`（`packages/node-sdk/package.json` 有 `publishConfig: { access: "public", provenance: true }`）。

所以准确说法是：它"**按第三方库的标准来写**"，但被圈在 monorepo 内部；"像第三方"是形态和纪律像，不是真的独立发布了。

## 证据索引

| 证据 | 位置 |
|---|---|
| agent-core 发布物形态（version/exports/files） | `packages/agent-core/package.json` |
| CLI 不依赖 agent-core，只依赖 sdk | `apps/kimi-code/package.json` devDependencies |
| "禁止直接 import agent-core"硬规则 | `apps/kimi-code/AGENTS.md` |
| sdk 依赖 agent-core / agent-core-v2 | `packages/node-sdk/package.json` devDependencies |
| sdk 有 public 发布配置 | `packages/node-sdk/package.json` publishConfig |
| agent-core 初始提交即存在 | git log `842e699a6` |

## 关联

- 上游仓库: https://github.com/MoonshotAI/kimi-code
- 本仓库: git@github.com:notfresh/kimi-code-stories.git
