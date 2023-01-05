---
title: 'Thesis Reading: Lock-Free Locks Revisited'
categories:
- Parallel
tags:
- Thesis
- Lock
- Parallel
description: 'Thesis reading: Lock-Free Locks Revisited'
---

# Introduction

Lock-free 数据结构相对于传统有锁的数据结构会非常复杂，尤其对于一些比较复杂的数据结构比如 B-Tree 之类的。

另一方面，blocking 和 non-blocking 的算法性能依赖于运行环境，比如线程数、CPU核心数等

&nbsp;

## TSP-B

TSP-B 作为一种无锁算法的思想，主要想法是当一个线程获得锁后，它会留下一个描述字 descriptor，当其他线程也想获得该锁，首先会帮助该线程完成其工作。

使用这种方法，需要保证临界代码的 **幂等性 (idempotence)**，也就是多次运行该代码的结果跟一次运行的结果一样。因为在协助运行中可能会对同一段代码执行多次，因此保持幂等性是很重要的。

&nbsp;

## Idempotence

简单的理解，幂等性就是对于一系列操作，其多次执行的结果和单次执行的结果相同。

传统的保持幂等性的方法是保持一个上下文。但是那样效率不高，并且不是那么通用。

那么它的想法是对于这份合作执行的代码，维护一个共享的日志，然后跟踪对于共享数据的读取操作，包括内存分配之类的。一旦线程执行到了 loggable 的操作时，使用 CAS 操作将日志提交。第一个提交的值作为最后的结果。

&nbsp;

> **ABA 问题**
>
> 假设两个线程同时访问一个数据项，A 首先读出 1，然后 B 写入 0，最后 A 再次读得到 0。A 的两次读的结果是不一样的，当时就 A 的执行而言是抽象独立的（也就是不知道有其他线程会访问该项），因此可能会造成程序运行逻辑上出错。

对于一些只包含 CAS 的临界区代码，他们是不会产生 ABA 问题的（简单的说，因为每次写入都需要比较一下，当然需要一些组织技巧），因此是 naturally idempotent 的。但是大多数代码是不具有幂等性的。

通过在每个具有 ABA 问题的内存区域加入一个计数器 (counter)，每次更新时就更新计数器，就能解决 ABA 问题。

&nbsp;

定义一个 thunk 是一个没有参数的过程。

thunk T 的一个运行 E 是指由一个线程在共享数据上通过序列化的步骤 S 来（协助）执行 T ，执行到 T 的结尾即为结束，设其结果为 E|T。这个过程对于不同的线程来讲可能是相互交错的。

&nbsp;

> **形式化定义**
>
> 当任何有效的执行 E ，包括了在共享数据上任意交错执行 T 的过程，在序列 E|T 中存在一个子序列 E’：
>
> - 第一个结束的 T 过程的最后一步是 E’
> - 移除所有 T 中所有不属于 E’ 的步骤，那么剩下的 T’ 是一个有效的和单独运行 T 一致的结果。
>
> 称这个 thunk T 是幂等的

&nbsp;
&nbsp;

# Implementation

感觉它的做法就是某种意义的快照隔离加上线程协助机制。

协助机制本质上就是一个线程持有锁，其他线程要拿这个锁时，需要跟持有锁的线程完成一样其临界区的任务，直到锁释放。也就是将其他线程自旋的过程改成共同执行一段代码，有一定概率能够跑过原线程。（但是假如不是自旋而是直接 yield 呢？感觉这是个玄学~~）

快照隔离的版本依据不是时间戳而是具体的步骤（体现起来就是利用了一个特性，一个过程在同样的条件下，无论多少次执行，在同一个执行进度点下共享变量的值都应该是相同的，记录的快照数量也是相同的，因此将此刻访问的值做成快照/日志即可让之后重复执行的线程在相同点得到相同值）。



![Lock-free Lock](/images/Thesis Lock-Free-Lock/Lock-free Lock.jpg)

&nbsp;

## Definition

所有对共享内存的操作抽象为 5 种：read, write, CAS, sysAlloc, sysRetire

两种调用

- `sysAlloc`：返回一个未使用的内存块
- `sysRetire`：在确认安全后将内存块释放

&nbsp;

## Idempotence

这篇论文中使用共享日志的方式来保证幂等性。

![Algorithm 2](/images/Thesis Lock-Free-Lock/Algorithm2.png)

对应于抽象的五种操作，Algorithm 2 实现了 5 种操作：`load`、`store`、`CAM`、`allocate`、`retire`。其中 CAM 比较特殊，他就是一个不返回任何值得 CAS 操作。

对于一个可变的共享内存变量，其在 Algorithm 2 中的角色是结构体 `mutable<V>`

> **假设 1**
>
> We assume that CAMs and stores cannot race on the same location.
>
> 在相同内存位置的 CAM 操作和 store 操作不会有抢占行为。

&nbsp;

每个 thunk 对应一个描述字，包括了一个日志指针和一个 thunk，以及一个完成 Flag。

每个进程包含一个日志指针和在日志中的位置。



- **初始化时**（开始执行一个新的 thunk 时），日志指针初始化为指向对应 thunk 的日志。
- **在开始执行时**（`run` 函数），它会存储旧的日志和旧的执行位置以便协助后返回。
- **在协助时**，它会将日志指针指向对应 thunk 的地址，然后在那个共享日志上做事情。
- **在把值存储到日志时**，使用 `commitValue` 函数。假如不存在日志，那就直接返回（只发生在使用这个值时未进入任何锁的范围）；假如存在日志，那需要对对应位置作 CAS 操作：
	- CAS 的 compare 的值是 empty，保证只有第一个提交的数据有效。
	- 返回时需要将日志的内部指针（position）向前移，再次读取（防止没有存储成功），然后返回。
	- 只有写入成功，第二项返回值才是 true。
- **在读出值时**，则只需要从相应位置读出值，然后调用 `commitValue` 函数，这样它读出的值就和别人读出来的一样了。
- **在存储值时**，首先读出值，然后通过 CAS 操作替换掉值。这样的话只有第一次存储的线程能够成功。
- **在 CAM 时**，和存储相似，但是还需要额外检查保证读取到的是期望的值。
- **在分配空间时**，使用 `sysAlloc` 分配内存，然后用 `commitValue` 将其提交到日志，假如提交失败则释放掉。保证只有第一个提交的成功。
- **在释放空间时**，需要保存一个奇怪的值，假如是第一次返回的，那就将对应空间释放掉，因为不会有线程写这个地方了（不可能在该操作之前的写操作取得成功）。

> 这里可能会有疑惑，存储在日志的值和存储值有什么区别。能够发现，在 commitValue 中我们读取的值来自于日志，而存储值时写入的是共享内存。这样即使出现 读-写-读的操作，那都能够读取到对应时间段应该出现的值（类似快照隔离），而不会出现 ABA 问题。

> CAM 操作前做的检查工作是为了保证从日志读取的值也是满足期望的，防止 ABA 问题。



### Evaluation

比较适应于较小的 thunk 这种细粒度的临界隔离，对于较大的 thunk，这些个方法会有额外可观的消耗，因为每次协助都从头开始而不是从执行的中间开始。



### Relax Definition

要将对幂等性的定义作放松使得释放操作能够延迟。（毕竟很多时候这是个耗时操作，要尽量减少临界区，也能减少其他线程协助的负担）。

放松的实现方式是，在线程来到释放空间的地方，它可以进入睡眠，然后其他线程帮助完成剩余的工作。



### Correction

> **Theorem 3.1**
>
> 将所有 thunk T 访问的可变共享变量用 mutable 结构体替代；所有对象的分配和释放用 `sysAlloc` 和 `sysRetire` 替换，能够产生 T 的一个具有幂等性的版本

幂等性并不能保证不会重复分配或者重复释放。所以使用共享日志来防止多次分配和多次释放的操作。

> **Implicit Theorem**
>
> 所有在共享内存变量上的 load、store、CAM 操作在执行中是线性的，而且只会执行一次。

&nbsp;

## Lockless Locks

一个值得讨论的是锁的嵌套，也就要求锁本身是幂等性的或者在多线程共同持有下是安全的。

在之前对于幂等性读/写/CAM的实现机制下，锁很好实现，但是需要保证锁本身是幂等性的。

![Lock Structure](/images/Thesis Lock-Free-Lock/LockStructure.png)

一个锁描述字 `lockDescr` 由 一个描述字 `descriptor` 和 一个表示是否上锁的布尔变量 组成。

一个锁就是一个锁描述字的 mutable 变量。

获取一个锁：

- 读取锁，检查是否已经上锁。
	- 假如没有上锁，那就创造一个新的 thunk f 的描述字，然后标记上锁。接着尝试通过 CAM 操作放入描述字。然后再次读出锁并且检查是否成功上锁。
		- 假如成功上锁或者之前上过锁了，那么执行代码然后释放锁
		- 假如没能成功上锁但是 `currentDescr` 被上锁了，那么算法会协助这个执行过程然后释放锁。
		- 无论是否成功你上所， `myDescr` 都需要被释放掉。
	- 假如上锁了，那么算法将协助执行然后释放它。
- 最后返回结果。只有成功上锁并且 thunk 返回 true 才返回 true。



### Correction

对 `tryLock` 正确的定义是：

假如失败了，那么不会执行临界区代码并且返回 false；假如成功了，那么执行临界区代码然后返回 true，且此时不会有其他在相同锁上的临界区代码会被执行。

同时（为了避免全部直接返回 false），要求成功的 `tryLock` 需要在时将锁指向对应描述字**进入**，在对应锁从 locked 转变为 unlocked 时**退出**。

> **Theorem 4.1**
>
> The tryLock in Algorithm 3 is correct as long as run(descriptor) runs the user code in the thunk 𝑓 idempotently, and the operations on a Lock (load, CAM and store) and on descriptors (createDescriptor and retireDescriptor) are idempotent.
>
> 只要 run(descriptor) 函数操作**幂等地**执行 thunk 中的用户代码，并且在锁和描述字上的操作（load、store、CAM、createDescriptor、retireDescriptor）都是幂等的，那么Algorithm 3 中的 tryLock 是正确的

> **Theorem 4.2**
>
> Consider an algorithm using simply nested tryLocks for which the maximum step count for any tryLock, not including helping, is bounded. In such an algorithm, a tryLock, including any helping it does, will run in bounded steps, and for every bounded number of tryLock attempts at least one top-level tryLock will recursively succeed.
>
> 考虑一个不包括协助 tryLock 程序，它的 tryLock 操作步骤是有限的，那么一个包括协助的 tryLock 也将执行有限的步骤。并且每种 tryLock 的尝试都至少会有一个取得成功。
>
> 
>
> *注：这里感觉需要参考 《多处理器编程的艺术》 一书中关于 lock-free 和 wait-free 的定义*

这个定理表明，简单嵌套的 tryLock 是 lock-free 的，因为一个 tryLock 会在有限步骤内取得成功。

但是可能存在一个获取锁的过程一直失败，所以不是 wait-free 的。同时也不能保证使用简单嵌套 tryLock 的算法是 lock-free 的（不是 **复合的**）。

> 这里涉及的概念（lock-free/wait-free/复合）十分晦涩，建议参考 《多处理器编程的艺术》 中的相关定义。

tryLock 失败的原因不仅有获取锁失败，还有可能因为一些一致性检查失败了。但是因为这些一致性检查失败时，一般程序都能够有所进展（也就是至少这个过程一直有在向前推进而不是原地踏步一直失败）。

同时在 thunk 还未完成时提前释放锁有时候会有点用。因此它还搞了个 `unlock` 操作。这种操作在 hand-over-hand locking 会比较有用。

