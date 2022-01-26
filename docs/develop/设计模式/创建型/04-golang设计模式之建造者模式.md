# 4.建造者(生成器)模式

## 一 建造者模式简介

造者模式(Builder Pattern)：将一个复杂对象的构建与它的表示分离，使得同样的构建过程可以创建不同的表示。

建造者模式是一步一步创建一个复杂的对象，它允许用户只通过指定复杂对象的类型和内容就可以构建它们，用户不需要知道内部的具体构建细节。建造者模式属于对象创建型模式。根据中文翻译的不同，建造者模式又可以称为生成器模式。

当一个方法有多个变量的时候，我们在调用该方法的时候可能会因为参数的顺序、个数错误，而造成调用错误或者不能达到我们预期的目的。针对这个问题，我们的建造设计模式可以完美的解决这个问题

## 二 代码

### 2.1 开发代码

```go
package builder

import "fmt"

type Deployment struct {
    Name string
    // 镜像信息
    Image, ContainerName string
    // 端口信息
    SvcPort, ContainerPort int
}

func (d *Deployment) String() string {
    return fmt.Sprintf("Name:%s,Image:%s,ContainerName:%s,SvcPort:%d,ContainerPort:%d",
        d.Name,d.Image,d.ContainerName,d.SvcPort,d.ContainerPort)
}

type DeploymentBuilder struct {
    deployment *Deployment
}

// 总体deployment 工厂函数
func NewDeploymentBuilder() *DeploymentBuilder  {
    return &DeploymentBuilder{deployment: &Deployment{}}
}

// Deployment 镜像建造者
type DeployImgBuilder struct {
    DeploymentBuilder
}

// Deployment 端口建造者
type DeployPortBuilder struct {
    DeploymentBuilder
}

// 镜像建造
func (d *DeploymentBuilder) imgs() *DeployImgBuilder {
    return &DeployImgBuilder{*d}
}

// 端口建造
func (d *DeploymentBuilder) ports() *DeployPortBuilder  {
    return &DeployPortBuilder{*d}
}

// 使用 DeployImgBuilder/DeployPortBuilder 创建一些行为/函数/方法，用于设置Deployment的属性
func (d *DeploymentBuilder) Name(name string) *DeploymentBuilder  {
    d.deployment.Name = name
    return d
}

// DeployImgBuilder 设置镜像名称
func (i *DeployImgBuilder) Image(n string) *DeployImgBuilder {
    i.deployment.Image = n
    return i
}

// DeployImgBuilder 设置容器名称
func (i *DeployImgBuilder) ContainerName(n string) *DeployImgBuilder  {
    i.deployment.ContainerName = n
    return i
}

// DeployPortBuilder 设置svc端口
func (p *DeployPortBuilder) SvcPort(o int) *DeployPortBuilder  {
    p.deployment.SvcPort = o
    return p
}

// DeployPortBuilder 设置容器端口
func (p *DeployPortBuilder) ContainerPort(o int) *DeployPortBuilder  {
    p.deployment.ContainerPort = o
    return p
}

// 最后使用DeploymentBuilder 来返回builder
func (b *DeploymentBuilder) build() *Deployment  {
    return b.deployment
}
```

### 2.2 测试

```go
package builder

import (
    "fmt"
    "testing"
)

func TestNewDeploymentBuilder(t *testing.T) {
    db := NewDeploymentBuilder()
    deploy := db.Name("deployname").
        imgs().
            ContainerName("cname").
            Image("cimage").
        ports().
            ContainerPort(80).
            SvcPort(30088).
        build()
    t.Logf(deploy.String())
    fmt.Println(deploy.String())
}

```

## 三 应用场景

- 需要生成的产品对象有复杂的内部结构，这些产品对象通常包含多个成员属性。
- 需要生成的产品对象的属性相互依赖，需要指定其生成顺序。
- 对象的创建过程独立于创建该对象的类。在建造者模式中引入了指挥者类，将创建过程封装在指挥者类中，而不在建造者类中。
- 隔离复杂对象的创建和使用，并使得相同的创建过程可以创建不同的产品。

## 四 特点

### 4.1 优点

*  生成不同形式的产品时， 你可以复用相同的制造代码。
* 你可以分步创建对象， 暂缓创建步骤或递归运行创建步骤。
* 单一职责原则。 你可以将复杂构造代码从产品的业务逻辑中分离出来。

### 4.2 缺点

* 由于该模式需要新增多个类， 因此代码整体复杂程度会有所增加。

## 参考链接

* https://juejin.cn/post/6844903704189992967
* https://github.com/silsuer/golang-design-patterns
* https://refactoringguru.cn/design-patterns/factory-method
* https://www.bookstack.cn/read/design-patterns/98fde31c319ecbaa.md