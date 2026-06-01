# @sailorgundam/gundam-html-template-react

> H5 页面 HTML 外壳模板

## 概述

`sailor-gundam-html-template-react` 提供 H5 页面的 HTML 外壳模板，包括开发环境模板和生产环境的 FreeMarker 模板，以及小程序的 HTML 模板。

## 目录结构

```
├── index.html        # 开发环境 HTML 外壳
├── main.ftl          # 生产环境 FreeMarker 模板（默认机房）
├── main-eu.ftl       # 生产环境 FreeMarker 模板（欧洲机房）
├── mini-html.js      # 小程序 HTML 模板
└── publish.js        # 发布脚本
```

## 文件说明

| 文件 | 说明 |
|------|------|
| `index.html` | 开发环境使用的 HTML 外壳，包含热更新等开发时功能 |
| `main.ftl` | 生产环境 FreeMarker 模板，用于服务端渲染 HTML 外壳（默认/香港机房） |
| `main-eu.ftl` | 欧洲机房的生产环境 FreeMarker 模板，适配欧洲 IDC 的 CDN 和资源配置 |
| `mini-html.js` | 小程序环境下的 HTML 模板实现 |
| `publish.js` | 模板发布脚本，处理模板的构建与发布流程 |
