---
title: '读论文：LLVM: a compilation framework for lifelong program analysis & transformation'
date: 2024-07-09T20:50:02+08:00
draft: true
author: Zijian Zang
UseHugoToc: true
tags: 
 - Compilers
 - LLVM
---

<!--more-->

LLVM的提出者认为，程序分析与转化有必要在程序的整个生命周期中进行，既是为了程序优化，也是为了灵活集成其他工具。LLVM旨在通过一种新的中间语言和编译器实现，来达成这一目标。LLVM中间码是一种类RISC语言，但有着更高级的信息（为了程序分析），例如类型信息、控制流图、数据流。LLVM中间码具有独立的一套类型系统，能够用于实现高级语言的类型；同时其具有能够保存类型信息的计算操作。LLVM中间码还具有两个的异常处理指令，用来实现高级语言中的异常机制。除此之外，LLVM中间码提供了一套内存抽象系统。

基于LLVM中间码，LLVM编译框架能够提供一系列功能，包括：

1. 存在于程序完整生命周期的编译模型。以此，可以在程序的任何一个阶段进行优化，包括运行阶段，交互阶段。
2. 离线代码生成。依然可以生成优化激进的，性能强的代码。尽管这意味着无法生成运行时代码。
3. 基于用户的性能分析和优化。LLVM框架可以在运行时收集分析信息，
4. 透明的运行时模型，没有使用对象、异常语法或运行时环境，任何语言都能编译为LLVM中间码。
5. 独立于语言，程序可以集成多语言编写的代码，编译为统一的中间码。

LLVM致力优化三个方面：1. 中间码的体积与性能。2. 编译器的表现。3. 提供一些示例，用于展现LLVM解决挑战性问题的能力。

### LLVM中间码

LLVM中间码的核心创新点在于：1. LLVM类型系统与`getelementptr`指令。2. LLVM内存模型。3. `invoke`与`unwind`指令，用于实现异常处理。

LLVM指令集描述了处理器的原始操作，但是避免提及硬件规范，例如寄存器、流水线。LLVM提供了**无限的带类型虚拟寄存器**，可以保存原始类型如：布尔（boolean）、整型（integer）、浮点数（floating point）和指针（pointers）。虚拟寄存器使用**SSA**（Static Single Assignment）形式进行计算。LLVM使用**load/store结构**，这意味着内存访问指令和算术、逻辑指令分开处理。load/store指令基于有类型的指针来进行。

LLVM指令集（在论文发表时，2003年）只有31个操作符。这是因为：1. LLVM避免实现功能相同的操作码，例如用xor取代not。2. 大多数操作符支持重载，例如`add`可以适用于整型或浮点型。大多数指令（算术和逻辑运算）为**三地址码**（Three-address code，TAC）。

为了实现SSA，LLVM提供了`phi`指令用来实现&Phi;算符。LLVM也使得每个函数都可以明确地以控制流图来表示。

#### 类型系统

每个SSA寄存器和内存对象都有一个关联类型。额外的类型信息实现了操作符的重载，也有助于帮助实现一些高级代码的转化和优化。除了原始类型，LLVM也提供了四种可派生类型：指针（pointer）、数组（array）、结构体（structure）和函数（function）。作者认为大部分高级语言的数据类型最终都是用这些可派生类型来表示的。

`cast`指令用于将转化值到任意类型，它是唯一一种转换类型的方式。尽管转换类型是任意的，类型信息整体来说依然是可靠的。

地址运算是另一大难题。`getelementptr`指令用于实现类型安全的地址运算。该指令用于获取指向某对象的子字段的指针。例如，给定一个数组指针和一个偏移量数字，该指令将返回指向对应元素的指针。由于Load/Store指令只接受一个指针，故该指令使得访存简便许多。

#### 显示内存分配和统一内存模型

LLVM提供了带类型的内存分配指令`malloc`，它在堆区内存分配一个或多个指定类型的元素，返回一个指向该内存的带类型信息的指针。`free`指令用于释放内存。`alloca`用于在当前函数的栈区开辟内存，栈区指令的扩展都是用`alloca`显示执行的。

LLVM一切左值（addressable object）都是被显示分配的。而全局变量和函数的定义都是通过对象地址来进行的。这样就允许LLVM使用统一的内存模型，所有的内存操作都基于带类型的指针进行，没有隐式访问内存的方法。这简化了访存分析，而且不需要重定义取地址运算符。

#### 函数调用和异常处理

LLVM的函数调用基于`call`指令。该指令接受一个带类型的函数指针（函数名或地址）与其他带类型的参数。这一层函数调用的抽象同样是与硬件无关的。

LLVM的最大一个特征是提供了一个显式的、低层级的、硬件无关的实现异常功能的机制。这个机制也支持实现C语言中的`setjmp`与`longjmp`，能和其他语言中的异常机制一样用相同的处理办法优化。

异常处理机制基于两个指令，`invoke`和`unwind`。它们提供了一种基于**堆栈展开**（stack unwinding）的异常实现方法。`invoke`和`call`类似，但在调用时会添加另一个基本块（basic block）用于异常处理。当调用`unwind`后，程序回退到`invoke`指令，并将控制流转化到`invoke`创建的另一个基本块中。原论文中提供了一个C++语言的示例。

```c++
{
Class Object; // Has a destructor
func(); // Might throw
...
}
```

```llvm
...
    ; Allocate stack space for object:
    %Object = alloca %Class, uint 1
    ; Construct object:
    call void %Class::Class(%Class* %Object)
    ; Call ‘‘func()’’:
    invoke void %func() to label %OkLabel
                    except label %ExceptionLabel
OkLabel:
    ; ... execution continues...
    ExceptionLabel:
    ; If unwind occurs, excecution continues
    ; here. First, destroy the object:
    call void %Class::~Class(%Class* %Object)
    ; Next, continue unwinding:
    unwind
```

#### 文本，二进制与内存中表示

一言以蔽之，LLVM中间码的表示在这三者间是等价的，可以互相转化而不用担心信息丢失。

### LLVM编译器结构

在前端编译器编译代码到LLVM中间码后，LLVM链接器负责进行链接工作并进行优化，同时输出优化完毕的机器码和LLVM中间码。在运行时，一个轻量级的指令系统可以分析并进行简单的优化，这些分析信息可以被附加到程序中，使得离线优化器可以执行大量激进的优化工作。

尽管如此，语言层级的优化只能在编译器前端执行。而且诸如Java这样需要复杂运行时的语言未必能从LLVM优化中获益。

## 参考文献
