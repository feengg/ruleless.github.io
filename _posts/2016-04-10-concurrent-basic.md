---
layout: post
title: "并发编程基础"
description: ""
category: concurrent
tags: [并发, 原子操作]
---
{% include JB/setup %}

## 多线程及并发

线程是操作系统进行作业调度的最小单位，也是进程内部的一条执行路径。
与进程不同，线程并没有对操作系统的资源所有权，也就是说同一个进程内的多个线程对资源的访问权是共享的。
一个进程中的所有线程共享一个地址空间或者诸如打开的文件之类的其他资源，一个进程对这些资源的任何修改，都会影响到本进程中其他线程的运行。
因此，需要对多个线程的执行进行细致地设计，使它们能够互不干涉，并且不破坏共享的数据及资源。

在单处理器中的并发系统里，不同进程和线程之间的指令流是交替执行的，
但由于调度系统及CPU时钟的配合，使得程序对外表现出一种同时执行的外部特征；
而在并行多处理器系统中，指令流之间的执行则是重叠的。
无论是交替执行还是重叠执行，实际上并发程序都面临这同样的问题，
即指令流的执行速度不可预测，这取决于其他指令流的活动状态、操作系统处理中断的方式及操作系统的调度策略。
这给并发程序设计带来了如下的一些问题：

  1. 多个进程（或线程）对同一全局资源的访问可能造成未定义的后果。
     例如，如果两个并发线程都使用同一个全局变量，并且都对该变量执行读写操作，
	 那么程序执行的结果就取决于不同的读写执行顺序——而这些读写执行顺序是不可预知的。
  2. 操作系统难以对资源进行最优化分配。这涉及到死锁及饥饿的问题。
  3. 很难定位程序的错误。在多数情况下，并发程序设计的失误都是很难复现的，在一次执行中出现一种结果，而在下一次执行中，往往会出现迥然不同的其他结果。

因此，在进行多线程程序或者并发程序的设计时，尤其需要小心。
可以看到的是，绝大多数并发程序的错误都出现在对共享资源的访问上，
因此，如何保证对共享资源的访问以一种确定的、我们可以预知的方式运行，成为并发程序设计的首要问题。
在操作系统领域，对共享资源的访问有个专用的数据，称为临界区。

临界区是一段代码，在这段代码中，进程将访问共享资源。
当另外一个进程已经在这段代码中执行时，这个进程就不能在这段代码中执行。

也就是说，临界区是一个代码段，这个代码段不允许两条并行的指令流同时进入。
提供这种保证的机制称为互斥：当一个进程在临界区访问共享资源时，其他进程不能进入该临界区。

## 锁及互斥

实现互斥的机制，最重要的是互斥锁（mutex）。
互斥锁实际上是一种二元信号量（只有0和1），专用于多任务之间临界区的互斥操作。
（关于信号量及互斥锁的区别，可以参看操作系统相关知识）

mutex本质上是一个信号量对象，只有0和1两个值。
同时，mutex还对信号量加1和减1的操作进行了限制，即某个线程对其进行了+1操作，则-1操作也必须由这个线程来完成。
mutex的两个值也分别代表了mutex的两种状态。
值为0, 表示锁定状态，当前对象被锁定，用户进程/线程如果试图Lock临界资源，则进入排队等待；
值为1，表示空闲状态，当前对象为空闲，用户进程/线程可以Lock临界资源，之后mutex值减1变为0。

mutex可以抽象为创建（Create），加锁（Lock），解锁（Unlock），及销毁（Destroy）等四个操作。
在创建mutex时，可以指定锁的状态是空闲或者是锁定，在linux中，这个属性的设置主要通过 `pthread_mutex_init` 来实现。

在使用mutex的时候，务必需要了解其本质：mutex实际上是一个在多个线程之间共享的信号量，当其进入锁定状态时，再试图对其加锁，则会阻塞线程。
例如，对于两个线程A和B，其指令序列如下：

线程A

  1. lock(&mutex)
  2. do something
  3. unlock(&mutex)

线程B

  1. lock(&mutex)
  2. do something
  3. unlock(&mutex)

在线程A的语句1处，线程A对mutex进行了加锁操作，mutex变为锁定状态。
在线程A的语句2及线程B的语句1处，A尚未对mutex进行解锁，
而B则试图对mutex进行加锁操作，因此线程B被阻塞，直到A的语句3处，线程A对mutex进行了解锁，B的语句1才得以继续执行，将mutex进行加锁并继续执行语句2和语句3。
因此，如果在do something中有对共享资源的访问操作，那么do something就是一个临界区，每次都只有一个线程能够进入这段代码。

## 原子操作

无论是信号量，还是互斥锁，其中最重要的一个概念就是原子操作。
所谓原子操作，就是不会被线程调度机制所打断的操作——从该操作的第一条指令开始到最后一条指令结束，
中间不会有任何的上下文切换（context switch）。

在单处理器系统上，原子操作的实现较为简单：第一种方式是一些单指令即可完成的操作，
如 `compare and swap`、`test and set` 等，由于上下文切换只可能出现在指令之间，因此单处理器系统上的单指令操作都是原子操作；
另一种方式则是禁用中断，通过汇编语言支持，在指令执行期间，禁用处理器的所有中断操作，
由于上下文切换都是通过中断来触发的，因此禁用中断后，可以保证指令流的执行不会被外部指令所打断。

而在多处理器系统上，情况要复杂一些。由于系统中有多个处理器在独立地运行，即使能在单条指令中完成的操作也有可能受到干扰。
如，在不同的CPU运行的两个线程都在执行一条递减指令，即对内存中某个内存单元的值-1，则指令流水线可能是这样：（省略了取指）

![](/images/concurrent/concurrentop.png)

假设原来内存单元中存储的值为5，那么：A、B处理器所读到的内存值都为5，其往内存单元中写入的值都为4。
因此，虽然进行了两次-1操作，但实际上运行的结果和执行了1次是一样的。

> 注：这是一个数据相关问题（关于数据相关问题，可以参考计算机体系结构中指令流水线的设计及数据相关的避免等资料），
在单处理机中，这个问题可以通过检查处理机中的指令寄存器，来检查在流水线中的指令之间的相关性，如果出现数据相关的情况，可以通过延迟相关指令执行的方法来规避；
而在对称多处理机中，由于CPU之间相互不知道对方的指令寄存器状态，那么这种流水线作业引起的数据竞跑就无法避免。

为了对原子操作提供支持，在 x86 平台上，CPU提供了在指令执行期间对总线加锁的手段。
CPU芯片上有一条引线#HLOCK pin，如果汇编语言的程序中在一条指令前面加上前缀"LOCK"，
经过汇编以后的机器代码就使CPU在执行这条指令的时候把#HLOCK pin的电位拉低，持续到这条指令结束时放开，从而把总线锁住，
这样同一总线上别的CPU就暂时不能通过总线访问内存了，保证了这条指令在多处理器环境中的原子性。

可以看出，其实 `pthread_mutex_lock` 及 `pthread_mutex_unlock` 就是一个原子操作。
它保证了两个线程不会同时对某个mutex变量加锁或者解锁，否则的话，互斥也就无从实现了。

i++和++i是原子操作吗？

有一个很多人也许都不是很清楚的问题：i++或++i是一个原子操作吗？
在上一节，其实已经提到了，在SMP（对称多处理器）上，即使是单条递减汇编指令，其原子性也是不能保证的。
那么在单处理机系统中呢？

在编译器对C/C++源代码进行编译时，往往会进行一些代码优化。
例如，对i++这条指令，实际上编译器编译出的汇编代码是类似下面的汇编语句：

    1. mov eax,[i]
	2. add eax,1
	3. mov [i],eax

语句1是将i所在的内存读取到寄存器中，而语句2是将寄存器的值加1，语句3是将寄存器值写回到内存中。
之所以进行这样的操作，是为了CPU访问数据效率的高效。
可以看出，i++是由一条语句被编译成了3条指令，因此，即使在单处理机系统上，i++这种操作也不是原子的。
这是由于指令之间的乱序执行而造成的。

## GCC的内建原子操作

在GCC中，从版本4.1.2起，提供了 `__sync_` 系列的 `built-in` 函数，用于提供加减和逻辑运算的原子操作。
这些操作通过锁定总线，无论在单处理机和多处理机上都保证了其原子性。GCC提供的原子操作主要包括：

``` c++
type __sync_fetch_and_add (type *ptr, type value, ...);
type __sync_fetch_and_sub (type *ptr, type value, ...);
type __sync_fetch_and_or (type *ptr, type value, ...);
type __sync_fetch_and_and (type *ptr, type value, ...);
type __sync_fetch_and_xor (type *ptr, type value, ...);
type __sync_fetch_and_nand (type *ptr, type value, ...);
```

这六个函数的作用是：取得ptr所指向的内存中的数据,同时对ptr中的数据进行修改操作(加,减,或,与,异或,与后取非)等。

``` c++
type __sync_add_and_fetch (type *ptr, type value, ...);
type __sync_sub_and_fetch (type *ptr, type value, ...);
type __sync_or_and_fetch (type *ptr, type value, ...);
type __sync_and_and_fetch (type *ptr, type value, ...);
type __sync_xor_and_fetch (type *ptr, type value, ...);
type __sync_nand_and_fetch (type *ptr, type value, ...);
```

这六个函数与上六个函数基本相同，不同之处在于，上六个函数返回值为修改之前的数据，而这六个函数返回的值为修改之后的数据。

``` c++
bool __sync_bool_compare_and_swap (type *ptr, type oldval type newval, ...);
type __sync_val_compare_and_swap (type *ptr, type oldval type newval, ...);
```

比较并交换指令。如果ptr所指向的内存中的数据等于oldval，则设置其为newval，同时返回true；否则返回false。

``` c++
type __sync_lock_test_and_set (type *ptr, type value, ...);
```

测试并置位指令。

``` c++
void __sync_lock_release (type *ptr, ...);
```

将ptr设置为0。

其中，这些操作的操作数（type）可以是1,2,4或8字节长度的int类型，即：

  1. `int8_t / uint8_t`
  2. `int16_t / uint16_t`
  3. `int32_t / uint32_t`
  4. `int64_t / uint64_t`