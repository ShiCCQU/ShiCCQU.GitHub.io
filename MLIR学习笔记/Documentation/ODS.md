---
layout: default
title: ODS（操作定义规范）
parent: 文档翻译&阅读笔记
nav_order: 1
---

除了专门的mlir::Op C++模板之外，MLIR还支持以表驱动（table-driven）的方式定义操作（operations）和数据类型（data types）。

>table-driven参考资料——LLVM官方文档阅读：
>
> [1.LLVM介绍](https://zhuanlan.zhihu.com/p/446800631)
>
> [2.TableGen_Overview](https://zhuanlan.zhihu.com/p/447318642)
>
> [3.TableGen Language Reference](https://zhuanlan.zhihu.com/p/447728683)

## Motivation

MLIR支持pluggable的方言，方言中包含一系列操作。开放、可扩展的生态导致了“stringly”类型的IR问题，例如，在优化和分析过程中重复的字符串比较、返回类型更通用的不直观访问方法、没有默认参数的冗长和通用构造函数、冗长的文本IR转储等等。

此外，操作验证（operation verification，~~其实不太懂这里的operation概念~~）有三种情况：

1. 最好情况：一个中央字符串到验证函数映射；
2. 中间情况：在代码库中重复验证；或者
3. 最坏情况：没有验证函数。

解决上述问题的方法是以表驱动的方式定义操作。然后对于每个方言，我们可以有一个集中存放关于每个操作所需知道的所有内容（包括其约束条件、自定义汇编形式等）的地方。这个描述也被用来生成帮助函数和类，以便进行构建、验证、解析、打印、分析等多种操作。

**看起来是为了将所有dialect集中管理，方便扩展的意思。**

## Benefits

与C++模板对比，table-driven有如下优点：

- 单一真实来源（Single source of truth）：我们努力将有关操作的所有事实编码到记录中，以便读者无需在代码片段之间跳来跳去就能完全理解一个操作。
- 消除样板代码（Removing boilerplate）： 我们可以从记录中自动生成操作数/属性/结果获取方法、操作生成方法、操作验证方法以及许多其他实用工具。这极大地减少了定义新操作所需的样板代码。
- 促进自动生成（Facilitating auto-generation）：这些操作信息记录的使用绝不仅限于操作定义本身。我们可以使用它们驱动许多其他组件的自动生成，比如计算图序列化。

> 在软件开发中，Single Source of Truth (SSOT) 是一个重要的概念，它的核心理念是在系统或组织中有一个主要的数据源或系统，所有的信息都是从这个主要源获取和更新的。这个主要源就是真理的唯一来源。
>
> 参考资料：[什么是软件设计领域的 Single Source of Truth？](https://zhuanlan.zhihu.com/p/643672084)

## TableGen Syntax（语法）

我们使用TableGen作为指定操作信息的语言。TableGen本身只提供了用于编写记录的语法；在TableGen文件（通常以`.td`为后缀）中允许使用的语法和结构可以参考[llvm-TableGen](https://llvm.org/docs/TableGen/ProgRef.html)。

- TableGen的`class`和C++类相似，可以模版化和子类化。
- TableGen的`def`和C++对象相似，可以通过实例化一个TableGen类来声明（例如，`def MyDef : MyClass<...>;`），也可以完全独立地声明（例如，`def MyDef;`）。它不能进一步进行模板化或子类化（因为已经类似于实例化对象）。
- TableGen的`dag`是一个专门用于有向无环图元素的类型。`dag`具有一个运算符和零个或多个参数。它的语法是`(operator arg0, arg1, argN)`。operator可以是任何TableGen`def`；arg可以是任何东西，包括`dag`本身。我们可以给运算符和参数都附上名称，例如`(MyOp:$op_name MyArg:$arg_name)`。

> 详细介绍：[TableGen Programmer’s Reference](https://llvm.org/docs/TableGen/ProgRef.html)。

## Operation Definition

MLIR通过特殊的[TableGen后端](https://llvm.org/docs/TableGen/BackEnds.html#introduction)：[`OpDefinitionsGen`](https://github.com/llvm/llvm-project/blob/main/mlir/tools/mlir-tblgen/OpDefinitionsGen.cpp)来定义和提供操作的语义，定义了几个常见构造（constructs）。这些构造在[`OpBase.td`](https://github.com/llvm/llvm-project/blob/main/mlir/include/mlir/IR/OpBase.td)中进行了定义。其中主要的有：

- `Op` class：它是定义操作的主要构造。在使用该类时，所有与操作相关的事实都将通过以下构造来指定。
- `Dialect` 类：同一逻辑组（logic group）的操作被放置在相同的方言中。`Dialect`类包含方言级别的信息。
- 多层次的`OpTrait`类：它们用于指定操作的特殊属性和约束，包括操作是否具有副作用以及其输出是否与输入具有相同的形状。
- `ins/outs`标记：这是`OpDefinitionsGen`后端内置的两个特殊标记。它们分别指向操作数/属性和结果的定义。
- 多层次的`TypeConstraint`类：它们用于指定操作数或结果的约束条件。一个值得注意的子类层次结构是`Type`，代表了对常见C++类型的约束条件。
- 多层次的`AttrConstraint`类：它们用于指定属性上的约束条件。一个值得注意的子类层次结构是`Attr`，它代表了对值为常见类型的属性进行约束的条件。

