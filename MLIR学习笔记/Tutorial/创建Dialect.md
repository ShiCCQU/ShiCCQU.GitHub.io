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

```cmake
add_mlir_dialect(FooOps foo)
add_mlir_doc(FooOps FooDialect Dialects/ -gen-dialect-doc)
```

这将生成正确的规则来运行mlir-tblgen，同时还会生成一个名为‘MLIRFooOpsIncGen’的目标，可以用于声明依赖关系。

方言转换通常在文件`FooTransforms.td`中声明。TableGen的目标以典型的llvm方式描述。

```cmake
set(LLVM_TARGET_DEFINITIONS FooTransforms.td)
mlir_tablegen(FooTransforms.h.inc -gen-rewriters)
add_public_tablegen_target(MLIRFooTransformIncGen)
```

结果是另一个运行 mlir-tblgen 的 "IncGen "目标。

### 库目标Library Targets

Dialect 可能有多个库。每个库通常使用`add_mlir_dialect_library()`声明。Dialect 库经常依赖于从 TableGen生成的头文件（由 `DEPENDS`关键字指定）。Dialect 库可以依赖于其他 dialect 的库，这种依赖关系由`target_link_libraries()`和`PUBLIC`关键字声明。例如：

```cmake
add_mlir_dialect_library(MLIRFoo
  DEPENDS
  MLIRFooOpsIncGen
  MLIRFooTransformsIncGen

  LINK_COMPONENTS
  Core

  LINK_LIBS PUBLIC
  MLIRBar
  <some-other-library>
  )
```

`add_mlir_dialect_library()`是`add_llvm_library()`的简单封装，它收集了所有 Dialect 的列表，这个列表通常对于链接工具（如mlir-opt）非常有用。这个列表也被链接到`libMLIR.so`中。可以从`MLIR_DIALECT_LIBS`全局属性中检索到这个列表。

```
get_property(dialect_libs GLOBAL PROPERTY MLIR_DIALECT_LIBS)
```

请注意，尽管 Bar 方言也使用 TableGen 来声明其操作，但不必显式依赖于相应的 IncGen 目标。`PUBLIC` 链接依赖已足够。还要注意我们避免明确使用 add_dependencies，因为这些依赖关系需要在底层的 `add_llvm_library()` 调用中可用，以使其能够正确地创建具有相同源代码的新目标。然而，依赖于 LLVM IR 的方言可能需要依赖于 LLVM 'intrinsics_gen' 目标，以确保 tablegen 生成了 LLVM 头文件。

此外，与 MLIR 库的链接是通过 LINK_LIBS 描述符指定的，并且与 LLVM 库的链接是通过 LINK_COMPONENTS 描述符指定的。这样可以让 cmake 基础设施生成具有正确链接关系的新库目标，特别是当 `BUILD_SHARED_LIBS=on `或者 `LLVM_LINK_LLVM_DYLIB=on` 时。

## Dialect转换

从“X”到“Y”的转换分别位于：

mlir/include/mlir/Conversion/XToY

mlir/lib/Conversion/XToY

mlir/test/Conversion/XToY

转换的默认文件名应该省略其名称中的“Convert”，例如lib/VectorToLLVM/VectorToLLVM.cpp。

转换pass应该与转换本身分开，以方便那些只关心pass而不关心其与模式或其他基础设施的实现的用户。例如：`include/mlir/VectorToLLVM/VectorToLLVMPass.h`。

### CMake最佳实践

每个转换通常存在于单独的库中，使用`add_mlir_conversion_library()`声明。转换库通常依赖于其源和目标方言，但也可能依赖其他方言（例如MLIRFunc）。通常使用`target_link_libraries()`和`PUBLIC`关键字来指定这种依赖关系。例如：

```cmake
add_mlir_conversion_library(MLIRBarToFoo
  BarToFoo.cpp

  ADDITIONAL_HEADER_DIRS
  ${MLIR_MAIN_INCLUDE_DIR}/mlir/Conversion/BarToFoo

  LINK_LIBS PUBLIC
  MLIRBar
  MLIRFoo
  )
```

`add_mlir_conversion_library()`是对`add_llvm_library()`的一个精简封装，它收集所有转换库的列表。这个列表通常对于链接工具（例如mlir-opt），它们应该可以访问所有方言，非常有用。此列表还在`libMLIR.so`中链接。可以从`MLIR_CONVERSION_LIBS`全局属性中检索到该列表：

```cmake
get_property(dialect_libs GLOBAL PROPERTY MLIR_CONVERSION_LIBS)
```

请注意，只需要在编译时和链接时指定对dialect的公共依赖关系，并不需要显式地依赖于dialect的IncGen目标。然而，直接包含LLVM IR头文件的转换可能需要依赖于LLVM 'intrinsics_gen'目标，以确保tablegen生成了LLVM头文件。