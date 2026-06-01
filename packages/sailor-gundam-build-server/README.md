# @sailorgundam/gundam-build-server

> Gundam CI/CD 构建服务 · 基于 Egg.js

## 概述

`sailor-gundam-build-server` 是基于 Egg.js 框架的 CI/CD 构建服务，为高达项目提供自动化的持续集成与部署能力，支持多 IDC（香港/欧洲）环境部署。

## 运行环境

- **端口**：8002
- **Worker 数**：8
- **操作系统**：CentOS 7
- **Node.js**：14.19.0

## 目录结构

```
├── app.js                    # 应用入口
├── app/
│   ├── controller/           # 构建控制器
│   └── service/              # 构建服务（CI/CD 流水线、多 IDC 支持）
├── config/                   # 环境配置（Lion 配置中心、MySQL 等）
├── deploy/                   # 部署配置
└── templates/                # 项目脚手架模板
```

## 核心功能

### 自动化 CI/CD

构建服务提供完整的 CI/CD 流水线，从代码拉取、依赖安装、项目构建到产物部署的全流程自动化。

### 多 IDC 支持

支持香港（HK）和欧洲（EU）等多个 IDC 环境的构建与部署，确保各地区的独立发布能力。

### Lion 配置中心

集成美团 Lion 配置中心，实现运行时配置的动态管理与下发。

### Mustache 模板引擎

使用 Mustache 模板引擎生成项目配置和部署文件，支持条件渲染和变量替换。

## 架构

```
请求 → Controller → Service → CI/CD Pipeline
                            → 多 IDC 部署
                            → Lion 配置
```
