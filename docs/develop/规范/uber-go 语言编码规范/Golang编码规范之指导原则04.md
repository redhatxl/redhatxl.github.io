## Golang编码规范之指导原则04

## 一 不要使用 panic

在生产环境中运行的代码必须避免出现 panic。panic 是 [级联失败](https://en.wikipedia.org/wiki/Cascading_failure) 的主要根源 。如果发生错误，该函数必须返回错误，并允许调用方决定如何处理它。

* Bad

```go
func run(args []string) {
  if len(args) == 0 {
    panic("an argument is required")
  }
  // ...
}

func main() {
  run(os.Args[1:])
}
```

* Good

```go
func run(args []string) error {
  if len(args) == 0 {
    return errors.New("an argument is required")
  }
  // ...
  return nil
}

func main() {
  if err := run(os.Args[1:]); err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)
  }
}
```

panic/recover 不是错误处理策略。仅当发生不可恢复的事情（例如：nil 引用）时，程序才必须 panic。程序初始化是一个例外：程序启动时应使程序中止的不良情况可能会引起 panic。

```
var _statusTemplate = template.Must(template.New("name").Parse("_statusHTML"))
```

即使在测试代码中，也优先使用`t.Fatal`或者`t.FailNow`而不是 panic 来确保失败被标记。

* Bad

```go
// func TestFoo(t *testing.T)

f, err := ioutil.TempFile("", "test")
if err != nil {
  panic("failed to set up test")
}
```

* Good

```go
// func TestFoo(t *testing.T)

f, err := ioutil.TempFile("", "test")
if err != nil {
  t.Fatal("failed to set up test")
}
```

## 二 使用使用 atomic

使用 [sync/atomic](https://golang.org/pkg/sync/atomic/) 包的原子操作对原始类型 (`int32`, `int64`等）进行操作，因为很容易忘记使用原子操作来读取或修改变量。

[go.uber.org/atomic](https://godoc.org/go.uber.org/atomic) 通过隐藏基础类型为这些操作增加了类型安全性。此外，它包括一个方便的`atomic.Bool`类型。

- Bad

```go
type foo struct {
  running int32  // atomic
}

func (f* foo) start() {
  if atomic.SwapInt32(&f.running, 1) == 1 {
     // already running…
     return
  }
  // start the Foo
}

func (f *foo) isRunning() bool {
  return f.running == 1  // race!
}
```

- Good

```go
type foo struct {
  running atomic.Bool
}

func (f *foo) start() {
  if f.running.Swap(true) {
     // already running…
     return
  }
  // start the Foo
}

func (f *foo) isRunning() bool {
  return f.running.Load()
}
```

## 三 避免可变全局变量

使用选择依赖注入方式避免改变全局变量。 既适用于函数指针又适用于其他值类型

- Bad

```go
// sign.go
var _timeNow = time.Now
func sign(msg string) string {
  now := _timeNow()
  return signWithTime(msg, now)
}

// sign_test.go
func TestSign(t *testing.T) {
  oldTimeNow := _timeNow
  _timeNow = func() time.Time {
    return someFixedTime
  }
  defer func() { _timeNow = oldTimeNow }()
  assert.Equal(t, want, sign(give))
}
```

- Good

```go
// sign.go
type signer struct {
  now func() time.Time
}
func newSigner() *signer {
  return &signer{
    now: time.Now,
  }
}
func (s *signer) Sign(msg string) string {
  now := s.now()
  return signWithTime(msg, now)
}

// sign_test.go
func TestSigner(t *testing.T) {
  s := newSigner()
  s.now = func() time.Time {
    return someFixedTime
  }
  assert.Equal(t, want, s.Sign(give))
}
```



## 四 避免在公共结构中嵌入类型

这些嵌入的类型泄漏实现细节、禁止类型演化和模糊的文档。

假设您使用共享的 `AbstractList` 实现了多种列表类型，请避免在具体的列表实现中嵌入 `AbstractList`。 相反，只需手动将方法写入具体的列表，该列表将委托给抽象列表。

```
type AbstractList struct {}
// 添加将实体添加到列表中。
func (l *AbstractList) Add(e Entity) {
  // ...
}
// 移除从列表中移除实体。
func (l *AbstractList) Remove(e Entity) {
  // ...
}
```



- Bad

```go
// ConcreteList 是一个实体列表。
type ConcreteList struct {
  *AbstractList
}
```

- Good

```go
// ConcreteList 是一个实体列表。
type ConcreteList struct {
  list *AbstractList
}
// 添加将实体添加到列表中。
func (l *ConcreteList) Add(e Entity) {
  l.list.Add(e)
}
// 移除从列表中移除实体。
func (l *ConcreteList) Remove(e Entity) {
  l.list.Remove(e)
}
```

Go 允许 [类型嵌入](https://golang.org/doc/effective_go.html#embedding) 作为继承和组合之间的折衷。外部类型获取嵌入类型的方法的隐式副本。默认情况下，这些方法委托给嵌入实例的同一方法。

结构还获得与类型同名的字段。 所以，如果嵌入的类型是 public，那么字段是 public。为了保持向后兼容性，外部类型的每个未来版本都必须保留嵌入类型。

很少需要嵌入类型。 这是一种方便，可以帮助您避免编写冗长的委托方法。

即使嵌入兼容的抽象列表 *interface*，而不是结构体，这将为开发人员提供更大的灵活性来改变未来，但仍然泄露了具体列表使用抽象实现的细节。

- Bad

```go
// AbstractList 是各种实体列表的通用实现。
type AbstractList interface {
  Add(Entity)
  Remove(Entity)
}
// ConcreteList 是一个实体列表。
type ConcreteList struct {
  AbstractList
}
```

- Good

```go
// ConcreteList 是一个实体列表。
type ConcreteList struct {
  list AbstractList
}
// 添加将实体添加到列表中。
func (l *ConcreteList) Add(e Entity) {
  l.list.Add(e)
}
// 移除从列表中移除实体。
func (l *ConcreteList) Remove(e Entity) {
  l.list.Remove(e)
}
```

无论是使用嵌入结构还是嵌入接口，都会限制类型的演化。

- 向嵌入接口添加方法是一个破坏性的改变。
- 从嵌入结构体删除方法是一个破坏性改变。
- 删除嵌入类型是一个破坏性的改变。
- 即使使用满足相同接口的类型替换嵌入类型，也是一个破坏性的改变。

尽管编写这些委托方法是乏味的，但是额外的工作隐藏了实现细节，留下了更多的更改机会，还消除了在文档中发现完整列表接口的间接性操作。






