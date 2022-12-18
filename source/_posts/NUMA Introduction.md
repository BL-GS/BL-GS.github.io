---
title: NUMA Introduction
---
# NUMA

> **非统一内存访问架构 `Non-Uniform Memory Access`(NUMA)** 是一种为多处理器的电脑设计的内存架构，内存访问时间取决于内存相对于处理器的位置。在NUMA下，处理器访问它自己的本地内存的速度比非本地内存（内存位于另一个处理器，或者是处理器之间共享的内存）快一些。
>
> 非统一内存访问架构的特点是：被共享的内存物理上是分布式的，所有这些内存的集合就是全局地址空间。所以处理器访问这些内存的时间是不一样的，显然访问本地内存的速度要比访问全局共享内存或远程访问外地内存要快些。另外，NUMA中内存可能是分层的：本地内存，群内共享内存，全局共享内存。
>
> —— 维基百科



1. CPU 厂商把内存控制器集成到 CPU 内部，一般一个 CPU socket 会有一个独立的内存控制器（IMC，Integrated Memory Controller）。
2. 每个 CPU scoket 独立连接到一部分内存，即本地内存。
3. CPU 之间通过 QPI（Quick Path Interconnect）总线进行连接，从而可以通过它访问属于其他 socket 的内存。



其实简单而言，就是将CPU、Cache分成若干个组，每个组都会对应一定的内存空间。每个 CPU 所在的 socket 都有自己的本地内存空间，都可以访问所有的内存空间，但是有一个距离的概念，对于越“远”(remote)的内存空间，访问的速度越慢；反之，对于本地(local)的内存空间，其访问的速度最快。



> 与 NUMA 相对的是 **统一内存访问架构 `Uniform memory access`（UMA）**。是指多个处理器通过统一总线访问存储器，每个处理器对内存的访问都是一致的。
>
> 这种方式有一个缺点，任何数据都经过北桥中的总线来进行，导致当 CPU 核心增加且总线带宽有限的情况下，内内存的访问延迟增大。

# NUMA 相关工具



对于本地机器上的高吞吐程序而言，NUMA 对性能的影响是不容忽视的，通常需要了解如下一些问题：

* 对于程序的运行平台，它的拓扑是怎样的
* 了解访问不同 NUMA 节点的内存空间的消耗
* 控制数据等的存放位置
* 控制进程、线程的工作方式（比如对 CPU 的亲和度 `affinity`）



## 环境检测

我们可以通过 `lscpu` 或者 `numactl -hardware` 查看当前机器的 NUMA 分布：

![image-20221216220906102](/images/NUMA%20Introduction/lscpu_result.png)

![image-20221216220934534](/images/NUMA%20Introduction/numactl_result.png)

可以看到，当前这台机器上共有四个 NUMA 节点，分别对应 [0-23，96-119],[24-47,120-143],[48-71,144-167],[72-95,168-191] ID 的逻辑 CPU，而每个 CPU socket 对应两个逻辑 CPU。

> CPU 不等于物理核或者逻辑核。
>
> 物理核是指真实存在的 CPU 核心，是相互独立的电路，具有独立的 Cache，可以独立执行指令
>
> 逻辑核是指在逻辑上展现给用户的 CPU 核心，比如说 2ms 的刷新间隔就能够使得人眼分辨不出数码管的刷新动作，这时候我有个 1ms 一次的 CPU，它就可以装成两个核心，然后同时处理两个数码管。
>
> 超线程技术是在一个逻辑核等待指令执行的间隔，把时间片分配给另一个逻辑核，但是对用户而言是透明的。



同样的，通过 `hwloc` 也能够检测到机器上的 NUMA 配置。在 CentOS 上执行 `lstopo --no-io topology.pdf` 就能够生成该机器的 NUMA 配置（依赖于 hwloc），如下所示：

![](/images/NUMA%20Introduction/lstopo_result.png)

能够看到总共有 4 个 NUMA 节点，同时每个节点包含 24 个物理 CPU，每个物理 CPU 包含 2 两个逻辑 CPU。每个 NUMA 节点掌管 94 GB 的本地内存空间。



## Linux 上的 NUMA 配置

通过 `numactl` 命令可以进行 NUMA 的配置，其常用选项有：

* `numactl --hardware` 或 `numactl --show` ：打印机器上的 NUMA 配置
* `numactl --cpubind=0 [exe file]` ：只在 0 号 NUMA 节点上的 CPU 运行该可执行程序
* `numactl --membind=0 [exe file]` ：只在 0 号 NUMA 节点上的内存运行该可执行程序
* `numactl --interleave=[nodes]` ：其中 [nodes] = {All, A-B, A,B,...N} ，(A,B,N 代表 NUMA 节点编号），表示将对应 NUMA 节点的视为整体进行资源分配
* `numactl --preferred=0`： 优先考虑从 node 0 上分配资源。



## 专属 Linux 的 NUMA 亲和设置

可以在程序中检查 /proc/cpuinfo 查找对应的 CPU 信息，确定不同 NUMA 节点的范围，然后利用 sched_setaffinity() 确定 进程/线程 运行的 CPU。 

当然在运行程序前用 `numactl` 进行指定也是可以的。



## HWLOC 设置 NUMA 亲和

采用的 proTBB 书中的做法。

首先需要解决如下几个问题：

* 如何在程序中获取当前设备 NUMA 的配置情况
* 如何在程序中按照 NUMA 节点的分布情况分配内存
* 如何在程序中按照 NUMA 节点的分布情况分配线程

对于检测，可以通过 `hwloc_get_nbobjs_by_type` 来获得当前机器的 NUMA 节点情况

```c++
#include <hwloc.h>

int main() {
    hwloc_topology_t topo;
    hwloc_topology_init(&topo);
    hwloc_topology_load(topo);
    
	int num_nodes = hwloc_get_nbobjs_by_type(topo, HWLOC_OBJ_NUMANODE);
    std::cout << "There are " << num_nodes << " NUMA node(s)" << std::endl;
	
    return 0;
}
```

输出当前系统上 NUMA 节点的个数。同理可以将 `HWLOC_OBJ_NUMANODE` 改成 `HWLOC_OBJ_PU` 或者 `HWLOC_OBJ_CORE` 来分别获取物理 CPU 和逻辑 CPU 的信息。

那么如何按照 NUMA 节点来分配空间呢：

```c++
int num_nodes = hwloc_get_nbobjs_by_type(topo, HWLOC_OBJ_NUMANODE);
for (int i = 0; i < num_nodes; ++i) {
    // Return a handle to the ith NUMA node object
    hwloc_obj_t numa_node =
        hwloc_get_obj_by_type(topo, HWLOC_OBJ_NUMANODE, i);
    char *s;
    // numa_node->nodeset: a similar bitmask that identifies the node
    hwloc_bitmap_asprintf(&s, numa_node->nodeset);
    std::cout << "Allocate data on node " << i << " with node bitmask " << s
        << '\n';
    std::free(s);

    data[i] = (double *)hwloc_alloc_membind(
        topo, size * sizeof(double), numa_node->nodeset, HWLOC_MEMBIND_BIND,
        HWLOC_MEMBIND_BYNODESET);
}
```

该段程序中，首先获取不同 NUMA 节点对应的 hwloc_obj_t 对象，然后利用该对象，对每个 NUMA 节点分别通过函数 `hwloc_alloc_membind` 在对应 NUMA 节点中分配了一段内存空间。

如何按照 NUMA 节点来分配线程呢？

```c++
int num_nodes = hwloc_get_nbobjs_by_type(topo, HWLOC_OBJ_NUMANODE);
std::vector<std::thread> vth;
for (int i = 0; i < num_nodes; ++i) {
    vth.emplace_back([i, num_nodes, &topo, &sout]() {
        int err;
        sout.local() << "I'm master thread: " << i << " out of " << num_nodes
            << '\n';
        sout.local() << "Before: thread: " << i << " with tid "
            << std::this_thread::get_id() << " on core "
            << sched_getcpu() << '\n';

        hwloc_obj_t numa_node =
            hwloc_get_obj_by_type(topo, HWLOC_OBJ_NUMANODE, i);
        err =
            hwloc_set_cpubind(topo, numa_node->cpuset, HWLOC_CPUBIND_THREAD);
        assert(!err);

        sout.local() << "After: thread: " << i << " with tid "
            << std::this_thread::get_id() << " on core "
            << sched_getcpu() << '\n';
    });
}
```

基本思路就是，首先通过系统接口创建线程，然后通过 `hwloc_set_cpubind` 将当前线程 pin 到该 NUMA 节点对应的 CPU 之一上运行即可。

总的程序如下：

```c++
#include <cassert>
#include <cstdlib>
#include <iostream>
#include <sstream>
#include <thread>
#include <vector>

#include <sched.h>
#include <hwloc.h>
#include <hwloc/bitmap.h>
#include <tbb/enumerable_thread_specific.h>

void alloc_mem_per_node(hwloc_topology_t topo, double **data, long size) {
	int num_nodes = hwloc_get_nbobjs_by_type(topo, HWLOC_OBJ_NUMANODE);
	for (int i = 0; i < num_nodes; ++i) {
		// Return a handle to the ith NUMA node object
		hwloc_obj_t numa_node =
			hwloc_get_obj_by_type(topo, HWLOC_OBJ_NUMANODE, i);
		char *s;
		// numa_node->cpuset: a bitmask identifying the logical cores included in the node.
		hwloc_bitmap_asprintf(&s, numa_node->cpuset);
		std::cout << "NUMA node " << i << " has cpu bitmask: " << s << '\n';
		std::free(s);
		// numa_node->nodeset: a similar bitmask that identifies the node
		hwloc_bitmap_asprintf(&s, numa_node->nodeset);
		std::cout << "Allocate data on node " << i << " with node bitmask " << s
				  << '\n';
		std::free(s);
		// Allocate memory on specific numa node.
		data[i] = (double *)hwloc_alloc_membind(
			topo, size * sizeof(double), numa_node->nodeset, HWLOC_MEMBIND_BIND,
			HWLOC_MEMBIND_BYNODESET);
	}
}

void alloc_thr_per_node(
	hwloc_topology_t									topo,
	tbb::enumerable_thread_specific<std::stringstream> &sout) {

	int num_nodes = hwloc_get_nbobjs_by_type(topo, HWLOC_OBJ_NUMANODE);
	std::vector<std::thread> vth;
    // for every numa node, create a thread on it.
	for (int i = 0; i < num_nodes; ++i) {
		vth.emplace_back([i, num_nodes, &topo, &sout]() {
			int err;
			sout.local() << "I'm master thread: " << i << " out of " << num_nodes
				 << '\n';
			sout.local() << "Before: thread: " << i << " with tid "
						 << std::this_thread::get_id() << " on core "
						 << sched_getcpu() << '\n';

			hwloc_obj_t numa_node =
				hwloc_get_obj_by_type(topo, HWLOC_OBJ_NUMANODE, i);
            // Bind to specific cpu with a specific numa node.
			err =
				hwloc_set_cpubind(topo, numa_node->cpuset, HWLOC_CPUBIND_THREAD);
			assert(!err);

			sout.local() << "After: thread: " << i << " with tid "
						 << std::this_thread::get_id() << " on core "
						 << sched_getcpu() << '\n';
		});
	}
	for (auto &th : vth) { th.join(); }
}



int main(void) {
	hwloc_topology_t topo;
	hwloc_topology_init(&topo);
	hwloc_topology_load(topo);

	// Print the number of NUMA nodes.
	int num_nodes = hwloc_get_nbobjs_by_type(topo, HWLOC_OBJ_NUMANODE);
	std::cout << "There are " << num_nodes << " NUMA node(s)\n";

	double **data = new double *[num_nodes];
	// Allocate some memory on each NUMA node
	long size = 1024 * 1024;
	alloc_mem_per_node(topo, data, size);

	tbb::enumerable_thread_specific<std::stringstream> sout;
	// One master thread per NUMA node
	alloc_thr_per_node(topo, sout);

	for (auto &s : sout) {
		std::cout << s.str();
	}

	// Free the allocated data and topology
	for (int i = 0; i < num_nodes; ++i) {
		hwloc_free(topo, data[i], size);
	}
	hwloc_topology_destroy(topo);
	delete[] data;
	return 0;
}
```

输出结果：

```shell
There are 4 NUMA node(s)
NUMA node 0 has cpu bitmask: 0x00ffffff,,,0x00ffffff
Allocate data on node 0 with node bitmask 0x00000001
NUMA node 1 has cpu bitmask: 0x0000ffff,0xff000000,,0x0000ffff,0xff000000
Allocate data on node 1 with node bitmask 0x00000002
NUMA node 2 has cpu bitmask: 0x000000ff,0xffff0000,,0x000000ff,0xffff0000,0x0
Allocate data on node 2 with node bitmask 0x00000004
NUMA node 3 has cpu bitmask: 0xffffff00,,,0xffffff00,,0x0
Allocate data on node 3 with node bitmask 0x00000008

I'm master thread: 0 out of 4
Before: thread: 0 with tid 140737289062144 on core 175
After: thread: 0 with tid 140737289062144 on core 0
I'm master thread: 1 out of 4
Before: thread: 1 with tid 140737280669440 on core 176
After: thread: 1 with tid 140737280669440 on core 24
I'm master thread: 2 out of 4
Before: thread: 2 with tid 140737272276736 on core 175
After: thread: 2 with tid 140737272276736 on core 48
I'm master thread: 3 out of 4
Before: thread: 3 with tid 140737263884032 on core 175
After: thread: 3 with tid 140737263884032 on core 175
```

能够看到，经过上述分配的线程能够分散到不同 NUMA 节点中的 CPU 上运行了。



## oneTBB 的 NUMA 亲和设置

proTBB 书中给出的例子，在某些版本的 TBB 中并不能使用，因为 `tbb::task_scheduler_observer` 只能够作为全局配置实现，同时由于 TBB 在运行中可能不是严格的多线程并行，因此会造成一些个问题。

这里介绍 oneTBB 上的实现，感觉更加干净和整洁一些。

```c++
#include <iostream>
#include <vector>

#define __TBB_PREVIEW_TASK_ARENA_CONSTRAINTS_EXTENSION_PRESENT 1

#include <oneapi/tbb/info.h>
#include <oneapi/tbb/task_arena.h>
#include <oneapi/tbb/parallel_invoke.h>

int main() {
	// Print the number of NUMA nodes.
	std::vector<oneapi::tbb::numa_node_id> numa_indexes = oneapi::tbb::info::numa_nodes();
	std::vector<oneapi::tbb::task_arena> arenas(numa_indexes.size());
	std::vector<oneapi::tbb::task_group> task_groups(numa_indexes.size());

	std::cout << "NUMA nodes: ";
	for (auto node_id: numa_indexes) {
		std::cout << node_id  << ' ';
	}
	std::cout << std::endl;

	for(unsigned j = 0; j < numa_indexes.size(); j++) {
		auto constraint = oneapi::tbb::task_arena::constraints(numa_indexes[j]);
		constraint.set_max_threads_per_core(1);
		arenas[j].initialize(constraint);
		arenas[j].execute([&task_groups, &j](){
			task_groups[j].run([j](){
				std::cout << "After: thread: " << j << " with tid "
				                     << std::this_thread::get_id() << " on core "
				                     << sched_getcpu() << '\n';
			});
		});
	}

	for(unsigned j = 0; j < numa_indexes.size(); j++) {
		arenas[j].execute([&task_groups, &j](){ task_groups[j].wait(); });
	}
}
```

该程序中，首先需要通过 `oneapi::tbb::info::numa_nodes()` 初始化 NUMA 配置信息，需要注意的是，这里不能通过 hwloc 替代，因为这些数据会存入到 oneapi::tbb 库中对应的全局变量里。

然后，通过 `tbb::task_arena::constraints` 来限制 NUMA 节点和每个核心上的最大线程数量，用它来创建 task_arena，之后通过该 arena 运行的线程，即可被固定在对应的 NUMA 节点上了。

运行结果：

```shell
NUMA nodes: 0 1 2 3 
After: thread: 0 with tid 139885207942912 on core 0
After: thread: 1 with tid 139885203744512 on core 24
After: thread: 2 with tid 139885182752512 on core 48
After: thread: 3 with tid 139885195347712 on core 72
```

能够看到，线程同样被分配到了对应 NUMA 节点上。



# NUMA 亲和策略

这里介绍论文 Zen+: a robust NUMA-aware OLTP engine optimized for non-volatile main memory 中做的 NUMA 策略分析。

## 整体访问

即将所有 NUMA 节点作为一个整体进行资源的分配

![image-20221218094601132](/images/NUMA%20Introduction/numa_strategy1.png)

优点是简单，不需要额外的判断处理

缺点是太简单。。。对 remote write 和 remote read 都没有处理，可能会拖累性能

## 全局读-本地写

由于 Zen+ 的事务逻辑采用类似 MVCC 的做法，它并不需要覆盖或修改原有的数据，而是增加一个新的 tuple 来表示。

因此它的做法是对于读操作，允许当前 NUMA 节点的 CPU 读取其他任意节点的东西，但是写入操作只发生在当前 NUMA 节点的内存里。

![image-20221218094952792](/images/NUMA%20Introduction/numa_strategy2.png)

优点是消灭了 remote write，尤其对于 NVM 来说，其写入操作的延迟要高很多。

缺点是 local write 会造成更多的 remote read（相对于随机读写来说）。

## 动态 Partition

Zen+ 中提出一种在处理事务前先对其进行 **逻辑上的** NUMA 分块，对应事务会运行在分配给的 NUMA 节点上，然后程序根据不同 NUMA 节点的访问热度进行动态调整，使得其访问尽量地均衡。

当然，这是在全局读-本地写的基础上进行改动的。

![image-20221218095401132](/images/NUMA%20Introduction/numa_strategy3.png)

具体的算法感觉不是很难但是较为繁琐，可以参照原论文中的代码进行理解。







# 参考资料

1. [每个程序员都应该知道的 CPU 知识：NUMA - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/336365600)
2. [NUMA架构的CPU -- 你真的用好了么？ • cenalulu's Tech Blog](http://cenalulu.github.io/linux/numa/)
3. [Uniform memory access - Wikipedia](https://en.wikipedia.org/wiki/Uniform_memory_access)
4. [Non-uniform memory access - Wikipedia](https://en.wikipedia.org/wiki/Non-uniform_memory_access)
5. 《proTBB》 —— Michael Voss，Rafael Asenjo，James Reinders
6. [oneAPI Threading Building Blocks (oneTBB) — oneTBB documentation (oneapi-src.github.io)](https://oneapi-src.github.io/oneTBB/)
7. Liu, Gang, Leying Chen, and Shimin Chen. “Zen+: A Robust NUMA-Aware OLTP Engine Optimized for Non-Volatile Main Memory.” *The VLDB Journal*, April 6, 2022. https://doi.org/10.1007/s00778-022-00737-1.

