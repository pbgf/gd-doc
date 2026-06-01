# templates/ — 项目模板目录

Gundam CLI 使用的项目模板文件，用于 `gundam init` 命令生成新的组件项目。模板由 `@sailorgundam/gundam-generator` 处理，支持 Mustache 变量替换和条件文件选择。

---

## 目录结构

```
templates/
├── entry/                           # 开发入口模板
│   ├── main.tsx                     # H5 开发入口
│   ├── h5main.tsx                   # 备用 H5 入口
│   └── index.html                   # HTML 外壳
└── component/                       # 组件项目模板
    ├── pkg.json                     # package.json 模板
    ├── mach.config.js               # MachPro 配置模板
    ├── src/                         # 组件源码模板
    │   ├── desc.ts                  # 组件描述/元数据
    │   ├── index.function.tsx       # 函数组件模板
    │   ├── index.function.supply.tsx # 函数供给组件模板
    │   ├── index.class.tsx          # 类组件模板
    │   ├── index.class.supply.tsx   # 类供给组件模板
    │   ├── index.less               # BEM 样式模板
    │   ├── model.ts                 # 供给组件数据模型
    │   └── types.d.ts              # 全局类型声明
    └── mock/                        # Mock 数据模板
        ├── globalData.ts            # 模拟后端数据
        ├── i18nResource.ts          # 国际化资源
        ├── index.tsx                # Mock 组合根
        └── pageConfig.ts            # 模拟页面容器配置
```

---

## entry/ — 开发入口模板

### `main.tsx` — H5 开发入口

**功能**: H5 模式开发时的应用入口点。

**实现细节**:
1. 导入 `pageInit` 初始化 Gundam 运行时
2. 加载 mock 数据（`globalData`、`pageConfig`）
3. 通过 Redux store 的 `dispatch` 注入模拟数据
4. 渲染目标组件

**关键流程**:
```typescript
// 伪代码
import { pageInit } from '@sailorgundam/gundam-core'
import globalData from '../mock/globalData'
import pageConfig from '../mock/pageConfig'

pageInit({ globalData, pageConfig })
  .then(() => {
    // 加载并渲染组件
  })
```

**用途**: `gundam dev --type h5` 时 Vite 以此文件为入口启动开发服务器。

---

### `h5main.tsx` — 备用 H5 入口

**功能**: H5 模式的备用入口点，与 `main.tsx` 类似但使用不同的初始化策略。

**区别**: 适用于不需要完整 Gundam 运行时初始化的轻量开发场景。

---

### `index.html` — HTML 外壳

**功能**: H5 开发模式下的完整 HTML 外壳，包含所有必要的脚本和样式。

**关键内容**:

| 元素 | 说明 |
|------|------|
| `<meta viewport>` | 移动端视口配置，禁用缩放 |
| Owl 脚本 | 前端监控 SDK，用于错误追踪和性能监控 |
| LXAnalytics 脚本 | 灵犀埋点 SDK，用于用户行为分析 |
| flexible rem 脚本 | 阿里 flexible 方案，动态设置根字体大小实现 rem 适配 |
| KeeTa 字体 | 品牌字体 CDN 引入 |
| H5Guard | H5 安全防护脚本 |

**实现细节**:
- viewport 设置为 `width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no`
- flexible 脚本根据设备宽度动态计算 `html` 的 `font-size`
- 所有外部脚本通过 CDN 加载
- 字体使用 `@font-face` 声明，优先使用本地缓存

---

## component/ — 组件项目模板

### `pkg.json` — package.json 模板

**功能**: 组件项目的 `package.json` 模板，使用 Mustache 变量替换。

**Mustache 变量**:

| 变量 | 来源 | 说明 |
|------|------|------|
| `{{&scope}}` | init 问答 | npm scope（如 `@sailorgundam`） |
| `{{&appName}}` | init 问答 | 组件名称 |

**模板示例**:
```json
{
  "name": "{{&scope}}/{{&appName}}",
  "version": "0.0.1",
  "main": "dist/cjs/index.js",
  "module": "dist/esm/index.js",
  "types": "dist/esm/index.d.ts"
}
```

**生成逻辑**:
- `gundam-generator` 读取此模板
- 用用户在 init 问答中的回答替换 Mustache 变量
- 写入目标项目的 `package.json`

---

### `mach.config.js` — MachPro 配置模板

**功能**: MachPro 跨端框架的项目配置文件模板。

**关键配置项**:
- 项目入口文件
- 平台目标（H5/Native/小程序）
- 构建选项
- 路径别名

**用途**: `gundam dev --type machpro` 和 `gundam build`（Native 模式）时读取此配置。

---

### `src/desc.ts` — 组件描述/元数据

**功能**: 定义组件的可视化编辑器元数据，供 Gundam 可视化搭建平台使用。

**导出内容**:
- 组件名称和描述
- 属性列表（名称、类型、默认值、描述）
- 事件列表（名称、参数、描述）
- 组件分类和标签

**设计意图**:
- 构建时生成 `gdc-description.json`
- 可视化编辑器读取此文件来渲染组件的属性面板
- 使非开发人员也能通过可视化界面配置组件

---

### `src/index.function.tsx` — 函数组件模板

**功能**: 纯函数组件的标准模板。

**关键特征**:
- 使用 `GdcHOC` 高阶组件包裹
- GdcHOC 自动注入：埋点、错误边界、首屏曝光
- 接收标准 props（由 Gundam 运行时注入）

**模板结构**:
```typescript
import { GdcHOC } from '@sailorgundam/gundam-core'

interface Props {
  // 组件属性
}

const MyComponent: React.FC<Props> = (props) => {
  return (
    <div className="my-component">
      {/* 组件内容 */}
    </div>
  )
}

export default GdcHOC(MyComponent)
```

---

### `src/index.function.supply.tsx` — 函数供给组件模板

**功能**: 函数式供给组件的标准模板，在函数组件基础上增加数据供给能力。

**与普通函数组件的区别**:
- 导入 `connect` 从 `@sailorgundam/gundam-store`
- 使用 Redux `connect` HOC 连接全局 store
- 通过 `mapStateToProps` 获取供给数据
- 通过 `mapDispatchToProps` 分发数据请求

**模板结构**:
```typescript
import { GdcHOC } from '@sailorgundam/gundam-core'
import { connect } from '@sailorgundam/gundam-store'

const MySupplyComponent: React.FC<Props> = (props) => {
  // props.dataList 来自供给逻辑
  return (...)
}

export default connect(
  mapStateToProps,
  mapDispatchToProps
)(GdcHOC(MySupplyComponent))
```

---

### `src/index.class.tsx` — 类组件模板

**功能**: 类组件的标准模板，使用 React Class Component 语法。

**关键特征**:
- 继承 `React.Component`
- 同样使用 `GdcHOC` 包裹
- 包含 `render()` 方法和生命周期方法

**适用场景**: 兼容旧代码或需要复杂生命周期管理的组件。

---

### `src/index.class.supply.tsx` — 类供给组件模板

**功能**: 类式供给组件的标准模板，结合类组件和 Redux connect。

**与函数供给组件的关系**: 功能等价，仅写法不同（class vs function）。

---

### `src/index.less` — BEM 样式模板

**功能**: 组件的样式文件模板，遵循 BEM 命名规范。

**BEM 命名规范**:
- Block: `.my-component`
- Element: `.my-component__element`
- Modifier: `.my-component--modifier`

**模板内容**:
```less
@import '~@sailorgundam/gundam-core/dist/esm/style/variables.less';

.my-component {
  &__header { ... }
  &__content { ... }
  &__footer { ... }
  &--active { ... }
}
```

**设计意图**:
- 统一的 BEM 命名避免样式冲突
- 导入 Gundam 设计变量确保视觉一致性
- Less 预处理器支持嵌套和变量

---

### `src/model.ts` — 供给组件数据模型

**功能**: 定义供给组件的数据模型，用于 `@sailorgundam/gundam-store` 的 Rematch model。

**关键内容**:
- `state`: 初始状态（数据列表、加载状态、错误信息等）
- `reducers`: 同步状态更新函数
- `effects`: 异步数据请求函数（如 `getProvideData`）

**模板结构**:
```typescript
import { createModel } from '@sailorgundam/gundam-store'

export default createModel({
  name: 'mySupplyModel',
  state: {
    dataList: [],
    loading: false,
  },
  reducers: {
    setDataList: (state, payload) => {
      state.dataList = payload
    },
  },
  effects: {
    async getProvideData() {
      // 数据请求逻辑
    },
  },
})
```

---

### `src/types.d.ts` — 全局类型声明

**功能**: 声明 MachPro 和 JSX 的全局类型，确保 TypeScript 编译通过。

**声明的全局类型**:
- MachPro 环境变量类型
- JSX 内置元素类型扩展
- Gundam 运行时全局对象类型

**关键声明**:
```typescript
declare namespace JSX {
  interface IntrinsicElements {
    // MachPro Native 组件类型
    view: any
    text: any
    image: any
    scroller: any
    // ...
  }
}

declare namespace NodeJS {
  interface ProcessEnv {
    MACH_ENV: 'h5' | 'native' | 'mp'
  }
}
```

---

## mock/ — Mock 数据模板

### `globalData.ts` — 模拟后端数据

**功能**: 模拟 Gundam 运行时从后端获取的 `globalData`。

**数据结构**:
```typescript
export default {
  pageId: 'mock_page_id',
  userId: 'mock_user_id',
  // ... 其他后端注入数据
}
```

**用途**: `gundam dev` 时注入到 Gundam 运行时，模拟真实后端数据环境。

---

### `i18nResource.ts` — 国际化资源

**功能**: 空的国际化资源模板。

**数据结构**:
```typescript
export default {
  zh: {},
  en: {},
  // ... 其他语言
}
```

**用途**: 开发时提供空的 i18n 资源占位，运行 i18nSync 后填充真实翻译。

---

### `index.tsx` — Mock 组合根

**功能**: 将所有 mock 数据组合起来，作为开发时的数据入口。

**组合内容**:
- 导入 `globalData`、`pageConfig`、`i18nResource`
- 组合为统一的 mock 数据对象
- 供 `entry/main.tsx` 使用

---

### `pageConfig.ts` — 模拟页面容器配置

**功能**: 模拟 Gundam 页面容器的配置数据。

**数据结构**:
```typescript
export default {
  pageContainer: {
    // 页面容器配置
  },
  renderTree: {
    // 渲染树配置（组件树结构）
  },
}
```

**用途**: `gundam dev` 时模拟页面容器的配置，使组件在开发环境中获得与生产环境相同的容器上下文。

---

## 模板选择逻辑

`gundam init` 根据用户问答结果选择模板：

| 问答答案 | 选择的模板文件 |
|----------|---------------|
| 组件风格 = function, 供给 = no | `index.function.tsx` |
| 组件风格 = function, 供给 = yes | `index.function.supply.tsx` |
| 组件风格 = class, 供给 = no | `index.class.tsx` |
| 组件风格 = class, 供给 = yes | `index.class.supply.tsx` |

未选中的变体文件在项目生成后被删除，仅保留匹配的组件文件并重命名为 `index.tsx`。

---

## Mustache 变量一览

模板中使用的所有 Mustache 变量：

| 变量 | 来源 | 使用的模板文件 |
|------|------|---------------|
| `{{&scope}}` | init 问答 - npm scope | `pkg.json` |
| `{{&appName}}` | init 问答 - 组件名 | `pkg.json` |

> `{{&}}` 语法表示不转义 HTML，因为 scope 中可能包含 `/` 等特殊字符。

---

## 模板处理流程

```
gundam init
    → 9 个交互问答
    → 构造模板变量
    → gundam-generator 接管
        ├── 复制 templates/ 到目标目录
        ├── Mustache 变量替换
        ├── 按条件删除不需要的变体文件
        ├── 重命名保留的组件文件为 index.tsx
        ├── 添加/删除 i18n 相关文件
        ├── 添加/删除 launch 相关文件
        ├── pnpm install
        ├── i18n 同步
        └── git init + husky
```
