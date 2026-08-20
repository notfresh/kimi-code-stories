---
story_id: 3
title: 通用 Agent 和 Coding Agent 到底差在哪？——一场从闲聊到源码实测的讨论
date: 2026-08-20
author: notfresh
status: recorded
upstream_repo: MoonshotAI/kimi-code
tags: [kimi-code, hermes, 通用agent, coding-agent, 对比, 架构]
---

# Story 3: 通用 Agent 和 Coding Agent 到底差在哪？

## 起因

这天晚上从一句闲聊开始："大模型厉害就厉害在大上"——信息都知道、不会乱、还能产生巧妙的关联。聊着聊着，话题从大模型本身滑到了产品形态：Hermes 也是一个"大而全"的产品，各种信息打通；而很多 coding agent 有 workspace 概念，记忆不共享、session 不共享，入口还要先打开工作区。于是一场关于"通用 Agent vs Coding Agent"的完整讨论就此展开——翻遍历史会话，这个主题之前从来没有系统聊过，这是第一轮正式交锋。

## 讨论怎么走的（六幕）

### 第一幕：大模型为什么厉害

我的直觉是"大"。被拆成了四层：

1. **"大"有实证**：规模定律（Kaplan 2020 / Chinchilla 2022）——损失随参数量、数据量、算力呈幂律下降，不是玄学。
2. **"不会乱"被反驳了**：幻觉、对抗样本、上下文污染都说明它的稳定是统计性的，不是逻辑性的。它像一个博闻强记但偶尔自信出错的人，不是一台永不故障的机器。
3. **"巧妙关联"的机制**：Hinton 的"压缩即理解"——预测 next token 迫使模型把海量文本压成一套生成规则；分布式表征让"关联"不是检索，而是高维向量空间里的几何运算（king − man + woman ≈ queen），自注意力又让任意两个词直接可达。
4. **补刀"大 ≠ 懂"**：它知道的只是语料里的相关性，没吃过苹果就不知道甜味；数据质量可以部分替代规模（phi 系列小模型同样惊艳）。

遗留一个开放问题：压缩算"理解"，还是只算"像理解"？

### 第二幕：从模型到产品——Hermes vs coding agent

我提出 Hermes 是"大而全"、信息打通，coding agent 则被 workspace 框住。结论是三点：

- **记忆载体差异是本质**：Hermes 是系统级记忆（agent 自主判断写入 memory store、每轮注入、session_search 全文检索历史会话）；coding agent 是文件级记忆（CLAUDE.md 项目局部、要显式写）。一个记忆是 agent 自己的器官，一个只是项目目录里的附件。
- **入口形态差异**：coding agent 是嵌入式（先 cd 进项目它才认得路），Hermes 是环绕式（gateway 常驻，飞书里喊一声就来，cron 到点主动干活）。
- **一个反驳**：workspace 隔离是特性不是缺陷——代码库上下文极其庞大，必须要有边界；而全局记忆也有代价（Hermes 的 memory 只有 2200 字符上限，逼着每条都精炼）。

一句话定位：**coding agent 是工具，通用 agent 是伙伴。**

### 第三幕：我为什么偏爱 Hermes

"我一般多用 Hermes，因为它更方便。"——方便是决定性的一票：我的任务（学习、投研、日常管理）大多是对话型的，天然适合随时喊一嘴的形态；代码任务占比低，coding agent 的工作区流程就成了多余摩擦。但方便的另一面是依赖：gateway 挂了就抓瞎。所以结论是"Hermes 主力、coding agent 备用"。

### 第四幕：未来会双向趋同

我的预测是"记忆可以渐进式载入，未来 coding agent 会学通用 agent"。被修正了一半：

- **对的部分**：记忆分层是必经之路（核心常驻 + 按需加载 + 可检索历史，对应认知科学的工作记忆/长时记忆模型）。
- **修正的部分**：coding agent 不是不会渐进式，是记忆对象本质不同——代码库是**封闭可穷尽的世界**（一个仓库就那么大，窗口够大就能全量塞进去，错误代价高所以宁可冗余）；用户的生活是**开放无边界的**（永远不知道下一步问什么，只能分层）。一个为省成本，一个不得已。
- **coding agent 真正该学通用 agent 的三样**：自主记忆写入（agent 自己判断什么值得记）、跨项目长期记忆（A 项目踩的坑 B 项目自动知道）、常驻（gateway + cron，而不是等用户打开）。其实 Claude Code 已经在学了：全局 CLAUDE.md、跨项目 skills 目录、headless 模式。
- **反向也在发生**：Hermes 也有 workdir、项目上下文注入（AGENTS.md）。最终形态是**带边界的通用 agent**——常驻、有长期记忆，需要时能整体进入某个项目的完整上下文。

### 第五幕：翻旧账

问"之前聊过吗"，答案是：没有专门聊过。但有四块碎片：8/12 harness 概念与任务拆分三档强度、8/13 Harness 岗突围（Agent = Model + Harness）、8/16 Pi 的 harness 研究、当天上午"agent-core 为何像第三方库"（通用 agent 引擎 vs CLI 宿主）。

### 第六幕：拿源码实测一份对比稿

后来拿到一份现成的对比稿（专才 vs 多面手），逐条用源码验证，发现三处问题——见下一节。

## 源码实证（本轮讨论最有价值的部分）

### 1. "Kimi Code 支持多模态输入"——属实 ✅

`packages/agent-core/src/agent/turn/media-resolve.ts` 明确处理 `video_url`：粘贴进 prompt 的视频会解析成最终投递形式（上传给 provider 的 `ms://` 引用），失败则退化成 `<video path="...">` 标签，由模型用 `ReadMediaFile` 工具在回合内读取。配套还有剪贴板图片提示（`apps/kimi-code/src/tui/controllers/clipboard-image-hint.ts`）和图片缩略图组件（`apps/kimi-code/src/tui/components/media/image-thumbnail.ts`）。"根据视频/设计稿生成代码"不是宣传话术。

### 2. "Kimi Code 基于 Kimi K2.5/K3 模型"——不准确 ❌

kimi-code 是**模型无关**的：`packages/kosong` 是 LLM/provider 抽象层，任何 OpenAI 兼容 provider 都能接。最硬的证据是自己的使用经历——我在 kimi-code 的 config.toml 里配过 `/mm = "/model MiniMax-M3 --provider minimax-cn"`，DeepSeek 也接过。准确说法是"**默认主推** Kimi 模型，但可接第三方 provider"。Claude Code 同理（主推 Claude，但 ANTHROPIC_BASE_URL 可接第三方）。

### 3. "Hermes 支持微信入口"——是企业微信 ❌

Hermes 源码是 `plugins/platforms/wecom/`（**WeCom 企业微信**），不是个人微信。官方口径是 "Telegram, Discord, Slack, and ~20 other platforms"（含飞书、企业微信等），个人微信不在列。

### 4. 架构理念的一个修正 ✏️

"编程语言图灵完备 → coding agent 能解决任何可计算问题"——这个推理链有漏洞：图灵完备论证的是"能计算"，而通用 agent 的瓶颈从来不在"能算"，在**能感知（工具/数据/世界接口）和能行动（执行/验证）**。编程的真正优势是**代码执行 = 可验证的行动闭环**（写 → 跑 → 看结果 → 改），这是唯一能自动验证自己行为的领域——这才是 coding agent 作为通用 agent 最佳基座的核心理由。

## 沉淀的三条共识

1. **不同物种，不是优劣**：通用 agent 是伙伴（环绕式、常驻、长期记忆），coding agent 是工具（嵌入式、工作区、可验证闭环），形态由任务形态决定。
2. **本质差异在记忆载体**：系统级自主记忆（agent 自己写、跨会话跨项目）vs 文件级项目局部记忆（CLAUDE.md 显式维护）。
3. **未来双向趋同**：分层记忆架构是共同方向（核心常驻 + 按需加载 + 可检索），但 workspace 边界会保留——"带边界的通用 agent"。

## 感受

这场讨论的价值不在于得出结论，在于**结论被源码验证过**。对比稿里三条断言，两条被源码推翻、一条被自己的使用经历推翻——"读过源码"和"跑过源码"的区别就在这里：后者能直接给出文件级证据。另外，我从"记忆渐进式载入"这个直觉出发的预测，被修正成了更准确的"双向趋同 + 边界保留"，这种被反驳后更清晰的感觉，比一开始就对更有收获。

## 证据索引

| 断言 | 证据位置 |
|---|---|
| kimi-code 支持视频输入 | `packages/agent-core/src/agent/turn/media-resolve.ts`（video_url 处理 / ReadMediaFile 退化路径） |
| kimi-code 支持图片输入 | `apps/kimi-code/src/tui/controllers/clipboard-image-hint.ts`、`apps/kimi-code/src/tui/components/media/image-thumbnail.ts` |
| kimi-code 模型无关 | `packages/kosong`（provider 抽象层）+ 个人 config.toml 的 MiniMax/DeepSeek 配置 |
| Hermes 支持的是企业微信 | `hermes-agent-plus/plugins/platforms/wecom/`；AGENTS.md "~20 other platforms" |
| Hermes 微内核架构 | hermes-agent-plus AGENTS.md："The core is a narrow waist; capability lives at the edges" |
