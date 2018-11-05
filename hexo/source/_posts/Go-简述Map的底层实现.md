---
title: '[Go]简述Map的底层实现'
date: 2018-03-07 16:20:28
tags: Go
---

## 数据结构
和大多数语言一样，Golang中的Map同样是利用哈希表作为基础结构来实现的。

```go
// package runtime  hashmap.go
type hmap struct {
	count     int // map长度
	flags     uint8  //标记
	B         uint8  // log以2为底哈希桶的对数 (最多能存 6.5 * 2^B 个元素)
	noverflow uint16 // 溢出桶的个数（近似）
	hash0     uint32 // 哈希种子

	buckets    unsafe.Pointer // 含 2^B 个桶的数组
	oldbuckets unsafe.Pointer // 一半大小的老的桶数组
	nevacuate  uintptr        // 扩容计数器 

	extra *mapextra // 可选区域
}

type mapextra struct {  
    overflow [2]*[]*bmap    // 溢出桶，只有当key&value没有对应指针时使用
    nextOverflow *bmap      // 下一个可用的溢出桶
}

// A bucket for a Go map.
type bmap struct {
	// 首先是tophash，存储了桶内哈希value的高八位byte值
	tophash [bucketCnt]uint8   // bucketCnt预设值为8
	
	// 接下来还有8个key和8个value
	// 最后是一个 overflow *bmap 指针
}
```

通过阅读源码，我们发现哈希桶是Golang中Map的基本存储单位。每个哈希桶存储了8对key-value，当发生溢出（地址冲突）时以指针链表的方式连接下个哈希桶。 在具体的Map中是以桶指针数组的形式包含了所有元素。 在源码中可以看到，桶数组有两个：buckets和oldbuckets。这是为扩容做准备的，后文会详述。

## 初始化
makemap函数是Map的初始化方法.

```go
// makemap implements a Go map creation make(map[k]v, hint)
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap {

    // 检测map size是否合法
	if sz := unsafe.Sizeof(hmap{}); sz > 48 || sz != t.hmap.size {
		println("runtime: sizeof(hmap) =", sz, ", t.hmap.size =", t.hmap.size)
		throw("bad hmap size")
	}

    // 将非法hint值置0
	if hint < 0 || hint > int64(maxSliceCap(t.bucket.size)) {
		hint = 0
	}

    // 检测key类型是否合法
	if !ismapkey(t.key) {
		throw("runtime.makemap: unsupported map key type")
	}

	// 通过编译和映射方法检测key和value的size是否合法
	if t.key.size > maxKeySize && (!t.indirectkey || t.keysize != uint8(sys.PtrSize)) ||
		t.key.size <= maxKeySize && (t.indirectkey || t.keysize != uint8(t.key.size)) {
		throw("key size wrong")
	}
	if t.elem.size > maxValueSize && (!t.indirectvalue || t.valuesize != uint8(sys.PtrSize)) ||
		t.elem.size <= maxValueSize && (t.indirectvalue || t.valuesize != uint8(t.elem.size)) {
		throw("value size wrong")
	}

	// 对只读变量值进行检测
	if t.key.align > bucketCnt {
		throw("key align too big")
	}
	// 省略多项检测

	// 初始化B值，以获取足够大的容量
	B := uint8(0)
	for ; overLoadFactor(hint, B); B++ {
	}

	// 分配内存并初始化哈希表
	// 如果 B == 0，哈希桶将稍后分配
	// 如果hint值很大，初始化内存会需要段时间
	buckets := bucket
	var extra *mapextra
	if B != 0 {
		var nextOverflow *bmap
		buckets, nextOverflow = makeBucketArray(t, B)
		if nextOverflow != nil {
			extra = new(mapextra)
			extra.nextOverflow = nextOverflow
		}
	}

	// 初始化Hmap
	if h == nil {
		h = (*hmap)(newobject(t.hmap))
	}
	h.count = 0
	h.B = B
	h.extra = extra
	h.flags = 0
	h.hash0 = fastrand()
	h.buckets = buckets
	h.oldbuckets = nil
	h.nevacuate = 0
	h.noverflow = 0

	return h
}
```
可以看出，makemap的主要工作有三个部分：变量及类型检测、哈希表内存分配及初始化、Hmap的初始化。 再看一下其中的makeBucketArray：

```go
func makeBucketArray(t *maptype, b uint8) (buckets unsafe.Pointer, nextOverflow *bmap) {
	base := uintptr(1 << b)
	nbuckets := base
	// 如果b很小，溢出可能性不大，没必要过度运算
	if b >= 4 {
		nbuckets += 1 << (b - 4)
		sz := t.bucket.size * nbuckets
		up := roundupsize(sz)
		// 根据实际申请的内存空间调整桶的个数
		if up != sz {
			nbuckets = up / t.bucket.size
		}
	}
	buckets = newarray(t.bucket, int(nbuckets))
	if base != nbuckets {
		// 对overflow哈希桶预分配内存
		nextOverflow = (*bmap)(add(buckets, base*uintptr(t.bucketsize)))
		last := (*bmap)(add(buckets, (nbuckets-1)*uintptr(t.bucketsize)))
		last.setoverflow(t, (*bmap)(buckets))
	}
	return buckets, nextOverflow
}

```
从代码可以看出，当算出桶的个数不等于 1<<b 时，才会生成溢出桶指针nextOverflow。换句话说，hmap也才会使用mapextra。

## 扩容机制
扩容机制对Map的性能和可用性有较大影响。一个好的扩容机制至少要做到以下两点：
	
1. 扩容后Map性能不受影响；
2. 扩容过程中Map读写不被阻塞。

我们来看Golang中Map是怎么实现的。  
扩容的触发发生在新增key-value过程中：

```go
// func mapassign
	if !h.growing() && (overLoadFactor(int64(h.count), h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)    //触发扩容
		goto again 
	}
```
在Map不在扩容过程的前提下，当达到最大负载因子或者溢出桶数量过多时则需要执行`hashGrow(t,h)`函数进行扩容。

```go
func hashGrow(t *maptype, h *hmap) {
	// 如果达到负载因子，增量扩容
	// 如果只是溢出桶过多，则保持桶数量“原量增长”（这个稍后再说）
	bigger := uint8(1)
	if !overLoadFactor(int64(h.count), h.B) {
		bigger = 0
		h.flags |= sameSizeGrow
	}
	// oldbuckets指向当前桶，并生成新的桶数组
	oldbuckets := h.buckets
	newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger)

	flags := h.flags &^ (iterator | oldIterator)
	if h.flags&iterator != 0 {
		flags |= oldIterator
	}
	// 确定增长，更新hmap属性，h.buckets指向新的桶数组
	h.B += bigger
	h.flags = flags
	h.oldbuckets = oldbuckets
	h.buckets = newbuckets
	h.nevacuate = 0
	h.noverflow = 0

	if h.extra != nil && h.extra.overflow[0] != nil {
		// h.extra.overflow[0]和h.extra.overflow[1]值做交换
		if h.extra.overflow[1] != nil {
			throw("overflow is not nil")
		}
		h.extra.overflow[1] = h.extra.overflow[0]
		h.extra.overflow[0] = nil
	}
	if nextOverflow != nil {
		if h.extra == nil {
			h.extra = new(mapextra)
		}
		h.extra.nextOverflow = nextOverflow
	}

	// 实际拷贝操作在growWork()和evacuate()中执行
}
```
hashGrow()做了一些准备性的工作，`growWork()`主要是判断当前是否在迁移状态，而实际内存迁移主要在`evacuate()`中执行。由于`evacuate()`代码量较大，以下只简述其逻辑。
首先需要判断是不是等量扩容：

- 如果是等量扩容，即overflow buckets数量超过一定的量但又未达到装载因子，则需要在保持原容量的基础上对数据进行迁移，提高buckets的利用率；
- 如果是增量扩容（扩容到原来两倍），则需要将新的桶数组分为低位x和高位y的等量两部分，在x中插入第一个值。重新计算哈希值对原来的数据进行遍历并逐个迁移，以分散到新的哈希桶数组中。迁移完成后，oldbuckets置为nil。
- 在桶的迁移过程中，如果发生插入操作会直接插入到新桶。有个evacuated标记判断是否已迁移，迁移过程中的删改直接会把最终结果记录在newbuckets，oldbuckets中对应数据标记为evacuated。


## 增删改查
_鉴于原码篇幅有点长，这一部分同样不贴代码只说逻辑。_
### 查找／删除
作为增删改的基础逻辑先说查找。除去一些异常以及并发检查，查找过程总结起来可分为6步：

1. 计算key的哈希值hash；
2. 对 (1<<B - 1) 进行取余运算，求出key的桶序号m；
3. 取出hash的高八位，与桶bmap中tophash数组中元素进行比较，初步判断value位置；
4. 根据value位置再对key进行比较确认；
5. 重复3、4步直至确认key,取出对应value；
6. 如遍历完对应哈希桶仍未找到key，则返回value类型初始值。

删除过程基于查找，在找到目标后：将key&value置为初始值（清空内存），清理tophash，最后map长度减1即可。

### 插入／更新
插入／更新过程同样与查找很像。首先当然是要计算key的哈希值查找看是否已存在，如果存在则直接更新。如果不存在，插入的过程如下：

1. 遍历目标桶时，如果有空位，对空位进行标记。确认key不存在后，在标记的空位插入key&value；
2. 如果目标桶没有空位，检查最大负载因子和已有的溢出桶的数量，如果达到最大负载或者已有溢出桶过多则需要扩容后再插入；
3. 如果目标桶没有空位并且也不需要扩容，overflow指针会指向一个新生成的溢出桶，将新值插入溢出桶。

---
Reference：

- [如何设计并实现一个线程安全的 Map ？(上篇)](https://halfrost.com/go_map_chapter_one/)