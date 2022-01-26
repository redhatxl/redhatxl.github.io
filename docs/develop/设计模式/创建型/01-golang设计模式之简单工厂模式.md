# 1.简单工厂模式

## 一 简单工厂模式

简单工厂模式并不属于 GoF 23 个经典设计模式，但通常将它作为学习其他工厂模式的基础，它的设计思想很简单，其基本流程如下：

首先将需要创建的各种不同对象（例如各种不同的 Chart 对象）的相关代码封装到不同的类中，这些类称为具体产品类，而将它们公共的代码进行抽象和提取后封装在一个抽象产品类中，每一个具体产品类都是抽象产品类的子类；然后提供一个工厂类用于创建各种产品，在工厂类中提供一个创建产品的工厂方法，该方法可以根据所传入的参数不同创建不同的具体产品对象；客户端只需调用工厂类的工厂方法并传入相应的参数即可得到一个产品对象。


简单工厂模式（Simple Factory Pattern）：定义一个工厂类，它可以根据参数的不同返回不同类的实例，被创建的实例通常都具有共同的父类。因为在简单工厂模式中用于创建实例的方法是静态（static）方法，因此简单工厂模式又被称为静态工厂方法（Static Factory Method）模式，它属于类创建型模式。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211213133721.png)



## 二 代码实现

* 工厂方法

```go
package samplefactory

import "fmt"

// 简单工厂模式实现一个缓存库，支持set/get方法
// 模拟支持Redis/Memcache

type Cache interface {
	Set(key, value string)
	Get(key string)
}

type Redis struct {
	data map[string]string
}

func (r *Redis) Set(key, value string) {
	r.data[key] = value
}

func (r *Redis) Get(key string) {
	fmt.Println("redis ", r.data[key])
}

type Memcache struct {
	data map[string]string
}

func (m *Memcache) Set(key, value string) {
	m.data[key] = value
}

func (m *Memcache) Get(key string) {
	fmt.Println("memcache ", m.data[key])
}

type cacheType int

const (
	r cacheType = iota
	m
)

func NewCache(i cacheType) Cache {
	switch i {
	case r:
		return &Redis{data: map[string]string{}}
	case m:
		return &Memcache{data: map[string]string{}}
	}
	return nil
}

```



* 测试代码

```go
package samplefactory

import (
	"testing"
)

func TestNewCache(t *testing.T) {
	r := NewCache(0)
	r.Set("name", "xuel")
	r.Get("name")

	m := NewCache(1)
	m.Set("age", "22")
	m.Get("age")
}

```

## 三 应用场景

当在代码里看到switch的时候，就应该思考是否用简单工厂模式。

## 四 特点

### **4.1 优点**

1. 将客户端和创建产品实例解耦开来，使客户端只需要关注如何获取实例。
2. 符合单一职责。

### **4.2 缺点**

* 增加新翻译类时还是需要改动工厂类，没有符合开闭原则。

## 参考链接

* https://juejin.cn/post/6844903703447765005