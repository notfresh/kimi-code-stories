---
story_id: 8
title: kimi-code 编译速记 —— 我加特性时只用三条命令
date: 2026-09-17
author: notfresh
status: recorded
upstream_repo: MoonshotAI/kimi-code
tags: [build, sea, node, vscode, developer-notes]
---

# kimi-code 编译速记 —— 我加特性时只用三条命令

> 2026-09-17，本来要给同事看，结果发现三条命令就够我开发了

## 序

事情的起因很简单：我在 kimi-code 里加了一个特性，改完了想验证一下能跑、想打个二进制看效果、想打 VSIX 给同事测。结果一头扎进 `apps/kimi-code/package.json` 和 `apps/vscode/scripts/` 里，30 多个脚本名晃得我眼晕——`build:native:sea`、`build:native:release`、`build:native:js`、`vsix-package`、`vsix-verify`、`build:plugin-marketplace`……

我想要的东西特别朴素：**改完代码，能跑、能验、偶尔打个包**。不想搞清楚 SEA 的 ELF section 怎么塞 blob，也不想知道 esbuild 和 vite 为啥要分两路打。我就是想写代码、验证代码。

于是我决定只留三条命令，其他全忘掉。这篇故事记的就是这三条，以及它们背后在我脑子里留个印象就够的原理。

## 第一章 三条命令

工作目录固定在 `apps/kimi-code/` 下——所有命令都从这里走。

```bash
# 1. 改完代码，跑一下确认没崩
pnpm run build:native:sea

# 2. 试跑产物
./dist-native/bin/linux-x64/kimi --version

# 3. 进入开发模式，改 src/ 自动重 bundle
pnpm run dev
```

就这三行。能覆盖我 90% 的场景：**加特性 → 编译验证 → 跑一下确认 → watch 模式继续开发**。

剩下 10% 是打 VSIX 给同事：

```bash
cd apps/vscode && pnpm run build && node scripts/vsix-package.mjs --target linux-x64
```

产物在 `apps/vscode/artifacts/vsix/kimi-code-linux-x64.vsix`。

## 第二章 三条命令背后在干啥（够用就行版）

我不打算深挖每个技术细节——那些 `node --experimental-sea-config`、`postject` 注入 ELF section、`codesign --options runtime` 的细节，真出问题再去翻文档。我只留"出问题知道往哪查"的印象。

### `build:native:sea` 出的是一个**单文件可执行**

以前我以为 CLI 必须装 Node 才能跑。这次才知道 kimi-code 把 Node 引擎 + JS 代码 + native assets **全打成一个文件**——这就是 SEA（Single Executable Applications），Node 22+ 自带能力。

简单说就是把 `node` 可执行文件复制一份，把 JS 代码以 blob 形式塞进去。用户拿到的就是一个 178MB 的 `kimi`，双击能跑，不用装 Node。

`build:native:sea` 是这个流程的**本地调试版**：跳过签名、跳过打包成 tarball。release 版叫 `build:native:release`，会跑签名——但本地自测用不上。

产物落在 `apps/kimi-code/dist-native/bin/<你的平台>/kimi`，比如 `linux-x64/kimi`、`darwin-arm64/kimi`。

### `pnpm run dev` 是**自动重编译**

不是"启动 kimi"，而是开一个 watch 进程。改了 `src/` 下的任何文件，自动重新 bundle 到 `dist/main.mjs`。然后我自己用 `node dist/main.mjs` 跑起来看效果。

这样比每次手动 `build:native:sea` 快了 10 倍——出二进制要 30-60 秒，出 JS bundle 只要几秒。

### 偶尔打 VSIX：esbuild + vite 各打各的

VSCode 扩展和 kimi CLI 是**两条独立的产物线**。VSCode 扩展里代码跑在两个完全不同的环境：

- **扩展主进程**：在 Node 环境里，能用 `vscode.*` API → 用 **esbuild** 打
- **webview UI**：在浏览器 iframe 里，不能用 Node → 用 **vite** 打

为啥不能合并？因为浏览器里没有 `require('vscode')`，Node 进程不需要处理 CSS。两个打包器各管一摊。

打 VSIX 就是 `build` + `vsix-package` 两步，把上面两个产物 + 静态资源 + manifest 打成 zip 重命名。

## 第三章 我没去搞清楚但遇到会查的东西

留个目录，下次真出问题知道往哪查：

| 出错现象 | 大概率在哪 |
|---|---|
| `tsc ... has errors` | 类型错，看具体文件:行号 |
| `Web assets missing` | `apps/kimi-code/dist-web/` 不在——web UI 是从 code-app 仓同步过来的，本仓不编 |
| `pnpm install` 报 Node 版本 | 引擎 strict，要 `>=24.15.0` |
| SEA 二进制跑崩 | 多半是 native dep 没注册到 `scripts/native/native-deps.mjs` |
| VSIX 装上行为不对 | webview 缓存——`F1 → Developer: Reload Window` |

SEA 的 ELF 注入、codesign 签名细节、Vite 的 chunk 拆分策略——这些我都没去深挖。等真出问题再翻 `scripts/native/build.mjs` 和 `apps/vscode/scripts/vsix-package.mjs`。

## 收尾

回头看这件事，我学到的不是"怎么编译 kimi-code"，而是 **"够用就好"** 这件事。

`apps/kimi-code/package.json` 里塞了 13 个 build 相关脚本、`apps/vscode/scripts/` 里 5 个 mjs、`scripts/native/` 里 14 个工具脚本——加起来 30+ 个工具。设计者必然需要这些（要支持 6 个 target、要签名、要验证、要 notarize、要打 marketplace），但**对加特性的开发者来说，只需要 3 条**。

这和读代码一个道理：第一次读源码要追到底，第二次读要明白边界，第三次读要知道**哪些不用看**。

我把这三条命令记下来，发给同事也存一份。下次再有人问"kimi-code 怎么编译"，就回一句：

> 三条命令，剩下有问题再说。

---

## 证据索引

| 断言 | 出处 |
|------|------|
| `build:native:sea` 输出在 `dist-native/bin/<triple>/kimi` | `apps/kimi-code/package.json:39` scripts 段 + `scripts/native/paths.mjs` |
| SEA 跳过签名 | `apps/kimi-code/scripts/native/build.mjs:55-58`（`identity = profile === 'release' ? ... : '-'`） |
| esbuild 打扩展主进程 / vite 打 webview | `apps/vscode/package.json` scripts 段（`build:extension` esbuild + `build:webview` vite） |
| web UI 不在本仓编译 | 仓库根 `AGENTS.md` "Project Map" 段 |
| Node 版本要求 `>=24.15.0` | `apps/kimi-code/scripts/native/build.mjs:24-30` 强制校验 |
| 6 个支持 target | `apps/kimi-code/scripts/native/native-deps.mjs:18` `SUPPORTED_TARGETS` |
