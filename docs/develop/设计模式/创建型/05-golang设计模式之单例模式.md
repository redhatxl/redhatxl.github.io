# 5.单例模式

## 一 单例模式简介

> 单例模式，也叫单子模式，是一种常用的软件设计模式。在应用这个模式时，单例对象的类必须保证只有一个实例存在。许多时候整个系统只需要拥有一个的全局对象，这样有利于我们协调系统整体的行为。比如在某个服务器程序中，该服务器的配置信息存放在一个文件中，这些配置数据由一个单例对象统一读取，然后服务进程中的其他对象再通过这个单例对象获取这些配置信息。这种方式简化了在复杂环境下的配置管理。

单例模式要实现的效果就是，对于应用单例模式的类，整个程序中只存在一个实例化对象

go并不是一种面向对象的语言，所以我们使用结构体来替代

## 二 代码

### 2.1 开发代码

```go
package singleton

import (
    "fmt"
    "sync"
)

// 懒汉模式, 再需要的时候进行初始化, 相较于饿汉模式比较块
type Deploy struct {
    name string
    image string
}
var d *Deploy

func CreateDeploy() *Deploy  {
    if d == nil {
        d = new(Deploy)
    }
    return d
}

func DeployTest()  {
    dObj := CreateDeploy()
    dObj.name = "dname"
    dObj.image = "dimage"
    fmt.Println(dObj.name,dObj.image)

    d2 := CreateDeploy()
    fmt.Println("懒汉模式:",d2.name)
}

// 饿汉模式,初始化包的时候就使用，但是相对比较安全
type Service struct {
    name string
}

var s *Service

func init()  {
    s = new(Service)
    s.name = "sname"
}

func getService() *Service {
    return s
}

func ServiceTest()  {
    sObj := getService()
    fmt.Println("饿汉模式:",sObj.name)
}

// 单独例模式
type SinglePod struct {
    name ,image string
}

var pod1 *SinglePod
var once sync.Once

func GetPod() *SinglePod {
    once.Do(func() {
        pod1 = new(SinglePod)
        pod1.name = "podname"
        pod1.image = "podimage"
    })
    return pod1
}

func PodTest()  {
    fmt.Println("单例子模式")
    p1 := GetPod()
    fmt.Println(p1.name, p1.image)

    p2 := GetPod()
    fmt.Println(p2.name, p2.image)
}

```

### 2.2 测试

```go
package singleton

import "testing"

// 懒汉模式
func TestCreateDeploy(t *testing.T) {
    DeployTest()
}

// 饿汉模式
func TestServiceTest(t *testing.T) {
    ServiceTest()
}

// 单例模式
//
func TestPodTest(t *testing.T) {
   PodTest()
}
```

## 三 应用场景

- 系统只需要一个实例对象，如系统要求提供一个唯一的序列号生成器，或者需要考虑资源消耗太大而只允许创建一个对象。
- 客户调用类的单个实例只允许使用一个公共访问点，除了该公共访问点，不能通过其他途径访问该实例。
- 在一个系统中要求一个类只有一个实例时才应当使用单例模式。反过来，如果一个类可以有几个实例共存，就需要对单例模式进行改进，使之成为多例模式

## 四 特点

### 4.1 优点

*   你可以保证一个类只有一个实例。
*   你获得了一个指向该实例的全局访问节点。
*   仅在首次请求单例对象时对其进行初始化。

### 4.2 缺点

* 违反了_单一职责原则_。 该模式同时解决了两个问题。
*  单例模式可能掩盖不良设计， 比如程序各组件之间相互了解过多等。
*  该模式在多线程环境下需要进行特殊处理， 避免多个线程多次创建单例对象。
*  单例的客户端代码单元测试可能会比较困难， 因为许多测试框架以基于继承的方式创建模拟对象。 由于单例类的构造函数是私有的， 而且绝大部分语言无法重写静态方法， 所以你需要想出仔细考虑模拟单例的方法。 要么干脆不编写测试代码， 或者不使用单例模式。

## 参考链接

* https://refactoringguru.cn/design-patterns/singleton

