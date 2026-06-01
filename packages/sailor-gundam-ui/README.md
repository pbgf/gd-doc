# @sailorgundam/gundam-ui

> **版本**: 0.149.0  
> **定位**: 基于 Mach Pro 跨端引擎的业务 UI 组件库，遵循统一设计规范

---

## 概览

`sailor-gundam-ui` 是 Gundam 活动页的 UI 组件库，基于 Mach Pro 跨端框架构建，所有组件在 H5 和 Native 端表现一致。组件库严格遵循设计规范（Design Tokens），提供统一的颜色、字号、字重、圆角、动画等视觉变量，确保各业务场景下的视觉一致性。

### 设计原则

- **统一 Token**: 所有组件使用统一的设计变量（颜色/字号/字重/动画/圆角）
- **跨端一致**: H5 和 Native 端渲染结果一致
- **尺寸体系**: 每个组件提供多种预设尺寸，覆盖不同业务场景
- **可组合性**: 组件间可灵活组合，构建复杂 UI

---

## 安装

```bash
pnpm add @sailorgundam/gundam-ui
```

---

## 组件列表

| 组件 | 说明 | 关键特性 |
|------|------|---------|
| [Button](#button) | 按钮 | 5种尺寸, 3种类型, 可选箭头 |
| [Card](#card) | 卡片容器 | 3种尺寸, logo+内容+底部 |
| [Description](#description) | 描述文本 | 4种尺寸, 8种灰度 |
| [HighTag](#hightag) | 高亮标签 | 好评/排名/VIP 类型 |
| [Image](#image) | 增强图片 | 标签覆盖, 描述, 状态覆盖 |
| [Layout](#layout) | 两栏布局 | 6种尺寸 |
| [LongTag](#longtag) | 长标签 | blur/highlight 类型, 5种颜色 |
| [Modal](#modal) | 对话框 | 标题+消息+按钮+关闭+警告样式 |
| [Price](#price) | 价格展示 | 现价/原价, 5种尺寸, 整数/小数分离 |
| [PriceGroup](#pricegroup) | 价格组布局 | 4种方向 |
| [Row](#row) | 水平布局 | 对齐+间距+末尾省略 |
| [Score](#score) | 评分展示 | 星级图标 |
| [ShortTag](#shorttag) | 短标签 | 5种状态, 8种业务类型, 图片模式 |
| [ShortTagGroup](#shorttaggroup) | 标签组 | 无缝拼接, 分隔线模式 |
| [Switch](#switch) | 开关 | 3种尺寸 |
| [Title](#title) | 标题 | 5种尺寸, 粗体, 省略, 双行 |
| [Toast](#toast) | 提示 | 命令式 API, 自动隐藏 |
| [Border](#border) | 虚线边框 | 虚线/点线（Mach 兼容方案） |

---

## 组件详解

---

### Button

按钮组件，支持多种尺寸和样式类型。

**尺寸**: `xs` | `sm` | `md` | `lg` | `xl` | `xxl`

**类型**: `brand`（品牌主色） | `hollow`（镂空） | `default`（默认）

**属性**:

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `size` | `'xs' \| 'sm' \| 'md' \| 'lg' \| 'xl' \| 'xxl'` | `'md'` | 按钮尺寸 |
| `type` | `'brand' \| 'hollow' \| 'default'` | `'default'` | 按钮类型 |
| `arrow` | `boolean` | `false` | 是否显示右侧箭头 |
| `disabled` | `boolean` | `false` | 是否禁用 |
| `onPress` | `() => void` | — | 点击回调 |

```tsx
<Button size="md" type="brand" arrow onPress={() => console.log('clicked')}>
  查看全部
</Button>
```

---

### Card

卡片容器组件，由 logo 区域、内容区域和底部区域三部分组成。

**尺寸**: `l` | `m` | `s`

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `size` | `'l' \| 'm' \| 's'` | `'m'` | 卡片尺寸 |
| `logo` | `ReactNode` | — | Logo 区域内容 |
| `bottom` | `ReactNode` | — | 底部区域内容 |

```tsx
<Card
  size="m"
  logo={<Image source={{ uri: shopLogo }} />}
  bottom={<Price now="25.9" />}
>
  <Title>店铺名称</Title>
  <Description>店铺描述信息</Description>
</Card>
```

---

### Description

描述文本组件，用于展示辅助说明文字。

**尺寸**: `sm` | `md` | `lg` | `xl`

**灰度**: 8 种灰度级别，从最浅到最深

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `size` | `'sm' \| 'md' \| 'lg' \| 'xl'` | `'md'` | 文本尺寸 |
| `gray` | `1 \| 2 \| 3 \| 4 \| 5 \| 6 \| 7 \| 8` | `4` | 灰度级别 |
| `numberOfLines` | `number` | — | 最大行数（超出省略） |

```tsx
<Description size="md" gray={5} numberOfLines={2}>
  月售1234份 · 好评率96%
</Description>
```

---

### HighTag

高亮标签组件，用于突出展示特殊标签信息。

**类型**: `praise`（好评） | `rank`（排名） | `vip`（VIP）

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `type` | `'praise' \| 'rank' \| 'vip'` | `'praise'` | 标签类型 |
| `text` | `string` | — | 标签文本 |

```tsx
<HighTag type="praise" text="好评优选" />
<HighTag type="rank" text="No.1" />
<HighTag type="vip" text="VIP" />
```

---

### Image

增强图片组件，在基础图片上支持标签覆盖层、描述文字和状态覆盖层。

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `source` | `{ uri: string }` | — | 图片源 |
| `tags` | `ReactNode[]` | — | 标签覆盖层（如折扣角标） |
| `description` | `ReactNode` | — | 图片底部描述 |
| `statusOverlay` | `ReactNode` | — | 状态覆盖层（如售罄遮罩） |
| `placeholder` | `ReactNode` | — | 加载占位 |

```tsx
<Image
  source={{ uri: 'https://...' }}
  tags={[<ShortTag type="discount" text="7折" />]}
  statusOverlay={isSoldOut && <MaskAndPlaceholderView />}
  description={<Description>商品描述</Description>}
/>
```

---

### Layout

两栏布局组件，左侧固定宽度，右侧自适应。

**尺寸**: `xxl` | `xl` | `l` | `m` | `s` | `xs`（控制左侧宽度）

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `size` | `'xxl' \| 'xl' \| 'l' \| 'm' \| 's' \| 'xs'` | `'m'` | 布局尺寸（左侧宽度） |
| `left` | `ReactNode` | — | 左侧内容 |
| `right` | `ReactNode` | — | 右侧内容 |

```tsx
<Layout size="m" left={<Image source={{ uri: thumbnail }} />}>
  <Title>商品标题</Title>
  <Price now="19.9" origin="39.9" />
</Layout>
```

---

### LongTag

长标签组件，包含标签名（label）和标签值（content）两部分。

**类型**: `blur`（模糊/渐变） | `highlight`（高亮）

**颜色**: 5 种预设颜色

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `type` | `'blur' \| 'highlight'` | `'blur'` | 标签样式类型 |
| `color` | `string` | — | 预设颜色索引（5种） |
| `label` | `string` | — | 标签名称 |
| `content` | `string` | — | 标签内容/值 |

```tsx
<LongTag type="blur" color="red" label="配送" content="约30分钟" />
<LongTag type="highlight" color="orange" label="减" content="满20减5" />
```

---

### Modal

对话框组件，支持标题、消息、按钮组和关闭交互。

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `visible` | `boolean` | `false` | 是否显示 |
| `title` | `string` | — | 对话框标题 |
| `message` | `string \| ReactNode` | — | 对话框内容 |
| `buttons` | `ModalButton[]` | — | 按钮列表 |
| `closable` | `boolean` | `false` | 是否显示关闭按钮 |
| `alert` | `boolean` | `false` | 警告样式（图标+文字） |
| `onClose` | `() => void` | — | 关闭回调 |

```tsx
<Modal
  visible={showDialog}
  title="提示"
  message="确定要删除吗？"
  closable
  buttons={[
    { text: '取消', onPress: () => setShowDialog(false) },
    { text: '确定', type: 'brand', onPress: handleConfirm },
  ]}
/>
```

---

### Price

价格展示组件，支持现价/原价两种模式，以及整数与小数分离显示。

**尺寸**: `xs` | `sm` | `md` | `lg` | `xl`

**模式**: `now`（现价） | `origin`（原价，带删除线）

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `size` | `'xs' \| 'sm' \| 'md' \| 'lg' \| 'xl'` | `'md'` | 价格尺寸 |
| `now` | `string` | — | 现价 |
| `origin` | `string` | — | 原价（自动添加删除线） |
| `mode` | `'now' \| 'origin'` | `'now'` | 价格模式 |
| `newMode` | `boolean` | `false` | 整数+小数分离显示（整数大字，小数小字） |
| `symbol` | `string` | `'¥'` | 货币符号 |

```tsx
{/* 现价 + 原价 */}
<Price now="19.9" origin="39.9" size="lg" />

{/* 整数小数分离 */}
<Price now="19.9" size="xl" newMode />

{/* 仅原价 */}
<Price origin="39.9" mode="origin" />
```

---

### PriceGroup

价格组布局组件，将多个价格元素按指定方向排列。

**方向**: 4 种布局方向

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `direction` | `string` | — | 布局方向（4种） |

```tsx
<PriceGroup direction="horizontal">
  <Price now="19.9" size="lg" newMode />
  <Price origin="39.9" mode="origin" size="sm" />
</PriceGroup>
```

---

### Row

水平 Flex 布局组件，用于一行排列多个元素。

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `align` | `'start' \| 'center' \| 'end'` | `'center'` | 垂直对齐方式 |
| `spacing` | `number` | `0` | 子元素间距 |
| `lastChildEllipsis` | `boolean` | `false` | 最后一个子元素省略（单行截断） |

```tsx
<Row align="center" spacing={8} lastChildEllipsis>
  <Title>商品名称很长很长</Title>
  <ShortTag text="热销" />
</Row>
```

---

### Score

评分展示组件，使用星级图标显示评分。

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `value` | `number` | — | 评分值（0-5） |
| `showText` | `boolean` | `false` | 是否显示评分数字 |

```tsx
<Score value={4.5} showText />
```

---

### ShortTag

紧凑标签组件，提供多种状态色和业务类型预设。

**状态**: 5 种状态（如 default/success/warning/error/info）

**业务类型**: 8 种业务预设（如 discount/new/hot/vip 等）

**图片模式**: 支持图片标签

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `status` | `string` | `'default'` | 状态色（5种） |
| `type` | `string` | — | 业务类型（8种预设） |
| `text` | `string` | — | 标签文本 |
| `imageSource` | `{ uri: string }` | — | 图片标签源 |

```tsx
<ShortTag type="discount" text="7折" />
<ShortTag type="new" text="新品" />
<ShortTag status="warning" text="即将售罄" />
<ShortTag imageSource={{ uri: 'https://...' }} />
```

---

### ShortTagGroup

标签组组件，将多个 ShortTag 无缝拼接或用分隔线连接。

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `separator` | `boolean` | `false` | 是否显示分隔线模式 |
| `tags` | `ShortTagProps[]` | — | 标签配置列表 |

```tsx
<ShortTagGroup
  tags={[
    { type: 'discount', text: '7折' },
    { type: 'new', text: '新品' },
  ]}
/>

{/* 分隔线模式 */}
<ShortTagGroup separator tags={[...]} />
```

---

### Switch

开关切换组件。

**尺寸**: `l` | `m` | `s`

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `size` | `'l' \| 'm' \| 's'` | `'m'` | 开关尺寸 |
| `value` | `boolean` | `false` | 开关状态 |
| `onChange` | `(value: boolean) => void` | — | 状态变更回调 |
| `disabled` | `boolean` | `false` | 是否禁用 |

```tsx
<Switch size="m" value={enabled} onChange={setEnabled} />
```

---

### Title

标题文本组件，支持多种尺寸和显示模式。

**尺寸**: `xs` | `sm` | `md` | `lg` | `xl`

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `size` | `'xs' \| 'sm' \| 'md' \| 'lg' \| 'xl'` | `'md'` | 标题尺寸 |
| `bold` | `boolean` | `false` | 是否粗体 |
| `ellipsis` | `boolean` | `false` | 单行省略 |
| `twoLine` | `boolean` | `false` | 最多两行（超出省略） |

```tsx
<Title size="lg" bold ellipsis>
  商品标题
</Title>

<Title size="md" twoLine>
  两行标题最多展示两行超出部分省略号
</Title>
```

---

### Toast

提示组件，采用命令式 API 调用，支持自动隐藏。

| 方法 | 说明 |
|------|------|
| `Toast.show(options)` | 显示 Toast |
| `Toast.hide()` | 手动隐藏 Toast |

**Options**:

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `message` | `string` | — | 提示内容 |
| `duration` | `number` | `2000` | 自动隐藏时间（ms） |
| `position` | `string` | `'center'` | 显示位置 |

```tsx
Toast.show({ message: '已加入购物车', duration: 1500 });

// 手动隐藏
Toast.hide();
```

---

### Border

虚线边框组件。由于 Mach Pro Native 端不支持 CSS `border-style: dashed/dotted`，此组件用 View 模拟实现虚线和点线边框效果。

**类型**: `dashed`（虚线） | `dotted`（点线）

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `type` | `'dashed' \| 'dotted'` | `'dashed'` | 边框类型 |
| `color` | `string` | — | 边框颜色 |
| `borderWidth` | `number` | `1` | 边框宽度 |

```tsx
<Border type="dashed" color="#999">
  <View style={{ padding: 12 }}>
    <Text>虚线边框内的内容</Text>
  </View>
</Border>
```

---

## 辅助模块

### `helpers/classnames.ts`

CSS 类名构建器，用于条件组合 CSS 类名。

```typescript
import { classnames } from '@sailorgundam/gundam-ui/helpers/classnames';

classnames('base', isActive && 'active', { 'is-disabled': disabled });
// => 'base active is-disabled'
```

### `helpers/rem.ts`

rpx 到 dp 的单位转换工具，用于 Mach Pro Native 端的尺寸适配。

```typescript
import { rpxToDp } from '@sailorgundam/gundam-ui/helpers/rem';

const dp = rpxToDp(750); // 将设计稿 rpx 值转换为 Native dp 值
```

---

## 设计 Token (`styles/variable.less`)

所有组件使用的统一设计变量，定义在 `styles/variable.less` 中：

### 颜色体系

| Token 分类 | 说明 |
|-----------|------|
| 品牌色 | 主色/辅助色/品牌渐变 |
| 功能色 | 成功/警告/错误/信息 |
| 灰度色 | 8 级灰度（用于文本/背景/边框） |
| 业务色 | 价格色/折扣色/标签色等 |

### 字体体系

| Token 分类 | 说明 |
|-----------|------|
| 字号 | xs/sm/md/lg/xl/xxl 等预设尺寸 |
| 字重 | regular/medium/bold |

### 间距 & 尺寸

| Token 分类 | 说明 |
|-----------|------|
| 圆角 | 无/小/中/大/圆形 |
| 间距 | 内边距/外边距预设值 |

### 动画

| Token 分类 | 说明 |
|-----------|------|
| 时长 | 快/中/慢 |
| 缓动 | ease-in/ease-out/ease-in-out |

---

## 依赖关系

```
@sailorgundam/gundam-ui
├── @sailorgundam/gundam-multiports  # 跨端 UI 基础组件 (View/Text/Image/Scroller 等)
├── @sailorgundam/gundam-core        # 核心运行时
└── @wmfe/mach-pro-knb              # Native 桥接
```

---

## License

Internal — 美团内部使用
