---
title: '我如何使用PROSE'
date: 2024-02-22T11:21:17+08:00
draft: true
author: Zijian Zang
UseHugoToc: true
tags: 
 - prose
 - program by example
---

由于我的毕设涉及到prose，而prose本身无论国内外相关文章都比较少。因此，我决定以记录毕设过程的方式，聊一聊prose学习与使用过程中遇到的一系列问题。

<!--more-->

## 从哪里学习prose？

[本文（bushi）](./try-prose.md)。学习prose的主要资料是prose的[官方文档[1]](https://www.microsoft.com/en-us/research/project/prose-framework/tutorial/)。然而，官方文档存在大量问题，包括但不限于：
1. 仅安装官网指示的prose [nuget包](https://www.nuget.org/packages/Microsoft.ProgramSynthesis/)并不能正常运行代码合成，这是因为`Compiler`模块从prose第三版开始就被独立为单独模块了（而prose文档很多还在第一版），需要另外从nuget上搜索下载安装。
2. `tutorial/#syntax`下，程序执行代码块无法正常运行。这是因为prose已经彻底废弃了`HumanReadable`的`ASTSerializationFormat`，目前想要从文本创建program的AST，唯一方法是提供完整的XML文本，用简单的一行`"Substring(x, PositionPair(AbsolutePosition(x, 0), AbsolutePosition(x, 5)))"`是生成不了SubString程序的。
3. 没有API文档。因此，prose学习基本依赖面向示例。示例在下文指示的prose仓库

prose的[github仓库[2]](https://github.com/microsoft/prose)提供了大量示例，包括两个方面：已有的PBE框架的API、prose本身的示例程序。通过学习`dsl-samples/tutorial`可以基本掌握prose的功能。

## 我如何使用prose？



## 参考文献

[1] Prose Framework.https://www.microsoft.com/en-us/research/project/prose-framework/.
[2] Prose Github Repositry.https://github.com/microsoft/prose.