# native/ — Native 端实现层

基于 MachPro 跨端引擎的 Native 端实现，提供所有核心模块在 iOS/Android 原生环境下的运行时能力。

## 目录结构

```
native/
├── Core.ts                           # NativeCore 中控台类
├── Ajax/                             # 网络请求
│   ├── index.ts                      #   Ajax 实例（DSL聚合+重试+拦截器链）
│   ├── pure-ajax.ts                  #   底层请求封装
│   ├── request-interceptor.ts        #   请求拦截器
│   ├── swimlane-interceptor.ts       #   泳道拦截器
│   └── url-interceptor.ts            #   URL拦截器
├── Bridge.ts                         # Native桥接 (mach-pro-knb)
├── CompLoader/                       # 组件动态加载
│   ├── index.tsx                     #   AsyncLoadComp HOC
│   └── comp-load-manager.ts          #   Bundle加载管理器
├── Env.ts                            # 环境检测
├── Feature/                          # 设备能力聚合
│   ├── index.ts                      #   Feature 单例
│   ├── app-info.ts                   #   应用信息
│   ├── device-info.ts                #   设备信息
│   ├── get-location.ts               #   定位
│   ├── login.ts                      #   登录
│   ├── network-type.ts               #   网络类型
│   ├── scroll-to-id.ts               #   滚动定位
│   ├── user-info.ts                  #   用户信息
│   └── ab-strategy.ts               #   AB实验策略
├── I18n.ts                           # 国际化 (i18n-mach-pro)
├── Link.ts                           # 路由 (Router模块)
├── Log.ts                            # 埋点 (Statistics模块)
├── Monitor/                          # 监控
│   ├── index.ts                      #   Monitor 单例 (Owl+Logan+SpeedReport)
│   ├── owl-manager.ts                #   Owl 管理器
│   ├── speed-report.ts               #   测速上报
│   └── logan.ts                      #   Logan 本地日志
├── Storage/                          # 存储
│   ├── index.ts                      #   Storage 单例 (Cookie+LocalStorage)
│   ├── Cookie.ts                     #   Cookie 实现
│   ├── LocalStorage.ts               #   LocalStorage 实现
│   └── util.ts                       #   存储工具函数
├── Store/                            # 状态管理
│   ├── index.ts                      #   store/connect/createModel/addModel
│   └── global.ts                     #   app model (initPageData effect)
├── Util.ts                           # 工具函数
├── PageDataPreprocessor/             # 页面数据预处理
│   ├── index.ts                      #   PageDataPreProcessor 主入口
│   ├── load-page-data.ts             #   页面数据加载
│   ├── speed-wrap.ts                 #   测速包装
│   └── version-helpers.ts            #   版本辅助函数
├── KeetaDesign.ts                    # 设计系统
├── FingerPrint.ts                    # 设备指纹
└── utils/                            # 辅助工具函数
    ├── debug.ts                      #   调试工具
    ├── get-collection-page-mode.ts    #   获取集合页模式
    ├── get-core-instance.ts          #   获取Core实例
    ├── get-render-info.ts            #   获取渲染信息
    ├── init-page-data.ts             #   初始化页面数据
    ├── measure.ts                    #   节点测量
    ├── preload-image.ts              #   图片预加载
    └── render-page.tsx               #   页面渲染
```

## 核心文件详解

### Core.ts — NativeCore 中控台

**功能**: Native 端的核心入口类，实现 `Icore` 接口，聚合所有子模块。

**导出**: `NativeCore` 类

**实现接口**: `Icore`

**关键实现**:

```typescript
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

    constructor() {
        // 按顺序初始化各子模块
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
        this.fingerprint = FingerPrint;
        this.compLoader = CompLoader;
    }

    renderPage(): void {
        // 渲染 Native 页面
        // 1. 初始化页面数据
        // 2. 初始化 Store
        // 3. 加载组件 Bundle
        // 4. 渲染组件树
    }
}
```

**依赖**: 本目录下所有子模块

**关键设计**:
- 初始化顺序有依赖关系：Bridge/Env 先初始化，其他模块可能依赖它们
- `renderPage()` 是 Native 页面渲染的入口方法

---

### Ajax/ — 网络请求模块

#### Ajax/index.ts — Ajax 实例

**功能**: Native 端网络请求主入口，支持 DSL 聚合请求、自动重试、拦截器链。

**导出**: `Ajax` 单例

**实现接口**: `IAjax`

**关键实现**:
- `request(options)`: 标准请求，经过完整拦截器链
- `pureRequest(options)`: 纯请求，绕过拦截器
- `get(url, data, options)`: GET 请求快捷方法
- `post(url, data, options)`: POST 请求快捷方法
- DSL 聚合: 将多个 API 请求合并为一个 DSL 请求
- 自动重试: 请求失败时按策略重试

**依赖**:
- `pure-ajax.ts`: 底层请求封装
- `request-interceptor.ts`: 请求拦截器
- `swimlane-interceptor.ts`: 泳道拦截器
- `url-interceptor.ts`: URL 拦截器
- `common/utils/common-interceptors.ts`: 公共参数拦截器
- `common/utils/retry-util.ts`: 重试工具

**拦截器链执行顺序**:
```
请求 → request-interceptor → swimlane-interceptor → url-interceptor → 
common-interceptors → 发送请求 → 响应拦截器 → 返回
```

#### Ajax/pure-ajax.ts — 底层请求封装

**功能**: 不经过拦截器的底层 HTTP 请求封装。

**关键实现**:
- 基于 Mach 模块发送 HTTP 请求
- 支持超时设置
- 支持请求取消
- 错误处理和转换

#### Ajax/request-interceptor.ts — 请求拦截器

**功能**: 通用请求拦截，处理请求发送前的通用逻辑。

**关键实现**:
- 注入请求头
- 参数序列化
- 请求日志记录

#### Ajax/swimlane-interceptor.ts — 泳道拦截器

**功能**: 泳道/灰度环境路由，确保请求发送到正确的服务实例。

**关键实现**:
- 读取泳道标识（Cookie/URL 参数）
- 在请求头中注入泳道路由信息
- 支持多泳道并行测试

#### Ajax/url-interceptor.ts — URL 拦截器

**功能**: URL 处理和拼接，统一 URL 格式。

**关键实现**:
- URL 拼接（基础URL + 路径 + 查询参数）
- URL 协议处理（http/https）
- 接口环境切换（测试/预发/生产）

---

### Bridge.ts — Native 桥接

**功能**: 提供 Native 桥接能力，通过 KNB 协议与客户端通信。

**导出**: `Bridge` 单例

**实现接口**: `IBridge`

**关键实现**:

```typescript
class Bridge implements IBridge {
    private knbInstance: any;

    getBridge(): any {
        // 返回 mach-pro-knb 实例
        return this.knbInstance;
    }
}
```

**依赖**: `@wmfe/mach-pro-knb`

**关键设计**:
- 单例模式，全局唯一的 KNB 实例
- 通过 KNB 实例可调用：打开页面、分享、获取设备信息、扫码等 Native 能力
- 所有 Native 交互通过此桥接层统一管理

---

### CompLoader/ — 组件动态加载

#### CompLoader/index.tsx — AsyncLoadComp HOC

**功能**: 动态加载组件的高阶组件，按需加载组件 Bundle。

**导出**: `AsyncLoadComp`, `requireBundleAsync`

**实现接口**: `ICompLoader`

**关键实现**:

```typescript
// AsyncLoadComp: 接受组件信息，返回异步加载的 React 组件
function AsyncLoadComp(compInfo: ComponentInfo): React.ComponentType {
    // 1. 根据 compInfo.bundle 获取 Bundle URL
    // 2. 动态加载 Bundle
    // 3. 返回加载中/加载成功/加载失败的组件
    // 4. 加载成功后缓存组件实例
}

// requireBundleAsync: 异步加载指定 Bundle
async function requireBundleAsync(bundleName: string): Promise<any> {
    // 1. 查找 Bundle URL
    // 2. 加载并执行 Bundle
    // 3. 返回模块导出
}
```

**依赖**: `comp-load-manager.ts`

**关键设计**:
- 异步加载避免首屏加载所有组件代码
- 加载状态展示（Loading 占位）
- 加载失败兜底（错误提示组件）
- Bundle 缓存机制

#### CompLoader/comp-load-manager.ts — Bundle 加载管理器

**功能**: 管理组件 Bundle 的加载、缓存和版本管理。

**关键实现**:
- Bundle URL 解析和映射
- 已加载 Bundle 缓存
- Bundle 版本管理（处理版本升级时的缓存更新）
- 加载队列管理（避免重复加载）

---

### Env.ts — 环境检测

**功能**: 检测 Mach Native 运行环境信息。

**导出**: `Env` 单例

**实现接口**: `IEnv`

**关键实现**:
- `isH5`: false
- `isNative`: true
- `isIOS` / `isAndroid`: 基于 Mach API 检测
- `locale` / `region`: 从 Mach 配置获取
- 运行模式检测（开发/测试/生产）

**依赖**: `@wmfe/mach-pro-knb`

---

### Feature/ — 设备能力聚合

#### Feature/index.ts — Feature 单例

**功能**: 聚合所有设备能力子模块，对外提供统一接口。

**导出**: `Feature` 单例

**实现接口**: `IFeature`

**依赖子模块**:
- `app-info.ts`: 应用信息
- `device-info.ts`: 设备信息
- `get-location.ts`: 定位
- `login.ts`: 登录
- `network-type.ts`: 网络类型
- `scroll-to-id.ts`: 滚动定位
- `user-info.ts`: 用户信息
- `ab-strategy.ts`: AB 实验策略

#### Feature/app-info.ts — 应用信息

**功能**: 获取宿主 App 的版本号、包名等信息。

**关键实现**: 通过 KNB 获取 AppInfo

#### Feature/device-info.ts — 设备信息

**功能**: 获取设备型号、系统版本、屏幕尺寸等信息。

**关键实现**: 通过 KNB 获取 DeviceInfo

#### Feature/get-location.ts — 定位

**功能**: 获取用户当前位置（经纬度）。

**关键实现**: 通过 KNB 定位 API，支持权限请求

#### Feature/login.ts — 登录

**功能**: 触发用户登录流程。

**关键实现**: 通过 KNB 登录 API，唤起登录页面

#### Feature/network-type.ts — 网络类型

**功能**: 获取当前网络类型（WiFi/4G/5G 等）。

**关键实现**: 通过 KNB 获取网络状态

#### Feature/scroll-to-id.ts — 滚动定位

**功能**: 滚动到指定 ID 的元素位置。

**关键实现**: 调用 Native 滚动 API

#### Feature/user-info.ts — 用户信息

**功能**: 获取当前登录用户的信息（用户ID、昵称等）。

**关键实现**: 通过 KNB 获取 UserInfo

#### Feature/ab-strategy.ts — AB 实验策略

**功能**: 获取 AB 实验的策略分组。

**关键实现**: 通过 KNB/接口获取实验分组信息

---

### I18n.ts — 国际化

**功能**: Native 端国际化实现，基于 `i18n-mach-pro`。

**导出**: `I18nClass` 单例

**实现接口**: `I18N`

**关键实现**:
- 基于 `i18n-mach-pro` 初始化 i18next
- 支持多命名空间资源加载
- 支持语言切换（RTL/LTR 自动适配）
- 城市ID驱动的时区和日期格式化

**依赖**: `i18n-mach-pro`, `i18next`, `react-i18next`

---

### Link.ts — 路由跳转

**功能**: Native 端路由跳转，基于 Mach Router 模块。

**导出**: `LinkHandler` 单例

**实现接口**: `ILink`

**关键实现**:
- `navigateTo({ url })`: 跳转到新页面
- `navigateBack()`: 返回上一页
- `registorBackInterceptor(handler)`: 注册返回拦截

**依赖**: Mach Router 模块

---

### Log.ts — 埋点

**功能**: Native 端埋点实现，基于 Statistics 模块。

**导出**: `Log` 单例

**实现接口**: `ILog`

**关键实现**:
- 基于 Mach Statistics 模块上报事件
- 支持 PV（页面浏览）和 Event（自定义事件）两类埋点
- 自动注入公共参数（设备信息、用户信息等）
- 支持 BID（业务ID）维度

**依赖**: Mach Statistics 模块

---

### Monitor/ — 监控模块

#### Monitor/index.ts — Monitor 单例

**功能**: Native 端监控聚合入口，集成 Owl、Logan 和 SpeedReport。

**导出**: `Monitor` 单例

**实现接口**: `IMonitor`

**关键实现**:
- 错误上报: 通过 Owl 上报 JS 错误和自定义错误
- 本地日志: 通过 Logan 记录本地日志
- 测速上报: 通过 SpeedReport 上报 19 个测速点数据

**依赖**: `owl-manager.ts`, `speed-report.ts`, `logan.ts`

#### Monitor/owl-manager.ts — Owl 管理器

**功能**: Owl 监控 SDK 的初始化和错误上报管理。

**关键实现**:
- Owl SDK 初始化
- 错误分类和上报
- 错误采样率控制

#### Monitor/speed-report.ts — 测速上报

**功能**: 首屏测速数据收集和上报。

**关键实现**:
- 19 个测速点的数据收集
- 测速数据聚合和计算
- 上报到 Owl/SpeedReport 后台

#### Monitor/logan.ts — Logan 本地日志

**功能**: Logan 本地日志记录，用于问题排查。

**关键实现**:
- 本地日志写入
- 日志级别管理（debug/info/warn/error）
- 日志上传

---

### Storage/ — 存储模块

#### Storage/index.ts — Storage 单例

**功能**: 存储聚合入口，提供 Cookie 和 LocalStorage 两类存储。

**导出**: `Storage` 单例

**实现接口**: `IStorage`

**关键实现**:

```typescript
const Storage: IStorage = {
    cookie: Cookie,
    localStorage: LocalStorage
};
```

#### Storage/Cookie.ts — Cookie 实现

**功能**: Native 端 Cookie 操作实现。

**实现接口**: `ICookie`

**关键实现**:
- 基于 KNB 的 Cookie API
- 支持 Cookie 过期时间、路径、域名设置
- 支持安全 Cookie（Secure）

#### Storage/LocalStorage.ts — LocalStorage 实现

**功能**: Native 端 LocalStorage 操作实现。

**实现接口**: `ILocalStorage`

**关键实现**:
- 基于 Mach 存储 API
- 键值对存储
- 容量管理

#### Storage/util.ts — 存储工具函数

**功能**: 存储模块的辅助工具函数。

**关键实现**:
- Cookie 序列化/反序列化
- 存储键名前缀管理
- 过期时间计算

---

### Store/ — 状态管理模块

#### Store/index.ts — Rematch Store

**功能**: 基于 Rematch v2 的状态管理，提供 store、connect、createModel、addModel。

**导出**:
- `store`: Rematch store 实例
- `connect`: React HOC 连接器
- `createModel`: 类型安全的 model 创建工具
- `addModel`: 动态注册 model

**关键实现**:
- 全局单例 (`globalThis.__GUNDAM_PRO_STORE__`)
- 初始化时注册 `global` model
- 支持动态添加 model

**依赖**: `@sailorgundam/gundam-store`, `global.ts`

#### Store/global.ts — App 全局 Model

**功能**: App 级别的 Rematch model，管理全局数据。

**导出**: `app` model

**关键实现**:

```typescript
const app = {
    name: 'app',
    state: {
        pageData: null,
        globalData: null,
        pageConfig: null,
        renderTree: null,
    },
    effects: {
        async initPageData(payload, rootState) {
            // 1. 加载页面数据
            // 2. 初始化全局数据
            // 3. 构建渲染树
        },
    },
    reducers: {
        setPageData(state, payload) { ... },
        setGlobalData(state, payload) { ... },
        // ...
    },
};
```

**关键设计**: `initPageData` effect 是页面数据加载的入口

---

### Util.ts — 工具函数

**功能**: Native 端工具函数集合。

**导出**: `Util` 单例

**实现接口**: `IUtil`

**关键实现**:
- `cn()`: classNames 条件类名拼接
- `qs()`: 对象转查询字符串
- `getGlobalData()`: 获取全局数据
- `reload()`: 重新加载页面
- `randomId()`: 生成随机ID
- `dp2Rpx()` / `rpx2Dp()`: 单位转换
- `preloadImage()`: 图片预加载
- `measure()`: 节点尺寸测量

**依赖**: `common/utils/` 下的工具函数

---

### PageDataPreprocessor/ — 页面数据预处理

#### PageDataPreprocessor/index.ts — 预处理主入口

**功能**: 页面数据预处理器，在渲染前对数据进行转换，包含 H5 降级判断。

**导出**: `PageDataPreProcessor` 单例

**实现接口**: `IPageDataPreProcessor`

**关键实现**:

```typescript
class PageDataPreProcessor implements IPageDataPreProcessor {
    async preProcessPageContent(pageContent: PageContent): Promise<PageContent> {
        // 1. 版本兼容性检查
        // 2. H5 降级判断（组件是否支持 Native）
        // 3. 组件树裁剪（移除不支持的组件）
        // 4. 数据适配
        return processedPageContent;
    }
}
```

**依赖**: `load-page-data.ts`, `speed-wrap.ts`, `version-helpers.ts`

**关键设计**:
- H5 降级: 当 Native 不支持某组件时，自动降级为 H5 渲染
- 版本兼容: 检查组件版本是否与当前 Core 版本兼容

#### PageDataPreprocessor/load-page-data.ts — 页面数据加载

**功能**: 从服务端加载页面数据。

**关键实现**:
- 请求页面数据接口
- 数据解析和校验
- 错误处理和重试

#### PageDataPreprocessor/speed-wrap.ts — 测速包装

**功能**: 在页面数据加载过程中记录测速点。

**关键实现**:
- 包装加载函数，自动记录开始/结束时间
- 上报测速点到 Monitor

#### PageDataPreprocessor/version-helpers.ts — 版本辅助函数

**功能**: 组件版本比较和兼容性判断。

**关键实现**:
- 版本号解析
- 版本比较（semver）
- 兼容性检查

---

### KeetaDesign.ts — 设计系统

**功能**: Keeta 设计系统的 Native 端适配。

**关键实现**: 提供设计 Token（颜色、字号、间距等）的 Native 端实现

---

### FingerPrint.ts — 设备指纹

**功能**: Native 端设备指纹获取。

**导出**: `FingerPrint` 单例

**实现接口**: `IFingerprint`

**关键实现**: 通过 KNB 获取设备指纹，用于用户追踪和安全验证

---

### utils/ — 辅助工具函数

#### utils/debug.ts — 调试工具

**功能**: 开发环境调试辅助。

**关键实现**:
- 调试日志输出
- 开发模式检测
- 调试面板控制

#### utils/get-collection-page-mode.ts — 集合页模式

**功能**: 获取当前集合页的展示模式。

**关键实现**: 判断页面是普通模式还是集合页模式

#### utils/get-core-instance.ts — 获取 Core 实例

**功能**: 获取全局 NativeCore 实例。

**关键实现**: 返回 `globalThis.__GUNDAM_CORE__` 或创建新实例

#### utils/get-render-info.ts — 获取渲染信息

**功能**: 获取当前页面的渲染配置信息。

**关键实现**: 解析页面渲染参数（如组件版本、渲染模式等）

#### utils/init-page-data.ts — 初始化页面数据

**功能**: 初始化页面数据的入口函数。

**关键实现**:
- 调用 PageDataPreprocessor 处理数据
- 初始化 Store
- 设置测速点

#### utils/measure.ts — 节点测量

**功能**: 测量 Native 节点的尺寸和位置。

**关键实现**: 调用 Mach measure API 获取节点布局信息

#### utils/preload-image.ts — 图片预加载

**功能**: 预加载图片资源，加速后续渲染。

**关键实现**: 调用 Mach 图片预加载 API

#### utils/render-page.tsx — 页面渲染

**功能**: Native 页面渲染入口，创建组件树并渲染。

**关键实现**:
- 解析渲染树（RenderItem[]）
- 递归创建组件实例
- 用 GdcHOC 包裹每个组件
- 处理组件间通信
- 上报首屏测速点

**依赖**: `CompLoader`, `Store`, `common/utils/gdc-hoc.ts`
