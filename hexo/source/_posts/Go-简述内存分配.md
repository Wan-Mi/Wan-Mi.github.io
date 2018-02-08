---
title: '[Go]简述内存分配'
date: 2018-02-08 17:32:58
tags: Go
---
## 内存分配的核心目标
- 每次申请大内存块，自主管理，减少系统调用
- 基于块的内存复用，加快内存分配和回收

## 内存分配的基本策略

1. 每次从系统申请大块内存以减少系统调用；  
2. 将申请到的大块内存进行切分构成链表；  
3. 为对象分配内存时，只需从大小合适的链表中提取一小块；  
4. 回收对象内存时，会将提取的小块内存归还原链表；
5. 如闲置内存过多则尝试归还部分内存给操作系统；

## 内存块
Go的内存分配器将管理的内存块分为两种：

- span：由多个连续的页（Page）组成的大块内存；

```
//malloc.go
 PageShift = 13,
 PageSize = 1<<PageShift,   //8192 bytes
```

- object：将span按特定大小切成小块，每一小块可存储一个对象。

## 管理组件
Go采用了`tcmalloc`的内存分配架构，分配器由三种组建组成：  

- heap：全局根对象。负责向操作系统申请新内存，并管理闲置span。

```
//mheap.go
type mheap struct {	free      [_MaxMHeapList]mSpanList // 127页以内的闲置span链表
	freelarge mSpanList                // 超过127页（>=1M）的大span链表
	
	//每个central对应一种sizeclass
	central [_NumSizeClasses]struct {
		mcentral mcentral
		pad      [sys.CacheLineSize]byte
	}
} ```

- central：从heap获取空闲span，切分好，供给cache使用：  

```
//mcentral.go
type mcentral struct {
	lock      mutex
	sizeclass int32    // 规格大小
	nonempty  mSpanList // 尚有空闲对象的span链表
	empty     mSpanList // 无空闲对象的span列表
}
```

- cache：每个运行的工作线程都会绑定一个cache，用于无锁对象分配。内部有个以等级为序号的数组，持有多个切分好的span。不够时向对应的central获取新span：

```
//mcache.go 
type mcache struct {
	// 以sizeclass为索引管理多个用于分配的span
	alloc [_NumSizeClasses]*mspan 
}
```

## 分配流程
- 大于32KB的对象使用heap直接分配；
- 小于16B的对象使用cache的小对象分配器进行分配；
- 中间大小的对象会从`mcache.alloc[sizeclass]`中找到对象size等级对应的span进行内存分配；  如span不足则根据`heap.central[sizeclass] `找到对应central获取span；  如central无可用span则向heap申请;  heap也没有空间则向系统申请新的内存。


```
	_TinySize      = 16        // 16B
	_MaxSmallSize  = 32768     // 32KB

```

## 回收流程
 针对heap上分配的对象，直接在heap上回收。其他对象回收方式如下：
 
- 垃圾回收器将可回收对象交还给所属`span.freelist`；
- 将该span放回central以便任意cache重新获取；
- 如一个span收回了所有的对象，则将其交还给heap，以便重新切分复用；
- 垃圾回收器定期扫描获取heap里长期闲置的span，释放其占用内存。

--

Reference:

- [《Go 1.5源码剖析》](https://github.com/qyuhen/book/blob/master/Go%201.5%20%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90%20%EF%BC%88%E4%B9%A6%E7%AD%BE%E7%89%88%EF%BC%89.pdf)
- [《Go 学习笔记 第四版》](https://github.com/qyuhen/book/blob/master/Go%20%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0%20%E7%AC%AC%E5%9B%9B%E7%89%88.pdf)
- [《Implementation of Golang》](https://tracymacding.gitbooks.io/implementation-of-golang/content/memory/memory_alloc_alg.html)

