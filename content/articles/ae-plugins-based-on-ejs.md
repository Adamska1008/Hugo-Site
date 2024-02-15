---
title: 使用ExtendScript编写AE插件
date: 2024-02-14T19:15:02+08:00
draft: true
author: Zachary Zang
tags: 
    - ExtendScript
---

最近写毕设需要为AE写插件，然而国内用AE的人不少，写AE插件的倒是不多。故记录一下学习编写插件的过程，想要自行编写插件的读者也可借鉴。

<!--more-->

## 环境配置

除非使用Adobe的付费工具，不然ExtentScript的编程体验都是比较差的。一般使用免费的VSCode进行开发，需要安装 **ExtentScript Debugger** 插件和任意一个 **AE Runner** 插件。这两个插件为我们提供了即时运行插件，查看插件效果的功能。然而遗憾的是，目前没有插件能够较好的提供ExtendScript的Intellisence，因此只能随用随查[文档](https://ae-scripting.docsforadobe.dev/)了。

环境配置到此为止，ExtendScript只是一个脚本语言，其运行时都在AE中，故完全无需配置运行环境，只需要新建一个jsx文件，编写脚本，并在VSCode中右键运行即可。

## Hello world

环境配置完毕，我们目前的目标是在AE中跑起一个具备基本功能的脚本，包括：

1. 可用的用户界面（UI）
2. 与AE正确进行交互

实际上，这正对应了插件编写的两大部分：AE Scripting和ScriptUI。由于ExtendScript是Adobe各产品通用的插件语言，故其编写UI的逻辑与软件交互的逻辑是完全无关的，而是独立为[ScriptUI](https://extendscript.docsforadobe.dev/index.html)。

### ScriptUI

首先介绍UI。[Window](https://extendscript.docsforadobe.dev/user-interface-tools/window-object.html)对象是插件界面的主题，后续的按钮，输入框等等部件都布局在插件的Window对象上。

```js
var myWindow = new Window("palette", "My Window", undefined);
```

`palette`是Window对象的类型，而`My Window`则是Window对象的标题，`undefined`处则是Window对象的其他property，没有特别需要的话为`undefined`即可。许多ScriptUI的组件初始化都遵循`(类型，名称，参数)`范式。

Window具有一些可设置的属性，例如`orientation`，它控制了其子组件的排列方式。

```js
myWindow.orientation = "column";
```

其值可以为`column`或`row`，代表子组件纵向或横向排列。类似的，其他属性的功能可以查看文档。

可以向Window对象中添加子组件，它们在ScriptUI中被称为[Control objects](https://extendscript.docsforadobe.dev/user-interface-tools/control-objects.html)。统一使用Window对象的`add`方法添加子组件。

```js
var button = group.add("button", undefined, "Button One");
```

button组件并不是独立的类，而是Control objects中的一个type。类似的还有dropdownlist、checkbox等等。可以使用group或panel将子组件进行聚类，从而细化布局。

```js
var group = myWindow.add("group", undefined, "My Group");
group.orientation = "row";
var buttonOne = group.add("button", undefined, "Button One");
var buttonTwo = group.add("button", undefined, "Button Two");
```

编写一个简单的插件，使用上述组件足以。完成编写后，在脚本中调用`show`方法展示插件界面。

```js
myWindow.center();
myWindow.show();
```

这些组件如何与AE进行交互呢？使用回调（Callback）是一种比较好的方式

```js
myButton.onClick = function(){ /* ... */ } 
```

### AE Scripting

解决了UI问题，接下来就是要看如何通过脚本操作AE。脚本，就是通过代码来替代手动编辑。脚本最基本的功能是选中某个对象并修改其属性，AE Scripting要如何实现这一点呢？

首先介绍AE Scripting中对象系统的基本概念。AE这一软件本身是全局的`app`对象，`app.project`则是当前软件的当前项目，大部分修改操作是基于`app.project`进行的。在AE的基本布局中，`app.project`的内容对应了左侧project panel中的内容，其中包含合成以及其他的一些素材，这些即是`project`对象包含的`item`对象。使用`project.item(i)`或`project.item[i]`可以访问这些对象。

我们继续以合成对象为例，合成对象在AE Scripting中为[CompItem](https://ae-scripting.docsforadobe.dev/items/compitem.html)，在AE的基本布局中，选中合成对象，其内容会在下侧的Comp Panel中展示，而其中的内容主要为图层（layer）。通过`item.layer(i)`接口可以获取第`i`个item，我们也可以调用`item.layers()`方法获取[LayerCollection对象](https://ae-scripting.docsforadobe.dev/layers/layercollection.html)。

以一个脚本为例，它弹窗展示目前Comp项目的名称（假设它是项目中的第一个item），以及CompItem下每一个图层的名称。

```js
var comp = app.project.item(1);
alert(comp.name)
var layersNum = comp.layers.length;
for (var i = 1; i <= layersNum; i++) {
    alert(comp.layer(i).name)
}
```

编写脚本时可以直接参考AE软件中的对象命名，一般来说脚本中的对象参数名与AE软件中操作的参数名是一致的，前提是将AE的语言修改为英文。例如，随意创建一个矩形对象，可以在Comp Panel中看到，它具有Contents与Transform两个可调整属性。在脚本编辑中，我们只需要访问对应layer对象的`contents`与`transform`属性，即可完成对应的调整。调整的具体操作参考文档即可。

规范的说，诸如`transform`这样的属性，属于AE Scripting中的[Property](https://ae-scripting.docsforadobe.dev/properties/property.html)。

## 参考文献

[1] After Effects Scripting Guide.https://ae-scripting.docsforadobe.dev/

[2] After Effects Scripting For Beginners. https://www.youtube.com/watch?v=DTBtfFiyjNU

[3] JavaScript Tools Guide CC. https://extendscript.docsforadobe.dev/index.html