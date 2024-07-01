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

Users ID与Groups ID在进程中用于决定进程访问资源的权限。有时进程需要额外的访问权限，Unix系统提供了*uid-setting system calls*来提权与降权。

一个进程实际上会存储三个uid：*real user ID(ruid)* 用于标识进程拥有者、*effective user ID(euid)* 用于访问控制、*saved user ID(suid)* 用于保存过去的uid，以便恢复。特别的，Linux中进程还拥有*file system user ID(fsuid)* 用于文件的访问控制。当调用setuid系列系统调用时，操作系统首先检查进程是否有修改uid的权限。如果有，则根据调用setuid的具体规则修改uid。

由`fork`创建的进程会继承父进程的uid。对于由`exec`执行的文件，如果它的`setuid`位为0，则继承调用者进程的uid，否则其euid与suid将被设置为该文件拥有者的uid。后者是由贝尔实验室提出的一种策略，其在1979年被授予专利。



## 参考文献

[1] Hao Chen, David Wagner. Setuid Demystified. 