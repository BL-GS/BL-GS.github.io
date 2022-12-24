---
title: Coroutine Introduction
categories:
- Coroutine
tags:
- Coroutine
- C++
description: Coroutine, a kind of light thread, is supported in C++20. In this article, I am going to introduce what is coroutine and how it works in C++20.
---

# 协程的含义

> **协程**（coroutine）是计算机程序的一类组件，推广了[协作式多任务](https://zh.wikipedia.org/wiki/协作式多任务)的[子例程](https://zh.wikipedia.org/wiki/子例程)，允许执行被挂起与被恢复。相对子例程而言，协程更为一般和灵活，但在实践中使用没有子例程那样广泛。协程更适合于用来实现彼此熟悉的程序组件，如[协作式多任务](https://zh.wikipedia.org/wiki/协作式多任务)、[异常处理](https://zh.wikipedia.org/wiki/异常处理)、[事件循环](https://zh.wikipedia.org/wiki/事件循环)、[迭代器](https://zh.wikipedia.org/wiki/迭代器)、[无限列表](https://zh.wikipedia.org/wiki/惰性求值)和[管道](https://zh.wikipedia.org/wiki/管道_(软件))。
>
> ——wiki 百科

其实简而言之，协程就是函数层面的轻量级“线程”（可以把线程理解为硬件层面或者进程层面的东西），某个函数执行到一半的时候，可以通过某些机制保存当前现场然后暂停执行，之后可以通过外部代码重新恢复执行。

这样说可能过于笼统了，通常将其与函数调用和线程调度相比较来理解。

&nbsp;

## 与函数调用的比较

首先函数调用是怎么样的？考虑下面一段程序：

```c
void work() { do some work; }
void fun() { have some fun; }
void main() {
    work();
    fun();
}
```

此时执行 main 函数，它会先运行 work 函数，等到所有 work 都做完，work 函数返回，然后才能调用 fun 函数，同样地去找完所有乐子再返回。



现在假如我想，在工作的时候摸鱼，找些乐子，然后等下回来，继续我的工作。那么我会这样做：

```c
void work() {
	do some work;
	mind = fun();
	do some work;
}
Mind fun() { 
    have some fun; 
    return happy;
}
void main() {
    work();
    fun();
}
```

这时候就能在做完一些工作后，执行 fun 函数，去摸鱼。



但是又有问题了，可能我的乐子有很多，但是我不能总摸鱼对吧（恼），我希望找一个乐子之后，就回去工作，工作一会儿再回来找乐子。比如：

```c
void work() {
	while (true) {
        do some work
        mind = fun();
    }
}
Mind fun() { 
	while (true) {
        have a fun
        work();
    }
    return happy;
}
void main() {
    work();
}
```

这时候问题来了，我在 fun 函数里调用 work 函数后，这时候栈的布局是这样的：

![Stack layout](/images/Coroutine/Coroutine_stack.jpg)

也就是说第二次调用的 work 和原来的 work 用的不是同一个栈，我不能回到我原来的工作去了！！。我不能接收做完全部工作再找乐子，但是摸鱼又只能摸一会儿（悲），而且之后要回到原来的工作上。

那要是这样：

```c
void work() {
	while (true) {
        do some work
        mind = fun();
    }
}
Mind fun() { 
	have a fun
	return happy;
	have some fun
}
void main() {
    work();
}
```

这样确实能够回到原来的位置，但是每次只能执行 fun 的第一条语句，只能摸同样的鱼（悲。我不能到达之后的 have some fun 处。

> 当然可以说我让 work 和 fun 函数只做一个 work 或者 fun，但是毕竟示例嘛，不要太纠结。。。



仔细分析，抽象出几个需求：

* 两个工作可以在运行过程中调用彼此
* 每个工作可以有多个入口，也就是我可以在 have a fun、have two fun 之后的位置进入 fun 函数。
* 当从另一个工作返回时（无论中途返回或者最后返回），能够返回之前的状态（执行语句、变量等）。反之亦然。
* 即便中途返回，也能够得到一个值。可以多次返回，同样的可以多次返回值。

仔细想想，和线程十分的相似。所以我们将目光转向线程。

&nbsp;

## 与线程调度的比较

协程和线程其实非常相似，但是协程是协助式多任务的，线程则是抢占式多任务的。通俗的讲，协程的例程入口是程序员预先确定的，只能在规定的地方切换协程，但是线程可以在任何地方彼此切换（阻塞除外）。

这就意味着**协程提供并发性而不是并行性**，而线程能够提供并行性。

> 并发是指两个进程/线程/协程在宏观角度而言是同时运行的，实际上可能只是交替运行而已。
>
> 并行则是指物理层面上的，比如 CPU0 和 CPU1 在同一时刻运行两个不同的进程/线程/协程

这就使得协程相对于线程有一些**好处**：

* 协程切换在一定程度上是可控的，能够预知的，所以不需要通过同步性原语（信号量等）保护临界资源。
* 能够更加简单的完成一些协助任务，而不需要设计各种栅栏之类的东西阻塞其运行。

但是也有一些**坏处**：

* 它毕竟是逻辑层面上的并发，意味着其只能在软件层面上实现，不能充分利用多核并行的优势
* 当出现阻塞而没有预先设置切换逻辑时，需要等待阻塞，系统不会自动给你切换。

因此，通常线程作为协程的底层基底运行。也就是**多线程并行，各个线程中协程并发**。


&nbsp;

## 有栈协程 & 无栈协程

协程的概念在那了，但是实现起来就有很多技巧了。很多语言包括 Golang、python、C++20 都原生地支持了协程，还有很多库的实现（比如 C 的 libco、C++ 的 Boost），但是总体就分为两大类：**有栈协程** `stackful coroutine` 和 **无栈协程** `stackless coroutine`

> 首先需要了解的，是无论有栈协程还是无栈协程，其都是需要栈的支持的。这个有栈无栈只是指协程的调度过程中上下文 `context` 切换的方式不同。

有栈协程，其上下文切换，本质上和线程一样，只不过所有的工作在用户态进行（类似用户态线程），当然保存的东西也没有线程那么多（只有函数的上下文），切换时需要将整个栈替换掉。

无栈协程，其上下文切换，则是通过指针来实现的。函数的上下文保存在堆中的某个帧上，函数的调用，其栈中保存的不是具体的变量，而是指向保存这些变量的内存帧的指针。每次调用，只需要控制这个指针进出栈。

因此两者各有特点：

- 有栈协程简单粗暴对吧，很多在线程上的理论也可以照搬过来，毕竟这个栈的切换跟线程如出一辙。
- 无栈协程在实现起来比较麻烦，毕竟函数调用不再能够简单的往栈里写东西，而要写到规定的地方。近乎于创建了一个虚拟环境，像C/C++ 这种对堆栈空间管理有严格定义尤其是具有指针这种东西的语言，有语言原生的支持很重要。
- 有栈协程切换开销更大一些，毕竟需要完整替换整个栈，同时由于需要进行栈空间的分配，难免产生内存碎片。
- 无栈协程基于实现可能在函数返回、栈动态增长上比较麻烦，引入一些开销
- 更加底层的，完全替换栈（有栈协程）会导致原有 Cache 条目的失效。而

详细的权衡可以康康这篇文章的讨论：[async/await异步模型是否优于stackful coroutine模型？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/65647171/answer/233495694)

&nbsp;

## 对称协程和非对称协程

对称协程就是说，我调用了一个协程之后，该协程与我当前的协程是一定意义上的相互独立的关系（不考虑彼此的协作行为）。这个被调用的协程在返回时，也不一定会返回原来调用它的协程。

非对称协程则具有类似“父子关系”的东西，被调用者只能返回调用它的协程。

看到一个理论，对称协程和非对称协程其实本质上都差不多，一定意义上是可以互相实现的。所以 Boost 在 Coroutine2 中并未实现对称协程。



想必到这里，读者应该对协程有一些基本的了解。当看完之后的实现过程，相信理解能够更加深刻。


&nbsp;
&nbsp;

# 协程的 C++20 实现

C++20 添加了对协程的支持，其实现是无栈协程，也是非对称协程。

在最新版的 GCC 和 Clang 中已经能够直接支持协程了，不需要再加上 `-fcoroutines` 的编译选项，Clang 也不需要使用 `<experiment/coroutine>` 头文件，毕竟已经被移除了

C++ 的优点在于它对运行时性能的病态追求~~当然也导致一些问题，包括没有通用的垃圾回收器，接口太过底层。包括协程，其提供的支持也是十分的贴近底层，面向编译器的。本质上，其对协程的支持是通过关键字的展开和模板来实现的。

这里的所有代码例子来自博客 [My tutorial and take on C++20 coroutines (stanford.edu)](https://www.scs.stanford.edu/~dm/blog/c++-coroutines.html)

&nbsp;

## co_await & Coroutine handles

C++ 提供了这个新的关键字 `co_await`，在其之后接一个变量会导致

* 该函数中的栈帧管理是无栈协程的形状，也就是局部变量会分配在堆中的一块地方，栈中仅留下该变量（通常拥有对应栈帧的句柄）
* 编译器会将其展开，调用若干个函数，通过 C++ 强大的模板功能，能够支持自定义的初始化、suspend 操作、resume 操作等。
* 将该函数的栈帧交付给该变量来管理。

该变量通常为 `std::coroutine_handle<>` 类型。

一个协程句柄跟 C 指针一样，能够随意复制而不造成实际资源的复制或销毁。所以，在适当的时候需要手动通过 `coroutine_handle::destroy` 释放对应的资源。当资源被释放，它所指向的东西是未定义的。（其实就像个指向栈帧的指针）。

以下面的程序为例：

```c++
#include <concepts>
#include <coroutine>
#include <exception>
#include <iostream>

struct ReturnObject {
	struct promise_type {
		ReturnObject get_return_object() { 
			return {}; 
		}

		std::suspend_never initial_suspend() { 
			return {}; 
		}

		std::suspend_never final_suspend() noexcept { 
			return {}; 
		}

        void unhandled_exception() {}
    };
};

struct Awaiter {
	std::coroutine_handle<> *hp_;

	// If it returns true, then co_await does not suspend the function.
    constexpr bool await_ready() const noexcept { 
		return false; 
	}

	// Every time await_suspend is called, the handle will be the same.
	// So the *hp_ will be assigned only once.
    void await_suspend(std::coroutine_handle<> h) { 
		if (hp_) {
			*hp_ = h;
			hp_ = nullptr;
		}
	}

	// Here the function await_resume return nothing, but the return value of co_await is
	// will be what the function returns.
    constexpr void await_resume() const noexcept {}
};

// Every call co_await, it will invoke the function await_suspend of Awaiter.
// Because the handler will not change, so the iterator i keep the state before suspending.
ReturnObject counter(std::coroutine_handle<> *continuation_out) {
	Awaiter a{continuation_out};

	for (unsigned i = 0;; ++i) {
		co_await a;
		std::cout << "counter: " << i << std::endl;
	}
}

int main() {
	// A coroutine_handle is like a pointer. It points to the content of suspended functions.
	std::coroutine_handle<> h;

	counter(&h);
	for (int i = 0; i < 3; ++i) {
		std::cout << "In main1 function\n";
		h();
	}
	// Without GC, C++ will not release resources the handle has, 
	// so we need to release it manually.
	h.destroy();
}
```

我们可以康康 `counter` 在编译器中展开成的样子

```c++
// Every call co_await, it will invoke the function await_suspend of Awaiter.
// Because the handler will not change, so the iterator i keep the state before suspending.
struct __counterFrame
{
  void (*resume_fn)(__counterFrame *);
  void (*destroy_fn)(__counterFrame *);
  std::__coroutine_traits_impl<ReturnObject>::promise_type __promise;
  int __suspend_index;
  bool __initial_await_suspend_called;
  std::coroutine_handle<void> * continuation_out;
  Awaiter a;
  unsigned int i;
  std::suspend_never __suspend_48_14;
  Awaiter __suspend_52_12;
  std::suspend_never __suspend_48_14_1;
};

ReturnObject counter(std::coroutine_handle<void> * continuation_out)
{
  /* Allocate the frame including the promise */
  __counterFrame * __f = reinterpret_cast<__counterFrame *>(operator new(__builtin_coro_size()));
  __f->__suspend_index = 0;
  __f->__initial_await_suspend_called = false;
  __f->continuation_out = std::forward<std::coroutine_handle<void> *>(continuation_out);
  
  /* Construct the promise. */
  new (&__f->__promise)std::__coroutine_traits_impl<ReturnObject>::promise_type{};
  
  ReturnObject __coro_gro = __f->__promise.get_return_object() /* NRVO variable */;
  
  /* Forward declare the resume and destroy function. */
  void __counterResume(__counterFrame * __f);
  void __counterDestroy(__counterFrame * __f);
  
  /* Assign the resume and destroy function pointers. */
  __f->resume_fn = &__counterResume;
  __f->destroy_fn = &__counterDestroy;
  
  /* Call the made up function with the coroutine body for initial suspend.
     This function will be called subsequently by coroutine_handle<>::resume()
     which calls __builtin_coro_resume(__handle_) */
  __counterResume(__f);
  
  
  return __coro_gro;
}

/* This function invoked by coroutine_handle<>::resume() */
void __counterResume(__counterFrame * __f)
{
  try 
  {
    /* Create a switch to get to the correct resume point */
    switch(__f->__suspend_index) {
      case 0: break;
      case 1: goto __resume_counter_1;
      case 2: goto __resume_counter_2;
    }
    
    /* co_await insights.cpp:48 */
    __f->__suspend_48_14 = __f->__promise.initial_suspend();
    if(!__f->__suspend_48_14.await_ready()) {
      __f->__suspend_48_14.await_suspend(std::coroutine_handle<ReturnObject::promise_type>::from_address(static_cast<void *>(__f)).operator coroutine_handle());
      __f->__suspend_index = 1;
      __f->__initial_await_suspend_called = true;
      return;
    } 
    
    __resume_counter_1:
    __f->__suspend_48_14.await_resume();
    __f->a = {__f->continuation_out};
    for( __f->i = 0; ; ++__f->i) {
      
      /* co_await insights.cpp:52 */
      __f->__suspend_52_12 = __f->a;
      if(!__f->__suspend_52_12.await_ready()) {
        __f->__suspend_52_12.await_suspend(std::coroutine_handle<ReturnObject::promise_type>::from_address(static_cast<void *>(__f)).operator coroutine_handle());
        __f->__suspend_index = 2;
        return;
      } 
      
      __resume_counter_2:
      __f->__suspend_52_12.await_resume();
      std::operator<<(std::cout, "counter: ").operator<<(__f->i).operator<<(std::endl);
    }
    
    goto __final_suspend;
  } catch(...) {
    if(!__f->__initial_await_suspend_called) {
      throw ;
    } 
    
    __f->__promise.unhandled_exception();
  }
  
  __final_suspend:
  
  /* co_await insights.cpp:48 */
  __f->__suspend_48_14_1 = __f->__promise.final_suspend();
  if(!__f->__suspend_48_14_1.await_ready()) {
    __f->__suspend_48_14_1.await_suspend(std::coroutine_handle<ReturnObject::promise_type>::from_address(static_cast<void *>(__f)).operator coroutine_handle());
  } 
  
  ;
}

/* This function invoked by coroutine_handle<>::destroy() */
void __counterDestroy(__counterFrame * __f)
{
  /* destroy all variables with dtors */
  __f->~__counterFrame();
  /* Deallocating the coroutine frame */
  operator delete(__builtin_coro_free(static_cast<void *>(__f)));
}
```

看起来还是蛮复杂的，不过其实就是加入了若干个函数调用：

* 在进入 counter 函数后，它首先创建其对应的栈（20-24行），其中会存储该协程的状态信息
* 然后为函数的返回值做成一个 promise 类型（C++ 的promise 类型主要用于异步协作中的阻塞）（26-29行）
* 声明并存储当前协程的 resume、destroy 函数 （31-37行），定义在函数之外。
* 调用 resume 函数，找到函数的入口并跳转
	* 假如是第一次调用（`__f->__suspend_index == 0`），则需要做一些初始化操作（60-67行）
	* 可以看到 `co_await` 将整个程序分成两半，一个是 co_await 之前的东西（`__resume_counter_1` 和 `__resume_counter_2` 之间的东西）和 co_await 之后的东西 （`__resume_counter_2` 和 `__final_suspend` 之间的东西）。
	* 在所有程序的入口处，会调用 `await_ready()` 决定是否 suspend，当返回 true 时，则协程不会 suspend。同时能够看到，当 suspend 的时会设置 `__f->__suspend_index` 决定下一次运行的入口。在 suspend 当前协程前，会执行 `await_suspend()` 函数。
	* 在运行结束之后，会运行 `final_suspend()` 函数做最后处理，同样会调用 `await_ready()` 决定是否 suspend，当然这时就不需要设置入口了。
* 最后执行 destroy 时，会执行对应栈帧的析构函数并将其释放。



能够发现 `Awaiter` （也就是跟在 `co_await` 后的变量）的作用：

* 其 `await_ready`，返回一个 bool 值，决定是否 suspend。
* 其 `await_suspend`，每次决定要 suspend 后，在 suspend 之前调用。会传入一个 handle。
* 其 `await_resume` 会在每次协程 suspend 返回后被调用。

这里我们还能够看到 `ReturnObject` 的作用：

* 其中的 `__f->__promise` 就是指向该类型的东西（在构建 promise 类型时确定的）。
* 其 `get_return_object()` 函数将在调用 resume 之前调用
* 其 `initial_suspend()` 函数将在第一次进入 resume 时调用
* 其 `final_suspend()` 将在 resume 运行结束，做最后处理时调用
* 其 `unhandled_exception()` 将在执行函数内容时接收到异常时调用



`<coroutine>` 提供了两个预定义的等待程序，`std::suspend_always` 和 `std::suspend_never`。顾名思义，`suspend_always::await_ready` 总是返回 false，而 `suspend_never::await_ready` 总是返回 true。这些类的其他方法是空的，什么也不做。



![Coroutine_process](/images/Coroutine/Coroutine_process.jpg)


当然，在第一次调用 `await_suspend` 确定了 handle 之后，其内容不会再改变，因此可以不用再赋值了。如下程序改变 `await_suspend` 之后仍然能够正常运行。

```c++
void await_suspend(std::coroutine_handle<> h) { 
    if (hp_) {
        *hp_ = h;
        hp_ = nullptr;
    }
}
```



但是到目前为止，我们的协程和主程序之间没有任何的数据交流，这对协程合作没有什么意义。我们可以通过 C++ 的 promise/future 机制来进行协程间的数据交流。

```c++
#include <concepts>
#include <coroutine>
#include <exception>
#include <iostream>

template <typename PromiseType>
struct GetPromise {
	PromiseType *p_;
	bool		 await_ready() { return false; } // says yes call await_suspend
	bool		 await_suspend(std::coroutine_handle<PromiseType> h) {
		p_ = &h.promise();
		return false; // says no don't suspend coroutine after all
	}
	PromiseType *await_resume() { return p_; }
};

struct ReturnObject3 {
	struct promise_type {
		unsigned value_;

		ReturnObject3 get_return_object() {
			return ReturnObject3{
				.h_ = std::coroutine_handle<promise_type>::from_promise(*this)};
		}
		std::suspend_never initial_suspend() { return {}; }
		std::suspend_never final_suspend() noexcept { return {}; }
		void			   unhandled_exception() {}
	};

	std::coroutine_handle<promise_type> h_;
	operator std::coroutine_handle<promise_type>() const { return h_; }
};

ReturnObject3 counter3() {
	auto pp = co_await GetPromise<ReturnObject3::promise_type>{};

	for (unsigned i = 0;; ++i) {
		pp->value_ = i;
		co_await std::suspend_always{};
	}
}

int main() {
	std::coroutine_handle<ReturnObject3::promise_type> h	   = counter3();
	ReturnObject3::promise_type						  &promise = h.promise();
	for (int i = 0; i < 3; ++i) {
		std::cout << "counter3: " << promise.value_ << std::endl;
		h();
	}
	h.destroy();

	return 0;
}
```

协程状态包含一个 `promise_type` 的实例，我们可以向该类型添加一个字段 `value_` 并使用该字段将值从协程传输到我们的主函数。在 main 函数中访问这个值。我们可以将其保留为 `std::coroutine_handle<ReturnObject3::promise_type>`，而不是将协程句柄转换为 `std::coroutine_handle<>`。然后通过方法 `promise() ` 将返回我们需要的 `promise_type&`。

协程自己该如何获取这个 promise 对象呢？也是类似的方法，如 23 行所示。

定义了一个新的等待者类型 `GetPromise`，它包含一个字段 `promise_type *p_`。我们让它的 `await_suspend` 方法将 promise 对象的地址存储在 `p_` 中，然后返回 false 以避免实际挂起协程。

到目前为止，我们只看到过 void 类型的 `co_await` 表达式。这一次，我们希望我们的 `co_await` 返回 promise 对象的地址，所以我们还添加了一个返回 `p_` 的 `await_resume` 函数。从而再调用 counter 时能够获得对应的 handle 对象，并从中提取出 promise。

&nbsp;

## co_yield

通过 `co_yield`，我们能够简化这个 promise 类型的获取过程，而无需显式的在函数中存储这个值，更加符合思维习惯地 “返回” 对应的值。

```c++
#include <concepts>
#include <coroutine>
#include <exception>
#include <iostream>

struct ReturnObject4 {
	struct promise_type {
		unsigned value_;

		ReturnObject4 get_return_object() {
			return {
				.h_ = std::coroutine_handle<promise_type>::from_promise(*this)};
		}
		std::suspend_never	initial_suspend() { return {}; }
		std::suspend_never	final_suspend() noexcept { return {}; }
		void				unhandled_exception() {}

	};

	std::coroutine_handle<promise_type> h_;
};

ReturnObject4 counter4() {
	for (unsigned i = 0;; ++i)
		co_yield i; // co yield i => co_await promise.yield_value(i)
}

int main() {
	auto  h		  = counter4().h_;
	auto &promise = h.promise();
	for (int i = 0; i < 3; ++i) {
		std::cout << "counter4: " << promise.value_ << std::endl;
		h();
	}
	h.destroy();
	return 0;
}
```

&nbsp;

## co_return

到目前为止，我们的主函数在读取前三个整数后简单地销毁了协程状态。但是假如主程序并不知道具体子程序需要生产多少个数据，具体细节交给子协程去把握，我只要在协程发出结束信号后再结束这个过程该怎么办捏。

这个时候就是 `co_return ` 的工作了，有三种做法能够发出信号：

* 调用 `co_return e;` ，则能够返回值 `e`。要求具有 `return_value(e)` 函数。
* 调用 `co_return;`，则返回 void。要求具有 `return_void()` 函数。
* 不调用 `co_return` 让其自行结束

能够注意到前两个条件都要求有对应的声明函数（得益于模板的特性，当不需要用到时不需要声明），假如缺少声明，则会在编译期就报错。值得注意的是，`promise_type::return_void()` 和 `promise_type::return_value(e)`这两个函数都是返回 void。

> This is presumably out of a desire to unify handling of return values and exceptions.



```c++
#include <concepts>
#include <coroutine>
#include <exception>
#include <iostream>

struct ReturnObject5 {
	struct promise_type {
		unsigned value_;

		~promise_type() { std::cout << "promise_type destroyed" << std::endl; }
		ReturnObject5 get_return_object() {
			return {
				.h_ = std::coroutine_handle<promise_type>::from_promise(*this)};
		}
		std::suspend_never	initial_suspend() { return {}; }
		std::suspend_always final_suspend() noexcept { return {}; }
		void				unhandled_exception() {}
		std::suspend_always yield_value(unsigned value) {
			value_ = value;
			return {};
		}
		void return_void() {}
	};

	std::coroutine_handle<promise_type> h_;
};

ReturnObject5 counter5() {
	for (unsigned i = 0; i < 3; ++i)
		co_yield i;
	// falling off end of function or co_return; => promise.return_void();
	// (co_return value; => promise.return_value(value);)
    
    // co_return; // Which requires return_void function.
    
    // co_return ...; // Which requires return_value function.
}

int main() {
	auto  h		  = counter5().h_;
	auto &promise = h.promise();
	while (!h.done()) { // Do NOT use while(h) (which checks h non-NULL)
		std::cout << "counter5: " << promise.value_ << std::endl;
		h();
	}
	h.destroy();
	return 0;
}
```



当协程返回时，将会隐式地等待 `promise.final_suspend()` 的结果。

如果 `final_suspend` 真的挂起协程，那么协程状态会最后一次更新并保持有效，协程外的代码将负责通过调用协程句柄的 `destroy()` 方法释放协程对象。

如果 `final_suspend` 没有挂起协程，那么协程状态会自动销毁。

因此假如将 `ReturnObject5::promise_type::final_suspend()` 更改为返回 `std::suspend_never`，那协程运行结束后不会阻塞，直接释放了对象。因此 `h.done()` 会导致未定义行为，一般直接就报错了。

&nbsp;

## 总结

C++20 的协程还是蛮有意思的，不过感觉框架比较简单。一些东西还不是特别方便和符合思维，当然可能因为协程较线程的积累并不是那么深厚。

&nbsp;
&nbsp;

# Reference

[1] [My tutorial and take on C++20 coroutines (stanford.edu)](https://www.scs.stanford.edu/~dm/blog/c++-coroutines.html)

[2] [C++20 Coroutine实例教学 - 腾讯云开发者社区-腾讯云 (tencent.com)](https://cloud.tencent.com/developer/article/1894832)

[3] [async/await异步模型是否优于stackful coroutine模型？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/65647171/answer/233495694)

[4] [理解有栈无栈协程 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/65956116)

[5] [协程 - 维基百科，自由的百科全书 (wikipedia.org)](https://zh.wikipedia.org/wiki/协程)