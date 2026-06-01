# _interface/ — 接口定义层

所有核心模块的 TypeScript 接口定义，是 Native 和 H5 实现的契约层。本目录下的每个文件定义了一个或多个接口/类型，`native/` 和 `h5/` 中的实现类必须遵循这些接口。

## 设计原则

1. **接口与实现完全分离**: 接口定义不包含任何具体实现逻辑
2. **TypeScript 强类型**: 所有接口使用 TypeScript 定义，确保两端实现的类型安全
3. **最小知识原则**: 每个接口只暴露必要的 API，隐藏实现细节
4. **单一职责**: 每个文件对应一个独立的功能域

## 文件清单

### i-core.ts — 核心中控台接口

**功能**: 定义 `Icore` 接口，是所有子模块的聚合入口，Core 类必须实现此接口。

**导出**: `Icore`

**属性/方法** (共 13 个):

```typescript
interface Icore {
    ajax: IAjax;              // 网络请求
    bridge: IBridge;          // Native桥接
    i18n: I18N;               // 国际化
    log: ILog;                // 埋点日志
    monitor: IMonitor;        // 监控
    storage: IStorage;        // 存储
    feature: IFeature;        // 设备能力
    link: ILink;              // 路由跳转
    util: IUtil;              // 工具函数
    env: IEnv;                // 环境变量
    eventBus: IEventBus;      // 事件总线
    fingerprint: IFingerprint; // 设备指纹
    compLoader: ICompLoader;  // 组件加载器
}
```

**实现类**: `NativeCore` (native/Core.ts), `H5Core` (h5/Core.ts)

**依赖**: 引用本目录下所有其他接口类型

---

### i-ajax.ts — 网络请求接口

**功能**: 定义网络请求相关接口，包括标准请求、纯请求和自定义 API 配置。

**导出**:
- `IAjax` — Ajax 请求主接口
- `IAjaxOptions` — 标准 Ajax 请求选项
- `IPureAjaxOptions` — 纯请求选项（无拦截器）
- `ICustomApiConfigBuilder` — 自定义 API 配置构建器

**关键签名**:

```typescript
interface IAjax {
    request(options: IAjaxOptions): Promise<any>;
    pureRequest(options: IPureAjaxOptions): Promise<any>;
    get(url: string, data?: any, options?: Partial<IAjaxOptions>): Promise<any>;
    post(url: string, data?: any, options?: Partial<IAjaxOptions>): Promise<any>;
}

interface IAjaxOptions {
    url: string;
    method?: string;
    data?: any;
    headers?: Record<string, string>;
    timeout?: number;
    withCredentials?: boolean;
    // ... 其他请求配置
}

interface IPureAjaxOptions extends IAjaxOptions {
    // 绕过拦截器链的请求配置
}

interface ICustomApiConfigBuilder {
    // DSL 聚合请求的自定义 API 配置
}
```

**实现类**: Native `Ajax/index.ts`, H5 `Ajax/index.ts`

**关键设计**: 
- `request` 经过完整拦截器链
- `pureRequest` 绕过拦截器，用于底层请求
- `ICustomApiConfigBuilder` 用于 DSL 聚合场景

---

### i-storage.ts — 存储接口

**功能**: 定义 Cookie 和 LocalStorage 存储接口。

**导出**:
- `IStorage` — 存储主接口（聚合 Cookie + LocalStorage）
- `ICookie` — Cookie 操作接口
- `ILocalStorage` — LocalStorage 操作接口
- `ICookieOptions` — Cookie 设置选项

**关键签名**:

```typescript
interface IStorage {
    cookie: ICookie;
    localStorage: ILocalStorage;
}

interface ICookie {
    get(key: string): string | undefined;
    set(key: string, value: string, options?: ICookieOptions): void;
    remove(key: string, options?: ICookieOptions): void;
}

interface ILocalStorage {
    get(key: string): string | null;
    set(key: string, value: string): void;
    remove(key: string): void;
}

interface ICookieOptions {
    expires?: number | Date;
    path?: string;
    domain?: string;
    secure?: boolean;
}
```

**实现类**: Native `Storage/index.ts`, H5 `Storage.ts`

---

### i-i18n.ts — 国际化接口

**功能**: 定义国际化能力，包括资源加载、语言/地区/时区获取、日期格式化。

**导出**: `I18N`

**关键签名**:

```typescript
interface I18N {
    loadResource(options: { ns: string; lng?: string }): Promise<void>;
    getLocale(): string;       // 获取当前语言（如 'zh-CN', 'ar-SA'）
    getRegion(): string;       // 获取当前地区（如 'HK', 'SA', 'AE'）
    getTimezone(): string;     // 获取当前时区
    formatDateTimeByCityId(cityId: number, timestamp: number, format: string): string;
    // i18next 实例也提供 t() 翻译函数
}
```

**实现类**: Native `I18n.ts` (基于 i18n-mach-pro), H5 `I18n.ts` (基于 i18n-web)

**关键设计**: 
- 支持多语言命名空间（ns），按需加载
- 城市ID驱动的时区/日期格式化（适配中东等多时区地区）

---

### i-log.ts — 埋点日志接口

**功能**: 定义基于 LX（灵犀）分析系统的埋点接口。

**导出**:
- `ILog` — 埋点主接口
- `ILxInitOptions` — LX 初始化选项
- `ILxEventOptions` — 事件埋点选项
- `ILxEventOptionsWithBid` — 带 BID 的事件埋点选项

**关键签名**:

```typescript
interface ILog {
    init(options: ILxInitOptions): void;
    sendEvent(options: ILxEventOptions | ILxEventOptionsWithBid): void;
    sendPV(options?: any): void;
    // ... 其他埋点方法
}

interface ILxInitOptions {
    bid?: string;          // 业务ID
    appId?: string;        // 应用ID
    // ... 其他初始化参数
}

interface ILxEventOptions {
    eventId: string;       // 事件ID
    label?: string;        // 事件标签
    params?: Record<string, any>;  // 事件参数
}

interface ILxEventOptionsWithBid extends ILxEventOptions {
    bid: string;           // 业务ID（必填）
}
```

**实现类**: Native `Log.ts` (基于 Statistics 模块), H5 `Log.ts` (基于 LXAnalytics)

---

### i-monitor.ts — 监控接口

**功能**: 定义性能监控和错误上报接口，支持 Owl/Raptor 监控体系。

**导出**:
- `IMonitor` — 监控主接口
- `SpeedPoint` — 测速点枚举（19 个测速点）
- `IOwlError` — Owl 错误上报选项

**关键签名**:

```typescript
interface IMonitor {
    init(options?: any): void;
    reportError(error: Error | IOwlError): void;
    setSpeedPoint(point: SpeedPoint, time?: number): void;
    reportSpeed(): void;
    // ... 其他监控方法
}

enum SpeedPoint {
    // 19 个测速点，覆盖页面加载各阶段
    // 包括: 页面开始、数据加载、组件渲染、首屏完成等
}

interface IOwlError {
    name?: string;
    message: string;
    stack?: string;
    category?: number;
    level?: string;
    tags?: Record<string, string>;
}
```

**实现类**: Native `Monitor/index.ts` (Owl + Logan + SpeedReport), H5 `Monitor.ts` (window.Owl + Logan)

**关键设计**: 
- 19 个测速点精确定位首屏各阶段耗时
- 错误分级上报（warn/error/fatal）
- Logan 本地日志兜底

---

### i-bridge.ts — Native 桥接接口

**功能**: 定义与 Native 客户端通信的桥接接口。

**导出**: `IBridge`

**关键签名**:

```typescript
interface IBridge {
    getBridge(): any;  // 返回 KNB 实例，用于调用 Native 能力
}
```

**实现类**: Native `Bridge.ts` (返回 mach-pro-knb 实例), H5 `Bridge.ts` (返回 window.KNB)

**关键设计**: 
- 统一 KNB 接口，两端获取方式不同但 API 一致
- 通过 KNB 实例可调用：打开页面、分享、获取设备信息等 Native 能力

---

### i-feature.ts — 设备能力接口

**功能**: 定义设备能力聚合接口，覆盖 App 信息、设备信息、定位、登录、分享等。

**导出**: `IFeature`

**关键签名**:

```typescript
interface IFeature {
    getAppInfo(): Promise<AppInfo>;        // 获取应用信息
    getDeviceInfo(): Promise<DeviceInfo>;  // 获取设备信息
    getLocation(): Promise<LocationInfo>;  // 获取定位信息
    getUserInfo(): Promise<UserInfo>;      // 获取用户信息
    login(options?: any): Promise<void>;   // 登录
    share(config: ShareConfig): Promise<ShareResponse>;  // 分享
    scrollToId(id: string): void;          // 滚动到指定元素
    getStrategyAsync(options: any): Promise<any>;  // AB 实验策略
}
```

**实现类**: Native `Feature/index.ts` (Compass + SLCommon), H5 `Feature/index.ts` (navigator + location)

---

### i-link.ts — 路由接口

**功能**: 定义路由跳转和返回拦截接口。

**导出**: `ILink`

**关键签名**:

```typescript
interface ILink {
    navigateTo(options: { url: string; animated?: boolean }): void;  // 跳转到指定页面
    navigateBack(options?: { delta?: number }): void;                // 返回上一页
    registorBackInterceptor(handler: () => boolean): void;           // 注册返回拦截器
}
```

**实现类**: Native `Link.ts` (基于 Router 模块), H5 `Link.ts` (基于 history)

**关键设计**: 
- `registorBackInterceptor` 允许拦截返回操作，用于未保存提醒等场景

---

### i-util.ts — 工具函数接口

**功能**: 定义通用工具函数接口。

**导出**: `IUtil`

**关键签名**:

```typescript
interface IUtil {
    cn(obj: any): string;                          // classNames - 条件类名拼接
    qs(query: Record<string, any>): string;        // 对象转查询字符串
    getGlobalData(key: string): any;               // 获取全局数据
    reload(): void;                                // 重新加载页面
    randomId(): string;                            // 生成随机ID
    dp2Rpx(dp: number): number;                    // dp 转 rpx
    rpx2Dp(rpx: number): number;                   // rpx 转 dp
    preloadImage(urls: string[]): void;             // 预加载图片
    measure(node: any, callback: Function): void;   // 测量节点尺寸
}
```

**实现类**: Native `Util.ts`, H5 `Util.ts`

---

### i-env.ts — 环境变量接口

**功能**: 定义环境变量相关属性和方法，用于运行时环境判断。

**导出**: `IEnv`

**关键属性/方法**:

```typescript
interface IEnv {
    isH5: boolean;                    // 是否 H5 环境
    isNative: boolean;                // 是否 Native 环境
    isIOS: boolean;                   // 是否 iOS
    isAndroid: boolean;               // 是否 Android
    isProduction: boolean;            // 是否生产环境
    isDevelopment: boolean;           // 是否开发环境
    locale: string;                   // 当前语言
    region: string;                   // 当前地区
    // ... 其他环境变量
}
```

**实现类**: Native `Env.ts`, H5 `Env.ts`

---

### i-eventbus.ts — 事件总线接口

**功能**: 定义全局事件总线的发布/订阅接口。

**导出**: `IEventBus`

**关键签名**:

```typescript
interface IEventBus {
    emit(event: string, ...args: any[]): void;              // 触发事件
    on(event: string, handler: (...args: any[]) => void): void;    // 监听事件
    off(event: string, handler?: (...args: any[]) => void): void;  // 取消监听
    addEventListener(event: string, handler: Function): void;      // 添加事件监听（别名）
    removeEventListener(event: string, handler: Function): void;   // 移除事件监听（别名）
}
```

**实现类**: `common/utils/event-bus.ts` (共享实现)

---

### i-share.ts — 分享配置接口

**功能**: 定义分享相关的配置和响应类型。

**导出**:
- `ShareConfig` — 分享配置
- `ShareResponse` — 分享响应

**关键签名**:

```typescript
interface ShareConfig {
    title: string;
    content?: string;
    url?: string;
    imageUrl?: string;
    // ... 其他分享配置
}

interface ShareResponse {
    success: boolean;
    // ... 其他响应数据
}
```

**被引用**: `IFeature.share()` 使用 `ShareConfig` 和 `ShareResponse`

---

### i-fingerprint.ts — 设备指纹接口

**功能**: 定义设备指纹获取接口，用于用户追踪和安全验证。

**导出**: `IFingerprint`

**关键签名**:

```typescript
interface IFingerprint {
    getFingerprint(): Promise<string>;  // 获取设备指纹
}
```

**实现类**: Native `FingerPrint.ts`

---

### i-comp-loader.ts — 组件加载接口

**功能**: 定义组件动态加载接口，用于按需加载组件 Bundle。

**导出**: `ICompLoader`

**关键签名**:

```typescript
interface ICompLoader {
    AsyncLoadComp: (compInfo: ComponentInfo) => React.ComponentType;  // 异步加载组件 HOC
    requireBundleAsync(bundleName: string): Promise<any>;             // 异步加载 Bundle
}
```

**实现类**: Native `CompLoader/index.tsx`

**关键设计**: 
- `AsyncLoadComp` 是高阶组件，返回异步加载的组件
- `requireBundleAsync` 直接加载 Bundle 资源

---

### i-page-data.ts — 页面数据类型

**功能**: 定义页面数据结构的类型，用于描述整个页面的配置和组件树。

**导出**:
- `PageData` — 页面完整数据
- `PageContent` — 页面内容
- `RenderItem` — 渲染项（组件节点）
- `ComponentInfo` — 组件信息
- `ComponentBundle` — 组件 Bundle 信息

**关键签名**:

```typescript
interface PageData {
    pageContent: PageContent;
    globalData: IGlobalData;
    // ... 其他页面级数据
}

interface PageContent {
    renderList: RenderItem[];  // 组件渲染树
    // ... 其他页面内容
}

interface RenderItem {
    componentInfo: ComponentInfo;
    children?: RenderItem[];
    // ... 其他渲染项属性
}

interface ComponentInfo {
    name: string;           // 组件名称
    version: string;        // 组件版本
    bundle?: ComponentBundle; // 组件 Bundle 信息
    // ... 其他组件属性
}

interface ComponentBundle {
    url: string;            // Bundle URL
    name: string;           // Bundle 名称
    // ... 其他 Bundle 属性
}
```

**被引用**: `PageDataPreprocessor`, `Store/global.ts`, `renderPage()`

---

### i-p-data-pre-processor.ts — 页面数据预处理接口

**功能**: 定义页面数据预处理接口，在渲染前对页面数据进行转换和降级判断。

**导出**: `IPageDataPreProcessor`

**关键签名**:

```typescript
interface IPageDataPreProcessor {
    preProcessPageContent(pageContent: PageContent): Promise<PageContent>;
}
```

**实现类**: Native `PageDataPreprocessor/index.ts`

**关键设计**: 
- 预处理阶段包含 H5 降级判断逻辑
- 可能修改组件树（移除不支持的组件、替换降级方案）

---

### i-global-data.ts — 全局数据类型

**功能**: 定义全局共享数据的类型。

**导出**:
- `IGlobalData` — 全局数据接口
- `RenderItem` — 渲染项（与 i-page-data.ts 中可能重复定义或重新导出）
- `ComponentInfo` — 组件信息（与 i-page-data.ts 中可能重复定义或重新导出）

**关键签名**:

```typescript
interface IGlobalData {
    pageId?: string;
    gundamId?: string;
    renderItems?: RenderItem[];
    componentInfos?: ComponentInfo[];
    // ... 其他全局数据
}
```

**被引用**: `Store/global.ts`, `renderPage()`

## 接口间依赖关系

```
Icore (聚合入口)
 ├── IAjax
 │    └── IAjaxOptions, IPureAjaxOptions, ICustomApiConfigBuilder
 ├── IStorage
 │    └── ICookie, ILocalStorage, ICookieOptions
 ├── I18N
 ├── ILog
 │    └── ILxInitOptions, ILxEventOptions, ILxEventOptionsWithBid
 ├── IMonitor
 │    └── SpeedPoint, IOwlError
 ├── IBridge
 ├── IFeature
 │    └── ShareConfig (from i-share.ts), ShareResponse (from i-share.ts)
 ├── ILink
 ├── IUtil
 ├── IEnv
 ├── IEventBus
 ├── IFingerprint
 ├── ICompLoader
 │    └── ComponentInfo (from i-page-data.ts)
 └── (间接依赖)
      ├── PageData, PageContent, RenderItem (from i-page-data.ts)
      └── IGlobalData (from i-global-data.ts)
```

## 实现映射

| 接口 | Native 实现 | H5 实现 |
|------|-----------|--------|
| `Icore` | `native/Core.ts` → `NativeCore` | `h5/Core.ts` → `H5Core` |
| `IAjax` | `native/Ajax/index.ts` | `h5/Ajax/index.ts` |
| `IStorage` | `native/Storage/index.ts` | `h5/Storage.ts` |
| `I18N` | `native/I18n.ts` | `h5/I18n.ts` |
| `ILog` | `native/Log.ts` | `h5/Log.ts` |
| `IMonitor` | `native/Monitor/index.ts` | `h5/Monitor.ts` |
| `IBridge` | `native/Bridge.ts` | `h5/Bridge.ts` |
| `IFeature` | `native/Feature/index.ts` | `h5/Feature/index.ts` |
| `ILink` | `native/Link.ts` | `h5/Link.ts` |
| `IUtil` | `native/Util.ts` | `h5/Util.ts` |
| `IEnv` | `native/Env.ts` | `h5/Env.ts` |
| `IEventBus` | `common/utils/event-bus.ts` | `common/utils/event-bus.ts` |
| `IFingerprint` | `native/FingerPrint.ts` | — (无H5实现) |
| `ICompLoader` | `native/CompLoader/index.tsx` | — (无H5实现) |
| `IPageDataPreProcessor` | `native/PageDataPreprocessor/index.ts` | — (无H5实现) |
