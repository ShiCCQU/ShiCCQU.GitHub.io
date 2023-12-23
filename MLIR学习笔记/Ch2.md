## Chapter 2：发射（emitting）基本MLIR

### 介绍：多层中间表达

其他编译器，如 LLVM，提供了一组固定的预定义类型和（通常是低级/类 RISC）指令。在输出 LLVM IR 之前，特定语言的前端需要执行任何特定语言的类型检查、分析或转换。例如，Clang 不仅会使用其 AST 进行静态分析，还会进行转换，如通过 AST 克隆和重写进行 C++ 模板实例化。最后，在比 C/C++ 更高层次上构建的语言可能需要从其 AST 中进行非实质性的降级，才能生成 LLVM IR。

因此，多个前端最终都要重新实现大量的基础架构，以支持这些分析和转换的需要。MLIR 通过设计可扩展性解决了这一问题。因此，在MLIR术语中只有少数预定义指令（操作）或类型存在。

### MLIR接口

MLIR的设计目标是成为一个完全可扩展的基础架构；没有固定的属性集合（类似于constant metadata）、操作或类型。MLIR通过方言[Dialect](https://mlir.llvm.org/docs/LangRef/#dialects)的概念来支持这种可扩展性。方言提供了一种在独立`namespace`下进行抽象组织的机制。

在MLIR中，操作[Operations](https://mlir.llvm.org/docs/LangRef/#operations)是抽象和计算的核心单元，与LLVM指令在许多方面相似。操作可以具有特定于应用程序的语义，并可用于表示LLVM中所有核心IR结构：指令、全局变量（如函数）、模块等。

这是Toy转置操作的MLIR汇编代码：

```mlir
%t_tensor = "toy.transpose"(%tensor) {inplace = true} : (tensor<2x3xf64>) -> tensor<3x2xf64> loc("example/file/path":12:1)
```

让我们来分解一下这个MLIR<u>操作</u>的结构：

- `%t_tensor`：这个操作定义的结果被赋予一个名称（其中包括了一个前缀符号用来避免冲突）。一个操作可以定义0个或多个结果（在Toy中限制操作为单一结果），这些结果都是SSA值。该名称在解析过程中使用，但不是持久化的（例如，在内存表示中不跟踪SSA值）。
- `"toy.transpose"`：操作名称，它应该是一个唯一的字符串，“·”之前代表Dialect的命名空间。`"toy.transpose"`可以理解为`toy`方言中的`transpose`转置操作。
- `(%tensor)`：一个包含零个或多个输入操作数（或参数）的列表，这些操作数是由其他操作定义的SSA值或块参数的引用（？referring to block arguments）。
- `{ inplace = true }`：一个包含零个或多个属性的字典，这些属性是始终保持不变的特殊操作数，这里定义了一个名为`inplace`的布尔属性，其常量值为`true`。
- `(tensor<2x3xf64>) -> tensor<3x2xf64>`：这是指函数形式中的操作类型，括号内列出参数类型，然后再列出返回值类型。
- `loc("example/file/path":12:1)`：这是该操作在源代码中的位置。

这里展示了一个操作的一般形式。如上所述，MLIR中的操作集是可扩展的。使用一小组概念对操作进行建模，使得可以以通用方式推理和处理操作。这些概念包括：

- A name for the operation. 操作的名称。
- A list of SSA operand values. SSA 操作数值列表。
- A list of [attributes](https://mlir.llvm.org/docs/LangRef/#attributes). 属性列表。
- A list of [types](https://mlir.llvm.org/docs/LangRef/#type-system) for result values. 结果值类型列表。
- A [source location](https://mlir.llvm.org/docs/Diagnostics/#source-locations) for debugging purposes. 用于调试目的的源位置。
- A list of successors [blocks](https://mlir.llvm.org/docs/LangRef/#blocks) (for branches, mostly). 后继块列表（主要用于分支）。
- A list of [regions](https://mlir.llvm.org/docs/LangRef/#regions) (for structural operations like functions). 区域列表（用于函数等结构性操作）。

在 MLIR 中，每个操作都有一个与之相关的强制源位置。在 LLVM 中，调试信息位置是元数据，可以丢弃；而在 MLIR 中，位置是核心要求，API 依赖并操作它。因此，放弃一个位置是一个显式的选择，不可能在错误的情况下发生。

举个例子来说明：如果一个转换将一个操作替换为另一个操作，那么新的操作仍然必须有一个附加的位置。这样可以追踪该操作来自哪里。

值得注意的是，mlir-opt工具（用于测试编译器传递的工具）默认情况下不会在输出中包含位置信息。使用`-mlir-print-debuginfo`标志可以指定包含位置信息。（运行`mlir-opt --help`获取更多选项。）

### 不透明（Opaque） API