---
title: 'MiniLSM lab Week1'
slug: 'MiniLSM lab Week1'
date: 2025-01-02T20:22:25+08:00
draft: false
author: Zijian Zang
toc: true
description: 关于MiniLSM Lab Week1的一些想法
image: image.png
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

截至文章发布，我已经完成了Week1的内容。整体来说并不难实现，作者将任务的粒度分的比较细，大多数都为给定函数设计实现功能，一些可能踩坑的点也在介绍任务时讲明了。应该说，只要认真阅读任务要求，做起来是比较舒服的。本文主要简单概括一下每个任务的要求、作者没有详细介绍的知识点和基本的实现思路。

## Week1Day1

Week1Day1的主要任务是实现MemTable（Memory Table）相关功能。MemTable是key/value在内存中的存储容器，也是查找时首先访问的数据结构。活跃的MemTable只有一个，当活跃MemTable到达容量上限时，会被freeze，转化为Immutable MemTable，不会再写入数据，只会读它。这种不可变的MemTable后续将用于持久化存储。

MemTable没有删除方法。这是因为在LSM中，删除也通过插入来实现：插入一个特殊的Log（MiniLSM的实现为空值的Log）。虽然这一设计对于内存容器似乎很奇怪，但在后续持久化存储相关的功能中有着很大的用处。

### Task1

Task1并没有需要详细陈述的地方。主要需要了解两个依赖库的使用方法：`bytes`与`crossbeam-skiplist`。

`crossbeam`本身是一个常用并发编程的工具集。`crossbeam-skiplist`暂时是实验性质的，没有放在`crossbeam`项目中，因此需要单独引入。它实现了基于无锁跳表的Map和Set。无锁编程是一种基于原子类型的并发编程范式，不多赘述。得益于此，可以保证容器操作都是原子的，我们不需要在MemTable容器操作时上锁，并且允许`put`方法不可变地借用自身。至于使用跳表本身，应该没有特殊理由，正如redis作者对于为什么使用跳表的答复：比红黑树容易实现。

`bytes`是一个字节功能相关的库，核心类型`bytes::Bytes`指向一个可以被共享的内存块，允许自身被clone后仍然指向原地址，由此降低了clone的开销，实现零拷贝。它也提供了大量简便功能，后续会使用。

明白了这些基本知识后，实现`MemTable::get`与`MemTable::put`便比较简单，只要封装一下`SkipMap`的功能，处理一下`&[u8]`到`Bytes`的转换就行。

### Task2

Task2的工作是把MemTable放到LsmStorageInner中。已有的代码已经完成了LsmStorage的定义工作，其存储被存放在`state`成员中，`state`包含了存储键值对的所有数据结构，MemTable是其中的一部分。Task2只要求完成唯一活跃的MemTable相关代码。

基本上，Task2也只是封装一下刚刚MemTable实现的方法。但是LsmStorageInner内部是使用锁的。唯一活跃MemTable使用了读写锁，读写锁允许多个线程同时读取内容，或者单一线程写入内容。

>MiniLSM没有使用标准库的读写锁，而是使用了`parking_lot`库实现的读写锁。相较于标准库，`parking_lot`优化了接口与性能表现，同时实现了两个比较重要的feature
>1. 任务公平性。当有一个线程请求写时，会阻塞读请求，从而避免写者饥饿问题。标准库的实现与操作系统有关，在Linux下默认采用读者优先策略。
>2. `parking_lot`的锁在持有线程panic时不会进入中毒状态，而是会直接释放锁。

有一个细节问题：当put了一个空`Bytes`表示删除时，应该在MemTable层级就返回`None`吗？实际上不能这么做，因为需要区分一个键究竟是没有存放过还是被删除了，这一区别将用于判断是否需要继续在freezed tables中查找——如果没存放过就找，如果被删除了就应当返回不存在。

### Task3

Task3将 *冻结（Freezing）* 活跃的MemTable。冻结后MemTable将不可变，类似于历史记录，只能查询过去存储了什么key/value，不能修改。它的作用在于隔离写入活跃的区域和可以持久化存储的区域。

Freezing将在MemTable超容量后执行。那么，首先就需要维护已有key/value的空间占用，这就需要修改此前编写的`put`代码。注意到MemTable采用无锁编程的范式，我们维护`size`变量也应当使用原子类型。

接下来实现freeze的代码。方法调用链为`put() -> force_freeze_memtable()`。思路并不复杂，伪代码可以表示为

```rust
fn put(key, value) {
   state.memtable.put(key, value);
   if state.memtable.approximate_size() >= options.target_sst_size {
      force_freeze_memtable();
   }
}

fn force_freeze_memtable() {
   let sst_id = next_sstid();
   state.imm_memtable.insert(0, state.memtable);
   state.memtable = MemTable::new(sst_id);
}
```

似乎几行代码就能实现？但是实际上会复杂一点，因为需要考虑并发编程。Lab作者详细介绍了两处细节：

1. 使用`state_lock`做更细粒度的并发控制。从发现MemTable超出容量，到开始修改state，这一段期间（在未来）可以做许多操作，例如为MemTable创建预写日志（WAL）。我们希望做这些操作时，依然允许MemTable中读甚至写一些内容——即使超出设定容量，为了响应速度也是值得的。为了实现这一点我们需要第二个互斥量，它只防止二次freeze，但不限制通过RwLock进行读写操作。这就是源代码中`state_lock`的作用。
2. 使用双重条件检查。双重条件检查类似如下代码，会在上锁前后各判断一次条件。
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
   这有什么用？首先我们必须要根据条件才能上锁，所以第一个条件判断是必须的。那么，第二个条件判断的作用是什么？
   由于`if cond { *guard = true; }`不是一个原子操作，可能会有两个线程同时进行条件判断，同时判断为真，于是最终执行两次内部代码。通过第二个条件判断，我们保证在上述情况中，即使第二个线程也进入了内部，依然会在第二次判断中获得false，于是跳出，避免了执行两次操作。

此外，还有一点与rust本身的特性相关：state被定义为`Arc<RwLock<Arc<LsmStorageState>>>`。由于RwLock锁上的依然是不可变的`Arc`，即使获取了读锁，实际上也是不能改变LsmStorageState内部的，例如为imm_memtables添加内容。我们只能为`state`赋予新值。

```rust
let mut guard = state.write();
let mut new_state = guard.as_ref().clone();
// ...
*guard = Arc::new(new_state);
```

这种不可变性的编程思想在函数式编程领域使用较多，基于不可变状态，可以规避并发编程中许多资源竞争的问题。

### Task4

如果你顺利实现了Task3，那么实现Task4大概只是顺手的事。先查找memtable，然后从新到旧在imm_memtables中逐个查找即可。

## Week1Day2

Day2的任务是实现MergeIterator，从多个MemTable（包括一个活跃和多个不可变）中整合信息。MergeIterator也可用于其他实现了StorageIterator的类。

### Task1

Task1的描述中给出了StorageIterator这个trait的定义，后续实现功能时切记与定义一致。

首先需要实现在单一MemTable上做范围查找。有序的Map支持范围查找。对于含有n个节点，深度为h的Map，如果符合范围的有k个元素，则有序的Map可以在O(logn + k)，无序的Map则只能遍历全部，时间复杂度为O(n)。

跳表是有序的，可以做范围查找。`crossbeam-skiplist`的SkipMap提供了`range`方法，查阅文档即可实现。

Task1的主要难点在于学习操作自引用的类型，详细内容建议查阅原教学和`ouroboros`库文档理解。只需要学习如何使用，还是比较简单的。

### Task2

Task2是实现MergeIterator，这是Day2的难点。MergeIterator的任务可以看作是合并几个有序序列为一个有序序列，刷leetcode多的应该不陌生。一种简单方法是每轮比较所有序列头部，取出最小值，这样每轮比较的时间是O(n)。更好的办法，也是MiniLSM使用的方法是使用堆维护这些迭代器，堆顶是头部最小的迭代器。同时，MergeIterator总是从堆顶提取出最小值作为`current`成员变量，易于访问。

注意，此处的合并有序序列，与leetcode不同的地方在于：对于key相同的key/value序列，总是只取第一次访问到的（最新的）。旧的记录会被新的覆盖。实现方法是在`next`方法中，若堆顶迭代器的key与当前`current`成员的key相同，则调整该迭代器指向下一个key，并整理其在堆中的位置，继续检查堆顶的值，重复以上过程，直到堆顶的key与current不同。上述内容的简单伪代码如下：

```rust
fn next() {
   if !self.is_valid() {
      return;
   }
   while let top = heap.peek() {
      if top.key() == current.key() {
         top.pop(); // 注意是top的list pop，而不是heap pop。这个迭代器依旧是需要的。
         if !top.is_valid() {
            heap.pop(top); // 需要堆支持移除任意位置的对象
         }
      } else {
         break;
      }
   }
   current.next();
   if current.is_valid() {
      heap.push(current);
   }
   current = heap.pop(); // 将current放回heap中，从而避免手动比较current与堆顶。此处实现与官方给的实现不同。
}
```

### Task3

Task3要实现LsmIterator和FusedIterator，实际是实现FusedIterator，因为LsmIterator的功能需求，实现上的坑，要到Task4的测试中才能发现。

FusedIterator的目标是，记录`next`是否造成了迭代器非法（`!is_valid()`），如果非法则禁止调用`next`，会返回`Err()`枚举。关键是在每个方法中总是要先检查`has_errored`。

### Task4

Task4要实现scan，只要在每个MemTable上调用Task1实现的scan，并将得到的所有迭代器放在MergeIterator中就行。关键是LsmIterator中的一些细节。

由于我们在MemTable这个抽象并没有特殊处理被删除的键，因此必须在LsmIterator上处理它（对于单点查询，则是在`get`方法内部处理）。具体是在每一步操作过后，总是检查下一步的值是否为空，跳过所有空值。

## Week1Day3

Day3主要面向二进制字节流，需要实现持久化存储的最小单位Block。整体任务量较小。

### Task1

Task1需要实现BlockBuilder。Block的定义原文讲的很清楚。要向一个（未来的）Block中添加键值对，必须通过BlockBuilder进行。它在内部维护一个`data: Vec<u8>`，与Block中的data_section区保持一致。每次添加键值对，总是会立即将其序列化并放到`data`中。这样，当调用`build`方法时，不需要逐个处理键值对，只要将offset_section和num_of_elements处理好即可。

注意可以利用`bytes`提供的`BytesMut`trait来实现功能。`bytes`为`Vec<u8>`实现了这一trait，可以直接调用诸如`put_u16_ne`（ne表示natural ending，与操作系统一致）这样方便的方法，而不需要手动序列化u16到`[u8;2]`。

### Task2

实现BlockIterator，支持在一个输出好的Block上进行迭代。Block的定义非常简单，只包括原始二进制data_section和一个`offsets: Vec<usize>`成员变量，根据offsets查找并反序列化键值对并不困难。

## Week1Day4

Day4的任务围绕SST（Sorted String Table）展开。SST是底层存储单元，由Block组成。多层的SST是LSM数据在磁盘中存储的格式。

### Task1

Taks1需要实现SSTBuilder，除此之外还需要实现`Metadata`类的编解码。一个SST内部包含多个Block，而SSTBuilder维护一个正在添加的Block，以及一系列做好序列化的Block数据（`data`成员变量）。当这个Block到达容量上限后，应当将这个Block序列化并添加到`data`中。

同时，需要实现`Metadata`相关功能。`Metadata`描述了Block在SST中的存放方式，需要通过`Metadata`才能读取Block。原文没有给出`Metadata`的序列化实现细节，为了能够知道`first_key`与`last_key`的具体长度，我们需要在序列化时手动加入记录长度的字节。

### Task2

Task2为SST实现迭代器。注意，为了查找效率，我们应当对所有Block做二分查找。在Block内部，则是直接线性迭代。通过之前维护的`Metadata`，要进行二分查找并不困难，但需要注意的一个细节是，`find_block_index`（二分查找方法）的返回值只确保`first_key <= key`，不能确保`key`就在这个block里。由于我们找的是第一个大于等于`key`的block，当`block`内部都找不到时，应当直接移到下一个block开头；如果这个block是最后一个则不动。实现二分查找可以直接使用`partition_point`方法。

### Task3

原文实际上没有对Task3的功能做测试：毕竟使用cache主要是为性能而非功能做考虑。`read_block_cached`方法首先检查是否有`Cache`成员，若无则依旧使用平凡方法。`mock-rs`提供的缓存在查询时接受一个闭包函数，用于当缓存未命中时，加载数据。了解使用方法后不难使用。

## Week1Day5

之前分别实现了内存与磁盘中的数据结构，Day5的目标是结合二者，实现读取操作，具体包括重写单一查询`get`和范围查询`scan`。`LsmStorageState`内部维护了一个`l0_sstables`数组，用于保存`sst_id`的同时也维护顺序，越是靠前的SST越新。而`sstables`成员则是一个Map。要以从新到旧的顺序访问SST，应当先遍历`l0_sstables`，再通过id查找Map，而非直接查找Map，那是随机顺序的。

### Task1

Task1是一个类似于双有序列表合并的问题，区别在于需后插入的值应插入前插入的值，查询键应返回最新值。此类问题整体上难度不大，但是需要注意边界检查。`key()`和`value()`方法都要注意边界检查，`is_valid()`方法则是需要检查两者都合法。`next()`方法稍微复杂一点，因为和`merge_iterator`一样，我们希望跳过a、b中都存在的键。做完边界检查后，需要判断a与b的key是否相等。不相等的情况下迭代小的那个。相等的情况下则需要都迭代一次。

为什么这个`next()`的逻辑是对的呢？我们可以考虑各种情况：如果a和b的key不相同，由于对称性只考虑a的key较小的情况，则现在的key就是a的key，下一个key无非是a的下个key或b的下个key（或者二者相同），直接迭代a后，获取key就是比较上述两者，是合理的。如果a和b的key相同，则下一个key是a的下一个key或b的下一个key，因此需要迭代二者。

### Task2

Task2需要应用刚刚实现的TwoMergeIterator来包装两个MergeIterator，二者分别维护了一组MemTable与一组SSTable，由此实现`scan`方法。然而，我们在`MemTable`中可以直接调用`SkipMap`的`range`方法进行返回查找，而`SSTable`中并没有现成的方式，需要自己实现。

我的实现和官方实现差距较大。文档说可以修改LsmIterator为SsTableIterator提供end_bound，但是我考虑到既然MemTable实现了`scan`，根据对称性也可在SsTable上实现`scan`方法。但是SsTableIterator内部有`Arc<SsTable>`，在不借助外部库的情况下clone自身Arc比较复杂，故改为实现`SsTableIterator::scan`的静态方法。

为SSTable实现了范围查找后，只需要在`LsmStorageInner`的内部各自对每个SST调用一次，放到MergeIterator之中即可。

### Task3

Task3是实现`get`。唯一的改动是在`imm_memtable`找不到时，继续在SST中查找。

## Week1Day6

Day6的核心任务是实现从MemTable到SST的转化。我们之前已经实现了SSTBuilder，完成Day6并不困难。

### Task1

实现`MemTable::flush`只需遍历key/value，SSTBuilder逐个调用add方法。`force_flush_next_imm_memtable`结构上类似于`force_freeze_memtable`，但后者并不是在`get`中调用的，而是一个线程不断轮询条件来判断是否调用的。因此，`force_flush_next_imm_memtable`负责获取`state_lock`锁，它在方法开头需要同时获取两把锁。

### Task2

Task2要实现的`trigger_flush`是`force_flush_next_imm_memtable`的入口，用于条件判断。注意`trigger_flush`获取不可变内存表时要及时释放锁，以免导致后续死锁。

### Task3

为每个迭代器实现`num_active_iterators`方法。注意MergeIterator中，`current`自身也是一个额外的Iterator。除此之外，由于我的Day5实现与原文有较大差异，所以实际上不需要优化就能做到原文优化过的效果。