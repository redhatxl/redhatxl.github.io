# 3.抽象工厂模式

## 一 抽象工厂模式简介

抽象工厂模式（英语：Abstract factory pattern）是一种软件开发设计模式。抽象工厂模式提供了一种方式，可以将一组具有同一主题的单独的工厂封装起来。在正常使用中，客户端程序需要创建抽象工厂的具体实现，然后使用抽象工厂作为接口来创建这一主题的具体对象。客户端程序不需要知道（或关心）它从这些内部的工厂方法中获得对象的具体类型，因为客户端程序仅使用这些对象的通用接口。抽象工厂模式将一组对象的实现细节与他们的一般使用分离开来。


## 二 代码

### 2.1 开发代码

```go
package abstractfactory

type Cache interface {
    Get(key string) string
    Set(key, value string)
}

type Redis struct {
    date map[string]string
}

func NewRedis() *Redis {
    return &Redis{
        date: make(map[string]string),
    }
}

func (r *Redis) Get(key string) string  {
    if v, ok := r.date[key] ; ok {
        return v
    }
    return ""
}

func (r *Redis) Set(key,value string)  {
    r.date[key]=value
}

type Memcache struct {
    date map[string]string
}

func NewMemcache() *Memcache {
    return &Memcache{
        date: make(map[string]string),
    }
}

func (r *Memcache) Get(key string) string  {
    if v, ok := r.date[key] ; ok {
        return v
    }
    return ""
}

func (r *Memcache) Set(key,value string)  {
    r.date[key]=value
}

type CacheSampleFactory interface {
    CreateRedis() Cache
    CreateMemcache() Cache
}

type SampleFactory struct {
}

func NewSampleFactory() *SampleFactory {
    return &SampleFactory{}
}

func (s *SampleFactory) CreateRedis() Cache  {
    return NewRedis()
}

func (s *SampleFactory) CreateMemcache() Cache  {
    return NewMemcache()
}
```



### 2.2 测试

```go
package abstractfactory

import (
    "fmt"
    "testing"
)

func TestNewSampleFactory(t *testing.T) {
    s := NewSampleFactory()
    r := s.CreateRedis()

    r.Set("r", "rv")
    t.Logf(r.Get("r"))
    fmt.Println(r.Get("r"))

    m := s.CreateMemcache()
    m.Set("m","mv")
    t.Logf(m.Get("m"))
    fmt.Println(m.Get("m"))
}

```



## 三 应用场景

* 抽象工厂为你提供了一个接口， 可用于创建每个系列产品的对象。 只要代码通过该接口创建对象， 那么你就不会生成与应用程序已生成的产品类型不一致的产品。
* 如果你有一个基于一组的类，且其主要功能因此变得不明确，那么在这种情况下可以考虑使用抽象工厂模式。
* 在设计良好的程序中， *每个类仅负责一件事*。 如果一个类与多种类型产品交互， 就可以考虑将工厂方法抽取到独立的工厂类或具备完整功能的抽象工厂类中。

## 四 特点

- 优点: 抽象工厂模式除了具有工厂方法模式的优点外，最主要的优点就是可以在类的内部对产品族进行约束。所谓的产品族，一般或多或少的都存在一定的关联，抽象工厂模式就可以在类内部对产品族的关联关系进行定义和描述，而不必专门引入一个新的类来进行管理。
- 缺点: 产品族的扩展将是一件十分费力的事情，假如产品族中需要增加一个新的产品，则几乎所有的工厂类都需要进行修改。所以使用抽象工厂模式时，对产品等级结构的划分是非常重要的

## 参考链接

* https://juejin.cn/post/6844903705091768334
* https://github.com/silsuer/golang-design-patterns
* https://refactoringguru.cn/design-patterns/abstract-factory