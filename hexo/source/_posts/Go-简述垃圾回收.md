---
title: '[Go]简述垃圾回收（GC）'
date: 2018-02-09 11:49:31
tags: Go
---

GC问题的核心是：抑制堆增长，充分利用CPU资源。Go所采用的GC算法是经典标记-清除算法的进一步改良。本文在介绍Go中GC逻辑之前会先介绍基本的GC算法。

## 基本GC算法
基本的垃圾回收（GC）算法有四种：引用计数法（Reference Counting）、复制法（Copying）、标记-清除法（Mark-Sweep）和标记-压缩法（Mark-Compact）。  
**注**：在某些语言环境中会同时使用其中的多种算法进行垃圾回收。

### 基本流程
1. 检查分配内存总量是否超过预设阈值；
2. 如超过阈值，暂停一切用户逻辑，进行垃圾回收（不释放物理内存）；
3. 垃圾回收完成后，恢复用户逻辑，将阈值设为存活对象所用内存的2被；
4. 专有线程定期（如2分钟）回收物理内存。闲置超过一定时间（如5分钟）的span物理内存也会被操作系统收回。

### 基本概念
- mutator和collector：collector指的是垃圾回收器；mutator原意是增变基因，java中代指设值方法，此处是指程序本身（非collector的部分），mutator的作用是分配内存和内存读写。
- mutator root：mutator根对象指分配在堆内存以外，可被mutator直接访问的变量。如：静态／全局变量和局部变量。
- 可达对象：从mutator根对象遍历，可直接访问到的变量。

### 基本原理
- 引用计数法（Reference Counting）：初始化对象后令其引用计数为1，增加一个引用计数加1，释放一个引用计数减1，GC时会回收引用计数为0的对象。该方法最大的缺点是无法解决循环引用的问题，python和iOS使用该方法进行垃圾回收。
- 复制法（Copying）：此算法将内存划分为两个相同大小的区域，每次只使用其中一个区域，GC进行时将使用区域中正在使用的对象复制到空的那片区域，再回收原区域。该方法的优点是在GC回收的同时进行了碎片整理，缺点是需要较大的内存空间。
- 标记-清除法（Mark-Sweep）：顾名思义，就是先标记后清除。具体过程是先暂停mutator（StopTheWorld）；然后通过mutator根对象遍历内存空间中的对象，进行标记；collector回收未被标记的内存空间；恢复mutator运行。该方法会产生一定的内存碎片。
- 标记-压缩法（Mark-Compact）：结合复制法和标记清除法的优点，在标记清除法先标记再清除的基础上，再对标记对象进行复制整理存放至堆的某一块区域，以清理碎片。  

在上述算法的基础上还会有一些综合策略：

- 分代回收法（Generational Collecting）：把对象分为“新生代”、“老生代”和“持久代”，对不同生命周期状态的对象采用不同的垃圾回收算法。  对新生对象（新生代）进行高频回收（称为“小回收”），对所有对象进行低频回收（称为“大回收”）。每次小回收后存活的对象归为老生代。  分代回收法可以有效**缩短GC的平均中断时间**。

---
_写屏障（Write Barrier）：指程序对所有涉及修改对象内容的地方进行保护。  在分代回收中，如果某个“新生代对象“被“老生代对象”所引用，它就不应该在“小回收”时被回收。因而需要引入写屏障，用一个记录集来记录老生代对新生代的引用关系，确保有引用关系的新生代不被“小回收”。_

---

- 增量回收法（Incremental Collecting）：对于实时性比较强的应用场景，需要控制**GC的最长中段时间**。于是有了增量回收法，将一个完整的GC中断切分为多个中断，每个中断时间不超过阈值T。 由于不同段的GC过程中，对象引用可能会发生改变，所以同样需要写屏障。  该方式虽然限制了最长中断时间，但由于次数增加，总的中断时间会有所增加。

- 并发回收（Concurrent Collecting）：经典的垃圾回收算法在GC过程中会暂停用户行为（StopTheWorld）。并发回收在多线程垃圾回收的基础上，不暂停用户操作环境，即在程序运行的同时并发进行垃圾回收（此处说的“不暂停”并非完全完全不暂停）。

## Go中的GC算法

Go语言GC相关机制和内容在源码`mgc.go`中有详细介绍：

```
// Garbage collector (GC).
//
// The GC runs concurrently with mutator threads, is type accurate (aka precise), allows multiple
// GC thread to run in parallel. It is a concurrent mark and sweep that uses a write barrier. It is
// non-generational and non-compacting. Allocation is done using size segregated per P allocation
// areas to minimize fragmentation while eliminating locks in the common case.
......
```

直译就是：通过“写屏障”并发的标记和清除（回收内存），不分代，不压缩。  
相信有了前一小节的基础，这句话就很好理解了。下文还会介绍一些Go中GC算法的基本实现过程和组件。

### 三色标记法
Go中的GC使用了三色标记法，它是标记操作和用户代码并发的基本保障。基本步骤：

- 起初标记所有对象为白色；
- 扫描出所有可达对象（mutator root）标记为灰色，放入带处理队列；
- 从队列提取灰色对象，将其引用对象标记为灰色放入队列，自身标记为黑色；
- 重复上一步骤，直至灰色队列中无对象；
- 写屏障监控对象内存修改，重新标色或放回队列，重复上述步骤；
- 回收所有白色对象。

---
三色标记法有个潜在缺陷：由于GC和Mutator同时进行，当垃圾回收速度赶不上产生速度时，垃圾会越来越多。

---


### 控制器
```
// GC Controller
//
// gcController implements the GC pacing controller that determines
// when to trigger concurrent garbage collection and how much marking
// work to do in mutator assists and background marking.
//
// It uses a feedback control algorithm to adjust the memstats.gc_trigger
// trigger based on the heap growth and GC CPU utilization each cycle.
// This algorithm optimizes for heap growth to match GOGC and for CPU
// utilization between assist and background marking to be 25% of
// GOMAXPROCS. The high-level design of this algorithm is documented
// at https://golang.org/s/go15gcpacing.
```
GC控制器会全程参与GC过程，并完成以下工作：

- 控制GC过程中“标记”过程的工作模式和数量；
- 通过反馈算法，监控堆的增长速度和GC CPU的利用率来调整GC触发时机；

### 辅助回收
前文提到，三色标记法中，对象分配速度比后台标记速度慢时会产生问题。因而GO GC中引入了“辅助回收”机制，当遇到上述问题时，会让用户程序线程与后台线程一起参与GC过程，从而平衡内存的分配和回收。

---
Reference：

- [《Go 1.5源码剖析》](https://github.com/qyuhen/book/blob/master/Go%201.5%20%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90%20%EF%BC%88%E4%B9%A6%E7%AD%BE%E7%89%88%EF%BC%89.pdf)  
- [标记-清除算法](https://www.jianshu.com/p/b0f5d21fe031)
- [垃圾回收（GC）的三种基本方式](https://www.kawabangga.com/posts/1336)
- [JVM调优总结（2）：基本垃圾回收算法](http://www.importnew.com/18740.html)



