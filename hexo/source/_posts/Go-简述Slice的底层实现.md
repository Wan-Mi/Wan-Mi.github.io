---
title: '[Go]简述Slice的底层实现'
date: 2018-03-07 00:36:42
tags: Go
---

## 数据结构
作为比较，分别查看array和slice的数据结构：
 
```go
// package types
// type.go
type Array struct {
	len  int64
	elem Type
}

// All types implement the Type interface.
type Type interface {
	Underlying() Type
	String() string
}

// package runtime
// slice.go 
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
从结构中可以看出，虽然两者功能相似，但结构完全不同。slice通过内部指针的方式引用了一个数组片段，以实现变长方案。len表示slice元素数量，cap表示slice最大元素容量。由于这种特殊的结构，虽然slice本身是结构体，但却是引用类型。

---
_unsafe.Pointer：通过unsafe包中的注释我们可以发现“任何类型的指针值都可以转换为unsafe.Pointer”。换句话说，unsafe.Pointer是一个可以指向任何类型的指针。其类型ArbitraryType并非一个具体类型，更像一个占位符。_

```go 
// unsafe.go
type ArbitraryType int
type Pointer *ArbitraryType
```
---

## 重组（reslice）
reslice是指基于已有slice创建新slice的过程。其最大特点是新slice依旧指向原底层数组，因而新slice对原slice的取值范围不可超过原slice的cap。

```go
func resliceDemo() {
	s := []int32{
		0, 1, 2, 3, 4, 5,
	}
	fmt.Println("0-2:", s[:3])
	fmt.Println("4-5:", s[3:6][1:])
	// panic: runtime error
	// fmt.Println("4-5", s[4:7])
	
	t := s[1:]
	fmt.Printf("%p,%p\n", s, t)
}

```
运行结果：

```
0-2: [0 1 2]
4-5: [4 5]
0xc42000a240,0xc42000a244
```
前两行输出可以看出，通过reslice的方式构建新slice时，取值范围为“左闭右开”，且允许连续截取。最后一行中，t是从s的第二个元素开始截取的新slice，由于指向原底层数组，其地址为原地址+4（int32，4字节），输出符合预期。

## 扩容（append）
append函数可向slice尾部添加元素。

```go
func appendDemo() {
    s := []int32{
		  0, 1, 2, 3, 4, 5,
	   }
    s1 := append(s, 6)
    s2 := append(s1, 7, 8)
    fmt.Println("s2:", s2)
    fmt.Println("cap:", cap(s), cap(s1), cap(s2))
    fmt.Printf("addr: %p,%p\n", s, s1)
}
```
运行结果：

```
s2: [0 1 2 3 4 5 6 7 8]
cap: 6 12 12
addr: 0xc42000a260,0xc4200180f0
```
demo中，s通过字面量形式初始化，len和cap为元素个数6。append函数可一次在slice尾部添加一个或多个元素。当元素个数大于cap时，slice的cap会进行扩容。当slice进行扩容时，会重新生成一个大容量底层数组，再将原数组值和添加值复制到新数组。由于数组copy通过值传递，会消耗等量内存且耗时。个人建议在slice初始化时设定足够大的cap用来append，可减少内存和时间损耗。  
通常扩容后的slice cap会增大到原来两倍，当元素个数超过1024时，增大倍数会减少到1.25倍。

```go
func appendCapDemo() {
   m, n := 1000, 5000
	a, b := make([]int32, m, m), make([]int32, n, n)
	a1, b1 := append(a, 1), append(b, 1)
	fmt.Println(cap(a1), cap(b1))
}
```
运行结果：

```
2048 6816
```

## 拷贝（copy）
copy函数可用于slice拷贝。

```go
func appendCapDemo() {
	s := []int32{
		0, 1, 2, 3, 4, 5,
	}	
	s3 := s[:5]
	s4 := s[5:]
	s5 := copy(s3, s4) // des:s3, src:s4
	fmt.Println(s3, s4, s5)
}
```
运行结果：

```
[5 1 2 3 4] [5] 1
```
demo中可看出，slice拷贝更准确的说应该是slice的元素拷贝。其所要拷贝的元素个数以短slice len为准。有两小点要注意的是：copy函数两个参数，目标slice在前，来源slice在后；copy函数运行成功后目标slice直接被改变，copy函数的返回值为拷贝的元素个数（打印的S5），并非返回slice。

---
Reference：

- [《Go 1.5源码剖析》](https://github.com/qyuhen/book/blob/master/Go%201.5%20%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90%20%EF%BC%88%E4%B9%A6%E7%AD%BE%E7%89%88%EF%BC%89.pdf) 
- [[译]Go里面的unsafe包详解](https://gocn.io/question/371)