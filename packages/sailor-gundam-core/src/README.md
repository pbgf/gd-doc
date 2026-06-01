# src/ 目录概述

Gundam Core 的源码目录，采用**接口-实现分离**的三层架构。

## 目录结构

```
src/
├── _interface/      # 接口定义层
├── native/          # Native 端实现层
├── h5/              # H5 端实现层
└── common/          # 共享代码层
```

## 架构分层

### 1. 接口定义层 (`_interface/`)

定义所有核心模块的 TypeScript 接口，是 Native 和 H5 实现的契约。包含 17 个接口文件，覆盖：

- **核心接口**: `Icore` - 13 个属性/方法，是所有子模块的聚合入口
- **功能接口**: `IAjax`, `IStorage`, `I18N`, `ILog`, `IMonitor`, `IBridge`, `IFeature`, `ILink`, `IUtil`, `IEnv`
- **辅助接口**: `IEventBus`, `IFingerprint`, `ICompLoader`
- **类型定义**: `i-share.ts`, `i-page-data.ts`, `i-global-data.ts`

详见 [_interface/README.md](./_interface/README.md)

### 2. Native 实现层 (`native/`)

基于 MachPro 跨端引擎的 Native 端实现。每个接口对应一个实现模块，部分模块（Ajax、Feature、Monitor、Storage、CompLoader、PageDataPreprocessor）因逻辑复杂拆分为子目录。

关键特点：
- 桥接层使用 `@wmfe/mach-pro-knb`
- 国际化使用 `i18n-mach-pro`
- 埋点使用 `Statistics` 模块
- 监控使用 Owl + Logan + SpeedReport
- 包含 H5 降级机制（Native 加载失败时降级到 H5）
- 组件动态加载能力（AsyncLoadComp HOC）

详见 [native/README.md](./native/README.md)

### 3. H5 实现层 (`h5/`)

基于浏览器环境的 H5 端实现。结构比 Native 层简单，大部分模块为单文件实现。

关键特点：
- 网络请求使用 `axios`
- 桥接使用 `window.KNB`
- 国际化使用 `i18n-web` + 阿拉伯字体动态加载
- 埋点使用 `LXAnalytics`
- 监控使用 `window.Owl` + Logan
- 存储使用 `tiny-cookie` + `localStorage`
- 无组件动态加载和页面数据预处理（这些仅 Native 需要）

详见 [h5/README.md](./h5/README.md)

### 4. 共享代码层 (`common/`)

两端共用的工具函数、组件和状态模型。不依赖特定平台 API。

关键特点：
- `GdcHOC` 高阶组件工厂：自动注入曝光/点击埋点和错误边界
- `EventBus` 全局事件总线
- 监控组件（Image/Lottie/Vap 加载监控）
- `compModel` 组件数据获取 Rematch model
- 通用工具函数（qs、deep-merge、dp2rpx 等）

详见 [common/README.md](./common/README.md)

## 层间依赖关系

```
                    _interface/ (接口定义)
                    ↑           ↑           ↑
                   实现         实现        使用
                    │           │           │
              native/          h5/        common/
                │               │           │
                └───────┬───────┘           │
                        │                   │
                        └─── 互相引用 ───────┘
```

- `native/` 和 `h5/` 都实现 `_interface/` 中的接口
- `common/` 中的代码被 `native/` 和 `h5/` 共享使用
- `common/` 中的类型可能引用 `_interface/` 的定义
- `native/` 和 `h5/` 之间不直接互相引用

## 入口文件

### Native 入口

```typescript
// native/Core.ts
class NativeCore implements Icore {
    ajax: IAjax;
    bridge: IBridge;
    i18n: I18N;
    log: ILog;
    monitor: IMonitor;
    storage: IStorage;
    feature: IFeature;
    link: ILink;
    util: IUtil;
    env: IEnv;
    eventBus: IEventBus;
    fingerprint: IFingerprint;
    compLoader: ICompLoader;
    // ... 初始化和 renderPage()
}
```

### H5 入口

```typescript
// h5/Core.ts
class H5Core implements Icore {
    ajax: IAjax;
    bridge: IBridge;
    i18n: I18N;
    log: ILog;
    monitor: IMonitor;
    storage: IStorage;
    feature: IFeature;
    link: ILink;
    util: IUtil;
    env: IEnv;
    eventBus: IEventBus;
    // renderPage() 为空实现
}
```

## 构建与多态

构建时通过多态文件协议（`.h5.ts`/`.native.ts`）选择对应端的实现：

```typescript
// 使用时
import Core from 'gundam-core/Core';  // 构建工具自动选择 .h5.ts 或 .native.ts
```

环境判断：
- `process.env.MACH_ENV` 决定加载哪端实现
- 运行时通过 `env.isH5` / `env.isNative` 区分平台
