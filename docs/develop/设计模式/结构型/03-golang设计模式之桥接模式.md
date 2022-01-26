# 3.桥接模式

## 一 桥接模式简介

[桥接（Bridge）模式](https://www.zhihu.com/search?q=桥接（Bridge）模式&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A389870230})的定义：将抽象与实现分离，使它们可以独立变化。它是用[组合关系](https://www.zhihu.com/search?q=组合关系&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A389870230})代替继承关系来实现，从而降低了抽象和实现这两个可变维度的耦合度。

定义来看，桥接模式遵循了[里氏替换原则](https://www.zhihu.com/search?q=里氏替换原则&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A389870230})和依赖倒置原则，最终实现了开闭原则，对修改关闭，对扩展开放。

## 二 代码

### 2.1 开发代码

```go
package bridge

import "fmt"

// 桥接模式分离抽象部分和实现部分。使得两部分独立扩展。

// 定义两种实现类型抽象定义，例如定义电脑接口和定义打印机接口

// 一个类型内部包含方法定义，

// 一个类型内部实现自身业务逻辑

// 客户端实例话对应对象，进行对象任意组合

// 电脑接口
type computer interface {
    print()
    setPrinter(printer)
}

// 打印机接口
type printer interface {
    printFile()
}

type epson struct {
}

func (e *epson) printFile()  {
    fmt.Println("printing by epson printer")
}

type hp struct {
}

func (p *hp) printFile()  {
    fmt.Println("printing by hp printer")
}



type Win struct {
    printer printer
}

func (w *Win) print()  {
    fmt.Println("print win")
    w.printer.printFile()
}

func (w *Win) setPrinter(p printer)  {
    w.printer = p
}

type Mac struct {
    printer printer
}

func (m *Mac) print()  {
    fmt.Println("print mac")
    m.printer.printFile()
}

func (m *Mac) setPrinter(p printer)  {
    m.printer = p
}
```



### 2.2 测试

```go
package bridge

import (
    "fmt"
    "testing"
)

func TestBridge(t *testing.T)  {
    hpPrinter := &hp{}

    epsonPrinter := &epson{}

    m := Mac{}

    m.setPrinter(hpPrinter)
    m.print()
    m.setPrinter(epsonPrinter)
    m.print()

    fmt.Println("------")

    w := Win{}
    w.setPrinter(epsonPrinter)
    w.print()
    w.setPrinter(hpPrinter)
    w.print()
}
```

### 2.3 运行输出

```go
=== RUN   TestBridge
print mac
printing by hp printer
print mac
printing by epson printer
------
print win
printing by epson printer
print win
printing by hp printer
--- PASS: TestBridge (0.00s)
PASS
```

## 三 应用场景

* 如果你想要拆分或重组一个具有多重功能的庞杂类（例如能与多个数据库服务器进行交互的类），可以使用桥接模式。

* 如果你希望在几个独立维度上扩展一个类， 可使用该模式。
*  如果你需要在运行时切换不同实现方法， 可使用桥接模式。

## 四 特点

### 4.1 优点

- 抽象与实现分离，扩展能力强
- 符合开闭原则
- 符合[合成复用原则](https://www.zhihu.com/search?q=合成复用原则&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"article"%2C"sourceId"%3A389870230})
- 其实现细节对客户透明

### 4.2 缺点

- 由于聚合关系建立在抽象层，要求开发者针对抽象化进行设计与编程，能正确地识别出系统中两个独立变化的维度，这增加了系统的理解与设计难度。

## 参考链接

* https://www.bookstack.cn/read/Design-Pattern/lesson22-README.md
* https://refactoringguru.cn/design-patterns/bridge