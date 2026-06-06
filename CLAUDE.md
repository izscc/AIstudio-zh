# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

本仓库是一个**单文件 Tampermonkey / Violentmonkey 用户脚本**，用于把 Google AI Studio（`https://aistudio.google.com/*`）的界面汉化为简体中文。全部逻辑都在 `google-ai-studio-zh.user.js` 中，其余文件（README、CHANGELOG、LICENSE）是文档。`.venv/` 是开发者抓取页面文案时的 Python 环境，已被 gitignore，不属于发布产物。

没有构建系统、包管理清单、测试或 lint 配置——脚本即成品，浏览器直接加载运行。

## 常用命令

```bash
# 语法检查（脚本是纯 JS，可用 node 校验，不会真正运行）
node --check google-ai-studio-zh.user.js

# 检测翻译表里的重复 key（同名 key 后者覆盖前者，见下文陷阱）
python3 - <<'PY'
import re
from collections import Counter
keys = re.findall(r'^\s*"((?:[^"\\]|\\.)+)":\s*"', open('google-ai-studio-zh.user.js',encoding='utf-8').read(), re.M)
for k,v in Counter(keys).items():
    if v>1: print(v, k)
PY
```

本地验证只能靠手动：把脚本装进 Tampermonkey/Violentmonkey，访问 `aistudio.google.com` 的目标页面肉眼检查。补漏翻时按 CHANGELOG 记录的做法，**用已登录的真实 Chrome 窗口**查看页面真实文案，而不是临时 profile。

## 发版流程

1. 改 `google-ai-studio-zh.user.js`，并在 UserScript 头部递增 `@version`（第 4 行）。
2. 在 `CHANGELOG.md` 顶部追加对应版本条目。
3. 推送到 GitHub `main`，打 tag / 创建 GitHub Release。
4. Greasy Fork（脚本 ID 556804）通过头部的 `@updateURL` / `@downloadURL` 自动同步。

`@version` 与 CHANGELOG 顶部版本必须一致；`@downloadURL` / `@updateURL` 指向 Greasy Fork 而非 GitHub，**改动后不要动这两个 URL**。

## 架构

整个脚本是一个 IIFE（`google-ai-studio-zh.user.js:17`），`@grant none`、`@run-at document-idle`。结构分为「数据」和「引擎」两部分。

### 翻译数据（文件前 ~1950 行）

四种数据结构，合并/匹配优先级各不相同：

- **`translations`**（`:27` 起的对象字面量）——主映射表，英文原文 → 中文，**精确匹配**。条目按版本/巡检批次分组，组与组之间用注释分隔。它在文件后段被两次扩充：`Object.assign(translations, Object.fromEntries(longFormTranslations))`（`:1798`）和 `Object.assign(translations, {...})`（`:1829`，v3.7.1 巡检批次）。
- **`longFormTranslations`**（`:1771` 的 `Map`）——长段落（歌词、TTS 示例脚本等），用数组对形式书写，避免在主对象里塞超长字符串，最终并入 `translations`。
- **`defaultValueTranslations`**（`:1944` 的 `Map`）——仅用于 `<textarea>` / `<input>` 的 `value`（表单默认值）。**刻意只做精确匹配**，这是为了不动用户自己输入的内容（v3.7.4 的设计决定）。
- **`regexReplacements`**（`:1950` 的数组）——动态文案，每条是 `{ regex, replacement }`，`replacement` 可为字符串或函数。处理价格、Token 数、相对时间（"5 minutes ago"）、层级、发言者编号、日期等带变量的句子。日期由 `translateEnglishDateFragments()`（`:1809`）+ `monthNameToNumber()` 转成「YYYY年M月D日」。绝大多数正则用 `^...$` 锚定整段，避免误伤。

### 翻译引擎（文件末 ~330 行）

`init()`（`:2268`）做两件事：对 `document.body` 跑一次 `walkAndTranslate`，然后挂上 `MutationObserver` 处理后续动态 DOM。

- **`walkAndTranslate(rootNode)`**（`:2203`）——用 `TreeWalker` 遍历元素和文本节点，对每个节点调 `translateNode`。
- **`translateNode(node)`**（`:2130`）——核心。
  - 元素节点：先调 `applySpecialElementTranslations`，再翻译白名单属性 `aria-label` / `aria-description` / `placeholder` / `mattooltip` / `title` / `data-tooltip` / `data-after-input` / `dialoglabel`。
  - 文本节点：匹配顺序为 **`translationCache` → `translations` 精确匹配 → cookie 提示前缀特例 → `regexReplacements` 逐条 `test`**。命中即停。改写时用 `originalText.replace(text, translated)` 以**保留首尾空白**。
- **`applySpecialElementTranslations(node)`**（`:2053`）——三种普通遍历命不中的特例：`<textarea>`/`<input>` 默认值（改后派发 `input` 事件）、`.v3-token-count-value` 的 Token 计数、App Builder 首页被拆成 5 个 `.hero-word` 动画单词的标题（按单词逐个替换）。
- **`shouldSkipNode(node)`**（`:2097`）——代码保护。跳过 `SCRIPT`/`STYLE`/`PRE`/`CODE` 以及 `monaco-editor`/`code-block`/`hljs`/`cm-editor`/`prism` 等编辑器容器，并**向上回溯 5 层父节点**确认不在代码块内，避免汉化用户代码。
- **`MutationObserver`**（`:2246`）——**50ms 防抖批处理**。`childList` 新增节点走 `walkAndTranslate`；`characterData` 重译目标；`attributes`（仅监听上述白名单属性）先 `processedNodes.delete` 再重译。

### 两个全局缓存

- `processedNodes`（`WeakSet`）——防止同一节点重复处理。
- `translationCache`（`Map`）——缓存「原文 → 译文」，让正则只算一次。

## 编辑翻译时的关键规则

- **加新词条前先搜原文是否已存在**。表中目前有 ~26 个重复 key（如 `Start building`、`Model settings`），JS 语义下**后定义覆盖先定义**——重复添加会造成冲突或冗余，用上面的 python 命令可检出。
- **精确匹配的 key 必须与页面文案逐字一致**（含标点、`'` 与 `’` 等 Unicode 差异、首尾空格已被 `trim`）。匹配不上时优先考虑是不是该走 `regexReplacements`。
- **含变量的文案（数字、价格、日期、模型名）一律用 `regexReplacements`**，不要把每个具体值都塞进精确表。新正则注意用 `^...$` 锚定，并放在数组中合适的位置（同类里更具体的规则应在更宽泛的规则之前）。
- 表单默认值（用户可编辑的 `value`）只加到 `defaultValueTranslations`，**不要**加到 `translations`，否则会覆盖用户输入。
