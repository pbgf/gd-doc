# @sailorgundam/gundam-cli

> Gundam CLI 工具 — 组件脚手架、开发、构建、发布与管理的命令行工具

**版本**: 0.147.0  
**二进制名**: `gundam`  
**基础框架**: [yargs](https://github.com/yargs/yargs)

---

## 概览

`gundam-cli` 是 Gundam 活动页跨端开发框架的命令行入口，提供了从项目初始化到发布的完整工作流。它支持 H5（浏览器）和 Native（MachPro）两种开发/构建模式，并通过模板系统快速生成标准化的组件项目。

### 核心能力

| 能力 | 说明 |
|------|------|
| 脚手架生成 | 交互式问答创建新组件项目，支持多种组件变体 |
| 本地开发 | H5 模式（Vite 热更新）与 MachPro 模式（node-pty + 模拟器） |
| 项目构建 | Native（tsc ESM+CJS+资源+描述符）与 H5（gundam-h5-build） |
| 包发布 | mnpm 仓库发布，含 unpublish + sleep + pack + publish 流程 |
| DSL 生成 | 从 js-to-groovy 模板生成 `src/dsl.js` |
| 国际化同步 | 调用 i18n sync 二进制同步翻译资源 |

---

## 命令一览

| 命令 | 别名 | 功能 | 详细说明 |
|------|------|------|----------|
| `gundam init [appName]` | `i` | 交互式创建新项目 | 9 个交互问题，委托 gundam-generator 执行 |
| `gundam dev` | `D` | 启动开发服务器 | H5: Vite + Preact；Native: mach-pro dev via node-pty |
| `gundam build` | `B` | 构建项目 | Native: tsc → ESM+CJS+copyfiles+gdc-description.json；H5: gundam-h5-build |
| `gundam publish` | `p` | 发布到 mnpm 仓库 | unpublish → sleep → pack → publish |
| `gundam dsl` | `d` | 初始化 DSL 配置文件 | 生成 `src/dsl.js` |
| `gundam i18nSync` | `i18n` | 同步国际化资源 | 运行 i18n sync 二进制 |

---

## 架构

```
gundam-cli/
├── src/                          # 源码目录
│   ├── index.ts                  # 主入口 — yargs 引导程序
│   ├── commands/                 # 命令定义与处理
│   │   ├── index.ts              # 桶导出（init, dev, build, publish, i18n, dsl）
│   │   ├── init/index.ts         # gundam init 命令
│   │   ├── dev/index.ts          # gundam dev 命令定义
│   │   ├── dev/handler.ts        # dev 命令处理器（H5/MachPro 双模式）
│   │   ├── build/index.ts        # gundam build 命令定义
│   │   ├── build/handler.ts      # build 命令处理器（Native/H5 双模式）
│   │   ├── publish/index.ts      # gundam publish 命令定义
│   │   ├── publish/handler.ts    # publish 命令处理器
│   │   ├── dsl/index.ts          # gundam dsl 命令
│   │   └── i18n/index.ts         # gundam i18nSync 命令
│   └── utils/                    # 工具函数
│       ├── index.ts              # execShell, toHump, getProjectTypeFromArgv 等
│       ├── logger.ts             # Logger 类（带前缀栈）
│       └── generate-tsconfig.ts  # 生成 tsconfig.json + vite.config.js + .babelrc.js
└── templates/                    # 项目模板
    ├── entry/                    # 开发入口模板
    │   ├── main.tsx              # H5 开发入口
    │   ├── h5main.tsx            # 备用 H5 入口
    │   └── index.html            # HTML 外壳
    └── component/                # 组件项目模板
        ├── pkg.json              # package.json 模板（Mustache 变量）
        ├── mach.config.js        # MachPro 配置模板
        ├── src/                  # 组件源码模板
        │   ├── desc.ts           # 组件描述/元数据
        │   ├── index.function.tsx         # 函数组件模板
        │   ├── index.function.supply.tsx  # 函数供给组件模板
        │   ├── index.class.tsx            # 类组件模板
        │   ├── index.class.supply.tsx     # 类供给组件模板
        │   ├── index.less                # BEM 样式模板
        │   ├── model.ts                  # 供给组件数据模型
        │   └── types.d.ts               # 全局类型声明
        └── mock/                 # Mock 数据模板
            ├── globalData.ts     # 模拟后端数据
            ├── i18nResource.ts   # 国际化资源
            ├── index.tsx         # Mock 组合根
            └── pageConfig.ts     # 模拟页面容器配置
```

---

## 命令详解

### `gundam init [appName]`

交互式创建新的 Gundam 组件项目。

**交互问题（9 个）**:

1. 项目类型（page / component）
2. 组件风格（class / function）
3. 是否为供给组件（supply）
4. 是否需要国际化（i18n）
5. 是否需要数据提取
6. 是否需要投放功能（launch）
7. 接口聚合方式（aggregated / DSL / 无）
8. 项目描述
9. 作者信息

**内部流程**: 收集回答后委托 `@sailorgundam/gundam-generator` 执行脚手架生成。

---

### `gundam dev`

启动本地开发服务器，支持两种模式：

| 选项 | 说明 | 默认值 |
|------|------|--------|
| `--type` | 开发模式：`h5` 或 `machpro` | 从 `mach.config.js` 自动推断 |
| `--env` | 环境变量注入 | — |

**H5 模式**:
- 基于 Vite dev server
- Preact 替代 React 以提升热更新速度
- 自动加载 mock 数据

**MachPro 模式**:
- 通过 `node-pty` 启动 `mach-pro dev` 进程
- 自动检测 QR 码并打开 iOS 模拟器
- 支持实时日志输出

---

### `gundam build`

构建组件项目，根据项目类型自动选择构建策略。

**Native 模式构建流程**:
```
tsc 编译 (ESM + CJS)
    → copyfiles 复制静态资源
    → 生成 gdc-description.json（组件描述符）
    → 输出到 dist/
```

**H5 模式构建流程**:
```
委托 @sailorgundam/gundam-h5-build
    → Rollup/Vite 构建
    → 输出 UMD bundle
```

---

### `gundam publish`

发布组件到 mnpm 内部仓库。

**发布流程**:
```
1. mnpm unpublish（移除旧版本）
2. sleep（等待仓库同步）
3. npm pack（打包）
4. mnpm publish（发布新版本）
```

> ⚠️ 发布前请确保已执行 `gundam build` 且构建产物正确。

---

### `gundam dsl`

初始化 DSL 配置文件。基于 `@sailorgundam/gundam-js-to-groovy` 模板，生成 `src/dsl.js` 文件，用于编写可转换为 Groovy 脚本的 JavaScript DSL 逻辑。

---

### `gundam i18nSync`

同步国际化翻译资源。调用 i18n sync 二进制工具，从翻译平台拉取最新的翻译内容到项目本地。

---

## 依赖关系

```
gundam-cli
  ├── @sailorgundam/gundam-generator    # init 命令的脚手架引擎
  ├── @sailorgundam/gundam-h5-build     # H5 构建引擎
  ├── @sailorgundam/gundam-js-to-groovy # DSL 模板和转换
  ├── yargs                              # 命令行框架
  ├── node-pty                           # MachPro 模式的伪终端
  ├── inquirer                           # 交互式问答（init 命令）
  └── vite                               # H5 开发服务器
```

---

## 开发指南

### 本地开发 CLI

```bash
# 克隆仓库
git clone <repo-url>
cd packages/sailor-gundam-cli

# 安装依赖
pnpm install

# 开发模式（监听变更）
pnpm dev

# 构建
pnpm build
```

### 添加新命令

1. 在 `src/commands/` 下创建新目录（如 `src/commands/mycommand/`）
2. 创建 `index.ts` 定义命令（遵循 yargs `CommandModule` 接口）
3. 创建 `handler.ts` 实现命令逻辑
4. 在 `src/commands/index.ts` 中导出新命令
5. 在 `src/index.ts` 中注册新命令

---

## 典型工作流

```
gundam init my-component
    → 创建组件项目目录
    → 生成模板文件
    → 安装依赖

cd my-component
gundam dev --type h5
    → 启动 Vite 开发服务器
    → 浏览器自动打开
    → 热更新开发

gundam build
    → 编译 TypeScript
    → 生成 ESM + CJS 产物
    → 生成 gdc-description.json

gundam publish
    → 发布到 mnpm 仓库
```
