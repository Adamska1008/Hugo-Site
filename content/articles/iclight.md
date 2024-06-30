---
title: 'IC-Light：有用，但不万能'
date: 2024-07-01T00:12:37+08:00
draft: true
author: Zijian Zang
UseHugoToc: true
tags: 
---

<!--more-->

[IC-Light](https://github.com/lllyasviel/IC-Light)整体效果还是比较惊艳的。我在移动端4060上使用它处理768\*512大小的图片，显存消耗比想象中大不少，速度也远比SDXL慢，8G显存很快就爆了，使用内存交换勉强可以跑起来。关掉重绘放大后人物变形明显，不得不开。

人脸基本偏暗，可能是模型导致的。看了一下README中提到的训练的基本逻辑，发现其训练中使用的光源均在冠状面上，导致模型没有训练顺光，故难以合成顺光的图片。对于一些特殊场景，如需要左右冷暖光对比，或者顶光烘托氛围的情况，可能比较使用。但是一般人像摄影中，总会有顺光或侧光作为主光源，此时模型就无能为力了。

使用的模型为SD1.5，既不是SDXL也不是SD3，看来实验时间比较长了。不清楚作者未来是否考虑更新到最新模型。

## 参考文献

[1] IC-Light Github Page. https://github.com/lllyasviel/IC-Light.