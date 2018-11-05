---
title: '[Go]并发同步的两种方式'
date: 2018-01-22 18:49:40
tags: Go
---

适用于并发查询计算等业务场景。
## 使用channel
```go
// channel
func TestChan() {
	numbers := []int32{
		1, 2, 3, 4, 5,
	}
	ch := make(chan int32, 0)

	for _, x := range numbers {
		go func(x int32) {
			x++
			ch <- x
		}(x)
	}

	resNums := []int32{}
	for range numbers {
		select {
		case y := <-ch:
			resNums = append(resNums, y)
		case <-time.After(5 * time.Second):
			fmt.Println("time out")
		}
	}
	fmt.Println("results is:", resNums)
}
```
上文首先定义了包含5个数字的`int32`型Slice以及对应长度类型的Channel；然后分5个Goroutine对Slice中的每个数字进行自增运算形成新的Slice并输出。运行结果为（并发输出顺序随机，以下只是其中一种可能）：  

```  
results is: [2 3 6 2 5]  
```
该方法的基本思想是：将每个Goroutine的处理结果通过Chan传到指定Goroutine，使用select语句收集传输过来的内容。属于"以通讯共享内存"的内容安全策略。 
注意：Channel的收发个数要保持一致，当收多于发时会造成死锁，最好添加超时语句做异常情况保护`case <-time.After(10 * time.Second):`。

## 使用sync.WaitGroup
```go
// waitGroup
func TestWaitGroup() {
	numbers := []int32{
		1, 2, 3, 4, 5,
	}
	resNums := []int32{}

	var wg sync.WaitGroup
	var mt sync.Mutex

	wg.Add(len(numbers))
	for _, val := range numbers {
		go func(t int32) {
			defer wg.Done()
			mt.Lock()
			resNums = append(resNums, t)
			mt.Unlock()
		}(val)
	}
	wg.Wait()

	fmt.Println("results is:", resNums)
}

```
功能与第一段代码相同，运行结果： 
 
```  
results is: [5 1 2 3 4]  
```  
该方法的基本思想是：声明一个长度为`len`的`sync.WaitGroup`队列`wg`,各个Goroutine中每执行一次`wg.Done()`队列长度减1，`wg.Wait()`会对队列长度进行判断，为0时程序才往下进行。该方法直接对内存进行了共享。  
注意：`wg.Add(len)`中的len应等于`wg.Done()`的执行次数,len一直未置0会造成死锁。  
另: 由于`append()`函数非并发安全，需要添加并发锁`sync.Mutex`。

--
建议：从实际使用的角度来说使用WaitGroup会使代码更为简洁，但从内容安全以及设计者初衷来说应选择使用channel。


