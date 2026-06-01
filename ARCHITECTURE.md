# Sailor Gundam V2 MachPro 项目架构文档

> 本文档旨在为 AI Agent 提供足够的信息，使其能够从零克隆一个功能等价的项目。

---

## 1. 项目概览

**项目名称**: `sailor_gundam_v2_machpro`  
**定位**: 美团外卖高达（Gundam）活动页跨端开发框架，基于 MachPro 跨端引擎  
**核心能力**: 提供活动页的组件开发、构建、发布、运行时渲染的完整工具链  
**运行平台**: H5（浏览器）、Native（iOS/Android via MachPro）、微信小程序  
**包管理**: pnpm workspace monorepo  
**Node 版本**: >=14.0.0 <15.0.0  
**包管理器**: pnpm@7.33.5  

---

## 2. 顶层目录结构

```
sailor_gundam_v2_machpro/
├── packages/                    # 所有子包
├── patches/                     # pnpm patch 补丁
├── scripts/                     # 顶层构建/发布脚本
├── package.json                 # 根 package.json
├── pnpm-workspace.yaml          # workspace 配置
├── pnpm-lock.yaml
└── tsconfig.root.json           # 根 TypeScript 配置
```

### 2.1 顶层 scripts/ 目录

| 脚本文件 | 功能 |
|---------|------|
| `publish.js` | 发布 npm 包（支持 --tag=latest/beta/test） |
| `publish-cdn.js` | 发布 CDN 资源 |
| `commit-after-publish.js` | 发布后自动提交版本变更 |
| `sync-pkg-versions.js` | 同步各包版本号 |
| `set-publishable-pkg-versions.js` | 设置可发布包的版本 |
| `checkBranch.js` | 检查当前分支是否允许发布 |
| `lib-build.js` | 通用构建脚本 |
| `cdn.js` / `lion.js` | CDN 和 Lion 配置中心操作 |
| `addOwner.js` | npm 包添加 owner |
| `get-cli-pkg-version.js` | 获取 CLI 包版本 |
| `run-command.js` | 通用命令执行器 |
| `update-core-cdn-deps.js` | 更新核心 CDN 依赖 |

---

## 3. 包依赖关系图

```
                          ┌─────────────────┐
                          │  gundam-cli     │  ← 用户入口 (gundam 命令)
                          └────────┬────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
              ┌─────▼─────┐ ┌─────▼─────┐ ┌─────▼──────────┐
              │gundam-    │ │gundam-    │ │gundam-         │
              │h5-build   │ │generator  │ │js-to-groovy    │
              └─────┬─────┘ └───────────┘ └────────────────┘
                    │
              ┌─────▼──────────────────────────────────┐
              │           gundam-core (核心运行时)        │
              │  ┌──────────┬──────────┬──────────┐    │
              │  │gdAjax    │gdLog     │gdMonitor │    │
              │  │gdStore   │gdBridge  │gdI18n    │    │
              │  │gdLink    │gdStorage │gdFeature │    │
              │  │gdUtil    │CompLoader│PagePrePro│    │
              │  └──────────┴──────────┴──────────┘    │
              └──┬────────┬────────┬────────┬──────────┘
                 │        │        │        │
    ┌────────────▼┐  ┌────▼────┐  ┌▼──────┐ ┌▼──────────────┐
    │gundam-store │  │gundam-  │  │gundam-│ │gundam-preload-│
    │(Rematch)    │  │utils    │  │multi- │ │api-register   │
    └─────────────┘  └─────────┘  │ports  │ └───────────────┘
                                  └───┬───┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    │                 │                  │
              ┌─────▼─────┐    ┌─────▼─────┐    ┌──────▼──────┐
              │gundam-ui  │    │gundam-seg-│    │gundam-base- │
              │(UI组件库) │    │logic      │    │components   │
              └───────────┘    │(供给逻辑) │    └─────────────┘
                               └───────────┘

    ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
    │gundam-preact     │  │gundam-dom-api    │  │gundam-node-     │
    │(Preact兼容层)    │  │(DOM API)         │  │polyfills        │
    └──────────────────┘  └──────────────────┘  └──────────────────┘

    ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
    │gundam-launch-core│  │gundam-mach-inject│  │gundam-playcenter│
    │(投放核心)        │  │(Mach注入)        │  │-api             │
    └──────────────────┘  └──────────────────┘  └──────────────────┘

    ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
    │gundam-recycle-   │  │gundam-recycle-   │  │gundam-html-     │
    │list              │  │list-ctx          │  │template-react   │
    │(虚拟滚动列表)   │  │(列表上下文)      │  │(HTML模板)       │
    └──────────────────┘  └──────────────────┘  └──────────────────┘

    ┌──────────────────┐  ┌──────────────────┐
    │gundam-common-core│  │gundam-build-     │
    │(公共核心类型)    │  │server            │
    └──────────────────┘  │(构建服务端)      │
                          └──────────────────┘
```

---

## 4. 各包功能详解

### 4.1 `@sailorgundam/gundam-core` — 核心运行时

**版本**: 0.149.0  
**定位**: 整个框架的核心，提供所有运行时能力  
**架构**: 双端分离架构（Native/H5），统一接口层

| 子模块 | 功能 | Native实现 | H5实现 |
|--------|------|-----------|--------|
| `gdAjax` | 网络请求 | 基于 Mach 模块 | 基于 axios |
| `gdLog` | 埋点日志 | 基于 Statistics 模块 | 基于 LXAnalytics |
| `gdMonitor` | 监控 | Owl + Logan + SpeedReport | window.Owl + Logan |
| `gdStore` | 状态管理 | Rematch store | Rematch store |
| `gdBridge` | Native桥接 | mach-pro-knb | window.KNB |
| `gdI18n` | 国际化 | i18n-mach-pro | i18n-web |
| `gdLink` | 路由跳转 | Router 模块 | window.location |
| `gdStorage` | 存储 | Cookie + LocalStorage | tiny-cookie + localStorage |
| `gdFeature` | 设备能力 | Compass + SLCommon | navigator + location |
| `gdUtil` | 工具函数 | 通用实现 | 通用实现 |
| `CompLoader` | 组件加载 | AsyncLoadComp HOC | — |
| `PagePreprocessor` | 页面数据预处理 | 含 H5 降级机制 | — |

**关键设计模式**:
- 接口-实现分离: 所有模块在 `_interface/` 定义接口，`native/` 和 `h5/` 分别实现
- 单例模式: 所有核心模块全局唯一
- Core 中控台: NativeCore/H5Core 组合所有子模块
- GdcHOC 高阶组件: 自动注入埋点、错误边界、首屏曝光
- 拦截器链: Ajax 请求经过多个拦截器处理
- 19 个测速点: 精确测量首屏各阶段耗时

**关键依赖**:
- `@sailorgundam/gundam-store` (状态管理)
- `@sailorgundam/gundam-multiports` (跨端组件)
- `@sailorgundam/gundam-utils` (工具函数)
- `@sailorgundam/gundam-preload-api-register` (预加载)
- `@wmfe/mach-pro-knb` (Native桥接)
- `axios` (H5 HTTP)
- `i18next` + `react-i18next` (国际化)

---

### 4.2 `@sailorgundam/gundam-cli` — 命令行工具

**版本**: 0.147.0  
**二进制**: `gundam`  
**基于**: yargs

| 命令 | 别名 | 功能 |
|------|------|------|
| `gundam init [appName]` | `i` | 交互式创建新项目（page/component） |
| `gundam dev` | `D` | 启动开发服务器（H5/MachPro） |
| `gundam build` | `B` | 构建项目（ESM+CJS+资源+描述符） |
| `gundam publish` | `p` | 发布到 mnpm 仓库 |
| `gundam dsl` | `d` | 初始化 DSL 配置文件 |
| `gundam i18nSync` | `i18n` | 同步国际化资源 |

**开发模式**:
- H5: Vite dev server + Preact 替代 React
- Native: mach-pro dev (通过 node-pty 启动)，自动检测 QR 码并打开 iOS 模拟器

**构建模式**:
- Native: tsc 编译 → ESM + CJS → copyfiles 资源 → 生成 gdc-description.json
- H5: 委托给 gundam-h5-build

---

### 4.3 `@sailorgundam/gundam-h5-build` — H5 构建工具

**版本**: 0.148.0  
**定位**: 多端构建系统（H5/小程序/Native）  
**核心引擎**: Rollup + Vite 双引擎

**构建流程 (Rollup)**:
```
入口 → 多态文件解析 → CSS处理 → JSON解析 → TS路径解析 → 
环境变量替换 → CommonJS转换 → Node polyfills → 模块解析 → 
TypeScript编译 → 资源处理 → Babel(JSX/字体/CSS Module) → 
代码清理 → 压缩 → 依赖分析可视化 → 输出
```

**构建流程 (Vite/H5开发)**:
```
入口HTML → HTML模板渲染 → 多态文件 → JSX转译+路由注入 → 
Vite配置合并 → TS路径别名 → PostCSS处理 → 输出
```

**关键插件**:
- `rollup-plugin-multil-import`: 多态协议（.h5.js/.native.js/.mp.js）
- `rollup-plugin-css`: CSS处理（PostCSS + px转换 + RTL + 字体替换 + CSS Module）
- `rollup-plugin-assets`: 资源处理（CDN上传 + base64内联）
- `rollup-plugin-compile-output`: 产物后处理（CSS→JSON→Bytecode→zip）
- `rollup-plugin-injectjs`: 代码注入（MACH_PROJECT_CONFIG + 基础运行时）
- `rollup-plugin-main/sub`: 入口代码注入和子包处理

**输出格式**: UMD (component/library/pageContainer) / IIFE (page)

---

### 4.4 `@sailorgundam/gundam-build-server` — 构建服务端

**定位**: 基于 Egg.js 的 CI/CD 服务  
**端口**: 8002  
**Worker 数**: 8  
**部署**: CentOS 7 + Node 14.19.0

**核心功能**:
- 自动化 CI/CD 流水线（源码 → CDN产物 + DD平台Bundle）
- 多机房支持（HK/EU）
- Lion 配置中心集成
- Mustache 模板生成项目脚手架
- 调用 gundam/mach-pro/pnpm/mnpm 执行编译和发布

---

### 4.5 `@sailorgundam/gundam-js-to-groovy` — JS转Groovy转换器

**版本**: 0.67.0  
**定位**: 将 JavaScript DSL 配置转换为 Groovy 脚本  
**基于**: Babel AST

**两阶段转换**:
1. `JSToGroovyTranspiler`: JS AST → Groovy 代码
2. `GroovyPostProcessor`: 后处理（添加 extends Script、转换 run 方法参数为 binding.getVariable）

**关键语法转换**:
- `===` → `==`，`!==` → `!=`
- `||` → `?:` (Elvis)
- 箭头函数 → Groovy闭包
- `JSON.parse()` → `JsonSlurper().parseText()`
- `.map()` → `.collect()`
- `.includes()` → `.contains()`
- `Date.now()` → `System.currentTimeMillis()`

**CLI**: `js2groovy <input> [-o output] [-v]`

---

### 4.6 `@sailorgundam/gundam-generator` — 项目脚手架

**版本**: 0.148.0  
**定位**: 初始化新的 Gundam 组件项目

**流程**: 复制模板 → Mustache变量替换 → 按条件删除/重命名文件 → pnpm install → i18n同步 → Git初始化 + Husky

**支持的组件变体**:
- Class/Function × 普通/供给(supply)
- 可选: i18n、数据提取、投放(launch)、接口聚合(aggregated/DSL)

---

### 4.7 `@sailorgundam/gundam-store` — 状态管理

**版本**: 0.149.0  
**定位**: 基于 Rematch v2 模式的状态管理库  
**核心**: Model驱动的 Redux 封装

**API**:
- `init(models)`: 初始化 store
- `connect(mapStateToProps, mapDispatchToProps)`: React HOC 连接
- `addModel(model)`: 动态注册 model
- `reset(name)`: 重置 model 状态
- `createModel`: 类型安全的 model 创建工具

**特性**:
- 全局单例 (`globalThis.__GUNDAM_PRO_STORE__`)
- 自实现 connect HOC（不依赖 react-redux 运行时）
- Memoized computed selectors
- 支持 ESM/CJS/UMD 三种格式

---

### 4.8 `@sailorgundam/gundam-utils` — 工具函数库

**版本**: 0.150.0  
**定位**: 基础工具函数集

**核心模块**:
| 模块 | 功能 |
|------|------|
| `getJumpUrl` | URL跳转处理（透传UTM/泳道/业务参数） |
| `qs` | URL参数解析（支持Mach环境） |
| `querystring-es6` | URL编解码 |
| `classNames` | CSS类名拼接 |
| `compareVersions` | 语义化版本比较 |
| `deepEqual` | 深度相等比较 |
| `throttle` | 节流函数 |
| `isH5` | 运行时环境判断 |
| `isValidUrl` | URL合法性校验 |
| `jsonSafeParse` | 安全JSON解析 |
| `dp2Rpx` | 单位转换 |
| `useRequest` | 异步请求Hook |
| `useAsyncEffect` | async Effect Hook |

---

### 4.9 `@sailorgundam/gundam-multiports` — 跨端UI组件

**版本**: 0.149.0  
**定位**: 跨平台UI基础组件库（H5 + Native双端实现）

**组件列表**:
| 组件 | H5实现 | Native实现 |
|------|--------|-----------|
| `View` | `<div>` | `<view>`/`<exposure>` |
| `Text` | `<p><span>` | `<text>` |
| `Image` | `<div>`+bgImage | `<image>` |
| `ImageBackground` | View+Image组合 | `<image-background>` |
| `Scroller` | `<div>`+overflow:scroll | `<scroller>` |
| `Modal` | View+body scroll lock | `<modal>` |
| `RichText` | 遍历渲染Text/Image | `<rich-text>` |
| `Lottie` | lottie-web (CDN) | `<lottie>` |
| `Vap` | LingboEffects (CDN) | `<infinite-effect>` |
| `Fixed` | View+fixed/absolute | View+Portal |
| `Portal` | Root-level渲染 | Root-level渲染 |

**关键设计模式**:
- 双端实现: 每个组件有 `index.h5.tsx` 和 `index.native.tsx`
- 环境选择: `process.env.MACH_ENV` 决定加载哪端实现
- 曝光检测: IntersectionObserver(H5) + polling(Native)
- 点击归一化: h5HandleClickEvent / machHandleClickEvent
- Measure API Polyfill: H5 DOM 添加 measure/measureInWindow 方法

---

### 4.10 `@sailorgundam/gundam-ui` — UI组件库

**版本**: 0.149.0  
**定位**: 基于设计规范的业务UI组件库

**组件列表**:
| 组件 | 功能 |
|------|------|
| `Button` | 按钮（5种尺寸 + 3种类型 + 箭头） |
| `Card` | 卡片容器（3种尺寸 + logo + 内容 + 底部） |
| `Description` | 描述文本（4种尺寸 + 8种灰度） |
| `HighTag` | 高亮标签（好评/排名/VIP） |
| `Image` | 增强图片（标签覆盖 + 状态覆盖） |
| `Layout` | 两栏布局（6种尺寸） |
| `LongTag` | 长标签（blur/highlight + 5种颜色） |
| `Modal` | 对话框（标题+消息+按钮+关闭+警告样式） |
| `Price` | 价格展示（现价/原价 + 5种尺寸 + 整数/小数分离） |
| `PriceGroup` | 价格组布局（4种方向） |
| `Row` | 水平Flex布局（对齐+间距+末尾省略） |
| `Score` | 评分展示（星级+分数） |
| `ShortTag` | 短标签（5种状态 + 8种业务类型） |
| `ShortTagGroup` | 标签组（无缝拼接 + 分隔线模式） |
| `Switch` | 开关（3种尺寸） |
| `Title` | 标题（5种尺寸 + 省略 + 双行） |
| `Toast` | 提示（命令式API + 自动隐藏） |
| `Border` | 虚线边框（Mach不支持CSS虚线的替代方案） |

**设计规范**: 所有组件使用统一的设计 Token（颜色/字号/字重/动画/圆角）

---

### 4.11 `@sailorgundam/gundam-seg-logic` — 供给组件逻辑

**版本**: 0.149.0  
**定位**: 供给组件的数据获取、渲染、筛选、跳转核心逻辑

**核心流程**:
1. `createLogicModel()` 创建全局逻辑 model → `wnInit` 获取 segmentIds
2. 组件 `getProvideData` → `checkWnSegmentIds` 判断召回
3. `getWnSegment` 执行数据请求（最多3次）→ filter → format → 数据适配
4. 图片优化: H5 自动 webp + 尺寸调整
5. 交互: handleClick → go() 跳转；handleView → 埋点

**关键组件**:
- `List`: 列表组件（分页/自动加载/骨架屏/筛选/懒渲染）
- `ProvideList`: 旧版供给列表容器
- `FilterBar`: 筛选栏（Button/DropDown样式）
- `ComponentTitle`: 标题组件
- `Skeleton`: 骨架屏

**Hook系统**: SyncWaterFallHook 实现的 filter/format/mutation/optControl 四类 Hook

---

### 4.12 `@sailorgundam/gundam-preact` — Preact兼容层

**定位**: 在 H5 构建中替代 React，减小包体积  
**内容**: 基于 Preact 的 compat 兼容层，支持 React API  
**使用**: 通过 Vite/Rollup 别名将 `react`/`react-dom` 指向此包

---

### 4.13 `@sailorgundam/gundam-dom-api` — DOM API

**定位**: 提供 MachPro Native 环境下的 DOM 操作 API  
**内容**: document.createElement / setAttribute / addEventListener 等 DOM 标准API的 Native 实现

---

### 4.14 `@sailorgundam/gundam-node-polyfills` — Node.js Polyfills

**版本**: 0.2.4  
**定位**: 为浏览器环境提供 Node.js 核心模块的 polyfill  
**内容**: buffer, process, stream, util, path, url, events 等模块的浏览器兼容实现  
**构建**: Rollup 打包

---

### 4.15 `@sailorgundam/gundam-base-components` — 基础组件

**定位**: 提供通用基础组件（如 StickyView 粘性定位等）  
**依赖**: gundam-core, gundam-multiports

---

### 4.16 `@sailorgundam/gundam-common-core` — 公共核心类型

**定位**: 跨包共享的 TypeScript 类型定义和常量  
**内容**: 编译产物（dist/cjs/types），无独立 src 目录

---

### 4.17 `@sailorgundam/gundam-launch-core` — 投放核心

**定位**: 投放(launch)组件的运行时逻辑  
**内容**: 编译产物格式

---

### 4.18 `@sailorgundam/gundam-mach-inject` — Mach注入

**定位**: MachPro 框架的运行时注入代码  
**内容**: 单文件包，构建时注入到 Native 产物中

---

### 4.19 `@sailorgundam/gundam-playcenter-api` — 玩法中心API

**定位**: 与 Gundam 玩法中心后端交互的 API 封装

---

### 4.20 `@sailorgundam/gundam-preload-api-register` — 预加载API注册

**版本**: 0.121.0  
**定位**: 注册预加载 API 配置，用于接口聚合和预请求  
**构建**: Rollup

---

### 4.21 `@sailorgundam/gundam-recycle-list` — 虚拟滚动列表

**定位**: 高性能虚拟滚动列表组件  
**内容**: 编译产物格式，支持大量数据的高效渲染

---

### 4.22 `@sailorgundam/gundam-recycle-list-ctx` — 列表上下文

**定位**: 虚拟滚动列表的上下文管理  
**内容**: 编译产物格式

---

### 4.23 `@sailorgundam/gundam-html-template-react` — HTML模板

**定位**: H5 页面的 HTML 外壳模板  
**内容**: 
- `index.html`: 开发用 HTML 壳
- `main.ftl` / `main-eu.ftl`: FreeMarker 模板（线上渲染）
- `mini-html.js`: 小程序 HTML 模板
- `publish.js`: 发布脚本

---

## 5. 核心数据流

### 5.1 组件开发流程

```
gundam init → 生成组件项目 → 编写组件代码 → 
gundam dev (H5/Native) → 调试 → 
gundam build → ESM + CJS + gdc-description.json → 
gundam publish → mnpm 仓库
```

### 5.2 页面运行时流程

```
页面数据加载 → PageDataPreprocessor(预处理+H5降级判断) → 
Store初始化(globalData/pageConfig/renderTree) → 
CompLoader(动态加载组件Bundle) → 
GdcHOC(注入埋点+错误边界) → 渲染组件树 → 
首屏曝光完成 → 上报测速点
```

### 5.3 供给组件数据流

```
createLogicModel → wnInit(获取segmentIds) → 
组件getProvideData → checkWnSegmentIds(判断召回) → 
getWnSegment(请求数据) → filter Hook → format Hook → 
dataAdapter(适配数据模型) → setDataList → 渲染列表
```

### 5.4 构建流程

```
源码(TSX/TS) → Rollup/Vite构建 → 
  ├─ H5: Preact替代React + 多态文件 + CSS处理 + 代码注入 → bundle.js + bundle.css
  ├─ Native: TypeScript编译 + CSS→JSON + Bytecode编译 + zip打包
  └─ 小程序: ES6→ES5 + 变量名压缩 + wxml/wxss模板
```

---

## 6. 关键架构决策

1. **双端分离**: Native 和 H5 的实现完全分离，通过接口抽象统一
2. **Preact替代**: H5 环境用 Preact 替代 React，显著减小包体积
3. **Rematch状态管理**: 自实现轻量版 Rematch，不依赖 redux 运行时
4. **多态文件协议**: `.h5.js` / `.native.js` / `.mp.js` 后缀自动选择对应端实现
5. **GdcHOC埋点**: 所有组件通过高阶组件自动注入埋点和错误边界
6. **拦截器链**: Ajax 请求经过请求→泳道→URL→指纹→公共参数拦截器
7. **H5降级机制**: Native 端加载失败时自动降级到 H5 页面
8. **19个测速点**: 精确追踪首屏各阶段耗时
9. **JS-to-Groovy**: 前端用 JS 编写 DSL，服务端用 Groovy 执行
10. **虚拟滚动**: 长列表使用 RecycleList 实现高性能渲染

---

## 7. 技术栈总结

| 层面 | 技术 |
|------|------|
| 框架 | MachPro (跨端), React-like (Preact/React) |
| 状态管理 | 自实现 Rematch v2 |
| 构建 | Rollup + Vite + TypeScript |
| 样式 | Less + PostCSS + CSS Module + px2rpx |
| 埋点 | LXAnalytics (灵犀) + Owl (监控) + Logan (日志) |
| 国际化 | i18next + react-i18next |
| Native桥接 | mach-pro-knb |
| 路由 | Mach Router (Native) / history (H5) |
| 发布 | mnpm (内部npm) + CDN + DD平台 |
| CI/CD | Egg.js 构建服务 |
| AST转换 | Babel (@babel/parser + @babel/traverse) |
