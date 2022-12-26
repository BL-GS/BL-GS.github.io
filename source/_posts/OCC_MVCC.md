---
title: OCC & MVCC
categories:
- Transaction
tags:
- Concurrent Control
- Transaction
mathjax: true
description: Simple introduction about OCC and MVCC.
---

# OCC

**乐观并发控制协议 (Optimistic Concurrent Control, OCC)** 表示一种乐观并发控制的思想，只在事务提交时对事务是否符合串行化进行验证。

> 与之相对的是 **悲观并发控制 (Pessimistic Concurrent Control)**，它会在事务执行的过程中每个操作进行串行化验证。比较典型的是 2PL 机制


&nbsp;

## Optimization Consideration

在 OCC 提出来的年代，比较流行的是 2PL，针对这一并发控制协议，主要有如下的问题：

1. 加锁协议的开销大，对于只读事务也要加读锁保护。
2. 可能产生死锁，需要进行死锁预防或者检测和破除。
3. 持有锁的同时可能进行阻塞操作（I/O），降低了系统并发吞吐
4. 加锁事务回滚时也需要加锁，直到事务回滚结束


&nbsp;

## Overview

OCC 将事务分为三个阶段：

- `read phase`：读取记录，存储本地副本，修改副本上数据
- `validation phase`：冲突检查，若存在冲突则 Abort
- `write phase`：事务提交，将更新持久化

用时间戳排序的技术来决定可串行化次序。


&nbsp;

## Implementation

为每个事务涉及的数据创建一个私有副本，所有的更新都在该副本上执行，最后提交时通过判断是否有冲突来决定是否能够提交事务。

同时为了执行有效性检查，需要直到事务的各个阶段的开始时间，因此定义三个时间戳

- **StartTS($T_i$)**：事务 $T_i$ 开始的时间
- **ValidationTS($T_i$)**：事务 $T_i$ 完成其读阶段并开始其有效性检查的时间
- **Finish($T_i$)**：事务 $T_i$ 完成其写阶段的时间

令 TS($T_i$) = ValidationTS($T_i$)，通过时间戳排序的技术来决定可串行化的次序。



### Read phase

该阶段将事务涉及的数据复制一份副本，放到本地工作空间，之后执行读操作读取本地的数据，写操作将被记录到本地工作空间的临时文件中。

此时事务尚未提交，这些数据对其他事务不可见。



### Validation phase

当事务准备提交时进入校验阶段，首先检查在此期间事务是否与其他事务产生冲突，如果没有产生冲突，则事务顺利提交；如果产生了冲突，则中止事务。

事务 $T_i$ 的验证，要求满足 TS($T_k$) < TS($T_i$) 的所有事务 $T_k$ （事实上只需要考虑在该事务开始后仍然活跃的事务）必须满足下面两个条件之一：

- FinishTS($T_k$) < StartTS($T_i$)。

	因此 $T_k$ 在 $T_i$ 开始之前就完成了其执行，故可串行化次序是可保证的。

- $T_k$ 所写的数据项集合与 $T_i$ 所读的数据项集合并 **不相交**，并且在 $T_i$ 开始其有效性检查阶段之前，$T_k$ 就完成了其写阶段 StartTS($T_i$) < FinishTS($T_k$) < ValidationTS($T_i$)

	这个条件保证 $T_k$ 和 $T_i$ 的写不重叠。因为 $T_k$ 的写不影响 $T_i$ 的读，又因为 $T_i$ 不可能影响 $T_k$ 的读，从而保证了串行化次序。

> 有效性检查的机制自动预防了级联回滚，因为只有发出写操作的事务提交之后实际的写才会发生。



### Write phase / Commit phase

如果成功通过校验阶段，那么该阶段将会把本地工作空间中的数据永久地写到数据库或存储中持久化存储，完成整个事务。对于只读事务，则忽略该阶段。


&nbsp;

## Advantage & Properties

OCC 适合冲突率较低、短事务的场景。OCC仅支持包括{read-only, update} transaction, 很显然, 不支持多次交互的conversational事务.

* **读写冲突**：写入的数据没有提交前是不可见的，所以读取的还是写入前的数据，所以没有冲突
* **写读冲突**：读的事务在验证阶段会发现自己读的数据被最近提交的事务修改过，读事务会回滚
* **脏读**：未提交的数据不会被其他事务读到，因此不会发生脏读
* **写写覆盖**：事务的顺序是按照提交的顺序决定的，先开始的事务先提交，后开始的事务后提交，因此不会发生先开始的事务覆盖后开始的事务数据的问题


&nbsp;

## Problem

乐观并发控制多数用于数据竞争(data race)不大、冲突较少的环境中，这种环境中，偶尔回滚事务的成本会低于读取数据时锁定数据的成本，因此可以获得比其他并发控制方法更高的吞吐量。

* 可能导致长事务饿死

	原因是短事务可能会经常和长事务冲突导致其崩溃，可以通过暂时阻塞冲突事务来解决饥饿问题。

* 每次读取数据都需要把数据拷贝到本地空间，读取效率较低

* 并行度不高


&nbsp;

## Optimizations

OCC 的原始版本中，三个阶段 `read phase`、`validation phase`、`write phase`。其中 read phase 允许并行进行，而 validation phase 和 write phase 作为整体串行进行。只需要检查 RAW 依赖。

* Read/Write 并发、Validation 串行

	需要检查 RAW 和 WAW（写覆盖问题）依赖。可参考 [OCC系列之二: 两种简单的实现 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/35098431)

* Validation 并发

* Validation 进一步并发（通过 CAS 指令或者其他策略）

同时对于冲突的检查，也有不同的做法：

- 检查 read set 和 write set
- 只检查 write set，如 Omid
- 只检查 read set，如 Silo 和 TicToc


&nbsp;
&nbsp;

# Silo & TicToc

Silo 和 TicToc 都是在 OCC 的基础上实现 Validation 并行化的 OCC 变种。当然在实现上会有其他的优化，此处仅就其事务逻辑进行分析。

&nbsp;

## Overview

将事务操作的对象记录视为一个四元组 (key, value, tid, lock)

- `key`：tuple 的主键
- `value`：tuple 的值
- `tid`：最后一次更新该记录的事务 TID
- `lock`：锁

在 read phase 和其他 OCC 协议类似

在 validation 阶段，具有较为宽松的事务 TID 分配机制，只保证修改同一条记录的两个事务的 TID 不同即可。

同时需要做一些额外的依赖检测


&nbsp;

## Implementation

### TID Allocation

对于 Silo 中，只需要满足：

- 当前 epoch 分配的时间戳一定小于下一个 epoch 分配的.
- 同一个工作线程中, 分配的时间戳严格单调递增.
- 修改同一条记录的事务, 时间戳严格单调递增.

对于 TicToc 中，只需要满足：

- 时间戳分配, 由记录的 WTS 和 RTS 推算出, 只能保证偏序关系.
- write set无交集的两个事务的时间戳无法反应修改的前后顺序.



### Validation

- 对 write set 中所有记录上锁。需要注意的是，可以按照 key 的顺序进行加锁，避免死锁。
- 检查 read set 中是否存在被其他线程修改过的痕迹，假如有就崩溃。
- 分配 TID 并且提交

伪代码如下[4]：

```c++
// input: write_set, read_set;

// VALIDATION PHASE
// lock record in write set one by one in primary key order
for r in sortedByKey(write_set) {
    lock(r)
}

valid = true
for r in read_set {
    // anti-dependency conflict check
    if isModifiedOrLockedByAnotherTransaction(r) {
        valid = false
        break
    }
}

tid = -1
if valid {
    // allocate globally unique TID
    tid = allocateTID()
} else {
    // abort and release all lock
    for r in write_set {
        unlock(r)
    }
    abort()
}

// WRITE PHASE
for r in write_set {
    //establish new value and release locks.
    install(r, tid)
    unlock(r)
}
```


&nbsp;

## 问题[4]

1. 锁已被其他事务持有而无法授予当前事务. 当前事务可以选择wait和no-wait两种策略. 可能造成饥饿问题。
2. 记录锁的粒度太小导致开销过大. 每条记录都有一把锁, 当记录数量极大时, 维护锁需要的内存就越大。
3. 如果更新事务的write操作touch很多记录, 在竞争激烈的情况下, 冲突率上升导致延迟增大或者abort rate上升.
4. install新副本的延迟过长影响并发性能, 所以这种方式不利于磁盘数据库, 而且日志落盘也应该采用group commit, 应该采用多个日志, 在内存中保存多个log tail, 减低并发事务竞争写log tail引起的同步代价.
5. 分配 TID 可能需要原子操作



&nbsp;
&nbsp;


# MVCC

OCC 每次读取都需要把数据拷贝到本地空间，为了解决这个问题，MVCC 通过对一个 Logical tuple 存储多个版本的 Physical tuple。

> 需要注意的是这里的 Logical 和 Physical 关系。对于一个关系 tuple，MVCC 中会有多个版本的 tuple 与之对应，但是逻辑上只有一份关系。因此 Logical-Physical 是一对多的关系。

&nbsp;

## 优化方向

* 写不会阻塞读（在 OCC 中写数据需要是原子操作，否则对写一半的 tuple 读会出现未定义现象）
* 只读事务无需上锁，就可以获得一致性的快照
* Multi Version 只是一个思想，可以和 2PL、Timestamp-Order、OCC相结合

每个写操作会创建对应数据项的新版本，读操作则会选择数据项的一个版本进行读取       


&nbsp;

## MVTO

对于每个事务 $T_i$，将一个静态的唯一性时间戳与之关联，记作 TS($T_i$)，在**事务开始执行时赋予该时间戳**

每个数据项 Q，有一个版本 \<Q1, Q2, ..., Qm\> 与之关联，每个版本包含三个字段：

- **Content**：Q 在该版本的值
- **W-timestamp(Q)**：创建该版本的事务的时间戳
- **R-timestamp(Q)**：所有成功读取过该版本的任意事务的最大时间戳



令 Qk 为写时间戳**小于等于 TS($T_i$) 的最大写时间戳** 的版本。

### 写操作

* 若 TS($T_i$) < R-timestamp(Qk)，则系统回滚 $T_i$
* 若 TS($T_i$) = W-timestamp(Qk)，则系统覆盖 Qk 的内容
* 若 TS($T_i$) > R-timestamp(Qk)，则创建一个 Q 的新版本。该版本中保存其值，同时将 W-timestamp(Q) 和 R-timestamp(Q) 初始化为 TS($T_i$)。

在一个事务太晚执行写操作时将其终止，也就是 $T_i$ 尝试写入一个其他新事务可能读取的版本。

### 读操作

读取数据项 Qk 的内容且 TS($T_i$) > R-timestamp($Q_k$) 时，系统就更新 $Q_k$ 的 R-timestamp 的值。

保证事务读取时间上在它之前的最新版本数据

### 提交阶段

释放所有的锁。

### 回收数据

对于两个数据项版本，假如都满足：时间戳小于当前最老事务时间戳还小的版本的时间戳，则删除其中较老的

### 解决的问题

- **读写冲突：**写会创建一个新的数据版本，因为 read-timestamp < write timestamp，所以会读到写之前的数据，因此不会发生冲突
- **写读冲突：**读会更新 read-ts，写的事务会发现自己的 timestamp < read-ts，因此会写失败
- **脏读：**通过创建新的版本来实现写，因此一个事务写回滚，不会影响另外一个事务读
- **写写覆盖：**写写通过加锁进行互斥访问，先获得锁的事务写入成功


&nbsp;

## MV2PL

一个数据项的每个版本有一个单独的时间戳，记为 ts-counter，它再提交处理期间递增

### 更新事务

更新时执行强二阶段锁。也就是说，持有全部需要更新数据项的锁直到事务结束。

* 读取一个数据项时，获得数据项上的共享锁，读取该数据项的最新版本
* 写一个数据项时，获得该数据项上的排他锁，创建一个该数据项的新版本并上锁，在新版本上做写操作。

### 只读事务

对于只读事务，返回一个写时间戳 **小于等于 TS($T_i$) 的最大写时间戳** 的版本数据。只读事务不需要等待加锁。

### 提交阶段

对于更新事务，**同一时间只允许一个更新事务提交**，将每个版本的时间戳设置为 ts-commit + 1，然后将 ts-commit 加上 1，提交。

对于只读事务，将看到 ts-commit 的旧值直到 $T_i$ 成功提交，即只能够读到当前事务开始之前成功提交的数据。

释放所有读写锁。

### 回收数据

类似 MVTO



### 解决的问题

- **读写冲突：**读写谁先加到锁，谁成功
- **写读冲突：**读写谁先加到锁，谁成功
- **脏读：**读取的都是commit的数据，不会发生脏读
- **写写覆盖：**写锁互斥方式


&nbsp;

## MVOCC

### 读操作

返回一个写时间戳 **小于等于 TS($T_i$) 的最大写时间戳** 的版本数据，将其加入读集合。

### 写操作

找到该数据项的最新版本 Qk，如果其未上锁，则获取数据项的锁，为其创建一个新版本并上锁，将其加入写集合。否则崩溃。

### 验证阶段

获取事务 commit-ts（有并发控制器提供的最新时间戳），决定了其串行化的次序

检查读集合中的数据项是否被已经 commit 的事务所修改，如果冲突就回滚。

### 提交阶段

把修改过的数据时间戳设置为 commit-ts。这时候才对其他事务可见。

### 回收数据

类似 MVTO

### 解决的问题

- **读写冲突：**因为有多版本，所以后面的写不会影响前面的读，不会冲突
- **写读冲突：** 虽然读取了老的数据，但是会在validataion的时候发现有事务已经更新了数据，因此读会回滚
- **脏读：**只会读取commit的数据，因此不会出现脏读
- **写写覆盖：**写写通过加锁进行互斥访问，先获得锁的事务写入成功




&nbsp;
&nbsp;



# Reference

[1] 《数据库系统概念 第7版》

[2] [乐观并发控制 - 维基百科，自由的百科全书 (wikipedia.org)](https://zh.wikipedia.org/wiki/乐观并发控制)

[3] [多版本并发控制 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/168672853)

[4] [OCC系列之一: Validation-based Concurrency Control - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/35097664)



