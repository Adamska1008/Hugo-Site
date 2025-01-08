---
title: 'MiniLSM lab Week1'
date: 2025-01-02T20:22:25+08:00
draft: true
author: Zijian Zang
UseHugoToc: true
tags: 
 - database
 - rust
categories:
 - 实现MiniLSM
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

## Week1Day2

Task1同样是比较好理解的。主要难点在于理解Iterator的自引用设计，`ouroboros`库做了什么，如何使用其提供的方法。判断`MemTableIterator`是不是合法，只检查键是不是空，不能检查值。

Task2是实现MergeIterator，这是Day2的难点。MergeIterator的任务是合并几个有序序列为一个有序序列，刷leetcode多的应该不陌生。一种简单方法是每轮比较所有序列头部，取出最小值，这样每轮比较的时间是O(n)。更好的办法是使用堆维护这些序列，堆顶即为最小值。同时，`MergeIterator`维护一个`current`的键值对，作为当前迭代的对象，而不是每次都访问堆顶。

除此之外，Task2还需要解决一个问题：当有新值时，不采用旧值。实现方法是在检查堆顶时，若其当前`current`的`key`，则直接后移堆顶迭代器并整理其在堆中的位置，继续检查堆顶的值。上述内容的简单伪代码如下：

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

Task3要实现LsmIterator和FusedIterator。前者需要注意，应当跳过被删除的键值对（值为空），在MergeIterator层级没有检查这一点。后者需要注意`has_errored`后，不应当允许用户访问内部迭代器的方法。因此总是要先检查`has_errored`。

Task4的实现就比较显然了。在前面的实现正确的情况下，应该不会有什么问题。

## Week1Day3

Day3整体的任务是比较少比较简单的。主要聊聊设计思路。

Block是SST存储的最小单位，其设计面向磁盘读写。虽然如此，在encode/decode时，并不会解析key/value数据本身，data_session在Block中全程是以纯粹的字节表示的。反序列kv数据的功能完全被放在了迭代器中，由迭代器实现。

这一点在builder中也是同样实现的。builder并不会维护一个`Vec<(KeySlice, &[u8])>`，而是在添加键值时直接做好序列化，并存储到data中。

## Week1Day4

Task1的任务是要实现SSTBuilder。其实现过程并不复杂，只谈几个技术点：

1. 在将block写入SSTBuilder的data区时，可以使用`std::mem::replace()`处理需要move的成员变量。由于方法签名是`&mut self`，我们不能直接move成员变量。
2. 使用`bytes`提供的`Buf`和`BufMut`trait。有函数签名
   ```rust
   /// Decode block meta from a buffer.
   pub fn decode_block_meta(buf: impl Buf) -> Vec<BlockMeta> {
         unimplemented!()
   }
   ```
   `bytes`为一些类如`Vec<u8>`实现了`Buf`trait，我们可以直接从buf中读取`u16`等数据，不需要手动操作。我们也可以将encode方法中的buf类型改为`impl BufMut`而非使用原本的`Vec<u8>`
   ```rust
       /// Encode block meta to a buffer.
    /// You may add extra fields to the buffer,
    /// in order to help keep track of `first_key` when decoding from the same buffer in the future.
    pub fn encode_block_meta(block_meta: &[BlockMeta], buf: &mut impl BufMut) {
        for meta in block_meta {
            buf.put_u32_le(meta.offset as u32);
            buf.put_u16_le(meta.first_key.len() as u16);
            buf.put(meta.first_key.raw_ref());
            buf.put_u16_le(meta.last_key.len() as u16);
            buf.put(meta.last_key.raw_ref());
        }
    }
   ```
   如此实现可以说是比较方便的。Day3的一些操作也可以使用这两个`trait`来完成。

Task2的实现本身技术难度也不大，但是需要理清各类的职责。需要注意的一个细节是，find_block_index的返回值只确保`first_key <= key`，不能确保`key`就在这个block里。由于我们找的是第一个大于等于`key`的block，当`block`内部都找不到时，应当直接移到下一个block开头；如果这个block是最后一个则不动。二分查找可以直接使用`partition_point`方法。

Task3更是简单。唯一的小问题是如何处理`try_get_with`返回的`Result`。作者的解法是

```rust
let blk = block_cache
      .try_get_with((self.id, block_idx), || self.read_block(block_idx))
      .map_err(|e| anyhow!("{}", e))?;
```

看上去不是很好，但是不清楚有无其他解法。

## WeekDay5

Task1是一个类似于双有序列表合并的问题，区别在于对于同一个键只需要一个值。这种问题一般来说难度不大，但是需要注意边界检查。`key()`和`value()`方法都要注意边界检查，`is_valid()`方法则是需要检查两者都合法。`next()`方法稍微复杂一点，因为和`merge_iterator`一样，我们希望跳过a、b中都存在的键。做完边界检查后，需要判断a与b的key是否相等。不相等的情况下迭代小的那个。相等的情况下则需要都迭代一次。

为什么这个`next()`的逻辑是对的呢？我们可以考虑各种情况：如果a和b的key不相同，由于对称性只考虑a的key较小的情况，则现在的key就是a的key，下一个key无非是a的下个key或b的下个key（或者二者相同），直接迭代a后，获取key就是比较上述两者，是合理的。如果a和b的key相同，则下一个key是a的下一个key或b的下一个key，因此需要迭代二者。

Task2，我看了一下，我的实现和官方实现差距较大。文档说可以修改LsmIterator为SsTableIterator提供end_bound，但是我考虑到MemTable实现了`scan`，也希望在SsTable上实现`scan`方法。但是SsTableIterator内部有`Arc<SsTable>`，在不借助外部库的情况下clone自身为Arc比较复杂，故改为实现`SsTableIterator::scan`的静态方法。实现起来比较自然。

Task3更是没什么好说的。
