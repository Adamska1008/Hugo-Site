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

1. 解决 *read amplification*：原本的SST只能做到内部有序，压缩后的SSTs保证整体有序，key_range不会重叠，即使SST数量较大，二分查找也能较有效地找到对应的SST。
2. 解决 *space amplification*：只保留最新的记录（通过MergeIterator自动完成），对于空值直接删除（MergeIterator无法做到，需要额外判断）。

`MiniLSM`在创建时会建立一个压缩线程来专注于压缩，我们不需要关注它，只需要关注实现压缩算法本身即可。

### Task1

Task1只需要实现两个方法：`compact`和`force_full_compaction`，前者是算法实现，后者则包括一些功能性代码。

首先从`force_full_compaction`讲起，它负责读取需要压缩的SST（本Task中为全部），调用`compact`获取压缩后的SST，并更新SST。这意味着该方法需要在调用`compact`各拿一次锁，第一次是拿读锁，第二次则是写锁和`state_lock`锁，因为我们既不希望向SST中再写入数据，又不希望发生MemTable的freeze或SST的flush操作。

读取所有SST并不难。更新时，要注意兼顾多个成员变量，包括从l0中移除压缩前SST的ID索引，从`sstables`变量中移除对应SST，直接覆盖`levels[0]`。

`compact`是核心压缩算法，但也并不难实现，因为可以利用之前实现的`MergeIterator`来完成排序和去重的操作，只需要自行添加删除的功能即可。由于`CompactionTask`是Serializable的，传入`compact`的参数只能是id不能直接是SSTs本身，故在内部还要通过一个读锁来获取SSTs。

压缩算法维护一个MemTable作为临时容器，而原本的SST全部获取迭代器后放到MergeIterator中，不断取出key/value直到用尽迭代器。如果发现值为空说明被删除了，跳过即可。如果超过了sst_size限制，则写入到一个SST中，并放到新创建的SST数组内。

### Task2

Task2与压缩算法并无直接关系，我们需要实现一个`ConcatIterator`，它与`MergeIterator`相对，后者实现了为多个有序Iterator保证整体有序的机制，同时完成了去重工作，而前者则简单许多，它只会依次迭代内部的每个Iterator，但也因此没有MergeIterator那么大的计算量，并且只会在内存中保留一个SST以及其正在维护的Block，内存占用也非常少。当使用`ConcatIterator`的`create_and_seek_to_key`方法时，它假定每个内部迭代器不仅内部有序，而且整体有序，因此会通过读取内部SstIterator的`first_key`来快速定位到对应`SST`中。

`ConcatIterator`本身实现起来比`MergeIterator`简单。需要额外注意的是，为了保证它内部的一个SST访问完之后切换到下一个SST，需要实现一个`skip_invalid`方法并在`next`方法中调用。

### Task3

`ConcatIterator`的作用是，对于某一个level的SST（除了L0），可以直接放到一个`ConcatIterator`里（因为compact保证除了l0的SST都全局有序），此时的读取效率是比较好的，远好于l0 SST。Task3的任务就是应用好之前写的两个组件，修改`get`和`scan`方法在读流程中加入其他level的SST。本例中只考虑l1即可。

`get`中，由于l1 SST是有序的，可以使用`ConcatIterator`包裹l1的所有SSTs，并调用`create_and_seek_to_key`方法。或者也可以与l0 SST用相同的写法，在功能性上不会有什么问题。

`scan`中，则需要在已有的TwoMergeIterator上多加一层，形成`TwoMergeIterator<TwoMergeIterator, ConcatIterator>`的结构，同时修改`LsmIteratorInner`的定义。当后续添加多层level时，还将改成`TwoMergeIterator<TwoMergeIterator, MergeIterator<ConcatIterator>>`的结构，亦可现在直接实现。
