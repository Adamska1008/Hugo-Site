---
title: 'Sum Type in Cpp'
date: 2024-09-18T19:40:00+08:00
draft: false
author: Zijian Zang
UseHugoToc: true
tags: 
 - C++
---

<!--more-->

Sum Type，直译为*和类型*，类似于布尔运算中的和运算，表示为*或*，即一个类型可以是某几种类型中的一种。

C++没有提供语法层面的Sum Type。由于C++的枚举是不能够携带值的，因此枚举最多只能用来实现Sum Type，而本身并不是Sum Type。

## `std::variant`

C++17标准引入了std::variant。它可以用来表示和类型。

```C++
struct circle { std::size_t radius; };

struct square { std::size_t size; };

using shape = std::variant<circle, square>;
```

配合标准库的一系列函数可以访问内部成员。

```C++
// shape s
bool is_circle = std::holds_alternative<circle>(s);
circle* c = std::get_if<circle>(&s);
circle c = std::get<circle>(s); // may throw std::bad_variant_access
// ...
```

## Nested Class & Union & Enum

还有一种常见的方法来实现Sum Type，它基于Union，同时利用Nested Class来补充类型信息与暴露对外接口。

```C++
struct shape
{
    enum class type
    {
        CIRCLE,
        SQUARE,
    };

    struct circle { std::size_t radius; };
    struct square { std::size_t size; };

    union inner 
    {
        circle c;
        square s;
    };

    type t;
    inner i;
};
```

有了enum，就可以在switch语句中使用。

```C++
// shape s
switch (s.type)
{
case shape::type::CIRCLE:
    auto c = &inner.i.c;
    // ...
    break;
case shape::type::SQUARE;
    auto s = &inner.i.s;
    // ...
    break;
}
```

相较于`variant`，这一方法的主要优势是能够提供类型信息。
