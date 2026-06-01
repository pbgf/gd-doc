# @sailorgundam/gundam-h5-build

> Gundam H5 React 构建工具 — 基于 Rollup + Vite 的多端构建系统

**版本**: 0.148.0  
**定位**: 美团外卖高达（Gundam）活动页框架的 H5/Native/微信小程序构建工具  
**核心引擎**: Rollup（生产构建）+ Vite（H5 开发服务）双引擎架构

---

## 目录

- [架构概览](#架构概览)
- [构建流程](#构建流程)
- [核心模块](#核心模块)
  - [入口与配置](#入口与配置)
  - [类型定义与常量](#类型定义与常量)
  - [Rollup 构建](#rollup-构建)
  - [Vite 开发服务](#vite-开发服务)
  - [Rollup 插件](#rollup-插件)
  - [CSS 处理插件](#css-处理插件)
  - [资源处理插件](#资源处理插件)
  - [产物后处理插件](#产物后处理插件)
  - [Babel 插件](#babel-插件)
  - [工具函数](#工具函数)
  - [运行时基座代码](#运行时基座代码)
  - [模板文件](#模板文件)
- [依赖关系](#依赖关系)

---

## 架构概览

```
                          ┌──────────────────┐
                          │    build(params)  │  ← CLI 入口
                          └────────┬─────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
              ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼──────────┐
              │  Vite Dev  │ │Rollup Build│ │  Vite Build    │
              │  (H5 开发) │ │(component/ │ │  (H5 page)     │
              │            │ │ pageContain│ │                │
              │            │ │ /library/  │ │                │
              │            │ │ page)      │ │                │
              └─────┬──────┘ └─────┬──────┘ └─────┬──────────┘
                    │              │              │
              ┌─────▼──────┐ ┌────▼────────────┐ │
              │machpro-vite│ │h5RollupConfig   │ │
              │  Plugins   │ │ + Rollup Plugins│ │
              └─────┬──────┘ └────┬────────────┘ │
                    │              │              │
                    └──────────────┼──────────────┘
                                   │
                          ┌────────▼─────────┐
                          │   产物输出         │
                          │ H5: bundle.js/css │
                          │ Native: qbc+zip   │
                          │ WeChat: wxss+wxml │
                          └──────────────────┘
```

---

## 构建流程

### Rollup 构建流程（component / pageContainer / library / page）

```
入口 → 多态文件解析 → CSS处理(PostCSS+CSS Module) → JSON解析 → TS路径解析 →
环境变量替换 → CommonJS转换 → Node Polyfills → 模块解析(node-resolve) →
TypeScript编译 → 资源处理(Babel) → JSX+CSS Module+字体替换(Babel) →
代码清理 → 压缩(terser) → 依赖分析可视化 → 输出
```

**输出格式**:
- `component` / `pageContainer` / `library` → **UMD**，React/Preact 作为外部依赖
- `page` → **IIFE**，所有依赖内联

### Vite 开发流程（H5 dev）

```
入口HTML → HTML模板渲染(mustache+cheerio) → 多态文件解析 →
虚拟基座代码注入(virtual:h5-base) → JSX转译+路由注入(machProReact) →
Vite配置合并(machProConfig) → TS路径别名 → PostCSS处理 → HMR
```

### Native/WeChat 构建后处理

```
bundle.js + bundle.css →
  Native: CSS→JSON → JS→Bytecode(qjsc) → meta.json → ZIP打包
  WeChat: JS降级(ES6→ES5) → 变量名压缩 → wxml/wxss模板拷贝 → ZIP打包
```

---

## 核心模块

### 入口与配置

#### `src/index.ts` — 主入口

构建系统的入口文件，导出 `build()` 和 `runBuild()` 两个核心函数。

| 函数 | 说明 |
|------|------|
| `build(params: ICliParams)` | 主入口函数。根据 `buildType` 路由到 Vite 或 Rollup：H5 开发模式走 Vite `runDev()`；component/pageContainer/library/page 走 Rollup `runBuild()`；其余走 Vite `runBuild()` |
| `runBuild()` | Rollup 构建执行函数。获取 Rollup 配置、执行 `rollup.rollup()`、写入产物、输出耗时 |

构建路由逻辑：
- `params.dev` + H5 → `vite.runDev()`
- `buildType` 为 component/pageContainer/library/page → `runBuild()`（Rollup）
- 其他 → `vite.runBuild()`

---

#### `src/getConfig.ts` — 配置管理

管理所有构建配置的读取、合并和默认值处理。

| 函数/变量 | 说明 |
|-----------|------|
| `projectRoot` | 项目根目录（`process.cwd()`） |
| `setCliParams(params)` | 设置 CLI 参数（全局缓存） |
| `getCliParams()` | 获取 CLI 参数副本 |
| `getMachConfig()` | 获取 mach.config.js 配置副本 |
| `getInjectMachConfig()` | 获取注入到 JS 代码中的 Mach 配置（bundleName/biz/page/version/router/pxTransform） |
| `createPxTransform(res)` | 创建 CSS 单位转换配置，合并默认规则（PX→px, dp→px）与用户自定义规则 |
| `getConfig()` | **核心函数**，合并 CLI 参数 + mach.config.js 配置 + tsconfig.json + 用户自定义 Rollup/Vite 配置，返回完整的 `IConfig` 对象 |

`getConfig()` 返回的 `IConfig` 包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `bundleName` | `string` | 包名称 |
| `cliParams` | `ICliParams` | CLI 入口参数 |
| `machConfig` | `IMachConfig` | mach.config.js 配置 |
| `projectRoot` | `string` | 项目根目录 |
| `sourcePath` | `string` | 代码源目录 |
| `input` | `string` | Rollup 主入口文件路径 |
| `output` | `string` | Rollup 输出目录 |
| `distPath` | `string` | 最终产物输出目录 |
| `wechatDistPath` | `string` | 小程序产物输出目录 |
| `tsconfig` | `object` | TypeScript 配置 |
| `rollupConfig` | `function` | 用户自定义 Rollup 配置修改函数 |
| `viteConfig` | `function` | 用户自定义 Vite 配置修改函数 |
| `pxTransform` | `object` | CSS 单位转换配置 |
| `fontFamilyTransform` | `object` | 字体替换配置 |

入口文件映射（`INPUT_MAP`）：

| buildType | 开发模式入口 | 生产模式入口 |
|-----------|------------|------------|
| `library` | — | `src/index.ts` |
| `pageContainer` | `src/h5/main.tsx` | `src/h5/index.ts` |
| `component` | `src/.gundam/main.tsx` | `src/index.tsx` |
| `page` | — | `src/main.tsx` |

---

### 类型定义与常量

#### `src/types.ts` — 接口定义

| 接口 | 说明 |
|------|------|
| `ICliParams` | CLI 参数接口。包含 `env`、`isWechat`、`sub`（子包）、`child`（本地子包）、`bundle`、`bundleName`、`machConfig`、`dev`、`type`(MACH_ENV)、`worker`、`args`、`buildType`（page/component/pageContainer/library） |
| `IMachConfig` | Mach 配置接口。包含 `root`、`input`、`output`、`distPath`、`wechatDistPath`、`wechatTemplatePath`、`subTreeFilePath`、`biz`、`page`、`version`、`router`、`viteConfig`、`rollupConfig`、`pxTransform`、`fontFamilyTransform` 等 |

#### `src/constant.ts` — 常量

| 常量 | 值 | 说明 |
|------|---|------|
| `MACH_ENV` | `{ WECHAT: 'wechat', REACT: 'react', H5: 'h5' }` | 目标平台枚举 |
| `CSS_FILE_NAME` | `'bundle.css'` | 输出 CSS 文件名 |
| `GLOBAL_CSS_FUNC_KEY` | `'global._rcls_'` | CSS Module 运行时 class 解析函数的全局变量名 |
| `RUNTIME_CLA_CAL_FUNC` | 函数字符串 | 运行时 class 名解析函数实现，将 CSS Module 的 hash 映射还原为实际 class 名 |

#### `src/type.d.ts` — 模块声明

为 `@babel/parser`、`babel-types`、`babel-generator`、`source-map`、`fast-glob`、`terser`、`magic-string` 等无类型定义的包提供模块声明。

---

### Rollup 构建

#### `src/h5RollupConfig.ts` — Rollup 配置

生成完整的 Rollup 构建配置，包含所有插件的组装。

**外部依赖映射** (`EXTERNAL_LIB_MAP`):

| 包名 | 全局变量名 |
|------|-----------|
| `@sailorgundam/gundam-core` | `GundamCore` |
| `@sailorgundam/gundam-seg-logic` | `GundamSegLogic` |
| `@sailorgundam/gundam-ui` | `GundamUI` |
| `@sailorgundam/gundam-store` | `GundamStore` |
| `@sailorgundam/gundam-utils` | `GundamUtils` |
| `@sailorgundam/gundam-base-components` | `GundamBaseComponents` |
| `@gdc/sailor_gundam_containercomp_machpro` | `GundamH5Container` |

**插件加载顺序**：

1. `rollupMultilImport('h5')` — 多态文件解析
2. `postcssForMachPro()` — CSS 处理（非 worker 进程）
3. `pluginJSON()` — JSON 导入
4. `rollupPluginTsConfigPaths()` — TS 路径别名
5. `replace()` — 环境变量替换（`process.env.MACH_ENV`、`NODE_ENV`、`SUB_BUNDLE`、`BUNDLE_NAME` 等）
6. `commonjs()` — CommonJS 转换
7. `nodePolyfills()` — Node.js Polyfills
8. `resolve()` — 模块解析（优先 `h5main` > `module` > `main`）
9. `typescript()` — TypeScript 编译
10. `rollupPluginAssets()` — 资源文件处理
11. `babel()` — Babel 转换（CSS Module + JSX + 字体替换 + 用户自定义 Babel 配置）
12. `cleanUpPluginGetter()` — 代码清理
13. `terser()` — 代码压缩
14. `visualizer()` — 依赖分析可视化（非 UMD 构建）

**输出配置**：
- 格式：UMD（component/pageContainer/library）或 IIFE（page）
- SourceMap：启用
- React/Preact 全局映射为 `Preact`

---

### Vite 开发服务

#### `src/utils/vite/index.ts` — Vite 入口

| 函数 | 说明 |
|------|------|
| `runDev()` | 创建 Vite 开发服务器，应用 `machproVitePlugin()` 插件，监听端口并打印 URL |
| `runBuild()` | 执行 Vite 生产构建 |

#### `src/utils/vite/machpro-vite.ts` — H5 Vite 插件集

提供 7 个 Vite 插件，完整支持 H5 开发和构建：

| 插件 | 说明 |
|------|------|
| `dynamicHtmlConfig` | 入口 HTML 转换。使用 Mustache 渲染模板变量，通过 Cheerio 注入入口 `<script>` 标签 |
| `resolve` | 模块解析，配置 `.h5.js` 等扩展名 |
| `viteCommonjs` | CommonJS 兼容 |
| `virtualBaseCode` | 虚拟模块 `virtual:h5-base`，加载 H5 基座代码 + MachPro Element 依赖 |
| `rollupMultilImport('h5')` | 多态文件解析 |
| `machProReact` | JSX 转译插件。对入口文件注入基座代码、路由代码；对所有 TS/TSX 文件执行 Babel JSX 转换 |
| `machProConfig` | Vite 配置合并。设置 dev server 端口/代理、PostCSS 插件、环境变量定义、Node Polyfills 别名、Preact 替代 React 等 |

`machProConfig` 关键配置：
- `resolve.alias`：将 `react`/`react-dom`/`preact` 统一指向 `@sailorgundam/gundam-preact/esm`
- `css.postcss.plugins`：`postCssAttributePrefix('h5')` + `postCssPxtransform` + `rtlcss`
- `define`：注入 `process.env.MACH_ENV`/`NODE_ENV`/`SERVER_ENV`/`BUNDLE_NAME`/`SUB_BUNDLE`
- `server.proxy`：微前端子应用代理支持

#### `src/utils/vite-inject-code.ts` — Vite 代码注入

为 Vite 开发服务提供代码注入功能：

| 函数 | 说明 |
|------|------|
| `injectOwl(config)` | 注入 Owl 监控 SDK 启动代码 |
| `injectLX(config)` | 注入灵犀(LXAnalytics)埋点 SDK 配置代码 |
| `injectMachProElement(isMicro)` | 注入 `MACH_PROJECT_CONFIG` 全局配置 |
| `injectInput(machConfig, microName)` | 根据路由配置生成路由代码，使用 `preact-router` + `history` 实现页面导航，支持微前端导出 |

路由注入生成代码示例：
```javascript
import { render } from "preact";
import Router, { Route } from "preact-router";
import Component0 from './pages/Home';
import Component1 from './pages/Detail';

function App() {
  return (<Router>
    <Route component={Component0} path="/home" default />
    <Route component={Component1} path="/detail" />
  </Router>)
}
render(<App/>, document.body);
```

---

### Rollup 插件

#### `plugins/rollup-plugin-html.ts` — HTML 模板生成

基于 `@rollup/plugin-html` 生成入口 HTML 文件，注入 viewport meta 标签和 `bundle.css`/`bundle.js` 引用。

---

#### `plugins/rollup-plugin-main.ts` — 路由渲染代码注入

在主包入口文件中注入路由渲染代码：

- 检测 `machConfig.router` 配置
- 在入口文件末尾追加 `import App` + `render(<App />, document.body)`
- 使用 MagicString 保持 SourceMap 准确

---

#### `plugins/rollup-plugin-sub.ts` — 子包代码变换

将子包代码包装为回调函数格式：

- 解析 AST → 将整个 `Program` 包裹进 `globalThis.__bundleCallback__` 函数
- 返回 `app` 变量作为模块导出
- 保证子包可被主包动态加载

---

#### `plugins/rollup-plugin-injectjs.ts` — 统一代码注入

根据目标平台注入不同的基座代码，是构建系统的核心注入逻辑：

| 导出 | 说明 |
|------|------|
| `injectMachConfig()` | 在产物头部注入 `MACH_PROJECT_CONFIG` 全局变量 |
| `injectBaseCode()` | 根据平台注入基座代码：Native 注入 mach-pro-render + Preact + ElementShim；WeChat 注入 base-wechat + Preact |
| `rollupPluginInject()` (default) | 统一入口，组合 machconfig + main/sub + baseCode 插件。Worker 不注入；子包使用 `rollupPluginSub()`；主包使用 `rollupPluginMain()` + `injectBaseCode()` |

---

#### `plugins/rollup-plugin-stat.ts` — 构建统计

构建完成后输出文件大小统计表格：

| 功能 | 说明 |
|------|------|
| 文件大小 | 显示每个产物文件的原始大小 |
| Gzip 大小 | 显示 gzip 压缩后大小 |
| 颜色标记 | >500KB 红色、>300KB 黄色、其他绿色 |
| TypeScript 警告 | 拦截并格式化 TS 编译警告 |
| 合计行 | 显示所有文件的总大小和总 Gzip 大小 |

---

#### `plugins/rollup-plugin-multil-import.ts` — 多态文件解析

实现 Gundam 框架的多端文件协议，根据目标平台自动选择对应端的实现文件：

**支持的文件匹配模式**：
- `./{name}.{type}.{ext}` — 同目录下的分端文件
- `./{name}/index.{type}.{ext}` — 目录下的 index 文件
- `./{name}/{name}.{type}.{ext}` — 目录下同名文件

**后缀映射**：
| buildType | 后缀 |
|-----------|------|
| `react` → `native` | `.native.js` |
| `wechat` → `mp` | `.mp.js` |
| `h5` | `.h5.js` |

同时支持样式文件的多态导入（`.h5.css`、`.native.css`、`.mp.css`）。

---

#### `plugins/rollup-plugin-tsconfig-paths.ts` — TS 路径解析

从 `tsconfig.json` 的 `compilerOptions.paths` 解析路径别名，使 Rollup/Vite 能正确处理 TypeScript 路径映射。

| 选项 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `absolute` | `boolean` | `true` | 是否解析为绝对路径 |
| `nonRelative` | `boolean` | `false` | 是否基于 `baseUrl` 解析非相对路径 |
| `preserveExtensions` | `boolean` | `true` | 是否保留 `.ts`/`.tsx` 扩展名 |
| `tsConfigPath` | `string` | 自动检测 | 自定义 tsconfig.json 路径 |
| `transform` | `function` | — | 路径转换钩子 |

---

#### `plugins/rollup-plugin-cleanup/index.ts` — 代码清理

根据构建类型选择不同的清理策略：
- `component` / `pageContainer` → 使用旧版清理插件（兼容性处理）
- 其他 → 使用标准 `rollup-plugin-cleanup`

---

#### `plugins/replace-react-import.ts` — React 导入替换

Babel 插件，将 `import React from 'react'` / `import { useState } from 'react'` 等 React 导入语句替换为对目标模块的解构赋值，用于 Preact 替代 React 的场景。

转换示例：
```javascript
// 输入
import React, { useState, useEffect } from 'react';
// 输出
const { useState, useEffect } = Preact;
const React = Preact;
```

---

#### `plugins/wechat-template-copy.ts` — 微信模板拷贝

将微信小程序模板文件拷贝到产物目录，支持：
- 用户自定义模板路径（`wechatTemplatePath`）
- 默认内置模板（`template/` 目录）
- 拷贝到开发目录 + `dist/wechat` 目录

---

#### `plugins/static-temp-process.ts` — 微信静态模板处理

生成微信小程序的 `static-temp.wxml` 模板文件：
- 注入 WXS 图片优化函数（`@500w_1l` CDN 参数）
- 将自定义组件的 WXML 收集为 `<template>` 标签
- 仅在微信小程序环境下生效

---

#### `plugins/rollup-plugin-meta.ts` — meta.json 生成

在构建完成后生成 `meta.json` 元信息文件：

| 字段 | 说明 |
|------|------|
| `md5.bundle.qbc` | Bytecode 文件的 MD5 |
| `md5.bundle.css.json` | CSS JSON 文件的 MD5 |
| `dependencies` | 依赖信息（来自 mach.config.js） |
| `bundleType` | Bundle 类型 |
| `quickjsVersion` | QuickJS 编译器版本号 |

---

### CSS 处理插件

#### `plugins/rollup-plugin-css/index.ts` — CSS 处理入口

根据目标平台选择不同的 PostCSS 配置：

| 导出 | 适用平台 | 说明 |
|------|---------|------|
| `postCssPlugin` | Native | 提取 CSS + pxTransform + fontFamilyTransform + polyfill + CSS Module |
| `postCssPluginH5` | H5 | 可选提取 CSS + attributePrefix + pxTransform(H5规则) + fontFamilyTransform + RTL + 可选 CSS Module |
| `postCssPluginWechat` | WeChat | 提取 wxss + pxTransform + fontFamilyTransform + polyfill + CSS Module |
| `postcssForMachPro` | 统一入口 | 根据 `process.env.MACH_ENV` 自动选择，并注入 CSS Module 运行时代码 |

H5 CSS 单位转换规则：
| 源单位 | 比例 | 目标单位 |
|--------|------|---------|
| `px` / `PX` | 1 | `px` |
| `rpx` / `RPX` | 0.01 | `rem` |
| `dp` / `DP` | 1 | `px` |

---

#### `plugins/rollup-plugin-css/css-module.ts` — CSS Module 支持

实现 CSS Module 的类名 hash 和运行时注入：

| 函数 | 说明 |
|------|------|
| `getHash(str)` | 基于 `string-hash` 生成 6 位短 hash |
| `cssModuleConfig(bundleName)` | 配置 CSS Module：开发环境使用 `[local]-[hash:base64:5]`，生产环境使用 `[hash:base64:5]`，以 bundleName 作为 hash 前缀 |
| `rollupPluginInjectCssModule(cssTag, extractCSS)` | 在产物中注入 CSS Module 运行时代码：CSS Map JSON + class 名解析函数（component 用 `component-class.js` 模板，page 用 `class.js` 模板） |

---

#### `plugins/rollup-plugin-css/babel-plugin-css-module.ts` — JSX className 变换

Babel 插件，自动转换 JSX 中的 `className`/`class`/`hoverClass` 属性：

- 扫描文件中的 CSS 导入，记录每个 CSS 文件的 hash
- 将 `className="foo"` 转换为 `className={_rcls_("foo", hash1, hash2)}`
- 递归处理子 JSX 元素
- 跳过已使用 CSS Module 变量的表达式

---

#### `plugins/rollup-plugin-css/post-css-pxtransform/index.ts` — CSS 单位转换

PostCSS 插件，将 CSS 中的 `px`/`rpx`/`dp` 等单位按配置的转换规则替换：

- 支持自定义转换规则（`replace` 映射表）
- 支持属性白名单/黑名单
- 支持选择器黑名单
- 支持最小像素值过滤（`minPixelValue`）
- 支持 1px 不转换选项（`onePxTransform`）
- 支持媒体查询中的单位转换

---

#### `plugins/rollup-plugin-css/post-css-polyfill/index.ts` — 环境前缀 CSS 处理

PostCSS 插件，处理跨端 CSS 差异：

| 功能 | 说明 |
|------|------|
| 环境前缀 | `-native-`/`-mp-`/`-web-` 前缀属性，只保留当前平台的，其余移除 |
| 渐变处理 | Native: `background-image:gradient` → `background-color`；WeChat: 反向 |
| max-lines | WeChat: `max-lines: N` → `-webkit-line-clamp: N` + flex 布局 |
| resize-mode | 收集 `resize-mode` 属性值供后续处理 |

---

#### `plugins/rollup-plugin-css/post-css-font-family-transform/index.ts` — 字体名替换

PostCSS 插件，在 CSS 中替换字体名称：

- 不区分大小写的字体名匹配
- 支持字符串或对象形式的替换规则
- 支持保留原有字体顺序（`preserveOrder`）
- 支持添加 fallback 字体（`addFallback` + `fallbackFonts`）
- 支持选择器黑名单和媒体查询处理
- 默认配置将 KeeTa 系列字体替换为 MT + NotoSans 系列字体

---

#### `plugins/rollup-plugin-css/post-css-attribute-prefix.ts` — CSS 属性前缀移除

PostCSS 插件，根据目标平台处理 CSS 属性前缀：

| 前缀 | 保留平台 | 说明 |
|------|---------|------|
| `-web-` | H5 | Web 专属属性 |
| `-native-` | Native | Native 专属属性 |
| `-mp-` | WeChat | 小程序专属属性 |

非当前平台的前缀属性直接移除，当前平台的去掉前缀保留。

---

### 资源处理插件

#### `plugins/rollup-plugin-assets/index.ts` — 资源文件处理

Rollup 插件，通过 Babel 转换资源文件路径：
- 使用 `babelPluginAssets` 处理图片导入
- 微信环境下启用远端资源（`enableRemoteAssets`）
- 通过 `this.emitFile` 将资源文件输出到 `assets/` 目录

#### `plugins/rollup-plugin-assets/babel-plugin-assets/index.ts` — 图片路径变换

Babel 插件，处理三种图片引用方式：

| 引用方式 | 处理逻辑 |
|---------|---------|
| JSX `<image src="...">` | 替换 `src`/`error`/`placeholder` 属性值 |
| `import img from './img.png'` | 将导入转换为 `const img = 'url'` |
| `require('./img.png')` | 替换为 URL 字符串 |
| `assets://xxx` 字符串 | 查找项目 assets 目录并替换路径 |

路径处理策略：
- 小程序：上传到 Venus CDN，替换为 HTTPS 链接
- 其他平台：<5KB 转 base64 内联；≥5KB 上传 Venus CDN

#### `plugins/rollup-plugin-assets/babel-plugin-assets/Cache.ts` — 图片上传缓存

基于 MD5 的图片上传缓存系统：

| 函数 | 说明 |
|------|------|
| `readFileMD5(filePath)` | 计算文件 MD5 |
| `loadCacheFile()` | 启动时加载 `~/.r2x/image_cache.json` 缓存文件 |
| `addCacheImage(filePath)` | 异步上传：MD5 命中缓存则直接返回，否则上传后缓存 |
| `addCacheImageSync(filePath)` | 同步上传：通过子进程执行上传脚本 |

#### `plugins/rollup-plugin-assets/babel-plugin-assets/image.ts` — CDN 上传

通过 `@mtfe/venus` SDK 上传图片到 CDN：

| 函数 | 说明 |
|------|------|
| `uploadImg(filePath)` | 异步上传图片到 `sailorgundambuild` bucket，返回 HTTPS 链接 |
| `uploadImgSync(filePath)` | 同步上传：通过子进程执行 Node 脚本完成上传 |

---

### 产物后处理插件

#### `plugins/rollup-plugin-compile-output/index.ts` — Native/WeChat 产物处理

构建完成后的产物后处理：

| 导出 | 说明 |
|------|------|
| `compileDistToOutput()` | Native 产物处理流程 |
| `compileOutput()` (default) | Rollup 插件入口，根据平台调用不同处理流程 |

**Native 处理流程** (`compileDistToOutput`)：
1. `transformCssToJson()` — CSS → JSON 格式（选择器去掉 `.` 前缀）
2. `compileToBytecode()` — JS → QuickJS Bytecode（`.qbc` 文件）
3. `createMetaFile()` — 生成 meta.json（MD5 + 依赖 + QuickJS 版本）
4. `copyMockFile()` — 拷贝 mock.json
5. `fileZip()` — 打包为 ZIP

**WeChat 处理流程** (`copyWechatOutputToDist`)：
1. Babel 降级（可选链、空值合并等 ES2020 语法 → ES5）
2. API 名称压缩（`React.attr` → `React.a` 等）
3. Terser 变量名混淆（`mangle.toplevel`）
4. 拷贝到 `dist/wechat/` + `meta.json` + `meta.js`
5. 触发外卖小程序重建（可选）
6. `fileZipWechat()` — 打包为 ZIP

#### `plugins/rollup-plugin-compile-output/hoist.ts` — 变量提升

Babel 插件，将函数声明和 `var` 声明提升到所在作用域顶部：
- `FunctionDeclaration` → 克隆到父作用域顶部，移除原位置
- `VariableDeclaration` (var) → 拆分为声明 + 赋值，声明提升到父作用域顶部

#### `plugins/rollup-plugin-compile-output/file-zip.ts` — ZIP 打包

使用 JSZip 打包产物：

| 函数 | 说明 |
|------|------|
| `fileZip()` | Native ZIP：只包含 `bundle.css.json` + `bundle.qbc` + `meta.json` + `mock.json` + assets，DEFLATE 压缩级别 9。同时生成 sourcemap ZIP（包含所有文件） |
| `fileZipWechat()` | WeChat ZIP：打包 `dist/wechat/` 下所有文件 |

---

### Babel 插件

#### `plugins/babel-plugin-font-family-transform/index.ts` — JS 字体替换

Babel 插件，在 JavaScript 代码中替换 `fontFamily` 属性值：

- 处理 `ObjectProperty` 中 `key` 为 `fontFamily` 的属性值
- 处理 `AssignmentExpression` 中 `left.property` 为 `fontFamily` 的赋值
- 支持字符串字面量和模板字符串
- 不区分大小写的替换规则
- 支持添加 fallback 字体

与 PostCSS 版 `post-css-font-family-transform` 的区别：此插件处理 JS 代码中的 `style.fontFamily`，PostCSS 版处理 CSS 中的 `font-family`。

---

### 工具函数

#### `src/utils/utils.ts` — 通用工具

| 函数 | 说明 |
|------|------|
| `getMachConfig()` | 读取 `mach.config.js` 配置文件 |
| `setProjectType(type)` | 设置 `process.env.MACH_ENV` 环境变量 |
| `getProjectType()` | 获取当前目标平台 |
| `isWechatProject()` | 判断是否为微信小程序项目 |
| `isNative()` | 判断是否为 Native 项目 |
| `isH5()` | 判断是否为 H5 项目 |
| `isDev()` | 判断是否为开发模式 |
| `getHash(str)` | 基于 `string-hash` 生成 6 位短 hash |
| `camelize(pkgName)` | 将包名转换为驼峰命名（如 `@sailorgundam/gundam-core` → `GundamCore`） |

#### `src/utils/sourcemap.ts` — Source Map 解析

解析 SourceMap 并还原原始代码位置，用于线上错误定位：

| 函数 | 说明 |
|------|------|
| `lookupSourceMap(line)` | 根据产物行号查找源码位置，返回 `{ source, line, codeFrame }` |
| `lookupError(error)` | 解析 Native 错误堆栈，将产物行号映射回源码位置 |

#### `src/utils/print.ts` — CLI 输出

基于 `chalk` 的彩色终端输出工具：

| 函数 | 说明 |
|------|------|
| `print(content, color)` | 彩色输出（支持 chalk 颜色名） |
| `print.error(message)` | 红色错误输出（🚨 前缀） |
| `print.warn(message)` | 黄色警告输出（⚠️ 前缀） |

#### `src/utils/find-pkg.ts` — 包路径查找

带缓存的包目录路径查找器：

| 函数 | 说明 |
|------|------|
| `findPkgDirPath(pkgName, prePkgs?)` | 查找指定包的目录路径，支持通过父包路径链式解析，结果缓存在 `pkgPathCache` 中 |

#### `src/utils/prepare-babel-plugin-istanbul.ts` — 代码覆盖率

Istanbul 代码覆盖率插件配置：

| 函数 | 说明 |
|------|------|
| `prepareBabelPluginIstanbul()` | 返回 `@waimai/babel-plugin-istanbul` 插件和 `.istanbul.json` 配置文件生成器 |

默认 Istanbul 配置排除：`coverage/`、`test/`、`node_modules/`、`mock/`、`*.d.ts` 等。

---

### 运行时基座代码

#### `src/temp/base-h5.ts` — H5 基座运行时

H5 环境的基座代码，注入到所有 H5 产物中：

| 模块 | 说明 |
|------|------|
| `Mach` 全局对象 | 环境信息（appName/version/osName/uuid/screenWidth 等）、事件系统 |
| `Mach.env` | 设备环境变量，从 URL 参数或 KNB 桥接获取 |
| `Mach.on`/`Mach.emit` | 事件注册和触发 |
| `KNB` 桥接 | 封装 Native 桥接调用 |

#### `src/temp/base-wechat.ts` — 微信基座运行时

微信小程序的基座代码：
- 提供 `window`/`document` 模拟对象
- 实现 App/Page 生命周期管理
- 开发模式下添加 console 增强

#### `src/temp/base-r-wechat.ts` — R 环境微信运行时

R 环境（特殊部署环境）的微信基座代码，与 `base-wechat.ts` 类似但增加了环境适配。

#### `src/temp/speedTest.ts` — DOM 操作计时

性能测试工具，监控 DOM 操作的执行频率和渲染时间：
- 拦截 `appendChild`/`insertBefore`/`removeChild`/`setAttribute` 等 DOM 方法
- 统计每个方法的调用次数
- 计算 DOM 操作完成后的渲染耗时

#### `src/temp/elementFactoryNative.ts` — Native 元素工厂

Native 环境下的 DOM 元素工厂函数：
- `elementFactory(elmName, Element)` — 创建特定标签的元素实例
- 支持 `button`（带 disabled 状态检测）等特殊元素

#### `sourcemap.js` — Source Map 工具

独立的 SourceMap 解析脚本，用于开发调试时定位编译产物对应的源码位置。

---

### 模板文件

#### `template/` 目录 — 微信小程序模板

标准微信小程序组件模板：

| 文件 | 说明 |
|------|------|
| `index.js` | 小程序入口逻辑 |
| `index.json` | 小程序配置 |
| `index.wxml` | 小程序模板 |
| `runtime-config.js` | 运行时配置 |
| `component-class.js` | 组件 CSS Module Handlebars 模板 |
| `class.js` | 页面 CSS Module Handlebars 模板 |
| `custom-component/` | 自定义组件模板（含 list 组件） |

#### `r-template/` 目录 — R 环境模板

R 环境微信小程序模板：

| 文件 | 说明 |
|------|------|
| `index.js` | R 环境入口逻辑 |
| `index.json` | 配置 |
| `index.wxml` | 模板 |
| `runtime-config.js` | 运行时配置 |

---

## 依赖关系

### 核心 Rollup 插件链

```
rollup-plugin-multil-import          ← 多态文件
rollup-plugin-postcss                ← CSS 处理
  ├── post-css-pxtransform           ←   单位转换
  ├── post-css-polyfill              ←   环境差异
  ├── post-css-font-family-transform ←   字体替换
  ├── post-css-attribute-prefix      ←   属性前缀
  ├── css-module                     ←   CSS Module
  └── postcss-rtlcss                 ←   RTL 支持
@rollup/plugin-json                  ← JSON 导入
rollup-plugin-tsconfig-paths         ← TS 路径
@rollup/plugin-replace               ← 环境变量
@rollup/plugin-commonjs              ← CJS 兼容
rollup-plugin-node-polyfills         ← Node Polyfills
@rollup/plugin-node-resolve          ← 模块解析
@rollup/plugin-typescript            ← TS 编译
rollup-plugin-assets                 ← 资源处理
  └── babel-plugin-assets            ←   图片路径变换
      ├── Cache.ts                   ←     MD5 缓存
      └── image.ts                   ←     Venus CDN 上传
@rollup/plugin-babel                 ← Babel 转换
  ├── babel-plugin-css-module        ←   JSX className
  └── babel-plugin-font-family-transform ← JS 字体替换
rollup-plugin-cleanup                ← 代码清理
rollup-plugin-terser                 ← 代码压缩
rollup-plugin-visualizer             ← 依赖分析
```

### Vite 插件链

```
dynamicHtmlConfig                    ← HTML 模板
@rollup/plugin-node-resolve          ← 模块解析
vite-plugin-commonjs                 ← CJS 兼容
virtualBaseCode                      ← 虚拟基座模块
rollup-plugin-multil-import          ← 多态文件
machProReact                         ← JSX 转译 + 路由注入
machProConfig                        ← Vite 配置合并
rollup-plugin-tsconfig-paths         ← TS 路径
```

### 外部依赖

| 包名 | 版本 | 用途 |
|------|------|------|
| `rollup` | 2.52.3 | Rollup 核心 |
| `vite` | 2.9.14 | Vite 开发服务 |
| `@babel/core` | ^7.13.5 | Babel 核心 |
| `typescript` | ^4.4.3 | TypeScript 编译 |
| `@mtfe/venus` | 0.0.13 | 图片 CDN 上传 |
| `@wmfe/mach-pro-preact` | 3.3.29-next.84 | Preact 兼容层 |
| `@wmfe/mach-pro-render` | 3.3.29-next.84 | Native 渲染引擎 |
| `postcss` | ^8.2.6 | PostCSS 核心 |
| `jszip` | ^3.10.0 | ZIP 打包 |
| `chalk` | ^4.1.0 | 终端彩色输出 |
| `cheerio` | 1.0.0-rc.12 | HTML 解析 |
| `mustache` | ^4.2.0 | 模板渲染 |
| `source-map-js` | ^0.6.2 | SourceMap 解析 |
