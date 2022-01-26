# 1.装饰器模式

## 一 装饰器模式简介

**装饰模式**是一种结构型设计模式， 允许你通过将对象放入包含行为的特殊封装对象中来为原对象绑定新的行为。

*封装器*是装饰模式的别称， 这个称谓明确地表达了该模式的主要思想。  “封装器” 是一个能与其他 “目标” 对象连接的对象。 封装器包含与目标对象相同的一系列方法， 它会将所有接收到的请求委派给目标对象。 但是， 封装器可以在将请求委派给目标前后对其进行处理， 所以可能会改变最终结果。


## 二 代码

### 2.1 开发代码

```go
package decorator

import "fmt"

// 装饰器模式

// 定义Going 的接口,包含 run 方法
type Going interface {
	Run()
}

// 定义 Car 对象, 实现run方法
type Car struct {
	name string
}

func (d *Car) Run() {
	fmt.Println("decorator do function")
}

// 为了给Car 方法增加额外功能，例如fly
// 该装饰接口包含被装饰的接口，并且新增额外方法
type AdvantageCar interface {
	Going
	Fly
}

// 定义新增方法接口
type Fly interface {	
	Flying()
}

// 定义装饰后的对象, 其属性包含被装饰的接口
type ACar struct {
	Going
}

// 新对象实现额外新增功能接口方法
func (a *ACar) Flying() {
	fmt.Println("flaying ")
}

func (a *ACar) Run() {
	// 先实现之前被装饰前的方法
	a.Going.Run()
	// 在实现额外新增功能方法
	a.Flying()
}

```



### 2.2 测试

```go
package decorator

import (
	"fmt"
	"testing"
)

func TestCar_Run(t *testing.T) {
	c := Car{name: "BYD"}
	c.Run()
}

func TestACar_Run(t *testing.T) {
	fmt.Println("acar test......")
	ac := ACar{}

	g := &Car{name: "BMW"}
	ac.Going = g

	ac.Run()

}


/*
decorator do function
acar test......
decorator do function
flaying 
PASS
ok      github.com/kaliarch/mycode/design-patterns/structural/decorator 0.015s
*/
```



## 三 应用场景

*  如果你希望在无需修改代码的情况下即可使用对象，且希望在运行时为对象新增额外的行为，可以使用装饰模式， 装饰能将业务逻辑组织为层次结构， 你可为各层创建一个装饰， 在运行时将各种不同逻辑组合成对象。 由于这些对象都遵循通用接口， 客户端代码能以相同的方式使用这些对象。

*  如果用继承来扩展对象行为的方案难以实现或者根本不可行， 你可以使用该模式， 许多编程语言使用 `final`最终关键字来限制对某个类的进一步扩展。 复用最终类已有行为的唯一方法是使用装饰模式： 用封装器对其进行封装。

  

## 四 特点

### 4.1 优点: 

- 你无需创建新子类即可扩展对象的行为。
-  你可以在运行时添加或删除对象的功能。
-  你可以用多个装饰封装对象来组合几种行为。
-  *单一职责原则*。 你可以将实现了许多不同行为的一个大类拆分为多个较小的类。

### 4.2 缺点: 

-  在封装器栈中删除特定封装器比较困难。
-  实现行为不受装饰栈顺序影响的装饰比较困难。
-  各层的初始化配置代码看上去可能会很糟糕。

## 参考链接

* https://github.com/silsuer/golang-design-patterns
* https://refactoringguru.cn/design-patterns/decorator