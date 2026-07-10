---
title: 'Web Component 如何继承全局样式'
pubDate: '2026-07-09'
---

Web Component 的一大卖点是 **Shadow DOM 的样式隔离**：组件内部的样式不会泄漏出去，外部的样式也进不来。这在写通用组件时很省心，但当你希望组件复用页面里已有的一套设计系统（比如 Tailwind、UnoCSS 生成的全局样式，或是自定义的 CSS 变量与工具类）时，隔离反而成了阻碍——Shadow DOM 里的元素拿不到 `document` 上定义的任何样式。

本文的做法是：把页面上现有的全局样式表**复制一份注入到 Shadow Root**，让组件在保持隔离的同时，仍能「继承」全局样式。

## 为什么全局样式进不来

普通元素的样式来自文档级的样式表（`document.styleSheets`），而 Shadow DOM 是一个独立的样式作用域。除了少数可穿透的机制（如继承属性、CSS 自定义属性、`::part` 等），文档里的 CSS 规则默认不会应用到 Shadow Root 内部。

所以要让组件用上全局样式，最直接的思路就是：把这些规则也放进 Shadow Root 自己的样式作用域里。

## 核心实现

现代浏览器提供了 [Constructable Stylesheets](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleSheet/CSSStyleSheet) 和 `adoptedStyleSheets`，可以让多个 Shadow Root **共享同一份样式表实例**，而不必在每个组件里重复写 `<style>`。

```ts
let globalSheets: CSSStyleSheet[] | null = null

function getGlobalStyleSheets() {
  if (globalSheets === null) {
    globalSheets = Array.from(document.styleSheets).map((item) => {
      const sheet = new CSSStyleSheet()
      const css = Array.from(item.cssRules)
        .map((rule) => rule.cssText)
        .join(' ')
      sheet.replaceSync(css)
      return sheet
    })
  }

  return globalSheets
}

function addGlobalStylesToShadowRoot(shadowRoot: ShadowRoot) {
  shadowRoot.adoptedStyleSheets.push(...getGlobalStyleSheets())
}
```

然后在组件的 `connectedCallback` （在可访问 `shadowRoot` 的阶段）中调用 `addGlobalStylesToShadowRoot`：

```ts
connectedCallback() {
  addGlobalStylesToShadowRoot(this.shadowRoot)
}
```

## 工作原理

整个过程可以拆成三步：

1. **读取文档样式表**：`document.styleSheets` 拿到页面上所有已加载的样式表（`<style>` 和 `<link rel="stylesheet">`）。
2. **重建为可复用样式表**：遍历每张样式表的 `cssRules`，把规则文本拼接后用 `replaceSync` 写进一个新建的 `CSSStyleSheet`。之所以要「重建」，是因为原始的样式表对象不能直接放进 `adoptedStyleSheets`，而 Constructable Stylesheet 可以被多个 Shadow Root 共享。
3. **注入 Shadow Root**：通过 `shadowRoot.adoptedStyleSheets.push(...)` 把这些样式表挂到组件上，Shadow DOM 内部即可命中全局规则。

> 这里用一个模块级的 `globalSheets` 做了**缓存**：第一次调用时才构建，之后所有组件复用同一批 `CSSStyleSheet` 实例。这样既避免了重复解析全局 CSS 的开销，也让多个组件真正共享同一份样式表，内存更友好。

## 注意事项

- **跨域样式表会抛错**：如果某张 `<link>` 引入的是跨域样式表，访问它的 `cssRules` 会触发 `SecurityError`。稳妥起见可以在 `map` 里加上 `try/catch` 跳过这类样式表：

  ```ts
  globalSheets = Array.from(document.styleSheets).flatMap((x) => {
    try {
      const sheet = new CSSStyleSheet()
      const css = Array.from(x.cssRules)
        .map((rule) => rule.cssText)
        .join(' ')
      sheet.replaceSync(css)
      return [sheet]
    } catch {
      return []
    }
  })
  ```

- **快照而非实时同步**：`getGlobalStyleSheets` 只在首次调用时读取一次全局样式。如果之后全局 CSS 发生变化（例如运行时动态插入新样式表），组件不会自动更新，需要手动清空缓存重建。

- **调用时机**：确保调用时全局样式表已经加载完成，否则可能只拿到部分规则。放在 `connectedCallback` 中通常足够，但如果样式是异步注入的，需要留意顺序。

- **`adoptedStyleSheets` 的兼容性**：主流现代浏览器均已支持数组的 `push` 写法；面向老旧环境时可改用整体赋值 `shadowRoot.adoptedStyleSheets = [...]`。
