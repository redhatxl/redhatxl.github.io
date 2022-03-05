## Golang编码规范之指导原则01

## 一 指向interface的指针

在Golang中接口实质上在底层用两个字段表示：

1. 一个指向某些特定类型信息的指针。您可以将其视为"type"。
2. 数据指针。如果存储的数据是指针，则直接存储。如果存储的数据是一个值，则存储指向该值的指针。

您几乎不需要指向接口类型的指针。您应该将接口作为值进行传递，在这样的传递过程中，实质上传递的底层数据仍然可以是指针。

如果希望接口方法修改基础数据，则必须使用指针传递 (将对象指针赋值给接口变量)。

```golang
type interfaceFunc interface {
  f()
}

type S1 struct{}

func (s S1) f() {}

type S2 struct{}

func (s *S2) f() {}

// f1.f() 无法修改底层数据
// f2.f() 可以修改底层数据，给接口变量 f2 赋值时使用的是对象指针
var f1 F = S1{}
var f2 F = &S2{}
```

## 二 Interface的合理性验证

在编译时验证接口的符合性。这包括：

- 将实现特定接口的导出类型作为接口 API 的一部分进行检查
- 实现同一接口的 (导出和非导出) 类型属于实现类型的集合
- 任何违反接口合理性检查的场景，都会终止编译，并通知给用户

补充：上面 3 条是编译器对接口的检查机制， 大体意思是错误使用接口会在编译期报错。 所以可以利用这个机制让部分问题在编译期暴露。

| Bad                                                          | Good                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `// 如果 Handler 没有实现 http.Handler，会在运行时报错 type Handler struct {   // ... } func (h *Handler) ServeHTTP(   w http.ResponseWriter,   r *http.Request, ) {   ... }` | `type Handler struct {   // ... } // 用于触发编译期的接口的合理性检查机制 // 如果 Handler 没有实现 http.Handler，会在编译期报错 var _ http.Handler = (*Handler)(nil) func (h *Handler) ServeHTTP(   w http.ResponseWriter,   r *http.Request, ) {   // ... }` |

如果 `*Handler` 与 `http.Handler` 的接口不匹配， 那么语句 `var _ http.Handler = (*Handler)(nil)` 将无法编译通过。

赋值的右边应该是断言类型的零值。 对于指针类型（如 `*Handler`）、切片和映射，这是 `nil`； 对于结构类型，这是空结构。

```go
type LogHandler struct {
  h   http.Handler
  log *zap.Logger
}
var _ http.Handler = LogHandler{}
func (h LogHandler) ServeHTTP(
  w http.ResponseWriter,
  r *http.Request,
) {
  // ...
}
```

## 三 接收器与接口

使用值接收器的方法既可以通过值调用，也可以通过指针调用。

带指针接收器的方法只能通过指针或 [addressable values](https://golang.org/ref/spec#Method_values) 调用。

```
type S struct {
  data string
}

func (s S) Read() string {
  return s.data
}

func (s *S) Write(str string) {
  s.data = str
}

sVals := map[int]S{1: {"A"}}

// 你只能通过值调用 Read
sVals[1].Read()

// 这不能编译通过：
//  sVals[1].Write("test")

sPtrs := map[int]*S{1: {"A"}}

// 通过指针既可以调用 Read，也可以调用 Write 方法
sPtrs[1].Read()
sPtrs[1].Write("test")
```

类似的，即使方法有了值接收器，也同样可以用指针接收器来满足接口。

```
type F interface {
  f()
}

type S1 struct{}

func (s S1) f() {}

type S2 struct{}

func (s *S2) f() {}

s1Val := S1{}
s1Ptr := &S1{}
s2Val := S2{}
s2Ptr := &S2{}

var i F
i = s1Val
i = s1Ptr
i = s2Ptr

//  下面代码无法通过编译。因为 s2Val 是一个值，而 S2 的 f 方法中没有使用值接收器
//   i = s2Val
```

[Effective Go](https://golang.org/doc/effective_go.html) 中有一段关于 [pointers vs. values](https://golang.org/doc/effective_go.html#pointers_vs_values) 的精彩讲解。

补充：

- 一个类型可以有值接收器方法集和指针接收器方法集
  - 值接收器方法集是指针接收器方法集的子集，反之不是
- 规则
  - 值对象只可以使用值接收器方法集
  - 指针对象可以使用 值接收器方法集 + 指针接收器方法集
- 接口的匹配 (或者叫实现)
  - 类型实现了接口的所有方法，叫匹配
  - 具体的讲，要么是类型的值方法集匹配接口，要么是指针方法集匹配接口

具体的匹配分两种：

- 值方法集和接口匹配
  - 给接口变量赋值的不管是值还是指针对象，都 ok，因为都包含值方法集
- 指针方法集和接口匹配
  - 只能将指针对象赋值给接口变量，因为只有指针方法集和接口匹配
  - 如果将值对象赋值给接口变量，会在编译期报错 (会触发接口合理性检查机制)

为啥 i = s2Val 会报错，因为值方法集和接口不匹配。

## 四 零值 Mutex 是有效的

零值 `sync.Mutex` 和 `sync.RWMutex` 是有效的。所以指向 mutex 的指针基本是不必要的。

| Bad                               | Good                          |
| --------------------------------- | ----------------------------- |
| `mu := new(sync.Mutex) mu.Lock()` | `var mu sync.Mutex mu.Lock()` |

如果你使用结构体指针，mutex 应该作为结构体的非指针字段。即使该结构体不被导出，也不要直接把 mutex 嵌入到结构体中。

| Bad                                                          | Good                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `type SMap struct {   sync.Mutex    data map[string]string }  func NewSMap() *SMap {   return &SMap{     data: make(map[string]string),   } }  func (m *SMap) Get(k string) string {   m.Lock()   defer m.Unlock()    return m.data[k] }` | `type SMap struct {   mu sync.Mutex    data map[string]string }  func NewSMap() *SMap {   return &SMap{     data: make(map[string]string),   } }  func (m *SMap) Get(k string) string {   m.mu.Lock()   defer m.mu.Unlock()    return m.data[k] }` |
| `Mutex` 字段， `Lock` 和 `Unlock` 方法是 `SMap` 导出的 API 中不刻意说明的一部分。 | mutex 及其方法是 `SMap` 的实现细节，对其调用者不可见。       |











