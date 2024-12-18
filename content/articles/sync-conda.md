---
title: 'Sync Conda Environment'
date: 2024-09-30T17:53:55+08:00
draft: true
author: Zijian Zang
UseHugoToc: true
tags: 
 - Conda
---

<!--more-->

最近手里的电脑多了起来（自己的一台电脑，实验室的一台电脑，还有一台笔记本）。有时需要在不同电脑上同步conda环境，虽说conda提供了`conda env export`和`conda env create`来导出和创建环境，但是这个过程还是需要较多的手动操作，比较麻烦。因此打算写个脚本用于同步Conda环境。

## 基本理念

基本的想法是：基于github private repo，实现push env和pull env指令用于同步环境。同时，由于git提供了版本管理能力，如果conda环境出现了问题，可以通过git回退版本，只要将脚本操作和git操作绑定起来即可实现这个功能。

## 实现思路

### 导出环境

有两种方式可以导出环境：

1. 使用`conda env export`导出环境，这种方法会导出所有依赖。
2. 使用`conda pack`导出环境，这种方法会将环境打包成一个tar包，包含了所有依赖。

前者只是导出了配置信息，后者则是将环境打包成了一个tar包，包含了所有依赖。

显然，第二种方式更加方便，因为包含了所有依赖，所以更加完整。但是，这种方式也有缺点，就是文件较大，且包含了所有依赖，所以不太适合用于同步环境。
