# @sailorgundam/gundam-seg-logic

> **版本**: 0.149.0  
> **定位**: 供给组件逻辑库 — 负责高达（Gundam）活动页 segment 数据的获取、渲染、筛选与跳转

---

## 概览

`sailor-gundam-seg-logic` 是 Gundam 供给组件的核心逻辑层，提供从数据请求到页面渲染的完整链路。它封装了 segment 数据的召回判断、接口请求（含重试）、数据适配与格式化、筛选交互、跳转及埋点等能力，使供给组件开发者只需关注 UI 展示，无需处理复杂的数据流逻辑。

### 核心数据流

```
createLogicModel → wnInit(获取 segmentIds) →
组件 getProvideData → checkWnSegmentIds(判断召回) →
getWnSegment(请求数据, 最多3次重试) → filter Hook → format Hook →
dataAdapter(适配数据模型) → setDataList → 渲染列表
```

---

## 安装

```bash
pnpm add @sailorgundam/gundam-seg-logic
```

---

## 快速开始

```typescript
import { Store, Components, SegHOC, utils } from '@sailorgundam/gundam-seg-logic';

// 1. 创建逻辑 Model（注册到全局 Store）
const logicModel = Store.createLogicModel({
  segmentList: ['segment_001', 'segment_002'],
});

// 2. 创建组件 Model（管理组件级状态）
const componentModel = Store.createComponentModel('myListComponent');

// 3. 使用 List 组件渲染供给列表
<Components.List
  model={componentModel}
  renderItem={(item) => <MyItem data={item} />}
/>
```

### 便捷导出

所有能力也可以通过 `GundamSegLogic` 命名空间统一访问：

```typescript
import GundamSegLogic from '@sailorgundam/gundam-seg-logic';

GundamSegLogic.Component  // 组件
GundamSegLogic.Jump       // 跳转
GundamSegLogic.Request    // 请求
GundamSegLogic.Report     // 埋点
GundamSegLogic.Asset      // 资源/图片
GundamSegLogic.Monitor    // 监控
GundamSegLogic.Util       // 工具函数
```

---

## 模块详解

---

## 1. 入口文件

### `src/index.ts`

主入口文件，统一导出以下模块：

| 导出 | 说明 |
|------|------|
| `Store` | 状态管理层（createComponentModel, createLogicModel） |
| `Components` | 组件层（ProvideList, List, ComponentTitle 等） |
| `SegHOC` | 供给高阶组件 |
| `utils` | 工具函数 |
| 类型 & 枚举 | TypeScript 类型定义与枚举常量 |

### `src/gundamSegLogic.ts`

便捷导出，将所有能力分组为 `GundamSegLogic` 命名空间：

| 分组 | 包含能力 |
|------|---------|
| `GundamSegLogic.Component` | UI 组件（List, FilterBar, ComponentTitle 等） |
| `GundamSegLogic.Jump` | 跳转相关（go, goMp, goH5OrApp, goTT） |
| `GundamSegLogic.Request` | 请求相关（fetchSegmentIds, fetchWnSegmentData） |
| `GundamSegLogic.Report` | 埋点相关（handleSegmentLog, handleView） |
| `GundamSegLogic.Asset` | 资源相关（图片优化、格式转换） |
| `GundamSegLogic.Monitor` | 监控相关（addError, addLogan, speed） |
| `GundamSegLogic.Util` | 工具函数（qs, numberToFixed 等） |

---

## 2. Store 层 (`src/store/`)

基于 Rematch Model 模式的状态管理，分为逻辑 Model 和组件 Model 两级。

### `store/index.ts`

| 函数 | 说明 |
|------|------|
| `createComponentModel()` | 创建组件级 Model，管理单个供给组件的状态（列表数据、分页、加载状态等） |
| `createLogicModel()` | 创建逻辑级 Model，管理全局 segment 数据（segmentList、segmentListMap、udsRandomRate） |

### `store/state.ts`

定义组件级状态的接口与初始值。

```typescript
interface ISegmentState {
  isPrefetch: boolean;      // 是否预取
  idsCheckIn: boolean;      // ID 召回校验结果
  isChecked: boolean;       // 是否已完成校验
  isFetching: boolean;      // 是否正在请求
  isFetchEnd: boolean;      // 是否请求结束（无更多数据）
  list: any[];              // 当前列表数据
  pageNum: number;          // 当前页码
  offset: number;           // 数据偏移量
  rankTraceId: string;      // 排序追踪 ID
}
```

| 函数 | 说明 |
|------|------|
| `getCommonState()` | 返回 `ISegmentState` 的默认初始值 |

### `store/reducers.ts`

| 函数 | 说明 |
|------|------|
| `getCommonReducers()` | 返回标准 reducer 集合 |

**Reducer 列表**:

| Reducer | 说明 |
|---------|------|
| `setDataList` | 替换整个列表数据 |
| `updateList` | 追加数据到列表（分页加载） |
| `setPageNum` | 设置当前页码 |
| `setOffset` | 设置数据偏移量 |
| `setIsFetching` | 设置请求中状态 |
| `setIsFetchError` | 设置请求错误状态 |
| `setIsFetchEnd` | 设置请求结束状态 |
| `updateCheckStatus` | 更新召回校验状态 |
| `setFormatedProvideRes` | 设置格式化后的供给响应数据 |
| `resetState` | 重置所有状态到初始值 |

### `store/selectors.ts`

| 函数 | 说明 |
|------|------|
| `getCommonSelectors()` | 返回空 selector 集合，预留扩展 |

### `store/model/index.ts`

逻辑 Model 定义：

| 字段/Effect | 类型 | 说明 |
|-------------|------|------|
| `segmentList` | State | Segment ID 列表 |
| `segmentListMap` | State | Segment ID 映射表（快速查找） |
| `udsRandomRate` | State | UDS 随机命中率 |
| `wnInit` | Effect | 初始化获取 segmentIds |

### `store/effects/index.ts`

| 函数 | 说明 |
|------|------|
| `getCommonEffects` | 返回标准 effect 集合 |

**Effect 列表**:

| Effect | 说明 |
|--------|------|
| `getProvideData` | 获取供给数据的总入口，串联校验→请求→适配→存储 |
| `checkWnSegmentIds` | 判断当前 segment 是否被召回（白名单匹配） |
| `getWnSegment` | 执行 segment 数据请求，失败自动重试最多 3 次 |

### `store/effects/payload.ts`

| 函数 | 说明 |
|------|------|
| `getFormatedPayload()` | 格式化请求参数，合并公共参数与业务参数 |
| `addPoiZnjt()` | 附加 POI 附近标签参数 |

### `store/effects/data-adapter.ts`

| 函数 | 说明 |
|------|------|
| `dataAdapter()` | 数据适配器，区分 `ProvideResModel`（标准模型）与 `ProvideResModelUds`（UDS 模型）两种响应格式，统一转换为组件可消费的数据结构 |

### `store/hooks/format/`

数据格式化 Hook 集合：

| Hook | 说明 |
|------|------|
| `setAnchorMapItem` | 设置锚点映射项，用于楼层定位 |
| `formatImg` | H5 图片优化：自动添加 webp 格式参数 + 按容器尺寸 resize |
| `setLogData` | 设置埋点数据 |
| `setBestShippingInfo` | 设置最优配送信息 |

---

## 3. Helper 层 (`src/helper/`)

提供 API 请求、数据校验、Hook 系统、监控、图片处理、URL 构造等辅助能力。

### 3.1 API 请求 (`helper/api/`)

#### `helper/api/segment.ts`

| 函数 | 说明 |
|------|------|
| `fetchWnSegmentData()` | 请求 WN Segment 数据（供给列表） |
| `fetchSegmentIds()` | 请求 Segment ID 列表（用于召回判断） |

#### `helper/api/request.ts`

| 函数 | 说明 |
|------|------|
| `fetch()` | 基于 `gdUtil.request` 封装的通用请求方法，自动附加公共参数与拦截器 |

#### `helper/api/ajax-wn-url.ts`

| 函数 | 说明 |
|------|------|
| `getWnSegmentUrl()` | 获取 WN Segment 接口的完整 URL（根据环境切换域名） |

#### `helper/api/add-extra-params.ts`

| 函数 | 说明 |
|------|------|
| `addExtraParams()` | 为请求参数追加额外参数（如 UTM、泳道、实验参数等） |

### 3.2 Segment 校验 (`helper/segment/`)

#### `helper/segment/check-segment.ts`

| 函数 | 说明 |
|------|------|
| `checkBySegmentIds()` | 根据 Segment ID 列表判断是否召回 |
| `checkByActivityIds()` | 根据活动 ID 列表判断是否召回 |

#### `helper/segment/supply-data.ts`

| 函数 | 说明 |
|------|------|
| `getLastSupplyIdStr()` | 获取最近一次供给 ID 字符串，用于去重和追踪 |

#### `helper/segment/is-hit-segment-uds.ts`

| 函数 | 说明 |
|------|------|
| `isHitSegmentUds()` | 根据 UDS 随机命中率判断当前请求是否命中 UDS 实验 |

### 3.3 Hook 系统 (`helper/hook/`)

基于 `SyncWaterFallHook` 实现的插件式 Hook 系统，允许组件在数据流的各阶段注入自定义逻辑。

#### `helper/hook/SegmentHook.ts`

`SegmentHook` 类 — 供给组件的核心 Hook 管理器。

| Hook 类型 | 说明 | 典型用途 |
|-----------|------|---------|
| `optControl` | 选项控制 | 动态决定是否执行某步操作 |
| `filter` | 数据过滤 | 过滤掉不满足条件的数据项 |
| `format` | 数据格式化 | 对数据项进行二次格式化（如图片优化、标签映射） |
| `mutation` | 数据变更 | 对数据进行结构性变更（如排序、分组） |

#### `helper/hook/SyncWaterFallHook.ts`

`SyncWaterFallHook` 类 — 同步瀑布流 Hook 实现。

- 上一个 Hook 的返回值作为下一个 Hook 的输入
- 支持注册多个回调，按优先级顺序执行
- 类似 Tapable 的 `SyncWaterfallHook`

#### `helper/hook/HookFactory.ts`

`HookFactory` 类 — Hook 工厂，用于批量创建和管理 Hook 实例。

### 3.4 监控 (`helper/monitor/`)

#### `helper/monitor/index.ts`

| 函数/常量 | 说明 |
|-----------|------|
| `addError()` | 上报错误到 Owl 监控 |
| `addLogan()` | 写入 Logan 日志 |
| `customAjaxErrorInfo()` | 自定义 Ajax 错误信息 |
| `getOwlErrName()` | 获取 Owl 错误名称（用于分类） |

#### `helper/monitor/speed.ts`

| 函数 | 说明 |
|------|------|
| `addSegmentIdsStartPoint()` | 添加 SegmentIds 请求起始测速点 |
| `addSegmentIdsEndPoint()` | 添加 SegmentIds 请求结束测速点 |
| `addFirstSegmentDataEnd()` | 添加首条 Segment 数据到达测速点 |

### 3.5 图片处理 (`helper/image/`)

#### `helper/image/format-image-helper.ts`

| 函数/常量 | 说明 |
|-----------|------|
| `imgOptimizer` | 图片优化器（自动 webp + 尺寸调整） |
| `isSupportWebp` | 检测当前环境是否支持 WebP 格式 |
| `convertImg2Webp()` | 将图片 URL 转换为 WebP 格式（添加查询参数） |
| `initWebpInCookie()` | 将 WebP 支持状态写入 Cookie（用于服务端判断） |

#### `helper/image/ban-more-to-normal-img-info.ts`

| 函数 | 说明 |
|------|------|
| `banMoreToNormalImgInfo()` | 禁止超出阈值的图片信息转换（防止大图被过度压缩） |

### 3.6 URL 构造 (`helper/urls/`)

#### `helper/urls/channel.ts`

| 函数 | 说明 |
|------|------|
| `getH5UrlHostAndPath()` | 获取 H5 页面的 Host + Path |
| `getAppUrlCollections()` | 获取 App 端 URL 集合（多端 Scheme） |
| `getTicketQueryString()` | 获取票据查询字符串（登录态透传） |
| `getCallScheme()` | 构造唤起 App 的 Scheme URL |

#### `helper/urls/vip-url.ts`

| 函数 | 说明 |
|------|------|
| `getVipUrl()` | 获取 VIP 页面 URL |

### 3.7 工具函数 (`helper/tool/`)

| 文件 | 导出 | 说明 |
|------|------|------|
| `no-concurrent.ts` | `NoConcurrent` | 装饰器，防止函数并发执行（如重复请求） |
| `qs.ts` | — | URL 查询参数解析/序列化 |
| `get-shop-schemes.ts` | — | 获取门店 Scheme URL |
| `get-app-info.ts` | — | 获取 App 信息（版本号、渠道等） |
| `get-coupon-config.ts` | — | 获取优惠券配置 |
| `is-sg-app.ts` | — | 判断是否为新加坡 App |
| `debug.ts` | — | 调试工具函数 |

---

## 4. Component 层 (`src/components/`)

### `List/index.tsx`

列表组件 — 供给数据的核心渲染容器。

| 特性 | 说明 |
|------|------|
| 分页加载 | 支持翻页请求更多数据 |
| 自动加载 | 滚动到底部自动触发下一页 |
| 骨架屏 | 首次加载时展示骨架占位 |
| 筛选 | 支持 FilterBar 联动筛选 |
| 懒渲染 | 支持首屏分步渲染优化 |

```tsx
<List
  model={componentModel}
  renderItem={(item, index) => <MyItem data={item} />}
  pageSize={10}
  autoLoad={true}
  showSkeleton={true}
/>
```

### `List/hooks.ts`

| Hook | 说明 |
|------|------|
| `useFetchDataFlow` | 数据请求流程 Hook，封装加载→请求→适配→渲染的完整流程 |

### `List/LoadMoreButton/index.tsx`

| 组件 | 说明 |
|------|------|
| `LoadMoreBtn` | "加载更多"按钮，点击触发下一页请求 |

### `List/Fallback/index.tsx`

| 组件 | 说明 |
|------|------|
| `Fallback` | 列表降级视图，数据加载失败或为空时的兜底展示 |

### `Filter/filter-bar/index.tsx`

| 组件 | 说明 |
|------|------|
| `FilterBar` | 筛选栏组件，支持 Button（平铺）和 DropDown（下拉）两种样式 |

```tsx
<FilterBar
  filters={[
    { key: 'category', label: '分类', options: [...] },
    { key: 'sort', label: '排序', options: [...] },
  ]}
  onChange={(filterValues) => { /* 筛选变更回调 */ }}
/>
```

### `Filter/filter-bar/filter-bar-item/index.tsx`

| 组件 | 说明 |
|------|------|
| `FilterBarItem` | 单个筛选项组件，支持 Button/DropDown 两种交互模式 |

### `Filter/filter-bar/filter-panel/index.tsx`

| 组件 | 说明 |
|------|------|
| `FilterPanel` | 筛选面板组件，支持单选和多选模式 |

### `ProvideList/index.tsx`

| 组件 | 说明 |
|------|------|
| `ProvideList` | 旧版供给列表容器（Legacy），保留向后兼容 |

### `ComponentTitle/index.tsx`

| 组件 | 说明 |
|------|------|
| `ComponentTitle` | 标题组件，支持文本类型和图片类型 |

```tsx
<ComponentTitle
  title="热门推荐"
  type="text"
  // 或
  titleImageUrl="https://..."
  type="image"
/>
```

### `EmptyAndErrorView/index.tsx`

| 组件 | 说明 |
|------|------|
| `EmptyAndErrorView` | 空数据和错误状态视图 |

### `StrokeText/index.tsx`

| 组件 | 说明 |
|------|------|
| `StrokeText` | 描边文字组件，用于突出显示文本 |

### `MaskAndPlaceholderView/index.tsx`

| 组件 | 说明 |
|------|------|
| `MaskAndPlaceholderView` | 图片占位 + 售罄遮罩组件，展示售罄/下架等状态 |

### `HorizontalList/index.tsx`

| 组件 | 说明 |
|------|------|
| `HorizontalList` | 横向滚动列表，用于一行展示多个商品/店铺卡片 |

### `Skeleton/index.tsx`

| 组件 | 说明 |
|------|------|
| `Skeleton` | 骨架屏组件，数据加载前的占位展示 |

### `Replacement/index.tsx`

| 组件 | 说明 |
|------|------|
| `Replacement` | 双层切换组件，支持 A/B 内容切换（如售罄替换） |

### `SegmentImg/index.tsx`

| 组件 | 说明 |
|------|------|
| `SegmentImg` | 楼层图片组件，支持图片优化与懒加载 |

### `api/types.ts`

核心数据类型定义：

| 类型 | 说明 |
|------|------|
| `DataItem` | 通用数据项 |
| `ProductDataItem` | 商品数据项 |
| `ShopDataItem` | 门店数据项 |
| `ListVO` | 列表视图对象（含分页信息） |
| `ListRequest` | 列表请求参数 |

### `utils/index.ts`

组件层工具函数：

| 函数 | 说明 |
|------|------|
| `reportNetworkError` | 上报网络错误 |
| `getCommonRequestParams` | 获取公共请求参数 |
| `getSoldoutStatus` | 获取售罄状态 |

---

## 5. Utils 层 (`src/utils/`)

### `handle-click.ts`

| 函数 | 说明 |
|------|------|
| `handleClick()` | 统一点击处理 — 曝光追踪 + 跳转执行 |

### `handle-jump/go.ts`

统一跳转方法集：

| 函数 | 说明 |
|------|------|
| `go()` | 通用跳转，自动识别目标类型 |
| `goMp()` | 跳转小程序页面 |
| `goH5OrApp()` | 跳转 H5 或 App 原生页面（根据环境自动选择） |
| `goTT()` | 跳转头条小程序 |

### `handle-jump/handle-jump-to.ts`

| 函数 | 说明 |
|------|------|
| `jumpToShopOrMultipleSpecification()` | 跳转到门店页或多规格选择页 |

### `handle-jump/handle-go-xbk.ts`

| 函数 | 说明 |
|------|------|
| `handleGoXBK()` | 跳转星巴克专属页面 |

### `handle-jump/handle-go-phf.ts`

| 函数 | 说明 |
|------|------|
| `handleGoPHF()` | 跳转品好饭专属页面 |

### `handle-view.ts`

| 函数 | 说明 |
|------|------|
| `handleView()` | 曝光处理 — 触发曝光埋点上报 |

### `handle-segment-log.ts`

| 函数 | 说明 |
|------|------|
| `handleSegmentLog()` | Segment 日志记录 — 统一处理点击和曝光日志 |

### `measure.ts`

| 函数 | 说明 |
|------|------|
| `measureInWindow()` | 测量元素在窗口中的位置和尺寸 |
| `measure()` | 测量元素相对于父容器的位置和尺寸 |

### `format.ts`

| 函数 | 说明 |
|------|------|
| `protocolFormat()` | 协议格式化（如补全 URL 协议头） |
| `numberToFixed()` | 数字保留小数位（安全版，避免浮点精度问题） |

### `fetchData.ts`

| 函数 | 说明 |
|------|------|
| `fetchDataByCache` | 带缓存的数据请求，避免重复请求相同数据 |

### `showcomp-func-map.ts`

| 导出 | 说明 |
|------|------|
| `showcompFuncMap` | 组件显示条件函数映射表 |
| `defaultShowComp` | 默认组件显示条件（始终显示） |

---

## 6. Enum 层 (`src/enum/`)

### `enum/const/index.ts`

| 常量 | 说明 |
|------|------|
| `LIB_STORE_ID` | 库级 Store ID 标识 |
| `DEFAULT_FIRST_PAGE_SIZE` | 默认首页数据条数 |
| `DEFAULT_PER_PAGE_SIZE` | 默认每页数据条数 |

### `enum/segment/`

Segment 相关枚举：

| 文件 | 枚举 | 说明 |
|------|------|------|
| `coupon.ts` | — | 优惠券类型枚举 |
| `datatype.ts` | — | 数据类型枚举（商品/门店/优惠券等） |
| `discount.ts` | — | 折扣类型枚举 |
| `poi.ts` | — | POI（门店）类型枚举 |
| `sku.ts` | — | SKU（商品）类型枚举 |
| `recommendreason.ts` | — | 推荐理由类型枚举 |
| `adaptertype.ts` | — | 数据适配器类型枚举 |

### `enum/monitor/`

| 文件 | 枚举 | 说明 |
|------|------|------|
| `owlErrCategory.ts` | — | Owl 错误分类枚举 |
| `owlErrLevel.ts` | — | Owl 错误等级枚举 |

### `enum/hook/`

| 文件 | 枚举 | 说明 |
|------|------|------|
| `segmentHookIdEnum.ts` | — | Segment Hook ID 枚举 |

### `enum/channel/`

| 文件 | 枚举 | 说明 |
|------|------|------|
| `channelEnum.ts` | — | 渠道枚举 |
| `ctypeEnum.ts` | — | 内容类型枚举 |

---

## 7. Interface 层 (`src/interface/segment/`)

Segment 相关 TypeScript 接口定义：

| 接口 | 说明 |
|------|------|
| `IPoiBase` | 门店基础信息 |
| `ISkuBase` | 商品基础信息 |
| `ICoupon` | 优惠券信息 |
| `IDiscount` | 折扣信息 |
| `ISku` | 完整商品信息（继承 ISkuBase + ICoupon + IDiscount 等） |
| `IPoi` | 完整门店信息（继承 IPoiBase + 关联 SKU 列表等） |
| `IProvideRes` | 供给响应数据结构 |

---

## 8. Class 层 (`src/class/segment/`)

Segment 相关数据模型类：

| 类 | 说明 |
|----|------|
| `ProvideResModel` | 供给响应标准模型 — 包含列表数据、分页信息、排序信息 |
| `SkuModel` | 商品数据模型 — 封装 SKU 属性与展示逻辑 |
| `PoiModel` | 门店数据模型 — 封装 POI 属性与展示逻辑 |
| `Tag` | 标签模型 — 通用标签数据结构 |
| `TopicData` | 话题数据模型 |
| `SpecAttr` | 规格属性模型 — 多规格商品属性 |
| `SkuKanoLabel` | 商品 Kano 标签模型 |
| `SkuDiscount` | 商品折扣模型 |
| `SkuCoupon` | 商品优惠券模型 |

---

## 9. Prop-Info 层 (`src/prop-info/`)

组件属性配置信息（用于 Gundam 平台的可视化配置）：

| 文件 | 说明 |
|------|------|
| `componentTitle` | ComponentTitle 属性配置 |
| `perPageSize` | 每页条数配置 |
| `newComponentStyle` | 新组件样式配置 |
| `bellyBand` | 腰带样式配置 |
| `shopSegmentIds` | 门店 Segment ID 配置 |
| `foodSegmentIds` | 商品 Segment ID 配置 |
| `moreButton` | "更多"按钮配置 |
| `filter` | 筛选配置 |
| `ComponentEmptyConfig` | 空状态配置 |

---

## 依赖关系

```
@sailorgundam/gundam-seg-logic
├── @sailorgundam/gundam-core        # 核心运行时（gdAjax, gdLog, gdMonitor, gdStore, gdBridge 等）
├── @sailorgundam/gundam-store       # Rematch 状态管理
├── @sailorgundam/gundam-utils       # 工具函数
├── @sailorgundam/gundam-multiports  # 跨端 UI 基础组件
└── @sailorgundam/gundam-ui          # UI 组件库
```

---

## License

Internal — 美团内部使用
