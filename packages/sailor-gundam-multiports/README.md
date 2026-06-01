# sailor-gundam-multiports

> v0.149.0 · Cross-platform UI component library with dual H5/Native implementations

`sailor-gundam-multiports` 是 Gundam 跨平台 UI 组件库，每个组件均提供 H5 和 Native 双端实现，通过统一的 Props 接口实现一套代码、双端运行。基于 MachPro 框架，支持 View、Text、Image、Scroller、Modal、RichText、Lottie、VAP 等核心 UI 组件，以及 Fixed 定位、Portal 渲染、曝光检测等能力。

---

## 安装

```bash
npm install sailor-gundam-multiports
# 或
yarn add sailor-gundam-multiports
```

---

## 快速开始

```tsx
// H5 环境 — 自动引入 H5 实现
import { View, Text, Image } from 'sailor-gundam-multiports';

function ProductCard({ title, imageUrl }) {
  return (
    <View className="card">
      <Image src={imageUrl} resizeMode="cover" placeholder="default.png" />
      <Text content={title} maxLines={2} ellipsizeMode="tail" />
    </View>
  );
}
```

```tsx
// Native 环境 — 自动引入 Native 实现
import { View, Text, Image } from 'sailor-gundam-multiports/native';

// 同样的组件 API，底层渲染为 <view>、<text>、<image>
```

---

## 组件列表

| 组件 | H5 实现 | Native 实现 | 说明 |
|------|---------|-------------|------|
| View | `<div>` + IntersectionObserver | `<view>` / `<exposure>` | 基础容器 |
| Text | `<p><span>` | `<text>` | 文本显示 |
| Image | `background-image` | `<image>` | 图片展示 |
| ImageBackground | View + Image 复合 | `<image-background>` | 带背景图的容器 |
| Scroller | `overflow:scroll` + polyfill | `<scroller>` | 滚动容器 |
| Modal | body scroll lock | `<modal>` | 模态弹窗 |
| RichText | 迭代渲染 | `<rich-text>` | 富文本 |
| Lottie | lottie-web CDN | `<lottie>` | Lottie 动画 |
| Vap | LingboEffects CDN | `<infinite-effect>` | 视频动画 |
| Fixed | 固定/绝对定位 + RTL | 同左 + Portal | 固定定位 |
| Portal | RootSiblings 渲染 | 同左 | 根级渲染 |

---

## 组件详解

### View

基础容器组件，支持曝光检测。

**Props（`ViewProps`）：**

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `className` | `string` | — | CSS 类名 |
| `style` | `CSSProperties` | — | 内联样式 |
| `onClick` | `Function` | — | 点击事件 |
| `onExposure` | `Function` | — | 曝光回调（H5: IntersectionObserver） |

**H5 实现（`ViewH5`）：** 渲染为 `<div>`，通过 `IntersectionObserver` 实现曝光检测。

**Native 实现（`ViewNative`）：** 渲染为 `<view>` 或 `<exposure>`（当配置了 `onExposure` 时）。

---

### Text

文本显示组件，支持省略和多行截断。

**Props（`TextProps`）：**

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `content` | `string` | — | 文本内容 |
| `ellipsizeMode` | `'head' \| 'middle' \| 'tail' \| 'clip'` | `'tail'` | 省略模式 |
| `maxLines` | `number` | — | 最大行数 |

**H5 实现（`TextH5`）：** 渲染为 `<p>` + `<span>`，通过 CSS `-webkit-line-clamp` 实现多行截断。

**Native 实现（`TextNative`）：** 渲染为 `<text>`，使用原生 `ellipsizeMode` 和 `numberOfLines` 属性。

---

### Image

图片展示组件，支持懒加载、占位图、错误图和自动 RTL。

**Props（`ImageProps`）：**

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `src` | `string` | — | 图片地址 |
| `resizeMode` | `'contain' \| 'cover' \| 'stretch' \| 'center'` | `'cover'` | 缩放模式 |
| `placeholder` | `string` | — | 占位图 |
| `error` | `string` | — | 错误图 |
| `lazy` | `boolean` | `false` | 懒加载 |
| `autoRTL` | `boolean` | `false` | RTL 自动翻转 |

**`modeStyle.ts`：** 将 `resizeMode` 映射为 CSS `background-size` / `object-fit` 样式。

**H5 实现（`ImageH5`）：** 使用 `background-image` + CSS 实现，懒加载通过 `IntersectionObserver`。

**Native 实现（`ImageNative`）：** 渲染为 `<image>`，使用原生属性。

```tsx
<Image
  src="https://example.com/photo.jpg"
  resizeMode="cover"
  placeholder="placeholder.png"
  error="error.png"
  lazy
  autoRTL
/>
```

---

### ImageBackground

带背景图的容器组件，允许在图片上方叠加子元素。

**H5 实现（`ImageBackgroundH5`）：** View + Image 复合渲染，子元素通过 `position: relative` 叠加。

**Native 实现（`ImageBackgroundNative`）：** 渲染为 `<image-background>`。

```tsx
<ImageBackground src="bg.jpg" resizeMode="cover">
  <Text content="Overlay Text" />
</ImageBackground>
```

---

### Scroller

滚动容器组件。

**Props（`ScrollerProps`）：**

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `scrollDirection` | `'horizontal' \| 'vertical'` | `'vertical'` | 滚动方向 |
| `showScrollbar` | `boolean` | `true` | 显示滚动条 |
| `onScroll` | `Function` | — | 滚动事件 |

**H5 实现（`ScrollerH5`）：** `overflow: scroll` + 惯性滚动 polyfill。

**Native 实现（`ScrollerNative`）：** 渲染为 `<scroller>`。

---

### Modal

模态弹窗组件。

**Props（`ModalProps`）：**

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `visible` | `boolean` | `false` | 是否显示 |
| `onClose` | `Function` | — | 关闭回调 |
| `animationType` | `'none' \| 'fade' \| 'slide'` | `'fade'` | 动画类型 |

**H5 实现（`ModalH5`）：** 弹窗显示时锁定 body 滚动（`overflow: hidden`），关闭时恢复。

**Native 实现（`ModalNative`）：** 渲染为 `<modal>`，使用原生动画。

---

### RichText

富文本渲染组件，支持文本、图片、占位符混合内容。

**Props（`IRichTextProps`）：**

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `content` | `Array<TextItem \| ImageItem \| PlaceholderItem>` | — | 内容数组 |

每个内容项包含 `type` 字段标识类型：

- `text`：文本内容
- `image`：图片内容
- `placeholder`：占位内容

**H5 实现（`RichTextH5`）：** 迭代渲染内容数组，文本渲染为 `<span>`，图片渲染为 `<img>`。

**Native 实现（`RichTextNative`）：** 渲染为 `<rich-text>`。

```tsx
<RichText
  content={[
    { type: 'text', value: 'Hello ' },
    { type: 'image', src: 'icon.png', width: 16, height: 16 },
    { type: 'text', value: ' World' },
  ]}
/>
```

---

### Lottie

Lottie 动画播放组件。

**Props（`LottieProps`）：**

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `path` | `string` | — | 动画 JSON URL |
| `animationData` | `object` | — | 动画 JSON 数据（与 path 二选一） |
| `autoplay` | `boolean` | `false` | 自动播放 |
| `loop` | `boolean` | `true` | 循环播放 |

**H5 实现（`LottieH5`）：** 通过 CDN 动态加载 `lottie-web` 库，使用 `<canvas>` 渲染。

**Native 实现（`LottieNative`）：** 渲染为 `<lottie>`。

```tsx
<Lottie
  path="https://cdn.example.com/animation.json"
  autoplay
  loop
/>
```

---

### Vap

视频动画播放组件（LingboEffects / InfiniteEffect）。

**Props（`VapProps`）：**

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `src` | `string` | — | 视频资源地址 |
| `content` | `string` | — | 动画配置 |
| `autoplay` | `boolean` | `false` | 自动播放 |

**H5 实现（`VapH5`）：** 通过 CDN 动态加载 LingboEffects 库。

**Native 实现（`VapNative`）：** 渲染为 `<infinite-effect>`。

```tsx
<Vap
  src="https://cdn.example.com/effect.mp4"
  content="effect-config"
  autoplay
/>
```

---

### Fixed

固定/绝对定位组件，支持 RTL 布局和 Portal 集成。

**PureComponent 实现**，根据配置渲染为 `position: fixed` 或 `position: absolute`，自动处理 RTL 环境下的 `left/right` 互换。

当结合 Portal 使用时，Fixed 组件可渲染到 DOM 根节点，避免被父容器 `overflow: hidden` 裁切。

```tsx
<Fixed position="fixed" top={0} left={0}>
  <Text content="Sticky Header" />
</Fixed>
```

---

### Portal

根级渲染组件，通过 RootSiblings 机制将子组件渲染到 DOM 根节点。

**PureComponent 实现**，常与 Fixed 组件配合使用，实现弹窗、浮层等场景。

```tsx
<Portal>
  <Modal visible={showModal}>
    <Text content="Modal Content" />
  </Modal>
</Portal>
```

---

## 工具函数

### `src/elements/utils/const.ts`

| 导出 | 说明 |
|------|------|
| `EQUAL_MODE_MAP` | 曝光模式映射常量 |
| `getSupportExposureComponent()` | 获取当前平台支持的曝光组件 |

---

### `src/elements/utils/classnames.ts`

CSS 类名构建器（同 `sailor-gundam-utils` 中的 `classNames`，组件库内置版本）。

---

### `src/elements/utils/add-measure-function.ts`

DOM measure API polyfill，为 H5 组件提供 `measure` / `measureInWindow` 等布局测量方法。

```typescript
// 为 ref 添加 measure 方法
addMeasureFunction(ref);
ref.current.measure((x, y, width, height) => {
  console.log({ x, y, width, height });
});
```

---

### `src/elements/utils/handle-click-event.ts`

| 函数 | 说明 |
|------|------|
| `h5HandleClickEvent(event, callback)` | H5 点击事件处理（从 SyntheticEvent 提取坐标） |
| `machHandleClickEvent(event, callback)` | Native 点击事件处理（从 MachEvent 提取坐标） |

---

### `src/elements/utils/use-ref-hand-click.ts`

#### `useRefHandClick`

Hook：统一处理 H5/Native 点击事件差异。

```typescript
const handleClick = useRefHandClick((x, y) => {
  console.log('clicked at', x, y);
});
```

---

### `src/elements/utils/use-on-layout.ts`

基于 `MutationObserver` 的布局检测 Hook，监听元素尺寸变化。

```typescript
const ref = useRef(null);
const layout = useOnLayout(ref);
console.log(layout); // { x, y, width, height }
```

---

### `src/elements/utils/root-siblings.tsx`

React Portal-to-Root 机制，实现组件渲染到 DOM 根节点。Portal 和 Fixed 组件的底层实现。

---

### `src/elements/utils/onViewObserver.ts`

H5 轮询曝光检测器（20ms 轮询间隔），用于 View 组件的曝光回调。

---

### `src/elements/utils/onViewHelper.ts`

Native 轮询曝光检测器（20ms 轮询间隔），用于 Native View 组件的曝光回调。

---

### `src/elements/utils/event-transform.ts`

TouchEvent → MachPro 拖拽事件转换器，将标准 TouchEvent 转换为 MachPro 拖拽事件格式。

---

### `src/elements/utils/h5-load-js.ts`

动态 `<script>` 加载器，用于 H5 环境下按需加载外部 JS 库（如 lottie-web、LingboEffects）。

```typescript
loadJS('https://cdn.example.com/lib.js').then(() => {
  console.log('Library loaded');
});
```

---

### `src/elements/utils/compare-versions.ts`

语义化版本比较工具（同 `sailor-gundam-utils` 中的实现，组件库内置版本）。

---

## 类型系统

### `src/types.d.ts` — MachPro 全局类型

为 MachPro 框架提供全局类型声明：

| 类型 | 说明 |
|------|------|
| JSX 元素类型 | View、Text、Image 等 Native 标签的类型声明 |
| `Env` | MachPro 环境配置类型 |
| `Mach` | Mach 运行时全局对象类型 |
| 动画相关类型 | 动画配置、过渡效果类型声明 |

---

### `src/h5-api/mach-api.ts` — Mach 运行时 Shim

为 H5 环境提供 Mach 运行时 API 的兼容层（shim），使 H5 代码可以使用 Native 专属的 `Mach` API。

---

## 架构概览

```
sailor-gundam-multiports
├── src/
│   ├── index.ts                     # H5 barrel 导出
│   ├── index.native.ts              # Native barrel 导出
│   ├── types.d.ts                   # MachPro 全局类型系统
│   ├── h5-api/
│   │   └── mach-api.ts              # H5 Mach 运行时 Shim
│   └── elements/
│       ├── view/
│       │   ├── index.ts             # H5 导出
│       │   ├── index.h5.tsx         # ViewH5（<div> + IntersectionObserver）
│       │   ├── index.native.tsx     # ViewNative（<view>/<exposure>）
│       │   ├── type.ts              # ViewProps
│       │   └── demo.tsx             # 使用示例
│       ├── text/
│       │   ├── index.ts             # H5 导出
│       │   ├── index.h5.tsx         # TextH5（<p><span>）
│       │   ├── index.native.tsx     # TextNative（<text>）
│       │   ├── type.ts              # TextProps
│       │   └── demo.tsx
│       ├── image/
│       │   ├── index.ts
│       │   ├── index.h5.tsx         # ImageH5（background-image）
│       │   ├── index.native.tsx     # ImageNative（<image>）
│       │   ├── type.ts              # ImageProps
│       │   ├── modeStyle.ts         # resizeMode → CSS 映射
│       │   └── demo.tsx
│       ├── image-background/
│       │   ├── index.ts
│       │   ├── index.h5.tsx         # ImageBackgroundH5（View+Image）
│       │   ├── index.native.tsx     # ImageBackgroundNative（<image-background>）
│       │   ├── type.ts
│       │   └── demo.tsx
│       ├── scroller/
│       │   ├── index.ts
│       │   ├── index.h5.tsx         # ScrollerH5（overflow:scroll + polyfill）
│       │   ├── index.native.tsx     # ScrollerNative（<scroller>）
│       │   ├── type.ts
│       │   └── demo.tsx
│       ├── modal/
│       │   ├── index.ts
│       │   ├── index.h5.tsx         # ModalH5（body scroll lock）
│       │   ├── index.native.tsx     # ModalNative（<modal>）
│       │   ├── type.ts
│       │   └── demo.tsx
│       ├── rich-text/
│       │   ├── index.ts
│       │   ├── index.h5.tsx         # RichTextH5（迭代渲染）
│       │   ├── index.native.tsx     # RichTextNative（<rich-text>）
│       │   ├── type.ts              # IRichTextProps
│       │   └── demo.tsx
│       ├── lottie/
│       │   ├── index.ts
│       │   ├── index.h5.tsx         # LottieH5（lottie-web CDN）
│       │   ├── index.native.tsx     # LottieNative（<lottie>）
│       │   ├── type.ts              # LottieProps
│       │   └── demo.tsx
│       ├── vap/
│       │   ├── index.ts
│       │   ├── index.h5.tsx         # VapH5（LingboEffects CDN）
│       │   ├── index.native.tsx     # VapNative（<infinite-effect>）
│       │   ├── type.ts              # VapProps
│       │   └── demo.tsx
│       ├── Fixed/
│       │   ├── index.ts
│       │   ├── index.h5.tsx         # Fixed PureComponent
│       │   ├── index.native.tsx
│       │   ├── type.ts
│       │   └── demo.tsx
│       ├── Portal/
│       │   ├── index.ts
│       │   ├── index.h5.tsx         # Portal PureComponent
│       │   ├── index.native.tsx
│       │   ├── type.ts
│       │   └── demo.tsx
│       └── utils/
│           ├── const.ts             # EQUAL_MODE_MAP, getSupportExposureComponent
│           ├── classnames.ts        # CSS 类名构建器
│           ├── add-measure-function.ts   # DOM measure API polyfill
│           ├── handle-click-event.ts     # H5/Native 点击事件处理
│           ├── use-ref-hand-click.ts     # useRefHandClick Hook
│           ├── use-on-layout.ts          # MutationObserver 布局检测
│           ├── root-siblings.tsx         # React portal-to-root 机制
│           ├── onViewObserver.ts         # H5 曝光检测器（20ms 轮询）
│           ├── onViewHelper.ts           # Native 曝光检测器（20ms 轮询）
│           ├── event-transform.ts        # TouchEvent → MachPro 拖拽事件
│           ├── h5-load-js.ts             # 动态 <script> 加载器
│           └── compare-versions.ts       # 语义化版本比较
```

---

## 双端实现机制

每个 UI 组件均遵循以下文件结构：

| 文件 | 说明 |
|------|------|
| `type.ts` | 统一的 Props 类型定义 |
| `index.h5.tsx` | H5 端实现（React DOM） |
| `index.native.tsx` | Native 端实现（MachPro Native 组件） |
| `index.ts` | H5 端导出入口 |
| `demo.tsx` | 使用示例 |

**自动平台选择：** H5 项目引入 `sailor-gundam-multiports` 时自动加载 H5 实现，Native 项目引入 `sailor-gundam-multiports/native` 时加载 Native 实现。

---

## 注意事项

- Lottie 和 VAP 组件在 H5 环境下通过 CDN 动态加载依赖库，首次渲染可能有延迟
- 曝光检测使用 20ms 轮询机制，高频场景下注意性能影响
- `ModalH5` 打开时会锁定 body 滚动，关闭后自动恢复
- `Fixed` 组件在 RTL 布局下自动处理 `left/right` 互换
- `Portal` 组件通过 RootSiblings 机制渲染到 DOM 根节点，可避免父容器 `overflow: hidden` 裁切
- `ImageH5` 使用 `background-image` 实现而非 `<img>` 标签，`resizeMode` 映射为 CSS `background-size`
- H5 环境下的 `mach-api.ts` 提供了 Mach 运行时 Shim，使 H5 代码可无缝调用 Native 专属 API
