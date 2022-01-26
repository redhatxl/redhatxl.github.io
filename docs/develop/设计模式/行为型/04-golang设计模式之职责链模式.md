# 4.职责链模式

## 一 职责链模式简介

职责链模式用于分离不同职责，并且动态组合相关职责。

Golang实现职责链模式时候，因为没有继承的支持，使用链对象包涵职责的方式，即：

- 链对象包含当前职责对象以及下一个职责链。
- 职责对象提供接口表示是否能处理对应请求。
- 职责对象提供处理函数处理相关职责。

同时可在职责链类中实现职责接口相关函数，使职责链对象可以当做一般职责对象是用。

## 二 代码

### 2.1 开发代码

```go
package responsibilitychain

import "fmt"

type patient struct {
	name              string
	registrationDone  bool
	doctorCheckUpDone bool
	medicineDone      bool
	paymentDone       bool
}

type department interface {
	// 这个部门进行处理
	execute(*patient)
	// 设置下一个部门
	setNext(department)
}

type reception struct {
	next department
}

func (r *reception) execute(p *patient) {
	if p.registrationDone {
		fmt.Println("registration done")
		r.next.execute(p)
		return
	}
	fmt.Println("registration doing")
	p.registrationDone = true
	r.next.execute(p)
}

func (r *reception) setNext(next department) {
	r.next = next
}

type doctor struct {
	next department
}

func (d *doctor) execute(p *patient) {
	if p.doctorCheckUpDone {
		fmt.Println("docker done")
		d.next.execute(p)
		return
	}
	fmt.Println("docker doing")
	p.doctorCheckUpDone = true
	d.next.execute(p)
}

func (d *doctor) setNext(next department) {
	d.next = next
}

type medical struct {
	next department
}

func (m *medical) execute(p *patient) {
	if p.medicineDone {
		fmt.Println("medical done")
		m.next.execute(p)
		return
	}
	fmt.Println("medical doing")
	p.medicineDone = true
	m.next.execute(p)
}

func (m *medical) setNext(next department) {
	m.next = next
}

type cashier struct {
	next department
}

func (c *cashier) setNext(next department) {
	c.next = next
}

func (c *cashier) execute(p *patient) {
	if p.paymentDone {
		fmt.Println("cashier done")
		fmt.Println("patient end")
		return
	}
	fmt.Println("cashier doing")
	fmt.Println("patient end")
}

func main() {
	p := &patient{name: "xiaoming"}

	c := &cashier{}

	m := &medical{}
	m.setNext(c)

	d := &doctor{}
	d.setNext(m)

	r := &reception{}
	r.setNext(d)

	r.execute(p)
}
```

### 2.2 测试

```go
package responsibilitychain

import "testing"

func TestPatient(t *testing.T) {
	p := &patient{name: "xiaoming"}

	c := &cashier{}

	m := &medical{}
	m.setNext(c)

	d := &doctor{}
	d.setNext(m)

	r := &reception{}
	r.setNext(d)

	r.execute(p)
}

```

### 2.3 输出

```shell
registration doing
docker doing
medical doing
cashier doing
patient end
```



## 三 应用场景

1. 多个对象可以处理一个请求，但具体由哪个对象处理该请求在运行时自动确定。
2. 可动态指定一组对象处理请求，或添加新的处理者。
3. 需要在不明确指定请求处理者的情况下，向多个处理者中的一个提交请求。

## 四 特点

### 4.1 优点

-  降低了对象之间的耦合度。该模式使得一个对象无须知道到底是哪一个对象处理其请求以及链的结构，发送者和接收者也无须拥有对方的明确信息。
-  增强了系统的可扩展性。可以根据需要增加新的请求处理类，满足开闭原则。
-  增强了给对象指派职责的灵活性。当工作流程发生变化，可以动态地改变链内的成员或者调动它们的次序，也可动态地新增或者删除责任。
-  责任链简化了对象之间的连接。每个对象只需保持一个指向其后继者的引用，不需保持其他所有处理者的引用，这避免了使用众多的 if 或者 if···else 语句。
-  责任分担。每个类只需要处理自己该处理的工作，不该处理的传递给下一个对象完成，明确各类的责任范围，符合类的单一职责原则。

### 4.2 缺点

-  不能保证每个请求一定被处理。由于一个请求没有明确的接收者，所以不能保证它一定会被处理，该请求可能一直传到链的末端都得不到处理。
-  对比较长的职责链，请求的处理可能涉及多个处理对象，系统性能将受到一定影响。
-  职责链建立的合理性要靠客户端来保证，增加了客户端的复杂性，可能会由于职责链的错误设置而导致系统出错，如可能会造成循环调用。

## 参考链接

* https://refactoringguru.cn/design-patterns/chain-of-responsibility
* https://github.com/ssbandjl/golang-design-pattern/tree/master/21_chain_of_responsibility

