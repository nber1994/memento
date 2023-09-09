---
title: go-GMP模型 
date: 2020-07-02
categories: 
- go 
---

# 上古时代

>  一切要从1946年说起，当世界第一台计算机问世时，世界拥有了第一台计算机。（废话文学）

![埃尼阿克(eniac)](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-20211106180745370.png)

当时的埃尼阿克(eniac)是为了计算炮弹轨迹而发明的，将你要计算的程序打到纸带上，接通电源，等待执行结果。期间不能暂停，也不能做其他任何事情。它上面没有操作系统，更别提进程、线程和协程了。



# 进程时代



![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-03-apple-II.jpeg)

## 单进程时代

后来，现代化的计算机有了操作系统，每个程序都是一个进程，但是操作系统在一段时间只能运行一个进程，直到这个进程运行完，才能运行下一个进程，这个时期可以成为**单进程时代——串行时代**。

和ENIAC相比，单进程是有了几万倍的提度，但依然是太慢了，比如进程要读数据阻塞了，CPU就在哪浪费着，伟大的程序员们就想了，不能浪费啊，**怎么才能充分的利用CPU呢？**

## 多进程时代

后来操作系统就具有了**最早的并发能力：多进程并发**，当一个进程阻塞的时候，切换到另外等待执行的进程，这样就能尽量把CPU利用起来，CPU就不浪费了。

# 线程时代

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-03-Macintosh.jpeg)

多进程真实个好东西，有了对进程的调度能力之后，伟大的程序员又发现，进程拥有太多资源，在创建、切换和销毁的时候，都会占用很长的时间，CPU虽然利用起来了，但CPU有很大的一部分都被用来进行进程调度了，**怎么才能提高CPU的利用率呢？**

大家希望能有一种轻量级的进程，调度不怎么花时间，这样CPU就有更多的时间用在执行任务上。

后来，操作系统支持了线程，线程在进程里面，线程运行所需要资源比进程少多了，跟进程比起来，切换简直是“不算事”。

一个进程可以有多个线程，CPU在执行调度的时候切换的是线程，如果下一个线程也是当前进程的，就只有线程切换，“很快”就能完成，如果下一个线程不是当前的进程，就需要切换进程，这就得费点时间了。

这个时代，**CPU的调度切换的是进程和线程**。多线程看起来很美好，但实际多线程编程却像一坨屎，一是由于线程的设计本身有点复杂，而是由于需要考虑很多底层细节，比如锁和冲突检测。



# 进程 & 线程的问题

### 从一道面试题谈起

> 进程和线程的分别是什么？

> 对于操作系统层面，标准的回答是：进程是资源分配的最小单位，线程是cpu调度的最小单位。

## 进程

简而言之，进程是资源分配的最小单位，且是一个应用的实例，为了保证进程间不互相影响，各搞各的，进程间：

1. 数据完全隔离
2. 指令流完全隔离

具体怎么实现的：

### 虚拟内存

进程启动时，操作系统会为进程分配一块内存空间，进程的视角看这块内存是连续的，但是实际上在机器内存中是一块块的内存碎片，这个假象是由虚拟内存技术实现的。这么做是为了方便编译链接，内存隔离等。

![image-20211106173230814](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-20211106173230814.png)

进程需要访问某个地址时，例如0x004005，都会先去查询页表，定位到对应的PTE（页表项），PTE记录着对应的真实的物理内存地址，继而再对这个真实地址进行访问。

每个进程都会有相同的页表，且内存结构也相同，对于每个进程来说都认为自己占用了机器全部的内存，操作系统使用虚拟内存保证进程间的内存是隔离的。

### 进程内存地址空间

32位机器：

![image-20211106170342935](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/image-20211106170342935.png)

从低位向高位看：

- 代码段总是从地址0x400000开始的，存储着用户程序的代码，字面量等
- 然后是数据段，存储用户程序的一些宏，全局变量，代码等
- 之后是堆，程序运行过程中产生的变量等都会存放在堆中，且堆空间向高位地址延伸
- 在用户栈和堆之间存在一块为共享库分配的内存空间，这里存储着例如.so动态链接文件等
- 再之后是用户栈，函数调用的入参，临时变量，以及调用信息等存放于此
- 内存最高位则是内核内存空间，用户栈是从2^48-1处开始向下延伸的，这段空间固定映射到一部分连续的物理地址上

#### 进程控制块 PCB

内核为每个进程维护了一个pcb的结构，主要用于记录进程状态，在Linux中，PCB结构为task_struct；

task_struct是Linux内核的一种数据结构，它会被装载到RAM里并且包含进程的信息，每个进程都把它的信息放在task_struct这个数据结构里。那么这个结构体里存了哪些数据呢：

**1.进程状态：是调度和兑换的依据**

| linux进程的状态      |                    |
| -------------------- | ------------------ |
| 内核表示             | 含义               |
| TASK_RUNNING         | 可运行             |
| TASK_INTERRUPTIBLE   | 可中断的等待状态   |
| TASK_UNINTERRUPTIBLE | 不可中断的等待状态 |
| TASK_ZOMBIE          | 僵死               |
| TASK_STOPPED         | 暂停               |
| TASK_SWAPPING        | 换入/换出          |

**2.标识符：描述本进程的唯一标识符，用来区别其它进程**

每个进程都有一个唯一的标识符，内核通过这个标识符来识别不同的进程，同时，进程标识符PID也是内核提供给用户程序的接口，用户程序通过PID对进程发号施令。PID是32位的无符号整数，它被顺序编号：新创建进程的PID通常是前一个进程的PID加1。然而，为了与16位硬件平台的传统Linux系统保持兼容，在Linux上允许的最大PID号是32767，当内核在系统中创建第32768个进程时，就必须重新开始使用已闲置的PID号。

| 各种标识符   |                                      |
| ------------ | ------------------------------------ |
| 域名         | 含义                                 |
| pid          | 进程标识符                           |
| ppid         | 父进程                               |
| uid、gid     | 用户标识符、组标识符                 |
| euid、egid   | 有效用户标识符、有效组标识符         |
| suid、sgid   | 备份用户标识符、备份组标识符         |
| fsuid、fsgid | 文件系统用户标识符、文件系统组标识符 |

**3.进程调度信息**

调度程序利用这部分信息决定系统中哪个进程应该优先运行，并结合进程的状态信息保证系统运转的公平和高效。这一部分信息通常包括进程的类别（普通进程还是实时进程）、进程的优先级（priority）等等

| 进程调度信息 |            |
| ------------ | ---------- |
| 域名         | 含义       |
| need_resched | 调度标志   |
| nice         | 静态优先级 |
| counter      | 动态优先级 |
| policy       | 调度策略   |
| rt_priority  | 实时优先级 |

**4.程序计数器：程序中即将被执行的下一条指令的地址**

**5.内存指针：包括程序代码和进程相关数据指针，还有和其他进程共享的内存块的指针**

**6.与处理器相关的上下文数据：程序执行时处理器的寄存器中的数据**

**7.I/O状态信息：包括显示的I/O请求，分配给进程的I/O设备和被进程使用的文件列表**

**8.记账信息：可以包括处理器时间总和，使用的时钟数总和、时间限制、记账号等**

那么我们看到，内核为每个进程维护的PCB中，保存了每个进程当前的程序计数器，每当本进程被唤醒执行时，会将当前进程PCB信息载入寄存器中，并会将当前的程序计数器放到CS/IP上继续执行，这也就保证了 **进程指令流的安全切换** 。



## 线程





# 协程

![Dieci anni di MacBook Air, la rivoluzione silenziosa - macitynet.it](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/steve-jobs-macbook-air.jpg)

多进程、多线程已经提高了系统的并发能力，但是在当今互联网高并发场景下，为每个任务都创建一个线程是不现实的，因为会消耗大量的内存（每个线程的内存占用级别为MB），线程多了之后调度也会消耗大量的CPU。伟大的程序员们有开始想了，**如何才能充分利用CPU、内存等资源的情况下，实现更高的并发**？

既然线程的资源占用、调度在高并发的情况下，依然是比较大的，是否有一种东西，更加轻量？

你可能知道：线程分为内核态线程和用户态线程，用户态线程需要绑定内核态线程，CPU并不能感知用户态线程的存在，它只知道它在运行1个线程，这个线程实际是内核态线程。

**用户态线程实际有个名字叫协程（co-routine）**，为了容易区分，我们使用协程指用户态线程，使用线程指内核态线程。

协程跟线程是有区别的，线程由CPU调度是抢占式的，**协程由用户态调度是协作式的**，一个协程让出CPU后，才执行下一个协程。

协程和线程有3种映射关系：

- N:1，N个协程绑定1个线程，优点就是**协程在用户态线程即完成切换，不会陷入到内核态，这种切换非常的轻量快速**。但也有很大的缺点，1个进程的所有协程都绑定在1个线程上，一是某个程序用不了硬件的多核加速能力，二是一旦某协程阻塞，造成线程阻塞，本进程的其他协程都无法执行了，根本就没有并发的能力了。
- 1:1，1个协程绑定1个线程，这种最容易实现。协程的调度都由CPU完成了，不存在N:1缺点，但有一个缺点是协程的创建、删除和切换的代价都由CPU完成，有点略显昂贵了。
- M:N，M个协程绑定N个线程，是N:1和1:1类型的结合，克服了以上2种模型的缺点，但实现起来最为复杂。

协程是个好东西，不少语言支持了协程，比如：Lua、Erlang、Java（C++即将支持），就算语言不支持，也有库支持协程，比如C语言的[coroutine](https://github.com/cloudwu/coroutine/)（云风大牛作品）、Kotlin的kotlinx.coroutines、Python的gevent。



# goroutine

**Go语言的诞生就是为了支持高并发**，有2个支持高并发的模型：CSP和Actor。[鉴于Occam和Erlang都选用了CSP](https://golang.org/doc/faq)(来自Go FAQ)，并且效果不错，Go也选了CSP，但与前两者不同的是，Go把channel作为头等公民。



> 两类通用并发模型：参考七周七并发模型

- > 共享内存型Shared Memory

  - > 线程Threads

  - > 锁Locks

  - > 互斥l量Mutexes

- > 消息传送型（CSP和Actor模型）

  - > 进程Processes

  - > 消息Messages

  - > 不共享数据(状态)No shared data

> 重点介绍消息传送型的两种模型Actor和CSP（Communicating Sequential Process）的各项对比

> [https://segmentfault.com/a/1190000038526143](https://segmentfault.com/a/1190000038526143)



就像前面说的多线程编程太不友好了，**Go为了提供更容易使用的并发方法，使用了goroutine和channel**。goroutine来自协程的概念，让一组可复用的函数运行在一组线程之上，即使有协程阻塞，该线程的其他协程也可以被`runtime`调度，转移到其他可运行的线程上。最关键的是，程序员看不到这些底层的细节，这就降低了编程的难度，提供了更容易的并发。

Go中，协程被称为goroutine（Rob Pike说goroutine不是协程，因为他们并不完全相同），它非常轻量，一个goroutine只占几KB，并且这几KB就足够goroutine运行完，这就能在有限的内存空间内支持大量goroutine，支持了更多的并发。虽然一个goroutine的栈只占几KB，但实际是可伸缩的，如果需要更多内容，`runtime`会自动为goroutine分配。





## 协程调度 & 线程调度的性能差别

线程切换大概在几微秒级别，协程切换大概在百 ns 级别。

线程切换过程:

1. 进入系统调用
2. 调度器本身代码执行
3. 线程上下文切换: PC, SP 等寄存器，栈，线程相关的一些局部变量，还涉及一些 cache miss 的情况；
4. 退出系统调用

协程切换不需要进入和退出系统调用, 在进行上下文切换时也更轻量, 只需要切换几个寄存器, 协程 `runtime.g` 结构只有 40 多个字段, 而线程的 task struct 有大概 300 个字段.

[进程/线程上下文切换会用掉你多少CPU？](https://zhuanlan.zhihu.com/p/79772089)

[协程究竟比线程能省多少开销？](https://zhuanlan.zhihu.com/p/80037638)





# 调度器

**调度器的任务是在用户态完成goroutine的调度，而调度器的实现好坏，对并发实际有很大的影响，并且Go的调度器就是M:N类型的，实现起来也是最复杂**。

Goroutine的并发编程模型基于GMP模型，简要解释一下GMP的含义：

**G**:表示goroutine，每个goroutine都有自己的栈空间，定时器，初始化的栈空间在2k左右，空间会随着需求增长。

**M**:抽象化代表内核线程，记录内核线程栈信息，当goroutine调度到线程时，使用该goroutine自己的栈信息。

**P**:代表调度器，负责调度goroutine，维护一个本地goroutine队列，M从P上获得goroutine并执行，同时还负责部分内存的管理。



## 老版本调度器

现在的Go语言调度器是2012年重新设计的（[设计方案](https://golang.org/s/go11sched)），在这之前的调度器称为老调度器，老调度器的实现不太好，存在性能问题，所以用了4年左右就被替换掉了，老调度器大概是下面这个样子：

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-03-old-scheduler.png)

最下面是操作系统，中间是runtime，runtime在Go中很重要，许多程序运行时的工作都由runtime完成，调度器就是runtime的一部分，虚线圈出来的为调度器，它有两个重要组成：

- **M，代表线程**，它要运行goroutine。
- Global G Queue，是全局goroutine队列，所有的goroutine都保存在这个队列中，**goroutine用G进行代表**。

M想要执行、放回G都必须访问全局G队列，并且M有多个，即多线程访问同一资源需要加锁进行保证互斥/同步，所以全局G队列是有互斥锁进行保护的。

老调度器有4个缺点：

1. 创建、销毁、调度G都需要每个M获取锁，这就形成了**激烈的锁竞争**。
2. M转移G会造成**延迟和额外的系统负载**。比如当G中包含创建新协程的时候，M创建了G’，为了继续执行G，需要把G’交给M’执行，也造成了**很差的局部性**，因为G’和G是相关的，最好放在M上执行，而不是其他M’。
3. M中的mcache是用来存放小对象的，mcache和栈都和M关联造成了大量的内存开销和差的局部性。
4. 系统调用导致频繁的线程阻塞和取消阻塞操作增加了系统开销。



## 新版本调度器

面对以上老调度的问题，Go设计了新的调度器，设计文稿：https://golang.org/s/go11sched

新调度器引入了：

- **P**：**Processor，它包含了运行goroutine的资源**，如果线程想运行goroutine，必须先获取P，P中还包含了可运行的G队列。
- work stealing：当M绑定的P没有可运行的G时，它可以从其他运行的M’那里偷取G。

现在，**调度器中3个重要的缩写你都接触到了，所有文章都用这几个缩写，请牢记**：

- **G**: goroutine
- **M**: 工作线程
- **P**: 处理器，它包含了运行Go代码的资源，M必须和一个P关联才能运行G。

这篇文章的目的不是介绍调度器的实现，而是调度器的一些理念，帮助你后面更好理解调度器的实现，所以我们回归到调度器设计思想上。

![thoughts-of-scheduler](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-03-thoughts-of-scheduler.png)

调度器的有**两大思想**：

**复用线程**：协程本身就是运行在一组线程之上，不需要频繁的创建、销毁线程，而是对线程的复用。在调度器中复用线程还有2个体现：1）work stealing，当本线程无可运行的G时，尝试从其他线程绑定的P偷取G，而不是销毁线程。2）hand off，当本线程因为G进行系统调用阻塞时，线程释放绑定的P，把P转移给其他空闲的线程执行。

**利用并行**：GOMAXPROCS设置P的数量，当GOMAXPROCS大于1时，就最多有GOMAXPROCS个线程处于运行状态，这些线程可能分布在多个CPU核上同时运行，使得并发利用并行。另外，GOMAXPROCS也限制了并发的程度，比如GOMAXPROCS = 核数/2，则最多利用了一半的CPU核进行并行。

调度器的**两小策略**：

**抢占**：在coroutine中要等待一个协程主动让出CPU才执行下一个协程，在Go中，一个goroutine最多占用CPU 10ms，防止其他goroutine被饿死，这就是goroutine不同于coroutine的一个地方。

**全局G队列**：在新的调度器中依然有全局G队列，但功能已经被弱化了，当M执行work stealing从其他P偷不到G时，它可以从全局G队列获取G。



> 上面提到并行了，关于并发和并行再说一下：Go创始人Rob Pike一直在强调go是并发，不是并行，因为Go做的是在一段时间内完成几十万、甚至几百万的工作，而不是同一时间同时在做大量的工作。**并发可以利用并行提高效率，调度器是有并行设计的**。

>并行依赖多核技术，每个核上在某个时间只能执行一个线程，当我们的CPU有8个核时，我们能同时执行8个线程，这就是并行。

> ![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-03-concurrency-parallelism.png)

## Scheduler的宏观组成

[Tony Bai](https://tonybai.com/)在[《也谈goroutine调度器》](https://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)中的这幅图，展示了goroutine调度器和系统调度器的关系，而不是把二者割裂开来，并且从宏观的角度展示了调度器的重要组成。

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-03-goroutine-scheduler-model.png)



自顶向下是调度器的4个部分：

1. **全局队列**（Global Queue）：存放等待运行的G。
2. **P的本地队列**：同全局队列类似，存放的也是等待运行的G，存的数量有限，不超过256个。新建G’时，G’优先加入到P的本地队列，如果队列满了，则会把本地队列中一半的G移动到全局队列。
3. **P列表**：所有的P都在程序启动时创建，并保存在数组中，最多有GOMAXPROCS个。
4. **M**：线程想运行任务就得获取P，从P的本地队列获取G，P队列为空时，M也会尝试从全局队列**拿**一批G放到P的本地队列，或从其他P的本地队列**偷**一半放到自己P的本地队列。M运行G，G执行之后，M会从P获取下一个G，不断重复下去。

**Goroutine调度器和OS调度器是通过M结合起来的，每个M都代表了1个内核线程，OS调度器负责把内核线程分配到CPU的核上执行**。

### 调度器的生命周期

接下来我们从另外一个宏观角度——生命周期，认识调度器。

所有的Go程序运行都会经过一个完整的调度器生命周期：从创建到结束。

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-03-scheduler-lifetime.png)

即使下面这段简单的代码：

```
package main

import "fmt"

// main.main
func main() {
	fmt.Println("Hello scheduler")
}
```

也会经历如上图所示的过程：

1. runtime创建最初的线程m0和goroutine g0，并把2者关联。
2. 调度器初始化：初始化m0、栈、垃圾回收，以及创建和初始化由GOMAXPROCS个P构成的P列表。
3. 示例代码中的main函数是`main.main`，`runtime`中也有1个main函数——`runtime.main`，代码经过编译后，`runtime.main`会调用`main.main`，程序启动时会为`runtime.main`创建goroutine，称它为main goroutine吧，然后把main goroutine加入到P的本地队列。
4. 启动m0，m0已经绑定了P，会从P的本地队列获取G，获取到main goroutine。
5. G拥有栈，M根据G中的栈信息和调度信息设置运行环境
6. M运行G
7. G退出，再次回到M获取可运行的G，这样重复下去，直到`main.main`退出，`runtime.main`执行Defer和Panic处理，或调用`runtime.exit`退出程序。

调度器的生命周期几乎占满了一个Go程序的一生，`runtime.main`的goroutine执行之前都是为调度器做准备工作，`runtime.main`的goroutine运行，才是调度器的真正开始，直到`runtime.main`结束而结束。



### GMP的可视化感受

上面的两个宏观角度，都是根据文档、代码整理出来，最后我们从可视化角度感受下调度器，有2种方式。

**方式1：go tool trace**

trace记录了运行时的信息，能提供可视化的Web页面。

简单测试代码：main函数创建trace，trace会运行在单独的goroutine中，然后main打印”Hello trace”退出。

```
func main() {
	// 创建trace文件
	f, err := os.Create("trace.out")
	if err != nil {
		panic(err)
	}
	defer f.Close()

	// 启动trace goroutine
	err = trace.Start(f)
	if err != nil {
		panic(err)
	}
	defer trace.Stop()

	// main
	fmt.Println("Hello trace")
}
```

运行程序和运行trace：

```
➜  trace git:(master) ✗ go run trace1.go
Hello trace
➜  trace git:(master) ✗ ls
trace.out trace1.go
➜  trace git:(master) ✗
➜  trace git:(master) ✗ go tool trace trace.out
2019/03/24 20:48:22 Parsing trace...
2019/03/24 20:48:22 Splitting trace...
2019/03/24 20:48:22 Opening browser. Trace viewer is listening on http://127.0.0.1:55984
```

效果：

![trace1](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-03-go-tool-trace.png)



从上至下分别是goroutine（G）、堆、线程（M）、Proc（P）的信息，从左到右是时间线。用鼠标点击颜色块，最下面会列出详细的信息。

我们可以发现：

- `runtime.main`的goroutine是`g1`，这个编号应该永远都不变的，`runtime.main`是在`g0`之后创建的第一个goroutine。
- g1中调用了`main.main`，创建了`trace goroutine g18`。g1运行在P2上，g18运行在P0上。
- P1上实际上也有goroutine运行，可以看到短暂的竖线。

go tool trace的资料并不多，如果感兴趣可阅读：https://making.pusher.com/go-tool-trace/ ，中文翻译是：https://mp.weixin.qq.com/s/nf_-AH_LeBN3913Pt6CzQQ 。

**方式2：Debug trace**

示例代码：

```
// main.main
func main() {
	for i := 0; i < 5; i++ {
		time.Sleep(time.Second)
		fmt.Println("Hello scheduler")
	}
}
```

编译和运行，运行过程会打印trace：

```
➜  one_routine2 git:(master) ✗ go build .
➜  one_routine2 git:(master) ✗ GODEBUG=schedtrace=1000 ./one_routine2
```

结果：

```
SCHED 0ms: gomaxprocs=8 idleprocs=5 threads=5 spinningthreads=1 idlethreads=0 runqueue=0 [0 0 0 0 0 0 0 0]
SCHED 1001ms: gomaxprocs=8 idleprocs=8 threads=5 spinningthreads=0 idlethreads=3 runqueue=0 [0 0 0 0 0 0 0 0]
Hello scheduler
SCHED 2002ms: gomaxprocs=8 idleprocs=8 threads=5 spinningthreads=0 idlethreads=3 runqueue=0 [0 0 0 0 0 0 0 0]
Hello scheduler
SCHED 3004ms: gomaxprocs=8 idleprocs=8 threads=5 spinningthreads=0 idlethreads=3 runqueue=0 [0 0 0 0 0 0 0 0]
Hello scheduler
SCHED 4005ms: gomaxprocs=8 idleprocs=8 threads=5 spinningthreads=0 idlethreads=3 runqueue=0 [0 0 0 0 0 0 0 0]
Hello scheduler
SCHED 5013ms: gomaxprocs=8 idleprocs=8 threads=5 spinningthreads=0 idlethreads=3 runqueue=0 [0 0 0 0 0 0 0 0]
Hello scheduler
```

看到这密密麻麻的文字就有点担心，不要愁！因为每行字段都是一样的，各字段含义如下：

- SCHED：调试信息输出标志字符串，代表本行是goroutine调度器的输出；
- 0ms：即从程序启动到输出这行日志的时间；
- gomaxprocs: P的数量，本例有8个P；
- idleprocs: 处于idle状态的P的数量；通过gomaxprocs和idleprocs的差值，我们就可知道执行go代码的P的数量；
- threads: os threads/M的数量，包含scheduler使用的m数量，加上runtime自用的类似sysmon这样的thread的数量；
- spinningthreads: 处于自旋状态的os thread数量；
- idlethread: 处于idle状态的os thread的数量；
- runqueue=0： Scheduler全局队列中G的数量；
- `[0 0 0 0 0 0 0 0]`: 分别为8个P的local queue中的G的数量。

看第一行，含义是：刚启动时创建了8个P，其中5个空闲的P，共创建5个M，其中1个M处于自旋，没有M处于空闲，8个P的本地队列都没有G。

再看个复杂版本的，加上`scheddetail=1`可以打印更详细的trace信息。

命令：

```
➜  one_routine2 git:(master) ✗ GODEBUG=schedtrace=1000,scheddetail=1 ./one_routine2
```

结果：

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-03-for-print-syscall.png)
*截图可能更代码匹配不起来，最初代码是for死循环，后面为了减少打印加了限制循环5次*

每次分别打印了每个P、M、G的信息，P的数量等于`gomaxprocs`，M的数量等于`threads`，主要看圈黄的地方：

- 第1处：P1和M2进行了绑定。
- 第2处：M2和P1进行了绑定，但M2上没有运行的G。
- 第3处：代码中使用fmt进行打印，会进行系统调用，P1系统调用的次数很多，说明我们的用例函数基本在P1上运行。
- 第4处和第5处：M0上运行了G1，G1的状态为3（系统调用），G进行系统调用时，M会和P解绑，但M会记住之前的P，所以M0仍然记绑定了P1，而P1称未绑定M。



# 图解调度器

如果你已经阅读了前2篇文章：[《调度起源》](http://lessisbetter.site/2019/03/10/golang-scheduler-1-history/)和[《宏观看调度器》](http://lessisbetter.site/2019/03/26/golang-scheduler-2-macro-view/)，你对G、P、M肯定已经不再陌生，我们这篇文章就介绍Go调度器的基本原理，本文总结了12个主要的场景，覆盖了以下内容：

1. G的创建和分配。
2. P的本地队列和全局队列的负载均衡。
3. M如何寻找G。
4. M如何从G1切换到G2。
5. work stealing，M如何去偷G。
6. 为何需要自旋线程。
7. G进行系统调用，如何保证P的其他G’可以被执行，而不是饿死。
8. Go调度器的抢占。

### 12场景

> 提示：图在前，场景描述在后。

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-04-image-20190331190809649-4030489.png)

> 上图中三角形、正方形、圆形分别代表了M、P、G，正方形连接的绿色长方形代表了P的本地队列。

**场景1**：p1拥有g1，m1获取p1后开始运行g1，g1使用`go func()`创建了g2，为了局部性g2优先加入到p1的本地队列。

![img](https://lessisbetter.site/images/2019-04-image-20190331190826838-4030506.png)

**场景2**：**g1运行完成后(函数：`goexit`)，m上运行的goroutine切换为g0，g0负责调度时协程的切换（函数：`schedule`）**。从p1的本地队列取g2，从g0切换到g2，并开始运行g2(函数：`execute`)。实现了**线程m1的复用**。

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-04-image-20190331160718646-4019638.png)

**场景3**：假设每个p的本地队列只能存4个g。g2要创建了6个g，前4个g（g3, g4, g5, g6）已经加入p1的本地队列，p1本地队列满了。

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-04-image-20190331160728024-4019648.png)

> 蓝色长方形代表全局队列。

**场景4**：g2在创建g7的时候，发现p1的本地队列已满，需要执行**负载均衡**，把p1中本地队列中前一半的g，还有新创建的g**转移**到全局队列（实现中并不一定是新的g，如果g是g2之后就执行的，会被保存在本地队列，利用某个老的g替换新g加入全局队列），这些g被转移到全局队列时，会被打乱顺序。所以g3,g4,g7被转移到全局队列。

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-04-image-20190331161138353-4019898.png)

**场景5**：g2创建g8时，p1的本地队列未满，所以g8会被加入到p1的本地队列。

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-04-image-20190331162734830-4020854.png)

**场景6**：**在创建g时，运行的g会尝试唤醒其他空闲的p和m执行**。假定g2唤醒了m2，m2绑定了p2，并运行g0，但p2本地队列没有g，m2此时为自旋线程（没有G但为运行状态的线程，不断寻找g，后续场景会有介绍）。

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-04-image-20190331162717486-4020837.png)

**场景7**：m2尝试从全局队列(GQ)取一批g放到p2的本地队列（函数：`findrunnable`）。m2从全局队列取的g数量符合下面的公式：

```
n = min(len(GQ)/GOMAXPROCS + 1, len(GQ/2))
```

公式的含义是，至少从全局队列取1个g，但每次不要从全局队列移动太多的g到p本地队列，给其他p留点。这是**从全局队列到P本地队列的负载均衡**。

假定我们场景中一共有4个P，所以m2只从能从全局队列取1个g（即g3）移动p2本地队列，然后完成从g0到g3的切换，运行g3。

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2020-09-go-scheduler-p8.png)

**场景8**：假设g2一直在m1上运行，经过2轮后，m2已经把g7、g4也挪到了p2的本地队列并完成运行，全局队列和p2的本地队列都空了，如上图左边。

**全局队列已经没有g，那m就要执行work stealing：从其他有g的p哪里偷取一半g过来，放到自己的P本地队列**。p2从p1的本地队列尾部取一半的g，本例中一半则只有1个g8，放到p2的本地队列，情况如上图右边。

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/VK0PNS.jpg)

**场景9**：p1本地队列g5、g6已经被其他m偷走并运行完成，当前m1和m2分别在运行g2和g8，m3和m4没有goroutine可以运行，m3和m4处于**自旋状态**，它们不断寻找goroutine。为什么要让m3和m4自旋，自旋本质是在运行，线程在运行却没有执行g，就变成了浪费CPU？销毁线程不是更好吗？可以节约CPU资源。创建和销毁CPU都是浪费时间的，我们**希望当有新goroutine创建时，立刻能有m运行它**，如果销毁再新建就增加了时延，降低了效率。当然也考虑了过多的自旋线程是浪费CPU，所以系统中最多有GOMAXPROCS个自旋的线程，多余的没事做线程会让他们休眠（见函数：`notesleep()`）。

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/GurO8V.jpg)

**场景10**：假定当前除了m3和m4为自旋线程，还有m5和m6为自旋线程，g8创建了g9，g8进行了**阻塞的系统调用**，m2和p2立即解绑，p2会执行以下判断：如果p2本地队列有g、全局队列有g或有空闲的m，p2都会立马唤醒1个m和它绑定，否则p2则会加入到空闲P列表，等待m来获取可用的p。本场景中，p2本地队列有g，可以和其他自旋线程m5绑定。

**场景11**：（无图场景）g8创建了g9，假如g8进行了**非阻塞系统调用**（CGO会是这种方式，见`cgocall()`），m2和p2会解绑，但m2会记住p，然后g8和m2进入系统调用状态。当g8和m2退出系统调用时，会尝试获取p2，如果无法获取，则获取空闲的p，如果依然没有，g8会被记为可运行状态，并加入到全局队列。

**场景12**：（无图场景）Go调度在go1.12实现了抢占，应该更精确的称为**请求式抢占**，那是因为go调度器的抢占和OS的线程抢占比起来很柔和，不暴力，不会说线程时间片到了，或者更高优先级的任务到了，执行抢占调度。**go的抢占调度柔和到只给goroutine发送1个抢占请求，至于goroutine何时停下来，那就管不到了**。抢占请求需要满足2个条件中的1个：1）G进行系统调用超过20us，2）G运行超过10ms。调度器在启动的时候会启动一个单独的线程sysmon，它负责所有的监控工作，其中1项就是抢占，发现满足抢占条件的G时，就发出抢占请求。

### 场景融合

如果把上面所有的场景都融合起来，就能构成下面这幅图了，它从整体的角度描述了Go调度器各部分的关系。图的上半部分是G的创建、负债均衡和work stealing，下半部分是M不停寻找和执行G的迭代过程。

如果你看这幅图还有些似懂非懂，建议赶紧开始看[雨痕大神的Golang源码剖析](https://lessisbetter.site/2019/04/04/golang-scheduler-3-principle-with-graph/github.com/qyuhen/book)，章节：并发调度。

![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-04-arch.png)

总结，Go调度器和OS调度器相比，是相当的轻量与简单了，但它已经足以撑起goroutine的调度工作了，并且让Go具有了原生（强大）并发的能力，这是伟大的。如果你记住的不多，你一定要记住这一点：**Go调度本质是把大量的goroutine分配到少量线程上去执行，并利用多核并行，实现更强大的并发。**

# 源码

### 阅读前提

阅读Go源码前，最好已经掌握Go调度器的设计和原理，如果你还无法回答以下问题：

1. 为什么需要Go调度器？
2. Go调度器与系统调度器有什么区别和关系/联系？
3. G、P、M是什么，三者的关系是什么？
4. P有默认几个？
5. M同时能绑定几个P？
6. M怎么获得G？
7. M没有G怎么办？
8. 为什么需要全局G队列？
9. Go调度器中的负载均衡的2种方式是什么？
10. work stealing是什么？什么原理？
11. 系统调用对G、P、M有什么影响？
12. Go调度器抢占是什么样的？一定能抢占成功吗？



![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2019-04-arch-20211109194527274.png)

调度器整体流转流程



![img](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/gBUJAv.jpg)



````go
func schedinit() {
    // 设置最大 M 数量
    sched.maxmcount = 10000

    // 初始化栈空间复用管理链表
    stackinit()

    // 初始化当前 M
    mcommoninit(_g_.m)

    // 默认值总算从 1 调整为 CPU Core 数量了
    procs := int(ncpu)
    if n := atoi(gogetenv("GOMAXPROCS")); n > 0 {
        if n > _MaxGomaxprocs {
            n = _MaxGomaxprocs
        }
        procs = n
    }

    // 调整 P 数量
    // 注意：此刻所有 P 都是新建的，所以不可能返回有本地任务的 P
    if procresize(int32(procs)) != nil {
        throw("unknown runnable goroutine during bootstrap")
    }
}
````

```go
func procresize(nprocs int32) *p {
        old := gomaxprocs
        // 新增
        for i := int32(0); i < nprocs; i++ {
                pp := allp[i]
                // 申请新 P 对象
                if pp == nil {
                        pp = new(p)
                        pp.id = i
                        pp.status = _Pgcstop
                        // 保存到 allp
                        atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
                }
                // 为 P 分配 cache 对象
                if pp.mcache == nil {
                        if old == 0 && i == 0 {
                                // bootstrap
                                pp.mcache = getg().m.mcache
                        } else {
                                // 创建 cache
                                pp.mcache = allocmcache()
                        }
                }
        }

        // 释放多余的 P
        for i := nprocs; i < old; i++ {
                p := allp[i]
                // 将本地任务转移到全局队列
                for p.runqhead != p.runqtail {
                        p.runqtail--
                        gp := p.runq[p.runqtail%uint32(len(p.runq))]
                        globrunqputhead(gp)
                }
                if p.runnext != 0 {
                        globrunqputhead(p.runnext.ptr())
                        p.runnext = 0
                }
                // 释放当前 P 绑定的 cache
                freemcache(p.mcache)

                p.mcache = nil
                // 将当前 P 的 G 复用链转移到全局
                gfpurge(p)
                // 似乎就丢在那里不管了，反正也没剩下啥
                p.status = _Pdead
                // can't free P itself because it can be referenced by an M in syscall

        }

        _g_ := getg()
        // 如果当前正在用的 P 属于被释放的那拨，那就换成 allp[0]
        // 调度器初始化阶段，根本没有 P，那就绑定 allp[0]
        if _g_.m.p != 0 && _g_.m.p.ptr().id < nprocs {
                // 继续使用当前 P
                _g_.m.p.ptr().status = _Prunning
        } else {
                // 释放当前 P，因为它已经失效
                if _g_.m.p != 0 {
                        _g_.m.p.ptr().m = 0
                }
                _g_.m.p = 0
                _g_.m.mcache = nil
                // 换成 allp[0]
                p := allp[0]
                p.m = 0
                p.status = _Pidle
                acquirep(p)
        }
        // 将没有本地任务的 P 放到空闲链表
        var runnablePs *p
        for i := nprocs - 1; i >= 0; i-- {
                p := allp[i]
                // 确保不是当前正在用的 P
                if _g_.m.p.ptr() == p {
                        continue
                }
                p.status = _Pidle

                if runqempty(p) {
                        // 放入空闲链表
                        pidleput(p)
                } else {
                        // 有本地任务，构建链表
                        p.m.set(mget())
                        p.link.set(runnablePs)
                        runnablePs = p
                }
        }
        // 返回有本地任务的 P（链表）
        return runnablePs
}

// 将 P 放入空闲链表
func pidleput(_p_ *p) {
        _p_.link = sched.pidle
        sched.pidle.set(_p_)
        xadd(&sched.npidle, 1)
}
```

默认只有 schedinit 和 startTheWorld 会调用 procresize 函数。在调度器初始化阶段，所有 P 对象都是新建 的 。除分配给当前主线程的外，其他都被放入空闲链表。而 startTheWorld 会激活全部有本地任务的 P 对象



在完成调度器初始化后，引导过程才创建并运行 main goroutine。

asm_amd64.s

```
TEXT runtime.rt0_go(SB),NOSPLIT,$0
        // save m->g0 = g0
        MOVQ CX, m_g0(AX)
        // save m0 to g0->m
        MOVQ AX, g_m(CX)
        CALL runtime.schedinit(SB)
        // 创建 main goroutine，并将其放入当前 P 本地队列
        MOVQ $runtime.mainPC(SB), AX
        PUSHQ AX
        PUSHQ $0
        CALL runtime.newproc(SB)
        POPQ AX
        POPQ AX
        // 让当前 M0 进入调度，执行 main goroutine
        CALL runtime.mstart(SB)
        // M0 永远不会执行这条崩溃测试指令
        MOVL $0xf1, 0xf1 // crash
        RET
```

虽然可在运行期用 runtime.GOMAXPROCS 函数修改 P 的数量，但须付出极大代价。

debug.go

```

func GOMAXPROCS(n int) int {
        if n > _MaxGomaxprocs {
                n = _MaxGomaxprocs
        }
        // 返回当前值（这个才是最常用的做法）
        ret := int(gomaxprocs)
        if n <= 0 || n == ret {
                return ret
        }
        // STW !!!
        stopTheWorld("GOMAXPROCS")
        newprocs = int32(n)
        // 调用 procresize，并激活有任务的 P
        startTheWorld()
        return ret
}

```



我们已经知道编译器会将“go func(...)”语句翻译成 newproc 调用，但这中间究竟有什 么不为人知的秘密？

test.go

```go
package main

import ()

func add(x, y int) int {
	z := x + y
	return z
}

func main() {
 	x := 0x100
 	y := 0x200
 	go add(x, y)
}
```

```assembly
$ go build -o test test.go

$ go tool objdump -s "main\.main" test

TEXT main.main(SB) test.go
 test.go:10 SUBQ $0x28, SP
 test.go:11 MOVQ $0x100, CX
 test.go:12 MOVQ $0x200, AX
 test.go:13 MOVQ CX, 0x10(SP) 		// 实参 x 入栈
 test.go:13 MOVQ AX, 0x18(SP) 		// 实参 y 入栈
 test.go:13 MOVL $0x18, 0(SP) 		// 参数长度入栈
 test.go:13 LEAQ 0x879ff(IP), AX 	// 将函数 add 地址存入 AX 寄存器
 test.go:13 MOVQ AX, 0x8(SP) 			// 地址入栈
 test.go:13 CALL runtime.newproc(SB)
 test.go:14 ADDQ $0x28, SP
 test.go:14 RET
```

从反汇编代码可以看出，Go 采用了类似 C/cdecl 的调用约定。由调用方负责提供参数空 间，并从右往左入栈。

```go
type funcval struct {
    fn uintptr
    // variable-size, fn-specific data here
}


func newproc(siz int32, fn *funcval) {
    // 获取第一参数地址
    argp := add(unsafe.Pointer(&fn), ptrSize)
  
    // 获取调用方 PC/IP 寄存器值
    pc := getcallerpc(unsafe.Pointer(&siz))
  
    // 用 g0 栈创建 G/goroutine 对象
    systemstack(func() {
        newproc1(fn, (*uint8)(argp), siz, 0, pc)
    })
}

```



```go
type p struct {
        gfree    *g
        gfreecnt int32
}

func newproc1(fn *funcval, argp *uint8, narg int32, nret int32, callerpc uintptr) *g {
        _g_ := getg()

        //“参数 + 返回值”所需空间（对齐）
        siz := narg + nret
        siz = (siz + 7) &^ 7

        // 从当前 P 复用链表获取空闲的 G 对象
        _p_ := _g_.m.p.ptr()
        newg := gfget(_p_)

        // 获取失败，新建
        if newg == nil {
                newg = malg(_StackMin)
                casgstatus(newg, _Gidle, _Gdead)
                allgadd(newg)
        }

        // 测试 G stack
        if newg.stack.hi == 0 {
                throw("newproc1: newg missing stack")
        }

        // 测试 G status
        if readgstatus(newg) != _Gdead {
                throw("newproc1: new g is not Gdead")
        }

        // 计算所需空间大小，并对齐
        totalSize := 4*regSize + uintptr(siz)
        totalSize += -totalSize & (spAlign - 1)

        // 确定 SP 和参数入栈位置
        sp := newg.stack.hi - totalSize
        spArg := sp

        // 将执行参数拷贝入栈
        memmove(unsafe.Pointer(spArg), unsafe.Pointer(argp), uintptr(narg))

        // 初始化用于保存执行现场的区域
        memclr(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
        newg.sched.sp = sp
        newg.sched.pc = funcPC(goexit) + _PCQuantum
        newg.sched.g = guintptr(unsafe.Pointer(newg))
        gostartcallfn(&newg.sched, fn)

        // 初始化基本状态
        newg.gopc = callerpc
        newg.startpc = fn.fn
        casgstatus(newg, _Gdead, _Grunnable)

        // 设置唯一 id
        if _p_.goidcache == _p_.goidcacheend {
                // sched.goidgen 是一个全局计数器
                // 每次取回一段有效区间，然后在该区间分配，避免频繁地去全局操作

                // [sched.goidgen+1, sched.goidgen+GoidCacheBatch]
                _p_.goidcache = xadd64(&sched.goidgen, _GoidCacheBatch)
                _p_.goidcache -= _GoidCacheBatch - 1
                _p_.goidcacheend = _p_.goidcache + _GoidCacheBatch
        }

        newg.goid = int64(_p_.goidcache)
        _p_.goidcache++

        // 将 G 放入待运行队列
        runqput(_p_, newg, true)

        // 如果有其他空闲 P，则尝试唤醒某个 M 出来执行任务
        // 如果有 M 处于自旋等待 P 或 G 状态，放弃
        // 如果当前创建的是 main goroutine (runtime.main)，那么还没有其他任务需要执行，放弃
        if atomicload(&sched.npidle) != 0 &&
                atomicload(&sched.nmspinning) == 0 &&
                unsafe.Pointer(fn.fn) != unsafe.Pointer(funcPC(main)) {
                wakep()
        }

        return newg
}

```



```go
func gfget(_p_ *p) *g {
retry:
        // 从 P 本地队列提取复用对象
        gp := _p_.gfree
        // 如果提取失败，尝试从全局链表转移一批到 P 本地
        if gp == nil && sched.gfree != nil {
                // 最多转移 32 个
                for _p_.gfreecnt < 32 && sched.gfree != nil {
                        _p_.gfreecnt++
                        gp = sched.gfree
                        sched.gfree = gp.schedlink.ptr()
                        sched.ngfree--
                        gp.schedlink.set(_p_.gfree)
                        _p_.gfree = gp
                }
                // 再试
                goto retry
        }

        // 如果成功获取复用对象
        if gp != nil {
                // 调整 P 复用链表
                _p_.gfree = gp.schedlink.ptr()
                _p_.gfreecnt--
                // 检查 G stack
                if gp.stack.lo == 0 {
                        // 分配新栈
                        systemstack(func() {
                                gp.stack, gp.stkbar = stackalloc(_FixedStack)
                        })
                        gp.stackguard0 = gp.stack.lo + _StackGuard
                        gp.stackAlloc = _FixedStack
                } else {
                }
        }
        return gp
}

```

而当 goroutine 执行完毕，调度器相关函数会将 G 对象放回 P 复用链表。





创建完毕的 G 任务被优先放入 P 本地队列等待执行，这属于无锁操作。

```go
type schedt struct {
		runqhead guintptr
		runqtail guintptr
		runqsize int32
}
type p struct {
		runqhead uint32
		runqtail uint32
		runq [256]*g // 本地队列，访问时无须加锁
		runnext guintptr // 优先执行
}
type g struct {
		schedlink guintptr // 链表
}


func runqput(_p_ *p, gp *g, next bool) {
        if randomizeScheduler && next && fastrand1()%2 == 0 {
                next = false
        }

        // 如果可能，将 G 直接保存在 P.runnext，作为下一个优先执行任务
        if next {
        retryNext:
                oldnext := _p_.runnext
                if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
                        goto retryNext
                }
                if oldnext == 0 {
                        return
                }
                // 原本的 next G 会被放回本地队列
                gp = oldnext.ptr()
        }

retry:
        // runqhead 是一个数组实现的循环队列
        // head、tail 累加，通过取模即可获得索引位置，很典型的算法
        h := atomicload(&_p_.runqhead)
        t := _p_.runqtail
        // 如果本地队列未满，直接放到尾部
        if t-h < uint32(len(_p_.runq)) {
                _p_.runq[t%uint32(len(_p_.runq))] = gp
                atomicstore(&_p_.runqtail, t+1)
                return
        }
        // 放入全局队列
        // 因为需要加锁，所以 slow
        if runqputslow(_p_, gp, h, t) {
                return
        }
        goto retry
}

```

任务队列分为三级，按优先级从高到低分别是 P.runnext、P.runq、Sched.runq，很有些 CPU 多级缓存的意思。

往全局队列添加任务，显然需要加锁，只是专门取名为 runqputslow 就很有说法了。去 看看到底怎么个慢法

```go
func runqputslow(_p_ *p, gp *g, h, t uint32) bool {
// 这意思显然是要从 P 本地转移一半任务到全局队列
// "+1" 是别忘了当前这个 gp
var batch [len(_p_.runq)/2 + 1]*g
// 计算一半的实际数量
n := t - h
n = n / 2
// 从队列头部提取
for i := uint32(0); i < n; i++ {
batch[i] = _p_.runq[(h+i)%uint32(len(_p_.runq))]
}
// 调整 P 队列头部位置
if !cas(&_p_.runqhead, h, h+n) {
return false
}
// 加上当前 gp 这家伙
batch[n] = gp
// 对顺序进行洗牌
if randomizeScheduler {
for i := uint32(1); i <= n; i++ {
j := fastrand1() % (i + 1)
batch[i], batch[j] = batch[j], batch[i]
}
}
// 串成链表
for i := uint32(0); i < n; i++ {
batch[i].schedlink.set(batch[i+1])
}
// 添加到全局队列尾部
globrunqputbatch(batch[0], batch[n], int32(n+1))
return true
}

func globrunqputbatch(ghead *g, gtail *g, n int32) {
gtail.schedlink = 0
if sched.runqtail != 0 {
sched.runqtail.ptr().schedlink.set(ghead)
} else {
sched.runqhead.set(ghead)
}
sched.runqtail.set(gtail)
sched.runqsize += n
}
```

若本地队列已满，一次性转移半数到全局队列。这个好理解，因为其他 P 可能正饿着 呢。这也正好解释了 newproc1 最后尝试用 wakep 唤醒其他 M/P 去执行任务的意图，毕 竟充分发挥多核优势才是正途。



## 6.5.2 数据结构 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#652-数据结构)

相信各位读者已经对 Go 语言调度相关的数据结构已经非常熟悉了，但是我们在一些还是要回顾一下运行时调度器的三个重要组成部分 — 线程 M、Goroutine G 和处理器 P：

![golang-scheduler](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2020-02-05-15808864354595-golang-scheduler.png)

**图 6-29 Go 语言调度器**

1. G — 表示 Goroutine，它是一个待执行的任务；
2. M — 表示操作系统的线程，它由操作系统的调度器调度和管理；
3. P — 表示处理器，它可以被看做运行在线程上的本地调度器；

我们会在这一节中分别介绍不同的结构体，详细介绍它们的作用、数据结构以及在运行期间可能处于的状态。

### G [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#g)

Goroutine 是 Go 语言调度器中待执行的任务，它在运行时调度器中的地位与线程在操作系统中差不多，但是它占用了更小的内存空间，也降低了上下文切换的开销。

Goroutine 只存在于 Go 语言的运行时，它是 Go 语言在用户态提供的线程，作为一种粒度更细的资源调度单元，如果使用得当能够在高并发的场景下更高效地利用机器的 CPU。

Goroutine 在 Go 语言运行时使用私有结构体 [`runtime.g`](https://draveness.me/golang/tree/runtime.g) 表示。这个私有结构体非常复杂，总共包含 40 多个用于表示各种状态的成员变量，这里也不会介绍所有的字段，仅会挑选其中的一部分，首先是与栈相关的两个字段：

```go
type g struct {
	stack       stack
	stackguard0 uintptr
}
```

Go

其中 `stack` 字段描述了当前 Goroutine 的栈内存范围 [stack.lo, stack.hi)，另一个字段 `stackguard0` 可以用于调度器抢占式调度。除了 `stackguard0` 之外，Goroutine 中还包含另外三个与抢占密切相关的字段：

```go
type g struct {
	preempt       bool // 抢占信号
	preemptStop   bool // 抢占时将状态修改成 `_Gpreempted`
	preemptShrink bool // 在同步安全点收缩栈
}
```

Go

Goroutine 与我们在前面章节提到的 `defer` 和 `panic` 也有千丝万缕的联系，每一个 Goroutine 上都持有两个分别存储 `defer` 和 `panic` 对应结构体的链表：

```go
type g struct {
	_panic       *_panic // 最内侧的 panic 结构体
	_defer       *_defer // 最内侧的延迟函数结构体
}
```

Go

最后，我们再节选一些作者认为比较有趣或者重要的字段：

```go
type g struct {
	m              *m
	sched          gobuf
	atomicstatus   uint32
	goid           int64
}
```

Go

- `m` — 当前 Goroutine 占用的线程，可能为空；
- `atomicstatus` — Goroutine 的状态；
- `sched` — 存储 Goroutine 的调度相关的数据；
- `goid` — Goroutine 的 ID，该字段对开发者不可见，Go 团队认为引入 ID 会让部分 Goroutine 变得更特殊，从而限制语言的并发能力[10](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#fn:10)；

上述四个字段中，我们需要展开介绍 `sched` 字段的 [`runtime.gobuf`](https://draveness.me/golang/tree/runtime.gobuf) 结构体中包含哪些内容：

```go
type gobuf struct {
	sp   uintptr
	pc   uintptr
	g    guintptr
	ret  sys.Uintreg
	...
}
```

Go

- `sp` — 栈指针；
- `pc` — 程序计数器；
- `g` — 持有 [`runtime.gobuf`](https://draveness.me/golang/tree/runtime.gobuf) 的 Goroutine；
- `ret` — 系统调用的返回值；

这些内容会在调度器保存或者恢复上下文的时候用到，其中的栈指针和程序计数器会用来存储或者恢复寄存器中的值，改变程序即将执行的代码。

结构体 [`runtime.g`](https://draveness.me/golang/tree/runtime.g) 的 `atomicstatus` 字段存储了当前 Goroutine 的状态。除了几个已经不被使用的以及与 GC 相关的状态之外，Goroutine 可能处于以下 9 种状态：

| 状态          | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| `_Gidle`      | 刚刚被分配并且还没有被初始化                                 |
| `_Grunnable`  | 没有执行代码，没有栈的所有权，存储在运行队列中               |
| `_Grunning`   | 可以执行代码，拥有栈的所有权，被赋予了内核线程 M 和处理器 P  |
| `_Gsyscall`   | 正在执行系统调用，拥有栈的所有权，没有执行用户代码，被赋予了内核线程 M 但是不在运行队列上 |
| `_Gwaiting`   | 由于运行时而被阻塞，没有执行用户代码并且不在运行队列上，但是可能存在于 Channel 的等待队列上 |
| `_Gdead`      | 没有被使用，没有执行代码，可能有分配的栈                     |
| `_Gcopystack` | 栈正在被拷贝，没有执行代码，不在运行队列上                   |
| `_Gpreempted` | 由于抢占而被阻塞，没有执行用户代码并且不在运行队列上，等待唤醒 |
| `_Gscan`      | GC 正在扫描栈空间，没有执行代码，可以与其他状态同时存在      |

**表 7-3 Goroutine 的状态**

上述状态中比较常见是 `_Grunnable`、`_Grunning`、`_Gsyscall`、`_Gwaiting` 和 `_Gpreempted` 五个状态，这里会重点介绍这几个状态。Goroutine 的状态迁移是个复杂的过程，触发 Goroutine 状态迁移的方法也很多，在这里我们也没有办法介绍全部的迁移路线，只会从中选择一些介绍。

![goroutine-status](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2020-02-05-15808864354603-goroutine-status.png)

**图 6-30 Goroutine 的状态**

虽然 Goroutine 在运行时中定义的状态非常多而且复杂，但是我们可以将这些不同的状态聚合成三种：等待中、可运行、运行中，运行期间会在这三种状态来回切换：

- 等待中：Goroutine 正在等待某些条件满足，例如：系统调用结束等，包括 `_Gwaiting`、`_Gsyscall` 和 `_Gpreempted` 几个状态；
- 可运行：Goroutine 已经准备就绪，可以在线程运行，如果当前程序中有非常多的 Goroutine，每个 Goroutine 就可能会等待更多的时间，即 `_Grunnable`；
- 运行中：Goroutine 正在某个线程上运行，即 `_Grunning`；

![golang-goroutine-state-transition](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2020-02-05-15808864354615-golang-goroutine-state-transition-20211109211124554.png)

**图 6-31 Goroutine 的常见状态迁移**

上图展示了 Goroutine 状态迁移的常见路径，其中包括创建 Goroutine 到 Goroutine 被执行、触发系统调用或者抢占式调度器的状态迁移过程。

### M [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#m)

Go 语言并发模型中的 M 是操作系统线程。调度器最多可以创建 10000 个线程，但是其中大多数的线程都不会执行用户代码（可能陷入系统调用），最多只会有 `GOMAXPROCS` 个活跃线程能够正常运行。

在默认情况下，运行时会将 `GOMAXPROCS` 设置成当前机器的核数，我们也可以在程序中使用 [`runtime.GOMAXPROCS`](https://draveness.me/golang/tree/runtime.GOMAXPROCS) 来改变最大的活跃线程数。

![scheduler-m-and-thread](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2020-02-05-15808864354634-scheduler-m-and-thread.png)

**图 6-32 CPU 和活跃线程**

在默认情况下，一个四核机器会创建四个活跃的操作系统线程，每一个线程都对应一个运行时中的 [`runtime.m`](https://draveness.me/golang/tree/runtime.m) 结构体。

在大多数情况下，我们都会使用 Go 的默认设置，也就是线程数等于 CPU 数，默认的设置不会频繁触发操作系统的线程调度和上下文切换，所有的调度都会发生在用户态，由 Go 语言调度器触发，能够减少很多额外开销。

Go 语言会使用私有结构体 [`runtime.m`](https://draveness.me/golang/tree/runtime.m) 表示操作系统线程，这个结构体也包含了几十个字段，这里先来了解几个与 Goroutine 相关的字段：

```go
type m struct {
	g0   *g
	curg *g
	...
}
```

Go

其中 g0 是持有调度栈的 Goroutine，`curg` 是在当前线程上运行的用户 Goroutine，这也是操作系统线程唯一关心的两个 Goroutine。

![g0-and-g](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2020-02-05-15808864354644-g0-and-g.png)

**图 6-33 调度 Goroutine 和运行 Goroutine**

g0 是一个运行时中比较特殊的 Goroutine，它会深度参与运行时的调度过程，包括 Goroutine 的创建、大内存分配和 CGO 函数的执行。在后面的小节中，我们会经常看到 g0 的身影。

[`runtime.m`](https://draveness.me/golang/tree/runtime.m) 结构体中还存在三个与处理器相关的字段，它们分别表示正在运行代码的处理器 `p`、暂存的处理器 `nextp` 和执行系统调用之前使用线程的处理器 `oldp`：

```go
type m struct {
	p             puintptr
	nextp         puintptr
	oldp          puintptr
}
```

Go

除了在上面介绍的字段之外，[`runtime.m`](https://draveness.me/golang/tree/runtime.m) 还包含大量与线程状态、锁、调度、系统调用有关的字段，我们会在分析调度过程时详细介绍它们。

### P [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#p)

调度器中的处理器 P 是线程和 Goroutine 的中间层，它能提供线程需要的上下文环境，也会负责调度线程上的等待队列，通过处理器 P 的调度，每一个内核线程都能够执行多个 Goroutine，它能在 Goroutine 进行一些 I/O 操作时及时让出计算资源，提高线程的利用率。

因为调度器在启动时就会创建 `GOMAXPROCS` 个处理器，所以 Go 语言程序的处理器数量一定会等于 `GOMAXPROCS`，这些处理器会绑定到不同的内核线程上。

[`runtime.p`](https://draveness.me/golang/tree/runtime.p) 是处理器的运行时表示，作为调度器的内部实现，它包含的字段也非常多，其中包括与性能追踪、垃圾回收和计时器相关的字段，这些字段也非常重要，但是在这里就不展示了，我们主要关注处理器中的线程和运行队列：

```go
type p struct {
	m           muintptr

	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	runnext guintptr
	...
}
```

Go

反向存储的线程维护着线程与处理器之间的关系，而 `runqhead`、`runqtail` 和 `runq` 三个字段表示处理器持有的运行队列，其中存储着待执行的 Goroutine 列表，`runnext` 中是线程下一个需要执行的 Goroutine。

[`runtime.p`](https://draveness.me/golang/tree/runtime.p) 结构体中的状态 `status` 字段会是以下五种中的一种：

| 状态        | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| `_Pidle`    | 处理器没有运行用户代码或者调度器，被空闲队列或者改变其状态的结构持有，运行队列为空 |
| `_Prunning` | 被线程 M 持有，并且正在执行用户代码或者调度器                |
| `_Psyscall` | 没有执行用户代码，当前线程陷入系统调用                       |
| `_Pgcstop`  | 被线程 M 持有，当前处理器由于垃圾回收被停止                  |
| `_Pdead`    | 当前处理器已经不被使用                                       |

**表 7-4 处理器的状态**

通过分析处理器 P 的状态，我们能够对处理器的工作过程有一些简单理解，例如处理器在执行用户代码时会处于 `_Prunning` 状态，在当前线程执行 I/O 操作时会陷入 `_Psyscall` 状态。

### 小结 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#小结-1)

我们在这一小节简单介绍了 Go 语言调度器中常见的数据结构，包括线程 M、处理器 P 和 Goroutine G，它们在 Go 语言运行时中分别使用不同的私有结构体表示，我们在下面会深入分析 Go 语言调度器的实现原理。

## 6.5.3 调度器启动 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#653-调度器启动)

调度器的启动过程是我们平时比较难以接触的过程，不过作为程序启动前的准备工作，理解调度器的启动过程对我们理解调度器的实现原理很有帮助，运行时通过 [`runtime.schedinit`](https://draveness.me/golang/tree/runtime.schedinit) 初始化器：

```go
func schedinit() {
	_g_ := getg()
	...

	sched.maxmcount = 10000

	...
	sched.lastpoll = uint64(nanotime())
	procs := ncpu
	if n, ok := atoi32(gogetenv("GOMAXPROCS")); ok && n > 0 {
		procs = n
	}
	if procresize(procs) != nil {
		throw("unknown runnable goroutine during bootstrap")
	}
}
```



在调度器初始函数执行的过程中会将 `maxmcount` 设置成 10000，这也就是一个 Go 语言程序能够创建的最大线程数，虽然最多可以创建 10000 个线程，但是可以同时运行的线程还是由 `GOMAXPROCS` 变量控制。

我们从环境变量 `GOMAXPROCS` 获取了程序能够同时运行的最大处理器数之后就会调用 [`runtime.procresize`](https://draveness.me/golang/tree/runtime.procresize) 更新程序中处理器的数量，在这时整个程序不会执行任何用户 Goroutine，调度器也会进入锁定状态，[`runtime.procresize`](https://draveness.me/golang/tree/runtime.procresize) 的执行过程如下：

1. 如果全局变量 `allp` 切片中的处理器数量少于期望数量，会对切片进行扩容；
2. 使用 `new` 创建新的处理器结构体并调用 [`runtime.p.init`](https://draveness.me/golang/tree/runtime.p.init) 初始化刚刚扩容的处理器；
3. 通过指针将线程 m0 和处理器 `allp[0]` 绑定到一起；
4. 调用 [`runtime.p.destroy`](https://draveness.me/golang/tree/runtime.p.destroy) 释放不再使用的处理器结构；
5. 通过截断改变全局变量 `allp` 的长度保证与期望处理器数量相等；
6. 将除 `allp[0]` 之外的处理器 P 全部设置成 `_Pidle` 并加入到全局的空闲队列中；

调用 [`runtime.procresize`](https://draveness.me/golang/tree/runtime.procresize) 是调度器启动的最后一步，在这一步过后调度器会完成相应数量处理器的启动，等待用户创建运行新的 Goroutine 并为 Goroutine 调度处理器资源。

## 6.5.4 创建 Goroutine [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#654-创建-goroutine)

想要启动一个新的 Goroutine 来执行任务时，我们需要使用 Go 语言的 `go` 关键字，编译器会通过 [`cmd/compile/internal/gc.state.stmt`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.stmt) 和 [`cmd/compile/internal/gc.state.call`](https://draveness.me/golang/tree/cmd/compile/internal/gc.state.call) 两个方法将该关键字转换成 [`runtime.newproc`](https://draveness.me/golang/tree/runtime.newproc) 函数调用：

```go
func (s *state) call(n *Node, k callKind) *ssa.Value {
	if k == callDeferStack {
		...
	} else {
		switch {
		case k == callGo:
			call = s.newValue1A(ssa.OpStaticCall, types.TypeMem, newproc, s.mem())
		default:
		}
	}
	...
}
```

Go

[`runtime.newproc`](https://draveness.me/golang/tree/runtime.newproc) 的入参是参数大小和表示函数的指针 `funcval`，它会获取 Goroutine 以及调用方的程序计数器，然后调用 [`runtime.newproc1`](https://draveness.me/golang/tree/runtime.newproc1) 函数获取新的 Goroutine 结构体、将其加入处理器的运行队列并在满足条件时调用 [`runtime.wakep`](https://draveness.me/golang/tree/runtime.wakep) 唤醒新的处理执行 Goroutine：

```go
func newproc(siz int32, fn *funcval) {
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	gp := getg()
	pc := getcallerpc()
	systemstack(func() {
		newg := newproc1(fn, argp, siz, gp, pc)

		_p_ := getg().m.p.ptr()
		runqput(_p_, newg, true)

		if mainStarted {
			wakep()
		}
	})
}
```



[`runtime.newproc1`](https://draveness.me/golang/tree/runtime.newproc1) 会根据传入参数初始化一个 `g` 结构体，我们可以将该函数分成以下几个部分介绍它的实现：

1. 获取或者创建新的 Goroutine 结构体；
2. 将传入的参数移到 Goroutine 的栈上；
3. 更新 Goroutine 调度相关的属性；

首先是 Goroutine 结构体的创建过程：

```go
func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) *g {
	_g_ := getg()
	siz := narg
	siz = (siz + 7) &^ 7

	_p_ := _g_.m.p.ptr()
	newg := gfget(_p_) //获取空闲的g
	if newg == nil {
		newg = malg(_StackMin)
		casgstatus(newg, _Gidle, _Gdead)
		allgadd(newg)
	}
	...
```



上述代码会先从处理器的 `gFree` 列表中查找空闲的 Goroutine，如果不存在空闲的 Goroutine，会通过 [`runtime.malg`](https://draveness.me/golang/tree/runtime.malg) 创建一个栈大小足够的新结构体。

接下来，我们会调用 [`runtime.memmove`](https://draveness.me/golang/tree/runtime.memmove) 将 `fn` 函数的所有参数拷贝到栈上，`argp` 和 `narg` 分别是参数的内存空间和大小，我们在该方法中会将参数对应的内存空间整块拷贝到栈上：

```go
	...
	totalSize := 4*sys.RegSize + uintptr(siz) + sys.MinFrameSize
	totalSize += -totalSize & (sys.SpAlign - 1)
	sp := newg.stack.hi - totalSize
	spArg := sp
	if narg > 0 {
		memmove(unsafe.Pointer(spArg), argp, uintptr(narg))
	}
	...
```



拷贝了栈上的参数之后，[`runtime.newproc1`](https://draveness.me/golang/tree/runtime.newproc1) 会设置新的 Goroutine 结构体的参数，包括栈指针、程序计数器并更新其状态到 `_Grunnable` 并返回：

```go
	...
	memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
	newg.sched.sp = sp
	newg.stktopsp = sp
	newg.sched.pc = funcPC(goexit) + sys.PCQuantum
	newg.sched.g = guintptr(unsafe.Pointer(newg))
	gostartcallfn(&newg.sched, fn)
	newg.gopc = callerpc
	newg.startpc = fn.fn
	casgstatus(newg, _Gdead, _Grunnable)
	newg.goid = int64(_p_.goidcache)
	_p_.goidcache++
	return newg
}
```

我们在分析 [`runtime.newproc`](https://draveness.me/golang/tree/runtime.newproc) 的过程中，保留了主干省略了用于获取结构体的 [`runtime.gfget`](https://draveness.me/golang/tree/runtime.gfget)、[`runtime.malg`](https://draveness.me/golang/tree/runtime.malg)、将 Goroutine 加入运行队列的 [`runtime.runqput`](https://draveness.me/golang/tree/runtime.runqput) 以及设置调度信息的过程，下面会依次分析这些函数。

### 初始化结构体 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#初始化结构体)

[`runtime.gfget`](https://draveness.me/golang/tree/runtime.gfget) 通过两种不同的方式获取新的 [`runtime.g`](https://draveness.me/golang/tree/runtime.g)：

1. 从 Goroutine 所在处理器的 `gFree` 列表或者调度器的 `sched.gFree` 列表中获取 [`runtime.g`](https://draveness.me/golang/tree/runtime.g)；
2. 调用 [`runtime.malg`](https://draveness.me/golang/tree/runtime.malg) 生成一个新的 [`runtime.g`](https://draveness.me/golang/tree/runtime.g) 并将结构体追加到全局的 Goroutine 列表 `allgs` 中。

![golang-newproc-get-goroutine](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/golang-newproc-get-goroutine-20211109211412866.png)

**图 6-34 获取 Goroutine 结构体的三种方法**

[`runtime.gfget`](https://draveness.me/golang/tree/runtime.gfget) 中包含两部分逻辑，它会根据处理器中 `gFree` 列表中 Goroutine 的数量做出不同的决策：

1. 当处理器的 Goroutine 列表为空时，会将调度器持有的空闲 Goroutine 转移到当前处理器上，直到 `gFree` 列表中的 Goroutine 数量达到 32；
2. 当处理器的 Goroutine 数量充足时，会从列表头部返回一个新的 Goroutine；

```go
func gfget(_p_ *p) *g {
retry:
	if _p_.gFree.empty() && (!sched.gFree.stack.empty() || !sched.gFree.noStack.empty()) {
		for _p_.gFree.n < 32 {
			gp := sched.gFree.stack.pop()
			if gp == nil {
				gp = sched.gFree.noStack.pop()
				if gp == nil {
					break
				}
			}
			_p_.gFree.push(gp)
		}
		goto retry
	}
	gp := _p_.gFree.pop()
	if gp == nil {
		return nil
	}
	return gp
}
```

Go

当调度器的 `gFree` 和处理器的 `gFree` 列表都不存在结构体时，运行时会调用 [`runtime.malg`](https://draveness.me/golang/tree/runtime.malg) 初始化新的 [`runtime.g`](https://draveness.me/golang/tree/runtime.g) 结构，如果申请的堆栈大小大于 0，这里会通过 [`runtime.stackalloc`](https://draveness.me/golang/tree/runtime.stackalloc) 分配 2KB 的栈空间：

```go
func malg(stacksize int32) *g {
	newg := new(g)
	if stacksize >= 0 {
		stacksize = round2(_StackSystem + stacksize)
		newg.stack = stackalloc(uint32(stacksize))
		newg.stackguard0 = newg.stack.lo + _StackGuard
		newg.stackguard1 = ^uintptr(0)
	}
	return newg
}
```

Go

[`runtime.malg`](https://draveness.me/golang/tree/runtime.malg) 返回的 Goroutine 会存储到全局变量 `allgs` 中。

简单总结一下，[`runtime.newproc1`](https://draveness.me/golang/tree/runtime.newproc1) 会从处理器或者调度器的缓存中获取新的结构体，也可以调用 [`runtime.malg`](https://draveness.me/golang/tree/runtime.malg) 函数创建。

### 运行队列 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#运行队列)

[`runtime.runqput`](https://draveness.me/golang/tree/runtime.runqput) 会将 Goroutine 放到运行队列上，这既可能是全局的运行队列，也可能是处理器本地的运行队列：

```go
func runqput(_p_ *p, gp *g, next bool) {
	if next {
	retryNext:
		oldnext := _p_.runnext
		if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
		gp = oldnext.ptr()
	}
retry:
	h := atomic.LoadAcq(&_p_.runqhead)
	t := _p_.runqtail
	if t-h < uint32(len(_p_.runq)) {
		_p_.runq[t%uint32(len(_p_.runq))].set(gp)
		atomic.StoreRel(&_p_.runqtail, t+1)
		return
	}
	if runqputslow(_p_, gp, h, t) {
		return
	}
	goto retry
}
```

Go

1. 当 `next` 为 `true` 时，将 Goroutine 设置到处理器的 `runnext` 作为下一个处理器执行的任务；
2. 当 `next` 为 `false` 并且本地运行队列还有剩余空间时，将 Goroutine 加入处理器持有的本地运行队列；
3. 当处理器的本地运行队列已经没有剩余空间时就会把本地队列中的一部分 Goroutine 和待加入的 Goroutine 通过 [`runtime.runqputslow`](https://draveness.me/golang/tree/runtime.runqputslow) 添加到调度器持有的全局运行队列上；

处理器本地的运行队列是一个使用数组构成的环形链表，它最多可以存储 256 个待执行任务。

![golang-runnable-queue](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2020-02-05-15808864354654-golang-runnable-queue-20211109211423829.png)

**图 6-35 全局和本地运行队列**

简单总结一下，Go 语言有两个运行队列，其中一个是处理器本地的运行队列，另一个是调度器持有的全局运行队列，只有在本地运行队列没有剩余空间时才会使用全局队列。

### 调度信息 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#调度信息)

运行时创建 Goroutine 时会通过下面的代码设置调度相关的信息，前两行代码会分别将程序计数器和 Goroutine 设置成 [`runtime.goexit`](https://draveness.me/golang/tree/runtime.goexit) 和新创建 Goroutine 运行的函数：

```go
	...
	newg.sched.pc = funcPC(goexit) + sys.PCQuantum
	newg.sched.g = guintptr(unsafe.Pointer(newg))
	gostartcallfn(&newg.sched, fn)
	...
```

Go

上述调度信息 `sched` 不是初始化后的 Goroutine 的最终结果，它还需要经过 [`runtime.gostartcallfn`](https://draveness.me/golang/tree/runtime.gostartcallfn) 和 [`runtime.gostartcall`](https://draveness.me/golang/tree/runtime.gostartcall) 的处理：

```go
func gostartcallfn(gobuf *gobuf, fv *funcval) {
	gostartcall(gobuf, unsafe.Pointer(fv.fn), unsafe.Pointer(fv))
}

func gostartcall(buf *gobuf, fn, ctxt unsafe.Pointer) {
	sp := buf.sp
	if sys.RegSize > sys.PtrSize {
		sp -= sys.PtrSize
		*(*uintptr)(unsafe.Pointer(sp)) = 0
	}
	sp -= sys.PtrSize
	*(*uintptr)(unsafe.Pointer(sp)) = buf.pc
	buf.sp = sp
	buf.pc = uintptr(fn)
	buf.ctxt = ctxt
}
```

Go

调度信息的 `sp` 中存储了 [`runtime.goexit`](https://draveness.me/golang/tree/runtime.goexit) 函数的程序计数器，而 `pc` 中存储了传入函数的程序计数器。因为 `pc` 寄存器的作用就是存储程序接下来运行的位置，所以 `pc` 的使用比较好理解，但是 `sp` 中存储的 [`runtime.goexit`](https://draveness.me/golang/tree/runtime.goexit) 会让人感到困惑，我们需要配合下面的调度循环来理解它的作用。

## 6.5.5 调度循环 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#655-调度循环)

调度器启动之后，Go 语言运行时会调用 [`runtime.mstart`](https://draveness.me/golang/tree/runtime.mstart) 以及 [`runtime.mstart1`](https://draveness.me/golang/tree/runtime.mstart1)，前者会初始化 g0 的 `stackguard0` 和 `stackguard1` 字段，后者会初始化线程并调用 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 进入调度循环：

```go
func schedule() {
	_g_ := getg()

top:
	var gp *g
	var inheritTime bool

	if gp == nil {
		if _g_.m.p.ptr().schedtick%61 == 0 && sched.runqsize > 0 {
			lock(&sched.lock)
			gp = globrunqget(_g_.m.p.ptr(), 1)
			unlock(&sched.lock)
		}
	}
	if gp == nil {
		gp, inheritTime = runqget(_g_.m.p.ptr())
	}
	if gp == nil {
		gp, inheritTime = findrunnable()
	}

	execute(gp, inheritTime)
}
```

Go

[`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 函数会从下面几个地方查找待执行的 Goroutine：

1. 为了保证公平，当全局运行队列中有待执行的 Goroutine 时，通过 `schedtick` 保证有一定几率会从全局的运行队列中查找对应的 Goroutine；
2. 从处理器本地的运行队列中查找待执行的 Goroutine；
3. 如果前两种方法都没有找到 Goroutine，会通过 [`runtime.findrunnable`](https://draveness.me/golang/tree/runtime.findrunnable) 进行阻塞地查找 Goroutine；

[`runtime.findrunnable`](https://draveness.me/golang/tree/runtime.findrunnable) 的实现非常复杂，这个 300 多行的函数通过以下的过程获取可运行的 Goroutine：

1. 从本地运行队列、全局运行队列中查找；
2. 从网络轮询器中查找是否有 Goroutine 等待运行；
3. 通过 [`runtime.runqsteal`](https://draveness.me/golang/tree/runtime.runqsteal) 尝试从其他随机的处理器中窃取待运行的 Goroutine，该函数还可能窃取处理器的计时器；

因为函数的实现过于复杂，上述的执行过程是经过简化的，总而言之，当前函数一定会返回一个可执行的 Goroutine，如果当前不存在就会阻塞等待。

接下来由 [`runtime.execute`](https://draveness.me/golang/tree/runtime.execute) 执行获取的 Goroutine，做好准备工作后，它会通过 [`runtime.gogo`](https://draveness.me/golang/tree/runtime.gogo) 将 Goroutine 调度到当前线程上。

```go
func execute(gp *g, inheritTime bool) {
	_g_ := getg()

	_g_.m.curg = gp
	gp.m = _g_.m
	casgstatus(gp, _Grunnable, _Grunning)
	gp.waitsince = 0
	gp.preempt = false
	gp.stackguard0 = gp.stack.lo + _StackGuard
	if !inheritTime {
		_g_.m.p.ptr().schedtick++
	}

	gogo(&gp.sched)
}
```

Go

[`runtime.gogo`](https://draveness.me/golang/tree/runtime.gogo) 在不同处理器架构上的实现都不同，但是也都大同小异，下面是该函数在 386 架构上的实现：

```go
TEXT runtime.gogo(SB), NOSPLIT, $8-4
	MOVL buf+0(FP), BX     // 获取调度信息
	MOVL gobuf_g(BX), DX
	MOVL 0(DX), CX         // 保证 Goroutine 不为空
	get_tls(CX)
	MOVL DX, g(CX)
	MOVL gobuf_sp(BX), SP  // 将 runtime.goexit 函数的 PC 恢复到 SP 中
	MOVL gobuf_ret(BX), AX
	MOVL gobuf_ctxt(BX), DX
	MOVL $0, gobuf_sp(BX)
	MOVL $0, gobuf_ret(BX)
	MOVL $0, gobuf_ctxt(BX)
	MOVL gobuf_pc(BX), BX  // 获取待执行函数的程序计数器
	JMP  BX                // 开始执行
```

Go

它从 [`runtime.gobuf`](https://draveness.me/golang/tree/runtime.gobuf) 中取出了 [`runtime.goexit`](https://draveness.me/golang/tree/runtime.goexit) 的程序计数器和待执行函数的程序计数器，其中：

- [`runtime.goexit`](https://draveness.me/golang/tree/runtime.goexit) 的程序计数器被放到了栈 SP 上；
- 待执行函数的程序计数器被放到了寄存器 BX 上；

在函数调用一节中，我们曾经介绍过 Go 语言的调用惯例，正常的函数调用都会使用 `CALL` 指令，该指令会将调用方的返回地址加入栈寄存器 SP 中，然后跳转到目标函数；当目标函数返回后，会从栈中查找调用的地址并跳转回调用方继续执行剩下的代码。

[`runtime.gogo`](https://draveness.me/golang/tree/runtime.gogo) 就利用了 Go 语言的调用惯例成功模拟这一调用过程，通过以下几个关键指令模拟 `CALL` 的过程：

```go
	MOVL gobuf_sp(BX), SP  // 将 runtime.goexit 函数的 PC 恢复到 SP 中
	MOVL gobuf_pc(BX), BX  // 获取待执行函数的程序计数器
	JMP  BX                // 开始执行
```

Go

![golang-gogo-stack](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2020-02-05-15808864354661-golang-gogo-stack.png)

**图 6-36 runtime.gogo 栈内存**

上图展示了调用 `JMP` 指令后的栈中数据，当 Goroutine 中运行的函数返回时，程序会跳转到 [`runtime.goexit`](https://draveness.me/golang/tree/runtime.goexit) 所在位置执行该函数：

```go
TEXT runtime.goexit(SB),NOSPLIT,$0-0
	CALL	runtime.goexit1(SB)

func goexit1() {
	mcall(goexit0)
}
```

Go

经过一系列复杂的函数调用，我们最终在当前线程的 g0 的栈上调用 [`runtime.goexit0`](https://draveness.me/golang/tree/runtime.goexit0) 函数，该函数会将 Goroutine 转换会 `_Gdead` 状态、清理其中的字段、移除 Goroutine 和线程的关联并调用 [`runtime.gfput`](https://draveness.me/golang/tree/runtime.gfput) 重新加入处理器的 Goroutine 空闲列表 `gFree`：

```go
func goexit0(gp *g) {
	_g_ := getg()

	casgstatus(gp, _Grunning, _Gdead)
	gp.m = nil
	...
	gp.param = nil
	gp.labels = nil
	gp.timer = nil

	dropg()
	gfput(_g_.m.p.ptr(), gp)
	schedule()
}
```

Go

在最后 [`runtime.goexit0`](https://draveness.me/golang/tree/runtime.goexit0) 会重新调用 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 触发新一轮的 Goroutine 调度，Go 语言中的运行时调度循环会从 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 开始，最终又回到 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule)，我们可以认为调度循环永远都不会返回。

![golang-scheduler-loop](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2020-02-05-15808864354669-golang-scheduler-loop.png)

**图 6-36 调度循环**

这里介绍的是 Goroutine 正常执行并退出的逻辑，实际情况会复杂得多，多数情况下 Goroutine 在执行的过程中都会经历协作式或者抢占式调度，它会让出线程的使用权等待调度器的唤醒。

## 6.5.6 触发调度 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#656-触发调度)

这里简单介绍下所有触发调度的时间点，因为调度器的 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 会重新选择 Goroutine 在线程上执行，所以我们只要找到该函数的调用方就能找到所有触发调度的时间点，经过分析和整理，我们能得到如下的树形结构：

![schedule-points](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2020-02-05-15808864354679-schedule-points-20211109211339021.png)

**图 6-37 调度时间点**

除了上图中可能触发调度的时间点，运行时还会在线程启动 [`runtime.mstart`](https://draveness.me/golang/tree/runtime.mstart) 和 Goroutine 执行结束 [`runtime.goexit0`](https://draveness.me/golang/tree/runtime.goexit0) 触发调度。我们在这里会重点介绍运行时触发调度的几个路径：

- 主动挂起 — [`runtime.gopark`](https://draveness.me/golang/tree/runtime.gopark) -> [`runtime.park_m`](https://draveness.me/golang/tree/runtime.park_m)
- 系统调用 — [`runtime.exitsyscall`](https://draveness.me/golang/tree/runtime.exitsyscall) -> [`runtime.exitsyscall0`](https://draveness.me/golang/tree/runtime.exitsyscall0)
- 协作式调度 — [`runtime.Gosched`](https://draveness.me/golang/tree/runtime.Gosched) -> [`runtime.gosched_m`](https://draveness.me/golang/tree/runtime.gosched_m) -> [`runtime.goschedImpl`](https://draveness.me/golang/tree/runtime.goschedImpl)
- 系统监控 — [`runtime.sysmon`](https://draveness.me/golang/tree/runtime.sysmon) -> [`runtime.retake`](https://draveness.me/golang/tree/runtime.retake) -> [`runtime.preemptone`](https://draveness.me/golang/tree/runtime.preemptone)

我们在这里介绍的调度时间点不是将线程的运行权直接交给其他任务，而是通过调度器的 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 重新调度。

### 主动挂起 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#主动挂起)

[`runtime.gopark`](https://draveness.me/golang/tree/runtime.gopark) 是触发调度最常见的方法，该函数会将当前 Goroutine 暂停，被暂停的任务不会放回运行队列，我们来分析该函数的实现原理：

```go
func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
	mp := acquirem()
	gp := mp.curg
	mp.waitlock = lock
	mp.waitunlockf = unlockf
	gp.waitreason = reason
	mp.waittraceev = traceEv
	mp.waittraceskip = traceskip
	releasem(mp)
	mcall(park_m)
}
```

Go

上述会通过 [`runtime.mcall`](https://draveness.me/golang/tree/runtime.mcall) 切换到 g0 的栈上调用 [`runtime.park_m`](https://draveness.me/golang/tree/runtime.park_m)：

```go
func park_m(gp *g) {
	_g_ := getg()

	casgstatus(gp, _Grunning, _Gwaiting)
	dropg()

	schedule()
}
```

Go

[`runtime.park_m`](https://draveness.me/golang/tree/runtime.park_m) 会将当前 Goroutine 的状态从 `_Grunning` 切换至 `_Gwaiting`，调用 [`runtime.dropg`](https://draveness.me/golang/tree/runtime.dropg) 移除线程和 Goroutine 之间的关联，在这之后就可以调用 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 触发新一轮的调度了。

当 Goroutine 等待的特定条件满足后，运行时会调用 [`runtime.goready`](https://draveness.me/golang/tree/runtime.goready) 将因为调用 [`runtime.gopark`](https://draveness.me/golang/tree/runtime.gopark) 而陷入休眠的 Goroutine 唤醒。

```go
func goready(gp *g, traceskip int) {
	systemstack(func() {
		ready(gp, traceskip, true)
	})
}

func ready(gp *g, traceskip int, next bool) {
	_g_ := getg()

	casgstatus(gp, _Gwaiting, _Grunnable)
	runqput(_g_.m.p.ptr(), gp, next)
	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 {
		wakep()
	}
}
```

Go

[`runtime.ready`](https://draveness.me/golang/tree/runtime.ready) 会将准备就绪的 Goroutine 的状态切换至 `_Grunnable` 并将其加入处理器的运行队列中，等待调度器的调度。

### 系统调用 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#系统调用)

系统调用也会触发运行时调度器的调度，为了处理特殊的系统调用，我们甚至在 Goroutine 中加入了 `_Gsyscall` 状态，Go 语言通过 [`syscall.Syscall`](https://draveness.me/golang/tree/syscall.Syscall) 和 [`syscall.RawSyscall`](https://draveness.me/golang/tree/syscall.RawSyscall) 等使用汇编语言编写的方法封装操作系统提供的所有系统调用，其中 [`syscall.Syscall`](https://draveness.me/golang/tree/syscall.Syscall) 的实现如下：

```go
#define INVOKE_SYSCALL	INT	$0x80

TEXT .Syscall(SB),NOSPLIT,$0-28
	CALL	runtime.entersyscall(SB)
	...
	INVOKE_SYSCALL
	...
	CALL	runtime.exitsyscall(SB)
	RET
ok:
	...
	CALL	runtime.exitsyscall(SB)
	RET
```

Go

在通过汇编指令 `INVOKE_SYSCALL` 执行系统调用前后，上述函数会调用运行时的 [`runtime.entersyscall`](https://draveness.me/golang/tree/runtime.entersyscall) 和 [`runtime.exitsyscall`](https://draveness.me/golang/tree/runtime.exitsyscall)，正是这一层包装能够让我们在陷入系统调用前触发运行时的准备和清理工作。

![golang-syscall-and-rawsyscal](https://cdn.jsdelivr.net/gh/nber1994/fu0k@master/uPic/2020-02-05-15808864354688-golang-syscall-and-rawsyscall.png)

**图 6-38 Go 语言系统调用**

不过出于性能的考虑，如果这次系统调用不需要运行时参与，就会使用 [`syscall.RawSyscall`](https://draveness.me/golang/tree/syscall.RawSyscall) 简化这一过程，不再调用运行时函数。[这里](https://gist.github.com/draveness/50c88883f30fa99d548cf1163c98aeb1)包含 Go 语言对 Linux 386 架构上不同系统调用的分类，我们会按需决定是否需要运行时的参与。

|     系统调用     |    类型    |
| :--------------: | :--------: |
|     SYS_TIME     | RawSyscall |
| SYS_GETTIMEOFDAY | RawSyscall |
|  SYS_SETRLIMIT   | RawSyscall |
|  SYS_GETRLIMIT   | RawSyscall |
|  SYS_EPOLL_WAIT  |  Syscall   |
|        …         |     …      |

**表 7-5 系统调用的类型**

由于直接进行系统调用会阻塞当前的线程，所以只有可以立刻返回的系统调用才可能会被设置成 `RawSyscall` 类型，例如：`SYS_EPOLL_CREATE`、`SYS_EPOLL_WAIT`（超时时间为 0）、`SYS_TIME` 等。

正常的系统调用过程相对比较复杂，下面将分别介绍进入系统调用前的准备工作和系统调用结束后的收尾工作。

#### 准备工作 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#准备工作)

[`runtime.entersyscall`](https://draveness.me/golang/tree/runtime.entersyscall) 会在获取当前程序计数器和栈位置之后调用 [`runtime.reentersyscall`](https://draveness.me/golang/tree/runtime.reentersyscall)，它会完成 Goroutine 进入系统调用前的准备工作：

```go
func reentersyscall(pc, sp uintptr) {
	_g_ := getg()
	_g_.m.locks++
	_g_.stackguard0 = stackPreempt
	_g_.throwsplit = true

	save(pc, sp)
	_g_.syscallsp = sp
	_g_.syscallpc = pc
	casgstatus(_g_, _Grunning, _Gsyscall)

	_g_.m.syscalltick = _g_.m.p.ptr().syscalltick
	_g_.m.mcache = nil
	pp := _g_.m.p.ptr()
	pp.m = 0
	_g_.m.oldp.set(pp)
	_g_.m.p = 0
	atomic.Store(&pp.status, _Psyscall)
	if sched.gcwaiting != 0 {
		systemstack(entersyscall_gcwait)
		save(pc, sp)
	}
	_g_.m.locks--
}
```

Go

1. 禁止线程上发生的抢占，防止出现内存不一致的问题；
2. 保证当前函数不会触发栈分裂或者增长；
3. 保存当前的程序计数器 PC 和栈指针 SP 中的内容；
4. 将 Goroutine 的状态更新至 `_Gsyscall`；
5. 将 Goroutine 的处理器和线程暂时分离并更新处理器的状态到 `_Psyscall`；
6. 释放当前线程上的锁；

需要注意的是 [`runtime.reentersyscall`](https://draveness.me/golang/tree/runtime.reentersyscall) 会使处理器和线程的分离，当前线程会陷入系统调用等待返回，在锁被释放后，会有其他 Goroutine 抢占处理器资源。

#### 恢复工作 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#恢复工作)

当系统调用结束后，会调用退出系统调用的函数 [`runtime.exitsyscall`](https://draveness.me/golang/tree/runtime.exitsyscall) 为当前 Goroutine 重新分配资源，该函数有两个不同的执行路径：

1. 调用 [`runtime.exitsyscallfast`](https://draveness.me/golang/tree/runtime.exitsyscallfast)；
2. 切换至调度器的 Goroutine 并调用 [`runtime.exitsyscall0`](https://draveness.me/golang/tree/runtime.exitsyscall0)；

```go
func exitsyscall() {
	_g_ := getg()

	oldp := _g_.m.oldp.ptr()
	_g_.m.oldp = 0
	if exitsyscallfast(oldp) {
		_g_.m.p.ptr().syscalltick++
		casgstatus(_g_, _Gsyscall, _Grunning)
		...

		return
	}

	mcall(exitsyscall0)
	_g_.m.p.ptr().syscalltick++
	_g_.throwsplit = false
}
```

Go

这两种不同的路径会分别通过不同的方法查找一个用于执行当前 Goroutine 处理器 P，快速路径 [`runtime.exitsyscallfast`](https://draveness.me/golang/tree/runtime.exitsyscallfast) 中包含两个不同的分支：

1. 如果 Goroutine 的原处理器处于 `_Psyscall` 状态，会直接调用 `wirep` 将 Goroutine 与处理器进行关联；
2. 如果调度器中存在闲置的处理器，会调用 [`runtime.acquirep`](https://draveness.me/golang/tree/runtime.acquirep) 使用闲置的处理器处理当前 Goroutine；

另一个相对较慢的路径 [`runtime.exitsyscall0`](https://draveness.me/golang/tree/runtime.exitsyscall0) 会将当前 Goroutine 切换至 `_Grunnable` 状态，并移除线程 M 和当前 Goroutine 的关联：

1. 当我们通过 [`runtime.pidleget`](https://draveness.me/golang/tree/runtime.pidleget) 获取到闲置的处理器时就会在该处理器上执行 Goroutine；
2. 在其它情况下，我们会将当前 Goroutine 放到全局的运行队列中，等待调度器的调度；

无论哪种情况，我们在这个函数中都会调用 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 触发调度器的调度，因为上一节已经介绍过调度器的调度过程，所以在这里就不展开了。

### 协作式调度 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#协作式调度)

我们在设计原理中介绍过了 Go 语言基于协作式和信号的两种抢占式调度，这里主要介绍其中的协作式调度。[`runtime.Gosched`](https://draveness.me/golang/tree/runtime.Gosched) 函数会主动让出处理器，允许其他 Goroutine 运行。该函数无法挂起 Goroutine，调度器可能会将当前 Goroutine 调度到其他线程上：

```go
func Gosched() {
	checkTimeouts()
	mcall(gosched_m)
}

func gosched_m(gp *g) {
	goschedImpl(gp)
}

func goschedImpl(gp *g) {
	casgstatus(gp, _Grunning, _Grunnable)
	dropg()
	lock(&sched.lock)
	globrunqput(gp)
	unlock(&sched.lock)

	schedule()
}
```

Go

经过连续几次跳转，我们最终在 g0 的栈上调用 [`runtime.goschedImpl`](https://draveness.me/golang/tree/runtime.goschedImpl)，运行时会更新 Goroutine 的状态到 `_Grunnable`，让出当前的处理器并将 Goroutine 重新放回全局队列，在最后，该函数会调用 [`runtime.schedule`](https://draveness.me/golang/tree/runtime.schedule) 触发调度。

## 6.5.7 线程管理 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#657-线程管理)

Go 语言的运行时会通过调度器改变线程的所有权，它也提供了 [`runtime.LockOSThread`](https://draveness.me/golang/tree/runtime.LockOSThread) 和 [`runtime.UnlockOSThread`](https://draveness.me/golang/tree/runtime.UnlockOSThread) 让我们有能力绑定 Goroutine 和线程完成一些比较特殊的操作。Goroutine 应该在调用操作系统服务或者依赖线程状态的非 Go 语言库时调用 [`runtime.LockOSThread`](https://draveness.me/golang/tree/runtime.LockOSThread) 函数[11](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#fn:11)，例如：C 语言图形库等。

[`runtime.LockOSThread`](https://draveness.me/golang/tree/runtime.LockOSThread) 会通过如下所示的代码绑定 Goroutine 和当前线程：

```go
func LockOSThread() {
	if atomic.Load(&newmHandoff.haveTemplateThread) == 0 && GOOS != "plan9" {
		startTemplateThread()
	}
	_g_ := getg()
	_g_.m.lockedExt++
	dolockOSThread()
}

func dolockOSThread() {
	_g_ := getg()
	_g_.m.lockedg.set(_g_)
	_g_.lockedm.set(_g_.m)
}
```

Go

[`runtime.dolockOSThread`](https://draveness.me/golang/tree/runtime.dolockOSThread) 会分别设置线程的 `lockedg` 字段和 Goroutine 的 `lockedm` 字段，这两行代码会绑定线程和 Goroutine。

当 Goroutine 完成了特定的操作之后，会调用以下函数 [`runtime.UnlockOSThread`](https://draveness.me/golang/tree/runtime.UnlockOSThread) 分离 Goroutine 和线程：

```go
func UnlockOSThread() {
	_g_ := getg()
	if _g_.m.lockedExt == 0 {
		return
	}
	_g_.m.lockedExt--
	dounlockOSThread()
}

func dounlockOSThread() {
	_g_ := getg()
	if _g_.m.lockedInt != 0 || _g_.m.lockedExt != 0 {
		return
	}
	_g_.m.lockedg = 0
	_g_.lockedm = 0
}
```

Go

函数执行的过程与 [`runtime.LockOSThread`](https://draveness.me/golang/tree/runtime.LockOSThread) 正好相反。在多数的服务中，我们都用不到这一对函数，不过使用 CGO 或者经常与操作系统打交道的读者可能会见到它们的身影。

### 线程生命周期 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#线程生命周期)

Go 语言的运行时会通过 [`runtime.startm`](https://draveness.me/golang/tree/runtime.startm) 启动线程来执行处理器 P，如果我们在该函数中没能从闲置列表中获取到线程 M 就会调用 [`runtime.newm`](https://draveness.me/golang/tree/runtime.newm) 创建新的线程：

```go
func newm(fn func(), _p_ *p, id int64) {
	mp := allocm(_p_, fn, id)
	mp.nextp.set(_p_)
	mp.sigmask = initSigmask
	...
	newm1(mp)
}

func newm1(mp *m) {
	if iscgo {
		...
	}
	newosproc(mp)
}
```

Go

创建新的线程需要使用如下所示的 [`runtime.newosproc`](https://draveness.me/golang/tree/runtime.newosproc)，该函数在 Linux 平台上会通过系统调用 `clone` 创建新的操作系统线程，它也是创建线程链路上距离操作系统最近的 Go 语言函数：

```go
func newosproc(mp *m) {
	stk := unsafe.Pointer(mp.g0.stack.hi)
	...
	ret := clone(cloneFlags, stk, unsafe.Pointer(mp), unsafe.Pointer(mp.g0), unsafe.Pointer(funcPC(mstart)))
	...
}
```

Go

使用系统调用 `clone` 创建的线程会在线程主动调用 `exit`、或者传入的函数 [`runtime.mstart`](https://draveness.me/golang/tree/runtime.mstart) 返回会主动退出，[`runtime.mstart`](https://draveness.me/golang/tree/runtime.mstart) 会执行调用 [`runtime.newm`](https://draveness.me/golang/tree/runtime.newm) 时传入的匿名函数 `fn`，到这里也就完成了从线程创建到销毁的整个闭环。

## 6.5.8 小结 [#](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/#658-小结)

Goroutine 和调度器是 Go 语言能够高效地处理任务并且最大化利用资源的基础，本节介绍了 Go 语言用于处理并发任务的 G - M - P 模型，我们不仅介绍了它们各自的数据结构以及常见状态，还通过特定场景介绍调度器的工作原理以及不同数据结构之间的协作关系，相信能够帮助各位读者理解调度器的实现。







# 参考

[大彬go调度器系列，这个写的还是不错的](https://lessisbetter.site/2019/03/10/golang-scheduler-1-history/)

[Golang并发模型之GMP](https://studygolang.com/articles/30961)

[GMP 模型，为什么要有 P？](https://eddycjy.com/posts/go/go-tips-gmp-p/)

[从源码角度看 Golang 的调度（上）](https://www.infoq.cn/article/r6wzs7bvq2er9kuelbqb)

[从源码角度看 Golang 的调度（下）](https://www.infoq.cn/article/NuvRPz1cPk9Cw0HP3bkY)

[深度解密调度器源码系列](https://qcrao.com/2019/09/06/dive-into-go-scheduler-source-code/)

[面向信仰编程-调度器](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-goroutine/)

[Asciidoc 笔记](https://qcrao.com/ishare/go-scheduler/#true%E6%BA%90%E7%A0%81%E9%98%85%E8%AF%BB)

