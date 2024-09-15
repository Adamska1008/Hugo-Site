---
title: 'Overload String Params'
date: 2024-09-15T17:21:31+08:00
draft: false
author: Zijian Zang
UseHugoToc: true
tags: 
 - C++
---

<!--more-->

假定一个函数`F`，它接受一个字符串类型的参数。为了优化性能，考虑为右值和左值各重载一个函数实现；

```C++
void F(std::string_view attr);

void F(std::string && attr);
```

理论上，左值和右值两种情况都被覆盖了。即使二者函数内部的实现完全相同，也可以避免传参时的额外拷贝。这似乎很不错。然而，假如你使用了字符串字面量作为参数，你会发现二者都包含了隐式转换，从而导致`C2668 ambiguous call to overloaded function`。

针对这一问题，最简单的方式是添加一个新的重载。

```C++
void F(const char * attr) { return F(std::string_view(attr)); }
```

`string_view`与`string&&`都包含了一次隐式转换生成新对象，因此编译器无法判断应该重载哪一个函数。而`const char *`作为参数时，并没有隐式转换，因此编译器会选择调用该重载。

为什么内部直接调用了`string_view`的重载呢？因为在设计的角度上来说，`const char *`相当于是对静态区的左值引用，与`string_view`起到相同的作用。

但这个简单的方法并没有从根本上解决问题。`std::string_view`的一大作用就在于，可以直接匹配`const char *`与`const string &`作为参数，从而避免实现两个重载。而在上文中，为了解决冲突，依然需要提供`const char *`的重载。这使得`string_view`的使用失去了意义。

从这一点上考虑，只能说`string_view`与`string &&`非常不适合作为重载的参数。更好的解决办法是思考函数真的需要两个不同的重载吗？是否需要获取字符换的所有权？

## 参考文献

[1] CppCon 2018: Titus Winters “Modern C++ Design
[2] StackOverflow [std::string&& vs std::string_view as arguments for functions.](https://stackoverflow.com/questions/75592458/stdstring-vs-stdstring-view-as-arguments-for-functions)
