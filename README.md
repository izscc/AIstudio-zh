# Google AI Studio 汉化脚本（船仓完美版）

一个用于汉化 [Google AI Studio](https://aistudio.google.com/) 的 Tampermonkey / Violentmonkey 用户脚本。

## 安装

- Greasy Fork：<https://greasyfork.org/zh-CN/scripts/556804>
- GitHub Raw：<https://raw.githubusercontent.com/izscc/AIstudio-zh/main/google-ai-studio-zh.user.js>

推荐从 Greasy Fork 安装，后续版本会通过 Greasy Fork 的 `@updateURL` 自动更新。

## 本次覆盖重点

- AI Studio 首页、Playground / Chat、Build Apps、API Keys / Dashboard 等主要可见页面。
- 左下角“新功能”弹层、设置菜单、账号区域、模型选择器、按钮 tooltip / aria-label / placeholder。
- 动态日期、Token、层级、价格、用量等常见 UI 文案。

## 发版流程

1. 修改 `google-ai-studio-zh.user.js` 并递增 `@version`。
2. 推送到 GitHub `main`。
3. 创建 tag / GitHub Release。
4. Greasy Fork 从 GitHub Raw URL 同步新版本。

## 许可证

MIT
