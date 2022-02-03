# Go 内存对齐



```go
package main

import (
	"fmt"
	"unsafe"
)
// mem align
type Person struct {
	a byte
	b int32
	c int64
}

type Person2 struct {
	a byte
	b int64
	c int32
}

func main() {
	p := Person{}
	fmt.Println(unsafe.Alignof(p)) // 8
	fmt.Println(unsafe.Sizeof(p))  // 16

	p2 := Person2{}
	fmt.Println(unsafe.Alignof(p2)) // 8
	fmt.Println(unsafe.Sizeof(p2))  // 24
}

```



# 参考链接

* https://juejin.cn/post/6975840893676814344