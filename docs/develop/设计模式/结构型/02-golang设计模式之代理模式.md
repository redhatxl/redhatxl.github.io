# 2.代理模式

## 一 代理模式简介

代理模式（英语：Proxy Pattern）是程序设计中的一种设计模式。

所谓的代理者是指一个类可以作为其它东西的接口。代理者可以作任何东西的接口：网络连接、内存中的大对象、文件或其它昂贵或无法复制的资源。

著名的代理模式例子为引用计数（英语：reference counting）指针对象。

当一个复杂对象的多份副本须存在时，代理模式可以结合享元模式以减少内存用量。典型作法是创建一个复杂对象及多个代理者，每个代理者会引用到原本的复杂对象。而作用在代理者的运算会转送到原本对象。一旦所有的代理者都不存在时，复杂对象会被移除。

## 二 代码

### 2.1 代码

```go
package proxy

const (
    maxRequest int =3
)

// 定义公共server接口
type Server interface {
    handleRequest(url, method string) (int, string)
}

// 定义具体实体类
type Application struct {
}

func (a Application) handleRequest(url , method string) (int, string) {
    if url == "" && method == "" {
        return 401, "Url and Method is empty"
    }
    if url == "/" && method == "POST" {
        return 200, "ok"
    }
    return 404, "not found"
}

// 定义代理 nginx 类型, 内部包含原始nginx
type Nginx struct {
    a *Application
    maxAllowedRequest int
    rateLimter map[string]int
}

func NewNginx(limit int) *Nginx  {
    return &Nginx{
        a: &Application{},
        maxAllowedRequest: limit,
        rateLimter: make(map[string]int),
    }
}

// nginx 新增业务逻辑
func (n *Nginx) checkRateLimit(url string) bool  {
    if n.rateLimter[url] == 0 {
        n.rateLimter[url] = 1
    }

    if n.rateLimter[url] > n.maxAllowedRequest {
        return false
    }
    n.rateLimter[url] = n.rateLimter[url] +1
    return true
}

// nginx 实现原始业务
func (n *Nginx) handleRequest(url, method string) (int,string)  {
    if !n.checkRateLimit(url) {
        return 403, "Not Allowed"
    }
    return n.a.handleRequest(url, method)
}


```



### 2.2 测试

```go
package proxy

import (
    "reflect"
    "testing"
)

type http struct {
    limit int
    url string
    method string
}

type Rest struct {
    code int
    msg string
}

func TestNewNginx(t *testing.T) {
    //got1 := http{
    //    limit: 2,
    //    url: "/",
    //    method: "GET",
    //}

    got2 := http{
        limit: 2,
        url: "/",
        method: "POST",
    }
    want1 := Rest{
        code: 200,
        msg: "ok",
    }

    //n := NewNginx(got1.limit)
    //code, msg := n.handleRequest(got1.url,got1.method)
    //
    //if !reflect.DeepEqual(want1.code, code) {
    //    t.Errorf("want %d, got: %d", want1.code, code)
    //}
    //if !reflect.DeepEqual(want1.msg, msg) {
    //
    //    t.Errorf("want %s, got: %s", want1.msg, msg)
    //}


    n2 := NewNginx(got2.limit)
    code2, msg2 := n2.handleRequest(got2.url,got2.method)

    if !reflect.DeepEqual(want1.code, code2) {
        t.Errorf("want %d, got: %d", want1.code, code2)
    }
    t.Logf("test success %d,%d",want1.code, code2)


    if !reflect.DeepEqual(want1.msg, msg2) {
        t.Errorf("want %s, got: %s", want1.msg, msg2)
    }
    t.Logf("test success: %s,%s",want1.msg, msg2)

}
```



## 三 应用场景

- 远程(Remote)代理：为一个位于不同的地址空间的对象提供一个本地 的代理对象，这个不同的地址空间可以是在同一台主机中，也可是在 另一台主机中，远程代理又叫做大使(Ambassador)。
- 虚拟(Virtual)代理：如果需要创建一个资源消耗较大的对象，先创建一个消耗相对较小的对象来表示，真实对象只在需要时才会被真正创建。
- Copy-on-Write代理：它是虚拟代理的一种，把复制（克隆）操作延迟 到只有在客户端真正需要时才执行。一般来说，对象的深克隆是一个 开销较大的操作，Copy-on-Write代理可以让这个操作延迟，只有对象被用到的时候才被克隆。
- 保护(Protect or Access)代理：控制对一个对象的访问，可以给不同的用户提供不同级别的使用权限。
- 缓冲(Cache)代理：为某一个目标操作的结果提供临时的存储空间，以便多个客户端可以共享这些结果。
- 防火墙(Firewall)代理：保护目标不让恶意用户接近。
- 同步化(Synchronization)代理：使几个用户能够同时使用一个对象而没有冲突。
- 智能引用(Smart Reference)代理：当一个对象被引用时，提供一些额外的操作，如将此对象被调用的次数记录下来等。

## 四 特点

### 4.1 优点

- 代理模式能够协调调用者和被调用者，在一定程度上降低了系 统的耦合度。
- 远程代理使得客户端可以访问在远程机器上的对象，远程机器 可能具有更好的计算性能与处理速度，可以快速响应并处理客户端请求。
- 虚拟代理通过使用一个小对象来代表一个大对象，可以减少系 统资源的消耗，对系统进行优化并提高运行速度。
- 保护代理可以控制对真实对象的使用权限。

### 4.2 缺点

- 由于在客户端和真实主题之间增加了代理对象，因此 有些类型的代理模式可能会造成请求的处理速度变慢。
- 实现代理模式需要额外的工作，有些代理模式的实现 非常复杂。

## 参考链接

* https://github.com/silsuer/golang-design-patterns
* https://www.bookstack.cn/read/Design-Pattern/lesson9-README.md
* https://refactoringguru.cn/design-patterns/proxy