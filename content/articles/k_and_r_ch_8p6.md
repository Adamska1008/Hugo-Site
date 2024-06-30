---
title: '闲聊 K&R ch 8.6：过时的一章'
date: 2024-07-01T00:03:37+08:00
draft: false
author: Zijian Zang
UseHugoToc: true
tags: 
 - linux
 - system programming
---

<!--more-->

《The C Programming Language》是一本经典的C语言入门书籍，其第八章介绍了一部分系统调用相关的知识。8.6章描述了如何基于系统调用动手写一个`ls`，然而，如果你尝试运行示例代码，你会发现：

1. `opendir`、`readdir`、`closedir`函数已被定义。
2. `open(dir_name, O_RDONLY)`虽然能返回正确的fd，但`read(fd)`将产生`errno 21`，目录错误。

先说第一个问题，这是由于这三个函数已经在`sys/dir.h`中定义了。K&R中为了使用`dirent`结构体，选择`#include<sys/dir.h>`，但我们实质上是要自己实现这个头文件，故造成了重复定义。

第二个问题是的原因是，由于年代久远，K&R对目录的处理已经完全过时了。

> Because the format of the directory is considered file system metadata, the file system considers itself responsible for the integrity of directory data; thus, you can only update a directory indirectly by, for example, creating files, directories, or other object types within it. In this way, the file system makes sure that directory contents are as expected.[1]

实际上，目前的linux系统接口中，使用`read`系统调用读取目录数据是不被允许的，一种解释是read被设计为获取线性的数据，而目录数据本身是一个结构化的对象，不符合read的设计原则。唯一获取目录数据方式是使用`getdents`，即Get Directory Entries。不过C语言没用封装这个系统调用，只能通过`syscall`函数调用。

这里提供一个正确的`readdir`实现。

```c
// It works like an iterator.
linux_dirent *readdir(Dir *dp)
{
    static linux_dirent dirent;
    static unsigned long offset;
    static char *buf;
    static int nread;
    // Initialization Stage
    if (buf == NULL)
    {
        buf = (char *)malloc(BUFSIZ);
        nread = syscall(SYS_getdents, dp->fd, buf, BUFSIZ);
        if (nread == -1)
        {
            perror("syscall(SYS_getdents)");
            exit(EXIT_FAILURE);
        }
        if (nread == 0)
            return NULL;
    }
    if (offset >= nread)
        return NULL;
    dirent = *(linux_dirent *)(buf + offset);
    offset += dirent.d_reclen;
    return &dirent;
}
```

其中`linux_dirent`结构体需要自己编写，建议用它更换书中`Dirent`的定义。

```c
typedef struct
{
    long d_ino;
    off_t d_off;
    unsigned short d_reclen;
    char d_name[256];
} linux_dirent;
```

## 参考文献

[1] Remzi H. Arpaci-Dusseau, Andrea C. Arpaci-Dusseau. Operating Systems: Three Easy Pieces.