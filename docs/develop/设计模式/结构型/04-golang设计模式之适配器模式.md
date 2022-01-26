# 4.适配器模式

## 一 适配器模式简介

适配器模式用于转换一种接口适配另一种接口。

实际使用中Adaptee一般为接口，并且使用工厂函数生成实例。

在Adapter中匿名组合Adaptee接口，所以Adapter类也拥有SpecificRequest实例方法，又因为Go语言中非入侵式接口特征，其实Adapter也适配Adaptee接口。

在现实生活中，经常出现两个对象因接口不兼容而不能在一起工作的实例，这时需要第三者进行适配。例如，讲中文的人同讲英文的人对话时需要一个翻译，用直流电的笔记本电脑接交流电源时需要一个电源适配器，用计算机访问照相机的 SD 内存卡时需要一个读卡器等。

在软件设计中也可能出现：需要开发的具有某种业务功能的组件在现有的组件库中已经存在，但它们与当前系统的接口规范不兼容，如果重新开发这些组件成本又很高，这时用适配器模式能很好地解决这些问题。

![](https://kaliarch-bucket-1251990360.cos.ap-beijing.myqcloud.com/blog_img/20211213131826.png)

## 二 代码

### 2.1 开发代码

```go
package adapter

import "fmt"

var ExpectContent string = "AdapteeImple"

// 被适配接口
type Adaptee interface {
	SpecificRequest() string
}

// 被适配对象
type AdapteeImpl struct {
}

// 被适配对象工厂函数
func NewAdaptee() *AdapteeImpl {
	return &AdapteeImpl{}
}

func (a *AdapteeImpl) SpecificRequest() string {
	return fmt.Sprintf("%s", ExpectContent)
}

// 目标接口
type Target interface {
	Request() string
}

// 适配器对象
type Adapter struct {
	Adaptee
}

func NewAdapter(adaptee Adaptee) *Adapter {
	return &Adapter{
		Adaptee: adaptee,
	}
}
func (ad *Adapter) Request() string {
	return ad.SpecificRequest()
}

```



### 2.2 测试

```go
package adapter

import (
	"testing"
)

func TestNewAdapter(t *testing.T) {
	adaptee := NewAdaptee()

	adapter := NewAdapter(adaptee)

	res := adapter.SpecificRequest()

	if res != ExpectContent {
		t.Fatalf("expect: %s, actual: %s", ExpectContent, res)
	}

}

```

### 2.3 运行输出

```go
PASS
ok      github.com/kaliarch/mycode/design-patterns/structural/adapter   0.637s
```

## 三 应用场景

- 以前开发的系统存在满足新系统功能需求的类，但其接口同新系统的接口不一致。
- 使用第三方提供的组件，但组件接口定义和自己要求的接口定义不同。

## 四 优缺点

### 4.1 优点

- 客户端通过适配器可以透明地调用目标接口。
- 复用了现存的类，程序员不需要修改原有代码而重用现有的适配者类。
- 将目标类和适配者类解耦，解决了目标类和适配者类接口不一致的问题。
- 在很多业务场景中符合开闭原则。

### 4.2 缺点

- 适配器编写过程需要结合业务场景全面考虑，可能会增加系统的复杂性。
- 增加代码阅读难度，降低代码可读性，过多使用适配器会使系统代码变得凌乱。

# 参考链接

* https://github.com/ssbandjl/golang-design-pattern/tree/master/02_adapter