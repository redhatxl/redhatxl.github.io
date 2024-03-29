# 1.策略模式

## 一 策略模式简介

**策略模式**是一种行为设计模式， 它能让你定义一系列算法， 并将每种算法分别放入独立的类中， 以使算法的对象能够相互替换。

策略模式建议找出负责用许多不同方式完成特定任务的类， 然后将其中的算法抽取到一组被称为*策略*的独立类中。

名为*上下文*的原始类必须包含一个成员变量来存储对于每种策略的引用。 上下文并不执行任务， 而是将工作委派给已连接的策略对象。

上下文不负责选择符合任务需要的算法——客户端会将所需策略传递给上下文。 实际上， 上下文并不十分了解策略， 它会通过同样的通用接口与所有策略进行交互， 而该接口只需暴露一个方法来触发所选策略中封装的算法即可。

因此， 上下文可独立于具体策略。 这样你就可在不修改上下文代码或其他策略的情况下添加新算法或修改已有算法了。



在导游应用中， 每个路线规划算法都可被抽取到只有一个 `build­Route`生成路线方法的独立类中。 该方法接收起点和终点作为参数， 并返回路线中途点的集合。

即使传递给每个路径规划类的参数一模一样， 其所创建的路线也可能完全不同。 主要导游类的主要工作是在地图上渲染一系列中途点， 不会在意如何选择算法。 该类中还有一个用于切换当前路径规划策略的方法， 因此客户端 （例如用户界面中的按钮） 可用其他策略替换当前选择的路径规划行为。


## 二 代码

### 2.1 开发代码

```go
package stragtegy

import "fmt"

// 策略模式

// Person 定义 Person 结构体对象,Going 接口属性
type Person struct {
	Going
}

// Going 定义 Goging 方法,内包含 Run方法
type Going interface {
	Run()
}

// 定义Going 的不同策略

// 1. 定义cat对象

type Car struct {
	name string
}

func (c *Car) Run() {
	fmt.Println(c.name, "Running")
}

// 2. 定义train
type Train struct {
	name string
}

func (t *Train) Run() {
	fmt.Println(t.name, "Running")
}

```

### 2.2 测试

```go
package stragtegy

import (
	"fmt"
	"testing"
)

func TestCar_Run(t *testing.T) {
	c := &Car{name: "BYD"}
	c.Run()
}

func TestTrain_Run(t *testing.T) {
	c := &Train{name: "tianqi01"}
	c.Run()
}

// 单元测试
// 对定义的对象中属性接口进行不同的策略，以达到不同的效果
func TestPerson_Run(t *testing.T) {
	fmt.Println("Person Test......,car")
	p := Person{}
	c := &Car{name: "BMW"}
	p.Going = c
	p.Run()

	fmt.Println("Person Test......,train")
	tr := &Train{name: "tianqi01"}

	p.Going = tr
	p.Run()

}

/*
BYD Running
tianqi01 Running
Person Test......,car
BMW Running
Person Test......,train
tianqi01 Running

*/
```

## 三 应用场景

* 当你想使用对象中各种不同的算法变体， 并希望能在运行时切换算法时， 可使用策略模式。策略模式让你能够将对象关联至可以不同方式执行特定子任务的不同子对象， 从而以间接方式在运行时更改对象行为。
* 当你有许多仅在执行某些行为时略有不同的相似类时， 可使用策略模式。 策略模式让你能将不同行为抽取到一个独立类层次结构中， 并将原始类组合成同一个， 从而减少重复代码。
* 如果算法在上下文的逻辑中不是特别重要， 使用该模式能将类的业务逻辑与其算法实现细节隔离开来。策略模式让你能将各种算法的代码、 内部数据和依赖关系与其他代码隔离开来。 不同客户端可通过一个简单接口执行算法， 并能在运行时进行切换。
* 当类中使用了复杂条件运算符以在同一算法的不同变体中切换时， 可使用该模式。策略模式将所有继承自同样接口的算法抽取到独立类中， 因此不再需要条件语句。 原始对象并不实现所有算法的变体， 而是将执行工作委派给其中的一个独立算法对象。

## 四 特点

### 4.1 优点

-  你可以在运行时切换对象内的算法。
- 你可以将算法的实现和使用算法的代码隔离开来。
- 你可以使用组合来代替继承。
- *开闭原则*。 你无需对上下文进行修改就能够引入新的策略。

### 4.2 缺点

- 如果你的算法极少发生改变， 那么没有任何理由引入新的类和接口。 使用该模式只会让程序过于复杂。
- 客户端必须知晓策略间的不同——它需要选择合适的策略。
-  许多现代编程语言支持函数类型功能， 允许你在一组匿名函数中实现不同版本的算法。 这样， 你使用这些函数的方式就和使用策略对象时完全相同， 无需借助额外的类和接口来保持代码简洁。

## 参考链接

* https://github.com/silsuer/golang-design-patterns
* https://refactoringguru.cn/design-patterns/strategy