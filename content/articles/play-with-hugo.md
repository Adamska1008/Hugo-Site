---
title: '玩转Hugo'
date: 2024-02-13T19:55:16+08:00
draft: true
author: Zachary Zang
TocOpen: true
tags: ['Hugo']
---
当你安装完Hugo并按照[Getting Started](https://gohugo.io/getting-started/)走一遍流程之后，虽然你拥有了一个可以编写内容的博客，但是它的功能还是过于简陋了，难以作为一个正式的博客被他人阅读。你可能会希望为自己的Hugo博客添加功能、美化界面或做其他一系列建设博客的工作，为此提供一些我个人的想法。

<!--more-->


## archetypes（原型）

archetypes目录，或者说 “原型”，是在使用`hugo new content ...`指令创建文件时所使用的模板。Hugo默认模板为`archetypes/default.md`，其内容为
```toml
+++
date = '{{ .Date }}'
draft = true
title = '{{ replace .File.ContentBaseName `-` ` ` | title }}'
+++
```

原型的主要用途是描述博客的元数据（front matter），默认使用`toml`作为元数据，你也可以使用`yaml`或`json`，但要注意同时修改分隔符为`---`或`{}`。

在原型中可以看到`data`属性被指定为`{{ .Date }}`，而`title`属性被指定为`{{ replace .File.ContentBaseName '-' ' ' | title }}`。使用`{{}}`扩起的内容是模板（template），用于动态编辑内容。`.Date`是一个对象，用于表示当前时间，也就是渲染模板时的时间。而`replace`和`title`都是函数，前者用于将字符串中的`-`替换为空格，后者则将原字符串添加为类似标题的样式（例如首字母大写）。`|`用于将前者的输出转化为后者的输入。

编辑元数据未必需要使用模板，你可以直接添加固定的元数据，例如文章作者，是否开启ToC等。当然，除了元数据之外的内容也可以被添加到archetypes中，例如你可能希望文末总是有参考文献，文章开头总是一段概述。这样的一个archetypes类似于

```
---
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
date: {{ .Date }}
draft: true
author: Zachary Zang
UseHugoToc: true
tags: 
---

<!--more-->

## 参考文献
```

## 菜单

可以通过设置配置文件中的`menu.main`属性来添加菜单项。`menu.main`是一个数组，其每一项是一个菜单子项，可以通过修改`identifier`、`name`、`url`来指定一个菜单子项，例如

```yaml
menu:
    main:
        - identifier: 0
          name: 文章
          url: /articles/
        - identifier: 1
          name: 标签
          url: /tags/

```

修改完毕后，在每个页面的顶部可以见到菜单，点击菜单子项即可跳转到对应目录。`url`属性用于控制跳转的位置，其功能即为直接跳转到根目录+路由对应的页面。

## content文件夹

Content目录存放了博客的文章内容。Hugo支持通过树状结构来组织和管理文章，你可以通过子文件夹来更好地管理文章。例如，你可以将长文章放在`/articles`目录下，然后在浏览器中访问`/articles`URL，即可查看所有长文章。

## 参考文献

[1] Hugo Doc.https://gohugo.io/documentation/

[2] Luke Smith.Hugo Actually Explained (Websites, Themes, Layouts, and Intro to Scripting).https://www.youtube.com/watch?v=ZFL09qhKi5I

[3] PaperMod Wiki.https://github.com/adityatelange/hugo-PaperMod/wiki/