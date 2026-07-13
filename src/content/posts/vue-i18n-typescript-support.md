---
title: 'Vue-i18n TypeScript 补全提示'
pubDate: '2026-07-13'
---

Vue-i18n 是 Vue 生态中最流行的国际化方案，但原生对 TypeScript 的类型提示比较有限——默认情况下 `$t('key')` 的参数是 `string`，写错 key 也不会报错。本文介绍一种基于 JSON 语言文件 + 模块增强的配置方式，让 IDE 能自动提示所有翻译 key，并在传入不存在的 key 时给出类型报错。

## 环境准备

本文以 **Vue 3 + Vite + TypeScript** 项目为例，`vue-i18n` 版本为 v11.4.6。

## 项目结构

```
root/
│── src/
│   ├── locales/            # 语言文件目录
│   │   ├── index.ts        # 语言文件索引（导出所有 locale）
│   │   ├── en-US.json      # 英文语言文件
│   │   └── zh-CN.json      # 中文语言文件
│   ├── App.vue
│   ├── main.ts
│   ├── global.d.ts         # 全局类型声明
│   ├── composables/
│   │   └── useTranslation.ts  # 封装 t 函数
├── index.html
├── vite.config.ts
└── package.json
```

> 本文中出现的代码基于此目录结构，若项目结构不同，需要相应调整。

## 第一步：准备语言文件

语言文件使用 JSON 格式，key 的层级结构决定了后续类型提示的路径。

```json
// src/locales/en-US.json
{
  "message": {
    "hello": "Hello World",
    "description": "This is a description"
  },
  "nav": {
    "home": "Home",
    "about": "About"
  }
}
```

```json
// src/locales/zh-CN.json
{
  "message": {
    "hello": "你好世界",
    "description": "这是一段描述"
  },
  "nav": {
    "home": "首页",
    "about": "关于"
  }
}
```

## 第二步：导出语言文件索引

创建 `src/locales/index.ts`，将所有语言文件按 locale key 导出：

```ts
import enUS from './en-US.json'
import zhCN from './zh-CN.json'

export default {
  'en-US': enUS,
  'zh-CN': zhCN
} as const
```

## 第三步：全局类型声明

创建 `src/global.d.ts`，通过模块增强（Module Augmentation）让 `vue-i18n` 识别你的语言文件结构：

```ts
import type { DefineLocaleMessage } from 'vue-i18n'
import locales from './locales'

declare global {
  // 将语言文件结构暴露为全局类型，方便在其他地方引用
  type MessageSchema = (typeof locales)['en-US']
}

// vue-i18n@9+ 模块增强
declare module 'vue-i18n' {
  // eslint-disable-next-line @typescript-eslint/no-empty-object-type
  interface DefineLocaleMessage extends MessageSchema {}
  interface GeneratedTypeConfig {
    locale: keyof typeof locales
  }
}
```

### 关键点解释

- `DefineLocaleMessage`：`vue-i18n` 用来约束翻译消息结构的接口，扩展它就等于把所有语言文件的 key 注入了类型系统。
- `MessageSchema`：从 `locales` 的 `'en-US'` 值推导出的类型，代表了所有翻译 key 的完整结构。
- `GeneratedTypeConfig`：约束 `locale` 类型，确保只能传入 `locales` 中定义的语言（本文中为 `en-US` 和 `zh-CN`）。

## 第四步：初始化 VueI18n 实例

在 `main.ts` 中初始化 i18n 实例，并传入语言文件：

```ts
import { createApp } from 'vue'
import { createI18n } from 'vue-i18n'
import App from './App.vue'
import locales from './locales'

const i18n = createI18n({
  legacy: false, // 使用 Composition API 模式
  locale: 'zh-CN',
  fallbackLocale: 'en-US',
  messages: locales
})

const app = createApp(App)
app.use(i18n)
app.mount('#app')
```

> **注意**：`legacy: false` 表示使用 Vue I18n v9+ 的 Composition API 模式。如果使用 Options API，则设置为 `true`，并在组件中通过 `this.$t` 访问。

## 第五步：在组件中使用

### SFC 模板中使用

```vue
<template>
  <!-- IDE 会自动提示 message.hello、message.description、nav.home 等 key -->
  <h1>{{ $t('message.hello') }}</h1>
  <p>{{ $t('message.description') }}</p>
  <nav>
    <a href="/">{{ $t('nav.home') }}</a>
    <a href="/about">{{ $t('nav.about') }}</a>
  </nav>
</template>
```

### TS 脚本中使用

在 `<script setup lang="ts">` 中：

```vue
<script setup lang="ts">
const { t } = useI18n()

// 有类型提示
const title = t('message.hello')
</script>
```

## 进阶：封装 composable 实现严格类型检查

原生 `t` 函数的参数类型是 `string`，即使传入不存在的 key 也不会在编译时报错（[vue-i18n issue #1116](https://github.com/intlify/vue-i18n/issues/1116)）。可以通过封装一个自定义 `useTranslation` composable 来进一步限制 key 的类型。

创建 `src/composables/useTranslation.ts`：

```ts
import type { NamedValue } from 'vue-i18n'
import { useI18n } from 'vue-i18n'

// 递归提取对象的所有叶子路径，如 "message.hello"
type PathToLeaves<T, Cache extends string = ''> = T extends PropertyKey
  ? Cache
  : {
      [P in keyof T]: P extends string
        ? Cache extends ''
          ? PathToLeaves<T[P], `${P}`>
          : PathToLeaves<T[P], `${Cache}.${P}`>
        : never
    }[keyof T]

type TranslationPaths = PathToLeaves<MessageSchema>

export const useTranslation = () => {
  const { t } = useI18n<{ messages: MessageSchema }>()

  return {
    t: <const Path extends TranslationPaths>(path: Path, named?: NamedValue) => (named ? t(path, named) : t(path))
  }
}
```

### 使用封装后的 t

```vue
<script setup lang="ts">
import { useTranslation } from '@/composables/useTranslation'

const { t } = useTranslation()

// ✅ 正确的 key，有类型提示
const hello = t('message.hello')

// ❌ 编译报错：类型 '"message.nonexistent"' 不满足 TranslationPaths
// const wrong = t('message.nonexistent')
</script>
```

通过以上配置，你的 Vue-i18n 就能获得完整的 TypeScript 类型提示，在编译阶段就发现翻译 key 的拼写错误，而不是等到运行时才暴露问题。
