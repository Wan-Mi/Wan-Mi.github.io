---
title: '[Go]slice、map、channel与struct的初始化'
date: 2018-01-20 01:09:07
tags: Go
---
本文介绍golang中的三种引用类型slice、map和channel以及结构体的初始化方法以及一些潜在的坑。
先上代码：

```go
package main

import (
    "fmt"
)

type Location struct {
    UUID       string
    Longitude  float64
    Latitude   float64
    DetectedAt int64
}

func main() {

    // slice
    var s1 []int
    s1 = append(s1, 1)

    s2 := make([]int,0)
    s2 = append(s2, 2)

    s3 := []int{}
    s3 = append(s3, 3)

    fmt.Println(s1,s2,s3)

    // map
    var m1 map[string]int
    //    m1["key"] = 1

    m2 := make(map[string]int)
    m2["key"] = 2

    m3 := map[string]int{}
    m3["key"] = 3

    fmt.Println(m1,m2,m3)

    // channel
    //    var c1 chan int
    //    go func() {
    //        c1<- 1
    //    }()
    //    fmt.Println(<-c1)

    c2 := make(chan int,0)
    go func() {
    c2<- 2
    }()
    fmt.Println(<-c2)

    // stuct
    var t1 *Location
    //    t1.UUID = "abc"

    t2 := new(Location)
    t2.UUID = "abc"

    t3 := &Location{}
    t3.UUID = "abc"

    fmt.Println(t1,t2,t3)
}
```
上述代码在 https://play.golang.org 中的运行结果为：

```
[1] [2] [3]
map[] map[key:2] map[key:3]
2
<nil> &{abc 0 0 0} &{abc 0 0 0}
```
以下进行详细说明：
**slice：** 可以使用var声明、make、以及字面量三种方式初始化。可以看出s1、s2、s3均可以正常使用append方法进行slice扩充。比较有意思的是var声明后的slice初始值为nil，但使用起来不会报错（panic）。 对此Golang.org blog中的说法是：

```
a nil slice is functionally equivalent to a zero-length slice, even
though it points to nothing. It has length zero and can be appended
to, with allocation.
```
也就是说nil slice和zero-length slice在功能上是相等的。而且，为了避免不表要的内存分配，**个人建议使用`var s1 []int`的方式初始化。**

**map：** 使用了var声明、make、以及字面量三种方式初始化。其中m2和m3的效果相同，而m1的赋值之所以被注释掉，因为其与slice不同，nil map的赋值会引起panic。

**channel：** 使用了var声明、make两种方式初始化。其中c1被注释掉是因为nil channel无法传递信息，会造成死锁。所以只建议使用`make(chan int,len)`方法进行初始化，也方便用第二个参数设置channel的容量。

**struct：** 使用了var声明、new、以及字面量三种方式初始化。t1被注释掉的原因同m1相同，对nil的结构体内部赋值会造成panic。**这也是新手经常会踩的一个坑，当函数的返回值为结构体时一定要判断是否为空，否则一访问其中字段就崩溃。** 而new和字面量都是比较安全的初始化方式。
