# sailor-gundam-utils

> v0.150.0 · Base utility functions for Gundam H5 activity pages

`sailor-gundam-utils` 为 Gundam H5 活动页提供基础设施工具库，涵盖 URL 解析与跳转、CSS 类名构建、调试日志、版本比较、深度比较、节流、JSON 安全解析、单位转换、React Hooks 等常用功能。

---

## 安装

```bash
npm install sailor-gundam-utils
# 或
yarn add sailor-gundam-utils
```

---

## 快速开始

```typescript
import { qs, getJumpUrl, classNames, DebugLog, compareVersions } from 'sailor-gundam-utils';

// 解析 URL 参数
const params = qs(); // { key: 'value', ... }

// 获取跳转 URL（自动处理 UTM/swimlane/biz 参数透传）
const url = getJumpUrl('https://example.com/page');

// CSS 类名拼接
const cls = classNames('btn', { active: true, disabled: false }, ['large']); // "btn active large"

// 调试日志
const log = DebugLog('MyModule');
log('something happened'); // 仅 isLocal 或 isDebug 时输出

// 版本比较
compareVersions('2.0.0', '1.9.9'); // 1
compareVersions.validate('1.2.3');  // true
```

---

## 核心 API

### URL 与跳转

#### `qs(link?, splitter?)`

解析 URL 查询参数。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `link` | `string` | `location.href` | 要解析的 URL |
| `splitter` | `string` | `'&'` | 参数分隔符 |

**返回值：** `Record<string, string>`

在 Native 环境下返回 `Mach.env.schemeParams`。

```typescript
qs('https://example.com?foo=bar&baz=qux');
// { foo: 'bar', baz: 'qux' }

qs(); // 解析当前页面 URL
```

---

#### `getJumpUrl(link)`

获取跳转 URL，自动透传 UTM、swimlane、biz 等参数。

| 参数 | 类型 | 说明 |
|------|------|------|
| `link` | `string` | 目标链接 |

**返回值：** `string`

```typescript
getJumpUrl('https://example.com/page');
// 若当前页面带有 UTM/swimlane/biz 参数，将自动附加到目标 URL
```

---

#### `addParams(url, params)`

向 URL 添加查询参数。

| 参数 | 类型 | 说明 |
|------|------|------|
| `url` | `string` | 基础 URL |
| `params` | `Record<string, string>` | 要添加的参数 |

**返回值：** `string`

---

#### `isH5Url(url)`

判断 URL 是否为 H5 页面链接。

| 参数 | 类型 | 说明 |
|------|------|------|
| `url` | `string` | 待检测 URL |

**返回值：** `boolean`

---

#### `urlParams(url, customParams?)`

从 URL 提取并合并参数，支持自定义参数覆盖。

| 参数 | 类型 | 说明 |
|------|------|------|
| `url` | `string` | 源 URL |
| `customParams` | `Record<string, string>` | 自定义参数（优先级更高） |

**返回值：** `string` — 合并后的完整 URL

---

#### `addParamsToUrl(url, params)`

`addParams` 的别名，以 `addParamsToUrl` 名称重新导出。

---

### querystring-es6

#### `encode(obj, sep?, eq?, name?)`

将对象编码为查询字符串。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `obj` | `object` | — | 要编码的对象 |
| `sep` | `string` | `'&'` | 参数分隔符 |
| `eq` | `string` | `'='` | 键值分隔符 |
| `name` | `string` | — | 嵌套时的键名前缀 |

```typescript
encode({ foo: 'bar', baz: 'qux' }); // 'foo=bar&baz=qux'
```

#### `decode(qs, sep?, eq?, options?)`

将查询字符串解码为对象。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `qs` | `string` | — | 查询字符串 |
| `sep` | `string` | `'&'` | 参数分隔符 |
| `eq` | `string` | `'='` | 键值分隔符 |
| `options.maxKeys` | `number` | `1000` | 最大解析键数 |

```typescript
decode('foo=bar&baz=qux'); // { foo: 'bar', baz: 'qux' }
```

#### `parse(qs, sep?, eq?, options?)`

`decode` 的别名。

#### `stringify(obj, sep?, eq?, name?)`

`encode` 的别名。

---

### 枚举常量

#### `searchScheme`

搜索 Scheme URL 常量：

| 常量 | 值 | 说明 |
|------|------|------|
| `searchScheme.WM` | 对应 URL | WM 搜索 Scheme |
| `searchScheme.MT` | 对应 URL | MT 搜索 Scheme |
| `searchScheme.DP` | 对应 URL | DP 搜索 Scheme |

#### `UTM`

UTM 参数名数组，用于 URL 参数透传时自动识别和携带 UTM 相关参数。

#### `WP_REQUIRED`

需要 WP（Webview Parameters）的参数名数组。

#### `BIZ_REQUIRED`

需要 BIZ（业务参数）的参数名数组。

#### `SWIMLANE_REQUIRED`

需要泳道参数的参数名数组。

---

### CSS 类名

#### `classNames(...args)`

动态构建 CSS 类名字符串。

支持以下类型的参数：

| 类型 | 说明 | 示例 |
|------|------|------|
| `string` | 直接作为类名 | `'btn'` → `'btn'` |
| `number` | 转为字符串 | `42` → `'42'` |
| `Array` | 递归展开 | `['a', 'b']` → `'a b'` |
| `Object` | 值为 truthy 时取键名 | `{ active: true, disabled: false }` → `'active'` |
| `null/undefined` | 忽略 | — |

```typescript
classNames('btn', 'primary');                          // 'btn primary'
classNames('btn', { active: isActive, large: isLarge }); // 按条件拼接
classNames('btn', ['icon', 'small'], { loading });     // 混合使用
```

---

### 调试

#### `DebugLog(module)`

创建条件日志函数，仅在本地或调试模式下输出日志。

| 参数 | 类型 | 说明 |
|------|------|------|
| `module` | `string` | 模块名称，用于日志前缀 |

**返回值：** `(message: string, ...args: any[]) => void`

**触发条件：** `Mach.env.isLocal === true` 或 `Mach.env.isDebug === true`

```typescript
const log = DebugLog('PaymentModule');
log('order created', { orderId: '123' });
// [PaymentModule] order created { orderId: '123' }
```

---

### 版本比较

#### `compareVersions(v1, v2)`

语义化版本比较。

| 参数 | 类型 | 说明 |
|------|------|------|
| `v1` | `string` | 版本号 1 |
| `v2` | `string` | 版本号 2 |

**返回值：** `-1 | 0 | 1`

```typescript
compareVersions('2.0.0', '1.9.9');  // 1  (v1 > v2)
compareVersions('1.0.0', '1.0.0');  // 0  (v1 === v2)
compareVersions('1.0.0', '2.0.0');  // -1 (v1 < v2)
```

#### `compareVersions.validate(version)`

验证版本号格式是否合法。

```typescript
compareVersions.validate('1.2.3');    // true
compareVersions.validate('invalid');  // false
```

#### `compareVersions.compare(v1, v2, operator)`

使用操作符比较版本号。

| 参数 | 类型 | 说明 |
|------|------|------|
| `operator` | `string` | `>`, `>=`, `<`, `<=`, `=`, `==`, `===` |

```typescript
compareVersions.compare('2.0.0', '1.0.0', '>');   // true
compareVersions.compare('1.0.0', '2.0.0', '>=');  // false
```

---

### 深度比较

#### `deepEqual(x, y)`

深度比较两个值是否相等，支持对象、数组、函数。

| 参数 | 类型 | 说明 |
|------|------|------|
| `x` | `any` | 值 1 |
| `y` | `any` | 值 2 |

**返回值：** `boolean`

```typescript
deepEqual({ a: [1, 2] }, { a: [1, 2] }); // true
deepEqual([1, 2, 3], [1, 2, 4]);         // false
deepEqual(() => {}, () => {});           // false（函数引用比较）
```

---

### 节流

#### `throttle(fn, wait?)`

节流函数，限制调用频率。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `fn` | `Function` | — | 要节流的函数 |
| `wait` | `number` | `100` | 节流间隔（ms） |

**返回值：** `Function` — 节流后的函数

```typescript
const throttledScroll = throttle(handleScroll, 200);
window.addEventListener('scroll', throttledScroll);
```

---

### 环境检测

#### `isH5()`

运行时 H5 环境检测。

**返回值：** `boolean` — 当前是否运行在 H5 环境中

```typescript
if (isH5()) {
  // H5 特有逻辑
}
```

---

### URL 验证

#### `isValidUrl(str)`

验证字符串是否为合法 URL。

| 参数 | 类型 | 说明 |
|------|------|------|
| `str` | `string` | 待验证字符串 |

**返回值：** `boolean`

```typescript
isValidUrl('https://example.com');   // true
isValidUrl('not a url');             // false
```

---

### JSON 安全解析

#### `jsonSafeParse<T>(value?)`

安全解析 JSON 字符串，解析失败时返回默认值。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `value` | `string` | — | JSON 字符串 |

**返回值：** `T | undefined` — 解析结果，失败时返回 `undefined`

```typescript
jsonSafeParse<{ name: string }>('{"name":"test"}'); // { name: 'test' }
jsonSafeParse('invalid json');                      // undefined
jsonSafeParse('{}');                                // {}
```

---

### 单位转换

#### `dp2Rpx(dp)`

将 dp 单位转换为 rpx 单位，基于 `Mach.env.screenWidth` 计算。

| 参数 | 类型 | 说明 |
|------|------|------|
| `dp` | `number` | dp 值 |

**返回值：** `number` — rpx 值

```typescript
dp2Rpx(100); // 根据 screenWidth 换算
```

---

### 字符串辅助

#### `padStart(num, targetLength, padChar?)`

字符串前补零/补位。

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `num` | `number` | — | 要补位的数字 |
| `targetLength` | `number` | — | 目标长度 |
| `padChar` | `string` | `'0'` | 补位字符 |

**返回值：** `string`

```typescript
padStart(5, 3);     // '005'
padStart(42, 4);    // '0042'
padStart(9, 2, 'x'); // 'x9'
```

---

## React Hooks

### `useRequest(requestFn, options?)`

异步请求 Hook，管理 loading/data/err 状态。

| 参数 | 类型 | 说明 |
|------|------|------|
| `requestFn` | `(...args) => Promise<T>` | 异步请求函数 |
| `options` | `object` | 配置项 |
| `options.manual` | `boolean` | 是否手动触发（默认自动） |
| `options.defaultParams` | `any[]` | 默认参数 |
| `options.onSuccess` | `(data: T) => void` | 成功回调 |
| `options.onError` | `(err: Error) => void` | 失败回调 |

**返回值：**

| 属性 | 类型 | 说明 |
|------|------|------|
| `loading` | `boolean` | 是否加载中 |
| `data` | `T \| undefined` | 响应数据 |
| `err` | `Error \| undefined` | 错误信息 |
| `run` | `(...args) => Promise<T>` | 手动触发请求 |
| `refresh` | `() => Promise<T>` | 使用上次参数重新请求 |

```typescript
function UserList() {
  const { loading, data, err, run } = useRequest(fetchUsers, {
    manual: false,
    onSuccess: (data) => console.log('loaded', data),
  });

  if (loading) return <div>Loading...</div>;
  if (err) return <div>Error: {err.message}</div>;
  return <ul>{data?.map(user => <li key={user.id}>{user.name}</li>)}</ul>;
}
```

---

### `useAsyncEffect`

支持 async 回调的 `useEffect`。

```typescript
useAsyncEffect(async () => {
  const data = await fetchData();
  setData(data);
}, [dependency]);
```

### `useAsyncLayoutEffect`

支持 async 回调的 `useLayoutEffect`。

```typescript
useAsyncLayoutEffect(async () => {
  const layout = await measureLayout();
  setDimensions(layout);
}, [elementRef]);
```

---

## 类型声明

### `src/types.d.ts` — 全局类型

为 `Mach` 运行时环境提供全局类型声明：

| 类型 | 说明 |
|------|------|
| `Mach` | Mach 运行时对象，包含 env、performance 等 |
| `Mach.env` | 环境变量（isLocal、isDebug、screenWidth、schemeParams 等） |
| `MachPerformance` | 性能监测接口 |
| `Env` | 环境配置类型 |
| `Window` | 扩展 Window 接口，挂载 Mach 全局变量 |

---

## 模块详解

### `src/qs.ts`

URL 查询参数解析。在 Native 环境下直接返回 `Mach.env.schemeParams`，在 H5 环境下解析 `location.href`。

### `src/get-jump-url.ts`

URL 跳转处理核心模块：

- **`addParams(url, params)`**：向 URL 附加参数
- **`isH5Url(url)`**：判断是否为 H5 URL
- **`getJumpUrl(link)`**：获取跳转 URL，自动透传 UTM/swimlane/biz 参数
- **`urlParams(url, customParams?)`**：提取并合并 URL 参数

UTM 参数、泳道参数、业务参数的透传逻辑基于 `enums/redirect-params` 中定义的常量数组。

### `src/_querystring.ts`

重新导出 `querystring-es6` 模块。

### `src/querystring-es6/`

完整的 querystring 编解码实现，兼容 Node.js `querystring` 模块 API：

- `encode.ts`：对象 → 查询字符串
- `decode.ts`：查询字符串 → 对象
- `index.ts`：统一导出 `encode`、`decode`、`parse`（decode 别名）、`stringify`（encode 别名）

### `src/enums/`

枚举常量定义：

- **`searchScheme`**：WM/MT/DP 搜索 Scheme URL
- **`redirect-params/`**：UTM、WP_REQUIRED、BIZ_REQUIRED、SWIMLANE_REQUIRED 参数名数组

### `src/add-params-to-url.ts`

将 `addParams` 以 `addParamsToUrl` 名称重新导出，提供更语义化的 API 名称。

### `src/classnames.ts`

CSS 类名构建器，支持 string/number/array/object 类型的参数，灵活组合类名。

### `src/debug.ts`

`DebugLog(module)` 工厂函数，创建仅在本地或调试模式下输出的日志函数。适合开发调试，生产环境零开销。

### `src/compare-versions.ts`

语义化版本比较工具。支持标准比较（`-1/0/1`）、格式验证和操作符比较。

### `src/deep-equal.ts`

递归深度比较。支持原始类型、对象、数组的深度比较。函数按引用比较。

### `src/throttle.ts`

节流函数，默认 100ms 间隔。首次调用立即执行，后续在间隔时间内仅执行最后一次。

### `src/is-h5.ts`

H5 环境运行时检测，基于 `Mach` 全局对象判断。

### `src/is-valid-url.ts`

URL 格式合法性验证。

### `src/json.ts`

`jsonSafeParse<T>` 安全 JSON 解析，解析失败不抛异常，返回 `undefined`。

### `src/dp-to-rpx.ts`

dp 到 rpx 的单位转换，基于 `Mach.env.screenWidth` 按比例计算。

### `src/string-helpers.ts`

`padStart` 字符串补位工具。

### `src/hooks/`

React Hooks 集合：

- **`use-request.ts`**：`useRequest` 异步请求 Hook
- **`use-async-effect.ts`**：`useAsyncEffect` 和 `useAsyncLayoutEffect` 异步 Effect Hook

### `src/scroll-into-view.ts`

空文件（预留接口）。

---

## 架构概览

```
sailor-gundam-utils
├── src/
│   ├── index.ts                  # Barrel 导出
│   ├── qs.ts                     # URL 查询参数解析
│   ├── get-jump-url.ts           # URL 跳转处理
│   ├── _querystring.ts           # querystring-es6 重导出
│   ├── querystring-es6/          # querystring 编解码实现
│   │   ├── index.ts              # 统一导出
│   │   ├── encode.ts             # 对象 → 查询字符串
│   │   └── decode.ts             # 查询字符串 → 对象
│   ├── enums/                    # 枚举常量
│   │   ├── index.ts              # searchScheme
│   │   └── redirect-params/      # UTM/WP/BIZ/SWIMLANE 参数
│   │       └── index.ts
│   ├── add-params-to-url.ts      # addParams 别名导出
│   ├── classnames.ts             # CSS 类名构建器
│   ├── debug.ts                  # 条件调试日志
│   ├── compare-versions.ts       # 版本比较
│   ├── deep-equal.ts             # 深度比较
│   ├── throttle.ts               # 节流
│   ├── is-h5.ts                  # H5 环境检测
│   ├── is-valid-url.ts           # URL 验证
│   ├── json.ts                   # 安全 JSON 解析
│   ├── dp-to-rpx.ts              # 单位转换
│   ├── string-helpers.ts         # 字符串补位
│   ├── hooks/                    # React Hooks
│   │   ├── index.ts              # Hooks barrel 导出
│   │   ├── use-request.ts        # 异步请求 Hook
│   │   └── use-async-effect.ts   # 异步 Effect Hook
│   ├── types.d.ts                # 全局类型声明
│   └── scroll-into-view.ts       # （空文件）
```

---

## 注意事项

- `DebugLog` 在生产环境不输出日志（`isLocal` 和 `isDebug` 均为 `false`）
- `jsonSafeParse` 解析失败返回 `undefined`，不会抛出异常
- `dp2Rpx` 依赖 `Mach.env.screenWidth`，需在 Mach 运行时环境中使用
- `qs()` 在 Native 环境下直接读取 `Mach.env.schemeParams`，不解析 URL
- `getJumpUrl` 会自动透传当前页面的 UTM、泳道、业务参数到目标 URL
