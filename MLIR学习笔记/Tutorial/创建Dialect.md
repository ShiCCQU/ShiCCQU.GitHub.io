---
layout: default
title: 创建Dialect
parent: Tutorial
nav_order: 1
---

公开Dialect通常分为至少3个目录：
- mlir/include/mlir/Dialect/Foo（用于public include files）
- mlir/lib/Dialect/Foo（用于sources）
- mlir/lib/Dialect/Foo/IR（用于operations)）
- mlir/lib/Dialect/Foo/Transforms（用于transforms）
- mlir/test/Dialect/Foo（用于tests）

除其他公共头文件外，"include "目录还包含一个 ODS 格式的 TableGen 文件，用于描述方言中的操作。该文件用于生成operation的声明（FooOps.h.inc）和定义（FooOps.cpp.inc），以及operation接口的声明（FooOpsInterfaces.h.inc）和定义（FooOpsInterfaces.cpp.inc）。

> 操作定义规范（Operation Definition Specification, ODS）