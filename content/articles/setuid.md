---
title: '读论文：Setuid Demystified'
date: 2024-01-24T19:54:18+08:00
draft: true
author: Zijian Zang
UseHugoToc: true
tags: 
 - Linux
 - Unix
 - System Programming
---



<!--more-->

## 基本知识

Users ID与Groups ID在进程中用于决定进程访问资源的权限。有时进程需要额外的访问权限，Unix系统提供了*uid-setting system calls*来提权与降权。

一个进程实际上会存储三个uid：*real user ID(ruid)* 用于标识进程拥有者、*effective user ID(euid)* 用于访问控制、*saved user ID(suid)* 用于保存过去的uid，以便恢复。特别的，Linux中进程还拥有*file system user ID(fsuid)* 用于文件的访问控制。当调用setuid系列系统调用时，操作系统首先检查进程是否有修改uid的权限。如果有，则根据调用setuid的具体规则修改uid。

由`fork`创建的进程会继承父进程的uid。对于由`exec`执行的文件，如果它的`setuid`位为0，则继承调用者进程的uid，否则其euid与suid将被设置为该文件拥有者的uid。后者是由贝尔实验室提出的一种策略，其在1979年被授予专利。

## 混乱的开始

虽然uid相关的系统调用看起来似乎并没有什么特别的，但如果你仔细阅读POSIX规范中关于uid的部分，你会发现它模糊地使用了*appropriate privileges*这个词汇，该词汇定义在POSIX 1003.1-1988第2.3节：

>An implementation-defined means of associating privileges with a process with regard tothe function calls and function call options defined in this standard that need special privileges. There may be zero or more such means.

这里还是把setuid的定义贴出来。

---

If {_POSIX_SAVED_IDS} is defined:

1. If the process has appropriate privileges, the setuid() function sets the real user ID, effective user ID, and the [saved user ID] to newuid.
2. If the process does not have appropriate privileges, but newuid is equal to the real user ID or the [saved user ID], the setuid() function sets the effective user ID to newuid; the real user ID and [saved user ID] remain unchanged by this function call.

Otherwise:

1. If the process has appropriate privileges, the setuid() function sets the real user ID and effective user ID to newuid.
2. If the process does not have appropriate privileges, but newuid is equal to the real user ID, the setuid() function sets the effective user ID to newuid; the real user ID remains unchanged by this function call.

(POSIX 1003.1-1988, Section 4.2.2.2)

---

所以说，POSIX对该词的定义中，既没有明确哪些操作需要什么特权，也没有声明该功能需要如何实现（甚至可以不实现）。这就导致了不同操作系统对*appropriate privileges*做出了不同的诠释。

在**Solaris**（Solaris 8）中，*appropriate privileges*指的是进程euid为0，即root。而**Free BSD**（Free BSD 4.4）允许将real ID设置为effective ID，因此当`newuid==geteuid()`时，`setuid(newuid)`符合*appropriate privileges*。`euid`为root也是符合的。值得一提的是，Free BSD没有定义`_POSIX_SAVED_IDS`，但它确实有saved ID。为了符合POSIX标准，`setuid`不会修改saved ID。

## 参考文献

[1] Hao Chen, David Wagner. Setuid Demystified.

[2] Portable Operating System Inteface for Unix(POSIX).
