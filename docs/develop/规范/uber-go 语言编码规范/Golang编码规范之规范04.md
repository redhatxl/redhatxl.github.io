## Golang编码规范之规范04

## 一 初始化结构体

### 1.1 使用字段名初始化结构

初始化结构时，几乎应该始终指定字段名。目前由 [`go vet`](https://golang.org/cmd/vet/) 强制执行。

- Bad

```go
k := User{"John", "Doe", true}
```

- Good

```go
k := User{
    FirstName: "John",
    LastName: "Doe",
    Admin: true,
}
```

例外：当有 3 个或更少的字段时，测试表中的字段名*may*可以省略。

```
tests := []struct{
  op Operation
  want string
}{
  {Add, "add"},
  {Subtract, "subtract"},
}
```

### 1.2 省略结构中的零值字段

初始化具有字段名的结构时，除非提供有意义的上下文，否则忽略值为零的字段。 也就是，让我们自动将这些设置为零值

- Bad

```go
user := User{
  FirstName: "John",
  LastName: "Doe",
  MiddleName: "",
  Admin: false,
}
```

- Good

```go
user := User{
  FirstName: "John",
  LastName: "Doe",
}
```

这有助于通过省略该上下文中的默认值来减少阅读的障碍。只指定有意义的值。

在字段名提供有意义上下文的地方包含零值。例如，[表驱动测试](https://github.com/xxjwxc/uber_go_guide_cn#表驱动测试) 中的测试用例可以受益于字段的名称，即使它们是零值的。

```go
tests := []struct{
  give string
  want int
}{
  {give: "0", want: 0},
  // ...
}
```

### 1..3 对零值结构使用 `var`

如果在声明中省略了结构的所有字段，请使用 `var` 声明结构。

- Bad

```go
user := User{}
```

- Good

```go
var user User
```

这将零值结构与那些具有类似于为 [初始化 Maps](https://github.com/xxjwxc/uber_go_guide_cn#初始化-maps) 创建的，区别于非零值字段的结构区分开来， 并与我们更喜欢的 [声明空切片](https://github.com/golang/go/wiki/CodeReviewComments#declaring-empty-slices) 方式相匹配。

### 1.4 初始化 Struct 引用

在初始化结构引用时，请使用`&T{}`代替`new(T)`，以使其与结构体初始化一致。

- Bad

```go
sval := T{Name: "foo"}

// inconsistent
sptr := new(T)
sptr.Name = "bar"
```

- Good

```go
sval := T{Name: "foo"}

sptr := &T{Name: "bar"}
```

## 二 初始化 Maps

对于空 map 请使用 `make(..)` 初始化， 并且 map 是通过编程方式填充的。 这使得 map 初始化在表现上不同于声明，并且它还可以方便地在 make 后添加大小提示。

- Bad

声明和初始化看起来非常相似的

```go
var (
  // m1 读写安全;
  // m2 在写入时会 panic
  m1 = map[T1]T2{}
  m2 map[T1]T2
)
```

- Good

声明和初始化看起来差别非常大。

```go
var (
  // m1 读写安全;
  // m2 在写入时会 panic
  m1 = make(map[T1]T2)
  m2 map[T1]T2
)
```



- Bad

```go
m := make(map[T1]T2, 3)
m[k1] = v1
m[k2] = v2
m[k3] = v3
```

- Good

```go
m := map[T1]T2{
  k1: v1,
  k2: v2,
  k3: v3,
}
```

基本准则是：在初始化时使用 map 初始化列表 来添加一组固定的元素。否则使用 `make` (如果可以，请尽量指定 map 容量)。

## 三 字符串 string format

如果你在函数外声明`Printf`-style 函数的格式字符串，请将其设置为`const`常量。

这有助于`go vet`对格式字符串执行静态分析。

- Bad

```go
msg := "unexpected values %v, %v\n"
fmt.Printf(msg, 1, 2)
```

- Good

```go
const msg = "unexpected values %v, %v\n"
fmt.Printf(msg, 1, 2)
```

## 四 命名 Printf 样式的函数

声明`Printf`-style 函数时，请确保`go vet`可以检测到它并检查格式字符串。

这意味着您应尽可能使用预定义的`Printf`-style 函数名称。`go vet`将默认检查这些。有关更多信息，请参见 [Printf 系列](https://golang.org/cmd/vet/#hdr-Printf_family)。

如果不能使用预定义的名称，请以 f 结束选择的名称：`Wrapf`，而不是`Wrap`。`go vet`可以要求检查特定的 Printf 样式名称，但名称必须以`f`结尾。

```
$ go vet -printfuncs=wrapf,statusf
```

另请参阅 [go vet: Printf family check](https://kuzminva.wordpress.com/2017/11/07/go-vet-printf-family-check/).





