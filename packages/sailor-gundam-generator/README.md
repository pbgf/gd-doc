# @sailorgundam/gundam-generator

> v0.148.0 · Gundam 项目脚手架 / 代码生成器

## 概述

`sailor-gundam-generator` 是高达项目的脚手架工具，负责根据用户选择创建组件工程或页面工程。它使用 Mustache 模板引擎生成项目文件，并自动完成依赖安装、i18n 同步和 Git 钩子配置。

## 目录结构

```
src/
├── index.ts                  # 入口：initGundamProject() 路由到组件/页面处理器
├── handlers/
│   ├── component.ts          # 组件工程处理器
│   └── page.ts               # 页面工程处理器（Stub）
└── utils/
    ├── execute-cmd.ts        # executeCMD() — 基于 spawn 的命令执行器
    ├── ensure-git.ts         # ensureGit() — Git 初始化守卫
    └── logger.ts             # Logger 类 — 带前缀栈的日志输出

templates/
├── component/                # 组件工程模板
│   ├── pkg.json              # package.json 模板
│   ├── mach.config.js        # MachPro 构建配置
│   ├── i18n-mach.config.js   # 国际化构建配置
│   ├── launchTemplate.js     # 启动模板
│   ├── tsconfig.json         # TypeScript 配置
│   ├── vite.config.js        # Vite 构建配置
│   ├── src/                  # 组件源码模板
│   │   ├── desc.ts           # 组件描述
│   │   ├── data_extract.ts   # 数据抽取逻辑
│   │   ├── dsl.js            # DSL 配置
│   │   ├── height-config.js  # 高度配置
│   │   ├── i18n.config.ts    # 国际化配置
│   │   ├── index.*.tsx       # 组件入口（按平台区分）
│   │   ├── model.ts          # 数据模型
│   │   └── types.d.ts        # 类型声明
│   └── mock/                 # Mock 数据模板
│       ├── globalData.ts     # 全局数据
│       ├── i18nResource.ts   # 国际化资源
│       ├── index.tsx         # Mock 入口
│       └── pageConfig.ts     # 页面配置
└── entry/                    # 入口模板
    ├── index.html            # HTML 入口
    └── main.tsx              # 主入口脚本
```

## 核心模块

### `initGundamProject()`

入口函数，根据用户选择路由到对应的处理器：

- **组件工程** → `handlers/component.ts`
- **页面工程** → `handlers/page.ts`（暂不支持）

### 组件工程处理器 (`handlers/component.ts`)

1. **`copyTemplate`** — 使用 Mustache 模板引擎渲染项目文件，支持条件文件生成
2. **`installDependencies`** — 自动安装项目依赖
3. **`i18nSync`** — 同步国际化资源
4. **`installPreCommit`** — 安装 Git pre-commit 钩子

### 页面工程处理器 (`handlers/page.ts`)

当前为 Stub 实现，提示"暂不支持页面工程"。

### 工具函数

| 模块 | 函数/类 | 说明 |
|------|---------|------|
| `execute-cmd.ts` | `executeCMD()` | 基于 `child_process.spawn` 的命令执行器 |
| `ensure-git.ts` | `ensureGit()` | 检查并初始化 Git 仓库 |
| `logger.ts` | `Logger` | 带前缀栈的日志类，支持嵌套上下文输出 |
