---
title: 'CMake Version Control Best Practise'
date: 2024-09-13T23:11:55+08:00
draft: true
author: Zijian Zang
UseHugoToc: true
tags: 
 - C++
 - CMake
---

并非Best Pracise，只是抛砖引用，仅仅谈谈我的理解。

<!--more-->

下文中所述内容基于CMake 3.22版本，因为这是Ubuntu 22.04LTS自带的版本，以此版本为基础兼容性较强，且不需要顾虑过久的版本。

CMake 3.12中引入了`CMAKE_PROJECT_VERSION`。首先，可以在`project()`中指定版本号。

```cmake
project(first VERSION 1.2.3)
```

之后可以使用`CMAKE_PROJECT_VERSION`变量引用该版本号。除了该变量以外，还有多个变量与版本号有关：

- `CMAKE_PROJECT_VERSION_MAJOR`或`${project_name}_VERSION_MAJOR` 大版本号。上例中为1。
- `CMAKE_PROJECT_VERSION_MINOR`或`${project_name}_VERSION_MINOR`，小版本号，上例中为2。
- `CMAKE_PROJECT_VERSION_PATCH`或`${project_name}_VERSION_PATCH`，补丁版本号，上例中为3。

准确的说，`CMAKE_PROJECT_VERSION_***`是顶层项目的版本号，而`${project_name}_VERSION_***`则并非一定是顶层项目的，任意子项目都可以。

使用CMake控制版本号的意义在于，可以将版本号在编译期嵌入到代码中，这是git tag做不到的。CMake Tutorial中有[案例](https://cmake.org/cmake/help/v3.18/guide/tutorial/#adding-a-version-number-and-configured-header-file)，案例中，通过定义`.h.in`模板文件，替换文件中的占位符，即可动态设置版本号。

除此以外，还有其他方法可以配置版本号。例如在编译二进制文件时，为target添加definition。

```CMake
target
```

## 参考文献

[1] CMake Documentation.
[2] CMake Tutorial.
