---
title: 'Chrome滚动搜索插件 Infinite Search'
pubDate: '2026-07-07'
type: 'Extension'
---

[Infinite Search](https://github.com/katiua/infinite-search) 是一个为 Google 搜索结果页提供**无限滚动（Infinite Scroll）**的 Chrome 插件。它把原本需要一页页点击分页器的搜索体验，变成了一条滑动到底的连续信息流。

## 它能做什么

在 Google 搜索时，结果页底部通常是一个分页导航（`1 2 3 … Next`）。装了 Infinite Search 之后：

- 页面底部的分页器与页脚会被自动隐藏；
- 当你滚动到当前结果末尾时，插件会自动抓取并加载下一页；
- 下一页内容以无缝衔接的方式插入，形成一条连续的搜索结果流；
- 每一段内容顶部会标注 `Page N`，方便你了解当前所处位置。

效果上，你只需要在搜索结果里一直往下滚，直到看完所有结果，而不再需要手动点「下一页」。

## 工作原理

插件的核心脚本是一个注入到 `https://*.google.com/search*`。整体流程如下：

1. **提取分页链接**：解析底部的分页导航，收集所有页码对应的 URL。
2. **隐藏冗余元素**：把分页器、页脚、每页重复的顶部搜索栏、应用栏等元素隐藏掉，避免插入后内容重复。
3. **渲染下一页**：通过 `fetch` 拉取下一页 HTML，塞进一个无边框、自适应高度的 `iframe` 中渲染。
4. **滚动触发加载**：用 `IntersectionObserver` 监听最后一页元素，当它进入视口就加载下一页，如此循环直到所有页都加载完。

为避免请求过快，插件内置了一个**加载间隔（interval）**控制，默认 1000ms，加载下一页前会展示一个加载中的提示。

## 可配置项

设置通过 `chrome.storage.local` 持久化（`src/utils/settings.ts`），共有三项：

| 配置                   | 默认值  | 说明                                       |
| ---------------------- | ------- | ------------------------------------------ |
| `interval`             | `1000`  | 加载下一页的最小间隔（毫秒）               |
| `autoUpdatePagination` | `false` | 滚动时是否自动更新分页链接（应对动态分页） |
| `debug`                | `false` | 是否开启调试日志                           |

## 安装方式

由于尚未上架 Chrome Web Store，目前通过 Release 包手动安装：

1. 从 [Releases](https://github.com/katiua/infinite-search/releases) 下载压缩包；
2. 解压到本地目录；
3. 打开 Chrome，访问 `chrome://extensions/`，开启「开发者模式」；
4. 将解压后的文件夹拖入 Chrome 窗口即可。

## 小结

Infinite Search 是一个轻量、聚焦单一痛点的小工具：它不改变 Google 搜索本身，只是把「翻页」这个打断连续阅读的动作自动化掉。

- 仓库地址：<https://github.com/katiua/infinite-search>
