---
title: '读论文：Setuid Demystified'
date: 2024-01-24T19:54:18+08:00
draft: true
author: Zijian Zang
toc: true
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

### 鉴权机制

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

在**Solaris**（Solaris 8）中，*appropriate privileges*指的是进程euid为0，即root。而**Free BSD**（Free BSD 4.4）允许将real ID设置为effective ID，因此当`newuid==geteuid()`时，`setuid(newuid)`符合*appropriate privileges*。euid为root也是符合的。值得一提的是，Free BSD没有定义`_POSIX_SAVED_IDS`，但它确实有saved ID。为了符合POSIX标准，`setuid`不会修改saved ID。

**Linux**提供了一种细颗粒度的鉴权方法（Capability Model）。Linux使用多个*capability bits*决定对资源的访问控制。其中，`SETUID`比特用于判断进程是否具有POSIX定义下的*appropriate privileges*。实际上，Linux系统本身也是依据euid来判断的，它追踪uid的变动，当euid为0（root）时，`SETUID`为1，否则为0。虽然如此，`SETUID`本身是可以被修改的。一个进程可以关闭自身的`SETUID`，而一个具有`SETPCAP`的进程可以修改其他进程的`SETUID`。

### 系统调用的区别

`setresuid`是现代类Unix系统为了控制saved user ID而设计的系统调用，它可以同时修改三个uid。`setresuid`的权限要求为，要么该进程的euid为root，要么三个调用参数都符合，其等于该进程的三个uid之一（即uid之间可以相互赋值）。该系统调用确保一致性，即要么所有uid都生效，要么全不生效。`setresuid`的系统调用没有歧义。

`seteuid`也是比较清晰的系统调用。其唯一在各个操作系统间不统一的地方是，当euid不为root时，判断进程是否具有权限的标准不一致。Solaris与Linux允许euid被设置为任意一个进程自身的uid，而Free BSD不允许euid被设置为euid自身。即，`seteuid(geteuid())`是会导致无权限错误的。

`setreuid`系统调用修改real user ID和effective user ID，但某些情况下也可能修改saved user ID，具体规则十分复杂。不同操作系统之间的权限规则可能不同。Solaris和Linux允许ruid和euid互换，而Free BSD**可能**会报错。

`setuid`作为唯一一个进入POSIX规范的系统调用，也是语义最混乱的系统调用。在权限上，正如POSIX规定的一样，Solaris和Linux都要求newuid等于ruid或suid。也就是说，`setuid(geteuid())`会失败，这一设计本身就很奇怪。然而，在Free BSD上，`setuid(geteuid())`是可行的。同样的，在系统调用的效果方面，Solaris与Linux和POSIX规范一致，而Free BSD则总会同时修改三个uid。

`setfsuid`是Linux设计的用于管理文件权限的系统调用。fsuid基本上与euid保持一致，除非使用`setfsuid`另行设置。Linux旨在将fsuid设计为与基本的三个uid保持一致。只有在三个uid至少一个为root时，fsuid才能为root。然而，由于Linux的一个BUG（版本2.4.18），在调用`setresuid`成功时不会将fsuid设置为euid，可以构造出fsuid为root而三个uid都不为root的情况。

## 参考文献

[1] Hao Chen, David Wagner. Setuid Demystified.

[2] Portable Operating System Inteface for Unix(POSIX).
