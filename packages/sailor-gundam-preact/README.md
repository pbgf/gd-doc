# @sailorgundam/gundam-preact

> Preact 兼容层 — 在 H5 构建中替代 React

## 概述

`sailor-gundam-preact` 是基于 Preact 的 React 兼容层，在 H5 构建中作为 React 的替代品，以显著减小包体积。它提供了完整的 React 兼容 API，使得现有 React 代码无需修改即可运行。

## 工作原理

通过 Vite/Rollup 构建工具的 alias 机制，将 `react` 和 `react-dom` 的导入重定向到 `@sailorgundam/gundam-preact`：

```
react        → @sailorgundam/gundam-preact
react-dom    → @sailorgundam/gundam-preact
```

## 目录结构

```
├── compat/          # React 兼容 API 实现
├── hooks/           # React Hooks 兼容实现
├── jsx-runtime/     # JSX 运行时（React 17+ JSX Transform 兼容）
└── rollup.config.js # Rollup 构建配置
```

## 核心模块

| 目录 | 说明 |
|------|------|
| `compat/` | 提供 `createElement`、`Component`、`PureComponent` 等 React 核心 API |
| `hooks/` | 提供 `useState`、`useEffect`、`useCallback`、`useMemo` 等 Hooks |
| `jsx-runtime/` | 支持 React 17+ 的新 JSX Transform，无需显式 import React |

## 构建

使用 Rollup 构建，输出兼容格式供 H5 环境使用。
