---
title: 'MiniLSM lab'
date: 2025-01-02T20:22:25+08:00
draft: true
author: Zijian Zang
UseHugoToc: true
tags: 
 - database
 - rust
---

<!--more-->

## Week1Day1

Week1Day1包含四个Task。Task1并没有需要详细陈述的地方，主要问题是阅读文档了解`SkipList`和`Bytes`的基本使用方式。Task2也比较基础，但有一个细节问题：当put了一个空`Bytes`表示删除时，应该在MemTable层级就返回`None`吗？实际上不能这么做，因为需要区分一个键究竟是还没有存放过还是被删除了，这一区别将用于判断是否需要继续在freezed tables中查找。MemTable对于删除操作应当如实返回，而在LsmStorageInner层级才通过返回`None`表示不存在了。

Task3是Week1Day1中最需要动脑的。在介绍需要做什么之前，lab先详细介绍了两个关注点：

1. 使用`state_lock`做更细粒度的并发控制。`state`本身就已经是`RwLock`了，为什么还加一把锁？`state_lock`用于需要freeze时，限制其他线程不做freeze操作。如果使用RwLock的写锁，则会面临一个问题：其实此时依旧允许其他线程来读取数据，因为在判断需要freeze和实际修改state之间还有一段真空区，后续可能做一些io操作。
   - 这一点被用于`put`的内部实现
2. 使用双重条件检查。双重检查如下：
   ```rust
   let lock = Mutex::new(false);
   let cond = true;
   if cond {
        *guard = true;
        if cond {
            // 执行操作
        } 
   }
   ```
   这是因为判断条件与上锁不是原子操作，如果没有内部条件判断，可能导致两个线程同时判断条件为真后都上锁，一个阻塞一个执行，执行完毕后另一个线程获取锁，再度执行一遍操作。内部条件判断用于在另一个线程阻塞后获取锁后，判断不满足条件从而直接跳出。
   - 这一点被用于`force_freeze_memtables`的内部实现。

同时，由于lab的代码定义`state`为`Arc<RwLock<Arc<LsmStorageState>>>`，即使获取了写锁`state`依旧是不可变的，只能创建新值。

Task4也是比较好理解的。前面说过delete的问题，在lab4中会遇到，如果不这么实现，会误判删除后的值是没有遇到的，进而向后查找freezed的表，返回错误的旧值。
