---
title: 'Thesis Reading: CoroBase: Coroutine-Oriented Main-Memory Database Engine'
categories:
- Transaction
tags:
- Thesis
- Coroutine
- Transaction
description: 'Thesis reading: CoroBase: Coroutine-Oriented Main-Memory Database Engine'
---

# Introduction

主要着眼于当前数据库中的问题：Data Stall。

对于一些依赖于指针操作的数据结构来说（也就是访问内存不连续的数据）， Cache miss 会导致很多的 Data Stall。

现代处理器来具有多层 Cache 架构和预取（prefetch）指令可能会缓解这些问题，但是硬件预取等做法对于灵活的指针操作到底不够精确，需要手动的异步或者流水线算法或者状态机来运用。

协程具有相对于 last-level cache miss 而言较低的切换开销，但是软预取 (software prefetch) 也具有一些挑战：

- 很多现在的事务逻辑采用 **内部聚簇** (intra-transaction batching)的做法。这种做法提供多键值接口，破坏了向后兼容性 (backward compatibility)。换句话说，给的接口的操作粒度太大了，一个事务里不能单独控制一两个数据的读写。

	需要考虑 **如何在不做应用层修改的条件下，在数据库系统中实现协程（软预取）**

- 评估软预取会对一个完整的数据库引擎带来怎样的好处



> 感觉对于 Data Stall 问题，假如想要用 Coroutine 来解决，那应该得用 Stackless Coroutine，因为 Stackful Coroutine 的协程切换会导致很多缓存失效。



它在协程上应用软预取的思路是，对应一个线程，我批处理一组事务，然后每次做 **索引** 或者 **读写** tuple 时，发出预取指令然后切换协程。

总体而言，我感觉这是提升响应速度、总的吞吐量的，但是会降低单个事务的执行速率（也就是事务的延迟提高了），不适合需要快速完成的场景。

&nbsp;
&nbsp;

# Implementation

## Transaction

它基于现成的基于 MVCC 的数据库引擎 ERMIA 做文章，将其转化为协程-线程的模式。当然做了一点修改。

![Modification on ERMIA](/images/Thesis CoroBase/Modification.png)

找数据项的步骤是：

1. 线程通过索引找到一个间接数组，通过这个间接数组上的指针找到对应的实际数据记录
2. 通过搜索版本链得到合适的数据版本

> snapshot isolation with the serial safety net [60] for serializability
>
> 感觉有点意思

类似于ERMIA, CoroBase采用串行安全网(serial safety net, SSN)，可以应用在快照隔离实现可串行性。在协程环境下无需修改

对于物理层的数据结构（索引和版本链），当使用 latch 时会有一点问题，死锁之类的。

因此它数据结构使用乐观逻辑，不使用 latch；只在树节点需要更新时上锁（hand-over-hand locking)。它发现这时候协程 suspend 没啥用，所以就没用

&nbsp;

## Fully-Nested Coroutine-to-Transaction

每个线程拥有 thread local 的协程调度器，在同一个 batch 的事务中调度，直到当前 batch 内事务全部完成（为了保持局部性）。

主要想法是将那些会导致 cache miss 的函数调用链转化为无栈协程，那些不会影响 cache miss 的函数则保持原状

> 需要注意的是，在 C++20 中协程的实现是会修改函数的调用方式的，当“普通地”调用一个协程函数（实际上调用协程函数地需要有协程框架），会有一些额外地消耗。

- 对于 **索引**，在每次 **预取之后** 使用 co_await 进行 suspend
- 对于 **版本链搜索**，在解引用一个链表节点前作预取和 suspend

当 suspend 后直接返回 scheduler，转而去作那些 batch 中未完成的其他事务。

&nbsp;

## Two-Level Coroutine-to-Transaction

数据库引擎较深的函数调用层次，可能会导致一些切换的代价比较大，毕竟不是每次解引用指针都会导致 cache miss。同时又要保证各种逻辑的分离（这种强迫症很合我心~~）

所以它将数据库引擎中的函数调用展平了，也就是 inline，构成了两层的结构。（其实我感觉用个模板函数和偏特化就行了，简单又易看）。一些顺序执行的不会造成 suspend 的函数就不需要 inline 了。

展平可能造成指令的 cache 失效，因为一些重复使用的代码缓存了没用上。论文中说可能乱序执行会有影响，但我感觉毕竟顺序执行，指令预取也不是不可以，但是费电费时间还是有点。

&nbsp;

## Resource Management

协程的多个函数入口导致资源分配回收的时间不是太明确。

### Epoch-based Reclamation

对于一些无锁数据结构而言，在并发的情况下，有一些数据项虽然被删除了，但是它的内存还有可能被访问。

Epoch-based Reclamation 的基本思想是为每个线程在访问的内存上**注册**，然后保证那些要被删除的数据项在所有注册线程完成事务后释放空间（就是引用计数）。

在 thread-to-transaction 的情况下，一个事务使用的资源跟其线程是一致的。这个时候事务和线程的释放资源可以是同步的。

但是在 coroutine-to-transaction 的情况下，会出现一个线程上一个 batch 中多个事务共同使用同一个数据项的情况。这个时候就需要整个 batch 都做完再释放资源。

### Thread-Local Storage

thread-local 的资源和 coroutine 有些冲突，毕竟本来是事务自己拥有的现在要一个 batch 的事务一起共享。

因此把资源改成 transaction-local 

&nbsp;

## Discussion

- 协程还不错，不需要修改太多东西
- 对于 shared-everything 的部分（索引、数据项），使用乐观并发控制协议的多版本机制挺搭。虽然添加可串行性会扩大冲突。
- 对于悲观锁，协程可能造成更高的死锁概率。
- shared-nothing 的系统分区（thread-local的东西），不需要同步和并发控制。但是引入协程需要考虑一些东西（比如改成 transaction-local）
- 对于一个事务拆分成几部分，分给不同线程来做，就能达到一个很好的局部性和无冲突化。但是对于协程来说，仍需考虑。
- 它移除了 multi-key operation 的需要，但是仍然支持这种操作？？当做了多值操作时，访问一个 key 引发的 suspend 可能不会导致返回 scheduler 而是去到其他 key 的访问处。

&nbsp;
&nbsp;

# Reference

[1] Yongjun He, Jiacheng Lu and Tianzheng Wang. [CoroBase: Coroutine-Oriented Main-Memory Database Engine](http://www.vldb.org/pvldb/vol14/p431-he.pdf). VLDB 2021.

