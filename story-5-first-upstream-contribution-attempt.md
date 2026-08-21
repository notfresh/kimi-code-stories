---
story_id: 5
title: 开源贡献正在变难——以 alias 功能设计为例
date: 2026-08-21
author: notfresh
status: recorded
upstream_repo: MoonshotAI/kimi-code
upstream_commit: c8029f9f3 (PR #3089, 贡献政策文档对齐)
tags: [kimi-code, opensource, contribution]
---

# 开源贡献正在变难

## 经历

我给 kimi-code 做了一个用户自定义 slash 命令别名功能，编译验证通过，联系负责人，回复：**最近不接外部 feature，审核不过来**。

查证后确认：上游 CONTRIBUTING.md 在 2026-08-19 更新（PR #3089），明确**外部 PR 仅收获批准的 bug fix，外部 feature PR 一律不收**；且政策"执行在先、文档对齐在后"。

## 为什么变难

AI 代码洪水——人人都能生成 PR，维护者审核不过来，只能收紧。后果：想靠"交 PR"进入开源圈子的路变窄了；变贵的是"理解代码、能负责"的贡献。

## 也有反例：Hermes 这类项目为什么更开放

同样被组织主导，Hermes（Nous Research）对开源宽容激进得多。区别在北极星：

- 公司主导（Moonshot）：开源是手段——获客、招聘、生态，服务商业化。外部 PR 干扰路线图时就收紧。kimi-code 拒 feature 是生意逻辑。
- 研究实验室（Nous Research）：开源是目的——影响力、研究声誉、社区信任就是资产。贡献者涌入是资产不是负担。

Nous 的事实（据公开报道）：纽约开源 AI 实验室，早期 bootstrapped 自筹资金；2025年4月拿 Paradigm 领投的 $50M A 轮（JPMorgan、BlackRock、Bezos 参投）。关键：Paradigm 是 crypto/去中心化方向基金，投的就是"开源+去中心化 AI"叙事，投资人利益与开放度一致。Hermes 模型权重全开源，Hermes Agent 2026年2月以 MIT 开源发布。

个人体验对照：给 kimi-code 提 feature 被拒（产品逻辑），在 Hermes 生态写的 skill 被收下（社区逻辑）。选项目先看北极星——kimi-code 的代码是产品的一部分，Hermes 的代码是产品本身。

## 怎么办

- 按官方流程走：feature 走 issue 讨论，入口没关死。我准备了中英双语 feature request（git alias 设计、TUI 内配置、优先级、冲突检测）
- fork 自用版，当自己的微型工业界练
- 底线：个人的严谨必须保持——不能 Agentic Coding 甩手交给 AI 就提交；否则靠开源刷绩效、刷履历，依然行不通

## 证据

- PR #3089（政策）：https://github.com/MoonshotAI/kimi-code/pull/3089
- PR #2614（内置别名先例）：https://github.com/MoonshotAI/kimi-code/pull/2614
- alias-issue.md / alias-issue.zh-CN.md（研究仓库 002study/，draft）
