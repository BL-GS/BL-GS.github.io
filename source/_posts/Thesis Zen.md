---
title: 'Thesis Reading: Zen, a robust NUMA-aware OLTP engine optimized for non-volatile main memory'
categories:
- Transaction
tags:
- HTM
- Transaction
mathjax: true
description: 'Thesis reading: Zen, a robust NUMA-aware OLTP engine optimized for non-volatile main memory'
---

# Introduction

传统的非易失性存储器（如硬盘）只支持块级存储，同时速度比 DRAM 慢几个数量级。

NVM 的特点：

* 按字节访问的，在一些场景能够达到接近 DRAM 的速度，具有比 DRAM 大若干倍的空间。
* 写带宽远低于读，随机读写更慢



因此需要重新设计事务数据库，主要考虑到 

- **元数据的修改** `tuple metadata modification`

	主流的内存 OLTP 都为每个 tuple 维持少量的元数据用于并发控制。这些元数据会在 tuple 读或者写的时候更新，因此造成昂贵的 NVM 写。

- **NVM 写冗余** `NVM write redundancy`

	传统的 OLTP 的持久性依赖于日志和检查点/快照，但是这个会造成一定的写冗余。

- **NVM 空间管理** `NVM space management`

	NVM 的空间分配需要保证持久性，因此需要记录一些分配操作的元数据。传统的 OLTP 事务通常执行很多的插入删除，导致很大的分配开销。



Zen 引入了：元数据增强的 tuple cache，log-free 的持久化事务，轻量级的空间管理。

Zen+ 引入了：基于 MVCC 的适应性执行，NUMA-aware 的软分块(soft partition)技术。

&nbsp;

## 基于 NVM 数据结构和系统的设计原则

1. 尽可能多的将数据结构放在 DRAM。尤其是当它们是瞬态的 `transient` 或者能够在恢复后重建
2. 尽可能地减少 NVM 写
3. 尽可能地减少持久化操作。


&nbsp;
&nbsp;


# 基于 NVM 的 OLTP 的数据管理

![OLTP-Memory-Architecture](/images/Thesis%20Zen/OLTP-Memory-Architecture.png)

&nbsp;

## MMDB (Main Memory Database)

如 [图(a)](OLTP-Memory-Architecture)，当 OLTP 数据库的大小超过 DRAM 时，MMDB 将部分 NVM 视为 DRAM 使用，将部分的 tuple 和 索引放在 NVM 上。

但是恢复时依赖的仍然是 NVM 上的 Log 和检查点。

**缺点**：修改一个 tuple 还需要修改 WAL 和检查点，导致两倍的写放大。当数据存放在 NVM 上，则还需要写入 NVM；当数据库增大，会有更多 tuple 需要存放在 NVM 上，导致大量的 NVM 写。

&nbsp;

## WBL (Write-behind logging)

如 [图(b)](OLTP-Memory-Architecture)，WBL 维持一个 tuple 的缓存。在事务执行过程中，tuple 从 tuple cache 中获取。同时 WBL 通过每个 tuple 的元数据（如 transaction ID，cts，版本指针）来支持在 NVM 上的多个版本。

跟 WAL 不同，WBL log 不包括修改后的数据，日志在一系列事务提交后持久化。由于持久化需要内存屏障（SFENCE），因此当日志持久化后代表在其之前所有的事务都完成了。

在恢复时只需要查找最后一个日志项然后撤销操作即可。

**优点**：相比于 MMDB，它的日志大小比较小并且不需要维护检查点（恢复时只看最后一个日志项）。因此对于一个 tuple 的更新只需要写一次 NVM。

**缺点**：它需要为每个 tuple 维护一个元数据（在 NVM 中），因此需要对 NVM 做很多随机写（主要是时间戳啥的）

&nbsp;

## FOEDUS 

如 [图(c)](OLTP-Memory-Architecture)，FOEDUS 将 tuple 存在 NVM 中的一个快照页中，在 DRAM 中创建一个页的 Cache。DRAM 中页的索引维护两个指针，一个指向 NVM 中的快照页，一个指向 DRAM 中页的 Cache。当　DRAM 中不存在对应 Cache，则从 NVM 载入。

因此需要有一个垃圾回收机制回收不需要的快照。

**优点**：避免了对每个 NVM 中的 tuple 进行修改。

**缺点**：页粒度的 Cache 造成了读放大。同时对快照的回收和分配造成了 NVM 的写。

&nbsp;

## Zen

如 [图(d)](OLTP-Memory-Architecture)，为 tuple 维护一个元数据 Cache，但是粒度比 FOEDUS 小，是 tuple 级别。相比于 WBL，它只在 Cache 上更改数据。

Zen 完全移除日志。同时还使用所谓的 light-weight NVM space allocation


&nbsp;
&nbsp;

# Zen 的组成架构

![Zen-Architecture](/images/Thesis%20Zen/Zen-Architecture.png)

每张 Table 都有一个 HTable `hybrid table`。其中包含了：

* NVM 中的一个 tuple 堆
* DRAM 中的一个 Met-Cache
* 线程独有的 NVM tuple 管理器

除此之外，Zen 还在 NVM 上存了元数据（如分配的结构），在 DRAM 上维持事务的都有数据和一些辅助信息（如索引）。

&nbsp;

## NVM-tuple Heap `thread local`

Zen 将所有 tuple 存储在这个 NVM-tuple Heap 中，类似一个表。

* 一个表包含了若干个固定大小的页，每个页包含固定数目的 tuple 槽位。
* 一个 NVM-tuple 包含了一个 header（transaction ID, cts, deleted, LP） 和数据。
* 一个堆中可能包含若干个版本的 tuple

header 中数据位的用途：

| Kind           | Meaning                               |
| -------------- | ------------------------------------- |
| Transaction ID | 事务的 ID                             |
| cts            | 提交的时间戳                          |
| Deleted        | 该逻辑 tuple 是否被删除               |
| LP             | 该 tuple 是否是该事务的最后一个 tuple |

&nbsp;

## Met-Cache `thread local`

为 NVM-tuple Heap 中的数据做一个缓存。为每个线程都有一份。

Cache 中数据位的作用：

| Kind        | Meaning                           |
| ----------- | --------------------------------- |
| NVM-Pointer | 指向 NVM 中对应 tuple             |
| dirty       | 是否被一个活跃的事务使用          |
| clock       | 用于伪LRU替换算法（时钟替换算法） |
| copy        | 该 entry 是否已经被复制           |
| CC-Meta     | 保留，用于并发控制                |

&nbsp;

## Indices in DRAM

在 DRAM 中为每个 H-Table 创建一个 Indices，在崩溃恢复时重建

对于 primary index，其指向最新版本的 tuple，即 Met-Cache 或者 NVM 中的数据。

要求该索引结构满足并发需要，同时只有被提交的项目才是可视的。

&nbsp;

## NVM Space Management

管理 NVM 的空间回收和分配。


&nbsp;
&nbsp;

# Metadata-enhanced tuple cache

## 基本思路

* 不为每个 tuple 创建一个元数据
* 在 DRAM 中为每个线程创建一个 Met-Cache 用来协助当前事务运行，比如缓存最近使用的 tuple，为并发控制提供元数据协助。

## 具体做法

* 线程只允许写属于自己的 Met-Cache，但是可以读其他线程的 Met-Cache 或者 Tuple Heap（通过索引）。
* 当需要修改的 tuple 来自其他线程的领域时，需要设置 copy 位
* 通过 CLOCK 算法，伪 LRU 地替换 Cache 中数据

## 特点

* 消除不同线程在数据上的竞争
* 将 NVM 上的读写转移到 DRAM 上
* 崩溃的事务不会造成多余的 NVM 写入
* 在 DRAM 上实现并发控制


&nbsp;
&nbsp;

# Log-free persistent transactions

## 基本思路

* 移除 log 和检查点
* 通过元组具有的 ID 属性和 cts `commit timestamp` 、事务的提交标志位 `LP` 来保证原子性。
* 在写入数据时不覆盖过去的数据，类似 MVCC。


## 具体做法

分为三个阶段：

- `Perform`，在 DRAM 中执行事务
- `Persist`，将新的 tuple 持久化到 NVM 中
- `Maintenance`，垃圾回收，收集淘汰的 tuple


### Perform

* 通过主索引查找 tuple。如果 tuple 在 NVM，则将它读到 Met-Cache 中，更新索引
* 在 DRAM 上执行事务
* 检查所有接触的 Met-Cache 条目，假如有 dirty 的，将其从 NVM-tuple 恢复，以便重试时在 Met-Cache 中找到该条目。

> **Zen 在替换出 Met-Cache 中 tuple 条目时不需要将其写回 NVM**
>
> 有三种情况：
>
> 1. 如果它是只读的，则不需要更改
> 2. 如果它是之前的事务留下的，则已经在 commit 时被持久化到 NVM 中了
> 3. 如果它被一个已经崩溃的事务修改过的，则本来就是无效的


### Persist

- 将一个 tuple 持久化到一个 NVM-tuple Heap 的空槽中（不覆盖原有数据）
- 持久化的顺序无所谓，在最后一个 tuple 持久化后加上一个 LP 位，表示已经提交完成。

### Maintenance

* 每个线程都有一个垃圾回收器，回收 NVM 上的 tuple：
	* 当提交了一个与其他 NVM tuple 相同键值的条目，则将旧项目加入回收队列
	* 在从 Met-Cache 逐出一个条目时，垃圾回收器查看该条目是否有 copy 位，有的时候才能被收集其 NVM tuple。（意味着该条目被其他已经提交的事务复制了，“Note that E must have been copied to another region by a committed transaction T , and T must have written a new version of the tuple”，这里感觉怪怪的，之前明明设置 copy 位时事务还没提交[Metadata-enhanced tuple cache](Metadata-enhanced tuple cache)）
* 回收队列上的条目不能被马上释放，因为可能还有事务在用到该版本。
* 通过用每个线程最后提交事务的cts，定期计算全局的最小 min_cts，将所有 cts < min_cts 的都释放掉

### Crash Recovery

* 重建索引和 NVM 空间管理数据结构，其他的比如 Met-Cache 就不需要恢复了。
* 对于没有 LP 位的版本数据，就是没有提交成功的。首先得到最大的已提交事务的时间戳，然后通过和那个最大时间戳比，得到所有已经提交的 tuple。
* 提出了一个算法能够扫描一遍得出所有未提交事务，同时建立索引 Algorithm2
	* 扫描一遍，当遇到一个有 LP 位的 tuple 时，令 ts-commit = max(ts-commit, cts)。同时更新索引。如果对应 tuple 索引已经存在，则比较 cts 留下最新的那个。
	* 扫描期间，遇到了 timestamp > ts-commit 的，放在一个列表 pending list 里
	* 扫描完成后，此时 ts-commit 为已提交事务的最大 cts。
	* 扫描完成后，对 pending list 中所有 tuple，再次扫描，所有 timestamp > ts-commit 的都是未提交成功的。而对于那些提交成功的，更新索引。


&nbsp;
&nbsp;

# Lightweight NVM space management


## 基本思路

* 目的是减少 NVM 空间管理的持久化操作。


## 具体做法

* 完全将分配的元数据放在 DRAM
* 每个线程都有自己的堆区域和分配器
* 按页分配~~


&nbsp;
&nbsp;

# MVCC-based adaptive execution

## 基本思路

* 目的是有效的执行长事务同时保证系统资源的利用率
* 长事务容易引发饥饿问题或者不能充分利用并行的问题。

## 具体做法

考虑两种长事务：只读长事务和读写长事务。前者较为常见，后者比较少见。因此对前者做详细处理，后者简单粗暴一点~~

一些细节详见论文 4.1


### 长事务的执行

对于只读长事务：

- 设置其 cts 位当前全局的最小 cts - 1。这样只读长事务只能看到一个在其他活跃更新事务提交之前的快照。
- 当出现 Met-Cache 缺失时，不需要再往 Met-Cache 替换东西，直接从 NVM 读就是了。节省了资源利用
- 采用两种状态：排他状态和检测状态。检测状态中，事务正常执行，需要时不时检查资源使用情况，当有频繁的崩溃重试等导致资源浪费，则进入排他阶段。排他阶段中，选中一个长事务，单线程执行到结束。

对于更新长事务，直接进入排他阶段。



### 长事务的检测

* 事务具有更新，它访问的 tuple 数量大于一个阈值（95%）。
* NVM 分配器内可用的空间资源小于一定阈值（5%）。
* 崩溃的工作达到一定阈值（16 million）。


&nbsp;
&nbsp;

# NUMA-aware soft partition

## 基本思路

* 目的是充分利用 NUMA 的特点，最小化 remote 访问。
* 尽量不使用数据迁移
* 线程隔离的资源，能够做成本地 NUMA 节点的
* 允许本地写和任意读
* 对于 Zen 这种本地写来说，其实 NUMA 的最大限制来自于其事务分配线程的随机性；假如同一个键值的事务都能让同一个线程来进行，那样就能一直 NUMA local。但是这样对于集中访问比如 Zipfian 就很不友好，因为很多资源没有利用上。于是用软分块的方式（键值 Hash）将其分散到各个 NUMA 节点上

## 具体做法

主要思想是用软分块 `soft partition`。动态使用工作负载的访问特点。

- 将表 logically 分为若干个 partition，映射到不同 NUMA 节点上。
- 对于一个事务，通过其要访问的 partition，计算其 NUMA 亲和度。将其分配到最相关的 NUMA 节点上。具体算法感觉怪怪的，详见论文4.2.2 Algorithm3、Algorithm4。感觉就是最多最近访问，毕竟对于 Zen 这种本地写来说，多次本地写意味着访问的 tuple 都是本地的。
- 分配事务到对应 NUMA 节点的线程上进行。