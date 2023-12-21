---
layout: default
title: 创建Dialect
parent: Tutorial教程
nav_order: 1
---

公开Dialect通常分为至少3个目录：

- mlir/include/mlir/Dialect/Foo（用于public include files）
- mlir/lib/Dialect/Foo（用于sources）
- mlir/lib/Dialect/Foo/IR（用于operations)）
- mlir/lib/Dialect/Foo/Transforms（用于transforms）
- mlir/test/Dialect/Foo（用于tests）

除其他公共头文件外，"include"目录还包含一个 ODS 格式的 TableGen 文件，用于描述方言中的操作。该文件用于生成operation的声明（`FooOps.h.inc`）和定义（`FooOps.cpp.inc`），以及operation接口的声明（`FooOpsInterfaces.h.inc`）和定义（`FooOpsInterfaces.cpp.inc`）。

> 操作定义规范（Operation Definition Specification, ODS）

“IR”目录通常包含方言的函数实现，这些函数不是由ODS自动生成的。这些函数通常在`FooDialect.cpp`中定义，其中包括`FooOps.cpp.inc`和`FooOpsInterfaces.h.inc`。

“Transforms”目录包含方言的重写规则，通常使用DDR格式在TableGen文件中描述。

> 表格驱动的声明式重写规则（Table-driven Declarative Rewrite Rule, DRR）

请注意，方言名称通常不应以“Ops”作为后缀，尽管一些仅涉及方言操作的文件（例如`FooOps.cpp`）可能会这样命名。

## CMake最佳实践

### TableGen目标

在方言中，通常使用ODS格式在表格生成器（tablegen）中声明操作，在名为`FooOps.td`的文件中。这个文件是方言的核心，并且通过`add_mlir_dialect()`进行声明。

```cpp
add_mlir_dialect(FooOps foo)
add_mlir_doc(FooOps FooDialect Dialects/ -gen-dialect-doc)
```

