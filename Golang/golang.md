- [GoLang](#golang)
	- [GMP模型](#gmp模型)
		- [用户线程和系统线程的区别](#用户线程和系统线程的区别)
		- [GMP单独介绍](#gmp单独介绍)
		- [P和M的数量](#p和m的数量)
		- [P和M是什么时候被创建的](#p和m是什么时候被创建的)
		- [调度器的设计策略](#调度器的设计策略)
		- [调度流程](#调度流程)
	- [协程交替打印1-20](#协程交替打印1-20)
		- [使用channel](#使用channel)
		- [使用锁](#使用锁)
	- [Golang GC](#golang-gc)
		- [堆和栈](#堆和栈)
		- [设计原理](#设计原理)
			- [标记清除](#标记清除)
			- [三色抽象](#三色抽象)
				- [不足](#不足)
			- [屏障机制](#屏障机制)
				- [混合写屏障(Go 1.8+)](#混合写屏障go-18)
	- [Mutex的实现原理](#mutex的实现原理)
		- [正常模式和饥饿模式](#正常模式和饥饿模式)
		- [互斥锁的状态(state)](#互斥锁的状态state)
		- [Lock()](#lock)
			- [判断当前Goroutine能否进入自旋](#判断当前goroutine能否进入自旋)
			- [通过自旋等待互斥锁的释放](#通过自旋等待互斥锁的释放)
			- [计算互斥锁的最新状态](#计算互斥锁的最新状态)
			- [更新互斥锁的状态并获取锁](#更新互斥锁的状态并获取锁)

# GoLang

## GMP模型

> Refs:
> - https://www.programmersought.com/article/98681581962/
> - https://www.jianshu.com/p/fa696563c38a
> - https://blog.csdn.net/u010182186/article/details/77473547

### 用户线程和系统线程的区别

**系统线程**是由内核控制，当线程进行切换的时候，由用户态转化为内核态，切换完毕要从内核态返回用户态，可以利用多核CPU。**用户线程**内核的切换由用户态程序自己控制内核切换,不需要内核干涉，少了进出内核态的消耗，但不能很好的利用多核CPU。

![Golang调度器整体模型](ANDQLx3g9U.png)

为了既利用系统线程对于多核心CPU的优化，又为了使用用户线程的轻量级进程切换，因此Golang使用协程调度器来实现上述要求。

### GMP单独介绍

G是GoRoutine的缩写，包含函数执行指令和一些参数，比如任务对象、线程上下文切换、必须的寄存器的字段保护和恢复（field protection and field recovery required registers）等。

M是一个线程(thread)或者机器(machine)的缩写，所有的线程共享一个线程栈，如果你没有给线程栈分配内存，系统会自动分配内存。一个M中，包括一个指向g的指针，一个sp指针和一个pc指针，分别用户现场保护和现场恢复。当线程栈被指定后，G.stack = M.stack，且M的PC寄存器会指向G提供的函数，然后执行该函数。

P是一个抽象的概念，不代表一个真正的CPU核心。P会创建或者唤醒一个系统线程去执行队列中的任务。P决定同时并行(simultaneous concurrent)的任务数量，`GOMAXPROCS`限制了操作系统线程，它可以同时执行用户级任务。

![与系统调度器的关系](11093205-a79d5ae002d02d72.webp)

Goroutine调度器和OS调度器是通过M结合起来的，每个M都代表了1个内核线程，OS调度器负责把内核线程分配到CPU的核上执行。

### P和M的数量

P的数量理论上是不限制的，但是实际情况下P的数量最好与核心数相同。而M的数量在Golang中最大限制为10000，但是在实际使用中则通过runtime提供的`SetMaxThreads()`方法来设置。

### P和M是什么时候被创建的

在确定了几个P之后P立即被创建，M是则是采用懒加载的方案。

### 调度器的设计策略

复用线程：避免频繁的创建、销毁线程，而是对线程的复用。
- work stealing机制: 当本线程无可运行的G时，尝试从其他线程绑定的P偷取G，而不是销毁线程。
- hand off机制: 当本线程因为G进行系统调用阻塞时，线程释放绑定的P，把P转移给其他空闲的线程执行。

利用并行：GOMAXPROCS设置P的数量，最多有GOMAXPROCS个线程分布在多个CPU上同时运行。GOMAXPROCS也限制了并发的程度，比如GOMAXPROCS = 核数/2，则最多利用了一半的CPU核进行并行。

抢占：在coroutine中要等待一个协程主动让出CPU才执行下一个协程，在Go中，一个goroutine最多占用CPU 10ms，防止其他goroutine被饿死，这就是goroutine不同于coroutine的一个地方。

全局G队列：在新的调度器中依然有全局G队列，但功能已经被弱化了，当M执行work stealing从其他P偷不到G时，它可以从全局G队列获取G。

### 调度流程

![调度流程](11093205-9bc6e1e4a6d9d341.webp)

## 协程交替打印1-20

### 使用channel

> Ref: https://blog.csdn.net/Gusand/article/details/99442113

```go
package main

import (
	"fmt"
)

func main() {
	A := make(chan bool, 1)
	B := make(chan bool)
	Exit := make(chan bool)

	go func() {
		for i := 0; i < 20; i++ {
			if i%2 != 0 {
				continue
			}
			if ok := <-A; ok {
				fmt.Println(i)
				B <- true
			}
		}
	}()

	go func() {
		defer func() {
			close(Exit)
		}()
		for i := 0; i < 20; i++ {
			if i%2 == 0 {
				continue
			}
			if ok := <-B; ok {
				fmt.Println(i)
				A <- true
			}
		}
	}()

	A <- true
	<-Exit
}
```

### 使用锁

> Ref: https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/#mutex

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var oddMut sync.Mutex
	var evenMut sync.Mutex
	var wg sync.WaitGroup

	oddMut.Lock()
	wg.Add(2)

	go func() {
		defer wg.Done()
		for i := 0; i < 20; i++ {
			if i%2 != 0 {
				continue
			}
			evenMut.Lock()
			fmt.Println(i)
			oddMut.Unlock()
		}
	}()

	go func() {
		defer wg.Done()
		for i := 0; i < 20; i++ {
			if i%2 == 0 {
				continue
			}
			oddMut.Lock()
			fmt.Println(i)
			evenMut.Unlock()
		}
	}()

	wg.Wait()
}
```

## Golang GC

> Refs: 
> - [垃圾收集器](https://draveness.me/golang/docs/part3-runtime/ch07-memory/golang-garbage-collector/)
> - [C/C++的内存分配？栈和堆的区别？为什么栈快？](https://blog.csdn.net/baidu_37964071/article/details/81428139)
> - [Golang三色标记、混合写屏障GC模式图文全分析](https://segmentfault.com/a/1190000022030353)

### 堆和栈

栈区(stack)由编译器自动分配释放，存放为运行函数而分配的局部变量、函数参数、返回数据、返回地址等。其操作方式类似于数据结构中的栈；

堆区(heap)一般由程序员分配释放，若程序员不释放，程序结束时可能由OS回收。分配方式类似于链表。

### 设计原理

![垃圾回收架构图](2020-03-16-15843705141774-mutator-allocator-collector.png)

用户程序（Mutator）会通过内存分配器（Allocator）在**堆**上申请内存，而垃圾收集器（Collector）负责回收堆上的内存空间，内存分配器和垃圾收集器**共同管理**着程序中的堆内存空间。

#### 标记清除

顾名思义，分为标记（Mark）和清除（Sweep）两个阶段：

- 标记阶段：从根对象出发查找并标记堆中所有存活的对象；
- 清除阶段：遍历堆中的全部对象，回收未被标记的垃圾对象并将回收的内存加入空闲链表。

从根对象出发依次遍历对象的子对象并将从根节点可达的对象都标记成**存活状态**，不可达的标记为**垃圾**。如下图A是根，CD都是可达的，BEF都会被标记为垃圾。

![](2020-03-16-15843705141797-mark-sweep-mark-phase.png)

标记阶段结束后会进入清除阶段，释放其中没有被标记的BEF三个对象并将新的空闲内存空间**以链表的结构串联**起来，方便内存分配器的使用。

缺点是标记清除阶段需要暂停程序的运行，即STW，需要一种更先进的方法来改进。

#### 三色抽象

为了缩短STW的时间，Golang使用三种颜色标记对象：

- 白色对象：潜在的垃圾，其内存可能会被垃圾收集器回收；
- 黑色对象：活跃的对象，包括不存在任何引用外部指针的对象以及从根对象可达的对象；
- 灰色对象：活跃的对象，因为存在指向白色对象的外部指针，垃圾收集器会扫描这些对象的子对象；

这样做的好处是整个GC过程可以以并发的形式来做。

![三色抽象](2020-03-16-15843705141814-tri-color-mark-sweep.png)

在垃圾收集器开始工作时，程序中不存在任何的黑色对象，垃圾收集的根对象会被标记成灰色，垃圾收集器只会从灰色对象集合中取出对象开始扫描，当灰色集合中不存在任何对象时，标记阶段就会结束，也就是说在标记阶段结束时，只有黑色和白色两种对象。

![](2020-03-16-15843705141821-tri-color-mark-sweep-after-mark-phase.png)

因为用户程序可能在标记执行的过程中修改对象的指针，所以三色标记清除算法本身是不可以并发或者增量执行的，它仍然需要 STW，在如下所示的三色标记过程中，用户程序建立了从 A 对象到 D 对象的引用，但是因为程序中已经不存在灰色对象了，所以 D 对象会被垃圾收集器错误地回收。

![](2020-03-16-15843705141828-tri-color-mark-sweep-and-mutator.png)

##### 不足

普通的三色抽象机制是需要STW的，否在在同时满足以下条件时三色抽象机制会错误的把不该回收的对象回收：

- 一个白色对象被黑色对象引用(白色被挂在黑色下)
- 灰色对象与它之间的可达关系的白色对象遭到破坏(灰色同时丢了该白色)

但是STW对于性能的影响是非常显著的，因此需要有一种方案既能解决上述问题，而且也不能过多的影响性能。解决思路也非常的简单，即破坏同时满足上述两个条件的可能。因此诞生了屏障机制。

#### 屏障机制

避免上述的问题需要满足的要求是：

- 强三色不变式：不存在黑色对象引用到白色对象的指针；![](992905300-c6a852e9583c76cb_fix732.jpeg)
- 弱三色不变式：所有被黑色对象引用的白色对象都处于灰色保护状态![](1795230545-4937ebb20519cc34_fix732.jpeg)

强三色不变式避免了白色被挂在黑色下，而弱三色不变式保证所有的白色节点还有被扫描到的可能性。

插入屏障是指**在A对象引用B对象的时候，会将B对象标记为灰色**，即满足了强三色不变式。但是插入屏障仅仅会作用于栈空间，不会作用于堆空间，因为要保证栈的运行效率。在每一阶段的结束后需要一个STW来重新扫描堆空间中的元素。

删除屏障是指如果被删除的对象自身为白色，那么被标记为灰色。这样做满足弱三色不变式。这种方式不区分栈和堆，而且一个对象即使被删除了最后一个指向它的指针也依旧可以活过这一轮，在下一轮GC中被清理掉。

##### 混合写屏障(Go 1.8+)

目前屏障机制的问题是：

- 插入写屏障：结束时需要STW来重新扫描栈，标记栈上引用的白色对象的存活；
- 删除写屏障：回收精度低，<u>GC开始时STW扫描堆栈来记录初始快照，这个过程会保护开始时刻的所有存活对象。（这个地方我没有理解）</u>

混合写屏障的规则：

- GC开始将栈上的对象全部扫描并标记为黑色(之后不再进行第二次重复扫描，无需STW)；
- **GC期间**，任何在栈上创建的新对象，均为黑色；
- 被删除的对象标记为灰色；
- 被添加的对象标记为灰色。

## Mutex的实现原理

> 看这部分知识前，需要先理解锁的相关知识，详见[Lock](../Lock/lock.md)。
> 
> Refs: 
> - [同步原语与锁](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-sync-primitives/)
> - [sync.Mutex互斥锁的实现原理](https://segmentfault.com/a/1190000023874384)
> - [golang中的Mutex设计原理详解（一）](https://zhuanlan.zhihu.com/p/339981535)
> - [Go 中的 Mutex 设计原理详解（二）](https://zhuanlan.zhihu.com/p/341887600)
> - [Go 中的 Mutex 设计原理详解（三）](https://zhuanlan.zhihu.com/p/342706674)
> - [Go 中的 Mutex 设计原理详解（四）](https://zhuanlan.zhihu.com/p/344977623)

Go的Mutex结构体构成如下，其中state标志互斥锁的状态，sema标志信号量。

```go
type Mutex struct {
	state int32
	sema  uint32
}
```

### 正常模式和饥饿模式

在正常模式下，锁的等待者会按照先进先出的顺序获取锁。如果有Goroutine在1ms以内没有竞争到锁后，则从正常模式切换为饥饿模式，防止部分Goroutine被“饿死”。

### 互斥锁的状态(state)

![](2020-01-23-15797104328010-golang-mutex-state.png)

在默认情况下，互斥锁的所有状态位都是 0，int32中的不同位分别表示了不同的状态：

- mutexLocked：表示互斥锁的锁定状态；
- mutexWoken：表示从正常模式被从唤醒；
- mutexStarving：当前的互斥锁进入饥饿状态；
- waitersCount：当前互斥锁上等待的Goroutine个数；

### Lock()

Mutex的Lock()方法如下，首先尝试获取state的`mutexLocked`状态，如果为0则表明获到取锁并返回。如果获取不到锁，则执行`m.lockSlow()`方法，进入自旋锁。

互斥锁的加锁过程比较复杂，它涉及自旋、信号量以及调度等概念：

- 如果互斥锁处于初始化状态，会通过置位 mutexLocked 加锁；
- 如果互斥锁处于 mutexLocked 状态并且在普通模式下工作，会进入自旋，执行 30 次 PAUSE 指令消耗 CPU 时间等待锁的释放；
- 如果当前 Goroutine 等待锁的时间超过了 1ms，互斥锁就会切换到饥饿模式；
- 互斥锁在正常情况下会通过 runtime.sync_runtime_SemacquireMutex 将尝试获取锁的 Goroutine 切换至休眠状态，等待锁的持有者唤醒；
- 如果当前 Goroutine 是互斥锁上的最后一个等待的协程或者等待的时间小于 1ms，那么它会将互斥锁切换回正常模式；

互斥锁的解锁过程与之相比就比较简单，其代码行数不多、逻辑清晰，也比较容易理解：

- 当互斥锁已经被解锁时，调用 sync.Mutex.Unlock 会直接抛出异常；
- 当互斥锁处于饥饿模式时，将锁的所有权交给队列中的下一个等待者，等待者会负责设置 mutexLocked 标志位；
- 当互斥锁处于普通模式时，如果没有 Goroutine 等待锁的释放或者已经有被唤醒的 Goroutine 获得了锁，会直接返回；在其他情况下会通过 sync.runtime_Semrelease 唤醒对应的 Goroutine；

```go
func (m *Mutex) Lock() {
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		return
	}
	m.lockSlow()
}
```

对于lockSlow()方法，主要包含四个部分：

- 判断当前Goroutine能否进入自旋；
- 通过自旋等待互斥锁的释放；
- 计算互斥锁的最新状态；
- 更新互斥锁的状态并获取锁。

#### 判断当前Goroutine能否进入自旋

如果自旋锁进入的时机不好，会非常影响整体的运行速度，因此自旋锁进入的条件非常苛刻：

- 互斥锁只有在普通模式才能进入自旋；
- `runtime.sync_runtime_canSpin`需要返回true：
  - 运行在多CPU的机器上；(自旋锁会不断占用CPU)
  - 当前Goroutine为了获取该锁进入自旋的次数小于四次；
  - 当前机器上至少存在一个正在运行的处理器P并且处理的运行队列为空；

#### 通过自旋等待互斥锁的释放

一旦当前Goroutine能够进入自旋就会调用`runtime.sync_runtime_doSpin`和`runtime.procyield`并执行30次的PAUSE指令，该指令只会占用CPU并消耗CPU时间。

#### 计算互斥锁的最新状态

略。

```go
// 将旧状态赋值给新状态
new := old
// 现在和原来都不是饥饿模式
if old & mutexStarving == 0 {
	// 新状态为锁定状态，如果old是锁定状态或者mutexLocked是锁定状态
	// 新状态为不锁定状态，如果old和mutexLocked同时为不锁定状态
	new |= mutexLocked
}
// (mutexLocked | mutexStarving) = 0，如果既是不锁定状态且正常模式
// old & (mutexLocked | mutexStarving) != 0，
if old & (mutexLocked | mutexStarving) != 0 {
	new += 1 << mutexWaiterShift
}
if starving && old&mutexLocked != 0 {
	 |= mutexStarving
}
if awoke {
	new &^= mutexWoken
}
```

#### 更新互斥锁的状态并获取锁

计算了新的互斥锁状态之后，会使用CAS函数`sync/atomic.CompareAndSwapInt32`更新状态。
