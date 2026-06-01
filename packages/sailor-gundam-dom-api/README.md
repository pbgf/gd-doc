# @sailorgundam/gundam-dom-api

> DOM API 实现 — MachPro Native 环境的 DOM 接口

## 概述

`sailor-gundam-dom-api` 为 MachPro Native 环境提供标准 DOM API 的实现。在 Native 渲染环境中没有浏览器 DOM，本包实现了 `document.createElement`、`setAttribute`、`addEventListener` 等核心 DOM 接口，使得基于 DOM API 的组件代码可以在 Native 环境中正常运行。

## 核心功能

提供以下标准 DOM API 的 Native 实现：

- `document.createElement` — 创建元素
- `setAttribute` — 设置元素属性
- `addEventListener` / `removeEventListener` — 事件监听
- `appendChild` / `removeChild` — 子节点操作
- `setProperty` — 样式属性设置
- 其他 DOM 操作接口

## 编译输出

```
dist/
├── cjs/     # CommonJS 格式
└── types/   # TypeScript 类型声明
```
