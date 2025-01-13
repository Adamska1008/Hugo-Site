---
title: '修复Hugo显示中文阅读时间不准确'
slug: 'Fix_reading_speed_for_chinese'
date: '2025-01-13T20:39:34+08:00'
draft: true
author: Zijian Zang
toc: true
description: 
image: 
tags: 
 - hugo
categories:
keywords:
 - hugo
---

Hugo提供了`PAGE.ReadingTime`方法，用于获取估计阅读时间。然而，它的计算方法对于中文来说是不准确的：

{{< quote source="hugo doc" url="https://gohugo.io/methods/page/readingtime/">}}
The estimated reading time is calculated by dividing the number of words in the content by the reading speed.
{{< /quote >}}

也就是说，`$ReadingTime := div $WordCount $ReadingSpeed`。然而，`WordCount`是以空格划分的，而中文几乎不会使用空格，这种计算方式会导致
