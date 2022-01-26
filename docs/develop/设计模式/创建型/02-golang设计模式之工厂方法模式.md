# 2.工厂方法模式

## 一 工厂方法模式简介

工厂方法模式（英语：Factory method pattern）是一种实现了“工厂”概念的面向对象设计模式。就像其他创建型模式一样，它也是处理在不指定对象具体类型的情况下创建对象的问题。工厂方法模式的实质是“定义一个创建对象的接口，但让实现这个接口的类来决定实例化哪个类。工厂方法让类的实例化推迟到子类中进行。”

上面是 维基百科 中对工厂方法的定义，在上一篇 [golang设计模式之简单工厂模式](https://juejin.cn/post/6844903703447765005) 中我们介绍过，唯一的一个工厂控制着 所有产品的实例化，而 `工厂方法` 中包括一个工厂接口，我们可以动态的实现多种工厂，达到扩展的目的

- 简单工厂需要:
  1. 工厂结构体
  2. 产品接口
  3. 产品结构体
- 工厂方法需要:
  1. 工厂接口
  2. 工厂结构体
  3. 产品接口
  4. 产品结构体

在 `简单工厂` 中，依赖于唯一的工厂对象，如果我们需要实例化一个产品，那么就要向工厂中传入一个参数获取对应对象，如果要增加一种产品，就要在工厂中修改创建产品的函数，耦合性过高 ，而在 `工厂方法` 中，依赖工厂接口，我们可以通过实现工厂接口，创建多种工厂，将对象创建由一个对象负责所有具体类的实例化，变成由一群子类来负责对具体类的实例化，将过程解耦。


## 二 代码

### 2.1 开发代码

```go
package factorymethod

// 需求：工厂方法模式模拟实现一个缓存库，支持get/set方法，同时支持Redis/Memcache

// 定义缓存库
type Cache interface {
	Set(key, value string)
	Get(key string) string
}

// 定义Redis对象
type Redis struct {
	data map[string]string
}

func NewRedis() *Redis {
	return &Redis{
		data: make(map[string]string),
	}
}

func (r *Redis) Set(key, value string) {
	r.data[key] = value
}

func (r *Redis) Get(key string) string {
	return r.data[key]
}

// 定义Memcache对象
type Memcache struct {
	data map[string]string
}

func NewMemcache() *Memcache {
	return &Memcache{
		data: make(map[string]string),
	}
}

func (m *Memcache) Set(key, value string) {
	m.data[key] = value
}

func (m *Memcache) Get(key string) string {
	return m.data[key]
}

type CacheFactory interface {
	Create() Cache
}

type RedisFactory struct {
}

func (rf RedisFactory) Create() Cache {
	return NewRedis()
}

type MemcacheFactory struct {
}

func (mf MemcacheFactory) Create() Cache {
	return NewMemcache()
}

```



### 2.2 测试

```go
package factorymethod

import (
	"fmt"
	"testing"
)

func TestMemcache_Create(t *testing.T) {
	var memCacheFacotory CacheFactory

	memCacheFacotory = MemcacheFactory{}
	memcache := memCacheFacotory.Create()
	memcache.Set("name", "xuel")
	rest := memcache.Get("name")
	fmt.Println(rest)

}
func TestRedisFactory_Create(t *testing.T) {
	var redisFactory CacheFactory
	redisFactory = RedisFactory{}
	redis := redisFactory.Create()
	redis.Set("age", "22")
	rest := redis.Get("age")
	fmt.Println(rest)
}

```



## 三 应用场景

简单工厂模式和工厂方法模式看起来很相似，本质区别就在于，如果在包子店中直接创建包子产品，是依赖具体包子店的，扩展性、弹性、可维护性都较差，而如果将实例化的代码抽象出来，不再依赖具体包子店，而是依赖于抽象的包子接口，使对象的实现从使用中解耦，这样就拥有很强的扩展性了，也可以称为 『依赖倒置原则』

## 四 特点

### 4.1 优点:

1. 符合“开闭”原则，具有很强的的扩展性、弹性和可维护性。修改时只需要添加对应的工厂类即可
2. 使用了依赖倒置原则，依赖抽象而不是具体，使用（客户）和实现（具体类）松耦合
3. 客户只需要知道所需产品的具体工厂，而无须知道具体工厂的创建产品的过程，甚至不需要知道具体产品的类名。

### 4.2 缺点:

1. 每增加一个产品时，都需要一个具体类和一个具体创建者，使得类的个数成倍增加，导致系统类数目过多，复杂性增加
2. 对简单工厂，增加功能修改的是工厂类；对工厂方法，增加功能修改的是产品类。

## 参考链接

- https://juejin.cn/post/6844903704189992967
- https://github.com/silsuer/golang-design-patterns
- https://refactoringguru.cn/design-patterns/factory-method