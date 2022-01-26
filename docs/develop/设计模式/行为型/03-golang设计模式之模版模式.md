# 2.模版模式

## 一 模版模式简介

模版方法模式使用继承机制，把通用步骤和通用方法放到父类中，把具体实现延迟到子类中实现。使得实现符合开闭原则。

如实例代码中通用步骤在父类中实现（`准备`、`下载`、`保存`、`收尾`）下载和保存的具体实现留到子类中，并且提供 `保存`方法的默认实现。

因为Golang不提供继承机制，需要使用匿名组合模拟实现继承。

此处需要注意：因为父类需要调用子类方法，所以子类需要匿名组合父类的同时，父类需要持有子类的引用。

在面向对象程序设计过程中，程序员常常会遇到这种情况：设计一个系统时知道了算法所需的关键步骤，而且确定了这些步骤的执行顺序，但某些步骤的具体实现还未知，或者说某些步骤的实现与具体的环境相关。

例如，去银行办理业务一般要经过以下4个流程：取号、排队、办理具体业务、对银行工作人员进行评分等，其中取号、排队和对银行工作人员进行评分的业务对每个客户是一样的，可以在父类中实现，但是办理具体业务却因人而异，它可能是存款、取款或者转账等，可以延迟到子类中实现。

这样的例子在生活中还有很多，例如，一个人每天会起床、吃饭、做事、睡觉等，其中“做事”的内容每天可能不同。我们把这些规定了流程或格式的实例定义成模板，允许使用者根据自己的需求去更新它，例如，简历模板、论文模板、Word 中模板文件等。

以下介绍的模板方法模式将解决以上类似的问题。

## 二 代码

### 2.1 开发代码

```go
package templatemethod

import "fmt"

type iOtp interface {
	genRandomOTP(int) string
	saveOTPCache(string)
	getMessage(string) string
	sendNotification(string) error
	publishMetric()
}

// 模版模式基于继承，在类层面进行
type Otp struct {
	iOtp iOtp
}

func (i Otp) genAndSendMsg(optLength int) error {
	opt := i.iOtp.genRandomOTP(optLength)
	i.iOtp.saveOTPCache(opt)
	msg := i.iOtp.getMessage(opt)
	if err := i.iOtp.sendNotification(msg); err != nil {
		return err
	}
	i.iOtp.publishMetric()
	return nil
}

// 模版模式基于继承，在类层面进行
type Email struct {
	Otp
}

// 继承重写父类方法
func (e *Email) genRandomOTP(int) string {
	fmt.Println("email gen random otp")
	return fmt.Sprintf("%s, gen random otp", e)
}

func (e *Email) saveOTPCache(otp string) {
	fmt.Println("email save otp", otp)
}

func (e *Email) getMessage(opt string) string {
	return fmt.Sprintf("%s,msg", opt)
}
func (e *Email) sendNotification(msg string) error {
	fmt.Println("e send notification", msg)
	return nil
}
func (e *Email) publishMetric() {
	fmt.Println("email publish")
}

type SMS struct {
	Otp
}

// 继承重写父类方法
func (e *SMS) genRandomOTP(int) string {
	fmt.Println("SMS gen random otp")
	return fmt.Sprintf("%s, gen random otp", e)
}

func (e *SMS) saveOTPCache(otp string) {
	fmt.Println("SMS save otp", otp)
}

func (e *SMS) getMessage(opt string) string {
	return fmt.Sprintf("%s,msg", opt)
}
func (e *SMS) sendNotification(msg string) error {
	fmt.Println("SMS send notification", msg)
	return nil
}
func (e *SMS) publishMetric() {
	fmt.Println("SMS publish")
}

```

### 2.2 测试

```go
package templatemethod

import (
	"fmt"
	"testing"
)

const (
	emailLength = 10
	SmsLength   = 20
)

func TestOtp(t *testing.T) {

	emopt := &Email{}

	opt := Otp{
		iOtp: emopt,
	}

	_ = opt.genAndSendMsg(emailLength)

	fmt.Println("")

	smsopt := &SMS{}

	opt = Otp{
		iOtp: smsopt,
	}
	_ = opt.genAndSendMsg(SmsLength)
}

```

### 2.3 运行结果

```shell
email gen random otp
email save otp &{{<nil>}}, gen random otp
e send notification &{{<nil>}}, gen random otp,msg
email publish

SMS gen random otp
SMS save otp &{{<nil>}}, gen random otp
SMS send notification &{{<nil>}}, gen random otp,msg
SMS publish
PASS

```

## 三 应用场景

* 当一个对象状态的改变需要改变其他对象， 或实际对象是事先未知的或动态变化的时， 可使用观察者模式。
* 当应用中的一些对象必须观察其他对象时， 可使用该模式。 但仅能在有限时间内或特定情况下使用。

## 四 特点

### 4.1 优点

-  它封装了不变部分，扩展可变部分。它把认为是不变部分的算法封装到父类中实现，而把可变部分算法由子类继承实现，便于子类继续扩展。
-  它在父类中提取了公共的部分代码，便于代码复用。
-  部分方法是由子类实现的，因此子类可以通过扩展方式增加相应的功能，符合开闭原则。

### 4.2 缺点

-  对每个不同的实现都需要定义一个子类，这会导致类的个数增加，系统更加庞大，设计也更加抽象，间接地增加了系统实现的复杂度。
-  父类中的抽象方法由子类实现，子类执行的结果会影响父类的结果，这导致一种反向的控制结构，它提高了代码阅读的难度。
-  由于继承关系自身的缺点，如果父类添加新的抽象方法，则所有子类都要改一遍。

## 参考链接

* https://refactoringguru.cn/design-patterns/observer