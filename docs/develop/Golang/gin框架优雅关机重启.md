# Gin框架优雅关机和重启

# 一 背景

我们编写的Web项目部署之后，经常会因为需要进行配置变更或功能迭代而重启服务，单纯的`kill -9 pid`的方式会强制关闭进程，这样就会导致服务端当前正在处理的请求失败，那有没有更优雅的方式来实现关机或重启呢？

> 阅读本文需要了解一些UNIX系统中`信号`的概念，请提前查阅资料预习。

# 二 实现方案

## 2.1 Linux 信号

### 2.1.1 信号名称与编号

每个信号都有一个名字和编号，这些名字都以“SIG”开头，例如“SIGIO ”、“SIGCHLD”等等。
信号定义在`signal.h`头文件中，信号名都定义为正整数。
具体的信号名称可以使用`kill -l`来查看信号的名字以及序号，信号是从1开始编号的，不存在0号信号。kill对于信号0又特殊的应用。

```shell
root@1204nStrive:~# kill -l
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX
```

### 2.1.2 信号处理

信号的处理有三种方法，分别是：忽略、捕捉和默认动作

- 忽略信号，大多数信号可以使用这个方式来处理，但是有两种信号不能被忽略（分别是 `SIGKILL`和`SIGSTOP`）。因为他们向内核和超级用户提供了进程终止和停止的可靠方法，如果忽略了，那么这个进程就变成了没人能管理的的进程，显然是内核设计者不希望看到的场景
- 捕捉信号，需要告诉内核，用户希望如何处理某一种信号，说白了就是写一个信号处理函数，然后将这个函数告诉内核。当该信号产生时，由内核来调用用户自定义的函数，以此来实现某种信号的处理。
- 系统默认动作，对于每个信号来说，系统都对应由默认的处理动作，当发生了该信号，系统会自动执行。不过，对系统来说，大部分的处理方式都比较粗暴，就是直接杀死该进程。
   具体的信号默认动作可以使用`man 7 signal`来查看系统的具体定义。在此，我就不详细展开了，需要查看的，可以自行查看。也可以参考 《UNIX 环境高级编程（第三部）》的 P251——P256中间对于每个信号有详细的说明。

## 2.2 优雅关机

### 2.2.1 什么是优雅关机

优雅关机就是服务端关机命令发出后不是立即关机，而是等待当前还在处理的请求全部处理完毕后再退出程序，是一种对客户端友好的关机方式。而执行`Ctrl+C`关闭服务端时，会强制结束进程导致正在访问的请求出现问题。

### 2.2.2 实现优雅关机

Go 1.8版本之后，http.Server 内置的[Shutdown()](https://golang.org/pkg/net/http/#Server.Shutdown) 方法支持优雅关机，如下

# 三 实战

## 3.1 http.Server 内置Shutdown()方法

```go

package main

import (
	"context"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/gin-gonic/gin"
)

// @title Docker监控服务
// @version 1.0
// @description gin shutdown

// @contact.name API Support
// @contact.url http://www.swagger.io/support

// @license.name Apache 2.0
// @license.url http://www.apache.org/licenses/LICENSE-2.0.html

// @host 127.0.0.1:9009
// @BasePath
func main() {
	r := gin.Default()
	r.GET("/", func(c *gin.Context) {
		time.Sleep(5 * time.Second)
		c.String(http.StatusOK, "gin %s", "ok")
	})
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))

	server := http.Server{
		Addr:    ":8080",
		Handler: r,
	}

	go func() {
		if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			log.Fatal("server listen err:%s", err)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)

	// 在此阻塞
	<-quit

	ctx, channel := context.WithTimeout(context.Background(), 5*time.Second)

	defer channel()
	if err := server.Shutdown(ctx); err != nil {
		log.Fatal("server shutdown error")
	}
	log.Println("server exiting...")
}
```

如何验证优雅关机的效果呢？

上面的代码运行后会在本地的`8080`端口开启一个web服务，它只注册了一条路由`/`，后端服务会先sleep 5秒钟然后才返回响应信息。

我们按下`Ctrl+C`时会发送`syscall.SIGINT`来通知程序优雅关机，具体做法如下：

1. 打开终端，编译并执行上面的代码
2. 打开一个浏览器，访问`127.0.0.1:8080/`，此时浏览器白屏等待服务端返回响应。
3. 在终端**迅速**执行`Ctrl+C`命令给程序发送`syscall.SIGINT`信号
4. 此时程序并不立即退出而是等我们第2步的响应返回之后再退出，从而实现优雅关机。

## 3.2 优雅重启

优雅关机实现了，那么该如何实现优雅重启呢？

我们可以使用 [fvbock/endless](https://github.com/fvbock/endless) 来替换默认的 `ListenAndServe`启动服务来实现， 示例代码如下：

```go
package main

import (
	"log"
	"net/http"
	"time"

	"github.com/fvbock/endless"
	"github.com/gin-gonic/gin"
)

func main() {
	router := gin.Default()
	router.GET("/", func(c *gin.Context) {
		time.Sleep(5 * time.Second)
		c.String(http.StatusOK, "hello gin!")
	})
	// 默认endless服务器会监听下列信号：
	// syscall.SIGHUP，syscall.SIGUSR1，syscall.SIGUSR2，syscall.SIGINT，syscall.SIGTERM和syscall.SIGTSTP
	// 接收到 SIGHUP 信号将触发`fork/restart` 实现优雅重启（kill -1 pid会发送SIGHUP信号）
	// 接收到 syscall.SIGINT或syscall.SIGTERM 信号将触发优雅关机
	// 接收到 SIGUSR2 信号将触发HammerTime
	// SIGUSR1 和 SIGTSTP 被用来触发一些用户自定义的hook函数
	if err := endless.ListenAndServe(":8080", router); err!=nil{
		log.Fatalf("listen: %s\n", err)
	}

	log.Println("Server exiting")
}
```

如何验证优雅重启的效果呢？

我们通过执行`kill -1 pid`命令发送`syscall.SIGINT`来通知程序优雅重启，具体做法如下：

1. 打开终端，`go build -o graceful_restart`编译并执行`./graceful_restart`,终端输出当前pid(假设为43682)
2. 将代码中处理请求函数返回的`hello gin!`修改为`hello q1mi!`，再次编译`go build -o graceful_restart`
3. 打开一个浏览器，访问`127.0.0.1:8080/`，此时浏览器白屏等待服务端返回响应。
4. 在终端**迅速**执行`kill -1 43682`命令给程序发送`syscall.SIGHUP`信号
5. 等第3步浏览器收到响应信息`hello gin!`后再次访问`127.0.0.1:8080/`会收到`hello q1mi!`的响应。
6. 在不影响当前未处理完请求的同时完成了程序代码的替换，实现了优雅重启。

但是需要注意的是，此时程序的PID变化了，因为`endless` 是通过`fork`子进程处理新请求，待原进程处理完当前请求后再退出的方式实现优雅重启的。所以当你的项目是使用类似`supervisor`的软件管理进程时就**不适用**这种方式了。

# 注意点

1. 开启go协程，需要错误不等于http.ErrServerClosed 

```go
if err := rsv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
```

2. 定义信号，阻塞

```go
	// kill 默认会发送 syscall.SIGTERM 信号
	// kill -2 发送 syscall.SIGINT 信号，我们常用的Ctrl+C就是触发系统SIGINT信号
	// kill -9 发送 syscall.SIGKILL 信号，但是不能被捕获，所以不需要添加它
	// signal.Notify把收到的 syscall.SIGINT或syscall.SIGTERM 信号转发给quit
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)  // 此处不会阻塞
	<-quit  // 阻塞在此，当接收到上述两种信号时才会往下执行
	log.Println("Shutdown Server ...")
```

# 总结

无论是优雅关机还是优雅重启归根结底都是通过监听特定系统信号，然后执行一定的逻辑处理保障当前系统正在处理的请求被正常处理后再关闭当前进程。使用优雅关机还是使用优雅重启以及怎么实现，这就需要根据项目实际情况来决定了。