## Golang编码规范之指导原则03

## 一 Errors

### 1.1 错误类型

声明错误的选项很少。 在选择最适合您的用例的选项之前，请考虑以下事项。

- 调用者是否需要匹配错误以便他们可以处理它？ 如果是，我们必须通过声明顶级错误变量或自定义类型来支持 [`errors.Is`](https://golang.org/pkg/errors/#Is) 或 [`errors.As`](https://golang.org/pkg/errors/#As) 函数。
- 错误消息是否为静态字符串，还是需要上下文信息的动态字符串？ 如果是静态字符串，我们可以使用 [`errors.New`](https://golang.org/pkg/errors/#New)，但对于后者，我们必须使用 [`fmt.Errorf`](https://golang.org/pkg/fmt/#Errorf) 或自定义错误类型。
- 我们是否正在传递由下游函数返回的新错误？ 如果是这样，请参阅[错误包装部分](https://github.com/xxjwxc/uber_go_guide_cn#错误包装)。

| 错误匹配？ | 错误消息 | 指导                                                         |
| ---------- | -------- | ------------------------------------------------------------ |
| No         | static   | [`errors.New`](https://golang.org/pkg/errors/#New)           |
| No         | dynamic  | [`fmt.Errorf`](https://golang.org/pkg/fmt/#Errorf)           |
| Yes        | static   | top-level `var` with [`errors.New`](https://golang.org/pkg/errors/#New) |
| Yes        | dynamic  | custom `error` type                                          |

例如， 使用 [`errors.New`](https://golang.org/pkg/errors/#New) 表示带有静态字符串的错误。 如果调用者需要匹配并处理此错误，则将此错误导出为变量以支持将其与 `errors.Is` 匹配。

* 无错误匹配

```go
// package foo

func Open() error {
  return errors.New("could not open")
}

// package bar

if err := foo.Open(); err != nil {
  // Can't handle the error.
  panic("unknown error")
}
```

* 错误匹配

```go
// package foo

var ErrCouldNotOpen = errors.New("could not open")

func Open() error {
  return ErrCouldNotOpen
}

// package bar

if err := foo.Open(); err != nil {
  if errors.Is(err, foo.ErrCouldNotOpen) {
    // handle the error
  } else {
    panic("unknown error")
  }
}
```

对于动态字符串的错误， 如果调用者不需要匹配它，则使用 [`fmt.Errorf`](https://golang.org/pkg/fmt/#Errorf)， 如果调用者确实需要匹配它，则自定义 `error`。

- 无错误匹配

```go
// package foo

func Open(file string) error {
  return fmt.Errorf("file %q not found", file)
}

// package bar

if err := foo.Open("testfile.txt"); err != nil {
  // Can't handle the error.
  panic("unknown error")
}
```

- 错误匹配

```go
// package foo

type NotFoundError struct {
  File string
}

func (e *NotFoundError) Error() string {
  return fmt.Sprintf("file %q not found", e.File)
}

func Open(file string) error {
  return &NotFoundError{File: file}
}


// package bar

if err := foo.Open("testfile.txt"); err != nil {
  var notFound *NotFoundError
  if errors.As(err, &notFound) {
    // handle the error
  } else {
    panic("unknown error")
  }
}
```

请注意，如果您从包中导出错误变量或类型， 它们将成为包的公共 API 的一部分。

### 1.2 错误包装

There are three main options for propagating errors if a call fails: 如果调用失败，有三种主要的错误调用选项：

- 按原样返回原始错误
- add context with `fmt.Errorf` and the `%w` verb
- 使用`fmt.Errorf`和`%w`
- 使用 `fmt.Errorf` 和 `%v`

如果没有要添加的其他上下文，则按原样返回原始错误。 这将保留原始错误类型和消息。 这非常适合底层错误消息有足够的信息来追踪它来自哪里的错误。

否则，尽可能在错误消息中添加上下文 这样就不会出现诸如“连接被拒绝”之类的模糊错误， 您会收到更多有用的错误，例如“呼叫服务 foo：连接被拒绝”。

使用 `fmt.Errorf` 为你的错误添加上下文， 根据调用者是否应该能够匹配和提取根本原因，在 `%w` 或 `%v` 动词之间进行选择。

- 如果调用者应该可以访问底层错误，请使用 `%w`。 对于大多数包装错误，这是一个很好的默认值， 但请注意，调用者可能会开始依赖此行为。因此，对于包装错误是已知`var`或类型的情况，请将其作为函数契约的一部分进行记录和测试。
- 使用 `%v` 来混淆底层错误。 调用者将无法匹配它，但如果需要，您可以在将来切换到 `%w`。

在为返回的错误添加上下文时，通过避免使用"failed to"之类的短语来保持上下文简洁，当错误通过堆栈向上渗透时，它会一层一层被堆积起来：

* Bad

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "failed to create new store: %w", err)
}


// failed to x: failed to y: failed to create new store: the error
```

* Good

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "new store: %w", err)
}

// x: y: new store: the error

```

然而，一旦错误被发送到另一个系统，应该清楚消息是一个错误（例如`err` 标签或日志中的"Failed"前缀）。

另见 [不要只检查错误，优雅地处理它们](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)。

### 1.3 错误命名

对于存储为全局变量的错误值， 根据是否导出，使用前缀 `Err` 或 `err`。 请看指南 [对于未导出的顶层常量和变量，使用_作为前缀](https://github.com/xxjwxc/uber_go_guide_cn#对于未导出的顶层常量和变量使用_作为前缀)。

```
var (
  // 导出以下两个错误，以便此包的用户可以将它们与 errors.Is 进行匹配。

  ErrBrokenLink = errors.New("link is broken")
  ErrCouldNotOpen = errors.New("could not open")

  // 这个错误没有被导出，因为我们不想让它成为我们公共 API 的一部分。 我们可能仍然在带有错误的包内使用它。

  errNotFound = errors.New("not found")
)
```

对于自定义错误类型，请改用后缀 `Error`。

```
// 同样，这个错误被导出，以便这个包的用户可以将它与 errors.As 匹配。

type NotFoundError struct {
  File string
}

func (e *NotFoundError) Error() string {
  return fmt.Sprintf("file %q not found", e.File)
}

// 并且这个错误没有被导出，因为我们不想让它成为公共 API 的一部分。 我们仍然可以在带有 errors.As 的包中使用它。
type resolveError struct {
  Path string
}

func (e *resolveError) Error() string {
  return fmt.Sprintf("resolve %q", e.Path)
}
```



## 二 处理断言失败

[类型断言](https://golang.org/ref/spec#Type_assertions) 将会在检测到不正确的类型时，以单一返回值形式返回 panic。 因此，请始终使用“逗号 ok”习语。

* Bad

```go
t := i.(string)
```

* Good

```go
t, ok := i.(string)
if !ok {
  // 优雅地处理错误
}
```



