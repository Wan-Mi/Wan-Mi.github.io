---
title: '[Go]并发调度模型'
date: 2018-02-12 11:23:26
tags: Go
---

Go因Goroutine而与众不同。  
在Go程序中所有的用户代码都运行在Goroutine中，为了让程序高效运行需要一个合理的调度机制进行Goroutine的并发调度。  Go语言内置运行时，面向并发需求设计了全新的架构模型——GPM调度模型。  

- G：Goroutine的简称。  虽然Go进程内的一切都以G的方式运行，但G并非执行体，它仅保存并发任务状态，为任务执行提供栈内存空间；
- M：系统线程（OS Thread）的代称，M取自其寓意Machine的首字母。  M是代码的实际执行体。其通过修改寄存器，将执行栈指向G自带栈内存，并在此空间内分配栈堆帧，执行任务函数。  M必须绑定一个有效的P才被允许执行任务，否则只能休眠。
- P：Processor的简称，可理解为（调度）处理器，作用类似于CPU核，用来控制可同时并发执行的任务数量。P还为M提供执行资源，如：对象分配内存、本地任务队列。  P是G和M的中间层，其数量通过`GOMAXPROCS`控制。一个G想要运行起来首先需要被分配一个P，然后M再和P绑定以调度循环的方式不断执行G并发任务。

---
调度器初始化函数：`schedinit()`除了内存分配、垃圾回收等操作外还会设置MaxMcount（M的最大值）和 GOMAXPROCS(P最大值)。  由于在初始化后，引导过程才创建并运行main goroutine，因而在运行期间用`runtime.GOMAXPROCS()`修改P数量时需要StopTheWorld，代价很大。

---

下图为一个完整的Goroutine调度模型示意图。进程中的G进入Global队列，由调度器将它们分别分配至P的Local队列。调度器创建M与P进行绑定，并循环消费（运行）P队列中的G。当M因陷入系统调用而长期阻塞时，P会被调度器收回并唤醒（新建）M与其绑定。
![](goroutine-scheduler-model.png)
_注:图片引用自[也谈goroutine调度器](http://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)_  
    

另：Go中的抢占式调度通过为G设立“抢占标记”的形式进行——监控器会对运行时间过长的G进行标记。当其进行函数调用时，被编译器安插的指令就会检查该标记，若有标记则暂停该G将其扔回全局队列。对应M继续运行P队列中其他G。

---
Reference：

- [《Go 1.5源码剖析》](https://github.com/qyuhen/book/blob/master/Go%201.5%20%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90%20%EF%BC%88%E4%B9%A6%E7%AD%BE%E7%89%88%EF%BC%89.pdf) 
- [也谈goroutine调度器](http://tonybai.com/2017/06/23/an-intro-about-goroutine-scheduler/)
