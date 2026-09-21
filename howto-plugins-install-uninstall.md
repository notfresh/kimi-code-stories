---
title: "Kimi Code 怎么安装和卸载插件"
date: 2026-09-21
author: notfresh
status: recorded
tags: [kimi-code, plugins, howto, cli]
---

> 这篇面向"已经知道插件是什么、就想把它装上/卸掉"的用户。覆盖 Kimi Code 的 `/plugins` 面板、`/plugins install` 命令行、GitHub 直装、自定义 marketplace，以及配套的卸载、`/reload` 流程、shell `PATH` 问题。
>
> 信息来源：[Plugins | Kimi Code Docs](https://www.kimi.com/code/docs/en/kimi-code-cli/customization/plugins.html)。

---

## 第一部分：安装

### 一、装之前要知道的两件事

1. **插件是什么**。在 Kimi Code 里，一个插件就是一个**带 manifest 的目录或 zip**。它能给你塞 Agent Skills、slash 命令、自定义 agent、自动加载的 skill、系统提示词，以及 MCP 服务器。
2. **插件装哪**。无论你用什么方式装，Kimi 都会把插件**拷贝**到 `$KIMI_CODE_HOME/plugins/managed/<id>/`，从此 CLI 只跑这份 managed 副本——你修改原始源目录对它没影响，要重新装一份才会更新。

### 二、装插件的三种姿势

#### 1. 交互式：TUI 里 `/plugins` 打开面板

聊天框里输入：

```/plugins
```

进入单页面板，4 个 tab，`Tab` / `Shift-Tab` 切换：

| Tab | 内容 |
|---|---|
| **Installed** | 已装插件，可启用/禁用/移除/重载 |
| **Official** | Kimi 自营的官方插件市场 |
| **Curated** | Kimi 合作伙伴提供的第三方插件 |
| **Custom** | 从 URL 安装（自定义源） |

面板里的常用快捷键（都在 Installed tab）：

- `Space` — 启用/禁用选中的插件
- `D` — 移除选中的插件（需要二次确认）
- `R` — 重新加载 `installed.json` 和所有 manifest
- `M` — 管理该插件声明的 MCP 服务器
- `Enter` — Installed tab：有版本就更新，没版本看详情；Official/Curated tab：装/更新；Custom tab：装
- `I` — 查看详情（Installed tab）
- `Esc` — 返回/取消

#### 2. 命令式：slash 命令一行搞定

| 命令 | 作用 |
|---|---|
| `/plugins list` | 列出已装插件 |
| `/plugins install <path-or-url>` | 从本地目录、zip URL 或 GitHub URL 安装 |
| `/plugins marketplace [source]` | 浏览官方 marketplace，或传自定义 marketplace 路径 |
| `/plugins info <id>` | 查看插件详情和诊断 |
| `/plugins enable <id>` | 启用 |
| `/plugins disable <id>` | 禁用 |
| `/plugins remove <id>` | 移除（需要确认） |
| `/plugins reload` | 重新加载 `installed.json` 和所有 manifest |
| `/plugins mcp enable <id> <server>` | 启用插件声明的 MCP 服务器 |
| `/plugins mcp disable <id> <server>` | 禁用 |

最常用的一条就是：

```/plugins install <path-or-url>
```

`<path-or-url>` 可以是：

- 本地目录路径，例如 `/Users/me/projects/my-plugin`
- 一个 zip 的 URL，例如 `https://example.com/my-plugin.zip`
- GitHub URL，Kimi 支持四种：
  - `https://github.com/<owner>/<repo>` — 装最新 release；没 release 就回退到默认分支
  - `https://github.com/<owner>/<repo>/tree/<ref>` — 装指定 branch / tag / short SHA
  - `https://github.com/<owner>/<repo>/releases/tag/<tag>` — 钉到指定 tag
  - `https://github.com/<owner>/<repo>/commit/<sha>` — 钉到指定 commit

**网络细节**：Kimi 只走 `github.com` 的重定向和 `codeload.github.com` 的下载，**不会**调 `api.github.com`。这点对内网/受限网络环境很重要。

#### 3. 自定义 marketplace

如果你有内部插件目录（比如公司所有团队的插件索引），可以：

```
/plugins marketplace <我的 marketplace.json 路径或 URL>
```

或设环境变量 `KIMI_CODE_PLUGIN_MARKETPLACE_URL` 覆盖默认 catalog。marketplace JSON 形如：

```json
{
  "version": "2",
  "plugins": [
    { "id": "my-plugin", "displayName": "My Plugin", "source": "./my-plugin" }
  ]
}
```

### 三、manifest 长什么样

插件根目录里需要一个 `kimi.plugin.json`（或 `.kimi-plugin/plugin.json`，两者都存在时 `kimi.plugin.json` 优先）：

```json
{
  "name": "kimi-finance",
  "version": "1.0.0",
  "description": "Finance data and analysis workflows for Kimi Code CLI",
  "skills": "./skills/",
  "systemPromptPath": "./SYSTEM.md",
  "sessionStart": { "skill": "using-finance" },
  "interface": {
    "displayName": "Kimi Finance",
    "shortDescription": "Market data and financial analysis workflows"
  }
}
```

字段速查：

| 字段 | 用途 |
|---|---|
| `name` | 必填，插件 id。规则：`[a-z0-9][a-z0-9_-]{0,63}` |
| `version` / `keywords` / `author` / `homepage` / `license` | 展示用 |
| `interface` | 在 `/plugins` 面板里展示的名字、简介、详情、开发者名、官网 |
| `skills` | 一个或多个 `./` 路径，指向插件里的 Skill 根目录 |
| `agents` | 一个或多个 `./` 路径，指向 agent 文件目录（默认自动发现 `agents/`） |
| `sessionStart.skill` | 新会话/恢复会话时，自动加载指定 Skill 进主 Agent |
| `skillInstructions` | 当该插件的 Skill 被加载时，附加的额外指令 |
| `systemPrompt` | 插件启用时，注入主 Agent 系统提示的内联文本 |
| `systemPromptPath` | 同上，但内容放文件里（便于长文）。`systemPrompt` + 文件，各 32 KB UTF-8 上限 |
| `mcpServers` | 插件声明的 MCP 服务器，默认启用 |
| `hooks` | 同全局 hooks 的 schema，插件启用时才生效 |
| `commands` | 一个或多个 `./` 路径，指向 slash 命令目录或单个 `.md` 文件 |

**`tools` / `apps` / `inject` / `configFile` 这些是不被支持的运行时字段**，写上去会变成 diagnostic 但不影响其它会话。

### 四、slash 命令（`commands` 字段）怎么写

`commands/` 目录里的每个 `.md` 自动变成一个 slash 命令：

- 命名空间：`<plugin-id>:<command-path>`，例如 `kimi-finance:report` 或 `kimi-finance:frontend/component`（对应 `commands/frontend/component.md`）
- frontmatter（`---` 之间）是可选元数据，正文是发给 Agent 的 prompt

例 `commands/report.md`：

```markdown
---
description: Pull and summarize a stock's latest financials
---

Pull the latest financials for $ARGUMENTS and summarize revenue, profit, and key risks.
```

聊天里执行：

```/kimi-finance:report TSLA
```

Kimi 会把 `$ARGUMENTS` 替换成 `TSLA`。如果正文里没有 `$ARGUMENTS`，你传的参数会被追加成 `ARGUMENTS: <你打的字>` 放到正文末尾。

### 五、Skill 和 sessionStart 的关系

插件 Skill 用的是同一种 `SKILL.md` 格式。三种触发方式：

1. `sessionStart.skill` — 新会话/恢复会话时自动加载（适合初始化指令、工作流规则）
2. `/skill:<name>` — 用户手动触发
3. 模型自动调用 — 跟普通 Skill 一致

不管哪种方式加载，**该插件的 `skillInstructions` 都会一起出现**。

### 六、安全模型

装插件和启插件的时候，这几条**不会**发生：

- 插件里的 command-type 工具和旧版工具运行时**不会被执行**
- 所有路径必须在插件根内（symbolic link 解析后）
- MCP 服务器要 `/reload` 或新会话才启动，且随时可以从 `/plugins` 关掉
- 坏的 manifest 或不安全路径只在该插件的 `info` 诊断里出现，不影响其它会话

### 七、安装常见的两类坑

1. **改完插件代码没生效**。Kimi 跑的是 managed 副本，你改了原始目录不会自动同步。重新跑一次 `/plugins install` 或在面板里 `Enter` 触发更新。
2. **改完 manifest 没生效**。跑 `/plugins reload`，或在当前会话跑 `/new` 起个新会话。`sessionStart.skill`、`systemPrompt`、`systemPromptPath` 文件内容这些都只在 reload / 新会话时刷新。

### 八、安装流程图

```
用户敲：/plugins install <path-or-url>
        │
        ▼ Kimi 解析来源（local 目录 / zip URL / GitHub URL）
        │
        ▼
   复制到 $KIMI_CODE_HOME/plugins/managed/<id>/
        │
        ▼
   读 kimi.plugin.json（或 .kimi-plugin/plugin.json）
        │
        ▼
   注册 skills / agents / sessionStart / systemPrompt / hooks / mcpServers / commands
        │
        ▼
   提示用户："Plugin changes apply after /reload or in new sessions"
        │
        ▼
   用户跑 /reload（当前会话）或 /new（起新会话）— 真正生效
```

---

## 第二部分：卸载

### 一、官方卸载命令：有，且很简单

跟安装一样，Kimi 给的是 `/plugins` 面板 + 一组 slash 命令。

#### 1. 命令行一行

| 命令 | 作用 |
|---|---|
| `/plugins remove <id>` | **移除插件**（会要二次确认） |
| `/plugins disable <id>` | 禁用（不卸载，以后可重新启用） |
| `/plugins enable <id>` | 启用已装但禁用的 |
| `/plugins list` | 列出当前已装，确认 `<id>` |
| `/plugins info <id>` | 看插件详情和诊断 |
| `/plugins reload` | 重新加载 `installed.json` 和所有 manifest |

最常用：

```/plugins remove axgraph
```

确认一次，完成。

#### 2. TUI 面板里卸载

```/plugins
```

进入面板后：

1. `Tab` 切到 **Installed**
2. 方向键选要卸的插件
3. 按 **`D`** — 触发移除（会问一次确认）
4. 卸完按 **`R`** 重新加载 manifest（可选，见下文）

附：面板里其他常用键

- `Space` — 启用/禁用选中项（不卸载）
- `M` — 管理该插件的 MCP 服务器
- `Enter` — Installed tab 里，有版本就更新，没版本就查看详情
- `I` — 查看详情
- `Esc` — 返回/取消

### 二、`remove` vs `disable`：别用错

| 想做什么 | 用哪个 |
|---|---|
| 临时停一下，以后还想用 | `/plugins disable <id>` |
| 真的不想要了，以后要从头装 | `/plugins remove <id>` |

`disable` 不会删文件、不会动 `installed.json`，只是把插件设成 inactive，以后 `/plugins enable <id>` 就能恢复。

`remove` 会把安装记录和 managed 副本一起清掉（详见第三节）。

### 三、`/plugins remove` 到底删了什么

官方文档原话：

> "Removing a plugin only deletes the installation record; the managed copy and original source files remain on disk."

所以：`/plugins remove` 之后，

- ✅ 没了：`$KIMI_CODE_HOME/plugins/managed/<id>/` 这个 managed 副本、`installed.json` 里的注册记录、宿主对插件的引用。
- ❌ **没动**：你的原始源码目录（比如 `C:\projects\axgraph`、GitHub clone 下来的目录）、磁盘上其它相关文件、其它项目里残留的插件生成物。

> 如果你还要**彻底清盘**（managed 副本、原始来源目录、`$KIMI_CODE_HOME/plugins/managed/` 整个目录、或环境变量 `KIMI_CODE_PLUGIN_MARKETPLACE_URL`），那是另一回事，跟 `/plugins remove` 是**两个独立动作**。

### 四、卸载之后：必须 refresh

官方文档明确说：

> "Plugin changes apply after `/reload` or in new sessions. After installing, enabling/disabling, or removing a plugin, run `/reload` or `/new`; the current session will not update."

卸载之后，系统提示词、Skill 列表、agent 列表、MCP 服务器、hooks 这些都要重 build 才不会带旧插件的痕迹。所以：

- 想**立刻生效**，在当前会话里跑：`/reload`
- 想**重新干净开始**，跑：`/new`

不然当前会话还会带着被卸插件的 system prompt、Skill、MCP，看上去"没卸干净"。

`/reload` 之后会发生什么（同样适用于装/启用/禁用/移除）：

- 刷新 `installed.json` 和所有 manifest
- 触发 agent 列表重 build
- MCP 服务器按启用状态重新启动
- hooks 按新启用状态重新挂载
- `systemPrompt` / `systemPromptPath` 在下一次 prompt rebuild 时被替换

**v2 引擎上**（`/reload` 或 prompt rebuild 后），变化立刻生效。**老引擎**保持快照直到 `/reload` 或新会话。

### 五、卸载常见套路完整版

#### 场景 A：临时停一下 axgraph，改天还要用

```/plugins disable axgraph
```

完成。当前会话立刻不带 axgraph 的 system prompt。

#### 场景 B：永远不要 axgraph 了

```/plugins remove axgraph
```

确认 → 卸完跑：

```
/reload
```

或开新会话 `/new`。

#### 场景 C：卸完之后还想删干净磁盘上的 managed 副本和源码

```/plugins remove axgraph
```

然后**手动**清：

- 删除 managed 副本（如果系统提示 `installed copy remains on disk`，你想清的话）：

  - Windows (PowerShell):
    ```powershell
    Remove-Item -Recurse -Force "$env:KIMI_CODE_HOME\plugins\managed\axgraph"
    ```
  - Linux/macOS:
    ```bash
    rm -rf "$KIMI_CODE_HOME/plugins/managed/axgraph"
    ```

- 删除原始源码目录（比如你 clone 下来的 `C:\projects\axgraph` 或 `~/work/my-plugin`）。这一步**跟 `/plugins remove` 完全独立**——文档没说 remove 会替你做这事。

#### 场景 D：集体卸载（批量）

文档没给专门的"卸载所有插件"命令。要批量卸，在面板里：

1. `/plugins` 进 Installed tab
2. 方向键选第一个，按 `D` 确认 → 卸
3. 重复，直到空

或循环，手动对每个 id 跑 `/plugins remove <id>`。

### 六、卸载流程图

```
用户：/plugins remove axgraph
        │
        ▼ Kimi 弹出确认
        │
   用户：y
        │
        ▼
   删除 installed.json 里的 axgraph 记录
        │
        ▼
   删掉 $KIMI_CODE_HOME/plugins/managed/axgraph/
        │
        ▼
   提示："changes apply after /reload or in new sessions"
        │
        ▼
   用户：/reload   （或 /new）
        │
        ▼
   系统重 build：
     - system prompt 去掉 axgraph 段
     - Skill 列表去掉 axgraph/SKILL.md
     - agent 列表重算
     - MCP 服务器按状态启停
     - hooks 按启用状态挂载
        │
        ▼
   axgraph 在当前进程里彻底消失
```

### 七、踩坑提醒

1. **卸完没立刻见效**。没跑 `/reload` 或 `/new`，系统提示词、Skill 列表还是带旧插件。官方原话："the current session will not update"。
2. **想"撤回"卸载**。`/plugins remove` 不是软删除——官方没说会进回收站，managed 副本和原始源码是否还在磁盘上，看上一节。后悔了重新 `/plugins install` 同源即可。
3. **手动 `rm -rf`**。别绕过 `/plugins remove` 直接 `rm -rf $KIMI_CODE_HOME/plugins/managed/<id>/`——那样 `installed.json` 里还会留着记录，reload 会跟文件系统状态不一致，下次面板里显示成"幽灵已装"。
4. **MCP 服务器**。如果你卸的插件声明了 MCP，卸载 + `/reload` 后 MCP 进程会被关掉；`/plugins mcp disable <id> <server>` 只是关单台 MCP，不卸插件。
5. **误以为是浏览器插件**。Kimi Browser Extension 和 Kimi Computer Use 这两个官方插件有额外步骤：前者还要在浏览器里装扩展，后者 macOS 上要授权 Accessibility 和 Screen Recording。卸载时同理——`/plugins remove` 只卸 Kimi 这边，浏览器/系统权限那边是独立的事。

---

## 第三部分：案例——以 axgraph 为例把上述流程走一遍

> 适用读者：已经读完了第一/二部分，想看 axgraph 这个具体插件是怎么被 Kimi 装/卸，以及 `bin/ax` 的可达性是怎么解决的。axgraph 仓库本身就在 `C:\projects\axgraph\`，所以下面给的所有文件路径、字段值、行号都可以直接对照阅读。

### 一、源是什么：`/plugins install <axgraph 目录>` 时 Kimi 收到了什么

axgraph 是一个**本地目录插件**（`kimi.plugin.json` 在仓库根）。装它的命令，在 Kimi 聊天框里就是：

```/plugins install /c/projects/axgraph
```

（或在 `/plugins` 面板的 Custom tab 里选本地目录）

Kimi 在这一步做的事情（对应第一部分 §二的"命令式"）：

1. 校验源存在、是目录。
2. **整目录复制**到 `$KIMI_CODE_HOME/plugins/managed/axgraph/`。这是 managed 副本，从此 CLI 只跑这份——改原始 `C:\projects\axgraph\` 不会影响已装的 axgraph，要重装一次。
3. 读 manifest，注册字段（下一节）。
4. 提示 "Plugin changes apply after `/reload` or in new sessions"。

> 注意：`kimi.plugins.install <path>` 在 Kimi 文档里的 `<path-or-url>` 形式包含 local 目录、zip URL、GitHub URL。axgraph 用的是 local 目录形式，这是最直接的。

### 二、`kimi.plugin.json` 字段，Kimi 怎么处理每一个

`C:\projects\axgraph\kimi.plugin.json` 是 axgraph 的 manifest（共 42 行）。Kimi 装的时候**只**读这个文件（`.root/kimi.plugin.json` 比 `.kimi-plugin/plugin.json` 优先，axgraph 用的是前者）。

| 字段 | axgraph 的值 | Kimi 怎么处理 |
|---|---|---|
| `name` | `"axgraph"` | 变成插件 id。装好后落在 `managed/axgraph/`（以 name 命名） |
| `version` | `"1.1.3"` | 装/更新时面板用来对比版本。`installed.json` 里也存一份 |
| `description` / `author` / `license` / `homepage` / `keywords` | 字符串 | 仅展示，不影响行为 |
| `skills` | `"./skills/"` | **关键**：递归扫这下面的 `SKILL.md` |
| `commands` | `"./commands/"` | **关键**：递归扫 `.md`，每个注册成 slash 命令 |
| `sessionStart.skill` | `"using-axgraph"` | **关键**：新会话启动时，自动把 `skills/using-axgraph/SKILL.md` 注入主 Agent |
| `interface` | displayName / shortDescription / longDescription / developerName / websiteURL | 面板展示 |
| `pythonDependencies` | tomli 可选依赖 | **Kimi 不替你 pip install**，仅展示，文档提示用户 |

**axgraph 没声明的字段**（对应 Kimi 默认行为）：

- `systemPrompt` / `systemPromptPath` — 没声明 → 0 字节内联系统提示注入
- `mcpServers` — 没声明 → 无 MCP 服务器
- `agents` — 没声明 → 不自动发现 `agents/` 目录
- `tools` / `apps` / `inject` / `configFile` — **运行时不被支持**，即使写也会变 diagnostic 并被忽略

### 三、`hooks/hooks.json` 注册了什么

`kimi.plugin.json` 里**没有** `hooks` 字段。axgraph 的 hooks 走单独的 `hooks/hooks.json`：

```json
{
  "hooks": [
    {
      "event": "SessionStart",
      "matcher": "*",
      "command": "node ./hooks/session-start.mjs",
      "timeout": 5
    }
  ]
}
```

Kimi 在装插件时**会**自动读这个文件（对应官方文档 §"Hooks in Plugins"：hook 配置既可以写在 `kimi.plugin.json` 里，也可以写在单独的 `hooks/hooks.json` 里，axgraph 用后者）。字段语义：

| 字段 | 值 | 含义 |
|---|---|---|
| `event` | `"SessionStart"` | 新会话/恢复会话时触发 |
| `matcher` | `"*"` | 匹配所有 session |
| `command` | `"node ./hooks/session-start.mjs"` | **相对插件根**；Kimi 在调 hook 时 cwd 设成插件根 |
| `timeout` | `5` | 5 秒后超时 |

Kimi 调用 hook 时，会注入两个环境变量：

- `KIMI_CODE_HOME` — 用户的 Kimi 主目录
- `KIMI_PLUGIN_ROOT` — 插件根，绝对路径。`./hooks/session-start.mjs` 就是相对它解析的

> 官方原话："Each hook runs with its working directory set to the plugin root, so `command` can use `./` paths inside the plugin."

### 四、`skills/` 扫描结果

`skills: "./skills/"` 指向 `skills/` 目录。Kimi 递归扫里面的 `SKILL.md`（每个子目录一个 Skill）。axgraph 的 `skills/` 目录长这样：

```
skills/
└── using-axgraph/
    └── SKILL.md
```

所以装好后 Skill 表里多出 `using-axgraph`。它有三种触发方式：

1. **`sessionStart.skill: "using-axgraph"` → 自动注入** —— 新会话/恢复会话时，Kimi 主动把这个 Skill 注入主 Agent 的 system context。这就是为什么每个新会话开始，模型都能看到 "Plugin 'axgraph' was loaded" 这种 SKILL.md 注入的内容（对应第一部分 §五）。
2. 用户手敲 `/skill:using-axgraph` 显式加载。
3. 模型自动调用（跟普通 Skill 一样）。

**Skill 注入流程**（对应 `hooks/session-start.mjs:28-42`）：

```
新会话启动
    │ Kimi 读 installed.json，发现 axgraph enabled，sessionStart.skill = "using-axgraph"
    ▼ 触发 SessionStart 事件
    │ Kimi 调 hooks[1].command：node ./hooks/session-start.mjs
    │ cwd = KIMI_PLUGIN_ROOT
    │
    ▼ session-start.mjs 读 ../skills/using-axgraph/SKILL.md
    │ 包成 {hookSpecificOutput: {hookEventName: "SessionStart", additionalContext: "<banner + SKILL.md>"}}
    │ stdout 输出
    │
    ▼ Kimi 把 additionalContext 追加进主 Agent 的 system prompt
```

### 五、`commands/` 注册结果

`commands: "./commands/"` 指向 `commands/` 目录，Kimi 递归扫 `.md`，按文件路径推出命令名（去掉 `.md`，用 `/` 分隔）。命名空间前缀是插件 id，所以：

| 文件 | 推出的命令 |
|---|---|
| `commands/install.md` | `/axgraph:install` |
| `commands/init.md` | `/axgraph:init` |
| `commands/query.md` | `/axgraph:query` |
| `commands/purity.md` | `/axgraph:purity` |
| `commands/diagnose.md` | `/axgraph:diagnose` |

每个 `.md` 还可以有 frontmatter（`---` 之间）覆盖默认名/描述。axgraph 的 5 个命令文件里至少有 `Install` 一个明确写了 `description`（在 frontmatter 里）：

```markdown
---
description: Install `ax` onto the user's shell PATH so it works from any terminal (POSIX: symlink via install.sh; Windows: shim via install.ps1).
---
```

### 六、`bin/ax` 的处置：Kimi 装完没动它

这是本节最反直觉的事实，专门点出来。

`bin/ax` 是 axgraph 仓库根下的 Python wrapper（327 行）。**Kimi 装插件时对它零处理**——只是被原样复制到 `managed/axgraph/bin/ax`，然后**永远不会被 Kimi 自动调用**。理由：

- Kimi 插件机制只懂 manifest 字段（skills/commands/sessionStart/systemPrompt/mcpServers/hooks/agents），不识别二进制/脚本。
- 文档明确说"`tools` / `apps` / `inject` / `configFile` 是 unsupported runtime fields，会出现 diagnostic 并被忽略"——就算 manifest 写了 "tools include bin/ax"，Kimi 也不理。

`bin/ax` 在 managed 副本里静静躺着。可达性靠两条路径：

#### 路径 A — 用户从 shell 主动跑 `ax`（靠 `/axgraph:install` slash 命令）

`/axgraph:install` 这条 slash 命令的文档是 `commands/install.md`（153 行），它教模型做这件事：

1. **判 OS**：Windows → 跑 `install.ps1`；POSIX → 跑 `install.sh`。
2. **POSIX 路径**：软链 `~/.local/bin/ax → <axgraph>/bin/ax`；检查 `~/.local/bin` 在不在 `$PATH`，不在就**打印** `export PATH=...`（不自动改 `~/.bashrc`）。
3. **Windows 路径**：生成 `%USERPROFILE%\bin\ax.cmd` shim，内容是：

   ```cmd
   @echo off
   set "AX_GRAPH_ROOT=<managed>\axgraph"
   python "%AX_GRAPH_ROOT%\bin\ax" %*
   ```

   并把 `%USERPROFILE%\bin` 追加到 user-level PATH（`SetEnvironmentVariable`），大小写无关去重。
4. **告诉用户结果**：软链位置、PATH 状态、卸载命令。

`bin/ax` 自己有三段 fallback 找插件根（`bin/ax:38-59`）：

1. `$AX_GRAPH_ROOT` 环境变量——`install.ps1` 的 `ax.cmd` shim 里写死了这个变量，所以 Windows 用户通过 shim 调 `ax` 时，插件根是显式给的。
2. 从 `__file__` 往上 1~3 层找 `lib/graph_query.py`——处理 symlink、deep install 的情况（用户在 macOS/Linux 软链 `~/.local/bin/ax → <managed>/axgraph/bin/ax` 时用）。
3. hard fail，提示用户设 `$AX_GRAPH_ROOT`。

> 软链 POSIX / shim Windows，都是为了一个目的：**用户敲 `ax` 时，shell 能找到 wrapper**。一旦找到，wrapper 内部的 `_resolve_plugin_root()` 反向回溯插件根，**不需要软链留在原位**（因为 `__file__` 解析是 realpath，跟软链无关）。

#### 路径 B — 模型通过 slash 命令 prompt 主动跑 `ax`

`/axgraph:query`、`/axgraph:purity`、`/axgraph:diagnose` 这三条 slash 命令的 prompt 引导模型用 Bash 工具跑 `ax <subcommand> ...`，然后把 stdout 拿回来给用户。也就是说：

> **slash 命令的真正执行点不是 Kimi，而是 Bash 工具 + `bin/ax`**。Kimi 提供了 slash 命令的入口（`/axgraph:*`），但执行最终落在用户 shell 里的 `bin/ax` 进程上。

这是为什么 `/axgraph:install` 是必须的——如果用户没装 `bin/ax` 到 PATH，模型调 `Bash("ax query ...")` 时 shell 找不到 `ax`，slash 命令就废了。

### 七、完整调用链：从 `/plugins install` 到用户在会话里敲 `ax query`

```
用户在 Kimi 聊天框里：/plugins install /c/projects/axgraph
        │
        ▼ Kimi 解析源，复制到 managed/axgraph/
        │
        ▼ 读 kimi.plugin.json，注册字段
        │
        ▼ 读 hooks/hooks.json，注册 hooks
        │
        ▼ 扫 skills/，列 SKILL.md
        │
        ▼ 扫 commands/，列 slash 命令
        │
        ▼ 写 installed.json（id=axgraph, enabled=true, sessionStart.skill=using-axgraph）
        │
        ▼ 提示："Plugin changes apply after /reload or in new sessions"

用户跑 /reload  （或 /new）
        │
        ▼ 新会话启动 / 当前会话 reload
        │
        ▼ Kimi 触发 SessionStart 事件
        │
        ▼ 调 hook：node ./hooks/session-start.mjs
        │     cwd = KIMI_PLUGIN_ROOT（managed/axgraph）
        │     env = KIMI_CODE_HOME + KIMI_PLUGIN_ROOT
        │
        ▼ session-start.mjs 读 ../skills/using-axgraph/SKILL.md
        │     包成 {hookSpecificOutput: {additionalContext: ...}}
        │     stdout 输出
        │
        ▼ Kimi 把 SKILL.md 内容追加进 system prompt
        │
        ▼ 模型"认识" axgraph
        │
用户跑 /axgraph:query MOD.agent-claude-skill-utils
        │
        ▼ Kimi 读 commands/query.md 的 prompt
        │
        ▼ 模型按 prompt 决定调 Bash 工具
        │
        ▼ Bash("ax query MOD.agent-claude-skill-utils")
        │
        ▼ shell 找 $PATH 里的 ax （假设之前跑过 /axgraph:install）
        │     路径 A：POSIX 软链  ~/.local/bin/ax → <axgraph>/bin/ax
        │             或 Windows shim %USERPROFILE%\bin\ax.cmd
        │
        ▼ 找到 bin/ax，Python wrapper 启动
        │     1. _resolve_plugin_root()：优先 $AX_GRAPH_ROOT，否则从 __file__ 往上找
        │     2. _forward_to_lib("query")： 查表得到 graph_query.py
        │     3. _load_module()：            importlib 直加载，不 shell out
        │
        ▼ graph_query.main() 跑查询，读 .axgraph/ 下的 TOML
        │
        ▼ stdout 输出节点 + 边
        │
        ▼ 模型拿到 stdout，解读给用户
```

### 八、装/卸 axgraph 这边的完整命令清单

```
# 装
/plugins install /c/projects/axgraph
/reload                              # 或 /new

# 卸
/plugins remove axgraph              # 删 installed.json + managed 副本
/reload                              # 或 /new

# 仅禁用
/plugins disable axgraph             # 改 enabled=false，不删文件

# 仅启用
/plugins enable axgraph              # 反过来

# shell PATH 装 wrapper（bin/ax）
/axgraph:install                     # 引导用户跑 install.sh / install.ps1

# 卸载 wrapper
rm ~/.local/bin/ax                   # POSIX
Remove-Item "$env:USERPROFILE\bin\ax.cmd"   # Windows
```

---

**资料来源**：[Plugins | Kimi Code Docs](https://www.kimi.com/code/docs/en/kimi-code-cli/customization/plugins.html)。本文所有命令、行为、限制都直接抄自该页。axgraph 部分的字段值、文件路径、行号直接抄自 `C:\projects\axgraph\` 仓库。