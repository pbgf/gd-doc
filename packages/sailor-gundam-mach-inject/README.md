# @sailorgundam/gundam-mach-inject

> MachPro 框架运行时注入代码

## 概述

`sailor-gundam-mach-inject` 是一个单文件包，包含需要注入到 Native 构建产物中的 MachPro 框架运行时代码。在 Native 构建流程中，该代码会被嵌入到最终输出中，为 Native 环境提供必要的运行时支持。

## 说明

- 单文件包，结构极简
- 在构建时注入 Native 产物
- 提供 MachPro 框架在 Native 环境下运行所需的基础逻辑
