---
title: '[Go]基于Gin的Web服务端构建'
date: 2018-10-24 18:09:46
tags: Go Gin
---

_关于Gin的基本使用方法可移步[Gin中文文档](https://github.com/skybebe/gin-doc-cn#response-header)_  

本文以个人实际工程经验为基础，介绍了一种简易但实用的Web服务端构建方式，demo源码见文末。
## 包含状态的HTTPServer封装
要起一个Web服务，首先需要有一个server监听接口并接收http请求。Go在`net/http`包中有内置的http.server，结构包含了监听端口号Addr、处理方法Handler以及超时设置等属性。有一点不足的是，其没有一个属性或方法能返回当前server的状态。因而需要进行简单封装：
  
```go
type HTTPServer struct {
	server *http.Server
	status string
}

//status状态包括
const (
	starting = "starting"
	running  = "running"
	stopped  = "stoped"
)
```
接着需要一个server的初始化方法：

```go
func NewServer(addr string) *HTTPServer {
	srv := &http.Server{
		Addr:           addr,
		Handler:        GetHandler(),
		ReadTimeout:    10 * time.Second,
		WriteTimeout:   10 * time.Second,
		MaxHeaderBytes: 1 << 20,
	}

	return &HTTPServer{
		server: srv,
		status: starting,
	}
}
```
NewServer()方法对server的Addr，Handler、读写超时、header最大容量等做了初始化，并将其初始状态设为starting。其中的GetHandler()方法下下节会有详细介绍。  
在设定完初始状态后还需要有个interface方法集，用于控制server，及查看其状态：

```go
type IServer interface {
	Start()
	Running() bool
	Stop()
}

func (s *HTTPServer) Start() {
	s.status = running
	err := s.server.ListenAndServe()
	if err != nil {
		s.status = stopped
		os.Exit(1)
	}
}

func (s *HTTPServer) Running() bool {
	return s.status == running
}

func (s *HTTPServer) Stop() {
	if s.server != nil {
		fmt.Println("Server is closed")
	}
}

```
Start()方法用于启动server——`s.server.ListenAndServe()`，并将其状态置为running。Running()用于查看server是否属于running状态。Stop()则在服务终止时打印日志通知管理员。

## 服务运行状态处理
服务运行过程中，除了要对网络请求进行处理外，还需要处理一些系统异常信号。

```go
func Run(servers ...IServer) {

	// defer panic
	defer func() {
		if err := recover(); err != nil {
			log.Fatalf("server start err: %+v", err)
		}
	}()

	// start servers
	for _, s := range servers {
		go s.Start()
	}

	// check servers status
	for {
		count := 0
		for _, s := range servers {
			if s.Running() {
				count++
			}
		}
		time.Sleep(1)
		if count == len(servers) {
			log.Println("servers starting ...")
			break
		}
	}

	// deal with signal
	sig := make(chan os.Signal, 1)
	signal.Notify(sig, os.Interrupt, syscall.SIGTERM)

	for exitSig := range sig {
		log.Println("receive exit signal ", exitSig)
		switch exitSig {
		case os.Interrupt:
			fallthrough
		case syscall.SIGTERM:
			log.Println("exiting ...")
			for _, s := range servers {
				s.Stop()
			}
			log.Println("exit success")
			os.Exit(0)
		}
	}

}
```
Run()方法用于server的运行。通过注释可把方法分为四个部分，首先是启动过程中的panic异常捕获；然后使用server的Start()方法启动服务；接着检查server属性，并输出日志告诉管理员管理员启动状态；最后还需要考虑的是，当发生系统中断os.Interrupt，或者管理员手动终止服务syscall.SIGTERM时，不仅要终止服务，还需要对信号进行监听并输出服务终止日志告知管理员。

## 处理函数的封装
前文提到的http处理Handler本质上是：

```go
// package http
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
gin框架中有对应的结构体类型`gin.Engine`对Handler interface进行了继承。实际上`gin.Engine`所包含的内容也不仅ServeHTTP()方法这么简单。

```go
var handler *gin.Engine
var once sync.Once

func GetHandler() *gin.Engine {
	once.Do(func() {
		handler = gin.Default()
		RegisteMiddleware(handler)
	})
	return handler
}

```
GetHandler()方法对`*gin.Engine`类型的handler进行了初始化。为保证全局的唯一性，使用了sync.Once对初始化语句以及中间件的注册进行了限定。  
接下来，我们要考虑的是建立一个不同业务方法的共通模版actFunc()。

```go
func ping(c *gin.Context) (status int, result interface{}, err error) {
	return server.StatusOK, "ping success", nil
}
```
以上以ping()方法为例建立了个方法模版。可以看到，入参为`*gin.Context`，该类型包含了请求的上下文信息，开发者可以拿到所有的请求数据进行处理。回参有三个：(status int, result interface{}, err error)，分别对应http请求状态码，请求结果以及error信息。以上基本包含了所有的处理结果信息，另外如“response header”可以通过中间件或直接在业务方法中对`*gin.Context`的指针对象进行处理。  
此外，对于每一种请求，有一些共通的操作可以提前写好,以下对实际业务方法actFunc()进行了简单封装。

```go
func wrapHandler(actFunc func(c *gin.Context) (status int, res interface{}, err error)) gin.HandlerFunc {
	return func(c *gin.Context) {
		sucChan := make(chan interface{}, 1)
		var (
			status int
			res    interface{}
			err    error
		)
		
	// get actFunc & recover()
		go func(c *gin.Context) {
			defer func() {
				if r := recover(); r != nil {
					status, res, err = StatusInternalServerError, nil, errors.New("unknown error")
					log.Println("panic error ")
					sucChan <- true
				}
			}()
			status, res, err = actFunc(c)
			sucChan <- true
		}(c)

    // time out control
		select {
		case <-time.After(time.Second):
			status, res, err = StatusRequestTimeout, nil, errors.New("time out")
			log.Println("time out")
		case <-sucChan:
			log.Println("http request success")
		}

    // write response
		WriteResponse(c, status, res, err)
	}
}
```
通过注释可以看出封装方法wrapHandler()进行了三步操作：首先获取actFunc()的返回值，并对方法运行过程中的异常进行捕获；其次是对每次请求进行超时设置，demo中设的是1秒；最后，对response进行编辑用于返回。  
现在，server有了，server的handle有了,处理方法actFunc()及其封装wrapHandler()也有了。接下来就是要把它们串联起来：

```go
// package server
// register GET method, other methods are same
func GET(path string, actFunc func(c *gin.Context) (int, interface{}, error)) {
	GetHandler().GET(path, wrapHandler(actFunc))
}

// package router
func init() {
	server.GET("/ping", ping)
}
```
上述代码注册了一个GET()方法，并调用了`*gin.Engine`类型的原生GET()请求方法，将path和actFunc()作为入参传入，对server和handler进行了绑定。开发者在使用过程中只需以`server.GET("/ping", ping)`形式进行调用即可。

## 中间件与返回方法

在上节GetHandler()方法中，对中间件进行了注册，以下作为示例对RegisteMiddleware()进行实现。

```go
func cors() gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Writer.Header().Set("Access-Control-Allow-Origin", "*")
		c.Writer.Header().Set("Access-Control-Allow-Methods", "POST,GET,PUT,OPTIONS,DELETE")
		c.Writer.Header().Set("Access-Control-Allow-Headers", "Content-Type")

		c.Next()
	}
}

func RegisteMiddleware(c *gin.Engine) {
	c.Use(cors())
	c.Use(gin.Recovery())
}
```
示例中注册了两个Middleware，一个是Gin原生的异常捕获gin.Recovery()，cors()则是对response header进行了设置以支持跨域访问。此外，常见的Middleware还包括一些统计和打点信息，开发者可根据业务所需进行定制。  

实际工程中有时还需要对返回结构体进行一定封装：

```go
type MetaData struct {
	Status  int
	Message string
}

type ReturnStr struct {
	Meta   MetaData
	Result interface{}
}

func WriteResponse(c *gin.Context, status int, res interface{}, err error) {

	message := "OK"
	if err != nil {
		message = err.Error()
	}
	c.JSON(StatusOK, ReturnStr{
		Meta: MetaData{
			Status:  status,
			Message: message,
		},
		Result: res,
	})
}
```
在WriteResponse()中只要请求成功达到服务器，都会返回200的状态码。实际返回结构体ReturnStr分为两个部分，Meta返回定制的状态码和信息，Result返回请求结果。这样做的好处是，可以对http原生的状态码进行定制和扩展，便于发现问题和前后端业务逻辑的交互。

## 启动服务
万事具备，只需要在main函数中启动server就可以了。

```go
func main() {
	httpServer := server.NewServer(":8989")
	server.Run(httpServer)
}
```
如上，先初始化了httpServer监听8989端口，再通过server.Run()将其跑起来就可以了。

----
以上就是Web服务端的基本架构方法，希望对你有帮助。
[代码仓库地址](https://github.com/Wan-Mi/ginWebService)
