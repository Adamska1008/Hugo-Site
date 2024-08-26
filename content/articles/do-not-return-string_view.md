---
title: '不要返回string_view'
date: 2024-08-26T20:44:20+08:00
draft: false
author: Zijian Zang
UseHugoToc: true
tags: 
 - C++
---

使用C++时，尽量不要返回引用类型。而返回string_view所造成的bug可能比返回string&更恶劣。

<!--more-->

由于C++中缺乏管理引用类型（包括指针或引用）的生命周期的能力，返回任何引用类型都是非常危险的。栈内存的引用可能被回收，而堆内存的引用何时被回收也无人知晓。

如果需要引用类型时，常常也是用引用而非裸指针，这是由于访问引用时会进行内存检查，比裸指针安全——程序会稳定的触发段错误，而不是指向无法预料的数据，而导致程序似乎跑起来了，实际结果完全不可预料。同时，编译器也会检查函数返回引用是否是栈内存的，并报警告。

string_view的内部使用了指针，并且没有提供任何检查内存合法性的手段。这意味着string_view的内存安全性跟裸指针是一个级别的。相较于char指针，string_view的错误使用更加隐蔽，因为为了便捷性，string_view和string可以互相隐式转化，而string无法直接转化为char*。

例如：

```C++
std::string_view f()
{  
    std::string str;
    // after some operations
    return str;
}
```

仅看函数体内部是没有任何问题的，但由于隐式转换，它返回的是指向函数栈内存的指针，而不是完整的string数据，而

```C++
char* f()
{
    std::string str;
    // after some operations
    return str.c_str();
}
```

的错误就非常明显，难以被忽略。

如果不是由于性能问题必须使用零拷贝，或代码编写者能够百分百明确生命周期无问题，函数都不应该返回string_view。
