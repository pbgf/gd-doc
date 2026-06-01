# sailor-gundam-store

> v0.149.0 · Rematch v2-based state management library for Gundam

`sailor-gundam-store` 提供基于 [Rematch v2](https://rematchjs.org/) 的状态管理方案，为 Gundam 活动页提供全局单例 Store、Model 动态注册、Memoized Selector 以及自定义 React Connect HOC。

---

## 安装

```bash
npm install sailor-gundam-store
# 或
yarn add sailor-gundam-store
```

---

## 快速开始

```typescript
import { init, addModel, connect } from 'sailor-gundam-store';

// 1. 定义 Model
const counter = {
  name: 'counter',
  state: { count: 0 },
  reducers: {
    increment: (state, payload: number = 1) => ({ count: state.count + payload }),
    decrement: (state, payload: number = 1) => ({ count: state.count - payload }),
  },
  effects: {
    async asyncIncrement(payload: number = 1) {
      await new Promise(r => setTimeout(r, 1000));
      this.increment(payload);
    },
  },
};

// 2. 注册 Model
addModel(counter);

// 3. 初始化 Store
const store = init();

// 4. 使用 dispatch
store.dispatch.counter.increment(5);
console.log(store.getState().counter.count); // 5

// 5. 在 React 组件中使用 Connect
const mapState = (state) => ({ count: state.counter.count });
const mapDispatch = { increment: () => ({ type: 'counter/increment' }) };

@connect(mapState, mapDispatch)
class Counter extends React.Component {
  render() {
    const { count, increment } = this.props;
    return <button onClick={increment}>{count}</button>;
  }
}
```

---

## 核心 API

### `init()`

初始化全局 Store。内部通过 `globalThis.__GUNDAM_PRO_STORE__` 保证全局单例，多次调用安全。

**返回值：** `RematchStore` — 包含 `getState`、`dispatch`、`subscribe` 等标准 Redux Store 方法。

```typescript
const store = init();
```

---

### `addModel(model)`

动态注册 Model 到 Store。必须在 `init()` 之前或之后均可调用。

| 参数 | 类型 | 说明 |
|------|------|------|
| `model` | `Model` | Rematch Model 定义对象 |

```typescript
addModel({
  name: 'user',
  state: { name: '' },
  reducers: {
    setName: (state, payload: string) => ({ name: payload }),
  },
});
```

---

### `connect(mapStateToProps?, mapDispatchToProps?)`

自定义 React HOC，将 Store 状态和 dispatch 映射到组件 Props。

- **浅比较优化**：内部 `compareProps` 对 props 进行浅比较，避免不必要的重渲染
- **批量更新**：使用 `setTimeout` 实现 React 批量更新
- **forwardRef 支持**：支持 `forwardRef` 转发 ref

| 参数 | 类型 | 说明 |
|------|------|------|
| `mapStateToProps` | `MapStateToProps` | 将 Store state 映射为 Props |
| `mapDispatchToProps` | `MapDispatchToProps` | 将 dispatch 映射为 Props |

```typescript
// 函数式映射
connect(
  (state) => ({ count: state.counter.count }),
  (dispatch) => ({
    increment: () => dispatch.counter.increment(),
  })
)

// 对象式映射
connect(
  (state) => ({ count: state.counter.count }),
  { increment: () => ({ type: 'counter/increment' }) }
)
```

---

### `reset()`

重置 Store 状态。清除所有已注册的 Model，恢复到初始状态。

```typescript
reset();
```

---

### `createModel(model)`

创建 Model 工厂函数，用于类型安全地定义 Model。

```typescript
const userModel = createModel({
  name: 'user',
  state: { name: '', age: 0 },
  reducers: {
    setName: (state, payload: string) => ({ ...state, name: payload }),
  },
  effects: {
    async fetchUser() {
      const user = await api.getUser();
      this.setName(user.name);
    },
  },
});
```

---

## Selectors（选择器）

基于 `createSelector` 实现的 Memoized Selector，支持高效的状态派生计算。

```typescript
import { createSelector } from 'sailor-gundam-store';

// 创建带缓存的 Selector
const getCount = (state) => state.counter.count;
const getDoubled = createSelector([getCount], (count) => count * 2);

// 使用
const state = store.getState();
console.log(getDoubled(state)); // 缓存计算结果
```

### `createSelector(deps, resultFn)`

| 参数 | 类型 | 说明 |
|------|------|------|
| `deps` | `Selector[]` | 依赖的选择器数组 |
| `resultFn` | `(...results) => R` | 结果计算函数 |

所有依赖和结果函数均通过 `memoize()` 进行缓存，仅当输入引用变化时重新计算。

---

## 模块详解

### `src/index.ts` — 包入口

全局单例通过 `globalThis.__GUNDAM_PRO_STORE__` 实现，确保同一页面中无论加载多少次该包，Store 实例始终唯一。

导出：`init`、`connect`、`addModel`、`reset`、`createModel`

---

### `src/typings.ts` — 类型定义（~1080 行）

完整的 TypeScript 类型系统，包含以下核心类型：

| 类型 | 说明 |
|------|------|
| `Action` | Redux Action 标准类型 |
| `Reducer` | Reducer 函数类型 |
| `Model` | Rematch Model 完整定义类型 |
| `Models` | 多个 Model 的集合类型 |
| `RematchStore` | Rematch Store 实例类型 |
| `RematchDispatch` | Rematch Dispatch 类型（支持 model/effect 路径调用） |
| `RematchRootState` | 全局 State 类型 |
| `Selector` | 选择器函数类型 |
| `SelectorCreator` | createSelector 工厂类型 |
| `ModelCreator` | createModel 工厂类型 |
| `Connect` | React Connect HOC 类型 |
| `MapStateToProps` | state → props 映射函数类型 |
| `MapDispatchToProps` | dispatch → props 映射类型 |

同时对 `redux` 和 `react-redux` 模块进行了类型增强（Module Augmentation），确保与 Rematch 类型的无缝集成。

---

### `src/rematch.ts` — 核心 Store 创建

`createRematchStore(name)` 是核心工厂函数，返回 `{ store, addModel, init, connect, reset }`。

**关键实现细节：**

- **`createDispatcher`**：为每个 Model 动态创建 dispatcher，使 `dispatch.modelName.action()` 语法可用
- **`createDispatcherModel`**：将 Model 的 effects 绑定到 `dispatch[modelName]` 上下文，使 `this.action()` 在 effect 中可用
- **`createStore`**：实际创建 Rematch Store 实例
- **Selector 支持**：内置 `createSelector` 集成，支持 memoized 计算属性
- **Effect this-binding**：Effect 函数中的 `this` 被绑定到 `dispatch[modelName]`，可直接调用同 Model 的其他 action

```typescript
// Effect 中 this 绑定示例
const model = {
  name: 'cart',
  state: { items: [] },
  reducers: {
    addItem: (state, item) => ({ items: [...state.items, item] }),
  },
  effects: {
    async addToCart(item) {
      // this 指向 dispatch.cart
      this.addItem(item);  // 等价于 dispatch.cart.addItem(item)
    },
  },
};
```

---

### `src/store.ts` — 轻量 Flux 实现

`createFlux()` 提供不依赖 Rematch 的轻量 Flux Store 实现（**未直接导出**，仅供内部使用）。

**API：**

| 方法 | 说明 |
|------|------|
| `getState()` | 获取当前状态 |
| `dispatch(action)` | 分发 action（验证 action.type 必须包含 `/`） |
| `subscribe(listener)` | 订阅状态变化 |
| `addModels(models)` | 批量注册 Model |

action type 格式要求：`modelName/reducerName`，例如 `counter/increment`。

---

### `src/connect.ts` — React Connect HOC

`createConnect(rematchStore)` 创建自定义的 React Connect 高阶组件。

**实现特点：**

1. **`compareProps`**：浅比较优化，逐 key 对比新旧 props，避免不必要的组件重渲染
2. **批量更新**：使用 `setTimeout` 将 state 更新推迟到下一个事件循环，实现 React 批量更新（兼容旧版 React）
3. **forwardRef**：支持 ref 转发，通过 `forwardRef` 参数获取组件实例

```typescript
@connect(null, null, { forwardRef: true })
class MyComponent extends React.Component {
  myMethod() { /* ... */ }
}

// 父组件可通过 ref 调用 myMethod
const ref = React.createRef();
<MyComponent ref={ref} />;
ref.current.myMethod();
```

---

### `src/selector.ts` — Memoized Selector

提供高效的缓存计算机制，避免重复计算。

#### `equals(a, b)`

严格相等比较（`===`）。

#### `memoize(fn)`

为函数添加缓存，仅当参数变化时重新执行。

```typescript
const expensive = memoize((a, b) => {
  console.log('computing...');
  return a + b;
});

expensive(1, 2); // "computing..." → 3
expensive(1, 2); // (缓存命中) → 3
expensive(3, 4); // "computing..." → 7
```

#### `createSelector(depSelectors, resultFn)`

创建组合 Selector，所有依赖和结果函数均被 memoize。

```typescript
const getX = (state) => state.x;
const getY = (state) => state.y;
const getSum = createSelector([getX, getY], (x, y) => x + y);
```

---

### `src/utils.ts` — 工具函数

#### `type(value)`

获取值的类型字符串（类似 `Object.prototype.toString` 的简化版）。

```typescript
type([]);        // 'Array'
type({});        // 'Object'
type(null);      // 'Null'
type(1);         // 'Number'
type('hello');   // 'String'
```

#### `deepClone(value)`

深拷贝对象/数组，递归复制所有嵌套属性。

```typescript
const original = { a: { b: [1, 2, 3] } };
const cloned = deepClone(original);
cloned.a.b.push(4);
console.log(original.a.b); // [1, 2, 3] — 不受影响
```

---

## 架构概览

```
sailor-gundam-store
├── src/
│   ├── index.ts          # 包入口，全局单例管理，导出公共 API
│   ├── typings.ts        # 完整 TypeScript 类型定义（~1080 行）
│   ├── rematch.ts        # 核心：createRematchStore 工厂函数
│   ├── store.ts          # 备用：createFlux 轻量 Flux 实现
│   ├── connect.ts        # React Connect HOC 实现
│   ├── selector.ts       # Memoized Selector（equals/memoize/createSelector）
│   └── utils.ts          # 工具函数（type/deepClone）
```

---

## 与 Rematch v2 的关系

本包基于 Rematch v2 核心构建，但做了以下增强：

1. **全局单例**：通过 `globalThis.__GUNDAM_PRO_STORE__` 确保多实例场景下的 Store 唯一性
2. **自定义 Connect**：替代 `react-redux` 的 `connect`，针对 Gundam 场景优化批量更新和浅比较
3. **内置 Selector**：集成 `createSelector`，无需额外引入 `reselect`
4. **Effect this-binding**：自动将 effect 中的 `this` 绑定到 `dispatch[modelName]`，简化 effect 内部调用

---

## 注意事项

- `init()` 在应用生命周期中只需调用一次，重复调用返回同一 Store 实例
- `addModel()` 支持动态注册，但建议在 `init()` 之前完成所有 Model 注册
- `reset()` 会清除所有 Model 和状态，通常在页面卸载或场景切换时使用
- `src/store.ts` 中的 `createFlux()` 未对外导出，仅作为内部备用实现
