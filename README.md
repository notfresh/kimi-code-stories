# kimi-code-stories

**读 kimi-code 源码的故事集** —— 为什么一个开源项目会这样设计，以及我读它时的真实感受。

## 为什么开这个仓库

我在学习 [MoonshotAI/kimi-code](https://github.com/MoonshotAI/kimi-code) 的源码。这是一个很大的 TypeScript monorepo，核心是 `packages/agent-core` 这个 agent 引擎——正好是我想搞懂的东西：一个 coding agent 的内部到底是怎么运转的。

读源码的过程中，我经常遇到一些"为什么"：

- 为什么这个核心包放在 `packages/` 里，却长得像一个第三方库？
- 为什么 CLI 被禁止直接 import 核心引擎？
- 为什么同时存在 v1 和 v2 两套引擎？

这些问题在文档里找不到答案，答案藏在 package.json、AGENTS.md 和 git 历史里。我不想让这些发现溜走，于是决定**把它们写成故事存下来**。

技术书解释"是什么"，这个仓库记录"为什么"和"我怎么想"。它更像一个读码手记，而不是技术文档。

## 组织方式

每个故事一个文件，命名 `story-N-<kebab-case-标题>.md`，用 YAML frontmatter 记录元数据（story 编号、日期、作者、上游锚点、标签）。

```markdown
---
story_id: 1
title: ...
date: 2026-08-20
author: notfresh
status: recorded
upstream_repo: MoonshotAI/kimi-code
tags: [...]
---
```

故事正文统一遵循"问题 → 基于源码的证据 → 结论 → 证据索引"的结构：每个结论都标得出出处，不写没验证的话。

## 已收录

| # | 标题 | 日期 |
|---|------|------|
| 1 | [为什么 agent-core 看起来像第三方包](story-1-why-core-in-package-directory.md) | 2026-08-20 |
| 2 | [为什么读 kimi-code 源码](story-2-why-read-kimi-code-source.md) | 2026-08-20 |
| 3 | [通用 Agent 和 Coding Agent 到底差在哪？](story-3-general-agent-vs-coding-agent.md) | 2026-08-20 |
| 4 | [Hermes 简史——从 7 个文件长成一座城](story-4-hermes-history.md) | 2026-08-20 |
| 5 | [开源贡献正在变难——以 alias 功能设计为例](story-5-first-upstream-contribution-attempt.md) | 2026-08-21 |
| 6 | [Kimi Code 的野望——从 68 个 Release 到 500 亿美元](story-6-kimi-code-position-and-capital.md) | 2026-08-20 |
| 7 | [How do I build an agent？——我为什么做、又怎么做自己的 coding agent](story-7-how-i-built-my-own-coding-agent.md) | 2026-08-24 |

## 关联

- 上游仓库: https://github.com/MoonshotAI/kimi-code
- 我的 kimi-code 研究仓库: https://github.com/notfresh/kimi-code-study
