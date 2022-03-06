## Golang编码规范之规范03

## 一 使用字段名初始化结构体

初始化结构体时，应该指定字段名称。现在由 [`go vet`](https://golang.org/cmd/vet/) 强制执行。

* Bad

```go
k := User{"John", "Doe", true}
```

* Good

```go
k := User{
    FirstName: "John",
    LastName: "Doe",
    Admin: true,
}
```

例外：如果有 3 个或更少的字段，则可以在测试表中省略字段名称。

```go
tests := []struct{
  op Operation
  want string
}{
  {Add, "add"},
  {Subtract, "subtract"},
}
```

## 二 本地变量声明

如果将变量明确设置为某个值，则应使用短变量声明形式 (`:=`)。

- Bad

```go
var s = "foo"
```

- Good

```go
s := "foo"
```

但是，在某些情况下，`var` 使用关键字时默认值会更清晰。例如，[声明空切片](https://github.com/golang/go/wiki/CodeReviewComments#declaring-empty-slices)。

- Bad

```go
func f(list []int) {
  filtered := []int{}
  for _, v := range list {
    if v > 10 {
      filtered = append(filtered, v)
    }
  }
}
```

- Good

```go
func f(list []int) {
  var filtered []int
  for _, v := range list {
    if v > 10 {
      filtered = append(filtered, v)
    }
  }
}
```

## 三 nil 是一个有效的 slice

`nil` 是一个有效的长度为 0 的 slice，这意味着，

1. 您不应明确返回长度为零的切片。应该返回`nil` 来代替。

- Bad

```go
if x == "" {
  return []int{}
}
```

- Good

```go
if x == "" {
  return nil
}
```

2. 要检查切片是否为空，请始终使用`len(s) == 0`。而非 `nil`。

- Bad

```go
func isEmpty(s []string) bool {
  return s == nil
}
```

- Good

```go
func isEmpty(s []string) bool {
  return len(s) == 0
}
```

3. 零值切片（用`var`声明的切片）可立即使用，无需调用`make()`创建。

- Bad

```go
nums := []int{}
// or, nums := make([]int)

if add1 {
  nums = append(nums, 1)
}

if add2 {
  nums = append(nums, 2)
}
```

- Good

```go
var nums []int

if add1 {
  nums = append(nums, 1)
}

if add2 {
  nums = append(nums, 2)
}
```

记住，虽然 nil 切片是有效的切片，但它不等于长度为 0 的切片（一个为 nil，另一个不是），并且在不同的情况下（例如序列化），这两个切片的处理方式可能不同。

## 四 缩小变量作用域

如果有可能，尽量缩小变量作用范围。除非它与 [减少嵌套](https://github.com/xxjwxc/uber_go_guide_cn#减少嵌套)的规则冲突。

- Bad

```go
err := ioutil.WriteFile(name, data, 0644)
if err != nil {
 return err
}
```

- Good

```go
if err := ioutil.WriteFile(name, data, 0644); err != nil {
 return err
}
```

如果需要在 if 之外使用函数调用的结果，则不应尝试缩小范围。

- Bad

```go
if data, err := ioutil.ReadFile(name); err == nil {
  err = cfg.Decode(data)
  if err != nil {
    return err
  }

  fmt.Println(cfg)
  return nil
} else {
  return err
}
```

- Good

```go
data, err := ioutil.ReadFile(name)
if err != nil {
   return err
}

if err := cfg.Decode(data); err != nil {
  return err
}

fmt.Println(cfg)
return nil
```

## 五 避免参数语义不明确 (Avoid Naked Parameters)

函数调用中的`意义不明确的参数`可能会损害可读性。当参数名称的含义不明显时，请为参数添加 C 样式注释 (`/* ... */`)

- Bad

```go
// func printInfo(name string, isLocal, done bool)

printInfo("foo", true, true)
```

- Good

```go
// func printInfo(name string, isLocal, done bool)

printInfo("foo", true /* isLocal */, true /* done */)
```

对于上面的示例代码，还有一种更好的处理方式是将上面的 `bool` 类型换成自定义类型。将来，该参数可以支持不仅仅局限于两个状态（true/false）。

```
type Region int

const (
  UnknownRegion Region = iota
  Local
)

type Status int

const (
  StatusReady Status= iota + 1
  StatusDone
  // Maybe we will have a StatusInProgress in the future.
)

func printInfo(name string, region Region, status Status)
```

## 六 使用原始字符串字面值，避免转义

Go 支持使用 [原始字符串字面值](https://golang.org/ref/spec#raw_string_lit)，也就是 " ` " 来表示原生字符串，在需要转义的场景下，我们应该尽量使用这种方案来替换。

可以跨越多行并包含引号。使用这些字符串可以避免更难阅读的手工转义的字符串。

- Bad

```go
wantError := "unknown name:\"test\""
```

- Good

```go
wantError := `unknown error:"test"`
```

