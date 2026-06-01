# @sailorgundam/gundam-node-polyfills

> v0.2.4 · Node.js 核心模块的浏览器环境 Polyfill

## 概述

`sailor-gundam-node-polyfills` 为浏览器环境提供 Node.js 核心模块的兼容实现。在高达 H5 构建中，部分第三方依赖引用了 Node.js 核心模块，本包确保这些模块在浏览器中可用。

## 支持的模块

| 模块 | 说明 |
|------|------|
| `buffer` | Buffer 实现 |
| `process` | process 全局对象 |
| `stream` | 流（Stream）实现 |
| `util` | 工具函数 |
| `path` | 路径处理 |
| `url` | URL 解析 |
| `events` | EventEmitter |

以及更多 Node.js 核心模块。

## 目录结构

```
├── polyfills/              # 各模块的独立实现
│   ├── buffer.js
│   ├── process.js
│   ├── stream.js
│   └── ...
└── rollup.config.js       # Rollup 构建配置
```

## 构建

使用 Rollup 构建，将各 polyfill 模块打包为浏览器可用的格式。
