# @sailorgundam/gundam-core

**版本**: 0.149.0  
**定位**: Gundam 活动页框架的核心运行时，提供所有组件和页面的基础能力

## 概述

`gundam-core` 是整个 Gundam 活动页框架的中枢，为组件开发、页面渲染、数据获取、状态管理等提供完整的运行时支持。采用**双端分离架构**（Native/H5），通过统一的接口层抽象，使上层业务代码无需关心底层平台差异。

## 架构设计

```
┌─────────────────────────────────────────────────────────┐
│                    gundam-core                          │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │              _interface/ (接口定义层)               │  │
│  │  ICore · IAjax · IStorage · I18N · ILog · IMonitor│  │
│  │  IBridge · IFeature · ILink · IUtil · IEnv · ... │  │
│  └─────────────┬─────────────────┬───────────────────┘  │
│                │                 │                       │
│  ┌─────────────▼───────┐ ┌──────▼──────────────────┐   │
│  │   native/ (Native端) │ │    h5/ (H5端)           │   │
│  │  NativeCore          │ │  H5Core                 │   │
│  │  mach-pro-knb        │ │  window.KNB             │   │
│  │  i18n-mach-pro       │ │  i18n-web               │   │
│  │  Statistics          │ │  LXAnalytics            │   │
│  │  Owl + Logan         │ │  window.Owl + Logan     │   │
│  │  Router              │ │  history                │   │
│  └─────────────────────┘ └─────────────────────────┘   │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │              common/ (共享代码层)                   │  │
│  │  GdcHOC · EventBus · compModel · utils · ...      │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 设计原则

1. **接口-实现分离**: 所有模块在 `_interface/` 定义 TypeScript 接口，`native/` 和 `h5/` 分别实现
2. **单例模式**: 所有核心模块（Ajax、Log、Monitor、Bridge 等）全局唯一实例
3. **Core 中控台**: `NativeCore`/`H5Core` 组合所有子模块，对外提供统一入口
4. **拦截器链**: Ajax 请求经过多个拦截器（泳道、URL、指纹、公共参数等）处理
5. **19 个测速点**: 精确测量首屏各阶段耗时，用于性能监控

## 目录结构

```
sailor-gundam-core/
├── src/
│   ├── _interface/          # 接口定义层 - 所有模块的 TypeScript 接口
│   │   ├── i-core.ts        #   ICore 核心接口
│   │   ├── i-ajax.ts        #   IAjax 网络请求接口
│   │   ├── i-storage.ts     #   IStorage 存储接口
│   │   ├── i-i18n.ts        #   I18N 国际化接口
│   │   ├── i-log.ts         #   ILog 埋点接口
│   │   ├── i-monitor.ts     #   IMonitor 监控接口
│   │   ├── i-bridge.ts      #   IBridge 桥接接口
│   │   ├── i-feature.ts     #   IFeature 设备能力接口
│   │   ├── i-link.ts        #   ILink 路由接口
│   │   ├── i-util.ts        #   IUtil 工具接口
│   │   ├── i-env.ts         #   IEnv 环境变量接口
│   │   ├── i-eventbus.ts    #   IEventBus 事件总线接口
│   │   ├── i-share.ts       #   分享配置接口
│   │   ├── i-fingerprint.ts #   IFingerprint 指纹接口
│   │   ├── i-comp-loader.ts #   ICompLoader 组件加载接口
│   │   ├── i-page-data.ts   #   页面数据类型定义
│   │   ├── i-p-data-pre-processor.ts # 页面数据预处理接口
│   │   └── i-global-data.ts #   全局数据类型定义
│   │
│   ├── native/              # Native 端实现
│   │   ├── Core.ts          #   NativeCore 类
│   │   ├── Ajax/            #   网络请求（DSL聚合+重试+拦截器）
│   │   ├── Bridge.ts        #   Native桥接 (mach-pro-knb)
│   │   ├── CompLoader/      #   组件动态加载
│   │   ├── Env.ts           #   环境检测
│   │   ├── Feature/         #   设备能力聚合
│   │   ├── I18n.ts          #   国际化 (i18n-mach-pro)
│   │   ├── Link.ts          #   路由 (Router模块)
│   │   ├── Log.ts           #   埋点 (Statistics)
│   │   ├── Monitor/         #   监控 (Owl+Logan+SpeedReport)
│   │   ├── Storage/         #   存储 (Cookie+LocalStorage)
│   │   ├── Store/           #   状态管理 (Rematch)
│   │   ├── Util.ts          #   工具函数
│   │   ├── PageDataPreprocessor/  # 页面数据预处理（含H5降级）
│   │   ├── KeetaDesign.ts   #   设计系统
│   │   ├── FingerPrint.ts   #   指纹
│   │   └── utils/           #   Native端辅助函数
│   │
│   ├── h5/                  # H5 端实现
│   │   ├── Core.ts          #   H5Core 类
│   │   ├── Ajax/            #   网络请求 (axios)
│   │   ├── Bridge.ts        #   H5桥接 (window.KNB)
│   │   ├── Env.ts           #   H5环境检测
│   │   ├── Feature/         #   设备能力
│   │   ├── I18n.ts          #   国际化 (i18n-web + 阿拉伯字体)
│   │   ├── Link.ts          #   路由 (history)
│   │   ├── Log.ts           #   埋点 (LXAnalytics)
│   │   ├── Monitor.ts       #   监控 (window.Owl + Logan)
│   │   ├── Storage.ts       #   存储 (tiny-cookie + localStorage)
│   │   ├── Store/           #   状态管理 (Rematch)
│   │   └── Util.ts          #   工具函数
│   │
│   └── common/              # 共享代码
│       ├── utils/           #   通用工具函数
│       ├── components/      #   通用组件（Image/Lottie/Vap监控）
│       └── comp-fetch-store/ #  组件数据获取 model
```

## 核心模块对照表

| 模块 | 功能 | Native 实现 | H5 实现 |
|------|------|-----------|--------|
| `Core` | 中控台，聚合所有模块 | `NativeCore` | `H5Core` |
| `Ajax` | 网络请求 | 基于 Mach 模块，DSL 聚合 | 基于 axios，拦截器链 |
| `Bridge` | Native 桥接 | `mach-pro-knb` | `window.KNB` |
| `I18n` | 国际化 | `i18n-mach-pro` | `i18n-web` + 阿拉伯字体加载 |
| `Log` | 埋点 | `Statistics` 模块 | `LXAnalytics` |
| `Monitor` | 监控 | Owl + Logan + SpeedReport | window.Owl + Logan |
| `Storage` | 存储 | Cookie + LocalStorage | tiny-cookie + localStorage |
| `Feature` | 设备能力 | Compass + SLCommon | navigator + location |
| `Link` | 路由跳转 | Router 模块 | history |
| `Store` | 状态管理 | Rematch store | Rematch store |
| `Util` | 工具函数 | 通用实现 | 通用实现 |
| `CompLoader` | 组件加载 | AsyncLoadComp HOC | — |
| `PageDataPreprocessor` | 页面数据预处理 | 含 H5 降级机制 | — |
| `Env` | 环境检测 | Mach 环境 | 浏览器环境 |

## 关键依赖

| 依赖包 | 用途 |
|--------|------|
| `@sailorgundam/gundam-store` | 基于 Rematch v2 的状态管理 |
| `@sailorgundam/gundam-multiports` | 跨端 UI 组件库 |
| `@sailorgundam/gundam-utils` | 基础工具函数集 |
| `@sailorgundam/gundam-preload-api-register` | 预加载 API 注册 |
| `@wmfe/mach-pro-knb` | Native 桥接 SDK |
| `axios` | H5 HTTP 请求库 |
| `i18next` + `react-i18next` | 国际化框架 |

## 核心数据流

### 页面渲染流程

```
页面数据加载 → PageDataPreprocessor(预处理+H5降级判断) →
Store初始化(globalData/pageConfig/renderTree) →
CompLoader(动态加载组件Bundle) →
GdcHOC(注入埋点+错误边界) → 渲染组件树 →
首屏曝光完成 → 上报测速点
```

### Ajax 请求流程

```
业务调用 → 请求拦截器链(泳道→URL→指纹→公共参数) →
发送请求 → 响应拦截器 → 返回数据
```

## 关键设计模式

### GdcHOC 高阶组件

所有组件通过 `createHOC` 工厂创建的高阶组件包裹，自动注入：
- 曝光埋点（进入视口自动上报）
- 点击埋点
- 错误边界（捕获渲染错误，防止白屏）

### 拦截器链

Ajax 请求按顺序经过以下拦截器：
1. **请求拦截器** (`request-interceptor`) - 通用请求处理
2. **泳道拦截器** (`swimlane-interceptor`) - 泳道/灰度环境路由
3. **URL 拦截器** (`url-interceptor`) - URL 处理和拼接
4. **公共参数拦截器** (`common-interceptors`) - 注入公共参数

### 19 个测速点

通过 `SpeedPoint` 枚举定义 19 个关键时间点，覆盖：
- 页面初始化各阶段
- 接口请求耗时
- 组件加载耗时
- 首屏渲染完成时间

## 子目录详细文档

- [src/ 目录](./src/README.md) - 源码目录概述
- [接口层](./src/_interface/README.md) - 所有接口定义
- [Native 层](./src/native/README.md) - Native 端实现
- [H5 层](./src/h5/README.md) - H5 端实现
- [通用层](./src/common/README.md) - 共享代码
