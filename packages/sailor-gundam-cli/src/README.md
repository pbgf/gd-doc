# src/ — 源码目录

Gundam CLI 的全部源代码，基于 yargs 命令行框架实现 6 个子命令。

---

## 文件结构

```
src/
├── index.ts                        # 主入口
├── commands/                       # 命令定义与处理
│   ├── index.ts                    # 桶导出
│   ├── init/index.ts               # init 命令
│   ├── dev/index.ts                # dev 命令定义
│   ├── dev/handler.ts              # dev 命令处理器
│   ├── build/index.ts              # build 命令定义
│   ├── build/handler.ts            # build 命令处理器
│   ├── publish/index.ts            # publish 命令定义
│   ├── publish/handler.ts          # publish 命令处理器
│   ├── dsl/index.ts                # dsl 命令
│   └── i18n/index.ts               # i18nSync 命令
└── utils/                          # 工具函数
    ├── index.ts                    # 通用工具
    ├── logger.ts                   # 日志类
    └── generate-tsconfig.ts        # 配置文件生成
```

---

## 文件详解

### `index.ts` — 主入口

**功能**: yargs 引导程序，注册所有子命令并启动 CLI。

**导出**:

| 导出 | 类型 | 说明 |
|------|------|------|
| `default()` | `() => void` | 默认导出，初始化 yargs 并注册 6 个命令 |
| `handleBuild` | `function` | 构建命令的处理函数，可独立调用 |

**注册的命令**: `init`, `dev`, `build`, `publish`, `i18nSync`, `dsl`

**实现细节**:
- 使用 yargs 的 `command()` 方法链式注册所有命令
- 设置全局选项（如 `--version`, `--help`）
- `handleBuild` 被单独导出，供构建服务器（gundam-build-server）远程调用

---

### `commands/index.ts` — 桶导出

**功能**: 集中导出所有命令模块。

**导出内容**:

```typescript
export { default as initCommand } from './init'
export { default as devCommand } from './dev'
export { default as buildCommand } from './build'
export { default as publishCommand } from './publish'
export { default as i18nCommand } from './i18n'
export { default as dslCommand } from './dsl'
```

**设计意图**: 作为命令注册的唯一入口，`index.ts` 只需从此文件导入所有命令，避免在主入口中维护多个导入路径。

---

### `commands/init/index.ts` — gundam init 命令

**功能**: 交互式创建新的 Gundam 组件项目。

**命令定义**:
- 命令名: `init [appName]`
- 别名: `i`
- 描述: 创建新项目

**交互问题（9 个）**:

| # | 问题 | 选项 | 影响 |
|---|------|------|------|
| 1 | 项目类型 | `page` / `component` | 决定项目模板 |
| 2 | 组件风格 | `class` / `function` | 选择类组件或函数组件模板 |
| 3 | 是否供给组件 | `yes` / `no` | 选择 supply 变体模板 |
| 4 | 国际化支持 | `yes` / `no` | 添加 i18n 配置 |
| 5 | 数据提取 | `yes` / `no` | 添加数据提取逻辑 |
| 6 | 投放功能 | `yes` / `no` | 添加 launch 相关代码 |
| 7 | 接口聚合方式 | `aggregated` / `DSL` / `无` | 决定 API 集成方式 |
| 8 | 项目描述 | 自由文本 | 写入 package.json |
| 9 | 作者 | 自由文本 | 写入 package.json |

**处理流程**:
1. 收集用户回答
2. 构造配置对象
3. 委托 `@sailorgundam/gundam-generator` 执行脚手架生成
4. gundam-generator 负责模板复制、Mustache 变量替换、条件文件处理、依赖安装

**依赖**: `inquirer`（交互式问答）、`@sailorgundam/gundam-generator`

---

### `commands/dev/index.ts` — gundam dev 命令定义

**功能**: 定义 `dev` 命令的 yargs 配置。

**命令定义**:
- 命令名: `dev`
- 别名: `D`
- 描述: 启动开发服务器

**选项**:

| 选项 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `--type` | `string` | 自动推断 | 开发模式：`h5` 或 `machpro` |
| `--env` | `string` | — | 注入环境变量 |

**`--type` 自动推断逻辑**: 读取项目根目录的 `mach.config.js`，若存在则默认 `machpro`，否则默认 `h5`。

---

### `commands/dev/handler.ts` — dev 命令处理器

**功能**: 根据项目类型启动不同的开发服务器。

**导出**: `devHandler(argv: yargs.Arguments)`

**H5 模式流程**:
1. 调用 `generate-tsconfig()` 生成/更新项目配置
2. 启动 Vite dev server
3. 配置 Preact 别名（`react` → `preact/compat`）
4. 自动打开浏览器

**MachPro 模式流程**:
1. 读取 `mach.config.js` 获取 MachPro 配置
2. 通过 `execShell()` 启动 `mach-pro dev` 进程（使用 node-pty 伪终端）
3. 监听标准输出，自动检测 QR 码
4. QR 码检测到后，自动在 iOS 模拟器中打开
5. 实时输出日志到终端

**关键实现细节**:
- `node-pty` 用于创建伪终端，支持交互式进程（MachPro dev 需要用户输入）
- QR 码检测通过正则匹配终端输出中的二维码模式实现
- iOS 模拟器通过 `xcrun simctl` 命令控制

**依赖**: `vite`、`node-pty`、`@sailorgundam/gundam-core`（间接）

---

### `commands/build/index.ts` — gundam build 命令定义

**功能**: 定义 `build` 命令的 yargs 配置。

**命令定义**:
- 命令名: `build`
- 别名: `B`
- 描述: 构建项目

**选项**: 无额外选项，项目类型从 `mach.config.js` 自动推断。

---

### `commands/build/handler.ts` — build 命令处理器

**功能**: 根据项目类型执行不同的构建流程。

**导出**: `buildHandler(argv: yargs.Arguments)`

**Native 模式构建流程**:

```
1. 调用 generate-tsconfig() 生成编译配置
2. 执行 tsc 编译
   ├── 输出 ESM 格式 (dist/esm/)
   └── 输出 CJS 格式 (dist/cjs/)
3. 执行 copyfiles 复制静态资源
   ├── *.less 样式文件
   ├── *.png/*.jpg 图片资源
   └── 其他非 TS 文件
4. 生成 gdc-description.json
   ├── 读取 src/desc.ts 中的组件描述
   ├── 包含组件名、属性、事件等信息
   └── 供可视化编辑器和构建平台使用
5. 输出到 dist/ 目录
```

**H5 模式构建流程**:

```
1. 调用 generate-tsconfig() 生成编译配置
2. 委托 @sailorgundam/gundam-h5-build 执行构建
   ├── Rollup 构建引擎
   ├── 多态文件解析
   ├── CSS 处理（PostCSS + px 转换）
   ├── 资源处理（CDN 上传 / base64 内联）
   └── 代码注入（MACH_PROJECT_CONFIG）
3. 输出 UMD bundle
```

**gdc-description.json 说明**:
- Native 组件的元数据描述文件
- 描述组件的属性、事件、默认值等
- 构建平台和可视化编辑器依赖此文件来识别组件

**依赖**: `typescript`、`copyfiles`、`@sailorgundam/gundam-h5-build`

---

### `commands/publish/index.ts` — gundam publish 命令定义

**功能**: 定义 `publish` 命令的 yargs 配置。

**命令定义**:
- 命令名: `publish`
- 别名: `p`
- 描述: 发布项目到 mnpm 仓库

**选项**: 无额外选项。

---

### `commands/publish/handler.ts` — publish 命令处理器

**功能**: 执行组件发布到 mnpm 仓库的完整流程。

**导出**: `publishHandler(argv: yargs.Arguments)`

**发布流程**:

```
1. mnpm unpublish <pkg>@<version>
   └── 移除同版本旧包（如果存在）
2. sleep(等待时间)
   └── 等待 mnpm 仓库同步删除操作
3. npm pack
   └── 将 dist/ 目录打包为 .tgz
4. mnpm publish
   └── 发布 .tgz 到 mnpm 仓库
```

**关键实现细节**:
- unpublish + sleep 的设计是为了处理 mnpm 仓库的最终一致性延迟
- 如果同版本已存在，直接发布会失败，因此先 unpublish 再重新发布
- sleep 时间通常为几秒，确保仓库同步完成

**依赖**: `mnpm`（美团内部 npm 客户端）

---

### `commands/dsl/index.ts` — gundam dsl 命令

**功能**: 初始化 DSL 配置文件，从 js-to-groovy 模板生成 `src/dsl.js`。

**命令定义**:
- 命令名: `dsl`
- 别名: `d`
- 描述: 初始化 DSL 配置

**处理流程**:
1. 读取 `@sailorgundam/gundam-js-to-groovy` 包中的模板
2. 复制模板到项目的 `src/dsl.js`
3. 开发者可在 `dsl.js` 中编写 JavaScript DSL 逻辑
4. 构建时 js-to-groovy 会将 JS DSL 转换为服务端可执行的 Groovy 脚本

**JS-to-Groovy 转换示例**:
- `===` → `==`
- `||` → `?:` (Elvis)
- 箭头函数 → Groovy 闭包
- `.map()` → `.collect()`

---

### `commands/i18n/index.ts` — gundam i18nSync 命令

**功能**: 同步国际化翻译资源。

**命令定义**:
- 命令名: `i18nSync`
- 别名: `i18n`
- 描述: 同步国际化资源

**处理流程**:
1. 定位 i18n sync 二进制工具
2. 执行同步命令
3. 从翻译平台拉取最新翻译内容
4. 更新项目本地 i18n 资源文件

**依赖**: i18n sync 二进制工具

---

### `utils/index.ts` — 通用工具函数

**功能**: 提供 CLI 通用工具函数。

**导出**:

| 函数 | 签名 | 说明 |
|------|------|------|
| `execShell` | `(command: string, args?: string[], options?: ExecOptions) => Promise<void>` | 基于 node-pty 的 Shell 命令执行器，支持交互式进程 |
| `toHump` | `(name: string) => string` | 将下划线命名转换为驼峰命名（如 `my_component` → `myComponent`） |
| `getProjectTypeFromArgv` | `(argv: yargs.Arguments) => 'h5' \| 'machpro'` | 从命令行参数或项目配置推断项目类型 |
| `readMachConfig` | `(cwd?: string) => MachConfig \| null` | 读取 `mach.config.js` 配置文件 |
| `copyWriteFile` | `(src: string, dest: string, replacer?: (content: string) => string) => void` | 复制文件并可选地修改内容 |

**`execShell` 实现细节**:
- 使用 `node-pty` 创建伪终端
- 支持交互式命令（如 `mach-pro dev` 需要用户输入）
- 实时输出 stdout/stderr
- 返回 Promise，命令完成后 resolve

**`getProjectTypeFromArgv` 推断逻辑**:
1. 如果 `argv.type` 明确指定，直接使用
2. 否则读取 `mach.config.js`，存在则返回 `machpro`
3. 不存在则返回 `h5`

---

### `utils/logger.ts` — Logger 类

**功能**: 提供带前缀栈的日志输出工具。

**导出**: `Logger` 类

**类签名**:

```typescript
class Logger {
  private prefixStack: string[]
  
  constructor(prefix?: string)
  
  push(prefix: string): void    // 压入前缀
  pop(): void                    // 弹出前缀
  log(message: string): void     // 带前缀的日志输出
  warn(message: string): void    // 带前缀的警告输出
  error(message: string): void   // 带前缀的错误输出
  success(message: string): void // 带前缀的成功输出
}
```

**实现细节**:
- 前缀栈支持嵌套（如 `[gundam][build] Compiling...`）
- 不同日志级别使用不同颜色
- `success` 使用绿色，`warn` 使用黄色，`error` 使用红色

---

### `utils/generate-tsconfig.ts` — 配置文件生成器

**功能**: 为组件项目生成 TypeScript 和构建配置文件。

**导出**: `generateTsconfig(cwd?: string): void`

**生成的文件**:

| 文件 | 说明 |
|------|------|
| `tsconfig.json` | TypeScript 编译配置，设置 JSX、模块解析、路径别名等 |
| `vite.config.js` | Vite 开发服务器配置，包含 Preact 别名和代理设置 |
| `.babelrc.js` | Babel 配置，支持 JSX 转换和装饰器 |

**tsconfig.json 关键配置**:
```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "ESNext",
    "moduleResolution": "node",
    "jsx": "react-jsx",
    "esModuleInterop": true,
    "paths": {
      "@/*": ["src/*"]
    }
  }
}
```

**vite.config.js 关键配置**:
- `resolve.alias`: `react` → `preact/compat`，`react-dom` → `preact/compat`
- `server.proxy`: 代理 API 请求到后端开发服务器

**设计意图**: 组件项目本身不需要维护这些配置文件，CLI 根据项目类型自动生成，确保所有项目使用统一的构建标准。

---

## 命令注册模式

所有命令遵循 yargs `CommandModule` 接口：

```typescript
interface CommandModule {
  command: string        // 命令名和参数定义
  aliases?: string[]     // 别名
  describe: string       // 命令描述
  builder?: Function     // 定义命令选项
  handler: Function      // 命令处理逻辑
}
```

每个命令目录包含两个文件：
- `index.ts` — 命令定义（command、aliases、describe、builder）
- `handler.ts` — 命令处理逻辑（handler）

这种分离设计使得：
1. 命令定义与处理逻辑解耦
2. `handleBuild` 可以被单独导出供构建服务器远程调用
3. 处理逻辑可独立测试

---

## 依赖关系图

```
index.ts
  └── commands/index.ts
        ├── init/index.ts ─────→ gundam-generator
        ├── dev/index.ts ──────→ dev/handler.ts ──→ utils/ (execShell, generate-tsconfig)
        │                                           └── vite, node-pty
        ├── build/index.ts ───→ build/handler.ts ─→ utils/ (generate-tsconfig)
        │                                           └── tsc, copyfiles, gundam-h5-build
        ├── publish/index.ts ─→ publish/handler.ts → mnpm
        ├── dsl/index.ts ────────────────────────→ gundam-js-to-groovy
        └── i18n/index.ts ──────────────────────→ i18n sync binary
```
