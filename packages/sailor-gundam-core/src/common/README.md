# common/ — 共享代码层

Native 和 H5 端共用的工具函数、组件和状态模型。不依赖特定平台 API，可被两端安全引用。

## 目录结构

```
common/
├── utils/                            # 通用工具函数
│   ├── gdc-hoc.ts                    #   GdcHOC 高阶组件工厂
│   ├── event-bus.ts                  #   全局事件总线
│   ├── constants.ts                  #   常量定义
│   ├── get-global-data.ts            #   全局数据获取
│   ├── deep-merge.ts                 #   深度合并
│   ├── dp-to-rpx.ts                  #   dp 转 rpx
│   ├── rpx-to-dp.ts                  #   rpx 转 dp
│   ├── qs.ts                         #   查询字符串处理
│   ├── debug.ts                      #   调试工具
│   ├── get-ffp-result.ts             #   FFP 结果获取
│   ├── filter-component-props.ts     #   组件属性过滤
│   ├── construct-url.ts              #   URL 构造
│   ├── share.ts                      #   分享逻辑
│   ├── resolve-i18n.ts               #   国际化解析
│   ├── location-wrapper.ts           #   定位封装
│   ├── retry-util.ts                 #   重试工具
│   ├── common-interceptors.ts        #   公共拦截器
│   ├── calculate-renderable-component.ts  # 渲染组件计算
│   ├── no-concurrent.ts             #   防并发
│   ├── timezone.ts                   #   时区处理
│   ├── is-key-has-value.ts          #   键值检测
│   └── enums.ts                      #   枚举定义
│
├── components/                       # 通用组件
│   ├── image.tsx                     #   图片监控组件
│   ├── lottie.tsx                    #   Lottie 动画监控组件
│   ├── vap.tsx                       #   VAP 动效监控组件
│   └── template.tsx                  #   模板组件
│
└── comp-fetch-store/                 # 组件数据获取 Store
    └── index.ts                      #   compModel (Rematch model)
```

## utils/ — 通用工具函数

### gdc-hoc.ts — GdcHOC 高阶组件工厂

**功能**: 创建高阶组件（HOC）的工厂函数，为 Gundam 组件自动注入曝光埋点、点击埋点和错误边界。

**导出**: `createHOC` 工厂函数

**关键签名**:

```typescript
interface GdcHOCOptions {
    autoExposure?: boolean;    // 是否自动曝光埋点（默认 true）
    autoClick?: boolean;       // 是否自动点击埋点（默认 true）
    errorBoundary?: boolean;   // 是否启用错误边界（默认 true）
    exposureThreshold?: number;// 曝光阈值（0-1）
}

function createHOC(options?: GdcHOCOptions): (
    WrappedComponent: React.ComponentType,
    compInfo: ComponentInfo
) => React.ComponentType;
```

**关键实现**:

```typescript
function createHOC(options: GdcHOCOptions = {}) {
    return function enhanceComponent(WrappedComponent, compInfo) {
        class GdcHOC extends React.Component {
            // 1. 曝光检测
            //    - 使用 IntersectionObserver (H5) 或 polling (Native)
            //    - 组件进入视口时自动上报曝光埋点
            //    - 支持 exposureThreshold 配置曝光比例

            // 2. 点击埋点
            //    - 拦截 onClick 事件
            //    - 在原始处理函数执行前上报点击埋点

            // 3. 错误边界
            //    - componentDidCatch 捕获子组件渲染错误
            //    - 防止单个组件错误导致整个页面白屏
            //    - 错误发生时展示降级 UI

            render() {
                return <WrappedComponent {...this.props} />;
            }
        }
        return GdcHOC;
    };
}
```

**依赖**: `compInfo` (组件信息), `ILog` (埋点), `IMonitor` (错误上报)

**关键设计**:
- 工厂模式: `createHOC` 返回配置好的 HOC 函数，而非直接返回 HOC
- 曝光去重: 每个组件实例只上报一次曝光
- 错误隔离: 单个组件的错误不影响其他组件
- 性能优化: 曝光检测使用防抖，避免频繁计算

---

### event-bus.ts — 全局事件总线

**功能**: 提供全局事件发布/订阅机制，实现跨组件通信。

**导出**: `EventBus` 单例

**实现接口**: `IEventBus`

**关键签名**:

```typescript
class EventBus implements IEventBus {
    private events: Map<string, Set<Function>>;

    emit(event: string, ...args: any[]): void {
        // 触发指定事件的所有监听器
    }

    on(event: string, handler: (...args: any[]) => void): void {
        // 注册事件监听器
    }

    off(event: string, handler?: (...args: any[]) => void): void {
        // 移除事件监听器
        // 不传 handler 则移除该事件所有监听器
    }

    addEventListener(event: string, handler: Function): void {
        // on 的别名
    }

    removeEventListener(event: string, handler: Function): void {
        // off 的别名
    }
}
```

**关键设计**:
- 单例模式，全局共享
- 支持多监听器
- `addEventListener`/`removeEventListener` 是 `on`/`off` 的别名，提供 DOM 风格 API
- 类型安全：事件名用字符串，参数用 any[]

---

### constants.ts — 常量定义

**功能**: 定义全局共享常量。

**导出**:
- `COMP_STORE_ID`: 组件 Store ID 前缀，用于生成组件唯一的 Store key
- `NOT_LOGIN_RESPONSE`: 未登录响应标识

**关键实现**:

```typescript
export const COMP_STORE_ID = 'gdc_comp_';     // 组件 Store ID 前缀
export const NOT_LOGIN_RESPONSE = 'NOT_LOGIN'; // 未登录响应标识
```

---

### get-global-data.ts — 全局数据获取

**功能**: 获取全局共享数据，提供统一的数据访问接口。

**导出**: `getGlobalData` 函数

**关键签名**:

```typescript
function getGlobalData(key: string): any {
    // 从全局 Store 中获取指定 key 的数据
    // 通常读取 Store 中的 app model 的 globalData
}
```

**依赖**: `Store`

---

### deep-merge.ts — 深度合并

**功能**: 深度合并两个对象，用于配置合并等场景。

**导出**: `deepMerge` 函数

**关键签名**:

```typescript
function deepMerge(target: any, source: any): any {
    // 递归合并 source 到 target
    // 数组: 直接替换（不合并元素）
    // 对象: 递归合并
    // 基本类型: source 覆盖 target
}
```

---

### dp-to-rpx.ts — dp 转 rpx

**功能**: 将 dp（密度无关像素）单位转换为 rpx 单位。

**导出**: `dp2Rpx` 函数

**关键签名**:

```typescript
function dp2Rpx(dp: number): number {
    // dp × 转换系数 = rpx
}
```

**相关**: `rpx-to-dp.ts` 提供反向转换

---

### rpx-to-dp.ts — rpx 转 dp

**功能**: 将 rpx 单位转换为 dp 单位。

**导出**: `rpx2Dp` 函数

**关键签名**:

```typescript
function rpx2Dp(rpx: number): number {
    // rpx × 转换系数 = dp
}
```

---

### qs.ts — 查询字符串处理

**功能**: URL 查询字符串的解析和序列化。

**导出**: `qs` 对象

**关键签名**:

```typescript
const qs = {
    parse(queryString: string): Record<string, any> {
        // 将查询字符串解析为对象
        // 支持: ?a=1&b=2 → { a: '1', b: '2' }
    },
    stringify(params: Record<string, any>): string {
        // 将对象序列化为查询字符串
        // 支持: { a: 1, b: 2 } → 'a=1&b=2'
    },
};
```

---

### debug.ts — 调试工具

**功能**: 开发环境调试辅助函数。

**导出**: 调试相关的工具函数

**关键实现**:
- 开发模式检测
- 调试日志输出（仅在开发环境生效）
- 性能计时辅助

---

### get-ffp-result.ts — FFP 结果获取

**功能**: 获取 FFP（First Frame Paint，首帧绘制）结果。

**导出**: `getFfpResult` 函数

**关键签名**:

```typescript
function getFfpResult(): Promise<number | null> {
    // 获取首帧绘制时间
    // 使用 PerformanceObserver 监听 first-contentful-paint
}
```

**依赖**: Performance API

---

### filter-component-props.ts — 组件属性过滤

**功能**: 过滤组件属性，移除无效或不允许的属性。

**导出**: `filterComponentProps` 函数

**关键签名**:

```typescript
function filterComponentProps(
    props: Record<string, any>,
    allowedProps?: string[]
): Record<string, any> {
    // 过滤掉不在白名单中的属性
    // 如果不传白名单，移除 undefined 和 null 值
}
```

---

### construct-url.ts — URL 构造

**功能**: 构造完整的请求 URL，处理基础URL拼接和参数添加。

**导出**: `constructUrl` 函数

**关键签名**:

```typescript
function constructUrl(base: string, path: string, params?: Record<string, any>): string {
    // 拼接 base + path + ?params
    // 处理路径中的重复斜杠
    // 处理已有查询参数的URL
}
```

---

### share.ts — 分享逻辑

**功能**: 通用的分享逻辑处理。

**导出**: 分享相关的辅助函数

**关键实现**:
- 分享参数标准化
- 分享渠道判断
- 分享回调处理

---

### resolve-i18n.ts — 国际化解析

**功能**: 解析国际化配置和资源。

**导出**: `resolveI18n` 函数

**关键签名**:

```typescript
function resolveI18n(options: {
    ns: string;
    lng: string;
}): Promise<void> {
    // 解析并加载国际化资源
    // 处理语言回退（如 ar-SA → ar → en）
}
```

---

### location-wrapper.ts — 定位封装

**功能**: 封装定位逻辑，提供统一的定位接口。

**导出**: `locationWrapper` 对象

**关键实现**:
- 封装定位 API 调用
- 处理定位权限
- 定位缓存
- 超时处理

---

### retry-util.ts — 重试工具

**功能**: 请求失败时的自动重试工具。

**导出**: `retryUtil` 函数

**关键签名**:

```typescript
interface RetryOptions {
    maxRetries?: number;      // 最大重试次数（默认 3）
    retryDelay?: number;      // 重试延迟（毫秒）
    retryCondition?: (error: any) => boolean;  // 重试条件判断
}

function retryUtil<T>(
    fn: () => Promise<T>,
    options?: RetryOptions
): Promise<T> {
    // 执行异步函数，失败时按策略重试
    // 指数退避: 每次重试延迟加倍
    // 条件重试: 仅在满足条件时重试
}
```

**被引用**: Native Ajax 模块

---

### common-interceptors.ts — 公共拦截器

**功能**: Ajax 请求的公共拦截器，注入公共参数。

**导出**: 公共拦截器函数

**关键实现**:
- 注入公共请求头（设备信息、版本号等）
- 注入公共查询参数（渠道、平台等）
- 指纹参数注入
- 请求签名（如需要）

**被引用**: Native `Ajax/index.ts`, H5 `Ajax/index.ts`

---

### calculate-renderable-component.ts — 渲染组件计算

**功能**: 根据页面数据和组件信息，计算实际需要渲染的组件列表。

**导出**: `calculateRenderableComponent` 函数

**关键签名**:

```typescript
function calculateRenderableComponent(
    renderItems: RenderItem[],
    componentInfos: ComponentInfo[]
): RenderItem[] {
    // 过滤掉不需要渲染的组件
    // 处理组件间的依赖关系
    // 按渲染优先级排序
}
```

**被引用**: `render-page.tsx`

---

### no-concurrent.ts — 防并发

**功能**: 防止同一异步函数并发执行，确保前一次执行完成后再执行下一次。

**导出**: `noConcurrent` 装饰器/高阶函数

**关键签名**:

```typescript
function noConcurrent<T extends (...args: any[]) => Promise<any>>(
    fn: T
): T {
    // 返回包装后的函数
    // 如果上一次调用尚未完成，忽略新的调用（或排队等待）
}
```

---

### timezone.ts — 时区处理

**功能**: 时区相关的工具函数，支持多时区地区的日期时间处理。

**导出**: 时区处理函数

**关键实现**:
- 时区列表和偏移量映射
- 时区转换
- 日期格式化（按城市ID）

---

### is-key-has-value.ts — 键值检测

**功能**: 检查对象中指定键是否有有效值（非 undefined、null、空字符串）。

**导出**: `isKeyHasValue` 函数

**关键签名**:

```typescript
function isKeyHasValue(obj: any, key: string): boolean {
    // 检查 obj[key] 是否有有效值
    // undefined → false
    // null → false
    // '' → false
    // 0 → true
    // false → true
}
```

---

### enums.ts — 枚举定义

**功能**: 定义跨模块共享的枚举类型。

**导出**: 共享枚举值

**关键实现**:
- 组件类型枚举
- 渲染模式枚举
- 环境类型枚举
- 其他共享枚举

---

## components/ — 通用组件

### image.tsx — 图片监控组件

**功能**: 包装图片组件，添加加载监控能力。

**导出**: `withImageLoadMonitor(monitor)` 高阶组件

**关键签名**:

```typescript
function withImageLoadMonitor(monitor: IMonitor): React.ComponentType {
    return function ImageWithMonitor(props) {
        // 监控图片加载耗时
        // 加载开始时记录时间点
        // 加载完成/失败时上报耗时
        // 使用 <Image /> 组件
    };
}
```

**关键设计**:
- 记录图片从开始加载到加载完成/失败的耗时
- 上报慢加载图片（超过阈值）
- 加载失败上报错误
- 不影响图片组件本身的功能

---

### lottie.tsx — Lottie 动画监控组件

**功能**: 包装 Lottie 动画组件，添加加载监控能力。

**导出**: `withLottieLoadMonitor(monitor)` 高阶组件

**关键签名**:

```typescript
function withLottieLoadMonitor(monitor: IMonitor): React.ComponentType {
    return function LottieWithMonitor(props) {
        // 监控 Lottie JSON 加载耗时
        // 监控 Lottie 渲染首帧耗时
        // 上报加载和渲染性能数据
        // 使用 <Lottie /> 组件
    };
}
```

**关键设计**:
- Lottie 动画通常是较大的 JSON 文件，加载耗时需要监控
- 区分资源加载耗时和渲染耗时
- 超时/加载失败上报

---

### vap.tsx — VAP 动效监控组件

**功能**: 包装 VAP 视频动效组件，添加加载监控能力。

**导出**: `withVapLoadMonitor(monitor)` 高阶组件

**关键签名**:

```typescript
function withVapLoadMonitor(monitor: IMonitor): React.ComponentType {
    return function VapWithMonitor(props) {
        // 监控 VAP 视频资源加载耗时
        // 监控 VAP 渲染首帧耗时
        // 上报加载和渲染性能数据
        // 使用 <Vap /> 组件
    };
}
```

**关键设计**:
- VAP 是视频动效，资源较大，加载耗时是关键性能指标
- 监控视频解码耗时
- 超时/加载失败上报

---

### template.tsx — 模板组件

**功能**: 通用模板组件，用于组件渲染的占位或框架。

**导出**: `Template` 组件

**关键实现**:
- 提供组件渲染的基础模板结构
- 处理组件通用逻辑（样式注入、事件绑定等）
- 可能用于渲染动态组件

---

## comp-fetch-store/ — 组件数据获取 Store

### comp-fetch-store/index.ts — compModel

**功能**: 基于 Rematch 的组件数据获取 model，管理组件的异步数据获取状态。

**导出**: `compModel` (Rematch model)

**关键签名**:

```typescript
const compModel = {
    name: 'compFetch',  // model 名称

    state: {
        dataMap: {},           // 组件数据映射 { compId: data }
        loadingMap: {},        // 加载状态映射 { compId: boolean }
        errorMap: {},          // 错误信息映射 { compId: Error }
    },

    effects: {
        // 获取组件数据
        async fetchCompData(payload: {
            compId: string;
            apiConfig: any;
            params?: any;
        }, rootState) {
            // 1. 设置 loading 状态
            // 2. 调用 API 获取数据
            // 3. 成功: 存储数据到 dataMap
            // 4. 失败: 存储错误到 errorMap
            // 5. 清除 loading 状态
        },
    },

    reducers: {
        setData(state, { compId, data }) {
            // 更新指定组件的数据
        },
        setLoading(state, { compId, loading }) {
            // 更新指定组件的加载状态
        },
        setError(state, { compId, error }) {
            // 更新指定组件的错误信息
        },
        clearCompData(state, { compId }) {
            // 清除指定组件的数据
        },
    },
};
```

**依赖**: `@sailorgundam/gundam-store`, `IAjax`

**关键设计**:
- 每个组件实例有独立的数据状态（通过 compId 区分）
- 支持加载状态追踪，便于 UI 展示 Loading
- 错误状态管理，便于展示错误 UI 和重试
- 通过 `addModel` 动态注册到 Store

---

## 模块间依赖关系

```
gdc-hoc.ts ──────→ event-bus.ts (组件间通信)
     │
     ├────→ constants.ts (COMP_STORE_ID)
     │
     └────→ IMonitor (错误边界上报)

common-interceptors.ts ──→ construct-url.ts (URL构造)
       │
       └──→ get-global-data.ts (获取全局参数)

retry-util.ts ────→ 被 Native Ajax 引用

qs.ts ────────────→ 被 Util.ts 引用

dp-to-rpx.ts ←──→ rpx-to-dp.ts (互为反向转换)

calculate-renderable-component.ts ──→ 被 render-page.tsx 引用

comp-fetch-store/index.ts ──→ gundam-store (Rematch)
                            ──→ IAjax (数据请求)

components/ (image/lottie/vap) ──→ IMonitor (性能上报)
```

## 使用场景

### GdcHOC 使用

```typescript
// 在 render-page.tsx 中
const enhanceComponent = createHOC({
    autoExposure: true,
    autoClick: true,
    errorBoundary: true,
});

// 包裹每个组件
const EnhancedComp = enhanceComponent(OriginalComp, compInfo);
```

### EventBus 使用

```typescript
// 组件 A 发送事件
core.eventBus.emit('cart-updated', { count: 3 });

// 组件 B 监听事件
core.eventBus.on('cart-updated', (data) => {
    console.log('购物车更新:', data.count);
});
```

### compModel 使用

```typescript
// 注册 model
store.addModel(compModel);

// 获取组件数据
await store.dispatch.compFetch.fetchCompData({
    compId: 'comp-001',
    apiConfig: { url: '/api/data' },
});

// 读取数据
const data = store.getState().compFetch.dataMap['comp-001'];
```