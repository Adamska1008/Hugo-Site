---
title: 'MiniLSM lab Week2'
slug: 'MiniLSM lab Week2'
date: 2025-01-16T17:29:31+08:00
draft: false
author: Zijian Zang
toc: true
description: 关于MiniLSM Lab Week2的一些想法
image: cover.png
tags: 
 - database
 - rust
series:
 - minilsm-lab
categories:
 - minilsm-lab
keywords:
 - minilsm
 - lsm
 - database
---

Week2的围绕compaction展开。虽然Week1已经实现了LSM基本的功能，但是目前的实现会在性能上有很大的问题。

首先，SST中的数据会随着增删改不断增长，因为LSM处理删除的方式是加入一条空数据。虽然查找时，只需要根据顺序查找最新数据即可，但是空间上的增长是无法避免的，这被称为 *space amplification*。

其次，存在 *read amplification* 的问题，原文在此处的讲解比较好。简单来说，我们希望SST的key_range能够尽量不重叠，这样是有利于查找效率的。

*压缩（compaction）* 是一种操作，读取一些文件中的所有数据，进行一些处理（通常对读取性能有益）并写回磁盘且删除旧数据。它可以缓解读数据的效率，但是本身需要占用写入资源，这被成为 *write amplification*，因此需要权衡压缩的频率与力度。

## Day1

在Day1，我们需要实现一个简单的压缩策略：全部压缩（Full Compaction）。实际上就是把所有的SST全部重排，并写入到level1中。它主要做两件事：

1. 解决 *read amplification*：原本的SST只能做到内部有序，压缩后的SSTs保证整体有序，key_range不会重叠。
2. 解决 *space amplification*：只保留最新的记录（通过MergeIterator自动完成），对于空值直接删除（MergeIterator无法做到，需要额外判断）。

`MiniLSM`在创建时会建立一个压缩线程来专注于压缩，我们不需要关注它，只需要关注实现压缩算法本身即可。

