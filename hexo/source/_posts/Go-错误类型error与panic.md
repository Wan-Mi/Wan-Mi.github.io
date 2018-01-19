---
layout: post
title: '[Go]错误类型error与panic'
date: 2018-01-19 17:16:24
tags: Go
---
## error类型与package errors
先看error类型定义：

type error interface {
Error() string
}
由定义可看出error是一个含有Error()函数的接口。换句话说，任何结构体只要实现了该函数就继承了error类型。常见的创建error类型对象的方式有两种：

errors.New("errorString")
fmt.Errorf("errorString")
其中，package errors源码为：

package errors

// New returns an error that formats as the given text.
func New(text string) error {
return &errorString{text}
}

// errorString is a trivial implementation of error.
type errorString struct {
s string
}

func (e *errorString) Error() string {
return e.s
}

errors包中的errorString类型实现了Error()函数，并通过New()方法初始化。
fmt.Errorf()源码：

func Errorf(format string, a ...interface{}) error {
return errors.New(Sprintf(format, a...))
}
可见，fmt.Errorf()只是换种方式的errors.New()。

## panic()与recover()
panic和recover都是Go标准库中的函数。panic的字面意思是“恐慌的”，往往是指那些无法修复的错误。比如数字除以0，数组越界，空指针取值等。由于panic类型错误往往可使得程序崩溃，因而需要使用recover对panic进行捕获，从而做进一步处理避免崩溃。
还是先看定义：

func panic(v interface{})
func recover() interface{}
其中，panic是入参为空接口（可传任何值）返参为空的一个函数，而recover正好反过来。当二者配合使用时，recover捕获的正好是panic传入的值。使用示例：

func testPanic() {
// 必须使用defer关键字在延迟调用函数内直接recover()
defer func() {
fmt.Println(recover())
}()

a := 0
fmt.Println(1 / a)
}
上述代码的运行结果为：

runtime error: integer divide by zero
例子中故意使用了一个除数为0的运算触发panic（当然也可以直接调用`panic("message")`函数触发）。不过panic被成功捕获并打印出来，程序得以继续运行。如果panic没有被捕获，错误信号会一直向外层传递产生运行时错误，导致程序崩困。
**值得说明的是，recover()函数必须放在defer func()中直接延迟调用才能生效，`defer recover()`这种写法也不行。如果不用defer关键字，panic会直接触发没有机会运行recover；而如果不放在闭包中，recover将无法捕获所在函数或方法层的panic。**

最后，是否需要将一些error转为panic。个人建议是尽量对每个error进行处理，只有当在主流程遇到无法修复的错误时，才考虑使用panic()。

