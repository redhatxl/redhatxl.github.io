# 2.观察者模式

## 一 观察者模式简介

**观察者模式**是一种行为设计模式， 允许你定义一种订阅机制， 可在对象事件发生时通知多个 “观察” 该对象的其他对象。

多个对象同时观察一个对象，当这个被观察的对象发生变化的时候，这些对象都会得到通知，可以做一些操作...

拥有一些值得关注的状态的对象通常被称为*目标*， 由于它要将自身的状态改变通知给其他对象， 我们也将其称为*发布者* （publisher）。 所有希望关注发布者状态变化的其他对象被称为*订阅者* （subscribers）。

观察者模式建议你为发布者类添加订阅机制， 让每个对象都能订阅或取消订阅发布者事件流。 不要害怕！ 这并不像听上去那么复杂。 实际上， 该机制包括 1） 一个用于存储订阅者对象引用的列表成员变量； 2） 几个用于添加或删除该列表中订阅者的公有方法。

## 二 代码

### 2.1 开发代码

```go
package observer

import "fmt"


// 定义观察者接口
type observer interface {
    // 更新数据,发送通知
    update(string)

    // 获取具体数据对象id
    getID() string
}


// 具体观察者
type customer struct {
    id string
}

// 更新，发送email給用户
func (c *customer) update(itemName string)  {
    fmt.Printf("Sending email to customer %s for item %s\n", c.id,itemName)
}

// 获取用户id
func (c *customer) getID() string  {
    return c.id
}

// 定义被观察者, 也就是主题
type subject interface {
    register(Observer observer)
    deregister(Observer observer)
    notifyAll()
}

// 定义具体发布者被观察者主体
type item struct {
    observerList []observer
    name string
    isStock bool
}

func newItem(name string) *item {
    return &item{
        name: name,
    }
}

func (i *item) updateAvailability()  {
    fmt.Printf("Item %s is now in stock\n",i.name)
    i.isStock = true
    i.notifyAll()
}

// 注册到被观测者
func (i *item) register(o observer)  {
    i.observerList = append(i.observerList, o)
}

// 注销
func (i *item) deregister(o observer)  {
    i.observerList = removeFromslice(i.observerList, o)
}

// 通知
func (i *item) notifyAll()  {
    for _, observer := range i.observerList {
        observer.update(i.name)
    }
}

// 将一个元素从slice中移除
func removeFromslice(observerList []observer, observerToRemove observer) []observer  {
    observerListLength := len(observerList)
    for i,observer := range observerList {
        if observerToRemove.getID() == observer.getID() {
            observerList[observerListLength-1], observerList[i] = observerList[i],observerList[observerListLength-1]
            return observerList[:observerListLength-1]
        }
    }
    return observerList
}
```

### 2.2 测试

```go
package observer

import (
    "testing"
)
func TestObserver(t *testing.T)  {
    shirtItem := newItem("nike")

    observerOne := &customer{id: "xuel"}
    observertwo := &customer{id: "wangj"}

    shirtItem.register(observerOne)
    shirtItem.register(observertwo)


    shirtItem.updateAvailability()

    shirtItem.deregister(observertwo)

    shirtItem.updateAvailability()
}
```

### 2.3 运行结果

```shell
=== RUN   TestObserver
Item nike is now in stock
Sending email to customer xuel for item nike
Sending email to customer wangj for item nike
Item nike is now in stock
Sending email to customer xuel for item nike
--- PASS: TestObserver (0.00s)
PASS
```

## 三 应用场景

当一个对象状态的改变需要改变其他对象， 或实际对象是事先未知的或动态变化的时， 可使用观察者模式。

当你使用图形用户界面类时通常会遇到一个问题。 比如， 你创建了自定义按钮类并允许客户端在按钮中注入自定义代码， 这样当用户按下按钮时就会触发这些代码。

观察者模式允许任何实现了订阅者接口的对象订阅发布者对象的事件通知。 你可在按钮中添加订阅机制， 允许客户端通过自定义订阅类注入自定义代码。

 当应用中的一些对象必须观察其他对象时， 可使用该模式。 但仅能在有限时间内或特定情况下使用。

 订阅列表是动态的， 因此订阅者可随时加入或离开该列表。

## 四 特点

### 4.1 优点

-  开闭原则。 你无需修改发布者代码就能引入新的订阅者类 （如果是发布者接口则可轻松引入发布者类）。
-  你可以在运行时建立对象之间的联系。

### 4.2 缺点

-  订阅者的通知顺序是随机的。

# 参考链接

* https://refactoringguru.cn/design-patterns/observer