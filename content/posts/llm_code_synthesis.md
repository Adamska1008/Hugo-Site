---
title: '大语言模型在代码补全中的应用'
date: 2024-11-29T23:07:03+08:00
draft: true
author: Zijian Zang
UseHugoToc: true
tags: 
---

## 前置知识

对于没有开发过LLM应用的程序员来说，可能不知道一个小技巧：你可以人工编写AI消息，并放到消息队列中。它主要有两个用途：

1. 作为历史对话，让LLM模仿我们手动编写的消息来生成文本。这叫做Few-Shot Prompt，应用在多个场景下，例如Auto CoT。
2. 作为最后一条消息，让LLM补全这条消息。

作用2是本文的重点。例如，我们为LLM写好消息的开头。

````plaintext
User:
实现一个矩阵相乘的函数
LLM:
```python
def multiply_matrix(a: list[list[int]], b: list[list[int]]) -> list[list[int]]
    assert a.len() == b.len()
    assert a[0].len() == b[0].len()
````

那么LLM就会自动续写这个消息，也就是做代码补全工作。这就是基于LLM的代码补全的核心思想。

<!--more-->

## 参考文献
