# @sailorgundam/gundam-common-core

> 共享 TypeScript 类型定义与常量

## 概述

`sailor-gundam-common-core` 提供高达项目各包之间共享的 TypeScript 类型定义和常量。作为类型基础设施，确保各模块间的类型一致性。

## 编译输出

```
dist/
├── cjs/     # CommonJS 格式
└── types/   # TypeScript 类型声明
```

## 说明

- 本包无独立的 `src/` 目录，类型定义直接在包根目录或通过编译输出提供
- 主要导出接口类型、枚举常量等共享定义
