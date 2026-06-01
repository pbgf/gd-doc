# h5/ — H5 端实现层

基于浏览器环境的 H5 端实现，提供所有核心模块在 Web 环境下的运行时能力。与 Native 层相比，H5 层结构更简洁，大部分模块为单文件实现。

## 目录结构

```
h5/
├── Core.ts           # H5Core 中控台类
├── Ajax/             # 网络请求
│   └── index.ts      #   Ajax 实例 (axios + 拦截器链)
├── Bridge.ts         # H5 桥接 (window.KNB)
├── Env.ts            # H5 环境检测
├── Feature/          # 设备能力
│   └── index.ts      #   Feature 单例
├── I18n.ts           # 国际化 (i18n-web + 阿拉伯字体)
├── Link.ts           # 路由跳转 (history)
├── Log.ts            # 埋点 (LXAnalytics)
├── Monitor.ts        # 监控 (window.Owl + Logan)
├── Storage.ts        # 存储 (tiny-cookie + localStorage)
├── Store/            # 状态管理
│   └── index.ts      #   store/connect/createModel
└── Util.ts           # 工具函数
```

## 核心文件详解

### Core.ts — H5Core 中控台

**功能**: H5 端的核心入口类，实现 `Icore` 接口，聚合所有子模块。

**导出**: `H5Core` 类

**实现接口**: `Icore`

**关键实现**:

```typescript
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
    // 注意: H5 没有 fingerprint 和 compLoader

    constructor() {
        // 初始化各子模块
        this.bridge = Bridge;
        this.env = Env;
        this.storage = Storage;
        this.i18n = I18n;
        this.log = Log;
        this.monitor = Monitor;
        this.feature = Feature;
        this.link = Link;
        this.util = Util;
        this.ajax = Ajax;
        this.eventBus = EventBus;
    }

    renderPage(): void {
        // H5 端 renderPage 为空实现
        // H5 页面渲染由宿主 HTML 模板驱动
    }
}
```

**与 NativeCore 的差异**:
- 无 `fingerprint` 和 `compLoader` 属性（H5 不需要）
- `renderPage()` 为空实现（H5 由 HTML 模板驱动渲染）
- 子模块使用 H5 特定实现

---

### Ajax/index.ts — 网络请求

**功能**: H5 端网络请求实现，基于 axios，支持拦截器链。

**导出**: `Ajax` 单例

**实现接口**: `IAjax`

**关键实现**:

```typescript
class Ajax implements IAjax {
    private axiosInstance: AxiosInstance;

    constructor() {
        this.axiosInstance = axios.create({
            timeout: 10000,
            withCredentials: true,
        });
        // 注册拦截器
        this.setupInterceptors();
    }

    request(options: IAjaxOptions): Promise<any> {
        return this.axiosInstance.request(options);
    }

    pureRequest(options: IPureAjaxOptions): Promise<any> {
        // 使用独立 axios 实例（无拦截器）
        return axios.request(options);
    }

    get(url, data, options): Promise<any> { ... }
    post(url, data, options): Promise<any> { ... }
}
```

**依赖**: `axios`, `common/utils/common-interceptors.ts`

**拦截器链**:
```
请求 → 公共参数拦截器 → URL拦截器 → 发送请求 → 响应拦截器 → 返回
```

**与 Native Ajax 的差异**:
- 基于 axios 而非 Mach HTTP 模块
- 不支持 DSL 聚合请求
- 拦截器链更简单（无泳道拦截器）
- 跨域通过 `withCredentials` 处理

---

### Bridge.ts — H5 桥接

**功能**: H5 端桥接实现，通过 `window.KNB` 与客户端通信。

**导出**: `Bridge` 单例

**实现接口**: `IBridge`

**关键实现**:

```typescript
class Bridge implements IBridge {
    getBridge(): any {
        return window.KNB;
    }
}
```

**与 Native Bridge 的差异**:
- 直接使用 `window.KNB` 而非 `mach-pro-knb`
- 在纯浏览器环境（无客户端）时 KNB 不可用，需要做兼容处理

**兼容处理**:
```typescript
getBridge(): any {
    if (typeof window !== 'undefined' && window.KNB) {
        return window.KNB;
    }
    return null; // 或返回 Mock 对象
}
```

---

### Env.ts — H5 环境检测

**功能**: 检测浏览器运行环境信息。

**导出**: `Env` 单例

**实现接口**: `IEnv`

**关键实现**:
- `isH5`: true
- `isNative`: false
- `isIOS` / `isAndroid`: 基于 `navigator.userAgent` 检测
- `locale` / `region`: 从 URL 参数或 Cookie 获取
- 运行模式检测（基于 hostname 判断开发/测试/生产）

**与 Native Env 的差异**:
- 通过 `navigator.userAgent` 而非 Mach API 检测
- 环境变量来源不同（URL参数/Cookie vs Mach配置）

---

### Feature/index.ts — 设备能力

**功能**: H5 端设备能力实现，基于浏览器 API。

**导出**: `Feature` 单例

**实现接口**: `IFeature`

**关键实现**:
- `getAppInfo()`: 从 URL 参数解析 App 信息
- `getDeviceInfo()`: 从 `navigator.userAgent` 解析设备信息
- `getLocation()`: 基于 `navigator.geolocation` API
- `getUserInfo()`: 从 Cookie/接口获取用户信息
- `login()`: 跳转到 SSO 登录页
- `share()`: 调用 KNB 分享或 Web Share API
- `scrollToId()`: `document.getElementById().scrollIntoView()`
- `getStrategyAsync()`: 请求 AB 实验接口

**与 Native Feature 的差异**:
- 不拆分子文件，所有能力在单文件中实现
- 使用浏览器标准 API 替代 KNB 调用
- 部分能力受限（如定位需要用户授权）

---

### I18n.ts — 国际化

**功能**: H5 端国际化实现，基于 `i18n-web`，支持阿拉伯字体动态加载。

**导出**: `I18nClass` 单例

**实现接口**: `I18N`

**关键实现**:
- 基于 `i18n-web` 初始化 i18next
- 支持多命名空间资源加载
- **阿拉伯字体加载**: 检测语言为 ar（阿拉伯语）时，动态加载阿拉伯字体 CSS
- RTL 布局支持: 自动添加 `dir="rtl"` 和 RTL 样式

**依赖**: `i18n-web`, `i18next`, `react-i18next`

**与 Native I18n 的差异**:
- 使用 `i18n-web` 而非 `i18n-mach-pro`
- 特有的阿拉伯字体动态加载逻辑
- RTL 支持更完善（H5 需要处理更多样式细节）

**阿拉伯字体加载实现**:
```typescript
async loadArabicFont() {
    if (this.getLocale().startsWith('ar')) {
        // 动态创建 <link> 标签加载阿拉伯字体 CSS
        const link = document.createElement('link');
        link.rel = 'stylesheet';
        link.href = ARABIC_FONT_CSS_URL;
        document.head.appendChild(link);
    }
}
```

---

### Link.ts — 路由跳转

**功能**: H5 端路由跳转实现，基于 history API。

**导出**: `LinkHandler` 单例

**实现接口**: `ILink`

**关键实现**:

```typescript
class LinkHandler implements ILink {
    navigateTo({ url }: { url: string }): void {
        // 优先使用 history.pushState（SPA 内跳转）
        // 外部链接使用 window.location.href
        if (isInternalUrl(url)) {
            history.pushState(null, '', url);
        } else {
            window.location.href = url;
        }
    }

    navigateBack({ delta = 1 } = {}): void {
        history.go(-delta);
    }

    registorBackInterceptor(handler: () => boolean): void {
        // 监听 popstate 事件
        window.addEventListener('popstate', handler);
    }
}
```

**与 Native Link 的差异**:
- 基于 `history` API 而非 Mach Router
- `navigateTo` 区分内部/外部链接
- `registorBackInterceptor` 基于 `popstate` 事件

---

### Log.ts — 埋点

**功能**: H5 端埋点实现，基于 LXAnalytics（灵犀分析）。

**导出**: `Log` 单例

**实现接口**: `ILog`

**关键实现**:
- 基于 `LXAnalytics` SDK 上报事件
- 支持 PV 和 Event 两类埋点
- 自动注入公共参数
- 支持 BID 维度

**依赖**: `LXAnalytics`

**与 Native Log 的差异**:
- 使用 `LXAnalytics` 而非 Mach Statistics
- 初始化方式不同（SDK script 加载 vs 模块初始化）

---

### Monitor.ts — 监控

**功能**: H5 端监控实现，基于 `window.Owl` 和 Logan。

**导出**: `Monitor` 单例

**实现接口**: `IMonitor`

**关键实现**:
- 错误上报: 通过 `window.Owl` 上报
- 本地日志: 通过 Logan 记录
- 测速上报: 记录并上报测速点数据

**与 Native Monitor 的差异**:
- 单文件实现（Native 拆分为 3 个文件）
- 使用 `window.Owl` 而非 Owl SDK
- 无独立的 SpeedReport 模块
- 错误捕获通过 `window.onerror` 和 `window.addEventListener('unhandledrejection')`

---

### Storage.ts — 存储

**功能**: H5 端存储实现，基于 `tiny-cookie` 和 `localStorage`。

**导出**: `Storage` 单例

**实现接口**: `IStorage`

**关键实现**:

```typescript
const Storage: IStorage = {
    cookie: {
        get(key: string): string | undefined {
            return tinyCookie.get(key);
        },
        set(key: string, value: string, options?: ICookieOptions): void {
            tinyCookie.set(key, value, options);
        },
        remove(key: string, options?: ICookieOptions): void {
            tinyCookie.remove(key, options);
        },
    },
    localStorage: {
        get(key: string): string | null {
            return window.localStorage.getItem(key);
        },
        set(key: string, value: string): void {
            window.localStorage.setItem(key, value);
        },
        remove(key: string): void {
            window.localStorage.removeItem(key);
        },
    },
};
```

**依赖**: `tiny-cookie`

**与 Native Storage 的差异**:
- Cookie 使用 `tiny-cookie` 而非 KNB Cookie API
- LocalStorage 直接使用 `window.localStorage`
- 单文件实现（Native 拆分为 4 个文件）

---

### Store/index.ts — 状态管理

**功能**: H5 端 Rematch Store，与 Native 端基本一致。

**导出**:
- `store`: Rematch store 实例
- `connect`: React HOC 连接器
- `createModel`: 类型安全的 model 创建工具

**关键实现**: 与 Native Store 基本一致，使用 `@sailorgundam/gundam-store`

**与 Native Store 的差异**:
- 可能不包含 `global` model 的 `initPageData` effect（H5 页面数据由模板驱动）

---

### Util.ts — 工具函数

**功能**: H5 端工具函数集合。

**导出**: `Util` 单例

**实现接口**: `IUtil`

**关键实现**: 与 Native Util 基本一致，大部分方法由 `common/utils/` 提供

**与 Native Util 的差异**:
- `measure()`: 基于 `getBoundingClientRect()` 而非 Mach measure API
- `preloadImage()`: 使用 `new Image()` 加载而非 Mach API
- `reload()`: 使用 `window.location.reload()` 而非 Mach API

---

## H5 vs Native 差异总结

| 维度 | Native | H5 |
|------|--------|-----|
| **网络请求** | Mach HTTP + DSL 聚合 | axios |
| **桥接** | mach-pro-knb | window.KNB |
| **国际化** | i18n-mach-pro | i18n-web + 阿拉伯字体加载 |
| **埋点** | Statistics 模块 | LXAnalytics |
| **监控** | Owl SDK + Logan + SpeedReport | window.Owl + Logan |
| **存储** | KNB Cookie/LocalStorage | tiny-cookie + localStorage |
| **路由** | Mach Router | history API |
| **设备能力** | KNB/Compass/SLCommon | navigator/location API |
| **组件加载** | AsyncLoadComp HOC | 不需要 |
| **页面预处理** | PageDataPreprocessor | 不需要 |
| **页面渲染** | renderPage() | HTML 模板驱动 |
| **设备指纹** | KNB 指纹 | 不支持 |
| **文件结构** | 多文件拆分 | 单文件为主 |

## 关键设计决策

1. **无 CompLoader**: H5 环境下组件代码通过构建工具打包到单文件 Bundle，无需动态加载
2. **无 PageDataPreprocessor**: H5 页面数据由后端模板渲染，无需客户端预处理
3. **无 FingerPrint**: H5 环境下设备指纹获取不可靠，不提供此能力
4. **阿拉伯字体加载**: H5 特有的国际化处理，确保阿拉伯语显示正确
5. **axios 拦截器链**: 比Native端更简单的拦截器链，无DSL聚合和泳道路由
